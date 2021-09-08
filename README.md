# memcached-operator

Operator SDK 中的 Go 编程语言支持可以利用 Operator SDK 中的 Go 编程语言支持，为 Memcached 构
建基于 Go 的 Operator 示例、分布式键值存储并管理其生命周期。

#### 前置条件

* 安装 Docker Desktop，并启动内置的 Kubernetes 集群
* 注册一个 [hub.docker.com](https://hub.docker.com/) 账户，需要将本地构建好的镜像推送至公开仓库中
* 安装 operator SDK CLI: `brew install operator-sdk`
* 安装 Go: `brew install go`

#### 创建项目

使用 Operator SDK CLI 创建名为 memcached-operator 的项目。
```shell
mkdir -p $HOME/projects/memcached-operator
cd $HOME/projects/memcached-operator
go env -w GOPROXY=https://goproxy.cn,direct

operator-sdk init \
--domain=jxlwqq.github.io \
--repo=github.com/jxlwqq/memcached-operator \
--skip-go-version-check
```