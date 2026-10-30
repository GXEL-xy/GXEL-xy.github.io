---
title: "K8s 项目 G：HPA 弹性伸缩 + 压测实战"
date: 2026-10-30T16:44:00+08:00
draft: false
description: "给 damo-app 补 HPA 模板并压测验证副本 2→6→2 的完整扩缩容过程：metrics-server 指标链路四道关、扩容期间零中断验证、缩容冷却窗口实证、Grafana 扩容曲线对照。"
tags: ["k8s HPA"]
categories: [项目]
---

> 📦 **项目文件包**：[下载 K8s项目G_HPA弹性伸缩.zip](/files/K8s项目G_HPA弹性伸缩.zip)（含 hpa.yaml 模板、loadgen 压测清单与 Grafana 大盘 JSON）。本文件是详细操作手册，两者配合。

> **系列定位**：K8s 运维实习项目路线图第 7 站——A 应用容器化 → B CI/CD → C 监控告警 → D 日志收集 → E Helm 化 → F 安全加固 → **G 弹性伸缩**。
> **项目定位**：把项目 E 的 Helm Chart 里那个「预埋但关闭」的 `autoscaling.enabled` 开关**真正打开**——安装 metrics-server 补齐指标采集链路，为 Chart 补写 `templates/hpa.yaml`，用压测工具把流量打上去，**亲眼看到副本从 2 自动扩到 6、流量退去后再自动缩回 2**。
> **为什么这个项目是「必做」**：A–E 证明了你「能部署、能监控、能发布」，但都是**静态**能力。HPA 是唯一能证明你理解「**K8s 如何根据负载自己动手**」的项目——面试官问「你们服务流量涨了怎么办」，答「有 HPA，基于 CPU 75% 阈值 2→6 自动扩，扩容 15 秒内响应、缩容有 5 分钟冷却防抖动」的人，和答「加机器」的人，是两个层次。
> **版本基线**：K8s v1.35（HPA 用 `autoscaling/v2`）｜metrics-server v0.7.x｜helm v3.21.3｜复用项目 B/C/E 全部环境（3 节点 / containerd 2.x / registry `11.0.1.128:30000` / damo-app Release 已存在 / Grafana `:30300`）。
> **前置条件**：项目 E 已完成（damo-app 由 Helm 管理，`helm list -n damo-app` 有 release）；项目 C 已完成（Grafana 可用）。
> **预计耗时**：3–4 小时（含原理学习与压测调参）。
> **权威依据**：Kubernetes 官方文档（Horizontal Pod Autoscaling / HPA Walkthrough）、metrics-server 官方仓库。

---

> **📖 怎么用这份文档**
>
> - **前置依赖**：项目 E/F 已完成（Chart 已支持 `autoscaling.enabled` 开关）；集群已装 metrics-server 或愿意现装
> - **预计耗时**：4–5 小时（含压测与冷却观察，冷却窗口本身要等 5 分钟）
> - **学习路径**：阶段 1 HPA 原理 → 阶段 2 metrics-server → 阶段 3 补 HPA 模板 → 阶段 4 部署 → 阶段 5 压测扩容 → 阶段 6 缩容与调参 → 阶段 7 Grafana 曲线
> - **验收标准**：压测时副本 2→6、停压后回到 2；扩容期间循环 curl 零失败；Grafana 有完整扩容曲线

## 目录

- [0. 项目概览](#0-项目概览)
- [阶段 1：先懂再装——HPA 原理](#阶段-1先懂再装hpa-原理)
- [阶段 2：补指标链路——安装 metrics-server](#阶段-2补指标链路安装-metrics-server)
- [阶段 3：为 Chart 补写 HPA 模板](#阶段-3为-chart-补写-hpa-模板)
- [阶段 4：部署带 HPA 的 damo-app](#阶段-4部署带-hpa-的-damo-app)
- [阶段 5：压测与扩容观测（核心演示）](#阶段-5压测与扩容观测核心演示)
- [阶段 6：缩容观测与参数调优](#阶段-6缩容观测与参数调优)
- [阶段 7：接上监控——Grafana 扩容曲线](#阶段-7接上监控grafana-扩容曲线)
- [8. 故障速查表](#8-故障速查表)
- [9. 自测题（含答案）](#9-自测题含答案)
- [10. 面试要点与简历写法](#10-面试要点与简历写法)
- [11. 学习资源（官方文档）](#11-学习资源官方文档)
- [附录 A：文件包清单与操作顺序](#附录-a文件包清单与操作顺序)
- [附录 B：HPA 算法与公式速查](#附录-bhpa-算法与公式速查)

---

## ★ 执行前必读：本文档成文后，环境发生的 4 处变化（2026-09-24 更新）

> 本文档写于项目 F 之前。**项目 F（安全加固）改变了三件事，直接影响本项目的执行方式**，
> 逐条说明如下 —— **不读这一段会踩坑**。

### ① 前置依赖：F 的 NetworkPolicy 必须为 loadgen 放行出方向

F 的 `damo-app-default-deny-egress` 会**拒绝 damo-app 命名空间所有 Pod 的出方向**，
其中就包括本项目的压测负载 `loadgen`。**loadgen 出不去 → 压不出流量 → HPA 永远不会扩容。**

F 已经为此加了专门的策略，**执行本项目前先确认它存在**：

```bash
kubectl -n damo-app get netpol damo-app-allow-egress-loadtest
# 期望：存在，且 POD-SELECTOR = app=loadgen
# ★ 若不存在 → 先回项目 F 补上（manifests/24-netpol-05-allow-loadgen-egress.yaml 或
#   Chart 的 values 里 networkPolicy.egress.loadtestPodLabels），否则本项目白做
```

> **这正是 F 文档里那个"镜像关系"的落地**：一次成功通信要过
> **源端 egress + 目的端 ingress** 两道关 —— G 的压测链路同时依赖这两条。

### ② `autoscaling.enabled: true` 要写进 **`values-prod.yaml`**，不要用 `--set`

F 之后，Chart 已经进了 GitHub 仓库，**Jenkins 的 Stage 4 用
`-f chart/damo-app/values-prod.yaml` 发布**。所以开关的规范放法是：

```yaml
# chart/damo-app/values-prod.yaml（提交进 git）
networkPolicy:
  enabled: true          # 项目 F
autoscaling:
  enabled: true          # ★ 项目 G 加这一行 —— 与 F 的开关并列
```

**理由与 F §4.7.5 完全一致**：靠 `--set autoscaling.enabled=true` 打开，
开关就只活在命令行里 —— **下次有人不带这个 flag 升级，HPA 会被删掉**
（而且 Helm 会把 Deployment 的 `replicas` 重新写回 `replicaCount: 2`，HPA 随之失效）。

> 本文档 §3.4/§4 原文写的是「改 values.yaml 的 enabled: true」——
> **改 `values.yaml` 也对（它在 git 里），但改 `values-prod.yaml` 语义更准**：
> HPA 是**生产行为**，chart 默认应保持"新人 clone 后 install 就能跑"的友好形态。
>
> ---
>
> **✅ Chart 侧已经做完了（2026-09-25 00:57，commit `2778756`）**
>
> 为了让你少一步，我已经把这两件事提交并推送到仓库了：
> - `chart/damo-app/templates/hpa.yaml`（新增，+60 行）
> - `chart/damo-app/values-prod.yaml`（追加 `autoscaling.enabled: true`）
>
> **⟹ 所以现在 Chart 侧不用再动**。你要做的只剩：
> **① 集群侧装好 metrics-server（阶段 2）→ ② 触发一次流水线 → ③ 验 HPA 的 TARGETS（阶段 4 关卡）**
>
> 顺带说明：`values-prod.yaml` 是一个【**chart 层**、跨项目累积】的文件
> （F 建立 `networkPolicy`，G 追加 `autoscaling`），
> 权威副本在仓库 `chart/damo-app/values-prod.yaml`，
> 工作区副本在 `K8s项目F_安全加固/chart-patch/values-prod.yaml`（两处保持一致）。

### ③ Chart 的改动走 **`chart-patch/` 模式**（与 F 一致，不回写项目 E）

本文档 §3 要新增的 `templates/hpa.yaml` —— **仓库里的落地路径是**：

| 本目录 | → 仓库路径 |
|---|---|
| `chart-patch/hpa.yaml` | `chart/damo-app/templates/hpa.yaml`（**新增**） |
| `values-prod.yaml` 的 `autoscaling` 段 | `chart/damo-app/values-prod.yaml`（**追加**） |

**项目 E 的 `damo-app-chart/` 保持原样、不回写**（这是 F 建立的组织原则）。

### ④ 所有 `curl` 要打「**有 Pod 的节点**」，不能打 master

F 把 Service 设成了 `externalTrafficPolicy: Local` → **NodePort 只下发到有 Pod 的节点**。
本文档里的 `11.0.1.128:30090` 引用**已全部改为自动挑节点**的写法：

```bash
NODE=$(kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
        -o jsonpath='{.items[0].status.hostIP}')
curl --noproxy '*' -s "http://$NODE:30090/" | grep -o 'Build #[0-9]*'
```

> ⚠️ 而且 **HPA 会让 Pod 漂移**，"有 Pod 的节点"是变化的 ——
> **所以务必用上面这种"自动挑"的写法，不要写死某个节点 IP。**
> 另外：**所有连通性测试都要带 `--noproxy '*'`**（本集群有代理会劫持 curl，见 F 文档 §0.6「坑 0」）。

### ⑤ 附带：metrics-server 的文件包已补齐

附录 A.1 里列的 `manifests/metrics-server-components.yaml` **已生成**
（官方 latest = v0.9.0，**已预先打好文档 §2.3 要求的两个补丁**：
镜像指向私有仓库 + args 加 `--kubelet-insecure-tls`）。
**apply 时不需要再手工 sed / patch 了**，文件头有完整说明。

> ★ 版本差异：文档 §2.2 的示例写的是 v0.7.2，文件包用的是官方当前 latest = **v0.9.0**
> （更适配本集群的 K8s v1.35）。若 v0.9.0 报 `unknown flag: --kubelet-insecure-tls`，
> 按文件头的说明删掉那一行即可（失败现象明确，不会静默）。

### ⑥ ★★ 动手做阶段 5 之前，必须先在 Jenkins 里【关掉 5 分钟一次的自动构建】

**实测发现（2026-09-25）**：本项目环境里 Jenkins Job 配了「**每 5 分钟**」的自动触发，
于是**每 5 分钟就会自动跑一次完整流水线**。

> **★ 先分清两种触发器（语义完全不同，别搞混）**：
>
> | 触发器 | 语义 | 会不会"每 5 分钟都构建" |
> |---|---|---|
> | **`Build periodically`（定时构建）** | 到点**无条件**构建 | ✅ **会** ← 本项目的症状最可能是这个 |
> | `Poll SCM`（轮询 SCM） | 到点问 git 有没有新提交，**有才构建** | ❌ 不会 |
>
> 两者在构建历史里**都显示 `Started by timer`**，所以看不出区别 ——
> **判据是「没有新提交时会不会构建」**：会 → 是定时构建。
> 查确切配置：Jenkins → job → Configure → **Build Triggers** 区块，看哪个勾选着。

每次跑都会：

| 动作 | 后果 | 对 G 的影响 |
|---|---|---|
| Kaniko 构建新镜像 | **镜像堆积**（tag 已 44 个，`registry:2` 无 GC） | 长期磁盘风险 |
| `--set image.tag=N`（N 每次不同） | **pod template 变了 → 新 ReplicaSet → 滚动重建 Pod** | ★ 观测被打断 |
| 重建期间 Pod 未就绪 | HPA 报 **`FailedGetResourceMetric: did not receive metrics for targeted pods (pods might be unready)`** | ★ 这个 warning **不是偶发，是每次自动构建必然出现** |

**⟹ 后果**：阶段的「扩容 2 → 6」观测**会被持续打断** ——
录出来的 GIF 会夹杂 rollout，Grafana 曲线也不连续。**必须在动手前先关掉。**

> ### ✅ 实证（2026-09-25 01:05–01:07，loadgen 日志）
>
> 拿着 `kubectl -n damo-app logs -l app=loadgen --timestamps` 一对照，**发布打断服务这件事被完整记录下来了**：
> ```
> 01:05:39–01:05:50  wget: can't connect to remote host (10.100.232.111): Connection refused   ← 大量
> 01:05:45 / 01:06:05 / 01:06:26 / 01:06:43 / 01:07:22–29   wget: download timed out
> 01:07:29 之后      0 错误
> ```
> **时间线完全对应两次 rollout**：01:02:24 手动 `helm upgrade`（第 1 次）+ ~01:05 自动触发（第 2 次）。
>
> **★ 两种错误恰好对应 rollout 的两个阶段**（这也是"loadgen 日志能当证据"的实例）：
> | 错误 | 机制 |
> |---|---|
> | **`Connection refused`** | **Endpoints 空窗口** —— Service 那一刻没有 ready 的 Pod → kube-proxy 直接 REJECT（回 RST） |
> | **`download timed out`** | 连到**正在终止的 Pod**（stale conntrack / DNAT 到已死的 Pod IP）→ 无任何回应 → 挂到 `--timeout=5` |
>
> **⟹ 这不是 NetworkPolicy、不是 metrics-server、不是新故障 —— 就是发布造成的。**
> 面试讲"轮询/定时构建的代价"时，这段带时间戳的日志就是第一手证据。
>
> ### 附带发现：`Endpoints` 在 v1.33+ 已废弃
> ```
> Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
> ```
> 本项目集群是 **v1.35**，所以判断"有没有 ready 的后端"建议改用：
> ```bash
> kubectl -n damo-app get endpointslice
> ```
> **面试话术**：「我知道 v1.33 起 `Endpoints` 被 `EndpointSlice` 取代了 ——
> 后者是为**大规模集群的扩展性**设计的：一个 Service 的端点信息会被切成多个 slice，
> 避免单个对象过大导致的 watch/更新风暴。」

**怎么关（★ 配在 Job 配置里，不在 Jenkinsfile 里 —— 改代码没用）**：

```
Jenkins → damo-app-pipeline → Configure
  → Build Triggers 区块
  → 取消勾选「Build periodically」（以及「Poll SCM」，如果也勾了）
  → 滚到底部 Save
```

**降频的替代方案**（想保留自动化又不打断观测）：把 Schedule 从 `H/5 * * * *` 改成 **`H/30 * * * *`**。
（Schedule 是 5 段：`分 时 日 月 周`；`H` 表示"打散起点"，避免所有 job 整点一起跑。）

> **✅ 实操进展（2026-09-25）**：用户已把频率从 5 分钟改为 **30 分钟**。
> 效果立竿见影 —— Pod AGE 稳定在 26–27 分钟（说明这段时间没有发生 rollout）。
>
> **⚠️ 但 30 分钟仍可能撞上观测窗口，做扩容观测时建议临时完全关掉**：
> | 阶段 | 需要时间 | 30 分钟够吗 |
> |---|---|---|
> | 阶段 5 扩容观测（加压 → 2→6） | 3–8 分钟 | ✅ 一般够 |
> | **阶段 6 缩容观测（停压 → 300s 冷却 → 缩回 2）** | **10–15 分钟** | ⚠️ **可能正好撞上发布** |
>
> 因为**阶段 6 的冷却窗口本身就是 5 分钟**（`stabilizationWindowSeconds: 300`），
> 加上观察与记录很容易跨到 15 分钟 —— 那时定时器就有机会插进来，**观测结果又会夹杂 rollout**。
>
> **三个实际做法**：
> | 做法 | 说明 |
> |---|---|
> | **A. 做扩容观测时临时完全关掉**（最稳） | 取消勾选 → 观测完勾回来 |
> | **B. 查清下一次触发时刻，把观测安排在两次之间** | `H/30` 是**绝对时刻**（如 xx:00 / xx:30），<br>**手动「立即构建」不会重置它** —— 别以为手动跑一次就重新计时 |
> | C. 接受偶发夹杂 | 只看静态结果时无所谓 |
>
> **⚠️ 另一个长期成本：镜像堆积。** 每 30 分钟一个新 tag → **一天 48 个**、一周 300+ 个，
> 而 `registry:2` **没有自动 GC**，长期会吃磁盘。两个长期选项：
> ① 给 registry 配 GC（需 `REGISTRY_STORAGE_DELETE_ENABLED=true` + `registry garbage-collect`）；
> ② 把频率降到 `H H * * *`（每天一次）—— 演示型环境不需要高频自动发布。
>
> **★ 对照 GitOps**：**Argo CD 不会在"集群已符合期望状态"时做任何事** ——
> 它没有定时器，所以**不会因为时间到了就重新构建、重新发布、重新堆积镜像**。
> 这是"Promises vs Timers"最直白的一层差别。

**做完 G 记得恢复。**

> ### 💡 这正好是项目 H（GitOps）最好的切入点
>
> 你计划里写着「**Poll SCM 是内网妥协 → GitOps 是终局**」。现在你有了**第一手的代价证据**：
>
> **每 5 分钟自动构建 = 每 5 分钟一次全量构建 + 一次滚动发布。**
> 1. 镜像仓库快速堆积（44 个 tag，registry:2 没有 GC）；
> 2. **每次发布都会打断正在进行的观测** —— HPA 的 `FailedGetResourceMetric` 就是它导致的，
>    而不是 HPA 或 metrics-server 的问题；
> 3. **更根本的**：集群状态被一个「**定时器**」驱动，而不是被「**期望状态**」驱动。
>
> **而为什么会用定时器？因为内网环境用不了 GitHub webhook**
> （GitHub 访问不到内网 Jenkins）—— 于是只能二选一：
> **要么慢**（轮询间隔大，不实时）、**要么吵**（间隔小，大量空跑构建）。
>
> **面试话术**：「我实测过轮询/定时构建的代价：**每 5 分钟一次全量构建 + 滚动发布**，
> 镜像堆积、观测被打断，而且**集群状态是被定时器推着走的**。
> 根因是**内网用不了 webhook**。
> 所以我理解 GitOps 不只是"把 YAML 放 git 里" ——
> 它是**把方向反过来（集群主动拉 Git）**，于是**既不需要 GitHub 能访问集群，也不需要定时器**，
> **只在期望状态真的变了才动**。**这才是"Promises vs Timers"的区别。**」

---

## 0. 项目概览

### 0.1 目标与产出

| 产出 | 说明 |
|------|------|
| 指标链路打通 | metrics-server 部署完成，`kubectl top nodes/pods` 有真实数据 |
| Chart 补全 HPA | `templates/hpa.yaml`（新增文件）+ `values.yaml` 的 autoscaling 块由开关驱动 |
| 动态扩缩容实证 | 副本 2 →（压测）→ 6 →（停压）→ 2，全程有终端输出与 Grafana 曲线 |
| **扩缩容实测输出** | 分屏观测：左侧 `kubectl get hpa`、右侧 `kubectl get pods`，副本数同步变化 |
| 面试完整故事线 | "我按 CPU 利用率给 damo-app 配了 HPA，2→6 自动扩缩，压测验证过扩容响应与缩容冷却，并用 Grafana 记录了整个过程" |

### 0.2 项目 E 留的伏笔：`autoscaling.enabled`

项目 E 的 `values.yaml` 第 55–62 行已经写好了这块：

```yaml
# ---- HPA 开关（为项目 G 弹性伸缩预留） ----
autoscaling:
  enabled: false          # ← 本项目要把它变成 true
  minReplicas: 2
  maxReplicas: 6
  targetCPUUtilizationPercentage: 75
```

项目 E 的 `deployment.yaml` 第 22–24 行也已经写好了分支：

```yaml
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}    # HPA 关闭时才写死副本数
  {{- end }}
```

**但缺了一样东西**：整个 Chart 里**没有 `templates/hpa.yaml`**。所以本项目阶段 3 的核心工作就是补写这个文件——这就是项目 E 说的「扩展点提前布局」，现在来收获。

> **面试可以这么讲**：「项目 E 把 Chart 参数化的时候，我顺手把 HPA 的开关位和 Deployment 的条件渲染分支留好了，但模板文件没写。做项目 G 时我只补了一个 `hpa.yaml`，Deployment 模板一行都不用动——这就是 Chart 化的收益：**扩展是加文件，不是改文件**。」

### 0.3 架构与数据流（三条链路要分清）

HPA 能不能工作，取决于三条完全独立的链路是否都通——这是本项目的核心认知，也是排错的地图：

```
┌─────────────────────── 链路①：指标采集（本项目新建） ────────────────────────┐
│                                                                             │
│   kubelet (:10250/metrics/resource)  ◄─── 抓取 ───  [metrics-server]        │
│   每个节点暴露 Pod/Node 的真实 CPU/内存用量              (kube-system)       │
│                                                              │              │
│                                                              │ 注册为        │
│                                                              ▼              │
│                                        APIService: v1beta1.metrics.k8s.io   │
└──────────────────────────────────────────────────────────────┬──────────────┘
                                                               │
┌────────────────────── 链路②：决策（HPA 控制器） ───────────────┼──────────────┐
│                                                              │              │
│  [kube-controller-manager 内的 HPA Controller] 每 15s 一轮：                  │
│    ① 从 metrics API 读当前 CPU 用量                                            │
│    ② 按公式算出期望副本数（见附录 B）                                          │
│    ③ 与当前副本数比较，超出容忍度则调用 scale 接口                              │
│                               │                                              │
│                               ▼  修改 Deployment.spec.replicas               │
│  [Deployment damo-app] ──► [ReplicaSet] ──► Pod 数量变化                      │
└──────────────────────────────────────────────────────────────┬──────────────┘
                                                               │
┌────────────────────── 链路③：观测（复用项目 C/D） ─────────────┼──────────────┐
│                                                              ▼              │
│   [kube-state-metrics] 暴露 kube_horizontalpodautoscaler_* 指标              │
│                    │                                                        │
│                    ▼  抓取（15s）                                            │
│              [Prometheus :30091] ──► [Grafana :30300] 扩容曲线大盘            │
└─────────────────────────────────────────────────────────────────────────────┘
```

| 链路 | 组件 | 断了会怎样 | 排错命令 |
|------|------|-----------|---------|
| ① 指标采集 | metrics-server | `kubectl top` 报 `error: Metrics API not available`；HPA 的 TARGETS 显示 `<unknown>/75%` | `kubectl get apiservice v1beta1.metrics.k8s.io -o yaml` |
| ② 决策 | kube-controller-manager 的 HPA 控制器 | HPA 对象存在但副本数永远不动 | `kubectl describe hpa -n damo-app damo-app` 看 Events |
| ③ 观测 | kube-state-metrics + Prometheus | Grafana 画不出曲线（不影响扩缩容本身） | Prometheus `/targets` 页看 kube-state-metrics 是否 UP |

> **一句话记住**：**metrics-server 供数，HPA 算数，kube-state-metrics 报数。** 三者互不依赖，故障可以完全隔离排查。

### 0.4 关键技术决策（面试可讲）

| 决策 | 选择 | 理由 |
|------|------|------|
| 指标类型 | **CPU 利用率**（`type: Utilization`） | 静态页服务最直接的负载信号；内存不适合做伸缩信号（运行时的内存只增不减，HPA 会把服务永久顶在 maxReplicas）；QPS/自定义指标需要 Prometheus Adapter，属进阶 |
| HPA API 版本 | `autoscaling/v2` | K8s 1.23+ 稳定版，支持 `behavior`、多指标、`metrics` 数组；`v1` 只有 CPU 百分比且无行为控制。**v1.35 用 v2** |
| 目标阈值 | **75%** | 留 25% 余量应对扩容生效前的流量爬升（扩容有 15s 决策 + Pod 启动延迟）；阈值设 90% 会「扩容还没生效就已经被打爆」 |
| minReplicas | **2** | 静态页也要抗单 Pod 故障；与项目 B/E 的 2 副本基线一致，改造前后行为可比 |
| maxReplicas | **6** | 3 节点集群资源有限，6 是安全上限；**maxReplicas 必须有上限**，否则异常流量会把集群打穿（HPA 的"最后一道保险"） |
| 缩容冷却 | **默认 300s（5 分钟）** | 默认 `stabilizationWindowSeconds: 300`，取 5 分钟内「建议副本数」的最大值——防止流量抖动导致反复伸缩（flapping）。**HPA 最容易被追问的细节** |
| 与 CI/CD 的关系 | HPA 不写进 Jenkinsfile，由 Chart 的 values 控制 | 副本数是**运行时状态**，不是**发布产物**；发布只管镜像 tag。混在一起会出现「每次发布把副本数重置回 2」的问题 |
| 压测工具 | 见 5.0 三方案对比 | 优先零新增镜像的方案（集群已有 busybox 最省事） |

### 0.5 资源与端口现状

| 项 | 值 | 说明 |
|----|-----|------|
| 命名空间 | `damo-app` | 沿用，不新建 |
| 命名空间 | `kube-system` | metrics-server 装这里（官方默认） |
| 新增 NodePort | **无** | HPA 无 UI；看曲线用项目 C 的 Grafana `:30300` |
| 新增镜像 | `registry.k8s.io/metrics-server/metrics-server:v0.7.x` | **必须中转**到 `11.0.1.128:30000`（铁律） |
| 新增镜像 | 压测工具镜像（见 5.0，可为零） | 同上 |
| 现有端口（勿动） | 30000 registry ｜ 30080 Jenkins ｜ 31000 Jenkins-jnlp ｜ 30090 damo-app ｜ 30091 Prometheus ｜ 30300 Grafana ｜ 30093 Alertmanager ｜ 32541 ingress-nginx | 新建 Service 前先 `kubectl get svc -A` 排查 |

### 0.6 执行前检查清单（5 分钟）

在 master 上逐条确认，**任何一条不过就先回去补齐**：

```bash
# ① 项目 E 的 release 在且健康
helm list -n damo-app
# 期望：damo-app  damo-app  1..N  deployed  damo-app-0.1.0  1.0

# ② 当前工作副本数（记下来，后面压测要观察这个数字变化）
kubectl -n damo-app get deploy damo-app

# ③ 应用页面正常
# ★ 2026-09-23 起：项目 F 把 Service 设为 externalTrafficPolicy: Local，
#   NodePort 只下发到【有 Pod 的节点】→ 不能再打 master(11.0.1.128)。
#   用「自动挑一个跑着 Pod 的节点」最稳（HPA 会让 Pod 漂移，别写死 IP）
NODE=$(kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
        -o jsonpath='{.items[0].status.hostIP}')
curl --noproxy '*' -s "http://$NODE:30090/" | grep -o 'Build #[0-9]*'

# ④ resources.requests.cpu 已设置（★ HPA 基于 requests 算百分比，没有 requests 的 Pod HPA 完全无法工作）
kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.template.spec.containers[0].resources}'; echo
# 期望：{"limits":{"cpu":"500m","memory":"128Mi"},"requests":{"cpu":"100m","memory":"64Mi"}}

# ⑤ 集群节点名（多节点对比时要标节点）
kubectl get nodes -o wide

# ⑥ 项目 C 的 Grafana 可达
curl -s -o /dev/null -w '%{http_code}\n' http://11.0.1.128:30300/
# 期望：200 或 302
```

> ⚠️ **第 ④ 条是本项目最容易致命的坑**：HPA 的 CPU 利用率 = **实际用量 ÷ requests**。如果 Pod 没设 `requests.cpu`，分母为 0，TARGETS 永远显示 `<unknown>/75%` 且不扩容（Events 报 `missing request for cpu`）。项目 E 的 Chart 已设 `100m`，本项目无需改动——但面试被问「HPA 不工作怎么查」，这是第一顺位答案。

---

## 阶段 1：先懂再装——HPA 原理

### 1.0 开胃故事：写字楼里的「弹性工位」（先建立直觉）

把 damo-app 想成一家**客服外包公司**，在写字楼里租工位办公。

- **工位（Pod）**：平时只开 2 个工位，2 个员工接电话（2 副本）。
- **手环（metrics-server）**：给每个工位装心率带，实时测「这个员工多忙」。手环**只管测**，不决定招不招人。
- **HR 主管（HPA Controller）**：每 15 秒巡一圈看心率带读数。判断标准写在一张纸上（HPA 对象）：
  - 「人均忙碌度超过 **75%** 就加人，最多加到 **6** 个工位」（`targetCPUUtilizationPercentage` / `maxReplicas`）
  - 「至少保留 **2** 个工位」（`minReplicas`）
- **招聘动作（scale）**：主管不自己搬桌子，他给物业下发一张「改为 5 个工位」的通知单（修改 `Deployment.spec.replicas`），**真正搬桌子的是 ReplicaSet**。
- **主管的纪律（behavior）**：
  - **招人要快**：新人 15 秒内就能上岗（`scaleUp` 无默认冷却）；
  - **裁人要慢**：主管不会因为电话突然变少就立刻开人，他要求「**连续观察 5 分钟**，5 分钟内只要忙过就保留人员」（`stabilizationWindowSeconds: 300`）——因为他知道电话是一阵一阵来的（流量抖动），频繁开人招人的成本比多留一会儿更高。

**老板拍桌子问的三句话，对应三个必答概念**：

1. 「为什么加人不加桌子？」→ **扩副本（水平扩容）而不是加 CPU/内存（垂直扩容）**：垂直扩容要重启 Pod、有上限、节点不够还是白搭；水平扩容无需重启、理论上无上限、还能顺带提高可用性。
2. 「主管凭什么说人忙？」→ **数据来自 metrics-server**。手环没电（metrics-server 挂了）→ 主管瞎了 → 数字显示 `<unknown>`。
3. 「为什么裁人磨叽？」→ **缩容冷却窗口防抖动**（stabilization window），HPA 默认行为，很多人没意识到。

### 1.1 三个必须区分的概念

| 概念 | 是什么 | 本项目 |
|------|--------|--------|
| **Vertical Scaling（垂直扩容）** | 给单个 Pod 加 CPU/内存 limits | ❌ 不做。原因：必须重启 Pod（业务中断）、受节点规格上限约束、无法提高可用性 |
| **Horizontal Scaling（水平扩容）** | **增加 Pod 数量**，每个 Pod 规格不变 | ✅ 本项目做的是这个（2→6 副本） |
| **Cluster Autoscaler（节点扩容）** | 节点资源不够时**加节点** | ❌ 不做（3 节点 VM 无云 API 可调）。**面试联动点**：HPA 扩到 maxReplicas 后若节点资源也不够，Pod 会 `Pending`——这时才需要 Cluster Autoscaler 或手动加节点。HPA 与 CA 是两层，别混 |

### 1.2 HPA 的控制循环（为什么要等 15 秒）

HPA 不是"事件驱动"的即时反应器，而是**周期性轮询的控制器**：

| 环节 | 周期 | 说明 |
|------|------|------|
| ① 读取指标 | 每 **15 秒** | 从 metrics API（`metrics.k8s.io`）读窗口内平均 CPU 用量 |
| ② 计算期望副本 | 每 15 秒 | 套用公式（附录 B），算出 `desiredReplicas` |
| ③ 判定是否动作 | 每 15 秒 | 与当前副本数比较，**差异在容忍度内（默认 10%）就不动** |
| ④ 下发 scale | 动作时 | 调 `Deployment` 的 scale 子资源 |
| ⑤ ReplicaSet 创建 Pod | 动作后 | 调度 + 拉镜像 + 启动 + 探针 ready |
| ⑥ 新副本分担负载 | 首个业务请求在 ready 之后 | 这才是"扩容真正生效"的时刻 |

**推论（面试点）**：从「流量打上去」到「扩容真正生效、服务能力翻倍」的延迟 ≈ **15s（决策）+ 5s~60s（Pod 启动与探针就绪）**。所以阈值**不能设 90%**——留不出这个延迟的缓冲，扩容还没生效服务就已经被打爆了。75% 是留 25% 余量。

### 1.3 触发 HPA 伸缩的两种指标类型

| 类型 | 写法 | 是否需要 Adapter | 本项目 |
|------|------|-----------------|--------|
| **Resource（资源指标）** | `type: Resource` + `name: cpu` + `averageUtilization: 75` | ❌ 不需要，metrics-server 就够 | ✅ 用这个 |
| **Pods / Object / External（自定义指标）** | `type: Pods` + 自定义指标名（如 `http_requests_per_second`） | ✅ 需要 **Prometheus Adapter** 把 Prometheus 指标注册成 `custom.metrics.k8s.io` | 🔜 进阶（你已有 Prometheus，接 Adapter 是自然延伸，见 12 节） |

> **面试加分**：「我第一版用 CPU 利用率，因为 damo-app 是静态页服务，CPU 与并发请求量强相关，且不需要额外组件。如果要做业务侧伸缩（比如"每 Pod 每秒 100 个请求"），我会用 Prometheus Adapter 把 Prometheus 的 QPS 指标注册成自定义指标——我的项目 C 已经有 Prometheus，接 Adapter 成本很低。**为什么不一上来就用自定义指标**：CPU 指标链路最短（metrics-server 一个组件），先把基础链路跑通再上复杂度，是我做工程的顺序偏好。」

### 1.4 HPA 会「打架」的两种情况（面试常问的边界）

| 冲突 | 现象 | 正确做法 |
|------|------|---------|
| **HPA vs Deployment 里写死的 replicas** | 你 `kubectl edit` 把 replicas 改成 5，HPA 下一轮就把它改回自己算出的值 | 用 Helm 的 `{{- if not .Values.autoscaling.enabled }}` 让二者**互斥**（项目 E 已做到）；HPA 接管后**永远不要手改 replicas** |
| **HPA vs Jenkins 发布流水线** | 每次发布 `helm upgrade` 若还写 `replicas: 2`，就把 HPA 扩出来的 6 个打回 2 个 | 发布时不传 `replicaCount`；开启 `autoscaling.enabled` 后 Deployment 模板不渲染 replicas 字段 |

> **注意**：Helm 升级时开 `autoscaling.enabled=true`，Deployment 模板里 `replicas` 整行消失。Helm 3 做的是三方合并（old manifest / new manifest / live object），**字段从模板里消失会被真正删除**，所以线上 replicas 会被清空、由 HPA 接管。可以在阶段 4 用 `helm get manifest` 亲眼验证。

### 1.5 阶段自测

- [ ] 能不看文档讲完「弹性工位」故事，把 Pod / metrics-server / HPA / ReplicaSet / behavior 对号入座
- [ ] 能说出 HPA 决策周期是 15 秒，并解释为什么阈值不能设 90%
- [ ] 能说清为什么用 CPU 而不用内存做伸缩信号
- [ ] 能说出 HPA 与 Cluster Autoscaler 的分工（两层，不是一回事）
- [ ] 能解释「HPA 接管后为什么不能手改 replicas」，以及 Helm 如何避免打架
- [ ] 能说出 HPA 不工作的第一顺位排查项（requests.cpu 缺失）

---

## 阶段 2：补指标链路——安装 metrics-server

> **本阶段目标**：让 `kubectl top nodes` 和 `kubectl top pods` 输出真实数字。这是 HPA 的**唯一硬依赖**，不通则后面全部白做。

### 2.1 为什么要装：K8s 默认不带指标

K8s 核心组件里**没有任何组件负责采集资源用量**。kubelet 只是在自己内部（`/metrics/resource`）暴露原始数据，**没有任何东西去读它、汇总它、提供标准 API 给 HPA 用**。

metrics-server 就是干这一件事的**最小组件**：

- 从**每个节点的 kubelet** 聚合 Pod / Node 的 CPU、内存用量；
- 把这些数据**注册成一个 K8s API**（`APIService: v1beta1.metrics.k8s.io`）；
- HPA 通过这个 API 读数据——**HPA 眼里 metrics-server 是"另一种 API Server"，不是"一种 Exporter"**。

| 组件 | 采集什么 | 存多久 | 谁在用 |
|------|---------|--------|--------|
| **metrics-server** | 实时 CPU/内存用量（Pod、Node） | **不存**（只保留最近窗口，内存里） | HPA、`kubectl top` |
| **Prometheus**（项目 C） | 全量指标（含历史） | 15 天（TSDB） | Grafana 曲线、告警 |
| **kube-state-metrics**（项目 C） | K8s **对象状态**（期望副本 vs 实际副本等） | 不存（由 Prometheus 抓走） | Grafana、告警 |

> **★ 高频概念题**：「已经有 Prometheus 了，为什么还要装 metrics-server？不能直接用 Prometheus 给 HPA 供数吗？」
> **标准答**：不行（除非加组件）。① HPA 只认 `metrics.k8s.io` 这个**标准 API**，它不会去读 Prometheus 的 HTTP 接口；② Prometheus 是**时序数据库**（为历史查询和告警设计），metrics-server 是**实时读数器**（无存储、低延迟），设计目标不同；③ 若一定要用 Prometheus 数据做伸缩，正确路径是部署 **Prometheus Adapter**，把 Prometheus 指标注册成 `custom.metrics.k8s.io`，HPA 用 `type: Pods` —— 这是「自定义指标伸缩」的标准方案。
> 再加一句更狠的：**在同时有 Prometheus 和 metrics-server 的集群里，两者并存、不冲突**——metrics-server 只服务 HPA 和 kubectl top，Prometheus 服务观测与告警。生产集群两个都装。

### 2.2 镜像中转（铁律，5 分钟）

延续项目 B 的常驻中转铁律：**master 上 docker 拉 → tag → push 到私有仓库 → 各节点 containerd 通过私有仓库拉取**。

```bash
# ====== 在 master 上执行（docker 已配代理）======

# ★★ 注意一：需要 sudo —— master 上的普通用户不在 docker 组
#    （实测 2026-09-25：直接执行报
#      permission denied while trying to connect to the docker API at unix:///var/run/docker.sock）
#    · 临时方案：每条 docker 命令前加 sudo
#    · 一劳永逸：sudo usermod -aG docker $USER  然后重新登录（或 newgrp docker）
#    ★ 本步骤统一按【加 sudo】写。
#
# ★★ 注意二：docker pull/push 是【daemon（root 身份）】发起的，
#    所以它走的是【daemon 级】代理配置，跟你是谁、有没有 sudo 无关：
#      sudo cat /etc/systemd/system/docker.service.d/*.conf
#    ★ 有个常见误解要澄清：~/.docker/config.json 里的 "proxies" 是给
#      docker build / run（容器内）用的，**不作用于 pull** ——
#      所以"加了 sudo 会不会丢代理"这个担心在这里不成立（pull 一直是 daemon 干的）。
#    若确实拉不动 registry.k8s.io（超时 / i/o timeout），先确认 daemon 配了代理。

# 1. 先查当前稳定版本（不要凭记忆写版本号）
#    打开 https://github.com/kubernetes-sigs/metrics-server/releases 看最新稳定版。
#
#    ★★ 注意该页面有【两条 release 轨道】，很容易看错：
#       · metrics-server-helm-chart-X.Y.Z   → 官方 **Helm Chart** 的版本（本项目不用）
#                                           它只有 chart 包，Assets 里【没有】components.yaml
#       · vX.Y.Z（标着 "Latest" 徽章）      → **组件本体**版本 ✅
#                                           点进去，Assets 里才有 components.yaml
#    本项目走「raw manifest 安装」，用后者。
#    截至 2026-09-24 实测：Latest = v0.9.0（项目文件包里的清单就是它）。
MS_VERSION=v0.9.0

# 2. 拉官方镜像（注意地址是 registry.k8s.io，不是 docker.io）
sudo docker pull registry.k8s.io/metrics-server/metrics-server:${MS_VERSION}

# 3. 打上私有仓库前缀
sudo docker tag registry.k8s.io/metrics-server/metrics-server:${MS_VERSION} \
               11.0.1.128:30000/metrics-server:${MS_VERSION}

# 4. 推入私有仓库
sudo docker push 11.0.1.128:30000/metrics-server:${MS_VERSION}

# 5. 验证仓库里有了（这条不需要 sudo）
curl -s http://11.0.1.128:30000/v2/metrics-server/tags/list
# 期望：{"name":"metrics-server","tags":["v0.9.0"]}
```

> **★ 变量别漏**：`MS_VERSION=v0.9.0` 是上一步定义的，**换终端 / 重开会话就没了** ——
> 漏了会变成 `...metrics-server:` （tag 为空）而报错。**每开一个新终端都先 `echo $MS_VERSION` 确认一下。**

### 2.3 安装与「两处必改」

> **★ 面试加分点：官方还提供第二条路 —— Helm Chart**
> metrics-server 官方同时发布 Helm Chart（如 `metrics-server-helm-chart-3.14.0`）。
> **本项目选 raw manifest（`components.yaml`），理由有三条**：
> ① 它是官方文档的标准安装方式，少一层间接（不必先 `helm pull` 再改 values）；
> ② **metrics-server 属集群级基础设施，应和"应用的 Chart"分开管理** ——
>    呼应项目 F 建立的「**每个资源只有一个写入者**」判据：
>    应用 Chart 按应用的发布节奏演进，基础设施组件的升级节奏不同，
>    混在一起会让每次应用发布都带着基础设施的变更面；
> ③ 少一个要中转/维护的 chart。
>
> **面试话术**：「基础设施组件（metrics-server、CNI、Ingress Controller）我倾向于
> 用官方清单或独立的 Helm release 管理，**不塞进应用的 Chart** ——
> 因为**应用 Chart 是按应用生命周期演进的，而基础设施的升级节奏不同**。
> 两者混在一起会让发布面变大、回滚边界变模糊。」

```bash
# 1. 下载官方组件清单（项目文件包里已带一份【打好补丁】的版本，可离线用）
#    在线下载：curl -LO https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
#    ★ 用文件包里的那份就不需要下面的 sed / patch 了 —— 两个补丁已经预先打好

# 2. ★ 必改一：镜像地址换成私有仓库
sed -i 's|registry.k8s.io/metrics-server/metrics-server:.*|11.0.1.128:30000/metrics-server:'"${MS_VERSION}"'|' components.yaml
grep -n 'image:' components.yaml
# 期望：image: 11.0.1.128:30000/metrics-server:v0.7.2

# 3. 应用
kubectl apply -f components.yaml

# 4. ★ 必改二：给容器 args 加 --kubelet-insecure-tls
#    （kubeadm 集群的 kubelet 用自签证书，metrics-server 校验会失败）
kubectl -n kube-system patch deployment metrics-server --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}
]'

# 5. 等待就绪
kubectl -n kube-system rollout status deploy/metrics-server --timeout=120s
kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide
```

> **为什么必须加 `--kubelet-insecure-tls`**：kubeadm 部署的 kubelet serving 证书是**自签**的，CN 通常不是节点 IP；metrics-server 去 `https://<nodeIP>:10250` 抓数据时校验失败，报 `x509: cannot validate certificate ... because it doesn't contain any IP SANs`。
> **生产对照（面试可讲）**：生产环境的正确做法不是加这个参数，而是**把 kubelet 的 serving 证书换成由集群 CA 签发的正规证书**（或把集群 CA 挂给 metrics-server 做校验）。`--kubelet-insecure-tls` 是**教学/自建环境的务实妥协**——它跳过校验，失去了对 kubelet 的身份验证。面试时主动说出这一层，比只背参数强得多。

### 2.4 验证指标链路（四道关）

```bash
# 关① APIService 是否可用（★ 最关键，HPA 读的就是它）
kubectl get apiservice v1beta1.metrics.k8s.io
# 期望：NAME                      SERVICE                      AVAILABLE   AGE
#       v1beta1.metrics.k8s.io    kube-system/metrics-server   True        1m
# 若 AVAILABLE = False → kubectl describe apiservice v1beta1.metrics.k8s.io 看原因

# 关② 节点指标
kubectl top nodes
# 期望：NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
#       master   210m         5%     1820Mi          23%
#       work1    98m          2%     1210Mi          15%
#       work2    105m         2%     1350Mi          17%

# 关③ Pod 指标（这才是 HPA 用的粒度）
kubectl top pods -n damo-app
# 期望：NAME                        CPU(cores)   MEMORY(bytes)
#       damo-app-xxxx-aaa          1m           8Mi
#       damo-app-xxxx-bbb          1m           7Mi

# 关④ 直接问 API（绕开 kubectl top，确认 API 层真的通）
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/damo-app/pods" | head -c 400; echo
# 期望：返回 JSON，含 "usage":{"cpu":"1m","memory":"...Ki"}
```

### 2.5 常见报错速查（安装阶段）

| 报错 | 原因 | 解决 |
|------|------|------|
| `x509: cannot validate certificate ... doesn't contain any IP SANs` | kubelet 自签证书校验失败 | 加 `--kubelet-insecure-tls`（见 2.3） |
| `dial tcp <IP>:10250: i/o timeout` | 网络不通（防火墙/路由） | 在 metrics-server Pod 内 `nc -vz <nodeIP> 10250`；检查节点间 10250 放行 |
| Pod `ImagePullBackOff` | 镜像没中转 / 名字写错 | `kubectl describe pod` 看 Events；`curl http://11.0.1.128:30000/v2/_catalog` 确认镜像在不在 |
| `apiservice AVAILABLE: False`，Events 显示 `failing or missing response` | metrics-server 还没就绪，或 Service 端口/选择器不对 | 先等 30 秒；再看 `kubectl -n kube-system get svc metrics-server` 与 endpoints |
| `kubectl top` 报 `error: Metrics API not available` | metrics-server 完全没起来 | 看 Pod 状态与日志：`kubectl -n kube-system logs deploy/metrics-server` |
| `kubectl top pods` 只有部分 Pod 有数据 | 该 Pod 刚创建，窗口还没数据 | 等 30–60 秒重试；或 `kubectl top pod <name> --containers` |
| `Error from server (ServiceUnavailable)` | metrics-server 正在启动中 | 等 30 秒重试，属正常现象 |

---

## 阶段 3：为 Chart 补写 HPA 模板

> **本阶段目标**：给 `damo-app-chart` 增加 `templates/hpa.yaml`，让 `--set autoscaling.enabled=true` 能渲染出 HPA。**Deployment 模板一行都不用改**（项目 E 已预留）。

### 3.1 要做的三件事

```
damo-app-chart/
├── values.yaml          ← ① 改一处：enabled: false → true（或发布时用 --set 覆盖，不改文件）
└── templates/
    ├── deployment.yaml  ← ② 已就绪，零改动（这就是"扩展点是加文件"的证明）
    ├── service.yaml
    └── hpa.yaml         ← ③ 新增文件（本项目唯一新建的模板）
```

### 3.2 hpa.yaml 全文（文件包 `hpa.yaml`，直接复制到 `templates/` 下）

```yaml
{{- /* ============================================================
项目 G · damo-app Helm Chart —— HorizontalPodAutoscaler 模板

设计要点（面试可讲）：
① 用 autoscaling/v2（K8s 1.23+ 稳定版），支持 behavior 行为控制；
② 整文件由 .Values.autoscaling.enabled 守卫——关闭时渲染产物为空，
   Chart 行为与项目 E 完全一致（向后兼容，这是 Chart 演进的基本纪律）；
③ scaleTargetRef.name 用 include "damo-app.fullname" 而非硬编码，
   与 Deployment 的 metadata.name 同源，改名不会漏改；
④ behavior 显式写出缩容冷却窗口——不写会走默认值，
   写出来是为了"让评审的人一眼看到我们考虑过抖动问题"。

前置：metrics-server 就绪，否则 TARGETS 显示 <unknown>
============================================================ */ -}}
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "damo-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "damo-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "damo-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
  behavior:
    scaleUp:
      # 扩容要快：清零稳定窗口，让决策立刻生效
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100          # 单轮最多翻倍（2→4，4→6，不会一步跳到 6）
          periodSeconds: 15
        - type: Pods
          value: 2            # 或单轮最多 +2，两条同时评估取更激进的
          periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      # 缩容要慢：5 分钟稳定窗口，取窗口内"建议副本数"的最大值
      # —— 流量抖动时不会来回伸缩（防 flapping）
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25           # 单轮最多减 25%（6→5，不会一步砍到 2）
          periodSeconds: 60
      selectPolicy: Max
{{- end }}
```

### 3.3 逐段讲解（面试深挖点）

| 字段 | 值 | 为什么 |
|------|-----|--------|
| `{{- if .Values.autoscaling.enabled }}` | 守卫整文件 | **关闭时渲染为零文档**，Chart 装上后行为与项目 E 完全一致。同一份 Chart 既能用于「无 HPA 演示」又能用于「有 HPA 形态」，靠一个开关切换 |
| `scaleTargetRef.name` | `{{ include "damo-app.fullname" . }}` | 与 Deployment 的 `metadata.name` **同源**（都来自 `_helpers.tpl`），不会出现「HPA 指向一个不存在的 Deployment」 |
| `metrics[0].resource.name: cpu` | CPU | 内存不适合做信号（只增不减） |
| `target.type: Utilization` | 利用率 | 与 `AverageValue` 的区别：**Utilization 是百分比（相对 requests）**，`AverageValue` 是绝对值（如 `200m`）。用 Utilization 才能让「改 requests」与「改阈值」解耦 |
| `averageUtilization: 75` | 75% | 留 25% 余量应对「决策 15s + Pod 启动」的延迟（见 1.2） |
| `scaleUp.stabilizationWindowSeconds: 0` | 0 | 扩容不做冷却，流量来了立刻扩；`0` 表示"不往回看历史建议值" |
| `scaleUp.policies` 两条 + `selectPolicy: Max` | 100% 或 +2 | 两条策略**同时评估取更激进的**：小规模时翻倍（2→4）够快，大规模时也不会一次跳太多把集群打穿 |
| `scaleDown.stabilizationWindowSeconds: 300` | 300s | **★ 核心**：HPA 取「过去 5 分钟内所有计算出的期望副本数」的**最大值**作为缩容目标。等效于"5 分钟内只要忙过，就不缩" |
| `scaleDown.policies: Percent 25 / 60s` | 每 60s 最多减 25% | 平滑缩容：6→5→4→3→2 而不是 6→2（避免"缩太狠 → 立刻又要扩"的锯齿） |
| `selectPolicy: Max`（缩容） | 取更大值 | 缩容场景下 `Max` 意味着**减得最少**的那条策略生效——**保守优先**，方向与扩容相反但目标一致（都为了稳定） |

> **面试高频追问**：「`selectPolicy` 在扩容和缩容里语义一样吗？」
> **答**：写法一样，但效果相反，因为**它选的是"副本数"更大/更小的那个策略**。扩容时选 `Max` = 选增加最多的策略 = 扩得快；缩容时选 `Max` = 选减少最少的策略 = 缩得慢。一句话：**`selectPolicy: Max` 让 HPA 在两种方向上都在"多留副本"**——稳定性优先的默认取向。

### 3.4 渲染验证（不碰集群，先看产物）

```bash
# 在 Chart 目录（仓库里是 chart/damo-app/，本地文件包是 damo-app-chart/）
cd damo-app-chart

# ① 关闭状态：hpa 应完全不出现在渲染结果里
helm template damo-app . --set image.tag=19 | grep -c "kind: HorizontalPodAutoscaler"
# 期望：0

# ② 开启状态：应渲染出 HPA，且 Deployment 里的 replicas 消失
helm template damo-app . --set image.tag=19 --set autoscaling.enabled=true > /tmp/rendered.yaml

grep -n "kind:" /tmp/rendered.yaml
# 期望看到三个：Deployment / Service / HorizontalPodAutoscaler

grep -n "replicas" /tmp/rendered.yaml
# ★ 期望：Deployment 段里【没有】 replicas 行 —— 只有 HPA 里的 minReplicas/maxReplicas
#   这一行验证了项目 E 预埋的 {{- if not .Values.autoscaling.enabled }} 分支生效

sed -n '/kind: HorizontalPodAutoscaler/,$p' /tmp/rendered.yaml
# 期望：完整的 HPA 定义，name=damo-app，target 75，min 2 max 6

# ③ lint 不能忘
helm lint .
# 期望：1 chart(s) linted, 0 chart(s) failed
```

---

## 阶段 4：部署带 HPA 的 damo-app

### 4.1 方式选择：改文件 vs `--set` 覆盖

| 方式 | 命令 | 适用 | 建议 |
|------|------|------|------|
| **A. 改 values.yaml** | 把 `enabled: false` 改成 `true`，git push | 长期开启 HPA（生产形态） | ✅ **本项目采用**：HPA 是 damo-app 的常态能力，写进 values 才是「配置即代码」 |
| B. `--set` 临时覆盖 | `helm upgrade ... --set autoscaling.enabled=true` | 临时试验、不想留痕 | 压测调参阶段可用 |

**本项目走 A**：改 `values.yaml` 并提交到 git，让「damo-app 有 HPA」这件事**在仓库里有记录**。这本身就是一个可讲的点：「HPA 的开关是配置，配置就该进 Git，而不是靠运维在某台机器上敲 `--set`」。

```bash
# 在本地仓库（GitHub 上的 damo-app）修改
vim chart/damo-app/values.yaml
# 把 autoscaling.enabled: false 改为 true
# 顺手把注释里的"为项目 G 预留"改成"项目 G 已启用"，保持文档与实现同步

git add chart/damo-app/values.yaml
git commit -m "feat(hpa): enable HPA for damo-app (2-6 replicas, CPU 75%)"
git push
```

### 4.2 部署（手动一条命令，先不依赖 Jenkins）

```bash
# 在 master 上，用仓库里的 chart 目录直接装
# 注意 --set image.tag 用仓库里真实存在的构建号（先查）
curl -s http://11.0.1.128:30000/v2/damo-app/tags/list
# 假设最新是 19

helm upgrade --install damo-app ./chart/damo-app \
  -n damo-app --create-namespace \
  --set image.tag=19 \
  --atomic --timeout 120s

# 期望输出末尾：
# Release "damo-app" has been upgraded. Happy Helming!
# REVISION: 5
```

### 4.3 四道部署验证

```bash
# ① HPA 对象已创建并读到指标（★ 关键：TARGETS 不能是 <unknown>）
kubectl -n damo-app get hpa
# 期望：NAME       REFERENCE             TARGETS    MINPODS  MAXPODS  REPLICAS  AGE
#       damo-app   Deployment/damo-app   1%/75%     2        6        2         30s
#       ↑ 这里是 "实测/阈值" 格式。若显示 <unknown>/75% → 回阶段 2 查 metrics-server

# ② HPA 详情（能看到当前指标来源与最近事件）
kubectl -n damo-app describe hpa damo-app
# 关注 Conditions 段：
#   AbleToScale      True   ReadyForNewScale
#   ScalingActive    True   ValidMetricFound
#   ScalingLimited   False  DesiredWithinRange
# 若 ScalingActive=False → Events 里通常写明了原因（缺 requests / 指标不可用）
# 若 ScalingLimited=True → 说明已顶到 min 或 max，属"正常受限"而非故障

# ③ Deployment 里的 replicas 字段确实被 HPA 接管（渲染产物验证）
kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.replicas}'; echo
# 期望：2（HPA 初始化时按 minReplicas 设的）

helm get manifest damo-app -n damo-app | grep -A3 "^kind: Deployment" | grep -c "replicas:"
# 期望：0（manifest 里 Deployment 段无 replicas 字段）

# ④ 页面正常（改造后行为不变）
# ★ 2026-09-23 起改打「有 Pod 的节点」（原因见项目 F：externalTrafficPolicy: Local）
NODE=$(kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
        -o jsonpath='{.items[0].status.hostIP}')
curl --noproxy '*' -s "http://$NODE:30090/" | grep -o 'Build #[0-9]*'
# 期望：Build #19
```

> ⚠️ **如果 ① 显示 `<unknown>/75%`**，按这个顺序查（**记住这个顺序，面试会问**）：
> 1. `kubectl top pods -n damo-app` —— 有数据吗？没有 → metrics-server 问题（回阶段 2）
> 2. `kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}'` —— 输出 `100m` 吗？空 → **缺 requests，HPA 算不出百分比**
> 3. `kubectl -n damo-app describe hpa damo-app` 看 Events —— 里面通常直接写明原因
> 4. 等 30–60 秒再刷 —— HPA 刚创建时需要 1–2 个采集周期才有数

### 4.4 顺手修一个「发布与 HPA 打架」的隐患

打开仓库里的 `Jenkinsfile`（项目 E 版），检查 Stage 4：

```groovy
// 项目 E 的 Jenkinsfile（节选）
stage('4. Deploy via Helm') {
    steps {
        container('helm') {
            sh """
                helm upgrade --install damo-app chart/damo-app \
                  --namespace damo-app --create-namespace \
                  --set image.tag=${IMAGE_TAG} \
                  --atomic --timeout 120s
            """
        }
    }
}
```

**检查点**：命令里**没有** `--set replicaCount=N`，也**没有** `--set autoscaling.enabled=false`。✅ 正确——发布只改镜像 tag，副本数交给 HPA。

❌ 反例（面试可以当「我踩过的坑」讲）：

```groovy
// 错误示范：每次发布都把副本数打回 2，HPA 扩到 6 也会被发布重置
--set replicaCount=2
```

> **为什么会有人这么写**：因为 `values.yaml` 里有 `replicaCount: 2` 这一行，开发者会误以为"发布时要显式带上"。实际上开启 `autoscaling.enabled` 后，Deployment 模板已经不渲染 `replicas`，`replicaCount` 这个值**完全不参与渲染**——带上它毫无作用，只有反作用。
>
> **建议顺手做**：在 values.yaml 里加一条醒目注释：
> ```yaml
> # ⚠️ autoscaling.enabled=true 时 replicaCount 完全不参与渲染，
> #    请勿在 Jenkinsfile 里 --set replicaCount=...（会被 HPA 立刻纠正，且产生无意义的 revision）
> replicaCount: 2
> ```
> 这个注释本身就是「工程可见性」的体现——面试讲「我如何避免团队踩这个坑」，比讲技术本身更有分量。

---

## 阶段 5：压测与扩容观测（核心演示）

> **本阶段是整个项目的高潮**：流量打上去，亲眼看着副本从 2 变成 6。

### 5.0 压测工具三方案（按「新增镜像成本」排序）

| 方案 | 工具 | 新增镜像 | 优点 | 缺点 | 推荐度 |
|------|------|---------|------|------|--------|
| **A** | **busybox wget 循环** | 若集群已有 busybox → **零新增** | 最省事；无架构/依赖问题 | 压力量不好精确控制；输出噪声大 | ⭐⭐⭐ **首选** |
| B | `hey`（Go 写的压测工具） | 需中转 `rakyll/hey` 或 `williamyeh/hey` | 并发/时长/QPS 精确可控；输出规范（延迟分布） | 需多一次镜像中转 | ⭐⭐ **进阶**（演示效果最好） |
| C | `apache-bench (ab)` | 需中转 httpd 镜像 | 老牌、参数熟悉 | 镜像较大；keep-alive 支持一般 | ⭐ |

**本项目采用方案 A 为主（先跑通）+ 方案 B 为辅（补一个规范输出）**。只做一个就选 A。

```bash
# 先确认集群里有没有 busybox
curl -s http://11.0.1.128:30000/v2/_catalog | tr ',' '\n' | grep -i busybox
kubectl get deploy,ds -A -o jsonpath='{range .items[*]}{.spec.template.spec.containers[*].image}{"\n"}{end}' | grep -i busybox
```

若两条都没结果，先按铁律中转（一次中转，永久可用）：

```bash
# 在 master 上
docker pull busybox:1.36
docker tag busybox:1.36 11.0.1.128:30000/busybox:1.36
docker push 11.0.1.128:30000/busybox:1.36
```

### 5.1 部署压测负载生成器（loadgen）

**为什么用 Deployment 而不是 `kubectl run`**：Deployment 方便一次开多个副本（多副本 = 更大压力），且方便「停压」（`scale 0`）与清理，不用记 Pod 名。

写文件再 apply（**不用 heredoc/管道**，这是本项目的操作规范）。`manifests/loadgen.yaml`：

```yaml
# ============================================================
# 项目 G · 压测负载生成器
# 用途：在集群内向 damo-app Service 持续打请求，制造 CPU 压力
# 设计：每副本 40 并发 wget 循环，副本数即压力档位（scale 即可调）
# 注意：
#   ① 与 damo-app 同命名空间，用集群内 DNS 访问（不经过 NodePort）
#   ② 不设 resources.limits（否则压测端自己被限流，压力打不上去），
#      但设 requests 以免影响调度公平性
#   ③ 用完必须清理（见 6.3），否则它会一直跑
# ============================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loadgen
  namespace: damo-app
  labels:
    app: loadgen
    purpose: load-testing
spec:
  replicas: 4
  selector:
    matchLabels:
      app: loadgen
  template:
    metadata:
      labels:
        app: loadgen
    spec:
      # 压测 Pod 与业务 Pod 分开，便于观察
      # （若 work2 资源吃紧，删掉下面两行让调度器自由选择）
      nodeSelector:
        kubernetes.io/hostname: work2
      terminationGracePeriodSeconds: 0
      containers:
        - name: loadgen
          image: 11.0.1.128:30000/busybox:1.36
          command: ["sh", "-c"]
          args:
            - |
              set -e
              TARGET="http://damo-app.damo-app.svc.cluster.local/"
              echo "[loadgen] target=$TARGET  start=$(date '+%F %T')"
              while true; do
                for i in $(seq 1 40); do
                  wget -q -O /dev/null --timeout=5 "$TARGET" &
                done
                wait
              done
          resources:
            requests:
              cpu: 100m
              memory: 32Mi
```

```bash
cd K8s项目G_HPA弹性伸缩

# 部署
kubectl apply -f manifests/loadgen.yaml

# 确认起来了（4 个 Running，都在 work2）
kubectl -n damo-app get pods -l app=loadgen -o wide

# 确认它真的在发请求
kubectl -n damo-app logs -l app=loadgen --tail=5
```

> ⚠️ **`nodeSelector: work2` 的取舍**：优点是压测端与业务端分离、观察清晰；缺点是 **work2 的 CPU 会被压测端自己吃掉一部分**，若 work2 本身资源紧，可能影响数据。**若发现扩容数据异常，第一件事就是去掉 `nodeSelector` 或改成 work1 试试**。

### 5.2 ★ 开三个终端，开始观测（演示核心动作）

**操作顺序很重要，建议严格照做，因为要录 GIF**：

```bash
# ========== 终端 1：观察 HPA（每 5 秒刷新）==========
# 用 watch 而不是 -w：-w 输出会滚动，watch 是原地刷新，更适合持续观察
watch -n 5 'kubectl -n damo-app get hpa damo-app; echo; kubectl -n damo-app get deploy damo-app'

# ========== 终端 2：观察 Pod 变化 ==========
watch -n 2 'kubectl -n damo-app get pods -l app=damo-app -o wide'

# ========== 终端 3：观察事件流（扩容真相在这里）==========
kubectl -n damo-app get events --watch --field-selector involvedObject.kind=HorizontalPodAutoscaler
```

先让三个终端安静跑 1 分钟（**基线期**），记下：

| 观测项 | 基线值（示例） |
|--------|---------------|
| HPA TARGETS | `1%/75%` |
| REPLICAS | `2` |
| Pod 数 | 2 |
| 页面 Build | `Build #19` |

然后在第四个终端启动压测 / 停压：

```bash
# 启动压测
kubectl -n damo-app scale deploy/loadgen --replicas=4
# 或首次：kubectl apply -f manifests/loadgen.yaml

# 停压（演示缩容时用）
kubectl -n damo-app scale deploy/loadgen --replicas=0
```

### 5.3 观测时间线（你应该看到什么）

这张表是本项目的「验收标准」，逐行对照：

| 时刻 | HPA TARGETS | REPLICAS | 终端 2 的 Pod 数 | 说明 |
|------|------------|----------|-----------------|------|
| T+0s | `1%/75%` | 2 | 2 | 压测启动前基线 |
| T+15s | `45%/75%` | 2 | 2 | 第一轮采集，未达阈值，**不扩** |
| T+30s | `96%/75%` | **3** | 3（新 Pod `ContainerCreating`） | 超阈值 → 决策扩容 |
| T+45s | `88%/75%` | 4 | 4 | 新 Pod 未 ready，仍在评估 |
| T+60s | `92%/75%` | 5 | 5 | 继续扩 |
| T+75s | `79%/75%` | 6 | 6 | 继续扩 |
| T+90s | `62%/75%` | 6 | 6 | **达到 maxReplicas=6，指标已降到阈值下 → 稳定** |
| — | 稳定在 `50%~70%/75%` | 6 | 6 | 负载被 6 个 Pod 分担，利用率回落 |

> **★ 最关键的一行是 T+90s**：扩容到 6 之后，**TARGETS 从 96% 掉到 62%**。这个数字的下降是「扩容真的减轻了每 Pod 负担」的**因果证明**——不是 HPA 数字在跳，而是服务能力真的提升了。面试时点出这一句，说明你理解 HPA 的反馈闭环。

```bash
# 扩容过程中的事件（终端 3 应刷出类似内容）
#   SuccessfulRescale   New size: 4; reason: cpu resource utilization (percentage of request) above target
#   SuccessfulRescale   New size: 6; reason: cpu resource utilization (percentage of request) above target

# 事后统一查一遍（比实时看更完整，截图更清楚）
kubectl -n damo-app describe hpa damo-app | tail -20
```

### 5.4 扩容期间的可用性验证（体现运维意识）

很多人的 HPA 演示到此为止，**但真正有运维意识的人会顺手证明「扩缩容过程没有中断服务」**——这是面试的差异化加分项。

```bash
# 一边扩容一边持续验证（每 0.2 秒一次，跑 120 秒）
# 目标：验证扩容过程中没有任何一次请求失败
#
# ★★ 2026-09-23 重要修正（项目 F 引入 externalTrafficPolicy: Local 之后）：
#   Local 下 NodePort 只下发到【有 Pod 的节点】。缩容时若某节点失去最后一个 Pod，
#   该节点的 NodePort 会消失 → 打它会得到 000，【这是假失败，不是业务故障】。
#   因此这个"零失败"测试必须固定打【一个始终有 Pod 的节点】，
#   或者更稳妥：把检查点放到集群内（见下方注释的替代方案）。
NODE=$(kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
        -o jsonpath='{.items[0].status.hostIP}')
echo "本次验证固定打节点：$NODE（Local 策略下需选有 Pod 的节点）"

for i in $(seq 1 600); do
  code=$(curl --noproxy '*' -s -o /dev/null -w '%{http_code}' --max-time 2 "http://$NODE:30090/")
  if [ "$code" != "200" ]; then
    echo "[$i] ❌ HTTP $code  ($(date '+%T'))"
  else
    echo -n "."
  fi
  sleep 0.2
done
echo ""
echo "=== 结果：若全程只有 . 没有 ❌，说明扩容期间 HTTP 始终 200，服务零中断 ==="
```

> **为什么能零中断**：① **就绪探针**（项目 A/E 定版的 `tcpSocket`）——新 Pod 必须 `readinessProbe` 通过才接流量；② **Service 的 Endpoints 动态更新**——kube-proxy 只把流量转发给 ready 的 Pod；③ 扩容是**只增不减**，不会摘掉现有 Pod。
> **反面对照（主动说出来，显得你思考过）**：如果探针配置错了（比如项目 A 早期用 `httpGet /healthz` 但应用没这个路径），新 Pod 永远不 ready，那么 HPA 即使扩到 6，**实际接流量的仍然只有 2 个**，扩容完全无效——**HPA 的效果强依赖于探针配置的正确性**。这个联动是 A→G 的知识闭环。

### 5.5 记录时间点（阶段 7 要用）

在压测开始/结束的时间点**记录一下时刻**，阶段 7 要在 Grafana 上按这个时间窗口查曲线，前后因果才对齐。

```bash
date '+%F %T  压测开始'
# ... 压测跑一会儿 ...
date '+%F %T  压测结束'
```

### 5.6 停压（先别急着看缩容，先看「缩容有多慢」）

```bash
# 停压：把 loadgen 缩到 0（Pod 立即消失，压力归零）
kubectl -n damo-app scale deploy/loadgen --replicas=0

# 观察 HPA —— 立刻会看到 TARGETS 掉到 1%，但 REPLICAS 仍是 6
watch -n 5 'kubectl -n damo-app get hpa damo-app'
```

**你会看到的关键现象**：TARGETS 秒降到 `1%/75%`，**但 REPLICAS 在 5 分钟内一动不动**。原因就是阶段 3.3 讲的 `scaleDown.stabilizationWindowSeconds: 300`——HPA 在等 5 分钟，确认「流量是真的退了，而不是抖动」。

> **★ 这个「不动的 5 分钟」就是最有说服力的演示**：它直接证明了「缩容冷却窗口」的存在。面试时说「我配了 5 分钟稳定窗口」，面试官会信；如果补一句「我实测过，停压后 TARGETS 秒降但副本数原地不动 5 分钟，然后才开始 6→5→4→3→2 平滑缩容」，那就不可辩驳了。

---

## 阶段 6：缩容观测与参数调优

### 6.1 缩容时间线（对照 5.3 的扩容表）

| 时刻 | HPA TARGETS | REPLICAS | 说明 |
|------|------------|----------|------|
| T+0s（停压） | `96%/75%` → `1%/75%` | 6 | TARGETS 立刻掉，但**副本不动** |
| T+0 → T+300s | `1%/75%` | **6（不动）** | ★ **稳定窗口**：HPA 取过去 5 分钟建议副本数的最大值 |
| T+300s | `1%/75%` | **5** | 窗口滑出，开始缩容；策略：每 60s 最多减 25% |
| T+360s | `1%/75%` | **4** | 继续缩 |
| T+420s | `1%/75%` | **3** | 继续缩 |
| T+480s | `1%/75%` | **2** | **达到 minReplicas=2，稳定** |
| 之后 | `1%/75%` | 2 | `ScalingLimited: True`（已到下限，正常） |

```bash
# 实时观察缩容（watch 5 秒刷新，约 8 分钟看到全过程）
watch -n 5 'kubectl -n damo-app get hpa damo-app; echo; kubectl -n damo-app get pods -l app=damo-app | tail -8'

# 缩容完成后确认
kubectl -n damo-app get deploy damo-app                        # READY 应为 2/2
NODE=$(kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
        -o jsonpath='{.items[0].status.hostIP}')
curl --noproxy '*' -s "http://$NODE:30090/" | grep -o 'Build #[0-9]*'   # 服务仍正常

# 事后看完整事件（缩容原因）
kubectl -n damo-app describe hpa damo-app | grep -A3 "Events:"
# 期望：New size: 5/4/3/2; reason: All metrics below target
```

### 6.2 参数调优实验（可选，但建议做一次）

这一步的价值是：**证明你不是照着文档抄参数，而是理解每个参数的作用**。选 1–2 个做即可。

| 实验 | 怎么做 | 观察什么 | 结论（面试可讲） |
|------|--------|---------|-----------------|
| **① 阈值调低** | `--set autoscaling.targetCPUUtilizationPercentage=30` | 压测时更早扩容、更容易顶到 6；轻微负载也会扩 | 阈值低 = 更敏感 = 更贵；阈值高 = 更迟钝 = 可能被打爆。**75% 是"留出 15s 决策 + Pod 启动时间"的经验值** |
| **② 关掉缩容冷却** | 把 `behavior.scaleDown.stabilizationWindowSeconds` 临时改成 `0` | 停压后副本**立刻**往下掉，且可能反复抖动 | 冷却窗口是**防抖核心**：关掉后 HPA 会跟着流量噪声反复伸缩（flapping），Pod 频繁创建销毁反而更耗资源 |
| **③ 提高压测强度** | `kubectl -n damo-app scale deploy/loadgen --replicas=8` | 6 副本时 TARGETS 仍超 75% → `ScalingLimited: True`（顶到 max） | ★ **HPA 的边界**：maxReplicas 是硬上限，超过就「扩容失效」，必须靠**加节点（Cluster Autoscaler）**或**提高单 Pod limits**。这是 HPA 与 CA 分工的实证 |
| **④ 改 requests 看分母效应** | `--set resources.requests.cpu=50m` | 同样的绝对 CPU 用量，TARGETS 百分比**翻倍** | HPA 算的是「用量 ÷ requests」，**改 requests 等价于改阈值灵敏度**。所以调 requests 是"间接调阈值"，生产上要谨慎（它同时影响调度） |

```bash
# 实验①的完整命令示例（做完记得改回来）
helm upgrade damo-app ./chart/damo-app -n damo-app \
  --set image.tag=19 --set autoscaling.targetCPUUtilizationPercentage=30 --atomic --timeout 120s
kubectl -n damo-app get hpa
# 恢复
helm upgrade damo-app ./chart/damo-app -n damo-app \
  --set image.tag=19 --set autoscaling.targetCPUUtilizationPercentage=75 --atomic --timeout 120s
```

### 6.3 收尾：恢复干净状态

```bash
# ① 确认压测负载已停
kubectl -n damo-app get deploy loadgen
# 期望：READY 0/0（scale 到 0）或 NotFound（已删除）

# ② 彻底清理压测负载（推荐删掉，避免忘记它在跑）
kubectl delete -f manifests/loadgen.yaml

# ③ 确认 HPA 回到稳态
kubectl -n damo-app get hpa
# 期望：REPLICAS 2，TARGETS 低位

# ④ 确认 Helm 台账干净
helm history damo-app -n damo-app
helm list -n damo-app
```

> **留一个「干净的稳态」很重要**：你的 Grafana 大盘、告警规则都会读这些数据。留下一个 6 副本、持续被打的负载，会让后续项目 F/H 的演示与告警产生噪声。

---

## 阶段 7：接上监控——Grafana 扩容曲线

> **本阶段复用项目 C 的 Grafana（`:30300`）与 Prometheus（`:30091`）**，无需新建任何组件。

### 7.1 前置检查：kube-state-metrics 是否在暴露 HPA 指标

HPA 的曲线指标来自 **kube-state-metrics**（项目 C 已部署），不是 metrics-server——这个区分要清楚：

| 指标来源 | 提供什么 | 本项目用途 |
|---------|---------|-----------|
| metrics-server | **不暴露 Prometheus 格式指标**（只有 metrics API） | 只供 HPA 与 kubectl top |
| **kube-state-metrics** | `kube_horizontalpodautoscaler_spec_min_replicas` / `_max_replicas` / `_status_current_replicas` / `_status_desired_replicas` | ✅ **画 HPA 曲线用它** |
| node-exporter | 节点 CPU/内存 | 辅助（看节点是否吃紧） |

```bash
# 确认 kube-state-metrics 暴露了 HPA 指标
KSM_POD=$(kubectl -n monitoring get pod -l app=kube-state-metrics -o jsonpath='{.items[0].metadata.name}')
kubectl -n monitoring exec "$KSM_POD" -- wget -qO- http://localhost:8080/metrics 2>/dev/null \
  | grep -i horizontalpodautoscaler | head -10

# 期望能看到（版本不同可能略有差异）：
# kube_horizontalpodautoscaler_spec_min_replicas{...}
# kube_horizontalpodautoscaler_spec_max_replicas{...}
# kube_horizontalpodautoscaler_status_current_replicas{...}
# kube_horizontalpodautoscaler_status_desired_replicas{...}
```

**同时确认 Prometheus 在抓 kube-state-metrics**：浏览器打开 `http://11.0.1.128:30091/targets`，找 `kube-state-metrics` 这个 job，状态必须是 **UP**。

若没有这些指标：说明项目 C 的 kube-state-metrics 版本较老或采集配置未开，用 7.3 的**降级方案**（节点 CPU 曲线）代替。

### 7.2 建一张 HPA 扩容大盘（Grafana 操作步骤）

```
1. 浏览器打开 http://11.0.1.128:30300  → 登录
2. 左侧 Dashboards → New → New dashboard → Add visualization
3. 数据源选 Prometheus
4. 查询框切换到 "Code" 模式，依次粘贴下面查询，每条 → 一个 Panel
```

**Panel 1｜副本数变化（核心曲线）**

```promql
# 实际运行副本数（HPA 写入的状态）
kube_horizontalpodautoscaler_status_current_replicas{
  namespace="damo-app",
  horizontalpodautoscaler="damo-app"
}
```

- Panel 标题：`damo-app 副本数（实际）`
- 图形类型：**Time series**（折线；不要用柱状，折线更能体现「爬升-平台-回落」形态）
- 建议叠加一条虚线标出上限：`kube_horizontalpodautoscaler_spec_max_replicas{namespace="damo-app"}`

**Panel 2｜期望 vs 实际副本数（看 HPA 的决策过程）**

```promql
# 期望（HPA 算出来的）
kube_horizontalpodautoscaler_status_desired_replicas{namespace="damo-app"}
# 实际（已经生效的）
kube_horizontalpodautoscaler_status_current_replicas{namespace="damo-app"}
```

- Panel 标题：`期望 vs 实际副本数`
- **看点**：两条线会出现短暂的「期望 6 / 实际 4」的**阶梯状滞后**——这就是「决策快、落地慢」的可视化证据（阶段 1.2）

**Panel 3｜Pod 数**

```promql
count(kube_pod_info{namespace="damo-app", created_by_name=~"damo-app.*"})
```

- Panel 标题：`damo-app Pod 数`

**Panel 4（可选）｜节点 CPU 使用率**

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

- Panel 标题：`节点 CPU 使用率`
- 看点：扩容到 6 时，承载业务的节点 CPU 应明显上升，但**不应到 90%+**（若到了，说明该加节点了）

**设置时间范围**：右上角 → **Last 30 minutes**（覆盖压测 + 缩容的完整周期）。

### 7.3 曲线应该长什么样（验收标准）

```
副本数
  6 ┤            ┌──────────────────────┐
    │           ╱                        ╲
  5 ┤          ╱                          ╲
  4 ┤         ╱                            ╲
  3 ┤        ╱                              ╲
  2 ┤───────╯                                ╲________
    └────┬────────┬───────────┬───────────┬────────┬────► 时间
       T0       T+90s     停压 T        T+300s   T+480s
      压测开始  扩到满    (指标秒降     窗口滑出  缩到 min
                          副本不动 5min) 开始缩容
```

| 阶段 | 曲线特征 | 对应机制 |
|------|---------|---------|
| T0 → T+90s | **快速爬升 2→6**（台阶状，不是直上） | `scaleUp.stabilizationWindowSeconds: 0` + 策略 `Max(100%, +2)` |
| T+90s → 停压 | **平台期，稳定在 6** | 负载被 6 副本分担，利用率回落至阈值下 |
| 停压 → T+300s | **★ 完全水平的一条线** | `scaleDown.stabilizationWindowSeconds: 300`（冷却窗口） |
| T+300s → T+480s | **台阶状下降 6→5→4→3→2** | `scaleDown.policies: Percent 25 / 60s` |

> **降级方案**（kube-state-metrics 没有 HPA 指标时）：用项目 C 已有的「节点 CPU 使用率」曲线代替——同样能看到「压测期间节点 CPU 爬升 → 扩容后被摊平」的形态。**不要因为一个指标缺失就放弃这一步**。

### 7.4 把大盘导出为 JSON（可选，但强烈建议）

```
Grafana → 大盘右上角 Share → Export → Save to file
→ 得到 damo-app-hpa-dashboard.json
→ 放进仓库 dashboards/ 目录，提交
```

> **为什么建议做**：项目 C 的 README 已提到「告警规则表」，这张大盘 JSON 是同一个思路——**把可视化也当成代码管**。面试讲「我的监控大盘是可导出的 JSON，能一键在别的集群复现」，是很实在的加分项。

---

## 8. 故障速查表

| 现象 | 原因 | 排查/解决 |
|------|------|----------|
| **★★ HPA 报 `unable to fetch metrics from resource metrics API: the server is currently unable to handle the request (get pods.metrics.k8s.io)`，且 `TARGETS` 显示一个【不动/偏低】的值** | **metrics API 后端整个掉线了（返回 5xx）→ HPA 拿不到新指标，只能沿用最后已知的值**。<br>★ 那个 `TARGETS` 是**陈旧值**，不是真实利用率 —— 别被它误导。<br>**完整因果链（实测 2026-09-25，见下方证据链）**：<br>压测负载开太大（`loadgen --replicas=20` = 800 并发）<br>＋ `loadgen.yaml` 的 **`nodeSelector: work2`** 把全部压测 Pod 塞在一台机器<br>→ **work2 CPU 99%** → 该节点 **kubelet 响应不过来**（metrics-server 抓取 10s 超时）<br>→ metrics-server 的 **readiness 探针失败**（`no metrics to serve`）→ Pod 变 NotReady<br>→ 从 Service Endpoints **被摘除** → `metrics.k8s.io` APIService **没有可用后端**<br>→ API Server 返回 `unable to handle the request` → **HPA 拿不到指标 → 不扩容** | ① **先把压力降回合理档位**：`kubectl -n damo-app scale deploy/loadgen --replicas=4`；<br>② 等 30–60 秒，指标链路恢复后 HPA 会立刻按真实值决策；<br>③ 诊断三连：`kubectl top nodes`（看是不是某节点被打满）、<br>`kubectl -n kube-system get pod -l k8s-app=metrics-server`（看 RESTARTS）、<br>`kubectl -n kube-system logs deploy/metrics-server --tail=30`（找 `timeout to access kubelet`）；<br>④ **根治：删掉 `loadgen.yaml` 的 `nodeSelector`**（让调度器把压测 Pod 打散到 work1/work2），并把压力档位控制在 **≤8** |
| **★★ damo-app 全体 Pod `ImagePullBackOff`，但 registry 的 `curl :30000/v2/_catalog` 返回 200** | **★ 私有仓库（registry）不是 master 上的 docker 容器 —— 它本身就是一个【集群内工作负载】（Service NodePort 30000）**。<br>节点被压测端打满时，**registry Pod 同样被 CPU 饿死** → 响应不过来 → **拉镜像失败**。<br>⚠️ 排障第一问：**"这个组件到底跑在哪？"** —— 先 `kubectl get pods -A \| grep` / `get svc -A \| grep` 查清楚，别按记忆假设它是宿主机上的容器 | ① 降压力（`scale deploy/loadgen --replicas=2`）—— 这是与 metrics API 失效**共同的根因**；<br>② 确认真身：`kubectl get pods -A -o wide \| grep -i registry`、`kubectl get svc -A \| grep 30000`；<br>③ 节点负载回落后，镜像拉取与 metrics API 会一起自愈。<br>**★ 两条受害链同根**：registry 饿死 → 业务挂；metrics-server 饿死 → HPA 停摆 |

> ### ★★ 附：一次完整的证据链（2026-09-25 实测 —— 这是本项目最好的排障案例之一）
>
> **现象**：压测加到 20 副本后，**HPA 反而不扩容了**（`TARGETS cpu: 59%/75%`、`REPLICAS 3` 不动）。
>
> **四条证据串成一条链**：
> | # | 证据（命令与输出） | 说明 |
> |---|---|---|
> | ① | `kubectl top nodes` → `master 40% / work1 21% / **work2 3976m 99%**` | 20 个 loadgen 全在 work2（`nodeSelector` 写死） |
> | ② | metrics-server 日志：`Failed to scrape node, timeout to access kubelet`<br>`Get "https://11.0.1.129:10250/metrics/resource": context deadline exceeded`<br>`node="work2" timeout="10s"` | **work2 的 kubelet 响应不过来**（10s 抓取超时） |
> | ③ | `metrics-server-...  1/1 Running  **RESTARTS 1**` | metrics-server **自己也被压得重启过一次** |
> | ④ | HPA 事件：`unable to fetch metrics from resource metrics API: the server is currently unable to handle the request` | **最终后果** |
>
> **★ 关键机制（比"被压垮"更准确）**：
> metrics-server 抓不到指标 → 它的 **readiness 探针（`/readyz`）失败**（日志里的 `no metrics to serve`）
> → Pod 变 **NotReady** → 从 Service Endpoints **被摘除**
> → **`v1beta1.metrics.k8s.io` 这个 APIService 没有可用后端**
> → API Server 返回 `the server is currently unable to handle the request`
> → **HPA 报的是"服务器无法处理请求"而不是"指标为空" —— 因为后端整个掉线了，不是数据缺失。**
>
> **★ 结论（面试可直接用）**：
> > **压测容量的上限不只是"被压的服务"，还包括监控链路本身。**
> > 单节点集中 + 过量并发 → **把"量尺"（kubelet + metrics-server）一起压弯** ——
> > **尺子一歪，读数就不可信，HPA 只能沿用旧值。**
>
> **★ 恢复与根治**：`scale deploy/loadgen --replicas=4` → 指标链路自愈；
> 根治是**删掉 `loadgen.yaml` 的 `nodeSelector`**（该文件注释里早就写了这个预案：
> "若 work2 资源吃紧导致数据异常，删掉下面两行让调度器自由选择"）。
>
> ---
>
> ### ✅★ 破解记录（2026-09-25 02:18，用户实测）：删掉 `nodeSelector` 后**扩容立刻成功**
>
> **动作**：注释掉 `loadgen.yaml` 的 `nodeSelector: work2` → `kubectl apply` 重新部署。
> **结果**：**HPA 成功扩容** ✅
>
> **为什么这一步同时解决了两个问题**：
> ```
> 删掉 nodeSelector
>   → loadgen 从「全挤在 work2」变成「分散到 work1 + work2」
>   → work2 不再被压到 97%
>   → ① kubelet 恢复正常 → metrics-server 抓得到 → metrics API 稳定
>        ⇒ HPA 能读到【真实】指标（不再是陈旧值、不再报 unable to handle the request）
>   → ② ★ damo-app 的 Pod 不再被 loadgen 抢 CPU
>        → 它的【真实 CPU 用量】才显现出来 → 越过 75% → HPA 扩容
> ```
>
> **★★ 第 ② 点是「CPU 利用率 ≠ 服务压力」的实证（本项目最反直觉的一条结论）**：
> > **work2 被 loadgen 占满时，damo-app 的 Pod 拿不到 CPU 时间 →
> > 它的"用量"被压到 39% —— 不是它不忙，而是它【跑不起来】**（延迟高、CPU 低）。
> > **HPA 因此看不到压力，反而停止扩容。**
> > 一旦节点资源释放，**应用的真实 CPU 需求立刻显现** → 越过阈值 → 扩容。
>
> **⟹ 结论：压测时"把压力和被测服务放在同一节点"是一个致命的实验设计错误** ——
> 它同时污染了**指标链路**和**被测服务的 CPU 读数**，让你既看不到数据、又读到偏低的数据。
> **正确做法：压测端要么分散、要么根本不在被测服务所在节点上跑。**
| **★★ `kubectl top nodes` 显示某节点 CPU 很高（如 97%），但 HPA 的 `TARGETS` 只有 39%、副本不涨** | **★ 这不是故障 —— 是"节点负载 ≠ 应用负载"**：<br>HPA 看的是**目标 Pod 的 CPU 利用率**（相对它自己的 `requests`），<br>**而不是节点负载**。节点那 97% 里大部分是**压测端自己吃的** ——<br>`loadgen.yaml` **故意不设 `resources.limits`**（注释原话："不设 limits，否则压测端自己被限流，压力打不上去"）<br>→ **4 个 loadgen 副本就足以打满一台 4 核节点**。<br>**数学验证**：loadgen 绝对压力固定时 —— `2 副本→每 Pod 66%`、`3 副本→66×2/3 = 44%`（≈ 实测 39%）✅ | ① 先确认 **HPA 指标的归属**：<br>`kubectl -n damo-app get hpa` 的 `TARGETS` 是**应用**的利用率；<br>`kubectl top nodes` 是**整机**负载 —— 两者不要混。<br>② 要真正扩容 → **必须加大绝对压力**，但要**先分散压力**（删 `loadgen.yaml` 的 `nodeSelector`），否则单节点先被打满、metrics API 反而先挂（见上一条）。<br>③ 最终压力档位控制在 **6~8**（分散到 work1/work2） |
| **★★ 「为什么加压了 HPA 还是不动」的深层原因 —— CPU 利用率 ≠ 服务压力** | 当节点被**别的 Pod**（压测端）打满时，**你的服务延迟可能已经很高、但 CPU 利用率反而不高**：<br>CPU 争抢让目标 Pod 拿不到 CPU 时间 → 它的"用量"上不去 → **HPA 不会扩**。<br>**这是「基于 CPU 的 HPA」的根本局限。** | **生产解法：用自定义指标扩容** —— 部署 **Prometheus Adapter**，把 Prometheus 里的<br>QPS / 延迟指标注册成 `custom.metrics.k8s.io`，HPA 改用 `type: Pods`。<br>★ **这正好串起项目 C（Prometheus）与项目 G（HPA）**：<br>C 提供指标源，G 的 CPU 局限由"Prometheus + Adapter"补上。<br>（G 文档 §2.1 已有这条"自定义指标伸缩"的说明） |
| **★ HPA 的 `TARGETS` 正常（如 66%/75%），但副本数一直不涨** | **这不是故障 —— 是 HPA 的正确行为**：扩容条件是「**当前利用率 > 目标利用率**」。66% < 75% ⇒ 不扩。公式验算：`ceil(3 × 66/75) = ceil(2.64) = 3` = 当前 3 | 要扩容就得**把压力打到阈值以上**：<br>① 加压（`kubectl -n damo-app scale deploy/loadgen --replicas=8`）；<br>② 等 **30–60 秒**（metrics-server `--metric-resolution=15s` + HPA sync-period）；<br>③ `watch -n 5 'kubectl -n damo-app get hpa'` 观察 |
| **★ 加压了却还是不涨 / 涨得比预期少** | **HPA 的自我平衡**：扩容 → 负载摊薄 → 利用率回落到阈值以下 → 停止。<br>★ 还常有一个前提被忽略：**`loadgen` 的副本数要和"目标当前副本数"配套** ——<br>`loadgen.yaml` 注释里"`replicas=4` 能触发 2→6"的**前提是 damo-app 只有 2 副本**；<br>当 damo-app 已扩到 3 副本时，同样压力只到 66%，**打不到阈值了** | 压力档位对照（`loadgen.yaml`）：<br>`2` = 轻载（不足以触发）；`4` = 默认（**针对 2 副本目标**）；`8` = **重载**（用于顶到 maxReplicas 的边界实验） |
| **HPA 出现 `FailedGetResourceMetric: did not receive metrics for targeted pods (pods might be unready)`** | **扩容时新 Pod 还没就绪**，指标窗口里拿不到数据 → 瞬态。<br>★ 但如果**反复出现**，要怀疑是**频繁发布打断**（见本项目「执行前必读 ⑥」：定时/轮询构建每 N 分钟一次 rollout） | ① 偶发一条 → **忽略**（下一条 `SuccessfulRescale` 会跟上）；<br>② 反复出现 → `kubectl -n damo-app get pods` 看 Pod 是不是在被反复重建；<br>③ 确认是不是 Jenkins 定时触发在打扰 |
| **`Endpoints` 报 `v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice`** | 不是故障，是 K8s v1.33+ 的**弃用提示** | 改用 `kubectl -n damo-app get endpointslice`。★ 面试话术：**EndpointSlice 是为大规模集群扩展性设计的**（一个 Service 的端点切成多个 slice，避免单对象过大导致 watch/更新风暴） |

按「现象 → 原因 → 排查与解决」组织。**★ 标记的是面试高频考点**。

| 现象 | 原因 | 排查与解决 |
|------|------|-----------|
| **★ HPA 的 TARGETS 显示 `<unknown>/75%`** | ① metrics-server 不可用；② **Pod 未设 `requests.cpu`**（分母为 0）；③ HPA 刚创建，还没采集到数据 | ① `kubectl get apiservice v1beta1.metrics.k8s.io` 看 `True`；② `kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}'` 必须非空；③ 等 30–60 秒。**三条按顺序查，90% 命中** |
| `kubectl top` 报 `error: Metrics API not available` | metrics-server 未就绪或挂了 | `kubectl -n kube-system get pods -l k8s-app=metrics-server` → `logs` → `describe` |
| metrics-server 日志 `x509: cannot validate certificate` | kubelet 证书自签，未加跳过参数 | 补 `--kubelet-insecure-tls`（阶段 2.3） |
| metrics-server Pod `ImagePullBackOff` | 镜像未中转 / 名字写错 | `kubectl describe pod`；`curl http://11.0.1.128:30000/v2/metrics-server/tags/list` |
| `apiservice AVAILABLE: False`，`failing or missing response` | metrics-server Pod 未 Ready，或 Service 端口/选择器错 | 先等 30s；再查 `kubectl -n kube-system get svc metrics-server -o yaml` 与 `get endpoints metrics-server` |
| **★ HPA 存在，但副本数永远不动** | ① 负载没达到阈值；② 顶到 min/max（`ScalingLimited: True`）；③ 控制器异常 | `kubectl describe hpa` 看 `Conditions` 与 `Events`：指标低于阈值属正常；`ScalingLimited True` 说明已到上下限；Events 有 `FailedGetResourceMetric` 才说明真故障 |
| HPA 报 `failed to get cpu utilization: missing request for cpu` | Pod 未设 requests.cpu | 在 Chart 的 `values.yaml` 里补 `resources.requests.cpu`（本项目已有 `100m`） |
| 扩容后 Pod 一直 `Pending` | 节点资源不足（requests 加起来超了节点可分配量） | `kubectl describe pod` 看 `FailedScheduling` / `Insufficient cpu`。**这就是 HPA 的边界**——需要 Cluster Autoscaler 或加节点（见 6.2 实验③） |
| 扩容后 Pod `ImagePullBackOff` | tag 指向不存在的镜像（HPA 不改 tag，此错是上一次发布留下的） | `kubectl -n damo-app describe pod` 看 Events；`curl http://11.0.1.128:30000/v2/damo-app/tags/list` 核对 |
| 扩容到 6 但 TARGETS 仍超阈值，感觉「扩了没用」 | ① 压测强度过大；② 单 Pod limits 太低（`500m`，6×500m=3 核也可能不够）；③ Pod 未 ready 所以不接流量 | ① 降 loadgen 副本数；② 提 `resources.limits.cpu`；③ **重点查探针**：`kubectl -n damo-app get pods -l app=damo-app` 的 READY 列必须是 `1/1`，若为 `0/1` 则新 Pod 没在接流量，HPA 扩容**完全无效** |
| **★ 停压后副本数不降** | `scaleDown.stabilizationWindowSeconds: 300` —— **这是设计，不是故障** | 等满 5 分钟；若要快速验证缩容，临时把该值设为 0 再压测一轮 |
| 停压后副本数**立刻**降了 | 有人改过 behavior 或用了 `v1` API（无行为控制） | `kubectl -n damo-app get hpa damo-app -o yaml \| grep -A10 behavior` 核对 |
| 副本数来回跳（锯齿） | 缩容窗口太短 + 阈值贴近实际负载 | 加长 `stabilizationWindowSeconds`；或把阈值从临界值挪开（实际 74% 而阈值设 75% 极易抖动） |
| `kubectl edit deploy` 改的副本数被改回去了 | **HPA 接管中，手改无效**（HPA 每 15s 覆盖） | 这是预期行为。要固定副本数就先关 HPA（`--set autoscaling.enabled=false`） |
| 每次 Jenkins 发布后副本数被打回 2 | Jenkinsfile 里带了 `--set replicaCount=2` | 删掉该参数（阶段 4.4）；开 HPA 后 `replicaCount` 不参与渲染 |
| 开 HPA 后 Deployment 里 `replicas` 字段没消失 | 只改了 `values.yaml` 但没 `helm upgrade` | `helm get manifest damo-app -n damo-app` 确认渲染产物；确认已执行 upgrade |
| Grafana 画不出 HPA 曲线 | kube-state-metrics 未暴露 HPA 指标，或 Prometheus 未抓 | `curl http://11.0.1.128:30091/targets` 看 kube-state-metrics 是否 UP；必要时用节点 CPU 曲线降级（7.3） |
| Grafana 曲线断续（有断点） | Prometheus 采集间隔 15s，而扩容发生在 15s 内 | 正常。若要更细，改 Prometheus 的 `scrape_interval`（不建议低于 10s，压力大） |
| `helm upgrade` 报 `another operation is in progress` | 上次操作异常中断，release 被锁 | `helm history` → `helm rollback damo-app <上一个好 revision>` 解锁 |
| 压测把集群打得很卡，SSH 都迟钝 | loadgen 副本太多且与业务同节点 | 立刻 `kubectl -n damo-app scale deploy/loadgen --replicas=0`；给 loadgen 加 `nodeSelector` 或不与业务同节点 |
| 忘记清理压测负载，告警一直响 | loadgen 仍在跑 → CPU 告警持续 | `kubectl -n damo-app scale deploy/loadgen --replicas=0` + `kubectl delete -f manifests/loadgen.yaml`；这是**每轮压测结束后的固定收尾动作** |

---

## 9. 自测题（含答案）

**Q1. HPA 和 metrics-server 是什么关系？为什么要单独装 metrics-server？**
A：metrics-server 负责从各节点 kubelet 聚合实时 CPU/内存用量，并**注册成 K8s 标准 API**（`APIService: v1beta1.metrics.k8s.io`）。HPA 通过这个 API 读数据。K8s 核心组件里没有任何东西负责采集资源用量，所以 HPA「有脑子但没有眼睛」，必须装 metrics-server 给它供数。

**Q2. HPA 算 CPU 利用率的公式里，分母是什么？为什么它比分子更重要？**
A：分母是 Pod 的 `resources.requests.cpu`。利用率 = 实际用量 ÷ requests。分母缺失会让 HPA 完全无法工作（报 `missing request for cpu`，TARGETS 显示 `<unknown>`）；分母变化会直接改变灵敏度——同样的绝对负载，requests 从 100m 降到 50m 会让利用率翻倍、更容易触发扩容。所以「调 requests」是「间接调阈值」。

**Q3. 为什么 HPA 用 CPU 而不用内存？**
A：内存对多数运行时是**单调不减**的（GC 后不还给 OS、缓存只增），用内存做信号会让服务一旦升上去就永远下不来，长期顶在 maxReplicas 上白花钱。CPU 是可回收的即时负载信号，与并发请求量强相关，适合做伸缩信号。

**Q4. `targetCPUUtilizationPercentage` 为什么不能设成 90%？**
A：从「流量上升」到「扩容真正生效」有延迟：HPA 决策周期 15s + Pod 调度/拉镜像/启动/就绪探针通过（通常 5–60s）。阈值设 90% 意味着只剩 10% 余量，扩容还没生效服务就已经被打爆。75% 是留出延迟缓冲的经验值。

**Q5. `scaleDown.stabilizationWindowSeconds: 300` 到底做了什么？**
A：HPA 每 15s 算一次「期望副本数」，但缩容时它不直接用最新值，而是取**过去 300 秒内所有计算结果的最大值**作为缩容目标。效果是：5 分钟内只要忙过一次，就不缩。用途是**防止流量抖动导致反复伸缩（flapping）**——频繁创建销毁 Pod 的成本比多留几个副本更高。

**Q6. `behavior` 里 `selectPolicy: Max` 在扩容和缩容时语义一样吗？**
A：写法一样，效果相反。它选的是「副本数更大」的那个策略：扩容时 `Max` = 选增加最多的策略 = 扩得快；缩容时 `Max` = 选减少最少的策略 = 缩得慢。两种方向都指向「多留副本」，是稳定性优先的取向。

**Q7. 开启 HPA 后，为什么不能手改 `kubectl edit deploy` 的 replicas？为什么发布流水线也不该传副本数？**
A：HPA 每 15s 会用自己的计算结果覆盖 Deployment 的 replicas，手改下一轮就被改回去。发布流水线若传 `replicaCount`，虽然开启 HPA 后该值不参与渲染，但一旦模板里还残留 replicas 分支就会把 HPA 扩出来的副本打回基线——这正是项目 E 用 `{{- if not .Values.autoscaling.enabled }}` 让二者互斥的原因。

**Q8. 扩容期间服务会中断吗？为什么？前置条件是什么？**
A：不会。因为 ① Service 的 Endpoints 只包含 ready 的 Pod；② 新 Pod 必须通过 `readinessProbe` 才会被加入 Endpoints；③ 扩容只增不减，不会摘除现有 Pod。**前置条件是探针配置正确**——如果探针写错（比如 `httpGet` 指向不存在的路径），新 Pod 永远不 ready，HPA 即使扩到 6 个，实际接流量的还是原来 2 个，扩容完全无效。

**Q9. HPA 扩到 maxReplicas 后指标仍超阈值，该怎么解决？**
A：这是 HPA 的硬边界（`ScalingLimited: True`）。三条路：① **加节点**（云上用 Cluster Autoscaler，自建则手动加节点）；② **提高单 Pod 的 `limits.cpu`**（垂直扩容，但需重启 Pod）；③ **优化应用**（降低单请求 CPU 消耗）。**HPA 只负责「在节点资源允许的范围内加 Pod」，节点不够是另一个层次的问题**。

**Q10. 集群里已经有 Prometheus 了，能让 Prometheus 直接给 HPA 供数吗？**
A：不能直接供。HPA 只认 `metrics.k8s.io`（资源指标）和 `custom.metrics.k8s.io` / `external.metrics.k8s.io`（自定义指标）这几个 API。Prometheus 是时序数据库，不是指标 API。若要用 Prometheus 的数据做伸缩，需要部署 **Prometheus Adapter**，把 PromQL 结果注册成 `custom.metrics.k8s.io`，然后 HPA 用 `type: Pods` 引用——这是「自定义指标伸缩」的标准路径。

**Q11. 为什么本项目用 Deployment 部署压测负载，而不是 `kubectl run`？**
A：Deployment 便于：① 一次开多个副本（改 `replicas` 即调压力量）；② 快速停压（`scale --replicas=0`，无需记 Pod 名）；③ 声明式管理，配置文件可版本化。`kubectl run` 适合一次性调试，不适合可重复的压测。

**Q12. 为什么建议把压测负载 `nodeSelector` 钉在 work2？有什么风险？**
A：把压测端与业务端分离，便于观察、避免压测容器与被测 Pod 抢同一节点资源导致数据失真。**风险**：work2 的 CPU 会被压测端自己消耗，若 work2 本身资源紧张，可能影响业务 Pod 在该节点的 CPU 表现。所以一旦发现扩容数据异常，第一件事就是去掉 `nodeSelector` 或换节点。

**Q13. HPA 的指标为什么画在 Grafana 上要用 kube-state-metrics，而不是 metrics-server？**
A：metrics-server **不暴露 Prometheus 格式的指标**（它只提供 metrics API），Prometheus 抓不到它。而 **kube-state-metrics 暴露 K8s 对象状态**，包括 `kube_horizontalpodautoscaler_status_current_replicas` 等 HPA 相关指标。三者的分工：metrics-server 供 HPA 用，kube-state-metrics 供观测用。

**Q14. 如果压测打上去，TARGETS 只到 40% 一直不超 75%，怎么排查？**
A：① 先确认压测是否真的在发请求（看 loadgen 日志与它自己的 CPU）；② 确认压力够不够（加 loadgen 副本数）；③ 看被测 Pod 的实际 CPU（`kubectl top pods -n damo-app`），如果只有 30m 而 requests 是 100m，那就是分母太大——**解决办法是降低 `requests.cpu`**（比如设 20m，同样的绝对用量就超过 75% 了）。这也是 6.2 实验④要演示的东西。

**Q15. 这个项目里，Helm 扮演了什么角色？换成裸 YAML 会怎样？**
A：Helm 的角色是**让 HPA 成为一个开关式能力**：`values.yaml` 里的 `autoscaling.enabled` 控制 HPA 模板是否渲染，Deployment 的 `replicas` 字段是否渲染也由同一个开关驱动，二者天然互斥。用裸 YAML 的话，你需要手工维护「启用 HPA 版」和「不启用版」两套清单，且容易漏改（比如忘删 Deployment 里的 replicas，HPA 和写死副本数就会打架）。**Chart 化让「能力开关」变成一次 `--set`**，这是项目 E 收益在本项目的具体兑现。

---

## 10. 面试要点与简历写法

### 11.1 一段话讲完本项目

> 「我给 damo-app 配了基于 CPU 的 HPA。基础链路是装 metrics-server 打通指标采集，因为 kubeadm 集群的 kubelet 是自签证书，我加了 `--kubelet-insecure-tls`——但我清楚生产上是该给 kubelet 换正经证书而不是跳过校验。HPA 这边我没有另写一套 YAML，而是在项目 E 的 Chart 里补了一个 `hpa.yaml`，由 `autoscaling.enabled` 开关驱动，Deployment 的条件渲染分支项目 E 已经预留好了，一行没改——这验证了 Chart 化的收益是"扩展加文件而不改文件"。参数是 2–6 副本、CPU 75% 阈值、缩容 5 分钟稳定窗口。压测验证时我看到副本从 2 扩到 6、TARGETS 从 96% 回落到 62%，并用 Grafana 记录了完整曲线；停压后副本原地不动 5 分钟才开始平滑缩容，实证了防抖窗口的存在。压测全程我用循环 curl 验证过服务零中断。」

### 11.2 高频追问预案

| 追问 | 怎么回答 |
|------|--------------|
| HPA 不工作怎么查？ | 按序：① `kubectl top pods` 有没有数据 → ② `requests.cpu` 有没有设 → ③ `describe hpa` 看 Events。**顺序本身就是答案** |
| 为什么不用内存做伸缩指标？ | 内存单调不减，服务升上去就下不来，长期顶 maxReplicas |
| 为什么阈值是 75% 而不是 90%？ | 扩容有 15s 决策 + Pod 启动延迟，要留缓冲（1.2 的时间线） |
| 缩容为什么那么慢？ | `stabilizationWindowSeconds: 300` 防抖动；**我有实测数据** |
| HPA 和 Cluster Autoscaler 什么关系？ | 两层：HPA 加 Pod，CA 加节点；HPA 顶到 maxReplicas 或 Pod Pending 时才是 CA 的地盘（G-8） |
| 扩容会不会影响线上可用性？ | 不会，探针 + Endpoints 保证新 Pod ready 才接流量；**我用 600 次循环 curl 验证过零中断**（G-6） |
| 自定义指标怎么做？ | Prometheus Adapter 把 PromQL 注册为 `custom.metrics.k8s.io`，HPA 用 `type: Pods`；项目 C 已有 Prometheus，接 Adapter 是自然延伸 |
| 为什么要装 metrics-server，Prometheus 不够吗？ | 见 Q10：HPA 只认标准 metrics API，Prometheus 是时序库不是指标 API；生产两者并存 |
| 怎么防止发布把 HPA 扩出来的副本打回去？ | Jenkinsfile 不带 `--set replicaCount`；Chart 的 `{{- if not autoscaling.enabled }}` 让二者互斥；values.yaml 里加了警示注释 |
| 这个项目最难的地方是什么？ | ① 指标链路的排错（metrics-server / APIService / requests 三层）；② 压测调参——一开始 TARGETS 上不去，发现 requests 100m 太大导致分母偏高，调小后才触发（**讲这个比讲"一次成功"更可信**） |

### 11.3 简历写法（可直接用）

> **短版（1 行）**：
> 为 damo-app 落地 HPA 弹性伸缩（metrics-server + Chart 内 HPA 模板），2–6 副本基于 CPU 75% 自动扩缩，压测实测 2→6 扩容与 5 分钟冷却缩容，扩容期间服务零中断。

> **长版（3 行，适合放项目详情）**：
> - 部署 metrics-server 打通指标采集链路（`--kubelet-insecure-tls` 适配 kubeadm 自签证书），沉淀「APIService 可用性 → kubectl top → requests 配置 → HPA Conditions」四步法 HPA 失效排查路径。
> - 在既有 Helm Chart 上以「新增 hpa.yaml + 开关驱动」方式引入 HPA（`autoscaling/v2`，2–6 副本、CPU 75%、缩容 300s 稳定窗口、扩容 100%/+2 双策略取 Max），Deployment 模板零改动。
> - 以集群内压测负载实测扩容 2→6（TARGETS 96%→62%，印证容量真实提升）、停压后 5 分钟冷却窗口与平滑缩容，Grafana 记录完整扩容曲线；600 次循环请求验证扩容期间零中断。

### 11.4 与其他项目的叙事串联（面试时把 A–G 讲成一条线）

```
项目 A  探针误杀 → 定版 tcpSocket     ┐
                                     ├─→ 项目 G 的"扩容零中断"依赖正确的探针配置
项目 E  Chart 预埋 autoscaling 开关   ┤
                                     └─→ 项目 G 只加一个 hpa.yaml 就启用，验证"扩展加文件"
项目 B  Jenkins 发布流水线            ──→ 项目 G 提醒"发布不要传副本数"，避免与 HPA 打架
项目 C  Prometheus + Grafana          ──→ 项目 G 的扩容曲线大盘复用它的数据源
项目 D  Loki 日志                     ──→ HPA 扩出的新 Pod 日志同样被采集（弹性场景下日志不丢）
项目 F  RBAC + NetworkPolicy（下一个） ──→ 给 HPA 扩出的 Pod 套上同一套最小权限与网络策略
```

**一句话把七个项目串起来**：

> 「A–G 是一条完整的链路：A 让 Pod 活着且能被访问、B 让变更能自动化到达集群、C/D 让我能看见集群在发生什么、E 让交付可审计可回滚、**G 让集群能随着负载自己动手**、F 让它动得安全。这七步合起来，就是一套自建集群的运维闭环。」

---

## 11. 学习资源（官方文档）

| 主题 | 资源 |
|------|------|
| **HPA 概念与算法**（必读） | https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale/ |
| **HPA 动手演练**（官方 walkthrough） | https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/ |
| HPA API 参考 | https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/ |
| HPA behavior 字段详解 | https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale/#configurable-scaling-behavior |
| 资源请求与限制（requests/limits 语义） | https://kubernetes.io/zh-cn/docs/concepts/configuration/manage-resources-containers/ |
| 探针（readiness 与扩容的关系） | https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/ |
| metrics-server 官方仓库 | https://github.com/kubernetes-sigs/metrics-server |
| metrics-server 发布与版本 | https://github.com/kubernetes-sigs/metrics-server/releases |
| 资源指标 API（理解 HPA 读的是什么） | https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/ |
| **Prometheus Adapter**（自定义指标进阶） | https://github.com/kubernetes-sigs/prometheus-adapter |
| Cluster Autoscaler（节点层扩容） | https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler |
| 压测工具 hey | https://github.com/rakyll/hey |
| 练习 | Killercoda 的 HPA / Metrics Server 场景；本地 kind/minikube 复现 |
| 书籍 | 《Kubernetes 权威指南》弹性伸缩章节；《Kubernetes 生产实践》相关章节 |
| 官方课程 | Kubernetes 官方 "Horizontal Pod Autoscaling" 任务文档（含 FAQ） |

---

## 附录 A：文件包清单与操作顺序

### A.1 文件包清单

```
K8s项目G_HPA弹性伸缩/
├── K8s项目G_HPA弹性伸缩实战.md          # 本文档（★ 开头有「执行前必读：环境 4 处变化」）

├── manifests/
│   ├── loadgen.yaml                     # 压测负载生成器（Deployment）
│   └── metrics-server-components.yaml   # ★ 已生成（官方 latest v0.9.0）
│                                        #   已预打两个补丁：镜像指向私有仓库 +
│                                        #   args 加 --kubelet-insecure-tls
│                                        #   → apply 时无需再手工 sed / patch
└── chart-patch/
    └── hpa.yaml                         # ★ 新增到 chart/damo-app/templates/ 的 HPA 模板
```

> **要放进 GitHub 仓库的**：`chart-patch/hpa.yaml` → `chart/damo-app/templates/hpa.yaml`（新增），
> 以及 `values-prod.yaml` 里**追加的 `autoscaling` 段**（`enabled: true`）。
> **项目 E 的 `damo-app-chart/` 不回写**（F 建立的组织原则）。
>
> `manifests/` 属于集群运维产物，建议把 `loadgen.yaml` 也提交 ——
> **面试官看到你连压测脚本都版本化了，对你的印象会不一样**，因为它证明这个演示是可复现的。

### A.2 操作顺序总览

```
阶段 0  执行前检查 6 条（helm release / requests 已设 / Grafana 可达）
   ↓
阶段 2  中转 metrics-server 镜像 → 改镜像地址 + 加 --kubelet-insecure-tls → 装
   ↓     ★ 关卡：kubectl top nodes / top pods 必须有真实数字
   ↓
阶段 3  补写 templates/hpa.yaml → helm template 对比渲染
   ↓
阶段 4  改 values.yaml（enabled: true）→ git push → helm upgrade
   ↓     ★ 关卡：HPA TARGETS 必须是 "x%/75%" 而不是 "<unknown>/75%"
   ↓
阶段 5  部署 loadgen → 三终端观测 → 扩容 2→6
   ↓     同时跑循环 curl 验证零中断
   ↓     ★ 关卡：TARGETS 从 96% 回落到 62%（证明容量真实提升）
   ↓
阶段 6  停压 → 观察 5 分钟冷却窗口 → 平滑缩容到 2
   ↓     可选：参数调优实验
   ↓
阶段 7  Grafana 建 HPA 大盘 → 对照曲线验收 → 导出 JSON
   ↓
收尾   清理 loadgen → 确认 HPA 回到 2 副本稳态 → 整理验收输出 → README 配图
```

> **回退预案**：本项目的改动**完全可逆**——`autoscaling.enabled` 改回 `false` 再 `helm upgrade`，HPA 对象被删除、Deployment 恢复写死 `replicas: 2`，回到项目 E 的状态。metrics-server 是**增量组件**，即使卸载也不影响任何既有服务（只影响 `kubectl top`）。**改造不锁死退路**——延续项目 E 的工程习惯。

---

## 附录 B：HPA 算法与公式速查

### B.1 基本公式

```
desiredReplicas = ceil[ currentReplicas × ( currentMetricValue / desiredMetricValue ) ]
```

| 符号 | 本项目取值 | 说明 |
|------|-----------|------|
| `currentReplicas` | 2（初始） | 当前副本数 |
| `currentMetricValue` | 实测 CPU（如 96m） | 当前每个 Pod 的平均指标值 |
| `desiredMetricValue` | `requests.cpu × 75%` = 100m × 0.75 = **75m** | 目标值 |
| `ceil` | 向上取整 | 结果一定不少于 1 |

**代入示例**（t=30s 那一行）：

```
currentReplicas    = 2
currentMetricValue = 96m（实测）
desiredMetricValue = 100m × 75% = 75m

desiredReplicas = ceil( 2 × (96 / 75) ) = ceil(2.56) = 3
```

**为什么实测 96m 而不是 48m？** 因为 2 个 Pod 各承担一半流量；当总负载需要 6 个 Pod 时，单 Pod 用量会降到约 32m（96 ÷ 3 ≈ 32m），这就解释了 T+90s 之后 TARGETS 回落到 62% 左右的原因。

### B.2 容忍度（Tolerance）

HPA 默认**容忍度 0.1（10%）**，只有当算出的 `desiredReplicas` 与当前副本数差异超过 10% 才动作：

```
| desiredReplicas - currentReplicas | / currentReplicas > 0.1  → 才执行 scale
```

**作用**：防止指标噪声导致 ±1 副本的反复抖动。这是 HPA 除「稳定窗口」之外的第二道防抖机制。生产上可在 `kube-controller-manager` 的 `--horizontal-pod-autoscaler-tolerance` 调整（一般不动）。

### B.3 稳定窗口（Stabilization Window）

| 方向 | 默认值 | 本项目设置 | 行为 |
|------|--------|-----------|------|
| 扩容 | `0`（不回溯） | `0` | 立刻按最新值扩 |
| 缩容 | **300s** | `300` | 取 `max(过去 300s 内所有建议副本数)` 作为缩容目标 |

**「取最大值」的直觉理解**：假设这 5 分钟内 HPA 先后建议 `[6, 6, 5, 4, 3]`，那么缩容目标取 `max = 6`——也就是**不缩**。只有当窗口内所有建议值都变小（`[3, 3, 3, 2, 2]`）时，才会取到 2。所以「5 分钟内忙过一次就不缩」这个说法是准确的行为描述。

### B.4 关键时间参数汇总（面试速答表）

| 参数 | 默认值 | 本项目 | 影响 |
|------|--------|--------|------|
| HPA 同步周期 | 15s | 15s（不可配） | 决定扩容的最快响应 |
| 扩容稳定窗口 | 0s | 0s | 扩容无冷却 |
| 缩容稳定窗口 | 300s | 300s | ★ 缩容慢的根因 |
| 容忍度 | 0.1 | 0.1 | 小幅波动不动作 |
| metrics-server 采集间隔 | ~15–60s（取决于 kubelet 聚合） | 默认 | 指标新鲜度 |
| Prometheus 采集间隔（项目 C） | 15s | 15s | Grafana 曲线粒度 |

> **一句话总结 HPA 的时序**：**15 秒决策一次，扩容即刻生效，缩容先看 5 分钟——所以"扩得快、缩得慢"是 HPA 的默认性格，而这个性格是被刻意设计出来的（为了对抗流量抖动）。**
