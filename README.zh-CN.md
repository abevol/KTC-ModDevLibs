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
