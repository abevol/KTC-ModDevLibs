English | [中文](README.zh-CN.md)

# KTC-ModDevLibs

Kingdom Two Crowns modding reference library collection. Provides compile-time dependency assemblies based on the [BepInEx](https://github.com/BepInEx/BepInEx) game mod projects, supporting both **IL2CPP** and **Mono** game versions.

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

### 1. Add as Git Submodule

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

### 2. Reference in .csproj

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

> **Cloning an existing project**: When cloning a project that already uses this submodule, run `git clone --recurse-submodules <url>` to pull everything at once. If you've already cloned without submodules, run `git submodule update --init --recursive`.

## Automated Updates

This repository is automatically updated via GitHub Actions [workflow](.github/workflows/update-deps.yml), eliminating manual maintenance.

- **Schedule**: Daily at UTC 18:00 via cron, with manual trigger support (`workflow_dispatch`)
- **Phase 1 — Game Update**: Uses [DepotDownloader](https://github.com/abevol/DepotDownloader) to detect game manifest changes, automatically downloads updated managed assemblies and generates IL2CPP interop files
- **Phase 2 — BepInEx Update**: Monitors [BepInEx Bleeding Edge](https://builds.bepinex.dev/projects/bepinex_be) builds, updates IL2CPP/Mono core runtimes, and applies custom Cpp2IL/Il2CppInterop patches to fix game incompatibility issues that cause crashes ([Issue #24](https://github.com/abevol/KingdomMod/issues/24), [commit 7795dc4](https://github.com/abevol/KingdomMod/commit/7795dc417126ef8c8b8b7f92bd92974c7062650f))
- **Version Tracking**: `.update-state.json` records component versions (game manifest IDs, BepInEx build ID) to avoid redundant updates

### Required Secrets

The workflow requires the following [GitHub Secrets](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions):

| Secret | Description |
|--------|-------------|
| `STEAM_ACC` | Steam account username for downloading game depots |
| `STEAM_ACCOUNT_CONFIG` | Base64-encoded Steam `account.config` for authentication |

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
