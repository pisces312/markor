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

**第二轮（已完成，结果见 §0.7 + §0.9）：工具链升级**
- 分支 `toolchain-upgrade`（基于 `flexmark-upgrade`）：Gradle wrapper 8.13 → **9.6.1**、AGP 8.13.2 → **9.4.0**（两者本机 `D:\dev\.gradle` 缓存已有，无需下载）。
- 已知预期待摩擦点（以第一次构建报错为准）：
  - Kotlin 插件 1.3.72 与 Gradle 9 不兼容（Gradle 9 需要 Kotlin 插件 ≥ 2.x；`enable_plugin_kotlin` 若关闭则可能不触发）；
  - `com.github.dcendents:android-maven-gradle-plugin:2.1`（build.gradle:31）是 Gradle 4 时代的插件，Gradle 9 大概率不能 apply；
  - AGP 9 移除一批废弃 DSL（PR #2407 的 `compileSdk`/`flavorDimensions = [...]` 改动正是适配方向）。
- 验证：独立一轮构建 + 冒烟；通过后再合并两分支/合回 master。
- 回退点：两轮各自独立提交，任何一轮失败可单独回滚不影响另一轮。

## 0.7 第二轮工具链升级结果（2026-10-18 实测）

**结论（2026-10-18 当天两次更新，最终版）：Gradle wrapper 8.13 → 9.6.1、AGP 8.13.2 → 9.4.0 已落地。**
中间一度认为 AGP 9 进不来而停在 8.13.2，原因是那 198 处 `switch (R.*)`；当天晚些时候按 §0.9 把它们改成 if/else-if 之后，AGP 9 编译通过。

两条互相独立的硬约束，均已取到原始报错，不是推测：

1. **Gradle 9.6 拒绝 AGP 8.x**。`Failed to apply plugin 'com.android.internal.application'` →
   `Plugin 'com.android.internal.application' relies on 'org.gradle.api.problems.internal.InternalProblems`,
   a Gradle internal API that was removed in Gradle 9.6.0. … or use Gradle 9.5.`
   （Gradle 官方升级文档锚点 `upgrading_version_9.html#agp_8x_incompatible`）
   → 反过来也成立：**AGP 9.4 要求 Gradle ≥ 9.6.0**（`Minimum supported Gradle version is 9.6.0. Current version is 9.5.1.`）。
   所以 AGP 8/9 与 Gradle 9.5/9.6 是一个不能交叉的 2×2：要么 8.13.2 + ≤9.5.x，要么 9.4.0 + ≥9.6.0。

2. **AGP 9 生成的 R 字段非 final**。新资源管线（`generate<Variant>RFile` → `compile_r_class_jar`）产出
   `public static int`（非 final），且 `android.nonFinalResIds=false` 已不再被采纳。
   javap 对照同一个字段 `R$string.abid_common_unordered_list_char`：AGP 8.13 的 R.jar 是 `public static final int`，AGP 9.4 是 `public static int`。
   后果：所有 `switch (R.id/R.string.*)` 报「需要常量表达式」，javac 首批报出 **198 处**、实际 **238 个 case 标签 / 20 个 switch / 15 个文件**。
   三个逃生门全部试过、全部无效：
   - `-Pandroid.enableAppCompileTimeRClass=true`（单独用、以及配 `nonTransitiveRClass=true`）→ 字段仍非 final；
     而 non-transitive 会额外让 `R.string.copy` 这类依赖库资源报「找不到符号」，属于反向代价；
   - `-Pandroid.enableNewResourceProcessing=false` → 插件 apply 直接失败（选项在 AGP 9 已移除）；
   - `-Pandroid.generateRJava=true` → 报「removed in version 4.1 of the Android Gradle plugin」。

   出路只有把 switch 改写成 if/else —— 见 §0.9，已实施。

AGP 9 适配期间改的 4 处，AGP 8/9 通用，**已保留在 `toolchain-upgrade`**：

| 位置 | 改动 | 原因 |
|---|---|---|
| `gradle.properties` + `app/build.gradle` `buildFeatures` | 删 `android.defaults.buildfeatures.buildconfig=true`，改模块内 `buildConfig = true` / `resValues = true` | 该属性 AGP 9.0 起是硬错误；`resValue` 在 AGP 9 需要 `buildFeatures.resValues` |
| `app/build.gradle` 末尾 `androidComponents` | 删 `setProperty("archivesBaseName", …)`，改用 `VariantOutput.outputFileName` | `archivesBaseName` 在 AGP 9 移除；新写法产物文件名与升级前逐字节一致 |
| `build.gradle:43-47` 仓库顺序 | jitpack 提到 `maven.google.com` 之前 | 本机连 `maven.google.com` 是 connect timeout，Gradle 视为致命错误而非"未命中"，导致只在 jitpack 的 `com.github.*`（gene-rate / AppIntro）整体解析失败 |
| `app/build.gradle` release `signingConfig` | 新增，显式 v1+v2+v3 | 见 §0.8 |

撤回过又落地的 1 处：`getDefaultProguardFile('proguard-android.txt')` → `proguard-android-optimize.txt`。
AGP 9 强制要求这个替换（旧文件含 `-dontoptimize`）。停在 8.13.2 期间曾撤回以对齐已验证基线，上 AGP 9 之后重新落地 —— 但 **release 包（R8 minify）尚未在 AGP 9 下复测**，见 §0.9 待办。

**验证（AGP 8.13.2 阶段）**：Gradle 9.5.1 + AGP 8.13.2 全量 `assembleFlavorDefaultDebug` 通过（编译 39s），装机 ACTION_VIEW 冒烟，
标题分隔线 / 表格（含右对齐列）/ 任务列表复选框 / Prism kotlin 高亮 / 引用 / 脚注上标+↩ 全部与第一轮一致，logcat 无 FATAL。
AGP 9.4.0 阶段的验证见 §0.9。

## 0.8 release 三签（2026-10-18 实测）

本仓库原本**没有任何 signingConfig**（debug 包的 v1+v2 是 AGP 按 minSdk 21 自动决定的）。新增的 release 配置四个值全部读
环境变量、缺失即 null（release 打包失败，debug 构建不受影响），并显式开 `enableV1/V2/V3Signing`。
在 8.13 基线上单独构建 `assembleFlavorDefaultRelease` 验证（变量隔离，不与工具链混测）：
apksigner（build-tools 37.0.0）报 v1/v2/v3 = true、v4 = false（只是没生成 .idsig，不影响校验），签名者 `CN=pisces312, O=pisces312`，
R8 minify 同时通过（13m33s，产物 12.5MB）。

**AGP 9.4.0 + Java 17 下复测（同日，§0.7 里"必须重新落地并复测 release 包"那条的收尾）**：`assembleFlavorDefaultRelease`
46s 通过（这次 Gradle 缓存热，13m33s 那次含全量依赖解析，两者不可比），apksigner 仍 v1/v2/v3 = true、v4 = false，
签名者 `CN=pisces312, O=pisces312`（cert SHA-256 `abadebd2fc9523628b5dacfa0fb40f652df0e161b71820c7a4b5a40653ee0b90`）。
产物 **11525666 字节**，比 8.13 那次的 12.5MB 小 —— 但同时换了 AGP、语言级别和 proguard 基础规则文件（`proguard-android.txt`
→ `proguard-android-optimize.txt`，见 §0.7），**没有做单变量对照，不能把这 1MB 归给其中任何一项**。
R8 唯一告警是 flexmark 的 `ImageUtils` 引用 `java.awt.AlphaComposite` / `BufferedImage`（桌面端代码路径，Android 上不会走到）。
装机验收：release 与 debug 签名不同 ⇒ 必须先 `uninstall`（笔记本在外部存储，卸载后 `smoke.md` 等文件仍在）；
冷启动进程存活、logcat 无 FATAL/VerifyError/NoClassDefFound，引导页正常渲染；
release 的 dex 只有 2 个（debug 是 18 个），逐个反汇编扫 `invoke-custom` / `invoke-polymorphic` 零命中 ⇒ R8 产物同样是 minSdk 21 可跑的。

## 0.9 switch (R.*) → if/else-if（2026-10-18 实测）

AGP 9 落地的唯一代码前置条件。范围：**15 个文件 / 20 个 switch / 238 个 case 标签**，全在 `app/src/main/java`，
`app/src/test`、`app/src/androidTest` 与 `app/thirdparty` 里没有 R switch（已 grep 确认）。

改写规则（保持语义，不做风格美化）：

- 一组 `case A: case B:` 合并成 `if (x == A || x == B)`；单标签组条件过长（>120 列）才折行。
- 分支体去掉只为结束 switch 而存在的 `break;`；`return` / `throw` 原样保留。
- 剥掉 case 体上多余的 `{ }` 包装，缩进按原基准重算。
- `default:` 一律变成链尾的 `else`；case 之间夹着的注释跟着它下面的标签走，不丢。
- 唯一的 fallthrough 形状（`MarkdownActionButtons.onActionClick` 里 `case abid_common_open_link_browser` 不 return 直接落到 `default`）
  保留原语义：该分支只 `if (followLinkUnderCursor()) return true;`，链尾统一 `return runCommonAction(action);`。
- `FormatRegistry.getFormat` 是**间接别名**：`static final int FORMAT_* = R.string.action_format_*`，
  标签本身不带 `R.`，但常量初始化不是编译期常量，AGP 9 下同样报「需要常量表达式」。它的 `default:` 写在
  `case FORMAT_MARKDOWN:` 之前（两者共用一个分支体），改成链尾 `else` 后语义不变（含 `formatId = FORMAT_MARKDOWN` 的回写）。
- 选择器不是单标识符的 3 处手写：`ActionButtonSettingsActivity.onOptionsItemSelected`、
  `GsFileBrowserDialog.onClicked`、`GsFileBrowserListAdapter.onClicked`，先取 `final int id = …` 再比较。
- `case android.R.id.home` 这类框架 R 仍是编译期常量，本来可以留在 switch 里；这 3 处一起改成 if 是为了同一方法内写法一致。

做法：先写了一个"拒绝优先"的批量转换器（`D:\workspace\markor-switch2if.py`，未入库），只处理能静态判定安全的形状
（选择器是标识符 + 标签全形如 `R.<type>.<name>` + 每个非末分支有终结语句 + default 在末尾），其余全部带行号报出来手工改；
编译器当裁判，改到 `grep -rnE "case +R\\.|case +android\\.R" app/src/main/java` 为空。
复核手段是 `git diff -w`（忽略缩进噪声后只剩结构变化），逐文件看过 20 个 switch 的分支归属与终结语句。

**验证**：Gradle 9.6.1 + AGP 9.4.0 `assembleFlavorDefaultDebug` 通过（25s，产物 14993229 字节），装机启动无 FATAL、
首屏引导页正常渲染。`testFlavorDefaultDebugUnitTest` 178 跑 / 11 失败，与 HEAD 基线工作区（Gradle 9.5.1 + AGP 8.13.2 + 未改 switch）
的失败集合逐条一致 → 既有环境失败（Windows 路径非法字符、时区假定），非本次回归。

**未做（下次装机必须覆盖）**：这 20 个 switch 全是点击/菜单分发，自动点击冒烟不在本 fork 的验证分工内，需要手过一遍——
编辑器工具栏各格式按钮（markdown / asciidoc / orgmode / wikitext / todotxt 长按与短按）、溢出菜单（分享系列、格式切换系列、
撤销重做、换行/行号/高亮开关）、文件浏览器对话框（根节点多选 / home / ok）、设置页按钮排序页。
release 包已在 AGP 9 下构建并装机通过，见 §0.8 末尾。

## 0.10 Java 语言级别 8 → 17（2026-10-18 实测）

`app/build.gradle` `compileOptions` 的 `sourceCompatibility` / `targetCompatibility` 从 `VERSION_1_8` 改成 `VERSION_17`，
**minSdk 保持 21**。顺手删掉根 `build.gradle` 里的 `version_minSdk = 17` —— 它在 `:app` 求值时被
`app/build.gradle:16` 的 `rootProject.ext.version_minSdk = 21` 无条件覆盖，是个死值（全仓只有 3 处引用，单模块 `:app`）。

抬语言级别与 §0.9 的 switch 改写**无关**：case 标签必须是常量表达式这条约束在 11/17 都没放宽，
模式匹配 switch 要到 21 preview / 25 才落地，而那时也只能写成 `case int i when i == R.string.x ->`，等价于 if。

真正的分界不是语言级别而是 **minSdk 决定的 API 可用性**：

- 到 17 后立刻可用（纯语法或 javac 侧降级，产物仍是 minSdk 21 的 dex）：`var`、箭头 switch 表达式 + `yield`、文本块、
  `instanceof` 模式、sealed。
- **不可用**：Java 9+ 标准库 API（`List.of` / `Map.of` / `String.isBlank` / `Optional.isEmpty` / `Files.readString`）——
  这跟 sourceCompatibility 无关，要么开 `coreLibraryDesugaring`（加依赖、加 dex 体积），要么抬 minSdk 到 24/26。
- **别用 records**：`equals/hashCode/toString` 由 `invokedynamic ObjectMethods.bootstrap` 生成，minSdk 21 上 D8 无法降级。

实测：`assembleFlavorDefaultDebug` 通过（12s），`javac/flavorDefaultDebug/.../FormatRegistry.class` 的 class major
从 52 变成 **61**（=Java 17），确认语言级别真的生效而不是只改了配置文本；
装机冷启动 pid 正常、logcat 无 `FATAL` / `VerifyError` / `NoClassDefFoundError`；
`testFlavorDefaultDebugUnitTest` 178 跑 / 11 失败，失败集合与 §0.9 之后逐条一致（仍是那批 Windows 路径 + 时区用例）。
另外逐 dex 反汇编扫过全部 18 个 `classes*.dex`，`invoke-custom` / `invoke-polymorphic`（两者都要 API 26+）命中数为 0
—— 说明 javac 17 为字符串 `+` 生成的 invokedynamic 已被 D8 按 minSdk 21 降回 `StringBuilder`，dex 层面确实是 21 可跑的。

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
