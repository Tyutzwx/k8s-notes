# Kubernetes（K8s）节点组件详解

Kubernetes 集群分为**控制平面节点（Control Plane）** 和**工作节点（Worker Node）** 两类，不同节点承载的核心组件不同，以下用 Markdown 清晰梳理各节点的核心组件及作用。

## 一、控制平面节点（Master 节点）

控制平面是集群的 “大脑”，负责全局决策（如调度、集群状态管理）和 API 交互，核心组件如下：

| 组件名称                 | 核心作用               | 关键说明                                                     |
| :----------------------- | :--------------------- | :----------------------------------------------------------- |
| kube-apiserver           | 集群唯一入口           | 所有操作（如创建 Pod、查看节点）都通过 API Server 完成，提供认证、授权、数据校验等能力，是组件间通信的枢纽。 |
| etcd                     | 集群数据库             | 存储集群所有状态数据（如 Pod 配置、节点信息、服务规则），是集群的 “数据中心”，需保证高可用（通常部署 3/5 节点集群）。 |
| kube-scheduler           | 调度器                 | 负责将未调度的 Pod 分配到合适的工作节点，调度依据包括节点资源（CPU / 内存）、标签、亲和性 / 反亲和性规则等。 |
| kube-controller-manager  | 控制器管理器           | 整合多种核心控制器（如 Node Controller、ReplicaSet Controller、Namespace Controller），持续监控集群状态，确保实际状态与期望状态一致（例如 Pod 挂掉后自动重建）。 |
| cloud-controller-manager | 云控制器管理器（可选） | 对接云厂商（如阿里云、AWS）的基础设施服务，实现节点管理、负载均衡、存储卷创建等云原生能力，仅在公有云环境中部署。 |

## 二、工作节点（Worker 节点）

工作节点是集群的 “手脚”，负责运行实际的应用容器，核心组件如下：

| 组件名称                        | 核心作用     | 关键说明                                                     |
| :------------------------------ | :----------- | :----------------------------------------------------------- |
| kubelet                         | 节点管家     | 运行在每个工作节点上，接收 API Server 的指令，管理本机的 Pod 生命周期（创建、启动、停止、监控 Pod），并向控制平面上报节点和 Pod 状态。 |
| kube-proxy                      | 网络代理     | 维护节点上的网络规则（如 iptables/IPVS 规则），实现 Pod 之间、Pod 与外部服务的网络通信，负责 Service 的负载均衡。 |
| 容器运行时（Container Runtime） | 容器执行环境 | 负责运行容器，K8s 支持的主流运行时有：- Docker（传统主流，需配合 cri-dockerd）- containerd（轻量、高性能，目前主流）- CRI-O（专为 K8s 设计的轻量运行时）。 |

补：容器运行时：
1. CRI（容器运行时接口，高层）：对接kubelet，负责镜像拉取，容器的生命周期管理，有continerd、CRI-O
2. OCI（开放容器标准，底层）：真正创建、启动容器的执行引擎，有runc（默认对接continerd）、crun、kata（沙箱技术）、gvisor（沙箱技术）


## 三、可选组件（跨节点 / 辅助组件）

以下组件不严格绑定某类节点，但对集群功能至关重要：

| 组件名称           | 核心作用       | 部署位置                                                     |
| :----------------- | :------------- | :----------------------------------------------------------- |
| CoreDNS            | 集群 DNS 服务  | 通常部署在工作节点，为 Pod 提供域名解析（如 Service 名称解析为 ClusterIP）。 |
| Ingress Controller | 南北向流量入口 | 部署在工作节点，通过 Ingress 资源管理外部访问集群内服务的规则（如域名转发、HTTPS 配置）。 |
| kubelet            | 节点管家       | 运行在每个工作节点上，接收 API Server 的指令，管理本机的 Pod 生命周期（创建、启动、停止、监控 Pod），并向控制平面上报节点和 Pod 状态。 |

------

### 总结

1. **控制平面核心**：API Server（入口）、etcd（存储）、调度器（分配 Pod）、控制器管理器（保证状态）是核心，负责集群决策和管理。
2. **工作节点核心**：kubelet（管理 Pod）、kube-proxy（网络代理）、容器运行时（运行容器）是核心，负责实际业务负载的执行。
3. **组件分工**：控制平面聚焦 “决策和管理”，工作节点聚焦 “执行和运行”，可选组件（如 CoreDNS、Ingress）补充集群网络能力。0

#k8s中各个组件是怎样协调的

## 6、先讲 K8s 核心架构

1. **Master 节点**
   - **kubectl**：客户端，发送命令（命令行核心工具，非组件）
   - **API Server**：集群唯一入口，认证、鉴权、操作 etcd
   - **Controller Manager**：管理控制器（Deployment 控制器）
   - **Scheduler**：调度器，为 Pod 选 Node
   - **etcd**：键值存储，保存集群所有状态数据
2. **Node 节点**
   - **kubelet**：管理本机 Pod，跟 APIServer 通信
   - **kube-proxy**：维护网络规则、Service 负载均衡
   - **容器运行时**：docker/containerd，拉镜像、启动容器


你执行 `kubectl apply -f deployment.yaml` 后，**整个流程按这 7 步走**：
### 1. **kubectl 向 API Server 发送创建请求**
- 客户端将 Deployment 配置提交给 **API Server**
- API Server 做认证、鉴权、校验
- **将 Deployment 资源存入 etcd**
### 2. **Deployment Controller 监听资源变化**
- Controller Manager 中的 **Deployment 控制器** 
- 发现新增 Deployment → **创建对应的 ReplicaSet**
- 将 ReplicaSet 写入 etcd
### 3. **ReplicaSet Controller 开始工作**
- 监听 ReplicaSet
- 根据 `replicas: 3` 这种副本数
- **计算期望状态和当前状态差 → 创建对应数量的 Pod**
- 只创建 Pod 定义，**不调度、不运行**，存入 etcd
### 4. **Scheduler 为 Pod 选择 Node（调度）**
- 调度器监测到 **未调度的 Pod**
- 通过预选、优选算法：
  - 资源充足？
  - 节点亲和？
  - 污点容忍？
- 最终给 Pod 绑定一个 Node
- **将 Pod .spec.nodeName 写入 etcd**
### 5. **目标节点的 kubelet 监测到自己的 Pod**
- kubelet 一直监听 API Server
- 发现有 Pod 调度到自己节点
- 开始工作：
### 6. **kubelet 让容器运行时创建容器**
- 调用 **containerd/docker**
- 拉取镜像
- 创建并启动容器
- 上报状态给 API Server
### 7. **Pod 状态更新**
- kubelet 持续上报：
  - 容器状态
  - Pod IP
  - 运行状态
- API Server 将状态更新到 etcd
- **最终 Pod 进入 Running 状态**
### 总结
**用户提交 Deployment → API Server 存储 → Controller 创建 RS → RS 创建 Pod → Scheduler 调度 Pod → 目标节点 kubelet 启动容器 → 最终 Pod 运行。**

###简要概括
1. **kubectl** 发请求给 **APIServer**
2. **Deployment 控制器** 创建 **RS**
3. **RS 控制器** 创建 **Pod**
4. **Scheduler** 给 Pod 选 **Node**
5. **kubelet** 启动容器，Pod 变成 **Running**

#k8s中pod一直pending

Pod 处于 **Pending** 本质是：**调度器（Scheduler）找不到合适的 Node 来运行它**。

按下面四步排查，99% 的问题都能定位，直接背这套逻辑：

### 1. 先看事件（Event）—— 最直接的线索

```
kubectl get events -n 你的命名空间
```

- **重点看**：`Reason` 字段和 `Message` 字段。
- 如果显示 `FailedScheduling`，说明调度失败。

### 2. 查资源不足（最常见原因）

用 `kubectl describe pod 你的Pod名` 看详细事件，通常会报：

- `Insufficient cpu` / `Insufficient memory`

  - Node 资源不够，抢不到 CPU / 内存。
  - **解决**：精简 Pod 资源请求（requests），或给 Node 扩容。

- `Too many pods`

  - 这个 Node 已经达到最大 Pod 数上限（通常由容器运行时决定）。

  

### 3. 查节点亲和性 / 污点与容忍（Toleration/Taint）

- `MatchNodeSelector failed`

  - Pod 的 NodeSelector 选不到符合标签的 Node。
  - 用 `kubectl get nodes --show-labels` 查看节点标签。

- `PersistentVolumeClaim 未绑定`

  - Pod 申请了 PVC，但没有可用的 PV 绑定，一直处于 Pending。
  - 用 `kubectl get pvc` 查看状态。

- `No scheduleable nodes`

  （通用报错）

  - 可能是节点有**污点（Taint）**，但 Pod 没有对应的**容忍（Toleration）**。
  
  - 用 `kubectl get nodes -o jsonpath='{.items[*].spec.taints}'` 查看节点污点
  

 一、节点亲和性（nodeAffinity）
    放在：spec.template.spec.affinity.nodeAffinity 

```
    yaml        apiVersion: apps/v1
    kind: Deployment
    spec:
      template:
        spec:
          #### 亲和性写在这里
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                    - ssd
```

一句话位置：Pod 模板的 spec 下，和 containers 同级，写 affinity → nodeAffinity   



二、污点（Taint） 污点不是写在 Deployment 里，是打在 Node 节点上的：
	运行kubectl taint nodes node1 key=value:NoSchedule查看节点污点



三、容忍

```
    yaml        
    apiVersion: v1
    kind: Node
    metadata:
      name: node1
    spec:
      ### 污点写在 node.spec.taints
      taints:
      - key: app
        value: backend
        effect: NoSchedule
          三、容忍（Tolerations） 容忍是Pod 级别的配置，放在：spec.template.spec.tolerations yaml        apiVersion: apps/v1
        kind: Deployment
        spec:
        template:
        spec:
          ### 容忍直接写在 pod.spec 下
          tolerations:
          - key: "key"
            operator: "Equal"
            value: "value"
            effect: "NoSchedule"
```

1. 节点亲和性 nodeAffinity→ spec.template.spec.affinity.nodeAffinity 
2. 污点 Taint→ 不在 Deployment，在 Node.spec.taints 
3. 容忍 Tolerations→ spec.template.spec.tolerations


### 4. 查网络插件（CNI）配置

如果是 初始化容器（Init Container）一直失败，也可能导致 Pod 挂在 Pending。

- 检查 CNI 插件（如 Calico/Flannel）是否正常运行。
- 用 `kubectl get pods -n kube-system | grep cni` 查看。


### 5.检查pv和pvc

检查 PVC 状态：kubectl get pvc

- 状态为 Pending → 存储没就绪
- 检查是否有可用 PV：kubectl get pv
- 检查 StorageClass 是否存在、是否正确
- 检查 PVC 申请的大小、权限是否有 PV 能满足



# 运行中 Pod 反复重启排查思路



### 1. 先看状态
```
kubectl get pod <pod-name>
```

出现 **`CrashLoopBackOff`**，说明容器启动后又退出，反复循环。

------

### 2. 看重启次数、退出码（最关键）

```
kubectl describe pod <pod-name>
```

重点看：

- **Last State**
- **Exit Code**

常见退出码含义：

- **0**：正常退出（说明镜像启动命令本身就执行完就退出，不是后台运行）
- **1**：程序错误（代码异常、配置错）
- **127**：命令不存在 / 镜像里没这个文件
- **137**：OOM killer 杀死（内存不够）
- **143**：正常终止信号

------

### 3. 看容器日志（定位真实原因）

```
kubectl logs <pod-name>
# 看上次崩溃的日志
kubectl logs -p <pod-name>
```

能看到：

- 代码报错
- 配置文件错误
- 连不上数据库 / Redis
- 端口被占用
- 权限不足

------

### 4. 常见根本原因

1. **应用本身启动失败**

   代码异常、配置错误、依赖服务（DB、MQ）没起来。

   

2. **进程不是前台运行**

   容器启动命令执行完就退出，导致容器认为服务结束，反复重启。

   例如：脚本执行完就退出，没有常驻进程。

   

3. **内存不足 OOM**

   资源限制 `resources.limits.memory` 太小，被系统杀死。

   出现 OOMKilled 本质是容器内存使用超过了 limits 限制，被系统 OOM killer 杀掉。我会从排查定位 → 分析原因 → 解决方案三步处理：
   1. 先确认现象
   用 kubectl describe pod 查看状态，确认是 OOMKilled，退出码 137。
   查看事件 kubectl get events，确认是内存超限触发。
   用 kubectl top pod 看当前内存使用，对比 limits 和 requests。
   2. 定位根因
   常见三种原因：
   内存限制设置过低：正常业务流量下就接近上限。
   应用内存泄漏：内存持续上涨，不回落。
   突发流量 / 大请求：瞬间内存飙升超过限制。
   3. 解决与优化
   合理设置资源限制
   调整 resources.limits.memory，给足缓冲；requests 按正常水位设置，保证调度。
   Java 应用要注意 -Xmx 不要超过 limits 的 80%，避免堆外内存导致 OOM。
   排查内存泄漏
   进入容器分析堆内存，比如 Java 用 jmap、jstack；Go 用 pprof，定位泄漏点并修复代码。
   避免单点压力
   配置 HPA 水平扩容，大请求做分页、异步、限流，防止瞬时内存打满。
   优化 QoS
   核心业务设置成 Guaranteed（request = limit），提高稳定性，避免被优先驱逐。

   

4. **健康检查失败**

   livenessProbe （存活探针）一直失败，kubelet 强制杀容器重启。


```
   yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx
           ports:
           - containerPort: 80

           # ✅ 存活探针 写在这里！
           livenessProbe:
             httpGet:
               path: /
               port: 80
             initialDelaySeconds: 10
             periodSeconds: 5
             timeoutSeconds: 1
             failureThreshold: 3
```


5. **端口冲突、文件权限不足、磁盘满**

   写不了日志、目录权限不对。

# K8s 到底靠哪些机制保证高可用（HA）？

K8s 保证高可用，核心就两件事
1. 自身控制面不挂（etcd + apiserver 多副本）
2. 业务 Pod 不挂、挂了自动恢复（自愈 + 多副本 + 调度）

### 一、高可用 = 服务不中断

某台服务器宕机 → 服务还能访问
某个容器崩了 → 自动重启
- 流量突增 → 自动扩容
- 发布新版本 → 不停机
- 数据库 / 中间件 → 不单点
- K8s 就是围绕这些做的一套自动保障体系。


### 二、K8s 自身控制面高可用
K8s 自己的组件：
apiserver / scheduler / controller-manager / etcd
这些是 “大脑”，大脑不能单点。

1. etcd 高可用（最重要）
	etcd 是 K8s 的唯一数据源，所有配置都存在这里。
	必须部署 3/5/7 节点奇数集群
	采用 Raft 协议，半数以上存活就能正常工作
	挂 1 个节点完全不影响
	→ 保证数据不丢、服务不停
2. kube-apiserver 多副本
	apiserver 是所有操作入口。
	部署 多个 apiserver
	前面挂负载均衡（LB/nginx/haproxy）
	任何一个挂了，流量自动切走
	→ 入口不会单点
3. controller-manager、scheduler 多副本 + 竞选
	启动多个，但同一时间只有一个干活	
	主节点挂了，自动选新的主
	→ 调度和控制逻辑不会断

### 三、业务应用高可用（Pod 层面）
我的服务怎么不挂？
1. 多副本（ReplicaSet / Deployment）
	你不会只跑 1 个 Pod，而是跑 3 个：
	yaml
	replicas: 3
	任何一个挂了，还有另外两个顶着
	流量自动分摊→ 单点变多点，天然高可用
	
2. 自愈能力（Self-healing）
	K8s 持续监控每个 Pod：
	Pod 挂了 → 自动重启
	节点宕机 → Pod 自动漂移到别的节点	
	应用假死、不响应 → 杀掉重建
	自愈是 K8s 高可用的灵魂。
	
3. 健康检查（存活探针 + 就绪探针）
	靠两个探针判断是否正常：
	livenessProbe：死没死？死了就杀
	readinessProbe：能不能扛流量？不能就先摘掉
	保证：
	不把流量转发给异常 Pod
	有问题自动剔除
	
4. 节点亲和 / 反亲和
	强制：同一个服务的多个副本不跑在同一台机器
	不跑在同一个机架
	即使一台机器 / 机架断电，服务依然可用。
	
5. 滚动更新（Rolling Update）
	发布新版本时：
	起新 Pod → 正常后 → 再杀旧 Pod
	始终保持有足够副本在线
	→ 发布不停机，发布出问题还能一键回滚。
	
### 四、流量层面高可用（外部能访问）
1. Service 负载均衡
	Service 固定 IP
	自动代理到后端多个 Pod
	任何 Pod 挂了，自动从转发列表剔除
	→ 流量永远只打正常实例
2. Ingress + 外部 LB
	入口高可用：
	多副本 Ingress
	前面加云厂商 LB / 硬件负载均衡→ 入口不会单点
	
### 五、存储高可用（数据不丢）
K8s 本身不存数据，但提供：PV/PVC 抽象
对接 Ceph / 云盘 / NAS / 共享存储，保证：Pod 漂移到别的节点，数据依然跟着走
存储本身做副本、冗余→ 数据不丢、服务可恢复

### 六、用最通俗的比喻总结整套高可用
把 K8s 看作一个医院体系：
etcd = 病历档案中心（多副本，绝不丢）
apiserver = 挂号收费窗口（多窗口，一个坏了换一个）
Node = 病房楼
Pod = 病人
Deployment = 管病人的护士站：时刻清点人数，少了立刻补，病重立刻换
Service = 导诊台，永远告诉你哪个病人能看病
任何环节坏了，都不影响整体运转。

### 七、极简面试背诵版（超好用）
K8s 高可用主要靠这 6 点：
控制面多副本：apiserver 多实例 + LB
etcd 集群：Raft 协议保证数据高可用
多副本部署：Deployment 保证业务多点
健康检查 + 自愈：挂了自动重启、漂移
滚动更新：发布不停机，支持回滚
Service 负载均衡：流量自动分发、剔除异常实例

# k8s中的网络
### 一、K8s 自身不实现网络

1. 只定义规范：CNI **网络插件** 
2. 网络插件必须满足 K8s 基础网络模型： 
	◦ 所有 Pod 可以直接通信（扁平网络） 
	◦ 每个 Pod 独立 IP 
	◦ 节点上的容器与 Pod 可通信   

## K8s Pod 之间靠什么通信
一句话核心：**同一节点靠网桥 + veth pair，跨节点靠集群网络（CNI）+ 二层 / 三层转发，本质都是IP 通信。**
### 1. 基础前提
每个 Pod 都有独立 IP（Pod IP）
Pod 之间默认全通（不考虑 NetworkPolicy）
通信协议：TCP/UDP/ICMP，和普通虚拟机一样
### 2. 同一 Node 上的 Pod 通信
每个 Pod 有一对 veth pair（虚拟网卡对）
一端在 Pod 网络命名空间，一端挂在 cni0/bridge 网桥
数据包通过内核网桥转发，不经过宿主机网卡，直接走内核
### 3. 不同 Node 上的 Pod 通信
由 CNI 插件实现，主流两种模式：
- （1）Overlay 网络（隧道模式）
Flannel（VXLAN）、Calico IPIP
原理：Pod 数据包封装成宿主机 IP 包，跨节点传输到对端后解包
优点：兼容任何底层网络，部署简单
缺点：有封装开销
- （2）Underlay 网络（BGP 路由）
Calico BGP
原理：每个 Node 广播自己的 Pod CIDR 路由，直接三层路由转发
优点：性能高，无封装开销
缺点：底层网络需允许 BGP

###4. 详细讲解pod通信
这是一个非常核心的问题，理解了 `veth pair`和 `网桥`，就理解了容器网络互联的基石。

简单来说，你可以把它们想象成：

- **`veth pair`**：一根虚拟的**网线**，用于连接两个独立的网络空间。

- **`cni0`/ 网桥**：一台虚拟的**交换机**，用于将多根“网线”连接在一起，让它们能互相通信。

下面我们来详细拆解。

### 1. veth pair（虚拟以太网设备对）

**是什么？**

`veth`是 Linux 内核提供的一种**虚拟网络设备**。它总是成对出现，就像一根网线的两端，因此叫 `veth pair`。从一端（`veth0`）进入的数据包，会**立刻**从另一端（`veth1`）出来。

**为什么需要它？**

为了实现**网络命名空间**之间的通信。每个 Docker 容器或 Kubernetes Pod 都拥有自己独立的网络命名空间，这相当于一个独立的、隔离的小型网络环境。`veth pair`就是穿透这个隔离墙的通道。

**在容器/Pod 中如何工作？**

1. 当创建一个 Pod 时，CNI 插件（如 Flannel、Calico）会做以下事情：

   - 在 Pod 的**网络命名空间**内创建一块虚拟网卡，这就是 `veth pair`的一端（通常叫 `eth0`）。

   - 在**主机**的根网络命名空间内创建 `veth pair`的另一端（名字通常像 `vethxxxxxx`）。

2. 这样，Pod 内的 `eth0`就和主机上的 `vethxxxxxx`直接连通了。Pod 内所有发往 `eth0`的网络流量，都会直接出现在主机的 `vethxxxxxx`接口上。

**作用**：**将容器/Pod 的内部网络“引出”到主机网络环境**。

------

### 2. 网桥（Bridge）与 `cni0`

**是什么？**

网桥是 Linux 内核实现的**二层网络交换机**。`cni0`是 Flannel 等 CNI 插件默认创建的一个**网桥设备**（名字可以自定义，`cni0`是常见默认名）。

**为什么需要它？**

光有 `veth pair`把 Pod 网络引出来还不够，我们需要一个设备让**同一个节点上的多个 Pod 能够互相通信**，同时也作为流量**进出节点**的枢纽。

**它是如何工作的？**

1. **连接**：主机上每个 Pod 对应的 `veth pair`的外端（`vethxxxxxx`）都会被“插”到 `cni0`这个虚拟交换机上。

2. **交换**：`cni0`像物理交换机一样，学习每个 `veth`端口的 MAC 地址。当 Pod A 发数据给同节点的 Pod B 时，数据包会通过 `veth`到达 `cni0`，`cni0`根据目标 MAC 地址将数据帧转发到 Pod B 对应的 `veth`端口，从而到达 Pod B。

3. **网关**：`cni0`网桥自身也会有一个 IP 地址（例如 `10.244.1.1/24`）。对于 Pod 来说，这个地址就是它们的**默认网关**。所有 Pod 发往**节点外部**（其他节点 Pod 或互联网）的流量，都会先发给 `cni0`。

**作用**：**实现同节点内 Pod 间的二层通信，并作为所有 Pod 流量的集中转发点（网关）**。

------

### 整体工作流程示例（以 Flannel VXLAN 模式为例）

假设 Node1 上有 Pod A (`10.244.1.2`)，要访问 Node2 上的 Pod B (`10.244.2.3`)。

1. **Pod A 内部**：
   - Pod A 通过自己的 `eth0`（`veth pair`的一端）发送数据包。
   - 目标 IP: `10.244.2.3`， 网关: `10.244.1.1`（即 `cni0`的地址）。
   
2. **Node1 主机网络栈**：
   - 数据包通过 `veth pair`从 Pod A 的 `eth0`瞬间到达主机侧的 `vethxxxx`接口。
   - `vethxxxx`连接在 `cni0`网桥上，包到达 `cni0`
   - `cni0`发现目标 MAC 不是本地任何端口，于是将包交给它的“上层”——主机内核协议栈处理。

3. **主机路由与封装**：
   - 主机内核查看路由表。Flannel 会添加类似这样的路由：`10.244.2.0/24 via 10.244.2.0 dev flannel.1`。意思是：去往 `10.244.2.0/24`网段的包，都发给 `flannel.1`设备。
   - 数据包被路由到 `flannel.1`（VXLAN 隧道端点）。
   - `flannel.1`将原始数据包封装进一个 VXLAN/UDP 包中，外层头部的目标地址是 Node2 的 IP。

4. **Node2 主机网络栈**：
   - 封装后的包经物理网络到达 Node2。
   - Node2 的 `flannel.1`设备解封装，得到原始数据包。
   - 查看路由表，发现目标 `10.244.2.3`是本机 `cni0`网桥下的子网（路由：`10.244.2.0/24 dev cni0`）。
   - 数据包被发送到 `cni0`网桥。

5. **Node2 的 `cni0`到 Pod B**：

   - `cni0`根据目标 IP/MAC，将数据帧转发给 Pod B 对应的 `veth`端口。
   - 数据包通过 `veth pair`进入 Pod B 的网络命名空间，到达其 `eth0`网卡。

### 总结

| 组件                 | 角色比喻       | 核心功能                                                     | 所在位置                                         |
| -------------------- | -------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| **`veth pair`**      | **虚拟网线**   | 连接两个隔离的网络命名空间（如 Pod 与主机）                  | 一端在 Pod 内 (`eth0`)，一端在主机上 (`vethxxx`) |
| **网桥 (如 `cni0`)** | **虚拟交换机** | 1. 实现**同节点内** Pod 间的二层通信。 2. 作为 Pod 的**默认网关**，集中处理进出节点的流量。 | 主机的根网络命名空间                             |

**简而言之**：`veth pair`把 Pod 连到主机，**网桥**把主机上所有 Pod 的 `veth`连接起来并管理它们之间的流量。这是几乎所有 Kubernetes CNI 网络方案（无论是 Overlay 还是路由模式）在**单个节点内部**共同依赖的基础设施。不同 CNI 插件的差异主要体现在**节点之间**的流量如何传输（如 VXLAN、BGP、eBPF 等）。

Pod 之间通过独立 Pod IP通信，同节点经veth pair + 网桥转发，跨节点由 CNI 网络插件（Overlay/Underlay）实现路由或隧道互通。

### 5.如何排查pod不通

- **同节点不通**：大概率是 **本机网桥、CNI 本地规则、IPtables、本机网络栈** 问题
- **跨节点不通**：大概率是 **节点间网络、隧道（VXLAN/IPIP）、路由、防火墙、CNI 集群配置** 问题


#### 一、相同节点 Pod 不通排查（同 Node）

##### 1. 先确认基础连通性
```bash
# 在 PodA 里 ping PodB IP
kubectl exec -it pod-a -- ping <pod-b-ip>
```

##### 2. 排查方向

1. pod 是否正常 Running
```bash
kubectl get pod -o wide
```
2. 看本机网桥 cni0 /flannel.1 状态
```bash
brctl show
ip link
```
3. IPtables 规则是否被清空 / 乱了
```bash
iptables -L -n | grep KUBE
```
4. 本机路由是否正常
```bash
ip route show
```
5. 查看 CNI 配置是否异常
```bash
ls /etc/cni/net.d/
```
6. 重启 kubelet + 网络插件
```bash
systemctl restart kubelet
```
##### 同节点不通的原因
- CNI 插件异常
- 本机网桥没 UP
- IPtables 规则丢失
- Pod 网关配置错误
- 应用只监听 127.0.0.1（最经典坑）

------

#### 二、不同节点 Pod 不通排查（跨 Node）

##### 1. 基础测试
```bash
# Node1 上的 Pod ping Node2 上的 Pod
kubectl exec -it pod-a -- ping <pod-b-ip>
```
##### 2.排查

1. 节点之间是否能互通
```bash
   ping node2-ip
```
2. 节点间是否能通 VXLAN/Flannel 隧道
```bash
ip addr | grep flannel
ip addr | grep calico
```
3. 路由是否指向正确的隧道设备
```bash
ip route | grep 10.244
```
4. UDP 8472（flannel）/ 4789（calico）是否被防火墙拦了
```bash
telnet node2 8472
```
5. 各节点 Pod 网段是否冲突
```bash
kubectl get nodes -o wide
```
6. 检查 CNI 状态
```bash
kubectl get pods -n kube-system -o wide
```

##### 跨节点不通的常见原因

- 节点间防火墙拒绝了隧道端口
- 云服务器安全组没放行
- 跨节点路由缺失
- CNI 集群地址分配不一致
- 节点内核不支持 VXLAN

------

#####满分回答

> 排查 K8s 网络不通我会分两种场景：
>
> 1. **相同节点 Pod 不通**
>
>    我会先检查 Pod 状态，再排查本机网桥、IPtables 规则、本地路由以及 CNI 插件配置，
>
>    常见原因是本机网络栈异常、IPtables 丢失或容器应用监听地址问题。
>
> 2. **跨节点 Pod 不通**
>
>       我会先检查节点间基础网络连通性，再排查 Flannel/Calico 隧道端口是否被防火墙拦截，
>
>    同时检查各节点路由表、Pod 网段是否冲突、CNI 集群配置是否一致，
>
>    大部分是节点间网络策略、安全组或隧道通信问题。
>

------

#### 总结

**同节点看本机，跨节点看隧道与防火墙。**


## Service 类型

### 1. ClusterIP（默认类型）
作用：仅集群内部访问，外部无法访问
场景：微服务之间互相调用
特点：分配一个集群内部 IP
最常用
### 2. NodePort
作用：让外部访问集群内服务
原理：在每个节点开一个端口（30000-32767）
访问方式：节点IP:端口
场景：测试、简单外部访问
优点：简单
缺点：端口固定，不适合大规模生产
### 3. LoadBalancer
作用：公有云环境对外暴露服务
原理：依赖云厂商的负载均衡器
自动分配公网 IP
场景：生产环境、正式业务
优点：高可用、稳定
缺点：必须在云环境（AWS、阿里云、腾讯云）

### 4. ExternalName
作用：把集群外的服务映射到集群内部
场景：访问外部数据库、外部中间件
特点：没有 IP，只有 DNS 别名
最精简记忆口诀（30 秒背会）
ClusterIP：内部用
NodePort：外部用（节点端口）
LoadBalancer：云生产用（负载均衡）
ExternalName：映射外部服务

### 面试官最爱问的 2 个问题
Q1：Service 有哪几种类型？
你直接背：
ClusterIP、NodePort、LoadBalancer、ExternalName 四种。
Q2：生产环境用哪种？
LoadBalancer（云环境）
没有云负载均衡器时用 NodePort 或 Ingress。

##Ingress和service

好的，我们来系统性地梳理 Kubernetes 中 **Service** 和 **Ingress** 的核心区别与协作关系。

首先给出最核心的结论：**Service 是四层（L4）的负载均衡与服务发现机制，而 Ingress 是七层（L7）的 HTTP(S) 路由管理机制。它们通常协同工作，共同构成从外部访问集群内应用的标准路径。**

下图清晰地展示了两者的层级关系和协同工作流程：

![](C:\Users\pc\Pictures\ingress&&srevice.png)


下面我们展开说明。

------

### 一、核心区别对比

| 特性维度           | Service                                                      | Ingress                                                      |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **API 对象**       | 核心 `v1`API 组中的标准资源类型，如 `kind: Service`。        | 网络扩展 API 组（`networking.k8s.io/v1`）中的资源类型，如 `kind: Ingress`。 |
| **核心功能**       | **服务发现** 与 **四层负载均衡**。为一组 Pod 提供一个稳定的虚拟 IP（VIP）和端口，实现 Pod 与 Pod 之间、外部与 Pod 之间的通信。 | **七层路由**。根据 HTTP/HTTPS 请求的主机名、路径、请求头等信息，将外部流量路由到集群内部不同的 Service。 |
| **工作层级**       | 主要工作在**传输层（OSI L4）**，处理 TCP/UDP 流量。          | 主要工作在**应用层（OSI L7）**，处理 HTTP/HTTPS 流量。       |
| **暴露服务的方式** | 有三种类型： • **ClusterIP**：集群内部虚拟 IP。 • **NodePort**：在每个节点上开放静态端口。 • **LoadBalancer**：通过云供应商创建外部负载均衡器。 | **本身不直接暴露服务**。它需要与一个 **Ingress Controller** 结合，并通过一个前置的 **Service**（通常为 LoadBalancer 或 NodePort 类型）来获得外部访问入口。 |
| **规则定义**       | 规则简单，主要定义**端口映射**（`port`和 `targetPort`）和**选择器**（`selector`）。 | 规则复杂，可以定义基于**主机**、**路径**、**TLS/SSL 证书**、**重定向**、**重写**等丰富的路由规则。 |
| **类比**           | 像**内部负载均衡器**或**服务注册中心**，负责将请求分发到后端的多个实例。 | 像**反向代理服务器**（如 Nginx）或**API 网关**，负责根据请求内容进行智能路由。 |

------

### 二、核心协作关系

Service 和 Ingress 不是替代关系，而是上下游的**协作与增强关系**。一个典型的、从公网访问集群内 Web 应用的流量路径如下：

1. **Pod 与 Service**：

   - 您部署了应用的多个 Pod（例如 `frontend-pod-1`, `frontend-pod-2`）。

   - 您创建一个 **ClusterIP 类型的 Service**（例如 `frontend-svc`）。这个 Service 通过标签选择器找到这些 Pod，并为其分配一个集群内虚拟 IP（如 `10.96.123.456`）。

   - **此时，集群内的其他服务可以通过 `frontend-svc:80`来访问前端应用，Service 负责将流量负载均衡到后端的 Pod。**

2. **Ingress 规则**：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: e-commerce-ingress
spec:
  rules:
  - host: shop.example.com          # 规则1：电商前端
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service  # 转发到前端服务service
            port:
              number: 80

  - host: api.example.com           # 规则2：后端API
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service      # 转发到用户服务的service
            port:
              number: 8080
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: product-service   # 转发到商品服务的service
            port:
              number: 8080
#还可以添加TLS规则，但是前提是得在同一命名空间下有密钥证书
```

   - 这个 Ingress 资源只是一份“路由说明书”，自己不会处理任何流量。

3. **Ingress Controller 执行（反向代理器，代理服务器端）**：
   - 集群中运行着一个 **Ingress Controller**（例如 `nginx-ingress-controller`的 Pod）。
   
   - Ingress Controller 实时监控集群中所有的 Ingress 资源变化。
   
   - 当它发现您创建的 Ingress 规则后，**立即将其翻译成自己（Nginx）的配置**（即在 Nginx 配置中新增一个 `server`块，监听 80/443 端口，并将 `app.mycompany.com`的流量代理到上游 `frontend-svc:80`）。
   
4. **外部访问入口**：

   - Ingress Controller 本身也是一个 Pod，需要被暴露出去。通常会为它创建一个 **LoadBalancer 类型的 Service**。

   - 云服务商会为这个 Service 分配一个**公网 IP**（例如 `203.0.113.100`）。

   - 您将域名 `app.mycompany.com`的 DNS A 记录指向这个公网 IP。

5. **完整的访问链路**：

   - 用户访问 `http://app.mycompany.com`

   - DNS 解析到公网 IP `203.0.113.100`，请求到达 **Ingress Controller**。

   - Ingress Controller 根据请求中的 `Host`头，匹配到 Ingress 规则。

   - 根据规则，将请求转发给 **Service** `frontend-svc:80`。

   - `frontend-svc`根据其负载均衡策略，将请求最终路由到其中一个健康的 **Pod**（例如 `frontend-pod-1`）。

   - Pod 处理请求并返回响应。

------

### 三、为什么需要两者协作？解决什么问题？

- **Service 的不足**：

  - **NodePort**：端口范围固定（30000+），不友好，且缺乏基于域名的路由。

  - **LoadBalancer**：每个需要外部访问的服务都要一个独立的负载均衡器和公网 IP，**成本极高，管理繁琐**。

  - 两者都**缺乏七层智能路由能力**（无法根据路径、请求头等细分流量）。

- **Ingress 的价值**：

  - **入口收敛**：使用一个公网 IP 和负载均衡器（Ingress Controller），通过不同的**域名和路径**为**成百上千个内部 Service** 提供外部访问能力，大幅降低成本。

  - **七层路由**：实现灰度发布（基于路径/头）、A/B 测试、多租户路由等复杂场景。

  - **统一安全**：在入口层统一实施 TLS 终止、WAF 防护、限流等策略。

### 四、总结

您可以这样记忆：

- **Service**  是 Kubernetes 中用于实现服务发现和负载均衡的核心抽象。它通过标签选择器（Label Selector）动态地指向一组 Pod，并为这组 Pod 提供一个稳定的虚拟访问端点（名称、虚拟IP和端口）它是内部通信的基石。

- **Ingress** 是 **“外部用户如何通过 HTTP/HTTPS 访问我集群内不同的 Service？”** 的答案。它是**外部访问的智能路由器**。

它们共同构成了 Kubernetes 微服务网络从内到外的完整暴露方案：**Ingress 对外提供智能路由，Service 对内提供稳定服务发现和负载均衡**。

## kube-proxy 是什么？

**它是 Service 的真正实现者！**

- 监听 apiserver 的 Service/Endpoint 变化
- 维护转发规则，实现：
- ClusterIP 负载均衡
- NodePort 转发
- 外部流量进入集群
- 它不负责 Pod 互通，只负责Service 转发。

### 4. kube-proxy 的 3 种模式（超级重点）

kube-proxy 是 Kubernetes 中实现 Service 抽象（服务发现与负载均衡）的核心组件。它通过在节点上配置网络规则，将发往 Service 虚拟 IP（ClusterIP）的流量转发到后端 Pod。其工作模式决定了转发性能、功能和可扩展性。目前主要有三种模式：**userspace（已弃用）、iptables（长期默认）和 IPVS（高性能）**。此外，官方已引入 **nftables** 模式作为 iptables 和 IPVS 的现代替代方案。

### 1. userspace 模式（已弃用）

这是 kube-proxy 最初的工作模式，现已基本不再使用，主要用于理解演进过程。

- **工作原理**：
  1. kube-proxy 在**用户空间**为每个 Service 监听一个随机端口（代理端口）。**用户空间**
  2. 它配置 **iptables 规则**，将所有发往 Service `ClusterIP:Port`的流量**重定向**到这个代理端口。**内核空间**
  3. kube-proxy 进程接收到代理端口的流量后，在用户空间**通过负载均衡算法（如轮询）** 选择一个后端 Pod，然后将请求转发过去。**用户空间**
- **优点**：实现简单，易于调试。
- **缺点**：
  - **性能差**：流量需要在**内核空间和用户空间之间来回拷贝和切换**，导致高延迟和高CPU开销。
  - **可靠性依赖进程**：如果 kube-proxy 进程异常，转发将中断。
- **现状**：**已弃用**，不应用于生产环境。

### 2. iptables 模式（传统默认模式）

这是过去很长一段时间内的默认模式，稳定且广泛使用。

- **工作原理**：
  1. kube-proxy 监听 API Server，获取 Service 和 Endpoint 的变化。
  2. 它直接在节点的 **iptables（Linux netfilter）** 中配置一系列**转发规则链**，完全在内核态完成数据包处理。
  3. 规则结构是分层的：
     - `PREROUTING/OUTPUT`-> `KUBE-SERVICES`链：匹配目标为 Service IP 的流量。
     - `KUBE-SVC-<HASH>`链：对应特定 Service，实现负载均衡。默认使用**随机**算法，通过 `--probability`参数设置权重。
     - `KUBE-SEP-<HASH>`链：对应每个后端 Pod，执行 **DNAT** 将目标地址改写为 Pod IP。
  4. 数据包按链匹配，最终被 DNAT 到某个 Pod，**无需经过 kube-proxy 进程转发**。
- **优点**：
  - **性能较好**：完全在内核空间处理，避免了用户空间切换的开销。
  - **稳定成熟**：依赖 Linux 内核的 netfilter，非常可靠。
- **缺点**：
  - **负载均衡算法单一**：仅支持随机（random）或等概率轮询，无法感知后端负载。
  - **无重试机制**：如果规则选中的第一个 Pod 无响应，连接会直接失败，依赖客户端重试。
  - **规则线性增长**：Service 和 Pod 数量增多时，iptables 规则数量呈**线性增长**。匹配规则需要**遍历链（O(n)复杂度）**，在大规模集群（如数千服务、数万Pod）中，规则更新和匹配效率会显著下降，影响性能。
  - **规则更新开销大**：早期版本更新任何规则都需要全量刷写，可能引起连接中断。新版本（v1.28+）已优化为增量更新。

### 3. IPVS 模式（高性能模式，现已被标记为弃用）

IPVS (IP Virtual Server) 是 Linux 内核内置的**四层负载均衡器**，专为高性能场景设计。

- **工作原理**：
  1. kube-proxy 使用 IPVS 的**内核API**创建“虚拟服务器”（对应 Service）和“真实服务器”（对应后端 Pod）kube-proxy只调用API，让内核去执行转发流量
  2. IPVS 使用**哈希表**存储转发规则，对数据包的转发决策是 **O(1) 的时间复杂度**，与后端数量无关，查询效率极高。
  3. 它仍然**依赖少量 iptables 规则**（主要用于设置数据包标记、SNAT等辅助功能），但核心的负载均衡和转发由 IPVS 模块完成。
  4. 支持**丰富的负载均衡算法**，如轮询（rr）、加权轮询（wrr）、最少连接（lc）、加权最少连接（wlc）等，可通过 `ipvs.scheduler`配置。
- **优点**：
  - **高性能**：哈希表查找，转发延迟低，能处理更高的并发连接数。
  - **调度算法丰富**：可根据业务场景选择更优的负载均衡策略。
  - **规则同步性能好**：增量更新规则，对大规模集群更友好。
  - **资源占用低**：在大规模集群中，CPU和内存消耗显著低于 iptables 模式。
- **缺点/现状**：
  - **内核依赖**：需要节点内核加载 `ip_vs`等模块。
  - **功能覆盖不全**：IPVS 内核 API 未能完美匹配 Kubernetes Service API 的所有边缘情况，导致某些功能（如特定类型的会话亲和性）可能无法完全实现。
  - **官方状态**：自 **Kubernetes v1.35 起已被标记为弃用 (deprecated)**。官方推荐其替代方案 **nftables 模式**。

### 模式对比与选型建议

| 特性             | userspace 模式                | iptables 模式                   | IPVS 模式                               |
| ---------------- | ----------------------------- | ------------------------------- | --------------------------------------- |
| **工作原理**     | 用户空间代理，iptables 重定向 | 内核空间，iptables 规则链       | 内核空间，IPVS 哈希表                   |
| **性能**         | 低（用户/内核空间切换）       | 中（规则链遍历 O(n)）           | **高**（哈希查找 O(1)）                 |
| **负载均衡算法** | 轮询                          | 随机                            | **rr, wrr, lc, wlc** 等10多种           |
| **规则管理**     | 简单                          | 复杂，线性增长                  | 中等，哈希表存储                        |
| **扩展性**       | 差                            | 中（大规模集群性能下降）        | **高**（适合大规模集群）                |
| **连接失败处理** | 可重试其他后端                | **无重试**，依赖客户端          | 依赖算法（如lc可避免）                  |
| **生产建议**     | **禁止使用**                  | **中小规模集群**（服务数<1000） | **大规模/高性能集群**（但注意已被弃用） |

**总结与最新建议**：

1. **userspace 模式**：仅用于历史学习，**切勿**在生产环境使用。
2. **iptables 模式**：**稳定、兼容性极佳**，是大多数**中小规模集群**的稳妥选择。在服务数量不多（例如数百个）时，其性能表现与 IPVS 差异不大，甚至可能更优。
3. **IPVS 模式**：为**超大规模集群**（数千服务）和高并发场景设计，能提供更稳定的高性能和更丰富的调度算法。**但请注意，它已被官方弃用**。如果您的内核版本较新（支持 nftables），应考虑转向 **nftables 模式**，它被设计为同时替代 iptables 和 IPVS，并拥有更好的性能。
4. **未来方向 (nftables)**：作为新一代的包过滤框架，nftables 旨在解决 iptables 的扩展性和性能问题，是 Kubernetes 推荐的演进方向。在新部署的集群中，如果环境支持，可以考虑评估 nftables 模式。

## 网络总览

- pod 之间能不能互通 → 靠 Calico / Flannel（CNI 插件）
- 外部流量进来先到哪里 → Ingress controller 根据ingress规则进行路由转发到相应的service
- Service 怎么转发到 Pod（ClusterIP / NodePort）→  靠 kube-proxy
- kube-proxy 用什么技术实现转发 →  iptables / IPVS / userspace 三种模式

好的，这是一个关于 Kubernetes 网络架构的核心问题。要理清 `kube-proxy`、`Ingress`、`Service`和“网络插件”（通常指 CNI 插件）的关系，我们需要将它们放在一个**分层协作的模型**中来理解。

简单来说，它们共同构建了 Kubernetes 集群内外部流量的完整通路，但职责分明。下图清晰地展示了它们在处理一个外部 HTTP 请求时的完整协作流程与层次关系：

![](C:\Users\pc\Pictures\k8swangluojiagou.png)

下面我们来详细拆解每个组件的角色和它们之间的交互。

### 一、各组件核心职责

1. **CNI 网络插件**

   - **作用**：它是 Kubernetes 网络的**基石**。负责在 Pod 创建时，为 Pod 分配 IP 地址，并确保**集群内所有 Pod 之间（无论是否在同一节点上）可以相互直接通信**（即“IP-per-Pod”模型）。它解决了 Pod 与 Pod 的网络连通性问题。
   - **类比**：相当于城市的“道路规划局”和“施工队”，负责修建和维护所有建筑（Pod）之间的道路（网络），让车辆（数据包）能从一个建筑直接开到另一个建筑。
   - **常见实现**：Calico, Flannel, Cilium, Weave Net 等。

2. **Service**
   - **作用**：Kubernetes 中的核心**抽象概念**，而非一个具体的进程。它定义了一个稳定的访问入口（虚拟 IP 和端口），为一组动态变化的 Pod（由 Label Selector 选定）提供**服务发现和负载均衡**。Service 的虚拟 IP 称为 `ClusterIP`。
   - **要解决的问题**：Pod 是动态的（IP 会变，会创建和销毁）。Service 提供了一个固定的访问点，并将流量平均分发到后端的健康 Pod。
   - **类比**：相当于一个公司的“总机号码”或“前台”。外部或内部需要联系某个部门（一组 Pod），只需拨打总机号（Service IP），前台会自动将电话转接到部门内一个空闲的员工（Pod）那里。

3. **kube-proxy**
   - **作用**：它是 **Service 抽象概念的具体实现者之一**。它是一个运行在每个 Kubernetes 节点上的**守护进程**。其核心任务是监听 API Server 中 Service 和 Endpoints（后端 Pod IP 列表）的变化，并**在本节点上配置相应的转发规则**，以便能将发往 `Service ClusterIP`的流量实际转发到后端的某个 `Pod IP`。
   - **工作模式**：主要通过维护节点上的 **iptables** 或 **IPVS** 规则来实现，是数据平面的重要组成部分。
   - **简单来说**：`kube-proxy`是让 `Service`这个“电话号码”真正能接通到“员工”的“电话交换机系统”。
   
4. **Ingress 与 Ingress Controller**

   - **作用**：
     - **Ingress**：一个 API 对象，定义了对集群内**Service 的 HTTP/HTTPS 路由规则**（例如，根据不同的域名或路径，将请求路由到不同的 Service）。
     - **Ingress Controller**：是规则的**执行者**。它是一个具体的、通常以 Pod 形式部署的**七层反向代理/负载均衡器**（如 Nginx, Traefik, Envoy）。它监听 Ingress 资源的变化，并动态更新自己的配置。
   - **要解决的问题**：Service 的 `LoadBalancer`或 `NodePort`类型暴露服务时，缺乏基于 HTTP 协议（主机头、路径等）的智能路由能力，且成本高。Ingress 提供了统一的 HTTP 流量入口。
   - **类比**：相当于公司的“智能总机”或“官方网站”。用户访问公司官网的不同页面（不同路径），这个智能系统会根据预设的规则，将请求转接到不同的内部部门总机（Service）。

### 二、核心协作关系与数据流

让我们追踪一个从外部访问的 HTTP 请求，看看四者如何协同工作：

1. **外部请求到达**：用户访问 `https://app.example.com`。
2. **Ingress Controller 处理**：
   - 请求首先到达配置了公网 IP 的 **Ingress Controller**（例如 `nginx-ingress`Pod）。
   - Ingress Controller 根据请求中的 `Host`头和 URL 路径，查询其内存中的规则（这些规则来自于它监听到的 **Ingress** 资源定义）。
   - 假设 Ingress 规则是：`app.example.com -> backend-service:80`。
3. **Ingress 到 Service**：
   - Ingress Controller 根据规则，将请求代理（转发）到名为 `backend-service`的 **Service** 的 `ClusterIP`和端口 `80`。**此时，Ingress Controller 成为了 Service 的一个客户端**。
4. **kube-proxy 介入**：
   - 请求数据包的目标 IP 是 `backend-service`的 `ClusterIP`。
   - 这个数据包进入节点内核后，被 **kube-proxy** 预先设置好的 **iptables/IPVS 规则** 拦截。
   - `kube-proxy`的规则会根据负载均衡算法（如随机），将数据包的目标 IP 从 `Service ClusterIP`**DNAT** 修改为某个健康后端 **Pod 的真实 IP**，假设是 `10.244.1.5`。
5. **CNI 网络插件保障最后一公里**：
   - 现在，数据包需要从当前节点（可能是 Ingress Controller 所在的节点，也可能是另一个节点）发送到目标 Pod IP `10.244.1.5`。
   - **CNI 网络插件** 已经确保了整个集群的 Pod 网络是三层连通的。无论这个 Pod 在哪个节点上，数据包都能通过 CNI 创建的 overlay 网络（如 VXLAN）或 underlay 网络（如 BGP）被正确路由到目标 Pod 所在的节点，并最终进入该 Pod 的 Network Namespace。

### 三、总结与关系图

| 组件                  | 解决的问题                                 | 工作层次 | 依赖关系                                            | 类比                  |
| --------------------- | ------------------------------------------ | -------- | --------------------------------------------------- | --------------------- |
| **CNI 网络插件**      | Pod 与 Pod 如何通信？                      | L2/L3    | 基础设施                                            | 修路、建桥            |
| **Service (抽象)**    | 如何发现和访问一组 Pod？                   | L4       | 依赖 CNI 保证 Pod 网络                              | 公司总机号码          |
| **kube-proxy (实现)** | 如何将 Service IP 的请求转到 Pod IP？      | L3/L4    | 实现 Service；依赖 CNI 完成最终路由                 | 电话交换机系统        |
| **Ingress**           | 外部如何用 HTTP 协议访问内部多个 Service？ | L7       | 依赖 Service 作为上游； 自身是 CNI 网络中的一个 Pod | 智能官网/客服导流系统 |

**核心协作链**：

**外部用户** -> **Ingress Controller (Pod)** -> (根据 **Ingress** 规则) -> **Service (ClusterIP)** -> (被 **kube-proxy** 规则转发) -> **目标 Pod IP** -> (通过 **CNI 网络** 送达) -> **最终的业务 Pod**。

简单来说：**CNI 铺路，Service 和 kube-proxy 建立内部通信系统，Ingress 负责对外接待和智能引导。** 它们共同构成了 Kubernetes 微服务网络从底层连通到上层路由的完整栈。

## 关于网络插件Flannel，Calico和Cilium

### 一、K8s 网络三大基本原则：

- 每个 Pod 拥有独立 IP
- 所有 Pod 可以直接通信，无需 NAT
- 节点上的容器可与所有节点 Pod 通信
  **CNI 插件负责实现以上能力，Flannel、Calico 都是 CNI。**
  

这三个插件代表了 Kubernetes CNI 的三种主流技术路线：**Flannel**（简单 Overlay）、**Calico**（BGP 路由/隧道）和 **Cilium**（eBPF 驱动）。它们的底层原理差异巨大，直接决定了其性能、功能和适用场景。

### 一、Flannel：简洁的 Overlay 网络

Flannel 的设计哲学是**简单易用**，它通过在主机间建立隧道，构建一个覆盖在物理网络之上的虚拟大二层网络，让所有 Pod 仿佛在同一局域网内。

#### **核心架构与组件**

- **flanneld**：运行在每个节点上的守护进程，是控制平面的核心。它从 Kubernetes API 或 etcd 获取集群网络配置（如 Pod CIDR），并维护节点子网与主机 IP 的映射关系。

- **后端（Backend）**：数据平面的具体实现。Flannel 支持多种后端，最常用的是 **VXLAN** 和 **host-gw**。

- **cni0 网桥**：节点上的 Linux 网桥，类似 `docker0`  Pod 的 `veth pair`一端连接于此，负责节点内 Pod 的通信。

#### **底层原理详解**

**1. VXLAN 模式（默认）**

这是一种 **Overlay（覆盖）网络**。当 Pod 跨节点通信时，原始数据包会被封装在一个新的 UDP 包中，外层使用宿主机的 IP 进行路由。
- **关键设备**：`flannel.1`，这是一个 **VTEP** 设备，负责 VXLAN 的封包和解包。
- **通信流程**：
  1. Pod A（Node1）发送数据包给 Pod B（Node2）
  2. 包经 `veth pair`到达 `cni0`网桥。
  3. 根据路由表，目标非本机子网的包被路由到 `flannel.1`设备
  4. `flannel.1`查询 **FDB（转发数据库）**，获知目标 Pod 所在节点 Node2 的 VTEP IP 和 MAC。
  5. `flannel.1`将原始 L2 帧封装进 VXLAN 头，外层再套上 Node1 到 Node2 的 UDP/IP 报文
  6. 封装后的包通过主机的物理网卡发出，经底层网络到达 Node2。
  7. Node2 的 `flannel.1`设备识别 VXLAN 包，解封装，取出原始帧。
  8. 解封后的包根据目标 IP，通过 `cni0`网桥送达 Pod B。

- **优点**：对底层网络无要求，能跨越三层网络，通用性强。

- **缺点**：封包/解包带来额外的 CPU 和带宽开销，性能有损耗。

**2. host-gw 模式**

这是一种 **纯三层路由** 方案，性能接近原生网络。
- **原理**：`flanneld`在每个节点上**维护主机路由表**，将其他节点的 Pod 子网直接指向对应节点的物理 IP。
  - 例如，在 Node1 上会有一条路由：`10.244.2.0/24 via 172.16.130.164 (Node2-IP)`。
- **通信流程**：
  1. Pod A 发往 Pod B 的包到达 `cni0`。
  2. 内核根据路由表，发现目标网段 `10.244.2.0/24`的下一跳是 Node2 的 IP `172.16.130.164`。
  3. 内核直接通过 **ARP** 获取 Node2 的 MAC 地址，将包封装成以 Node2 的 MAC 为目的地的二层帧，从物理网卡发出。
  4. 包经二层网络直达 Node2。
  5. Node2 收到后，根据自身路由表（`10.244.2.0/24 dev cni0`）将包转发给 `cni0`，最终到达 Pod B。

- **优点**：无隧道开销，性能高，延迟低。
- **缺点**：**要求所有节点在同一个二层网络（即同一子网且可二层互通）**，这在云环境或复杂网络中往往不满足。

### 二、Calico：高性能的三层网络与策略引擎

Calico 的核心思想是 **纯三层路由**，它希望像互联网一样，通过路由协议让每个 Pod 的 IP 都能被直接路由到，从而避免封包开销。当三层路由不可达时，则降级使用隧道。

#### **核心架构与组件**

- **Felix**：每个节点上的守护进程，是数据平面的“执行器”。它负责：
  - 配置本机路由、ARP 表、IP 转发规则
  - 编程 **iptables/ipset** 规则，实现网络策略（NetworkPolicy）、NAT 和访问控制。
  - 管理本机的虚拟网络接口（如 `caliXXX`）。
- **BIRD**：轻量级 BGP 客户端。它从 Felix 获取本机 Pod 的路由信息，并通过 **BGP 协议** 广播给集群中的其他 BGP Speaker（其他节点或路由反射器）。
- **Confd**：监控 etcd 或 Kubernetes API，动态生成 BIRD 配置文件。
- **CNI 插件**：响应 kubelet 调用，为 Pod 配置网络（创建 veth pair，设置 IP 等）。

#### **底层原理详解**

**1. BGP 模式（纯路由模式）**

这是 Calico 的**理想模式**，要求底层网络支持 BGP 路由传播或节点间二层互通。

- **工作原理**：
  1. 每个节点上的 **Felix** 为本地每个 Pod 配置一条主机路由（如 `10.244.1.10 dev cali12345`）。
  2. **BIRD** 将这些聚合后的路由（如 `10.244.1.0/24`）通过 **BGP 协议** 宣告给邻居节点。
  3. 其他节点的 BIRD 收到路由后，将其写入内核路由表（如 `10.244.1.0/24 via 192.168.1.10`）。
  4. 跨节点通信时，数据包基于节点的路由表直接转发，**无需任何封装**。目标节点收到包后，通过本地的 `cali`接口送入对应 Pod。
- **BGP 邻居模式**：
  - **Node-to-Node Mesh**：每个节点与其他所有节点建立 BGP 连接（全互联）。适用于小集群（<100节点）。
  -  **Route Reflector**：指定一个或几个节点作为路由反射器，其他节点只与 RR 建立连接，由 RR 反射全网路由。适用于大规模集群。

**2. IPIP 隧道模式**

当节点间**不在同一个二层网络**时，Calico 会自动或配置为使用 IPIP 隧道。
-  **工作原理**：
  1. 节点上会创建一个 `tunl0`隧道接口。
  2. BGP 宣告的路由下一跳不变，但出接口变为 `tunl0`。
  3. 当数据包需要发往其他节点的 Pod 时，内核路由将其导向 `tunl0`。
  4. `tunl0`设备将原始 IP 包（Pod IP）**封装在另一个 IP 包中**，外层头部的源/目的是宿主机的 IP。
  5. 封装后的包通过物理网络路由到目标节点。
  6. 目标节点的 `tunl0`接口解封装，取出内层包，再根据路由送到目标 Pod。

- **与 VXLAN 对比**：IPIP 是更简单的三层隧道（IP in IP），而 VXLAN 是二层 over UDP。IPIP 开销略小，但通用性不如 VXLAN。

**3. 网络策略实现**

Calico 的网络策略能力非常强大。Felix 会将 Kubernetes 的 NetworkPolicy 资源转换并编程为节点上的 **iptables 规则**。这些规则基于 Pod 的“身份”（通过标签选择器确定），在 `cali`接口的 INPUT/FORWARD 链上进行过滤，实现精细的 L3/L4 访问控制。

### 三、Cilium：基于 eBPF 的云原生网络平台

Cilium 代表了下一代 CNI，其核心是利用 Linux 内核的 **eBPF** 技术，将网络、安全、可观测性逻辑直接注入内核，**绕过传统的 iptables 和 Overlay 网络栈**，实现高性能和深度可编程性。

#### **核心架构与组件**

- **Cilium Agent (cilium-agent)**：每个节点上的守护进程。它监听 Kubernetes API，管理 eBPF 程序的生命周期（编译、加载、附加），并维护 eBPF Map。
- **Cilium Operator**：集群级的控制逻辑，负责诸如节点管理、服务同步、网络策略清理等任务。
- **CNI 插件**：响应 kubelet，调用 Cilium Agent 的 API 来配置 Pod 网络。
- **eBPF 程序**：真正的数据平面，以内核字节码形式运行，附着在内核的各种“钩子点”上。

#### **底层原理详解：eBPF 数据平面**

Cilium 在内核的关键路径上挂载 eBPF 程序，实现“短路”处理：

1. **网络连接处理（替代 kube-proxy）**：
   - 传统 `kube-proxy`使用 iptables 为每个 Service 创建大量规则，规模大时性能下降。
   - Cilium 将 **Service 的 IP:Port 到后端 Pod IP 的映射关系存储在 eBPF Map** 中。
   - 当数据包到达时，附着在 **TC（Traffic Control）** 或 **XDP（eXpress Data Path）** 钩子上的 eBPF 程序直接查询这个 Map，**在最早可能的时机完成 DNAT 和负载均衡**，然后重定向到目标 Pod 的虚拟网卡，完全绕过了内核的 Netfilter（iptables）框架，性能极高。

2. **网络策略执行**：
   - 传统网络策略也依赖 iptables，规则线性匹配，效率低。
   - Cilium 将策略编译成 eBPF 程序，并利用 **eBPF Map 进行高效的哈希查找**。数据包在 TC 入口处即可被允许或丢弃，速度快且扩展性好。

3. **可观测性**：
   - eBPF 程序可以内核态收集任何网络流量的元数据（L3/L4/L7），并推送至用户态，实现**深度网络可视化、服务依赖图、Prometheus 指标**等，无需修改应用代码。

4. **Socket 层加速**：
   - 对于**同节点 Pod 间通信**，Cilium 可以利用 **BPF_PROG_TYPE_SOCK_OPS** 等程序，在 socket 建立时记录其信息到 eBPF Map。

   - 当同节点两个 Pod 的 socket 通信时，eBPF 程序可以将数据**直接从源 socket 的发送队列重定向到目标 socket 的接收队列**，**完全绕过 TCP/IP 协议栈**，大幅降低延迟和 CPU 消耗。

5. **多集群与高级路由**：
   - 支持 Cluster Mesh，通过 BGP 或加密隧道连接多个集群。
   - 支持灵活的负载均衡算法、带宽管理、IPVLAN 等高级功能。

### 总结对比

| 特性                  | **Flannel**                                  | **Calico**                                              | **Cilium**                                                   |
| --------------------- | -------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| **核心设计**          | 简单 Overlay                                 | 三层路由 + 策略                                         | **eBPF 内核可编程**                                          |
| **数据平面**          | Linux 内核 VXLAN / 主机路由                  | Linux 内核路由 + iptables                               | **eBPF 程序（绕过 iptables）**                               |
| **控制平面**          | etcd/K8s API, flanneld 维护映射              | BGP 协议 (BIRD) + Felix                                 | K8s API, Cilium Agent 管理 eBPF                              |
| **网络模型**          | 大二层 Overlay (VXLAN) 或 三层路由 (host-gw) | **纯三层路由 (BGP)** 或 IPIP 隧道                       | 灵活，支持 Overlay、路由、IPVLAN                             |
| **性能**              | VXLAN有开销，host-gw性能好但有限制           | BGP模式性能**接近原生**，IPIP有开销                     | **最高**，尤其在大规模Service和策略下                        |
| **网络策略**          | 支持基础 NetworkPolicy                       | **支持强大且高性能的 NetworkPolicy** (基于iptables)     | **支持 L3-L7 策略，身份感知，性能极高**                      |
| **可观测性**          | 弱                                           | 基础                                                    | **深度、内核级可观测性**                                     |
| **服务发现/负载均衡** | 依赖 kube-proxy (iptables/IPVS)              | 依赖 kube-proxy (iptables/IPVS)                         | **原生 eBPF 实现，替代 kube-proxy**                          |
| **复杂度**            | **最低**，部署简单                           | 中等，需理解 BGP/路由                                   | **最高**，eBPF 技术栈较新                                    |
| **适用场景**          | 中小集群，追求简单稳定                       | 对网络性能、策略有要求的大中型集群，尤其是私有云/物理机 | 大规模、高性能、高安全要求的云原生集群，需要深度可观测性和服务网格集成 |

**简单来说**：

- **Flannel** 是“开箱即用”的简单选择，适合快速搭建和中小型集群。

- **Calico** 是“企业级网络”的标杆，在提供高性能路由的同时，提供了强大的网络策略能力，是许多生产环境的默认选择。

- **Cilium** 是“面向未来”的云原生网络平台，它利用 eBPF 从根本上提升了性能、可扩展性和功能深度，是追求极致性能和高级功能（如服务网格、安全）的集群的理想选择。



##**CoreDNS**

**一句话定义**：
“CoreDNS 是 Kubernetes 集群默认的、插件化的 DNS 服务器和服务发现组件。它负责将集群内的服务名（如 `my-svc.default.svc.cluster.local`）解析成对应的 ClusterIP，是服务间通信的基石。”

**回答结构（可按此顺序展开）：**

1.  **核心定位与作用**
    *   **服务发现**：自动为 K8s Service 和 Pod 创建 DNS 记录，让微服务之间能通过名称互相访问，而无需关心 Pod IP 的变化。
    *   **域名解析**：提供集群内外的 DNS 查询，是 Pod 访问外部网络（如互联网）的 DNS 中转站。

2.  **架构优势（对比旧版 kube-dns）**
    *   **插件化设计**：这是最大特点。所有功能（如读取 K8s 服务、缓存、转发、监控）都由插件实现，配置灵活，易于扩展。
    *   **单一进程**：用 Go 编写，性能好、内存安全，不像 kube-dns 由多个容器组成，更简单可靠。
    *   **CNCF 毕业项目**：说明其成熟度和社区认可度。

3.  **在 K8s 中的工作原理**
    *   **部署**：以 `Deployment` 形式运行在 `kube-system` 命名空间，并通过一个 `Service` 暴露。这个 Service 的 ClusterIP 会被自动注入到每个 Pod 的 `/etc/resolv.conf` 中，作为 DNS 服务器。
    *   **核心配置**：通过 `ConfigMap` 配置一个名为 `Corefile` 的文件。最重要的两个插件是：
        *   `kubernetes`：用于发现和解析集群内的 Service 和 Pod。
        *   `forward` 或 `proxy`：将非集群域名（如 `www.example.com`）的查询转发到上游 DNS（如 `114.114.114.114` 或公司内网 DNS）。

4.  **监控与日志**
    *   内置 `prometheus` 插件，暴露 `/metrics` 端点，可以监控 DNS 查询量、响应延迟、错误类型等关键指标。
    *   通过 `log` 插件或容器标准输出查看查询日志，用于问题排查。

5.  **常见问题排查思路**
    *   “Pod 内 DNS 解析失败”是典型问题，排查路径如下：
        1.  **查 CoreDNS Pod 状态**：`kubectl get pods -n kube-system` 看是否 `Running`。
        2.  **查 CoreDNS 日志**：`kubectl logs -n kube-system deployment/coredns` 看是否有错误。
        3.  **在 Pod 内做诊断**：用 `busybox` 镜像创建临时 Pod，执行 `nslookup` 分别测试集群内服务名和外部域名，判断问题是出在集群内解析还是上游转发。
        4.  **查网络策略**：确认 CNI 网络插件（如 Calico）没有阻断 Pod 与 CoreDNS Service 之间的 UDP/TCP 53 端口流量。

6.  **性能优化方向**
    *   **水平扩展**：根据集群规模适当增加 CoreDNS 的副本数。
    *   **启用节点本地 DNS**：部署 `node-local-dns`，在节点层面做缓存，大幅降低 CoreDNS 负载和解析延迟。
    *   **合理配置缓存**：调整 `cache` 插件的 TTL。
    *   **配置资源限制**：为 CoreDNS Pod 设置合适的 CPU/内存 `requests` 和 `limits`，防止其因资源不足被驱逐。

# K8s Pod 滚动更新
- 这是 Kubernetes 默认、最安全、最常用的发布方式，核心是不停机、无损、逐步替换旧 Pod。
### 一、一句话核心 
- 滚动更新 = 先启动新 Pod → 等就绪后 → 再杀掉旧 Pod → 逐步替换所有实例整个过程服务不中断。  
### 二、它是怎么工作的？
假设你有 3 个旧版本 Pod（v1），现在要更新到 v2： 
1. K8s 先创建 1 个新 Pod（v2） 
2. 等待新 Pod 变成 Ready（健康检查通过） 
3. 再删除 1 个旧 Pod（v1） 
4. 重复上面步骤，直到所有旧 Pod 都被替换成新 Pod 
5. 更新完成  整个过程始终有可用 Pod 提供服务，不会断服。   
### 三、关键控制参数
Deployment 里两个参数控制更新速度和稳定性： 
1. maxSurge 更新过程中最多能超出期望副本数多少个 Pod默认：25%→ 意思是：最多多跑几个新 Pod，加快更新 
2. maxUnavailable 更新过程中最多允许多少个 Pod 不可用默认：25%→ 意思是：保证最少多少个 Pod 在线，保证可用性 最常用配置示例 
```yaml
spec:
  strategy:
    type: RollingUpdate  # 默认就是这个，可以不写
    rollingUpdate:
      maxSurge: 1        # 每次只多启动1个新Pod
      maxUnavailable: 0  # 绝不允许服务不可用（蓝绿效果）
    maxUnavailable: 0 = 真正的无损发布（全程所有 Pod 都在线）
```

### 四、一条命令完成滚动更新 
```
kubectl set image deployment/你的部署名 容器名=新镜像:版本
kubectl set image deploy/myapp myapp=nginx:1.25
```

### 五、查看更新过程 
```
kubectl rollout status deployment/myapp
```
实时看新旧 Pod 替换运行    
```
kubectl get pods -w
```
### 六、更新失败？一键回滚！ 
滚动更新最大的好处：能随时回退
```
kubectl rollout undo deployment/myapp
```
回到指定版本也可以：
```
kubectl rollout history deploy/myapp
kubectl rollout undo deploy/myapp --to-revision=2
```

###七、特点总结（面试必背） 
• 不停机发布 • 逐步替换，不会同时杀旧 Pod 
• 可监控、可暂停、可回滚 
• Deployment 自带能力 
•是 K8s 生产环境标准发布方式   

### 八、一句话面试答案 K8s 的滚动更新是 Deployment 默认的更新策略
通过先启动新 Pod、就绪后再删除旧 Pod的方式逐步替换应用实例，结合 maxSurge 和 maxUnavailable 控制发布速度，实现无损、不中断的业务更新，并支持失败回滚。   
• 滚动更新 = 逐步替换，服务不中断
• 核心：先启新 → 再删旧 
• 关键参数：maxSurge、maxUnavailable 
• 支持：实时查看、回滚、历史版本



#Heml

**Helm = Kubernetes 的 应用安装包管理器 / 一键安装工具**

你可以把它理解成：

- **手机里的应用商店（App Store）**
- **电脑里的软件管家（360 软件管家）**
- **Linux 里的 yum /apt 安装工具**

只不过，Helm 是**专门给 K8s 用的**。

------

## 最通俗的例子：装 Nginx

不用 Helm 时，你在 K8s 装 Nginx 要：

1. 写 deployment.yaml
2. 写 service.yaml
3. 写 ingress.yaml
4. 写 configmap.yaml
5. 写 secret.yaml
6. 一条一条 `kubectl apply`

**又麻烦、又容易错、又乱。**

------

用 **Helm** 装 Nginx：

```
helm install my-nginx bitnami/nginx
```

**一行命令，全部搞定！**

Deployment、Service、Ingress、ConfigMap 一次性全部装好。

------

## 核心比喻（一定要记住）

### K8s = 房子

### 各种 yaml = 砖头、水泥、钢筋

## #Helm = 精装修套餐包

你不用自己搬砖、砌墙，**买个套餐，一键入住**。

这个**套餐**，在 Helm 里就叫：

## **Chart**

------

##Helm 里 3 个核心名词（超简单）

### 1. Chart

**应用安装包**

里面包含了运行这个软件需要的**所有 yaml 文件**

（Nginx、MySQL、Redis、Elasticsearch、Jenkins 都有）

### 2. Release

**Chart 安装后的运行实例**

就是你在 K8s 里真正跑起来的服务

### 3. Repository

**应用商店**

网上一堆公共 Chart 仓库，你可以直接搜软件安装

------

## Helm 能帮 SRE 解决什么痛？

你作为 SRE，每天要装各种中间件：

MySQL、Redis、ES、Kafka、RabbitMQ、Grafana、Prometheus...

**不用 Helm：**

- 写一堆 yaml
- 容易写错
- 版本混乱
- 无法批量更新
- 无法回滚
- 无法统一管理

**用 Helm：**

- 一行命令安装
- 一行命令更新
- 一行命令回滚
- 统一配置管理
- 环境一致性（测试 / 线上一样）
- 一键卸载

------

## 最常用的 7 条 Helm 命令（SRE 必背）

```
# 添加软件源（应用商店）
helm repo add bitnami https://charts.bitnami.com/bitnami

# 搜索应用
helm search repo mysql

# 安装应用
helm install my-mysql bitnami/mysql

# 查看已安装的应用
helm list

# 更新版本
helm upgrade my-mysql bitnami/mysql

# 回滚版本
helm rollback my-mysql 1

# 卸载
helm uninstall my-mysql
```

------

## 最核心的总结（记这 3 句就够）

1. **Helm 是 K8s 的应用安装管理器**
2. **Chart = 应用安装包（包含所有 yaml）**
3. **一行命令安装、更新、回滚、卸载服务，超级方便**



# k8s证书以及RBAC

## 1. Kubernetes 认证体系基于什么？为什么必须用证书？
基于 TLS 证书 + 秘钥 双向认证。
所有组件（apiserver、etcd、kubelet、controller-manager 等）通信必须加密。
防止中间人劫持、伪造请求、未授权访问集群。
一句话总结：
**证书是集群身份的唯一凭证，没有证书连 apiserver 都无法访问。**

## 2. k8s 里最重要的证书是什么？ca.key 为什么绝对不能泄露？
最重要：CA 根证书（ca.crt + ca.key）
ca.key 可以签发任意组件、任意用户的证书，拿到它就等于拥有 cluster-admin 集群最高权限。
生产只能放在 master 节点，绝不复制到 node、本地、代码仓库。

## 3. 证书过期会出现什么现象？怎么检查、怎么续签？
现象：
node 节点 NotReady
kubectl 无法连接 apiserver
组件日志报 x509: certificate has expired 错误
整个集群几乎不可用
检查：

```bash
kubeadm certs check-expiration
```
续签：
```bash
kubeadm certs renew all
```
面试加分：
证书默认 1 年，生产要做定期巡检 + 自动轮换，避免半夜集群挂掉。

## 补：生成证书的过程

## 1. 生成私钥（key）

```bash
openssl genrsa -out tls.key 2048
```

## 2. 生成证书请求文件（csr）

```bash
openssl req -new -key tls.key -out tls.csr
```

## 3. 生成自签证书（crt）

```bash
openssl x509 -req -days 365 -in tls.csr -signkey tls.key -out tls.crt
```

### 4. 在 K8s 中创建 Secret 存放证书

```bash
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```
### K8s 证书放在哪里？（必问）

```bash
/etc/kubernetes/pki/
```

里面最重要：

- **ca.crt**  根证书
- **ca.key** 根秘钥（最高权限）

------

### 四、证书过期了怎么办？

```bash
kubeadm certs check-expiration  # 查看过期时间
kubeadm certs renew all         # 一键续签
```


## 4. RBAC 中的 4 个核心资源是什么？分别作用？
- Role：命名空间级别权限
- ClusterRole：集群全局级别权限
- RoleBinding：把 Role 绑定给用户 / SA
- ClusterRoleBinding：把 ClusterRole 绑定给用户 / SA
一句话：
**Role 定义权限，Binding 把权限赋给人 / 账号。**

## 5. RBAC 规则里 apiGroups、resources、verbs 分别代表什么？
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
  apiGroups：API 分组，空字符串代表核心组（pod、svc 等）
  resources：资源对象（pods、deployments、secrets 等）
  verbs：操作权限（get、list、watch、create、update、delete 等）
```

## 6. 生产中 RBAC 权限设计原则是什么？

- 最小权限原则：给够用的，不给多余的
- 按 namespace 隔离：开发只能看自己业务空间
- 禁止普通用户使用 cluster-admin
- Pod 内部 SA 权限最小化，不需要就关闭自动挂载 token

## 7. Role 和 ClusterRole 区别？什么场景用哪个？
Role：局限在单个命名空间，适合业务开发、测试账号。
ClusterRole：全局有效，适合运维、监控、CI/CD 系统账号。
场景：
开发看自己项目 → Role
运维看整个集群节点、组件 → ClusterRole

## 8. ServiceAccount 是什么？为什么 Pod 里默认会挂载 token？
SA 是给 Pod 用的内部账号，用于容器内访问 apiserver。
默认自动挂载 /var/run/secrets/kubernetes.io/serviceaccount/
风险：如果 Pod 被入侵，攻击者可利用 token 操作集群。
加固：不需要访问 apiserver 的 Pod 直接关闭：

```yaml
automountServiceAccountToken: false
```

## 9. 如何给一个开发人员只开放某个 namespace 的只读权限？
思路：
- 建 Role（只读 pods、svc、cm 等）
- 建 RoleBinding 绑定到该用户
- 限定 namespace
- 面试官听到这几点就够：
命名空间隔离
1. 只给 get/list/watch

2. 不给 create/delete/update

3. 不绑定 ClusterRole

## 12.RBAC的配置

一句话流程：
**创建账号 → 创建权限 → 绑定权限 → 验证权限**
1. 第 1 步：创建用户 / ServiceAccount
	给人用 → User
	给 Pod/CI/CD 用 → ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dev-user
  namespace: prod
```
2. 第 2 步：创建 Role（定义权限）
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: dev-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
```
3. 第 3 步：创建 RoleBinding（绑定账号与权限）
```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
  namespace: prod
  name: dev-bind
  subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: prod
  roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```
  第 4 步：验证权限
```bash
kubectl auth can-i list pods --as=system:serviceaccount:prod:dev-user -n prod
```

返回 yes 即配置成功。

   

## 11. 集群安全方面，除了证书和 RBAC，你还做过哪些加固？
可以说 3 点，非常加分：
- 关闭 apiserver 匿名访问 --anonymous-auth=false
- 启用 NetworkPolicy 限制 Pod 间访问
- 开启审计日志，记录所有 apiserver 操作
- 限制 kubelet 权限，启用 NodeRestriction 插件
- 敏感配置放 Secret，不用 ConfigMap 存密码

## 终极万能收尾句
Kubernetes 安全以证书认证为基础，以 RBAC 最小权限为核心，配合网络策略、审计日志、密钥管理，形成完整的集群安全体系，满足金融级安全要求。

## 证书配置：

1. K8s 所有组件使用 TLS 证书双向认证。
2. 证书默认存放在 `/etc/kubernetes/pki`。
3. 最重要的是 CA 根证书，用来签发所有组件证书。
4. 可以用 kubeadm 自动生成，也可以手动生成、手动续签。
5. 生产必须保证证书不过期、不泄露、不滥用。

## RBAC 配置：

1. 使用 RBAC 实现最小权限控制，分为 Role、ClusterRole、RoleBinding、ClusterRoleBinding。
2. 先创建用户或 ServiceAccount。
3. 再创建 Role 定义允许操作的资源与权限。
4. 通过 RoleBinding 把账号和权限绑定。
5. 最后用 kubectl auth can-i 验证权限是否正确

# k8s中的动态扩缩容

K8s 动态扩缩容，是根据CPU、内存、自定义指标、流量负载，自动调整 Pod 副本数量，实现业务负载高自动扩容、负载低自动缩容，保障稳定性同时节省资源。

二、三种扩缩容方式（面试必答）

1. HPA 水平 Pod 自动扩缩容（最常用）
定义：横向增减 Pod 副本数，加实例不加配置。
监控指标：CPU 利用率、内存利用率、QPS、自定义业务指标
适配场景：微服务、接口服务、无状态应用
工作原理：
	HPA 控制器定期从 Metrics Server 获取 Pod 指标
	对比设定阈值，计算需要的副本数
	调用 Deployment 调整 replicas 数量

2. VPA 垂直 Pod 自动扩缩容
- 定义：不增加 Pod 数量，调整单个 Pod 的 CPU / 内存 requests、limits。
- 适配场景：Java 应用、需要固定实例数但资源要动态调整的服务
- 特点：会重启 Pod 生效，生产一般谨慎用

3. Cluster Autoscaler 集群节点自动扩缩容
- 定义：Pod 调度不上来时，自动新增节点；节点空闲时自动缩容销毁节点。
- 触发场景：HPA 扩容到没有节点可调度，处于 Pending 状态
- 层级：Pod 不够→HPA 扩 Pod；节点不够→CA 扩节点

三、HPA 工作流程（面试口述版）
- 依赖 Metrics Server 收集集群 CPU、内存指标；
- HPA 每 15 秒（默认）采集一次指标；
- 计算当前使用率和目标阈值差值；
- 自动调整 Deployment/StatefulSet 的副本数；
- 有冷却时间，防止频繁抖动、瞬间反复扩缩。

四、HPA 支持的指标类型
- 资源指标：CPU、内存使用率（最常用）
- 自定义指标：QPS、连接数、队列长度
- 外部指标：外网流量、消息队列堆积量

五、HPA 核心配置要点
- 必须给容器配置 requests/limits，不然 HPA 无法计算资源使用率
- 配置最小副本、最大副本，防止无限扩容
- 配置扩容、缩容冷却时间，避免流量抖动

六、三者层级关系（面试加分）
- 流量上涨 → HPA 扩 Pod → 节点资源不足 Pod Pending → CA 扩节点
- 流量下降 → HPA 缩 Pod → 节点空闲资源多 → CA 缩节点

七、面试一句话总结
K8s 动态扩缩容分三层：HPA 横向扩缩 Pod、VPA 纵向调整 Pod 资源、Cluster Autoscaler 自动扩缩集群节点。生产环境最常用 HPA 基于 CPU / 内存和业务指标自动弹性，配合 Metrics Server 采集指标，实现业务负载自动弹性，保障高可用又节约集群资源。

# k8s的资源隔离

###1. 容器底层基础
容器并非虚拟机，本质是运行在宿主机内核之上的受限进程，依靠 Linux 内核两大核心技术实现隔离与管控：
- Namespace 命名空间：实现资源视图隔离，让不同进程看到独立的主机名、网络、进程、文件挂载等资源视图；
- Cgroup 控制组：实现硬件资源限制，限制进程组可使用的 CPU、内存、磁盘 IO、网络带宽等资源；
二者组合构成Linux 原生轻量级隔离环境，也就是容器最基础的沙箱雏形。

### 2. Pod 核心定义
- Pod 是 Kubernetes 最小调度、部署、网络调度单元，一个 Pod 内可运行一个或多个业务容器。
- K8s 不直接创建业务容器隔离环境，而是通过Pause 容器统一预创建隔离环境，再让业务容器接入该环境。

### 3. Pause 容器定义
- Pause 是极简静态容器，无业务逻辑、无业务进程，仅负责创建、持有、维持整套 Pod 级隔离环境，是 K8s 默认容器模式下Pod 沙箱的实体载体。

### 4. 沙箱定义
沙箱是一套划定运行边界、限制行为、隔离风险的标准化运行环境，分为两类：
- 原生轻量沙箱：基于 Namespace+Cgroup 实现，共享宿主机内核；
- 强隔离安全沙箱：基于微型虚拟机实现，拥有独立内核，彻底隔离宿主机。

### 5. Pause 容器与沙箱的精准关系
概念区分
- 沙箱：抽象的隔离环境规则与边界（隔离策略、资源限制、权限管控）；
- Pause 容器：承载这套沙箱环境的常驻运行实例；

在 K8s 默认容器运行时（containerd/docker）环境中：
Pause 容器就是 Pod 对应的默认轻量沙箱；
沙箱环境的生命周期、网络栈、资源组全部依附 Pause 容器存在。
生命周期绑定
1. Pause 启动 → Pod 沙箱环境初始化完成；
2. Pause 正常运行 → 沙箱环境持续保留；
3. Pause 终止 / 销毁 → 整套沙箱环境直接释放，Pod 内所有业务容器强制退出。

### 6.基于 Pause 沙箱：Pod 内部共享机制（底层原理）
1. 实现原理
  Pod 启动顺序固定：优先启动 Pause 容器，由 Pause 预先创建好统一的 Namespace 集合与 Cgroup 资源组；后续所有业务容器不再新建命名空间，直接加入 Pause 已创建的隔离环境，以此实现资源共享。
2. 可共享资源明细
	（1）网络命名空间 NET（核心共享）
	底层原理：所有容器共用 Pause 创建的唯一网络命名空间；
	实际效果：
		1. 整个 Pod 仅分配一个集群 IP，所有容器共用；
		2.共享虚拟网卡、路由表、iptables 规则、socket 连接；
		3. 同 Pod 内容器可通过 127.0.0.1 本地互通，无需走集群网络；
		4. 端口全局互斥，同一端口仅能被一个容器监听。
	（2）UTS 主机名命名空间
	所有容器共用同一主机名、域名解析配置，视图完全一致。
	（3）IPC 进程间通信命名空间
	同 Pod 容器共享消息队列、共享内存等进程通信机制，支持高性能本地进程交互。
	（4）PID 进程命名空间（可选开启）
	配置 shareProcessNamespace: true 后，共用进程视图，可互相查看进程、管理进程、清理僵尸进程。
	（5）Cgroup 资源组
	整个 Pod 所有容器归属同一个 Cgroup，CPU、内存等资源为 Pod 整体配额，内部容器相互抢占资源，无法超出总限制。
	（6）存储卷 Volume 共享
	底层逻辑：容器自身根文件系统相互独立，K8s 将外部统一存储目录，挂载至多个业务容器相同路径；
	效果：挂载目录内文件双向互通读写，实现日志、配置、数据文件共享。

### 7.基于 Pause 沙箱：Pod 内部隔离机制（底层原理）
共享仅局限于 Pause 统一创建的公共命名空间，容器自身运行环境严格隔离：
1. MNT 挂载命名空间强隔离
每个业务容器拥有独立挂载命名空间，使用自身镜像内置的根文件系统；
不同容器的系统目录、内置程序、依赖库、内置配置完全不可见、互不干扰。
2. 运行生命周期隔离
各容器拥有独立启动命令、独立主进程、独立重启策略；
单个业务容器崩溃、重启、终止，不会影响同 Pod 其他容器与 Pause 沙箱。
3. 用户权限隔离
可单独为不同容器配置不同运行用户、UID、权限体系，实现运行权限隔离，避免权限越界。
4. 日志与配置隔离
容器自身标准输出、内置配置文件、环境变量默认独立，仅共享手动挂载的外部配置。

### 8.跨 Pod 隔离原理
每一个独立 Pod 都会创建专属独立的 Pause 沙箱：
不同 Pod 拥有完全独立的网络命名空间、独立集群 IP；
拥有独立 Cgroup 资源限制、独立进程视图；
默认二层网络隔离，无法直接通信，必须借助 Service、Ingress 等组件完成流量转发。

### 9.K8s 两类沙箱完整对比
1. 第一种：默认原生轻量沙箱（Pause 承载）
技术实现：Linux Namespace + Cgroup + 权限裁剪；
运行载体：宿主机原生容器，依托 Pause 构建 Pod 环境；
内核形态：共享宿主机内核；
隔离等级：进程级轻隔离；
安全特点：存在容器逃逸漏洞风险，系统调用限制较弱；
性能特点：启动快、资源占用极低、集群损耗小；
适用场景：绝大多数普通微服务、后端业务、日常生产业务。
2. 第二种：虚拟机级强安全沙箱（Kata Containers /gVisor）
技术实现：虚拟化技术，构建微型独立虚拟机；
运行载体：外层为独立虚拟机沙箱，虚拟机内部依旧运行微型 Pause 管理 Pod 容器；
内核形态：拥有独立内核，与宿主机内核完全隔离；
隔离等级：系统级强隔离；
安全特点：彻底杜绝容器逃逸，严格拦截高危系统调用，恶意代码无法穿透沙箱；
性能特点：启动速度慢，CPU、内存资源开销更高；
适用场景：金融业务、第三方不可信代码运行、公有云多租户、敏感数据服务。
3. 两层沙箱从属关系
普通模式：Pause = 唯一 Pod 沙箱，所有共享隔离规则由它定义；
强安全模式：外层虚拟机 = 全局安全沙箱，内层 Pause 仅负责虚拟机内部 Pod 容器的共享与环境管理。

### 10.Pod 标准完整启动流程
1. Kubelet 调用容器运行时（containerd）下发创建 Pod 请求；
2. 运行时优先创建 Pause 容器，初始化网络命名空间、PID/IPC/UTS 命名空间、Pod Cgroup 资源组、虚拟网卡与 Pod IP；
3. Pause 容器进入休眠状态，持续持有整套沙箱隔离环境；
4. 依次启动所有业务容器，让业务容器全部接入 Pause 已建好的沙箱环境；
5. 统一挂载声明的存储卷，完成目录共享配置；
6. 就绪探针检测通过，Pod 状态更新为 Running；
7. 销毁流程：先停止所有业务容器，最后销毁 Pause 容器，释放整套沙箱隔离资源。

### 11. 面试精简总结
Pause 容器是 K8s 默认环境下 Pod 的标准沙箱，负责搭建并持有 Pod 所有共享隔离环境；
同 Pod 容器共享由 Pause 统一创建的网络、进程通信、主机名视图与整体资源配额；
同 Pod 容器依靠独立挂载命名空间、独立运行进程实现文件系统与运行环境隔离；
原生容器沙箱依托宿主机内核实现

### 12.容器运行时调用链

1. 四大核心概念 
	- **CRI** 容器运行时接口，**K8s专属通信标准**，kubelet与容器引擎的调用协议，只是接口，不是运行时。  
	- **OCI** 全球通用容器开源标准，统一**镜像格式、容器运行规范**，行业通用规则。
	- **containerd** **高层容器运行时**，实现CRI接口，管理镜像、容器生命周期，K8s主流默认运行时。
	- **runc** **底层低级运行时**，遵循OCI标准，直接调用内核资源，**真正创建启动容器**。
2. 完整调用链路 
	- kubelet → CRI接口 → containerd → pause容器 → runc → Linux内核(namespace+cgroup)
3. Pause容器定位与作用 
	- 层级：属于**容器运行时层**，由containerd最先创建 
	- 地位：Pod内**第一个启动**的基础静默容器 
	- 核心作用：独占Pod网络命名空间，实现Pod内所有容器**共享网络、IP、端口** ## 四
4. 区分
	- CRI：K8s定的**调用规矩** 
	- OCI：容器行业**通用规矩** 
	- containerd：中间**管理者** 
	- runc：底层**执行者** 
	- pause：Pod**网络基座容器** 
5. 架构淘汰知识点 
	- K8s 1.24移除dockershim，**不再原生支持Docker**，统一改用containerd。  
6. 面试终极总结 
	- K8s通过**CRI**调用容器运行时containerd（高层容器运行时），高层容器运行时优先拉起pause容器，containerd再依靠**OCI标准**调用底层runc启动容器，runc（底层容器运行时）驱动linux内核（namespace，cgroup），最后启动pod的业务容器。
	- 旧 Docker 架构：kubelet → dockershim → docker → containerd → runc → 内核

# 沙箱技术 kata和gVisor
## 1. 核心概念对比（通俗比喻）

- **传统容器**：像住在公寓楼里，大家共用同一扇大门（宿主机内核），用墙壁（Namespace+Cgroup）隔开房间；便宜但不安全，小偷可能撬门影响整栋楼。
- **虚拟机**：像住在独立别墅，有自己的大门（独立内核）和围墙（硬件虚拟化）；安全但“成本高”，建房子慢、占地大。
- **Kata Containers**：像住在**移动别墅/轻量房车**里，既有别墅的独立安全（自己的内核+硬件隔离），又有公寓的便宜快捷（秒级启动、资源占用低）。

## 2. 工作原理三步曲

### 第一步：启动轻量虚拟机

当用K8s或Docker创建容器时，Kata不会直接启动容器，而是**先创建一个超级精简的微型虚拟机（microVM）**，相当于给容器准备了一个独立的“小操作系统”：

- 虚拟机仅几十MB大小，启动仅需几秒（比传统VM快10倍+）；
- 采用专门裁剪的Linux内核，只保留容器必需功能，剔除多余组件；
- 通过硬件虚拟化技术（如Intel VT-x/AMD-V）隔离，与宿主机内核彻底分离。

### 第二步：在虚拟机里跑容器

虚拟机启动后，Kata会在其内部启动一个**标准容器运行时**（如runc），再通过该运行时创建业务容器：

- 容器完全兼容Docker/OCI镜像，以为自己在普通Linux系统中运行；
- 同Pod的多个容器，会共享该虚拟机的网络、IPC等资源，与普通Pod行为一致；
- Pause容器也会在虚拟机内运行，负责管理Pod内容器的共享环境。

### 第三步：无缝衔接容器生态

Kata对外伪装成普通容器运行时，K8s、Docker完全感知不到其背后使用了虚拟机：

- 支持所有容器操作命令（run、exec、stop等）；
- 兼容Service、Ingress等K8s组件，网络流量正常转发；
- 资源监控、日志收集等工具可正常工作。

## 3. 核心组件

| 组件                                            | 作用                                 | 通俗比喻                           |
| :---------------------------------------------- | :----------------------------------- | :--------------------------------- |
| kata-runtime                                    | 调度器，负责创建虚拟机和管理生命周期 | 包工头，负责建房车和安排入住       |
| containerd-shim-kata-v2                         | 连接Docker/containerd和Kata的桥梁    | 物业前台，处理外面的请求           |
| kata-agent                                      | 虚拟机内的“管家”，管理容器生命周期   | 房车里的智能管家，负责日常服务     |
| 轻量虚拟机监控器（QEMU/Dragonball/Firecracker） | 硬件虚拟化引擎，创建隔离环境         | 房车的底盘和安全系统，提供物理隔离 |

# 二、gVisor vs Kata：两种安全容器的核心区别

## 1. 核心设计理念：两条完全不同的路

### gVisor：给容器加“安全门卫”

不搞虚拟机，而是在容器和宿主机内核之间，加了一个**用户态内核（Sentry）**，相当于超级严格的门卫：

- 容器的所有系统调用（读文件、联网等），都必须经过Sentry检查和处理；
- Sentry用Go语言开发，内存安全、漏洞少，仅实现容器常用的200+系统调用；
- 配合Gofer组件代理文件系统访问，彻底阻止容器直接接触宿主机文件。

### Kata：给容器建“独立小房子”

直接用**轻量虚拟机**把容器包起来，相当于给每个容器建了独立小房子：

- 容器有自己的内核，与宿主机内核物理隔离，即使容器被攻破也无法触及宿主机；
- 虚拟机提供完整的Linux系统环境，兼容性几乎100%；
- 通过硬件虚拟化技术实现隔离，安全性与传统VM同级。

### 2. 关键对比：优缺点一目了然（通俗版）

| 对比维度     | gVisor                                   | Kata Containers                 | 通俗结论                                  |
| :----------- | :--------------------------------------- | :------------------------------ | :---------------------------------------- |
| 隔离强度     | 中高（用户态内核拦截）                   | 极高（硬件虚拟化+独立内核）     | Kata更安全，像别墅vs公寓升级版            |
| 启动速度     | 快（毫秒-秒级）                          | 较快（秒级，比gVisor慢一点）    | 都比传统VM快，gVisor略胜                  |
| 资源占用     | 低（每容器几十MB内存）                   | 中（每容器几百MB内存）          | gVisor更省资源，适合高密度部署            |
| 兼容性       | 中（部分系统调用不支持，如ptrace、perf） | 高（几乎兼容所有Linux应用）     | Kata像“万能插座”，gVisor有“不兼容设备”    |
| 系统调用性能 | 一般（复杂调用会有开销）                 | 好（接近原生，硬件加速）        | 跑数据库/高IO应用选Kata，简单应用选gVisor |
| 硬件依赖     | 无（纯软件实现）                         | 有（需要CPU支持虚拟化）         | 老服务器用gVisor，新服务器用Kata更安全    |
| 适用场景     | 公共云多租户、轻量无状态服务             | 金融/政务敏感数据、高兼容性需求 | 安全第一选Kata，平衡性能选gVisor          |

### 3. 通俗比喻总结

- **gVisor**：给应用装了个**智能安全管家**，所有对外请求都要经过它检查；管家能力有限，但开销小。

- **Kata Containers**：给应用租了个**带独立安保的公寓**，有自己的门和保安；能力全面，但“租金”稍高。

### 4. 使用场景
- gVisor 解决的核心需求
  	1. 纯软件强隔离，不依赖 CPU 虚拟化硬件
  	2. 老旧服务器、低配集群也能部署，适配性极强。
  	3. 极致节省资源，支持超大规模高密度部署
  	4. 内存、CPU 开销极小，单节点可批量运行上百个隔离任务实例。
  	5. 严格拦截危险系统调用
  	6. 只放行业务必需调用，直接封杀绝大多数入侵、提权、恶意操作路径。
  	7. 轻量任务安全隔离
  	8. 用于无状态服务、通用自动化流程、普通业务脚本隔离。
- Kata Containers 解决的核心需求
  	1. 硬件级彻底隔离，彻底杜绝容器逃逸
  	2. 独立微型虚拟机 + 独立内核，容器被攻破也触碰不到宿主机内核，安全等级拉满。
  	3. 全场景完美兼容
  	4. 不限制系统调用，复杂程序、数据库、中间件、桌面程序都能正常运行，无兼容性报错。
  	5. 生产级稳定，高性能
  	6. 接近原生容器运行性能，适合核心业务、高 IO、高并发服务。
  	7. 涉密、高权限业务专属隔离
  	8. 满足数据加密、资金流程、政务涉密流程等高安全场景。

### 5. 实在智能业务场景选型建议

结合实在智能RPA/Agent业务，选型优先级如下：

1. **日常RPA流程**：选用gVisor，资源占用低、启动快，适配高密度部署需求；
2. **金融/政务高危流程**：选用Kata，独立内核+硬件隔离，彻底防止容器逃逸；
3. **Windows桌面自动化**：两者均需配合Windows容器/虚拟桌面技术，Kata兼容性更优。

轻隔离，性能优异；Kata 类沙箱依托独立虚拟机实现强安全隔离，牺牲性能换取安全；
Pod 多容器设计核心目的：共享网络实现高效本地通信，独立运行环境保障业务稳定性，完美适配 Sidecar 架构。

