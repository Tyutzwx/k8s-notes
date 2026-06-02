# K8s部署Prometheus（含Grafana）实操笔记

## 一、部署环境

- K8s集群：3节点（1个master节点，2个node节点），版本v1\.35\.0

- 节点状态：所有节点均为Ready状态

- 部署目标：在K8s集群中部署Prometheus（监控核心）\+ Grafana（可视化面板），通过NodePort暴露服务，实现基础监控

## 二、部署过程

### 1\. 初始尝试（Helm部署，失败）

最初尝试通过Helm仓库部署，选择prometheus\-community官方仓库，执行命令如下：

```bash
# 添加官方仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
# 更新仓库（尝试跳过索引错误，失败）
helm repo update --skip-index-errors
```

报错：`Error: unknown flag: \-\-skip\-index\-errors`，原因是当前Helm版本不支持该参数，且国内网络无法正常访问国外Helm仓库，后续切换阿里云仓库仍因仓库过旧无对应Chart包失败，放弃Helm部署方案。

### 2\. 切换YAML部署（原生K8s资源）

放弃Helm后，采用原生YAML部署，核心步骤如下：

1. 创建监控命名空间（monitoring），用于隔离监控相关资源

2. 编写Prometheus \+ Grafana的Deployment和Service YAML文件，包含容器配置、端口暴露、节点调度容忍等核心配置

3. 应用YAML文件，启动服务

4. 查看Pod和Service状态，验证部署结果

### 3\. 最终成功状态

通过调整YAML配置（容忍节点污点、更换国内镜像），最终实现：

- Prometheus Pod正常Running，无重启

- Service正常暴露，NodePort端口：Prometheus（9090:32112）、Grafana（3000:30374）

- 浏览器可通过`http://节点IP:32112`访问Prometheus面板，部署核心目标达成

## 三、部署过程中遇到的问题及解决方案（重点）

### 问题1：Helm命令报错 unknown flag: \-\-skip\-index\-errors

- 报错信息：`helm repo update \-\-skip\-index\-errors` 执行后提示未知参数

- 原因：当前使用的Helm版本不支持`\-\-skip\-index\-errors`参数，且国外官方Helm仓库国内网络无法访问

- 解决方案：放弃Helm部署，改用K8s原生YAML部署，避免依赖国外仓库

### 问题2：Pod一直处于Pending状态

- 报错信息：`0/3 nodes are available: 3 node\(s\) had untolerated taint\(s\)`

- 原因：K8s节点（尤其是master节点）默认存在污点，禁止普通Pod调度，导致Pod无法分配到节点

- 解决方案：在Pod的spec中添加容忍配置，让Pod无视节点污点，可正常调度：
`tolerations:
\- operator: Exists`

- 补充：尝试删除节点污点（`kubectl taint nodes \-\-all 污点名称\-`）失败，因未知污点名称，最终通过容忍配置彻底解决调度问题

### 问题3：镜像拉取失败（ErrImagePull/ImagePullBackOff）

- 报错信息：`failed to pull image \&\#34;prom/prometheus\&\#34;: failed to resolve image: dial tcp 31\.13\.94\.23:443: connect: connection refused`

- 原因：使用国外Docker官方镜像（docker\.io），国内网络被墙，无法拉取镜像

- 解决方案：更换为阿里云国内镜像，示例：
        

    - Prometheus镜像：`registry\.cn\-hangzhou\.aliyuncs\.com/chenby/prometheus:v2\.40\.0`

    - Grafana镜像：`registry\.aliyuncs\.com/acs/grafana:latest`

- 补充：Grafana因环境网络限制仍未拉取成功，但不影响Prometheus核心功能，达成部署目标

### 问题4：Namespace/Pod卡住处于Terminating状态

- 报错信息：`Error from server \(AlreadyExists\): object is being deleted: namespaces \&\#34;monitoring\&\#34; already exists`，Pod一直处于Terminating状态无法删除

- 原因：删除命名空间时，旧Pod未正常清理，导致命名空间卡住，无法重新创建

- 解决方案：
        

    1. 强制删除卡住的Pod：
                `kubectl delete pod Pod名称 \-n monitoring \-\-force \-\-grace\-period=0`

    2. 更换新的命名空间（如monitor），避开卡住的monitoring命名空间，重新部署YAML

## 四、核心总结（面试重点）

1. K8s部署Prometheus的核心方式：推荐使用Deployment \+ Service部署，灵活控制副本数，通过NodePort暴露服务便于外部访问
2. Pod Pending核心排查方向：节点状态（Ready）、节点污点（容忍配置）、资源是否充足
3. 镜像拉取失败解决方案：优先使用国内镜像仓库（阿里云、华为云等），避免依赖国外仓库
4. 命名空间/Pod卡住解决：强制删除资源（\-\-force \-\-grace\-period=0），或更换命名空间重建
5. 本次部署核心成果：Prometheus正常运行，Service成功暴露，可通过浏览器访问，实现K8s集群基础监控


# Grafana是通过什么方式暴露到集群的节点上的

- **Grafana 是通过 K8s 的 NodePort 类型 Service 暴露到节点上的，这是最标准、最常用的方式。**

## 一、核心原理（一句话） 
把 Service 类型设为 NodePort → K8s 在所有节点开放同一个固定端口 → 用 任意节点 IP: 端口 直接访问 Grafana。   

## 二、完整流程（极简） 
1. Grafana 以 Deployment 运行在 Pod 里，端口 3000
2. 创建 Service，类型写 type: NodePort
3. kube-proxy 在每个节点都监听这个端口 
4. 访问任意节点 IP + NodePort 就能打开 Grafana   

## 三、最常用配置（直接背、直接用） 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort  # 关键！
  selector:
    app: grafana
  ports:
  - port: 3000        # 集群内部访问端口
    targetPort: 3000  # 容器端口
    nodePort: 30000   # 节点暴露端口（30000-32767）
```
访问方式：  http://节点IP:30000
      
## 四、面试必背问答 
Q：Grafana 怎么暴露到节点？ 
用 NodePort 类型的 Service，在每个节点开放固定端口，通过节点 IP + 端口访问。
    
Q：除了 NodePort 还有哪些方式？ 

- ClusterIP：只能集群内访问，不对外 
- LoadBalancer：云厂商负载均衡器 
-  Ingress：生产用域名 + HTTPS 访问 
-  port-forward：临时调试用  
	

Q：为什么用 NodePort？ 
- 所有节点都能访问 
- 配置最简单 
- 不需要云厂商支持 
- 测试 / 自建集群最常用    
	
## 五、一句话终极答案 
Grafana 通过创建 type: NodePort 的 Service，在 K8s 所有节点开放固定端口，实现用节点 IP + 端口直接访问。



# Kubernetes 纯 YAML 部署 Prometheus + Grafana 完整笔记（Markdown 可直接保存 / 提交）

部署方式：纯原生 Kubernetes YAML，一步一讲
实现目标：K8s 集群监控 + Node 指标采集 + 告警规则 + 可视化大盘

## 整体架构说明
部署顺序（必须按此）
所有 YAML 完整文件 + 逐行详解
一键部署与访问验证
Grafana 配置步骤
面试高频问题（含标准答案）
整套流程口述总结

## 1. 整体架构
```
K8s 所有节点
   ↓
Node Exporter（DaemonSet）：采集系统指标
   ↓
Prometheus：拉取指标、存储、计算告警
   ↓        ↓
Alertmanager：处理告警    
Grafana：可视化大盘
```

###1. 组件职责
	- Node Exporter：采集节点 CPU、内存、磁盘、网络、负载
	- Prometheus：时序数据库、指标拉取、告警规则判断
	- Alertmanager：告警分组、去重、抑制、通知发送
	- Grafana：监控图表、大盘展示、历史趋势
	- Service：提供固定访问地址（Pod IP 会变，Service 域名不变）
	- ConfigMap：存放 Prometheus 配置 + 告警规则

###2. 部署顺序
	- 创建 Namespace
	- 部署 Node Exporter
	- 创建 Prometheus ConfigMap
	- 部署 Prometheus（StatefulSet + Service）
	- 部署 Alertmanager
	- 部署 Grafana
	访问验证 + 配置 Grafana

###3. 完整 YAML 文件 + 逐段详解

####3.1  00-namespace.yaml
  **作用：创建独立命名空间，隔离监控资源**
  为什么用 YAML：声明式、可 Git 管理、可加标签、统一风格

```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
  name: monitoring
  labels:
    name: monitoring
```
 执行

```shell
kubectl apply -f 00-namespace.yaml
```
####3.2  01-node-exporter.yaml
**作用：在每个节点运行采集器，暴露系统指标**

  - 为什么用 DaemonSet？：每节点 exactly 一个 Pod
  - DaemonSet和Deployment的关系：
    关键配置：hostNetwork、挂载 /proc/sys
```yaml
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
  name: node-exporter
  namespace: monitoring
  spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.8.2
        ports:
        - containerPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
      volumes:
      - name: proc
        hostPath:
          path: /proc 
      - name: sys
        hostPath:
          path: /sys
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    app: node-exporter
  ports:
  - port: 9100
```

####3.3 02-prometheus-config.yaml
**作用：Prometheus 核心配置文件**
**包含：抓取间隔、告警规则、Alertmanager 地址、K8s 服务发现**
```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: prometheus-config
    namespace: monitoring
    data:
    prometheus.yml: |
    global:
      scrape_interval: 15s

    rule_files:
    - /etc/prometheus/rules.yml

    alerting:
      alertmanagers:
      - static_configs:
        - targets: ["alertmanager:9093"]

    scrape_configs:
    - job_name: node-exporter
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_label_app]
        regex: node-exporter
        action: keep

    - job_name: kubernetes-nodes
      kubernetes_sd_configs:
      - role: node
 
  rules.yml: |
    groups:
    - name: node-alert
      rules:
      - alert: NodeDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "节点宕机"
          description: "实例 {{ $labels.instance }} 宕机 1 分钟"

      - alert: CPUHigh
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "CPU 使用率过高"
          description: "当前值: {{ $value }}%"
    
      - alert: MemoryHigh
        expr: 100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100) > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "内存使用率过高"
          description: "当前值: {{ $value }}%"
    
      - alert: DiskFull
        expr: 100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100) > 85
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "磁盘使用率过高"
          description: "当前值: {{ $value }}%"
```

####3.4 03-prometheus.yaml（重点：为什么两部分）
文件包含两个资源，用 --- 分隔
- StatefulSet：运行 Prometheus 容器、数据持久化
- Service：提供固定访问地址（域名不变）

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceName: prometheus
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.53.0
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: data
          mountPath: /prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: prometheus
  ports:
  - port: 9090
```

为什么写两部分？
- StatefulSet：负责运行程序 + 存数据（身体）
- Service：负责固定访问地址（嘴巴）
  用 --- 可以在一个文件里部署两个资源
  方便一键部署、结构统一、便于管理
  
 #### 3.5 04-alertmanager.yaml
 **作用：接收告警、分组、去重、路由、发送通知**
```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: alertmanager
    namespace: monitoring
    spec:
    replicas: 1
    selector:
    matchLabels:
      app: alertmanager
    template:
    metadata:
      labels:
        app: alertmanager
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager:v0.27.0
        ports:
        - containerPort: 9093
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  selector:
    app: alertmanager
  ports:
  - port: 9093
    3.6 05-grafana.yaml
    作用：可视化大盘、展示监控图表
    yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: grafana
    namespace: monitoring
    spec:
    replicas: 1
    selector:
    matchLabels:
      app: grafana
    template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:10.2.0
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort      
  selector:
    app: grafana
  ports:
  - port: 3000
```

### 4. 一键部署 & 访问

####4.1 一键部署
```bash
kubectl apply -f .
```

#### 4.2 查看 Pod 状态
```bash
kubectl get pod -n monitoring -w
```
#### 4.3 访问 Prometheus
```bash
kubectl port-forward -n monitoring svc/prometheus 9090:9090
```
访问：http://127.0.0.1:9090

####4.4 访问 Grafana
```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
````
访问：http://127.0.0.1:3000
默认账号：admin / admin

### 5. Grafana 配置步骤
左侧 → Configuration → Data sources → Add data source
选择 Prometheus
URL 填写：http://prometheus:9090
点击 Save & Test
左侧 → Dashboards → Import
输入仪表盘 ID：1860（Linux 节点监控）
选择 Prometheus 数据源，导入即可

### 6. 面试高频问题（内置标准答案）
6.1 为什么 Node Exporter 用 DaemonSet？

- 确保每个节点只运行一个采集 Pod，适合监控、日志类组件。

6.2 为什么 Prometheus 用 StatefulSet 不是 Deployment？

- Prometheus 是有状态应用，需要稳定存储、稳定网络标识、数据持久化。

6.3 一个 YAML 里为什么用 --- 分隔？

- 表示同一个文件中定义多个 Kubernetes 资源，方便统一管理、一键部署。

6.4 为什么需要 Service？

- Pod IP 会变化，Service 提供固定域名与稳定访问入口。

6.5 什么是 K8s 服务发现？

- Prometheus 自动发现 Node、Pod、Endpoint，无需手动写死 IP。

6.6 Prometheus 与 Alertmanager 关系？

- Prometheus：计算告警是否触发
- Alertmanager：处理告警、分组、去重、发送通知

6.7 告警规则里 for 字段作用？

- 持续满足条件一段时间才触发，避免指标抖动、误报。

6.8 为什么 Namespace 要写 YAML？

- 统一声明式部署、可 Git 版本控制、可重复、可扩展标签。

### 7. 整套流程口述总结
- 我在 Kubernetes 中使用纯 YAML 部署监控系统：
	创建 monitoring 命名空间
	用 DaemonSet 部署 Node Exporter 采集所有节点指标
	通过 ConfigMap 配置 Prometheus 抓取规则与告警规则
	用 StatefulSet 部署 Prometheus 实现指标存储与告警计算
	部署 Alertmanager 处理告警通知
	部署 Grafana 对接 Prometheus 并导入仪表盘
	最终实现 K8s 集群与节点统一监控、可视化展示、智能告警。

## 普罗米修斯是怎么获取exporter的数据指标的

**Prometheus 是主动拉取（Pull）模式，定期 HTTP 请求 Exporter 的 /metrics 接口，获取指标数据并存到时序数据库。**

#### 8.1 完整流程
- 发现目标
	普罗米修斯通过 kube_sd_configs 连接kube-apiserver，实时监听pod，Node，Service资源变化，自动生成抓取目标列表
	
- Exporter 暴露指标
	Exporter 是一个独立进程（如 node_exporter、mysqld_exporter）
	它会把监控指标以**固定文本格式metrics**暴露在 http://ip:port/metrics 接口上
	
- Prometheus 配置抓取目标
	在 prometheus.yml 里配置：

```yaml
scrape_configs:
  - job_name: node_exporter
    static_configs:
      - targets: ['192.168.1.10:9100']
```

- Prometheus 主动 Pull 拉取
	按照配置的 scrape_interval（默认 15s）
	定时 HTTP GET /metrics

- 解析返回的指标（如 node_cpu_seconds_total、node_memory_MemTotal_bytes）
	存储到时序数据库
	指标格式：指标名{标签=值} 数值
	存入 Prometheus 内置的 TSDB 时序库
	
- 之后可以通过 PromQL 查询、Grafana 展示

#### 8.2高频面试考点
1. Prometheus 是 Push 还是 Pull？为什么？
- **Pull 模式**
	1. 优点：
	- 便于控制抓取频率
	- 服务端集中管理，更易排查
	- 被监控端无需感知监控系统
	- 天然支持服务发现
	2. 缺点：
	- 要打通网络（Prometheus 能访问 Exporter）
	- 短暂任务（一次性脚本）不适合，这时用 Pushgateway
2. 数据格式是什么？
纯文本格式，类似：
```shell
node_load1 0.55
node_cpu_seconds_total{cpu="0",mode="idle"} 123456.7
```

3. 如果 Exporter 挂了会怎样？
Prometheus 抓取失败
指标消失
可以配置告警规则（up == 0）发送告警

4. 什么是 PushGateway？适用场景？
用于短暂任务、批处理任务、无法被拉取的任务。
**流程：业务 → 推送指标 → PushGateway ← Prometheus 拉取**

###最精简高分总结
Prometheus 采用 ** 主动拉取（Pull）** 模式，通过在配置文件中定义 Exporter 地址，定期 HTTP 请求 Exporter 的 /metrics 接口获取指标文本数据，解析后存入自身时序数据库，供查询和告警使用。

## 普罗米修斯数据存储
- 测试环境中：本地存储，数据直接存在本地磁盘上
- 生产环境中：一般会用Thanos或者M3DB
