# memcached-operator

Operator SDK 中的 Go 编程语言支持可以利用 Operator SDK 中的 Go 编程语言支持，为 Memcached 构
建基于 Go 的 Operator 示例、分布式键值存储并管理其生命周期。

### 前置条件

* 安装 Docker Desktop，并启动内置的 Kubernetes 集群
* 注册一个 [hub.docker.com](https://hub.docker.com/) 账户，需要将本地构建好的镜像推送至公开仓库中
* 安装 operator SDK CLI: `brew install operator-sdk`
* 安装 Go: `brew install go`

本示例推荐的依赖版本：

* Docker Desktop: >= 4.0.0
* Kubernetes: >= 1.21.4
* Operator-SDK: >= 1.11.0
* Go: >= 1.17


> jxlwqq 为笔者的 ID，命令行和代码中涉及的个人 ID，均需要替换为读者自己的，包括
> * `--domain=`
> * `--repo=`
> * `//+kubebuilder:rbac:groups=`
> * `IMAGE_TAG_BASE ?=`


### 创建项目

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

### 创建 API 和控制器

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

为资源类型更新生成的代码：
```shell
make generate
```

运行以下命令以生成和更新 CRD 清单：
```shell
make manifests
```

### 实现控制器

在本例中，将生成的控制器文件 controllers/memcached_controller.go 替换为以下示例实现：

```go
/*
Copyright 2021.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"reflect"
	"time"

	"context"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	ctrllog "sigs.k8s.io/controller-runtime/pkg/log"

	cachev1alpha1 "github.com/jxlwqq/memcached-operator/api/v1alpha1"
)

// MemcachedReconciler reconciles a Memcached object
type MemcachedReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=cache.jxlwqq.github.io,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.jxlwqq.github.io,resources=memcacheds/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=cache.jxlwqq.github.io,resources=memcacheds/finalizers,verbs=update
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the Memcached object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.9.2/pkg/reconcile
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := ctrllog.FromContext(ctx)

	// Fetch the Memcached instance
	memcached := &cachev1alpha1.Memcached{}
	err := r.Get(ctx, req.NamespacedName, memcached)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			log.Info("Memcached resource not found. Ignoring since object must be deleted")
			return ctrl.Result{}, nil
		}
		// Error reading the object - requeue the request.
		log.Error(err, "Failed to get Memcached")
		return ctrl.Result{}, err
	}

	// Check if the deployment already exists, if not create a new one
	found := &appsv1.Deployment{}
	err = r.Get(ctx, types.NamespacedName{Name: memcached.Name, Namespace: memcached.Namespace}, found)
	if err != nil && errors.IsNotFound(err) {
		// Define a new deployment
		dep := r.deploymentForMemcached(memcached)
		log.Info("Creating a new Deployment", "Deployment.Namespace", dep.Namespace, "Deployment.Name", dep.Name)
		err = r.Create(ctx, dep)
		if err != nil {
			log.Error(err, "Failed to create new Deployment", "Deployment.Namespace", dep.Namespace, "Deployment.Name", dep.Name)
			return ctrl.Result{}, err
		}
		// Deployment created successfully - return and requeue
		return ctrl.Result{Requeue: true}, nil
	} else if err != nil {
		log.Error(err, "Failed to get Deployment")
		return ctrl.Result{}, err
	}

	// Ensure the deployment size is the same as the spec
	size := memcached.Spec.Size
	if *found.Spec.Replicas != size {
		found.Spec.Replicas = &size
		err = r.Update(ctx, found)
		if err != nil {
			log.Error(err, "Failed to update Deployment", "Deployment.Namespace", found.Namespace, "Deployment.Name", found.Name)
			return ctrl.Result{}, err
		}
		// Ask to requeue after 1 minute in order to give enough time for the
		// pods be created on the cluster side and the operand be able
		// to do the next update step accurately.
		return ctrl.Result{RequeueAfter: time.Minute}, nil
	}

	// Update the Memcached status with the pod names
	// List the pods for this memcached's deployment
	podList := &corev1.PodList{}
	listOpts := []client.ListOption{
		client.InNamespace(memcached.Namespace),
		client.MatchingLabels(labelsForMemcached(memcached.Name)),
	}
	if err = r.List(ctx, podList, listOpts...); err != nil {
		log.Error(err, "Failed to list pods", "Memcached.Namespace", memcached.Namespace, "Memcached.Name", memcached.Name)
		return ctrl.Result{}, err
	}
	podNames := getPodNames(podList.Items)

	// Update status.Nodes if needed
	if !reflect.DeepEqual(podNames, memcached.Status.Nodes) {
		memcached.Status.Nodes = podNames
		err := r.Status().Update(ctx, memcached)
		if err != nil {
			log.Error(err, "Failed to update Memcached status")
			return ctrl.Result{}, err
		}
	}

	return ctrl.Result{}, nil
}

// deploymentForMemcached returns a memcached Deployment object
func (r *MemcachedReconciler) deploymentForMemcached(m *cachev1alpha1.Memcached) *appsv1.Deployment {
	ls := labelsForMemcached(m.Name)
	replicas := m.Spec.Size

	dep := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      m.Name,
			Namespace: m.Namespace,
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: &replicas,
			Selector: &metav1.LabelSelector{
				MatchLabels: ls,
			},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: ls,
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{{
						Image:   "memcached:1.4.36-alpine",
						Name:    "memcached",
						Command: []string{"memcached", "-m=64", "-o", "modern", "-v"},
						Ports: []corev1.ContainerPort{{
							ContainerPort: 11211,
							Name:          "memcached",
						}},
					}},
				},
			},
		},
	}
	// Set Memcached instance as the owner and controller
	ctrl.SetControllerReference(m, dep, r.Scheme)
	return dep
}

// labelsForMemcached returns the labels for selecting the resources
// belonging to the given memcached CR name.
func labelsForMemcached(name string) map[string]string {
	return map[string]string{"app": "memcached", "memcached_cr": name}
}

// getPodNames returns the pod names of the array of pods passed in
func getPodNames(pods []corev1.Pod) []string {
	var podNames []string
	for _, pod := range pods {
		podNames = append(podNames, pod.Name)
	}
	return podNames
}

// SetupWithManager sets up the controller with the Manager.
func (r *MemcachedReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&cachev1alpha1.Memcached{}).
		Owns(&appsv1.Deployment{}).
		Complete(r)
}
```

运行以下命令以生成和更新 CRD 清单：
```shell
make manifests
```

### 运行 Operator

捆绑 Operator，并使用 Operator Lifecycle Manager（OLM）在集群中部署。

修改 Makefile 中 IMAGE_TAG_BASE 和 IMG：

```makefile
IMAGE_TAG_BASE ?= docker.io/jxlwqq/memcached-operator
IMG ?= $(IMAGE_TAG_BASE):latest
```

构建镜像：

```shell
make docker-build
```

将镜像推送到镜像仓库：
```shell
make docker-push
```

成功后访问：https://hub.docker.com/r/jxlwqq/memcached-operator

运行 make bundle 命令创建 Operator 捆绑包清单，并依次填入名称、作者等必要信息:
```shell
make bundle
```

构建捆绑包镜像：
```shell
make bundle-build
```

推送捆绑包镜像：
```shell
make bundle-push
```

成功后访问：https://hub.docker.com/r/jxlwqq/memcached-operator-bundle

使用 Operator Lifecycle Manager 部署 Operator:

```shell
# 切换至本地集群
kubectl config use-context docker-desktop
# 安装 olm
operator-sdk olm install
# 使用 Operator SDK 中的 OLM 集成在集群中运行 Operator
operator-sdk run bundle docker.io/jxlwqq/memcached-operator-bundle:v0.0.1
```

查看 Pod：
```shell
NAME                                                              READY   STATUS      RESTARTS   AGE
7bb60cfcec4a426a75d6ca01153f5e6d3061e175d035791255372bdd7ebrvmd   0/1     Completed   0          67s
docker-io-jxlwqq-memcached-operator-bundle-v0-0-1                 1/1     Running     0          84s
memcached-operator-controller-manager-666cfff6c-fxhrk             2/2     Running     0          41s
```



### 创建自定义资源

编辑 config/samples/cache_v1alpha1_memcached.yaml 上的 Memcached CR 清单示例，使其包含
以下规格：

```yaml
apiVersion: cache.jxlwqq.github.io/v1alpha1
kind: Memcached
metadata:
  name: memcached-sample
spec:
  size: 2
```

创建 CR：
```shell
kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml
```

查看 Pod：
```shell
NAME                                                              READY   STATUS      RESTARTS   AGE
7bb60cfcec4a426a75d6ca01153f5e6d3061e175d035791255372bdd7ebrvmd   0/1     Completed   0          4m14s
docker-io-jxlwqq-memcached-operator-bundle-v0-0-1                 1/1     Running     0          4m31s
memcached-operator-controller-manager-666cfff6c-fxhrk             2/2     Running     0          3m48s
memcached-sample-6c765df685-d8ppn                                 1/1     Running     0          29s
memcached-sample-6c765df685-dtw9l                                 1/1     Running     0          29s
```

更新 CR：
```shell
kubectl patch memcached memcached-sample -p '{"spec":{"size": 3}}' --type=merge
```

查看 Pod：
```shell
NAME                                                              READY   STATUS      RESTARTS   AGE
7bb60cfcec4a426a75d6ca01153f5e6d3061e175d035791255372bdd7ebrvmd   0/1     Completed   0          6m52s
docker-io-jxlwqq-memcached-operator-bundle-v0-0-1                 1/1     Running     0          7m9s
memcached-operator-controller-manager-666cfff6c-fxhrk             2/2     Running     0          6m26s
memcached-sample-6c765df685-d8ppn                                 1/1     Running     0          3m7s
memcached-sample-6c765df685-dtw9l                                 1/1     Running     0          3m7s
memcached-sample-6c765df685-n7ctq                                 1/1     Running     0          6s
```

### 做好清理

```shell
operator-sdk cleanup memcached-operator
operator-sdk olm uninstall
```

### 更多

更多经典示例请参考：https://github.com/jxlwqq/kubernetes-examples
