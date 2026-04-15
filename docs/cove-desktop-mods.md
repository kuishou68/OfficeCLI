# OfficeCLI 上游魔改清单（同步副本）

> **本文件来源**：`OfficeCLI/docs/cove-desktop-mods.md` — 是 source of truth
>
> **区别于 WORKAROUNDS.md**：
> - `WORKAROUNDS.md`：cove-desktop 在适配层绕过上游缺陷（**不改上游源码**）
> - `UPSTREAM_MODS.md`（本文）：我们**直接修改了上游 OfficeCLI 源码**以满足 cove-desktop 需求
>
> 每次修改 OfficeCLI 源码，必须先在上游 `docs/cove-desktop-mods.md` 登记，再同步本文件。

---

## 结构说明

每个条目必须说明：

1. **What**：改了什么文件、什么函数、什么行为
2. **Why**：上游原行为是什么、cove-desktop 为什么不满足
3. **Solves**：解决 cove-desktop 的什么具体场景/bug
4. **Location**：OfficeCLI 仓库内的文件路径 + 关键函数
5. **Added**：日期 + commit hash（OfficeCLI 分支名）
6. **Risk**：是否影响其他消费者（cove-wps / 独立 CLI 用户）

---

## 当前魔改清单

### 1. Tracked find/replace 支持（set --prop tracked=true）

- **What**：新增 `WordHandler.Set.TrackedPatch.cs`，在 `set` 命令的 find/replace 流程中加 `tracked=true` 属性：命中文本包 `w:del`，替换文本包 `w:ins`，并自动写入 `comments.xml` 批注 + `settings.xml` 审阅标志
- **Why**：上游 `set --prop find=X --prop replace=Y` 是直接替换文本，没有修订标记能力。cove-desktop 的校对流程（校对修订 / 审阅模式）需要生成真正的 `w:del/w:ins` 修订标记
- **Solves**：
  - 替代 cove-desktop 原先在 Rust 里直接操作 DOCX zip + 正则拆 run 的 530 行代码（破坏性大、易出错）
  - 生成的修订标记 WPS / Word 打开后能直接进入审阅模式
- **Location**：`src/officecli/Handlers/Word/WordHandler.Set.TrackedPatch.cs`（新文件），`WordHandler.Set.cs` 加 tracked 路由
- **Added**：2026-04-12，branch `feat/tracked-changes`，commit `b92c90e`
- **Risk**：新增 opt-in 属性，不影响未使用者

### 2. HTML 预览支持 tracked changes 可视化（w:del 中划线灰 / w:ins 红色）

- **What**：`WordHandler.HtmlPreview.Text.cs` 中 `RenderParagraphContentHtml` 对 `w:del`/`w:moveFrom` 和 `w:ins`/`w:moveTo` 输出 `<del>`/`<ins>` HTML 标签（原实现删除内容跳过、插入内容无标记）；`RenderRunHtml` 支持 `DeletedText` 元素；`WordHandler.HtmlPreview.Css.cs` 的 `GenerateWordCss` 加 `<del>`/`<ins>` 默认样式
- **Why**：上游 HTML 预览跳过 `w:del` 内容，`w:ins` 渲染为普通文本，无法反映修订状态。cove-desktop 预览面板需要在不修改文档的前提下，用 strikethrough+灰色显示删除、红色显示插入
- **Solves**：
  - 校对修订后，用户在 cove-desktop 预览区域能直接看到 "浪朝（灰删）浪潮（红加）" 的 diff 效果
  - 用 Word/WPS 打开文档仍是正常的 `w:del/w:ins` 审阅模式，不影响下游编辑
  - 预览切换后端（docmod / officecli）时契约一致：都输出 `<del>/<ins>` HTML
- **Location**：
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.Text.cs` line 93-109（del/ins 分支）
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.Text.cs` line 213/272（DeletedText 支持）
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.Css.cs` line 1333-1339（CSS 注入）
- **CSS 实现要点**：使用 `del, del * { ... !important }` 和 `ins, ins * { ... !important }` —— 因为每个 run 输出为 `<span style="color:...">`，内联样式优先级高于普通样式表，需要 `!important` + 后代选择器 `*` 才能穿透所有子元素覆盖颜色
- **Added**：2026-04-14
- **Risk**：纯视觉层改动，HTML 消费者期望 `<del>/<ins>` 时会受影响；未使用 tracked changes 的文档无影响

### 3. GBT9704-cli skill：标题检测 & 版记段落插入修复

- **What**：修改 `GBT9704-cli/scripts/typeset.js` 两处：
  1. title 检测 regex 重构：排除发文字号 `〔\d{4}〕\d+号` / `^第\d+号$` 和签发人；非"关于..."形式要求文档类型词（通知/决定/函 等）位于段落**末尾**
  2. `addElement` 新增 `afterPath` 参数；`createBanjiParas` 显式 `--after` 锚定到当前文档最后一个非表格段落；用 getParagraphs + anchor 定位找到新插入段落的真实 path，不再用 pCount 位置计数
- **Why**：
  - 原 title regex `/通知|决定|请示|批复|函|报告|意见|方案/` 在段落中间匹配，导致 `XX函〔2024〕28号` 因含"函"被误判为 title，titleIndex 错位，版头检测（documentFlag/docNumber）因 `paraIndex < titleIndex` 失败而失效
  - 原 `addElement(filePath, '/body', 'paragraph', {})` 不指定 `--after`，OfficeCLI 可能不追加到 /body 末尾；之后 `const chaosongPath = '/body/p[${pCount}]'` 假设新段落是最后一个，若实际不是，边框打在错误段落（如 date）上
- **Solves**：
  - 党政公文排版：`XX市人民政府`、`XX函〔2024〕28号` 等版头元素正确识别并应用样式
  - 党政公文版记：`2024年4月20日` 和 `抄送：...` 不再错误地夹在两条版记横线之间
- **Location**：`OfficeCLI/bundled-skills/GBT9704-cli/scripts/typeset.js`
  - `addElement` 函数（加 afterPath 参数）
  - `generateCommands` 内 title 检测循环（新 regex + 排除逻辑）
  - `createBanjiParas` 函数（改用 anchor-based path discovery）
- **Added**：2026-04-14
- **Risk**：
  - title 检测更严格，可能漏检某些非标准标题（如仅含"函"字但无"关于"前缀、也不以类型词结尾的边缘情况）
  - `--after` 行为要求 OfficeCLI `add` 命令支持该选项（已确认支持）
  - 空印发场景下只产生 2 条横线（原代码产生 3 条空线），与 GB/T 9704 规范一致

### 4. Lazy resident 读命令按只读方式打开文档

- **What**：修改 `ResidentServer.cs` 的 lazy-open 路径：不再一律 `DocumentHandlerFactory.Open(..., editable: true)`，而是按命令判断。`query/get/view/raw/validate` 这类读命令走 `editable=false`；`set/add/remove/move/swap/raw-set/add-part` 和含写操作的 `batch` 才走 `editable=true`
- **Why**：
  - 上游为了支持 lazy resident（空闲时不持锁）把每条命令都改成了“执行前打开、执行后关闭”，但没有再区分读写，导致纯读 `query p` 也按可写模式打开
  - Word 可写打开会立刻执行 `EnsureAllParaIds()` / `EnsureDocPropIds()` 全量扫描；对 cove-desktop 的翻译链路来说，这意味着每次 `query p` 都被错误升级成一次整篇文档的修复/归一化过程
  - 直接 CLI 非 resident 路径的 `query` 本来就是只读打开；lazy resident 与 direct path 的语义和性能不应该在纯读命令上分叉
- **Solves**：
  - Cove 翻译 / 校对流程里的 `query p`、`query bookmark` 即使命中 resident，也不会再因为 lazy-open 而触发整篇文档的 editable 初始化
  - 避免纯读命令在无用户意图时修改 paraId/docProp 元数据，减小“读操作带写副作用”
- **Location**：`src/officecli/ResidentServer.cs`
  - `ExecuteCommand`
  - `RequestNeedsEditableAccess`
  - `BatchNeedsEditableAccess`
- **Added**：2026-04-14
- **Risk**：
  - 纯读命令不再顺手修复缺失/重复 paraId；这与 direct CLI path 现有行为一致，真正写入时仍会在 editable 打开时补齐
  - `batch` 需要解析一次命令数组来判断是否包含写操作；解析失败时回退到 `editable=true`，不会影响正确性

### 5. HTML 预览支持批注可视化（commentRangeStart/End → mark + aside）

- **What**：`WordHandler.HtmlPreview.Text.cs` 的 `RenderParagraphContentHtml` 对 `CommentRangeStart`/`CommentRangeEnd` 输出 `<mark data-id="cmX">...</mark>` 包裹注释范围文本；`RenderRunHtml` 对 `CommentReference` 收集 ID；新增 `RenderCommentsHtml` 从 `WordprocessingCommentsPart` 读取批注内容，在文档末尾输出 `<aside data-type="comments">` 块（含作者、日期、回复关系）；`WordHandler.HtmlPreview.Css.cs` 加默认 `mark`/`aside` 基础样式。支持跨段落批注（close/reopen mark at paragraph boundaries）
- **Why**：上游 HTML 预览完全跳过 `commentRangeStart`/`commentRangeEnd`/`commentReference` 元素，批注信息在预览中不可见。cove-desktop 校审面板需要在不修改文档的前提下显示批注标记和内容
- **Solves**：
  - 校审/审阅后，用户在 cove-desktop 预览区域能看到带①②③角标的批注标记和右侧批注面板
  - 批注标记与修订标记（del/ins）正交，同时可见
  - 前端 `comment-annotations.ts` 消费 `<mark>` + `<aside>` 标记，注入交互式 UI
- **Location**：
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.Text.cs`（CommentRangeStart/End 分支 + RenderCommentsHtml）
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.cs`（HtmlRenderContext.CommentRefs + ViewAsHtml 调用点）
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.Css.cs`（mark/aside 基础样式）
- **Added**：2026-04-15
- **Risk**：纯视觉层改动。无批注的文档无影响。HTML 消费者若解析 `<mark>` 元素需注意新增标记

### 6. HTML 预览按 DOCX 样式链和实际字体修正段落行高/段距

- **What**：修改 `WordHandler.HtmlPreview.Css.cs`：
  - `ResolveSpacingFromStyle` 改为按属性级别合并 `w:style` basedOn 链和 `docDefaults` 的 `w:spacing`
  - `GetParagraphInlineCss` 对每个段落稳定输出 inline `line-height`
  - `ResolveParaFontForLineHeight` 改为优先读取 `rFonts.eastAsia`，不再只看 `ascii/highAnsi`
- **Why**：
  - 原实现一旦在样式链上遇到第一个 `w:spacing` 节点就提前返回；如果上层样式只定义了 `before/after`，而 `line` 在更高层或 `docDefaults`，继承会被截断
  - 段落没有显式 `w:spacing` 时，会回退到全局 `<p>` CSS，并使用文档默认字体的 metrics；对中文文档来说，真实行高往往由 `eastAsia` 字体决定，结果会比 WPS/Word 更紧
- **Solves**：
  - Word/WPS 中依赖样式继承的段落，在 cove-desktop 预览里能拿到更接近原始 DOCX 的行高、段前段后距
  - 中文文档中 `宋体/仿宋/楷体` 等 EastAsia 字体的行框不再被西文字体 metrics 误导
- **Location**：
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.Css.cs`
  - `GetParagraphInlineCss`
  - `ResolveSpacingFromStyle`
  - `ResolveParaFontForLineHeight`
- **Added**：2026-04-15
- **Risk**：
  - HTML 预览对“无显式 spacing 的段落”会更积极输出 inline `line-height`，可能改变少量历史文档的旧预览截图
  - 改动只影响 `view ... html` 预览层，不影响 DOCX 写回

---

## 流程

1. **新增魔改**：先更新 `OfficeCLI/docs/cove-desktop-mods.md`，再同步本文件；代码处加 `// MOD(#N): see cove-desktop-mods.md`
2. **移除魔改**：验证 cove-desktop 不再依赖 → 删代码 → 删条目
3. **同步**：本文件和 OfficeCLI 侧保持完全一致
