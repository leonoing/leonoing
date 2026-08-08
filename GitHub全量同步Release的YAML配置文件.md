# <p align="center">GitHub全量同步Release的YAML配置文件</p>
## 使用 GitHub Actions 自动同步
通过编写一个定时执行的 GitHub Actions 工作流，利用脚本检测上游仓库（Upstream）的最新 Release，并自动在自己的仓库中创建对应的 Release。<br>
### 在你的仓库中创建文件
`.github/workflows/sync-release.yml`
### 同步所有Release的YAML配置代码文件
```yaml
name: Sync Upstream Release

on:
  schedule:
    - cron: '0 */3 * * *'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  sync-all-releases:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Sync All Releases from Upstream
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          UPSTREAM_REPO: "xxx/xxxx" #修改为上游仓库，确认是否为原作者/仓库名.例如 "cli/cli"
          TARGET_REPO: ${{ github.repository }}
        run: |
          # 1. 获取上游仓库所有的 Tag（倒序排列，优先从旧到新同步，保证最新版标签显示正确）
          ALL_TAGS=$(gh api "repos/$UPSTREAM_REPO/releases?per_page=100" --jq '.[].tag_name' | tac)

          if [ -z "$ALL_TAGS" ]; then
            echo "未检测到上游 Release 列表"
            exit 0
          fi

          # 2. 遍历每一个 Release 版本
          for TAG in $ALL_TAGS; do
            # 检查自己仓库是否已存在该 Release
            if gh release view "$TAG" --repo "$TARGET_REPO" >/dev/null 2>&1; then
              echo "Release $TAG 已存在，跳过。"
              continue
            fi

            echo "正在同步历史版本: $TAG ..."

            # 创建临时下载目录
            DIR_NAME="assets_$TAG"
            mkdir -p "$DIR_NAME" && cd "$DIR_NAME"

            # 下载上游 Release 资产与 Release Notes
            gh release download "$TAG" --repo "$UPSTREAM_REPO" --dir . || true

            TITLE=$(gh api "repos/$UPSTREAM_REPO/releases/tags/$TAG" --jq '.name // .tag_name')
            gh api "repos/$UPSTREAM_REPO/releases/tags/$TAG" --jq '.body // ""' > release_notes.md

            # 筛选文件并发布
            FILES=( $(ls -A | grep -v "^release_notes.md$") )

            if [ ${#FILES[@]} -gt 0 ]; then
              gh release create "$TAG" "${FILES[@]}" \
                --repo "$TARGET_REPO" \
                --title "$TITLE" \
                --notes-file release_notes.md
            else
              gh release create "$TAG" \
                --repo "$TARGET_REPO" \
                --title "$TITLE" \
                --notes-file release_notes.md
            fi

            # 清理临时文件并返回上一级
            cd .. && rm -rf "$DIR_NAME"
            echo "成功同步版本: $TAG"
          done
```
### 只同步最新Release的YAML配置代码文件
```yaml
name: Sync Upstream Releases

on:
  schedule:
    - cron: '0 * * * *' # 每小时检查一次
  workflow_dispatch: # 支持手动触发

permissions:
  contents: write # 赋予创建 Release 的权限

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Sync Releases from Upstream
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          UPSTREAM_REPO: "xxx/xxxx" #修改为上游仓库，确认是否为原作者/仓库名.例如 "cli/cli"
          TARGET_REPO: ${{ github.repository }}
        run: |
          LATEST_TAG=$(gh api repos/$UPSTREAM_REPO/releases/latest --jq '.tag_name' 2>/dev/null || echo "")

          if [ -z "$LATEST_TAG" ]; then
            echo "未找到上游 Release"
            exit 0
          fi

          if gh release view "$LATEST_TAG" --repo "$TARGET_REPO" >/dev/null 2>&1; then
            echo "Release $LATEST_TAG 已存在，跳过。"
            exit 0
          fi

          mkdir -p sync_assets && cd sync_assets
          gh release download "$LATEST_TAG" --repo "$UPSTREAM_REPO" --dir . || true

          TITLE=$(gh api repos/$UPSTREAM_REPO/releases/tags/$LATEST_TAG --jq '.name // .tag_name')
          gh api repos/$UPSTREAM_REPO/releases/tags/$LATEST_TAG --jq '.body // ""' > release_notes.md

          FILES=( $(ls -A | grep -v "^release_notes.md$") )

          if [ ${#FILES[@]} -gt 0 ]; then
            gh release create "$LATEST_TAG" "${FILES[@]}" --repo "$TARGET_REPO" --title "$TITLE" --notes-file release_notes.md
          else
            gh release create "$LATEST_TAG" --repo "$TARGET_REPO" --title "$TITLE" --notes-file release_notes.md
          fi
```
