# container-monitor-exporter
容器内部网络状态监控，文件系统监控  
# 开发初衷
在Kubernetes系统里，由kubelet内置的cadvisor组件收集每个容器资源监控信息，但官方基于性能相关的考虑，如果抓取这些每个容器中网络相关的指标，将会耗费大量的CPU内存资源，cadvisor中默认给关掉了网络等相关指标的收集   https://github.com/google/cadvisor/blob/master/docs/runtime_options.md#metrics  
https://github.com/kubernetes/kubernetes/issues/60279  
所以在prometheus默认抓取的kubelet cadvisor metrics端点，监控上报的数据中，容器网络相关的指标container_network_tcp_usage_total和container_network_udp_usage_total等都为0  
但实际业务监控中可能需要这些指标的收集，用于监控告警以及排查问题。

## 方案一：
如果想要从cadvisor中获取这些指标，需要修改kubelet文件里的配置，即需要修改kubelet源码并重新编译，这个并不native。

另一种方案是自己部署cadvisor daemonset，并开启tcp网络指标的收集，使用prometheus来抓取这些指标，但这样就是重复部署了cadvisor组件，并且经过测试，部署的cadvisor在64C384G运行有300个container的物理机上占用了5核CPU和5G内存的资源，资源占用巨大，这也是cadvisor默认关闭网络指标收集的原因。

同时prometheus的抓取配置文件也需要修改，不然会同时抓取生成两份cadvisor指标，具体的prometheus sd_config和relabel_config配置可参考这篇文章(https://godleon.github.io/blog/Prometheus/Prometheus-Relabel/)

经过以上测试，采用额外cadvisor部署并使用prometheus抓取的方案资源消耗巨大，会有重复指标的收集，并且需要自己再次从prometheus中查询处理并上传到open-falcon监控平台。
