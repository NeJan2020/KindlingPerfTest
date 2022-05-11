# Kindling-benchmark 性能基准测试
## 1. 背景
Kindling-agent是基于eBPF的监控工具Kindling中的采集端组件，能够通过采集和分析内核事件，获取运行于同一宿主机上的其他服务的业务、网络等指标。其工作模式是在主机上以独立进程的方式收集所需数据，所以只需要我们在应用所在主机部署kindling-agent即可启动相应能力，随后可以通过prometheus和grafana套件对不同机器上的探针采集的数据进行整合分析和查看，当然也可以用其他工具获取数据并进行分析展示。

尽管Kindling-agent基于eBPF的方式进行的监控方式减少了对被监控应用的侵入，但始终还是和用户应用共享同一台宿主机的CPU,内存，磁盘，网络等资源。这使得所有想要使用Kindling-agent的用户都会想知道该工具在真实环境中的性能表现和预期资源使用。

Kindling项目提供了一系列的测试来验证该采集工具的性能表现，这些测试反应了Kindling-agent在不同压力下良好的性能表现和可靠性。我们的性能测试使用了常用的APM工具Skywalking的两个测试用例。
### 1.1 测试目标

1. 检验高负载(5k tps)场景下，Kindling-agent对应用的性能影响和agent本身的资源使用
1. 计算常规负载(1k tps)下，Kindling-agent对应用的性能影响和agent本身的资源使用
## 2.测试环境和结果说明
每一组测试中包含了以下信息：

1. 基线指测试应用在无探针安装时的进行压力测试获得的指标，包括以下信息: 
   1. machine-cpu: 机器总CPU使用总体百分比
   1. machine-mem: 机器总内存使用总体百分比
   1. application-cpu: 测试应用CPU使用核数
   1. application-memory: 测试应用内存使用
   1. application-latency: 测试应用请求延迟
   1. application-tps：测试应用每秒事务数
2. 安装探针后的测试应用在压力测试时的性能指标
2. 探针自身的性能损耗，包括CPU和内存使用；在一些较低内核版本的机器中，Kindling使用内核模块代替eBPF实现了相同的功能，你将会在测试中看到两种实现下不同的性能表现

以下测试用例中涉及到的测试应用，Jmeter和Kindling-agent都以K8s工作负载的方式进行部署，测试应用和Jmeter分别运行在两台CentOS7(fedora)，内核版本为3.10.0-1160.53.1，机器资源为8C16G , CPU 为
 Intel(R) Xeon(R) Platinum 8269CY CPU @ 2.50GHz
## 3.测试用例
### 3.1 用例1
为了验证Kindling-agent在高负载下的性能表现，用例1使用了Skywalking的benchmark1程序。该程序为一个常规的Springboot应用，对外提供http服务，其预期tps为5000，预期延时为85ms。

Kindling会捕获该程序的异常/慢的请求数据（即Trace），并统计程序统计时间段内的关键性指标（Metric），如平均响应时间、错误率、请求字节数和请求数等。这些Trace和Metric能够有效的保障程序的可观测性。

下面的测试结果中是待测程序在5000tps下的性能表现,baseline表示未启用agent下的资源开销和性能表现。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2353740/1650954978262-bab21429-de30-4d1a-b65e-d94808ba5ddf.png#clientId=ub29f4671-8d2b-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=478&id=u93dcdcf8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=956&originWidth=1966&originalType=binary&ratio=1&rotation=0&showTitle=false&size=133052&status=done&style=none&taskId=ua380d848-1ed5-4766-99e7-417652e59e6&title=&width=983)
在资源使用上，Kindling-Agent  一共消耗了约 0.64C / 90MB 来处理并统计 5000 tps下的关键性能指标，并通过Prometheus暴露在http接口上。

对于应用程序的资源使用，在基线测试中，应用程序需要花费2.5C处理现有的业务请求，在部署了探针后，程序需要使用2.6C处理现有的业务请求，即相对于基线增加了4%的额外开销；内存方面则几乎没有影响。

对于应用程序的服务表现，可以看到，即使在5000tps的负载下，Kindling-Agent对应用程序的响应时间和TPS的影响都非常微小。大多数正常的业务都包含一定的处理逻辑，单节点吞吐量很少能够达到5000tps。因此，对于大多数的业务应用来说，不需要担心Kindling-Agent对应用本身的处理能力造成影响。
### 3.2 用例2
如之前所述，用例1中的tps明显高于正常的用户应用。为此，测试用例2增加了处理每个请求时的CPU使用，并下调了请求压力,使该场景更接近于生产环境下的常规压力。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2353740/1650957292500-c4c59d06-d20c-4ade-85c5-127b04fd18dd.png#clientId=ub29f4671-8d2b-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=466&id=u9e0ef5a4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=932&originWidth=2106&originalType=binary&ratio=1&rotation=0&showTitle=false&size=132243&status=done&style=none&taskId=u090a9f60-be4d-4be1-b9b2-8128b93c6c0&title=&width=1053)
在资源使用上，Kindling-agent 一共消耗了 0.12C / 130MB 用于数据处理和统计。

对于应用的资源使用，在1000tps下，基线使用1.37C 处理现有的请求，安装agent后相较于基线几乎没有额外开销；服务表现方面，在1000tps下，基线的响应时间为272ms , tps 为 1044 ; 安装agent后相较于基线几乎不变。总的来说，在常规负载下，Kindling-agent对用户应用几乎没有影响。
## 4. 总结
上述用例说明了Kindling-agent出色的性能表现。在较低的资源开销下，它支持轻量化部署，易于管理；能够深入分析请求到协议栈在内核执行情况；能够提供语言无关，应用无侵入的监控体验，为你的应用带来新一代的可观测能力。
## 5. 附录
测试中使用到的所有资源使用指标均通过Jmeter的ServerAgent采集，应用响应延时和TPS通过Jmeter性能面板插件采集，非GUI模式下以jtf格式保存原始数据，后续通过 Jmeter的图像生成工具生成折线图并自行统计。
### 以工作负载方式部署Skywalking的测试用例
原始测试用例是直接以Java应用的方式运行并部署在物理机上，这里提供了相关的Dockerfile和deployment文件修改成K8s工作负载。
```dockerfile
FROM openjdk:8u322-jdk
ADD benchmark-3.tar.gz /opt
CMD ["/opt/benchmark-3/bin/startUp.sh"]
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: benchmark-1
  namespace: benchmark
spec:
  replicas: 1
  selector:
    matchLabels:
      app: benchmark-1
  template:
    metadata:
      labels:
        app: benchmark-1
    spec:
      containers:
      - image: ${your-harbor}/${your-project}/benchmark-1
        imagePullPolicy: IfNotPresent
        name: benchmark
```
### Kindling-agent切换至内核模式运行的方法：

- 在 Kindling-agent 的daemonset文件中，为kindling-probe容器添加环境变量 PREFER_KERNEL_PROBE=1，则会优先使用内核模式。示例如下：
```yaml
...
      containers:
      - name: kindling-probe
        image: kindlingproject/kindling-probe:latest
        ...
        env:
        - name: HOST_PROC
          value: /host/proc
        - name: PL_HOST_PATH
          value: /host
        - name: SYSDIG_HOST_ROOT
          value: /host
        - name: PREFER_KERNEL_PROBE
          value: 1
        securityContext:
          privileged: true
...
```

相关连接：

1. 测试用例源码和压测脚本 [https://github.com/SkyAPMTest/Agent-Benchmarks](https://github.com/SkyAPMTest/Agent-Benchmarks)
1. 资源指标采集工具 [https://jmeter-plugins.org/wiki/PerfMon/](https://jmeter-plugins.org/wiki/PerfMon/)
1. 应用响应延时和TPS展示 [https://jmeter-plugins.org/wiki/ResponseTimesOverTime/](https://jmeter-plugins.org/wiki/ResponseTimesOverTime/)
1. 图形生成工具 [https://jmeter-plugins.org/wiki/JMeterPluginsCMD/](https://jmeter-plugins.org/wiki/JMeterPluginsCMD/)
1. 探针测试版本 [https://github.com/Kindling-project/kindling/tree/release-0.2.0](https://github.com/Kindling-project/kindling/tree/release-0.2.0)