## kubebuilder简介

- github仓库 `https://github.com/kubernetes-sigs/kubebuilder`
- 官方文档 `https://book.kubebuilder.io/introduction.html`
- 中文翻译 `https://xuejipeng.github.io/kubebuilder-doc-cn/`

#### kubebuilder安装

在wsl中的ubuntu中安装

```bash
# download kubebuilder and install locally.
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder && sudo mv kubebuilder /usr/local/bin/
```

指定版本安装

```bash
# download kubebuilder and install locally.
curl -L -o kubebuilder https://github.com/kubernetes-sigs/kubebuilder/releases/download/v3.7.0/kubebuilder_$(go env GOOS)_$(go env GOARCH)
chmod +x kubebuilder && sudo mv kubebuilder /usr/local/bin/
```



### 创建我们第一个operator

#### 准备工作

创建工程目录

```bash
mkdir -p ~/GoProjects/repos/demo01
cd ~/GoProjects/repos/demo01
```

初始化git
这里的目的是用git来记录kubebuilder都新建或修改了什么文件

```bash
git init
```

初始化go mod

```bash
go mod init github.com/lh-hub/operator_01
```

提交一个版本，作为开始基线

```bash
git add .
git commit -m "go mod init"
```

### 初始化 kubebuilder

```bash
kubebuilder init --domain github.com
```

提交一个版本，便于稍后查看都做了什么

```bash
git add .
git commit -m "kuberbuild init"
```

### 创建 api

```
kubebuilder create api --group demo --version v1 --kind App
```

提交一个版本，便于稍后查看都做了什么

```
git add .
git commit -m "kuberbuild create api"
```



# 6 简单分析两个命令都做了什么

## 6.1 init 命令

- 创建了必要的基础代码
- 创建了管理项目的makefile文件
- 创建了必要的配置文件

## 6.2 create api 命令

- 创建了api相关的代码
- 更新了api相关的配置

# 7 kustomize 介绍

Kustomize是一个定制Kubernetes配置的工具。一般有以下能力：

- 生成资源
- 设置资源字段
- 组合和定制资源集合

## 7.1 生成资源

例如

- ConfigMap 它的数据一般来源于config.yaml之类的配置文件，使用configMapGenerator，从文件中生成ConfigMap。

```
cat <<EOF >config.yaml
foo: bar
EOF

cat <<EOF >./kustomization.yaml
configMapGenerator:

- name: example-configmap-c
  files:
  - config.yaml
    EOF
```

在执行完 `kubectl kustomize ./` 之后，生成的configMap资源为:

```
apiVersion: v1
data:
config.yaml: |
foo: bar
kind: ConfigMap
metadata:
name: example-configmap-c-b5bdf7982h
```

- Secret 持敏感数据，同样它真正来源一般来自其他地方，比如password.txt密钥文件。使用secretGenerator，从文件Secret，并且会做base64转换。

```
cat <<EOF >./password.txt
username=admin
password=secret
EOF

cat <<EOF >./kustomization.yaml
secretGenerator:

- name: example-secret-p
  files:
  - password.txt
    EOF
```

在执行完 `kubectl kustomize ./` 之后，生成的secret资源为:

```
apiVersion: v1
data:
password.txt: dXNlcm5hbWU9YWRtaW4KcGFzc3dvcmQ9c2VjcmV0Cg==
kind: Secret
metadata:
name: example-secret-p-c4kts5h4ta
type: Opaque
```

## 7.2 资源设置字段

在一个项目中为Kubernetes的所有资源设置统一字段或者增加前后缀是很常见的。如

- 为所有资源设置相同的命名空间
- 添加相同的名称前缀或后缀
- 添加相同的标签(labels)集合
- 添加相同的注解(annotations)集合

```
cat <<EOF >./deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: nginx-deployment
labels:
app: nginx
spec:
selector:
matchLabels:
app: nginx
template:
metadata:
labels:
app: nginx
spec:
containers:

- name: nginx
  image: nginx
  EOF

cat <<EOF >./kustomization.yaml
namespace: mashibing
namePrefix: pro-
nameSuffix: "-app"
commonLabels:
app: web
commonAnnotations:
sync: true
resources:

- deployment.yaml
  EOF
```

在执行完 `kubectl kustomize ./` 之后，生成的deployment资源为:

```
apiVersion: apps/v1
kind: Deployment
metadata:
annotations:
sync: true
labels:
app: web
name: pro-nginx-deployment-app
namespace: mashibing
spec:
selector:
matchLabels:
app: web
template:
metadata:
annotations:
sync: true
labels:
app: web
spec:
containers:

- image: nginx
  name: nginx
```

## 7.3 组合和定制资源集合

```
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: myweb
spec:
selector:
matchLabels:
run: myweb
replicas: 2
template:
metadata:
labels:
run: myweb
spec:
containers:

- name: myweb
  image: nginx
  ports:
- containerPort: 80
  EOF

cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
name: myweb
labels:
run: myweb
spec:
ports:

- port: 80
  protocol: TCP
  selector:
  run: my-nginx
  EOF

cat <<EOF >./kustomization.yaml
resources:

- deployment.yaml
- service.yaml
  EOF
```

在执行 `kubectl kustomize ./` 之后，两个资源都会生成

# 8 编写我们的operator

执行命令

```
go mod tidy
```

下载需要的依赖，并删除无用的依赖

## 8.1 编写结构定义部分

修改文件 `api/v1/app_types.go`

## 8.2 编写业务逻辑部分

修改文件 `controllers/app_controller.go`

# 9 运行我们的operator

## 9.1 Makefile文件介绍

- generate 生成crd文件
- manifests 生成部署需要的文件
- build 编译operator
- install 安装crd
- run 运行operator

## 9.2 执行命令

```
make generate
make manifests
make build
make install
make run
```

```

```