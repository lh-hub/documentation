### kind 使用文档

创建集群，--name 为名称，--image 为镜像，镜像地址：https://hub.docker.com/r/kindest/node/

```bash
kind create cluster --name 1.21.2 --image kindest/node:v1.21.2
kind create cluster --name=1.24.3 --image kindest/node:v1.24.3
```

查看集群

```bash
kind get clusters
```

切换集群

```
kubectl cluster-info --context kind-1.21.2
kubectl cluster-info --context kind-1.24.3
```

导出配置文件

```bash
kind get kubeconfig --name 1.21.2 > ~/.kube1.21.2
```

