---
title: "用 podman 代替 docker"
date: 2019-04-20T20:11:26+08:00
draft: false
---

各种容器的实现简直和 UNIX 一样乱七八糟

并且全是用 Go 写的 (???

podman 背后由 Red Hat 支持，有望成为支持各发行版自带的容器管理实现

类似的先例是 pulseaudio, systemd 和未来的 wayland

podman 目标不是容器编排，可以使用更专业的 openshift/k8s 进行容器编排

### 使用
1. 把 `alias docker podman` 添加到你的 sh rc 文件


### 非 root 使用
1. sysctl -w kernel.unprivileged_userns_clone=1
1. 安装 slirp4netns ，这个程序提供 root less 时的网络命名空间
1. 参照 subuid(5) 的提示编写你的 `/etc/subuid` 和 `/etc/subgid` 文件，这个文件提供 uid 与 gid 映射
1. 或者按照 podman 官方的推荐运行 `sudo usermod --add-subuids 10000-75535 $(whoami)`

我是这样写的
```bash
# cat /etc/subuid
sa:10000:500

# cat /etc/subgid
sa:500:50
```

标识 sa 用户可以使用从 100000 开始的 500 各 uid 和从 500 开始的 50 个 gid

非 root 用户之间的容器镜像不共享，容器镜像保存在每个用户的 `$XDG_DATA_HOME/.local/share/containers/storage` 里


### 优点和与 docker lxd nspawn 等的不同

- docker lxd podman 都是 Go 写的

- podman 与 docker 都使用 runc (Go写的) 作为底层

- podman 与 docker 都支持 OCI Image Fromat (Go写的)，都能使用 docker hub 上的容器镜像。
  nspawn 没法使用它们的镜像

- podman 使用 CNI (Go写的) 作为网络底层，实现比 Docker 网络层略微简单但原理相同。
  相对于 lxd 与 nspawn CNI 可以避免编写大量的网络规则。

- podman 可以使用 slirp4netns，避免打 ip a 是看到一大坨 veth* 和 docker0, 并且性能更好

- podman 没有 daemon 进程，由 podman cli 直接 clone 并创建命名空间，这带来了其他好处：

    - root less：可以不使用 root 运行

    - 容器的父进程都是在终端模拟器里敲出的 podman 命令

    - 每个用户拥有自己的 podman 命名空间，

    - 可以用 systemd service 管理容器

- podman 里集成了 CRIU，所以 podman 里的容器可以在单机上热迁移
<script id="asciicast-205183" src="https://asciinema.org/a/205183.js" async></script>

### 快速体验
<script id="asciicast-QDOgqAMp7nP05otIIZPuWL0O8" src="https://asciinema.org/a/QDOgqAMp7nP05otIIZPuWL0O8.js" async></script><Paste>

### ref
1. Replacing Docke With Podman https://media.ccc.de/v/ASG2018-177-replacing_docker_with_podman
1. CNI https://github.com/containernetworking/cni
1. RunC https://github.com/opencontainers/runc
1. OCI https://github.com/opencontainers/image-spec
1. CRIU https://podman.io/blogs/2018/10/10/checkpoint-restore.html
1. slirp4nets https://github.com/rootless-containers/slirp4netns
