# Prometheus \+ Grafana 必考点（运维开发必考，拓展版）

## 一、Prometheus 是什么？（面试标准回答，拓展细节）

Prometheus 是一款开源的**时序数据库（TSDB）**，核心用于监控系统指标、业务指标的采集、存储、查询和告警，采用「拉取式（Pull）」为主的采集模式，也支持推送式（Push，需配合PushGateway），适配传统运维和云原生（K8s）场景，是东软运维自动化、监控体系中的核心工具之一。

补充面试加分点：

- 核心特点：高可用、可扩展、原生支持时序数据（按时间戳存储指标，适合监控场景），自带简单的可视化界面（但不如Grafana灵活，所以实际部署中常和Grafana搭配）。

- 适用场景：系统层（CPU、内存等）、应用层（接口QPS、错误率等）、中间件（MySQL、Redis等）、云原生组件（K8s Pod、Deployment等）的监控。

- 与Zabbix区别（面试官可能追问）：Prometheus更轻量、适合云原生，拉取模式更灵活；Zabbix更成熟、适合传统服务器监控，主动/被动模式都支持，各有优势，东软项目中两者都有使用。

## 二、核心组件（拓展功能、实操场景，面试必说）

Prometheus 核心组件分工明确，日常运维中需掌握各组件的作用、配置和基础排查，以下是面试高频表述：

### 1\. Exporter（指标采集器，核心）

Metrics 就是系统暴露的可量化监控数据，K8s 靠它实现监控、告警、HPA 自动扩缩容；Exporter 负责把各种组件转成 Metrics 格式给 Prometheus。

作用：负责采集目标对象（服务器、中间件、应用）的监控指标，将其转换为Prometheus可识别的格式，暴露一个HTTP接口（默认9100端口，不同Exporter端口不同），供Prometheus Server拉取。

高频考点\+拓展：

- 常用Exporter
  
    - node\_exporter：监控Linux服务器（核心，必说），采集CPU、内存、磁盘、IO、网络、负载等系统指标，是最常用的Exporter。
    
    - mysqld\_exporter：监控MySQL数据库，采集连接数、慢查询、表空间、主从同步状态等指标。
    
    - redis\_exporter：监控Redis，采集内存使用、连接数、命中率、key数量等指标。
    
    - nginx\_exporter：监控Nginx，采集请求量、错误率、响应时间等指标。
    
    - 自定义Exporter：若现有Exporter不满足业务需求，可通过Python/Shell编写简单Exporter（面试说这句话，加分），暴露自定义指标（如业务接口成功率、订单量等）。
    
- 实操细节：部署Exporter后，需在Prometheus Server的配置文件（prometheus\.yml）中添加「采集任务（job）」，指定Exporter的IP和端口，Prometheus会定期拉取指标。

### 2\. Prometheus Server（核心中枢）

作用：核心组件，负责三大核心操作——拉取指标、存储指标、提供查询接口（PromQL）。

高频考点\+拓展：

- 指标拉取：默认每15秒拉取一次Exporter的指标（可在配置文件中修改scrape\_interval），拉取失败会记录日志，可通过Prometheus自带界面查看拉取状态。

- 指标存储：默认将指标存储在本地磁盘（TSDB格式），也可配置远程存储（如Thanos、InfluxDB），解决本地存储容量不足、数据持久化问题（东软项目中，核心业务会配置远程存储，面试可提及）。

- 查询功能：提供PromQL查询语言（面试必提），可通过PromQL查询任意时间段的指标、计算指标（如增长率、平均值），用于排查问题、配置告警阈值。

- 高可用配置：可部署多个Prometheus Server，通过联邦集群（Federation）实现指标聚合，避免单点故障（面试提及，体现高可用意识，加分）。

### 3\. Alertmanager（告警管理组件）

作用：接收Prometheus Server发送的告警信息，对告警进行去重、抑制、分组、路由，最终发送到指定的告警媒介（钉钉、企业微信、邮件等），避免告警风暴，确保运维人员及时收到有效告警。

高频考点\+拓展（面试重点）：

- 核心功能（必说）：
       

    - 去重：同一指标多次触发告警，合并为一条，避免重复告警。

    - 抑制：当某个核心告警触发（如服务器宕机），会抑制该服务器上的其他次要告警（如端口不通），减少无效告警。

    - 分组：将同一类型、同一服务器的告警分组发送，便于运维人员批量处理（如多台服务器CPU高，分组后一次性接收）。

    - 路由：根据告警级别、告警类型，将告警发送到不同的接收人/渠道（如严重告警发电话\+钉钉，一般告警发邮件）。

- 实操细节：需单独部署Alertmanager，在Prometheus Server配置文件中指定Alertmanager的地址，让Prometheus将告警发送给Alertmanager。

### 4\. Grafana（可视化面板，核心搭配工具）

作用：开源的可视化工具，不负责采集和存储指标，主要用于连接Prometheus（及其他数据源），通过拖拽方式制作监控面板，将指标以图表（折线图、柱状图、仪表盘等）形式展示，直观呈现监控数据，支持大屏展示、多数据源切换。

高频考点\+拓展：

- 核心操作（必说）：
  
    - 添加数据源：在Grafana中添加Prometheus数据源，配置Prometheus Server的IP和端口，即可获取指标数据。
    
    - 导入模板：无需手动制作面板，可从Grafana官方模板库（Dashboards）导入现成模板（如node\_exporter的系统监控模板、MySQL监控模板），修改适配自身环境（实操性强，面试说这句话，加分）。
    
    - 自定义面板：根据业务需求，拖拽图表、配置PromQL查询语句，制作自定义面板（如业务接口监控面板、集群监控大屏）。
    
- 面试加分点：Grafana支持告警配置（可直接对接Prometheus指标），但实际运维中，更常用Prometheus\+Alertmanager负责告警，Grafana专注于可视化，分工更清晰。

## 三、常用监控指标（拓展含义、PromQL用法，面试直接说，可直接背诵）

以下指标均为node\_exporter采集的系统层核心指标，是东软运维开发岗面试高频提问点，需掌握指标含义\+简单PromQL用法，避免只说指标名，体现实操能力。

- 1\. node\_load15
        

    - 含义：服务器15分钟内的系统负载（核心指标），负载值一般建议不超过服务器CPU核心数（如8核CPU，负载≤8为正常）。

    - PromQL用法（面试必说）：直接查询当前负载 → node\_load15；查询负载占比 → node\_load15 / count\(node\_cpu\_seconds\_total\{mode=\&\#34;idle\&\#34;\}\) （计算15分钟负载与CPU核心数的比值，判断是否过载）。

- 2\. node\_cpu\_seconds\_total
        

    - 含义：CPU累计使用时间（单位：秒），按CPU模式（idle空闲、user用户态、system内核态）区分标签。

    - PromQL用法（面试必说）：计算CPU使用率（5分钟内）→ 1 \- rate\(node\_cpu\_seconds\_total\{mode=\&\#34;idle\&\#34;\}\[5m\]\) \* 100 （核心用法，排查CPU高的关键）。

- 3\. node\_memory\_MemAvailable\_bytes
        

    - 含义：服务器可用内存大小（单位：字节），比node\_memory\_MemFree\_bytes更准确（包含可回收缓存）。

    - PromQL用法（面试必说）：查询可用内存 → node\_memory\_MemAvailable\_bytes / 1024 / 1024 / 1024 （转换为GB，直观查看）；计算内存使用率 → \(1 \- node\_memory\_MemAvailable\_bytes / node\_memory\_MemTotal\_bytes\) \* 100。

- 4\. node\_filesystem\_avail\_bytes
        

    - 含义：磁盘分区的可用空间（单位：字节），按磁盘分区（如/、/home）区分标签。

    - PromQL用法（面试必说）：查询某个分区可用空间 → node\_filesystem\_avail\_bytes\{mountpoint=\&\#34;/\&\#34;\} / 1024 / 1024 / 1024；计算磁盘使用率 → \(1 \- node\_filesystem\_avail\_bytes\{mountpoint=\&\#34;/\&\#34;\} / node\_filesystem\_size\_bytes\{mountpoint=\&\#34;/\&\#34;\}\) \* 100。

- 5\. rate\(node\_network\_receive\_bytes\_total\[5m\]\)
        

    - 含义：服务器网络接收速率（单位：字节/秒），\[5m\]表示取5分钟内的平均速率，避免瞬时值波动。

    - PromQL用法（面试必说）：查询接收速率 → rate\(node\_network\_receive\_bytes\_total\[5m\]\) / 1024 / 1024 （转换为MB/s）；查询发送速率 → rate\(node\_network\_transmit\_bytes\_total\[5m\]\) / 1024 / 1024。

补充：面试时，可主动提及“这些指标是日常排查系统问题的核心，比如通过CPU使用率指标定位CPU高的服务器，通过磁盘使用率指标预警磁盘满的风险”，体现运维思维。



## 五、告警怎么配置？（拓展完整步骤、实操配置，面试必背）

Prometheus告警配置核心分3步：配置PrometheusRule（告警规则）→ 配置Prometheus与Alertmanager联动 → 配置Alertmanager告警路由，以下是完整拓展，含面试常考的配置细节和示例，可直接背诵。

### 1. 第一步：写PrometheusRule（告警规则配置，核心）

作用：定义“什么情况下触发告警”，包括告警名称、告警条件（PromQL查询语句）、告警级别、告警描述等，可单独创建规则文件（如prometheus\-rules\.yml），在Prometheus配置文件中引入。

面试高频示例（东软实操场景，必背）：

- 规则文件示例（简化版，面试可直接说）：
        `groups:
    \- name: 系统监控告警规则
  rules:
  \# 1\. CPU使用率告警（5分钟内平均超过80%，持续2分钟触发）
  \- alert: CPU使用率过高
    expr: 1 \- rate\(node\_cpu\_seconds\_total\{mode=\&\#34;idle\&\#34;\}\[5m\]\) \* 100 \&gt; 80
    for: 2m  \# 持续时间，避免瞬时波动导致误告警（面试必提）
    labels:
      severity: warning  \# 告警级别（warning/error/critical）
    annotations:
      summary: \&\#34;CPU使用率过高\&\#34;
      description: \&\#34;服务器\{\{ $labels\.instance \}\}（IP：\{\{ $labels\.ip \}\}）CPU使用率超过80%，当前使用率：\{\{ $value \| round 2 \}\}%\&\#34;
  \# 2\. 磁盘使用率告警（根分区使用率超过85%，持续5分钟触发）
  \- alert: 磁盘空间不足
    expr: \(1 \- node\_filesystem\_avail\_bytes\{mountpoint=\&\#34;/\&\#34;\} / node\_filesystem\_size\_bytes\{mountpoint=\&\#34;/\&\#34;\}\) \* 100 \&gt; 85
    for: 5m
    labels:
      severity: error
    annotations:
      summary: \&\#34;磁盘空间不足\&\#34;
      description: \&\#34;服务器\{\{ $labels\.instance \}\}根分区使用率超过85%，当前使用率：\{\{ $value \| round 2 \}\}%\&\#34;`

- 关键细节（面试必说）：
        

    - expr：告警条件，用PromQL查询语句定义，必须返回布尔值（true触发告警）。

    - for：持续时间，指标满足告警条件后，持续一段时间才触发告警，用于避免误告警（东软项目中常用2\-5分钟，面试可直接说）。

    - labels：告警标签，用于分类（如级别、类型），方便Alertmanager路由。

    - annotations：告警描述，用于说明告警详情（如服务器IP、当前指标值），方便运维人员快速定位问题。

- 配置引入：在Prometheus的prometheus\.yml中，添加rule\_files配置，指定规则文件路径，示例：
  `rule\_files:
  \- \&\#34;prometheus\-rules\.yml\&\#34;  \# 规则文件路径，可配置多个`

### 2\. 第二步：配置Prometheus与Alertmanager联动

作用：让Prometheus Server检测到告警后，将告警信息发送给Alertmanager，需在Prometheus配置文件中添加Alertmanager的地址。

配置示例（面试必说）：
`alerting:
  alertmanagers:
  \- static\_configs:
    \- targets:
      \- \&\#34;192\.168\.1\.100:9093\&\#34;  \# Alertmanager的IP和端口（默认9093）`

细节：配置完成后，重启Prometheus，可在Prometheus自带界面（Alerts页面）查看告警规则、告警状态（正常/触发/ pending）。

### 3\. 第三步：配置Alertmanager，路由到钉钉/企业微信/邮件

作用：Alertmanager接收Prometheus的告警后，通过配置“告警媒介”和“路由规则”，将告警发送给指定接收人，避免告警遗漏。

面试高频配置（以钉钉为例，东软常用，必背）：

- 1\. 配置告警媒介（alertmanager\.yml）：
        `global:
  resolve\_timeout: 5m  \# 告警恢复后，5分钟内发送恢复通知

route:
  group\_by: \[\&\#39;alertname\&\#39;\]  \# 按告警名称分组
  group\_wait: 10s  \# 同一组告警，等待10秒后合并发送
  group\_interval: 10s  \# 同一组告警，两次发送间隔10秒
  repeat\_interval: 1h  \# 同一告警，重复发送间隔1小时（避免频繁骚扰）
  receiver: \&\#39;dingtalk\&\#39;  \# 默认告警接收人

receivers:
\- name: \&\#39;dingtalk\&\#39;
  webhook\_configs:
  \- url: \&\#39;https://oapi\.dingtalk\.com/robot/send?access\_token=xxx\&\#39;  \# 钉钉机器人webhook地址
    send\_resolved: true  \# 告警恢复后，发送恢复通知（面试必提，体现专业性）
    http\_config:
      tls\_config:
        insecure\_skip\_verify: true`

- 2\. 关键细节（面试必说）：
       

    - route：路由配置，定义告警分组、发送间隔，避免告警风暴。

    - receivers：告警接收人配置，支持钉钉、企业微信、邮件、电话等，东软项目中最常用钉钉/企业微信（即时通知）\+ 邮件（留存记录）。

    - send\_resolved：开启后，当告警恢复（指标回到正常阈值），会发送恢复通知，方便运维人员确认问题已解决。

### 4\. 告警配置完整流程（面试万能话术，必背）

“我们在实际运维中，Prometheus告警配置分3步：首先编写PrometheusRule规则文件，定义告警条件、持续时间和告警描述，比如CPU使用率超过80%持续2分钟触发告警；然后在Prometheus配置文件中，引入规则文件并配置Alertmanager地址，让Prometheus将告警发送给Alertmanager；最后配置Alertmanager的告警媒介和路由规则，将告警路由到钉钉和邮件，开启恢复通知，确保运维人员及时收到告警、确认问题，同时避免告警风暴。”

## 五、面试高频追问（补充，应对面试官延伸提问）

- 追问1：Prometheus的拉取模式有什么优势？

回答：拉取模式更灵活，无需在被监控端配置推送地址，只需被监控端部署Exporter、暴露接口，Prometheus即可主动拉取；便于水平扩展，新增被监控对象时，只需在Prometheus中添加采集任务，无需修改被监控端配置。

- 追问2：如何避免Prometheus告警误触发？

回答：1\. 配置for持续时间（如2\-5分钟），避免瞬时指标波动；2\. 合理设置告警阈值（结合服务器实际配置，如8核CPU阈值设为80%）；3\. 开启Alertmanager的去重、抑制功能，减少无效告警；4\. 定期调整告警规则，适配业务变化。

- 追问3：Prometheus指标数据如何清理？
        

回答：1\. 配置本地存储保留时间（默认15天），在prometheus\.yml中通过\-\-storage\.tsdb\.retention\.time设置（如30d）；2\. 核心业务指标配置远程存储（如Thanos），本地只保留短期热数据，长期冷数据归档到远程存储，节省本地磁盘空间。



普罗米修斯除了 Pull 还有什么模式（面试满分版）
核心结论（开口就说）
Prometheus 原生只有 Pull（拉取） 一种采集模式；Push（推送）不是原生模式，是靠 PushGateway 组件实现的补充方案；另外还有 Remote Write（远程推送） 用于数据转发。
1. Push 模式（PushGateway 中转）
原理：
业务 / 脚本 主动推 → PushGateway（缓存） ← Prometheus 拉
本质还是 Pull，只是加了一层中转网关。
适用场景：
短生命周期任务（批处理、CronJob、一次性脚本）
网络隔离、Prometheus 无法主动拉取
缺点（必说）：
丢失 up 健康检查
单点故障、数据会残留
不建议当常规采集方式用
2. Remote Write 远程推送（架构级）
原理：
Prometheus 本地拉取后，主动推送到远端时序存储（如 Thanos、M3DB、另一个 Prometheus）Prometheus。
作用：
集中监控、多集群汇总
长期存储、扩容
特点：
采集依然是 Pull，只在后端转发时用 Push。
3. 一句话区分（最稳）
Pull：默认、主流、健康检查、长稳服务
Push：靠 PushGateway、短任务 / 网络受限用
Remote Write：数据转发、集中存储
面试官满分回答（30 秒）
Prometheus 原生只有 Pull 拉取模式；推送是通过 PushGateway 实现的补充方案，主要用于短任务和网络隔离场景。另外还有 Remote Write 远程推送，用于把采集的数据转发到远端存储，实现多集群统一监控与长期存储。

 
