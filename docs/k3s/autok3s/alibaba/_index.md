---
title: 创建阿里云集群
description: 本文介绍了如何在阿里云 ECS 中创建和初始化 K3s 集群，以及为已有的 K3s 集群添加节点的操作步骤。
keywords:
  - k3s中文文档
  - k3s 中文文档
  - k3s中文
  - k3s 中文
  - k3s
  - k3s教程
  - k3s中国
  - rancher
  - k3s 中文教程
  - AutoK3s
  - 创建阿里云集群
---

## 概述

本文介绍了如何在阿里云 ECS 中创建和初始化 K3s 集群，以及为已有的 K3s 集群添加节点的操作步骤。除此之外，本文还提供了在阿里云 ECS 上运行 AutoK3s 的进阶操作指导，如配置私有镜像仓库、启用阿里云 Terway CNI、启用阿里云 CCM、和启用 UI 组件。

## 前置要求

### 设置 RAM

关于 RAM 的描述，请参考[阿里云官方文档](https://www.alibabacloud.com/help/doc-detail/54235.htm)。

您的账号需要创建 ECS 及相关资源的权限，因此需要确保具有以下资源的权限：

```json
{
  "Version": "1",
  "Statement": [
    {
      "Action": [
        "ecs:Describe*",
        "ecs:AttachDisk",
        "ecs:CreateDisk",
        "ecs:CreateSnapshot",
        "ecs:CreateRouteEntry",
        "ecs:DeleteDisk",
        "ecs:DeleteSnapshot",
        "ecs:DeleteRouteEntry",
        "ecs:DetachDisk",
        "ecs:ModifyAutoSnapshotPolicyEx",
        "ecs:ModifyDiskAttribute",
        "ecs:CreateNetworkInterface",
        "ecs:AttachNetworkInterface",
        "ecs:DetachNetworkInterface",
        "ecs:DeleteNetworkInterface",
        "ecs:CreateNetworkInterface",
        "ecs:AttachNetworkInterface",
        "ecs:DetachNetworkInterface",
        "ecs:DeleteNetworkInterface",
        "ecs:AssignPrivateIpAddresses",
        "ecs:UnassignPrivateIpAddresses",
        "ecs:DeleteInstances",
        "ecs:RunInstances",
        "ecs:ListTagResources",
        "ecs:StartInstances",
        "ecs:StopInstances",
        "ecs:CreateSecurityGroup",
        "ecs:ModifySecurityGroupRule",
        "ecs:ModifySecurityGroupEgressRule",
        "ecs:DescribeSecurityGroup*",
        "ecs:AuthorizeSecurityGroup",
        "ecs:RevokeSecurityGroup",
        "ecs:RevokeSecurityGroupEgress"
      ],
      "Resource": ["*"],
      "Effect": "Allow"
    },
    {
      "Action": ["cr:Get*", "cr:List*", "cr:PullRepository"],
      "Resource": ["*"],
      "Effect": "Allow"
    },
    {
      "Action": ["slb:*"],
      "Resource": ["*"],
      "Effect": "Allow"
    },
    {
      "Action": ["cms:*"],
      "Resource": ["*"],
      "Effect": "Allow"
    },
    {
      "Action": ["vpc:*"],
      "Resource": ["*"],
      "Effect": "Allow"
    },
    {
      "Action": ["log:*"],
      "Resource": ["*"],
      "Effect": "Allow"
    },
    {
      "Action": ["nas:*"],
      "Resource": ["*"],
      "Effect": "Allow"
    }
  ]
}
```

### 设置安全组

ECS 实例**至少**需要应用以下安全组规则：

```bash
Rule        Protocol    Port      Source             Description
InBound     TCP         22        ALL                SSH Connect Port
InBound     TCP         6443      K3s agent nodes    Kubernetes API
InBound     TCP         10250     K3s server & agent Kubelet
InBound     UDP         8472      K3s server & agent (Optional) Required only for Flannel VXLAN
InBound     TCP         2379,2380 K3s server nodes   (Optional) Required only for embedded ETCD
OutBound    ALL         ALL       ALL                Allow All
```

## UI 使用说明

接下来我们将基于 AutoK3s 本地 UI 介绍如何使用 AutoK3s 工具，在 AWS EC2 主机上创建和管理 K3s 集群。如果您想了解 CLI 的使用，请移步到 [CLI 使用说明](#cli-使用说明)

您可以通过[快速体验](/docs/k3s/autok3s/_index#快速体验)中的描述，通过 Docker 或者 CLI 启动本地 UI，打开浏览器，访问目标端口 `8080` 即可。

### 快速创建集群

您可以使用快速创建功能，在指定的云提供商服务中，快速启动一个K3s集群。

由于 `rancher.cn` 域名正在备案中，因此使用默认的 `https://rancher-mirror.rancher.cn/k3s/k3s-install.sh` 地址部署集群会出现部署失败情况。如果发生此类问题，请使用以下方式解决： `--k3s-install-script=https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh`。

以下图为例，我们将在 aliyun ECS 使用默认配置创建一个单节点的 K3s 集群，ECS 虚拟机会在 `cn-hangzhou` 区域中创建。

![](/img/k3s/quick-start-alibaba.png)

**表 1：快速创建参数**

| 参数 | 说明 | 默认值
| :------------- | :----------------- | :-----------------
| Provider | 云提供商名称 | `alibaba`
| Name | K3s集群名称 | 
| Master | Master 节点数量 | `1`
| Worker | Worker 节点数量 | `0`
| Access Key | 云提供商的Access Key 访问密钥 |
| Secret Key | 云提供商的Secret Key 访问密钥 |
| Region | 虚拟机所在的区域（在快速创建页面不允许编辑默认区域，您可以通过模板功能修改默认参数配置） | `cn-hangzhou`
| Zone | 虚拟机所在的地区 | `cn-hangzhou-i`

### 自定义参数创建

如果您不想使用默认的配置，可以点击 **Advance** 按钮，进入自定义参数页面进行更多参数的设置。
或者您可以在集群列表页点击 **Create** 按钮进入自定义参数页面进行更多参数的设置。

使用 Alibaba 云提供商创建 K3s 集群的自定义参数配置分为四项，云供应商访问凭证、云提供商对应实例配置、K3s集群配置、高级选项。接下来对每个配置项进行详细说明。

#### 云提供商访问凭证

![](/img/k3s/custom-create-cluster-credential-alibaba.png)

**表 2：云提供商访问凭证参数**

| 参数 | 说明 
| :------------- | :--------------- 
| Provider | 云提供商名称 
| Name | K3s集群名称 
| Access Key | 云提供商的Access Key 访问密钥 
| Secret Secret | 云提供商的Secret Key 访问密钥  

#### 实例配置

实例配置主要配置的内容为对应云提供商虚拟机的配置，例如实例运行区域、地区、使用的操作系统类型、实例规格、网络配置等信息。

![](/img/k3s/custom-create-cluster-option-alibaba.png)

**表 3：实例配置参数**

| 参数 | 说明 | 默认值
| :------------- | :----------------------- |:------------- 
| Region | 阿里云 ECS 区域 | `cn-hangzhou`
| Zone | 阿里云 ECS 地区 | `cn-hangzhou-i` 
| Instance Type | 阿里云 ECS 实例规格 | `ecs.c6.large`(2vCPU/4GiB)
| Image | 阿里云 ECS 系统映像ID | `ubuntu_18_04_x64_20G_alibase_20200618.vhd`(Ubuntu 18.04)
| Disk Category | 根磁盘卷类型 | `cloud_ssd`
| Disk Size | 根磁盘卷大小 | `40`(GiB)
| Spot Strategy | 是否使用[抢占式实例](https://www.alibabacloud.com/help/zh/elastic-compute-service/latest/preemptible-instances-overview#section-pgb-zcy-wgb) | `NoSpot`
| Spot Duration | 抢占式实例的保留时长，单位为小时 | `1`
| Spot Price Limit | 设置实例的每小时最高价格，支持最大3位小数，仅当参数`SpotStrategy`设置为`SpotWithPriceLimit`时生效 | 
| V-Switch | 专有网络交换机 ID (如果您不设置，AutoK3s 将为您创建名称为 autok3s-aliyun-vswitch 的默认交换机) | 
| Internet Max Bandwidth Out | 网络带宽 | 5(MB)
| Security Group | 安全组(如果您不选择任何安全组，AutoK3s 将为您自动创建名称为 autok3s 的[默认的安全组](#设置安全组)) | 
| EIP | 是否使用弹性公网IP | false
| Key Pair | 密钥对名称(如果您不设置密钥对，AutoK3s 将自动为您生成一个密钥对) |
| SSH User | SSH 用户 | `root`
| SSH Port | SSH 端口 | `22`
| SSH Key Path | 如果您选择了已有的密钥对，需要指定SSH 私钥目录，如果您使用AutoK3s 自动生成的密钥，这里可以不填写任何内容 |
| SSH Key Passphrase | 如果您的私钥需要密码验证，请在这里输入密码 |
| SSH Password | 如果您使用Password连接到虚拟机，请输入密码 |
| SSH Agent Auth | 如果您配置 SSH 代理程序，可以开启此项配置 | false
| SSH Cert Path | 如果您配置了 ssh certificate，在这里您需要将 certificate path 传入进来，以保证 AutoK3s 可以通过ssh连接到远程虚拟机 |
| Tags | 虚拟机标签 |
| User Data | 配置实例的[自定义数据](https://help.aliyun.com/document_detail/108461.html) |

#### K3s 参数配置

K3s 参数配置项主要对 K3s 集群进行设置，例如是否部署 HA 模式、K3s 安装版本等。

![](/img/k3s/custom-create-cluster-k3s.png)

**表 4：K3s 配置参数**

| 参数 | 说明 | 默认值
| :------------- | :------------------ |:------------- 
| K3s Channel | 用于获取 K3s 下载 URL 的通道。选项包括：`stable`, `latest`, `testing`。 | `stable`
| K3s Version | 安装的 K3s 版本，如果设置则覆盖 Channel 选项 |
| Cluster | 启用嵌入式 DB 高可用 K3s（即开启 `--cluster-init` 设置） | false
| Datastore | 指定 etcd、Mysql、Postgres 或 Sqlite（默认）数据源名称 |
| K3s Install Script | K3s 安装脚本 | `https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh`
| Master | Master 节点数量 |
| Master Extra Args | Master 节点额外参数设置，例如 `--no-deploy traefik` |
| Worker | Worker 节点数量 |
| Worker Extra Args | Worker 节点额外参数设置，例如 `--node-taint key=value:NoExecute` |
| Token | 用于将server或agent加入集群的共享secret，如果不设置，会自动生成一个Token |
| Manifests | 自动部署应用清单目录，这里支持设置某个manifest文件或者包含多个manifest文件的目录路径（仅支持一层目录），具体功能可查看[这里](http://docs.rancher.cn/docs/k3s/advanced/_index/#%E8%87%AA%E5%8A%A8%E9%83%A8%E7%BD%B2%E6%B8%85%E5%8D%95) |
| TLS Sans | 在 TLS 证书中添加其他主机名或 IP 作为主题备用名称，具体功能可查看[这里](https://docs.rancher.cn/docs/k3s/installation/install-options/server-config/_index#%E7%9B%91%E5%90%AC) |
| Registry | [私有镜像仓库配置](https://docs.rancher.cn/docs/k3s/installation/private-registry/_index) |

#### 高级选项

配置要开启的 UI 组件（kubernetes-dashboard/kube-explorer），是否部署 Terway 网络插件以及是否部署阿里云 Cloud Provider。

![](/img/k3s/custom-create-cluster-additional-alibaba.png)

**表 4：高级选项**

| 参数 | 说明 | 默认值
| :------------- | :------------------- |:------------- 
| UI | 开启的 UI 组件（Kubernetes Dashboard/kube-explorer） | 
| Cloud Controller Manager | 是否部署阿里云 Cloud Provider | false
| Terway | 是否部署 Terway 网络插件 | false
| Terway Max Pool Size | Terway Max Pool Size | 5

> PS: AutoK3s 默认部署 `ENI` 模式的 Terway 插件，更多信息请参考[这里](https://www.alibabacloud.com/help/zh/doc-detail/97467.htm)

### 集群模板

您可以使用模板功能，提前预置好常用的集群模板，每次创建集群时可以用模板参数进行填充，极大精简了重复操作。一次编写，多次运行，提升效率。

创建集群模板的参数与上面描述的自定义参数创建集群的表单内容相同，在这里不做赘述。

![](/img/k3s/cluster-templates.png)

您可以选择最常用的模板，点击右侧下拉框中的 **Set Default** 按钮，将模板设置为默认模板。

![](/img/k3s/set-default-templates.png)

设置为默认模板后，您可以通过快速创建功能一键部署常用配置的 K3s 集群。

![](/img/k3s/quick-start-with-default-templates.png)

如果您想使用其他模板创建集群，您还可以在快速创建页面的右上角，或者在自定义创建集群的右上角筛选模板，点击 **Fill Form** 按钮后，会自动根据模板内容填充表单。

![](/img/k3s/fill-form-with-templates.png)

### 集群管理

您可以在集群列表页查看和管理使用 AutoK3s 创建的 K3s 集群。

#### 添加节点

选中您要添加节点的集群，点击右侧下拉菜单中的 **Join Node** 按钮，在弹出的窗口中设置要添加的节点数量即可。

![](/img/k3s/join-nodes.png)

#### 集群升级

选中您要升级的集群，点击右侧下拉菜单中的 **Upgrade** 按钮，在弹出窗口中会显示您当前集群配置的版本信息，例如下图示例中，显示集群版本为 `v1.19.16+k3s1`。

![](/img/k3s/upgrade-cluster.png)

如果我们要将集群升级到 `v1.20.15+k3s1`，则修改 Version 为指定版本号即可；如果想要将集群升级到最新的stable版本，则删除掉 Version 指定的版本号。

![](/img/k3s/upgrade-cluster-to-specified-version.png)

选择要升级的版本后，点击 **Upgrade** 按钮，等待集群升级完成。

![](/img/k3s/upgrade-cluster-result.png)

#### Kubectl

如果您想操作 K3s 集群数据，可以点击右上角 **Launch Kubectl** 按钮，在下拉框中选择要操作的集群后，便可以在 UI 控制台操作选中的集群了。

![](/img/k3s/launch-kubectl.png)

#### 下载 Kubeconfig 文件

如果您想在其他地方管理集群，可以单独下载指定集群的 Kubeconfig 文件。点击指定集群右侧下拉菜单中的 **Download KubeConfig** 按钮，在弹出窗口中选择复制或下载文件。

![](/img/k3s/download-kubeconfig.png)

#### SSH

如果您想连接到远程主机进行操作，您可以在集群列表页面点击集群名称，进入详情页面，选择要连接的主机，点击右侧 **Execute Shell** 按钮。

![](/img/k3s/ssh-node.png)

#### 开启 kube-explorer dashboard

您可以在创建集群时，通过 Additional Options 选择 explorer 选项开启 kube-explorer 功能。

![](/img/k3s/launch-kube-explorer.png)

也可以通过右侧下拉菜单中选择 Enable Explorer 功能来开启 kube-explorer。

![](/img/k3s/enable-kube-explorer.png)

开启后，在集群列表会增加 dashboard 跳转链接按钮，点击跳转链接便可以访问 kube-explorer dashboard 页面了。

![](/img/k3s/access-kube-explorer.png)

#### 关闭 kube-explorer dashboard

对于已经开启了 kube-explorer 功能的集群，可以在右侧下拉菜单中选择 Disable Explorer 功能来关闭 kube-explorer 服务。

![](/img/k3s/disable-kube-explorer.png)

## CLI 使用说明

在使用命令之前，您需要设置访问凭证。

```bash
export ECS_ACCESS_KEY_ID='<access-key>'
export ECS_ACCESS_KEY_SECRET='<secret-access>'
```

如想了解更多参数，请运行`autok3s <sub-command> --provider alibaba --help`命令。

### 创建普通集群

运行以下命令，在阿里云 ECS 上创建并启动创建一个名为 “myk3s”的集群，并为该集群配置 1 个 master 节点和 1 个 worker 节点。

```bash
autok3s -d create -p alibaba --name myk3s --master 1 --worker 1
```

### 创建高可用 K3s 集群

创建高可用集群的命令分为两种，取决于您选择使用的是内置的 etcd 还是外部数据库。

#### 嵌入式 etcd（k3s 版本 >= 1.19.1-k3s1)

运行以下命令，在阿里云 ECS 上创建并启动创建了一个名为“myk3s”的高可用 K3s 集群。

```bash
autok3s -d create -p alibaba --name myk3s --master 3 --cluster
```

#### 外部数据库

在高可用模式下使用外部数据库，需要满足两个条件：

- master 节点的数量不小于 1。
- 需要提供外部数据库的存储路径。

所以在以下的代码示例中，我们通过`--master 2`指定 master 节点数量为 2，满足 master 节点的数量不小于 1 这个条件；且通过`--datastore "PATH"`指定外部数据库的存储路径，提供外部数据库的存储路径。

运行以下命令，在阿里云 ECS 上创建并启动创建了一个名为“myk3s”的高可用 K3s 集群：

```bash
autok3s -d create -p alibaba --name myk3s --master 2 --datastore "mysql://<user>:<password>@tcp(<ip>:<port>)/<db>"
```

### 添加 K3s 节点

请使用`autok3s join`命令为已有集群添加 K3s 节点。

### 普通集群

运行以下命令，为“myk3s”集群添加 1 个 worker 节点。

```bash
autok3s -d join --provider alibaba --name myk3s --worker 1
```

### 高可用 K3s 集群

```bash
autok3s -d join --provider alibaba --name myk3s --master 2 --worker 1
```

### 删除 K3s 集群

删除一个 k3s 集群，这里删除的集群为 myk3s。

```bash
autok3s -d delete --provider alibaba --name myk3s
```

### 查看集群列表

显示当前主机上管理的所有 K3s 集群列表。

```bash
autok3s list
```

```bash
NAME     REGION     PROVIDER  STATUS   MASTERS  WORKERS    VERSION
myk3s  cn-hangzhou  alibaba   Running  2        2        v1.19.5+k3s2
myk3s  ap-nanjing   tencent   Running  2        1        v1.19.5+k3s2
```

### 查看集群详细信息

显示具体的 K3s 信息，包括实例状态、主机 ip、集群版本等信息。

```bash
autok3s describe cluster -n <clusterName> -p alibaba
```

:::note 注意
如果使用不同的 provider 创建的集群名称相同，describe 时会显示多个集群信息，可以使用`-p <provider>`对 provider 进一步过滤。例如：`autok3s describe -n myk3s -p alibaba`。
:::

```bash
Name: myk3s
Provider: alibaba
Region: cn-hangzhou
Zone: cn-hangzhou-i
Master: 2
Worker: 2
Status: Running
Version: v1.19.5+k3s2
Nodes:
  - internal-ip: x.x.x.x
    external-ip: x.x.x.x
    instance-status: Running
    instance-id: xxxxx
    roles: etcd,master
    status: Ready
    hostname: xxxxx
    container-runtime: containerd://1.4.3-k3s1
    version: v1.19.5+k3s2
  - internal-ip: x.x.x.x
    external-ip: x.x.x.x
    instance-status: Running
    instance-id: xxxxxx
    roles: <none>
    status: Ready
    hostname: xxxxxx
    container-runtime: containerd://1.4.3-k3s1
    version: v1.19.5+k3s2
  - internal-ip: x.x.x.x
    external-ip: x.x.x.x
    instance-status: Running
    instance-id: xxxxxxxx
    roles: etcd,master
    status: Ready
    hostname: xxxxxxxx
    container-runtime: containerd://1.4.3-k3s1
    version: v1.19.5+k3s2
  - internal-ip: x.x.x.x
    external-ip: x.x.x.x
    instance-status: Running
    instance-id: xxxxxxx
    roles: <none>
    status: Ready
    hostname: xxxxxxx
    container-runtime: containerd://1.4.3-k3s1
    version: v1.19.5+k3s2
```

### Kubectl

集群创建完成后, `autok3s` 会自动合并 `kubeconfig` 文件。

```bash
autok3s kubectl config use-context myk3s.cn-hangzhou.alibaba
autok3s kubectl <sub-commands> <flags>
```

在多个集群的场景下，可以通过切换上下文来完成对不同集群的访问。

```bash
autok3s kubectl config get-contexts
autok3s kubectl config use-context <context>
```

### SSH

SSH 连接到集群中的某个主机，这里选择的集群为 myk3s。

```bash
autok3s ssh --provider alibaba --name myk3s
```

### 升级 K3s 集群

您可以使用以下命令将指定的 K3s 集群升级到 `latest` 版本

```bash
autok3s upgrade --provider alibaba --name myk3s --k3s-channel latest
```

如果您想将 K3s 集群升级到指定版本，可以指定 `--k3s-version`

```bash
autok3s upgrade --provider alibaba --name myk3s --k3s-version v1.22.4+k3s1
```

### 进阶使用

AutoK3s 集成了一些与当前 provider 有关的高级组件，例如私有镜像仓库、Terway、CCM 和 UI。

#### 配置私有镜像仓库

在运行`autok3s create`或`autok3s join`时，您可以通过传递`--registry /etc/autok3s/registries.yaml`以使用私有镜像仓库，例如：

```bash
autok3s -d create \
    --provider alibaba \
    --name myk3s \
    --master 1 \
    --worker 1 \
    --registry /etc/autok3s/registries.yaml
```

使用私有镜像仓库的配置请参考以下内容，如果您的私有镜像仓库需要 TLS 认证，`autok3s`会从本地读取相关的 TLS 文件并自动上传到远程服务器中完成配置，您只需要编辑`registry.yaml`即可。

```bash
mirrors:
  docker.io:
    endpoint:
      - "https://mycustomreg.com:5000"
configs:
  "mycustomreg:5000":
    auth:
      username: xxxxxx # this is the registry username
      password: xxxxxx # this is the registry password
    tls:
      cert_file: # path to the cert file used in the registry
      key_file:  # path to the key file used in the registry
      ca_file:   # path to the ca file used in the registry
```

#### 启用阿里云 Terway CNI 插件

实例的类型决定了 K3S 集群可以分配给集群 POD 的 EIP 数量，更多详细信息请参见[这里](https://www.alibabacloud.com/help/zh/doc-detail/97467.htm)。

```bash
autok3s -d create \
    ... \
    --terway "eni"
```

#### 启用阿里云 CCM(Cloud Controller Manager)

更多关于 Aliyun Cloud Provider 信息，请参考[Cloud Provider 帮助文档（中文版）](https://github.com/kubernetes/cloud-provider-alibaba-cloud/blob/master/docs/zh/usage.md)。

```bash
autok3s -d create \
    ... \
    --cloud-controller-manager
```

#### 启用 UI 组件

AutoK3s 支持两种 UI 组件，包括 [kubernetes-dashboard](https://github.com/kubernetes/dashboard) 和 [kube-explorer](https://github.com/cnrancher/kube-explorer)

##### 开启 kubernetes dashboard

```bash
autok3s -d create -p alibaba \
    ... \
    --enable dashboard
```

访问 Token 等设置请参考 [此文档](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md) 。

##### 开启 kube-explorer

```bash
autok3s explorer --context <context> --port 9999
```
您可以通过 http://127.0.0.1:9999 访问 kube-explorer dashboard 页面。
