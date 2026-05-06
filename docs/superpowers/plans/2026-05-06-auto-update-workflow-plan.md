# Auto-Update GitHub Workflow 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建 GitHub Actions workflow，每天自动检测游戏版本和 BepInEx 前沿构建更新，下载处理后提交到仓库。

**Architecture:** 单 workflow 文件 `.github/workflows/update-deps.yml`，ubuntu-latest runner，分两个顺序阶段（游戏更新 → BepInEx 更新），每阶段独立检测、下载、提交。组件步骤使用 `continue-on-error` 容错。

**Tech Stack:** GitHub Actions, bash, DepotDownloader, il2cpp-interop-cli, jq, curl

---

### Task 1: 创建文件列表和初始状态文件

**Files:**
- Create: `linux-files.txt`
- Create: `win-files.txt`
- Create: `.update-state.json`
- Create: `.github/workflows/` (directory)

- [ ] **Step 1: 创建 linux-files.txt**

```bash
cat > linux-files.txt <<'EOF'
regex:^KingdomTwoCrowns_Data/Managed/.*$
EOF
```

- [ ] **Step 2: 创建 win-files.txt**

```bash
cat > win-files.txt <<'EOF'
KingdomTwoCrowns.exe
GameAssembly.dll
KingdomTwoCrowns_Data/globalgamemanagers
KingdomTwoCrowns_Data/il2cpp_data/Metadata/global-metadata.dat
EOF
```

- [ ] **Step 3: 创建初始状态文件**

使用占位值。首次运行时检测到差异会自动触发完整更新并回填正确值。

```bash
cat > .update-state.json <<'EOF'
{
  "game": {
    "linuxManifestId": "0",
    "windowsManifestId": "0",
    "updatedAt": ""
  },
  "bepinex": {
    "buildId": "0",
    "commitHash": "",
    "updatedAt": ""
  }
}
EOF
```

- [ ] **Step 4: 创建 workflows 目录**

```bash
mkdir -p .github/workflows
```

- [ ] **Step 5: 提交**

```bash
git add linux-files.txt win-files.txt .update-state.json .github/
git commit -m "$(cat <<'EOF'
chore: 添加 DepotDownloader 文件列表和初始状态文件
EOF
)"
```

---

### Task 2: 创建 Workflow YAML 文件

**Files:**
- Create: `.github/workflows/update-deps.yml`

- [ ] **Step 1: 写入完整 workflow 文件**

```bash
cat > .github/workflows/update-deps.yml <<'YAMLEOF'
name: Update Dependencies

on:
  schedule:
    - cron: '0 18 * * *'
  workflow_dispatch:

jobs:
  update:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      # ============================================================
      # 环境准备
      # ============================================================

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup .NET 8.0
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Download DepotDownloader
        run: |
          curl -sSL -o depotdownloader.zip \
            "https://github.com/SteamRE/DepotDownloader/releases/download/DepotDownloader_3.4.0/DepotDownloader-linux-x64.zip"
          unzip -q depotdownloader.zip -d depotdownloader
          chmod +x depotdownloader/DepotDownloader

      - name: Download il2cpp-interop-cli
        run: |
          curl -sSL -o il2cpp-cli.tar.gz \
            "https://github.com/abevol/BepInEx-CLI/releases/download/v0.1.0/il2cpp-interop-cli-v0.1.0-linux-x64.tar.gz"
          mkdir -p il2cpp-cli
          tar -xzf il2cpp-cli.tar.gz -C il2cpp-cli
          chmod +x il2cpp-cli/il2cpp-interop-cli

      # ============================================================
      # 阶段一：游戏更新
      # ============================================================

      - name: Check game manifest
        id: game-check
        run: |
          LINUX_OUTPUT=$(./depotdownloader/DepotDownloader \
            -app 701160 -depot 701163 -manifest-only \
            -os linux -osarch 64 \
            -username "$STEAM_ACC" -password "$STEAM_PWD" -remember-password 2>&1)
          echo "::group::Linux depot output"
          echo "$LINUX_OUTPUT"
          echo "::endgroup::"

          WINDOWS_OUTPUT=$(./depotdownloader/DepotDownloader \
            -app 701160 -depot 701163 -manifest-only \
            -os windows -osarch 64 \
            -username "$STEAM_ACC" -password "$STEAM_PWD" -remember-password 2>&1)
          echo "::group::Windows depot output"
          echo "$WINDOWS_OUTPUT"
          echo "::endgroup::"

          LINUX_MANIFEST=$(echo "$LINUX_OUTPUT" | grep -oP 'Manifest \K\d+')
          WINDOWS_MANIFEST=$(echo "$WINDOWS_OUTPUT" | grep -oP 'Manifest \K\d+')
          LINUX_TIME=$(echo "$LINUX_OUTPUT" | grep -oP 'Manifest \d+ \(\K[^)]+')

          echo "linux_manifest=$LINUX_MANIFEST" >> $GITHUB_OUTPUT
          echo "windows_manifest=$WINDOWS_MANIFEST" >> $GITHUB_OUTPUT
          echo "linux_time=$LINUX_TIME" >> $GITHUB_OUTPUT

          OLD_LINUX=$(jq -r '.game.linuxManifestId' .update-state.json)
          OLD_WINDOWS=$(jq -r '.game.windowsManifestId' .update-state.json)

          echo "Old Linux manifest: $OLD_LINUX"
          echo "New Linux manifest: $LINUX_MANIFEST"
          echo "Old Windows manifest: $OLD_WINDOWS"
          echo "New Windows manifest: $WINDOWS_MANIFEST"

          if [ "$LINUX_MANIFEST" != "$OLD_LINUX" ] || [ "$WINDOWS_MANIFEST" != "$OLD_WINDOWS" ]; then
            echo "game_changed=true" >> $GITHUB_OUTPUT
          else
            echo "game_changed=false" >> $GITHUB_OUTPUT
          fi
        env:
          STEAM_ACC: ${{ secrets.STEAM_ACC }}
          STEAM_PWD: ${{ secrets.STEAM_PWD }}

      - name: Download Linux Managed files
        id: download-managed
        if: steps.game-check.outputs.game_changed == 'true'
        continue-on-error: true
        run: |
          ./depotdownloader/DepotDownloader \
            -app 701160 -depot 701163 \
            -manifest ${{ steps.game-check.outputs.linux_manifest }} \
            -filelist linux-files.txt \
            -os linux -osarch 64 \
            -username "$STEAM_ACC" -password "$STEAM_PWD" -remember-password

          MANAGED_SRC=$(find depots/701163 -name "Accessibility.dll" -path "*/Managed/*" | head -1 | xargs dirname)
          echo "Managed source: $MANAGED_SRC"

          rm -rf BIE6_Mono/Managed/*
          cp -r "$MANAGED_SRC"/* BIE6_Mono/Managed/

          echo "Managed files copied: $(ls BIE6_Mono/Managed/ | wc -l) files"
        env:
          STEAM_ACC: ${{ secrets.STEAM_ACC }}
          STEAM_PWD: ${{ secrets.STEAM_PWD }}

      - name: Download Windows files and generate interop
        id: generate-interop
        if: steps.game-check.outputs.game_changed == 'true'
        continue-on-error: true
        run: |
          ./depotdownloader/DepotDownloader \
            -app 701160 -depot 701163 \
            -manifest ${{ steps.game-check.outputs.windows_manifest }} \
            -filelist win-files.txt \
            -os windows -osarch 64 \
            -username "$STEAM_ACC" -password "$STEAM_PWD" -remember-password

          GAME_ROOT=$(find depots/701163 -name "KingdomTwoCrowns.exe" -type f | head -1 | xargs dirname)
          echo "Game root: $GAME_ROOT"

          rm -rf BIE6_IL2CPP/interop/*
          ./il2cpp-cli/il2cpp-interop-cli \
            -g "$GAME_ROOT" \
            -m KingdomTwoCrowns \
            -o BIE6_IL2CPP/interop/

          echo "Interop files generated: $(ls BIE6_IL2CPP/interop/ | wc -l) files"
        env:
          STEAM_ACC: ${{ secrets.STEAM_ACC }}
          STEAM_PWD: ${{ secrets.STEAM_PWD }}

      - name: Commit game updates
        id: commit-game
        if: |
          steps.game-check.outputs.game_changed == 'true' &&
          (steps.download-managed.outcome == 'success' || steps.generate-interop.outcome == 'success')
        run: |
          # 更新状态文件
          jq --arg lm "${{ steps.game-check.outputs.linux_manifest }}" \
             --arg wm "${{ steps.game-check.outputs.windows_manifest }}" \
             --arg lt "${{ steps.game-check.outputs.linux_time }}" \
             '.game.linuxManifestId = $lm | .game.windowsManifestId = $wm | .game.updatedAt = $lt' \
             .update-state.json > .update-state.tmp && mv .update-state.tmp .update-state.json

          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          git add BIE6_Mono/Managed/ BIE6_IL2CPP/interop/ .update-state.json

          if git diff --quiet && git diff --cached --quiet; then
            echo "No game changes to commit"
            exit 0
          fi

          LINUX_ID="${{ steps.game-check.outputs.linux_manifest }}"
          WINDOWS_ID="${{ steps.game-check.outputs.windows_manifest }}"
          git commit -m "chore: update game deps to manifest linux/${LINUX_ID}, win/${WINDOWS_ID}"

          git push

      # ============================================================
      # 阶段二：BepInEx 更新
      # ============================================================

      - name: Check BepInEx build
        id: bepinex-check
        run: |
          HTML=$(curl -sSL "https://builds.bepinex.dev/projects/bepinex_be")

          BUILD_ID=$(echo "$HTML" | grep -oP '<span class="artifact-id">#\K\d+' | head -1)
          COMMIT_HASH=$(echo "$HTML" | grep -oP '<a class="hash-button" href="[^"]+">\K[^<]+' | head -1)
          BUILD_DATE=$(echo "$HTML" | grep -oP '<span class="build-date">\K[^<]+' | head -1)

          echo "build_id=$BUILD_ID" >> $GITHUB_OUTPUT
          echo "commit_hash=$COMMIT_HASH" >> $GITHUB_OUTPUT
          echo "build_date=$BUILD_DATE" >> $GITHUB_OUTPUT

          # 提取 IL2CPP-win-x64 和 Mono-linux-x64 的下载链接
          IL2CPP_PATH=$(echo "$HTML" | grep -oP '<a class="artifact-link"\s+href="\K[^"]*IL2CPP-win-x64[^"]*' | head -1)
          MONO_PATH=$(echo "$HTML" | grep -oP '<a class="artifact-link"\s+href="\K[^"]*Mono-linux-x64[^"]*' | head -1)

          echo "il2cpp_url=https://builds.bepinex.dev${IL2CPP_PATH}" >> $GITHUB_OUTPUT
          echo "mono_url=https://builds.bepinex.dev${MONO_PATH}" >> $GITHUB_OUTPUT

          # 提取 changelog
          CHANGELOG=$(echo "$HTML" | grep -oP '<div class="changelog">.*?</div>' | head -1 | grep -oP '<li>\K[^<]*' | sed 's/<[^>]*>//g')
          echo "changelog<<CHANGELOG_EOF" >> $GITHUB_OUTPUT
          echo "$CHANGELOG" >> $GITHUB_OUTPUT
          echo "CHANGELOG_EOF" >> $GITHUB_OUTPUT

          OLD_BUILD=$(jq -r '.bepinex.buildId' .update-state.json)
          echo "Old build: $OLD_BUILD"
          echo "New build: $BUILD_ID"

          if [ "$BUILD_ID" != "$OLD_BUILD" ] && [ -n "$BUILD_ID" ]; then
            echo "bepinex_changed=true" >> $GITHUB_OUTPUT
          else
            echo "bepinex_changed=false" >> $GITHUB_OUTPUT
          fi

      - name: Download and update IL2CPP core
        id: update-il2cpp-core
        if: steps.bepinex-check.outputs.bepinex_changed == 'true'
        continue-on-error: true
        run: |
          curl -sSL "${{ steps.bepinex-check.outputs.il2cpp_url }}" -o il2cpp.zip
          mkdir -p il2cpp-extract
          unzip -qo il2cpp.zip "BepInEx/core/*" -d il2cpp-extract/

          rm -rf BIE6_IL2CPP/core/*
          cp -r il2cpp-extract/BepInEx/core/* BIE6_IL2CPP/core/

          echo "IL2CPP core files: $(ls BIE6_IL2CPP/core/ | wc -l) files"

      - name: Download and update Mono core
        id: update-mono-core
        if: steps.bepinex-check.outputs.bepinex_changed == 'true'
        continue-on-error: true
        run: |
          curl -sSL "${{ steps.bepinex-check.outputs.mono_url }}" -o mono.zip
          mkdir -p mono-extract
          unzip -qo mono.zip "BepInEx/core/*" -d mono-extract/

          rm -rf BIE6_Mono/core/*
          cp -r mono-extract/BepInEx/core/* BIE6_Mono/core/

          echo "Mono core files: $(ls BIE6_Mono/core/ | wc -l) files"

      - name: Commit BepInEx updates
        id: commit-bepinex
        if: |
          steps.bepinex-check.outputs.bepinex_changed == 'true' &&
          (steps.update-il2cpp-core.outcome == 'success' || steps.update-mono-core.outcome == 'success')
        run: |
          # 更新状态文件
          jq --arg id "${{ steps.bepinex-check.outputs.build_id }}" \
             --arg hash "${{ steps.bepinex-check.outputs.commit_hash }}" \
             --arg date "${{ steps.bepinex-check.outputs.build_date }}" \
             '.bepinex.buildId = $id | .bepinex.commitHash = $hash | .bepinex.updatedAt = $date' \
             .update-state.json > .update-state.tmp && mv .update-state.tmp .update-state.json

          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          git add BIE6_IL2CPP/core/ BIE6_Mono/core/ .update-state.json

          if git diff --quiet && git diff --cached --quiet; then
            echo "No BepInEx changes to commit"
            exit 0
          fi

          BUILD_ID="${{ steps.bepinex-check.outputs.build_id }}"
          COMMIT_HASH="${{ steps.bepinex-check.outputs.commit_hash }}"
          BUILD_DATE="${{ steps.bepinex-check.outputs.build_date }}"

          git commit -m "$(printf 'chore: update BepInEx BE to #%s (%s)\n\n- Build date: %s\n- %s' \
            "$BUILD_ID" "$COMMIT_HASH" "$BUILD_DATE" "${{ steps.bepinex-check.outputs.changelog }}")"

          git push
YAMLEOF
```

- [ ] **Step 2: 验证 YAML 语法**

```bash
# 使用 Python 验证 YAML 语法（无需额外安装）
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/update-deps.yml'))" 2>/dev/null || \
  echo "PyYAML not installed, checking with basic parsing..." && \
  cat .github/workflows/update-deps.yml | grep -n "continue-on-error" && \
  echo "---" && \
  cat .github/workflows/update-deps.yml | grep -n "^\s\+- name:"
```

- [ ] **Step 3: 提交 workflow 文件**

```bash
git add .github/workflows/update-deps.yml
git commit -m "$(cat <<'EOF'
feat: 添加自动更新依赖 GitHub Actions workflow

每天 UTC 18:00 自动检测游戏版本和 BepInEx BE 构建更新，
下载处理后提交到仓库。
EOF
)"
```

---

### Task 3: 验证与收尾

- [ ] **Step 1: 检查所有文件已正确创建**

```bash
echo "=== 新建文件 ==="
ls -la linux-files.txt win-files.txt .update-state.json
ls -la .github/workflows/update-deps.yml
echo ""
echo "=== Git log ==="
git log --oneline -5
echo ""
echo "=== Workflow 步骤列表 ==="
grep -n "^\s\+- name:" .github/workflows/update-deps.yml
```

- [ ] **Step 2: 确认所有文件已提交且工作区干净**

```bash
git status
```

预期输出：`nothing to commit, working tree clean`

- [ ] **Step 3: 提醒用户配置 GitHub Secrets**

确认以下 secrets 已在仓库 `Settings → Secrets and variables → Actions` 中配置：
- `STEAM_ACC`：Steam 账号用户名
- `STEAM_PWD`：Steam 账号密码

并确认 Steam 账号已移除邮箱验证，确保 DepotDownloader 可在无头环境直接登录。
