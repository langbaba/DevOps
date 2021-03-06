# [Vault](https://github.com/hashicorp/vault)

Vault 和 Apollo 相比，最大的特点是它对敏感信息的支持非常完善：

1. 支持保存各种自定义信息、自动生成各类密钥，vault 自动生成的密钥还能自动轮转(rotate)
2. 支持接入各大云厂商的账号权限体系（比如阿里云RAM子账号），实现 APIKey/APISecret 的自动轮转。
3. 支持接入 kubernetes rbac 权限体系，通过 serviceaccount+role 为每个 Pod 单独配置权限。

## 一、部署 Vault

### 1. docker-compose 部署

>推荐用于本地开发测试环境

docker-compose 方式部署比较简单，暂时略过。。
大体流程和 helm 部署差不多，都需要部署、初始化并解封

### 2. 通过 helm 部署 vault 

>推荐用于生产环境

官方建议[通过 Helm 部署 vault](https://www.vaultproject.io/docs/platform/k8s/helm/run)，大概流程：

1. 在 `custom-vaults.yaml` 中添加自定义的 vault 配置，通过 helm 部署。
2. 初始化/解封 vault: vault 安全措施，每次重启必须解封。

vault 支持很多的数据存储方式，简单起见，下面的示例与 Apollo 一样使用 MySQL 做数据存储。

以 mysql 为后端存储的 HA 高可用的 `custom-vault.yaml` 可以在此文件夹中找到。

通过 helm 部署：

```
# 添加 valut 仓库
helm repo add hashicorp https://helm.releases.hashicorp.com
# 查看 vault 版本号
helm search repo hashicorp/vault -l | head
# 下载某个版本号的 vault
helm pull hashicorp/vault --version 0.7.0 --untar
# 安装/升级 valut
helm upgrade --install vault ./vault --namespace vault -f custom-values.yaml
```

### 2. 初始化(initalize)并解封(unseal) vault

>官方文档：[Initialize and unseal Vault - Vault on Kubernetes Deployment Guide](https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide?in=vault/kubernetes#install-vault)

通过 helm 部署 vault，默认会部署一个三副本的 StatefulSet，但是这三个副本都会处于 NotReady 状态，
接下来还需要手动初始化(initalize)并解封(unseal) vault，才能 `Ready`:

1. 第一步：从三个副本中随便选择一个，运行 vault 的初始化命令：`kubectl exec -ti vault-0 -- vault operator init`
   1. 初始化操作会返回 5 个 unseal keys，以及一个 Initial Root Token，这些数据非常敏感非常重要，一定要保存到安全的地方！
2. 第二步：在每个副本上，使用任意三个 unseal keys 进行解封操作。
   1. 一共有三个副本，也就是说要解封 3*3 次，才能完成 vault 的完整解封！

```shell
# 每个实例都需要解封三次！
## Unseal the first vault server until it reaches the key threshold
$ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 1
$ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 2
$ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 3
```

这样就完成了部署，但是要注意，**vault 实例每次重启后，都需要重新解封！也就是重新进行第二步操作！**

### 3. 设置自动解封

每次重启都要手动解封所有 vault 实例，实在是很麻烦。
为了简化这个流程，可以考虑配置 autounseal.

官方文档：[autounseal-transit](https://learn.hashicorp.com/tutorials/vault/autounseal-transit)

>不过貌似这个 autounseal 配置起来也很麻烦。。。

## 二、在 Kubernetes 中使用 vault 注入敏感配置

### 1. 向 vault 中添加敏感信息

vault 提供三种添加/更新敏感信息的方式：Web UI、HTTP API 以及 CLI。

比较方便的自动化方法，应该是使用 vault 的 python sdk: [hvac](https://github.com/hvac/hvac)

Web UI 适合手工操作，sdk 则适合用于搭建与更新测试环境的 vault 配置。

### 2. 将 vault 配置注入到 Pod 中

#### 2.1 部署并配置 vault

前面提到过 vault 支持通过 Kubernetes 的 ServiceAccount + Role 为每个 Pod 单独分配权限。

首先启用启用 Kubernetes 身份验证:

```shell
# 配置身份认证需要在 vault pod 中执行，启动 vault-0 的交互式会话
kubectl exec -n vault -it vault-0 -- /bin/sh
export VAULT_TOKEN='<your-root-token>'
export VAULT_ADDR='http://localhost:8200'
 
# 启用 Kubernetes 身份验证
vault auth enable kubernetes
 
# 添加 vault 用于访问 kube-apiserver API 的配置：jwt 内容、apiserver url、CA证书
vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

#### 2.2 关联 k8s rbac 权限系统和 vault

接下来需要做的事：

2. 通过 vault policy 定义好每个 role（微服务）能访问哪些资源。
1. 为每个微服务生成一个 kubernetes role，这个 role 需要绑定对应的 vault policy 及 kubernetes serviceaccount
3. 创建一个 ServiceAccount，并使用这个 使用这个 ServiceAccount 部署微服务

其中第一步和第二步都可以通过 vault api 自动化完成.
第三步可以通过 kubectl 部署时完成。

方便起见，vault policy / k8s role / k8s serviceaccount 这三个配置，都建议和微服务使用相同的名称。

>上述配置中，kubernetes role 起到一个承上启下的作用，它关联了 k8s serviceaccount 和 vault policy 两个配置。

比如创建一个名为 `my-app-policy` 的 vault policy，内容为:

```hcl
# 命名规则："<engine-name>/data/<path>/*"
path "my-app/data/*" {
   # 只读权限
   capabilities = ["read", "list"]
}
```

然后在 vault 中创建 k8s role `my-app-role`:
1. 关联 k8s default 名字空间中的 serviceaccount `my-app-account`，并创建好这个 serviceaccount.
2. 关联 vault token policy，这就是前面创建的 `my-app-policy`
3. 设置 token period（有效期）

这之后，每个微服务就能通过 serviceaccount 从 vault 中读取 `my-app` 中的所有信息了。

#### 2.3 部署 Pod

下一步就是将配置注入到微服务容器中，这需要使用到 Agent Sidecar Injector。
vault 通过 sidecar 实现配置的自动注入与动态更新。

具体而言就是在 Pod 上加上一堆 Agent Sidecar Injector 的注解，如果配置比较多，也可以使用 configmap 保存，在注解中引用。

需要注意的是 vault-inject-agent 有两种运行模式：

1. init 模式: 仅在 Pod 启动前初始化一次，跑完就退出（Completed）
2. 常驻模式: 容器不退出，持续监控 vault 的配置更新，维持 Pod 配置和 vualt 配置的同步。
