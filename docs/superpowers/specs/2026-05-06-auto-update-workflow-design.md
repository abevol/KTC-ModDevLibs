# Auto-Update GitHub Workflow 设计

## 概述

为 KTC-ModDevLibs 仓库创建 GitHub Actions workflow，自动检测游戏版本和 BepInEx 前沿构建是否更新，如有更新则自动下载、处理并提交到仓库。

## 触发方式

- `schedule`：每周一 UTC 00:00（`cron: 0 0 * * 1`）
- `workflow_dispatch`：手动触发

## Runner

`ubuntu-latest`

## Workflow 结构

单 workflow 文件 `.github/workflows/update-deps.yml`，单 job，分为两个顺序阶段：

### 阶段一：游戏更新

```
检测 Linux/Windows manifest ID → 对比状态文件 → 有变更则:
  ├─ 下载 Linux Managed 文件 → 替换 BIE6_Mono/Managed/
  ├─ 下载 Windows 文件 → il2cpp-interop-cli 生成 interop → 替换 BIE6_IL2CPP/interop/
  └─ 提交（如有实际文件变更）
```

### 阶段二：BepInEx 更新

```
抓取构建页面 → 解析最新 build ID → 对比状态文件 → 有变更则:
  ├─ 下载 IL2CPP-win-x64 zip → 提取 BepInEx/core → 替换 BIE6_IL2CPP/core/
  ├─ 下载 Mono-linux-x64 zip → 提取 BepInEx/core → 替换 BIE6_Mono/core/
  └─ 提交（如有实际文件变更）
```

每个组件的下载/替换步骤使用 `continue-on-error`，确保单个组件失败不影响其他。

## 环境准备（在阶段一之前执行）

1. `actions/checkout` 检出仓库（含 `fetch-depth: 0` 和必要的 token）
2. 安装 .NET 8.0 SDK（`actions/setup-dotnet`）
3. 下载 DepotDownloader：`https://github.com/SteamRE/DepotDownloader/releases/download/DepotDownloader_3.4.0/DepotDownloader-linux-x64.zip`
4. 下载 il2cpp-interop-cli：`https://github.com/abevol/BepInEx-CLI/releases/download/v0.1.0/il2cpp-interop-cli-v0.1.0-linux-x64.tar.gz`

## 状态文件

**位置**：`.update-state.json`

**格式**：

```json
{
  "game": {
    "linuxManifestId": "2442055016837937796",
    "windowsManifestId": "2442055016837937797",
    "updatedAt": "04/28/2026 11:47:12"
  },
  "bepinex": {
    "buildId": "755",
    "commitHash": "3fab71a",
    "updatedAt": "2026-03-07T13:11:44.6967620+00:00"
  }
}
```

## 游戏变更检测

对 Linux（`-os linux -osarch 64`）和 Windows（`-os windows -osarch 64`）分别执行：

```
DepotDownloader -app 701160 -depot 701163 -manifest-only \
  -os <platform> -osarch 64 \
  -username $STEAM_ACC -password $STEAM_PWD -remember-password
```

从输出中用正则 `/Manifest (\d+) \((\d{2}/\d{2}/\d{4} \d{2}:\d{2}:\d{2})\)/` 提取 manifest ID 和时间戳。

任一平台的 manifest ID 发生变化即触发游戏更新阶段。

## 游戏文件下载与处理

### Linux Managed（BIE6_Mono/Managed）

文件列表 `linux-files.txt`：
```
regex:^KingdomTwoCrowns_Data/Managed/.*$
```

下载命令：
```
DepotDownloader -app 701160 -depot 701163 -manifest $NEW_LINUX_MANIFEST \
  -filelist linux-files.txt -os linux -osarch 64 \
  -username $STEAM_ACC -password $STEAM_PWD -remember-password
```

下载后文件位于 `depots/701163/<depot_id>/KingdomTwoCrowns_Data/Managed/`，清空仓库中 `BIE6_Mono/Managed/` 后全量替换。

### Windows IL2CPP interop（BIE6_IL2CPP/interop）

文件列表 `win-files.txt`：
```
KingdomTwoCrowns.exe
GameAssembly.dll
KingdomTwoCrowns_Data/globalgamemanagers
KingdomTwoCrowns_Data/il2cpp_data/Metadata/global-metadata.dat
```

下载命令：
```
DepotDownloader -app 701160 -depot 701163 -manifest $NEW_WINDOWS_MANIFEST \
  -filelist win-files.txt -os windows -osarch 64 \
  -username $STEAM_ACC -password $STEAM_PWD -remember-password
```

将下载的文件按 il2cpp-interop-cli 要求组装成游戏目录结构后运行：
```
il2cpp-interop-cli -g <download-dir> -m KingdomTwoCrowns -o BIE6_IL2CPP/interop/
```

清空 `BIE6_IL2CPP/interop/` 后写入新文件。

### 提交

commit message 格式：
```
chore: update game deps to manifest linux/<id>, win/<id>
```

提交后更新状态文件中 `game` 段的两个 manifest ID 和 `updatedAt`。

## BepInEx 变更检测

抓取 `https://builds.bepinex.dev/projects/bepinex_be`，解析第一个 `<div class="artifact-item">`：

- `artifact-id` → build 号（如 `#755`）
- `hash-button` 的 `href` → commit hash（如 `3fab71a`）
- `build-date` → 构建时间
- `changelog` 中 `<li>` 内容 → 变更日志

与状态文件中 `buildId` 对比，不同则触发更新。

## BepInEx 下载与处理

从解析的 `<a class="artifact-link">` 中匹配两个目标文件名：
- `BepInEx-Unity.IL2CPP-win-x64-6.0.0-be.{buildId}+{hash}.zip`
- `BepInEx-Unity.Mono-linux-x64-6.0.0-be.{buildId}+{hash}.zip`

下载 URL：`https://builds.bepinex.dev/projects/bepinex_be/{buildId}/{filename}`

### IL2CPP core（BIE6_IL2CPP/core）

```
curl -L "$IL2CPP_URL" -o il2cpp.zip
unzip il2cpp.zip "BepInEx/core/*" -d il2cpp-extract/
```

清空 `BIE6_IL2CPP/core/`，将 `il2cpp-extract/BepInEx/core/` 内容写入。

### Mono core（BIE6_Mono/core）

```
curl -L "$MONO_URL" -o mono.zip
unzip mono.zip "BepInEx/core/*" -d mono-extract/
```

清空 `BIE6_Mono/core/`，将 `mono-extract/BepInEx/core/` 内容写入。

### 提交

commit message 格式：
```
chore: update BepInEx BE to #<buildId> (<hash>)

- Build date: <build-date>
- <changelog summary line 1>
- <changelog summary line 2>
```

提交后更新状态文件中 `bepinex` 段的 `buildId`、`commitHash`、`updatedAt`。

## 错误处理

| 场景 | 处理策略 |
|------|----------|
| Steam 登录失败 | 游戏阶段整体失败（不标记 continue-on-error） |
| 构建页面不可达 | 跳过 BepInEx 阶段，不标记失败 |
| 单个组件下载/处理失败 | continue-on-error，允许其他组件继续 |
| 无实际文件变更 | `git diff --quiet` 检查，跳过空提交 |
| 清空旧文件 | `rm -rf <dir>/*` 再写入，避免残留 |

## GitHub Secrets

| Secret | 说明 |
|--------|------|
| `STEAM_ACC` | Steam 账号用户名 |
| `STEAM_PWD` | Steam 账号密码 |

## 前提条件

- Steam 账号需移除邮箱验证，确保可在无头环境下直接登录
- 仓库 Settings → Actions → General → Workflow permissions 已设为 "Read and write permissions"
