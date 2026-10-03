# dsh-desktop-tray-restart

给 DeepSeek Harness 桌面壳的**托盘右键菜单**加一项 **「重启 DeepSeek Harness」**（英文界面为 "Restart DeepSeek Harness"）。

## 为什么必须改壳，插件做不到

托盘在桌面壳的**主进程**里（`new Tray` / `setContextMenu`）。插件只有两个落脚点——宿主机（纯 Node，**没有 Electron API**）和页面渲染进程（preload 只暴露固定的 `dshDesktop`：`browser`、`deviceInfo`、`shortcuts`、`updates`、`closeWindow`）——**都无法向托盘菜单添加任何一项**。所以这是一个壳侧改动，本工具把它在**已安装的** `resources/app.asar` 里**本地、可回退**地做掉。

补丁内容（`lib/main.js` 两处）：

- 托盘菜单里、在「退出 DeepSeek Harness」上面插入一项 **「重启 DeepSeek Harness」**（与「打开 DeepSeek Harness」同一命名模式，跟随语言切换）；
- 它调用壳**自己**的代码 `app.relaunch()` + `quitWithoutConfirmation()`（壳在应用菜单里用的就是这段），因此**托盘图标由 Electron 自己移除**，不会像"杀掉进程"那样留下幽灵图标。

## 用法

```bash
node patch-shell-tray.mjs status   # 已安装的 asar 是否带这块补丁
node patch-shell-tray.mjs build    # 从原始备份生成暂存包，并逐文件校验（零差异 + main.js 的 SHA256 重算）
node patch-shell-tray.mjs detach   # 45 秒后由 WMI 创建的独立进程完成切换（推荐）
node patch-shell-tray.mjs swap     # 立即切换（会先关闭应用）
node patch-shell-tray.mjs revert   # 还原原始 asar 并重启
```

可选参数：`--install=<目录>`、`--asar=<文件>`、`--log=<文件>`、`--delay=<毫秒>`。不传参数时会**自动探测**安装位置：先看运行中的进程路径，再回退到 `%LOCALAPPDATA%\Programs\DeepSeek Harness`。

日志默认写到 `%USERPROFILE%\.dsh\dsh-tray-restart\patch.log`。

## 安全设计（每一处都为了"改坏了也能回来"）

- **从原始备份重建**：如果 `app.asar.orig` 存在就用它做基底，避免在已打过补丁的包上再插一次（那会插入第二项）。
- **逐文件校验**：重新读回新包，与原始包逐个文件比对字节，并重算被改动文件的 SHA256；任何一处不符就中止，**不会切换**。
- **切换前备份 + 自动回退**：原文件备份为 `app.asar.orig`；若新包起不来，切换进程会在 40 秒内**还原原文件并重启应用**。
- **`detach` 而不是 `swap`**：Windows 上应用会把它的后代放进作业对象，直接在应用**内部**启动的切换进程会随应用一起被杀（实测：它死在"关闭应用"这一步，一个文件都没来得及碰）。用 WMI 创建的进程归 WMI 提供程序宿主所有，应用杀不到它。

## 如实说明

- 这是对**厂商打包应用**的本地补丁，不是官方功能；**应用更新或重装会覆盖 `app.asar`**，届时重跑 `build` + `detach` 即可。
- 长期解法是让壳自己带上这一项：见 [`docs/dsh-restart-api-request.zh.md`](docs/dsh-restart-api-request.zh.md)（精确到文件与行号的补丁，以及插件侧退出/重启 API 的请求）。
- 只在 **Windows** 上验证过——托盘本来就是 Windows 分支（`if (process.platform === "win32")`），macOS 走的是应用菜单。

## English

Adds a **"Restart DeepSeek Harness"** item to the desktop shell's tray menu, by patching `lib/main.js` inside the
installed `resources/app.asar` — locally and reversibly. The tray lives in the shell's main process, and a plugin
cannot reach it (the host has no Electron APIs; the page only gets the fixed `dshDesktop` preload surface), so this
has to be a shell-side change. The item calls the shell's own `app.relaunch()` + `quitWithoutConfirmation()`, which
is why Electron removes the tray icon itself instead of leaving a ghost icon behind.

Commands: `status`, `build` (staged archive + file-by-file verification), `detach` (recommended: a WMI-created
process swaps it in, because the application kills its descendants when it exits), `swap`, `revert`. Options:
`--install=<dir>`, `--asar=<file>`, `--log=<file>`, `--delay=<ms>`.

Being honest about it: this patches the vendor's packaged application; an application update or reinstall overwrites
`app.asar` and the patch has to be applied again. The original is backed up as `app.asar.orig`, and a patched archive
that fails to start is restored automatically within 40 seconds. The long-term fix is for the shell to carry the
item — see `docs/dsh-restart-api-request.md`. Windows only.
