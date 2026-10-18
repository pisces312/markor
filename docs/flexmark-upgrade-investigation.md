# 升级 Markor 的 Markdown 渲染库 Flexmark —— 调研与实施交接单

> 目的：把 Markor 渲染 markdown 用的第三方库 **Flexmark 从 `0.42.14` 升到 `0.64.8`**，以提升 md→HTML 的解析/渲染质量。
> 本文件是**给新会话的自包含交接单**（新会话看不到此前对话），所有路径/行号/字节数均可直接复核。
> 调研日期 2026-10-18。凡未构建/未跑过的结论都标了「未实测」。
> 归属：改动全落在本仓库（fork `pisces312/markor`）。与 M2Git 无关——M2Git 只是通过 `ACTION_VIEW text/markdown` 把 `.md` 递给 Markor 打开。

## 0. 结论速览（先看这条）

- **能升，机械改动很小**：版本号 + **6 行 import 改名**，分布在 **2 个文件**。
- **但必须真跑一次 Gradle 构建**才能确认没有更深的 API 漂移（本轮未构建 = 未实测）。
- **质量提升真实但温和**：主要是解析边界（嵌套列表/复杂表格/autolink/脚注/HTML 块）的规范符合度与 bug 修复；典型 README 肉眼差异不大。
- **flexmark 已停更**：`0.64.8`（2023-05-18）是 Maven Central 的 `<latest>`/`<release>`，`maven-metadata.xml` 的 `lastUpdated=20230523`。升到最新 = 升到一个 2023 年的版本，之后无更新活跃版本。
- **风险信号**：上游 Markor 到 v2.16.1（2026-03）仍 pin `0.42.14`，跨过 flexmark 3 年版本没升——需回归验证是否有行为/体积顾虑。

## 0.5 勘误与进展（2026-10-18 更新）

**上游已有现成升级 PR**：[gsantner/markor#2407](https://github.com/gsantner/markor/pull/2407)（guanglinn，2024-08 开，open）做了一模一样的升级，作者实测构建/运行通过，gsantner 2026-01-31 表态「下次更新后合并」但截至本次仍未合并。其 diff（`gh pr diff 2407 --repo gsantner/markor`）是已验证补丁，勘误两处：

1. **§2 改动面被低估**——除 `util.options→util.data` 外还有：
   - `util.builder.Extension` → `util.misc.Extension`（§2 说不用改，**是错的**）
   - `util.html.Attributes` → `util.html.MutableAttributes`（`LineNumberIdProvider.setAttributes` 签名随之变化）
   - `com.vladsch.flexmark.superscript.SuperscriptExtension` → `com.vladsch.flexmark.ext.superscript.SuperscriptExtension`
   - `flexmarkRenderer.withOptions(options).render(document)` → `flexmarkRenderer.builder(options).build().render(document)`
   - `AttributeProviderFactory.create` → `apply`（返回值加 `@NotNull`，`getBeforeDependents/getAfterDependents` 泛型改为 `Set<Class<?>>`）
   - katex 扩展（FlexmarkKatexExtension）：`InlineParser`→`LightInlineParser`、`CustomNodeRenderer`→`NodeRenderingHandler.CustomNodeRenderer`、`DelimitedNode` 移到 `util.ast`、`NodeRendererFactory.create`→`apply`
2. **§3 疑问已解：方案 A 成立**——`flexmark-util:$V` 依赖行不用动，0.60 拆分的子模块走 POM 传递依赖（PR 的 build.gradle 里 flexmark 依赖段零改动，作者构建通过）。

**本 fork 已落地**：`master` 同步上游至 `aff9492c`（2026-10-10，含 30 个新提交）；`flexmark-upgrade` 分支 cherry-pick 了 PR #2407 的全部 8 个内容提交（flexmark + minSdk 21 + androidx/commons-io 升级），两个 Java 文件与 PR head 逐字节一致。**未构建、未跑渲染回归**——下一步按 §4-5 执行。

## 0.6 两轮实施计划（2026-10-18 确定）

变量隔离原则：flexmark 升级与工具链升级分开验证，避免构建失败时无法归因。

**第一轮（进行中）：flexmark 升级**
- 分支 `flexmark-upgrade`（基于上游 `aff9492c`）：PR #2407 全部 8 个提交（flexmark 0.64.8 + minSdk 21 + androidx/commons-io）。
- 工具链不动：Gradle 8.13（wrapper）+ AGP 8.13.2 + Kotlin 插件 1.3.72。
- 验证：`assembleDebug` + 模拟器冒烟（标题/表格/任务列表/代码块/脚注）→ 提交推送。

**第二轮（随后）：工具链升级**
- 分支 `toolchain-upgrade`（基于 `flexmark-upgrade`）：Gradle wrapper 8.13 → **9.6.1**、AGP 8.13.2 → **9.4.0**（两者本机 `D:\dev\.gradle` 缓存已有，无需下载）。
- 已知预期待摩擦点（以第一次构建报错为准）：
  - Kotlin 插件 1.3.72 与 Gradle 9 不兼容（Gradle 9 需要 Kotlin 插件 ≥ 2.x；`enable_plugin_kotlin` 若关闭则可能不触发）；
  - `com.github.dcendents:android-maven-gradle-plugin:2.1`（build.gradle:31）是 Gradle 4 时代的插件，Gradle 9 大概率不能 apply；
  - AGP 9 移除一批废弃 DSL（PR #2407 的 `compileSdk`/`flavorDimensions = [...]` 改动正是适配方向）。
- 验证：独立一轮构建 + 冒烟；通过后再合并两分支/合回 master。
- 回退点：两轮各自独立提交，任何一轮失败可单独回滚不影响另一轮。

## 0.7 第二轮工具链升级结果（2026-10-18 实测）

**结论：Gradle wrapper 8.13 → 9.5.1 落地；AGP 停在 8.13.2 —— AGP 9 和 Gradle 9.6 两头都进不来。**

两条互相独立的硬阻塞，均已取到原始报错，不是推测：

1. **Gradle 9.6 拒绝 AGP 8.x**。`Failed to apply plugin 'com.android.internal.application'` →
   `Plugin 'com.android.internal.application' relies on 'org.gradle.api.problems.internal.InternalProblems`,
   a Gradle internal API that was removed in Gradle 9.6.0. … or use Gradle 9.5.`
   （Gradle 官方升级文档锚点 `upgrading_version_9.html#agp_8x_incompatible`）→ 原计划的 9.6.1 直接不可用，退回本机缓存里的 9.5.1。

2. **AGP 9.4 编不过本仓库代码**。AGP 9 的新资源管线（`generate<Variant>RFile` → `compile_r_class_jar`）生成的 R 字段是
   `public static int`（非 final），且 `android.nonFinalResIds=false` 已不再被采纳。
   javap 对照同一个字段 `R$string.abid_common_unordered_list_char`：AGP 8.13 的 R.jar 是 `public static final int`，AGP 9.4 是 `public static int`。
   后果：所有 `switch (R.id/R.string.*)` 报「需要常量表达式」，实测 **198 处**，集中在
   `format/ActionButtonBase.java`、`activity/DocumentEditAndViewFragment.java`、`format/*/​*ActionButtons.java` 等 10+ 文件。
   三个逃生门全部试过、全部无效：
   - `-Pandroid.enableAppCompileTimeRClass=true`（单独用、以及配 `nonTransitiveRClass=true`）→ 字段仍非 final；
     而 non-transitive 会额外让 `R.string.copy` 这类依赖库资源报「找不到符号」，属于反向代价；
   - `-Pandroid.enableNewResourceProcessing=false` → 插件 apply 直接失败（选项在 AGP 9 已移除）；
   - `-Pandroid.generateRJava=true` → 报「removed in version 4.1 of the Android Gradle plugin」。

   唯一可行出路是把那 198 处 switch 改写成 if/else —— 上游代码风格级改动，fork 不做。等 gsantner 侧动手再说。

AGP 9 适配期间改的 4 处，AGP 8/9 通用，**已保留在 `toolchain-upgrade`**：

| 位置 | 改动 | 原因 |
|---|---|---|
| `gradle.properties` + `app/build.gradle` `buildFeatures` | 删 `android.defaults.buildfeatures.buildconfig=true`，改模块内 `buildConfig = true` / `resValues = true` | 该属性 AGP 9.0 起是硬错误；`resValue` 在 AGP 9 需要 `buildFeatures.resValues` |
| `app/build.gradle` 末尾 `androidComponents` | 删 `setProperty("archivesBaseName", …)`，改用 `VariantOutput.outputFileName` | `archivesBaseName` 在 AGP 9 移除；新写法产物文件名与升级前逐字节一致 |
| `build.gradle:43-47` 仓库顺序 | jitpack 提到 `maven.google.com` 之前 | 本机连 `maven.google.com` 是 connect timeout，Gradle 视为致命错误而非"未命中"，导致只在 jitpack 的 `com.github.*`（gene-rate / AppIntro）整体解析失败 |
| `app/build.gradle` release `signingConfig` | 新增，显式 v1+v2+v3 | 见 §0.8 |

撤回的 1 处：`getDefaultProguardFile('proguard-android.txt')` → `proguard-android-optimize.txt`。AGP 9 强制要求这个替换（旧文件含 `-dontoptimize`），但它会改变 release 的 R8 行为，既然 AGP 9 进不来，就恢复成与已验证基线一致的原值。**将来真上 AGP 9 时必须重新落地并复测 release 包。**

**验证**：Gradle 9.5.1 + AGP 8.13.2 全量 `assembleFlavorDefaultDebug` 通过（编译 39s），装机 ACTION_VIEW 冒烟，
标题分隔线 / 表格（含右对齐列）/ 任务列表复选框 / Prism kotlin 高亮 / 引用 / 脚注上标+↩ 全部与第一轮一致，logcat 无 FATAL。

## 0.8 release 三签（2026-10-18 实测）

本仓库原本**没有任何 signingConfig**（debug 包的 v1+v2 是 AGP 按 minSdk 21 自动决定的）。新增的 release 配置四个值全部读
环境变量、缺失即 null（release 打包失败，debug 构建不受影响），并显式开 `enableV1/V2/V3Signing`。
在 8.13 基线上单独构建 `assembleFlavorDefaultRelease` 验证（变量隔离，不与工具链混测）：
apksigner（build-tools 37.0.0）报 v1/v2/v3 = true、v4 = false（只是没生成 .idsig，不影响校验），签名者 `CN=pisces312, O=pisces312`，
R8 minify 同时通过（13m33s，产物 12.5MB）。

## 1. 当前状态（实测）

| 项 | 值 | 来源 |
|---|---|---|
| 本地克隆 | `/d/3rd-party-projects/markor`，fork `origin=pisces312/markor` | `git remote -v` |
| HEAD | `81d040a3`（工作树仅 1 个未跟踪文件 `CLAUDE.md`） | `git log/status` |
| App 版本 | `versionName 2.16.1` / `versionCode 163`（`app/build.gradle:28-29`） | 与上游最新 release **一致**（`gh release list gsantner/markor`：v2.16.1，2026-03-19，Latest）→ **App 本体无需升级** |
| Flexmark pin | `ext.version_library_flexmark = "0.42.14"`（`app/build.gradle:19`，构建于 2020-01-23） | 源码 |
| Flexmark 依赖 | `app/build.gradle:110-126`，**19 个 artifact**（core + `flexmark-util` + 17 个 ext） | 源码 |
| 目标版本 | Flexmark **0.64.8**（构建于 2023-05-18） | Maven Central `maven-metadata.xml` |

渲染入口：`app/src/main/java/net/gsantner/markor/format/markdown/MarkdownTextConverter.java`（486 行，Flexmark `Parser`+`HtmlRenderer`，挂 20 个扩展）；共用 WebView 基类 `format/TextConverterBase.java`（208 行，`loadDataWithBaseURL`）。

## 2. 破坏性改动面（实测，很小）

唯一的 package 迁移：**`com.vladsch.flexmark.util.options.*` → `com.vladsch.flexmark.util.data.*`**（flexmark 0.60 模块化时改名）。
`util.ast` / `util.builder` / `util.html` / `util.sequence` 四个子包**未改名**——已逐一对比 `flexmark-util-0.42.14.jar` 与 `flexmark-util-data-0.64.8.jar` 的包路径确认。

全仓引用 flexmark 的 Java 文件只有 **2 个**（`grep -rl 'com.vladsch.flexmark' --include=*.java`），需改的 import 共 **6 行**：

### 2.1 `app/src/main/java/net/gsantner/markor/format/markdown/MarkdownTextConverter.java`（2 行）

```
:44  import com.vladsch.flexmark.util.options.MutableDataHolder;  →  util.data.MutableDataHolder
:45  import com.vladsch.flexmark.util.options.MutableDataSet;     →  util.data.MutableDataSet
```
（同文件 `:40 util.ast.Document`、`:41 util.ast.Node`、`:42 util.builder.Extension`、`:43 util.html.Attributes` **不用改**。）

### 2.2 `app/thirdparty/java/other/com/vladsch/flexmark/ext/katex/FlexmarkKatexExtension.java`（4 行，仓内 vendored 扩展）

```
:52  import com.vladsch.flexmark.util.options.DataHolder;         →  util.data.DataHolder
:53  import com.vladsch.flexmark.util.options.DataKey;            →  util.data.DataKey
:54  import com.vladsch.flexmark.util.options.MutableDataHolder;  →  util.data.MutableDataHolder
:55  import com.vladsch.flexmark.util.options.MutableDataSetter;  →  util.data.MutableDataSetter
```
（同文件 `:47-51` 的 `util.ast.*`/`util.builder.Extension`/`util.html.Attribute`、`:56 util.sequence.BasedSequence` **不用改**。）

## 3. 一个构建坑（实测，需构建确认解法）

`flexmark-util-0.64.8.jar` 是**空壳**：实测 1828 B，只含 `META-INF`（7 个条目，无 `.class`）。0.60 起 `flexmark-util` 被拆成独立 artifact：
`flexmark-util-{ast,builder,data,format,html,misc,options,sequence,visitor,collection,dependency,mappers}`，老坐标 `flexmark-util` 退化为聚合壳。

因此 `app/build.gradle:111` 的 `implementation "com.vladsch.flexmark:flexmark-util:$V"` 升级后有两种可能：
- **A**：其 POM 通过传递依赖把 `flexmark-util-*` 子模块带进来 → 只改版本号即可；
- **B**：传递依赖不带 class → 需显式补 `flexmark-util-ast/-data/-html/-sequence/-builder/...` 等用到的子模块。

**是 A 还是 B 未实测，以第一次 `assembleDebug` 的报错为准。**

## 4. 实施计划（建议顺序）

1. （可选，建议）切分支：`git checkout -b flexmark-upgrade`，避免直接动工作树。
2. 改 `app/build.gradle:19` → `ext.version_library_flexmark = "0.64.8"`。
3. 改 §2 的 6 行 import（`util.options` → `util.data`）。
4. `./gradlew :app:assembleDebug`（Gradle/AGP 与签名环境按本仓既有约定；本机 `GRADLE_USER_HOME=D:\dev\.gradle`）。
   - 若报 `flexmark-util` 找不到类 → 按 §3 方案 B 显式补子模块依赖。
   - 若报别的 API 漂移（重点看 `util.html.Attributes#addValue`、各 ext 的 option 常量如 `EmojiExtension.USE_IMAGE_TYPE`/`TablesExtension.*`/`TocExtension.*`/`WikiLinkExtension.LINK_ESCAPE_CHARS`/`AnchorLinkExtension.ANCHORLINKS_*`/`GitLabExtension.RENDER_BLOCK_MATH`）→ 逐个按 0.64.8 API 修。
5. 装到模拟器/真机做**渲染回归对照**（见 §5）。
6. 通过后再决定是否合回主分支 / 出版本。

## 5. 回归验证清单（升级后必须逐项对照渲染效果）

用一个覆盖各扩展的 md 文件，升级前后各渲染一次比对：

- [ ] 基础：标题 / 段落 / 强调 / 链接 / 图片
- [ ] GFM 表格（`TablesExtension`，含对齐、缺列）
- [ ] 任务列表（`TaskListExtension`）
- [ ] 删除线 / 下标（`StrikethroughSubscriptExtension`）、上标（`SuperscriptExtension`）、`<ins>`（`InsExtension`）
- [ ] 脚注（`FootnoteExtension`，注意 `MarkdownTextConverter:318` 有针对脚注的渲染后字符串修补，升级后需确认仍成立）
- [ ] TOC / SimToc（`TocExtension`/`SimTocExtension`）
- [ ] 数学 `$...$`（vendored `FlexmarkKatexExtension` + `katex/` 资产注入，`MarkdownTextConverter:263-270`）
- [ ] admonition `!!!`/`???`（`AdmonitionExtension` + `flexmark/admonition.{css,js}`，`:287-290`）
- [ ] mermaid ```` ```mermaid ````（`:279-284`，条件注入 `mermaid/mermaid.min.js`）
- [ ] 代码块高亮 ```` ``` ````（Prism，`getViewHlPrismIncludes`，`:273-276`）
- [ ] YAML/Jekyll front-matter、wikilink、typographic、gitlab、autolink、anchorlink
- [ ] beamer 演示模式（`:322-336` 依赖 `<hr />` 的渲染后处理，flexmark 输出若变化会受影响）

## 6. 体积与其它备注

- Flexmark jar 尺寸（Maven 实测，压缩包）：完全对齐 Markor 的 19-artifact 集，`0.42.14` = 1,345,971 B；单大头 `flexmark` 388,623 B + `flexmark-util` 385,068 B。0.64.8 因拆模块，artifact 数增多但同族，**dex 增量未实测**（需构建后 `unzip -l` 对比 `classes*.dex`）。
- 观感质量还受 Markor 自带 CSS + Prism/KaTeX/Mermaid 版本影响，flexmark 只决定 md→HTML 正确性这一层；若要进一步提升观感，需另评估这些 JS 资产的版本（不在本单范围）。
- 相关背景调研（Obsidian/Markor/M2Git 三方 md 渲染对照、体积测算）在 M2Git 仓：`D:\3rd-party-projects\m2git\docs\markdown-rendering-investigation.md`。
