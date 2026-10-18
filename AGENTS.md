# AGENTS.md — Markor fork 现状台账

> 本文件只记录当前状态（工具链 / 构建 / 产物 / 签名），深度调研与计划见 docs/（一个概念一份正文）。
> 更新规则：状态变了就改这里，不写历史沿革。

## 仓库

- 本地 `D:\3rd-party-projects\markor`；origin = `pisces312/markor`（fork），upstream = `gsantner/markor`
- `master` 与上游同步至 `aff9492c`（2026-10-10）
- 工作分支 `flexmark-upgrade`：上游 PR #2407 的 8 个提交 cherry-pick（flexmark 0.64.8 + minSdk 21 + androidx/commons-io）
- 文档指针：`docs/flexmark-upgrade-investigation.md`（升级交接单 + 两轮计划）、`docs/markdown-reader-survey-2026-10.md`（md 阅读器调研）

## 工具链（实测值）

| 项 | 值 |
|---|---|
| Gradle wrapper | 8.13（`gradle/wrapper/gradle-wrapper.properties`） |
| AGP | 8.13.2（`build.gradle:13` `version_gradle_tools`） |
| Kotlin 插件 | 1.3.72（`build.gradle:14`，`enable_plugin_kotlin` 控制是否 apply） |
| JDK | `D:\dev\AndroidStudio\jbr`（Java 21） |
| Android SDK | `D:\dev\android_sdk` |
| GRADLE_USER_HOME | `D:\dev\.gradle` |
| 依赖镜像 | `D:\dev\.gradle\init.d\mirror.gradle` → 阿里云 google/public/gradle-plugin，原仓库兜底 |

**网络坑（实测）**：本机直连 Maven Central/Google 会静默卡死（IPv6 `SYN_SENT` 悬挂、缓存零增长），
构建必须走上述镜像 init script；`github.com` web 直连 HTTPS 也超时，验证远端用 `gh api`。

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
- 产物：`app/build/outputs/apk/flavorDefault/debug/net.gsantner.markor-v163-2.16.1-flavorDefault-debug.apk`
- 单元测试：`./gradlew testFlavorDefaultDebugUnitTest`
- 日志读取坑：PowerShell `>` 重定向写出 UTF-16LE，先 `iconv -f UTF-16LE -t UTF-8` 再 grep
- PowerShell 调用坑：bash 里包 `-Command` 参数用单引号，否则 `$env` 被 bash 吞掉

## 签名

- release 签名信息一律从环境变量读取：`KEY_STORE` / `KEY_ALIAS` / `KEY_PASSWORD` / `KEY_STORE_PASSWORD`
- 禁止硬编码密码、禁止回退默认值；缺变量则构建失败（`KEY_ALIAS=pisces312`）

## 模拟器

- AVD `pixel6`（API 34，x86_64，`D:\dev\android_avd`）；启动必须用受管后台任务，普通后台启动会被回收
- `adb install -r` 装 APK；截图用 `adb exec-out screencap -p > out.png`
