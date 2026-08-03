# Sub2API 自定义版发版教程

本文适用于当前项目：

- 本地目录：`C:\Users\78139\Documents\Codex\2026-07-27\1\sub2api`
- 自定义分支：`custom-ui`
- 官方远程：`upstream`，指向 `Wei-Shaw/sub2api`
- 自有 GitHub 远程：`github`，指向 `yinyiming1223/yym_api`
- GitHub Actions：`Build Custom Image`
- 镜像：`ghcr.io/yinyiming1223/yym_api:custom-ui`
- 服务器目录：`/opt/sub2api`
- 站点：`https://www.yinyiming.me`

整个过程是：

```text
官方发布新版本
  -> 本地把官方版本合并到 custom-ui
  -> 推送 custom-ui 到自己的 GitHub
  -> GitHub Actions 构建并推送 custom-ui 镜像
  -> 服务器拉取新镜像并只重建 sub2api 容器
```

## 一、几个重要原则

1. 页面提示“有新版本”只代表官方发布了新版，不代表你的自定义版已经自动升级。
2. 你的版本包含自定义前端，不能直接在服务器拉取 `weishaw/sub2api:latest`，否则自定义内容会消失。
3. 更新应用容器不会主动删除 PostgreSQL 数据，但发版前仍应手动做一次数据库备份，以防新版本迁移数据库后需要恢复。
4. 不要执行 `docker compose down -v`。其中 `-v` 会删除数据库等 Docker 数据卷。
5. 不要删除或覆盖服务器上的 `/opt/sub2api/.env`。
6. 下面标注“本地 PowerShell”的命令在你的 Windows 电脑执行；标注“服务器”的命令在 MobaXterm 的 SSH 终端执行。

## 二、日常发版速查

### 1. 本地检查并获取官方版本

在 Windows PowerShell 执行：

```powershell
cd 'C:\Users\78139\Documents\Codex\2026-07-27\1\sub2api'
git switch custom-ui
git status --short --branch
git fetch upstream --prune --tags
$release = git tag --merged upstream/main --sort=-version:refname | Select-Object -First 1
Write-Host "准备合并官方版本：$release"
git describe --tags --abbrev=0 HEAD
```

说明：

- `$release` 是当前官方 `main` 分支中最新的正式版本，例如 `v0.1.170`。
- 最后一条命令显示你的自定义分支当前基于哪个官方版本。
- 如果两个版本相同，说明正式版本代码已经合并，不需要重复合并。
- `git status` 必须是干净状态。若显示有未提交文件，先确认并提交自己的修改，不要直接覆盖。

### 2. 建立更新前保护分支

```powershell
$backup = 'backup-before-update-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
git branch $backup
Write-Host "保护分支：$backup"
```

这只是一个本地保护点。即使合并结果不满意，也可以找到更新前的代码。

### 3. 合并官方正式版本

```powershell
git merge --no-edit $release
```

如果没有冲突，Git 会直接完成合并。然后检查自定义版相对官方版还保留了哪些修改：

```powershell
git status --short --branch
git diff --stat "$release..HEAD"
git log --oneline --decorate -8
```

如果出现 `CONFLICT`，不要继续推送。先在 IDE 中逐个解决冲突：既保留自己的品牌和界面改动，也接入官方的新结构。解决后执行：

```powershell
git status
git add <已经解决的文件>
git commit -m "merge: sync official $release"
```

如果暂时不会处理冲突，可以完全取消本次合并：

```powershell
git merge --abort
```

不要对所有冲突文件统一使用 `ours` 或 `theirs`，这样很容易一次性丢掉官方更新或自己的前端。

### 4. 推送并触发镜像构建

```powershell
git push github custom-ui
```

不需要执行 `git push --tags`。自定义镜像工作流会直接读取官方仓库的版本标签；推送 tag 反而可能触发仓库中不需要的 `Release` 工作流。

打开 GitHub 仓库：

```text
https://github.com/yinyiming1223/yym_api/actions
```

找到 `Build Custom Image`，必须等它显示绿色成功。成功后会生成两个镜像标签：

```text
ghcr.io/yinyiming1223/yym_api:custom-ui
ghcr.io/yinyiming1223/yym_api:sha-本次完整提交号
```

如果 Actions 失败，不要更新服务器，先打开失败步骤查看日志。

### 5. 发版前备份数据

进入 Sub2API 管理后台：

```text
系统设置 -> 数据备份 -> 立即备份
```

确认备份成功并已存到服务器之外的对象存储。镜像更新不会替代数据库备份。

### 6. 服务器拉取并发布

GitHub Actions 成功后，用 MobaXterm 登录服务器，执行：

```bash
cd /opt/sub2api
docker compose ps
docker compose config --images | grep 'ghcr.io/yinyiming1223/yym_api'
```

必须能看到：

```text
ghcr.io/yinyiming1223/yym_api:custom-ui
```

如果看不到，先停止操作，说明 Compose 没有使用你的自定义镜像。

为当前旧镜像建立本地回滚标签：

```bash
docker image inspect ghcr.io/yinyiming1223/yym_api:custom-ui >/dev/null 2>&1 && \
docker tag ghcr.io/yinyiming1223/yym_api:custom-ui ghcr.io/yinyiming1223/yym_api:rollback
```

拉取新镜像并只重建应用容器：

```bash
docker compose pull sub2api
docker compose up -d --no-deps --force-recreate sub2api
```

这里不会重建 PostgreSQL 和 Redis，也不会删除数据卷。

### 7. 验证发布结果

```bash
cd /opt/sub2api
docker compose ps
curl -fsS http://127.0.0.1:8080/health; echo
docker compose logs --tail=100 sub2api
```

正确结果应满足：

- `sub2api` 状态为 `Up` 或 `healthy`。
- 健康检查返回 `{"status":"ok"}`。
- 最近日志没有持续重复的数据库迁移错误或启动失败。

然后打开 `https://www.yinyiming.me`，按 `Ctrl+F5` 强制刷新，再查看版本号和自定义前端是否正常。最后实际发送一个短请求，确认中转链路可用。

## 三、出现问题时回滚

如果新容器无法启动，先查看日志：

```bash
cd /opt/sub2api
docker compose logs --tail=200 sub2api
```

需要恢复到发版前镜像时执行：

```bash
cd /opt/sub2api
docker image inspect ghcr.io/yinyiming1223/yym_api:rollback >/dev/null
docker tag ghcr.io/yinyiming1223/yym_api:rollback ghcr.io/yinyiming1223/yym_api:custom-ui
docker compose up -d --no-deps --force-recreate sub2api
curl -fsS http://127.0.0.1:8080/health; echo
```

回滚期间不要再次执行 `docker compose pull sub2api`，否则 `custom-ui` 会再次被远程新镜像覆盖。

注意：镜像回滚不会回滚数据库。如果新版已经执行了不兼容的数据库迁移，还需要使用发版前创建的数据库备份恢复。因此每次发版前的手动备份不能省略。

## 四、常见问题排查

### 页面仍显示旧版本

先确认 GitHub Actions 已经成功，然后在服务器执行：

```bash
docker inspect sub2api --format '{{.Image}}'
docker image inspect ghcr.io/yinyiming1223/yym_api:custom-ui --format '{{.Id}}'
```

两条命令应输出相同的镜像 ID。若不同，重新执行：

```bash
docker compose up -d --no-deps --force-recreate sub2api
```

之后在浏览器按 `Ctrl+F5`。

### Actions 构建失败

常见原因包括：

- 合并冲突没有完全解决。
- 官方升级后前端类型或组件接口变化，自定义代码需要同步调整。
- 官方升级了依赖锁文件，而冲突处理时保留了错误版本。
- GitHub 或 GHCR 暂时网络异常，可以在 Actions 中重新运行失败任务。

不要在 Actions 失败时继续到服务器执行发布命令。

### 容器启动后网站报 502

```bash
cd /opt/sub2api
docker compose ps
docker compose logs --tail=200 sub2api
curl -v http://127.0.0.1:8080/health
```

如果本机健康检查失败，问题在 Sub2API 容器；如果本机正常但域名失败，再检查 Caddy：

```bash
systemctl status caddy --no-pager
journalctl -u caddy -n 100 --no-pager
```

## 五、最短命令清单

确认没有冲突时，本地 PowerShell 的核心操作是：

```powershell
cd 'C:\Users\78139\Documents\Codex\2026-07-27\1\sub2api'
git switch custom-ui
git status --short --branch
git fetch upstream --prune --tags
$release = git tag --merged upstream/main --sort=-version:refname | Select-Object -First 1
git merge --no-edit $release
git push github custom-ui
```

等 GitHub Actions 变绿，管理后台手动备份数据库，然后服务器执行：

```bash
cd /opt/sub2api
docker tag ghcr.io/yinyiming1223/yym_api:custom-ui ghcr.io/yinyiming1223/yym_api:rollback
docker compose pull sub2api
docker compose up -d --no-deps --force-recreate sub2api
curl -fsS http://127.0.0.1:8080/health; echo
docker compose ps
```
