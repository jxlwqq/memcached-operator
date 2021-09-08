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

#### 创建 API 和控制器

使用 Operator SDK CLI 创建自定义资源定义（CRD）API 和控制器。

运行以下命令创建带有组 cache、版本 v1alpha1 和种类 Memcached 的 API：

```shell
operator-sdk create api \
--resource=true \
--controller=true \
--group=cache \
--version=v1alpha1 \
--kind=Memcached
```

定义 Memcached 自定义资源（CR）的 API。

修改 api/v1alpha1/memcached_types.go 中的 Go 类型定义，使其具有以下 spec 和 status

```go
type MemcachedSpec struct {
	// +kubebuilder:validation:Minimum=0
	// Size is the size of the memcached deployment
	Size int32 `json:"size"`
}

type MemcachedStatus struct {
    // Nodes are the names of the memcached pods
	Nodes []string `json:"nodes"`
}
```
