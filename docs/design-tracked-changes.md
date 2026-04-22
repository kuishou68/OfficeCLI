# OfficeCLI Tracked Changes (校对修订) Implementation Plan

## Background

cove-desktop 当前的校对流程直接在 Rust 中操作 DOCX ZIP:

```
读 zip → 正则定位段落 → 拆 run → 包 w:del/w:ins → 写 settings.xml → 回写 zip
```

这条链路绕过了 OpenXML SDK，对文档结构破坏性大、维护成本高。
目标是将校对能力下沉到 OfficeCLI，通过 OpenXML SDK 的对象模型实现，
cove-desktop 只需调用 CLI 命令。

## Architecture

### OfficeCLI 已有能力

| 能力 | 状态 | 位置 |
|------|------|------|
| find/replace 文本 | 有 | WordHandler.Helpers.cs `ProcessFind()` |
| 段落导航 (XPath) | 有 | WordHandler.Navigation.cs `NavigateToElement()` |
| settings.xml 写入 | 有 | WordHandler.Set.DocSettings.cs `TrySetDocSetting()` |
| accept all changes | 有 | WordHandler.Mutations.cs `AcceptAllChanges()` |
| reject all changes | 有 | WordHandler.Mutations.cs `RejectAllChanges()` |
| 批注创建 | 有 | WordHandler.Add.cs 中有 comment 相关逻辑 |

### 缺失能力（本次实现）

| 能力 | 优先级 |
|------|--------|
| find/replace 时生成 w:del + w:ins 修订标记 | P0 |
| 设置 trackRevisions 等审阅标志到 settings.xml | P0 |
| 自动为每处修订添加批注（"校对修订：X → Y"） | P1 |
| accept-revisions / reject-revisions CLI 命令 | P1 |
| 查询修订列表 (query revision) | P2 |

## Implementation Plan

### Phase 1: Tracked Find/Replace (P0)

**目标**: `set` 命令支持 `tracked=true` 参数，find/replace 时生成修订标记。

#### 1.1 新增文件 `WordHandler.Set.TrackedPatch.cs`

```
src/officecli/Handlers/Word/WordHandler.Set.TrackedPatch.cs
```

核心方法:

```csharp
/// <summary>
/// 对目标段落执行带修订标记的 find/replace。
/// 找到 find 文本后，将匹配部分包装为 w:del（标记删除），
/// 紧跟插入 w:ins 包裹的 replace 文本。
/// </summary>
private int ProcessTrackedFind(
    string path,
    string find,
    string replace,
    string author,
    DateTime date)
```

实现步骤:

1. 调用 `ResolveParagraphsForFind(path)` 获取目标段落列表
2. 对每个段落:
   a. `BuildRunTexts()` 拼接所有 run 的文本
   b. 定位 find 子串在拼接文本中的位置
   c. 计算 find 跨越了哪些 run，在 run 边界处拆分
   d. 将匹配的 run 序列包装为 `DeletedRun` (w:del):
      - 复制原始 run，将 `w:t` 改为 `w:delText`
      - 设置 author、date、w:id
   e. 在 `DeletedRun` 之后插入 `InsertedRun` (w:ins):
      - 创建新 run，复制第一个匹配 run 的 rPr
      - 设置 `w:t` 为 replace 文本
      - 设置 author、date、w:id
   f. 移除原始匹配 run
3. 返回替换计数

**参考代码**:
- 现有 `ProcessFindInParagraph()` (WordHandler.Helpers.cs:813-891) 的 run 拆分逻辑
- docmod `RevisionCommand.cs` 的 accept/reject 模式（反向参考）
- OpenXML SDK 类型: `InsertedRun`, `DeletedRun`, `DeletedText`

#### 1.2 修改 `WordHandler.Set.cs`

在 `Set()` 方法的 find 分支（约 line 35-85）中:

```csharp
// 现有代码
if (props.TryGetValue("find", out var findValue))
{
    // ...现有 ProcessFind 逻辑...
}

// 新增: 检查 tracked 标志
bool tracked = props.TryGetValue("tracked", out var tv) 
    && (tv == "true" || tv == "1");

if (tracked && !string.IsNullOrEmpty(findValue))
{
    string author = props.GetValueOrDefault("author", "Cove");
    string replace = props.GetValueOrDefault("replace", "");
    int count = ProcessTrackedFind(path, findValue, replace, author, DateTime.Now);
    // 自动开启审阅模式
    EnsureTrackRevisionSettings();
    _doc.MainDocumentPart?.Document?.Save();
    // 返回 match count 通过 props["_matchCount"] 传递
    return unsupported;
}
```

#### 1.3 新增 `EnsureTrackRevisionSettings()` in DocSettings

在 `WordHandler.Set.DocSettings.cs` 中添加:

```csharp
/// <summary>
/// 确保 settings.xml 中包含审阅显示标志。
/// 与 docmod 的 EnsureRevisionViewSettings 行为一致:
///   <w:trackChanges/> + <w:showDel/> + <w:showIns/> + <w:showMarkupBar/>
/// </summary>
private void EnsureTrackRevisionSettings()
{
    var settings = EnsureSettings();
    var root = settings.Settings;
    // idempotent: only add if not present
    if (root.GetFirstChild<TrackChanges>() == null)
        root.AppendChild(new TrackChanges());
    // ShowDel, ShowIns, ShowMarkupBar (W14 namespace variants if needed)
    // ...
    settings.Save();
}
```

### Phase 2: CLI Command Registration (P1)

#### 2.1 accept-revisions / reject-revisions 命令

在 `CommandBuilder.cs` 中注册顶层命令:

```
officecli accept-revisions <file> [--json]
officecli reject-revisions <file> [--json]
```

实现: 调用已有的 `AcceptAllChanges()` / `RejectAllChanges()`，
包装标准的 JSON 输出格式。

也可以通过 `set` 属性方式:

```
officecli set <file> / --prop acceptRevisions=true --json
officecli set <file> / --prop rejectRevisions=true --json
```

#### 2.2 批注自动添加 (P1)

在 `ProcessTrackedFind` 中，每处替换后:
1. 创建 `CommentRangeStart` + `CommentRangeEnd` 包裹修订区域
2. 在 `CommentRangeEnd` 后插入 `CommentReference` run
3. 在 `WordprocessingCommentsPart` 中添加 `Comment` 元素
4. 批注文本格式: `校对修订："{find}" → "{replace}"`

### Phase 3: Query Support (P2)

```
officecli query <file> revision --json
```

返回:
```json
{
  "results": [
    {
      "type": "insert|delete",
      "path": "/body/p[3]/ins[1]",
      "author": "Cove",
      "date": "2026-04-13T00:00:00Z",
      "text": "..."
    }
  ]
}
```

## CLI 调用示例

### 校对替换（带修订标记）

```bash
# 单次替换
officecli set doc.docx /body/p[3] \
  --prop "find=原文本" \
  --prop "replace=修改后文本" \
  --prop "tracked=true" \
  --prop "author=Cove" \
  --json

# 批量替换 (via batch)
officecli batch doc.docx --commands '[
  {"command":"set","path":"/body/p[3]","props":{"find":"错别字","replace":"正确字","tracked":"true","author":"Cove"}},
  {"command":"set","path":"/body/p[5]","props":{"find":"旧表述","replace":"新表述","tracked":"true","author":"Cove"}}
]' --json
```

### 接受/拒绝修订

```bash
officecli set doc.docx / --prop "acceptRevisions=true" --json
officecli set doc.docx / --prop "rejectRevisions=true" --json
```

## cove-desktop 集成变更

完成后 cove-desktop 侧的改动:

1. **`adapter.rs`**: 删除 `tracked_patch_xml()` 和 `apply_revision_xml()`
2. **`officecli_apply_text_ops()`**: 校对 op 改为调用 OfficeCLI:
   ```
   cli::run_cli(&["set", file, path,
     "--prop", &format!("find={find}"),
     "--prop", &format!("replace={replace}"),
     "--prop", "tracked=true",
     "--prop", "author=Cove",
     "--json"])
   ```
3. **`officecli_accept_revisions()`**: 改为调用:
   ```
   cli::run_cli(&["set", file, "/", "--prop", "acceptRevisions=true", "--json"])
   ```

## File Checklist

### New files
- [ ] `src/officecli/Handlers/Word/WordHandler.Set.TrackedPatch.cs`

### Modified files
- [ ] `src/officecli/Handlers/Word/WordHandler.Set.cs` — tracked 分支路由
- [ ] `src/officecli/Handlers/Word/WordHandler.Set.DocSettings.cs` — EnsureTrackRevisionSettings
- [ ] `src/officecli/Handlers/Word/WordHandler.Helpers.cs` — 可能需要提取 run 拆分为公共方法

### Test plan
1. 单段落单 run 的 find/replace → 验证 w:del + w:ins 结构
2. 跨 run 的 find/replace → 验证 run 拆分正确性
3. 同一段落多处匹配 → 验证 ID 不冲突
4. 空 replace（纯删除）→ 只生成 w:del
5. settings.xml 审阅标志 → 用 WPS/Word 打开验证审阅模式
6. batch 批量校对 → 验证多条命令顺序执行
7. accept/reject → 验证文本恢复正确
