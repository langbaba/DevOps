# Istio 使用说明


## VirtualService

VirtualService 详细介绍，参见官方文档：[VirtualService Reference - Istio Docs](https://istio.io/latest/docs/reference/config/networking/virtual-service)

### VirtualService 注意事项

1. hosts 属性可以使用 FQDN，也建议使用 FQDN
2. gateways 列表中的元素有三种可能：
   1. 网格内部流量：`mesh`，这也是默认的 gateway.
   1. 当前名字空间的网关：`<gateway-name>`，直接填网关名称。
   1. 别的名字空间的网关：` <gateway-namespace>/<gateway-name>`，比如 `my-namespace/my-gateway`
      1. gateway 名称不是域名 FQDN！而是使用 `/` 分隔的一个名称！

