# Ubuntu VPS + Neon 部署

本目录用于在单台 AMD64 Ubuntu VPS 上运行
`ghcr.io/xuweio/grok2api-n:latest`。数据库使用 Neon PostgreSQL，运行态使用
内存，媒体文件保存在 Docker Volume 中。HTTPS 配置不包含在本方案内。

## 1. 合并代码并等待镜像

合并部署改动到 `main` 后，仓库已有的 `GHCR Image` 工作流会执行测试并发布：

```text
ghcr.io/xuweio/grok2api-n:latest
```

在 GitHub 仓库的 Actions 页面确认工作流成功。GHCR 包首次创建后，可以把包
设为 Public；如果保持 Private，则在 VPS 上使用仅包含 `read:packages` 权限的
GitHub fine-grained token 登录：

```bash
echo 'YOUR_GITHUB_TOKEN' | docker login ghcr.io -u xuweio --password-stdin
```

不要把 token 放入仓库、Compose 文件或 shell 历史。

## 2. 安装 Docker

按照 Docker 官方文档为 Ubuntu 安装 Docker Engine 和 Compose plugin，然后确认：

```bash
docker version
docker compose version
```

## 3. 创建部署目录

```bash
sudo install -d -m 0750 /opt/grok2api-n
sudo chown "$USER":"$USER" /opt/grok2api-n
cd /opt/grok2api-n
```

从本仓库下载以下三个文件到该目录：

```text
deploy/vps/docker-compose.yml
deploy/vps/config.neon.example.yaml
deploy/vps/.env.example
```

然后创建仅 VPS 本地持有的配置：

```bash
cp config.neon.example.yaml config.yaml
cp .env.example .env
chmod 600 config.yaml .env
openssl rand -hex 32
openssl rand -base64 32
```

编辑 `config.yaml`，替换管理员密码、两个应用密钥和 Neon DSN。Neon DSN 必须
使用已经轮换过密码的新连接串，并保留 `sslmode=require`。应用不会展开 YAML
中的环境变量，所以不要把 `${DATABASE_URL}` 写入该文件。

## 4. 启动与验证

```bash
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100 grok2api
curl --fail http://127.0.0.1:8000/healthz
curl --fail http://127.0.0.1:8000/readyz
```

应用首次启动会自动在 Neon 中创建/迁移数据表。

HTTPS 尚未配置时，服务只监听 VPS 回环地址。可从本机建立 SSH 隧道：

```bash
ssh -L 8000:127.0.0.1:8000 VPS_USER@VPS_HOST
```

随后在本机访问 `http://127.0.0.1:8000`。首次登录后立即修改管理员密码，并从
VPS 的 `config.yaml` 中删除整个 `bootstrapAdmin` 配置段，再重启容器。

## 5. 升级与回滚

升级到 `main` 的最新镜像：

```bash
cd /opt/grok2api-n
docker compose pull
docker compose up -d
docker image prune -f
```

生产环境建议把 `.env` 中的镜像从 `latest` 改为一次已验证构建的 SHA/tag。回滚时
恢复上一镜像 tag，再运行 `docker compose up -d`。升级前应在 Neon 创建恢复点或
确认可用的备份，并备份 Docker Volume 中的媒体文件。

## 安全边界

- 不提交 `config.yaml`、Neon DSN、管理员密码、账号导出或 GitHub token。
- `credentialEncryptionKey` 写入账号后必须长期保存。
- HTTPS 完成前不要把 Compose 端口改成 `0.0.0.0`。
- 配置 HTTPS 后，将 `auth.secureCookies` 改为 `true`。
- Neon 保存业务数据和媒体元数据；图片、视频文件仍位于 VPS Docker Volume。
