[English](README.md) | 中文

# KTC-ModDevLibs

Kingdom Two Crowns 模组开发引用库集合。为基于 [BepInEx](https://github.com/BepInEx/BepInEx) 框架的游戏模组项目提供编译时所需的依赖程序集引用，支持 **IL2CPP** 与 **Mono** 两种游戏版本。

## 目录结构

```
BIE6_IL2CPP/          # IL2CPP 版本引用库
  core/               # BepInEx 核心程序集（0Harmony、BepInEx.Core、Il2CppInterop 等）
  interop/            # IL2CPP 互操作程序集（Unity 引擎、游戏程序集等）
BIE6_Mono/            # Mono 版本引用库
  core/               # BepInEx 核心程序集（0Harmony、BepInEx.Core、Mono.Cecil 等）
  Managed/            # Mono Managed 程序集（Unity 引擎、游戏程序集、公版化 Assembly-CSharp 等）
  MakePublic.bat      # Assembly-CSharp 公版化工具
  BepInEx.AssemblyPublicizer.Cli.exe  # 程序集公版化 CLI
```

## 使用方法

### 1. 添加为 Git Submodule

```bash
# 在模组项目根目录下
mkdir -p deps
git submodule add https://github.com/abevol/KTC-ModDevLibs.git deps/KTC-ModDevLibs
```

目录结构：

```
YourModProject/
  deps/
    KTC-ModDevLibs/     # 引用库（子仓库）
      BIE6_IL2CPP/
      BIE6_Mono/
  YourMod.csproj
  ...
```

### 2. 在 .csproj 中添加引用

添加 submodule 后，在 `.csproj` 中添加引用：

```xml
<!-- IL2CPP 版本引用示例 -->
<Reference Include="Assembly-CSharp">
  <HintPath>..\deps\KTC-ModDevLibs\BIE6_IL2CPP\interop\Assembly-CSharp.dll</HintPath>
  <Private>False</Private>
</Reference>

<!-- Mono 版本引用示例 -->
<Reference Include="UnityEngine.CoreModule">
  <HintPath>..\deps\KTC-ModDevLibs\BIE6_Mono\Managed\UnityEngine.CoreModule.dll</HintPath>
  <Private>False</Private>
</Reference>
```

> **克隆已有项目**：克隆已使用本 submodule 的项目时，用 `git clone --recurse-submodules <url>` 可一次性拉取所有内容。如果已克隆但未拉取 submodule，运行 `git submodule update --init --recursive`。

## 自动化更新

本仓库通过 GitHub Actions [workflow](.github/workflows/update-deps.yml) 自动更新依赖，无需手动维护。

- **触发**: 每天 UTC 18:00 定时运行，支持手动触发（`workflow_dispatch`）
- **阶段一 — 游戏更新**: 使用 [DepotDownloader](https://github.com/abevol/DepotDownloader) 检测游戏清单变化，自动下载最新托管程序集并生成 IL2CPP interop 文件
- **阶段二 — BepInEx 更新**: 监控 [BepInEx Bleeding Edge](https://builds.bepinex.dev/projects/bepinex_be) 最新构建，更新 IL2CPP/Mono core 运行时并应用自定义 Cpp2IL/Il2CppInterop 补丁，解决与游戏不兼容导致崩溃的问题（[Issue #24](https://github.com/abevol/KingdomMod/issues/24)、[commit 7795dc4](https://github.com/abevol/KingdomMod/commit/7795dc417126ef8c8b8b7f92bd92974c7062650f)）
- **版本追踪**: `.update-state.json` 记录各组件版本（游戏 manifest ID、BepInEx build ID），避免重复更新

### 所需 Secrets

Workflow 需要在仓库中配置以下 [GitHub Secrets](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)：

| Secret | 说明 |
|--------|------|
| `STEAM_ACC` | Steam 账号用户名（用于下载游戏 depot） |
| `STEAM_ACCOUNT_CONFIG` | Base64 编码的 Steam `account.config`（用于身份验证） |

## 程序集来源

| 程序集 | 来源 |
|--------|------|
| 0Harmony.dll | [Harmony](https://github.com/pardeike/Harmony) |
| BepInEx.Core.dll | [BepInEx](https://github.com/BepInEx/BepInEx) |
| Il2CppInterop.* | [Il2CppInterop](https://github.com/BepInEx/Il2CppInterop) |
| Mono.Cecil.dll | [Mono.Cecil](https://github.com/jbevain/cecil) |
| UnityEngine.* | Unity 引擎（游戏自带） |
| Assembly-CSharp.dll | Kingdom Two Crowns 游戏程序集（已公版化） |
| 其他第三方程序集 | 游戏自带（DOTween、PlayFab、Rewired、Steamworks.NET 等） |

## 许可证

[MIT License](LICENSE)

> **注意**：本仓库仅包含引用程序集（DLL 文件），用于模组项目编译时引用。不包含任何游戏资源或反编译源代码。
