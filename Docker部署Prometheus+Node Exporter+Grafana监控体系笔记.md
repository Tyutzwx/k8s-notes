# Docker部署Prometheus+Node Exporter+Grafana监控体系笔记

## 一、环境说明

- 操作系统：CentOS 7

- 部署方式：Docker 容器化

- 组件版本：latest（官方最新版）

- 监控目标：本机 Linux 系统指标（CPU、内存、磁盘、网络等）

## 二、部署架构

Node Exporter（采集主机指标） → Prometheus（存储+查询指标） → Grafana（可视化监控面板）

## 三、完整部署步骤

### 1. 准备工作：创建挂载目录（用于配置文件、数据持久化）

执行以下命令，创建Prometheus的配置目录和数据目录，避免容器重启后数据丢失：

```bash
mkdir -p /data/prometheus_config
mkdir -p /data/prometheus_data
chmod 777 /data/prometheus_data  # 赋予权限，避免挂载时权限不足
```

### 2. 编写Prometheus配置文件

配置文件用于指定数据采集频率、监控目标（Node Exporter），执行命令编辑配置文件：

```bash
vim /data/prometheus_config/prometheus.yml
```

配置文件内容（替换IP为自己的虚拟机真实IP，此处为192.168.174.130）：

```yaml
global:
  scrape_interval: 15s  # 数据采集间隔，默认15秒

scrape_configs:
  - job_name: 'node_exporter'  # 监控任务名称，自定义即可
    static_configs:
      - targets: ['192.168.174.130:9100']  # Node Exporter的地址和端口
```

### 3. 启动Prometheus容器

执行以下命令，启动Prometheus，挂载配置文件和数据目录，设置开机自启：

```bash
docker run -d \
  --name prometheus \
  --restart always \
  -p 9090:9090 \
  -v /data/prometheus_config:/etc/prometheus \
  -v /data/prometheus_data:/prometheus \
  prom/prometheus
```

### 4. 启动Node Exporter（主机指标采集工具）

Node Exporter用于采集本机CPU、内存、磁盘等指标，必须使用host网络模式，执行命令：

```bash
docker run -d \
  --name node_exporter \
  --net=host \
  --pid=host \
  --restart always \
  -v /:/host:ro \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```

### 5. 启动Grafana（可视化面板）

Grafana用于将Prometheus的指标以图表形式展示，执行命令启动，设置开机自启：

```bash
docker run -d \
  --name grafana \
  --restart always \
  -p 3000:3000 \
  -v grafana_data:/var/lib/grafana \
  grafana/grafana
```

默认账号密码：admin / admin（首次登录需修改密码，可直接跳过或设置简单密码）

### 6. Grafana配置Prometheus数据源

1. 浏览器访问：http://虚拟机IP:3000，使用默认账号密码登录

2. 点击左侧菜单栏「Configuration」→「Data Sources」→「Add data source」

3. 选择「Prometheus」作为数据源类型

4. 在「URL」栏填写：http://虚拟机IP:9090（此处为http://192.168.174.130:9090）

5. 点击页面底部「Save & Test」，显示「Data source is working」即为配置成功

## 四、监控面板使用（最简单可用方案）

因新版Grafana与部分旧面板模板不兼容，推荐手动新建空白面板，直接编写PromQL查询语句，避免报错：

### 1. 新建面板

点击左侧「+」→「新建仪表盘」→「添加面板」，进入面板编辑页面

### 2. 常用PromQL查询语句（直接复制使用）

- CPU使用率：`100 - (avg(irate(node_cpu_seconds_total{mode="idle",instance="192.168.174.130:9100"}[1m])) * 100)`

- 内存使用率：`100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)`

- 磁盘使用率（根目录）：`100 - (node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|devtmpfs"} / node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|devtmpfs"} * 100)`

- 网络入流量：`irate(node_network_receive_bytes_total{instance="192.168.174.130:9100"}[1m])`

- 网络出流量：`irate(node_network_transmit_bytes_total{instance="192.168.174.130:9100"}[1m])`

### 3. 查看图表

粘贴查询语句后，点击页面右上角「Run queries」，即可显示对应指标的图表；将时间范围改为「过去5分钟」，刷新后数据更清晰。

## 五、部署过程中遇到的问题及解决方案（重点）

### 问题1：容器名称冲突（最常见）

**报错信息**：docker: Error response from daemon: Conflict. The container name "/node_exporter" is already in use by container xxx.

**原因**：之前启动过同名容器，容器未删除，无法重复使用相同名称。

**解决方案**：强制删除同名容器，再重新启动，命令如下：

```bash
docker rm -f 容器名  # 例如：docker rm -f node_exporter
```

### 问题2：Prometheus查询无数据（No data）

**报错信息**：在Prometheus页面（http://IP:9090）查询node_cpu_seconds_total，显示「No data」。

**原因**：3种常见情况，优先级从高到低：

1. Prometheus配置文件中targets的IP填写错误，未指向本机真实IP；

2. Node Exporter未正常启动，无法提供指标；

3. 刚启动服务，Prometheus未到采集间隔（默认15秒），未拉取到数据。

**解决方案**：

1. 修改配置文件：vim /data/prometheus_config/prometheus.yml，确保targets为本机真实IP:9100；

2. 重启Prometheus：docker restart prometheus；

3. 检查Node Exporter状态：docker ps | grep node_exporter，若未运行则重启：docker restart node_exporter；

4. 等待30秒，再重新查询。

### 问题3：Grafana导入面板报错（e.replace不是函数）

**报错信息**：导入面板ID（1860、8919）后，页面显示「TypeError: e.replace is not a function」。

**原因**：旧版面板模板的语法的与新版Grafana不兼容，新版Grafana废弃了e.replace语法。

**解决方案**：放弃导入旧面板，采用「手动新建空白面板+直接写PromQL查询」的方式（本文第四部分内容），彻底避免报错。

### 问题4：Grafana页面脚本崩溃

**报错信息**：页面显示「TypeError：无法读取未定义属性（读取“textContent”）」，无法正常操作。

**原因**：Grafana页面缓存异常或JS脚本崩溃，与操作无关。

**解决方案**：重启Grafana容器，刷新浏览器即可恢复，命令如下：

```bash
docker restart grafana
```

### 问题5：Node Exporter无法采集指标

**报错信息**：执行curl localhost:9100/metrics，无法输出指标文本。

**原因**：Node Exporter未使用host网络模式，或未正确挂载根目录。

**解决方案**：删除原有Node Exporter容器，重新执行正确的启动命令（本文第三部分第4步），确保包含--net=host和-v /:/host:ro参数。

## 六、常用验证命令（快速排查问题）

```bash
# 1. 查看Prometheus监控目标状态（UP为正常）
http://虚拟机IP:9090/targets

# 2. 验证Node Exporter是否正常输出指标
curl localhost:9100/metrics | head -20  # 输出前20行，确认有node_开头的指标

# 3. 查看所有相关容器状态
docker ps | grep -E "prometheus|node_exporter|grafana"

# 4. 重启所有监控相关服务（快速恢复）
docker restart prometheus node_exporter grafana

# 5. 查看Prometheus配置文件（确认IP正确）
cat /data/prometheus_config/prometheus.yml
```

## 七、最终可用状态验证

完成部署后，确认以下4点，即为部署成功：

1. Prometheus：http://IP:9090，查询node_cpu_seconds_total有数据；

2. Node Exporter：curl localhost:9100/metrics能正常输出指标；

3. Grafana：数据源测试正常，新建面板能显示CPU、内存等图表；

4. 所有容器：docker ps查看，prometheus、node_exporter、grafana均为Up状态。

## 八、补充说明

- 所有容器均设置--restart always，服务器重启后会自动启动，无需手动操作；

- Prometheus数据挂载在/data/prometheus_data，Grafana数据挂载在grafana_data卷，容器删除后数据不丢失；

- 若需监控多台机器，只需在其他机器启动Node Exporter，修改Prometheus配置文件的targets，添加新机器的IP:9100即可。

