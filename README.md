# image-registry

个人 Docker 镜像仓库的入口仓库：记录镜像命名约定、推送与拉取方式，并作为所有自建镜像在 GitHub 上的归属仓库。

## 这不是一个 registry 服务

GitHub 账号本身自带容器镜像服务 **GHCR**（GitHub Container Registry，`ghcr.io`），不需要也无法通过"建一个 git 仓库"来创建。自建 registry（Harbor / Zot / registry:2）的路线已于 2026-11-10 明确放弃，镜像统一走 GHCR。本仓库的作用是：

- 作为自建镜像的**约定与文档入口**；
- 作为 GHCR 上各 package 的**归属仓库**（通过镜像 label `org.opencontainers.image.source` 关联，这样 package 会显示在本仓库的 Packages 区域）；
- 存放构建这些镜像的 Dockerfile 与 CI 工作流（后续按需添加）。

## 镜像命名约定

```
ghcr.io/arkmusn/<image>:<tag>
```

- `<image>`：镜像名，建议与对应服务或目录同名，全小写，用 `-` 分隔（例：`ghcr.io/arkmusn/multica-cli`）。
- `<tag>`：不放 `latest` 单标签，至少给出版本号或日期标签，便于回滚（例：`1.2.0`、`2026-11-10`）。

构建时为镜像加上来源标签，GHCR 才会把 package 关联到本仓库：

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/Arkmusn/image-registry"
```

## 推送镜像

需要一个带 `write:packages` 权限的 GitHub PAT（classic token 勾选 `write:packages`、`read:packages`，必要时加 `repo`）：

```bash
echo "$GITHUB_TOKEN" | docker login ghcr.io -u Arkmusn --password-stdin
docker build -t ghcr.io/arkmusn/<image>:<tag> .
docker push ghcr.io/arkmusn/<image>:<tag>
```

## 拉取镜像

package 是 public，直接拉取，不需要登录、不需要 imagePullSecrets：

```bash
docker pull ghcr.io/arkmusn/<image>:<tag>
```

k3s / Kubernetes 直接写镜像地址即可。只有拉私有 package 时才需要 `read:packages` 凭据，并在 Pod 上配 `imagePullSecrets`（`type: kubernetes.io/dockerconfigjson`，由 `docker login` 生成的 `~/.docker/config.json` 转换而来，不提交到任何仓库）。

## 可见性

仓库是 **public**。公开镜像免认证拉取，是这里最关键的一条——私有 package 会让每个拉取点都要配凭据，不值得。

代价是镜像内容对全网可见，所以**镜像里不允许出现密钥、token、私有配置**；需要保密的构建产物不要发到这里。

注意仓库与 package 的可见性是两套设置：首次 push 之后到 package 的 settings 里确认它确实是 public，如果默认成了 private 要单独改。

## 后续可能的内容

- 各服务 Dockerfile 的目录结构；
- GitHub Actions 构建并推送 workflow；
- 镜像清理策略（GHCR 不自动清理旧版本，需要手动或 workflow 定期清理）。