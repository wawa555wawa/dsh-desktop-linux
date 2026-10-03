# 补丁

按文件名顺序应用（见 `scripts/apply-patches.sh`），全部相对上游 `639ed01`（tag `dsh-v0.2.0-rc.2`）。

组织约定：**一个文件只属于一个补丁**。每个补丁都是相对同一个基线的独立 diff，
互不重叠，因此应用顺序无关（仍按编号执行）。补丁由 `git diff -- <files>` 从开发工作树生成；
两个补丁共用一个文件时（`electron-builder-config.mjs` 属 0003 与 0013，`src/main.ts` 属 0008、0013 与 0016），
必须按「基线 + 只有这一个补丁」的隔离树取 diff，否则会把另一个补丁的 hunk 一起带进来。

已验证（rc.2）：16 个补丁按序打在完整的上游 release 源码包（tag `dsh-v0.2.0-rc.2`）上，
`patch -Np1` 与 `git apply` 都干净通过，51 个触及文件；在打好全部补丁的树上逐个反打也全部干净
（`0016` 与 `0008` / `0013` 共用 `src/main.ts`，`0003` 与 `0013` 共用 `electron-builder-config.mjs`，
反打干净即各自的 hunk 互不重叠）。`pnpm install --frozen-lockfile` 与 `pnpm run build:official`
在同一棵树上通过，`build.sh` 出得来 linux-unpacked 与 rpm / deb，`verify.sh` 静态矩阵
**18 通过 / 0 失败 / 3 跳过**——其中包含针对 `0016` 的两条产物断言（主进程的去边框与窗口控制通道、
preload 的 caption 模块）。`0003` 新加的依赖声明实测进了包：rpm 的 `Requires` 里是
`(libdbusmenu or libdbusmenu-glib4)`、上游那 8 条默认依赖一条不少，deb 的 `Depends` 末尾是
`libdbusmenu-glib4`、上游那 9 条默认依赖一条不少（`fpm` 追加这条路为什么能用，见「注意」）。
`makepkg` 这一轮没跑（`pkgrel` 现为 2，上游 tag 未变）；更早那轮 15 补丁的验证里 `makepkg` 是
整包构建成功的（`dsh-desktop-linux-0.2.0rc2-1`，产物里 `resources/runtime/cli` 不存在，正是 0004
的 Linux 闸门在起作用）。
rc.1 → rc.2 只有 6 个文件变过（`pnpm-lock.yaml`、`src/main.ts`、`desktop-upload-plan.ts`、
`prepare-runtime.ts`、`prepare-dsh.ts`、`apps/desktop/package.json`），对应
0002 / 0004 / 0006 / 0008 / 0013 / 0014 六个补丁重生，其余 9 个逐字节未动。

## 让 Linux 成为受支持的 target（0001–0007）

| 补丁 | 覆盖文件 | 内容 |
|---|---|---|
| `0001-desktop-target-model-add-linux-x64.patch` | `desktop-build-paths.{mjs,d.mts}`、`desktop-auto-update-environment.{mjs,d.mts}` | target 白名单加 `linux-x64`；`desktopTargetPlatform` 返回 `'linux'`；新增 `desktopElectronExecutablePath()` |
| `0002-package-target-add-linux-x64.patch` | `package-target.ts`、`desktop-upload-plan.ts` | 打包目标表加 `linux-x64`；Linux 构建主机校验；放宽 `--unsigned` |
| `0003-electron-builder-linux-configuration.patch` | `electron-builder-config.mjs` | 允许 unsigned 的 Linux 构建；Linux 关闭 asar；显式 `executableName`；Linux 不嵌入强制更新策略；Linux 的打包格式与包元数据来自发布设置（见 `0011`）；显式钉住 deb/rpm 的 `packageName` / `packageCategory` 与 `linux.synopsis`；deb/rpm 声明托盘菜单依赖的 `libdbusmenu-glib.so.4`（走 fpm 追加，见「注意」）；用 `appImage.executableArgs: []` 去掉 legacy 工具集写死的 `--no-sandbox`；Linux 图标显式指定为 SVG —— 单个 PNG 文件会被 electron-builder **原样按自身像素尺寸**装进 `hicolor/1024x1024/apps`，而多个发行版的 `hicolor/index.theme` 并不声明该目录（Arch 就没有），图标会解析不到；SVG 落到 `hicolor/scalable/apps`，所有发行版都声明 |
| `0004-prepare-target-electron-distribution.patch` | `prepare-dsh.ts`、`prepare-runtime.ts` | Electron 分发路径按 target 推导，不再假设「非 mac 即 win32」；打包期 `pnpm install` / 运行时冒烟改用 Host 运行时；`versions.json.node` 记为 payload 实际运行的 Node 版本；Linux 上整块跳过 rc.2 新增的 `prepare:cli`（见「注意」） |
| `0005-desktop-linux-release-settings.patch` | `desktop-package-environment.{mjs,d.mts}`、`desktop-toolchain-preflight.ts`、`.gitignore`、`.env.linux.example`、`tests/desktop-package-environment.spec.ts` | 支持 `.env.linux`；Linux 不套用 Windows/macOS 专属设置；Linux 不要求策略 origin；修掉 `win32 ? … : macOS` 的隐含假设；Linux 文件白名单加 `MAINTAINER`/`HOMEPAGE`（并从环境里剥掉，保证发布设置只由文件拥有）；打了 rpm 才预检 `rpmbuild` |
| `0006-desktop-package-linux-scripts.patch` | `apps/desktop/package.json` | `package:linux:x64` / `package:linux:x64:dir` |
| `0007-tests-linux-x64-supported.patch` | 3 个 `tests/*.spec.ts` | 把「断言 Linux 抛错」改成「断言 Linux 受支持」 |

## 让 Host 跑在真 Node 上（0008–0010）

| 补丁 | 覆盖文件 | 内容 |
|---|---|---|
| `0008-desktop-host-runtime-standalone-node.patch` | `src/node-environment.ts`、`src/host-process.ts`、`src/main.ts`、`desktop-host/src/index.ts`、`scripts/node-bin/node`、4 个 spec | 引入 `DesktopNodeRuntime`；Linux 上 Host 走 primary-runtime 的真 Node；`ELECTRON_RUN_AS_NODE` 只在真 Electron 运行时下设置 |
| `0009-desktop-packaging-host-runtime.patch` | `scripts/dev.ts`、`scripts/smoke-{runtime,prepared-runtime,packaged-runtime}.ts`、`scripts/sign-primary-runtime.ts`、`tests/fixtures/runtime-payload-smoke.mjs`、`tests/prepared-runtime-smoke.spec.ts` | 把 Host 运行时贯穿 dev / 打包 / 冒烟；payload smoke 的 Electron 专属断言改为按平台判断 |
| `0010-desktop-linux-policy-opt-out.patch` | `desktop-policy-environment.{mjs,d.mts}`、`tests/desktop-policy-environment.spec.ts` | 新增 `desktopPlatformEmbedsPolicy()`：策略服务只认 `desktop-win` / `desktop-mac`，Linux 不参与 |

## 产物与包元数据（0011–0012）

| 补丁 | 覆盖文件 | 内容 |
|---|---|---|
| `0011-desktop-linux-package-metadata.patch` | `desktop-linux-packages.{mjs,d.mts}`、`tests/desktop-linux-packages.spec.ts` | 新增 `resolveDesktopLinuxFormats()`（`DSH_DESKTOP_TARGET_FORMATS`，缺省只出 AppImage）与 `resolveDesktopLinuxPackageMetadata()`（deb/rpm 必须给 `Name <email>` 形式的 maintainer 和绝对 http(s) homepage，两者一起报错） |
| `0012-desktop-build-commit-release-archive.patch` | `desktop-build-commit.mjs`、`tests/desktop-build-commit.spec.ts` | `readDesktopBuildCommit()` 在目录不是 git checkout 时读 `DSH_DESKTOP_BUILD_COMMIT` / `_DIRTY`，两者都没有才报错。上游从 checkout 构建，distro 打包从 release 源码包构建，后者没有 `.git` |

`0003` 消费 `0011`，`0005` 也消费 `0011`：格式选择和包元数据的**规则**只有一份（`0011`），
`0003` 拿去配 electron-builder，`0005` 拿去做打包最前面的预检。按「一个文件只属于一个补丁」，
规则本身单独成一个补丁。

## 让 Linux 也有托盘（0013–0014）

| 补丁 | 覆盖文件 | 内容 |
|---|---|---|
| `0013-desktop-linux-tray.patch` | `src/main.ts`、`src/tray.ts`、`src/background-notice.ts`、`scripts/electron-builder-config.mjs`、`scripts/render-tray-icon.ts` | 托盘的创建条件从只认 `win32` 放宽到 `win32 \|\| linux`；Linux 的托盘图是 `resources/tray-linux.png`（单张 PNG，托盘宿主自己缩放到面板；**用应用图标自身的留白，不做 Windows 那 20% 放大**，见「注意」）；首次关窗的一次性提示同样覆盖 Linux —— 它的文案本来就是「可在系统托盘中重新打开窗口」，在 Linux 上这句话只有有了托盘才成立；渲染器除 ICO 之外也输出那张 PNG；electron-builder 的 Linux `extraResources` 把它带进产物的 `resources/` |
| `0014-electron-version-tray-fix.patch` | `pnpm-lock.yaml` | 把 lockfile 里 electron 的解析从 `44.0.0` 提到 `44.4.5`。上游 `apps/desktop/package.json` 本来就写 `^44.0.0`（caret 就允许 44.4.5），只是 lockfile 把解析钉在 44.0.0 —— 而那个版本的 Linux 托盘在 KDE 与 GNOME 下都注册不上（上游回归，见「注意」）。改动只有 4 行：importer 的 `version`、`packages` 段的版本名与 integrity、`snapshots` 段的版本名；两个版本的依赖范围逐字相同，所以传递依赖一行都不用动 |

托盘 PNG 是二进制，`patches/` 装不下：makepkg 用的是 GNU patch，它不支持 git 的二进制补丁。
所以这张图作为普通本地 `source` 走，仓库里存在 `assets/tray-linux.png`，由两条路径各自放进
`apps/desktop/resources/`：PKGBUILD 的 `prepare()`（`makepkg` 路线）和 `scripts/apply-patches.sh`
（CI 与本地 `build.sh` 路线）。刷新它 = 在 `apps/desktop` 里跑 `pnpm run render:tray-icon`，
再把输出的 `resources/tray-linux.png` 拷进 `assets/`（图和 Windows 托盘同源，都是
`resources/icon-windows.svg`）。

## 让关窗提示在 Wayland 下真的显示出来（0015）

| 补丁 | 覆盖文件 | 内容 |
|---|---|---|
| `0015-desktop-linux-hidden-overlay-reveal.patch` | `src/update-overlay.ts` | 提示浮层是 `transparent: true` + `show: false` 的子窗口，只在 `ready-to-show` 里 `show()`；而这种窗口在 Wayland 下不报告首帧，于是浮层建好了、内容也渲染对了，却始终 `isVisible() === false` —— 用户既看不到也点不到，关窗看起来像卡住。Linux 上再用 `did-finish-load` 兜一次 `reveal()`；Windows/macOS 的 `ready-to-show` 行为一字未动 |

`0015` 是 `0013` 的下游：Linux 的首次关窗提示是 `0013` 打开的，而它用的正是这个浮层。

## 让 Linux 也有自绘标题栏（0016）

| 补丁 | 覆盖文件 | 内容 |
|---|---|---|
| `0016-desktop-linux-caption.patch` | `src/main.ts`、`src/ipc.ts`、`src/preload-app.ts`、`src/preload-linux-caption.ts` | Linux 主窗去掉 OS 边框（`frame: false`）；新的 preload 模块发布 Windows caption 用的同一个 `data-windows-titlebar` 座位（40 DIP 条、拖拽带、侧栏与浮层几何交给共享样式），并在右侧补上自绘的最小化 / 最大化 / 关闭；主进程加 `dsh-desktop:window-controls`（收按钮动作）与 `dsh-desktop:window-controls-state`（把最大化状态回推渲染进程）两个通道 |

复用 Windows 的座位而不是另起一套，是因为共享布局只认这一个标记：`ui-layout` 的 `AppFrame.module.css`
与 `AppFrame.tsx`、`ui-dockkit`、`ui-sidebar`、`ui-sidebar-right`、`ui-settings-account` 都按
`html[data-windows-titlebar]` 留白、铺拖拽带、避让浮层，Web UI 一行都不用改。

**兄弟项目里那两行「隐藏原生菜单条」没有移植过来**（`autoHideMenuBar: true` 与
`window.setMenuBarVisibility(false)`）：Electron 44.4.5 的 `RootView::SetMenu` 在
`!window_->has_frame()` 时直接 return，注释就写着 *"Do not show menu bar in frameless window"* ——
无边框窗口根本不会创建菜单条，`SetMenuBarVisibility` 只操作那个不存在的 `menu_bar_`，
`HandleKeyEvent` 也在 `!menu_bar_` 时返回，所以 Alt 同样唤不出来。两行都是死代码。

## 注意

- **dev 模式也需要 `0001`**——`dev.ts` 虽然不走 `package-target.ts`，
  但会调用 `resolveDesktopBuildTarget()`，Linux 上抛 `unsupported target linux-x64`。
- 上游有手写的 `.d.mts` 声明文件，改 `.mjs` 的 JSDoc **不够**，类型联合必须同步改 `.d.mts`，
  否则 `tsc` 报错。
- **`DSH_DESKTOP_NODE_ELECTRON` 的缺省值必须保持「Electron」**。调用方清空环境时不会带上
  这个变量，缺省若回落到 standalone，macOS / Windows 的包脚本会以 GUI 模式起 Electron。
- `0003` 与 `0005` 都涉及策略：`0003` 决定「产物里嵌不嵌」，`0005` 决定「打包期要不要 origin」。
  一个文件只属于一个补丁，所以这两半分在两处。
- **`DSH_DESKTOP_TARGET_FORMATS` 刻意不进 `.env.linux` 白名单**。它是「这次构建打什么」的
  选择器（和 `DSH_DESKTOP_TARGET_PLATFORM` / `_ARCH` 同类），不是发布设置；进了文件反而会让
  `build.sh --deb` 被文件里的值盖掉。
- **`0012` 是唯一为「非 git 源码」存在的补丁**。上游两条路都要 git：`scripts/build.ts` 经
  `repositoryCommitHash()` 读 `HEAD`（有 `DSH_CLIENT_COMMIT_HASH` 环境变量出口，不用改），
  `package-target.ts:364` 无条件调 `readDesktopBuildCommit()`（没有出口，所以要 `0012`）。
  PKGBUILD 从 release 源码包构建，所以两个变量都由它显式给出，`_DIRTY=1` 也是事实——这个
  构建确实打了补丁。
- **rc.2 新增的 `prepare:cli` 在 Linux 上整块跳过（`0004`）。** `prepareDesktopCli(dir, platform)` 的
  形参是 `'darwin' | 'win32'`（不含 `linux`），它拷的 `apps/desktop/cli/dsh` 是 macOS 脚本
  （内部 exec `$resources/../MacOS/DeepSeek Harness`），`link-entry` 要 clang 编译，
  `command-manager-entry.js` 走 `/usr/local/bin/dsh` 与 PowerShell；而 rc.2 里触发这套东西的菜单项
  本身就 gate 在 `darwin` / `win32`。Linux 上没有入口，硬跑还会把 macOS 启动器塞进产物，
  所以这一段包进 `if (platform !== 'linux')`，Linux 不产出 `runtime/cli`。
- **`0015` 的根因是量出来的，不是猜的。** 用 `--remote-debugging-port=9222 --inspect=9229` 起应用，
  从主进程读 `BrowserWindow.getAllWindows()`：关窗后浮层窗口存在、`modal: true`、内容与按钮位置都对，
  但 `isVisible()` 是 `false`；手动 `show()` 一次即正常显示且可用鼠标点。Debian 13（GNOME 48.7）与
  Ubuntu 26.04（GNOME 50.1）都如此，打上 `0015` 后两边都变成 `true`，鼠标点 Confirm 能写入 marker 并隐藏窗口。
  整个桌面端只有这一个窗口依赖 `ready-to-show`，所以只修这一处。
- **托盘的依赖问题全在 ELF 之外。** 实测（Electron 44 二进制）：Linux 托盘走进程内的
  StatusNotifierItem（二进制里有 `StatusIconLinuxDbus`、`org.kde.StatusNotifierWatcher`），
  全库没有 `appindicator` 字样，`ldd` / `NEEDED` 里也没有，所以**不需要** libappindicator。
  但托盘菜单要 `libdbusmenu-glib.so.4`——那个库名和 `dbusmenu_*` 符号名都在二进制里，
  却不在 `NEEDED` 里（1519 个未定义动态符号里没有它），即 dlopen。缺了它图标照出、菜单是空的，
  等于没有退出入口，而 `ldd` 和 namcap 都看不见，所以 Arch 包由 PKGBUILD 显式写进 `depends`，
  deb/rpm 由 `0003` 声明。**deb/rpm 这条不能走 electron-builder 的 `depends`**：`FpmTarget`
  只在没有 `depends` 时才回落到 `getDefaultDepends()`，给了它就整份替换掉上游那 9 / 8 条默认依赖，
  等于把默认表复制进本仓库、以后上游加一条我们不会知道。所以走 `fpm: ['-d', …]` 追加：
  `FpmTarget` 把 `options.fpm` 原样塞进 fpm 参数，默认依赖表不受影响。包名两边不同——
  Debian 是 `libdbusmenu-glib4`，Fedora 是 `libdbusmenu`（openSUSE 又是前者），RPM 侧用默认表
  已经在用的 rich dependency 语法写成 `(libdbusmenu or libdbusmenu-glib4)`。
- **Linux 上可靠的是托盘菜单，不是单击图标。** 实测把 SNI 的 `Activate` 调过去，Electron 的
  `tray.on('click')` 没有触发（StatusNotifierItem 宿主有权把左键用来弹菜单）。`DesktopTray`
  保留 click 处理是给 Windows 的；Linux 上「打开」走右键菜单，`verify.sh --runtime` 也是按
  菜单里的条目来断言。
- **托盘美术在 Linux 上刻意与 Windows 不同，Windows 那份一个字都没动。** 上游的 `tray-glyph`
  组只含鲸鱼（底在组外），渲染 Windows ICO 时会被绕方块中心放大 20% —— 因为 Windows 托盘逻辑
  尺寸是 16px，不放大就看不清。Linux 不同：托盘宿主按面板尺寸自己绘制（KDE 是 22px），1.2× 在
  那里会显得鲸鱼顶满方块。所以 `renderLinuxTrayIcon()` 按原比例渲（保留应用图标自身的留白，
  与启动器/任务栏图标观感一致），底图仍然保留，深浅面板都还有对比度。两边同源（都用
  `resources/icon-windows.svg`），改动只在渲染参数上。
- **`0016` 之后 Linux 既没有原生菜单条，也没有 caption 菜单。** 上游的 caption 菜单（Application /
  Edit）由 preload 的 `installWindowsMenu()` 画，而它只在 `syncWindowsAppearance()` 里被调用，后者的
  第一行是 `if (process.platform !== 'win32') return`。要给 Linux 挂上就得放宽 `main.ts` 里
  `if (process.platform === 'win32')` 那个块，而那块里同时有 `windowsAppearance` 处理器——它的
  `setTitleBarOverlay()` 在 Linux 上会抛：`BaseWindow::SetTitleBarOverlay` 在 `IS_WIN || IS_LINUX`
  下都暴露，但 WCO 未启用时执行 `args->ThrowTypeError('Titlebar overlay is not enabled')`，而 WCO
  要求 `title_bar_style() == kHidden && titlebar_overlay_`，`frame: false` 的 Linux 窗两者都没有。
  所以这两件事一起被排除，代价是**「关于」「检查更新」在界面上没有入口**（退出仍有托盘菜单与
  `Ctrl+Q`），已记进 README 的已知限制。菜单的加速器不受影响：`RootView::RegisterAcceleratorsWithFocusManager`
  在 `has_frame()` 判断之前就执行了。
- **标题栏三个按钮的 `title` / `aria-label` 是英文。** Windows 的 caption 按钮由系统绘制、本地化由
  系统提供；Linux 这份是自绘的，而 shell 的文案表（`src/locale.ts`）里没有窗口控制这一组词，
  加词要同时改 en / zh 两组文案，超出了这个补丁的范围。
- **`dsh-desktop:window-controls` 在主进程里校验三件事**：发送者是主窗、发送 frame 是主 frame、URL
  前缀是 `dsh-app://app/`，任一不满足就静默丢弃（与同处的 `windowsAppearance` 处理器同款；那条是
  `ipcMain.on` 而不是 `handle`，抛出去就是主进程未捕获异常）。反方向的状态只回推一个布尔：Linux 上
  只有主进程知道窗口是否最大化，所以由 `maximize` / `unmaximize` / `did-finish-load` 三处发
  `window-controls-state`，渲染进程据此把最大化按钮换成还原图形。
