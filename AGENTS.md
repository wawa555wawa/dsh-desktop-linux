# AGENTS.md

本仓库是 DeepSeek Harness 桌面端的 Linux 打包工程，不含应用源码：上游代码由
`scripts/fetch-upstream.sh` 取到 `upstream/`（不入库），这里只放补丁、PKGBUILD 和脚本。

## 硬规则

- 改代码或文档前先把方案给维护者过目；不要自行 push、打 tag 或改 GitHub release。
- [README.md](README.md) 只写重点、不写过程；与 [README.en.md](README.en.md) 必须同步修改。

## 发版流程

从跟上游到发版走固定的 14 站（按 `AGENTS.local.md` 走）。两条必须守住：

- **不跨站**：一站做完、判据过了才进下一站；「继续」只表示做完当前这一步。
- **tag 铁律**：打 tag 前必须空跑一次 CI，全绿再打；tag 绝不重打。

## 本仓库相对上游的本地修改（同步时必须保留）

本仓库是 [ffyfox/dsh-desktop-linux](https://github.com/ffyfox/dsh-desktop-linux) 的 fork：补丁系列
`0001`–`0018`、`scripts/`、CI 与 PKGBUILD 主体来自它。下面这些是**本仓库自己的修改**；同步上游
（把 `ffyfox/master` 合进来、重做补丁）时必须逐项保留/重做，不能当成上游旧内容删掉。

判断本地改动的办法：同步分支（基于 `ffyfox/master`）上 `git diff ffyfox/master..HEAD --stat`；
补丁层面看 `patches/` 里 `0001`–`0018` 之外的文件，以及 `0003` 里带「本 fork 增补」的段；仓库文件层面
看下表。

### 补丁

| 补丁 | 本地内容 | 同步时怎么保留 |
|---|---|---|
| `0003-electron-builder-linux-configuration.patch` | **增补**（不是新补丁）：deb/rpm 的 `fpm` 声明 `libdbusmenu-glib4`；接管 `afterInstall` 指向 `apps/desktop/scripts/linux-after-install.sh`（上游模板 + 清掉升级遗留的 `node_modules` 空目录）。补丁里带 `本 fork 增补` 注释 | 上游每轮重生 `0003` 后，把这几处重新叠回去（两个 `fpm` / `afterInstall` 段 + 新增脚本文件）；脚本内容以旧版补丁为准 |
| `0019-desktop-linux-frameless-caption.patch` | **新增**：普通模式主窗 `frame: false`、`dsh-desktop:window-controls` / `window-controls-state` 两个 IPC、preload 自绘最小化/最大化/关闭；玻璃模式保持原生边框 | 在「新基线 + `0001`–`0018`」之后重做；与 `0016`（caption 座位）、`0018`（玻璃模式）改同一批文件，必须排在它们之后 |
| `0020-desktop-linux-tray-glass-switch.patch` | **新增**：托盘「玻璃模式」勾选项 + 重启确认框，`settings.json` 的 `appearance.glass` 持久化，托盘选择优先于 `DSH_DESKTOP_GLASS` 环境变量；8 种语言字典同步 | 依赖 `0018`（`glass-mode.ts` 是 `0018` 新建的）与 `0019`（同改 `main.ts`），必须排在它们之后 |
| `0021-desktop-linux-official-branding.patch` | **新增**：Linux 的 unsigned 打包改用 official client profile（`build --profile official`，pack-local 按同一 profile 校验），让左上角显示官方 DSH 标识而不是「DSH 本地构建」徽标；macOS/Windows 的 unsigned 仍用 local | 依赖 `0002` 改过的 `package-target.ts`；排在最后生成。只改打包脚本与测试，不碰客户端源码 |

编号约定：上游已作废 `0014` / `0015` 且空号不复用；本地新增编号从 `0019` 起、只增不减。上游新版本若
占用了同号，把本地补丁整体顺延，并在 `PKGBUILD`、`patches/`、`scripts/verify.sh`、本文档里同步改号。

### 非补丁文件

| 文件 | 本地修改 | 同步时怎么保留 |
|---|---|---|
| [dsh-desktop-linux.install](dsh-desktop-linux.install) | `_prune_empty_node_modules` 及其 `post_install` / `post_upgrade` 调用（与 `0003` 的 afterInstall 同因） | 上游文件若变化，重新挂上这个函数 |
| [README.md](README.md) / [README.en.md](README.en.md) | 「这个 fork 与原项目的区别」章节、无边框标题栏说明、玻璃模式与托盘开关、已知限制（NAN 原生模块、窗口按钮英文 tooltip） | 上游 README 每轮更新后把这几段合回去，中英必须同步 |
| [patches/README.md](patches/README.md) | `0019` / `0020` / `0021` 章节、例外说明（`0017`–`0021` 依赖前置补丁）、本 fork 的验证记录 | 以上游版本为底稿再追加本地章节，不要删上游验证记录 |
| [scripts/verify.sh](scripts/verify.sh) | `PATCH_COVERAGE` 里 `0019` / `0020` / `0021` 的登记，以及产物 bundle 断言、0021 的活体品牌断言 | 上游脚本更新后把覆盖登记与这些断言补回去；**新增补丁必须在 `PATCH_COVERAGE` 登记**，否则 `verify.sh` 直接 FAIL |
| [docs/screenshot.png](docs/screenshot.png) | 本 fork 实际运行画面（无边框标题栏 + 官方品牌） | 可重拍，不要换回上游头图 |
| [PKGBUILD](PKGBUILD) | `source` / `sha256sums` 覆盖本地三个补丁（当前 19 个补丁 + 托盘 PNG + 源码包 = 21 条校验和） | 按 `source` 顺序重算 sha256；`pkgbuild/` 用 `scripts/pkgbuild-dir.sh` 重新摊平（不入库） |
| [AGENTS.md](AGENTS.md) 本文件 | 这一节 | 上游 AGENTS.md 每轮会覆盖，同步后把这一节补回来 |

### 不属于本地修改、不要删的东西

- 左侧栏底部的「更多」入口：来自上游 `packages/client/ui-entry-customization`（`more` 菜单、插件浏览 /
  创建草稿），0.2.2 基线树里就有，不是 fork 作者加的。
- `assets/tray-linux.png`、`patches/0013` 的托盘逻辑、`0016` caption、`0017` Host resources、`0018` 玻璃
  模式：都是 ffyfox 上游的补丁，本仓库原样继承。
- 官方品牌的注册代码（`packages/client/ui-brand-official`）：上游代码，`0021` 只是让 Linux 的 unsigned
  构建用上 official profile，不修改这段源码。

### 同步上游的固定流程

> **注（0.2.2-rc.1）**：本轮没有采用 ffyfox 的 `v0.2.2rc1`，而是把本仓库现有 19 条补丁在 alpha.2 上
> 按序重放后 rebase 到上游 `66491b05`；下面第 1–2 步的 ffyfox 基线法不适用于这一轮。

1. 从 `ffyfox/master` 开 `sync/<tag>` 分支，不直接改 `master`；先 `git fetch` 再对比。
2. 采用上游新的 `patches/`、`PKGBUILD`、`scripts/`、`README` 为基线；`0015` 保持作废，空号不复用。
3. 按顺序重做本地补丁：`0003` 增补（基于上游重生版）→ `0019` → `0020` → `0021`。生成方式：在
   `upstream/` 里先 `apply-patches.sh` 得到「新基线 + 前置补丁」状态，再在该状态上取隔离 `git diff`；
   不要对着旧基线生成，也不要把前置补丁的 hunk 带进来。
4. 更新 `PKGBUILD` 的 `source` 与 `sha256sums`（源码包 + 全部补丁 + `tray-linux.png`），跑
   `scripts/pkgbuild-dir.sh` 重新摊平。
5. 逐项核对上面三张表，把非补丁文件改动补回去（尤其 `dsh-desktop-linux.install`、`verify.sh`、
   中英 README）。
6. 验证门（缺一不可）：两条打补丁路径（`git apply` / GNU `patch -Np1`）干净且树零差异；
   `pnpm install --frozen-lockfile`；`build.ts --scope typecheck`；`apps/desktop` 单测；
   `build.sh --all`（三件套 + payload 冒烟）；`verify.sh` 与 `verify.sh --runtime`。
7. 上游对 `engines` / pnpm 版本 / CI Node 版本的调整照上游走，本地不另钉版本。

## 回家读

- 补丁系列的编号、每块在解决什么、有哪些约束：[patches/README.md](patches/README.md)
- 用户可见行为、支持矩阵、构建方式：[README.md](README.md)

## 动手前先知道

- 每个补丁都是相对 [PKGBUILD](PKGBUILD) 里 `_tag` 那个 commit 的 diff；改 `_tag` 就要同步重做补丁，
  否则 `scripts/apply-patches.sh` 直接失败。
- [pkgbuild/](pkgbuild/) 是 `scripts/pkgbuild-dir.sh` 生成的可直接 `makepkg` 的摊平目录
  （makepkg 只在同目录按 basename 找本地 source），不要手改；改了根 PKGBUILD 就重新生成。
- 构建与验证都走 `scripts/`：`fetch-upstream.sh`、`apply-patches.sh`、`build.sh`、`verify.sh`。
  隔离数据目录用 `DSH_DESKTOP_LINUX_HOME`（默认 `/tmp/dsh-desktop-test`），脚本刻意不继承 `DSH_HOME`。
- 结束应用用 `pkill -f '[d]eepseek-harness$'`；`pkill -x deepseek-harness` 打不中（comm 被截成 `deepseek-harnes`）。
- 左侧栏的「更多」入口来自上游 `packages/client/ui-entry-customization`，不是本 fork 加的，别为去品牌化删它；
  左上角的「DSH 本地构建」徽标才是需要处理的（由 `0021` 改成官方品牌）。
