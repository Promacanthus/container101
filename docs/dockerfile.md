# Dockerfile

[官方文档](https://docs.docker.com/engine/reference/builder/)

[最佳实践](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

Docker 为用户自己定义镜像提供了一个叫做 Dockerfile 的文件，在这个 Dockerfile 文件里，你可以设定自己镜像的创建步骤。制作镜像的时候，Docker 进程通过读取 Dockerfile （类似 Makefile） 文件中指定的命令来构建镜像的每一层。一个镜像不是说能 `run` 就完事了，需要保证如下两点：

1. 镜像**体积**足够小：包括 `rootfs`、安装的依赖、编译后的文件等

2. 容器**权限**足够小：只赋予容器内进程所必须的权限，缩小容器攻击面，减小对宿主机的影响

   对于函数计算这样的场景下，以上两点就是对镜像和容器的基本要求了。

> 写 Dockerfile 也是在写代码，可读性、可扩展性、可维护性、简洁性、可复用性、灵活性一个都不能少。

## 镜像体积足够小

一个容器镜像是很多个只读层组成，每一个只读层对应 Dockerfile 中的一条指令（Docker 指令而非 Linux 指令），每一层都是在上一层基础上的一个 delta 累加。
### 基础镜像
所有的镜像都是从一个父镜像（也称为基础镜像）开始构建的，所以基础选的好，构建镜像就很小。最小的基础镜像 `FROM scratch`，表示为空里面啥都没有，大部分情况下，应用都会对系统库有所依赖，所以在编译应用的时候需要使用静态编译将业务逻辑和依赖库一起打包在同一个二进制文件中，否则这样的容器是运行不起来的。

镜像只是将代码和依赖的文件打包在一起，真正运行的时候，还是要靠宿主机的内核去执行代码，所以不同的操作系统、不同的 CPU 架构就有不同的指令集，所以编译的时候需要特别注意这一点。

- 操作系统主要影响的是：系统依赖库和其他依赖的安装方式，

- CPU 架构主要影响的是：指令集。

大部分情况下，容器都是运行在 Linux 平台上的，也会有 Windows 或 Wasmer 这样的平台，一般会选择某个 Linux 发行版作为基础镜像。各种发行版基础镜像大小一览：（以 amd64 架构的 latest 标签版本为例，大部分云环境也都是这个架构）。

不同发行版之间最大的区别有三个：

1.  C 语言的运行时库，
2. 自带的常用软件工具，
3. 软件包管理工具。

|名称|描述|大小|
|---|---|---|
|busybox |将 Unix 中的常用工具的精简版集成到一个可执行文件中，比 GNU 工具集少，对于小型和嵌入式设备来说，已经是一个想当完整的环境 |746.79KB
|Alpine |基于 [musl libc](https://www.musl-libc.org/) 和 [busybox](https://www.busybox.net/) 组成，相比于 busybox 的优势是有一个[软件包仓库](https://pkgs.alpinelinux.org/packages) |2.68MB
|Debian |Linux 操作系统，有基于 GNU 协议的开源软件组成 |48.1MB|NeuroDebian |基于 Debian 但是提供很多神经科学研究所需的软件工具 |58.4MB|Ubuntu |基于 Debian 的 Linux 操作系统 |29.9MB
|Centos |基于 RHEL 的社区驱动的 Linux 发行版，现在已经木有了 |71.7MB
|fedora |Linux 操作系统 |58.97MB|oraclelinux |Linux 操作系统，与 Oracle 生态强绑定 |78.06MB
|opensuse/leap |Linux 操作系统 |40.44MB|archlinux |轻量、灵活的 Linux 操作系统 |128.36MB
|gcc GNU |编译器集合，支持各种编程语言的编译器系统，是GNU工具链的关键组成部分 |409.41MB

各个镜像或多或少都有一个叫 slim 的版本，这个版本会比上面表格中的镜像大小更小一些。

不同 Linux 发行版依赖库：

|名称|描述|常用|
|---|---|---|
|uClibc |基于 [Buildroot](https://buildroot.org/) 静态编译而成 |嵌入式设备|
|glibc |GNU发布的 libc 库是 linux 系统中最底层的 api，几乎其它任何运行库都会依赖于它 |Debian|
|musl libc |基于 Alpine 的 C 运行时库静态编译而成，比 glibc 更轻量级 |Alpine|

### distroless
上面说的基础镜像是将整个操作系统环境进行精简，还有另一种思路就是 [distroless](https://console.cloud.google.com/gcr/images/distroless/GLOBAL)，这样的镜像中只包含应用和运行时依赖，`gcr.io/distroless/base` 镜像大小为 19.2MB。
> 在 distroless 的镜像中没有软件包管理工具，没有 shell、也没有任何在标准 Linux 发行版中可用的软件工具。

这样的好处是，容器的攻击面最小，这样的坏处就是像进入容器这样的操作也是做不到的。

使用 distroless 作为基础镜像的示例：（distroless 的 python 版本是 3.7，可以更进一步优化，使用 pyinstaller，将 python 脚本都打包起来）

```Dockerfile
FROM docker-reg.devops.xiaohongshu.com/shequ/python3.7:slim-buster-gcc AS build
COPY function/requirements.txt requirements.txt
RUN pip3 config set global.index-url http://pypi.xiaohongshu.com/simple/ \&& 
	pip3 config set global.trusted-host pypi.xiaohongshu.com \&& 
	pip3 install -r requirements.txt
FROM gcr.io/distroless/python3-debian10:latest
COPY --from=build /usr/local/lib/python3.7/site-packages/ /usr/lib/python3.7/.
# 增加一些基础库和基础命令
COPY --from=build /usr/local/lib/ /usr/local/lib/
COPY --from=build /bin/bash /bin/bashCOPY --from=build /bin/ls /bin/ls
WORKDIR /home/app
COPY function/*.py .
CMD [ "index.py" ]
```

### 将镜像压缩为一层

> 如果将镜像压缩为一层，那么多个 image 共享 base image 以及加速 pull 的 feature 其实就用不到了。

docker 提供 `--squash` 参数，在构建的过程中将镜像的中间层都删除，然后以当前状态保存为一个单独的镜像层。好处是显而易见的，坏处就是镜像过度压缩，太小太专用了。

`docker-slim` 工具可以获取大型 Docker 镜像，临时运行它们，分析哪些文件在临时容器中是被真正使用的，然后生成一个新的、单层的 Docker 镜像，其中所有未使用的文件都会被删除。这样做有两个好处是镜像被缩小而且镜像变得更加安全，因为不需要的工具被删除了（例如 curl 或包管理器）。

### 构建镜像

#### 构建上下文

执行 `docker build` 命令时所指定的路径（通常是`.`，表示当前路径，也可以是相对路径或绝对路径）就是构建上下文，无关 Dockerfile 文件所在位置。可通过 `-f` 从指定文件或 `-f-` 从标准输入读取 Dockerfile。当前构建上下文中的文件和文件夹都会被递归的传递给 `docker daemon`。这样就可以在构建开始的日志信息中，看到如下信息：`Sending build context to Docker daemon xxx.xx MB`，这条信息是在告知需要发送给 `docker daemon` 的文件有多少 MB 大小。

```shell
docker build [OPTIONS] -
```

上面的连字符（`-`） 表示从标准输入中读取 Dockerfile 文件的内容，并且无需指定构建上下文，因为该构建上下文中只包含一个 Dockerfile 文件。不推荐，更好的做法是使用 `.dockerignore` 文件来指定构建是需要忽略的文件，只保留需要的，保证构建上下文尽可能的小。

#### 改动不频繁的内容往前方放


1. 利用镜像缓存：将不变的依赖库放在前面，经常变的代码放在后面；注意，缓存只在宿主机上，DockerInDocker 的化就没有效果了
1. 清理不需要的文件：apt、apk、yum 安装的时候会有缓存文件或相关文件

|效果|Debain类|Alpine|Centos类|
|---|---|---|---|
|不安装建议性（非必须）的依赖 |`apt-get install -y -no-install-recommends` |`apk add --update --no-cache`||
|清理安装后的缓存文件 |`rm -rf /var/lib/apt/lists/*` |`rm -rf /var/cache/apk/*` |`yum clean all`|

一些常见的包管理器删除缓存的方法：

- yum：`yum clean all`
- dnf：`dnf clean all`
- rvm：`rvm cleanup all`
- gem：`gem cleanup`
- cpan：`rm -rf ~/.cpan/{build,sources}/*`
- pip：`rm -rf ~/.cache/pip/*`
- apt-get：`apt-get clean`

**划重点，安装和清理缓存需要在同一层中进行，也就是写在同一个 RUN 指令中，同样的情况在设置 ENV 的时候也是。**因为在另一层进行操作的话，其实只是覆盖了上一层的文件，真实的文件还是在的。所以在单独一层中进行任何移动、更改（包括属性、权限等）、删除文件，都会出现文件复制一个副本，从而镜像非常大的情况。

```Dockerfile
RUN apt-get update \&& 
	apt-get install -y -no-install-recommends \
		aufs-tools \
		automake \
		build-essential \
		curl \
		dpkg-sig \
		libcap-dev \
		libsqlite3-dev \
		mercurial \
		reprepro \
		ruby1.9.1 \
		ruby1.9.1-dev \
		s3cmd=1.1.* \&& 
	rm -rf /var/lib/apt/lists/*
```
#### 健康检查

对于容器而言，最简单的健康检查是**进程级**的健康检查，即检验进程是否存活，`Docker Daemon` 会自动监控容器中的 PID1 进程。在很多实际场景下，仅使用进程级健康检查机制还远远不够。比如，容器进程虽然依旧运行却由于应用死锁无法继续响应用户请求，这样的问题是无法通过进程监控发现的。

在 Dockerfile 中使用 `HEALTHCHECK` 声明应用自身的健康监测配置

- HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
- HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉

> 注 ：HEALTHCHECK只可以出现一次，如果写了多个，只有最后一个生效。

**HEALTHCHECK 支持下列选项：**

- --interval=<间隔>：两次健康检查的间隔，默认为 30 秒;
- --timeout=<间隔>：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒;
- --retries=<次数>：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次；
- --start-period=<间隔>: 应用的启动的初始化时间，在启动过程中的健康检查失效不会计入，默认 0 秒。

**`docker run` 方式指定 health check**

- --health-cmd string：运行检查健康状况的命令
- --health-interval duration：运行间隔时间(ms|s|m|h)(缺省为 0s)
- --health-retries int：需要报告不健康的连续失败次数
- --health-start-period duration ：容器在开始健康重试倒计时之前初始化的起始周期(ms|s|m|h)(默认 0)
- --health-timeout duration：允许一次检查运行的最大时间(ms|s|m|h)(默认为 0s)
- --no-healthcheck：禁用任何容器指定的HEALTHCHECK，会使得 Dockerfile 构建出来的HEALTHCHECK功能失效。

#### 构建 Go 程序

```dockerfile
ENV CGO_ENABLED 0
ENV GOOS linux
ENV GOPROXY https://goproxy.cn,direct

RUN go build -ldflags="-s -w" -o /app/hello ./hello.go

RUN apk update --no-cache && apk add --no-cache ca-certificates tzdata
ENV TZ Asia/Shanghai
```

Dockerfile 文件内容说明，从上往下依次是：

- 默认禁用了 cgo
- 启用了 GOPROXY
- 去掉了调试信息 `-ldflags="-s -w"` 以减小镜像尺寸
- 安装了 `ca-certificates`，这样使用 TLS 证书就没问题了
- 自动设置了本地时区，这样我们在日志里看到的是北京时间了

#### 多阶段构建

通常分为编译阶段和运行阶段，这样可以最小化最后使用的镜像：

1. 在编译阶段，将业务代码打包为一个二进制可执行文件

2. 在运行阶段，使用最合适的基础镜像运行上面的二进制可执行文件

注意点：

1. 从上一个阶段拷贝内容：`COPY --from=0 ...`（阶段没有别名时）或者 `FROM golang:1.13 AS builder，COPY --from=builder ...`
1. 从一个已知镜像中拷贝内容：`COPY --from=nginx:latest ...`
1. 停留在某一个阶段：`docker build --target builder -t my-image:latest`，关键是这个 `--target`
1. 将前一个阶段作为新的阶段：`FROM golang:1.13 AS builder`，`FROM builder AS build 1`,`FROM builder AS build 2`

## 容器权限足够小
在容器运行起来后，在以镜像为基础的只读层上增加了一个可读写的容器层。这个处于运行状态的容器中的所有变更都是保存在这个容器层中，包括新建文件、修改文件、删除文件等操作。

如果一个服务运行的时候不需要 root 权限，那么在 Dockerfile 中使用 USER 来更改为普通用户。在更改用户之前，首先要保证用户已经创建好，可以通过如下命令创建。

```dockerfile
# UID/GID是在镜像已有的基础上递增的，也可以显式指定UID/GID的数值
RUN groupadd -r app && useradd --no-log-init -r -g app app
```
> Debian/Ubuntu 的 adduser 指令不支持 `--no-log-init` 参数。
如果需要类似 sudo 这样的功能来进行一些类似初始化守护进程这样的操作，可以考虑使用 [gosu](https://github.com/tianon/gosu)。

#### 以非 root 用户身份运行

默认情况下，当通过 `docker runyourImage:yourTag` 运行镜像时，这个容器（以及你在 `ENTRYPOINT/CMD` 中的程序）会以 `root` 用户身份运行（在容器和主机上）。这给了一个使用某种漏洞在你的运行容器中获得 `shell` 权限的攻击者以下权力：

- 对主机上所有显式挂载到容器中的目录的无限制写权限（因为是 `root`）。
- 能够在容器中做 Linux 根用户可以做的一切事情。例如，攻击者可以安装他们需要的额外工具来加载更多的恶意软件，比如说通过 `apt-get install`（非 `root` 用户无法做到这一点）。
- 如果你的镜像容器是用 `docker run --privileged` 启动的，攻击者甚至可以接管整个主机。

为了避免这种情况，应该以非 `root` 用户（在 `docker build` 过程中创建的一些用户）的身份运行应用程序。在 Dockerfile 中的某个地方（通常是在结尾处）放置以下语句：

```dockerfile
# Create a new user (including a home-directory, which is optional)
RUN useradd --create-home appuser
# Switch to this user
USER appuser
```

Dockerfile 中所有在 `USER appuser` 语句之后的命令（如 `RUN`、`CMD` 或 `ENTRYPOINT`）都将以这个用户运行。这里有一些需要注意的地方：

- 在切换到非 `root` 用户之前，通过 `COPY` 复制到镜像中的文件（或由某些 `RUN` 命令创建的文件）是由 `root` 用户拥有的，因此以非 `root` 用户身份运行的应用程序无法写入。为了解决这个问题，请把创建和切换到非 `root` 用户的代码移到 Dockerfile 的开头。
- 如果这些文件是在 Dockerfile 的开头以 `root` 用户身份创建的（存储在 `/root/` 下面，而不是 `/home/appuser/` 下面），那么程序期望在用户的主目录中的某个地方（例如 `~/.cache`）的文件，现在从应用程序的视角来看可能突然消失了。
- 如果应用程序监听一个 `TCP/UDP` 端口，就必须使用大于 1024 的端口。小于等于 1024 的端口只能以 `root` 用户身份使用，或者以一些高级 Linux 能力来使用，但不应该仅仅为了这个目的而给容器这些能力。

#### 测试镜像是否能在降低能力的情况下工作

Linux capabilities 是 Linux 内核的一个特性，它控制一个应用程序可以使用哪些内核特性，例如一个进程是否可以发送信号（如 `SIGKILL`）、配置网络接口、挂载磁盘，或调试进程等。一般来说，应用程序需要的功能越少越好。

启动镜像容器的所有人都可以给予（或拿走）这些能力，例如通过调用 “`docker run --cap-drop=ALL`“。默认情况下 Docker 会放弃所有能力，除了这里定义的那些以外，通常应用程序不需要所有这些功能。

作为一个最佳实践，可以尝试启动镜像容器，放弃所有能力（使用 `--cap-drop=ALL`），看看是否仍然正常工作。如果不能，搞清楚哪些功能是缺失的，并且判断是否真的需要它们。然后记录镜像需要哪些功能（以及为什么），这会给运行镜像的用户带去更多信心。

## 其他镜像分层技术

镜像的结构如果是满足 OCI 的标准规范的话，就可以在 OCI 的运行时中运行；换句话说只要能构建出满足 OCI 标准的镜像文件目录，就是一个标准的 Docker 镜像；现在也有了很多优秀的镜像分层技术，他们满足 OCI 标准，并且解决了 Docker 的一些缺点；合理的分层，可以使构建过程使用上大量的缓存，无需重复拉取，从而加快镜像的构建，下面看看一些比较流行的镜像分层技术。

#### [Podman](https://podman.io/)

Podman 提供与 Docker 非常相似的功能。可以说 Podman 就是为了替代 Docker 的，Podman 解决 docker 的一些痛点，比如 `docker daemon` 是一个守护进程、并需要 `root` 权限，但 podman 不需要在系统上运行任何守护进程，并且还可以在没有 `root` 权限的情况下运行。

Podman 可以管理和运行任何符合 OCI 规范的容器和容器镜像；Podman 的命令和 Docker 的命令，基本上是相同的，只需要将 `docker` 换为 `podman`，即可兼容 `docker` 的基本常用命令，Podman  也可以根据用户提供的 Dockerfile 文件构建镜像，不过一般不推荐使用 `podman build` 构建镜像，因为 podman 构建速度超慢，并且默认情况下使用 vfs 存储驱动程序会耗尽大量磁盘空间，一般使用 podman 的构建工具 Buildah。

#### [Buildah](https://buildah.io/)

Buildah 是一个专注于构建 OCI 容器镜像的工具，Buildah 构建速度非常快并使用覆盖存储驱动程序，可以节约大量的空间。

Buildah 使用 Dockerfile 构建时是在构建的最后一步进行的 commit，这样构建的镜像就只有一层，无法使用到缓存，也就是要做一些重复的拉取工作；如果使用 buildah 的原生命令构建镜像的话，分层会变得更加的灵活，可以自定义缓存点，在认为需要缓存的地方加上 commit 命令就能提交一层新的镜像。

Buildah mount 命令可以将容器的根目录挂载到主机的一个挂载点上；这使得我们可以使用主机上的工具进行构建和安装软件，不用将这些构建工具打包到容器镜像本身中。

## 检查构建产物

[dive](https://github.com/wagoodman/dive) 是一个 TUI，命令行的交互式 App，它可以让你看到 docker 每一层里面都有什么。

`dive ubuntu:latest` 命令可以看到 ubuntu image 里面都有什么文件。内容会显示为两侧，左边显示每一层的信息，右边显示当前层（会包含之前的所有层）的文件内容，本层新添加的文件会用黄色来显示。通过 tab 键可以切换左右的操作。

## 分析镜像安全

#### 漏洞扫描工具

查找 Docker 镜像中的漏洞的最简单方法是使用工具对它们进行检查。

[Anchore](https://anchore.com/) 是用于检查，分析和认证容器镜像的集中服务。它使用来自Red Hat，Debian 或 Alpine 等 OS 供应商的漏洞数据（提要）扫描镜像。对于非OS数据，它使用 NVD（国家漏洞数据库），其中包括 RPM，Deb，APK 以及 Python（PIP），Ruby Gems 等漏洞。

[Clair](https://github.com/quay/clair) 是 CoreOS 为 Docker 和 APPC 容器开发的静态分析器。它利用漏洞的元数据类似的来源为：Redhat，NVD，Ubuntu 的 CVE 跟踪，SecDB，Debian 安全 Bug 跟踪等。

#### 静态分析工具

使用静态分析工具，能够避免常见的错误，建立工程师自动遵循的最佳实践指南。

例如，[checkov](https://www.checkov.io/)、[Conftest](https://www.conftest.dev/)、[trivy](https://trivy.dev/) 或 [hadolint](https://github.com/hadolint/hadolint) 分析 Dockerfile 并列出不符合最佳实践规则的地方，它们是 Dockerfile 的 linter。

