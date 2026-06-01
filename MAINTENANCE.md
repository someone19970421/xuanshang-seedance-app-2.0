# GitHub Pages 热更新维护笔记

这个目录是 GitHub Pages 的发布仓库，远端为：

```powershell
git remote -v
```

当前正确维护方式：把已经生成好的构建产物提交到 `main`，由现有 GitHub Pages workflow 自动部署。不要删除或改动 `.github/workflows/static.yml`，否则 Pages 自动部署可能失效。

## 标准流程

1. 进入发布目录：

```powershell
cd "C:\Users\Powerful5090\Desktop\github项目\玄上V2 UI重构\dist"
```

2. 先确认仓库状态和远端：

```powershell
git status --short --branch
git remote -v
git fetch origin main
git rev-list --left-right --count HEAD...origin/main
```

`git rev-list` 输出应为 `0 0`，表示本地和远端同步。若不是，先处理同步问题，不要直接覆盖推送。

3. 检查 Pages workflow 是否还在：

```powershell
Test-Path .github/workflows/static.yml
git diff -- .github/workflows/static.yml
```

第一个命令应该输出 `True`，第二个命令应该没有输出。若 workflow 被构建过程删掉了，恢复它：

```powershell
git restore -- .github/workflows/static.yml
```

4. 检查 `index.html` 引用的新资源是否存在：

```powershell
Select-String -Path index.html -Pattern 'assets/[^"'']+' -AllMatches | ForEach-Object { $_.Matches.Value }
```

对输出的每个资源，确认对应文件在 `assets` 目录中存在。

5. 查看最终变更范围：

```powershell
git status --short --branch
git diff --name-status
git ls-files --others --exclude-standard
```

正常热更新通常只应该包含：

- `index.html` 中资源哈希引用更新
- `assets/` 下旧构建文件删除
- `assets/` 下新构建文件新增
- 必要时旧资源文件被 Git 识别为 rename

不应该包含：

- `.github/workflows/static.yml` 删除或修改
- `.gitignore`、仓库配置、部署配置等无关变更

6. 提交构建产物：

```powershell
git add -A
git commit -m "hotfix: update GitHub Pages build"
```

7. 推送到 GitHub：

```powershell
git push origin main
```

8. 推送后确认同步：

```powershell
git status --short --branch
git rev-list --left-right --count HEAD...origin/main
git log -1 --oneline
```

`git status` 应显示 `## main...origin/main`，`git rev-list` 应输出 `0 0`。

## 本次正确做法回顾

本次更新中，构建产物目录曾显示 `.github/workflows/static.yml` 被删除。正确处理是先执行：

```powershell
git restore -- .github/workflows/static.yml
```

然后只提交以下页面产物变化：

- `index.html` 改为引用新的 `assets/index-*.js`
- 新增新的构建 JS 文件
- 删除旧的构建 JS 文件
- `ParticleBackground` 文件因 hash 变化被识别为 rename

最终提交：

```text
476fabb hotfix: update GitHub Pages build
```

核心原则：发布仓库只做静态产物热更新，Pages workflow 是部署基础设施，除非明确要改部署方式，否则不要动它。
