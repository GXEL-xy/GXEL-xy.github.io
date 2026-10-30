---
title: "K8s 项目 D：日志收集实战（Fluent Bit + Loki + Grafana）"
date: 2026-09-10T09:55:00+08:00
draft: false
description: "在三节点 K8s 集群以 Fluent Bit DaemonSet 采集全集群容器日志推送 Loki，Grafana Explore 检索与日志面板与项目 C 指标同入口，Loki ruler 日志告警复用 Alertmanager 邮件链路，含 promtail→Fluent Bit 采集器切换实战复盘。"
tags: ["k8s日志收集"]
categories: [项目]
---

> 📦 **项目文件包**：[下载 K8s项目D_Loki日志.zip](/files/K8s项目D_Loki日志.zip)（含全部 YAML manifests——Loki 配置与部署、告警规则、Fluent Bit 在役配置及 promtail 存档，可直接拷到集群 master 使用）。本文件是详细操作手册，两者配合。

> **系列定位**：K8s 运维实习项目路线图第 4 站——A 应用容器化 → B CI/CD 流水线 → C 监控告警 → **D 日志收集**。
> **版本基线**：Kubernetes v1.35.7 ／ containerd 2.x ／ Ubuntu 24.04 ／ Grafana 11.1.0 ／ **Loki 3.3.2 ／ Fluent Bit 3.2.2（定版采集器）**。

---

> **📖 怎么用这份文档**
>
> - **前置依赖**：项目 C 已完成（Grafana 与私有仓库可直接复用）；集群节点有足够磁盘
> - **预计耗时**：4–6 小时
> - **学习路径**：阶段 1 日志采集原理 → 阶段 2 镜像准备 → 阶段 3 Loki → 阶段 4 Fluent Bit（核心）→ 阶段 5 Grafana 检索 → 阶段 6 保留策略 → 阶段 7 日志告警
> - **验收标准**：Grafana Explore 里能按 `namespace`/`pod` 标签查到业务日志；LogQL 四级查询都能跑通

## 目录

- [0. 项目概览](#0-项目概览)
- [阶段 1：先懂再装——日志采集原理](#阶段-1先懂再装日志采集原理)
- [阶段 2：镜像准备](#阶段-2镜像准备)
- [阶段 3：Loki](#阶段-3loki)
- [阶段 4：Fluent Bit 采集器★本项目核心](#阶段-4fluent-bit-采集器本项目核心)
- [阶段 5：Grafana 接入与日志检索](#阶段-5grafana-接入与日志检索)
- [阶段 6：日志保留与磁盘管理](#阶段-6日志保留与磁盘管理)
- [阶段 7：日志告警](#阶段-7日志告警)
- [阶段 8：全链路验证与面试演示](#阶段-8全链路验证与面试演示)
- [9. 故障速查表](#9-故障速查表)
- [10. 自测题](#10-自测题)
- [11. 面试要点与简历写法](#11-面试要点与简历写法)
- [12. 学习资源](#12-学习资源)
- [附录 A：文件清单](#附录-a文件清单)
- [附录 B：进阶路线——Grafana Alloy（生产主流）](#附录-b进阶路线grafana-alloy生产主流)

---

## 0. 项目概览

### 0.1 目标与产出

| 产出 | 说明 |
|------|------|
| 全集群日志采集 | 3 节点全部容器日志自动入库，Pod 重建后新日志自动跟上 |
| Loki 日志数据库 | 标签索引 + 压缩块存储（PVC 10Gi），数据保留 7 天 |
| Grafana 日志检索 | Explore 交互式检索 + 日志面板仪表盘（与项目 C 指标同入口），流标签带完整 namespace/pod/container |
| 日志告警链路（进阶） | Loki ruler 规则评估 → 复用项目 C 的 Alertmanager → QQ 邮箱真实收信 |
| 面试完整故事线 | "我搭了日志系统：采集器选型踩坑后切换（promtail → Fluent Bit），指标定位时间窗 → 日志确认根因" |

### 0.2 架构与数据流

```
┌────────────── 3 节点 K8s 集群（monitoring namespace）──────────────────┐
│                                                                        │
│  [Fluent Bit DaemonSet] × 3（每节点 1 个 Pod）                           │
│   ① tail 宿主机 /var/log/containers/*.log（软链 → /var/log/pods/...）    │
│   ② cri parser 解析 containerd 日志格式 → 真实时间戳 + 纯正文              │
│   ③ 正则从文件路径提取标签：pod / namespace / container（零 API 依赖）      │
│        │ positions DB 记录读到哪一行（断点续采）                            │
│        │ HTTP push（/loki/api/v1/push）                                  │
│        ▼                                                                 │
│  [Loki Deployment × 1]（work2，PVC 10Gi，:3100）                          │
│   接收 → WAL → 按"标签组合"分块压缩落盘（chunks + TSDB 索引）                │
│   compactor：按 168h 滚动清理过期日志                                      │
│   ruler（进阶）：周期评估日志告警规则 ──────────────┐                       │
│        │ 查询 API :3100                          │ 推告警                │
│        ▼                                         ▼                       │
│  [Grafana（项目 C 已部署）:30300]          [Alertmanager（项目 C）:30093]   │
│   数据源 Loki → Explore/仪表盘              分组/路由 → QQ 邮箱 📧          │
└────────────────────────────────────────────────────────────────────────┘
```

**一句话记住分工**：
- **Fluent Bit**：tail 日志文件、解析打标签、断点续采、推给 Loki（日志界的 node-exporter）
- **Loki**：收日志、按标签建索引、压缩存储、提供查询 API 和告警规则评估（日志界的 Prometheus）
- **Grafana**：只管查询和展示（数据源加一个 Loki，项目 C 的投资直接复用）

**数据流五步走**（面试可以画这张图）：

| 步骤 | 谁负责 | 频率 | 失败表现 |
|------|--------|------|---------|
| ① tail 文件 | Fluent Bit 直接读宿主机文件（**不依赖 API Server**） | 实时跟随 | 静默无日志（查挂载与正则） |
| ② 解析加工 | cri parser + 正则提取标签 | 每行 | 时间戳失真、正文带 CRI 前缀 |
| ③ 推送入库 | HTTP push → Loki | 批次（Flush 5s） | `no upstream connections` / retry 堆积 |
| ④ 存储清理 | Loki ingester + compactor | 持续 | 磁盘只进不出 |
| ⑤ 查询展示 | Grafana Explore / 仪表盘 | 按需 | 数据源 Save & test 失败 |

> 与多数教程不同：本方案的标签提取**不经过 kubernetes filter（不查 API Server）**——纯本地正则从文件名解析。这是在本环境踩坑后的实战定版（原因见 0.3 与附录 C），也是 Fluent Bit 的官方支持用法。

### 0.3 关键技术决策（面试可讲）

| 决策 | 选择 | 理由 |
|------|------|------|
| 日志方案 | **Loki 而非 EFK/ELK** | 只索引标签不索引正文：存储省、内存省、与已有 Grafana/PromQL 体系同构；EFK 的全文检索优势在 3 节点小集群用不上，JVM 成本先吃掉你 2Gi 内存 |
| 采集器 | **Fluent Bit 3.2.2（实战定版）** | 首选 promtail，但其 kubernetes_sd 在本环境（K8s 1.35）对 API Server **静默失效**（六层排除链 + 双版本 + 双连接方式全灭，复盘见附录 C）；Fluent Bit 用**纯本地正则提取标签（零 API 依赖）**，反而更稳。promtail 的 maintainer 官方继任者 Alloy（附录 B）是长期演进方向 |
| 标签提取 | 文件路径正则（`Path_Key` + parser filter） | 日志文件名 `<pod>_<ns>_<container>-<id>.log` 自带全部元数据——**零 API 依赖**意味着：不查 API Server、不需要 RBAC、不受 API 兼容性影响，纯本地正则一次到位 |
| Loki 部署形态 | monolithic 单副本（手写 YAML） | 教学直白、组件间调用链看得见；生产对照是 Simple Scalable 或微服务模式 |
| 标签设计 | 只取 node 环境注入 + pod/namespace/container 低基数标签 | 标签=索引：每个新标签组合都是一份索引成本；UID/时间戳这类高基数字段做标签 = 索引爆炸 |
| 存储 | local-path PVC 10Gi + **nodeSelector 固定节点** | 吸收项目 C 教训：local-path PV 绑定节点，Pod 漂移=数据"消失" |
| 镜像 | **Fluent Bit 直拉公共仓库（实战定版）**；Loki 走私有仓库中转 | fluent-bit 官方镜像是 Buildx 多架构清单，push 到 registry:2 会报 `does not provide any platform`（清单格式不兼容）；而 containerd 2.x **原生支持 OCI index 直拉**——绕开 registry:2 反而更稳 |
| 告警通知 | 复用项目 C 的 Alertmanager + QQ 邮箱 | ruler 直接推 alertmanager:9093，零新建通知渠道 |
| 查询入口 | Grafana 加 Loki 数据源 | 指标（Prometheus）+ 日志（Loki）一个入口，排障"指标定位时间窗 → 日志确认根因"不换窗口 |

### 0.4 执行前必须替换的占位符

本项目只有 **1 个占位符**（Fluent Bit 镜像直拉公共仓库无需中转；Loki 镜像已写死私有仓库地址）：

| 占位符 | 出现文件 | 替换成 | 怎么查 |
|--------|---------|--------|--------|
| `__NODE_NAME__` | manifests/02-loki.yaml（1 处文件 1 行） | 固定节点名 | `kubectl get nodes -o wide`（建议 work2） |

**替换方法**（在 master 上）：

```bash
cd ~/k8s-loki/manifests
kubectl get nodes -o wide
sed -i 's/__NODE_NAME__/work2/g' 02-loki.yaml
grep -rn "__NODE_NAME__" . || echo "✅ 占位符已全部替换"
```

> 为什么必须固定节点：local-path 存储的 PV 绑定在**首次消费它的节点**本地。Loki 写了 10Gi 日志数据后如果 Pod 漂移到别的节点，PV 挂不上 → 数据"丢失"。这是项目 C 5.3 节同款教训。

### 0.5 端口规划（接续项目 B/C，避免冲突）

| 已占用（项目 B/C） | 新分配（项目 D） |
|------------------|------------------|
| 30000 registry<br>30080 Jenkins Web<br>31000 Jenkins jnlp<br>30090 damo-app<br>30091 Prometheus Web<br>30300 Grafana Web<br>30093 Alertmanager Web | **3100** Loki HTTP（ClusterIP，仅集群内）<br>**2020** Fluent Bit 自带指标（Pod 内，不出节点）<br>**9096** Loki gRPC（内部占位） |

> 项目 D 的组件全部走 ClusterIP/节点内部端口，**不新增任何 NodePort**——日志查询入口就是项目 C 的 Grafana（:30300）。新建服务前照例先 `kubectl get svc -A | grep -E "3[0-2][0-9]{3}"` 排查占用。

---

## 阶段 1：先懂再装——日志采集原理

### 1.0 开胃故事：一座"图书馆"

日志系统的三个组件，合起来就是一座"给集装箱小区做 7×24 记录"的图书馆。**读不懂原理时，回来重读这一节。**

**① 抄表员（Fluent Bit）——每个楼栋派一个。**
他拿着登记本在小区里转：先看门牌就知道住户是谁（**从文件路径直接解析 pod/namespace，不问物业**），然后趴在每户门口抄记录（tail 日志文件），抄之前把"楼栋/房间号/户主"写在页眉上（打标签），抄到哪一页做个书签（DB 位点），抄满一沓就送去图书馆（push）。

**② 图书馆（Loki）——只建索引卡，不复印全书。**
收下登记册后，图书馆**不逐字阅读内容**，只按"页眉信息"（楼栋+房间+户主）做索引卡，然后把整册书压箱底（chunk 压缩落盘）。你要找书时，先说"我要 3 号楼 502 室的册子"（标签选择器），馆员按索引卡直接抽册子，**再当着你的面逐页翻**内容找关键词（管道过滤）。这就是 Loki 和传统图书馆（Elasticsearch 全文索引）的根本区别：**卡片极薄，书山便宜，但翻页要你自己在查询里完成**。

**③ 阅览室（Grafana）——只管借阅，不管藏书。**
阅览室不存一本书（Grafana 不存日志），你填一张借阅单（LogQL），它去 Loki 把册子调出来给你看，还能把常用借阅单贴墙上（Dashboard 面板）。

> 读原理卡壳时记住：**Fluent Bit 抄书打页眉，Loki 做卡片藏书，Grafana 给你看书**。

### 1.1 容器日志落在哪（K8s + containerd 视角）

动手前先搞清楚"日志文件到底在哪"——这是采集配置的物理基础：

| 层 | 位置 | 说明 |
|----|------|------|
| 真实日志 | `/var/log/pods/<ns>_<pod>_<uid>/<container>/0.log` | kubelet 让容器运行时写入，文件名带 Pod UID（重建即换目录） |
| 软链接 | `/var/log/containers/*.log` → 上面的真实文件 | 统一命名 `<pod>_<ns>_<container>-.log`，方便按 Pod 找文件 |
| 轮转策略 | kubelet 默认 10Mi/文件 × 最多 5 份 | 单容器最多保留 ~50Mi 日志，超出最旧的被删——**K8s 不给日志做长期保存** |
| 日志格式（containerd） | `2026-09-08T09:00:00.123456789+08:00 stdout F 真实日志内容` | CRI 统一格式：时间戳 + 流（stdout/stderr）+ F（Full）标记 + 正文 |

三个推论（每个都是排障/面试要点）：
1. **Pod 删除 = 日志目录删除**。`kubectl logs` 能看的是容器当前存活期间 + 轮转内的日志；想"事后查案"必须把日志**抄走集中存**——这就是本项目存在的意义。
2. **采集软链必须同时能解析真实文件**。只挂 `/var/log/containers` 而不挂 `/var/log/pods`，软链解不开 → 静默采不到（见 9.2）。
3. **CRI 前缀必须解析**。不解析的话每行日志的时间戳都是"被采集的时刻"，正文前面还挂着一大串元信息——`cri` parser 就是为它准备的。

### 1.2 Loki 数据模型：只索引标签

**① 日志流（stream）是 Loki 的最小管理单元。**
一组标签组合完全相同的日志行 = 一条流。例如 `{namespace="default", pod="damo-app-xxx", container="app"}` 就是一条流。Loki 收到日志先按标签算出流的指纹，同一条流的日志顺序压缩存储。

**② 只索引标签，不索引正文。**

| 对比项 | Elasticsearch（全文索引） | Loki（标签索引） |
|--------|--------------------------|------------------|
| 索引内容 | 每个词都进倒排索引 | 只有标签组合 |
| 索引体积 | 常接近甚至超过原文 | 极小（标签组合有限） |
| 找内容的方式 | 查索引直接命中 | 标签圈定流 → 顺序扫描正文（需要管道过滤） |
| 写入成本 | 高（分词+建索引） | 低（压缩直写） |
| 适合 | 海量日志的复杂全文搜索 | 已知上下文（哪个服务/Pod）的运维检索 |

> ⚠️ **推论：标签就是索引，加标签要克制。** `pod` 名可以（组合数=Pod 数），但 Pod UID、request id、trace id 绝不能做标签——每种新组合都是一份索引，查询和内存都会被拖垮（9.3 有真实排错）。**正文过滤交给管道（`\|=`），不进标签**。

**③ 时间戳由谁定？** 由日志行自己的时间戳决定（`cri` parser 解析出来的才是真实时间）。`reject_old_samples` 会拒收时间戳太旧的行——所以节点时间同步很重要。

### 1.3 LogQL 与 PromQL 的血缘

Loki 是 Prometheus 团队的"日志兄弟"，查询语言刻意对齐：

| | PromQL（项目 C） | LogQL（项目 D） |
|---|---|---|
| 选择器 | `up{job="node-exporter"}` | `{namespace="monitoring", pod=~"prometheus-.*"}` |
| 管道处理 | 函数/聚合 | `\|=` 行过滤 → `\| json` 解析 → 函数聚合 |
| 范围函数 | `rate(x[5m])` | `count_over_time(x[5m])` / `bytes_over_time` |
| 聚合 | `sum by (pod) (...)` | 完全相同 |
| 心智模型 | 时间序列 = 一组标签 + 数值 | 日志流 = 一组标签 + 文本行；**标签选择部分语法同构** |

> 💡 面试话术："PromQL 的标签选择语法可以直接迁移到 LogQL，学过 Prometheus 的人 Loki 上手成本极低——这也是我们技术栈统一选 Grafana 系的原因。"

### 1.4 组件职责速记表

| 组件 | 形态 | 端口 | 核心职责 | 绝不负责 |
|------|------|------|---------|---------|
| Fluent Bit | DaemonSet（3 节点） | 2020 | tail 文件、解析打标、断点续采、推送 | 存日志、查日志 |
| Loki | Deployment ×1 | 3100 | 收日志、标签索引、压缩存储、保留清理、查询 API、规则评估 | 采集、画图 |
| Grafana | 项目 C 已有 | 30300 | 数据源、Explore 检索、仪表盘、跳转 | 存日志、采日志 |

### 1.5 阶段自测

- [ ] **能不看文档用自己的话讲完"图书馆故事"，把三个组件对号入座**（面试开场就这套语言）
- [ ] 能说出容器日志的真实路径、软链路径、轮转策略三件事
- [ ] 能解释"Loki 为什么便宜"——只索引标签不索引正文，并说清代价（检索靠管道扫描）
- [ ] 能说出什么是日志流（stream），为什么高基数字段不能做标签
- [ ] 能对照说出 LogQL 与 PromQL 的两个同构点和一个差异点
- [ ] 知道 CRI 日志格式长什么样，cri parser 解决什么问题

---

## 阶段 2：镜像准备

### 路线 A：节点直拉公共仓库（Fluent Bit 定版路线）

Fluent Bit 官方镜像是 Buildx 多架构清单，**containerd 2.x 原生支持 OCI index**——节点直拉 docker.io 会自动选 amd64，既绕开 registry:2 的清单格式不兼容（中转 push 会报 `does not provide any platform`），又省掉中转步骤：

```bash
# 什么都不用做——13-fluent-bit-ds.yaml 的 image 就是 fluent/fluent-bit:3.2.2
# 部署后若卡 ImagePullBackOff 再回退路线 B
```

### 路线 B：私有仓库中转（Loki 定版路线 + Fluent Bit 备选）

Loki 存储端走项目 B/C 铁律（中转进私有仓库，YAML 引用 `11.0.1.128:30000/...`）：

```bash
docker pull grafana/loki:3.3.2
docker tag grafana/loki:3.3.2 11.0.1.128:30000/grafana/loki:3.3.2
docker push 11.0.1.128:30000/grafana/loki:3.3.2

# 验证
curl -s http://11.0.1.128:30000/v2/grafana/loki/tags/list
# 预期输出：{"name":"grafana/loki","tags":["3.3.2"]}
```

**涉及镜像清单**：

| 镜像 | 获取方式 | 用途 |
|------|---------|------|
| grafana/loki:3.3.2 | 私有仓库中转 | 日志存储与查询 |
| fluent/fluent-bit:3.2.2 | 节点直拉公共仓库 | 日志采集（与 Loki 配套） |

> ⚠️ 版本说明：Loki 与采集器**大版本配套**即可（push API 稳定）。若直拉/中转都遇到 Buildx 清单问题（`does not provide any platform`），解法：`docker pull --platform linux/amd64 <镜像>` 强制单架构清单再中转（手册 9.5 有记录）。

---

## 阶段 3：Loki

### 3.0 组件工作原理（先懂再装）

**Loki 收到一条日志后发生了什么？** 内部是一条流水线：

| 环节 | 干什么 | 关键文件/目录 |
|------|--------|--------------|
| ① 接收（distributor） | 校验时间戳（reject_old）、按标签算流的指纹 | — |
| ② 预写（ingester） | 日志先进 **WAL**（预写日志）再进内存 buffer——进程崩了能回放恢复 | `/loki/wal` |
| ③ 落盘（flush） | buffer 攒够/超时后压缩成 chunk 写盘 | `/loki/chunks` |
| ④ 索引 | 每条流的标签组合记入 TSDB 索引（按天分文件） | `/loki/index_*` |
| ⑤ 清理（compactor） | 周期删除超过 retention_period 的块和索引 | `/loki/compactor` |

**单机 monolithic 模式**：Loki 支持拆成十几个微服务组件跑分布式，但单机把所有角色合在一个进程里——学习集群的标准选择，`common.ring.kvstore: inmemory` + `replication_factor: 1` 就是这个模式的标志配置。

> 💡 **面试点**："Loki 的 WAL 和 Prometheus 的 WAL 是同一个思想——先写日志再进内存，crash 后回放，最多丢几秒数据。"（对照项目 C 自测 Q11）

### 3.1 部署

```bash
sed -i 's/__NODE_NAME__/work2/g' 02-loki.yaml      # 先完成 0.4 占位符替换
kubectl apply -f 01-loki-config.yaml
kubectl apply -f 02-loki.yaml
kubectl -n monitoring rollout status deploy/loki --timeout=180s
```

**预期输出**：

```
configmap/loki-config created
persistentvolumeclaim/loki-data created
deployment.apps/loki created
service/loki created
deployment "loki" successfully rolled out
```

**对应文件 1：`01-loki-config.yaml`**（完整逐字段注释见文件，核心如下）

```yaml
data:
  loki.yaml: |
    auth_enabled: false          # 关闭多租户（单机）
    server:
      http_listen_port: 3100
    common:
      path_prefix: /loki
      storage:
        filesystem:
          chunks_directory: /loki/chunks
          rules_directory: /loki/rules
      replication_factor: 1
      ring:
        kvstore: {store: inmemory}          # monolithic 标志配置
    schema_config:
      configs:
        - from: "2024-01-01"
          store: tsdb                       # 3.x 推荐索引引擎
          object_store: filesystem
          schema: v13
    ingester:
      wal: {enabled: true, dir: /loki/wal}  # 崩溃恢复
    limits_config:
      retention_period: 168h                   # 保留 7 天
      reject_old_samples: true
      reject_old_samples_max_age: 168h
      max_query_series: 500                    # OOM 防线
    compactor:
      working_directory: /loki/compactor
      retention_enabled: true                  # ★ 保留策略执行开关
      delete_request_store: filesystem         # ★ 3.x 硬校验：缺它起不来
    ruler:
      enable_api: true
      alertmanager_url: http://alertmanager:9093   # 复用项目 C
      poll_interval: 1m
```

**对应文件 2：`02-loki.yaml`**（PVC + Deployment + Service）

```yaml
# 关键段摘录（完整见文件）
spec:
  replicas: 1
  strategy:
    type: Recreate                 # ★ RWO 卷防双挂（项目 C 同款）
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: __NODE_NAME__   # ★ PV 绑节点，Pod 必须钉死
      containers:
        - name: loki
          image: 11.0.1.128:30000/grafana/loki:3.3.2
          args:
            - -config.file=/etc/loki/loki.yaml   # ★ 显式绝对路径（漏写=CrashLoop）
          readinessProbe:                        # WAL 回放完成前不接流量
            httpGet: {path: /ready, port: 3100}
            initialDelaySeconds: 30
            failureThreshold: 12
          livenessProbe:
            tcpSocket: {port: 3100}              # 存活探针别用 /ready（回放期误杀）
          volumeMounts:
            - {name: config, mountPath: /etc/loki}
            - {name: data, mountPath: /loki}
```

### 3.2 YAML 关键设计逐条讲（每条都是面试点）

| # | 设计 | 为什么（面试语言） |
|---|------|------------------|
| 1 | `strategy: Recreate` | local-path PV 是 RWO，RollingUpdate 会先起新 Pod 抢挂旧卷 → 卡 Pending。日志可容忍秒级断流，换部署稳定性 |
| 2 | `nodeSelector` 固定节点 | local-path 的 PV 绑定在首次消费节点本地，Pod 漂移 = 数据"丢失"（项目 C 5.3 同款） |
| 3 | `-config.file` 显式绝对路径 | Loki 默认按工作目录解析相对路径；漏写 → `open loki.yaml: no such file` CrashLoop。项目 C Prometheus 第一轮故障同源 |
| 4 | readiness `/ready` + liveness TCP | 首次启动要回放 WAL，`/ready` 可能长时间未就绪——readiness 容忍 2 分钟；liveness 用 TCP 避免回放期被误杀 |
| 5 | `retention` 三件套 | `retention_period` 定义保留期、`retention_enabled` 打开执行开关、`delete_request_store` 是 3.x 硬校验——缺一：要么日志只进不出撑爆磁盘，要么启动直接报错 |
| 6 | `limits_config` 双防线 | `reject_old_samples` 防时间错乱打爆存储；`max_query_series` 防巨型查询打爆内存 |
| 7 | ConfigMap 整目录挂载 | 改配置 → restart 即生效；项目 C 曾因 subPath 混用踩过"改了不生效"的坑 |

### 3.3 验证（三步，全部通过才算过）

```bash
# ① Pod 状态：Running 且 READY 1/1
kubectl -n monitoring get pods -l app=loki -o wide
# 预期：loki-xxx 1/1 Running <你的固定节点>

# ② 启动自检：日志末尾出现 started、无 error
kubectl -n monitoring logs deploy/loki --tail=20
# 预期末尾：msg="Loki started"

# ③ API 可达（port-forward 或 ClusterIP 直连均可）
curl -s 'http://10.106.154.53:3100/ready'        # 预期：ready
curl -s 'http://10.106.154.53:3100/loki/api/v1/labels'
# 预期：{"status":"success","data":["__name__"]}  ← 尚无业务标签，正常
```

> 💡 **考点**：Loki 容器是精简镜像（无 shell/cat），验证一律走 HTTP API（port-forward / ClusterIP 直连 / 工具箱 Pod），不要试图 exec 进容器。

---

## 阶段 4：Fluent Bit 采集器★本项目核心

### 4.0 组件工作原理（先懂再装）

Fluent Bit 每个采集周期做五件事（对照 0.2 数据流表）：

| # | 步骤 | 机制 | 失败表现 |
|---|------|------|---------|
| ① 发现文件 | glob 匹配 `/var/log/containers/*.log`（tail 插件内置 inotify 监听增删） | 纯文件系统 | 静默无日志（查挂载） |
| ② 解析行 | `cri` parser 拆 CRI 前缀、还原真实时间戳 | pipeline parser | 时间线全错 |
| ③ 提取标签 | **正则解析文件路径**：`<pod>_<ns>_<container>-<id>.log` → 三个标签字段 | parser filter + Path_Key | 标签缺失（本方案零 API 依赖，不存在此问题的网络变体） |
| ④ 断点续采 | DB 文件记 inode+offset | positions | 重采/丢日志 |
| ⑤ 推送 | HTTP push → `http://loki.monitoring.svc.cluster.local:3100/...` | loki output | `no upstream connections available`（★实战踩坑，见 4.2 设计#1） |

> 💡 **与常见教程的关键差异（本方案实战定版）**：教程普遍用 kubernetes filter 调 API Server 补全元数据——**该 filter 在本环境静默失效**（与 promtail 的 kubernetes_sd 同源，均为 grafana 系组件对 API 的连接兼容问题，复盘见附录 C）。定版方案**零 API 依赖**：文件名自带元数据，纯正则提取，稳定性反而更高。代价：拿不到 Pod 的 `app` 标签与节点名——核心检索场景不受影响。

### 4.1 部署

```bash
kubectl apply -f 12-fluent-bit-rbac.yaml     # SA/RBAC（零 API 方案下非必需，保留作升级余量）
kubectl apply -f 11-fluent-bit-config.yaml
kubectl apply -f 13-fluent-bit-ds.yaml
kubectl -n monitoring rollout status ds/fluent-bit --timeout=180s
```

**预期输出**：

```
serviceaccount/fluent-bit created
clusterrole.rbac.authorization.k8s.io/fluent-bit created
clusterrolebinding.rbac.authorization.k8s.io/fluent-bit created
configmap/fluent-bit-config created
daemonset.apps/fluent-bit created
daemonset "fluent-bit" successfully rolled out
```

**对应文件 1：`11-fluent-bit-config.yaml`**（灵魂文件，完整逐字段注释见文件）

```yaml
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Daemon        Off
        Log_Level     info
        Parsers_File  /fluent-bit/etc/parsers.conf
        network.dns.mode   TCP     # ★★ 实战修复：弃用内置 async DNS 解析器
                                   #    （对 Service DNS 兼容差，曾致 loki output
                                   #    "no upstream connections available" 全军覆没）
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Path_Key          source_path      # ★ 文件路径进记录（标签提取的原料）
        Parser            cri              # ★ 拆 CRI 前缀、还原真实时间戳
        DB                /var/log/flb_kube.db   # ★ 断点书签（hostPath 持久化）
        Mem_Buf_Limit     10MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name          parser               # ★ 零 API 标签提取：正则解析文件路径
        Match         kube.*
        Key_Name      source_path
        Parser        k8s_filename
        Reserve_Data  On                   # 解析失败保留原始记录（不丢日志）
        Preserve_Key  Off

    [OUTPUT]
        Name        loki
        Match       kube.*
        Host        loki.monitoring.svc.cluster.local   # ★ 若 DNS 仍有问题可换 ClusterIP
        Port        3100
        Labels      namespace=$namespace, pod=$pod, container=$container
        Line_Format json
```

```yaml
  parsers.conf: |
    # CRI 日志格式（containerd）：<时间> stdout F <正文>
    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

    # 日志文件名：<pod>_<namespace>_<container>-<64位容器ID>.log
    # container 用贪婪回退匹配（容器名可含连字符：kube-apiserver / wait-for-mysql）
    [PARSER]
        Name        k8s_filename
        Format      regex
        Regex       ^/var/log/containers/(?<pod>[^_]+)_(?<namespace>[^_]+)_(?<container>.+)-[a-z0-9]{64}\.log$
```

**对应文件 2：`13-fluent-bit-ds.yaml`**（核心段摘录）

```yaml
spec:
  template:
    spec:
      serviceAccountName: fluent-bit   # 零 API 方案下非必需，保留作升级余量
      tolerations:                     # master 日志也要采
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.2.2     # 节点直拉公共镜像（containerd 2.x 支持 OCI index）
          ports:
            - {name: http, containerPort: 2020}
          readinessProbe: {httpGet: {path: /, port: 2020}, initialDelaySeconds: 10}
          volumeMounts:
            - {name: config, mountPath: /fluent-bit/etc}
            - {name: varlogcontainers, mountPath: /var/log/containers, readOnly: true}
            - {name: varlogpods, mountPath: /var/log/pods, readOnly: true}   # ★ 缺它软链解不开
            - {name: positions, mountPath: /var/log/fluent-bit}
      volumes:
        - {name: config, configMap: {name: fluent-bit-config}}
        - {name: varlogcontainers, hostPath: {path: /var/log/containers}}
        - {name: varlogpods, hostPath: {path: /var/log/pods}}
        - {name: positions, hostPath: {path: /var/log/fluent-bit, type: DirectoryOrCreate}}
```

### 4.2 YAML 关键设计逐条讲

| # | 设计 | 为什么（面试语言） |
|---|------|------------------|
| 1 | `network.dns.mode TCP`（★实战第一坑） | fluent-bit 内置 **async DNS 解析器**对 `ndots:5` 的 Service DNS 兼容差——本环境实测 loki output 报 `no upstream connections available`、kubernetes filter 查 API 静默失败，**全部指向解析层**；改 TCP（libc 同步解析）后全部恢复。一个参数救活整个组件 |
| 2 | 标签从文件路径正则提取（★实战第二坑） | kubernetes filter 的 API enrich 在本环境**静默失败**（放行日志但不写元数据）；而文件名 `<pod>_<ns>_<container>-<id>.log` 自带全部元数据——`Path_Key` + parser filter 纯本地正则提取，**零 API 依赖**，不查 API、不需要 RBAC、不受 API 兼容性影响 |
| 3 | 双 hostPath（containers + pods） | 前者全是软链，真实文件在后者的 Pod UID 目录下。只挂前者 → 软链解不开 → 静默零采集（与 promtail 同款坑） |
| 4 | `DB` 位点文件 hostPath 持久化 | 相当于"书签"：Pod 重建后按 inode+offset 续采；丢了 → 全量重采 → Loki 拒收过旧时间戳（`entry too far behind`） |
| 5 | `cri` parser | containerd 的 CRI 格式（`时间戳 stdout F 正文`）必须拆包：不拆则时间线全错、正文带前缀，排障不可用 |
| 6 | `Skip_Long_Lines On` | 超长行（如堆栈大段输出）跳过防止内存缓冲爆掉；配合 `Mem_Buf_Limit 10MB` 双保险 |
| 7 | tolerations 容忍 control-plane | master 默认 NoSchedule 污点；不容忍则 master 日志全部漏采（DaemonSet 只剩 2 个节点） |
| 8 | 直拉公共镜像 | fluent-bit 官方镜像为 Buildx 多架构清单，中转 push 到 registry:2 会报格式不兼容；containerd 2.x 直拉原生支持 OCI index——**选直拉反而是兼容性最优解** |

### 4.3 验证

```bash
# ① Pod 分布：3 节点各 1 个，全部 1/1 Running
kubectl -n monitoring get pods -l app=fluent-bit -o wide

# ② 无异常日志（重点：不应出现 no upstream / failed to flush）
kubectl -n monitoring logs -l app=fluent-bit --prefix --tail=20 | grep -iE "error|warn" | tail -5
# 预期：仅 inotify_fs_add 的 info 信息（文件监听注册），无 error/warn

# ③ 制造可检索的测试日志
kubectl -n monitoring run fb-test --image=busybox:1.36 --restart=Never \
  -- /bin/sh -c 'while true; do echo "fb-verification-line $(date)"; sleep 5; done'
sleep 30    # 等 Flush 周期 + 推送

# ④ Loki 端到端验证（ClusterIP 直连，不过 port-forward）
curl -s -G 'http://10.106.154.53:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={pod=~"fb-test.*"}' \
  --data-urlencode 'limit=1' | head -c 600
# 预期：命中流，且流标签包含 pod="fb-test"、namespace="monitoring"、container="fb-test"
```

> 💡 **考点（双重静默失败的教训）**：本阶段排障中，fluent-bit 的两个故障（连不上 Loki、filter 不打标签）**全部是静默失败**——`proc_records` 计数在涨造成"在工作"的假象，实际全堆在 retry 队列。**通用对策：采集器验证必须"端到端造数据→查询回收"，绝不以进程状态和中间指标为准**。

---

## 阶段 5：Grafana 接入与日志检索

### 5.1 添加 Loki 数据源

复用项目 C 的 Grafana（浏览器 `http://11.0.1.128:30300`）：

1. 左侧菜单 **Connections → Data sources → Add data source**；
2. 类型选 **Loki**；
3. **URL** 填 `http://loki.monitoring.svc.cluster.local:3100`（跨 ns 场景写完整 FQDN；Grafana 与 Loki 同在 monitoring ns 时可写 `http://loki:3100` 短名）；
4. 点 **Save & test** → 预期绿色 `Successfully queried the Loki API.`

### 5.2 Explore 界面导览

| 界面元素 | 位置 | 作用 | 本项目用法 |
|---------|------|------|-----------|
| 数据源切换 | 左上角下拉 | 切 Prometheus / Loki | 查日志前先切 **Loki** |
| Label browser | 查询框旁按钮 | 点选标签和值，自动生成选择器 | 记不住语法时用它拼 |
| 查询输入框 | 页面中央 | 输入 LogQL | 冒号前必须有标签选择器 |
| 时间选择器 | 右上角 | 限定查询窗口 | 排障常用 Last 1 hour |
| **Live** 按钮 | 右上角 | 实时流式跟随（tail -f 体验） | 验证日志是否实时进来 |
| 日志行标签徽章 | 每行左侧 | 点徽章 = 加进选择器过滤 | 点 `pod=xxx` 快速收窄 |
| 日志行展开 | 每行右侧箭头 | 看完整行 + 原始 JSON | 看堆栈全文 |

### 5.3 LogQL 四级进阶

**L1 标签选择器**（先看流）：

```
{namespace="monitoring"}
{pod=~"prometheus-.*"}        # = 精确 / =~ 正则 / != 反向
```

**L2 行过滤**（在流内找内容）：

```
{namespace="monitoring"} |= "error"                # 包含 error（区分大小写）
{namespace="monitoring"} |~ "(?i)error|exception"  # 正则 + 不区分大小写
{pod=~"coredns-.*"} |= "timeout" != "health"       # 多管道叠加
```

**L3 解析**（结构化字段）：

```
{app="damo-app"} | json | level="error"     # JSON 日志拆字段后按字段过滤
{app="damo-app"} | logfmt | status="500"    # logfmt 键值对同理
```

**L4 日志转指标**（面试精华：把日志行数当指标算）：

```
# default 命名空间各 Pod 5 分钟错误日志行数
sum by (pod) (count_over_time({namespace="default"} |= "error" [5m]))

# 刷日志最凶的 Top5 Pod（字节量）
topk(5, sum by (pod) (bytes_over_time({namespace="default"} [5m])))

# 错误日志速率折线（配 Dashboard）
sum(count_over_time({namespace="monitoring"} |= "error" [1m]))
```

**常用查询速查表**：

| 需求 | 查询 |
|------|------|
| 某 Pod 全部日志 | `{pod="prometheus-0"}` |
| 某命名空间所有 error | `{namespace="default"} \|= "ERROR"` |
| 排除健康检查噪音 | `{namespace="default"} != "/healthz" != "/ready"` |
| CoreDNS 超时回查（项目 C 复盘） | `{pod=~"coredns-.*"} \|= "i/o timeout"` |
| 错误日志趋势（1 分钟粒度） | `sum(count_over_time({namespace="default"} \|= "error" [1m]))` |

### 5.4 端到端验收：亲手造一条日志走完全程

```bash
kubectl -n monitoring run fb-test --image=busybox:1.36 --restart=Never \
  -- /bin/sh -c 'while true; do echo "fb-verification-line $(date)"; sleep 5; done'
```

Grafana Explore（Last 15 minutes）执行 `{pod="fb-test"}` → 预期每 5 秒一条、Live 模式实时滚动。
验证完清理：`kubectl delete pod fb-test`。

### 5.5 日志仪表盘（Dashboard）

**① 建变量**：Dashboard settings → Variables → New variable：
- 变量 1：Name=`namespace`，Type=Query，数据源 Loki，**Label values**，Label=`namespace`；
- 变量 2：Name=`pod`，Label=`pod`，勾选 **Include All**。

**② 日志面板**：Add → Visualization → 面板类型 **Logs**，查询 `{namespace="$namespace", pod=~"$pod"}`。

**③ 错误率面板**（Time series）：

```
sum by (pod) (count_over_time({namespace="$namespace", pod=~"$pod"} |= "error" [5m]))
```

**Save dashboard**（命名 K8s-Logs-D）——与项目 C 的指标大盘并列，"指标 + 日志"一页排障。

---

## 阶段 6：日志保留与磁盘管理

### 6.1 确认保留机制在工作

```bash
kubectl -n monitoring logs deploy/loki | grep -i compactor | tail -5
# 预期：compactor started 等启动日志，无 error

du -sh /var/local-path-provisioner/*/ 2>/dev/null | sort -h | tail -5
# 预期：能看到 loki-data 对应目录及大小（随日志量增长）
```

### 6.2 磁盘水位与应对

| 信号 | 动作 |
|------|------|
| PVC 目录增长缓慢、远小于 10Gi | 正常，不动 |
| 磁盘水位告警（项目 C 的磁盘规则触发） | ① 确认 compactor 在跑；② `retention_period` 168h→72h：改 01 → apply → `rollout restart deploy/loki`；③ 排查刷日志的 Pod（5.3 的 topk bytes_over_time） |
| PVC 写满 | ⚠️ local-path 不支持在线扩容：缩 retention 让 compactor 清理；紧急情况重建 PVC（**丢全部日志**，指标不受影响） |

> 💡 **设计取舍（面试可讲）**：日志 7 天 vs 指标 15 天（项目 C）——日志单条体积大、检索价值随时间衰减快，保留期天然更短；排障通常只回看数小时到数天。

### 6.3 日常维护命令

```bash
kubectl -n monitoring get pods -l 'app in (loki,fluent-bit)' -o wide   # 组件状态
kubectl -n monitoring logs deploy/loki --tail=20                       # Loki 自检
kubectl -n monitoring logs -l app=fluent-bit --tail=10 --prefix        # 各节点采集器
# 配置变更标准三连（对 01/11 均适用，fluent-bit 不支持热重载）：
kubectl apply -f <文件> && kubectl -n monitoring rollout restart deploy/loki ds/fluent-bit
```

---

## 阶段 7：日志告警

### 7.0 原理：ruler 是什么，链路怎么走

Loki 内置 **ruler** 组件：周期执行 LogQL 表达式，把结果当告警状态机管理（与 Prometheus 规则引擎同语义：正常 → pending（for 计时）→ firing → resolved），然后把告警**推给 Alertmanager**——正是项目 C 已经搭好的那个。

```
Loki ruler（每 1m 评估 09 的规则）
   │ 命中阈值且 for 满足
   ▼
Alertmanager（项目 C，:9093）── 按 severity=warning 子路由 ──► QQ 邮箱 📧
```

> 与项目 C 的分工对照：Prometheus 评估**指标**告警、Loki ruler 评估**日志**告警，Alertmanager 只管通知策略——三个角色各司其职，一个通知出口。

### 7.1 前置确认（邮件链路健在）

```bash
kubectl -n monitoring get pods -l app=alertmanager    # Running
kubectl -n monitoring logs deploy/alertmanager --tail=10   # 无 notify error
```

### 7.2 部署告警规则

```bash
kubectl apply -f 09-loki-rules.yaml
```

**规则语义**（`09-loki-rules.yaml`）：`default` 命名空间任意 Pod，5 分钟窗口内命中 `error|exception`（不区分大小写）的日志行数超过 20 条，持续 2 分钟 → 触发 `AppErrorLogHigh`（severity=warning）。

**把规则挂进 Loki（⚠️ fake 租户坑）**：修改 `02-loki.yaml` 的 Deployment，追加挂载：

```yaml
          volumeMounts:
            # ... 原有 config、data 保持不动，追加：
            - name: rules
              mountPath: /loki/rules/fake     # ★ 必须带 fake 子目录！
      volumes:
        # ... 原有 config、data 保持不动，追加：
        - name: rules
          configMap:
            name: loki-rules
```

生效：

```bash
kubectl apply -f 02-loki.yaml
kubectl -n monitoring rollout restart deploy/loki     # 挂载变更需重启生效
```

> ⚠️ **两个必踩点提前拆雷**：
> ① `auth_enabled: false` 时单租户固定叫 **fake**，Loki 只按 `<rules_directory>/<tenant>/*.yml` 加载——挂到 `/loki/rules` 根目录则规则永远不加载，**且不报错**（静默失败，面试级细节）；
> ② 本项目用**整目录挂载**（非 subPath），`rollout restart` 即可让变更生效；`poll_interval: 1m` 会自动重读规则内容。

### 7.3 Alertmanager 放行

编辑项目 C 的 `07-alertmanager-config.yaml`，在 `route:` 下追加子路由（放在既有节点告警子路由之后）：

```yaml
      routes:
        # ... 项目 C 已有的节点/Pod 告警子路由保持不动，追加：
        - matchers:
            - severity = "warning"
          receiver: qq-mail     # 与既有邮件 receiver 名一致
```

```bash
kubectl apply -f 07-alertmanager-config.yaml
kubectl -n monitoring rollout restart deploy/alertmanager
```

### 7.4 全链路验证（四级证据链，每级都要亲眼看到）

```bash
# 制造高频错误日志
kubectl delete pod fb-test --force --grace-period=0 2>/dev/null
kubectl run fb-test --image=busybox:1.36 --restart=Never \
  -- /bin/sh -c 'i=0; while true; do echo "fake ERROR line $i"; i=$((i+1)); sleep 2; done'
```

| # | 环节 | 验证命令 | 预期 |
|---|------|---------|------|
| ① | ruler 已加载规则 | `kubectl -n monitoring logs deploy/loki \| grep -i ruler \| tail -5` | 规则文件加载日志 |
| ② | 告警进入 firing | 工具箱内 `curl -s http://loki:3100/loki/api/v1/alerts` | alerts 含 AppErrorLogHigh，state=firing |
| ③ | Alertmanager 收到 | `kubectl -n monitoring logs deploy/alertmanager --tail=20` | PENDING → FIRING |
| ④ | 邮件真实收到 | 查 QQ 邮箱 | 告警邮件，标题含 AppErrorLogHigh |

收尾：`kubectl delete pod fb-test` → 2 分钟后 expr 不再满足 → Alertmanager 发 Resolved 邮件（与项目 C 同款闭环体验）。

---

## 阶段 8：全链路验证与面试演示

| # | 演示动作 | 命令/操作 | 预期结果 |
|---|---------|----------|---------|
| 1 | 组件健康 | `kubectl -n monitoring get pods -l 'app in (loki,fluent-bit)' -o wide` | loki 1/1，fluent-bit 3/3 跨 3 节点 |
| 2 | 造日志 | `kubectl run fb-test ...`（5.4） | Pod Running |
| 3 | 实时检索 | Grafana Explore：`{pod="fb-test"}` + Live | 日志实时滚动 |
| 4 | 标签检索 | 点日志行 pod 徽章收窄；切 `{namespace="monitoring"}` | 过滤生效 |
| 5 | 指标化 | `sum by (pod) (count_over_time({pod="fb-test"} [5m]))` | 折线随日志增长 |
| 6 | 仪表盘 | 打开 K8s-Logs-D，切 namespace 下拉 | 面板联动 |
| 7 | 日志告警 | 阶段 7 的 fb-test ERROR 版 | QQ 收 FIRING + RESOLVED 两封 |
| 8 | 排障叙事 | 口述：指标告警（项目 C）定位时间窗 → 日志检索（项目 D）确认根因 | 完整可观测性故事线 |

**一键自检清单**：

```bash
kubectl -n monitoring get pods -l 'app in (loki,fluent-bit)' -o wide
kubectl -n monitoring logs deploy/loki --tail=5
kubectl -n monitoring get pvc loki-data        # Bound
kubectl -n monitoring get clusterrole fluent-bit   # 存在
```

---

## 9. 故障速查表

### 9.1 部署类（Loki）

| 现象 | 原因 | 解决 |
|------|------|------|
| 镜像拉取超时/404 | 没中转或 tag 错 | 回阶段 2 路线 B；`crictl pull 11.0.1.128:30000/grafana/loki:3.3.2` 验证 |
| Loki 反复重启：`open loki.yaml: no such file` | args 漏 `-config.file` 或相对路径 | 02 补显式绝对路径（项目 C 同款坑） |
| Loki 起不来：`delete_request_store must be specified` | 开了 `retention_enabled` 没配 `delete_request_store` | 01 的 compactor 三行配齐（3.x 硬校验） |
| 新 Pod 卡 Pending 抢卷 | strategy 还是 RollingUpdate | 02 改 Recreate（RWO 卷不能双挂） |
| Loki OOMKilled | 大查询 / 日志洪峰 | limits 提到 1.5Gi；根治靠 9.3 查询治理 |
| ConfigMap 改了行为没变 | Loki/Fluent Bit 不自动重读 | `rollout restart deploy/loki` / `ds/fluent-bit` |

### 9.2 采集类（Fluent Bit，按实战发生顺序）

| 现象 | 原因 | 解决 |
|------|------|------|
| **【实战案例 1】fluent-bit 1/1 Running、proc_records 在涨，但 Loki 查不到任何流** | loki output 报 `no upstream connections available`——**内置 async DNS 解析器**解析 `*.svc.cluster.local` 失败，连接从未建立（proc_records 只是"进插件"，不是"推成功"） | 11 的 `[SERVICE]` 加 `network.dns.mode TCP`（走 libc 同步解析）→ restart。**fluent-bit 部署到 K8s 后此参数应作为默认配置** |
| **【实战案例 2】推送恢复后，流标签只有 job/service_name，没有 namespace/pod** | kubernetes filter 的 API enrich 静默失败（与 promtail SD 同源）——定版方案**弃用 kubernetes filter**，改零 API 正则提标签 | 确认 11 用的是 parser filter + `Path_Key` 终局版配置 |
| 容器内连 `cat`/`getent` 都没有，无法 exec 取证 | fluent-bit 官方镜像为极简镜像 | 所有验证走 HTTP 端口（2020 的 / 与 /api/v1/metrics）+ Loki 侧 API |
| 只有部分节点有日志 | DaemonSet 数量不足 / tolerations 缺失 | `get pods -o wide` 对照节点数 |
| 日志时间戳不对（都是采集时刻） | 漏 `cri` parser | 11 补 pipeline 的 cri parser |
| 正文带 `stdout F` 前缀 | 同上（cri 未解析） | 同上 |
| Loki 报 `entry too far behind` | DB 位点丢失后全量重采旧日志 | 位点 hostPath 稳定性；临时调大 `reject_old_samples_max_age` |
| 同一段日志大量重复 | 同上（重采） | 保住 DB 文件即根治 |
| 某新 Pod 日志迟迟不出现 | Refresh_Interval 刷新周期未到 / 该 Pod 非 Running | 等 ~10s；`kubectl get pod --show-labels` 看相位 |

### 9.3 查询类（Loki）

| 现象 | 原因 | 解决 |
|------|------|------|
| 数据源 Save & test 失败 | URL 错 / Loki 没起 / 跨 ns 用了短名 | `http://loki.monitoring.svc.cluster.local:3100`（跨 ns 完整 FQDN）或同 ns 短名 `http://loki:3100` |
| 查询报 429 / 超时 | 标签基数爆炸 / 查询过大 | 检查标签是否含高基数字段；缩小时间窗；`max_query_series` 兜底 |
| 查到日志但时间是"未来/过去" | 节点时区/时间不同步 | 节点 `date` 核对；systemd-timesyncd 确认 active |
| `/label?since=3600` 报 not a valid duration | Loki 要求带时间单位 | `since=1h` / `since=3600s`（裸数字无效） |
| Grafana 日志面板空白 | 时间范围没数据 / 变量代入后选择器为空 | 换 Last 1 hour；检查变量默认值 |

### 9.4 告警类（阶段 7）

| 现象 | 原因 | 解决 |
|------|------|------|
| `/loki/api/v1/alerts` 空 | 规则没被加载 | 挂载路径必须是 `/loki/rules/fake/`（fake 租户！）；restart deploy/loki |
| 规则加载了但不触发 | expr 无返回 / for 未满 / 阈值过高 | Explore 手跑 expr 确认有数据；等 for；调阈值 |
| firing 但 Alertmanager 没收到 | `alertmanager_url` 错 | 01 核对 `http://alertmanager:9093`（同 ns 短名） |
| AM 收到但没邮件 | 子路由没匹配 / receiver 名错 | 核对 07 的 matchers 与 receiver 名（对照项目 C） |
| 告警风暴 | expr 太宽 / repeat 太短 | expr 收窄到具体 namespace；Silence 止血 |

### 9.5 国内网络与镜像专项（项目 B/C 同款坑的 D 版）

| 现象 | 解决 |
|------|------|
| docker.io 直拉超时 | Fluent Bit 走节点直拉（containerd 2.x 支持 OCI index）；Loki 走中转；master 拉不动换 `docker.m.daocloud.io/...` |
| **中转 push 报 `image ... was found but does not provide any platform`** | Buildx 多架构清单 + registry:2 不兼容——`docker pull --platform linux/amd64 <镜像>` 强制单架构再 tag/push（fluent-bit 实战踩中，故定版直拉路线） |
| YAML 里误写 `grafana/loki` 原名 | Loki 镜像写死 `11.0.1.128:30000/grafana/loki:3.3.2`；`grep -rn "image:" manifests/` 核查 |
| worker 拉私有仓库报 x509 | containerd hosts.toml 未信任 30000（项目 B 已配，核对是否还在） |

### 9.6 通用排障方法论（所有阶段通用，面试可讲）

```bash
# 第一层：资源状态对不对
kubectl get pods -n monitoring -o wide            # RUNNING + READY + 节点分布
kubectl get pvc -n monitoring                     # loki-data 是否 Bound

# 第二层：组件自己怎么说（★关键技巧：过滤噪音看真错误）
kubectl logs deploy/loki -n monitoring --tail=50
kubectl logs -l app=fluent-bit -n monitoring --prefix --tail=30 \
  | grep -viE "inotify_fs|token updated"          # 剔除周期性噪音，抓真错误

# 第三层：链路逐段验证（数据源头往下游，每段亲眼见数据）
#  Fluent Bit 侧：自带指标 :2020/api/v1/metrics（input/filter/output 三段计数）
#  Loki 侧：      /ready、/loki/api/v1/labels、distributor_lines_received_total
#  查询侧：       Grafana Save & test → Explore

# 原则：链路上"最后一步没数据"时，八成问题在前面某一段——
# 本项目实测：采集器的 output 计数会骗人（进插件≠推成功），
# 必须"端到端造数据→Loki 端回收"闭环验证，且用 Loki 自己的
# distributor 接收指标做 ground truth（与查询时间窗无关）。
```

---

## 10. 自测题

**Q1：Loki 为什么比 Elasticsearch 省资源？代价是什么？**
A1：只对标签建索引、不索引正文——索引体积从"接近原文"缩到"标签组合级别"，写入直写压缩块。代价：找内容时先用标签圈定日志流、再顺序扫描正文（管道过滤），复杂全文检索不如 ES；运维场景通常已知服务/Pod 上下文，扫描成本可接受。

**Q2：什么是日志流（stream）？标签基数为什么是命门？**
A2：标签组合完全相同的日志行构成一条流，Loki 以流为最小单元建索引和压缩存储。每种新标签组合都是一份新索引与内存开销——UID/时间戳/trace id 做标签会让组合数随请求量无限增长。标签只取 pod/namespace/container/node 这类低基数字段。

**Q3：为什么本项目从日志文件名直接正则提取标签，而不用 kubernetes filter 查 API？**
A3：实战复盘——本环境（K8s 1.35）中 promtail 的 kubernetes_sd 与 fluent-bit 的 kubernetes filter **先后以完全相同的方式静默失效**（放行日志但不产出元数据，无任何报错）。文件名 `<pod>_<ns>_<container>-<id>.log` 自带全部所需元数据，纯正则零 API 依赖，稳定性反超。教训：**grafana 系组件对 API 的连接兼容性问题会以"静默失败"形态出现，能不依赖就不依赖**。

**Q4：为什么必须同时挂 `/var/log/containers` 和 `/var/log/pods` 两个 hostPath？**
A4：前者全是软链接，真实文件在后者目录下（kubelet 按 Pod UID 组织）。只挂前者，软链解析不到目标文件，采集器静默零采集。

**Q5：DB 位点文件是干什么的？丢了会怎样？**
A5：记录每个日志文件读到的位置（inode+offset），相当于"书签"。持久化（hostPath）后 Pod 重建可断点续采；丢了则从文件头重新采集——Loki 拒收过旧时间戳（`entry too far behind`），且重复行污染指标统计。

**Q6：cri parser 解决什么问题？**
A6：containerd 按 CRI 格式写日志（`时间戳 stdout F 正文`）。cri parser 拆出真实时间戳（替换掉"被采集时刻"）、识别 stdout/stderr、剥前缀留正文。不加它：Grafana 里时间线全错、每行带元信息前缀，排障基本不可用。

**Q7：Loki 的 retention 为什么是"三件套"？缺一个分别会怎样？**
A7：`limits_config.retention_period` 定义保留期；`compactor.retention_enabled` 是执行开关（缺了 → 日志只进不出，磁盘写满）；`delete_request_store` 是 3.x 硬校验（缺了 → Loki 启动直接失败）。

**Q8：Loki 单副本为什么必须 Recreate + nodeSelector 固定节点？**
A8：local-path 的 PV 是 ReadWriteOnce 且绑定在首次消费节点本地：RollingUpdate 会先起新 Pod 抢挂旧卷（Pending 卡死）；Pod 漂移到其他节点则 PV 挂不上（数据"丢失"）。与项目 C 的 Prometheus 完全同源。

**Q9：日志告警（ruler）与指标告警（Prometheus rules）的关系？**
A9：语义同构（expr → pending(for) → firing → resolved），数据源不同：Prometheus 评估数值指标，Loki ruler 评估 LogQL（先把日志行聚合成数值再判阈值，如 count_over_time > N）。两者都推给同一个 Alertmanager，通知策略集中管理。

**Q10：为什么 ruler 规则必须挂到 `/loki/rules/fake/`？**
A10：`auth_enabled: false` 时单租户固定标识为 `fake`，Loki 只按 `<rules_directory>/<tenant>/<文件名>` 组织规则加载。挂到根目录规则永远不加载且不报错——典型静默失败。

**Q11：日志采集链路排障的标准顺序是什么？**
A11：从数据源头往下游逐段验证：① fluent-bit Pod 状态与分布 → ② 软链可解析（容器内 ls -l）→ ③ fluent-bit 自带指标 `:2020/api/v1/metrics` 的 input/output 计数 → ④ **Loki 侧 `distributor_lines_received_total` 是否在增长**（ground truth，不受查询时间窗影响）→ ⑤ query_range 能查到行 → ⑥ Grafana 数据源 Save & test。**特别注意：output 的 proc_records ≠ 推送成功**（本项目实战：1048 条堆在 retry 队列，errors 恒为 0）。

**Q12（进阶）：生产上你会怎么改这套部署？**
A12：① 采集器演进到 Grafana Alloy（Promtail 官方继任者，配置统一、支持 OTel）；② Loki 换 Simple Scalable（读写分离）+ 对象存储（S3/MinIO）；③ 保留策略按合规分级（error 类 30 天、info 类 7 天）；④ 部署事件日志与告警关联；⑤ 大盘即代码（Dashboard JSON 进 Git，接项目 B 的流水线）。

---

## 11. 面试要点与简历写法

**简历一句话（STAR）**：
> 在 3 节点 K8s 集群上基于 Fluent Bit + Loki + Grafana 从 0 搭建集中式日志系统：Fluent Bit DaemonSet 通过 tail + CRI 解析采集全集群容器日志，以文件路径正则零 API 依赖提取 pod/namespace/container 低基数标签（规避 grafana 系组件 API 兼容性静默失效问题），Loki 单机模式 TSDB 索引 + 文件系统块存储实现 7 天保留与 compactor 自动清理，Grafana 统一检索入口构建日志面板与 Pod 级错误速率监控，并打通 Loki ruler → Alertmanager → QQ 邮箱的日志告警链路，完成"指标定位时间窗 → 日志确认根因"的可观测性闭环。

**高频追问与你的弹药**：自测题 Q1-Q12 全覆盖；**王牌叙事（附录 C 完整复盘）**——promtail kubernetes_sd 静默挂死 → 六层排除链（配置/权限/文件/网络/双版本/双连接方式）→ static 对照实验二分定位 → 换 Fluent Bit 又踩 async DNS 坑 → 零 API 标签方案终局：**"四层全对仍故障时，用对照实验二分问题空间，果断换等价组件而非无限深挖"**。

**概念纵深清单**（被追问时的知识锚点，均可在本文档找到答案）：
- 只索引标签 vs 全文索引的存储模型（1.2）｜日志流与基数预算（1.2/Q2）
- CRI 日志格式与容器日志轮转（1.1）｜WAL 崩溃恢复（3.0，对照 Prometheus WAL）
- 文件路径正则的零 API 标签设计（4.0/4.2）｜async DNS 静默失效（9.2 案例 1）
- LogQL 与 PromQL 同构性（1.3）｜日志转指标（5.3 L4）
- retention 三件套（Q7）｜fake 租户目录（Q10）｜ruler 与 Prometheus 规则引擎同构（Q9）

**下一项目伏笔**：可观测性（指标+日志）已闭环，项目 E 可选 **ArgoCD GitOps**（发布自动化：配合项目 B 的 CI 形成 CD 闭环）或 **Grafana Alloy 迁移实验**（承接官方技术演进）。

---

## 12. 学习资源

| 类别 | 资源 |
|------|------|
| 官方文档 | Loki：https://grafana.com/docs/loki/latest/（安装/配置/概念） |
| 官方文档 | LogQL：https://grafana.com/docs/loki/latest/query/（查询全语法与日志指标函数） |
| 官方文档 | Fluent Bit：https://docs.fluentbit.io/manual/（tail input / parser / kubernetes 生态） |
| 官方文档 | Fluent Bit Loki output：https://docs.fluentbit.io/manual/pipeline/outputs/loki.html |
| 命令行工具 | logcli：https://grafana.com/docs/loki/latest/query/logcli/（不开浏览器查日志，可脚本化） |
| 在线练习 | Grafana Play：https://play.grafana.org（官方演示站，含 Loki 大盘，免搭建练手） |
| 视频 | Grafana Labs 官方 YouTube「Introduction to Loki / LogQL」系列 |
| 书籍 | 《Prometheus: Up & Running》（O'Reilly）——PromQL/标签模型与 LogQL 同源，夯实共同基础 |
| 书籍 | 《Observability Engineering》（O'Reilly）——Metrics/Logs/Traces 的工程方法论 |

---

## 附录 A：文件清单

```
K8s项目D_Loki日志/
└── manifests/
    ├── 01-loki-config.yaml            # ★ Loki 主配置（存储/索引/保留/告警对接）
    ├── 02-loki.yaml                   # ★ PVC+Deployment+Service（要替换 __NODE_NAME__）
    ├── 09-loki-rules.yaml             # 进阶：日志告警规则（阶段 7 才 apply）
    ├── 11-fluent-bit-config.yaml      # ★ Fluent Bit 配置（★零 API 标签方案 + DNS TCP 修复）
    ├── 12-fluent-bit-rbac.yaml        #    SA/RBAC（零 API 方案下非必需，保留作升级余量）
    ├── 13-fluent-bit-ds.yaml          # ★ DaemonSet（双 hostPath+DB 断点）
    └── archive/                       # promtail 三件套存档（本环境不可用，保留作排障复盘素材）
        ├── 03-promtail-config.yaml
        ├── 04-promtail-rbac.yaml
        └── 05-promtail-ds.yaml
```

部署顺序：02（先替换占位符）→ 01 → 12 → 11 → 13（→ 09 进阶）。
打包文件 `K8s项目D_Loki日志.zip` 与 `manifests/` 内容一致（含 `archive/` 存档子目录）。

## 附录 B：进阶路线——Grafana Alloy（生产主流）

Promtail 已进入维护态，官方继任者是 **Grafana Alloy**（基于 OTel collector 思想重构，兼容 Prometheus/OTel/Loki 多种管道）：

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install alloy grafana/alloy -n monitoring \
  --set alloy.loki.destination.url=http://loki:3100/loki/api/v1/push
```

Alloy 优势（面试可讲）：① 一个组件统一指标/日志/trace 多管道；② River/HCL 配置可条件化、模块化；③ 官方提供 Promtail 配置转换工具；④ 与 OTel 生态对齐。

> 面试话术："我在自建集群里经历过 promtail 的 kubernetes_sd 静默失效，最终用 Fluent Bit 的零 API 标签方案解决；如果重建集群，我会直接上 Alloy——采集层的这个坑让我把三个采集器的取舍都摸了一遍。"
