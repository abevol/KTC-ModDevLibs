# Apply Custom Patches to BepInEx Core — Implementation Plan

> **For agentic workers:** 使用 subagent-driven-development (推荐) 或 executing-plans 按任务逐步实现。步骤使用 checkbox (`- [ ]`) 语法跟踪。

**目标:** 在 `.github/workflows/update-deps.yml` 的阶段二中，在 `update-mono-core` 和 `commit-bepinex` 之间新增一个步骤，自动下载 Cpp2IL 和 Il2CppInterop 自定义补丁并解压到 `BIE6_IL2CPP/core/`。

**架构:** 单一步骤，原子执行两个补丁的下载和解压。不使用 `continue-on-error`（严格模式），补丁失败则跳过提交。无需修改现有步骤的 `if` 条件。

**涉及技术:** GitHub Actions YAML, bash, curl, unzip

---

### 任务 1: 在 update-deps.yml 中插入补丁步骤

**文件:**
- 修改: `.github/workflows/update-deps.yml:273-287`（在 `update-mono-core` 步骤之后、`commit-bepinex` 步骤之前插入新步骤）

**上下文:** 现有文件第 273-287 行是 `update-mono-core` 步骤的结束，第 288 行开始是 `commit-bepinex` 步骤。新步骤插入在两者之间。

- [ ] **Step 1: 在 `update-mono-core` 步骤之后插入新步骤**

在现有第 286 行（`echo "Mono core files: ..."`）之后、第 287 行（`# ===阶段二：BepInEx 更新===` 分隔注释）之前，插入以下内容：

```yaml
      - name: Apply custom patches (Cpp2IL + Il2CppInterop)
        id: apply-custom-patches
        if: |
          steps.bepinex-check.outputs.bepinex_changed == 'true' &&
          steps.update-il2cpp-core.outcome == 'success'
        run: |
          echo "::group::Apply Cpp2IL custom patch"
          curl -sSL "https://github.com/abevol/Cpp2IL/releases/download/v2022.1.0-asmresolver-beta.5/Cpp2IL-net6.0-asmresolver-beta.5.zip" -o cpp2il-patch.zip
          unzip -qo cpp2il-patch.zip -d BIE6_IL2CPP/core/
          echo "Cpp2IL patch applied"
          echo "::endgroup::"

          echo "::group::Apply Il2CppInterop custom patch"
          curl -sSL "https://github.com/abevol/Il2CppInterop/releases/download/v1.5.1-improve-genericmethod-signatures/Il2CppInterop-v1.5.1-net6.0.zip" -o il2cppinterop-patch.zip
          unzip -qo il2cppinterop-patch.zip -d BIE6_IL2CPP/core/
          echo "Il2CppInterop patch applied"
          echo "::endgroup::"

          echo "Patches applied to BIE6_IL2CPP/core/"
```

**插入位置说明:** 新步骤放在第 286 行之后，即 `update-mono-core` 步骤的 `run` 块结束后。与 `commit-bepinex` 步骤之间保留一个空行。

- [ ] **Step 2: 提交变更**

```bash
git add .github/workflows/update-deps.yml
git commit -m "feat: add custom Cpp2IL and Il2CppInterop patches to BepInEx update workflow"
```

---

### 验证清单

- [ ] 新步骤的 `id` 为 `apply-custom-patches`，与其他步骤命名风格一致
- [ ] `if` 条件正确检查 `bepinex_changed == 'true'` 和 `update-il2cpp-core.outcome == 'success'`
- [ ] 不使用 `continue-on-error`（严格模式）
- [ ] `unzip -o` 标志确保覆盖同名文件
- [ ] `unzip -d BIE6_IL2CPP/core/` 确保解压到正确目录
- [ ] ZIP 文件名使用独立命名（`cpp2il-patch.zip` 和 `il2cppinterop-patch.zip`），避免与已有的 `il2cpp.zip` 冲突
- [ ] `commit-bepinex` 步骤的 `if` 条件**未修改**（不需要改）
