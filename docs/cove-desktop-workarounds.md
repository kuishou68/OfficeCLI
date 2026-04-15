# cove-desktop 对 OfficeCLI 的魔改清单（镜像副本）

> **本文件来源**：`cove-desktop/src-tauri/src/officecli/WORKAROUNDS.md`
>
> **目的**：让 OfficeCLI 维护者在不切换仓库的前提下，掌握下游 cove-desktop 为何要对上游行为打补丁。每一条魔改都是潜在的上游 issue 候选 —— 如能在上游修复或暴露配置，下游就能删掉对应绕过代码。
>
> **同步约定**：两处文件保持完全一致。任一侧新增/移除魔改，必须同步更新另一侧。cove-desktop 侧是 source of truth，OfficeCLI 这份是 mirror。

---

## 结构说明

每个条目必须说明：

1. **What**：改了什么
2. **Why**：上游 OfficeCLI 的什么行为有缺陷 / 缺失 / 与 cove-desktop 需求不一致
3. **Solves**：解决了 cove-desktop 的什么具体问题
4. **Location**：cove-desktop 侧文件路径 + 代码位置
5. **Upstream note**：是否已提 issue / 预期何时可以移除

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
- **Added**：2026-04-14
- **Upstream note**：**建议上游新增 `trackComment=false` prop 关闭自动批注**。届时下游可删除 helper 并改传该 prop

---

### 3. 多段 insertAfter 同锚点的反序发送（已移除）

- **What**：曾在“导入到文档”入口中，对多行 `insertAfter` ops 做整体 `.reverse()` 后再提交
- **Why**：OfficeCLI 对同一锚点的多次 insertAfter 按到达顺序执行，每次都插入到紧随锚点之后，导致最终顺序颠倒
- **Solves**：`已移除` — 聊天消息上的“导入到文档”入口已于 2026-04-14 删除，不再依赖这层前端补偿
- **Location**：~~`src/components/chat/AssistantMessage.tsx` 的 `handleImportToDoc`~~（已删除）
- **Removed**：2026-04-14（本次）
- **Upstream note**：保留历史记录；若未来恢复类似入口，仍需注意 ordered-ops 的执行语义

---

### 4. Update/Rewrite 产生无追踪替换

- **What**：`OpKind::Update` 和 `OpKind::Rewrite` 都调用 `set --prop text=...`，无 tracked 标记
- **Why**：上游 OfficeCLI `set` 命令对 paragraph text 替换不支持 trackChanges。我们只在 Patch（find/replace 粒度）能做修订
- **Solves**：已知限制，docmod 的 update 会产生 w:del+w:ins，迁移到 officecli 后 update 变成 clean replace。semantic 仍保留在 op.kind，等上游支持后可修
- **Location**：`adapter.rs:545-555` `OpKind::Update | OpKind::Rewrite` 分支
- **Added**：OfficeCLI 初始接入即存在
- **Upstream note**：**建议上游给 `set` 加 `tracked=true` 支持段落级整体文本替换**

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
- **Removed**：2026-04-14，commit `047f47d`
- **Upstream note**：行为对齐 cove-wps，不涉及上游 OfficeCLI

---

### 7. `apply_text_ops` 改为单次 batch，Translation 样式插入不再单独补 bookmark

- **What**：`officecli_apply_text_ops` 不再对每个 op 单独 shell 出去执行 `set/add/remove`，而是先把一批 op 翻译成 `officecli batch ... --force --json`，用单次 open/save 周期完成。对于默认 `pstyle=Translation` 的翻译插入，不再在成功后额外逐段 `add bookmark`，直接依赖 `read_paragraphs` 里的 `style=Translation -> trans_<id>` 合成 bookmark
- **Why**：
  - 旧实现对翻译 `insertAfter` 的真实成本不是简单的 `N` 次调用，而是 `1 + 3N`：先 `query p` 建 ID 映射；每个插入再跑一次 `add paragraph`、一次 `query p`（找新段落路径）、一次 `add bookmark`。例如 5 个翻译段落实际会打 16 次 OfficeCLI 往返，10 多个段落很容易放大到 20 秒级
  - Batch API 本来就支持 “one open/save cycle”
  - Batch 模式下新插入段落的最终 path 不能在同一批里稳定拿到；而 contrast-mode 只需要 `trans_*` 标记，不要求真实 bookmark 节点存在
- **Solves**：
  - `WordAgent` 翻译整页/整章 DOCX 时，执行耗时从 “按段线性叠加的 sidecar 启动开销” 降到“每次 `doc_edit` 一次 batch”
- **Location**：
  - `adapter.rs` `officecli_apply_text_ops` / batch helpers
  - `adapter.rs` Translation-style insert 的 deferred bookmark 分支
- **Added**：2026-04-14（本次）
- **Upstream note**：如果上游未来给 batch 返回新建元素的稳定 path，或支持 paragraph add 时直接声明 bookmark，可恢复真实 bookmark 写入并删掉这层合成逻辑

---

### 8. Cove 调用 OfficeCLI 时禁用 auto-resident

- **What**：`run_cli_full` 在 shell 出去执行 OfficeCLI 时额外设置 `OFFICECLI_NO_AUTO_RESIDENT=1`，让 `query/get/batch` 默认走 direct path，而不是“先探测 resident；没有就自动拉起 resident”
- **Why**：
  - cove-desktop 已经在 Rust 侧做了自己的批处理编排；对 `doc_edit` 来说，最重要的是 `batch` 单次 open/save，而不是再让 OfficeCLI 额外起一层 resident
  - Cove 之前同时设置了 `OFFICECLI_NO_PERSISTENT_LOCK=1`，这会让 OfficeCLI resident 进入 lazy-open。上游 lazy-open 在未修补前会把 `query/get/view` 这类读命令也按 `editable=true` 打开，导致纯读 `query p` 也触发整篇文档的 paraId/docProp 扫描
  - auto-resident 还会把一次本来应当直接完成的读取，变成 “CLI 进程 → ping resident → resident 再 open/close 文件” 的多跳链路；resident 状态异常时还可能直接返回 `exit code 3 (busy/unresponsive)`
- **Solves**：
  - `doc_read` / `build_id_map(query p)` 退回 direct readonly 路径，不再被 lazy resident 升级成 editable 打开
  - 翻译、校对这类大量 `query p + batch` 组合调用减少 resident 启动/探测噪音，根因上避开 Cove 当前并不需要的 resident 层
- **Location**：`cli.rs` `run_cli_full`
- **Added**：2026-04-14（本次）
- **Upstream note**：如果上游未来提供“显式禁用 resident 的 CLI 开关”，或 cove-desktop 改成真正的 session-managed resident（明确 open/close 生命周期），可删掉这层 env workaround

---

## 流程规范（cove-desktop 侧）

### 新增魔改时必须做的事

1. **先定位根因**：用实际 OfficeCLI 命令复现问题（例如 `officecli query file.docx p --json` 直接看输出），不要仅凭报错猜
2. **查上游是否有配置**：测试所有可能的 prop 名变体（本文件 §2 就测过 6 种），确认无法通过上游参数规避
3. **代码内加上引用注释**：改动处注释必须写 `WORKAROUND(#N): see WORKAROUNDS.md`，N 为本文件章节号
4. **更新此文件**：按 Template 结构登记新条目，给出 file:line、commit hash、Added 日期
5. **Best-effort 降级**：魔改本身失败时不要抛错中断主流程，打 `eprintln!` warn 即可（参考 `cleanup_tracked_patch_comment` 的实现）

### 移除魔改时必须做的事

1. 确认上游版本已修复对应问题（跑真实 OfficeCLI 命令验证）
2. 更新 cove-desktop 捆绑的 officecli 二进制版本
3. 删除魔改代码同时更新本文件对应条目为 `Removed` 状态，保留历史记录
4. 测试 cove-desktop 所有涉及 officecli 的场景（doc_read / doc_edit / doc_comments / doc_format / 校对修订 等）
5. **同步更新 OfficeCLI 仓库侧 `docs/cove-desktop-workarounds.md`**

### 何时不该魔改

- 上游行为虽然不符合直觉但**未破坏** cove-desktop 任何用户可见场景
- 魔改成本超过向上游提 PR 的成本
- 魔改需要解析 OfficeCLI 非公开的实现细节（fragile，容易随版本失效）

以上场景优先走：**向 OfficeCLI 提 issue → 调整 cove-desktop 用法 / 文案 → 最后才魔改**。
