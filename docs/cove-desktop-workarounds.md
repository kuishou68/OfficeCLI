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

### 12. 主预览 HTML 复用 DOCX 磁盘缓存，避免重复冷启动 OfficeCLI

- **What**：`officecli_view_html` 在真正执行 `view <file> html --json` 之前，先按 `DOCX 字节内容 FNV-1a hash + renderer cache version` 查找 `<app_data_dir>/pdf-cache/*.officecli.vN.html`；命中时直接返回缓存 HTML，未命中才调用 OfficeCLI 并把结果写回磁盘缓存。当前 renderer cache version 已提升到 `v7`，用于覆盖页眉页脚定位、`docGrid linePitch` 行高修正，以及 macOS CJK 字体 metrics 别名补齐后的新 HTML
- **Why**：
  - 主文件预览路径此前每次打开 `.docx` 都会直连 `officecli_view_html`，完全绕过 `docx_to_html` 已有的 L2 HTML 缓存
  - OfficeCLI resident warm 之后很快，但首轮 `view html` 仍有明显冷启动成本；在 React StrictMode 或快速重开预览时，这个成本会被用户反复感知
  - 仅按 DOCX 内容 hash 命中缓存时，sidecar renderer 升级后会继续复用旧 HTML，导致“二进制已更新但预览还停留在旧版 DOM/CSS”
- **Solves**：
  - 主预览与浮窗预览在“同一份 DOCX 已经生成过 HTML”时共享磁盘缓存收益
  - 避免同一文档反复打开时重复触发 OfficeCLI 冷启动，缩短加载首屏时间
  - 当 OfficeCLI HTML 渲染逻辑升级（例如页眉页脚、页码、对齐、分页修复）时，旧缓存会自动失效，用户不需要手工删 cache 目录
- **Location**：`mod.rs` `officecli_view_html`
- **Added**：2026-04-15，commit `401c49f` 后续修复提交
- **Upstream note**：这是 cove-desktop 侧的预览缓存策略，不依赖上游改动；若未来 OfficeCLI 提供稳定的持久化 HTML cache API，可删掉本地文件缓存复用

---

### 13. 锁文件错误直传，禁止 batch 失败后退回逐条 patch

- **What**：
  - `cli.rs` 在 `run_cli` 中解析 OfficeCLI `--json` 的结构化失败体：即使非零退出码把错误写在 stdout，也会抽取真实错误信息
  - `adapter.rs` 的 `execute_ops_batch` 遇到“文件被其他进程占用”这类 batch 级失败时，不再回退到 legacy per-op `set/add/remove`，而是把同一个锁错误直接回写到各 op
- **Why**：
  - OfficeCLI v1.0.46 在 `.docx` 被外部进程持有排它锁时，`batch ... --json` 会 `exit 1`，但真正的原因只写在 stdout JSON：`"The process cannot access the file ... because it is being used by another process."`
  - 旧适配层对非零退出码只拼 `stderr`，导致 batch 根因被吞掉；随后又无条件 fallback 到逐条 `set`，每条 patch 都再次因同一把锁失败，最终只给用户留下多条 `[patch] ... officecli exited with code 1`
- **Solves**：
  - 校对/修订场景下，文档被 Word / WPS / 预览进程占用时，用户能直接看到“文件被占用”的根因，而不是误以为某几个段落 patch 本身坏了
  - 避免 batch 失败后再次触发一轮无意义的逐条写入，减少锁冲突放大和误导性日志
- **Location**：
  - `cli.rs` `run_cli`
  - `adapter.rs` `execute_ops_batch`
- **Added**：2026-04-15（本次）
- **Upstream note**：如果上游未来统一为“非零退出时把结构化错误写到 stderr，且为锁错误提供稳定错误码”，可删除本地 stdout 错误抽取和锁错误禁 fallback 逻辑

---

### 14. Translation 默认 bookmark 不再补第二轮 batch

- **What**：`officecli` adapter 里的 `insertAfter/insertBefore` 仍默认给段落打 `style=Translation`，但不再把默认 `trans_<id>` bookmark 物化成真实 bookmark 节点；读取阶段直接依赖 `style=Translation -> trans_<原文 id>` 的本地合成。只有调用方显式传入的非默认 bookmark，才继续走第二轮紧凑 bookmark batch
- **Why**：
  - 默认双语对照翻译并不需要真实 bookmark 节点，`confirm_edits` 的 collapse 只识别 `trans_*` 语义标记
  - 旧链路里主 batch 成功后还会再补一轮 `add bookmark`，大量翻译段落时相当于额外写一遍文档
- **Solves**：
  - `WordAgent` 双语对照翻译时，`doc_edit` 默认只做一次主 batch，不再为每个默认 Translation 插入追加第二轮写回
  - 保持现有 contrast-mode 语义不变，同时缩短整页/整章翻译耗时
- **Location**：
  - `src/lib/doc/adapters/officecli.ts`
  - `src-tauri/src/officecli/adapter.rs`
- **Added**：2026-04-15（本次）
- **Upstream note**：这是 cove-desktop 对 OfficeCLI 调用链的瘦身；如果上游未来让 batch 直接返回稳定的新建段落 path，并且我们重新需要真实 bookmark 节点，可恢复物化写入

---

### 15. 读命令不再直连 resident 主 pipe，取消 DOCX 高亮预热

- **What**：
  - `cove-desktop` 的 `cli.rs` 不再让 `query/view/get/raw` 走本地实现的 resident 主 pipe 直连，只保留写命令直连
  - 文件树第一次高亮 `.docx` 时，不再后台预热 `officecli_prepare_html_preview`
- **Why**：
  - cove 本地 Rust 直连 resident 主 pipe 的实现，没有复刻上游 `TryResident(...)` 的 ping pipe 探活 + busy main pipe 重试/降级
  - 一旦后台 `view html` 仍占着 resident 主 pipe，后续 `doc_read(query p)` / 再次 `view html` 会在 cove 侧排队到本地 60s timeout，而直接 shell 出 bundled `officecli` 实测只要约 `0.8s`
- **Solves**：
  - 翻译前 `doc_read` 不再被预览 resident 的长命令卡满
  - 仅高亮文件时不再偷跑预览导出，减少和文档编辑/翻译链路的资源争抢
  - 读命令继续享受上游 CLI 的 resident 复用，但 busy 情况交回上游处理，不再被 cove 本地直连放大
- **Location**：
  - `src-tauri/src/officecli/cli.rs`
  - `src/components/preview/FileTreePanel.tsx`
- **Added**：2026-04-15（本次）
- **Upstream note**：这是 cove-desktop 对 resident 编排的调用链修正，不依赖 OfficeCLI 上游改动；如果未来 cove 补齐与 `TryResident(...)` 等价的 ping+buzzy-handling，可再评估恢复读命令直连

---

### 16. 读路径先经 ping pipe 关闭同文件 resident

- **What**：
  - `cove-desktop` 的 `cli.rs` 新增 `close_resident_best_effort(file)`，直接向 `<pipe>-ping` 发送 `__close__`
  - `query_all_paragraphs`、`officecli_view_paragraphs`、以及主预览 cache miss 的 `view html` 在真正读文件前，都会先做一次 best-effort close；如果没有 resident，就静默跳过
- **Why**：
  - 即使 cove 侧已经不再“直连 resident 主 pipe”，上游 `officecli query/view` 仍会优先复用一个已经存在的 resident
  - 一旦这个 resident 本身就是陈旧/卡死/刚被别的 `view html` 打满主 pipe，新的 `doc_read` 还是会沿着上游 `TryResident(...)` 链路卡到 30s~60s
  - 对同一份 `/Users/pojian/Downloads/随便测试一下.docx` 的实测里，直接 shell 出 bundled `officecli query p --json` / `view html --json` 都在 `~0.8s`；而 UI 链路重复出现 `doc_read≈60.7s`，剩余差值只能落在“沿用了坏 resident”这层编排
- **Solves**：
  - 翻译前 `doc_read(query p)` 不再继承一个已经不健康的 resident
  - 主预览 cache miss 的 `view html` 也先清 resident，避免继续复用同一个坏实例
  - AI 工具走 `officecli_view_paragraphs` 时同样获得保护，减少文本读取偶发长卡顿
- **Location**：
  - `src-tauri/src/officecli/cli.rs`
  - `src-tauri/src/officecli/adapter.rs`
  - `src-tauri/src/officecli/mod.rs`
- **Added**：2026-04-15（本次）
- **Upstream note**：理想修复是 OfficeCLI 给 `query/view` 提供稳定的“禁止复用现有 resident”选项，或把 stale/busy resident 判定前移到 ping 阶段；在那之前，保留 cove 侧的 best-effort close

---

### 17. 读命令 shell-out 显式跳过 resident

- **What**：`cove-desktop` 的 `cli.rs` 在 shell 出去执行 `query/view/get/raw` 时额外注入 `OFFICECLI_SKIP_RESIDENT=1`。配合已修改的 OfficeCLI `TryResident(...)`，这些读命令会直接回落到 direct file access，不再探测/复用/自启动 resident
- **Why**：
  - 单靠 §16 的 best-effort `__close__` 只能清掉“空闲但坏掉”的 resident；如果 resident 正在跑慢 `view html`，`__close__` 会等它收尾，短超时后仍可能让后续读命令继续撞回同一个 busy resident
  - 用户这轮 43.2s 翻译里，真实写入 `doc_edit` 已经只有 `~0.9s`，剩下的大头就是读命令反复继承 resident 的等待成本
- **Solves**：
  - `doc_read(query p)`、预览 `view html`、以及工具态 `view text/get/raw` 都不会再因 resident main pipe 状态而出现 30s~60s 级卡顿
  - 读命令统一走 direct path；写命令仍保留 resident / batch 的性能收益，不把之前已优化好的写链路拖慢
- **Location**：
  - `src-tauri/src/officecli/cli.rs`
  - 依赖上游补丁：`OfficeCLI/src/officecli/CommandBuilder.cs`
- **Added**：2026-04-15（本次）
- **Upstream note**：如果 OfficeCLI 将来正式提供 `--no-resident` 或等效 flag，可删掉这个环境变量约定并改走官方参数

---

### 19. Translation 插入前检查相邻同文译文，阻止重复翻译

- **What**：`officecli_apply_text_ops` 在把 `insertAfter` / `insertBefore` 翻译为 batch 命令前，会检查目标原文段落相邻的 `style=Translation` 段落；若已存在文本完全相同的译文，则直接拒绝该 op，不再继续插入
- **Why**：
  - 对照翻译默认只靠 `style=Translation` 和 `trans_<id>` 语义恢复配对，并没有真实唯一键约束
  - 当用户在已插入过译文的文档上再次发起翻译，或代理因重试重复发送同一批 Translation insert op 时，旧链路会把同一段英文再次插到原文旁边
- **Solves**：
  - 修复 `doc_edit` 成功后文档里出现“双份同文译文段落”的问题
  - 让重复翻译退化为明确错误，而不是继续污染 DOCX 结构
- **Location**：
  - `src-tauri/src/officecli/adapter.rs`
  - `existing_translation_neighbor_has_same_text`
  - `prepare_batch_op`
- **Added**：2026-04-16（本次）
- **Upstream note**：这是 cove-desktop 在调用链上的幂等保护；若上游未来为 paragraph insert 提供稳定的“按 anchor + semantic key 去重”能力，可删除本地邻段扫描

---

### 20. Resident JSON 响应去 BOM，避免 batch 被重复执行

- **What**：`cli.rs` 在解析 resident 主 pipe / ping pipe 返回的 JSON 前，会先去掉 UTF-8 BOM 和前导空白，再交给 `serde_json`
- **Why**：
  - 当前本地 OfficeCLI resident 返回体前面可能带 BOM，例如 `\u{feff}{"ExitCode":0,...}`
  - 旧实现直接 `serde_json::from_str(&line)`，解析失败后被当成 resident miss，再回退到 shell-out
  - 对 `batch add paragraph` 这类写命令来说，resident 其实已经成功执行过一次；随后 shell-out 再跑一遍，就会把同一译文重复插入
- **Solves**：
  - 修复“单次 `doc_edit` 成功，但译文在文档里出现两份”的真正重复执行问题
  - 避免 ping pipe 的健康检查被 BOM 误判成坏 resident
- **Location**：
  - `src-tauri/src/officecli/cli.rs`
  - `run_resident`
  - `run_ping_request`
- **Added**：2026-04-16（本次）
- **Upstream note**：若上游确认 resident 永远输出无 BOM 的纯 JSON，此条仍可保留为兼容层；成本极低，不必急于移除

---

### 21. Contrast 确认走单次 Rust 快路径，避免长文档分页折叠

- **What**：`confirm_edits(mode="contrast")` 对 officecli backend 不再走“TS 分页 `readParagraphs` 全文 + 再调一次通用 `applyTextOps`”的默认折叠流程，而是直接调用新的 `officecli_collapse_contrast_pairs` Tauri 命令，在 Rust 侧一次性构建 rewrite/delete batch 并提交给 OfficeCLI
- **Why**：
  - 长文档翻译确认阶段原本要多次跨端 IPC 读取整篇段落，再把同一批折叠 op 重新发回 Rust，确认耗时会随着段落数线性放大
  - `doc_edit` 插入对照译文之后，session 里本来就有同一份段落快照；折叠再退回 TS 层重新分页读取，纯属重复搬运
- **Solves**：
  - 减少 contrast 确认阶段的 IPC 往返次数和分页读取开销
  - 为长文档翻译确认提供更稳定的耗时上界，避免“确认修改”阶段比真正写入还慢
- **Location**：
  - `src-tauri/src/officecli/adapter.rs`
  - `src-tauri/src/lib.rs`
  - `src/lib/doc/adapters/officecli.ts`
- **Added**：2026-04-16（本次）
- **Upstream note**：这是 cove-desktop 本地的折叠编排优化；若上游未来提供原生“merge contrast translations”命令，可以删掉本地快路径并改用官方能力

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
