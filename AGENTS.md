# AGENTS.md — Markor fork 现状台账

> 本文件只记录当前状态（工具链 / 构建 / 产物 / 签名），深度调研与计划见 docs/（一个概念一份正文）。
> 更新规则：状态变了就改这里，不写历史沿革。

## 仓库

- 本地 `D:\3rd-party-projects\markor`；origin = `pisces312/markor`（fork），upstream = `gsantner/markor`
- `master` 与上游同步至 `aff9492c`（2026-10-10）
- 工作分支 `flexmark-upgrade`：上游 PR #2407 的 8 个提交 cherry-pick（flexmark 0.64.8 + minSdk 21 + androidx/commons-io）
- 工作分支 `toolchain-upgrade`（基于 flexmark-upgrade）：Gradle wrapper 9.6.1 + AGP 9.4.0，
  前置改动是把 15 个文件里 20 处 `switch (R.*)` 改成 if/else-if 链（AGP 9 的 R 字段非 final，`case R.*` 不再是常量表达式）
- 文档指针：`docs/flexmark-upgrade-investigation.md`（升级交接单 + 两轮计划）、`docs/markdown-reader-survey-2026-10.md`（md 阅读器调研）

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
| 依赖镜像 | `D:\dev\.gradle\init.d\mirror.gradle` → 阿里云 google/public/gradle-plugin，原仓库兜底 |

**网络坑（实测）**：本机直连 Maven Central/Google 会静默卡死（IPv6 `SYN_SENT` 悬挂、缓存零增长），
构建必须走上述镜像 init script；`github.com` web 直连 HTTPS 也超时，验证远端用 `gh api`。
连不上 `maven.google.com` 时 Gradle 当作致命错误而不是"未命中"，所以只在 jitpack 上的 `com.github.*`
必须排在它前面（`build.gradle:43-47`），否则依赖解析整体失败。

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

- 版本：`versionName 2.16.1` / `versionCode 163`（`app/build.gradle:26-27`）
- debug 产物：`app/build/outputs/apk/flavorDefault/debug/net.gsantner.markor-v163-2.16.1-flavorDefault-debug.apk`
  （文件名由 `app/build.gradle` 末尾 `androidComponents` 重命名块生成，不再依赖已删除的 `archivesBaseName`；
  文件名里的 `net.gsantner.markor` 只是前缀，debug 实际包名是 `net.gsantner.markor.debug`，见下）
- release 产物（AGP 9 + Java 17 + optimize 规则实测）：同目录 `...-flavorDefault-release.apk` **11525704 字节**
  （体积随 `GITHASH`/`GITMSG` BuildConfig 字段小幅波动，别拿字节数当回归判据），
  R8 minify 46s 完成，dex 从 debug 的 18 个合成 2 个；唯一 R8 告警是 flexmark 引用 `java.awt.*`（桌面端代码路径，可忽略）
- 单元测试：`./gradlew testFlavorDefaultDebugUnitTest`
  基线：178 跑 / **11 失败**，全部在 `WikitextLinkResolverTests`（链接串里的 `:` 在 Windows 上是非法路径字符）
  与 `WikitextFileTests`（用例假定本机时区 ±1，本机 +8）。已在 HEAD 基线工作区实测，失败集合与升级后逐条一致，属既有环境失败，不是回归
- R 常量（AGP 9）：`R.id.*` / `R.string.*` 是非 final 字段，**不能出现在 `case` 标签里**；全仓 20 处
  `switch`（含 `FormatRegistry` 那类 `static final int FORMAT_* = R.string.…` 间接别名）已改为 if/else-if 链。
  新增按钮分发一律别再用 `switch (R.*)`；`android.R.id.home` 等框架 R 仍是常量，可以进 `case`
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
