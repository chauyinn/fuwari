---
title: Gitea Actions Runner 在开启登录限制时如何拉取本地 Actions
published: 2025-10-13
updated: 2025-10-13
description: '在私有化部署环境下，如何让 Gitea Actions Runner 拉取本地自定义 Actions，兼顾部分安全性与可用性'
image: ''
tags: ["Gitea", "CI/CD", "Actions", "Runner"]
category: '自动化工具'
draft: false
lang: 'zh-cn'
---

## 前言

[Gitea](https://docs.gitea.com/zh-cn/) 是一个开源的自托管 Git 服务，类似于 GitHub 和 GitLab。Gitea 支持通过 Actions 来实现持续集成和持续部署（CI/CD）。在某些情况下，我们可能希望将 Gitea 的 Runner 配置为拉取本地的 Actions，而不是从 GitHub 获取，这在国内某些网络环境下尤其有用。

## 问题来源

在某些私有化部署的环境下，特别还需要将服务暴露到公网上时，如果你配置了 `REQUIRE_SIGNIN_VIEW` 为 `true`（启用此项以强制用户登录以查看任何页面或使用 API），那么在拉取本地 Actions 时，会遇到认证问题，权限不足导致无法成功拉取。

如果你将 `REQUIRE_SIGNIN_VIEW` 设置为 `false`，这会带来数据代码泄露的安全隐患，因为任何人都可以通过页面上的 “探索” 按钮访问你的 Gitea 代码库。

这个问题官方在 GitHub 上也有做讨论，但目前还没有一个很好的解决方案，相关 issue 地址为：
- [Runners cannot fetch custom actions](https://github.com/go-gitea/gitea/issues/27933)
- [Action runner should be run as a logined user when REQUIRE_SIGNIN_VIEW is true](https://github.com/go-gitea/gitea/issues/28813)


## 有几种不算特别完美的解决方案

### 一种具备部分安全性的解决思路

在 [Gitea 配置说明](https://docs.gitea.com/zh-cn/administration/config-cheat-sheet#service---explore-serviceexplore) 中，关于 `SERVICE.EXPLORE` 有 `REQUIRE_SIGNIN_VIEW` 配置，跟 `Service` 的 `REQUIRE_SIGNIN_VIEW` 不同，`SERVICE.EXPLORE.REQUIRE_SIGNIN_VIEW` 只控制 “探索” 页面是否需要登录，而不会影响 API 的访问权限，所以我们可以将 `REQUIRE_SIGNIN_VIEW` 设置为 `false`，而将 `SERVICE.EXPLORE.REQUIRE_SIGNIN_VIEW` 设置为 `true`，这样设置后，用户访问 Gitea 探索页面时需要提示登录，但 Runner 拉取本地 Actions 时不会受到影响。

```ini
[service]
REQUIRE_SIGNIN_VIEW = false

[service.explore]
REQUIRE_SIGNIN_VIEW = true
```

~~不过这样设置也有一个问题，就是**假设仓库地址是被其他人知道的情况下，仍然可以通过网页访问到对应的仓库页面**，所以仍然存在一定的代码泄露的安全隐患，但相较于 `REQUIRE_SIGNIN_VIEW` 设置为 `false`，这种方式的安全性要高一些，因为外人无法直接通过 API 访问仓库列表，并且也不清楚你的 Gitea 到底有那些仓库。~~

:::caution
通过 API 测试，`REQUIRE_SIGNIN_VIEW` 设置为 `false`，`SERVICE.EXPLORE.REQUIRE_SIGNIN_VIEW` 设置为 `true` 后，仍然可以通过 `your_gitea_host/api/v1/repos/search` 访问到仓库列表，说明这种方式并不能有效防止代码泄露的风险，不要使用这种方式，可以参考下面的其他解决思路。
:::

### 传入 Git 凭据

可以在 Runner 的配置中传入 Git 凭据，这样 Runner 在拉取本地 Actions 时就会使用这些凭据进行认证，从而绕过登录限制。

- 创建 `.git-credentials` 文件，内容如下：

```ini
http://{你的用户名}:{你的 PAT Token}@{你的 Gitea 主机}:3000
```

- 创建 `.gitconfig` 文件，内容如下：

```ini
[credential]
    helper = store
    # 容器里的路径而非宿主机路径
    file = /root/.git-credentials
```

- 在 Runner 的配置文件 [config.yaml](https://gitea.com/gitea/act_runner/src/branch/main/internal/pkg/config/config.example.yaml) 中，在 `container` 节中的 `options` 和 `valid_volumes` 添加以下内容：

```yaml
container:
  # Specifies the network to which the container will connect.
  # Could be host, bridge or the name of a custom network.
  # If it's empty, act_runner will create a network automatically.
  network: ""
  # Whether to use privileged mode or not when launching task containers (privileged mode is required for Docker-in-Docker).
  privileged: false
  # And other options to be used when the container is started (eg, --add-host=my.gitea.url:host-gateway).
  options: >-
    --mount type=bind,source=/path/to/.gitconfig,target=/root/.gitconfig,readonly
    --mount type=bind,source=/path/to/.git-credentials,target=/root/.git-credentials,readonly
  # The parent directory of a job's working directory.
  # NOTE: There is no need to add the first '/' of the path as act_runner will add it automatically.
  # If the path starts with '/', the '/' will be trimmed.
  # For example, if the parent directory is /path/to/my/dir, workdir_parent should be path/to/my/dir
  # If it's empty, /workspace will be used.
  workdir_parent:
  # Volumes (including bind mounts) can be mounted to containers. Glob syntax is supported, see https://github.com/gobwas/glob
  # You can specify multiple volumes. If the sequence is empty, no volumes can be mounted.
  # For example, if you only allow containers to mount the `data` volume and all the json files in `/src`, you should change the config to:
  # valid_volumes:
  #   - data
  #   - /src/*.json
  # If you want to allow any volume, please use the following configuration:
  # valid_volumes:
  #   - '**'
  valid_volumes:
    - "/path/to/.gitconfig"
    - "/path/to/.git-credentials"
  # overrides the docker client host with the specified one.
  # If it's empty, act_runner will find an available docker host automatically.
  # If it's "-", act_runner will find an available docker host automatically, but the docker host won't be mounted to the job containers and service containers.
  # If it's not empty or "-", the specified docker host will be used. An error will be returned if it doesn't work.
  docker_host: ""
  # Pull docker image(s) even if already present
  force_pull: true
  # Rebuild docker image(s) even if already present
  force_rebuild: false
```

意思是 `valid_volumes` 的路径是允许 Runner 挂载的路径，`options` 中的 `--mount` 是将本地的 `.gitconfig` 和 `.git-credentials` 文件挂载到 Runner 创建的工作容器中，这样 Runner 在拉取本地 Actions 时就会使用这些凭据进行认证，从而绕过登录限制。

### 另辟蹊径的解决思路

- 可以创建另外一个 Gitea 实例，里面只存放 Actions 相关的镜像，不涉及其他代码，这也算是一个变通方案

**如果有发现更好的解决思路，欢迎邮箱交流**

