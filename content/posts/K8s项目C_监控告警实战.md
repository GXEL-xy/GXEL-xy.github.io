---
title: "K8s 项目 C：监控告警实战（Prometheus + Grafana + Alertmanager）"
date: 2026-09-05T00:40:00+08:00
draft: false
description: "在三节点 K8s 集群 monitoring 命名空间部署 node-exporter + kube-state-metrics + Prometheus + Grafana + Alertmanager，实现全集群指标采集、Node Exporter Full 仪表盘与 QQ 邮箱真实邮件告警链路。"
tags: ["k8s监控告警"]
categories: [项目]
---

> 📦 **项目文件包**：[下载 K8s项目C_监控告警.zip](/files/K8s项目C_监控告警.zip)（含全部 YAML manifests、镜像预拉取与验证脚本、README，可直接拷到集群 master 使用）。本文件是详细操作手册，两者配合。

> 适用环境：3 节点集群（1 master + 2 worker，K8s v1.35 + containerd 2.x + Ubuntu 24.04）
> master IP：`11.0.1.128`　私有仓库：`11.0.1.128:30000`
> 前置：项目 B 已完成（Jenkins CI/CD 正常运行）
> 预计耗时：4-6 小时（含原理学习、验证与踩坑）

---

## 0. 项目概览

### 0.1 目标与产出

| 产出 | 说明 |
|------|------|
| 全集群指标采集 | 3 个节点的 CPU/内存/磁盘/网络 + 所有 K8s 对象状态 |
| Prometheus 时序数据库 | 15s 粒度采集，数据保留 15 天 |
| Grafana 可视化 | 数据源 + Node Exporter Full 仪表盘，中文可切换 |
| 邮件告警链路 | 告警规则 → Alertmanager → QQ 邮箱真实收信 |
| 面试完整故事线 | "我从 0 搭建了监控体系，配了 X 条告警规则，触发过真实告警" |

### 0.2 架构与数据流

```
┌─────────────── K8s 集群（monitoring namespace）────────────────┐
│                                                                │
│  [node-exporter DaemonSet]      [kube-state-metrics]           │
│  3 节点 × 1 Pod（宿主机指标）     1 Pod（K8s 对象状态指标）        │
│      │ 9100                          │ 8080                     │
│      └────────────┬───────────────────┘                          │
│                   │  抓取（pull，15s）                            │
│          [Prometheus Deployment] ── 服务发现调 K8s API ──► API  │
│           TSDB 存储（PVC 5Gi）                                  │
│            │            │                                      │
│     规则评估(15s)    数据查询(:9090)                              │
│            │            │                                      │
│     告警推给▼        界面拉取▼  数据源查询                        │
│  [Alertmanager]◄──────┐    [Grafana NodePort 30300]             │
│   去重/分组/路由        │                                         │
│        │              │                                         │
│     QQ 邮箱 📧    浏览器 :30300                                  │
└────────────────────────────────────────────────────────────────┘
```

**一句话记住分工**：
- **Exporter**：把各种东西的状态翻译成 HTTP 指标（`/metrics` 文本）
- **Prometheus**：主动拉（pull）指标存起来 + 按规则判断要不要报警
- **Grafana**：只管查询和画图，不存指标数据
- **Alertmanager**：只管告警的"通知策略"（去重/分组/静默/路由），不判断告警条件

**数据流四步走**（面试可以画这张图）：

| 步骤 | 谁负责 | 频率 | 失败表现 |
|------|--------|------|---------|
| ① 发现目标 | Prometheus 调 K8s API | 周期刷新（~30s） | Targets 页空白 |
| ② 抓取数据 | Prometheus → 各 /metrics | 每 15s | Target 显示 DOWN |
| ③ 存储与评估 | Prometheus TSDB + 规则引擎 | 每 15s 评估 | 告警规则不触发 |
| ④ 通知 | Prometheus → Alertmanager → 邮箱 | 触发时 | Rules firing 但没邮件 |

### 0.3 关键技术决策（面试可讲）

| 决策 | 选择 | 理由 |
|------|------|------|
| 部署方式 | **手写 YAML 裸部署**（本项目） | 教学直白、每个组件看得见摸得着、排错锻炼价值高；面试能讲清原理 |
| 部署方式（生产对照） | kube-prometheus-stack（Helm + Operator） | 企业主流，组件全自动配好；但 Helm 模板黑盒、排错依赖 Operator 知识，见附录 B |
| Prometheus 服务发现 | `kubernetes_sd_configs` | 不写死 IP，节点/ Pod 自动发现——这是"云原生监控"和"传统监控"的分水岭 |
| 存储 | local-path PVC + **nodeSelector 固定节点** | 吸收项目 B 教训：local-path 的 PV 绑定节点，Pod 漂移=数据"消失" |
| 镜像 | 全部提前中转进私有仓库 | 吸收项目 B 教训：Docker Hub/quay.io 国内拉不动，一次中转全程畅通 |
| 通知渠道 | QQ 邮箱 SMTP | 无公网回调条件（Poll SCM 同款理由），邮件最通用；钉钉/企微 webhook 生产可换 |

### 0.4 执行前必须替换的占位符

| 占位符 | 出现文件 | 替换成 | 怎么查 |
|--------|---------|--------|--------|
| `__NODE_NAME__` | 05-prometheus.yaml、06-grafana.yaml（共 2 处文件 4 行） | 固定节点名 | `kubectl get nodes`（建议 work1） |
| `__SMTP_AUTH_CODE__` | 07-alertmanager-config.yaml | QQ 邮箱 SMTP 授权码（16 位） | 见阶段 7.1 |

**替换方法**（在 master 上）：

```bash
cd K8s项目C_监控告警
# 先看节点名，假设是 work1
kubectl get nodes
# 逐个文件 sed 替换（比 vim 稳，注意用单引号包表达式）
sed -i 's/__NODE_NAME__/work1/g' manifests/05-prometheus.yaml
sed -i 's/__NODE_NAME__/work1/g' manifests/06-grafana.yaml
vim manifests/07-alertmanager-config.yaml   # 授权码涉及敏感信息，建议手动编辑而非 sed
# 替换后核对（应无任何 __NODE_NAME__ 残留）
grep -rn "__NODE_NAME__\|__SMTP_AUTH_CODE__" manifests/ || echo "✅ 占位符已全部替换"
```

### 0.5 端口规划（接续项目 B，避免冲突）

| 已占用（项目 B） | 新分配（项目 C） |
|------------------|------------------|
| 30000 registry<br>30080 Jenkins Web<br>31000 Jenkins jnlp<br>30090 damo-app | **30091** Prometheus Web<br>**30300** Grafana Web<br>**30093** Alertmanager Web |

> 端口规划是生产规范：NodePort 合法范围 30000-32767，新建服务前先 `kubectl get svc -A | grep -E "3[0-2][0-9]{3}"` 排查已占用端口，避免"服务起不来"的低级坑。

---

## 阶段 1：先懂再装——指标采集原理

### 1.0 开胃故事：一家"医院持续健康监测中心"（先建立直觉）

五个技术组件听起来吓人，其实它们合起来就是一家"医院给宿舍楼做 7×24 健康监测"需要的五个角色。**读不懂原理时，回来重读这一节。**

**① 智能手环（node-exporter）——每个员工手腕上一个。**
它实时测心率、血压、体温（对应 CPU、内存、磁盘）。注意：手环**只负责测和显示，从不主动打电话**——它静静挂在那里，等别人来读。而且它只测"身体指标"，不管别的事。

**② 宿管阿姨（kube-state-metrics）——整栋楼一个。**
手环测"身体"，但楼里的"管理状态"没人管：305 房间**应住 4 人实际住了 2 人**（Deployment 副本不足）、谁搬走了没登记（Pod 挂了）。宿管阿姨手里有全楼的登记册（K8s API 对象），谁问都能立刻报出来。

**③ 巡诊医生（Prometheus）——一个医生，每 15 秒巡一圈。**
他挨个**读**手环数据、问宿管登记情况，全部记进病历本（TSDB 存储）。关键是：医生不光记录，还带着判断标准——"心率超 100 **连续 1 分钟**才算异常"（告警规则的 `for: 1m`，防止跑完楼梯心率快被误报）。医生只负责判断"有问题"，**不负责打电话**。

**④ 护士站大屏（Grafana）——把病历本的数字画成曲线。**
全员健康趋势一眼看全。大屏自己**不测数据也不存数据**，病历本丢了历史它就画不出历史——它只负责"让数据变得一眼能懂"。

**⑤ 医生助理（Alertmanager）——专门管"怎么通知"。**
医生喊"305 老王异常！"，助理的流程：先**等 30 秒**看有没有别人同时出事（`group_wait`，攒一批发，避免连环轰炸）→ 老王已进 ICU，就**不再播报他的感冒**（抑制规则）→ 打电话给家属（SMTP 邮件）→ 家属**4 小时内不会重复接电话**（`repeat_interval`）。注意：助理**自己不懂医术**，只执行通知策略——判断病情是医生的事。

**完整剧情**（把五个角色串起来）：

> 老王的手环测到心率飙高（**手环采集**）→ 医生巡诊读到，连续 1 分钟超标，病历本标红"警告"（**Prometheus 判断**）→ 助理等 30 秒确认无其他人出事，打电话"老王心率异常！"（**Alertmanager 通知**）→ 老王服药恢复 → 医生确认指标正常 → 助理补一通电话："已恢复"（RESOLVED）→ 全程护士站大屏上老王的曲线先冲高再回落（**Grafana 展示**）。

口诀：**手环测、医生判、大屏看、助理叫**（宿管阿姨报管理账）。

**比喻 ↔ 真实组件对照表**：

| 医院 | 项目 C 真身 | 端口 | 一句话职责 |
|------|------------|------|-----------|
| 智能手环 ×N | node-exporter（DaemonSet） | 9100 | 只测不报，等被抓 |
| 宿管阿姨 | kube-state-metrics | 8080 | 把 K8s 对象登记状态变成指标 |
| 巡诊医生 | Prometheus | 9090 | pull 采集 + TSDB 存储 + 规则判断 |
| 护士站大屏 | Grafana | 30300 | 查询+画图，不存数据 |
| 医生助理 | Alertmanager | 9093 / 30093 | 分组、抑制、通知，不懂医术 |

**故事里的两个细节，就是两个面试考点**：

- **为什么手环"等别人来读"而不是主动上报？**——pull 模型：医生读不到手环（手环没电了），本身就说明"这个人出问题了"（`up=0`）。如果手环主动推送，它没电时**静默失联**，反而没人发现它挂了。这就是自测题 Q1 的答案。
- **为什么宿管阿姨只有一个不能多雇？**——两个阿姨同时报数，病历本里同一条记录收到两份，数据就乱了（重复指标冲突）。所以 kube-state-metrics 必须单实例，而手环每人一个（DaemonSet）天然不冲突。

### 1.1 Prometheus 数据模型与 TSDB 工作原理

**数据模型**：每条指标 = **指标名 + 一组标签 + 时间戳上的值**：

```
node_memory_MemAvailable_bytes{instance="11.0.1.128:9100", node="master"}  8.2e+09
└──────指标名──────────────┘└──────────────标签（键值对）─────────────┘└─值─┘
```

**核心概念：时间序列（Time Series）**。指标名 + 每对标签的取值组合，唯一确定一条时间序列。比如 3 个节点 × 1 个内存指标 = 3 条序列：

```
node_memory_MemAvailable_bytes{node="master"}  → 序列 1
node_memory_MemAvailable_bytes{node="work1"}   → 序列 2
node_memory_MemAvailable_bytes{node="work2"}   → 序列 3
```

标签是查询和聚合的抓手：`sum by (node)`、`{namespace="damo-app"}` 都靠标签过滤。**标签基数字（cardinality）是 Prometheus 容量的第一杀手**——不要把 Pod UID、时间戳这种高变化值放进标签（面试可讲：基数爆炸 → 内存暴涨 → Prometheus OOM 的经典故障链）。

**TSDB 存储原理**（面试加分点）：

| 组成 | 作用 |
|------|------|
| 内存中的 head block | 最近 ~2 小时数据先写内存 |
| WAL（Write-Ahead Log） | 数据落盘前的预写日志——**进程崩溃靠 WAL 恢复内存数据**，这就是为什么强行 kill Prometheus 数据不丢 |
| 磁盘分块（block） | 每 2 小时压缩成一个不可变 block 目录，后台定期合并压缩 |
| 保留策略 | `--storage.tsdb.retention.time=15d` 超期数据自动删除（我们 YAML 里配的参数） |

### 1.2 四种指标类型（面试必背，逐个吃透）

| 类型 | 特点 | 典型例子 | 配套 PromQL 函数 |
|------|------|---------|-----------------|
| **Counter** | 只增不减（进程重启才清零） | 请求数、错误数、重启次数 | `rate()`、`increase()`——**永远不要直接查原始值** |
| **Gauge** | 可增可减的瞬时值 | 内存用量、队列长度、在线人数 | 直接查，或 `avg_over_time()` 平滑 |
| **Histogram** | 客户端分桶计数（桶边界固定） | 请求耗时分布 | `histogram_quantile()` 算分位数 |
| **Summary** | 客户端预计算分位数 | 同上（流式计算） | 直接取 quantile 标签 |

**Counter 为什么不能直接看？** Counter 的值是"进程启动以来的累计数"（如 `http_requests_total = 4839201`），单独一个数字没有意义。必须用 `rate()` 求变化速率才有业务含义：

```promql
# ❌ 错误：直接看 Counter 原始值——只是一个大数字
http_requests_total

# ✅ 正确：每秒请求速率（自动处理 Counter 重启清零）
rate(http_requests_total[5m])

# ✅ 对低频事件（如重启次数）用 increase 看增量
increase(kube_pod_container_status_restarts_total[10m])
```

**Histogram vs Summary 怎么选**（面试追问）：Histogram 的分位数在**服务端**（Prometheus）聚合计算，多个实例可 `sum` 合并后算全局分位数；Summary 的分位数在**客户端**预计算，无法跨实例聚合。集群场景默认选 Histogram。

### 1.3 监控什么：四黄金指标（Google SRE）

| 指标 | 对应我们集群的例子 |
|------|-------------------|
| 延迟（Latency） | 请求耗时（本项目暂不采集应用层，原理知晓即可） |
| 流量（Traffic） | damo-app 的访问 QPS |
| 错误（Errors） | HTTP 5xx 比例 |
| 饱和度（Saturation） | **节点 CPU/内存/磁盘水位（本项目重点）** |

**配套两套方法论**（面试说得出名字和适用场景）：

| 方法论 | 关注什么 | 适用场景 | 我们项目的例子 |
|--------|---------|---------|---------------|
| **USE**（Utilization/Saturation/Errors） | **资源**：使用率、饱和度、错误 | 基础设施层 | node-exporter 的指标就是按 USE 组织的 |
| **RED**（Rate/Errors/Duration） | **服务请求**：速率、错误、耗时 | 在线服务层 | 监控 damo-app 的 HTTP 指标时用 |

> 一句话区分：**USE 看机器，RED 看服务请求**。我们的项目 C 主体是 USE（节点层），应用层 RED 留作扩展。

### 1.4 Exporter 模式与 /metrics 文本格式

**Exporter 模式**：Prometheus 不侵入被监控对象，而是通过统一约定——对象暴露一个 HTTP 端点 `/metrics`，返回纯文本格式的指标。生态里已有 200+ 官方/社区 Exporter（MySQL、Redis、nginx、blackbox 黑盒探测……），没有现成 Exporter 的系统可以自己写一个。

**/metrics 文本长什么样**（这是后面所有环节的数据源头，务必亲眼看一次）：

```
# HELP node_memory_MemTotal_bytes Memory information field MemTotal_bytes.
# TYPE node_memory_MemTotal_bytes gauge
node_memory_MemTotal_bytes 8.334258176e+09
```

- `# HELP`：指标说明（人看的）
- `# TYPE`：指标类型声明（gauge/counter/...）
- 正文：`指标名{标签} 值`——这就是被 Prometheus 抓走、存进 TSDB、被 PromQL 查询、被 Grafana 画图、被告警规则判断的**同一份数据**。整条链路从头到尾只有这一种数据格式。

**白盒 vs 黑盒监控**（面试概念题）：

| 模式 | 监控什么 | 我们的例子 |
|------|---------|-----------|
| 白盒 | 系统内部状态（进程暴露自己的指标） | node-exporter、kube-state-metrics |
| 黑盒 | 从外部视角探测"服务还能不能用" | blackbox_exporter 对 damo-app 做 HTTP 探测（进阶扩展） |

### 1.5 组件职责速记表

| 组件 | 监听端口 | 职责 | 类比 |
|------|---------|------|------|
| node-exporter | 9100 | 采集宿主机（节点层）指标 | 体温计 |
| kube-state-metrics | 8080 | 把 K8s API 对象状态转成指标 | 值班登记员 |
| Prometheus | 9090 | 拉取+存储+规则评估 | 中央数据仓库 + 值班判断 |
| Grafana | 3000 | 可视化 | 大屏 |
| Alertmanager | 9093 | 告警通知策略 | 电话接线员 |

### 1.6 阶段自测

- [ ] **能不看文档、用自己的话讲完"医院故事"，并把五个角色对号入座**（面试开场讲架构就用这套语言）
- [ ] 能说出 pull 模型和 push 模型的区别（Prometheus 为什么选 pull？）
- [ ] 能说出 node-exporter 和 kube-state-metrics 各监控什么层
- [ ] 知道 Counter 和 Gauge的区别；知道 Counter 必须配 rate()
- [ ] 能说出一条数据从 /metrics 文本到 Grafana 曲线的完整路径
- [ ] 知道 USE 和 RED 各适合什么场景

---

## 阶段 2：镜像中转（国内网络预案，5 分钟）

> 吸收项目 B 的教训：quay.io / registry.k8s.io / Docker Hub 在国内节点直连基本拉不动。所有镜像先在 master 中转进私有仓库，YAML 里直接引用 `11.0.1.128:30000/xxx`。

### 2.1 执行中转脚本

把项目 C 的文件包解压到 master（和项目 B 相同流程），执行：

```bash
cd K8s项目C_监控告警
bash scripts/pull-images.sh
```

**脚本工作原理**（逐行理解，不是黑盒跑完就完）：

| 脚本动作 | 命令本质 | 为什么 |
|---------|---------|--------|
| ① 拉原始镜像 | `docker pull quay.io/prometheus/node-exporter:v1.8.2` | master 的 docker 配了代理 + NO_PROXY，能出外网 |
| ② 改名打标 | `docker tag <原始> 11.0.1.128:30000/node-exporter:v1.8.2` | 私有仓库地址就是镜像名前缀 |
| ③ 推入私有仓库 | `docker push 11.0.1.128:30000/node-exporter:v1.8.2` | daemon.json 已配 insecure-registries，http 可推 |

涉及 5 个镜像：

| 私有仓库名 | 原始地址 | 用途 |
|-----------|---------|------|
| node-exporter:v1.8.2 | quay.io/prometheus/node-exporter | 节点指标 |
| kube-state-metrics:v2.13.0 | registry.k8s.io/kube-state-metrics/kube-state-metrics | K8s 对象指标 |
| prometheus:v2.53.1 | prom/prometheus（LTS） | 采集存储 |
| grafana:11.1.0 | grafana/grafana | 可视化 |
| alertmanager:v0.27.0 | prom/alertmanager | 告警通知 |

**预期输出**（每个镜像重复这三行，以 node-exporter 为例）：

```
v1.8.2: Pulling from prometheus/node-exporter
Digest: sha256:xxxxxxxx...
Status: Downloaded newer image for quay.io/prometheus/node-exporter:v1.8.2
The push refers to repository [11.0.1.128:30000/node-exporter]
v1.8.2: digest: sha256:xxxx size: 528
```

### 2.2 验证

```bash
# 方法①：看仓库目录（应列出 5+ 个镜像名，含项目 B 的）
curl -s http://11.0.1.128:30000/v2/_catalog
# 预期输出：{"repositories":["damo-app","grafana","kube-state-metrics","nginx","node-exporter",...]}

# 方法②：从 worker 节点试拉一个（模拟 K8s 调度到 worker 时的真实拉取路径）
sudo crictl pull 11.0.1.128:30000/grafana:11.1.0
# 预期输出：Image is up to date for sha256:xxxx（或正常下载完成）
```

**常见报错排查**：

| 报错 | 原因 | 解决 |
|------|------|------|
| `docker pull` 超时/tls handshake timeout | master docker 没走代理 | 检查 `/etc/docker/daemon.json` 代理 env 或 systemd drop-in，`systemctl restart docker` |
| `push ... http: server gave HTTP response to HTTPS client` | 私有仓库没进 docker 的 insecure-registries | daemon.json 加 `11.0.1.128:30000`（项目 B 已配，说明被改回去了） |
| `manifest unknown` | tag 在原始仓库不存在 | 上 Docker Hub/quay.io 查同仓库正确 tag，同步改 pull-images.sh 与所有 YAML |
| 磁盘不足 | docker 层积累 | `docker system df` 看占用；`docker image prune -a` 清理（**慎用**，会清掉中转镜像） |

---

## 阶段 3：node-exporter（节点指标，15 分钟）

### 3.0 组件工作原理（先懂再装）

**node-exporter 是怎么拿到宿主机指标的？** Linux 内核把硬件与内核状态暴露在 `/proc`（进程、CPU、内存）和 `/sys`（块设备、网卡）虚拟文件里。node-exporter 内部跑着一组 **Collector**（采集器），启动时按需启用：

| Collector | 读取位置 | 产出指标 |
|-----------|---------|---------|
| cpu | `/proc/stat` | `node_cpu_seconds_total`（Counter） |
| meminfo | `/proc/meminfo` | `node_memory_*`（Gauge） |
| filesystem | `/proc/mounts` + statfs | `node_filesystem_*` |
| netdev | `/proc/net/dev` | `node_network_*` |
| diskstats | `/proc/diskstats` | `node_disk_*` |
| loadavg | `/proc/loadavg` | `node_load1/5/15` |

**关键点：容器隔离问题**。如果 node-exporter 只在容器里跑而不做特殊处理，它读到的 `/proc` 是**容器自己的**（只看到自己这个容器），指标全是错的。所以必须 `--path.rootfs=/host` + hostPath 挂载宿主机根目录——让 Collector 去读**宿主机的真实 /proc**。这是 node-exporter 部署里最重要的一个配置。

### 3.1 部署

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/01-node-exporter.yaml
```

**预期输出**：

```
namespace/monitoring created
daemonset.apps/node-exporter created
service/node-exporter created
```

> 注意：`kubectl apply` 幂等（重复执行显示 `unchanged` 不报错），这是和 `create` 的区别——YAML 管理资源的标准姿势。

**对应文件 1：`manifests/00-namespace.yaml`**

```yaml
# ============================================
# 项目 C 阶段 1：创建监控命名空间
# 用途: 给整套监控组件一个独立的"房间"，与业务（jenkins/registry/damo-app）隔离
#       便于统一授权、资源配额和清理（删 namespace = 全部删除）
# 用法: kubectl apply -f 00-namespace.yaml
# ============================================
apiVersion: v1          # K8s API 组/版本：Namespace 属于核心组 v1
kind: Namespace         # 资源类型：命名空间
metadata:
  name: monitoring      # 命名空间名：后续所有监控组件都部署在这里
  labels:
    # 给 namespace 打标签，是"监控是否覆盖"的常用标记（面试可讲）
    monitoring: enabled
```

**对应文件 2：`manifests/01-node-exporter.yaml`**

```yaml
# ============================================
# 项目 C 阶段 2：node-exporter（采集节点层指标）
# 每个 Node 一个 Pod（DaemonSet），采集 CPU/内存/磁盘/网络
# 用法: kubectl apply -f 01-node-exporter.yaml
# 注意: Service 的 label 和端口名是 Prometheus 服务发现的过滤条件，改了会抓不到（见 04）
# ============================================
apiVersion: apps/v1     # apps 组 v1：DaemonSet/Deployment 等工作负载都在这个组
kind: DaemonSet         # 资源类型：守护进程集——每个节点恰好一个 Pod，新节点自动部署
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:        # DaemonSet 通过这对标签认领自己管的 Pod
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter   # 必须与 selector 一致，也是 Service 的筛选条件
    spec:
      # 使用宿主机网络：node-exporter 直接监听节点 9100 端口
      # 好处：endpoints 就是节点真实 IP，指标里的网络数据是宿主机视角
      hostNetwork: true
      # 需要访问宿主机进程命名空间（部分指标如 process 相关）
      hostPID: true
      # 三节点全覆盖：master 上也跑（默认 DaemonSet 不调度到有污点的 master）
      # operator: Exists = 容忍任何污点，没有这条 master 会漏监控
      tolerations:
        - operator: Exists
      containers:
        - name: node-exporter
          # 原始镜像: quay.io/prometheus/node-exporter:v1.8.2（已中转进私有仓库）
          image: 11.0.1.128:30000/node-exporter:v1.8.2
          args:
            # 把宿主机根目录挂进容器后，告诉 node-exporter 从哪读
            # 文件系统/磁盘等指标才能反映宿主机真实情况
            # （缺了这条：指标全是容器自己的，数据全错且不报错！）
            - --path.rootfs=/host
          ports:
            - containerPort: 9100
              hostPort: 9100   # 配合 hostNetwork，节点 9100 直接可达
          resources:
            requests:          # 调度保底资源
              cpu: 50m         # 50m = 0.05 核
              memory: 64Mi
            limits:            # 硬上限，防止拖累节点
              cpu: 200m
              memory: 128Mi
          volumeMounts:
            - name: rootfs
              mountPath: /host   # 宿主机根目录挂到容器的 /host（只读）
              readOnly: true
      volumes:
        - name: rootfs
          hostPath:
            path: /            # 直接挂宿主机根分区——读宿主机真实 /proc /sys
            type: Directory    # 校验路径类型，不存在则 Pod 起不来（防配置错误）
---
# Headless Service：没有 ClusterIP，DNS 直接解析出所有 Pod IP
# Prometheus 通过 endpoints 服务发现拿到 "节点IP:9100" 列表
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    # !!! 这个 label 是 Prometheus 自动发现的匹配条件，别改 !!!
    app: node-exporter
spec:
  clusterIP: None     # None = Headless：不分配虚拟 IP，endpoints 直接暴露真实地址
  selector:
    app: node-exporter
  ports:
    # !!! 端口名必须叫 metrics，Prometheus relabel 规则按这个名字过滤 !!!
    - name: metrics
      port: 9100
      targetPort: 9100
```

### 3.2 YAML 关键设计逐条讲（每条都是面试点）

| 设计 | 为什么 | 不配会怎样 |
|------|--------|-----------|
| `hostNetwork: true` | Pod 直接用宿主机网络栈，监听节点 9100；endpoints 就是节点真实 IP | 用 Pod IP 也行（Service 能通），但节点真实 IP 更直观，且不受 CNI 波动影响 |
| `hostPID: true` | 看得见宿主机进程，进程相关指标才准 | process 相关 Collector 数据失真（其余 CPU/内存指标不受影响） |
| `tolerations: [{operator: Exists}]` | 容忍一切污点 | master 有 NoSchedule 污点，Pod 不会调度上去 → **3 节点只采到 2 个**（经典坑） |
| DaemonSet | 每节点恰好一个 Pod，新增节点自动部署 | 用 Deployment 固定副本数的话，扩节点要手改副本数，还会同节点跑多个 |
| `args: [--path.rootfs=/host]` + hostPath `/` | Collector 读宿主机真实 /proc /sys | **所有指标都是容器的而非宿主机的**（数据全错但不报错，极难发现） |
| Service `clusterIP: None` | Headless Service，endpoints 直接是 3 个节点 IP:9100 | 普通 ClusterIP 也能抓，但目标显示的是 Service IP，看不出节点 |
| 端口名 `metrics` | 后面 Prometheus 服务发现 relabel 按端口名过滤 | 过滤条件对不上 → Targets 抓不到（详见 5.2） |

### 3.3 验证（三步，全部通过才算过）

```bash
# ① Pod 数量与分布：应看到 3 个 node-exporter，分别在不同节点（含 master），全部 Running
kubectl get pods -n monitoring -o wide
# 预期输出（NAME 后缀随机）：
# NAME                  READY   STATUS    NODE
# node-exporter-2wm9p   1/1     Running   master
# node-exporter-8dk2f   1/1     Running   work1
# node-exporter-tq4xv   1/1     Running   work2

# ② 端点正确性：endpoints 应该是 3 个节点 IP:9100（而不是 Pod CIDR 地址）
kubectl get endpoints node-exporter -n monitoring
# 预期输出：node-exporter   11.0.1.128:9100,11.0.1.x:9100,11.0.1.y:9100

# ③ 指标可读：在 master 本地 curl 自己的指标
curl -s http://11.0.1.128:9100/metrics | head -20
```

**验证 ③ 的预期输出**（应有 HELP/TYPE 注释行 + 大量 `node_*` 指标行）：

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
...
# HELP node_memory_MemTotal_bytes Memory information field MemTotal_bytes.
# TYPE node_memory_MemTotal_bytes gauge
node_memory_MemTotal_bytes 8.334258176e+09
```

**常见报错排查**：

| 现象 | 原因 | 排查思路 |
|------|------|---------|
| 只有 2 个 Pod，master 上没有 | 污点没容忍 | `kubectl describe node master \| grep -A 5 Taints`；确认 YAML tolerations |
| Pod CrashLoopBackOff | 镜像拉不下来 / 端口冲突 | `kubectl describe pod` 看 Events；9100 被占用时换 `--web.listen-address=:9101` |
| 指标全是容器自己的（内存只有几百 MB） | 忘了 `--path.rootfs=/host` 或 hostPath 没挂 | 对比 `curl :9100/metrics` 的 MemTotal 与 `free -g` 实际值，相差一个量级即中招 |
| endpoints 为空 | Service selector 和 Pod label 不匹配 | `kubectl get endpoints node-exporter -n monitoring`；检查两边 `app` 标签是否一致 |

---

## 阶段 4：kube-state-metrics（K8s 对象指标，15 分钟）

### 4.0 组件工作原理（先懂再装）

**kube-state-metrics（KSM）的数据流**：

```
K8s API Server ──list/watch（全量+增量）──► KSM 内存对象缓存 ──每次被 scrape 时──► 生成全部指标文本
```

三个关键理解（面试高频）：

1. **它不存储历史**：每次 Prometheus 来抓时，KSM 现场把内存里当前所有对象状态**重新生成一遍指标文本**。历史数据只存在于 Prometheus 侧。所以 KSM 自己几乎不用配存储。
2. **它不主动推送**：API 有变化时 KSM 只更新内存缓存，指标永远是"被抓取时"的快照。
3. **为什么必须单实例**：多副本会导致每个 Prometheus 目标抓到**重复指标**（同一条序列多个实例各报一份），引发 TSDB 冲突错误。这是它和 node-exporter（DaemonSet 天然按节点区分）部署形态差异的根本原因。

**它产出什么指标**（记住前缀 `kube_*`）：

| 指标前缀 | 内容 | 用途示例 |
|---------|------|---------|
| `kube_pod_*` | Pod 状态、重启次数、容器状态 | 告警：Pod CrashLooping |
| `kube_deployment_*` | 期望/当前/可用副本数 | 告警：副本不足 |
| `kube_node_*` | 节点 Ready/内存可分配 | 告警：节点 NotReady |
| `kube_job_*` / `kube_cronjob_*` | Job 状态 | 监控 CI 任务 |

### 4.1 部署

```bash
kubectl apply -f manifests/02-kube-state-metrics.yaml
# 预期输出（6 个资源）：
# serviceaccount/kube-state-metrics created
# clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
# clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
# deployment.apps/kube-state-metrics created
# service/kube-state-metrics created
```

**对应文件：`manifests/02-kube-state-metrics.yaml`**

```yaml
# ============================================
# 项目 C 阶段 4：kube-state-metrics（采集 K8s 对象状态指标）
# 把 Deployment/Pod/Node 等 API 对象的状态转成 Prometheus 指标
# 例如：Pod 重启次数、Deployment 副本数是否达标、Node 是否 Ready
# 用法: kubectl apply -f 02-kube-state-metrics.yaml
# 注意: 必须单实例！多副本会对同一对象重复产出指标（数据冲突）
# ============================================
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: monitoring
---
# kube-state-metrics 需要 list/watch K8s 对象的权限，所以必须配 RBAC
# （它监控的是"集群对象状态"，不配权限就什么都看不到）
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole        # ClusterRole = 全集群范围授权（KSM 要看所有 namespace 的对象）
metadata:
  name: kube-state-metrics
rules:
  - apiGroups: [""]      # "" = 核心组（Pod/Node/Service 这些"原生"资源）
    resources:
      - pods             # Pod 状态/重启次数
      - nodes            # 节点 Ready/容量
      - services
      - endpoints
      - namespaces
      - secrets          # 只产出"是否存在"类指标，不读内容
      - configmaps
    verbs: ["get", "list", "watch"]   # watch 是增量同步的关键，没有它只能反复全量拉取
  - apiGroups: ["apps"]  # 工作负载组
    resources:
      - deployments      # 副本数指标（可用 vs 期望）
      - daemonsets
      - replicasets
      - statefulsets
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources:
      - jobs             # Job/CronJob 状态（监控 CI 任务用得上）
      - cronjobs
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding   # 把上面的角色绑定给 KSM 的 ServiceAccount
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
  - kind: ServiceAccount
    name: kube-state-metrics
    namespace: monitoring
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    app: kube-state-metrics
spec:
  replicas: 1            # !!! 单实例是设计约束，不是偷懒（多副本=重复指标）
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics   # Pod 挂上这个 SA 才有上面配的权限
      containers:
        - name: kube-state-metrics
          # 原始镜像: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.13.0（已中转）
          image: 11.0.1.128:30000/kube-state-metrics:v2.13.0
          ports:
            - containerPort: 8080
              name: metrics    # 端口名 metrics：服务发现过滤约定
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m        # 对象多时 KSM 内存线性增长，上限防拖垮节点
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    app: kube-state-metrics
spec:
  selector:
    app: kube-state-metrics
  ports:
    - name: metrics      # Prometheus 用静态配置抓这个 Service（kube-state-metrics:8080）
      port: 8080
      targetPort: 8080
```

**YAML 关键设计**：

| 设计 | 为什么 |
|------|--------|
| ServiceAccount + ClusterRole（get/list/watch pods,nodes,deployments...） | KSM 要读全集群 API 对象，权限不足会启动报错或指标缺失 |
| Deployment 单副本 | 见 4.0 第 3 点，多副本=重复指标 |
| `resources.limits`（256Mi） | 对象多时 KSM 内存线性增长，设上限防拖垮节点 |
| Service 端口名 `metrics` | 和 node-exporter 同理，服务发现过滤用 |

### 4.2 验证

```bash
# ① Pod Running
kubectl get pods -n monitoring -l app=kube-state-metrics
# 预期：NAME READY STATUS；1/1 Running

# ② 指标可用：port-forward 到 master 本机再 curl（⚠️ 不要 exec 进 KSM 容器——
#    官方镜像是 distroless 静态镜像，没有 shell、没有 wget/curl，exec 会报
#    "exec: \"wget\": executable file not found"，且错误走 stderr，容易被 2>/dev/null 吞掉变成"静默无输出"）
kubectl port-forward -n monitoring deploy/kube-state-metrics 18080:8080
#    （前台挂着不动，另开一个终端执行下面这条；验证完 Ctrl+C 断开）
curl -s http://localhost:18080/metrics | grep kube_pod_status_phase | head -5
# 预期输出：
# kube_pod_status_phase{namespace="kube-system",phase="Running"} 9
# kube_pod_status_phase{namespace="monitoring",phase="Running"} 1
# ...

# ②' 备选：用工具箱 Pod 走 Service DNS 测（alpine/k8s 里有 busybox wget）
kubectl run ksm-verify --rm -it --restart=Never --image=11.0.1.128:30000/alpine/k8s:1.35.0 \
  -n monitoring -- wget -qO- http://kube-state-metrics:8080/metrics | grep kube_pod_status_phase | head -5
# 预期同上；退出即自动删 Pod（--rm）
# 💡 考点：distroless 最小镜像是安全最佳实践（攻击面小、无 shell 可利用），
#    代价是无法 exec 调试——所以运维要习惯 port-forward / 临时工具箱 Pod 两种替代姿势

# ③ RBAC 生效验证：故意删掉权限再测会 403——正式环境不必做，知道报错样子即可
#    403 时指标端点返回：Forbidden: User "system:serviceaccount:monitoring:kube-state-metrics" cannot list resource...
```

**常见报错排查**：

| 现象 | 原因 | 解决 |
|------|------|------|
| `kubectl exec ... wget` 无任何输出（Pod 明明 Running） | KSM 官方镜像是 distroless，没有 wget/shell，exec 报 `executable file not found` 且错误走 stderr 被 `2>/dev/null` 吞掉 | 别 exec 容器，改用 port-forward + curl 或工具箱 Pod（见 4.2 ②）；排查此类"静默失败"第一步：去掉 `2>/dev/null` 看真实报错 |
| Pod 起来但日志狂刷 Forbidden | ClusterRole 资源列表不全（如漏了 `replicasets`） | 对照 02 YAML 补齐 resources；`kubectl auth can-i list pods --as=system:serviceaccount:monitoring:kube-state-metrics -n monitoring` 验证 |
| 指标里缺某类对象（如没有 kube_deployment_*） | 同上，RBAC 资源列表缺项 | 同上 |
| 内存持续增长 | 集群对象过多（本集群 3 节点不会遇到） | 生产按对象规模加 limit；了解即可 |

---

## 阶段 5：Prometheus（核心，60 分钟）

### 5.0 组件工作原理（先懂再装）

Prometheus 内部有**四个循环**，理解它们就理解了 Prometheus：

| 循环 | 周期 | 干什么 | 失败表现 |
|------|------|--------|---------|
| ① 服务发现循环 | ~30s 刷新 | 调 K8s API 拿 endpoints/pods 列表，经 relabel 生成抓取目标集 | Targets 空白/过期 |
| ② 抓取循环 | 每 `scrape_interval`（15s） | 对每个目标 GET /metrics，解析文本 | Target DOWN（up=0） |
| ③ 存储循环 | 持续 | 样本写入内存 head block + WAL，后台压缩成磁盘 block | 磁盘写满/OOM |
| ④ 规则评估循环 | 每 `evaluation_interval`（15s） | 执行告警规则 expr，状态机流转 | 告警不触发 |

**告警状态机**（规则评估的产物，面试必画）：

```
expr 不成立 ──► inactive（正常）
expr 成立且未满 for ──► pending（观察期）
expr 持续成立满 for ──► firing（推给 Alertmanager）
firing 后 expr 不再成立 ──► resolved（发恢复通知）
```

**为什么只有 Prometheus 会 DOWN/UP**：`up` 是 Prometheus 对每个抓取目标自动生成的合成指标（抓取成功=1，失败=0），不是 Exporter 提供的——这也是"pull 模型天然能发现目标死亡"的体现（对比自测题 Q1）。

### 5.1 RBAC：为什么 Prometheus 也要权限

```bash
kubectl apply -f manifests/03-prometheus-rbac.yaml
# 预期输出：
# serviceaccount/prometheus created
# clusterrole.rbac.authorization.k8s.io/prometheus created
# clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```

**对应文件：`manifests/03-prometheus-rbac.yaml`**

```yaml
# ============================================
# 项目 C 阶段 4：Prometheus 的 RBAC
# Prometheus 要调用 K8s API 做服务发现（自动找到要抓取的目标），
# 所以需要 list/watch 节点、Pod、Service、Endpoint 的权限
# 用法: kubectl apply -f 03-prometheus-rbac.yaml
# 注意: 权限不足的现象是 Targets 页面空白（不是 DOWN），容易被误判
# ============================================
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  # 服务发现需要：读取节点/端点/服务信息
  # - endpoints 是 role: endpoints 发现方式的直接数据来源（最核心）
  # - services 提供目标的元数据标签（Service 名称/标签）
  # - nodes/pods 供 role: node / role: pod 发现方式使用
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - nodes/metrics
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  # 非 RBAC 环境下访问 kubelet 指标需要
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources:
      - ingresses
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding   # 把角色绑定到 prometheus SA，Prometheus Pod 必须挂这个 SA
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
```

Prometheus 的 `kubernetes_sd_configs` 服务发现是**调 K8s API 查询** nodes/pods/services/endpoints 实现的——没权限就查不到目标，Targets 页面会是空的。这是"监控组件也要 RBAC"的活教材。

**权限设计对照表**：

| ClusterRole 里的资源 | 服务发现用它干什么 |
|---------------------|-------------------|
| `endpoints`（最核心） | role: endpoints 的发现来源——从 Endpoint 对象拿到目标 IP:Port |
| `services` | 元数据标签（Service 名称/标签） |
| `pods` | role: pod 发现时用 |
| `nodes` | 节点元数据 |

### 5.2 抓取配置精讲（04-prometheus-config.yaml，本项目的精华）

**prometheus.yml 顶层结构**（全局配置项含义）：

```yaml
global:                    # 全局默认值，各 job 可覆盖
  scrape_interval: 15s     # 抓取周期——监控分辨率的基础，越短越灵敏、成本越高；
                           # 生产常用 15s~30s，再短需要评估目标端与 Prometheus 双侧压力
  evaluation_interval: 15s # 规则评估周期，与抓取对齐，告警延迟上限 = 抓取周期 + for + 评估周期

rule_files:                # 告警/记录规则文件列表（支持通配）
  - /etc/prometheus/rules/*.yml

alerting:                  # Alertmanager 对接（告警推给谁）
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]   # 同 namespace Service 短名 DNS

scrape_configs:            # ★ 核心：抓取目标配置（一个 job = 一组同类目标）
  ...
```

**三段抓取配置，三种发现方式，覆盖了 Prometheus 发现目标的两大流派：**

| job | 发现方式 | 学到什么 | 适用场景 |
|-----|---------|---------|---------|
| prometheus | `static_configs`（写死 localhost） | 最朴素的静态目标 | 目标固定且少（<10 个） |
| node-exporter | `kubernetes_sd_configs`（role: endpoints） | **云原生服务发现**：不写死 IP，靠 relabel 过滤 | 目标动态变化（K8s 内标配） |
| kube-state-metrics | `static_configs`（写死 Service 短名） | 集群内 DNS 直连单实例 | 集群内固定单实例服务 |

**kubernetes_sd_configs 的 role 选择**（面试可讲，各 role 发现的对象不同）：

| role | 发现到什么 | 对应元数据标签 | 适合监控 |
|------|-----------|---------------|---------|
| `endpoints` | Service 背后的 Endpoint（IP:Port） | `__meta_kubernetes_endpoint_node_name` 等 | 有 Service 的 Exporter（我们的 node-exporter） |
| `node` | 每个 Node 一个目标 | `__meta_kubernetes_node_name` | 直连 kubelet 指标 |
| `pod` | 每个 Pod 的每个容器端口 | `__meta_kubernetes_pod_name` 等 | 没有 Service 的裸 Pod Exporter |
| `service` | Service（ClusterIP/DNS） | `__meta_kubernetes_service_name` | 抓 Service 虚 IP（少用） |

**relabel_configs 三步曲逐行拆解**（node-exporter 那段，面试可以白板写）：

```yaml
# ① keep：只留 "Service 带 app=node-exporter 标签" 的目标，其余全丢
- source_labels: [__meta_kubernetes_service_label_app]
  action: keep
  regex: node-exporter
# ② keep：只留 "端口名叫 metrics" 的目标（一个 Pod 可暴露多个端口）
- source_labels: [__meta_kubernetes_endpoint_port_name]
  action: keep
  regex: metrics
# ③ 标签加工：把"目标所在节点名"写成新标签 node
#    （没有这步，告警消息里就不知道是哪台机器）
- source_labels: [__meta_kubernetes_endpoint_node_name]
  target_label: node
```

**relabel 的 action 全集**（面试追问"keep 之外还有什么"）：

| action | 语义 | 典型用途 |
|--------|------|---------|
| `keep` | 匹配 regex 的留下，其余丢弃 | 只要 node-exporter 的目标 |
| `drop` | 匹配 regex 的丢弃 | 排除控制面节点 |
| `replace` | 用 source_labels 拼接+regex 提取，写入 target_label | 固化节点名/区域标签（默认 action） |
| `labelmap` | 对所有标签名做 regex 匹配复制 | 把 K8s 标签批量转成 Prometheus 标签 |
| `labeldrop` | 删除匹配的标签 | 精简基数 |

> 心法：`__meta_kubernetes_*` 是服务发现过程携带的"元数据临时标签"，relabel 就是从里面**筛选（keep/drop）**和**加工（replace）**出真正的抓取目标。理解这句话，Prometheus 服务发现就通了。

**告警规则速览**（5 条规则的逻辑都是同一个模板）：

```yaml
- alert: 规则名
  expr: PromQL 表达式          # 什么条件
  for: 持续时长                # 持续多久才算真告警（防抖动）
  labels: { severity: 级别 }   # Alertmanager 路由用
  annotations: { summary: "..." }  # 通知里展示的文案（支持 {{ $labels.xxx }} 模板）
```

### 5.3 部署本体

**先改占位符**：编辑 `manifests/05-prometheus.yaml`，把 2 处 `__NODE_NAME__` 改成实际节点名（建议 work1）：

```bash
kubectl get nodes   # 看节点名
vim manifests/05-prometheus.yaml
```

**为什么要固定节点（重要，面试点）**：local-path 存储类的 PV 本质是"某节点本地目录"，PV 跟着第一次消费它的节点走。Prometheus Pod 重启后如果漂到另一节点，旧 PV 挂不上 → 历史"丢失"。加 `nodeSelector` 把 Pod 钉死在同一节点，这是 **local-path 在单副本有状态应用上的标准处理**（项目 B 的 Jenkins 同理）。

**Deployment 关键参数逐个讲**（05-prometheus.yaml 的 args 部分）：

| 参数 | 含义 | 我们为什么这么配 |
|------|------|-----------------|
| `--storage.tsdb.path=/prometheus` | 数据目录（挂 PVC） | 数据要活过 Pod 重启 |
| `--storage.tsdb.retention.time=15d` | 数据保留期 | 教学/演示 15 天足够；生产按磁盘预算算：每秒样本数 × 15s × 天数 |
| `--web.enable-lifecycle` | 开启 `/-/reload` 和 `/-/quit` HTTP 接口 | **配置热加载不重启**（阶段 8.4 用到）——改 ConfigMap 后 POST 一下就生效 |
| `--config.file=/etc/prometheus/prometheus.yml` | 主配置路径 | 挂 ConfigMap 的位置，与 04 对应 |

```bash
kubectl apply -f manifests/04-prometheus-config.yaml
kubectl apply -f manifests/05-prometheus.yaml
kubectl rollout status deployment/prometheus -n monitoring
# 预期输出：Waiting for deployment "prometheus" rollout to finish... / successfully rolled out
```

**对应文件 1：`manifests/04-prometheus-config.yaml`（整个项目的精华）**

```yaml
# ============================================
# 项目 C 阶段 4：Prometheus 核心配置（抓取规则 + 告警规则）
# 用法: kubectl apply -f 04-prometheus-config.yaml
#
# 这是整个项目最核心的文件，包含两部分：
#   prometheus.yml —— 抓取哪些目标、怎么发现目标
#   rules.yml      —— 什么条件触发告警
# 注意: 是 ConfigMap 挂载进容器的；修改后要 POST /-/reload 或重启才生效
# ============================================
apiVersion: v1
kind: ConfigMap          # 配置与镜像解耦：配置进 ConfigMap，Pod 挂载引用
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  # ---- 第 1 部分：Prometheus 主配置 ----
  prometheus.yml: |
    global:
      # 全局抓取间隔：每 15 秒抓一次目标
      # （监控分辨率的基础：越短越灵敏，但目标和 Prometheus 双侧压力越大）
      scrape_interval: 15s
      # 告警规则评估间隔：每 15 秒算一次规则表达式
      evaluation_interval: 15s

    # 告警规则文件（容器内路径，Deployment 里挂载）
    rule_files:
      - /etc/prometheus/rules/*.yml

    # 告警发往 Alertmanager
    alerting:
      alertmanagers:
        - static_configs:
            # 同 namespace 下 Service 短名可解析
            # （全名 alertmanager.monitoring.svc.cluster.local:9093 也一样）
            - targets: ["alertmanager:9093"]

    scrape_configs:
      # ---- 1. Prometheus 自监控 ----
      # 监控监控系统自己：TSDB/抓取等指标都会出现在查询里
      - job_name: prometheus
        static_configs:
          - targets: ["localhost:9090"]

      # ---- 2. node-exporter：Kubernetes 服务发现（本项目精华）----
      # 不写死任何 IP！Prometheus 调 K8s API 自动发现所有节点上的
      # node-exporter，节点扩容/缩容时目标列表自动增减
      - job_name: node-exporter
        kubernetes_sd_configs:
          # 发现所有 Service 的 endpoints（即后端 Pod IP+端口）
          # role 可选: endpoints / node / pod / service（详见文档 5.2 对比表）
          - role: endpoints
        relabel_configs:
          # 只保留带 app=node-exporter 标签的 Service 的目标（其余丢弃）
          # ——source_labels 取元数据标签，regex 匹配才 keep
          - source_labels: [__meta_kubernetes_service_label_app]
            action: keep
            regex: node-exporter
          # 只保留端口名为 metrics 的目标（一个 Pod 可暴露多个端口）
          - source_labels: [__meta_kubernetes_endpoint_port_name]
            action: keep
            regex: metrics
          # 把"目标所在的节点名"写入指标标签 node（Grafana/告警规则用）
          # ——没有这步，告警消息里就不知道是哪台机器
          - source_labels: [__meta_kubernetes_endpoint_node_name]
            target_label: node

      # ---- 3. kube-state-metrics：单实例，静态配置即可 ----
      # 集群内 Service DNS 短名直连（同 namespace 可省略后缀）
      - job_name: kube-state-metrics
        static_configs:
          - targets: ["kube-state-metrics:8080"]

  # ---- 第 2 部分：告警规则（5 条，模板统一）----
  rules.yml: |
    groups:
      # ---- 节点层告警 ----
      - name: node-alerts
        rules:
          # 节点失联：抓取目标持续 1 分钟不可达
          # for: 1m 防抖——偶发一次抓取失败（网络抖动）不报警
          - alert: NodeDown
            expr: up{job="node-exporter"} == 0
            for: 1m
            labels:
              severity: critical        # 级别标签：Alertmanager 路由/抑制用
            annotations:
              summary: "节点 {{ $labels.node }} 不可达"
              description: "Prometheus 已 1 分钟抓不到该节点的 node-exporter"

          # 节点内存使用率超过 85% 持续 2 分钟
          # Gauge 直接做四则运算：1 - 可用/总量 = 使用率
          - alert: NodeMemoryHigh
            expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.85
            for: 2m
            labels:
              severity: warning
            annotations:
              summary: "节点 {{ $labels.node }} 内存使用率超过 85%"
              description: "当前使用率 {{ $value | humanizePercentage }}"

          # 节点根分区磁盘使用率超过 85%
          # mountpoint="/" 只看根分区，避免被临时挂载干扰
          - alert: NodeDiskHigh
            expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) > 0.85
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "节点 {{ $labels.node }} 根分区磁盘使用率超过 85%"

      # ---- 工作负载层告警 ----
      - name: pod-alerts
        rules:
          # Pod 10 分钟内重启超过 3 次（CrashLoop 的典型特征）
          # Counter 配 increase：10 分钟窗口内的增量，不是累计值
          - alert: PodCrashLooping
            expr: increase(kube_pod_container_status_restarts_total[10m]) > 3
            labels:
              severity: warning
            annotations:
              summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} 10 分钟内重启超过 3 次"

          # Deployment 可用副本数低于期望副本数
          # 滚动更新期间会短暂不匹配，for: 5m 避免误报
          - alert: DeploymentReplicasShort
            expr: kube_deployment_status_replicas_available < kube_deployment_spec_replicas
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} 副本不足"
              description: "可用 {{ with $labels }}{{ end }}{{ $value }} 个，持续 5 分钟"
```

**对应文件 2：`manifests/05-prometheus.yaml`**

```yaml
# ============================================
# 项目 C 阶段 4：Prometheus 本体（存储 + Web UI + 抓取引擎）
# 用法: 先把下面两个 __NODE_NAME__ 改成你的固定节点名，再 apply
#   查节点名: kubectl get nodes -o wide
#   建议: 固定到 work1（Prometheus 有本地数据，local-path PV 绑定节点，
#         Pod 重启后必须还在同一节点，否则数据"丢失"）
# ============================================
apiVersion: v1
kind: PersistentVolumeClaim   # 时序数据必须持久化，否则 Pod 重启 = 监控历史清零
metadata:
  name: prometheus-data
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce           # 单 Pod 读写（Prometheus 单实例够用）
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi            # 按保留期估算：15s 粒度 × 15 天，教学环境 5Gi 够用
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  strategy:
    # 更新时先停旧的再起新的（local-path 单节点数据，不能双 Pod 同时写）
    # 默认 RollingUpdate 会新旧并存，有状态单写应用必须 Recreate
    type: Recreate
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus   # 挂 03 配的 SA，服务发现才有 API 权限
      # !!! 把 worker1 改成你的实际节点名（出现两处都要改）!!!
      nodeSelector:
        kubernetes.io/hostname: __NODE_NAME__
      containers:
        - name: prometheus
          # 原始镜像: prom/prometheus:v2.53.1（LTS，已中转）
          image: 11.0.1.128:30000/prometheus:v2.53.1
          args:
            # 配置文件绝对路径（⚠️ 必须显式传：默认值是相对路径 prometheus.yml，
            # 会按工作目录 /prometheus 解析成 /prometheus/prometheus.yml，而 PVC 数据盘里没有配置 → 起不来报
            # "Error loading config ... open prometheus.yml: no such file or directory"）
            - --config.file=/etc/prometheus/prometheus.yml
            # 配置热加载端口（改 ConfigMap 后 curl 此端口免重启生效）
            # curl -X POST http://<prometheus>:9090/-/reload
            - --web.enable-lifecycle
            # 数据保留 15 天（教学环境够用，生产按磁盘预算调）
            # 超期数据 TSDB 自动压缩删除，磁盘不会无限涨
            - --storage.tsdb.retention.time=15d
          ports:
            - containerPort: 9090
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"          # 注意 YAML 里纯数字 CPU 要加引号
              memory: 1Gi       # Prometheus 吃内存大户，给足但设上限
          volumeMounts:
            - name: data
              mountPath: /prometheus        # TSDB 数据目录（PVC）
            - name: config
              mountPath: /etc/prometheus   # 整目录挂载 ConfigMap，两个键都落在这：
                                           # /etc/prometheus/prometheus.yml + /etc/prometheus/rules.yml
              # ⚠️ 教训（本项目踩过的真坑）：不要"subPath 单文件挂 + 整目录挂到 /etc/prometheus/rules"混用——
              # 整目录挂载会把 ConfigMap 的【所有键】都变成目录里的文件，主配置 prometheus.yml
              # 会混进规则目录，被 rule_files 通配符当规则文件解析（顶层是 global 不是 groups）→ 启动即炸；
              # 另外 subPath 挂的文件在 ConfigMap 更新后不同步，会把 8.4 的热加载流程整个废掉
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: prometheus-data
        - name: config
          configMap:
            name: prometheus-config   # 一个 ConfigMap 同时供 prometheus.yml 和 rules 目录
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
    - name: web
      port: 9090
      targetPort: 9090
      # 注意：30090 已被 damo-app 占用，监控体系避开
      nodePort: 30091
```

**部署后先看日志再开浏览器**（排障好习惯——K8s 组件启动日志即"自检报告"）：

```bash
kubectl logs -n monitoring deploy/prometheus --tail=30
# 预期应看到：
# ts=... caller=main.go:xxx level=info msg="Server is ready to receive web requests."
# 若看到 config load error / parse error → 04 的 YAML 缩进或语法问题（见 9.1）
```

### 5.4 验证（本阶段最重要的一步）

浏览器打开 **http://11.0.1.128:30091** → 顶部菜单 **Status → Targets**：

| Target | 期望状态 |
|--------|---------|
| job=node-exporter（3 条） | UP（绿）——每节点一条，LABELS 里能看到 node=xxx |
| job=kube-state-metrics（1 条） | UP |
| job=prometheus（1 条） | UP |

**Targets 全绿是项目 C 最大的里程碑**，等价于项目 B 的 "Finished: SUCCESS"。

**API 层验证**（不依赖浏览器，脚本可自动化）：

```bash
# 查 up 指标，预期 5 条记录 value 全为 1
curl -s 'http://11.0.1.128:30091/api/v1/query?query=up' | grep -o '"value":\[[0-9]*,"1"\]'
# 更直观的（装了 jq 的话）：
curl -s 'http://11.0.1.128:30091/api/v1/query?query=up' | grep metric
# 预期看到每条记录带 job/node 标签，如 "job":"node-exporter","node":"work1"
```

再点 **Status → Rules**：应看到 5 条告警规则（state=inactive 是正常的，表示"没触发"）。

**常见报错排查**：

| 现象 | 原因 | 排查思路 |
|------|------|---------|
| Pod CrashLoop，日志报 `error loading config ... no such file or directory`，file 解析到 `/prometheus/prometheus.yml` | **args 漏传 `--config.file`**：默认值是相对路径，按工作目录 `/prometheus`（数据盘）解析，那里没有配置 | 在 05 的 args 加 `- --config.file=/etc/prometheus/prometheus.yml` 再 apply |
| Pod CrashLoop，报 `loading groups failed ... /etc/prometheus/rules/prometheus.yml: field global not found in type rulefmt.RuleGroups` | **规则目录里混进了主配置**：把整个 ConfigMap 目录挂到 rules 下，prometheus.yml 键也被当成规则文件被 glob 吃掉 | 05 改成整目录挂 `/etc/prometheus`（去掉 subPath），04 的 rule_files 改 `/etc/prometheus/rules.yml` |
| Pod CrashLoop，日志报 config load error 且带 YAML 行号 | 04 的 ConfigMap 缩进/语法错 | `kubectl logs deploy/prometheus` 看报错行号；对照 04 修缩进 |
| Targets 页空白（不是 DOWN，是 0 条） | RBAC 不足或 relabel 全部过滤掉了 | 检查 03 是否 apply；relabel 的 regex 与 Service 标签是否一致 |
| 个别 Target DOWN，Error 显示 `connection refused` | 目标端 Pod 没起/端口错 | `kubectl get endpoints -n monitoring` 对照目标 IP:Port 是否可达 |
| 个别 Target DOWN，Error 显示 `context deadline exceeded` | 目标响应慢/网络抖动 | 多为瞬时，观察是否恢复；持续则查目标侧 |
| UI 打得开但查询报 `storage error` | TSDB 损坏（罕见，多为强杀+磁盘问题） | 删 PVC 重建（教学环境可接受），生产要先备份 |

### 5.5 PromQL 入门（在 Prometheus 首页输入框练习）

**先分清两种查询对象**：

| 类型 | 形态 | 例子 | 用途 |
|------|------|------|------|
| 即时向量（instant vector） | 每条序列一个当前值 | `up` | 看当前状态、告警规则常用 |
| 范围向量（range vector） | 每条序列一段时间内的 N 个值 | `up[5m]` | 必须配合 rate/increase 等函数用 |

**核心函数三兄弟**（面试高频）：

| 函数 | 语义 | 典型用法 |
|------|------|---------|
| `rate(计数器[5m])` | 区间内**每秒**平均增长率（指数平滑，处理了 Counter 重置） | CPU、QPS 等持续型指标 |
| `increase(计数器[10m])` | 区间内**增长总量**（= rate × 区间秒数） | 重启次数这类低频事件 |
| `histogram_quantile(0.95, ...)` | 从 Histogram 桶算 95 分位延迟 | 应用延迟 SLO |

**练习查询清单**（逐条在 :30091 首页执行，观察结果）：

| 查询 | 含义 |
|------|------|
| `up` | 所有目标的存活状态（1=UP）——应 5 条全 1 |
| `up{job="node-exporter"}` | 只看节点目标 |
| `(1 - node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes) * 100` | 每节点内存使用率 %（Gauge 直接算） |
| `increase(kube_pod_container_status_restarts_total[10m])` | 各 Pod 近 10 分钟重启次数（Counter 配 increase） |
| `sum by (node) (rate(node_cpu_seconds_total{mode!="idle"}[5m]))` | 每节点 5 分钟平均 CPU 使用率——注意 `sum by` 聚合（CPU 有多核多序列，按 node 加总才有意义） |
| `node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}` | 根分区可用比例 |

> 练习 10 分钟：把内存那条查出来，记下 3 个节点的数值；再执行 `free -g` 与系统实际值对照——**Prometheus 的数字和操作系统一致，才算采集链路真的通了**。

---

## 阶段 6：Grafana 可视化（40 分钟）

### 6.0 组件工作原理（先懂再装）

**Grafana 的定位：无状态的可视化渲染层**。

- **不存指标数据**：所有查询实时发给数据源（Prometheus），拿到结果后在前端渲染成图——所以 Grafana 挂了不影响 Prometheus 采集，反之 Prometheus 挂了 Grafana 只是"无数据"而非崩溃。
- **数据源代理**：浏览器 → Grafana 后端 → Prometheus。即查询请求是 Grafana 服务端代发的（`Access: Server` 模式），浏览器不需要能直连 Prometheus——**这就是为什么数据源 URL 填集群内 DNS**（Grafana Pod 在集群内可达），而不是 30091 NodePort。
- **仪表盘即 JSON**：每个 dashboard 是一份 JSON（面板布局 + 每个面板的 PromQL 查询 + 变量定义）。"导入 1860"的本质就是导入一份别人写好的 JSON——这也意味着**大盘可以进 Git 版本管理（Dashboards as Code）**。
- **变量系统（Template Variables）**：1860 大盘顶部的 `$node` 下拉框就是变量——查询里写 `{{instance}}` 或 `{node=~"$node"}`，切换下拉即切换过滤。理解这个，自制大盘的"单节点视图"就都会做了。

### 6.1 部署

**先把 `06-grafana.yaml` 里的 `__NODE_NAME__` 改成和 Prometheus 同一节点**（数据源不出本集群没关系，但 local-path PV 节点绑定同理），然后：

```bash
kubectl apply -f manifests/06-grafana.yaml
kubectl rollout status deployment/grafana -n monitoring
# 预期：deployment "grafana" successfully rolled out

# 日志自检：看到 "HTTP Server Listen" 即正常
kubectl logs -n monitoring deploy/grafana --tail=10
```

Grafana 的持久化内容：数据源配置、大盘 JSON、用户/密码——都在 `/var/lib/grafana`（挂了 PVC）。**指标数据不在 Grafana 里**，就算 PVC 丢了也只是重配一遍数据源和大盘。

**对应文件：`manifests/06-grafana.yaml`**

```yaml
# ============================================
# 项目 C 阶段 5：Grafana 可视化
# 用法: 先把 __NODE_NAME__ 改成 Prometheus 所在的同一节点，再 apply
# 注意: Grafana 只存"配置"（数据源/大盘/账号），不存指标数据——
#       指标数据永远在 Prometheus 里，Grafana 只是查询渲染
# ============================================
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-data
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi    # 只放配置和大盘 JSON，比 Prometheus 的 5Gi 小得多
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  replicas: 1
  strategy:
    type: Recreate    # 同 Prometheus：local-path PV 节点绑定，不能新旧并存
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      # !!! 与 Prometheus 同节点（local-path PV 节点绑定）!!!
      nodeSelector:
        kubernetes.io/hostname: __NODE_NAME__
      containers:
        - name: grafana
          # 原始镜像: grafana/grafana:11.1.0（已中转）
          image: 11.0.1.128:30000/grafana:11.1.0
          env:
            # 初始管理员密码（首次登录 admin / 这个密码）
            # Grafana 所有配置都可用 GF_ 前缀环境变量覆盖（Twelve-Factor 风格）
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "Grafana@2026"
            - name: TZ
              value: Asia/Shanghai   # 时区：大盘时间轴和邮件时间戳才与本地一致
          ports:
            - containerPort: 3000    # Grafana 默认端口
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          volumeMounts:
            - name: data
              mountPath: /var/lib/grafana   # Grafana 唯一的本地数据目录（SQLite+插件）
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: grafana-data
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
    - name: web
      port: 3000
      targetPort: 3000
      nodePort: 30300    # 浏览器访问入口
```

### 6.2 首次登录

浏览器 **http://11.0.1.128:30300**：

| 字段 | 填写 |
|------|------|
| Username | `admin` |
| Password | `Grafana@2026`（YAML 里 `GF_SECURITY_ADMIN_PASSWORD` 环境变量注入的——**Grafana 的所有配置都可用 GF_ 前缀环境变量覆盖**，这是它 Twelve-Factor 风格的设计） |
| 后续弹窗 | Skip（跳过改密与新功能引导） |

### 6.3 添加 Prometheus 数据源（逐字段）

左侧菜单 → **Connections → Data sources → Add data source → Prometheus**：

| 字段 | 填写 | 说明 |
|------|------|------|
| Name | `prometheus` | 默认即可；后续大盘引用按这个名字 |
| Prometheus server URL | `http://prometheus.monitoring.svc.cluster.local:9090` | 集群内 Service DNS 全名（`<svc>.<ns>.svc.cluster.local`）；同 ns 短名 `http://prometheus:9090`也行，全名更规范 |
| Access | **Server（默认）** | Server=Grafana 后端代理查询（我们用这个）；Browser=浏览器直连（要求浏览器能访问该 URL，仅数据源暴露公网时用——面试可讲两者区别） |
| Auth 区块 | 全部不勾 | 集群内通信，Prometheus 未开认证 |
| Scrape interval | 留空 | 用数据源默认（跟随 global 15s） |
| Save & test | 点它 | 显示 "Successfully queried the Prometheus API" ✅ 即成功 |

> 排错：Save & test 失败先看 URL 拼写；再用 `kubectl exec -n monitoring deploy/grafana -- wget -qO- http://prometheus:9090/-/healthy` 在容器内实测连通性（Grafana Pod 视角的网络可达性）。

### 6.4 导入 Node Exporter Full 仪表盘（经典必装）

左侧 → **Dashboards → New → Import**：

| 字段 | 填写 |
|------|------|
| 导入方式 | 在 "Import via grafana.com" 输入框填 **`1860`** |
| 点 Load | 自动加载仪表盘定义（联网拉 JSON） |
| 数据源选择 | 下拉选刚建的 `prometheus`（**必做**——JSON 里的占位数据源要映射到真实的） |
| 点 Import | 完成 |

**1860 大盘内容导读**（出图后别只看热闹，知道每块是什么）：

| 区域 | 内容 | 用到的指标族 |
|------|------|-------------|
| 顶部 Total/In use 等 | 节点资源汇总数字 | node_memory_* |
| Host base Memory(RAM) | 内存细分（total/used/cache） | node_memory_Mem* |
| CPU Cores / CPU Basic | 各核心与整体 CPU | node_cpu_seconds_total |
| Disk Space / IO | 文件系统与磁盘 IO | node_filesystem_*、node_disk_* |
| Network Traffic | 网卡收发速率 | node_network_* |

看到三节点的 CPU/内存/磁盘/网络全景大屏 = 监控体系"眼见为实"的时刻。

> **若 Load 失败**（Grafana 容器访问 grafana.com 被墙——大概率发生）：
> 方案 B：在本机浏览器下载 JSON（https://grafana.com/grafana/dashboards/1860 → 下载）→ Import 页改为 **Upload JSON file** 上传，数据源照选 prometheus。
> 方案 C：先跳过 1860，用 6.5 自制精简版出图，链路同样算通。

### 6.5 自制一个最小面板（理解 Grafana 工作原理，10 分钟）

Dashboards → New dashboard → Add visualization → 选 prometheus 数据源，第一个 panel：

> **界面导览（先认清三块再动手）**：中间大块=面板预览区（查询为空时显示 No data——面板只是"查询的可视化壳"）；**底部 Query 区=PromQL 的家**；右侧 Panel options=外观设置（标题/单位/图例）。
> **⚠️ 关键操作**：Query 区默认是 **Builder** 模式（下拉框拼指标，拼复杂公式很痛苦），点 Run queries 旁边的 **Builder | Code** 切换钮选 **Code**，出现文本框后直接粘贴下面的查询语句。Legend 填 `{{node}}` 的位置在查询框下方 **Options → Legend**。

| 字段 | 填写 |
|------|------|
| Title | `节点内存使用率` |
| 查询语句 | `(1 - node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes) * 100` |
| Legend | `{{node}}`（模板语法：把每条序列的 node 标签值显示为图例——和告警 annotations 一个原理） |
| Unit（右侧面板） | Percent (0-100) |
| 保存 | Ctrl+S → 命名 `集群监控-自制` |

重复加一个 CPU 面板（查询用 5.5 节最后一条）。**亲手做过一次"查询→面板"的映射，比导 10 个现成大盘都涨功力。**

**常见报错排查**：

| 现象 | 原因 | 解决 |
|------|------|------|
| 面板显示 "No data" | 时间范围内无数据 / 数据源错 / 查询语句语法错 | 右上角时间调 Last 15 minutes；面板上切换数据源；点 "Edit" 在 Prometheus 里单测查询 |
| 曲线断断续续 | 采集间隔与图表分辨率问题 | 时间范围调小（Last 1h）；属正常现象不必处理 |
| 大盘数字全 0 | 1860 的某些 panel 依赖特定采集器 | 只要不影响 CPU/内存/磁盘主图即可忽略 |
| 保存后大盘丢失 | PVC 未生效（Pod 重启后回滚） | `kubectl get pvc -n monitoring` 确认 grafana-data Bound |

---

## 阶段 7：告警链路（Alertmanager + 邮件，60 分钟）

### 7.0 告警全链路时序（先建立全局图，面试画这条线）

一条告警从"故障发生"到"邮件到达"的完整旅程（标注每步由谁负责、耗时）：

```
T+0s    故障发生（如 work2 的 node-exporter 挂了）
        │
T+0~15s Prometheus 抓取失败（抓取循环发现 up=0）
        │ 规则评估循环：expr up{job="node-exporter"}==0 首次成立
T+0~15s 规则进入 pending（for: 1m 计时开始）
        │
T+1m    pending 满 1 分钟 → firing，生成告警对象
        │ Prometheus POST 推给 alertmanager:9093
        │
T+1m    Alertmanager 收到 → 进入 group（alertname=NodeDown）
        │ group_wait: 30s —— 等 30 秒看有没有同类告警一起发
        │
T+1m30s 组装邮件 → SMTP 发往 smtp.qq.com:465 → QQ 邮箱
        │
T+2m    📧 收到 [FIRING] NodeDown 邮件
        │
T+~2m   DaemonSet 重建 Pod，抓取恢复，up=1
        │ expr 不再成立 → resolved
T+~3m   📧 收到 [RESOLVED] 恢复邮件
```

**时间线里每个延迟的出处**：15s（抓取）+ 15s（评估）+ 60s（for）+ 30s（group_wait）+ SMTP 投递 ≈ 2 分钟总延迟——**监控告警是"分钟级"系统，不要期望秒级**（面试聊"你们的告警多快能收到"时的标准答案框架）。

### 7.1 获取 QQ 邮箱 SMTP 授权码（前置，5 分钟）

**⚠️ 先做网络连通性预检（内网 VM 环境必做）**：授权码只是邮箱侧的认证凭证，与部署位置无关；但 **Alertmanager 必须能连上 `smtp.qq.com:465`** 才发得出邮件。本环境是内网 VM（出外网历来要靠代理，项目 B 的插件安装/镜像拉取都验证过），所以部署前先测通不通：

```bash
# ① 测节点网络（master 上执行）
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/smtp.qq.com/465' \
  && echo "✅ 节点可直连 smtp.qq.com:465" || echo "❌ 节点不可直连"
# 0. 先确认能直连公网 DNS（节点上跑）
nslookup smtp.qq.com 223.5.5.5

# 1. 改 CoreDNS 的上游：把 /etc/resolv.conf 换成显式公网 DNS
kubectl edit cm coredns -n kube-system
#   找到这一行（在 Corefile 段里）：
#     forward . /etc/resolv.conf
#   改成：
#     forward . 223.5.5.5 119.29.29.29
#   保存退出（vi: Esc → :wq）

# 2. 重启 CoreDNS 生效
kubectl rollout restart deploy/coredns -n kube-system
kubectl rollout status deploy/coredns -n kube-system

# 3. 复测解析
kubectl run dns-test --rm -it --restart=Never --image=11.0.1.128:30000/alpine/k8s:1.35.0 -n monitoring -- nslookup smtp.qq.com

# 4. 复测 SMTP 连通（回到预检原点）
kubectl run smtp-test --rm -it --restart=Never --image=11.0.1.128:30000/alpine/k8s:1.35.0 -n monitoring -- nc -zv -w 5 smtp.qq.com 465

# 测 Pod 网络（Alertmanager 实际运行的网络视角，这才是关键）
# 用私有仓库现成的 alpine/k8s 镜像（自带 nc），在 monitoring namespace 里测
kubectl run smtp-test --rm -it --restart=Never \
  --image=11.0.1.128:30000/alpine/k8s:1.35.0 -n monitoring -- \
  nc -zv -w 5 smtp.qq.com 465
# 预期成功输出：smtp.qq.com (xxx.xxx.xxx.xxx:465) open
# 失败输出：nc: connect timeout / connection refused
```

| 测试结果 | 结论 | 对策 |
|---------|------|------|
| ①② 都通 | 直接按文档流程走，无需任何改动 | 继续 7.1 授权码步骤 |
| ① 通 ② 不通 | 节点能出、Pod 网络不能出（NAT/DNS 差异） | **方案 A：节点 socat 中转**（见下） |
| ①② 都不通 | 内网防火墙拦 465 | **方案 B：改用 webhook 通知 + 代理**（见下） |

**方案 A（节点 socat 中转）**：在 master 上把本机一个端口转发到 smtp.qq.com:465，Alertmanager 改连节点内网地址：

```bash
# master 上安装 socat（Ubuntu: sudo apt install socat）并启动转发
# 用 systemd 托管避免重启丢失：也可先手动跑验证效果
nohup socat TCP-LISTEN:10465,fork,reuseaddr TCP:smtp.qq.com:465 &
# 07-alertmanager-config.yaml 里改一行：
#   smtp_smarthost: "11.0.1.128:10465"
```

**方案 B（webhook + 代理）**：Alertmanager 的 webhook 通知走 HTTP 客户端，**支持 HTTPS_PROXY 环境变量**（SMTP 协议不支持 HTTP 代理，这是本质区别）。给 08-alertmanager.yaml 的容器加代理环境变量（指向你的代理 `11.0.1.1:7890`，NO_PROXY 排除集群网段），通知渠道换成钉钉/企业微信机器人 webhook。**注意：SMTP 即使配 HTTPS_PROXY 也不生效**（Go 的 SMTP 实现不读代理变量）——这是本项目预埋的一个"知其所以然"考点。

1. 浏览器登录 QQ 邮箱网页版（mail.qq.com）
2. **设置 → 账户** → 下拉找到 **POP3/IMAP/SMTP...服务**
3. 开启 **SMTP 服务**（需要手机短信验证）
4. 验证通过后**生成 16 位授权码**——只显示一次，复制保存

> 原理：第三方程序（Alertmanager）用授权码而不是 QQ 密码登录 SMTP 发信，这是国内邮箱的通用安全机制（163/126 同理）。

### 7.2 替换占位符并部署

编辑 `manifests/07-alertmanager-config.yaml`，把 `__SMTP_AUTH_CODE__` 换成授权码，然后：

```bash
kubectl apply -f manifests/07-alertmanager-config.yaml
kubectl apply -f manifests/08-alertmanager.yaml
kubectl rollout status deployment/alertmanager -n monitoring
# 预期：deployment "alertmanager" successfully rolled out

# 日志自检：Alertmanager 启动日志会打印加载的配置与监听地址
kubectl logs -n monitoring deploy/alertmanager --tail=15
# 预期：msg="Loading configuration file" file=/etc/alertmanager/alertmanager.yml
#       msg="Listening for" ... :9093
```

**对应文件 1：`manifests/07-alertmanager-config.yaml`（要填授权码的那个）**

```yaml
# ============================================
# 项目 C 阶段 6：Alertmanager 配置（告警路由 + 邮件通知）
# 用法: 先把 __SMTP_AUTH_CODE__ 替换成 QQ 邮箱 SMTP 授权码，再 apply
#   授权码获取: QQ邮箱网页版 → 设置 → 账户 → 开启 SMTP 服务 → 生成授权码
#   注意: 是"授权码"，不是 QQ 密码！
# ============================================
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      # 告警恢复通知的超时：超时未收到"已解决"信号，自动标记 resolved
      resolve_timeout: 5m
      # QQ 邮箱 SMTP（465 = SSL 直连，连接即加密）
      smtp_smarthost: "smtp.qq.com:465"
      smtp_from: "2956446350@qq.com"          # 发件人显示地址
      smtp_auth_username: "2956446350@qq.com" # SMTP 登录账号（与 from 一致）
      # !!! 替换成你的 16 位 SMTP 授权码 !!!
      # 配错的典型报错：AM 日志 Error on notify ... 550 认证失败
      smtp_auth_password: "__SMTP_AUTH_CODE__"
      # 465 端口走 SSL（隐式 TLS）而不是 STARTTLS（587 端口），关掉
      # 端口和 TLS 模式必须配套：465→false，587→true（自测题 Q10）
      smtp_require_tls: false

    # 路由树：告警进来后按标签分流
    # 教学环境只配 root（全部走默认接收器）；生产会有子路由按 severity 分组
    route:
      # 默认接收器
      receiver: email-me
      # 同名告警分到一组（一封邮件说清楚，不轰炸）
      group_by: [alertname]
      # 第一条告警等 30s，看有没有同组的可以合并
      group_wait: 30s
      # 同组新告警的合并间隔（与 group_wait 区分：wait 管首条，interval 管追加）
      group_interval: 5m
      # 未恢复的告警每 4 小时重复提醒
      repeat_interval: 4h

    receivers:
      - name: email-me        # root route 引用的默认接收器
        email_configs:
          - to: "2956446350@qq.com"   # 收件邮箱（自己的 QQ 邮箱）

    # 抑制规则：节点挂了时，屏蔽该节点上的所有告警（避免告警风暴）
    # 语义：NodeDown 在场时，equal 里 node 标签相同的 warning 级告警全部静音
    # ——节点都挂了，它上面的内存/磁盘告警是噪音
    inhibit_rules:
      - source_match:          # 源告警（触发抑制的）
          alertname: NodeDown
        target_match_re:       # 目标告警（被抑制的），_re 表示 regex 匹配
          severity: warning
        equal: [node]          # 两者的 node 标签值相同才抑制
```

**对应文件 2：`manifests/08-alertmanager.yaml`**

```yaml
# ============================================
# 项目 C 阶段 6：Alertmanager 本体（告警去重/分组/路由/通知）
# 用法: kubectl apply -f 08-alertmanager.yaml
# 注意: 告警历史存 /alertmanager（emptyDir 即可，丢了不影响核心功能）
# ============================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
  labels:
    app: alertmanager
spec:
  replicas: 1
  strategy:
    type: Recreate
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
          # 原始镜像: prom/alertmanager:v0.27.0（已中转）
          image: 11.0.1.128:30000/alertmanager:v0.27.0
          args:
            # 配置文件路径（挂载 ConfigMap 中的 alertmanager.yml）
            - --config.file=/etc/alertmanager/alertmanager.yml
            # 静默/通知状态本地存储路径（单实例教学环境够用）
            - --storage.path=/alertmanager
          ports:
            - containerPort: 9093
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
          volumeMounts:
            - name: config
              mountPath: /etc/alertmanager/alertmanager.yml
              subPath: alertmanager.yml   # 只挂文件，不覆盖目录
      volumes:
        - name: config
          configMap:
            name: alertmanager-config     # 引用 07 的 ConfigMap
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager    # Service 名 = Prometheus alerting.targets 引用的短名，别改
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: alertmanager
  ports:
    - name: web
      port: 9093
      targetPort: 9093
      nodePort: 30093    # 浏览器访问 AM UI（查看告警/配置静默）
```

**Alertmanager 工作原理**（通知流水线，面试画它）：

```
Prometheus 推来告警
    │
    ▼
分组（group_by）── 同组的告警合并成一条通知
    │
    ▼
抑制（inhibit_rules）── 高级告警在场时屏蔽同源低级告警
    │
    ▼
静默（silences）── 运维手动屏蔽（维护窗口用，UI 操作）
    │
    ▼
路由（route 树）── 按 match 标签决定发给哪个 receiver
    │
    ▼
通知（email/webhook/钉钉...）── 失败自动重试
```

### 7.3 alertmanager.yml 配置项逐个详解（面试点）

| 概念 | 配置项 | 精确语义 | 我们配的值 | 常见误解 |
|------|--------|---------|-----------|---------|
| 路由树 | `route` | 告警入口节点，子 route 按 match 递归匹配 | 只配了 root（全走 email-me） | route 是树不是平级列表；子路由要 `continue` 才继续往下匹配 |
| 分组维度 | `group_by: [alertname]` | 相同标签值的告警合并为一条通知 | 按告警名合并 | 分组过粗会漏细节，过细会轰炸 |
| 首组等待 | `group_wait: 30s` | 新组的**第一条**通知前等待，攒同类告警 | 30s | 不是"每条告警都等 30s" |
| 组内间隔 | `group_interval: 5m` | 同组**后续新增**告警的两次通知最小间隔 | 5m | 与 group_wait 区分：wait 管首条，interval 管追加 |
| 重复提醒 | `repeat_interval: 4h` | 告警未恢复时重复通知的间隔 | 4h | 与 group_interval 叠加计算；调太小=骚扰 |
| 抑制 | `inhibit_rules` | 源告警（critical）发生时，equal 标签相同的目标告警（warning）被屏蔽 | NodeDown 抑制同 node 的 warning | `equal` 指定"哪些标签相同才算同一来源" |
| 静默 | （UI 操作） | 时间窗口内屏蔽匹配的告警 | 维护窗口用 | 与抑制的区别：静默是人手动设的、有期限 |

**5 条告警规则逐条拆解**（04 配置里的 rules.yml，每条的 PromQL 都值得在 Prometheus 手跑一遍）：

| 规则 | expr 拆解 | for | 为什么是这些参数 |
|------|----------|-----|-----------------|
| NodeDown | `up{job="node-exporter"} == 0` | 1m | 抓取失败即 up=0；留 1m 容忍单次网络抖动 |
| NodeMemoryHigh | `(1 - 可用/总量) > 0.85` | 2m | Gauge 直接算比例；内存冲高常见瞬时，观察期比 NodeDown 长 |
| NodeDiskAlmostFull | 根分区可用比 < 0.1 | 5m | 磁盘增长慢，长观察期+严阈值 |
| PodCrashLooping | `increase(重启次数[10m]) > 3` | 0m | 重启本身就是离散事件，无需 for |
| DeploymentReplicasMismatch | `可用副本 != 期望副本` | 5m | 部署滚动期间会短暂不匹配，for 避免误报 |

### 7.4 验证告警链路（三选一，都做最好）

**方法 ① 杀掉一个 node-exporter（最简单，触发 NodeDown）**

```bash
# 删掉 work2 上的 node-exporter（DaemonSet 会立刻重建，但 1 分钟的空窗够触发）
POD=$(kubectl get pods -n monitoring -l app=node-exporter -o wide | grep work2 | awk '{print $1}')
kubectl delete pod -n monitoring $POD
# 预期输出：pod "node-exporter-xxxxx" deleted
```

**预期时间线**：`kubectl get pods` 看到 node-exporter 重建（约 10-20s 拉起）→ 但 Prometheus 侧 up=0 的判定与恢复需要几个抓取周期 → 即使 Pod 很快恢复，**pending/firing 状态大概率还是会出现并自己消失**；如果 Pod 重建慢（镜像拉取），Firing 会更明确。

**方法 ② 给节点加内存压力（触发 NodeMemoryHigh）**

```bash
# 在任意 worker 上跑 2 分钟的内存压力（没有 stress-ng 的话用方法 ①③）
stress-ng --vm 2 --vm-bytes 1G --timeout 120s
```

**方法 ③ Prometheus UI 观察状态机（最快，纯观察）**

打开 http://11.0.1.128:30091 → **Alerts** 页 → 这页能看到全部规则及其状态：

| 状态显示 | 颜色 | 含义 |
|---------|------|------|
| INACTIVE | 灰 | expr 不成立 |
| PENDING (0:59) | 黄 | 成立但 for 未满（括号里是剩余秒数倒计时） |
| FIRING | 红 | 已推给 Alertmanager |

用方法 ① 删 Pod 后**盯住 Alerts 页刷新**，亲眼看 INACTIVE → PENDING → FIRING → RESOLVED 的完整流转——这是面试要讲的状态机，看过一次印象抵十遍文档。

### 7.5 验证

1. Prometheus → **Alerts**：目标规则变 **FIRING**（红色）
2. 打开 Alertmanager **http://11.0.1.128:30093** → 首页应看到 1 条告警（NodeDown，critical）→ 说明 Prometheus→AM 推送链路通了
3. 等 30s（group_wait）+ 发信延迟，**QQ 邮箱收信**（去垃圾箱也翻一下）
4. 告警恢复后还有一封 RESOLVED 邮件

**邮件预期**：标题 `[FIRING:1] NodeDown (critical ...)`；正文包含 annotations 里的 summary 与 labels（node=work2）——**检查 node 标签是否正确显示了**，能显示说明阶段 5 relabel 第 ③ 步（固化节点名标签）生效了，整条链路是真正打穿的。

**链路断点诊断**（按此顺序定位，邮件没到就逐段查）：

| 段 | 检查 | 方法 |
|----|------|------|
| ① 规则→Firing | Prometheus /alerts 页 | 状态机没到 firing → 查 expr |
| ② Firing→AM | AM 首页有没有告警 | 没有 → 查 Prometheus 配置的 alerting.targets；`kubectl logs deploy/prometheus \| grep -i alert` |
| ③ AM→SMTP | AM 日志的 notify error | `kubectl logs deploy/alertmanager` 找 `Error on notify`，常见为 550 认证失败（授权码错）或连接超时（网络） |
| ④ SMTP→邮箱 | 邮箱收件+垃圾箱 | 都没有 → 换授权码重试，检查 smtp_from 与 auth_username 一致性 |

---

## 阶段 8：全链路验证与面试演示

### 8.1 一键验证

```bash
bash scripts/verify.sh
```

脚本依次检查：Pod 状态、node-exporter 数量、三个 Web 入口健康度，并打印手动检查清单。

### 8.2 最终验证清单

| # | 检查项 | 通过标准 |
|---|--------|---------|
| 1 | monitoring 命名空间 Pod | **7 个**全部 Running（3 node-exporter + 1 kube-state-metrics + 1 prometheus + 1 grafana + 1 alertmanager） |
| 2 | Prometheus Targets | 5 条全 UP |
| 3 | Prometheus Rules | 5 条规则 loaded |
| 4 | Grafana 数据源 | Save & test 成功 |
| 5 | Grafana 仪表盘 | 1860 出图 / 自制面板出图 |
| 6 | 告警状态机 | 能看到 inactive→pending→firing 流转 |
| 7 | 邮件 | FIRING + RESOLVED 都收到过 |
| 8 | CI/CD 仍然健康 | curl http://11.0.1.128:30090/ 正常（监控自己别把业务搞挂） |

### 8.3 60 秒面试演示脚本

```
1. 打开 Grafana :30300 —— "这是我的集群大盘，三节点的 CPU/内存/磁盘实时水位"
2. 打开 Prometheus :30091/targets —— "所有采集目标，用 kubernetes_sd_configs
   服务发现自动维护，新节点加入自动纳管，不用改配置"
3. 打开 :30091/alerts —— "我定义了 5 条告警规则，这是状态机"
4. 讲一次真实触发 —— "我删过 node-exporter 容器，1 分钟后收到 NodeDown 邮件，
   恢复后自动收 RESOLVED——Alertmanager 的 group_wait 30s 合并机制防止告警风暴"
5. 反差点 —— "我用裸部署理解了每个组件的职责边界；生产上我会用
   kube-prometheus-stack(Helm) 让 Operator 管理生命周期"
```

### 8.4 监控项目 B 的组件（加分项，把两个项目串起来）

在 04-prometheus-config.yaml 的 ConfigMap 里给 Prometheus 加抓取 Jenkins 指标（Jenkins Prom 插件或 `/prometheus` 端点），ConfigMap 更新后热加载：

```bash
kubectl apply -f manifests/04-prometheus-config.yaml
# 热加载（Deployment 里开了 --web.enable-lifecycle）
curl -X POST http://11.0.1.128:30091/-/reload
```

**预期输出**：返回空内容 + HTTP 200 即重载成功；验证方式是 Prometheus /targets 里出现新 job 且 UP。

> 这个"配置热加载不重启"细节也是面试可讲的点——`--web.enable-lifecycle` 开启了 `/-/reload`，等价于给 Prometheus 发 SIGHUP。

---

## 9. 故障速查表

### 9.1 部署类

| 现象 | 原因 | 解决 |
|------|------|------|
| 镜像拉取超时/404 | 没跑中转脚本或 tag 不存在 | 回阶段 2 跑 `pull-images.sh`；`crictl pull 11.0.1.128:30000/xxx` 验证 |
| Prometheus 反复重启 | PVC 所在节点与 nodeSelector 不一致 / 配置文件语法错 | `kubectl logs -n monitoring deploy/prometheus`；PV 节点绑定见 5.3 |
| Grafana 起不来 | grafana-data PVC 属主问题 | `kubectl describe pod` 看 Mount 失败；确认 fsGroup 或重建 PVC |
| node-exporter 只有 2 个 | master 污点没容忍 | 确认 YAML 里 `tolerations: - operator: Exists` |
| Prometheus OOMKilled | 标签基数爆炸 / limit 给太低 | `kubectl describe pod` 看 Last State OOMKilled；排查高基数指标（如把 UID 做标签），适当调高内存 limit |
| ConfigMap 改了但行为没变 | Pod 内挂载的文件是只读同步的，但 Prometheus不会自动重读 | `curl -X POST http://11.0.1.128:30091/-/reload` 或重启 Deployment |

### 9.2 采集类（Targets DOWN 排查顺序）

| 现象 | 原因 | 解决 |
|------|------|------|
| node-exporter 3 条全 DOWN | relabel 过滤条件与 Service 标签/端口名不匹配 | Service 必须带 `app: node-exporter` 标签、端口名必须叫 `metrics`（对照 01/04 两个 YAML 的注释） |
| kube-state-metrics DOWN | Service 名不对/没起 | `kubectl get svc -n monitoring`；短名解析仅限同 namespace |
| Targets 页面空白 | Prometheus RBAC 不足，服务发现查询被拒 | 确认 03 的 ClusterRoleBinding 已 apply |
| Target Error: `connection refused` | 目标 Pod 没起或端口错 | `kubectl get endpoints -n monitoring` 对照 |
| Target Error: `server returned HTTP status 403 Forbidden` | 目标端有认证（如 kubelet metrics） | 本项目 targets 无认证，遇到即检查是否抓错了目标 |
| 目标 UP 但 Grafana 无数据 | 时间范围/数据源选错 | Grafana 右上角时间调 Last 15 minutes；数据源确认是 prometheus |

### 9.3 告警类

| 现象 | 原因 | 解决 |
|------|------|------|
| 规则一直 inactive | expr 写错查不出数据 / 阈值太高 | Prometheus 首页手动跑一遍 expr，确认有返回值 |
| 状态卡在 pending 不变 | for 时间没到 / 评估周期异常 | 等 for 满；检查 evaluation_interval |
| firing 但 Alertmanager 没收到 | alerting 段没配 / 地址错 | ConfigMap `alerting.alertmanagers.targets` 应为 `alertmanager:9093` |
| Alertmanager 收到但没邮件 | SMTP 授权码错 / 465 端口被拦 | `kubectl logs deploy/alertmanager -n monitoring` 找 notify error；授权码重新生成 |
| 邮件报 `550 User has no permission` | SMTP 服务未开启或授权码错 | 回 7.1 重新生成授权码 |
| 邮件进了垃圾箱 | 正常现象 | 标记非垃圾邮件 |
| 告警一直 Firing 不恢复 | expr 表达式的条件持续为真 | 检查 expr 语义（如内存阈值定太低永远超过） |
| 告警风暴（几十封邮件） | group_by 太细 / repeat_interval 太短 | 检查分组配置；恢复阈值；临时用 AM UI 的 Silence 止血 |

### 9.4 国内网络专项（项目 B 同款坑的 C 版）

| 现象 | 解决 |
|------|------|
| quay.io / registry.k8s.io 拉不动 | 全部走阶段 2 的私有仓库中转（YAML 已写死私有地址，不需要节点配 quay） |
| Grafana 导入 1860 Load 失败 | 6.4 的方案 B（本机下载 JSON 后 Upload）或方案 C（自制面板） |
| Grafana 插件/主题下载失败 | 不影响核心功能，跳过 |

### 9.5 通用排障方法论（所有阶段通用，面试可讲）

```bash
# 第一层：资源状态对不对
kubectl get pods -n monitoring -o wide          # RUNNING 才有下一步
kubectl get endpoints <svc> -n monitoring       # 网络端点是否存在
kubectl get pvc -n monitoring                   # 存储是否 Bound

# 第二层：组件自己怎么说
kubectl logs deploy/<name> -n monitoring --tail=50     # 启动自检报告
kubectl describe pod <pod> -n monitoring               # Events 看调度/挂载/探针失败原因

# 第三层：组件间链路通不通（在 Pod 视角测试）
kubectl exec -n monitoring deploy/grafana -- \
  wget -qO- http://prometheus:9090/-/healthy            # Grafana→Prometheus
curl http://11.0.1.128:30091/api/v1/query?query=up      # Prometheus 数据面

# 原则：从数据源头（/metrics）往下游逐段验证，每一段都要亲眼看到数据，
# 不要跳级排障——"最后一步没数据"时，八成问题在前面某一段。
```

---

## 10. 自测题（含答案）

**Q1：Prometheus 为什么用 pull 而不是 push？**
A1：① 拉取端（Prometheus）掌握采集节奏，目标故障时可发现"抓不到"（up=0 即告警）；push 模型目标"沉默"时难以区分"没数据"和"挂了"；② 简化目标侧配置（目标只需暴露 /metrics，不用知道监控中心在哪）；③ 配合服务发现天然适配动态环境。代价：短生命周期 Job 需 Pushgateway 补充。

**Q2：node-exporter 和 kube-state-metrics 的区别？**
A2：前者监控**节点主机层**（硬件/内核视角，读宿主机 /proc /sys，DaemonSet 每节点一个）；后者监控 **K8s 对象状态**（调 API list/watch，把 Pod/Deployment/Node 的状态转成指标，单实例 Deployment）。一个"看机器"，一个"看集群对象"。补充：KSM 不存历史（每次被抓时现生成快照）、不能多副本（会产出重复指标）。

**Q3：kubernetes_sd_configs 的 relabel 流程是怎么工作的？**
A3：服务发现先把每个候选目标注入 `__meta_kubernetes_*` 元数据标签；relabel_configs 依次执行——keep/drop 按条件筛选（我们按 Service label 和端口名过滤），replace 把有用的元数据（节点名）固化成业务标签；最后剩下的目标进入抓取循环。action 全集：keep/drop/replace/labelmap/labeldrop。

**Q4：为什么 Prometheus/Grafana 要 nodeSelector 固定节点？**
A4：local-path 存储的 PV 绑定在首次消费它的节点本地，Pod 漂移到别的节点时 PV 挂不上（数据"丢失"）。单副本 + 本地存储的组合必须 nodeSelector 钉死；生产用网络存储（NFS/Ceph）或 StatefulSet + 拓扑感知可解。

**Q5：告警从触发到收邮件经过哪些状态和组件？**
A5：Prometheus 规则评估：expr 成立→**pending**（for 未满）→持续满足→**firing**→推给 Alertmanager→按 route 分组（group_wait 30s 合并）→email_configs 发 SMTP→QQ 邮箱。恢复时 expr 不再成立→resolved→发 RESOLVED 邮件。整条链路分钟级延迟：15s 抓取 + 15s 评估 + for + group_wait + SMTP。

**Q6：Alertmanager 的 inhibit_rules 有什么用？举我们的例子。**
A6：告警抑制——当高优先级告警发生时屏蔽相关的低优先级告警，防止告警风暴。例子：NodeDown 触发时，该节点上必然的内存/磁盘告警都是噪音，inhibit 规则按 node 标签（equal 字段）把 warning 级别全部屏蔽。

**Q7：for 参数的作用？**
A7：防抖动。expr 短暂毛刺（如内存瞬时冲高）不触发告警，持续满足 for 时长才进入 firing。NodeDown 配 1m：抓取目标偶发一次失败不报警，真挂了 1 分钟后才报——用时效换准确。

**Q8：生产上你会怎么改这套部署？**
A8：① 换 kube-prometheus-stack（Helm+Operator）管理全生命周期；② Prometheus 双副本/联邦/Thanos 长期存储；③ 告警换钉钉/企微 webhook + 值班分级；④ registry/认证/网络存储/资源限额按生产基线补齐；⑤ Grafana SSO + 大盘即代码（JSON 进 Git）。

**Q9：怎么监控 Prometheus 自己？**
A9：job=prometheus 自抓 localhost:9090（我们配了）；配 `prometheus_tsdb_*` 系列指标告警（如存储增长过快）；更进一步的"元监控"（监控监控系统的存活）在大型生产里由第二个 Prometheus 实例或外部拨测承担。

**Q10：为什么邮件告警走 465 端口且要关 smtp_require_tls？**
A10：QQ 邮箱 SSL 直连端口是 465（隐式 TLS，连接即加密）；而 STARTTLS 是明文连上后升级加密（587 端口），Alertmanager 的 smtp_require_tls 控制 STARTTLS。465 + require_tls=false 才能对上 QQ 的加密方式，配 587 则要 true——端口和 TLS 模式必须配套。

**Q11（进阶）：Prometheus 的数据是怎么存的？进程被 kill -9 会丢数据吗？**
A11：三层结构——内存 head block（最近 ~2h）+ WAL 预写日志 + 磁盘 2h 不可变 block（后台压缩合并）。kill -9 后重启会**从 WAL 回放恢复**内存数据，基本不丢（最多丢 WAL 未刷盘的几秒）；数据文件损坏才需要重建。retention 参数控制磁盘数据的保留期。

**Q12（进阶）：Grafana 的数据源 Access 模式 Server 和 Browser 有什么区别？**
A12：Server（默认）= Grafana 后端代理查询，浏览器只见 Grafana，数据源 URL 可以是集群内地址——我们的场景（数据源是集群内 Service DNS）必须用它；Browser = 浏览器直连数据源，要求浏览器网络可达（数据源暴露公网或同内网），有把数据源地址暴露给用户侧的安全面。

---

## 11. 面试要点与简历写法

**简历一句话（STAR）**：
> 在 3 节点 K8s 集群上基于 Prometheus + Grafana + Alertmanager 从 0 搭建监控告警体系：node-exporter（DaemonSet+hostNetwork）与 kube-state-metrics 双层采集，kubernetes_sd_configs 服务发现 + relabel_configs 自动纳管全部节点，配置 5 条告警规则（节点宕机/内存/磁盘/CrashLoop/副本不足）接入 QQ 邮箱 SMTP 实现真实告警闭环，并验证了 inactive→pending→firing 完整状态机与 Alertmanager 分组/抑制策略。

**高频追问与你的弹药**：自测题 Q1-Q12 全覆盖；项目 B 的"30090 端口冲突"（0.5 节）、"local-path 节点绑定"（5.3）是跨项目串联的好素材。

**概念纵深清单**（被追问时的知识锚点，均可在本文档找到答案）：
- TSDB 三层存储与 WAL 恢复（1.1）｜Counter 必须 rate()（1.2）｜USE vs RED（1.3）
- Exporter 白盒/黑盒（1.4）｜服务发现四个 role（5.2）｜relabel 五种 action（5.2）
- 告警状态机与分钟级延迟构成（7.0）｜group_wait/interval/repeat 三兄弟辨析（7.3）
- 数据源 Server vs Browser 模式（6.0/6.3）

**下一项目伏笔**：监控告警解决"看得见"，项目 D 可选"日志体系（Loki）"或"GitOps（ArgoCD）"补齐可观测性与发布自动化两块。

---

## 12. 学习资源

| 类别 | 资源 |
|------|------|
| 官方文档 | Prometheus 官网 docs（Data model / Configuration / relabel）：https://prometheus.io/docs/ |
| 官方文档 | Alertmanager：https://prometheus.io/docs/alerting/latest/overview/ |
| 官方文档 | Grafana：https://grafana.com/docs/grafana/latest/ |
| 组件文档 | kube-state-metrics：https://github.com/kubernetes/kube-state-metrics |
| 书籍 | 《Prometheus 监控实战》（Joe Bodia）——与本项目结构高度对应 |
| 课程 | KodeKloud Prometheus Certified Associate (PCA) 备考课（英文，配实验环境） |
| 练习 | https://killercoda.com/playgrounds（免费 K8s 在线 playground，可练 PromQL） |
| 仪表盘 | Grafana Dashboards 库：https://grafana.com/grafana/dashboards/（1860 = Node Exporter Full） |
| CTA 考证 | Prometheus Certified Associate——监控方向的加分认证 |

---

## 附录 A：文件清单

```
K8s项目C_监控告警/
├── manifests/
│   ├── 00-namespace.yaml            # monitoring 命名空间
│   ├── 01-node-exporter.yaml        # DaemonSet + Headless Service
│   ├── 02-kube-state-metrics.yaml   # SA+RBAC+Deployment+Service
│   ├── 03-prometheus-rbac.yaml      # 服务发现所需权限
│   ├── 04-prometheus-config.yaml    # ★ 抓取配置 + 5 条告警规则
│   ├── 05-prometheus.yaml           # PVC+Deployment+Service(:30091)
│   ├── 06-grafana.yaml              # PVC+Deployment+Service(:30300)
│   ├── 07-alertmanager-config.yaml  # ★ SMTP 告警路由（要填授权码）
│   └── 08-alertmanager.yaml         # Deployment+Service(:30093)
└── scripts/
    ├── pull-images.sh               # 镜像私有仓库中转（阶段 2）
    └── verify.sh                    # 一键验证（阶段 8）
```

部署顺序即文件名编号：00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08。

> **文档内嵌 YAML 与文件包的关系**：阶段 3-7 的操作步骤下方已内嵌全部 9 个 YAML 的完整内容（注释更详细的"教学版"）；`manifests/` 目录下的同名文件是可直接 apply 的部署版（YAML 字段与内嵌版完全一致）。两处内容语义相同，修改时请保持同步。

## 附录 B：进阶路线——kube-prometheus-stack（生产主流）

```bash
# 国内网络需先给 helm 配置可用的 chart 源（如阿里云镜像）并注入代理
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30300
```

学完裸部署再上它，你会清楚 Operator 替你干了什么（CRD：Prometheus/PrometheusRule/ServiceMonitor 自动渲染配置）——**先手动再自动，是理解一切 Operator 的正确顺序**。
