# AGENTS.md

本仓库是 [Meloong-Git/PCL](https://github.com/Meloong-Git/PCL)（Plain Craft Launcher 2 开源版）的个人 fork，远程为 `git@github.com:Noplch0/PCL.git`。在此仓库工作时应了解以下背景与约定。

## 项目结构

- `Plain Craft Launcher 2/`：主项目，VB.NET WPF，目标框架 .NET Framework 4.8（旧式非 SDK 工程，`.vbproj`）
- `PCLCS/`、`MeloongCore/`（子模块）：C# 依赖库，SDK 风格工程，net48。主项目将这些 DLL **内嵌为 exe 资源**并在运行时经 `Application.xaml.vb` 的 `AssemblyResolve` 动态加载，因此分发只需单个 exe
- `patches/`：官方正式版的加密补丁（**不可读取**），官方的主题配色表、更新器、密钥等被裁剪内容都在其中，无法提取参考
- `.github/workflows/build.yml`：push 后自动构建 Debug/Release 两个 exe 产物，并会在构建时把 GitHub Secrets 注入 `ModSecret.vb`

## Fork 定制（2026-10-20 已完成，改动尚未提交）

核心改动集中在 `Modules/ModSecret.vb`（该文件是官方刻意裁剪的骨架，fork 的定制实现都补在这里）：

1. **全部主题默认解锁**
   - `ThemeRefresh`：自实现了 15 个主题（ID 0-14）→ HSL 配色映射（官方配色在加密补丁中，此为等价方案）；滑稽彩(12)随机变色，由主定时器 250ms 刷新；真·滑稽彩（`ThemeDontClick=2`，50ms 刷新）经设置页连点滑稽彩 5 次触发
   - `ThemeCheckAll`：启用全部主题按钮并移除解锁遮罩（玄素黑遮罩、秋仪金/活跃橙/极客蓝的半透明点击拦截层）
   - `PageSetupUI.xaml`：9 个主题单选按钮的 `IsEnabled="False"` 已移除；`PageSetupUI.xaml.vb`：正式版的赞助遮罩（`PanLauncherHide`）不再显示
2. **MC 本体下载限制解除**：`RestrictMcDownload = False`（上游 2.99.0.0 引入的 `True` 限制了下载安装页、整合包安装、版本修改页）
3. **更新检查定向至本 fork**：`UpdateCheckByButton` 查询 `api.github.com/repos/Noplch0/PCL/releases`（失败走 gh-proxy 镜像），与 `VersionBaseName` 比较，有新版弹窗并打开发布页。**仓库尚无任何 Release**，发布带版本号 tag（如 `2.99.0.1`）的 Release 后才会提示更新；开源版本无任何自动更新/官方推送逻辑，无需担心官方更新提示
4. 启动时的"开源版本说明"弹窗文案已改为本分支说明

## 敏感配置（不得明文进仓库）

- **微软登录 ClientId**：存于 GitHub Secret `CLIENT_ID`（CI 构建注入）+ 本机用户环境变量 `PCL_MS_CLIENT_ID`（本地 exe 运行时读取，见 `ModSecret.vb` 开头）。对应的 Azure 应用注册已启用"移动和桌面应用"平台与"允许公共客户端流"，设备码流程（登录步骤 1~3）实测可用
- **⚠️ 已知限制（2026-10 实测）**：登录步骤 4 `api.minecraftservices.com/authentication/login_with_xbox` 对自建 Azure 应用返回 403 `"Invalid app registration"`——Mojang 维护 client id 服务端白名单，自建 id 一律拒绝，且目前无公开申请渠道（见 aka.ms/AppRegInfo 及 Microsoft Q&A 多个无解帖子）。**个人 fork 无法合规实现正版登录**；正版需求请用官方 PCL 正式版（可与本 fork 共存共用 .minecraft），勿盗用其他启动器的 client id（会导致其被吊销）
- **CurseForge API Key**：已配置（2026-10-23 实测 `GET /v1/games/432` 返回 200）。key 只存于 GitHub Secret `CURSEFORGE_API_KEY`（CI 构建注入）与本机用户环境变量 `PCL_CURSEFORGE_API_KEY`（本地 exe 运行时读取），仓库内不出现明文。运行时由 `SecretHeadersSign` 给 `api.curseforge.com` / `forgecdn.net` 请求加 `x-api-key` 头
- **CI 注入实现约束**：`build.yml` 的 `Replace` 步骤必须用字符串 `.Replace()` 做字面量替换，**不能用 `-replace`**。标记串里的 `If(...)` 在正则里会被当成分组（实际匹配 `Ifxxx`）从而永不命中——旧写法因此一直静默失败，两个 Secret 都没真正注入；且 Secret 值形如 `$2a$10$...`，`$n` 在正则替换串中会被当作分组引用。脚本还会在找不到标记时直接 throw，避免再次静默产出无 key 的 exe

## 本地构建（本机无 .NET 4.8 开发包、BuildTools 缺 .NET SDK 的 workaround）

辅助文件在 `.tmp/`（已 gitignore，勿删）：

1. `git submodule update --init`（首次）
2. `dotnet restore PCLCS/PCLCS.csproj`，再 `dotnet build PCLCS/PCLCS.csproj -c Debug`
3. 编译主项目（Git Bash）：

```bash
MSBuildSDKsPath="D:/Projects/PCL/.tmp/Sdks" \
"C:/Program Files (x86)/Microsoft Visual Studio/2022/BuildTools/MSBuild/Current/Bin/amd64/MSBuild.exe" \
"Plain Craft Launcher 2/Plain Craft Launcher 2.vbproj" \
-p:Configuration=Debug -v:minimal -nologo \
"-p:TargetFrameworkRootPath=D:\Projects\PCL\.tmp\refasm\pkg\build\\" \
-p:BuildProjectReferences=false
```

要点：`.tmp/refasm/` 是 NuGet 包 `Microsoft.NETFramework.ReferenceAssemblies.net48` 的解压（`TargetFrameworkRootPath` 必须指向含 `.NETFramework` 的 `build` 目录且带尾反斜杠）；`.tmp/Sdks/` 是 dotnet 9 SDK 的 Sdks 副本加两个手建占位 SDK（`Microsoft.NET.SDK.WorkloadAutoImportPropsLocator`、`Microsoft.NET.SDK.WorkloadManifestTargetsLocator`，各含空的 Sdk.props/Sdk.targets 及 AutoImport.props / WorkloadManifest.targets）；`BuildProjectReferences=false` 规避 BuildTools 编 SDK 工程报 `GenerateDepsFile` 缺 `RuntimeGraphPath`。产物：`Plain Craft Launcher 2/bin/Plain Craft Launcher 2.exe`（单文件即可分发，bin 内其余 DLL/pdb/xml 均不需要）。

最省事的构建方式是直接 push，由 GitHub Actions 出产物（还能自动注入 Secrets）。
