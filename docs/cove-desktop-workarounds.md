# OfficeCLI 魔改方案（Workarounds）

> **镜像副本**：本文件在 `OfficeCLI/docs/cove-desktop-workarounds.md` 有一份同步副本，供上游维护者查阅。
>
> **Source of truth**：本文件（cove-desktop 侧）是主文件。任一条目新增/修改/移除，必须同步更新 OfficeCLI 仓库侧副本。
>
> **当前基线**：
> - 仓库当前本地 OfficeCLI sidecar 基线为 `v1.0.46`
> - `cove-desktop` 中已提交的 macOS arm64 sidecar 由 commit `d3e8dfa` 重建
> - `release.yml` 默认 `OFFICECLI_REF` 也应保持与该版本一致，避免本地 sidecar 与 CI 构建来源漂移

本文件记录 cove-desktop 的 OfficeCLI 适配层（`src-tauri/src/officecli/`）相对上游 OfficeCLI 的所有魔改、绕过、补丁。每项改动都必须说明：

1. **What**：改了什么
2. **Why**：上游 OfficeCLI 的什么行为有缺陷 / 缺失 / 与 cove-desktop 需求不一致
3. **Solves**：解决了 cove-desktop 的什么具体问题
4. **Location**：文件路径 + 代码位置
5. **Upstream note**：上游是否有对应 issue / 何时可以移除

> **维护原则**：任何新增的魔改必须先在此文件登记，再动代码。移除魔改前需验证上游已修复，否则保留。

---

## Template（新增条目时拷贝这个模板）

### [短标题]

- **What**：一句话描述改了什么
- **Why**：上游行为（版本、具体表现）
- **Solves**：cove-desktop 的具体场景 + issue/commit 引用
- **Location**：`path/to/file.rs:行号` + 关键函数名
- **Added**：日期 + commit hash
- **Upstream note**：是否已提 issue / 预期何时可删

---

## 当前魔改清单

### 1. Query JSON 字段名大小写兼容（`Results`/`results`、`Matches`/`matches`）

- **What**：所有 `query ... --json` 的结果解析都用 `.get("Results").or_else(|| d.get("results"))` 兼容两种大小写
- **Why**：OfficeCLI 从旧版（PascalCase `Results`/`Matches`）升级到 v1.0.43 后改为小写 `results`/`matches`。字段名未走 feature flag，直接 break change
- **Solves**：
  - `doc_read` 返回 `total=0`，导致 `导入到文档` 按钮点击后第一步就 abort
  - 所有 doc_edit / doc_comments / apply_text_ops 都依赖 query 构建段落 ID 映射，字段名错 → 映射空 → 所有 op 报 `paragraph id not found`
- **Location**：
  - `adapter.rs:238` `query_all_paragraphs`
  - `adapter.rs:273` `query_bookmarks`
  - `adapter.rs:460` `cleanup_tracked_patch_comment`
  - `adapter_comments.rs:191` `parse_comments_json`
  - `adapter_comments.rs:432` `find_anchor_path`
  - `mod.rs:416` `add_comment_by_find`
  - `mod.rs:551` test: comment count query
- **Added**：2026-04-13，commit `1e8d987`
- **Upstream note**：字段名属于上游 API 演进，不会回滚。保留兼容代码直到所有环境升级到 v1.0.43+，届时可删大写分支

---

### 2. 清理 tracked patch 自动生成的批注（"校对修订"）

- **What**：`OpKind::Patch` 调用 `set --prop tracked=true` 成功后，立即 query 并删除 OfficeCLI 自动附带的批注（identified by `author=Cove` + anchoredTo target + text 前缀 `"校对修订"`）
- **Why**：OfficeCLI v1.0.43+ 在执行 `set --prop tracked=true` 时无条件创建一条批注（text 形如 `"校对修订：\"旧\" → \"新\""`），且没有选项关闭。尝试过 `addComment=false` / `withComment=false` / `revisionComment=false` / `comment=false` / `trackComment=false` / `noComment=true` 全部被忽略
- **Solves**：`校对-修订` 快捷指令场景下，用户只要修订标记（w:del + w:ins），不要批注气泡。否则每次校对都在审阅面板产生一大堆 "校对修订：..." 批注干扰
- **Location**：
  - `adapter.rs:499-505` `OpKind::Patch` 分支调用点
  - `adapter.rs:449-487` `cleanup_tracked_patch_comment` helper
- **Added**：2026-04-14（本次）
- **Upstream note**：应向 OfficeCLI 提 issue 请求增加 `trackComment=false` 选项。届时可删除 helper 并改传该 prop

---

### 3. 多段 insertAfter 同锚点的反序发送

- **What**：导入多行内容时，所有 `insertAfter` ops 都 anchor 到文档最后一段 ID，但数组构造时整体 `.reverse()` 一次
- **Why**：OfficeCLI 对同一锚点的多次 insertAfter 按到达顺序执行，每次都插入到紧随锚点之后，导致最终顺序颠倒
- **Solves**：`导入到文档` 按钮追加多段内容时，段落顺序与 AI 输出一致
- **Location**：`src/components/chat/AssistantMessage.tsx` 的 `handleImportToDoc`（前端侧的逻辑，未在 officecli 模块内）
- **Added**：2026-04-13，commit `0a5ad6e`
- **Upstream note**：非上游问题，是 OfficeCLI ordered-ops API 的自然行为。保留前端 reverse

---

### 4. Update/Rewrite 产生无追踪替换

- **What**：`OpKind::Update` 和 `OpKind::Rewrite` 都调用 `set --prop text=...`，无 tracked 标记
- **Why**：上游 OfficeCLI `set` 命令对 paragraph text 替换不支持 trackChanges。我们只在 Patch（find/replace 粒度）能做修订
- **Solves**：已知限制，docmod 的 update 会产生 w:del+w:ins，迁移到 officecli 后 update 变成 clean replace。semantic 仍保留在 op.kind，等上游支持后可修
- **Location**：`adapter.rs:545-555` `OpKind::Update | OpKind::Rewrite` 分支
- **Added**：OfficeCLI 初始接入即存在
- **Upstream note**：等待上游给 `set` 加 `tracked=true` 支持段落级替换

---

### 5. Paragraph ID 失效时按 originalText 兜底扫描

- **What**：`resolve_path` 优先用 `pN` 查找 id_to_para 映射，miss 时按 `op.originalText.trim()` 精确匹配段落文字
- **Why**：OfficeCLI 的段落 path（如 `/body/p[@paraId=XXXX]` 或 `/body/p[3]`）在文档编辑后可能漂移 / 失效。模型基于 doc_read 的 `pN` ID 下发，但多批 op 之间文档已被改动
- **Solves**：
  - doc_edit 大量段落连续修改时的鲁棒性
  - 模型重试或拼接多轮 ops 时不会因 ID 过期全军覆没
  - 对齐 cove-wps 的 ParaID/text-hash 双轨策略
- **Location**：`adapter.rs:416-447` `resolve_path`
- **Added**：OfficeCLI 初始接入即存在
- **Upstream note**：cove-desktop 设计取舍，不是上游问题

---

### 6. Template 应用后的 CJK 字体后处理（已移除）

- **What**：曾经在 `apply_template` 成功后调用 `split_cjk_latin_fonts` 把 ascii 字体拆成与 CJK 字体独立设置
- **Why**：JS skills 产出的文档 ascii 和 CJK 字体不一致，与 cove-wps 渲染对不上
- **Solves**：`已移除` — 改由 JS skills 自己直接把 ascii font 设成 CJK font name（镜像 cove-wps 行为），不再需要 Rust 侧后处理
- **Location**：~~`template.rs:60-72`~~（已删除）
- **Removed**：2026-04-14（本次），commit `047f47d`
- **Upstream note**：行为对齐 cove-wps，不涉及上游 OfficeCLI

---

### 7. HTML 预览输出兼容（JSON 优先 + 临时文件路径回退）

- **What**：`officecli_view_html` 不再直接消费 `view <file> html` 的原始 stdout，而是改为 `view <file> html --json`；若返回体仍然是临时 `.html` 文件路径或 `file:///...html` URI，则读取文件内容后再返回给前端 iframe
- **Why**：OfficeCLI `html` 预览在不同环境下输出契约不稳定，有时是原始 HTML，有时是 JSON 包装，有时又只给临时 HTML 路径。路径场景下，桌面端会被系统当作本地文件打开，导致预览面板之外突然跳浏览器
- **Solves**：
  - 修复 macOS 预览 `.docx` 时跳出系统浏览器打开 `officecli_preview_*.html`
  - 兼容 UOS / 麒麟验证时出现过的 “预览返回非 HTML 正文” 问题，避免同类回归
- **Location**：
  - `mod.rs:85` `officecli_view_html`
  - `mod.rs:108` `extract_preview_html`
  - `mod.rs:128` `parse_preview_html_json`
  - `mod.rs:174` `extract_preview_html_path`
- **Added**：2026-04-14（本次）
- **Upstream note**：应推动 OfficeCLI 固化 `view ... html --json` 的返回协议，始终返回 HTML 正文而不是路径。上游修复后，可删除路径解析回退逻辑

---

## 流程规范

### 新增魔改时必须做的事

1. **先定位根因**：用实际 OfficeCLI 命令复现问题（例如 `/usr/bin/officecli query file.docx p --json` 直接看输出），不要仅凭报错猜
2. **查上游是否有配置**：测试所有可能的 prop 名变体（本文件 §2 就测过 6 种），确认无法通过上游参数规避
3. **代码内加上引用注释**：改动处注释必须写 `WORKAROUND(#N): see WORKAROUNDS.md`，N 为本文件章节号
4. **更新此文件**：按 Template 结构登记新条目，给出 file:line、commit hash、Added 日期
5. **Best-effort 降级**：魔改本身失败时不要抛错中断主流程，打 `eprintln!` warn 即可（参考 `cleanup_tracked_patch_comment` 的实现）

### 移除魔改时必须做的事

1. 确认上游版本已修复对应问题（跑真实 OfficeCLI 命令验证）
2. 更新 cove-desktop 捆绑的 officecli 二进制版本
3. 删除魔改代码同时更新本文件对应条目为 `Removed` 状态，保留历史记录
4. 测试 cove-desktop 所有涉及 officecli 的场景（doc_read / doc_edit / doc_comments / doc_format / 导入到文档 / 校对修订 等）

### 何时不该魔改

- 上游行为虽然不符合直觉但**未破坏** cove-desktop 任何用户可见场景
- 魔改成本超过向上游提 PR 的成本
- 魔改需要解析 OfficeCLI 非公开的实现细节（fragile，容易随版本失效）

以上场景优先走：**向 OfficeCLI 提 issue → 调整 cove-desktop 用法 / 文案 → 最后才魔改**。
