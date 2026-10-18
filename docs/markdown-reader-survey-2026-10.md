# 开源 Android Markdown 阅读器调研（2026-10-18）

> 调研日期 2026-10-18。数据来源：GitHub 仓库元数据（gh api，星数为当日快照）与各仓库 README。
> 凡未安装/未跑过的结论都标「未实测」。
> 背景：Markor 的 flexmark 升级见 [flexmark-upgrade-investigation.md](flexmark-upgrade-investigation.md)；
> M2Git 通过 `ACTION_VIEW text/markdown` 把 `.md` 递给外部阅读器打开，故「Open with 集成」是本次关注点之一。

## 1. 结论速览

- **没有出现能全面取代 Markor 的成熟品**：Markor 仍是功能面最全（编辑 + 多格式 + todo.txt/Zim）的开源 Android 选项。
- 2025-2026 出现一批小而专注的新应用，两类值得注意：
  - **纯阅读方向**：gander（万能离线查看器，零权限）——适合作为 M2Git 递文件的备选目标；
  - **渲染路线革新**：pilcrow（Compose 原生渲染，不用 WebView）——与 Markor 的 flexmark→HTML→WebView 路线根本不同。
- 对本 fork 的意义：升级 flexmark 只优化「md→HTML 正确性」这一层；若追求观感/性能突破，可另评估原生渲染路线（pilcrow 已证明可行）。

## 2. 候选清单（按星数排序，均为开源）

| 项目 | 星数 | 创建 | 语言/许可 | 定位 | 备注（未实测） |
|---|---|---|---|---|---|
| [gander](https://github.com/mokshablr/gander) | ⭐1273 | 2026-07 | Kotlin / MIT | 万能离线文件查看器（PDF/Office/md/代码/3D/音视频） | 零权限、5MB APK、无网络、支持分享/Open with 入口、zip 内文件直读 |
| [Kori](https://github.com/YangDai2003/Kori) | ⭐396 | 2025-04 | Compose MP | AI 加持的 md 记事本 | 跨平台（Compose Multiplatform） |
| [marktext-android](https://github.com/Renakoni/marktext-android) | ⭐140 | 2026-06 | — | 安静专注的 md 编辑器 | 中文社区出品 |
| [markleaf-android](https://github.com/jeiel85/markleaf-android) | ⭐125 | 2026-04 | — | 轻量 local-first md 笔记 | — |
| [OpenNote-Next](https://github.com/muxia0396/OpenNote-Next) | ⭐61 | 2026-08 | — | 本地 md/TXT/HTML 阅读编辑 | 中文 |
| [pilcrow](https://github.com/pilcrowmd/pilcrow) | ⭐55 | 2026-06 | Kotlin / GPL-3.0 | md 阅读器+编辑器 | **Compose 原生渲染、无 WebView**；已上 Google Play + F-Droid；Android 8.0+ |
| [mdview](https://github.com/mdview2026/mdview) | ⭐40 | 2026-08 | — | 轻量 md 查看器 | Win/mac/Android 多平台 |
| [mark-reader](https://github.com/usamaiqb/mark-reader) | ⭐40 | 2026-02 | — | 离线 md/代码阅读器 | 语法高亮 |

其他同期项目（相关性低，仅记录）：simple-notes-sync（⭐203，WebDAV 同步笔记）、obsidian-mobile-companion（⭐57，GitHub vault + Obsidian 渲染）、todosian-app（⭐45，Obsidian Tasks 集成 todo）、notes-widget-for-markdown（⭐28，桌面小组件）。

## 3. M2Git 场景关注点

M2Git 需求是「把 .md 用外部 app 打开」，候选里：

- **gander**：明确支持 share sheet / "Open with" 集成、零权限、离线，最贴近纯阅读目标（未实测 Intent 过滤器细节）。
- **Markor**：仍是编辑需求的首选，且本 fork 正升级其 flexmark 渲染。
- **pilcrow**：渲染质量取向，但它主打自己的 app 内阅读，未实测是否注册 `.md` 的 `ACTION_VIEW`。

## 4. 其它调研（与 md 阅读器同日）

### 4.1 上游 flexmark 升级 PR 已存在

上游 [gsantner/markor#2407](https://github.com/gsantner/markor/pull/2407)（guanglinn，2024-08 开，open）做了与本 fork 计划相同的升级：flexmark `0.42.14→0.64.8`，另有 minSdk 18→21 和 androidx 全家升级。作者实测可构建可运行；gsantner 2026-01-31 表态「下次更新后合并」，截至 2026-10-18 未合并。该 PR 修正了 flexmark-upgrade-investigation.md §2/§3 的两处误判，详见该文件「勘误」节。

### 4.2 各 fork 升级现状

GitHub 代码搜索（`version_library_flexmark`）：除上游外未发现任何 fork 升级过 flexmark（luckycloud-Android 等派生项目也停在 0.42.14）。升级诉求长期存在，早期 bump 尝试（PR #627/#779/#1272）均被关闭，卡点在 minSdk 提升。

## 5. 下一步

1. （进行中）本 fork `flexmark-upgrade` 分支已 cherry-pick PR #2407 的 8 个提交，待构建 + 渲染回归（见 flexmark-upgrade-investigation.md §5 清单）。
2. 若后续要为 M2Git 增加备选 md 阅读目标，实测 gander 的 `ACTION_VIEW text/markdown` 支持。
