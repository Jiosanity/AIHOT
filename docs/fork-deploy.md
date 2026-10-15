# Fork 的镜像发布与上游同步

本 fork 为 `Jiosanity/AIHOT`，上游为 `KKKKhazix/AIHOT`。沿用上游的 Dockerfile：同一份镜像通过不同启动命令运行 `setup`、`api`、`worker` 和 `web`，PostgreSQL 与可选的 Caddy 仍使用官方镜像。

## 发布 GHCR 镜像

1. 在 fork 的 Actions 页面启用工作流。
2. 将发布配置合入并推送到 `main`。现有 `Check` 完整通过后，`Publish image` 才会发布同一提交的镜像；PR 检查不会发布。也可以在 Actions 中手动运行 `Check`，选择 `main`，通过后同样发布。
3. 镜像地址为 `ghcr.io/jiosanity/aihot`，包含 `linux/amd64` 和 `linux/arm64`。工作流从仓库名生成地址，迁移到其他 fork 时，需同步修改部署文件中的默认地址及 `.env.example`。
4. 每次发布生成 `sha-<完整提交 SHA>` 标签。只有该提交仍是 `main` 最新提交时，才更新 `latest`，避免旧检查的重跑把 `latest` 改回旧版。
5. 首次发布后，在 GitHub 个人主页的 **Packages → aihot → Package settings → Change visibility** 中设为 **Public**，需要匿名拉取时才做这一步。公开仓库并不意味着镜像自动公开。

发布使用工作流的 `GITHUB_TOKEN` 与 `packages: write` 权限，不需要另存发布 PAT。若同名包已存在但没有关联此仓库，先在包设置中关联仓库并授予 Actions 访问权限。

在未登录 GHCR 的环境中检查匿名拉取和平台清单：

```bash
docker buildx imagetools inspect ghcr.io/jiosanity/aihot:latest
docker pull ghcr.io/jiosanity/aihot:latest
```

发布规则见 [GitHub 的镜像发布说明](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)；默认私有与权限规则见 [Container registry 文档](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)。

## 用镜像部署

使用支持 [`!reset`](https://docs.docker.com/reference/compose-file/merge/#reset-value) 的新版 Docker Compose。`docker-compose.ghcr.yml` 只替换四个应用服务的镜像并移除本地构建，数据库、依赖顺序、端口、凭据隔离、worker 关闭等待时间和卷名均沿用原文件。

服务器至少需要 `docker-compose.yml`、`docker-compose.ghcr.yml` 和 `.env`；使用内置 HTTPS 时还需 `deploy/Caddyfile`。按照 [部署文档](deploy.md) 生成 `.env`，或从 `.env.example` 手工配置。

无需在服务器上安装项目依赖或构建源码。`site/`、`industry/`、模块和网页构建结果已包含在镜像中，修改它们后要发布新的镜像。在 `.env` 中设置 `SITE_URL`、密码、密钥和模型配置；首次试运行可将 `COLLECT_ENABLED`、`MODEL_CALLS_ENABLED` 设为 `false`，确认配置后再启用。

```bash
docker compose -f docker-compose.yml -f docker-compose.ghcr.yml pull
docker compose -f docker-compose.yml -f docker-compose.ghcr.yml up -d --no-build
```

打开 `http://服务器地址:3000`，后台在 `/admin`。如果镜像保持私有，先用具有 `read:packages` 权限的 PAT 通过 `docker login ghcr.io` 登录。HTTPS 部署在每条 Compose 命令中加 `--profile https`，其余配置沿用 [部署文档](deploy.md#配域名和-https)。

生产环境建议在 `.env` 中把所有应用服务固定到同一份已验证镜像，例如：

```dotenv
DEPLOY_IMAGE=ghcr.io/jiosanity/aihot:sha-<完整提交 SHA>
```

也可以使用 `ghcr.io/jiosanity/aihot@sha256:<镜像摘要>` 固定内容。`latest` 方便试用，但它会随发布变化。

### 更新现有部署

先阅读新版本的 [更新说明](deploy.md#更新)，备份数据库和附件，再更新部署文件与 `.env` 中的 `DEPLOY_IMAGE`。沿用原部署的目录、Compose 项目名和卷；原文件的默认项目名为 `aihot`，不要在迁移到镜像部署时另起项目而创建一套空卷。

先拉镜像，成功后停止旧应用并执行迁移：

```bash
docker compose -f docker-compose.yml -f docker-compose.ghcr.yml pull
docker compose -f docker-compose.yml -f docker-compose.ghcr.yml stop api worker web
docker compose -f docker-compose.yml -f docker-compose.ghcr.yml run --rm --no-build setup && docker compose -f docker-compose.yml -f docker-compose.ghcr.yml up -d --no-build
```

迁移失败就停在这里排查。不要用自动更新容器工具跳过停服务、迁移和备份。固定标签便于取回旧镜像，但数据库迁移可能不支持降级；需要回滚数据时使用升级前的备份。不要执行 `down -v`，它会删除持久卷。

## 持续跟进上游

建议采用：**每小时检查上游 → 有更新时创建或更新一条同步 PR → 人工处理冲突并审阅 → 检查通过后合入 main → 自动发布镜像 → 按上述步骤更新服务器**。目前仓库只配置了镜像发布，尚未启用定时同步或服务器自动更新。

上游频繁更新时，不宜用强制同步或 `reset --hard upstream/main` 覆盖自己的分支，否则镜像工作流、站点配置和行业定制也会被覆盖。用普通 merge 保留双方历史；`site/`、`industry/` 出现冲突时逐项决定，不要一律选择某一方。

### 手动同步

本地已经有 `origin` 和 `upstream` 两个 remote。工作区干净时，可以先建同步分支：

```bash
git fetch origin
git fetch upstream
git switch -c sync-upstream origin/main
git merge upstream/main
```

处理冲突、检查 `docs/deploy.md` 的升级说明并完成项目检查后，将该分支推送到自己的 fork：

```bash
git push -u origin sync-upstream
```

在 **Jiosanity/AIHOT** 中创建从 `sync-upstream` 到 `main` 的 PR。保留 merge 提交，用普通 merge 合并同步 PR，使上游提交继续作为 `main` 的祖先；不要 squash 或 rebase 同步 PR，避免后续重复同步相同的上游历史。GitHub 网页的 [Sync fork](https://docs.github.com/en/pull-requests/how-tos/work-with-forks/syncing-a-fork) 也可用于手动同步，但仍需处理冲突和升级说明。

### 定时监测怎么做

后续可在 fork 增加单独的同步工作流，按小时比较上游 `main` 与本仓库 `main` 的提交历史；没有新提交时结束，有更新时维护固定同步分支和一条 PR。冲突时让工作流失败并提供日志，人工解决后继续，不自动丢弃定制。

GitHub Actions 的定时检查属于轮询，可能排队延迟；公共仓库 60 天没有活动时，定时工作流可能自动停用，因此不能承诺严格实时。详见 [定时工作流规则](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)。真正由上游每次提交立即触发，需要上游维护者配合 webhook 或额外的外部监听服务。

实现自动同步时还需考虑 [工作流触发规则](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow)：使用默认 `GITHUB_TOKEN` 推送的提交不会再次触发 push 工作流。同步机器人创建 PR 后，应按平台要求批准工作流运行，或使用权限受限的 GitHub App / PAT 触发检查；不能假定机器人推送后检查和镜像发布一定会自动开始。
