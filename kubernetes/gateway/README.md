# Gateway 网关

>网关如何暴露到集群外部：[kubernetes 如何将服务暴露到外部](../kubernetes%20如何将服务暴露到外部.md)

网关，就是指一个流量的集中式出入口。
Kubernetes 官方提供的网关抽象，叫 Ingress。Ingress 配置会被一个叫 Ingress Controller 的控制器读取，并负责具体的实现 。

Ingress Controller 有很多的社区实现可供选择，官方提供的是 Nginx-Ingress-Controller，但是它的功能欠佳。

这里着重介绍几个比较流行的社区实现，它们大都提供了一些更高级的特性，比如 JWT 身份验证、来源限流、或者直接提供了插件系统。

详细介绍参见：[API Gateway vs Service Mesh](https://www.cnblogs.com/kirito-c/p/12394038.html)

## 一、Ingress

目前最流行的貌似是 Traefik.

## 二、API Gateway

### 1. Kong

```shell
# jwt: secret(HS256) or public key(RS256)
kubectl create secret \
  generic admin-jwt  \
  --from-literal=kongCredType=jwt  \
  --from-literal=algorithm=HS256 \
  --from-literal=key="4" \
  --from-literal=secret="xxxx321"

# admin group，权限分组控制
kubectl create secret \
  generic admin-acl  \
  --from-literal=kongCredType=acl  \
  --from-literal=group=admin-group
```

```yaml
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: app-jwt  # use jwt plugin
plugin: jwt
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: admin-acl  # enable acl
plugin: acl
config:
  whitelist:  # allow pass
    - admin-group
  # blacklist: []  # whitelist or blacklist, cannot use both!
---
apiVersion: configuration.konghq.com/v1
kind: KongConsumer  # 消费者，可以对应微服务
metadata:
  name: admin
username: admin
credentials:  # 绑定 jwt 密钥，绑定 acl 群组
  - admin-jwt
  - admin-acl
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx
  annotations:
    plugins.konghq.com: app-jwt,admin-acl
spec:
  rules:
    - http:
        paths:
          - path: /
            backend:
              serviceName: nginx
              servicePort: 80
---
```

如此配置好后，kong 会根据 jwt 的状态，返回 40x，或者将请求路由到上游：

|request | proxied to upstream service |	response status code |
| --- | --- | --- |
|has no JWT   	|  no                 	|   401                |
|missing or invalid iss claim	|  no 	 |    401                |
|invalid signature 	|  no              	|    403               |
|valid signature 	  |  yes   | 	from the  upstream service    |
|valid signature, invalid verified claim optional	|  no |  401 |


#### 注意事项 - Kong 对比 Istio IngressGateway

Kong 的 jwt 认证行为和 Istio IngressGateway 不一致，IngressGateway 默认行为如下:

|request | proxied to upstream service |	response status code |
| --- | --- | --- |
|has no JWT   	|          yes    	|   来自上游服务（可以通过 authorization 插件 deny 掉这种请求，但是会返回 403!）    |
|missing or invalid iss claim	|  no 	 |    401                |
|invalid signature 	|  no              	|    401（这里也不一样）               |
|valid signature 	  |  yes   | 	from the  upstream service    |
|valid signature, invalid verified claim optional	|  no |  401 |

