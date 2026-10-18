# Markor 预览层 JS 资产升级计划（2026-10）

> 范围：只管 `app/thirdparty/assets/` 下的前端库（Prism / KaTeX / Mermaid / asciidoctor / org-js）。
> md→HTML 的**正确性**由 flexmark 决定，那部分在 `docs/flexmark-upgrade-investigation.md`（其 §6 明确把 JS 资产划到本单）。
> 本文是计划 + 实测依据，不是执行记录；执行进度回写 AGENTS.md 现状台账。

## 0. 结论速览

- 升级前，本 fork **一行 JS 资产都没动过**：`git log --oneline -- app/thirdparty/assets` 最后一条是上游的 `aff9492c`（"Update Mermaid from 11.6.0 to 11.17.2"）。现状 = 上游 Markor 2.16.1 附带的版本。
- 值得升的只有三项：**Prism 1.29.0→1.30.0**、**KaTeX 0.16.10→0.19.0**、**Mermaid 11.17.2→12.2.0**。
- **asciidoctor 与 org-bundle.js 不升**（理由见 §4 Step 6：前者跨大版本有 API 风险且收益不明，后者上游 2015-02-21 就停更了）。
- 全部资产走 `file:///android_asset/`，**没有 CDN、没有网络权限依赖**（`MarkdownTextConverter.java:96,98`；`MarkorWebViewClient.java:37` 负责放行该前缀）。所以"升级"= 换文件 + 改版本标记文件，不动构建配置。
- 关键判据是**渲染观感**，构建成功不代表升级正确。所以第 0 步先做渲染基线，而不是先换文件。
- **Step 0 已做完（2026-10-18），并且当场推翻了本单初稿的两处推断**（详见 §3.2、§4 Step 4）。升级判据由此确定为主用 **CDP DOM 探针**（§5），截图为辅。
- **Step 1（Prism 1.30.0）已做完并验收通过**（2026-10-18）：实际只改 2 个文件各 1 行，不是初稿写的"11 个产物"（§4 Step 1）。同一天 release 构建被一次仓库配置故障挡住，根因与本仓库的修法见 §5.3。
- **Step 2（删 `katex/auto-render.min.js`）已做完**（2026-10-18）：全仓零引用，删除后清单只差这一行。
- **Step 3（KaTeX 0.19.0）已做完并验收通过**（2026-10-18）：63 个文件（css/js/mhchem + 60 字体）整批换，`katexVer` 0.16.10→0.19.0，其余 DOM 读数逐项不变，顶部截图差异只在状态栏；"字体有没有换对"改用 CDP `CSS.getPlatformFontsForNode` 量化（§4 Step 3）。
- **Step 5（Mermaid 12.2.0）已做完并验收通过**（2026-10-18）：单文件替换，`div.mermaid[data-processed] svg` 仍为 1、`logLevel:5` 数字写法 v12 仍接受、四个节点名与边标签文本一个没丢；可见差异是 v12 的新样式（节点带 drop-shadow）。代价是 bundle +1,922,140 B、release 包 +593,295 B。**至此本单三项升级全部落地。**
- **收尾（2026-10-18）**：用户真机复测反馈"没问题"，本单的验收闭环（CDP 探针 + 产物字节 + 人工点击）三条都齐了。`toolchain-upgrade` 已推送到 fork。

## 1. 现状（实测 2026-10-18，Step 0-5 全部完成）

资产合计 **9,966,515 B / 153 个文件**（升级前 8,376,499 B / 154，净 **+1,590,016 B** = Mermaid +1,922,139 − KaTeX 326,738 − auto-render 3,481 − prism.js 行尾归一化 1,904；逐项拆解见各 Step），逐文件 sha256(前16位)|字节|路径 见 `docs/js-preview-libs-baseline.txt`（**该清单每过一步就地推进一次**，所以它始终等于当前 HEAD 工作树）。

| 库 | 版本判据（怎么读出来的） | 现版本 | 体积 | 加载点 |
|---|---|---|---|---|
| Prism | `prism/version` 文件内容 `Prism v1.30.0` | **1.30.0**（Step 1 已升） | 约 208 KB（core 58,240 + components 129,466 + plugins 14,954 + themes/css） | `MarkdownTextConverter.java:381-394`（`getViewHlPrismIncludes`），注入顺序见该函数 |
| KaTeX | `katex.min.js` 内 `version:"0.19.0"` | **0.19.0**（Step 3 已升） | 1,408,413 B（其中 `fonts/` 1,076,572 B；升级前 1,738,632 / 1,402,440 B） | `MarkdownTextConverter.java:101-104`（css + katex.min.js + katex-render.js + mhchem.min.js），`:105` 一条 `.katex{font-size:inherit}`；**受 `:273-279` 的开关控制，默认不注入**（§1.5） |
| Mermaid | `mermaid/version-info.txt` + bundle 内 `"12.2.0"` ×1 | **12.2.0**（Step 5 已升） | 5,494,801 B（升级前 3,572,661 B；占全部资产 55%） | `:107` 注入 + `:288-293` 条件 `mermaid.initialize(...)` |
| asciidoctor.js | bundle 头注释 `Asciidoctor.js 2.2.6` | 2.2.6 | 761,769 B（+ css 32 KB） | `AsciidocTextConverter.java:27,28,34`，JS API 调用在 `:52` |
| org-js | bundle 内**无版本标记**（webpack bootstrap） | 未知 | 55,788 B | `OrgmodeTextConverter.java:19`，JS API 调用在 `:44` |

两个版本标记文件（`prism/version`、`mermaid/version-info.txt`）是上游留的人类可读记录，**升级必须同步改**，否则下次查版本又只能靠 grep 压缩包。

## 1.5 运行时实测（Step 0，2026-10-18，pixel6 / API 34 / WebView Chrome 113）

用 `D:\workspace\js-libs\js-probe.md`（自建 fixture，含 6 种非预置语言代码块 + 行内/块级数学 + mhchem + mermaid + admonition + 表格 + 任务列表 + 脚注 + 上下标 + 未知围栏语言）在 debug 包（`net.gsantner.markor.debug`）里渲染，经 CDP 读回 DOM：

| 探针 | 结果 |
|---|---|
| Prism | 已注册 28 个语言；6 个代码块 token 数 `[17,10,12,11,5,52]`（python/bash/java/rust/yaml/javascript 全部高亮成功）；`Prism.plugins.autoloader.languages_path` = `file:///android_asset/prism/components/` |
| KaTeX | `katex.version` = 0.16.10；`.katex` 节点 14 个 = 7 条公式 × (占位 + KaTeX 自身输出)，`.katex .katex-html` 7 个，`.katex-error` 0 个；mhchem 已挂（`\ce{}` 三条正常） |
| Mermaid | `div.mermaid` 带 `data-processed="true"` 且内含 `<svg>`；无 "Syntax error in text"；额外生成 `div.mermaidTooltip` |
| Admonition | `!!!` 与 `???` 两种都渲染，折叠标题可点 |
| 围栏语言归属 | ` ```notalang ` 与 ` ```mermaid2 ` 都保持 `pre > code.language-*`，**只有 `mermaid` 变成 div** |

**两个前置条件，否则测不到东西**：

1. **数学默认是关的**：`AppSettings.java:222-224` 的 `pref_key__markdown_render_math` 默认 `false`，关掉时 `MarkdownTextConverter.java:273-279` 走 else 分支把每个 `$` 转义成 `\$`（KaTeX 的 css/js 完全不注入，`$...$` 还会被 `SuperscriptExtension` 吃掉 `^`）。本单初稿据此误判过一次"KaTeX 没加载"。
2. debug 包是新 applicationId，首次运行要过 intro + "All files access" 开关；基线截图前需手工完成（或照 §5 用 `adb root` 直接写 `app.xml`）。


## 2. 目标版本与获取通道（查询于 2026-10-18）

| 库 | 现 | 目标 | 依据 |
|---|---|---|---|
| Prism | 1.29.0 | **1.30.0** | npmmirror `prismjs/latest` = 1.30.0；`gh api repos/PrismJS/prism/releases/tags/v1.30.0` → 2025-03-10 |
| KaTeX | 0.16.10 | **0.19.0** | npmmirror `katex/latest` = 0.19.0；KaTeX releases v0.19.0 = 2026-10-01（0.18.x 系列最新 0.18.11 同日） |
| Mermaid | 11.17.2 | **12.2.0** | npmmirror `mermaid/latest` = 12.2.0；monorepo release tag `@mermaid-js/tiny@12.2.0` 2026-10-13 |
| asciidoctor | 2.2.6 | 4.1.1（**不升**） | npmmirror `asciidoctor` 与 `@asciidoctor/core` 均 4.1.1；release v4.1.1 2026-10-05 |
| org-js | 0.2.0（**不升**） | — | npm 包名是 `org`，latest 0.2.0；`mooz/org-js` 最后提交 2015-02-21（无 GitHub release） |

**下载通道坑（实测）**：npmmirror 的 `https://registry.npmmirror.com/<pkg>/-/<pkg>-<ver>.tgz` 返回 **302**，`curl` 不加 `-L` 会得到一个约 80 字节的文本文件（内容就是 `Redirecting to https://cdn.npmmirror.com/packages/...`），看起来像下载成功。要么 `curl -sL`，要么直接用 `cdn.npmmirror.com` 的地址。

## 3. 三条耦合（决定了不能乱升、也不能一把 `cp -r`）

### 3.1 KaTeX 是"三件套 + 字体"，必须同源同版本

- 注入的四个文件里，`katex.min.css` / `katex.min.js` / `fonts/` 必须配套；`mhchem.min.js` 现在就是 KaTeX 仓的 `contrib/mhchem`（`gh api repos/KaTeX/KaTeX/contents/contrib` 列出 `auto-render / copy-tex / mathtex-script-type / mhchem / render-a11y-string`）。**它随同一个 npm tarball 发布**（`package/dist/contrib/mhchem.min.js`），所以别分头下载——一次下载 tarball 取 4 处，版本天然对齐。
- **KaTeX 0.18.0 的 BREAKING 是 "prefix css classes"**（release note 原文：*users who apply custom styles or have allowlists targeting KaTeX's internal classes must update their selectors*）。这条对本仓影响已实测：
  - `katex-render.js:3` 用 `getElementsByClassName("katex")` 挑占位元素，`:10` 调 `katex.render(..., {throwOnError:false})`；
  - 占位的 class 由 vendored 扩展写死：`app/thirdparty/java/.../FlexmarkKatexExtension.java:85-86`（`INLINE_MATH_CLASS` / `DISPLAY_MATH_CLASS` 默认值都是 `"katex"`）；
  - 0.19.0 的 `katex.min.css` 里 `.katex` 仍出现 379 次（0.16.10 是 378 次），新增的是 `.katex-sizing`(121) / `.katex-inner`(7) 这类**带前缀的内部类**；Markor 只自定义了 `.katex{font-size:inherit}`（`:105`）→ **不冲突，无需改 Java/JS**。
  - 0.19.0 的另一条 BREAKING 是 `strict` 默认行为（缺字符度量 → `symbolNotInFont` 走 warn/callback）。Markor 没传 `strict`，影响只是控制台噪声。
- 字体：0.16.10 与 0.19.0 的 `dist/fonts` 都是 **60 文件 / 20 族**，且**文件名逐一对得上**（`diff` 两份 `ls` 结果为空、两版 `katex.min.css` 各引用同样的 60 个 `url(fonts/…)`，无重命名）→ 整目录替换，不用挑。但**字节全变了**：0.16.10 的 `fonts/` 1,402,440 B → 0.19.0 的 1,076,572 B（−325,868 B，60 个文件全部变小，字体重新裁剪过）。Step 3 实测 60 个字体文件的 sha256 全部改变，无一命中旧值。
- `katex/auto-render.min.js`（3,481 B）全仓零引用（`grep -ril "auto-render"` 只命中它自己）→ **死资产，升级时顺手删**，不要"因为它在 katex 目录里就一起更新"。

### 3.2 Mermaid 是大版本跳跃，而且"围栏 → 元素"这一步在库里而不在 Java 里

- 配置写法是老式：`:291-293` 拼出 `mermaid.initialize({theme:'dark'|'default',logLevel:5,securityLevel:'loose'})`。`logLevel` 用**数字 5** 属于旧接口，v11.17.2 实测接受，**v12.2.0 也已实测接受**（Step 5：图正常渲染、无 `Syntax error in text`，Java 侧零改动）→ 这条不再是风险，但它是"未升级前不要主动改成字符串"的理由。
- **本单初稿在这里判断错过一次**：当时按"Java 侧没有任何代码生成 `.mermaid` 元素"推断 mermaid 图其实没渲染。Step 0 实测推翻：`div.mermaid` 确实存在并出了 SVG。真实机制是 **mermaid 自己**把 `pre > code.language-mermaid` 改写成 `div.mermaid`（`data-processed="true"`，并沿用 flexmark 的 `line` 属性）；证据是同一文档里 ` ```notalang ` 与 ` ```mermaid2 ` 仍是 `pre > code`，只有 `mermaid` 被转换。
  → 于是 Step 5 的**头号风险变成"v12 是否保留这条 code-block 改写"**，而不是"要不要补 Java"。判据固定为：同一 fixture 下 `div.mermaid[data-processed] svg` 数量仍为 1。**Step 5 实测：保留**（`div.mermaid` 1、`[data-processed]` 1、`svg` 1，`notalang`/`mermaid2` 仍是 `pre > code`）。
  补充一条：两版 bundle 里都**搜不到字面量 `language-mermaid`**（各 0 次命中，`language-` 仅 1 次），所以这条改写不是按该字面量匹配的——判"改写有没有坏"只能读 DOM，别指望 grep 源码定位（Step 0 那条教训的又一例）。
- 体积：升级前单文件 3,572,661 B = 全部 thirdparty 资产的 43%；**升到 12.2.0 后 5,494,801 B（+1,922,140 B），占全部资产 55%**，release 包 11,274,486 → 11,867,781 B（+593,295 B，deflate 后增幅只有 bundle 的三成）。它仍是这一个库里最贵的一项，下次升级前先看这个数字。

### 3.3 有 5 个文件被本地改过，不能被上游覆盖

`git ls-files` 实测这 4 个是仓内自定义文件，命名带 `-markor`：

- `prism/prism-markor.js`（`usePrism` / `refreshLineNumbers` / `setLineNumbers` / `setWrapWords`，被 `:388` 注入）
- `prism/prism-markor.css`（`pre[class*="language-"]` 的行号/换行微调）
- `prism/plugins/line-numbers/prism-line-numbers-markor.{js,css}`（canvas 量字宽算左内边距；`getFontWidth` 就在这里）

第 5 个是 Step 1 才查出来的：**`prism/plugins/toolbar/prism-toolbar.css` 是一份被改过的上游文件**，与 npm 1.29.0 和 1.30.0 两份都不同（1801 B vs 1565 B）——tab 换空格、按钮字号 `0.8em→0.9em` 并加 `min-width:80%`、底色换 `rgba(215,215,215,.4)`、链接色改 `darkblue`、删掉 `-moz-/-ms-user-select` 与 IE11 注释。`git log` 显示这是**上游 Markor 自己**改的（Li Guanglin，PR #2345/#2406/#2542/#2698），不是本 fork 的手笔，但同样不能覆盖。

并且**上游 line-numbers 的 `.min.css` 仓内根本不存在**——`:392` 加载的是 markor 版 css，说明上游那份被有意替换掉了。结论：升级一律**逐文件替换**（只覆盖上游同名产物），禁止 `cp -r upstream/prism app/thirdparty/assets/prism`。

## 4. 分步计划（一步一个 commit，每步给验收判据）

### Step 0 · 渲染基线 —— ✅ 已完成（2026-10-18，结果见 §1.5）

用自建 fixture `D:\workspace\js-libs\js-probe.md`（覆盖面比 `samples/markor-markdown-reference.md` 更贴本单：加了 5 种非预置语言代码块与未知围栏语言对照）+ 上游 `samples/markor-markdown-reference.md` 各渲一次：

1. 重算资产 manifest 与 `docs/js-preview-libs-baseline.txt` 比对 → 零差异（HEAD 未动资产）。
2. debug 包装到 pixel6，先过 intro + "All files access"，再打开文件、切 view 模式，滚动截图存 `D:\workspace\js-libs\shots\before\`（`probe-01..09.png`）。
3. CDP 探针读数记录在 §1.5 表格里，作为"升级后不得倒退"的基线。
4. 交接单 §5 的 6 个未测项由此补齐：数学 ✓、mhchem ✓、admonition ✓、mermaid ✓、脚注 ✓、表格/任务列表 ✓（wikilink/beamer/TOC 属 flexmark 侧，本 fixture 未含，仍归交接单）。

### Step 1 · Prism 1.29.0 → 1.30.0 —— ✅ 已完成（2026-10-18）

**先纠正本单初稿的替换清单**：原列"11 个上游产物"，实测 1.29.0→1.30.0 整个 npm 包只动 4 个文件，其中被 Markor 用到的**只有 `prism.js` 一个**。

证据（两份 tarball 均已核对 sha512 = registry 的 `dist.integrity`，`E:\downloads\jslibs\prism{129,130}\package\`）：

```
diff -rq prism129/package prism130/package
Only in prism130/package: _headers                                  # 网站产物，与库无关
Files .../components/prism-core.js and ... differ                   # 仓内不发布此文件
Files .../components/prism-core.min.js and ... differ               # 仓内不发布此文件
Files .../package.json and ... differ                               # 版本号
Files .../prism.js and ... differ                                   # ← 唯一命中的
```

`prism.js` 里那一行的语义就是 Step 4 里暴露出的那条风险链：`util.currentScript()` 的判断从**特性检测** `'currentScript' in document` 改成**取值检测** `document.currentScript && document.currentScript.tagName === 'SCRIPT'`。1.29 的写法在 `document.currentScript` 为 `null`（async/defer/fragment 场景）时照样进分支并 `return null`，autoloader 拿 `script.src` 就抛 TypeError；1.30 则落到 IE11 的 stack-trace 兜底。Markor 注入的是普通 `<script src>`，两种写法在 WebView 里都走第一分支 → 预期无行为变化，实测确认（下表）。

实际改动 = 2 个文件各 1 行：`prism/prism.js`（换成 1.30.0，LF）+ `prism/version`（`Prism v1.29.0` → `Prism v1.30.0`，无尾换行，沿用原格式）。
`components.js` 与 `LICENSE` 归一化行尾后与 1.30.0 字节相同（两版本本就一致），**故意不再拷贝**，免得制造无意义的行尾抖动。

**行尾口径（以后每步 diff 都要按这个读）**：仓内所有 blob 是 LF，但工作树里 7 个 prism 文件是 CRLF（checkout 遗留，`core.autocrlf=input` 不会改写工作树）。所以清单里 `prism.js` 从 `60144 B` 变 `58240 B` 是**行尾归一化**（1904 个 CR），不是内容缩水；单行 `.min.js` 因此完全不受影响。

验收结果（全部通过）：

| 判据 | Step 0 基线 | Step 1 实测 |
|---|---|---|
| `prismLangs` | 28 | **28** |
| `tokensPerBlock`（python/bash/java/rust/yaml/js） | 17,10,12,11,5,52 | **17,10,12,11,5,52** |
| 对照围栏 `notalang` / `mermaid2` tokens | 0 / 0 | **0 / 0** |
| `autoloaderPath` | `file:///android_asset/prism/components/` | **同值** ← 1.30 新判据的正身验证 |
| KaTeX / mhchem / mermaid SVG | 0.16.10 · true · 1 | 未动，同值 |
| 截图逐像素比对 `before/probe-top.png` | — | **680 / 2,592,000 px = 0.026%，且全部落在 y=50..81（状态栏时钟）**，内容区零差异 |

APK 内部也直接验了，不靠任务名推断：`unzip -p <debug.apk> assets/prism/prism.js` → sha `b801451d9b4cbf18`、`grep -c currentScript.tagName` = 1、`assets/prism/version` = `Prism v1.30.0`。

顺带修掉探针自身三处定义错误（不是产品问题，但会让下一步误读）：`Prism.version` **在运行时不存在**（Prism core 不导出版本号），所以 `prismVer` 恒为 `null`，版本只能看 `assets/prism/version`；admonition 的真实 class 是 **`.adm-block`**（`adm-note` / `adm-warning adm-collapsed`）而不是 `.admonition`，改选择器后计数 2；`katexRendered` 的定义是"`.katex-html` 有子节点的 `.katex`"= 14，与 §1.5 手工数的"7 条公式"口径不同（占位与输出嵌套），**别把 7→14 读成回归**。

**未做完的一项**：无——release 与单测后来都过了（根因与修法见 §5.3，最终配置为**项目内阿里云镜像 + 无全局 init script + 无 `-I`**）：release 包 **11,519,670 B**，`apksigner verify -v` 仍 v1+v2+v3 = true / v4 = false，签名者 `CN=pisces312`、cert SHA-256 `abadebd2…53ee0b90` 与 AGENTS.md 记录一致；包内 `assets/prism/prism.js` = `b801451d9b4cbf18`（58,240 B）、`assets/prism/version` = `Prism v1.30.0`、`auto-render` 条目 0。单测 `178 tests completed, 11 failed`，失败集合与基线逐条一致（10 条 `WikitextLinkResolverTests` + 1 条 `WikitextFileTests`）。
（体积口径备注：同一次输入在 `-I` 临时配置下产出 11,521,413 B、项目内配置下 11,519,670 B，差 1,743 B 且源码/资产未变——印证 AGENTS.md "别拿字节数当回归判据"；资产与签名两项是按内容核对的，不受体积影响。release 包内 `assets/` 文件数 174 = debug 的 172 + `assets/dexopt/baseline.prof(.prom)` 两条，release 独有。）

### Step 2 · 删死资产 `katex/auto-render.min.js` —— ✅ 已完成（2026-10-18）

`git grep` 全仓（排除该文件自身）**零引用**复核通过：`auto-render` / `autoRender` / `renderMathInElement` 三个串一个都没有。数学渲染走的是另一条路——`katex/katex-render.js` 自己 `getElementsByClassName("katex")` 再逐个 `katex.render(...)`，压根不碰 auto-render 的入口。

一行 `git rm` 之后：清单差 = **只有 `katex/auto-render.min.js` 一行**（154→153 文件，8,374,595→8,371,114 B）。debug 包内资产条目 172 = 153（thirdparty）+ 19（`app/src/main/assets` 的 highlight/img/templates），逐项对得上，没有连带丢失。

验收：CDP 探针读数与 Step 1 **逐字段完全相同**（`katexSpans/Rendered` 14/14、`katexErrored` 0、`mhchem` true、Prism 六块 token 不变）；截图逐像素比对只剩状态栏 858 px（y=47..81，时钟）。→ 零功能影响，判断成立。

一处如实报出的余留疑问：debug APK 字节数 15,149,394 → 15,073,198（−76,196 B），远大于被删文件的 3,481 B。条目清单已核对一致，所以差值来自 zip 对齐/deflate 边界的填充位移（APK 共 1,682 个条目），不是内容变化；release 包侧下一步重新记一次字节数为准。

### Step 3 · KaTeX 0.16.10 → 0.19.0 —— ✅ 已完成（2026-10-18）

从 `katex-0.19.0.tgz`（sha512 `v6Tznz3z…Xpdw/Q==` 对上 registry `dist.integrity`，1,499,743 B）取 4 处：`dist/katex.min.css`、`dist/katex.min.js`、`dist/fonts/*`（整目录 60 文件）、`dist/contrib/mhchem.min.js`。
**不取** `dist/contrib/auto-render.*`（Step 2 已删）。`katex-render.js` 是 fork 自有文件，**保持不动** —— 实测 0.19 不需要新选项，`katex.render(text, elem, {throwOnError:false, displayMode})` 这条调用签名未变。

改动 = **63 个文件**（css 23,198→24,793、js 275,309→272,868、mhchem 33,730→33,706、60 个字体 1,402,440→1,076,572 B），无新增无删除，`git status` 恰好 63 个 `M`；资产清单 diff 也只有这 63 行，非 katex 路径零命中。

验收（`pref_key__markdown_render_math` 已确认在设备上为 `true`）：

| 判据 | Step 2 后 | Step 3 后 |
|---|---|---|
| `katexVer` | 0.16.10 | **0.19.0** |
| `katexSpans` / `katexRendered` / `katexErrored` | 14 / 14 / 0 | **14 / 14 / 0** |
| `mhchem`（`renderToString("\\ce{H2O}")` 无 error） | true | **true** |
| Prism 六块 token、`prismLangs`、`autoloaderPath` | 17,10,12,11,5,52 · 28 · `…/prism/components/` | **逐项相同** |
| Mermaid `divs/svg/error` | 1 / 1 / false | **1 / 1 / false** |
| 顶部截图逐像素比对 | — | **893 / 2,592,000 px = 0.034%，全部落在 y=47..81（状态栏时钟）** |

**"字体没换对 ⇒ 渲染成普通衬线字"这条判据做成了量化检查**，脚本 `D:\workspace\js-libs\katex_font_probe.py`（走 CDP 的 `CSS.getPlatformFontsForNode`，读的是真实绘字字体，不是猜）：

- `.katex .mop`（∫）→ `KaTeX_Size2` `isCustomFont=true`；`.katex .mathnormal`（x）→ `KaTeX_Math`；`.katex .mrel` → `KaTeX_Main`；对照节点 `p` → `Roboto` `isCustomFont=false`（证明这个探针确实能分辨回退）。
- `document.fonts` 声明 12 个 KaTeX 族，本 fixture 用到的 `KaTeX_Main/Math/Size2` = `loaded`，其余 `unloaded`（浏览器按需加载，不是缺文件）；`check("1em KaTeX_Main")` = true。
- 同一段 `\int_0^1 x^2 dx`：KaTeX 排版宽 55.73 px，serif 回退宽 68.45 px（比值 0.814）→ 用的是 KaTeX 自己的度量。
- 人眼复核：`shots/step3/probe-math.png` 里行内 `E=mc^2`、`α_i+β^2≤γ`，块级 `∫_{-∞}^{∞} e^{-x^2}dx=√π`、`∂u/∂t=c^2∇^2u`，mhchem `2H_2+O_2→2H_2O`、`CaCO_3⇌Ca^{2+}+CO_3^{2-}` 全部正常。

构建侧：release 包 **11,274,486 B**（Step 2 后 11,519,670 B，−245,184 B；字体是压缩存储所以降幅小于目录差值），构建日志零 WARNING，`apksigner verify -v` 仍 v1+v2+v3 = true / v4 = false；包内 `assets/katex/` 64 条（css + js + mhchem + render + 60 字体），`katex.min.js` sha `103a53763cc033bb`、内嵌 `version:"0.19.0"`、`KaTeX_AMS-Regular.woff2` 28,076 B = tarball 那份。单测 `178 tests completed, 11 failed`，与基线逐条一致。

§3.1 预判的两条 BREAKING 落地复核：`.katex` 选择器未冲突（`katex-render.js` 与 `FlexmarkKatexExtension` 零改动，DOM 计数不变即证）；`strict` 默认行为变化在设备上无可见影响（`katexErrored` 仍 0，截图无异常）。

### Step 4 · ~~决策项：Prism autoloader 的 `languages_path`~~ —— 本单初稿误判，已作废

初稿写的是"`languages_path` 全仓未设置，默认 `components/` 会相对 base URL 解析到不存在的路径，所以 `components/` 那 129 KB 是死重"。**Step 0 实测推翻**：运行时 `Prism.plugins.autoloader.languages_path` = `file:///android_asset/prism/components/`，python/bash/java/rust/yaml 五个非预置语言的代码块全部拿到 token。

机制（读上游 min.js 得到）：autoloader 先取 `Prism.util.currentScript().src`，用正则把尾部换成 `components/`，于是**从它自己的 `<script src>` 绝对地址推导出资源路径**，与页面 base URL 无关。Markor 一行都不用写。

**但这正好把 Step 1 的真实风险暴露出来**：Prism **1.30.0 的 release note 只有一条改动** —— *"check that `currentScript` is set by a script tag"*（PR #3863），改的就是这条推导链。所以 Step 1 换完文件后必须重跑 §5 的 Prism 探针，`prismLangs` 与 `tokensPerBlock` 不得倒退；只看代码块"有没有颜色"不够，因为预置的 javascript/yaml 那几个即使 autoloader 失效也可能看起来正常。

### Step 5 · Mermaid 11 → 12（单独一轮）—— ✅ 已完成（2026-10-18，判定为可升）

换 `mermaid.min.js` 单文件 + 改 `version-info.txt`（格式沿用原文件：无结尾换行，93→92 B）。来源 `mermaid-12.2.0.tgz`（sha512 `w7nr/SFr…tOiHzw==` 对上 registry `dist.integrity`，25,955,888 B），取 `dist/mermaid.min.js`；仓内那份 11.17.2 与 npm `dist/mermaid.min.js` **字节数完全相同**（3,572,661 B），证明当初就是走同一产物，替换是同一条链上的对换。

三个风险点逐条实测：

1. **code-block 改写仍在**（本步的"不可升"判据）：`div.mermaid` = 1、`div.mermaid[data-processed]` = 1、其内 `svg` = 1；同一文档里 `notalang` / `mermaid2` 两个围栏**仍然是 `pre > code`**（探针 `stillCodeBlocks` 逐项列出）。→ 不需要在 Java 侧补元素生成。
2. **`logLevel:5` 数字写法 v12 仍接受**：`MarkdownTextConverter.java:291-293` 那行配置零改动，图正常出、`Syntax error in text` 未出现。§3.2 记的"未实测"就此闭合。
3. **图形内容没丢，但画法变了**（v11 / v12 各装一个 APK，用同一表达式 `D:\workspace\js-libs\mermaid_geom.py` 量的）：

   | 字段 | 11.17.2 | 12.2.0 |
   |---|---|---|
   | `svg.viewBox` 盒 | 539.94 × 172.10 | **668.57 × 193.03**（+23.8% × +12.2%） |
   | 渲染宽 / 高 | 367 / 117 px | 367 / **106** px |
   | `g.node` / `path.flowchart-link` | 4 / 4 | **4 / 4** |
   | `.nodeLabel` | 4 | **4** |
   | `.edgeLabel` / `foreignObject` | 8 / 8 | **2 / 5** |
   | 字体族 | `"trebuchet ms", verdana, arial, sans-serif` | 同值 |

   `.edgeLabel` 8→2 不是丢标签：SVG 文本里 4 个节点名 + `Link text` 全在（v11 给每条边都建了标签元素，v12 只给有字的建）。另外 v12 的 SVG 内嵌样式里出现 `[data-look="neo"]`、`.collapsed-indicator`、`.swimlane.cluster` 这类 v11 没有的规则，节点带上了 `drop-shadow` —— **截图上能直接看出图形观感变新**，这是本步唯一可见的行为差异，属预期内的上游样式改动。

其余 DOM 读数与 Step 3 逐项相同（`prismLangs` 28、六块 token、`katexSpans/Rendered/Errored` 14/14/0、`mhchem` true、`admonition` 2、`footnotes` 2、`tableOk` 1、`checkbox` 3）。

产物侧：debug 包 16,911,849 B（+1,838,651 B）、**release 包 11,867,781 B**（Step 3 后 +593,295 B，压缩包增幅远小于 bundle 的 +1,922,140 B），构建日志零 WARNING，`apksigner verify -v` 仍 v1+v2+v3 = true / v4 = false；包内 `assets/mermaid/mermaid.min.js` = 5,494,801 B 且内嵌 `"12.2.0"`。单测 `178 tests completed, 11 failed`，与基线逐条一致。资产清单 153 文件 / **9,966,515 B**（Mermaid 现占 55%）。

### Step 6 · 明确不升的两个

- **asciidoctor**：仓内是 `Asciidoctor.js 2.2.6`（Opal 编译的浏览器包），Markor 调的是 `var asciidoctor = Asciidoctor();`（`AsciidocTextConverter.java:52`）——工厂式调用是 2.x 的 API 面。4.1.1 的浏览器包入口文件名与全局 API 形状**未核实**，且 asciidoc 预览在本 fork 的使用频率低。要升就得先做一个"4.x 浏览器包 + `Asciidoctor()` / `new Asciidoctor()` 在 API-34 WebView 里能否跑通"的最小实验。
- **org-bundle.js**：javadoc（`OrgmodeTextConverter.java:16-18`）写明是 browserify 打包的 `mooz/org-js`，npm 包名 `org` 0.2.0，上游最后提交 2015-02-21 → 没有可升的东西；bundle 里也没有版本标记。维持现状。

## 5. 每步统一验收动作

### 5.1 CDP DOM 探针（主判据，比截图客观且可量化）

Markor 在 `DocumentEditAndViewFragment.java:1160` 调了 `WebView.setWebContentsDebuggingEnabled(true)`，所以 debug 包的预览页可以直接用 Chrome DevTools Protocol 求值，不需要 root 之外的任何改动：

```bash
A=D:/dev/android_sdk/platform-tools/adb.exe; P=net.gsantner.markor.debug
$A -s emulator-5554 shell am start -a android.intent.action.VIEW \
    -d "file:///storage/emulated/0/Documents/markor/js-probe.md" -t "text/markdown" \
    -n $P/net.gsantner.markor.activity.DocumentActivity   # 进编辑页
$A -s emulator-5554 shell input tap 817 201              # 点 action_preview 切视图
S=$($A -s emulator-5554 shell "cat /proc/net/unix" | grep -o -E "webview_devtools_remote_[0-9]+" | sort -u | tail -1)
$A -s emulator-5554 forward tcp:9222 localabstract:$S
WS=$(curl -s http://127.0.0.1:9222/json | grep -o -m1 'ws://[^"]*')
python D:/workspace/js-libs/cdp_eval.py "$WS" 'JSON.stringify({katexVer:typeof katex!=="undefined"?katex.version:"NO",prismLangs:Prism?Object.keys(Prism.languages).length:-1,tokensPerBlock:[...document.querySelectorAll("pre > code")].map(c=>c.querySelectorAll(".token").length),mermaidSvg:document.querySelectorAll(".mermaid svg").length})'
```

要点（都是踩过的）：

- 打开文件**必须**用 `-n $P/net.gsantner.markor.activity.DocumentActivity` + `-t "text/markdown"`；对 `MainActivity` 发 `file://` VIEW 只会落到文件列表，不会打开文档。
- 预览是**新起的 WebView 进程**，`/proc/net/unix` 里会有两个 `webview_devtools_remote_<pid>`。**别照 pid 大小猜**：把两个各 forward 到一个端口，`curl -s http://127.0.0.1:9223/json` 里**有 `type:"page"` 目标的那个才是预览**（另一个返回空数组）。预览页的 url/title 都显示 `about:blank`，所以只能靠"有没有 page 目标"分辨。
- 点预览按钮的坐标别硬记：`uiautomator dump //sdcard/ui.xml` 之后 `tr '>' '\n' | grep action_preview | sed -n 's/.*bounds="\([^"]*\)".*/\1/p'` 取中心点（本机 1080×2400 上是 `[765,138][870,264]` → 817,201）。
- **数学字体有没有真的换上，用 `CSS.getPlatformFontsForNode` 判定，别看截图猜**：`D:\workspace\js-libs\katex_font_probe.py` 走 DOM/CSS 域读真实绘字字体（`.mop`→`KaTeX_Size2`、`.mathnormal`→`KaTeX_Math`、对照 `p`→`Roboto` `isCustomFont=false`），并量同一段公式在 KaTeX 排版与 serif 回退下的宽度比。Step 3 就是这么确认的。
- **Mermaid 的版本号读不到 DOM 上**，别把探针返回值当版本判据：`typeof mermaid !== "undefined"` 为真（全局在），但 `mermaid.version` 在 **v11.17.2 与 v12.2.0 上都是 undefined**（实测 `loaded-no-version`；写成 `(window.mermaid && window.mermaid.version) || "NOT-LOADED"` 就显示 `NOT-LOADED`）。和 `Prism.version` 同一类**探针定义错误**，不是产品问题。版本只能看 `assets/mermaid/version-info.txt` 与 bundle 内嵌串，或 `unzip -p <apk> assets/mermaid/mermaid.min.js | sha256sum` 对齐 tarball。
- **跨大版本 A/B 要用 DOM 几何，别用截图像素差**：`D:\workspace\js-libs\mermaid_geom.py`（返回 `divs/processed/stillCodeBlocks` + SVG 的 `viewBox`、节点/边/`.edgeLabel`/`foreignObject` 计数）配合 `D:\workspace\js-libs\open_preview.sh`（启动文档→点预览→自动挑出带 `type:"page"` 的那个 socket 并 forward）。两个 APK 各跑一次同一表达式即可。**截图 diff 在这里会误导**：v12 的新样式（`[data-look="neo"]` 节点投影）本身就让裁剪区 83% 像素变化，看不出"内容有没有丢"。
- `cdp_eval.py` 是零依赖的 stdlib WebSocket 客户端（`D:\workspace\js-libs\`），因为本机没有可用的 ws 库；表达式里只用双引号，整条用单引号包给 bash。
- 数学探针前要先开 `pref_key__markdown_render_math`：`adb root` → 改 `/data/data/$P/shared_prefs/app.xml` 加 `<boolean name="pref_key__markdown_render_math" value="true" />` → `cp` 回去（cp 覆盖会保留原 owner，无需 chown）→ force-stop 生效。`run-as` 读不到 `/sdcard`，`adb push` 到 `/data/local/tmp` 会 `fchown failed`，这两条都试过不通。

### 5.2 其余动作

1. 重算资产清单并与基线 diff（清单里存的是工作树字节数，多行文件按 Step 1 的行尾口径读）：
   ```bash
   cd app/thirdparty/assets && find . -type f | sort | while read f; do
     printf "%s|%s|%s\n" "$(sha256sum "$f" | cut -c1-16)" "$(stat -c %s "$f")" "${f#./}"
   done > D:/workspace/js-libs/assets-after-stepN.txt
   diff D:/workspace/js-libs/assets-after-stepN.txt ../../../docs/js-preview-libs-baseline.txt
   ```
   预期差异必须能被"本次替换的文件"解释；出现计划外的文件变化 = 覆盖错了目录。通过后把新清单覆盖回 `docs/js-preview-libs-baseline.txt`。
2. `GRADLE_USER_HOME=D:/dev/.gradle ./gradlew :app:assembleFlavorDefaultDebug` 必过；`testFlavorDefaultDebugUnitTest` 维持 178 跑 / 11 失败基线（JS 资产不参与单测，当回归哨兵用）。
   debug 包过了之后**还要开包核对**，别只看任务名：`unzip -p <apk> assets/<lib>/<file> | sha256sum` 必须等于 tarball 里那份。
3. release 包也构建一次并记字节数（R8 无关，但资产体积在 release 里最直观）。
4. 装到 pixel6 走 Step 0 的同一份 fixture，**前后截图逐像素比对**：`python D:\workspace\js-libs\shotdiff.py before/probe-top.png stepN/probe-top.png`。
   截图要滚回同一位置（预览里连划 6 下 `input swipe 540 600 540 1900` 足够到顶）。判读方式：把差异行数限制在状态栏（y<100，只有时钟）即视为内容区零差异。

### 5.3 构建突然起不来：`maven.google.com` 是死主机 + 缓存按仓库 id 分键（2026-10-18，已定位并在本仓库修好）

Step 1 的 debug 链路全绿之后，release 构建与单测起不来，**与 JS 替换无关**，但过程里纠正了一条长期写在 AGENTS.md 里的错记录，值得单记：

- 症状：`Could not resolve com.android.tools.build:gradle:9.4.0` → `Could not HEAD https://maven.google.com/...` → `Connection timed out`。加 `--offline` 变成 `android-maven-gradle-plugin-2.1.jar: No cached version available for offline mode`。
- 排除"真缺包"：两个 jar/pom **都在** `caches/modules-2/files-2.1/`，`metadata-2.107` 里也有 9.4.0 的 descriptor。
- 中间结论（当时用来绕开，**但根因判断错了一半**）：Gradle 的依赖缓存**按仓库 id 分键**，id 一变即判未命中 → 联网。共享 `init.d/mirror.gradle` 被改成只做 settings 级注入（`beforeSettings` → `pluginManagement` / `dependencyResolutionManagement`），够不到 root `build.gradle` 的 `buildscript{}`，而 `dependencyResolutionManagement` 又因项目自带 `allprojects{repositories}`（PREFER_PROJECT）而不生效。一次性 `-I` 脚本（`gradle.allprojects { project.buildscript.repositories {...} }`，项目创建期注入所以排在项目自带仓库之前）能立刻让 `--offline` 4s 通过，看似坐实。
- **真正的根因（逐主机 curl 推翻上面）**：死的是 `maven.google.com` 这一台（旧重定向主机，20s 超时 / 000），不是"Maven Central 与 Google 都连不上"。实测 `google()` 指向的 `dl.google.com/dl/android/maven2` 0.24s 可达、`repo.maven.apache.org` 可达、`jitpack.io` 可达、阿里云可达。本仓库 `build.gradle` 里硬编码了两处 `maven { url 'https://maven.google.com' }`，只要它排在解析链上，Gradle 就把连接超时当**致命错误**而不是"这台未命中、换下一台"，于是整体失败——与镜像在不在无关。
- 处置（用户定的口径：**不用全局 init script，但要共享全局缓存**）：把阿里云 google/public/gradle-plugin 写进本仓库 `build.gradle` 的 `buildscript.repositories` 与 `allprojects.repositories` 最前，并**删掉两处 `maven.google.com`**；`D:\dev\.gradle\init.d\mirror.gradle` 由用户删除。缓存目录 `modules-2/files-2.1` 本身按 sha1 内容寻址，所以共享的收益是**命中率/省下载时间**，不是磁盘去重。
- 验收（无 `-I`、无全局配置）：`--offline :app:tasks` 1s 通过（证明共享缓存重新命中）；`:app:assembleFlavorDefaultDebug :app:assembleFlavorDefaultRelease` 37s 通过，日志零 WARNING/FAILURE。
- 顺带一条：`--offline` 只对"仓库 id 对得上"的缓存有效；改了项目声明的仓库列表后再 `--offline`，会直接报 "No cached version available"，别误判成缺包。

## 6. 许可证与署名备注

- `app/src/main/res/raw/licenses_3rd_party.md` 的 MIT 段列了 KaTex / writeily / Mermaid / asciidoctor.js / asciidoctor-skins，**没有 Prism**（Apache 段列 flexmark-java）。该文件由上游维护；要不要在本 fork 补 Prism 由用户定。
- `asciidoc/dark.css` 来自 `darshandsoni/asciidoctor-skins`，且 javadoc 说明它被改过（`import` 块与 `literalblock` 块）→ 又一个"不能整目录覆盖"的例子。

## 附录 A · 基线清单

- 逐文件：`docs/js-preview-libs-baseline.txt`（当前 **153** 行，Step 2 之前是 154；`sha256 前16位|字节|相对路径`，路径相对 `app/thirdparty/assets/`）。
- 生成命令见 §5 第 1 条。每完成一步，若用户同意把该文件更新为新基线（否则它永远停在升级前）。
