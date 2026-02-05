1. doris 在 helm 中可插拔，可以使用外部 doris
2. otel-collector  gateway 需要
   1. 能够追踪 otel-collector 产生的日志 
   2. 实现高可用以及对应的负载均衡
- sts 方式去管理多 pod，提供过对 sts 的配置 svc 实现负载均衡，通过 sts 管理 pvc 的模板
实现日志的持久化
   