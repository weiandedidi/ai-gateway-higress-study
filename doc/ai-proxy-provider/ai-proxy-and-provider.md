maas是maas的部署
其中 部署ai代理需要有ai服务提供者

# 一、服务代理者的流量处理

```bash
客户端(OpenAI协议)
        │配置了ai的路由
        ▼
┌──────────────────────────┐
│ Ingress                  │
│ ai-route-xxx.internal    │
└──────────────────────────┘
        │找到插件实例
        ▼
┌──────────────────────────┐
│ WasmPlugin               │
│ ai-proxy-1.0.0           │
│ (路由绑定插件实例)         │
└──────────────────────────┘
        │找到provider的注册表（也是WasmPlugin，名字ai-proxy.internal）
        ▼
┌──────────────────────────┐
│ WasmPlugin               │
│ ai-proxy.internal        │
│ (AI Provider Registry)   │
└──────────────────────────┘
        │
        ▼
┌──────────────────────────┐
│ Provider                 │
│ base-higress-service-dev │
│ protocol=openai          │
└──────────────────────────┘
        │
        ▼
┌──────────────────────────┐
│ 实际后端AI服务             │
│ /inner/api/coding/v1     │
└──────────────────────────┘
```
# 二、各组件真正职责（核心）

| 组件               | K8S资源                         | 作用                     | 是否真正参与流量 | 备注                                                         |
| ------------------ | ------------------------------- | ------------------------ | ---------------- | ------------------------------------------------------------ |
| AI Route ConfigMap | ConfigMap                       | Console AI网关展示元数据 | ❌                | metadata.name 是ai路由的主键，详见console的源码com.alibaba.higress.sdk.service.ai.AiRouteServiceImpl#query 中configMapName拼装方法 |
| AI Route Ingress   | Ingress                         | 真正路由入口             | ✅                | com.alibaba.higress.sdk.service.ai.AiRouteServiceImpl#add > saveRoute(route) > com.alibaba.higress.sdk.service.RouteServiceImpl#add > createIngress |
| ai-proxy-1.0.0     | WasmPlugin                      | 给路由绑定AI代理插件     | ✅                |                                                              |
| ai-proxy.internal  | WasmPlugin                      | Provider注册中心         | ✅                |                                                              |
| providers[]        | ai-proxy.internal.defaultConfig | AI服务提供者定义         | ✅                | 名为ai-proxy.internal的WasmPlugin中defaultConfig配置         |
| provider.protocol  | provider配置                    | 定义协议转换方式         | ✅                | openai                                                       |
| provider.type      | provider配置                    | 指定厂商类型             | ✅                | openai qwen 等                                               |

# 三、用例各项目部署的顺序（从上到下）
1. ai提供者注册中心[ai-proxy.internal.yaml](helm%2Ftemplates%2Fai-proxy.internal.yaml)
2. ai代理插件实例[ai-proxy.yaml](helm%2Ftemplates%2Fai-proxy.yaml)
3. 编程套餐网关第二跳 ai网关的路由的configmap[ai-route-conding-plan-second-configmap.yaml](helm%2Ftemplates%2Fai-route-conding-plan-second-configmap.yaml)
4. 编程套餐网关第二跳 ai网关的路由的ingress[ai-route-conding-plan-second-ingress.yaml](helm%2Ftemplates%2Fai-route-conding-plan-second-ingress.yaml)
5. 编程套餐网关第一跳 服务网关[ctyun-ai-codingplan-distributor-ingress.yaml](helm%2Ftemplates%2Fctyun-ai-codingplan-distributor-ingress.yaml)
6. 编程套餐分发插件[ctyun-ai-codingplan-distributor-ingress.yaml](helm%2Ftemplates%2Fctyun-ai-codingplan-distributor-ingress.yaml)



