<p align="right">English · <a href="README.zh-CN.md">中文</a></p>

# KTC-ModDevLibs

Kingdom Two Crowns modding reference library collection. Provides compile-time dependency assemblies for [BepInEx](https://github.com/BepInEx/BepInEx) mod projects, supporting both **IL2CPP** and **Mono** game versions.

## Directory Structure

```
BIE6_IL2CPP/          # IL2CPP version reference libraries
  core/               # BepInEx core assemblies (0Harmony, BepInEx.Core, Il2CppInterop, etc.)
  interop/            # IL2CPP interop assemblies (Unity Engine, game assemblies, etc.)
BIE6_Mono/            # Mono version reference libraries
  core/               # BepInEx core assemblies (0Harmony, BepInEx.Core, Mono.Cecil, etc.)
  Managed/            # Mono managed assemblies (Unity Engine, game assemblies, publicized Assembly-CSharp, etc.)
  MakePublic.bat      # Assembly-CSharp publicizer tool
  BepInEx.AssemblyPublicizer.Cli.exe  # Assembly publicizer CLI
```

## Usage

### Adding as a Git Submodule (Recommended)

```bash
# In your mod project root directory
mkdir -p deps
git submodule add https://github.com/abevol/KTC-ModDevLibs.git deps/KTC-ModDevLibs
```

This creates the following directory structure:

```
YourModProject/
  deps/
    KTC-ModDevLibs/     # Reference library (submodule)
      BIE6_IL2CPP/
      BIE6_Mono/
  YourMod.csproj
  ...
```

### Reference in .csproj

After adding the submodule, add references in your `.csproj`:

```xml
<!-- IL2CPP build references -->
<Reference Include="Assembly-CSharp">
  <HintPath>..\deps\KTC-ModDevLibs\BIE6_IL2CPP\interop\Assembly-CSharp.dll</HintPath>
  <Private>False</Private>
</Reference>

<!-- Mono build references -->
<Reference Include="UnityEngine.CoreModule">
  <HintPath>..\deps\KTC-ModDevLibs\BIE6_Mono\Managed\UnityEngine.CoreModule.dll</HintPath>
  <Private>False</Private>
</Reference>
```

### Cloning a Project with Submodules

When cloning a project that uses this library as a submodule:

```bash
git clone --recurse-submodules <your-project-url>
```

Or if you already cloned without submodules:

```bash
git submodule update --init --recursive
```

## Assembly Sources

| Assembly | Source |
|----------|--------|
| 0Harmony.dll | [Harmony](https://github.com/pardeike/Harmony) |
| BepInEx.Core.dll | [BepInEx](https://github.com/BepInEx/BepInEx) |
| Il2CppInterop.* | [Il2CppInterop](https://github.com/BepInEx/Il2CppInterop) |
| Mono.Cecil.dll | [Mono.Cecil](https://github.com/jbevain/cecil) |
| UnityEngine.* | Unity Engine (bundled with game) |
| Assembly-CSharp.dll | Kingdom Two Crowns game assembly (publicized) |
| Other third-party assemblies | Bundled with game (DOTween, PlayFab, Rewired, Steamworks.NET, etc.) |

## License

[MIT License](LICENSE)

> **Note**: This repository contains only reference assemblies (DLL files) for compile-time referencing in mod projects. It does not include any game assets or decompiled source code.
