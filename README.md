# 公司 CI 公用模板

这个仓库用于保存公司内部可复用的 GitHub Actions 工作流模板。

它的目标是让每个业务仓库只关心自己的构建命令，而把通用的发布动作统一放在这里维护，例如：

- 创建或更新 GitHub Release
- 上传 Release 附件
- 自动生成 SHA256 校验信息
- 保持所有项目的发版流程一致

推荐远程仓库：

```text
https://github.com/rockyliang10/CI
```

## 当前提供的模板

| 模板文件 | 用途 |
| --- | --- |
| `.github/workflows/reusable-release-assets.yml` | 把业务仓库上传的构建产物发布到 GitHub Release |

## 适用场景

这个模板适合这些项目：

- 需要通过 tag 自动发版的项目
- 需要稳定下载链接的项目
- 需要把 zip、exe、msi、tar.gz 等文件作为 Release 附件上传的项目
- 希望非技术用户只点击 `releases/latest/download/...` 下载的项目

例如：

```text
https://github.com/公司账号/项目名/releases/latest/download/xxx.zip
```

## 最推荐的使用方式

业务仓库自己保留一个很薄的 `.github/workflows/release.yml`。

这个文件只做两件事：

1. 构建本项目自己的产物。
2. 调用本仓库的公用发布模板。

示例：

```yaml
name: Release

on:
  push:
    tags:
      - "v*"
  workflow_dispatch:
    inputs:
      tag:
        description: "Existing v* tag to publish"
        required: true
        type: string

permissions:
  contents: write
  actions: read

jobs:
  build:
    name: Build release assets
    runs-on: windows-latest
    outputs:
      release-tag: ${{ steps.release-tag.outputs.value }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Select release tag
        id: release-tag
        shell: pwsh
        run: |
          if ("${{ github.event_name }}" -eq "workflow_dispatch") {
            $tag = "${{ inputs.tag }}"
          } else {
            $tag = "${env:GITHUB_REF_NAME}"
          }

          if ($tag -notmatch '^v[\w.-]+$') {
            throw "Release tag must start with v and contain only letters, digits, dots, underscores, or hyphens: $tag"
          }

          if ("${{ github.event_name }}" -eq "workflow_dispatch") {
            git fetch --tags --force
            git checkout "refs/tags/$tag"
          }

          "value=$tag" | Out-File -FilePath $env:GITHUB_OUTPUT -Append -Encoding utf8

      - name: Build packages
        shell: pwsh
        run: |
          # 改成当前项目自己的构建命令
          powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\build-release.ps1

      - name: Upload release assets
        uses: actions/upload-artifact@v4
        with:
          name: release-assets
          if-no-files-found: error
          path: |
            downloads/你的项目.zip

  publish:
    name: Publish GitHub Release
    needs: build
    uses: rockyliang10/CI/.github/workflows/reusable-release-assets.yml@v1
    with:
      tag: ${{ needs.build.outputs.release-tag }}
      artifact-name: release-assets
```

## 业务仓库需要改哪里

复制 `examples/release-caller.yml` 到业务仓库：

```text
.github/workflows/release.yml
```

然后只需要改两个地方。

第一处：构建命令。

```yaml
- name: Build packages
  shell: pwsh
  run: |
    powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\build-release.ps1
```

把它换成当前项目自己的构建方式。

第二处：上传产物路径。

```yaml
path: |
  downloads/你的项目.zip
```

把它换成当前项目实际生成的文件。

可以上传多个文件：

```yaml
path: |
  dist/app-windows.zip
  dist/app-macos.zip
  dist/app-linux.tar.gz
```

也可以上传子目录里的文件：

```yaml
path: |
  releases/app-windows.zip
  releases/app-macos.zip
  checksums-sha256.txt
```

公用模板会递归读取 artifact 里的所有文件，所以 `releases/` 这类子目录里的附件也会一起上传到 GitHub Release。

## 发版流程

业务仓库接入后，正常发版只需要这样做：

```powershell
git tag -a v1.0.1 -m "v1.0.1"
git push origin v1.0.1
```

GitHub Actions 会自动：

1. 检出对应 tag 的代码。
2. 运行业务仓库自己的构建命令。
3. 上传构建产物为临时 artifact。
4. 调用本仓库的公用发布模板。
5. 创建或更新 GitHub Release。
6. 上传 Release 附件。
7. 在 Release 说明里写入每个附件的 SHA256。

## 手动发版

如果需要手动重新发布某个已有 tag：

1. 打开业务仓库的 GitHub Actions 页面。
2. 选择 `Release` 工作流。
3. 点击 `Run workflow`。
4. 输入已有 tag，例如：

```text
v1.0.1
```

## 下载链接怎么写

如果 Release 附件名固定，README 里可以长期使用 latest 下载链接。

格式：

```text
https://github.com/公司账号/项目名/releases/latest/download/附件名
```

例如：

```text
https://github.com/rockyliang10/claude-code-link-tool/releases/latest/download/AquaClaudeCode-zh-CN.zip
```

只要每次 Release 上传的附件名保持不变，这个链接就会一直指向最新版本。

## 模板版本管理

业务仓库应该优先引用稳定版本：

```yaml
uses: rockyliang10/CI/.github/workflows/reusable-release-assets.yml@v1
```

不要长期引用：

```yaml
uses: rockyliang10/CI/.github/workflows/reusable-release-assets.yml@main
```

原因是 `main` 可能变化，所有业务仓库会被动受到影响。

推荐做法：

```powershell
git tag -a v1 -m "Reusable release workflow v1"
git push origin v1
```

以后如果模板有不兼容改动，再发布 `v2`。

## 权限要求

业务仓库的调用工作流需要：

```yaml
permissions:
  contents: write
  actions: read
```

含义：

- `contents: write`：允许创建或更新 GitHub Release。
- `actions: read`：允许读取构建 job 上传的 artifact。

通常不需要自己配置 GitHub token。默认使用 GitHub Actions 提供的 `GITHUB_TOKEN` 即可。

公用模板内部已经设置：

```yaml
GH_REPO: ${{ github.repository }}
```

这样即使发布 job 没有 checkout 代码，`gh release ...` 也能明确知道要发布到当前业务仓库。

## 安全规范

请遵守这些规则：

- 不要把 GitHub token 写进 workflow。
- 不要把 API Key 写进 README。
- 不要把私钥、`.pem`、`.key` 文件提交到仓库。
- 不要把本地用户配置文件提交到仓库。
- 如果业务仓库需要额外密钥，应使用 GitHub Secrets。

## 常见问题

### 为什么业务仓库还需要自己的 release.yml？

因为每个项目的构建方式不同。

公用模板只负责发布，不负责猜测每个项目怎么构建。

### 可以上传多个附件吗？

可以。

在业务仓库的 `Upload release assets` 里写多个路径即可。

如果附件在子目录中，也可以正常上传。模板会递归收集 artifact 内的文件。

### 如果 Release 已经存在会怎样？

模板会更新 Release 标题和说明，并用新的附件覆盖同名旧附件。

### 如果 tag 写错了怎么办？

模板只接受 `v*` 格式的 tag，例如：

```text
v1.0.0
v1.0.1-test
v2
```

不符合格式会直接失败，避免误发版。

### 私有仓库能用吗？

可以，但需要确认公司 GitHub 设置允许私有仓库调用这个模板仓库的 reusable workflow。

如果遇到权限问题，优先检查：

- 业务仓库 Actions 权限
- 模板仓库 Actions 权限
- Organization 或账号级别的 Actions policy

### 发布 job 提示找不到目标仓库怎么办？

当前 `v1` 模板已经设置 `GH_REPO`，正常不会出现这个问题。

如果业务仓库仍然失败，请确认它引用的是：

```yaml
uses: rockyliang10/CI/.github/workflows/reusable-release-assets.yml@v1
```

不要引用旧提交。

### 为什么 Release 里只看到 checksums 文件，没有 zip？

通常是因为旧模板只扫描 artifact 根目录文件。

当前 `v1` 模板已经改为递归扫描 artifact，所以类似下面的结构可以正常发布：

```text
release-assets/
  checksums-sha256.txt
  releases/
    app-windows.zip
```
