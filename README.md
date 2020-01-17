# container-monitor-exporter
容器内部网络状态监控，文件系统监控  
## 开发初衷
在Kubernetes系统里，由kubelet内置的cadvisor组件收集每个容器资源监控信息，但官方基于性能相关的考虑，如果抓取这些每个容器中网络相关的指标，将会耗费大量的CPU内存资源，cadvisor中默认给关掉了网络等相关指标的收集   https://github.com/google/cadvisor/blob/master/docs/runtime_options.md#metrics  
https://github.com/kubernetes/kubernetes/issues/60279  
所以在prometheus默认抓取的kubelet cadvisor metrics端点，监控上报的数据中，容器网络相关的指标container_network_tcp_usage_total和container_network_udp_usage_total等都为0  
但实际业务监控中可能需要这些指标的收集，用于监控告警以及排查问题。

## 方案一：
如果想要从cadvisor中获取这些指标，需要修改kubelet文件里的配置，即需要修改kubelet源码并重新编译，这个并不native。

另一种方案是自己部署cadvisor daemonset，并开启tcp网络指标的收集，使用prometheus来抓取这些指标，但这样就是重复部署了cadvisor组件，并且经过测试，部署的cadvisor在64C384G运行有300个container的物理机上占用了5核CPU和5G内存的资源，资源占用巨大，这也是cadvisor默认关闭网络指标收集的原因。

同时prometheus的抓取配置文件也需要修改，不然会同时抓取生成两份cadvisor指标，具体的prometheus sd_config和relabel_config配置可参考这篇文章(https://godleon.github.io/blog/Prometheus/Prometheus-Relabel/)

经过以上测试，采用额外cadvisor部署并使用prometheus抓取的方案资源消耗巨大，会有重复指标的收集，并且需要自己再次从prometheus中查询处理并上传到open-falcon监控平台。

## 方案二：
想要获取每个容器内的网络连接指标，在每个容器中内置监控组件并不现实，并且容器的网络指标收集与虚拟机的网络指标收集有差异，监控agent并不通用。

我们知道容器的虚拟化是利用namespace把每个进程的网络做了隔离，使用nsenter工具可以在每个进程的网络空间内执行相关的网络操作命令。参考：https://stackoverflow.com/questions/40350456/docker-any-way-to-list-open-sockets-inside-a-running-docker-container

另外，项目容器名与宿主上设备的映射关系可以通过docker的api来查到，比如pid进程对应关系以及网卡设备对应关系等。

通过以上可知，自己手动编写脚本，只抓取需要的网络指标数据，在宿主机上定时执行获取每个应用容器的网络状态并格式化为监控数据格式上传到open-falcon监控平台中，做统一的监控告警。
以ss --summary为例，显示了各种状态的tcp连接情况，以及udp等连接情况，读取统计到的数据字段：
Total: 141267 (kernel 7503901)
TCP:   28090 (estab 80, closed 27999, orphaned 139, synrecv 0, timewait 6586/0), ports 0

Transport Total     IP        IPv6
*	  7503901   -         -
RAW	  0         0         0
UDP	  3         3         0
TCP	  91        91        0
INET	  94        94        0
FRAG	  0         0         0

在上述配置的机器(64C384G运行有300个container的物理机)中执行效率在分钟级。

使用ss命令获取网络状态信息较慢，考虑两个优化：

1、将串行执行每个容器的网络状态命令，改为多线程并发执行，控制并发数并注意CPU的消耗使用

2、将通过ss命令获取信息改为直接读取/proc/{pid}/net/sockstat文件获取对应进程的网络状态统计信息。


## 方案三：

使用方案二的优化版本，不使用ss命令来获取每个容器网络空间的网络状态，直接读取每个容器对应pid下的sockstat文件，即/proc/{pid}/net/sockstat文件，效率比ss命令快很多，并且通过docker api与docker交换，获取运行的每个容器对应的pod name和pid信息，效率比执行命令快很多。  

sockstat文件信息：

sockets: used 141118
TCP: inuse 89 orphan 96 tw 7181 alloc 21341 mem 13896
UDP: inuse 3 mem 116
UDPLITE: inuse 0
RAW: inuse 0
FRAG: inuse 0 memory 0

在上述配置的机器(64C384G运行有300个container的物理机)中执行效率在毫秒级(300ms左右)。  
所以该项目使用方案三实现。
