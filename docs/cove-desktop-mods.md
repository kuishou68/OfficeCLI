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

### 7. HTML 预览按缩进继承链和 style-only 段落修正边距/行高

- **What**：继续修改 `WordHandler.HtmlPreview.Css.cs`：
  - `ResolveIndentationFromStyle` 改为按属性级别合并 `w:style` basedOn 链和 `docDefaults` 的 `w:ind`
  - `ResolveParagraphStyleCss` 不再直接把样式里的 `spacing.Line/240` 塞进 CSS，而是复用 `BuildDefaultParagraphLineHeightCss` 的字体 metrics 路径
- **Why**：
  - 旧实现对缩进继承仍是“遇到第一个 `<w:ind>` 就整块返回”；如果派生样式只覆写 `firstLine`，而 `left/right` 仍在基样式，预览会丢掉段落左右边距
  - 对没有直接 `pPr` 的段落，样式链里的 `line` 之前按裸 `240` 倍率输出，没有乘上 EastAsia 字体 metrics，修订/正文混排时会比 WPS 更松或更紧
- **Solves**：
  - WPS / Word 中依赖样式继承的首行缩进、左右边距，在 cove-desktop 预览里更接近原始 DOCX
  - 修订段落和普通段落混排时，style-only 段落的行高不再和带直接 `pPr` 的段落走两套算法
- **Location**：
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.Css.cs`
  - `ResolveIndentationFromStyle`
  - `ResolveParagraphStyleCss`
- **Added**：2026-04-15
- **Risk**：
  - 仅影响 `view ... html` 预览层；历史截图或像素基准可能会小幅变化
  - 如果某些文档依赖“错误的首个 `<w:ind>` 短路行为”，预览边距会发生校正

---

### 8. `TryResident(...)` 支持 `OFFICECLI_SKIP_RESIDENT=1`

- **What**：修改 `src/officecli/CommandBuilder.cs` 的 `TryResident(...)`：当环境变量 `OFFICECLI_SKIP_RESIDENT=1/true` 时，直接返回 `null`，完全跳过 resident 探测、复用和 auto-start
- **Why**：
  - cove-desktop 的读命令链路需要一个“本次显式不要 resident”的强开关。现有 `OFFICECLI_NO_AUTO_RESIDENT` 只影响“没 resident 时是否 auto-start”，对“已经存在但状态不健康的 resident”无效
  - 这类坏 resident 即使不再持有文件锁，也会让 `query/view` 先走进 `TryResident(...)`，再卡在 main pipe busy timeout，最终把一次本可亚秒完成的读命令放大到 30s~60s
- **Solves**：
  - 让 cove-desktop 的 `query/view/get/raw` 可以稳定走 direct file access，彻底绕开 existing resident 的不确定性
  - 保留 OfficeCLI resident 机制给真正受益的写命令；只读链路由调用方按需 opt-out
- **Location**：`src/officecli/CommandBuilder.cs` `TryResident`
- **Added**：2026-04-15
- **Risk**：
  - 新增 opt-in 环境变量，不影响默认 CLI 用户
  - 调用方如果误把该变量用于写命令，会失去 resident 带来的性能/一致性保障；因此当前只供 cove-desktop 的读链路使用

---

### 9. HTML 预览分页模板补齐页眉/页脚，并显式解析 `PAGE/NUMPAGES`

- **What**：
  - `WordHandler.HtmlPreview.Text.cs` 对页眉/页脚中的 complex field / simple field 显式识别 `PAGE`、`NUMPAGES`、`SECTIONPAGES`，输出 `<span class="page-num">` / `<span class="page-count">` 占位符，而不再依赖 Word 预先写回缓存结果
  - `WordHandler.HtmlPreview.cs` 生成分页模板时，为首/奇/偶页同时保留 header/footer 模板；客户端自动分页时复制 header 和 footer；重新分页后统一回填当前页码和总页数
  - `NormalizeStandalonePageBreakParagraphs` 和 body 渲染路径继续配合，确保 standalone `w:br w:type="page"` 不再把 `<p>` 起止标签切到两页
  - 在嵌入式 iframe 预览场景下禁用 `scalePages()` 自动缩放，只保留分页，避免首屏先按自然页宽渲染、随后再被缩窄
- **Why**：
  - 上游 HTML 预览对 header/footer 里的 field code 基本依赖“结果 run 已经存在”。很多文档里 `PAGE/NUMPAGES` 只有指令，没有缓存数字，结果页脚渲染成“第  页 共  页”
  - 自动分页新增页面时，上游只复制 footer 模板或根本没有结构化页码占位符，导致长文档预览中页眉页脚缺失、页码不更新
  - standalone page break 段落如果仍被包在 `<p>` 里，分页切割后会形成跨页悬空标签，直接扰乱第一页布局和后续 DOM 结构
- **Solves**：
  - Cove 长文档预览里，每一页都能继承正确的页眉/页脚模板
  - `PAGE/NUMPAGES` 不再依赖 Word 先更新字段，HTML 预览中可直接显示并在自动分页后保持正确
  - 手动分页符附近不再出现“上一页开 `<p>`、下一页补 `</p>`”的畸形 HTML，减少第一页白底结构异常
  - cove-desktop 里 `srcdoc + iframe` 打开的长文档不会在几秒后再发生一次整页缩窄的视觉跳变
- **Location**：
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.Text.cs`
  - `src/officecli/Handlers/Word/WordHandler.HtmlPreview.cs`
- **Added**：2026-04-15，commit `pending`
- **Risk**：
  - 仅影响 `view ... html` 预览层，不影响 DOCX 写回
  - header/footer 中极少数非页码 field 仍走原有“显示缓存结果”路径；当前只对 cove-desktop 需要的分页字段做结构化处理

---

### 10. HTML 预览恢复 `w:jc` 对齐映射，避免标题和页眉页脚退回默认对齐

- **What**：修改 `WordHandler.HtmlPreview.Css.cs` 的段落对齐解析：`GetParagraphInlineCss` / `ResolveParagraphStyleCss` 不再依赖 `EnumValue<JustificationValues>.Value`，改为直接消费 OOXML 原始 `w:jc/@w:val` 文本（`Val.InnerText`）映射 CSS `text-align`
- **Why**：
  - 近期把 `jc.InnerText switch` 收敛成 enum helper 时，部分真实 DOCX 的 `EnumValue.Value` 没有稳定落到预期值，导致 direct paragraph properties 里的 `center/right` 被吃掉
  - 这个回归在封面标题、页眉、页脚里最明显：原本居中/右对齐的内容会退回页面默认 `justify/left`
- **Solves**：
  - Cove 预览中的封面标题、副标题重新与 WPS / Word 保持居中
  - 页眉右对齐、页脚居中页码恢复正确，不再看起来像“页边距对了但版心没对齐”
  - 保留 style-chain fallback，同时避免 enum 解析差异再次引入视觉漂移
- **Location**：`src/officecli/Handlers/Word/WordHandler.HtmlPreview.Css.cs`
- **Added**：2026-04-15，commit `pending`
- **Risk**：
  - 仅影响 `view ... html` 预览层，不影响 DOCX 写回
  - 对齐映射回到 OOXML 原始值后，少数依赖错误默认对齐的旧截图会发生校正

---

### 11. HTML 预览按页面边缘定位页眉页脚，并把 `docGrid linePitch` 压到段落行高

- **What**：继续修改 `WordHandler.HtmlPreview.Css.cs`：
  - `GenerateWordCss` 中 `.doc-header` / `.doc-footer` 改为以“页面边缘”为基准计算 `top/bottom`，并用 `padding-left/right` 承接页边距，而不是直接把 `header/footer distance` 当成内容区内偏移
  - `BuildDefaultParagraphLineHeightCss` 改为按“段落实际字号 × OOXML 行距倍率 × 字体 metrics”算出绝对 pt 行高，并在 body 段落里用 section `w:docGrid/@w:linePitch` 做下限夹紧
  - 新增 `ResolveParaFontSizePt` / `ClampBodyParagraphToDocGrid`，避免 `<p>` inline `line-height` 把全局 page grid 覆盖掉
- **Why**：
  - OOXML 的 `w:pgMar/@header`、`@footer` 是“距纸张边缘”的距离；旧 CSS 却在已经带 `padding` 的 `.page` 内容区里再加一遍偏移，页眉页脚会整体偏下、偏窄，看起来和 WPS/Word 的版心不一致
  - 旧实现虽然在 `.page` 上写了 `docGrid linePitch`，但段落和 `<p>` 默认样式随后又各自输出了更小的 `line-height`，实际浏览器渲染时网格被覆盖，正文比 WPS 更紧
  - WPS 导出的 `w:docGrid` 经常只写 `linePitch` 不写 `type`；如果解析层把“无 type”当成“无 grid”，正文会整体退回 10pt/11pt 级浏览器默认行距
- **Solves**：
  - 页眉/页脚的横向版心重新与页面边距对齐，纵向位置也回到 `header/footer distance` 的真实位置
  - 带 `w:docGrid linePitch` 的中文 DOCX，正文基线间距不再掉回浏览器默认单倍行距，预览松紧更接近 WPS / Word
  - 标题等大字号段落仍保留自身更高的行高，不会被 18pt grid 反向压扁
- **Location**：`src/officecli/Handlers/Word/WordHandler.HtmlPreview.Css.cs`
- **Added**：2026-04-15，commit `pending`
- **Risk**：
  - 仅影响 `view ... html` 预览层，不影响 DOCX 写回
  - 历史截图或像素基准会因为页眉页脚和正文行距更接近真实版式而发生变化

---

### 12. 字体 metrics 查找补齐 macOS CJK 别名，避免 `宋体/黑体` 行高退回 1.0

- **What**：修改 `src/officecli/Core/FontMetricsReader.cs` 的 `FindFontFile`：不再只按文件 stem 精确匹配传入的字体名，而是先展开一层 CJK 逻辑字体别名，再按归一化后的候选名匹配本机字体文件。新增 `宋体 -> Songti/Songti SC/STSong`、`黑体 -> STHeiti/Heiti SC`、`仿宋_GB2312 -> STFangsong/FangSong`、`楷体_GB2312 -> STKaiti/KaiTi` 等映射
- **Why**：
  - DOCX 经常写的是 Windows 逻辑字体名（如 `宋体`、`黑体`），但 macOS 实际字体文件名是 `Songti.ttc`、`STHeiti.ttc`
  - 旧实现只按文件 stem 精确匹配，`FontMetricsReader.GetRatio("宋体")` 在 mac 上通常直接 miss，退回 `1.0`
  - 一旦 metrics miss，HTML 预览虽然已经输出了绝对 pt 行高，数值仍然是按“无额外 ascent/descent”计算，中文段落会比浏览器实际使用的 `Songti SC` / `Heiti SC` 行框更紧
- **Solves**：
  - Cove 预览里 `宋体/黑体/仿宋/楷体` 中文段落的行高计算更接近 macOS 实际渲染字体，不再因为逻辑字体名 miss 而压扁
  - `BuildDefaultParagraphLineHeightCss` / `GenerateWordCss` 的 metrics 路径终于能命中 `Songti.ttc`、`STHeiti.ttc` 这类系统字体文件
- **Location**：`src/officecli/Core/FontMetricsReader.cs`
- **Added**：2026-04-16，commit `pending`
- **Risk**：
  - 仅影响依赖 `GetRatio()` / `GetAscentDescentOverride()` 的 HTML 预览层，不影响 DOCX 写回
  - 历史预览截图会因行高更接近系统真实字体而发生校正；CLI 其他 consumers 若依赖旧的 `1.0` fallback，视觉会同步变更

---

### 13. 嵌入态 DOCX HTML 预览默认启用 fit-to-width 缩放

- **What**：修改 `src/officecli/Handlers/Word/WordHandler.HtmlPreview.cs` 的 `shouldScalePages()`：不再用 `window.top === window.self` 限制缩放只在顶层窗口启用，而是默认返回 `true`；同时保留两个显式关闭开关：`window.__officecliDisablePageScaling === true` 和 `document.body[data-officecli-scale='off']`
- **Why**：
  - `v1.0.48-cove.1` 的 HTML 预览在 iframe 嵌入态会命中 `window.top !== window.self`，导致 `scalePages()` 被完全跳过
  - Cove 的 DOCX 预览正是 `srcdoc + iframe` 嵌入，所以横向页面会按原始页宽渲染，预览面板里看不完整
  - 之前条目 9 里的嵌入态缩放限制更适合“避免首屏宽度跳变”的场景，但对固定宽度预览面板来说，完整预览文档优先级更高
- **Solves**：
  - Cove 预览面板里的横向 DOCX 页面会像独立 HTML 预览一样自动 fit 到面板宽度
  - 纵向 / 横向文档都能在不改窗口宽度的前提下完整预览页面
  - 未来若其他宿主确实需要保留原始页宽，仍可通过显式开关关闭缩放
- **Location**：`src/officecli/Handlers/Word/WordHandler.HtmlPreview.cs` `shouldScalePages`
- **Added**：2026-04-22，branch `feat/embedded-docx-scale`，commit `8dbeec5`
- **Risk**：
  - 仅影响 `view ... html` 预览层，不影响 DOCX 写回
  - 依赖“iframe 中保持原始页宽”的宿主会看到行为变化；这类宿主需要显式设置关闭开关

### 14. PPT HTML 预览跳过母版/版式结构占位符

- **What**：修改 `PowerPointHandler.HtmlPreview.cs` 的 `RenderInheritedShapes`：从 SlideLayout / SlideMaster 继承渲染 shape 时，只允许 `date/footer/header/slide number` 这类元信息 placeholder 作为可见占位符输出；`title/body/subtitle/object` 以及未显式写 `type` 的 placeholder 一律视为结构占位符跳过
- **Why**：
  - 上游原逻辑只跳过显式 `title/body/subtitle/object`，但真实 PPTX 中 layout content placeholder 常只有 `idx/sz`，没有 `type`
  - OOXML/PowerPoint 语义里这类 placeholder 是给幻灯片内容继承样式和位置的结构槽位，不是要在最终预览中显示的真实文本
  - 旧 HTML renderer 会把母版提示文案（例如 `单击此处编辑母版文本样式` / `二级三级四级五级`）渲染成普通 `.shape`，覆盖幻灯片实际标题和正文
- **Solves**：
  - Cove 预览 `Cove + OfficeAI 演示用例20260407/产品资料/OfficeAI 产品介绍 1110.pptx` 时，slide 4/6/9/15 不再出现母版提示文字叠在实际内容上
  - 修复点位于 OfficeCLI renderer，Cove 适配层无需再通过 HTML 文案过滤脚本兜底
- **Location**：`src/officecli/Handlers/Pptx/PowerPointHandler.HtmlPreview.cs`
- **Added**：2026-04-27，branch `feat/tracked-changes`，commit `pending`
- **Risk**：
  - 影响 `view ... html` PPT/PPTX 预览层，不影响 PPTX 写回
  - 如果某个文档故意把 layout/master 的内容 placeholder 当作可见文本使用，预览会不再显示这类非标准内容；常规页脚、日期、页码、页眉 placeholder 仍保留

---

## 流程

1. **新增魔改**：先更新 `OfficeCLI/docs/cove-desktop-mods.md`，再同步本文件；代码处加 `// MOD(#N): see cove-desktop-mods.md`
2. **移除魔改**：验证 cove-desktop 不再依赖 → 删代码 → 删条目
3. **同步**：本文件和 OfficeCLI 侧保持完全一致
