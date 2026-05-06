# BepInEx 自定义补丁流程设计

日期: 2026-05-07

## 背景

KTC-ModDevLibs 仓库通过 GitHub Actions (`update-deps.yml`) 自动更新游戏依赖和 BepInEx 运行时。BepInEx 官方构建中包含的 Cpp2IL 和 Il2CppInterop 版本存在一些已知问题，需要使用自定义补丁版本替换。

## 目标

在 BepInEx 更新流程（阶段二）中，在下载官方 IL2CPP/Mono core 之后、提交之前，自动应用 Cpp2IL 和 Il2CppInterop 的自定义补丁。

## 需求

1. **触发条件**: 仅在 BepInEx 有新版本且 IL2CPP core 下载成功时执行
2. **失败策略**: 严格模式 — 任一补丁下载失败则跳过整个 BepInEx 提交
3. **补丁版本**: 固定 URL，无版本检测（URL 包含明确版本号）
4. **目标目录**: `BIE6_IL2CPP/core/`，覆盖同名文件

## 补丁来源

| 组件 | 补丁 URL |
|------|---------|
| Cpp2IL | `https://github.com/abevol/Cpp2IL/releases/download/v2022.1.0-asmresolver-beta.5/Cpp2IL-net6.0-asmresolver-beta.5.zip` |
| Il2CppInterop | `https://github.com/abevol/Il2CppInterop/releases/download/v1.5.1-improve-genericmethod-signatures/Il2CppInterop-v1.5.1-net6.0.zip` |

## 流程设计

### 整体流程

```
阶段二（修改后）:
  Check BepInEx build                 [bepinex-check]
    → Download and update IL2CPP core [update-il2cpp-core]
    → Download and update Mono core   [update-mono-core]
    → Apply custom patches ★          [patch-custom]        ← 新增
    → Commit BepInEx updates          [commit-bepinex]
```

### 新增步骤: apply-custom-patches

**ID**: `apply-custom-patches`

**触发条件**:
```yaml
if: |
  steps.bepinex-check.outputs.bepinex_changed == 'true' &&
  steps.update-il2cpp-core.outcome == 'success'
```

**执行逻辑**:
```bash
# 1. 下载并应用 Cpp2IL 补丁
curl -sSL "<cpp2il-url>" -o cpp2il.zip
unzip -qo cpp2il.zip -d BIE6_IL2CPP/core/

# 2. 下载并应用 Il2CppInterop 补丁
curl -sSL "<il2cppinterop-url>" -o il2cppinterop.zip
unzip -qo il2cppinterop.zip -d BIE6_IL2CPP/core/
```

**失败策略**: 不使用 `continue-on-error`。任何一步失败 → step 整体失败 → `commit-bepinex` 不被触发。

### 提交步骤无需修改

`commit-bepinex` 已有的 `git add BIE6_IL2CPP/core/` 会自动包含补丁文件。现有条件逻辑足以覆盖：

- 补丁成功 → commit 正常执行
- 补丁失败 → commit 由于步骤失败而跳过
- IL2CPP core 失败 → 补丁跳过（`if` 不满足），Mono 路径不受影响

`commit-bepinex` 现有条件：
```yaml
steps.bepinex-check.outputs.bepinex_changed == 'true' &&
(steps.update-il2cpp-core.outcome == 'success' || steps.update-mono-core.outcome == 'success')
```

无需修改。补丁成功 → IL2CPP core 成功 → 提交继续；补丁失败 → step 报错 → GitHub Actions 默认行为跳过后续步骤。

## 边界情况

| 场景 | 行为 |
|------|------|
| BepInEx 无更新 | 补丁步骤不执行 |
| IL2CPP core 下载失败 | 补丁跳过，仅 Mono core 提交 |
| Cpp2IL 补丁下载失败 | curl 非零退出 → step 失败 → BepInEx 提交被跳过 |
| Il2CppInterop 补丁下载失败 | 同上 |
| 两个补丁都成功 | 文件覆盖到 core 目录，一并提交 |

## 与现有模式的兼容性

- 补丁步骤不使用 `continue-on-error`，与 `update-il2cpp-core`/`update-mono-core` 不同（有意为之，因为用户选择严格模式）
- 补丁步骤的 `if` 条件依赖 `update-il2cpp-core.outcome`，与现有步骤间的依赖风格一致
