# AGENTS.md — Markor fork 现状台账

> 本文件只记录当前状态（工具链 / 构建 / 产物 / 签名），深度调研与计划见 docs/（一个概念一份正文）。
> 更新规则：状态变了就改这里，不写历史沿革。

## 仓库

- 本地 `D:\3rd-party-projects\markor`；origin = `pisces312/markor`（fork，public），upstream = `gsantner/markor`
- `master` = 上游 `adacc842`（**v2026.10.0**，2026-10-26）+ 本 fork 的 29 个提交 + 合并提交 `3c43292f`（2026-10-26）。
  上游已 squash 合入本 fork 的 PR #2407（`2bbe048c` flexmark 0.64.8），并自行做了 minSdk 21 + 移除 multidex——
  `ApplicationObject.java`、`FlexmarkKatexExtension.java` 与上游逐字节相同，`flexmark-upgrade` 分支的历史使命结束
- 工作分支 `flexmark-upgrade`（已合入 master）：上游 PR #2407 的 8 个提交 cherry-pick（flexmark 0.64.8 + minSdk 21 + androidx/commons-io）
- 工作分支 `toolchain-upgrade`（基于 flexmark-upgrade，已合入 master）：Gradle wrapper 9.6.1 + AGP 9.4.0，
  前置改动是把 15 个文件里 20 处 `switch (R.*)` 改成 if/else-if 链（AGP 9 的 R 字段非 final，`case R.*` 不再是常量表达式）
- 文档指针：`docs/flexmark-upgrade-investigation.md`（升级交接单 + 两轮计划）、`docs/markdown-reader-survey-2026-10.md`（md 阅读器调研）、
  `docs/js-preview-libs-upgrade-2026-10.md`（预览层 JS 资产升级计划，基线清单 `docs/js-preview-libs-baseline.txt`）
- 预览层 JS 资产（`app/thirdparty/assets/`，153 文件 / 9,966,515 B）：Prism **1.30.0（2026-10-18 Step 1 已升，本 fork 第一次动资产）**、KaTeX **0.19.0（同日 Step 3 已升：css/js/mhchem + 60 字体共 63 个文件，字体从 1,402,440 B 瘦到 1,076,572 B；死资产 `auto-render.min.js` 已在 Step 2 删除）**、Mermaid **12.2.0（同日 Step 5 已升：单文件 `mermaid.min.js` 3,572,661→5,494,801 B，现占全部资产 55%；v12 渲染样式改为 `[data-look="neo"]` 带节点投影）**、asciidoctor.js 2.2.6、org-bundle.js（无版本标记）。版本判据与升级步骤见上面的 JS 计划单。
  升级只能**逐文件替换**：`prism/` 下有 5 个被本地改过的文件（4 个 `-markor` + 上游改写的 `plugins/toolbar/prism-toolbar.css`），`cp -r` 会冲掉
- **数学渲染默认关闭**：`pref_key__markdown_render_math` 默认 `false`，关掉时 markdown 里每个 `$` 被转义、KaTeX 的 css/js 完全不注入（`MarkdownTextConverter.java:273-279`）。测数学前先开这个开关
- 预览页可用 CDP 探针量化验收（`setWebContentsDebuggingEnabled(true)` 在 `DocumentEditAndViewFragment.java:1160`），步骤与坑见 JS 计划单 §5.1

## 工具链（实测值）

| 项 | 值 |
|---|---|
| Gradle wrapper | 9.6.1（`gradle/wrapper/gradle-wrapper.properties`；`flexmark-upgrade` 分支仍 8.13） |
| AGP | 9.4.0（`build.gradle:13` `version_gradle_tools`）— 硬约束：**AGP 9 需 Gradle ≥ 9.6.0**，**AGP 8.x 需 Gradle ≤ 9.5.x**（9.6.0 删了 `InternalProblems`），两者不能交叉 |
| Kotlin 插件 | 1.3.72（`build.gradle:14`，`enable_plugin_kotlin` 控制是否 apply） |
| JDK | `D:\dev\AndroidStudio\jbr`（Java 21） |
| Java 语言级别 | **17**（`app/build.gradle` `compileOptions` 的 source/target，产物 class major 61）。minSdk 仍 21 ⇒ 只解锁语法（`var`、箭头 switch 表达式、文本块、sealed），**Java 9+ 标准库 API 一律不可用**（`List.of` / `String.isBlank` / `Optional.isEmpty` / `Files.readString`），要用得先开 core library desugaring；records 也别用（`equals/hashCode/toString` 走 invokedynamic，minSdk 21 上 D8 无法降级） |
| Android SDK | `D:\dev\android_sdk` |
| GRADLE_USER_HOME | `D:\dev\.gradle` |
| 依赖镜像 | **写在本仓库里**（`build.gradle:21-30` buildscript、`:45-53` allprojects）：阿里云 google/public（buildscript 另加 gradle-plugin）排最前，mavenCentral/jitpack/google() 兜底。**无全局 init script**（`D:\dev\.gradle\init.d\` 已清空，构建命令不再带 `-I`）；`GRADLE_USER_HOME=D:\dev\.gradle` 仍是全局共享缓存——各项目声明同一组仓库 id ⇒ 缓存只热一次 |

**与上游的工具链分叉（2026-10-26 起）**：上游 v2026.10.0 先升 AGP 9.4.1/Gradle 9.6.0（`da83752e`）又**降回 AGP 8.13.0/Gradle 8.14.6**
（`adacc842`，理由 "Gradle 9.x has some breaking changes"），并把 `android.nonFinalResIds=false` 加了回去（AGP 8 下有效，保住 `switch (R.*)` 写法）；
上游还在 `build.gradle` 里**重新加回了 `maven.google.com`**（本机实测连不上，fork 必须保持删除）。本 fork 决策：**留在 AGP 9.4.0/Gradle 9.6.1**。
后果：每次同步上游，`gradle/wrapper/gradle-wrapper.properties`、`build.gradle`、`gradle.properties`、`app/build.gradle` 四个文件必冲突，
前三个直接取 fork 侧（`git checkout --ours`），`app/build.gradle` 需手工融合（吸收上游版本号/依赖升级，保留 fork 的签名/Java17/`androidComponents`/`.debug` 后缀）。
上游新代码若引入 `case R.*` 必须转 if/else（本次 v2026.10.0 合并实测：上游新增代码零处 `case R.*`，合并后全仓仍为 0）。

**网络坑（2026-10-18 逐主机实测）**：唯一连不上的是 **`maven.google.com`**（旧重定向主机，curl 20s 超时 / 000）；
`google()` 实际指向的 `dl.google.com/dl/android/maven2`、`repo.maven.apache.org`、`jitpack.io` 与阿里云全部直连可用
（404 = 能连通，仓库根没有索引页属正常）。原先"直连 Maven Central/Google 会静默卡死、必须走镜像 init script"是错的记录，
`build.gradle` 里两处硬编码的 `maven { url 'https://maven.google.com' }` 已删除。`github.com` web 直连 HTTPS 仍超时，验证远端用 `gh api`。
Gradle 把连不上的仓库当**致命错误**而不是"未命中"，所以只在 jitpack 上的 `com.github.*` 必须排在可能失败的主机之前（`build.gradle:49-52`）。
**依赖缓存按仓库 id 分键**：改动项目声明的仓库列表会让已存在的产物被判为未命中而重新联网——构建起不来先核对仓库列表，别急着怀疑"缺包"。
定位过程见 `docs/js-preview-libs-upgrade-2026-10.md` §5.3。

## 构建

本 fork **只构建 F-Droid 版**：

```bash
GRADLE_USER_HOME=D:/dev/.gradle ./gradlew :app:assembleFlavorDefaultDebug
```

| flavor | 用途 | 说明 |
|---|---|---|
| **flavorDefault** | **F-Droid 版（本 fork 唯一构建目标）** | 继承 defaultConfig：`IS_GPLAY_BUILD=false` |
| flavorGplay | Google Play 版 | `IS_GPLAY_BUILD=true` |
| flavorAtest | 测试版 | 独立包名 `net.gsantner.markor_test`，`IS_TEST_BUILD=true` |

- 版本：`versionName 2026.10.0` / `versionCode 164`（`app/build.gradle`，随上游 v2026.10.0；上游改用日历年版本 scheme）
- debug 产物：`app/build/outputs/apk/flavorDefault/debug/net.gsantner.markor-v164-2026.10.0-flavorDefault-debug.apk`（16,043,489 B，2026-10-26 合并后实测）
  （文件名由 `app/build.gradle` 末尾 `androidComponents` 重命名块生成，不再依赖已删除的 `archivesBaseName`；
  文件名里的 `net.gsantner.markor` 只是前缀，debug 实际包名是 `net.gsantner.markor.debug`，见下）
- release 产物（AGP 9 + Java 17 + optimize 规则实测）：同目录 `...-flavorDefault-release.apk` **11,867,781 字节**（2026-10-18 Step 5 Mermaid 之后；Step 3 KaTeX 后是 11,274,486，`57d789a6` 那次是 11,521,389）
  （体积随 `GITHASH`/`GITMSG` BuildConfig 字段小幅波动，别拿字节数当回归判据），R8 minify ~40s，
  dex 从 debug 的 18 个合成 2 个；**构建日志已零 WARNING**——flexmark 桌面端类引用由 `app/proguard-rules.pro`
  末尾 7 条精确 `-dontwarn`（java.awt / java.beans / javax.imageio / javax.lang.model / javax.swing /
  org.apache.commons.{beanutils,collections}）消掉，原先是 45 行 `Missing class`
- multidex：`androidx.multidex:multidex` 依赖与 `multiDexEnabled true` **均已删**（上游 `11b765c1` 在 v2026.10.0 里自己做了同一件事，
  合并后跟随上游；此前 fork 实测删除不影响 dex 数：debug 仍 18 dex、release 仍 2 dex）。`ApplicationObject` 两侧改成同样的
  `extends android.app.Application`，该文件与上游逐字节一致
- 单元测试：`./gradlew testFlavorDefaultDebugUnitTest`
  基线：178 跑 / **11 失败**，全部在 `WikitextLinkResolverTests`（链接串里的 `:` 在 Windows 上是非法路径字符）
  与 `WikitextFileTests`（用例假定本机时区 ±1，本机 +8）。已在 HEAD 基线工作区实测，失败集合与升级后逐条一致，属既有环境失败，不是回归；
  v2026.10.0 合并后（2026-10-26）复测仍是 178/11，逐条相同
- R 常量（AGP 9）：`R.id.*` / `R.string.*` 是非 final 字段，**不能出现在 `case` 标签里**；全仓 20 处
  `switch`（含 `FormatRegistry` 那类 `static final int FORMAT_* = R.string.…` 间接别名）已改为 if/else-if 链。
  新增按钮分发一律别再用 `switch (R.*)`；`android.R.id.home` 等框架 R 仍是常量，可以进 `case`。
  **AGP 生成两个 R.jar，别看错对象**：`intermediates/compile_r_class_jar/<variant>/generate<Variant>RFile/R.jar`
  是 `public static int`（非 final，javac 读的就是它 ⇒ `case R.*` 编不过），而
  `compile_and_runtime_not_namespaced_r_class_jar/<variant>/process<Variant>Resources/R.jar` 仍是
  `public static final int`（打进 APK 的那份）。上游当年用 `android.nonFinalResIds=false`（commit `5de71d5e`）
  保住 final，AGP 9.4 已废弃该开关且**实测无效**——删掉前后 compile R.jar 的 sha256 逐字节相同。
  该开关本 fork 已从 `gradle.properties` 删除，别再去翻它找捷径
- 日志读取坑：PowerShell `>` 重定向写出 UTF-16LE，先 `iconv -f UTF-16LE -t UTF-8` 再 grep
- PowerShell 调用坑：bash 里包 `-Command` 参数用单引号，否则 `$env` 被 bash 吞掉

## 代码风格

- AOSP Java Code Style；提交前用 Android Studio 的 auto reformat（`README.md:138`）。
  注意别对 `AndroidManifest.xml` 自动格式化（文件头有 `Do NOT run auto format on this file`）

## debug / release 共存

- `buildTypes.debug { applicationIdSuffix ".debug" }`（`app/build.gradle:81-86`）
  ⇒ debug 包名 `net.gsantner.markor.debug`，release 仍 `net.gsantner.markor`，两者可同时安装。
  刻意**不加** `versionNameSuffix`：APK 文件名末尾已有 `-flavorDefault-debug`，再加会变成两个 debug 字样
- **零 Java 改动**：BuildConfig / R 生成在 `namespace`（`app/build.gradle:105`）下，不是 applicationId 下，所以
  `resValue "manifest_package_id", "net.gsantner.markor"`（`:24`）→ `GsContextUtils.getAppIdUsedAtManifest()`（`:347`）
  反射 `net.gsantner.markor.BuildConfig` 照样命中。现成反证：flavorAtest 的 applicationId 早就是
  `net.gsantner.markor_test`，它的 BuildConfig 仍在 `net/gsantner/markor/`。Provider authority
  `${applicationId}.provider`（`AndroidManifest.xml:145`）自动跟着分裂
- 图标区分靠 buildType 资源覆盖层 `app/src/debug/res/`：`drawable/ic_launcher_background.xml`
  （自适应底色 #eaeaea→#a5d6a7、顶栏 #797c76→#2e7d32）+ 5 个密度的 `drawable-*/ic_launcher.png`（同一改色的位图版）。
  清单不用动（`android:icon="@drawable/ic_launcher"` 资源名未变）；API<26 读 PNG，API26+ 读自适应 XML，两条路都覆盖
- 实测（emulator-5554）：`pm list packages` 两包并存（两者 `versionName` 都是 2.16.1，靠包名+图标区分）；debug 包内
  `res/drawable-xxxhdpi-v4/ic_launcher.png` 与 `src/debug` 逐字节一致，release 包内仍是 `src/main` 那张；
  编译后的 debug `ic_launcher_background.xml` 含 #ffa5d6a7/#ff2e7d32、不含原灰值；应用抽屉里两个 Markor 并排
- 隔离：SharedPreferences、运行时权限、私有目录各自独立（debug 首装要重新授权、会走 Intro）；
  **笔记本目录共用** `Documents/markor`（`AppSettings.getDefaultNotebookFile()` 由 `app_name` 字符串推出，与包名无关）
  ⇒ 方便 A/B 对比渲染，但别两边同时编辑同一个文件。deep-link scheme `markor` 两包都注册，从别的 App 打开会弹选择框
- 别拿改 `app_name` 的方式区分标签——那会把 debug 的笔记本目录变成 `Documents/markor debug`

## 签名

- release 签名信息一律从环境变量读取：`KEY_STORE` / `KEY_ALIAS` / `KEY_PASSWORD` / `KEY_STORE_PASSWORD`
- 禁止硬编码密码、禁止回退默认值；缺变量则构建失败（`KEY_ALIAS=pisces312`）
- release signingConfig 显式开 `enableV1/V2/V3Signing`（AGP 按 minSdk 21 只会自动开 v1+v2）。
  实测 release 包三项均 true，签名者 `CN=pisces312, O=pisces312`
  （cert SHA-256 `abadebd2…53ee0b90`，SHA-1 `b40d0eba…36f48d99`）；debug 包仍是 `CN=Android Debug` 的 v1+v2。
  该结论在 AGP 8.13.2 与 AGP 9.4.0 下各实测一次，均 v1+v2+v3 = true、v4 = false（只是没生成 .idsig）

## 模拟器

- AVD `pixel6`（API 34，x86_64，`D:\dev\android_avd`）；启动必须用受管后台任务，普通后台启动会被回收
- `adb install -r` 装 APK；截图用 `adb exec-out screencap -p > out.png`
