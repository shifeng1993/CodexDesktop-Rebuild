# Codex Desktop Rebuild 中文使用说明

这个项目用于从 upstream Codex Desktop 包中提取资源、打补丁，然后重新打包 Windows/macOS/Linux 版本。

本文以 Windows x64 构建为主，记录完整步骤和实际踩过的坑。

## 环境准备

### 1. 安装依赖

```powershell
cd /d F:\CodexDesktop-Rebuild
npm install
```

### 2. 安装 7-Zip

Windows 构建需要用 7-Zip 解压 MSIX，并在最后压缩 zip。

```powershell
winget install 7zip.7zip
```

安装后确认：

```powershell
where 7z
where 7zz
```

如果找不到，但实际安装在 `C:\Program Files\7-Zip`，可以给当前终端临时加 PATH：

```powershell
$env:Path = "C:\Program Files\7-Zip;$env:Path"
where 7z
```

如果脚本需要 `7zz`，但安装目录里只有 `7z.exe`，可以复制一份：

```powershell
Copy-Item "C:\Program Files\7-Zip\7z.exe" "C:\Program Files\7-Zip\7zz.exe"
```

然后确认：

```powershell
where 7zz
```

### 3. 使用官方 npm 源

`@cometix/codex` 在 `npmmirror` 上可能返回空元数据，导致脚本拼出错误包名。

建议切回官方 npm 源：

```powershell
npm config set registry https://registry.npmjs.org
npm config get registry
npm view @cometix/codex version
```

正常应该能看到类似：

```text
0.137.0-cometix
```

## Windows x64 构建步骤

### 1. 同步 upstream Windows 包

```powershell
npm run sync -- --skip-mac
```

这一步会下载 Windows MSIX，并生成：

```text
src\win\_asar
```

如果没有这个目录，后续 `patch` 和 `build` 都会失败。

### 2. 打 Windows 补丁

```powershell
npm run patch:win
```

### 3. 构建 Windows x64

```powershell
npm run build:win-x64
```

成功后会生成：

```text
out\Codex-win-x64-<version>.zip
```

例如：

```text
out\Codex-win-x64-26.608.12217.zip
```

## 常见问题

### win/_asar/ not found

报错：

```text
[x] win/_asar/ not found. Run sync-upstream first.
```

原因：还没有成功执行 upstream 同步，`src\win\_asar` 不存在。

解决：

```powershell
npm run sync -- --skip-mac
```

同步成功后确认：

```powershell
dir F:\CodexDesktop-Rebuild\src\win\_asar
```

### Failed to extract MSIX

报错：

```text
[x] win: Failed to extract ... .msix
```

原因通常是 `7z` / `7zz` 没有在 PATH 中。

解决：

```powershell
$env:Path = "C:\Program Files\7-Zip;$env:Path"
where 7z
npm run sync -- --skip-mac
```

下载好的 MSIX 会缓存在：

```text
C:\Users\<用户名>\AppData\Local\Temp\codex-sync
```

所以修好 7-Zip 后通常不需要重新下载。

### patch-copyright / patch-devtools: No main bundle found

报错：

```text
[x] No main bundle found
```

如果前面 `sync` 没成功，这个错误只是连锁反应。

先检查：

```powershell
dir F:\CodexDesktop-Rebuild\src\win\_asar
```

如果目录不存在，先重新跑：

```powershell
npm run sync -- --skip-mac
```

### npm pack @cometix/codex@-win32-x64

报错：

```text
npm pack @cometix/codex@-win32-x64
npm error code ETARGET
```

原因：脚本通过下面命令获取版本：

```powershell
npm view @cometix/codex version
```

如果 npm 源是 `https://registry.npmmirror.com`，可能返回空字符串，导致版本被拼成空：

```text
@cometix/codex@-win32-x64
```

解决：

```powershell
npm config set registry https://registry.npmjs.org
npm view @cometix/codex version
```

正确的 Windows 包版本形态类似：

```text
@cometix/codex@0.137.0-cometix-win32-x64
```

本项目脚本也应当对空版本做保护：如果拿不到 `@cometix/codex` 版本，就跳过替换，继续保留 upstream 自带的 `codex.exe`。

### 7zz 不是内部或外部命令

报错：

```text
'7zz' is not recognized
```

原因：脚本最后压缩 zip 时调用了 `7zz`，但 Windows 7-Zip 安装版有时只有 `7z.exe`。

解决方式一：确认 `7zz.exe` 是否存在：

```powershell
dir "C:\Program Files\7-Zip\7zz.exe"
```

解决方式二：如果只有 `7z.exe`，可以复制一份为 `7zz.exe`：

```powershell
Copy-Item "C:\Program Files\7-Zip\7z.exe" "C:\Program Files\7-Zip\7zz.exe"
```

解决方式三：把 `scripts\build-from-upstream.js` 里的：

```js
execSync(`7zz a -tzip -mx=5 "${zipPath}" .`, { cwd: outApp });
```

改成：

```js
execSync(`7z a -tzip -mx=5 "${zipPath}" .`, { cwd: outApp });
```

### 构建停在 [zip] 看起来不动

输出：

```text
[zip] Codex-win-x64-xxx.zip
```

这通常不是卡死，而是 7-Zip 正在压缩。脚本没有打印 7-Zip 进度，zip 文件会在后台增长。

可以另开终端查看：

```powershell
dir F:\CodexDesktop-Rebuild\out\*.zip
```

成功后会看到：

```text
[ok] F:\CodexDesktop-Rebuild\out\Codex-win-x64-xxx.zip (... MB)
```

### EPERM, Permission denied: out\win

报错：

```text
Error: EPERM, Permission denied: F:\CodexDesktop-Rebuild\out\win
```

原因通常是旧的输出目录正在被占用，例如你运行了：

```text
out\win\Codex-win32-x64\Codex.exe
```

解决：关闭从 `out\win` 启动的 Codex 程序，或关闭正在浏览该目录的资源管理器窗口，然后重新运行：

```powershell
npm run build:win-x64
```

### old hash not found in exe

警告：

```text
[!] old hash not found in exe
```

含义：脚本重新打包 `app.asar` 后，尝试把新的 ASAR integrity hash 写回 `Codex.exe`，但没有找到旧 hash。

这不是 zip 生成失败的直接原因，但可能影响应用启动。构建完成后建议解压并实际运行：

```text
out\win\Codex-win32-x64\Codex.exe
```

### @cometix/codex not found, keeping upstream codex

警告：

```text
[!] @cometix/codex not found, keeping upstream codex
```

含义：没有成功替换为 `@cometix/codex` 的 CLI，脚本会保留 upstream 包里的 `codex.exe`。

这不影响 zip 生成，但会影响你是否使用 Cometix 版本的 CLI。

## 推荐完整命令

在一个新的 PowerShell 终端中执行：

```powershell
cd /d F:\CodexDesktop-Rebuild

npm config set registry https://registry.npmjs.org
$env:Path = "C:\Program Files\7-Zip;$env:Path"

npm run sync -- --skip-mac
npm run patch:win
npm run build:win-x64
```

构建成功后，检查：

```powershell
dir F:\CodexDesktop-Rebuild\out\Codex-win-x64-*.zip
```

也可以用 7-Zip 校验压缩包：

```powershell
7z t F:\CodexDesktop-Rebuild\out\Codex-win-x64-*.zip
```

看到：

```text
Everything is Ok
```

说明 zip 文件本身是完整的。
