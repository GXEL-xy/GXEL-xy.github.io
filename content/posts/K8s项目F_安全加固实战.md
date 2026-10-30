---
title: "K8s 项目 F：安全加固实战（RBAC 最小权限 + NetworkPolicy 默认拒绝）"
date: 2026-10-30T16:42:00+08:00
draft: false
description: "从认证/授权/准入三层盘点集群「裸奔」现状，用 RBAC 把 Jenkins 从 cluster-admin 收到只够操作一个命名空间，再用 NetworkPolicy 做 Ingress + Egress 双默认拒绝后逐条放行。含 8 个真实故障，以及「资源健康 ≠ 业务可用」的判定方法。"
tags: ["k8s 安全加固"]
categories: [项目]
---

> 📦 **项目文件包**：[下载 K8s项目F_安全加固.zip](/files/K8s项目F_安全加固.zip)（含 RBAC / NetworkPolicy / PSA 全部 manifests、Chart 补丁与验收脚本，可直接 apply）。本文件是详细操作手册，两者配合。

> **系列定位**：K8s 运维实习项目路线图第 6 站——A 应用容器化 → B CI/CD → C 监控告警 → D 日志收集 → E Helm 化 → **F 安全加固** → G 弹性伸缩。
> **项目定位**：给现有集群做一次「**从裸奔到有边界**」的安全加固。两条主线：
> ① **RBAC 最小权限**——把「所有 Pod 都用 default ServiceAccount 且自动挂载 token」的默认状态，改成「专用 SA + 不需要就不给身份 + 发布者只够操作一个命名空间」；
> ② **NetworkPolicy 默认拒绝**——把「命名空间内所有 Pod 互通、所有方向全通」的默认状态，改成「**默认全拒 + 逐条放行**」。
> **为什么这个项目值得做**：A–E 五个项目都在证明你「能把东西跑起来」，F 证明你「**知道裸奔的系统危险在哪，并且真的动手收了权**」。面试官问「你的集群安全怎么做的」，答「内网环境不用管」和答「RBAC 最小权限 + 默认拒绝网络策略 + 逐条放行并验证」是两个层级。
> **版本基线**：K8s v1.35（NetworkPolicy `networking.k8s.io/v1`、Pod 安全准入 GA）｜Calico CNI（**tigera-operator 部署**，IPIP 封装，**支持 NetworkPolicy**；组件在 `calico-system` ns，operator 在 `tigera-operator` ns）｜复用项目 B/C/E 全部环境（3 节点 / containerd 2.x / damo-app Release 已存在 / Jenkins 动态 Agent）。
> **前置条件**：① 项目 E 已完成（damo-app 由 Helm 管理——**这一点很重要**，加固后的改动要能通过 `helm upgrade` 固化，而不是手改）；② 项目 G 的 loadgen 已清理（避免压测流量干扰验证）。
> **预计耗时**：3–4 小时（含原理学习与两组前后对比实验）。
> **权威依据**：Kubernetes 官方文档（RBAC 授权 / 网络策略 / Pod 安全准入）、Calico 官方文档。

---

> **📖 怎么用这份文档**
>
> - **前置依赖**：项目 A/B/E 已完成（有业务负载、有流水线、Chart 可发布）；理解 Service/Ingress 基本用法
> - **预计耗时**：10–12 小时（NetworkPolicy 逐条放行最耗时，建议分两次做）
> - **学习路径**：阶段 1 安全三层模型 → 阶段 2 现状盘点（先取证）→ 阶段 3 RBAC 最小权限 → 阶段 4 NetworkPolicy 默认拒绝与放行 → 阶段 5 追加加固 → 阶段 6 综合验证
> - **验收标准**：`verify-hardening.sh` 全绿；跨命名空间访问被拒；容器内看不到 API token；流水线仍能正常发布

## 目录

- [0. 项目概览](#0-项目概览)
- [阶段 1：先懂再做——K8s 安全三层模型与两个默认](#阶段-1先懂再做k8s-安全三层模型与两个默认)
- [阶段 2：现状盘点——你的集群现在有多「裸」](#阶段-2现状盘点你的集群现在有多裸)
- [阶段 3：RBAC 最小权限改造](#阶段-3rbac-最小权限改造)
- [阶段 4：NetworkPolicy 默认拒绝与逐条放行](#阶段-4networkpolicy-默认拒绝与逐条放行)
- [阶段 5：追加加固（可选，至少做一项）](#阶段-5追加加固可选但至少做一项)
- [阶段 6：综合验证与前后对比演示](#阶段-6综合验证与前后对比演示)
- [7. 故障速查表](#7-故障速查表)
- [8. 自测题（含答案）](#8-自测题含答案)
- [9. 面试要点与简历写法](#9-面试要点与简历写法)
- [10. 学习资源（官方文档）](#10-学习资源官方文档)
- [附录 A：文件包清单与操作顺序](#附录-a文件包清单与操作顺序)
- [附录 B：RBAC / NetworkPolicy 语法速查卡](#附录-brbac--networkpolicy-语法速查卡)

---

## 0. 项目概览

### 0.1 目标与产出

| 产出 | 说明 |
|------|------|
| **权限盘点评估** | 一份「当前集群谁有什么权限、谁能访问谁」的实测报告（`kubectl auth can-i` 矩阵 + 连通性实验） |
| **专用最小权限 SA** | `damo-app-sa`（应用无 API 需求 → **不挂 token**）+ `damo-deployer-sa`（发布者，只够操作 damo-app 命名空间） |
| **默认拒绝网络策略** | `damo-app` 命名空间 **Ingress + Egress 双默认拒绝**，再逐条放行到「刚好够用」 |
| **before/after 对比证据** | RBAC 与 NetworkPolicy **各一组**改造前后的实测输出（本项目最大的面试价值） |
| **可运行验收脚本** | `verify-hardening.sh`——只读检查、可反复运行的安全验收脚本 |
| 面试完整故事线 | "我从认证/授权/准入三层盘点了集群，用 RBAC 把 SA 权限收到最小、用 NetworkPolicy 做默认拒绝后逐条放行，并用 before/after 验证每条策略真实生效" |

### 0.2 「裸奔」体现在哪：两组要改造的默认行为

K8s 的设计哲学是「**默认开放**」——对上手友好，对生产危险。本项目要修的就是这两组默认：

```
┌───────────────────── 第①层：身份与权限（RBAC）─────────────────────┐
│                                                                     │
│  现状（裸奔）                          改造后（最小权限）             │
│  ┌────────────────────────┐          ┌────────────────────────┐   │
│  │ Pod 用 default SA       │          │ Pod 用 damo-app-sa      │   │
│  │ ✗ token 自动挂载到容器   │   ──►    │ ✓ automount=false       │   │
│  │ ✗ 容器里能拿到 API 凭证  │          │ ✓ 容器里没有 token       │   │
│  │ ✗ 一旦容器被攻破         │          │ ✓ 攻破了也拿不到集群凭证  │   │
│  │   攻击者拿到集群身份     │          │                        │   │
│  └────────────────────────┘          └────────────────────────┘   │
│                                                                     │
│  发布侧：Jenkins Agent 用什么身份？                                   │
│  现状：可能用 admin 的 kubeconfig（= 集群 root，Jenkins 被攻破全丢）  │
│  改造：damo-deployer-sa 只允许操作 damo-app 命名空间                  │
└─────────────────────────────────────────────────────────────────────┘

┌───────────────────── 第②层：网络边界（NetworkPolicy）───────────────┐
│                                                                     │
│  现状（无策略 = 全通）                    改造后（默认拒绝）          │
│                                                                     │
│   Pod A ──► Pod B      ✅ 通            Pod A ──► Pod B      ❌ 拒   │
│   Pod A ──► 任意 Pod   ✅ 通            Pod A ──► 白名单 Pod  ✅ 通   │
│   Pod A ──► 外网       ✅ 通            Pod A ──► 外网        ❌ 拒   │
│   任何 Pod ──► 任何 Pod ✅ 通            任何 Pod ──► 任何 Pod ❌ 拒   │
│                                                                     │
│  ★ 关键点：K8s 默认没有任何 NetworkPolicy，效果等同于"全允许"。       │
│    你不写策略 ≠ 网络被限制；不写 = 完全不做限制。                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 0.3 架构与数据流（改造后 damo-app 的完整访问路径）

改造后，进入 damo-app 的流量只剩**四条被显式允许的路径**——不多一条：

```
                                  ┌──────────────────────────┐
   ① 你的笔记本                   │  外部 / 集群外            │
      curl :30090                └────────────┬─────────────┘
                                              │ 请求 NodePort 30090
                                              ▼
                                  ┌──────────────────────────┐
                                  │ 节点（work1/work2）        │
                                  │ kube-proxy 做 DNAT        │
                                  │ ★ 源 IP 被 SNAT 成节点 IP  │
                                  └────────────┬─────────────┘
   ┌── kubelet 探针（★ 不进策略判定）            │
   │   目标=本机 Pod，走本机路由直达容器 veth     │
   │   【不经过 cali-* 链】→ 不受默认拒绝影响    │
   │                                            ▼
   │              ┌──────────────────────────────────────────┐
   │              │  NetworkPolicy 准入判定（Calico 执行）      │
   │              │  ┌────────────────────────────────────┐  │
   │              │  │ 默认拒绝（default-deny）            │  │
   │              │  │  ▼ 逐条放行：                       │  │
   │              │  │  ✓ 来源 ∈ 节点网段 11.0.1.0/24     │  │ ← 覆盖①（NodePort SNAT）
   │              │  │  ✓ 来源 pod 带 app=loadgen         │  │ ← ② 同命名空间压测
   │              │  │  ✓ 来源 ns 带 name=ingress-nginx   │  │ ← ③ 走 Ingress 的流量
   │              │  └────────────────────────────────────┘  │
   │              └────────────────────┬─────────────────────┘
   │                                   ▼
   └──────────────────►┌──────────────────────────┐
                       │  damo-app Pod            │
     ┌────────────────►│  （专用 SA，无 token）    │
     │ 出方向（egress）  └──────────────────────────┘
     │ 默认拒绝
     │  ✓ 放行 → DNS(UDP/TCP 53)
     │    kube-system/kube-dns ─────────► CoreDNS（kube-system）
     └────────────────────────────────────────────────────────
```

> ⚠️ **注意上图里 kubelet 探针的位置**：它**不经过** Calico 的策略判定框。因为 `tcpSocket` 探针的目标是**本机 Pod**，
> 流量从宿主机 kubelet 出发经本机路由直达容器 veth，**不跨节点、不进 `cali-*` iptables 链**。
> **这是 2026-09-23 的实测结论，推翻了"默认拒绝会误杀探针"的常见说法** —— 详见 §4.3.2。

**四条放行规则的由来（这张表是面试叙事的骨架，★ 已按实测归因修正）**：

| # | 放行什么 | 为什么必须放行 | 不放行的后果 | 怎么想到的 |
|---|---------|--------------|-------------|-----------|
| ① | 来源 = 节点网段 `11.0.1.0/24` | 放行**源 IP 确实落在节点网段**的流量。配合本项目的 **`externalTrafficPolicy: Local`**，外部 NodePort 客户端（如宿主机 `11.0.1.1`）的**真实源 IP 被保留**，从而命中这条规则 | 外部 NodePort 访问不通 | 默认拒绝后**先看外部访问**——它才是真正的受害者 |
| ② | 来源 = `app=loadgen`（同命名空间） | 压测/联调需要从集群内访问 | 无法压测与内网验证 | 项目 G 的压测链路需要 |
| ③ | 来源 = `ingress-nginx` 命名空间 | 若 damo-app 也走 Ingress 暴露 | Ingress 路径不通（本项目主用 NodePort，此项为通用性预留） | 项目 A 的 ingress 链路 |
| ④ | 出方向 → `kube-system/kube-dns` 的 53 端口 | **DNS 解析**（访问任何 Service 域名都需要） | `damo-app.damo-app.svc.cluster.local` 解析失败 | 默认拒绝 egress 后**第一个必查项** |

> **★★ 关于规则① 的两条修正（都是实测，面试高价值，详见 §4.4 的抓包证据）**
> 1. **「NodePort SNAT 后源 IP 变成节点 IP」这个说法在本集群不成立。**
>    抓包实测：Windows 客户端的 SYN 源是 `11.0.1.1`，但出 IPIP 隧道 `tunl0` 时被 **MASQUERADE 成 `10.244.219.64`
>    （落在 Pod 网段）**——所以 `ipBlock: 11.0.1.0/24` **永远匹配不到 NodePort 流量**，外部访问全断。
>    **解法：把 Service 设成 `externalTrafficPolicy: Local`**，NodePort 不再 SNAT，源 IP 保留为真实客户端 IP，
>    这条 `ipBlock` 才真正有意义。**代价**：流量只落到接收节点上的 Pod，所以 master 上无 Pod →
>    `11.0.1.128:30090` 不可用，外部访问要打 `11.0.1.129:30090` / `11.0.1.130:30090`。
>    ★ **面试话术：NodePort 会把客户端身份抹掉，`ipBlock` 因此写不出精确规则——这正是生产更推荐
>    Ingress/LoadBalancer 而非 NodePort 的原因之一。**
> 2. **网上普遍说这条是为了"放行 kubelet 探针"，但在本集群（Calico IPIP）探针根本不需要放行** ——
>    它不走策略判定链（§4.3.2）。把这两条因果关系都讲对，比照抄流传的说法更有说服力。

> **一句话记住**：**默认拒绝不难，难在"逐条放行到刚好够用"**——放多了等于没做，放少了服务挂。上面这张「四条规则表」就是本项目最有价值的产出。

### 0.4 关键技术决策（面试可讲）

| 决策 | 选择 | 理由 |
|------|------|------|
| **SA 是否需要 API 权限** | 应用 Pod（nginx）**完全不需要** → `automountServiceAccountToken: false` | 静态页服务不调用 K8s API。不挂 token 就没有凭证可泄漏——**最小权限的极致：不是少给权限，而是不给身份** |
| **发布 SA 的粒度** | `damo-deployer-sa` + 命名空间级 Role（不给 ClusterRole） | 发布只需操作 `damo-app` 一个命名空间；ClusterRole 会让一次 Jenkins 被攻破等于整个集群沦陷。**命名空间级 Role 是"爆炸半径"的天然边界** |
| **是否替换 Jenkins 现有权限** | ✅ 替换（但**先并存验证再切换**） | 直接换掉有把流水线搞挂的风险；先建新 SA、验证够用、再切调用方、最后回收旧权限——**生产改权限的标准节奏** |
| **网络策略的落地顺序** | **先只 deny Ingress → 验证 → 再加 deny Egress** | 一次性全拒会同时踩「探针」和「DNS」两个坑，无法定位是哪个。**分步做，每步都能确定原因** |
| **默认拒绝的写法** | `podSelector: {}` + `policyTypes: [Ingress,Egress]` + **无任何规则** | 官方标准的"全拒"写法。**注意：不写 rules 才是全拒**；漏写 `policyTypes` 会导致某方向不生效 |
| **按 IP 还是按标签放行** | 优先**标签**（`podSelector`/`namespaceSelector`），仅在必要时用 `ipBlock` | 标签是"身份"，IP 是"地址"——Pod IP 会变。**用标签的策略自解释且可移植**；节点网段是"必要之恶"（探针流量没有 Pod 身份） |
| **是否用 Calico 专有 CRD** | ❌ 只用标准 `NetworkPolicy` | 可移植（换 CNI 不用重写）。Calico 的 `GlobalNetworkPolicy`/`NetworkSet` 更强但绑定 Calico——**知道它的存在，但故意不用** |
| **Pod 安全（PSA）怎么上** | 先 `warn` 观察，再考虑 `enforce` | 直接 `enforce: restricted` 会让现有 Pod（root 用户、可写根文件系统）全部被拒。**渐进式加固**才是生产做法 |
| **加固怎么持久化** | 全部写进 Helm Chart（开关驱动，默认关闭） | 手 apply 的策略不在交付物里，命名空间重建就丢；写进 Chart 才能随发布收敛，且不破坏新人首次部署 |

### 0.5 资源与端口现状

| 项 | 值 | 说明 |
|----|-----|------|
| CNI | **Calico（tigera-operator 部署，IPIP）** | ✅ 支持 `NetworkPolicy`（Flannel 默认不支持——这是前提，先确认）。**operator 模式：`calico-node` 等组件在 `calico-system` ns，operator 自身在 `tigera-operator` ns，都不在 `kube-system`**；配置入口是 CR（`Installation` / `FelixConfiguration`）而不是 ConfigMap |
| 命名空间 | `damo-app` | 加固主战场（RBAC + NetworkPolicy） |
| 命名空间 | `kube-system` | DNS 出口放行的目标 |
| 新增 NodePort | **无** | 安全加固不引入新服务 |
| 镜像（★ 需先确认） | **`busybox:1.36` 不在私有仓库**（本项目所有 `netcheck` 命令都用它做连通性验证） | ⚠️ 原稿写"只需 busybox（项目 G 已中转）"是**错误前提**——项目 G 未执行，私有仓库里很可能没有它。**开做前先在 master 上中转一次**（见下方一键命令），或改用「直接 exec 进 damo-app 自己的容器」这条零新增镜像的路（§2.2 首选方案） |

> **一键中转 busybox（在 master 执行，走项目 B 的中转铁律）**
> ```bash
> docker pull busybox:1.36
> docker tag  busybox:1.36 11.0.1.128:30000/busybox:1.36
> docker push 11.0.1.128:30000/busybox:1.36
> curl -s http://11.0.1.128:30000/v2/busybox/tags/list   # 期望看到 1.36
> ```
| 新增集群对象 | ServiceAccount × 2、Role × 1、RoleBinding × 1、ClusterRole × 1、ClusterRoleBinding × 1、NetworkPolicy × 4、Secret × 1（§3.5 的长期 token，可选） | 全部轻量对象，无资源开销 |
| **探针 `timeoutSeconds`（★ 可选，属"显式化"而非"收紧"）** | 三探针可显式写 `timeoutSeconds: 1` | **理由不是"否则看不到现象"**（本集群探针不受策略影响，见 §4.3.2）。★★ 更要紧的是**纠正一个易错点**：`Probe.timeoutSeconds` 的 K8s 默认值**本来就是 1 秒**（不是 30 —— 30 是 `terminationGracePeriodSeconds` 的默认值）。所以显式写 1 **在功能上不改变当前行为**，它做的是**把隐含的 API 默认值显式化**（可读、可评审、防被改坏、可按环境覆写）。详见 §4.3.3 |
| 现有端口（勿动） | 30000 registry ｜ 30080 Jenkins ｜ 31000 Jenkins-jnlp ｜ 30090 damo-app ｜ 30091 Prometheus ｜ 30300 Grafana ｜ 30093 Alertmanager ｜ 32541 ingress-nginx | — |

> ### ⚠️⚠️ 访问入口变更（★ 阶段 4 之后必须知道，2026-09-23 起，已实测验证）
>
> 本项目在阶段 4 把 Service 设为 **`externalTrafficPolicy: Local`**（原因见 §4.4 的抓包证据）。
> 副作用是：**NodePort 只被下发到"有业务 Pod 的节点"**。本集群 master 上不跑业务 Pod，因此：
>
> | 访问地址 | 变更前 | 变更后（**实测 2026-09-23**） |
> |---|---|---|
> | `http://11.0.1.128:30090/`（master） | ✅ 通 | ❌ `000`（**refused**，master 无 Pod） |
> | `http://11.0.1.129:30090/`（work2） | ✅ 通 | ✅ **`200`**（work2 有 Pod） |
> | `http://11.0.1.130:30090/`（work1） | ✅ 通 | ✅ **`200`**（work1 有 Pod） |
> | Windows 宿主机（真实外部客户端） | ✅ 通 | ✅ **`200`** ★ 规则① 生效的直接证据 |
>
> **★ 推荐用"自动挑一个跑着 Pod 的节点"，别写死 IP**（因为项目 G 的 HPA 会让 Pod 漂移，
> 写死某个节点 IP 会间歇性失败）：
> ```bash
> NODE=$(kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
>          -o jsonpath='{.items[0].status.hostIP}')
> curl --noproxy '*' -s --max-time 5 "http://$NODE:30090/" | grep -o 'Build #[0-9]*'
> ```
> 需要验证"全节点可达性"时才用上面的三行 for 循环。
>
> 集群内访问（`damo-app.damo-app.svc.cluster.local`、ClusterIP）**不受影响**——
> `externalTrafficPolicy` 只管 NodePort 这一段。
>
> **⚠️ 连带影响（跨项目）**：项目 C / G 的文档里也有 `11.0.1.128:30090` 的引用，
> 同样需要改成"有 Pod 的节点"（已在各自的排障记录中标注）。

### 0.6 执行前检查清单（10 分钟）

```bash
# ① 确认 CNI 支持 NetworkPolicy（★ 整个项目的前提，不支持就别做）
#    ⚠️ 本项目 Calico 由 tigera-operator 部署：组件在 calico-system、operator 在 tigera-operator，
#       都不在 kube-system —— 查错命名空间会得出"CNI 不存在"的错误结论
kubectl -n calico-system get pods -o wide
# 期望：calico-node（DaemonSet，每节点 1 个）+ calico-kube-controllers + calico-typha 全部 1/1 Running
kubectl -n tigera-operator get deploy
# 期望：tigera-operator  1/1

# ①-2 operator 模式最权威的健康信号（manifest 模式没有这个）
kubectl get tigerastatus
# 期望：NAME 列各项 AVAILABLE=True、DEGRADED=False

# ①-3 确认是 Calico 而非 Flannel
kubectl get crd | grep -i calico | head -5
# 期望：出现 calico 相关 CRD，且包含 installations.operator.tigera.io（这是 operator 模式的标志）

# ①-4 读 CR 确认封装方式与 IP 池（别靠记忆）
kubectl get installation default -o yaml | grep -A12 calicoNetwork
# 期望：encapsulation: IPIP、CIDR 10.244.0.0/16

# ①-5 ★ tigera-operator 特有的"策略好像不生效"第一嫌疑：failsafe 端口
kubectl get felixconfiguration default -o yaml | grep -A8 failsafe
# 含义：failsafeInbound/OutboundHostPorts 里列的【宿主机端口】（默认含 SSH 22、BGP 179、
#       API Server 6443 等）上的流量被【无条件放行】，不受任何 NetworkPolicy 约束。
#       验证策略效果时，别选这些端口做目标，否则会以为策略没生效。

# ② 项目 E 的 release 在（加固要能通过 helm 固化）
helm list -n damo-app

# ③ 当前应用正常（改造前基线）
curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'

# ④ 项目 G 的压测负载已清理（避免干扰）
kubectl -n damo-app get deploy loadgen
# 期望：NotFound，或 READY 0/0

# ⑤ 记录当前 Pod 用的 ServiceAccount（改造前后对比要用）
kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
  -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName

# ⑥ 记录当前集群的 NetworkPolicy 数量（预期 0，这是"裸奔"的直接证据）
kubectl get networkpolicy -A
# 期望：No resources found. ★ 这正是要展示的"改造前"状态

# ⑦ ★ 连通性验证工具就位（否则本项目的 netcheck 命令全部会 timed out）
curl -s http://11.0.1.128:30000/v2/busybox/tags/list
# 期望：能看到 1.36。没有就先中转（命令见 §0.5），或全程改用 §2.2 方案 A
#       （直接 exec 进 damo-app 容器，零新增镜像）
kubectl -n damo-app exec -it deploy/damo-app -- sh -c 'wget --help >/dev/null 2>&1 && echo "wget OK"; nc --help >/dev/null 2>&1 && echo "nc OK"'

# ⑧ ★ 探针 timeoutSeconds 预检（健壮性建议项 —— 见 §4.3.3）
kubectl -n damo-app get deploy damo-app \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\treadiness="}{.readinessProbe.timeoutSeconds}{"\tliveness="}{.livenessProbe.timeoutSeconds}{"\n"}{end}'
# 期望：每个容器 readiness=1 liveness=1
# ★ 若为空或 30：建议按 §4.3.3 收敛（低风险高收益的健壮性投资）。
#   ⚠️ 但它【不是】阶段 4.3 能看到现象的前提 —— 本集群探针流量不进 Calico
#      策略链，无论超时多少，Pod 都不会被默认拒绝影响（实测 2026-09-23）。

# ⑨ ★★ 代理环境检查（本项目最阴的坑，务必先做 —— 见下方「坑 0」）
#    本集群节点与宿主机装了 HTTP 代理（11.0.1.1:7890 / 192.168.5.1:7890），
#    而 curl 会读 http_proxy，把【集群内地址】的请求错误地发给代理 → 全部超时。
#    症状极具误导性：curl 不通、但 ping 通；删了策略也不通 → 会让人误判"策略没生效"。
echo "http_proxy  = $http_proxy";  echo "HTTP_PROXY  = $HTTP_PROXY"
echo "no_proxy    = $no_proxy";    echo "NO_PROXY    = $NO_PROXY"
# ⚠️ curl 读的是【小写】http_proxy / no_proxy。若小写被 shell 覆盖成不含
#    集群网段的版本，大写 NO_PROXY 写得再对也没用（curl 不看它）。
#    典型冲突（本集群实测）：shell 覆盖层 no_proxy=localhost,127.0.0.1,::1,192.168.5.0/24
#    → 集群网段全丢 → 所有集群内 curl 全超时。

# ⑨-2 ★ 正确的连通性判定方式：强制绕过代理（比 export 更可靠，一次性生效）
curl --noproxy '*' -s -o /dev/null -w '%{http_code}\n' --max-time 5 http://11.0.1.128:30090/
# 期望：200。★ 本项目所有 curl 连通性测试都建议带 --noproxy '*'，否则结论不可信。
# ❌ 反例（必产出假象，不要用）：curl -s --max-time 5 http://11.0.1.128:30090/
#    它可能什么都没输出（错误被 -s 吞掉），让人以为是"策略拦住了"。

# ⑨-3 若确认是小写 no_proxy 被覆盖，两种解法（任选其一）
#   临时（当前会话）：unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
#   永久（写进 ~/.bashrc 末尾，放在代理客户端启动脚本【之后】）：
#     export no_proxy="localhost,127.0.0.1,::1,192.168.5.0/24,11.0.1.0/24,10.96.0.0/12,10.244.0.0/16,*.local"
#     export NO_PROXY="$no_proxy"
#   ⚠️ 排查覆盖来源：grep -rn "7890" ~/.bashrc ~/.profile /etc/profile.d/ 2>/dev/null
#   ⚠️ 注意大小写两套变量可能并存且值不同，判定"谁生效"要看你用什么工具：
#        curl / wget → 只看小写；部分工具只认大写。最稳是两套写成一样。
```

> ### 坑 0：代理环境让所有「不通」结论失真（★ 2026-09-23 实战踩坑，写进来避免重蹈）
>
> **现象**：阶段 4 验证放行时，`curl http://11.0.1.128:30090/` 无输出、`curl` Pod IP 无输出、
> 「在节点上也 curl 不通」；但同一时刻 `ping 10.244.123.6` 却**通**（0.2ms）。
> 更难缠的是：把两条 NetworkPolicy 都删了，curl **依然不通** —— 极易得出「跟策略无关，是网络本身坏了」的错误结论。
>
> **真凶**：`curl -v` 输出暴露 ——
> ```
> * Uses proxy env variable http_proxy == 'http://192.168.5.1:7890'
> *   Trying 192.168.5.1:7890...
> * Connection timed out after 5001 milliseconds
> ```
> curl **根本没连 Pod**，而是把请求发给了代理。ping 通是因为 ICMP 不读 `http_proxy`——
> 这正是「ping 通 curl 不通」这个矛盾组合的成因。
>
> **为什么这个坑特别贵**：
> 1. 它伪装成「策略生效」，而实际策略可能完全正确；
> 2. 它让排查方向跑偏到 CNI / 路由 / conntrack，浪费大量时间；
> 3. 删策略后仍不通，会彻底误导归因（本次就差点误判「与 NetworkPolicy 无关」）。
>
> **铁律**：**本项目所有连通性测试，curl 一律加 `--noproxy '*'`；不确定就用 `wget`（不读代理变量）。**
> 判定「到底是服务挂了还是流量被拦」时，优先看**错误语义**：
> `Connection refused` = 端口没人听（服务问题）；`Connection timed out` = 静默丢包（可能被策略拦，也可能是代理），
> 此时**必须先排除代理**再下结论。

> **⚠️ 关于阶段 4.3 的一条重要预期修正（实测结论）**：
> 网上广泛流传「默认拒绝会误杀 kubelet 探针 → Pod 不 Ready → Endpoints 为空的连环故障」，
> **但在本集群（Calico IPIP + tigera-operator）实测不成立**：策略生效后 Pod 依然 `READY 1/1`、Endpoints 仍有 Pod IP。
> 原因和正确的验证方法见 **§4.3.2 / §4.3.4**。
> **本阶段真正能演示的是**：`外部访问断` + `集群内访问从 refused 变 timed out`（静默丢包）——**判据是连接语义，不是 Pod 状态**。

> ⚠️ **如果 ① 显示的是 Flannel，请停下**：Flannel 默认**不实现** NetworkPolicy——写了策略也不生效，验证时会出现「策略明明存在但流量还是通」的诡异现象（新手最容易卡住的地方）。需要先换成 Calico/Cilium，或额外安装 Calico 的策略控制器。

---

## 阶段 1：先懂再做——K8s 安全三层模型与两个默认

### 1.0 开胃故事：一栋「默认不锁门」的写字楼（先建立直觉）

把 K8s 集群想成一栋写字楼，楼里每家公司（Pod）都有自己的办公室。

**第一层：门禁卡（认证 Authentication）——「你是谁？」**

大楼入口有保安，你刷卡，保安认卡不认人。K8s 里这张卡可以是：人拿的（证书 / Token / OIDC，比如你 `kubectl` 用的 kubeconfig）；**程序拿的（ServiceAccount 的 token，装在 Pod 里自证身份）**。

**第二层：工牌权限（授权 Authorization）——「你能进哪些门？」**

进了大楼，你的工牌决定你能去几楼、能开哪间办公室。K8s 里这层主要是 **RBAC**：谁（Subject）能对什么（Resource）做什么（Verb）。

**第三层：房间规矩（准入控制 Admission）——「你带的行李合规吗？」**

就算你有权限进这间办公室，保安还要检查你带进来的东西：不能带汽油桶（privileged 特权容器）、不能把窗户焊死（挂载宿主根目录）……K8s 里这层包括 **Pod 安全准入（PSA）**、ResourceQuota、LimitRange 等。

> **三层的记忆方式**：**认证 = 你是谁；授权 = 你能干啥；准入 = 你这么干合规吗。** 三层**串联**，任一层拒绝就失败。

**而这栋楼的两个致命默认**：

| 默认 | 后果 | 类比 |
|------|------|------|
| **每个办公室都自动配一张门禁卡**（SA token 自动挂载） | 只要有人溜进任何一个办公室，就能拿到一张能进机房的卡 | `damo-app` 容器被 RCE，攻击者直接读到 token 去访问 API Server |
| **所有办公室之间的门都不锁**（无 NetworkPolicy = 全通） | 一个办公室被攻破，整层楼随便走 | `damo-app` 被攻破 → 直连 Prometheus、Loki、乃至 Jenkins |

**所以本项目的两件事，就是「收卡」和「装锁」。**

### 1.1 第一层的两种身份（面试必区分）

| 维度 | User（用户） | ServiceAccount（服务账号） |
|------|-------------|--------------------------|
| 代表谁 | 人 | 程序（Pod 内的进程） |
| 存在哪 | **K8s 里没有 User 对象**（认证后 K8s 只知道"这个名字"） | **是 K8s 里的真实对象**（`kubectl get sa` 能看到） |
| 怎么创建 | 由认证方式决定（证书/OIDC），不能 `kubectl create user` | `kubectl create serviceaccount` 或 YAML |
| 归属 | 全局（ClusterRoleBinding 用） | **属于某个命名空间**（`system:serviceaccount:<ns>:<name>`） |
| 主体写法 | `name: alice` | `name: damo-app-sa` + **`namespace: damo-app`** |
| 本项目用途 | 你 `kubectl` 操作 | **Pod 的身份、Jenkins 发布者的身份** |

> **高频混淆点**：「SA 的 token 是长期凭证还是临时凭证？」
> **答**：看版本。**K8s 1.24 之前**，创建 SA 会自动生成一个**永不过期**的 Secret，挂载进 Pod——历史上最大的安全隐患之一（token 泄漏即永久失守）。**K8s 1.24 起**，SA 不再自动创建 Secret；Pod 挂载的是 kubelet 通过 **TokenRequest API** 申请的、**有有效期（默认 1 小时）、绑定到具体 Pod、Pod 删除即失效**的投影卷。**1.35 上默认就是后者**。这个演进本身是很好的面试素材：「**长期静态凭证 → 短期绑定凭证**，是云原生安全的通用趋势」。

### 1.2 RBAC 的四要素（背下来）

记住「**谁 → 绑到 → 什么角色 → 能干什么**」：

| 对象 | 作用域 | 回答什么问题 | 本项目实例 |
|------|--------|-------------|-----------|
| **Role** | **命名空间** | 「能干什么」（一组规则：verbs × resources） | `damo-deployer`（只能操作 `damo-app` ns 的特定资源） |
| **ClusterRole** | **集群** | 同上，但作用域全局（也用于非命名空间资源如 node/namespace） | `damo-deployer-ns-reader`（只读 namespaces） |
| **RoleBinding** | **命名空间** | 「谁绑到这个角色」 | `damo-deployer-binding` |
| **ClusterRoleBinding** | **集群** | 同上，全局 | `damo-deployer-ns-reader-binding` |

**四种绑定组合（必须背，面试常问）**：

| RoleBinding 引用 | 效果 | 说明 |
|-----------------|------|------|
| **Role**（同命名空间） | ✅ 标准用法：只在该命名空间生效 | 「我给 damo-app 建个 Role，然后绑定」 |
| **RoleBinding → ClusterRole** | ✅ 常见进阶：把集群级角色**限制到单个命名空间** | 「只想给某人 `view` 权限，但只在 damo-app 里」——**收敛作用域的正确姿势** |
| ClusterRoleBinding → ClusterRole | ⚠️ 全局生效，慎用 | `cluster-admin` 就是这种 |
| ClusterRoleBinding → Role | ❌ **不合法**（会报错） | `Role` 是命名空间资源，不能全局绑定 |

> **★ 面试高价值点**：「RoleBinding 可以绑定 ClusterRole」很多人不知道。它的用途是：**ClusterRole 定义"能干什么"，RoleBinding 决定"在哪能这么干"**。所以「**作用域最小化**」和「**权限最小化**」是两个独立维度：后者是少给 verb，前者是少给范围。

### 1.3 Subject 的三种类型与内置组

```yaml
subjects:
  - kind: User                  # 人（K8s 里无对象）
    name: alice
    apiGroup: rbac.authorization.k8s.io
  - kind: Group                 # 组（如 system:masters）
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount        # ★ 本项目主角
    name: damo-deployer-sa
    namespace: damo-app         # ★ SA 必须带 namespace，否则报错
```

| 内置组 | 含义 | 与本项目的关系 |
|--------|------|--------------|
| `system:authenticated` | 所有通过认证的身份 | 批量授权时的常用 subject |
| `system:serviceaccounts` | 集群内所有 SA | ⚠️ **一旦有角色绑到这个组，所有 Pod 立刻共享该权限**——排查权限放大必看 |
| **`system:masters`** | **超级管理员组（break-glass）** | ⚠️ **绕过 RBAC**、唯一的例外——**删任何绑定都撤不掉它**。kubeadm 1.29+ 起它**只留给 `super-admin.conf`**（应急用），不再是日常 admin 证书的组 |
| **`kubeadm:cluster-admins`** | **日常集群管理员组（kubeadm 1.29+ 新增）** | ★ **本集群的 admin kubeconfig 就属于这个组**（`kubectl auth whoami` 实测：`kubernetes-admin` / `[kubeadm:cluster-admins system:authenticated]`）。它**通过 ClusterRoleBinding 拿到 `cluster-admin`，受 RBAC 约束、理论上可撤销** —— 这是官方刻意的改进（把"不可撤销"换成"可撤销"）。⚠️ **但别真去删**：你 master 上用的就是这同一份证书，删了等于把自己锁在门外（除非 `super-admin.conf` 还在） |

### 1.4 RBAC 是纯白名单：默认拒绝

```
请求 → 认证（你是谁）→ 查找所有匹配你的 RoleBinding/ClusterRoleBinding
     → 汇总所有允许的规则 → 有允许？放行。没有 → 403 拒绝
```

**推论（★ 面试点）**：**新增的 ServiceAccount 默认没有任何权限**（连 `get pods` 都不行）。所以「default SA 很危险」要说精确：

- `default` SA **本身没有 RBAC 权限**；
- 但它**有 token**，而 token 意味着**一个可用的集群身份**。危险有三层：
  1. 一旦集群里**任何**角色被错误地绑到 `system:serviceaccounts` 组或 `default` SA，所有 Pod 立刻获得该权限；
  2. token 可用于**信息收集**（探测 API 资源、集群版本、命名空间是否存在）；
  3. **横向移动**：拿到 token 后若发现过度授权的 RoleBinding，就能提权。

> **准确的面试表述**：「`default` SA 默认没有 RBAC 权限，但**容器里挂载的 token 是一个有效的集群身份**。它的风险不在于"现在能干什么"，而在于"它给了我一个立足点"。所以我的做法是——**这个应用不需要 API 访问，那就干脆不挂 token**（`automountServiceAccountToken: false`），把立足点直接消灭，而不是纠结该给它多少权限。」

### 1.5 NetworkPolicy 的四要素与三个必须理解的行为

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: xxx
  namespace: damo-app          # ★ 策略属于哪个命名空间
spec:
  podSelector: {}              # ★ ① 选中哪些 Pod（{} = 本命名空间全部）
  policyTypes:                 # ★ ② 管控哪些方向
    - Ingress
    - Egress
  ingress:                     # ③ 入方向白名单（不写 = 全拒）
    - from: [...]
      ports: [...]
  egress:                      # ④ 出方向白名单（不写 = 全拒）
    - to: [...]
      ports: [...]
```

**三个必须理解的行为**：

| 行为 | 说明 | 实操含义 |
|------|------|---------|
| **① 默认全通** | 命名空间里**没有任何** NetworkPolicy 时，所有 Pod 双向全通 | 「不写策略」≠「安全」，而是「完全没限制」。**先 `kubectl get netpol -A` 确认** |
| **② 未被选中的 Pod 不受影响** | 只有 `podSelector` 选中的 Pod 受约束 | 「默认拒绝」要生效，`podSelector` 必须是 `{}`（全选） |
| **③ 有状态（stateful）** | 允许了入方向连接，其**回包自动放行**，无需在 egress 里再写 | 加了 egress 默认拒绝后，**入方向请求的响应仍然能返回**；真正被挡的是**主动发起**的出方向连接（DNS、外网调用） |

**`policyTypes` 与 rules 的组合（★ 高频错误）**：

| 写法 | 实际效果 | 是否正确 |
|------|---------|---------|
| `policyTypes: [Ingress]` + 有 `ingress:` 规则 | 只限制入方向 | ✅ |
| `policyTypes: [Ingress, Egress]` + 只有 `ingress:` 无 `egress:` | 入方向按白名单，**出方向全拒** | ✅ 这就是"入白名单+出全拒"的简写 |
| **不写 `policyTypes`**，只写 `ingress:` | K8s 推断为 `[Ingress]`；只写 `egress:` 推断为 `[Egress]` | ⚠️ 容易漏管控方向 |
| `policyTypes: [Ingress, Egress]` + **两个方向都不写规则** | **双向全拒** | ✅ 标准 default-deny 写法 |

> **最阴的坑**：想"默认拒绝全部"，写了 `podSelector: {}` 和 `policyTypes: [Ingress, Egress]`，但误写成 `ingress: [{}]`（**含一个空规则对象**）——那表示「**允许所有来源**」！正确写法是**根本不写 `ingress:` 段**（或写 `ingress: []` 空数组）。**一个括号之差，从全拒变成全通。** 所以 apply 之后务必 `-o yaml` 回看确认。

### 1.6 Calico 与 NetworkPolicy 的关系（排错必备）

| 角色 | 谁做 |
|------|------|
| **定义策略** | K8s API（`NetworkPolicy` 对象）——你写的 YAML |
| **执行策略** | **CNI 插件**（本项目是 Calico 的 `felix` 组件，每个节点上都有） |
| **策略不生效** | 99% 是 CNI 不支持/组件异常，而不是 YAML 写错（YAML 写错通常被 API 校验拦下） |

```bash
# 排错三板斧
kubectl get networkpolicy -n damo-app -o yaml          # ① 定义对不对（回看有没有意外写空）
kubectl -n calico-system logs ds/calico-node --tail=50 # ② 执行者有没有报错（★ 是 calico-system，不是 kube-system）
kubectl get tigerastatus                               # ②b operator 模式先看整体健康（DEGRADED=True 就不用往下查了）
kubectl get endpoints -n damo-app damo-app             # ③ 服务侧是否正常（排除业务问题）
```

**为什么命名空间是 `calico-system`**：本项目的 Calico 是 **tigera-operator 部署**的（不是 `kubectl apply -f calico.yaml` 那种 manifest 模式），两者差别要能讲清：

| 维度 | operator 模式（本项目） | manifest 模式 |
|---|---|---|
| **组件落点** | `calico-node`/`calico-kube-controllers`/`calico-typha` 在 **`calico-system`**；operator 自身在 **`tigera-operator`** | 全部在 **`kube-system`** |
| **配置入口** | **CR**：`Installation`（IP 池/封装）、`FelixConfiguration`（Felix 行为）、`BGPConfiguration` 等 | ConfigMap + DaemonSet 里的环境变量 |
| **健康检查** | `kubectl get tigerastatus`（`calico`/`ippools` 等条目 `AVAILABLE=True`） | 只能看 Pod 状态和日志 |
| **升级方式** | 改 `Installation` CR 或升级 operator 镜像，**由 operator 驱动滚动** | 手动替换 manifest 再 apply |
| **改封装/IP 池** | 改 CR 的 `spec.calicoNetwork.ipPools` | 改 ConfigMap 的 `CALICO_IPV4POOL_IPIP` 等环境变量 |
| **Tigera 官方建议** | ✅ **推荐**（能自动管理生命周期） | 仅在需要高度定制底层资源时用 |

> **面试话术**：「我的 Calico 是用 tigera-operator 装的，所以它的配置面是 CR 而不是 ConfigMap——查组件要去 `calico-system` 而不是 `kube-system`，健康状态用 `kubectl get tigerastatus` 一眼看全。这个区别在实际排障里很容易踩：**按老教程去 `kube-system` 找 `calico-node` 会找不到，然后误判成『CNI 没装/不支持 NetworkPolicy』**。」

> **⚠️ failsafe 端口：一个"策略看起来没生效"的经典误判源**
> `FelixConfiguration/default` 里的 `failsafeInboundHostPorts` / `failsafeOutboundHostPorts` 会**无条件放行**一批宿主机端口（默认含 SSH 22、BGP 179、API Server 6443 等）。如果你拿这些端口做策略验证的目标，会发现"策略加了也照样通"。**验证默认拒绝时，务必换一个不在 failsafe 列表里的端口。**

> **Calico 的进阶能力（知道但不用）**：`GlobalNetworkPolicy`（集群级，可管控 HostEndpoint）、`NetworkSet`（IP 集合复用）、`StagedNetworkPolicy`（先预览再生效，类似 dry-run）。本项目选择**只用标准 `NetworkPolicy`**——理由是**可移植性**：换成 Cilium/Antrea 也能用同一份 YAML。面试说出这个取舍，比炫耀用了 Calico 私有 CRD 更有分量。

### 1.7 阶段自测

- [ ] 能说出 K8s 安全三层各自解决什么问题
- [ ] 能说清 User 与 ServiceAccount 的关键差异（尤其"SA 是真实对象、属于命名空间"）
- [ ] 能背出 RBAC 四要素，并解释「RoleBinding 可以绑 ClusterRole」的用途
- [ ] 能解释为什么"default SA 没有权限"却仍然危险（立足点视角）
- [ ] 能说出 NetworkPolicy 的三个默认行为（全通 / 未选中不管 / 有状态）
- [ ] 能说出 `policyTypes` 不写会怎样，以及 `ingress: []` 与 `ingress: [{}]` 的区别
- [ ] 能解释"策略不生效"该往哪查（CNI 执行者，不是 YAML）

---

## 阶段 2：现状盘点——你的集群现在有多「裸」

> **本阶段目标**：产出一份**加固前的评估报告**。这份报告本身就是面试素材——它证明你是「先评估再动手」的工程师，而不是照抄一个 YAML 就交差。

### 2.1 检查一：网络策略现状

```bash
# 全集群的 NetworkPolicy 数量（预期：0，这就是"裸奔"）
kubectl get networkpolicy -A
kubectl get networkpolicy -A --no-headers 2>/dev/null | wc -l

# ★ 保存这一步的输出——加固之后这些「通」就不存在了，无法补测
```

**面试话术**：「我第一件事是盘点现状，结果发现全集群**零条** NetworkPolicy。这意味着三个命名空间（damo-app / monitoring / kube-system）之间以及内部，所有 Pod 双向全通。具体风险是：**`damo-app` 这个对外暴露的应用一旦被攻破，攻击者可以直接访问 Prometheus 的 9090、Loki 的 3100、以及 Jenkins 的 8080**——这就是典型的"横向移动"路径。」

### 2.2 检查二：命名空间的真实连通性（用实验证明，不靠推测）

**先选一条路，进到「damo-app 命名空间里的一个容器」：**

**方案 A（★ 首选，零新增镜像）**：直接 `exec` 进业务自己的容器 —— 它本来就是那个「对外暴露的应用」，**用它发起横向访问，叙事最真实**（不是我另起一个测试 Pod，是应用本身）。

```bash
kubectl -n damo-app exec -it deploy/damo-app -- sh
# nginx:1.27-alpine 基于 Alpine，自带 busybox 的 wget / nc / nslookup，够用
```

**方案 B**：起一个临时 Pod（需要 busybox 已在私有仓库，见 §0.5 的中转命令）

```bash
kubectl -n damo-app run netcheck -it --restart=Never \
  --image=11.0.1.128:30000/busybox:1.36 -- sh
# ★ 这里【故意不加 --rm】：出问题时 Pod 会留下来，方便 describe 看原因
#   用完手动删： kubectl -n damo-app delete pod netcheck
```

> ⚠️ **如果报 `error: timed out waiting for the condition`**
> 这句话的含义是「**Pod 在超时内没能进入 Running 状态**」——**不是"网络不通"**。而加了 `--rm` 会把失败的 Pod 立刻删掉，**现场一并消失**，所以你只会看到这一句、看不到真正原因。
> ```bash
> kubectl -n damo-app describe pod netcheck | tail -20                       # ★ 看 Events，最关键
> kubectl -n damo-app get pod netcheck -o jsonpath='{.status.containerStatuses[0].state}'; echo
> ```
> | Events 显示 | 原因 | 处理 |
> |---|---|---|
> | `ImagePullBackOff` / `manifest unknown` | **私有仓库里没有这个镜像**（本项目最可能：`busybox:1.36` 未中转） | 按 §0.5 中转，或改用方案 A |
> | `Pending: Insufficient cpu/memory` | 节点资源不足 | `kubectl describe node <node>` 看可分配量 |
> | `Forbidden`（PSA） | 已开 `enforce=restricted`，busybox 以 root 运行被拒 | 用 `warn` 模式，或给 Pod 补 securityContext |
> | `ContainerCreating` 长时间不动 | 拉镜像慢 / CNI 分配 IP 慢 | 先 `crictl pull` 预热，再等 |

**进到容器里后，依次执行（逐条记录结果）：**

```bash
wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ | head -3
# ① 访问自己的服务 —— 预期：通（正常业务）

wget -qO- --timeout=3 http://prometheus.monitoring.svc.cluster.local:9090/-/healthy
# ② 访问监控系统的 Prometheus —— 预期：通 ★★★ 这就是问题所在！

wget -qO- --timeout=3 http://loki.monitoring.svc.cluster.local:3100/ready
# ③ 访问日志系统 Loki —— 预期：通 ★★★

nc -vz jenkins.jenkins.svc.cluster.local 8080
# ④ 探测 Jenkins —— 预期：通 ★★★ Jenkins 被攻破 = CI/CD 供应链沦陷

wget -qO- --timeout=3 https://www.baidu.com > /dev/null && echo "外网可达"
# ⑤ 访问外网 —— 预期：通（可把数据外传）★★★

exit
```

> ⚠️ **命令里的服务名请按你的实际命名空间核实**：Prometheus/Loki 在 `monitoring`，Jenkins 在 `jenkins`（若部署在别处请替换）。`nc -vz` 不通不代表服务没起，先确认 Service 名与端口。
>
> **★ 第 ⑤ 条（外网）要先单独确认一次，别默认它是通的**：本集群出外网历来靠**宿主机代理**（`11.0.1.1:7890`，见项目 B 的 Jenkins 配置），而 Pod 内**没有代理环境变量**。所以 `wget https://www.baidu.com` 很可能**在加固之前就已经不通**——那样它就**不能充当 before/after 的对比项**（前后都不可达，证明不了 egress 收紧）。
> ```bash
> # 加固前先跑这一条，把结果记进基线
> kubectl -n damo-app exec -it deploy/damo-app -- \
>   wget -qO- --timeout=3 https://www.baidu.com >/dev/null && echo "外网可达" || echo "外网不可达"
> ```
> - 显示**可达** → 第 ⑤ 条可以照原计划用作对比项（这是最好的情况）。
> - 显示**不可达** → 换成**集群内可量化的对比项**（②③④ 的 Prometheus / Loki / Jenkins 直连），外网那条只作为"本环境出网受限"的说明，**不要写进简历当成 egress 控制的效果**。
>
> **这份结果就是「裸奔」最有力的证据**：一个**对外暴露的静态页应用**，竟然能直连集群的监控、日志和 CI 系统。

### 2.3 检查三：ServiceAccount 现状

```bash
# ① 每个命名空间都有 default SA（K8s 自动创建，且不能删）
kubectl get sa -A

# ② damo-app 的 Pod 用的是哪个 SA
kubectl -n damo-app get pods -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName

# ③ 该 SA 有没有被绑定任何权限？（预期：没有任何一行命中）
#    ★ 必须按【结构化字段】查，不能 grep 'system:serviceaccount:damo-app'
#      （那种字符串在 RBAC 对象的 JSON 里根本不存在，原因详见 §2.5 的警告框）
kubectl get rolebinding,clusterrolebinding -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\t"}{.roleRef.kind}{"/"}{.roleRef.name}{"\t"}{.subjects[*].kind}{"/"}{.subjects[*].namespace}{"/"}{.subjects[*].name}{"\n"}{end}' | grep 'ServiceAccount/damo-app/'
# 期望：无输出。注意【无输出】才是"没被绑定"——如果用了错误的 grep 写法，
#      不管有没有绑定都会无输出，那就等于没有验证（假阴性）。

# ④ ★ 顺手排查权限放大：有没有角色绑到了"所有 SA"这个组？
kubectl get clusterrolebinding -o json \
  | grep -B2 -A4 'system:serviceaccounts' || echo "无 SA 组级绑定（正常）"
# 若有输出，说明某个角色对所有 SA 生效 —— 这是权限放大的高危信号，必须查清
```

### 2.4 检查四：Pod 里能不能拿到 token（★ 最直观的风险演示）

```bash
POD=$(kubectl -n damo-app get pod -l app.kubernetes.io/name=damo-app -o jsonpath='{.items[0].metadata.name}')

# ① token 存在吗？
kubectl -n damo-app exec "$POD" -- ls -la /var/run/secrets/kubernetes.io/serviceaccount/
# 预期：能看到 token、ca.crt、namespace 三个文件 —— 这就是"门禁卡"

# ② 用这个 token 访问 API Server
kubectl -n damo-app exec "$POD" -- sh -c '
  TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  APISERVER=https://kubernetes.default.svc
  echo "=== 试着列本命名空间的 Pod ==="
  wget -qO- --no-check-certificate --timeout=5 \
    --header="Authorization: Bearer $TOKEN" \
    $APISERVER/api/v1/namespaces/damo-app/pods 2>&1 | head -c 300
  echo
'
```

**预期结果与解读（两种结果都要能解释）**：

| 结果 | 说明 |
|------|------|
| 返回 `403 Forbidden`（`system:serviceaccount:damo-app:default cannot list ...`） | ✅ RBAC 默认拒绝生效——**default SA 确实没权限**。但注意：**token 本身是有效的身份凭证**，403 只说明"这个身份权限不够"，不代表"没有身份" |
| 返回 Pod 列表 JSON | ❌ 有人给 SA 绑了权限（或绑到了 `system:serviceaccounts`）——**权限配置疏漏，必须查清并修** |

> **★ 这个实验的价值在于"眼见为实"**：很多人知道"token 会自动挂载"，但**看到 token 文件的真实存在**、并且能用它发出一个**被 API Server 受理（而不是连接被拒）**的请求，才会真正理解风险——**API Server 认得这个身份**。这就是 `automountServiceAccountToken: false` 有效的理由：把凭证从容器里**物理移除**。

### 2.5 检查五：Jenkins 发布用的什么身份（供应链侧）

```bash
# ① 先看 Jenkins 命名空间里有哪些 SA
kubectl -n jenkins get sa
# 本项目：ServiceAccount/jenkins（项目 B 部署时建的）

# ② ★ 查绑定别用「字符串 grep」——按结构化字段查，一行一个绑定便于过滤
kubectl get rolebinding,clusterrolebinding -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\t"}{.roleRef.kind}{"/"}{.roleRef.name}{"\t"}{.subjects[*].kind}{"/"}{.subjects[*].namespace}{"/"}{.subjects[*].name}{"\n"}{end}' | grep jenkins
# 本项目期望命中一行：
#   /jenkins-admin   ClusterRole/cluster-admin   ServiceAccount/jenkins/jenkins
#   ↑ 这一行就是"过度授权"的证据

# ③ 或者最直接：项目 B 建的绑定对象名就叫 jenkins-admin，直接看它
kubectl describe clusterrolebinding jenkins-admin
# 看 Role 列是不是 cluster-admin

# ④ ★ 关键问题：这个身份到底能干什么？（从主体视角反查，最有力）
kubectl auth can-i --list --as=system:serviceaccount:jenkins:jenkins -n damo-app
kubectl auth can-i '*' '*' --all-namespaces --as=system:serviceaccount:jenkins:jenkins
# cluster-admin 会输出 *.* 且结果 yes —— 说明 Jenkins 一旦被攻破等于整集群沦陷
# 注意：--as 的字符串里 SA 名和命名空间都是 jenkins，别漏了后半段
```

> ⚠️ **为什么 `grep 'system:serviceaccount:jenkins'` 一条都匹配不到**
>
> `system:serviceaccount:<ns>:<name>` 是**「用户名」的字符串形式**，它只出现在三个地方：
> ① `kubectl auth can-i --as=` 的参数；② **报错信息**（`User "system:serviceaccount:..." cannot ...`）；③ 审计日志。
>
> 而 **RoleBinding / ClusterRoleBinding 对象里的 ServiceAccount 主体是结构化字段**，长这样：
>
> ```json
> "subjects": [ { "kind": "ServiceAccount", "name": "jenkins", "namespace": "jenkins" } ]
> ```
>
> **那个字符串在对象里根本不存在**，所以 grep 必然为空 —— 而**空输出很容易被误读成"没有绑定"**，这是最危险的假阴性：你明明有一条 cluster-admin 绑定，却以为很干净。
>
> | 主体类型 | JSON 里的写法 | 能用字符串 grep 吗 |
> |---|---|---|
> | `ServiceAccount` | `{"kind":"ServiceAccount","name":...,"namespace":...}`（三个字段） | ❌ 不能 |
> | `User` | `{"kind":"User","name":"alice"}` | ✅ 能 |
> | `Group` | `{"kind":"Group","name":"system:serviceaccounts"}` | ✅ 能（这个字符串确实存在） |
>
> 这也是 §2.3 第 ③ 步必须用 jsonpath 按字段查、而不能 grep 同一个原因。

**风险等级判断**：

| 情况 | 风险 | 本项目动作 |
|------|------|-----------|
| Jenkins Agent 用 **admin kubeconfig**（本集群为 `kubernetes-admin` @ `kubeadm:cluster-admins`，等价集群管理员） | 🔴 **最高**——Jenkins 被攻破 = 集群彻底沦陷 | 阶段 3.4 替换为最小权限 SA |
| Jenkins Agent 用 SA，但绑定了 `cluster-admin` ClusterRole | 🔴 高 | 同上 |
| Jenkins Agent 用 SA，绑定命名空间级 Role | 🟡 可接受，检查 verb 是否过宽 | 视 verb 收敛 |
| Jenkins Agent 用 SA，无任何绑定 | 🟢 安全但**流水线应该跑不通**（报 forbidden） | 补最小权限 Role |

---

## 阶段 3：RBAC 最小权限改造

> **本阶段目标**：damo-app 的 Pod 拿掉 token；Jenkins 发布者从「高权限身份」换成「只在 damo-app 命名空间内够用的 SA」。每一步都有 before/after 验证。

### 3.1 先做「过度授权」的反面教材（10 分钟，值得）

**这一步是教学性的**：先故意展示「权限给多了会怎样」，后面的加固才有说服力。

```bash
# ① 创建一个测试 SA
kubectl -n damo-app create serviceaccount over-privileged-demo

# ② 把它绑到 cluster-admin（★ 反面教材，做完立刻清理）
kubectl create clusterrolebinding over-privileged-demo-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=damo-app:over-privileged-demo

# ③ 看它现在能干什么
kubectl auth can-i --list --as=system:serviceaccount:damo-app:over-privileged-demo
# 预期：输出一片 "*.*" —— 能对任何资源做任何事

kubectl auth can-i delete nodes --as=system:serviceaccount:damo-app:over-privileged-demo
# 预期：yes ★★★ 一个应用侧的 SA 能删节点

kubectl auth can-i get secrets -A --as=system:serviceaccount:damo-app:over-privileged-demo
# 预期：yes ★★★ 能读全集群所有 Secret

# ④ 立刻清理（这一步不能省）
kubectl delete clusterrolebinding over-privileged-demo-binding
kubectl -n damo-app delete serviceaccount over-privileged-demo
```

### 3.2 设计：damo-app 到底需要什么权限？

**核心方法：从「它需要做什么」倒推，而不是从「我能给什么」正推。**

| 角色 | 需要调用 K8s API 吗？ | 需要权限吗？ | 结论 |
|------|---------------------|-------------|------|
| **damo-app 应用容器**（nginx 静态页） | ❌ 完全不需要 | ❌ 不需要 | **不挂 token**（`automountServiceAccountToken: false`），SA 无需任何 Role |
| **loadgen 压测容器**（busybox wget） | ❌ 不需要 | ❌ 不需要 | 同上 |
| **Jenkins 发布 Agent**（helm/kubectl 发布） | ✅ 需要 | ✅ 需要，但**只需操作 damo-app 命名空间** | 最小 Role（见 3.4） |

> **★ 关键思维转变**：传统做法是「给应用建个 SA，然后想该给它哪些权限」。**最小权限的极致是：如果它根本不需要身份，就不给它身份。** `automountServiceAccountToken: false` 比「建个 SA 但只给 read 权限」更安全——**没有凭证就没有凭证泄漏**。

### 3.3 改造 damo-app：不挂 token（写进 Chart，不手改）

**为什么必须写进 Chart**（面试点）：手改 Deployment，下一次 Jenkins 发布 `helm upgrade` 就把它覆盖没了——**加固成果会被流水线冲掉**。所以加固必须**落进 Chart 模板**，让它是可持续的。

在 `chart/damo-app/templates/deployment.yaml` 的 `spec.template.spec` 下增加两处：

```yaml
    spec:
      # ① 使用专用 SA（即使用了没有绑定任何 Role，也比用 default 更明确——
      #    因为 default 是"共享身份"，未来任何针对 default 的授权都会波及它）
      serviceAccountName: {{ .Values.serviceAccount.name }}
      # ② ★ 核心加固：不向容器挂载 API 访问凭证
      #    理由：damo-app 是静态页服务，从不调用 K8s API，因此不需要身份。
      #    不挂载 = 容器被攻破也拿不到集群凭证（消灭横向移动的立足点）
      automountServiceAccountToken: {{ .Values.serviceAccount.automountToken }}
```

`values.yaml` 增加：

```yaml
# ---- ServiceAccount 与凭证挂载（项目 F 安全加固）----
serviceAccount:
  create: true
  # 专用 SA 名（不叫 default，避免"共享身份"的连带授权风险）
  name: damo-app-sa
  # ★ 是否向容器挂载 API token。
  #   false：容器内没有 /var/run/secrets/kubernetes.io/serviceaccount/token
  #          适合不需要调用 K8s API 的应用（本项目）——最小权限的极致
  #   true ：仅当应用确实需要访问 API（如 Operator、控制器）时才打开
  automountToken: false
```

新增 `templates/serviceaccount.yaml`：

```yaml
{{- /* 项目 F · 专用 ServiceAccount
     说明：K8s 每个命名空间自动创建 default SA，但它被所有 Pod 共用——
     任何针对 default 的授权会波及全部 Pod。为应用建专用 SA 是"身份隔离"第一步。
     注意：本 SA【不绑定任何 Role】——因为 damo-app 不需要 API 权限。
     "不绑定 Role 的 SA + 不挂载 token" 比 "绑定只读 Role 的 SA + 挂载 token" 更安全。 */ -}}
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.serviceAccount.name }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "damo-app.labels" . | nindent 4 }}
# ★ 双保险：即使某个 Pod 忘了写 automountServiceAccountToken，
#   这个字段也会让"由本 SA 运行的 Pod"默认不挂载 token。
automountServiceAccountToken: false
{{- end }}
```

```bash
# 部署并验证
helm upgrade --install damo-app ./chart/damo-app -n damo-app \
  --set image.tag=19 --atomic --timeout 120s

kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
  -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName
# 期望：SA 列显示 damo-app-sa

# ★ 验收：token 已从容器中消失
POD=$(kubectl -n damo-app get pod -l app.kubernetes.io/name=damo-app -o jsonpath='{.items[0].metadata.name}')
kubectl -n damo-app exec "$POD" -- ls /var/run/secrets/kubernetes.io/serviceaccount/ 2>&1
# 期望：No such file or directory  ★ 这就是加固成功的直接证据

# ★ 业务不受影响
curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'
```

### 3.4 改造 Jenkins 发布身份：最小权限发布者 SA

#### 3.4.1 先确定「发布需要哪些权限」（倒推练习）

发布动作（项目 E 的 Stage 4）执行 `helm upgrade --install`，helm 在背后会：

| helm 的动作 | 需要的 K8s 权限 |
|-------------|----------------|
| 读/写 release Secret（revision 台账） | `secrets` 的 `get/list/create/update/delete` |
| 创建/更新 Deployment | `deployments` 的 `get/list/watch/create/update/patch` |
| 读/写 Service | `services` 的 `get/list/watch/create/update/patch` |
| `--wait` 等 Pod Ready | `pods`、`replicasets` 的 `get/list/watch` |
| 创建 ServiceAccount（项目 F 新增模板） | `serviceaccounts` 的 `get/list/watch/create/update/patch` |
| 读/写 HPA（项目 G 后 Chart 会渲染） | `horizontalpodautoscalers` 的 `get/list/watch/create/update/patch` |
| 读/写 NetworkPolicy（项目 F 的 Chart 版） | `networkpolicies` 的 `get/list/watch/create/update/patch` |
| 读 Event 辅助判断 | `events` 的 `get/list/watch` |
| 确认命名空间存在 | `namespaces` 的 `get`（**集群级资源，需 ClusterRole**） |

> **★ 这一步的工程价值**：**权限清单不是抄来的，是从实际动作倒推出来的。** 面试时能说「我从 helm upgrade 会做哪些操作倒推出它需要哪些 verb」，比背一个 YAML 强得多。

**两个可选策略**：

| 策略 | 权限 | 取舍 |
|------|------|------|
| **A. 严格最小** | 命名空间级 Role，**不给 `namespaces` 的 get/create** | 最安全，但 `--create-namespace` 与部分检查会失败 |
| **B. 实用最小**（本项目采用） | 命名空间级 Role + 一个**极窄的 ClusterRole**（只允许 `namespaces` 的 `get`） | 平衡：ns 已存在所以只需 get；**不给 create**，少一个动词少一类事故 |

本项目采用 **B**，并明确说明取舍——**这本身就是「最小权限不是教条，而是权衡」的体现**。

#### 3.4.2 YAML 落地（写文件再 apply）

`manifests/10-rbac-damo-deployer.yaml`：

```yaml
# ============================================================
# 项目 F · RBAC：Jenkins 发布者的最小权限身份
#
# 设计要点（面试可讲）：
# ① SA 建在 damo-app 命名空间（SA 是命名空间资源）；
# ② 权限清单从"helm upgrade 实际会做哪些操作"倒推得出，不是抄的；
# ③ 用 RoleBinding 绑定 Role，作用域锁定在 damo-app 命名空间；
# ④ namespaces 是集群级资源，只能靠 ClusterRole 授权，
#    但只给 get（不给 create/delete/update）——把爆炸半径关在一个动词上；
# ⑤ secrets 权限是 Helm release 台账的硬需求（sh.helm.release.v1.*）。
# ============================================================
---

# ① 发布者 ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: damo-deployer-sa
  namespace: damo-app
  labels:
    app.kubernetes.io/part-of: damo-app
    purpose: ci-deployer
automountServiceAccountToken: false   # 由流水线显式注入 kubeconfig，不依赖自动挂载

---
# ② 命名空间级 Role：发布所需的最小 verbs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: damo-deployer
  namespace: damo-app
rules:
  # Deployment：helm upgrade 要创建/更新；get/list/watch 用于 --wait 判断
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # ReplicaSet / Pod：--wait 与 rollout status 需要读状态
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  # Service：Chart 里含 service.yaml
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # ServiceAccount：Chart 里含 serviceaccount.yaml（项目 F 新增）
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Secret：Helm release 台账（sh.helm.release.v1.*）就存在这里 ★ 必需
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # HPA：项目 G 后 Chart 会渲染 HPA，helm 需要读写
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # NetworkPolicy：项目 F 的 Chart 版策略
  - apiGroups: ["networking.k8s.io"]
    resources: ["networkpolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Event：helm 的 --wait 会读事件辅助判断（只读）
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]

---
# ③ ClusterRole：只为一个集群级资源开一道缝（namespaces 的 get）
#    注意：namespaces 是集群级资源，RoleBinding 对它的授权无效，
#    因此必须用 ClusterRoleBinding，但只给最弱的 get 动词。
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: damo-deployer-ns-reader
rules:
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get"]     # ★ 只给 get：能确认命名空间存在，不能创建/删除/修改
  # 说明：--create-namespace 需要 create 权限，本项目发布时命名空间已存在，
  # 因此故意不给 create —— 少一个动词，少一类事故。

---
# ④ 绑定：Role → RoleBinding（命名空间内生效）
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: damo-deployer-binding
  namespace: damo-app
subjects:
  - kind: ServiceAccount
    name: damo-deployer-sa
    namespace: damo-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: damo-deployer

---
# ⑤ 绑定：ClusterRole → ClusterRoleBinding（集群级，仅 namespaces get）
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: damo-deployer-ns-reader-binding
subjects:
  - kind: ServiceAccount
    name: damo-deployer-sa
    namespace: damo-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: damo-deployer-ns-reader
```

```bash
kubectl apply -f manifests/10-rbac-damo-deployer.yaml
```

#### 3.4.3 验证权限矩阵

```bash
SA=system:serviceaccount:damo-app:damo-deployer-sa

echo "=== 应该 ALLOW ==="
kubectl auth can-i get deployments    -n damo-app --as=$SA   # yes
kubectl auth can-i update deployments -n damo-app --as=$SA   # yes
kubectl auth can-i create secrets     -n damo-app --as=$SA   # yes（release 台账需要）
kubectl auth can-i get namespaces         --as=$SA          # yes（只读）

echo "=== 应该 DENY（这些是"权限边界"的证明）==="
kubectl auth can-i get deployments -n monitoring --as=$SA    # no ★ 跨命名空间被挡
kubectl auth can-i get secrets -A    --as=$SA               # no ★ 不能读别人的 Secret
kubectl auth can-i delete nodes          --as=$SA           # no ★ 碰不到节点
kubectl auth can-i create namespaces     --as=$SA           # no ★ 不能建命名空间
kubectl auth can-i '*' '*'              --as=$SA           # no ★ 不是万能

echo "=== 完整权限清单 ==="
kubectl auth can-i --list -n damo-app --as=$SA
```

### 3.5 改造 Jenkins：从高权限切到最小权限（先并存，再切换）

**风险提示**：直接替换 Jenkins 凭据会导致流水线中断。**正确节奏是「建好新的 → 验证能用 → 再切 → 观察 → 最后删旧的」**。

```bash
# ===== 步骤 1：为 damo-deployer-sa 生成 kubeconfig =====
# 注意：K8s 1.24+ 不再自动创建 SA 的长期 token Secret，
# 需手动申请一个带有效期的 token（生产建议短有效期 + 定期轮换）。
kubectl -n damo-app create token damo-deployer-sa --duration=24h > /tmp/deployer.token

# 组装 kubeconfig
CLUSTER_IP=11.0.1.128      # 换成你的 API Server 地址（kubectl config view 可查）
CA_DATA=$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
TOKEN=$(cat /tmp/deployer.token)

cat > /tmp/damo-deployer.kubeconfig <<EOF
apiVersion: v1
kind: Config
clusters:
- name: kubernetes
  cluster:
    server: https://${CLUSTER_IP}:6443
    certificate-authority-data: ${CA_DATA}
contexts:
- name: damo-deployer@kubernetes
  context:
    cluster: kubernetes
    user: damo-deployer-sa
    namespace: damo-app
current-context: damo-deployer@kubernetes
users:
- name: damo-deployer-sa
  user:
    token: ${TOKEN}
EOF

# ===== 步骤 2：用这份 kubeconfig 验证权限够用 =====
export KUBECONFIG=/tmp/damo-deployer.kubeconfig
kubectl config current-context
kubectl -n damo-app get deploy          # 应成功
kubectl -n monitoring get pods          # 应被拒（证明作用域生效）
unset KUBECONFIG

# ===== 步骤 3：在 Jenkins 里创建凭据（UI 操作）=====
# Manage Jenkins → Credentials → System → Global credentials → Add Credentials
#   Kind:        Secret file
#   ID:          kubeconfig-damo-deployer      ← 语义化 ID，便于 Jenkinsfile 引用
#   File:        上传 /tmp/damo-deployer.kubeconfig
#   Description: damo-app 命名空间最小权限发布者（仅 damo-app，无集群级权限）
#
# ⚠️ 上传后从 master 删除临时文件（凭证不该在磁盘留副本）：
#    rm -f /tmp/damo-deployer.kubeconfig /tmp/deployer.token

# ===== 步骤 4：改 Jenkinsfile，用新凭据执行发布 =====
```

#### 3.5.0 ★ 先明确：要改哪些文件 / 哪些对象

| 对象 | 位置 | 改什么 |
|---|---|---|
| **`Jenkinsfile`（仓库根目录）** ← 本节说的"对应改动" | **权威副本：GitHub `GXEL-xy/damo-app/Jenkinsfile`**（Job 配的是 *Pipeline script from SCM*，Jenkins 每次拉的就是它）；本地同步副本 `项目 E 文件包中的 Jenkinsfile` | **只有 Stage 4**：① 外层包 `withCredentials`；② 删掉 `--create-namespace` |
| **Jenkins PodTemplate（UI 操作，不是文件）** | Manage Jenkins → Clouds → Pod Templates → **Pod Template Volumes** | ★ **删掉 Secret `kubeconfig` 挂到 `/root/.kube/` 的那个 Volume** |
| `Secret/kubeconfig`（ns `jenkins`） | 项目 B 的 `scripts/setup-kubeconfig.sh` 建的 | 收权完成后删除（见步骤 6） |

> ⚠️ **为什么"只改 Jenkinsfile 不够"**
> 项目 B 阶段 4 用的是**方式 A —— 集群 admin 的 kubeconfig**（`kubectl config view --raw --minify` 取的 `admin.conf`，身份是 **`kubernetes-admin` @ `kubeadm:cluster-admins`**，等价集群管理员），以 Secret 形式**挂在 Pod 层、挂到 `/root/.kube/`，所有容器都能读到**。
> 只加 `withCredentials` 的话，`KUBECONFIG` 环境变量确实优先于 `~/.kube/config`（helm 会改用新凭据），**但旧的 admin 证书还躺在 Pod 里** —— 一条 `cat /root/.kube/config` 就能拿到，**等于权限没收**。面试被追问「你怎么确认旧权限真的没了」，这里就是硬伤。

Jenkinsfile 的对应改动（项目 E 的 Stage 4）：

```groovy
stage('4. Deploy via Helm') {
    steps {
        // ★ 用最小权限凭据，而不是 admin kubeconfig
        withCredentials([file(credentialsId: 'kubeconfig-damo-deployer', variable: 'KUBECONFIG')]) {
            container('helm') {
                sh """
                    helm upgrade --install damo-app chart/damo-app \
                      --namespace damo-app \
                      --set image.tag=${IMAGE_TAG} \
                      --atomic --timeout 120s
                    # 注意：去掉 --create-namespace（新 SA 没有 namespaces create 权限，
                    # 命名空间已存在，不需要创建）
                """
            }
        }
    }
}
```

```groovy
// ★ 另外：Stage 5 (Verify) 也要包一层同样的 withCredentials
//   原因：收权时要把 PodTemplate 里的 kubeconfig Volume 删掉，
//   Pod 里不再有 /root/.kube/config，Stage 5 的 kubectl / helm 同样只能靠这份凭据。
stage('5. Verify') {
    steps {
        withCredentials([file(credentialsId: 'kubeconfig-damo-deployer', variable: 'KUBECONFIG')]) {
            container('helm') {
                sh """
                    kubectl get pods -n damo-app -o wide
                    helm history damo-app -n damo-app
                """
            }
        }
    }
}
```

**现成的 F 版文件**：`项目 F 文件包中的 Jenkinsfile`（已按上面改好，含 Stage 4 + Stage 5 双包装与全部前置注释），可直接整份覆盖仓库里的 `Jenkinsfile`。

#### 3.5.1 步骤 5 · 提交改动（两种方式，按你的习惯选）

> ★ **提交前必须先确认**：Jenkins 凭据 `kubeconfig-damo-deployer` 已经创建好（步骤 3）。
> 因为提交本身就会触发构建（Poll SCM 每 5 分钟轮询），凭据没建好会直接跑出一个红色构建。

**方式 A · 命令行（原方案）**

```bash
# 在 master 或本机的仓库工作副本里
git add Jenkinsfile
git commit -m "security: use least-privilege SA for deploy stage"
git push
```

**方式 B · 直接在 GitHub 网页上改（★ 你用的这种）**

因为这次改动要**把整个 Stage 4 多缩进一层**（为了套 `withCredentials`）并删掉一行，逐行手改容易错位，**推荐整份替换**：

1. 打开 `https://github.com/GXEL-xy/damo-app/blob/main/Jenkinsfile`
2. 点右上角**铅笔图标**（Edit this file）→ 进入网页编辑器
3. `Ctrl+A` 全选 → **整份粘贴** `项目 F 文件包中的 Jenkinsfile` 的内容
4. 拉到页面底部 **Commit changes**：
   - 提交信息：`security: use least-privilege SA for deploy stage`
   - 选 **Commit directly to the `main` branch**（别新建分支，否则流水线拉不到）
   - 点 **Commit changes**
5. **什么都不用做**，等最多 5 分钟（Poll SCM 周期）自动触发构建

> ⚠️ **网页编辑器的两个风险**
> - **没有 Groovy 语法校验**：Job 是 *Pipeline script from SCM*，写错一个括号会在构建时直接 `WorkflowScript: ... unexpected token` 挂掉（而且是在解析阶段就挂，连 Stage View 都不出来）。
> - **粘贴整份文件容易带进全角字符/不可见空格**（从 Word、聊天工具复制时尤其常见）。

**★ 强烈建议：先用 Jenkins 的 Replay 验证一遍，再动 GitHub**

Replay 用的是「那次构建时 Jenkins 自己存快照的 Jenkinsfile」，**不依赖 Git** —— 可以把改好的内容在这里先跑通：

1. Jenkins → `damo-app-pipeline` → 打开最近一次**成功**的构建（如 `#34`）
2. 左侧点 **Replay**
3. 把编辑框里的内容**整份替换**为新的 Jenkinsfile
   ⚠️ **Replay 默认载入的是"那次构建当时的旧脚本"** —— 直接点 Run 而不粘贴，跑的还是旧逻辑，
   Stage 5 的 `kubectl auth whoami` 会显示 `kubernetes-admin`，看起来像"withCredentials 没生效"，其实是脚本没换。
4. 点 **Run**（会多占一个构建号，无妨——构建号只是序号；这条运行不算正式发布记录）
5. 看到 **Stage 4 成功 + `helm history` 正常输出** → 说明语法、凭据、权限三者都对 → **这时再去 GitHub 提交**，几乎不可能失败

> **顺序总结**：建凭据（步骤 3）→ Replay 验证 → GitHub 网页提交 → 等 Poll SCM 自动构建 → 绿了之后做步骤 6 的回收。
> 这样全程不会有「红色构建」这种不体面的中间态。
```bash
# ===== 步骤 6：观察一个完整发布周期后，再回收旧的高权限配置 =====
# ★ 先想清楚"旧权限"到底是什么：是 admin 的【客户端证书】（kubernetes-admin @ kubeadm:cluster-admins）。
#   真正要回收的是"Jenkins 那边的凭据副本"，即下面三样。
#   ⚠️⚠️ 千万不要去删 `clusterrolebinding kubeadm:cluster-admins`：
#     你 master 上用的就是同一份证书，删了【你自己也会立刻失去 cluster-admin】（自锁！），
#     除非你确认 super-admin.conf 还在可以救场。

# 6.1 确认新链路已成功发布至少 2 次
helm history damo-app -n damo-app | tail -3
curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'
#   ⚠️ 时间戳用的是 master 的本地时区（本环境是 UTC，比北京时间慢 8 小时），别被时差误导
#   ⚠️ 台账【不记发布者身份】—— 它证明不了"这两次用的是新凭据"，
#      取证要看 Jenkins Console Output（见 §3.6 完成标准的说明）

# 6.2 回收：① PodTemplate 的 Secret Volume（UI 里删掉那个挂载）
#           ② 集群里的凭据副本
kubectl -n jenkins delete secret kubeconfig
#           ③ 顺手清理项目 B 留下的"过度授权"绑定（它绑的是 SA，不是 Agent 里的证书）
kubectl describe clusterrolebinding jenkins-admin      # 先确认对象是它
kubectl delete clusterrolebinding jenkins-admin

# 6.3 验证收权真的生效（下次构建时进 Agent Pod 里看）
#   ls /root/.kube/        → 期望：没有 config 文件
#   cat /root/.kube/config → 期望：No such file or directory
```

> ⚠️ **别忽略 token 有效期**：步骤 1 用的是 `kubectl create token --duration=24h` —— **24 小时后这份 kubeconfig 就失效，流水线会在 Stage 4 直接 401/403**。
> 手工演示没问题，但**第二天要跑流水线前必须重签**。两种处理：
>
> **短期（演示够用）**：记下到期时间，需要时重跑步骤 1 并更新 Jenkins 凭据。
>
> **长期（推荐给作品集，一劳永逸）**：改用长期有效的 token Secret（K8s 1.24+ 不再自动创建，需手写）。
>
> **① 用文件创建**（★ 别用 heredoc —— 见下方踩坑记录）：
> ```bash
> kubectl apply -f manifests/30-deployer-token-secret.yaml
> # 文件内容见 项目 F 文件包中的 manifests/30-deployer-token-secret.yaml
> ```
>
> **② 重建 kubeconfig**（用 `kubectl config` 子命令拼装，**不写 YAML → 不会踩缩进的坑**）：
> ```bash
> K=/tmp/damo-deployer.kubeconfig
> TOKEN=$(kubectl -n damo-app get secret damo-deployer-token -o jsonpath='{.data.token}' | base64 -d)
>
> # 从当前 kubeconfig 里取出集群 CA（免 sudo，也不必读 /etc/kubernetes/pki）
> kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > /tmp/ca.crt
>
> kubectl config --kubeconfig=$K set-cluster kubernetes \
>   --server=https://11.0.1.128:6443 \
>   --certificate-authority=/tmp/ca.crt --embed-certs=true
> kubectl config --kubeconfig=$K set-credentials damo-deployer-sa --token="$TOKEN"
> kubectl config --kubeconfig=$K set-context damo-deployer@kubernetes \
>   --cluster=kubernetes --user=damo-deployer-sa --namespace=damo-app
> kubectl config --kubeconfig=$K use-context damo-deployer@kubernetes
> ```
> `--embed-certs=true` 是关键：**把 CA 内嵌进 kubeconfig，让它成为单文件**（Jenkins 的 Secret file 凭据只能收一个文件）。
>
> **③ 验证 + 更新凭据**：
> ```bash
> kubectl --kubeconfig=$K auth whoami                 # 期望：system:serviceaccount:damo-app:damo-deployer-sa
> kubectl --kubeconfig=$K get pods -n monitoring       # 期望：Forbidden（作用域生效）
> # → Jenkins UI 里更新 kubeconfig-damo-deployer 凭据，重新上传 $K
> rm -f $K /tmp/ca.crt /tmp/damo-deployer.kubeconfig   # 磁盘不留副本
> ```
>
> **★ 踩坑记录（2026-09-21）**：用 heredoc 创建这个 Secret 时踩了坑 ——
> ```
> error: ... from server for: "STDIN": resource name may not be empty
> ```
> **根因是 YAML 缩进在复制过程中丢了**：`name:` / `annotations:` 掉到了第 0 列，`metadata:` 变成空对象 → 解析不出 name。
> **两条通用教训**：
> ① **heredoc 里的 YAML 最怕缩进被吞**（从网页/聊天里复制、经过某些编辑器时尤其容易）——**本项目统一改用"写 .yaml 文件再 `kubectl apply -f`"**；
> ② 报错里的 **`Name: ""`** 就是"没解析到 metadata.name"的明确信号，见到它先回去看缩进，别去怀疑 RBAC 或权限。
>
> ⚠️ 长期 token 本身就是"静态凭证"，安全性低于短 token —— 生产更推荐**短 token + 定期轮换**或用 external-secrets/Vault。这个取舍能讲出来，比单纯"用长 token 图省事"更值钱。

> **★ 为什么必须"先并存再切换"**：权限收紧是**不可逆的破坏性变更**（一旦删掉旧绑定，流水线立刻跑不通）。生产改权限的标准节奏是：① **新增**最小权限身份 → ② **验证**它能完成全部必要动作 → ③ **切换**调用方 → ④ **观察**至少一个完整发布周期 → ⑤ **回收**旧权限。这个五步节奏适用于一切「收权」操作，面试讲出来非常显专业度。

> **★ 身份对比：本项目最有说服力的一组证据（实测于 2026-09-21）**
>
> | | 加固前 | 加固后 |
> |---|---|---|
> | `kubectl auth whoami` → Username | `kubernetes-admin` | **`system:serviceaccount:damo-app:damo-deployer-sa`** |
> | Groups | `[kubeadm:cluster-admins system:authenticated]` | `[system:serviceaccounts system:serviceaccounts:damo-app system:authenticated]` |
> | **credential-id 的类型** | `X509SHA256=...` ← **客户端证书** | `JTI=...` ← **ServiceAccount token** |
> | 权限范围 | 全集群 cluster-admin | 仅 `damo-app` 一个命名空间 |
>
> 最后一行是**最容易被忽略、也最值钱**的一条：**换掉的不只是"名字"，而是"凭据的种类"** ——
> 从「一份永久有效、绑在证书里的集群管理员身份」变成「一份有有效期、随 SA 生命周期管理的 token」。
> 面试时把这一行讲出来，说明你不只是照着清单做了动作，而是**看见了权限模型的实质变化**。

### 3.6 阶段 3 完成标准

- [ ] `kubectl -n damo-app exec <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount/` 报 **No such file**（token 已从容器中消失）
- [ ] 应用页面仍然正常（`curl :30090` 返回 Build 号）
- [ ] `kubectl auth can-i --list --as=...damo-deployer-sa -n damo-app` 输出有限条目，**跨命名空间全为 no**
- [ ] Jenkins 用新凭据**成功发布过至少 2 次**，页面 Build 号推进（回收旧权限的前置条件）
      - ★ **`helm history` 证明不了这件事** —— 台账只记「时间 / 状态 / Chart 版本」，**不记发布者身份**。拿它截图当"新链路已生效"的证据会被追问穿。
      - 正确取证：**Jenkins 构建的 Console Output**。F 版 Jenkinsfile 的 Stage 5 里有 `kubectl auth whoami`，
        输出 `system:serviceaccount:damo-app:damo-deployer-sa` 才算走的是新凭据。
      - ⚠️ **手动 `helm upgrade` 不算数**：例如 §3.3 改完 Chart 后你自己在 master 上跑的那次，用的是 master 的 admin kubeconfig，属"旧链路"。
        数次数时把它排除，否则会误判成"已经用新凭据发布 2 次了"。
- [ ] **旧的高权限凭据真的消失了**：Agent Pod 里 `cat /root/.kube/config` 报 No such file；`Secret/kubeconfig` 已删；PodTemplate 的 Volume 已移除
      （**注意**：本集群 admin 身份是 `kubernetes-admin` @ `kubeadm:cluster-admins`，受 RBAC 约束、**理论上可撤销**；但**不要用删 `clusterrolebinding kubeadm:cluster-admins` 的方式撤销** —— master 上你自己用的就是这份证书，删了会自锁。回收靠上面三样）

---

## 阶段 4：NetworkPolicy 默认拒绝与逐条放行

> **本阶段目标**：把 `damo-app` 命名空间从「全通」改成「默认全拒 + 四条白名单」，**每一步都验证，每一步都知道为什么**。
>
> **操作纪律（★ 本项目最重要的一条）**：**先只 deny Ingress → 验证 → 再 deny Egress**。一次性双拒会同时踩「探针挂」和「DNS 挂」两个坑，无法定位。

### 4.1 基线：先记录「改造前的通」

```bash
# ① 外部访问（NodePort）—— 记录基线
curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'

# ② 集群内访问
kubectl -n damo-app run netcheck --rm -it --restart=Never \
  --image=busybox:1.36 -- sh -c '
    wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ | head -2
    echo "--- 集群内访问：通 ---"
  '

# ③ 跨命名空间访问（不该通，但现在通）—— 这是改造的靶子
kubectl -n default run netcheck-x --rm -it --restart=Never \
  --image=busybox:1.36 -- sh -c '
    wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ | head -2
    echo "--- 从 default 命名空间访问 damo-app：通（说明没有网络边界）---"
  '
```

### 4.2 第一步：只做 Ingress 默认拒绝

`manifests/20-netpol-01-default-deny-ingress.yaml`：

```yaml
# ============================================================
# 项目 F · NetworkPolicy 第 1 步：Ingress 默认拒绝
#
# 机制说明（面试必讲）：
#   podSelector: {}        → 选中本命名空间【所有】Pod
#   policyTypes: [Ingress] → 只管控入方向
#   不写 ingress: 段        → 【没有任何允许规则】= 全拒
#
# ★ 反例警告：千万不要写成 ingress: [{}]
#   ingress: []   → 空数组 = 全拒（等价于不写）✅
#   ingress: [{}] → 含一个空规则对象 = 【允许所有来源】❌ 一个括号之差，从全拒变全通
#
# 本步不碰 egress，是为了把变量控制到最少：
# 如果服务不通，可以确定是"入方向被拒"，而不是"出方向被拒"。
# ============================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: damo-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

```bash
kubectl apply -f manifests/20-netpol-01-default-deny-ingress.yaml

# 验证策略已生效（回看确认没写成 ingress: [{}]）
kubectl -n damo-app get netpol default-deny-ingress -o yaml | grep -A6 "^spec:"
# 期望：spec 下只有 podSelector / policyTypes，【没有】 ingress 段
```

### 4.3 观察「断了什么」——这一步是教学核心

#### 4.3.1 实测结论（★ 先看这个，它推翻了网络上的常见说法）

**实测环境**：3 节点集群 / Calico IPIP（tigera-operator）/ K8s v1.35 / 策略只有 `default-deny-ingress`
**实测时间**：2026-09-23
**探针配置**：三探针 `tcpSocket`，已显式设置 `timeoutSeconds: 1`

| # | 检查项 | 命令 | 实测结果 |
|---|--------|------|---------|
| ① | 外部 NodePort 访问 | `curl --max-time 5 http://11.0.1.128:30090/` | ✅ **超时/空** —— 外部路径断了 |
| ② | Pod 就绪状态 | `kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app` | ⚠️ **`READY 1/1`，没有掉** |
| ③ | Service Endpoints | `kubectl -n damo-app get endpoints damo-app` | ⚠️ **`10.244.123.6:80,10.244.215.48:80`，正常** |
| ④ | 集群内访问 Service | `nc -vz -w 3 damo-app.damo-app.svc.cluster.local 80` | ✅ **`Connection timed out`**（不是 refused） |
| ⑤ | RESTARTS 计数 | `kubectl -n damo-app get pods` | ⚠️ **不增长** |

**三条命令的输出（原样留存，作为独立验证的证据）**：

```text
$ kubectl -n damo-app get netpol
NAME                   POD-SELECTOR   AGE
default-deny-ingress   <none>         12m
# ↑ 策略干净：只有全拒，没有白名单。排除「白名单把探针放行了」这一解释

$ kubectl -n damo-app get endpoints damo-app
NAME       ENDPOINTS                          AGE
damo-app   10.244.123.6:80,10.244.215.48:80   12d
# ↑ ★ 关键证据：默认拒绝已生效，Endpoints 却仍有真实 Pod IP
#   = Pod 依然 Ready = kubelet 探针根本没有被策略拦住

$ kubectl -n damo-app run netcheck --rm -it --restart=Never \
    --image=busybox:1.36 -- sh -c 'nc -vz -w 3 damo-app.damo-app.svc.cluster.local 80'
nc: damo-app.damo-app.svc.cluster.local (10.100.232.111:80): Connection timed out
# ↑ ★ 关键证据：ClusterIP 访问是【超时】而不是 refused
#   = Calico 在静默丢包 = 默认拒绝确实在拦截
```

**结论：`default-deny-ingress` 确实生效了，但它的受害者只有「外部流量」和「集群内跨 Pod 流量」——`kubelet` 的健康探针不受影响。**

#### 4.3.2 为什么「探针连环挂」这次没有出现（★ 面试高价值）

网上（包括很多博客和 AI 回答）有一个流传很广的说法：

> 「NetworkPolicy 默认拒绝的第一个受害者是 kubelet 探针——因为探针流量来自节点、没有 Pod 身份，任何基于标签的放行都覆盖不到它，所以探针会失败、Pod 变 NotReady、Endpoints 清空。」

**这个说法在"探针的流量需要经 CNI 转发"的实现里是成立的；但在本集群（Calico IPIP + tigera-operator）实测不成立。** 原因在于**探针的流量路径根本没有进入策略判定链路**：

```
   ┌──────────── 节点（work1）────────────┐
   │  kubelet                              │
   │    │ 探针：Pod IP:80（本机已分配的 Pod）│
   │    ▼                                  │
   │  本机转发（route_local / 容器 veth）   │
   │    │ ★ 这一步【不经过】Calico 的       │
   │    │   iptables 策略链（cali-*）       │
   │    ▼                                  │
   │  容器网络栈 → 80 端口有 nginx 在听     │
   └───────────────────────────────────────┘
                ▲
                │ 对比：外部 / 集群内其它 Pod 的流量
                │ ★ 走的是 cali-* 链 → 被策略判定 → 静默丢包
```

**为什么探针不受管**：`tcpSocket` 探针的目标是**本机 Pod 的 IP**，kubelet 从宿主机发起后，流量经由本机路由直达容器 veth —— 它**不跨节点、不经 IPIP 封装、不进入 Calico 用于策略执行的 `cali-*` iptables 链**。既然没进判定链，`podSelector: {}` 就管不到它。

**所以「放行节点网段 `11.0.1.0/24`」这条规则的必要性要重新归因**：

| 原来的说法 | 实测后的正确归因 |
|-----------|----------------|
| 为了 **kubelet 探针** 而放行节点网段（主因） | ❌ 探针不需要放行 —— 它不被策略管 |
| 为了 **NodePort 的 SNAT 源 IP** 而放行节点网段 | ✅ **这才是真正且唯一的必要性** |

> **★ 这条修正本身就是最值钱的面试点**：
> 「我在做 NetworkPolicy 默认拒绝时，预期会遇到"探针被拒 → Pod 不 Ready"的连环故障，但实测**什么都没有发生**：Endpoints 仍有 Pod IP、RESTARTS 不增长。
> 于是我分层排除了「策略没生效」（`nc` 超时证明在丢包）、「白名单放行了探针」（`get netpol` 只有一条全拒）、「探针超时太长」（`timeoutSeconds` 已经是 1），
> 最后定位到真正的机制：**kubelet 的 `tcpSocket` 探针目标是本机 Pod，流量不走 Calico 的策略判定链**。
>
> ⚠️ **但这条排除法后来被复查出不够严谨**（2026-09-24）：读出 `timeoutSeconds = 1`
> **只是 K8s 的 API 默认值**，不能证明"chart 里已经设置过"——
> 严格说它排除的是"探针超时**非默认**且过长"，而不是"超时太长"这个可能性本身。详见 §4.3.3。
>
> 所以「默认拒绝会误杀探针」这个流行说法**取决于 CNI 实现和探针的流量路径，不能当成通用结论**。
> 而放行节点网段的真正理由，**在 2026-09-23 抓包后又被修正过一次**：
> 不是"经 kube-proxy SNAT 后源 IP 变成节点 IP"（**该说法已被实测推翻**），
> 而是**配合 `externalTrafficPolicy: Local` 保留客户端真实源 IP** —— 详见 §4.4。

> **★ 可迁移的排查方法论（三条排除法）**：
> | 疑问 | 判定命令 | 判据 |
> |------|---------|------|
> | 策略到底生效了没？ | 从 netcheck Pod `nc -vz ... 80` | **`timed out`** = 在静默丢包（生效）；**`refused`** = 没生效 |
> | 是"策略问题"还是"Pod 本身问题"？ | 从**别的命名空间** `nc -vz <Pod IP> 80` | 本 ns 超时 + 别 ns `refused` → 策略生效且定位准确 |
> | 是"探针被拦"还是"探针没被管"？ | `kubectl get endpoints` | 默认拒绝下 **仍有 Pod IP** → 探针没被拦，别往这个方向查 |

#### 4.3.3 那么「探针超时」这件事还要不要管？要 —— 但事实和理由都得改对

> **★★ 先纠正一个事实错误（本文档早期写错，2026-09-24 更正）**
>
> 本文档此前写「`tcpSocket` 探针的 `timeoutSeconds` **默认 30s 太宽容**」——
> **这是错的。`Probe.timeoutSeconds` 的 K8s 默认值本来就是 `1` 秒**
> （30 是 `terminationGracePeriodSeconds` 的默认值，两者很容易混）。
>
> **连带纠正一个"实测"的解读**：阶段 4.3 排障时曾把
> 「`jsonpath` 读出 Pod 的 `timeoutSeconds` 已经是 `1`」当成"说明 chart 里已经改过" ——
> **其实那只是 API 默认值，什么都不能证明**（而当时的排障正是靠这条排除了"超时太长"）。
> ★ **这条教训比结论更值钱**：**"读到某个值" ≠ "这个值被显式配置过"**，
> 必须区分「**显式配置**」和「**API 默认**」——
> 这和 `helm get values`（只看你给的）vs `--all`（含 chart 默认）是**同一类问题**。
>
> **所以：显式写 `timeoutSeconds: 1` 在功能上【不改变任何行为】**，
> 它做的是**把隐含的 API 默认值显式化**。

> **机制（这部分仍然成立）**：`tcpSocket` 探针的判定规则是
> 「**在 `timeoutSeconds` 内没收到 TCP RST**」就算成功 ——
> 也就是说，**正常情况下探针是"靠被拒绝"来通过的**。
>
> | 场景 | 探针目标端口的响应 | 探针结果 |
> |------|------------------|---------|
> | 无策略（全通） | ✅ 本机栈立刻回 **RST** | 成功（端口正常监听） |
> | 进程挂了 | ✅ 本机栈立刻回 **RST** | **失败** ← 探针的正确检测行为 |
> | 探针流量被**静默丢包**（某些 CNI / `httpGet` + hairpin 场景） | ❌ 无响应 | **取决于 `timeoutSeconds`**：短 → 失败（异常暴露）；长 → 仍报成功（异常被掩盖） |

> **那显式写还值不值得？值得 —— 但理由要换成「显式化」，不是「收紧一个过宽的默认」**：
> ① **显式化契约**：值写在 chart 里，读代码/评审时一眼可见，不必去查 API 默认值；
> ② **防误改**：未来有人把探针换成 `httpGet`、或调整超时时，它是一个"锚点"；
> ③ **可参数化**：能按环境覆写（本项目的 `probes.timeoutSeconds` 就是这样一个参数）。
>
> **本集群的现实**：探针路径不进策略链（§4.3.2），所以这一项在当前环境**不改变任何现象**。
> 它真正的价值是**防止未来被改坏** —— **把 1 改成 30 才会真的掩盖异常**。

`chart/damo-app/values.yaml`（项目 F 的完整版见 `chart-patch/values.yaml`）：

```yaml
probes:
  # tcpSocket 的判定是"未收到 RST 即成功"；
  # 超时值决定了"静默丢包"能不能被识别（短 → 暴露异常，长 → 掩盖异常）。
  # ★ Probe.timeoutSeconds 的 K8s 默认值本来就是 1 秒
  #   （30 是 terminationGracePeriodSeconds 的默认值，别混）——
  #   所以这里显式写 1 是"把默认值显式化"，不改变当前行为。
  timeoutSeconds: 1
```

#### 4.3.4 操作命令与预期输出（按实测修正）

```bash
# ① 外部访问（预期：不通）★ 实测：超时/空
curl -s --max-time 5 http://11.0.1.128:30090/ | head -2

# ② 集群内访问（预期：不通）★ 实测：Connection timed out（不是 refused！）
kubectl -n damo-app run netcheck --rm -it --restart=Never \
  --image=11.0.1.128:30000/busybox:1.36 -- sh -c '
    nc -vz -w 3 damo-app.damo-app.svc.cluster.local 80
    wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ | head -2 || echo "内网访问失败"'
# ★ 记下区别：timed out = 被静默丢包（策略生效）。若出现 "Connection refused"，
#   说明流量根本没被判策略处理 —— 先查 CNI、再查策略 podSelector 是否匹配上 Pod。

# ③ Pod 状态（★ 实测：READY 不变，探针不受影响）
kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app
# 实测结论：READY 保持 1/1（或 2/2）、RESTARTS 不增长
# ⚠️ 如果你看到 READY 掉到 0/1 —— 说明你的 CNI/探针类型与本集群不同，
#    探针确实被拦了。那就按「放行节点网段」修复（本文件的白名单规则①已给出）。

# ④ Endpoints（★ 实测：仍有 Pod IP，不为空）
kubectl -n damo-app get endpoints damo-app
# 实测结论：ENDPOINTS 列显示真实 Pod IP
# ⚠️ 若为空 <none> —— 与 ③ 同因，探针被拦了，按上一条处理

# ⑤ 证明"策略确实在拦截"（本节的核心证据）★ 实测：超时
kubectl -n damo-app run netcheck2 --rm -it --restart=Never \
  --image=11.0.1.128:30000/busybox:1.36 -- sh -c '
    nc -vz -w 3 damo-app.damo-app.svc.cluster.local 80'
```

**判定矩阵（按实测结果扩展为通用结论）**：

| ③④ 的现象 | ⑤ 的现象 | 结论 | 处理 |
|-----------|---------|------|------|
| READY 掉 / Endpoints 空 | 超时 | **探针流量经 CNI 判定**（如某些 CNI、或探针走 Service/hairpin）→ 探针被拦 | 放行节点网段（白名单规则①） |
| **READY 正常 / Endpoints 有 IP（本集群）** | **超时** | **探针流量不经 CNI 判定** → 探针不受策略影响 | 无需处理；把"节点网段"归因于 NodePort SNAT |
| READY 正常 | **refused** | **策略根本没生效** | 查 CNI 是否支持 netpol；查 `podSelector` 是否匹配上 Pod |

> **★ 一句话记住本节**：**默认拒绝生效的判定标准不是"Pod 变 NotReady"，而是"连接从 refused 变成 timed out"。**
> 拿 Pod 状态当策略生效的证据，在某些 CNI 下会得出完全错误的结论。

#### 4.3.5 那这一步还值得做吗？值得 —— 演示主线换成它

原来的"探针连环挂"在本集群演示不出来，但**换成下面这条主线反而更干净、更有普遍性**：

```
   默认拒绝前                      默认拒绝后
   ─────────────                  ─────────────
   外部 curl :30090  ✅ 有内容      外部 curl :30090  ❌ 超时
   集群内 nc :80     ✅ succeeded   集群内 nc :80     ❌ timed out  ★ 从"被拒"变"丢包"
   探针/Endpoints    ✅ 正常        探针/Endpoints    ✅ 仍正常（本 CNI 下不受影响）
```

**这条主线的三个教学价值**：
1. **"refused → timed out"是策略生效的严谨证据** —— 比看 Pod 状态可靠；
2. **反直觉点更硬**：很多人以为"默认拒绝会把 Pod 搞挂"，实测是**Pod 完全健康、只是没人能连进来** —— 这恰恰是**最危险的形态**：服务看起来一切正常（Ready、Endpoints 齐全），但业务已经全断。**监控如果只盯 Pod 状态，这次故障会被完全漏掉**（对照项目 C 的告警：`curl` 可用性告警才能发现）。
3. **带出一个真实结论**：安全加固的效果验证**必须用"连接语义"（超时/拒绝）而不是"资源状态"** —— 这条可以迁移到任何 CNI、任何集群。

> **★ 与项目 C 的串联（面试可直接讲）**：
> 「默认拒绝之后，`damo-app` 的 Pod 依然 `READY 1/1`、Endpoints 齐全、`RESTARTS` 为 0 —— 从 K8s 资源状态看**完全健康**，
> 但外部访问已经全断。如果监控只做 Pod 存活告警，这次故障会**静默逃过所有告警**。
> 真正能发现它的是项目 C 里那条**基于 `curl` 可用性的业务告警**。
> **所以"资源健康"不等于"业务可用"——这是监控设计里最容易踩的坑。**」

> **★ 探针为什么"打不通"：先分清「拒绝」和「丢包」**
>
> 最容易卡住的疑问：「探针是发到 Pod **自己**的，难道不是"自己访问自己"、应该直接成功吗？」
> **不是。**探针的目标是 Pod IP 与容器端口之间的一条**真实 TCP 握手**——它走的是容器网络栈 + Calico 的**入方向策略判定**，不是"应用内部的函数调用"。在 NetworkPolicy 生效的 Pod 里，**pod 与 own-service** 的流量同样受策略管辖。
>
> | 情形 | 数据包命运 | 探针看到的 |
> |------|-----------|-----------|
> | **无策略（全通）** | 内核本机栈**立刻回绝**（容器里 80 端口没人监听） | `connection refused`（**收到 RST**） |
> | **默认拒绝** | 被 Calico 拦下，**不回 RST、也不回 ICMP**（静默丢弃，策略默认行为） | **无任何响应** → 只能等超时 |
>
> 所以现象是**从"立刻被拒"变成"一直没响应"**。而你新建的 `netcheck` Pod 访问 Service：
> Service VIP **不在任何 netpol 的白名单里**（白名单只放了节点网段 `11.0.1.0/24` 与 `app=loadgen`）→ 同样被静默丢弃 → `wget` 一直等到自己的 `--timeout=3` 才报失败。
>
> **两个网络机制验证（顺手做，面试能讲）**：
> ```bash
> # A. 确认"被拒绝 ≠ 没响应"：从 netcheck Pod 里探测，看是超时还是 refused
> kubectl -n damo-app run netcheck --rm -it --restart=Never \
>   --image=11.0.1.128:30000/busybox:1.36 -- sh -c '
>     nc -vz -w 3 damo-app.damo-app.svc.cluster.local 80
>     # 预期：nc: ... timed out  ← 超时 = 被静默丢包（若是 refused 则说明策略没生效）
>   '
>
> # B. ★ 同集群内的"对照组"：从别的命名空间访问同一个容器端口（不受本命名空间策略影响）
> kubectl -n default run ncpoint --rm -it --restart=Never \
>   --image=11.0.1.128:30000/busybox:1.36 -- sh -c '
>     nc -vz -w 3 <damo-app 的 Pod IP> 80'
>     # 预期：连接被拒绝 ← 表示"端口真的没人监听"，对照组通了
>     #      与 A 的"超时"对比，★ 直接证明"默认拒绝 = 静默丢包"
> ```
>
> | 现象组合 | 结论 | 面试价值 |
> |---------|------|---------|
> | 命名空间内**超时** ＋ 命名空间外 **refused** | Calico 正在静默丢包——策略**确实生效** | ★★★★ 直接实证"默认拒绝 = 静默丢包" |
> | 两者都是 **refused** | 策略没生效（查 CNI / 策略 `podSelector` 是否匹配上 Pod） | 快速区分"策略问题"和"应用问题" |
> | 两者都 **超时** | 可能 Pod 本身挂了 / 或该端口被别的机制拦住 | 回到 `describe pod` 看 Events |
>
> 其中 B 这一步是个通用技巧：**排查"连不上"时，是「端口没人听」还是「流量被拦」，用一个"放在别处的对照组"就能一刀切开**。

> **★ 面试话术（一句话版，已按实测修正）**：
> 「网上都说默认拒绝会先误杀 kubelet 探针，但我实测**并没有**——Endpoints 仍有 Pod IP、RESTARTS 不增长。分层排除后定位到：
> **kubelet 的 `tcpSocket` 探针目标是本机 Pod，流量走本机路由直达容器 veth，不进入 Calico 的 `cali-*` 策略链**，所以 `podSelector: {}` 管不到它。
> 而真正需要放行节点网段的理由是 **NodePort 经 kube-proxy SNAT 后源 IP 变成节点 IP**。
> 我也顺手验了策略确实在生效：集群内 `nc` 表现为 **`Connection timed out`（静默丢包）**，而外部 NodePort 直接超时。
> **判据是「连接语义从 refused 变成 timed out」，不是 Pod 状态。**」

> **★ 这一节的价值（面试可以直接讲，修正版）**：
> 「这个实验最大的收获不是"我配了默认拒绝"，而是**发现一个被广泛传错的因果关系**：
> 『探针流量来自节点、没有 Pod 身份、所以会被默认拒绝误杀』——这个链条**只在"探针流量需要经 CNI 转发判定"的实现下成立**。
> 在我这套 Calico IPIP 集群里，探针走本机路径、不进策略链，所以**什么都不会发生**。
> 我原本以为会看到"探针挂 → Endpoints 空 → 502"的连环效应，结果第三步命令返回的是 `READY 1/1` 和完整的 Endpoints ——
> **是这次"预期落空"逼着我把 CNI 的流量路径搞清楚，也让白名单里"放行节点网段"这条规则的必要性被重新归因。**」

> **★ 本节最有价值的一条通用结论**：
> **默认拒绝生效的严谨判据是"连接语义"而不是"资源状态"。**
> 本例里 `damo-app` 的 Pod 完全健康（Ready、Endpoints 齐全、零重启），但业务已经**全断**。
> **只做 Pod 存活告警的监控会把这次故障完整漏掉** —— 能发现它的是基于可用性（`curl`）的业务告警（对照项目 C）。

### 4.4 第二步：逐条放行（三条 Ingress 规则）

`manifests/21-netpol-02-allow-ingress.yaml`：

```yaml
# ============================================================
# 项目 F · NetworkPolicy 第 2 步：Ingress 白名单（三条）
#
# 每条规则的"必要性"都对应一个真实的失败现象——
# 不是照抄，是从"默认拒绝后什么坏了"倒推出来的。
#
# ★ 为什么可以和 default-deny 共存：
#   NetworkPolicy 是【叠加】的（多个策略之间取"允许"的并集），不存在优先级覆盖。
#   所以"全拒策略 + 白名单策略"共存，最终效果 = 白名单。
#   这是 NetworkPolicy 与防火墙规则最大的区别：没有"顺序"，只有"并集"。
# ============================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-damo-app-ingress
  namespace: damo-app
spec:
  podSelector:
    matchLabels:
      # 只作用于业务 Pod（不影响同命名空间的其它 Pod）
      app.kubernetes.io/name: damo-app
  policyTypes:
    - Ingress
  ingress:
    # ── 规则① 节点网段：覆盖 NodePort 的 SNAT 源 IP ──
    # 为什么必须（★ 已按实测归因，见 §4.3.2）：
    #   NodePort(默认 Cluster 策略) 的流量经 kube-proxy SNAT 后，源 IP 变成
    #   【接收请求的那个节点】的 IP —— 它没有 Pod 身份，只能用 ipBlock 覆盖。
    # ★ 常见误传：说这条规则是为了"放行 kubelet 探针"。实测不成立 ——
    #   tcpSocket 探针目标是本机 Pod，流量走本机路由直达容器 veth，
    #   【不进入 Calico 的 cali-* 策略链】，因此不受默认拒绝影响。
    #   （若你的 CNI/探针类型让探针流量需经策略判定，那探针也会需要这条放行。）
    # 为什么给整个节点网段而不是单个节点 IP：
    #   SNAT 源是"接收请求的那个节点"，三个节点都可能，写网段最简洁。
    # 取舍（面试可讲）：这是"必要之恶"——ipBlock 没有身份概念，粒度比标签粗。
    #   但 SNAT 后的流量确实没有身份可用，这是唯一的办法。
    - from:
        - ipBlock:
            cidr: 11.0.1.0/24
      # 只放行应用端口（80），不要放行全部端口
      ports:
        - protocol: TCP
          port: 80

    # ── 规则② 同命名空间的压测/联调负载 ──
    # 为什么必须：项目 G 的 loadgen 要从集群内打流量
    # 为什么用标签而不是命名空间：只要同命名空间里带这个标签的 Pod，
    #   范围比"整个命名空间"更小、更精确
    - from:
        - podSelector:
            matchLabels:
              app: loadgen

    # ── 规则③ ingress-nginx 命名空间（本项目主用 NodePort，此项为通用性预留）──
    # 为什么保留：项目 A 的 WordPress 走 Ingress 暴露；
    #   若将来 damo-app 也加 Ingress，这条规则无需再补
    # 为什么用 namespaceSelector 而非 podSelector：
    #   只关心"来自哪个命名空间"，不关心具体是哪个 Pod（ingress controller 会重建）
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
```

> ⚠️ **规则③ 的命名空间标签是自动的**：`kubernetes.io/metadata.name: <ns-name>` 是 K8s 1.21+ **自动为每个命名空间打的标签**，**不需要手动给 `ingress-nginx` 打标签**。验证时先确认：
> ```bash
> kubectl get ns ingress-nginx --show-labels
> ```

```bash
kubectl apply -f manifests/21-netpol-02-allow-ingress.yaml
```

#### 验证放行是否生效（四道关 · ★ 已按 2026-09-23 实测重写）

> **⚠️ 先读这条：验证方法本身最容易出错**
> 本次实操踩了两个"测法陷阱"，导致前几轮把「策略完全正确」误判成「策略不生效」：
> 1. **代理陷阱**：`curl` 读了 `http_proxy`，把请求发给了外部代理 → 全部假超时。**判定连通性一律加 `--noproxy '*'`**（详见 §0.6「坑 0」）。
> 2. **源身份陷阱**：**从节点（host）上 curl 集群地址，源身份是"节点"而不是"Pod"**，而白名单是按 Pod 标签写的 —— 节点流量本来就不在白名单里。**验证白名单必须从 Pod 内部发起，不要从节点发起。**
>
> 最可靠的验证方式是**用白名单里明确写了的那个身份去访问**——本项目的规则② 就是为此存在的。

```bash
# ────────────────────────────────────────────────────────────
# 关① ★ 白名单真的在放行（用白名单里的身份访问）
#   规则② 写的是 podSelector: app=loadgen
#   → 那就用一个 app=loadgen 的 Pod 去访问，这是【最能证明放行生效】的一关
#   （实测 2026-09-23：返回 <!DOCTYPE html>，放行生效 ✅）
# ────────────────────────────────────────────────────────────
kubectl -n damo-app run loadgen-test --rm -it --restart=Never \
  --labels="app=loadgen" --image=11.0.1.128:30000/busybox:1.36 -- \
  sh -c 'wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ | head -3'
# 期望：返回 <!DOCTYPE html>（页面内容）★ 这是"放行生效"最直接的证据
# ⚠️ 若报 ImagePullBackOff：busybox:1.36 没中转进私有仓库（§0.5 有中转命令）

# ────────────────────────────────────────────────────────────
# 关② ★ 边界真的存在（用【不在】白名单里的身份访问，必须被拒）
#   damo-app 自己的标签是 app.kubernetes.io/name=damo-app，
#   它【不在】白名单的任何一条规则里 → 按策略语义就该被拒
#   （实测 2026-09-23：wget: download timed out ✅）
#   ★ 这一关和关① 是"一正一反"，合起来才能证明策略是【精确】放行而非全通
# ────────────────────────────────────────────────────────────
kubectl -n damo-app exec deploy/damo-app -- \
  sh -c 'wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ | head -2'
# 期望：wget: download timed out  ★ 被拒才是对的！
# ⚠️ 别用 `... | head -2; echo $?` 判退出码 —— $? 是 head 的（恒为 0）。
#    要看真实退出码用：sh -c 'wget -qO- ... ; echo ${PIPESTATUS[0]}'

# ────────────────────────────────────────────────────────────
# 关③ Pod 与 Endpoints 健康（本集群默认拒绝期间本来就正常，只做"没回退"确认）
# ────────────────────────────────────────────────────────────
kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app
# 期望：READY 1/1、RESTARTS 稳定不增长
kubectl -n damo-app get endpoints damo-app
# 期望：ENDPOINTS 列出现 Pod IP:80（两个 IP 都在）

# ────────────────────────────────────────────────────────────
# 关④ ★ 跨命名空间访问仍然被拒（证明白名单没放太宽）
# ────────────────────────────────────────────────────────────
kubectl -n default run netcheck-x --rm -it --restart=Never \
  --image=11.0.1.128:30000/busybox:1.36 -- sh -c '
    wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ | head -2 \
      || echo "★ 从 default 访问被拒绝 —— 这正是我们要的效果"'
# 期望：timed out（静默丢包）
```

> **📌 「外部 NodePort 访问」（`curl http://11.0.1.128:30090/`）—— ★ 2026-09-23 抓包定位，推翻了本文档此前的假设**
>
> **双向抓包的实证（master + work2 同时抓，这是最有说服力的一对证据）**：
> ```
> # 【master 上】Windows 客户端的 SYN 进来，源 IP = 11.0.1.1（∈ 11.0.1.0/24，本来该命中规则①）
> 16:33:42.682982 ens33 In   IP 11.0.1.1.64243 > 11.0.1.128.30090: Flags [S]
> # 【master 上】同一个包出 IPIP 隧道时，源 IP 被 MASQUERADE 成 10.244.219.64（Pod 网段！），源端口也变了
> 16:33:42.683110 tunl0 Out IP 10.244.219.64.46979 > 10.244.123.6.80: Flags [S]
>
> # 【work2 上】对照组：源 = 节点自身 IP 11.0.1.129 → 有 SYN-ACK，放行 ✅
> 16:32:12.049939 cali56024dd9176 Out  11.0.1.129.58924 > 10.244.123.6.80: Flags [S]
> 16:32:12.050009 cali56024dd9176 In   10.244.123.6.80 > 11.0.1.129.58924: Flags [S.]   ← SYN-ACK 回来了
> # 【work2 上】NodePort 来的：源 = 10.244.219.64 → SYN 重传、无 SYN-ACK，说明被策略静默丢弃 ❌
> 16:32:20.493206 tunl0 In  10.244.219.64.44061 > 10.244.123.6.80: Flags [S]
> 16:32:21.505431 tunl0 In  10.244.219.64.44061 > 10.244.123.6.80: Flags [S]   ← 重传 = 没收到回应
> ```
>
> **结论（本集群实测，Calico IPIP + K8s v1.35）**：
> | 源 IP | ∈ `11.0.1.0/24`？ | 命中规则①？ | 结果 |
> |---|---|---|---|
> | `11.0.1.129`（节点自身 host 流量） | ✅ | ✅ | **放行**（SYN-ACK 正常） |
> | `10.244.219.64`（NodePort 出隧道后） | ❌（Pod 网段） | ❌ | **静默丢弃**（无 SYN-ACK） |
>
> **★ 所以 `ipBlock: 11.0.1.0/24` 永远匹配不到 NodePort 流量。**
> 「NodePort 经 kube-proxy SNAT 后源 IP 变成**节点 IP**」这个流传很广的说法，**在本集群不成立** ——
> Calico IPIP 在**出 `tunl0` 隧道时做了 MASQUERADE**，把源 IP 改写成了**节点隧道地址 `10.244.219.64`（落在 Pod 网段）**。
> 这不是配置写错，是**依据的前提本身不成立**。
>
> **修正方案（★ 本项目已采纳 A，2026-09-23）**：
>
> | 方案 | 做法 | 优点 | 代价 |
> |---|---|---|---|
> | **A ✅ 已采纳 `externalTrafficPolicy: Local`** | `kubectl -n damo-app patch svc damo-app -p '{"spec":{"externalTrafficPolicy":"Local"}}'`（固化路径：改 Chart 的 `service.externalTrafficPolicy`，见项目 E 的 `values.yaml`） | NodePort **不再 SNAT**，源 IP 保留为真实客户端 IP（Windows = `11.0.1.1` ∈ `11.0.1.0/24`）→ **规则① 才真正有意义**，边界精确 | 流量只落到**接收节点上的 Pod**；master 上没有 Pod → `11.0.1.128:30090` 会 refused，需改打 `11.0.1.129:30090` / `11.0.1.130:30090` |
> | B（未采纳）放宽 CIDR | 规则① 的 `cidr` 从 `11.0.1.0/24` 改为 `10.244.0.0/16` | 不用动 Service，NodePort 立即恢复 | **Pod 网段整体成为可信来源**，粒度很粗，会削弱"精确放行"的叙事 |
>
> **⚠️ 无论选哪个，都必须把「NodePort + NetworkPolicy 天然不兼容精确 ipBlock 放行」这条写进面试话术**——
> **这正是生产环境更推荐 Ingress/LoadBalancer 而非 NodePort 的原因之一**（NodePort 会把客户端身份抹掉）。

**执行与验证（采纳 A 之后照这个顺序做 —— ★ 以下为 2026-09-23 实测通过的结果）**：

```bash
# ① 改 Service（热更新，不需要重建 Pod —— unlike *Probe，这个字段是可变的）
kubectl -n damo-app patch svc damo-app -p '{"spec":{"externalTrafficPolicy":"Local"}}'
kubectl -n damo-app get svc damo-app -o jsonpath='{.spec.externalTrafficPolicy}{"\n"}'
# 实测：Local ✅

# ② ★ 遍历三节点看 NodePort 下发范围（这一步直接暴露 Local 的代价）
for ip in 11.0.1.128 11.0.1.129 11.0.1.130; do
  echo -n "$ip:30090 → "
  curl --noproxy '*' -s -o /dev/null -w '%{http_code}\n' --max-time 3 http://$ip:30090/ || echo "不通"
done
# 实测结果（★ 与预期完全一致）：
#   11.0.1.128:30090 → 000（master 无 Pod，refused）
#   11.0.1.129:30090 → 200（work2 有 Pod）✅
#   11.0.1.130:30090 → 200（work1 有 Pod）✅

# ③ ★ 在 Windows 宿主机上验证真实外部客户端 —— 规则① 的最终判定
#     PowerShell：curl.exe --noproxy "*" -s -o NUL -w "%{http_code}`n" --max-time 5 http://11.0.1.129:30090/
# 实测：200 ✅ ★ 外部访问恢复，证明 ipBlock: 11.0.1.0/24 现在能正确匹配了

# ④ （可选）抓包复核源 IP 已保留、不再被改写
#     在 work2 上：sudo tcpdump -i any -nn "host <PodIP> and tcp port 80" -c 10
#     期望：源 IP 是客户端真实 IP（如 11.0.1.1），而不再是 10.244.219.64
```

> **★ 结论**：`externalTrafficPolicy: Local` 让 NodePort 的源 IP 从"被改写的 Pod 网段地址"变回
> "客户端真实 IP"，规则① 的 `ipBlock: 11.0.1.0/24` 才真正生效。**代价是 master 节点不再提供
> NodePort 服务**（它没有业务 Pod）。
>
> **★ 这个代价在项目 G（HPA）下需要特别注意**：HPA 会让 Pod 漂移，**写死某个节点 IP 会间歇性失败**。
> 所以对外验证一律用「自动挑一个跑着 Pod 的节点」的写法（见 §0.5 的推荐命令）。

### 4.5 第三步：加 Egress 默认拒绝（分步做，别一次全拒）

```bash
# 先用 loadgen 制造一个"需要出方向"的场景，这样 egress 的效果可见
kubectl apply -f ../K8s项目G_HPA弹性伸缩/manifests/loadgen.yaml
kubectl -n damo-app get pods -l app=loadgen
kubectl -n damo-app logs -l app=loadgen --tail=3     # 基线：正常
```

`manifests/22-netpol-03-default-deny-egress.yaml`：

```yaml
# ============================================================
# 项目 F · NetworkPolicy 第 3 步：Egress 默认拒绝
#
# 与 Ingress 版同样的写法，只是方向相反：
#   podSelector: {}        → 本命名空间全部 Pod
#   policyTypes: [Egress]  → 只管控出方向
#   不写 egress: 段        → 全拒
#
# ★ 两个必须理解的机制：
# ① 有状态（stateful）：入方向被允许的连接，其【响应包自动放行】。
#    所以"加 egress 全拒"不会导致"外部来访问、但响应发不回去"。放心。
#    真正被挡住的是【主动发起】的出方向连接 —— 最主要的就是 DNS 查询。
# ② 出方向全拒会立刻打断 DNS 解析 —— 这是本步要演示的现象。
# ============================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: damo-app
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

```bash
kubectl apply -f manifests/22-netpol-03-default-deny-egress.yaml

# ════════════════════════════════════════════════════════════════════════
# ⚠️ 先读这条：★ 2026-09-23 实操教训 —— "分步做"必须真的分步
# ════════════════════════════════════════════════════════════════════════
#   【结论先行】本节"egress 全拒会打断 DNS"的预期是【对的】。
#     本次实操之所以没看到，是因为 4.6 的 `allow-egress-dns` 被【提前应用】了。
#
#   排查证据（一条命令就够）：
#     $ kubectl -n damo-app get netpol
#     NAME                     POD-SELECTOR                      AGE
#     allow-damo-app-ingress   app.kubernetes.io/name=damo-app   6h5m
#     allow-egress-dns         <none>                            5h15m   ← ★ 已经 apply 了
#     default-deny-egress      <none>                            5h24m   ← 只早 9 分钟
#     default-deny-ingress     <none>
#
#   → 当时观察到的"DNS 正常"——
#     $ kubectl -n damo-app exec "$POD" -- nslookup damo-app.damo-app.svc.cluster.local
#     Server:  10.96.0.10
#     Address: 10.96.0.10:53
#     Name:    damo-app.damo-app.svc.cluster.local
#     Address: 10.100.232.111
#   → 是【4.6 的 allow-egress-dns 规则】放行的，不是"策略盲区"。
#
#   ★★ 教训（比本节任何现象都重要）：
#     "分步做"的意义就是【每一步只引入一个变量】。
#     提前执行下一步，会让上一步该观察到的现象【消失】，
#     而且消失得毫无痕迹 —— 你会以为是机制不对，其实是自己做早了。
#   → 想复现"DNS 断"，先把 4.6 的规则删掉：
#       kubectl delete -f manifests/23-netpol-04-allow-egress-dns.yaml
#
#   ★ 意外收获（一个很漂亮的证据）：同端口 53，去不同目标结果不同 ——
#       nslookup damo-app.damo-app.svc.cluster.local              → 解析成功（去 kube-dns）
#       nslookup damo-app.damo-app.svc.cluster.local 10.100.232.111
#                                                                 → connection timed out（去 damo-app:53）
#     ★ 这恰好证明 23-*.yaml 的 egress 白名单是【按目标收窄】的，不是按端口放开的：
#       精确到"只有 kube-dns 的 53"，多一个目标都不给。
#       这是"放行要精确到刚好够用"的绝佳实证。
#
#   【判 egress 必须用"没有 ingress 混淆"的身份】—— 用 loadgen：
#     loadgen 命中 ingress 规则②（入方向已允许），于是"通不通"只取决于出方向。
#     若用 damo-app 自己的 Pod 去测，会被 ingress 白名单同时拦掉，无法归因。
#
#   【另一个必须记住的坑】wget 的报错文案【不能】用来区分故障类型：
#     Calico 是静默丢包，DNS 查询被丢时是"挂住"而不是"返回错误"，
#     wget 的 --timeout 闹钟先响 → 同样打印 `download timed out`。
#     "DNS 断了" 和 "TCP 被拦" 会打印【同一句话】。
#     → 判 DNS 用 nslookup / getent；判 TCP 用 IP 直连。一次只测一个变量。
# ════════════════════════════════════════════════════════════════════════

# 基线（可选，要在 apply egress deny 【之前】做）：loadgen 身份访问 → 能拿到页面
kubectl -n damo-app exec deploy/loadgen -- sh -c '
  wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ 2>&1 | head -2'
# 期望（egress 全拒【之前】）：返回 <!DOCTYPE html> ★ 规则② 放行了 ingress

# ────────────────────────────────────────────────────────────────────
# 观察 1（★ egress 收紧的直接证据）：用 loadgen —— 无 ingress 混淆
#   loadgen 带 app=loadgen → 命中规则②，入方向允许；
#   于是"通不通"只取决于【出方向】。
# ────────────────────────────────────────────────────────────────────
kubectl -n damo-app exec deploy/loadgen -- sh -c '
  wget -O /dev/null --timeout=5 http://damo-app.damo-app.svc.cluster.local/ 2>&1 | head -3'
# 实测（★ 与预期完全一致）：
#   Connecting to damo-app.damo-app.svc.cluster.local (10.100.232.111:80)
#   wget: download timed out
#   ★ 第一行说明【DNS 已经解析成功】（拿到了 IP），超时发生在 TCP 阶段
#     → 唯一拦截者就是 egress。这是干净证据。

# 观察 1b（★ 意外收获）：同端口 53、去不同目标，结果不同
kubectl -n damo-app exec deploy/loadgen -- sh -c '
  nslookup damo-app.damo-app.svc.cluster.local 2>&1 | head -4'
# 实测：解析出 10.100.232.111 —— 去 kube-dns 通（23-*.yaml 放行了它）
kubectl -n damo-app exec deploy/loadgen -- sh -c '
  nslookup damo-app.damo-app.svc.cluster.local 10.100.232.111 2>&1 | head -3'
# 实测：;; connection timed out; no servers could be reached —— 去 damo-app:53 不通
# ★ 这证明 23-*.yaml 的 egress 白名单是【按目标收窄】的：只有 kube-dns 的 53，
#   不是"所有 53 端口"。放行精确到"刚好够用"。

# ────────────────────────────────────────────────────────────────────
# 观察 2：★ 业务页面仍然正常（有状态，入方向响应不受影响）
#   ⚠️ 2026-09-23 起 Service 是 externalTrafficPolicy: Local，
#      必须打【有 Pod 的节点】，不能打 master(11.0.1.128)
# ────────────────────────────────────────────────────────────────────
NODE=$(kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
        -o jsonpath='{.items[0].status.hostIP}')
curl --noproxy '*' -s --max-time 5 "http://$NODE:30090/" | grep -o 'Build #[0-9]*'
# 预期：仍能返回 Build #N ★ 这正是"NetworkPolicy 是有状态的"的实证！
```

> **★ 观察 2 是本节的高光点**：它证明了「NetworkPolicy 有状态」——**入方向被允许的连接，其响应包自动放行**。
> 很多人担心"加了 egress 全拒服务就废了"，实际不会。真正被挡的是**主动发起的出方向连接**。
>
> **★ 而观察 1 是另一半**：**业务对外提供服务的链路完好，但它自己想访问任何东西都出不去。**
> 同一个 Pod，入方向全通、出方向全断 —— 这就是「方向性管控」最直观的画面。
>
> **★★ 本节最值钱的一课，其实是"分步做被破坏"的后果**：
> 我们在完成 4.5 的观察**之前**就把 4.6 的 `allow-egress-dns` 应用了，
> 于是 4.5 该出现的"DNS 断"现象**彻底消失、毫无痕迹** —— 排查时一度被误判成
> "Calico 对 DNS 有隐式豁免（failsafe）"，绕了一大圈才回到原点。
> **教训：分步实验的全部价值就是"每步只引入一个变量"。一旦跳步，
> 你连"我应该看到什么"都不知道，只能靠猜机制 —— 而猜出来的机制往往是错的。**
> 面试话术：「我遇到过一种很难查的情况 —— **验证时现象没出现，最后发现是我提前做了下一步**。
> 从那以后做加固我一定**一次只加一条策略、跑一次验证**：因为**跳过一步，
> 你会连"该看到什么"都不知道**。」
>
> **★ 另一个必须记住的坑**：「用 `wget` 的报错文案判断故障类型」在这里失效——
> **静默丢包让"DNS 故障"和"TCP 被拦"打印出同一句话**（都是 `download timed out`）。
> 判断 DNS 是否可用要用 `nslookup`/`getent`；判断 TCP 是否可达要用 **IP 直连**。
> **一次只测一个变量。**

### 4.6 第四步：放行 DNS（最小必要）—— ★ 实测发现"不放行也能通"

> **★ 本节是 4.5 的收尾，也是"分步做"的兑现点（2026-09-23 实测通过）**
>
> 4.5 apply `default-deny-egress` 之后，Pod 的 DNS 会**立刻断掉**（连去 kube-dns 的 53 也被拒）。
> 本节把 DNS 单独放回来 —— **只开一条缝，不是开一扇门**。
>
> **⚠️ 一条实操提醒**：本次实操中我们**提前**应用了本节的规则（`get netpol` 里
> `allow-egress-dns` 比 `default-deny-egress` 只晚 9 分钟），导致 4.5 的"DNS 断"现象
> **完全看不到了**，排查时一度误判成"Calico 有隐式豁免"（详见 §4.5 开头的教训）。
> **务必按 4.5 → 观察 → 4.6 的顺序做，一次只加一条策略。**
>
> **★★ 本节高光：这条规则"按目标收窄"的精确性，有实测证据** —— 同端口 53 去不同目标结果不同：
> ```
> nslookup damo-app.damo-app.svc.cluster.local                    → 解析成功（去 kube-dns，被放行）
> nslookup damo-app.damo-app.svc.cluster.local 10.100.232.111
>                                                                 → connection timed out（去 damo-app:53，被拒）
> ```
> **同样是 53 端口，去 kube-dns 通、去别处不通** —— 这就是"最小必要"落地后的样子，
> 也再次说明 **`ipBlock`/`port` 这类"宽带式"写法 vs `namespaceSelector + podSelector`
> 这种"精确到身份"的写法，安全含义完全不同**。

`manifests/23-netpol-04-allow-egress-dns.yaml`：

```yaml
# ============================================================
# 项目 F · NetworkPolicy 第 4 步：放行 DNS 出方向
#
# 关键设计（面试可讲）：
# ① 只放行 UDP/TCP 53，【不是】放行所有出方向 —— 最小必要；
# ② 目标用 namespaceSelector 锁死 kube-system，且用 podSelector
#    锁定 k8s-app=kube-dns 的 Pod —— 双重收敛；
# ③ 为什么不直接放行 0.0.0.0/0？那等于没有 egress 限制，
#    攻击者可以把数据外传（数据渗漏）。放行 DNS 是"够用即止"。
#
# 如果某天 damo-app 需要调用外部 API，再单独加一条 ipBlock 规则，
# 并写上注释说明为什么需要 —— 让每一条出方向放行都有明确理由。
# ============================================================
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns
  namespace: damo-app
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

> **注意上面 `to` 里的写法**：`namespaceSelector` 和 `podSelector` 写在**同一个数组元素**里，是 **AND** 关系（既在 kube-system 命名空间，又带 `k8s-app=kube-dns` 标签）。若把它们写成两个数组元素就是 **OR**——这是 NetworkPolicy 最阴的语法点，详见第 8 章 Q11。

```bash
kubectl apply -f manifests/23-netpol-04-allow-egress-dns.yaml

# ────────────────────────────────────────────────────────────────
# 验证 ①：★ DNS 恢复（本节的"生效证明"）
# ────────────────────────────────────────────────────────────────
POD=$(kubectl -n damo-app get pod -l app.kubernetes.io/name=damo-app -o jsonpath='{.items[0].metadata.name}')
kubectl -n damo-app exec "$POD" -- sh -c '
  nslookup damo-app.damo-app.svc.cluster.local 2>&1 | head -6'
# 期望：解析出 10.100.232.111
# ★ 对照组：只有 22（egress 全拒）而没有本条规则时，这条会失败 —— 一负一正才是证明。
# ★ 顺便它也是"回归测试"：namespaceSelector / podSelector 写错（比如拆成两个数组元素
#   变成 OR、或 k8s-app 标签拼错）时，这条同样会挂。

# ★★ 想【亲眼看到 DNS 断】、复现 4.5 的观察（推荐做一次，这是最直观的证明）：
#     kubectl delete -f manifests/23-netpol-04-allow-egress-dns.yaml
#     kubectl -n damo-app exec "$POD" -- nslookup damo-app.damo-app.svc.cluster.local
#   预期：;; connection timed out; no servers could be reached
#     ★ 这个报错文案本项目已经实测过 —— 就是"问 damo-app:53"那条的输出，
#       说明 Calico 对 DNS 的拦截是【静默丢包】，解析器只能等到超时。
#   看完记得重新 apply：
#     kubectl apply -f manifests/23-netpol-04-allow-egress-dns.yaml

# ────────────────────────────────────────────────────────────────
# 验证 ②：★ 真正能看出 egress 收紧的地方 —— loadgen 的压测链路还是断的
# ────────────────────────────────────────────────────────────────
kubectl -n damo-app exec deploy/loadgen -- sh -c '
  wget -O /dev/null --timeout=5 http://damo-app.damo-app.svc.cluster.local/ 2>&1 | head -3'
# 期望：download timed out
#   此时 DNS 是好的（failsafe 豁免），所以这个超时一定发生在 TCP 阶段 = 被 egress 丢了
#   ★ 注意：loadgen 的压测链路断了 —— 项目 G 的 HPA 会因此扩不起来（见下面第 5 步）

# ══════════════════════════════════════════════════════════════════
# ★ 4.6 补充（第 5 步）：放行 loadgen 的出方向 —— 让压测链路恢复
#    manifests/24-netpol-05-allow-loadgen-egress.yaml
# ══════════════════════════════════════════════════════════════════
kubectl apply -f manifests/24-netpol-05-allow-loadgen-egress.yaml

# 验证 ③：loadgen 的压测链路恢复（★ 这才是"出方向放行生效"的证据）
kubectl -n damo-app exec deploy/loadgen -- sh -c '
  wget -qO- --timeout=5 http://damo-app.damo-app.svc.cluster.local/ 2>&1 | head -2'
# 期望：返回 <!DOCTYPE html> ★ 从"download timed out"变回"有内容"

# 验证 ④：其它出方向仍然被堵住（"够用即止"的证据）
kubectl -n damo-app exec deploy/loadgen -- sh -c '
  wget -q -O /dev/null --timeout=5 http://www.baidu.com 2>&1 | head -2'
# 期望：不可达 —— 只开了"到业务自身的 80 + DNS"，没有开 0.0.0.0/0
#   ⚠️ 前提提醒：本集群 Pod 内【没有代理】，外网本来就可能不通（§0.6 第 ⑤ 条）。
#      想拿强证据，临时删掉 egress deny 再测一次：能通 → 说明确实是 egress 拦的。
```

> **★★ 4.6 真正的教学高光：一次成功请求要过两道关**
>
> 上面「验证 ③」才是本节最有价值的一步。它讲清了一个**最容易被忽略的机制**：
>
> ```
> loadgen Pod ──egress(24-*.yaml)──► damo-app Pod ──ingress(21-*.yaml 规则②)──► 通
>             ▲                                              ▲
>      源端出方向必须放行                            目的端入方向必须放行
> ```
>
> **一次成功的 Pod 间通信，去程要过"源端 egress + 目的端 ingress"两道策略。**
> 所以「我 ingress 白名单明明写了，为什么还是不通」—— 十有八九是**出方向没放**。
> **这个坑在 egress 默认拒绝之后会变得非常频繁**，面试时能讲出来很有说服力。
>
> **★ 关于 4.6 的"够用即止"**：放行之后，应用能解析域名、能访问业务自身，
> 但**外网和其它外部目标依然不可达** —— 因为规则只开了 DNS 和到 damo-app 的 80，
> 没有开 `0.0.0.0/0`。**这才是 egress 管控的真正价值：不是"断网"，而是"只开必要的那一条缝"。**
> 面试话术：「我放行 DNS 时没有图省事写 `0.0.0.0/0`，而是用
> `namespaceSelector + podSelector` 双重锁死到 `kube-system/kube-dns` 的 53 端口——
> **留一条缝和开一扇门，安全含义完全不同**。」
>
> **⚠️ 跨项目提醒**：上了 egress 默认拒绝之后，**项目 G 的 HPA 压测会失效**
> （loadgen 出不去）。所以 24-*.yaml 这一步**不是可选项**，是保住项目 G 的必要动作。

### 4.7 把 NetworkPolicy 也写进 Chart（★ 加固不能被流水线冲掉）

**和阶段 3.3 同样的道理**：手 apply 的策略在集群里，但**不在 git 里、不在流水线的交付物里**。一旦命名空间被重建（灾备演练、环境重建），策略就丢了 —— 而且**没人会发现，因为服务还是通的**（默认全通）。

> **★ 这一步把「实验成果」变成「交付物」**：4.1–4.6 是**手 apply** 的（学习路径，每一步都看得见）；
> 4.7 把它们**固化进 Chart**（交付路径）。固化完成后，**手 apply 的那批要删掉**，
> 从此以 Chart 为准 —— 否则集群里会同时存在两套同义策略，谁也不知道该改哪一份。

#### 4.7.1 三个文件动作

**① 放模板**：把 `chart-patch/networkpolicy.yaml` 拷到 Chart 里

```bash
# 在 master 上
cp ~/F/chart-patch/networkpolicy.yaml  <chart目录>/templates/networkpolicy.yaml

# ★★ 拷完立刻自检（这一步能提前抓住"文件没更新"）
grep -c "kind: NetworkPolicy" <chart目录>/templates/networkpolicy.yaml
# 期望：5
# ★ 若得到 4 → 你拷的是【旧版模板】（旧版只有 4 条、没有 ⑤）：
#    症状是"values 里配了 egress 也渲染不出第 5 条、loadgen 一直不通"，
#    因为旧模板里压根没有 ⑤ 那一段。详见 §4.7.1 的踩坑记录。
# ★ 数【模板文件本身】的条数，比数渲染结果更早暴露"文件没更新"。
```

**② 加 values**：用 `chart-patch/values.yaml` **覆盖** chart 的 `values.yaml`

```bash
cp chart-patch/values.yaml <chart目录>/values.yaml
```

> **★ 为什么是"覆盖"而不是"合并片段"**：项目 F 对 values 有**四处新增**
> （`service.externalTrafficPolicy` / `probes.timeoutSeconds` / `serviceAccount` 段 / `networkPolicy` 段），
> 分散在文件各处，摘片段容易漏。完整文件覆盖最不容易出错。
> **旧的 `chart-patch/values-fragment.yaml` 已被它取代**（保留仅作参考）。
>
> **★ 或者更省事**：`chart-patch/README.md` 里有一份**一次搞定的 `cp` 脚本**
> （把本目录 6 个文件按"文件 → chart 位置"的映射一次拷完），照着跑就行。

**③ 确认 Service 的前置条件**（★ 这一步不做，规则① 会"看起来没生效"）

```bash
kubectl -n damo-app get svc damo-app -o jsonpath='{.spec.externalTrafficPolicy}{"\n"}'
# 期望：Local
# 若不是 Local：NodePort 的源 IP 会被 MASQUERADE 成 Pod 网段地址，
# 规则① 的 ipBlock: 11.0.1.0/24 永远匹配不到 → 外部访问全断（详见 §4.4 抓包）
```

> **★★ 关键：它必须在 `values.yaml` 里，不能是"手 patch 出来的"**
>
> 阶段 4.4 时我们是**热更新**加的它：
> `kubectl patch svc damo-app -p '{"spec":{"externalTrafficPolicy":"Local"}}'`
> —— 这**只改线上，不进 git**。若 4.7 时不把它写进 `values.yaml`，就会出现：
>
> | 状态 | 在线上？ | 在 git？ | Helm 管？ |
> |---|---|---|---|
> | 手 patch 的 `Local` | ✅ | ❌ | ❌ → **编外状态** |
>
> **后果**：任何一次资源重建（`helm uninstall` 重装 / 命名空间重建 / 灾备演练）
> → `Local` 消失 → NodePort 源 IP 又被改写成 `10.244.219.64`
> → **规则① 失效、外部访问全断 —— 也就是 §4.4 那个问题原样复现。**
>
> **和"用 `--set` 打开加固开关"是同一类病**：状态在集群里，不在 git 里。
> **4.7 的固化必须是"完整"的** —— `NetworkPolicy` + `externalTrafficPolicy` **一起固化**，
> 因为它们是一对绑定关系（策略的正确性依赖后者的存在）。
>
> ```yaml
> # values.yaml 里必须有（与 type / port 同级）
> service:
>   type: NodePort
>   port: 80
>   nodePort: 30090
>   externalTrafficPolicy: Local      # ★ 规则① 的配套前置条件
> ```
>
> **并且 `templates/service.yaml` 要能渲染它**（只在 NodePort 类型下渲染）：
> ```yaml
> {{- if and (eq .Values.service.type "NodePort") .Values.service.externalTrafficPolicy }}
> externalTrafficPolicy: {{ .Values.service.externalTrafficPolicy }}
> {{- end }}
> ```
> 自查：`grep -n "externalTrafficPolicy" values.yaml templates/service.yaml`

模板会渲染出 **5 份策略**（2 全拒 + 3 白名单）—— **⚠️ 若只看到 4 份，说明 values 里漏了 `networkPolicy.egress` 段**：

| # | 策略名 | 作用 | 渲染条件 |
|---|---|---|---|
| ① | `damo-app-default-deny-ingress` | 入方向默认拒绝 | 总是 |
| ② | `damo-app-default-deny-egress` | 出方向默认拒绝 | 总是 |
| ③ | `damo-app-allow-ingress` | 白名单：节点网段 / `app=loadgen` / `ingress-nginx` 命名空间 | 总是 |
| ④ | `damo-app-allow-egress-dns` | 只放行 DNS（`namespaceSelector + podSelector` 双重锁定） | 总是 |
| ⑤ | `damo-app-allow-egress-loadtest` | 放行 loadgen 的出方向（**与 ③ 的规则② 镜像**） | **`networkPolicy.egress.loadtestPodLabels` 有值** |

> **★★ 踩坑记录（2026-09-24 实操）**：漏配 `networkPolicy.egress` 段时，
> `helm template \| grep -c "kind: NetworkPolicy"` 只得到 **4**（少了 ⑤），
> 症状是 **`loadgen` 访问业务 → `download timed out`**（因为它的出方向没被放行）。
> **判断方法：先数策略条数**——策略数量对不上，不用往下排查行为。
> （模板已改成 `if and .Values.networkPolicy.egress ...`，values 缺失时不会报错，只是不渲染。）

> **★ 为什么 ⑤ 必须单独一条**：② 和 ④ 的 `podSelector` 都是 `{}`（整个命名空间），
> 在里面加规则等于**给所有 Pod 都开了这个口子**。业务 Pod 的出方向应当保持全拒。
>
> **★ 为什么 ⑤ 和 ③ 是一对**：一次成功通信要过 **源端 egress + 目的端 ingress** 两道关——
> ③ 管"loadgen 能不能进来"，⑤ 管"loadgen 能不能出去"。**只放行一边照样不通。**

#### 4.7.2 ★ 关键动作：先删手 apply 的策略，再让 Chart 接管

**为什么必须删**：Chart 用的是 `damo-app-*` 前缀命名，与手 apply 的 `default-deny-*` / `allow-*`
**名字不同** → 若不移除，集群里会**同时存在两套同义策略**。
NetworkPolicy 是"并集"，功能上不会互相破坏，但**变成两处真相，日后必踩坑**。

```bash
cd ~/F/manifests

# ① 先删手 apply 的 5 条（先删默认拒绝，避免中间态把业务锁死太久）
kubectl delete -f 22-netpol-03-default-deny-egress.yaml   --ignore-not-found
kubectl delete -f 23-netpol-04-allow-egress-dns.yaml      --ignore-not-found
kubectl delete -f 24-netpol-05-allow-loadgen-egress.yaml  --ignore-not-found
kubectl delete -f 21-netpol-02-allow-ingress.yaml         --ignore-not-found
kubectl delete -f 20-netpol-01-default-deny-ingress.yaml  --ignore-not-found

kubectl -n damo-app get netpol
# 期望：No resources found.（此时"裸奔"，但只持续到下一步）

# ② 让 Helm 接管
helm upgrade damo-app <chart目录> -n damo-app \
  --set image.tag=19 \
  --set networkPolicy.enabled=true \
  --atomic --timeout 3m

# ★ --atomic：渲染/等待失败会自动回滚到上一个 revision —— 加固改造必须带这个
# ★ 想先"干跑"看渲染结果（强烈建议）：
#   helm template damo-app <chart目录> -n damo-app --set networkPolicy.enabled=true | grep -A3 "kind: NetworkPolicy"
```

#### 4.7.3 验证（Chart 版同样要过四关）

```bash
# ① 策略数量与命名（应看到 damo-app-* 前缀的 6 条，不再有手 apply 的那批）
kubectl -n damo-app get netpol

# ② ★ 业务仍然可用（不能因为固化把服务搞挂）
NODE=$(kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app \
        -o jsonpath='{.items[0].status.hostIP}')
curl --noproxy '*' -s --max-time 5 "http://$NODE:30090/" | grep -o 'Build #[0-9]*'
# 期望：Build #N

# ③ ★ 白名单在放行（loadgen 身份，镜像关系两边都在）
kubectl -n damo-app exec deploy/loadgen -- sh -c '
  wget -qO- --timeout=5 http://damo-app.damo-app.svc.cluster.local/ 2>&1 | head -2'
# 期望：<!DOCTYPE html>

# ④ ★ 边界仍在（跨命名空间被拒）
kubectl -n default run netcheck-x --rm -it --restart=Never \
  --image=11.0.1.128:30000/busybox:1.36 -- sh -c '
    wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ | head -2 \
      || echo "★ 被拒 —— 边界在"'

# ⑤ ★★ 最关键的验收：重建命名空间后策略还在吗？（这一步才是 4.7 的意义）
helm get manifest damo-app -n damo-app | grep -c "kind: NetworkPolicy"
# 期望：5 —— 策略已经在【交付物】里，命名空间重建/灾备恢复后会随发布一起回来
# ⚠️ 若得到 4：values 里漏了 networkPolicy.egress 段（⑤ 没渲染），见 4.7.1 的踩坑记录
```

#### 4.7.4 回退

```bash
# 关掉开关（保留 Chart，只是不渲染策略）
helm upgrade damo-app <chart目录> -n damo-app \
  -f <chart目录>/values-prod.yaml --set networkPolicy.enabled=false --atomic

# 或整体回滚到上一个 revision
helm rollback damo-app -n damo-app
```

> **★★ 常见报错：`another operation (install/upgrade/rollback) is in progress`**
>
> **含义**：Helm 的 release 被**状态锁**卡住了 —— 上一次 upgrade 没干净收尾
> （Ctrl+C、SSH 断开、终端关闭，或超时后回滚也失败），留下了 `pending-upgrade` 状态。
> **和 chart 内容无关**，不要以为是 YAML 写错了。
>
> **先诊断**：
> ```bash
> helm list -n damo-app
> helm history damo-app -n damo-app --max 5
> # 看最新那行的 STATUS：pending-upgrade / pending-install / pending-rollback = 就是它
> ```
>
> **修复（从安全到激进）**：
> ```bash
> # ① 先试回滚（最安全，多数情况够用）
> helm rollback damo-app -n damo-app
> helm list -n damo-app                       # STATUS 回到 deployed 就好了
>
> # ② ① 不行 → 删掉卡住的那条 release 记录
> kubectl -n damo-app get secret -l owner=helm,name=damo-app \
>   --sort-by=.metadata.creationTimestamp \
>   -o custom-columns=NAME:.metadata.name,STATUS:.metadata.labels.status,CREATED:.metadata.creationTimestamp
> kubectl -n damo-app delete secret sh.helm.release.v1.damo-app.v<N>   # ★ 只删 pending-* 的那条
>
> # ③ 锁定解除后重新升级
> helm upgrade damo-app <chart目录> -n damo-app -f <chart目录>/values-prod.yaml --atomic --timeout 3m
> ```
> ⚠️ **只删 `pending-*` 的**；`deployed` 的是正常历史，不要动。
>
> **★★ 怎么判断锁还在不在：看【最新一条 revision】的 STATUS，别管历史上那条残留的 pending。**
> **Helm 只检查最新 revision 的状态** —— 只要最新一条是 `deployed`，锁就是开的，
> 历史里留一条 `pending-upgrade` 记录**完全不阻塞任何操作**（不用管它）。
>
> 实例（2026-09-24 本次实操的 `helm history --max 3`）：
> ```
> REVISION  UPDATED                   STATUS           DESCRIPTION
> 20        Thu Sep 24 10:24:14 2026  pending-upgrade  Preparing upgrade        ← 被打断的那次（残留）
> 21        Thu Sep 24 11:19:15 2026  superseded       Rollback to 19           ← helm rollback 解开了锁
> 22        Thu Sep 24 11:19:56 2026  deployed         Upgrade complete         ← ★ 最新=deployed ⇒ 锁已开、升级成功
> ```
> **注意最后一行才是判据**：`deployed` = 一切正常。
>
> **★ 附带的一个观察点**：升级"成功"不等于"改动生效"。判断 Pod 模板有没有真的变，
> **看 Pod 的 AGE** —— 若 Pod AGE 远大于本次 upgrade 的时间，说明**模板没变、Pod 没重建**，
> 那新加的字段（如 `serviceAccountName`、`timeoutSeconds`）就**没生效**，要回查"文件是否真的拷上去了"。
> ```bash
> kubectl -n damo-app get pod -l app.kubernetes.io/name=damo-app        # 看 AGE
> kubectl -n damo-app get pod -l app.kubernetes.io/name=damo-app \
>   -o jsonpath='{range .items[*]}{.metadata.name}{"\treadiness="}{.spec.containers[0].readinessProbe.timeoutSeconds}{"\n"}{end}'
> # 期望 readiness=1（若为 30 或空 → 新 chart 没生效）
> ```
>
> **怎么避免**：① 一直带 `--atomic`（你已经在带）；
> ② **升级时别中断它** —— 让它跑完（最长 `--timeout` 那么久）；
> ③ 长任务先 `helm upgrade --dry-run` 或 `helm template` 干跑一遍再真升级。
>
> **排查提示**：卡住时顺手看一眼集群实际状态
> （`get netpol` / `get pods` / `get svc`）——
> 有时**上次其实已经生效了**，只是状态没写回，那修完锁就结束了。

#### 4.7.5 ★★ 两个诊断命令 + 为什么「用 `--set` 打开」不算固化

**`helm get values` 默认只显示「你命令行给的值」，不含 chart 自己的 `values.yaml`。**
这个区别很容易让人误判"我配的东西丢了"：

| 命令 | 显示什么 | 用途 |
|---|---|---|
| `helm get values damo-app -n damo-app` | **只看 `--set` / `-f` 传的值**（USER-SUPPLIED） | 「我改了什么」 |
| `helm get values damo-app -n damo-app --all` | **合并 chart 默认后的完整生效值** | 「现在实际是什么」 |
| `helm get manifest damo-app -n damo-app` | 渲染出的 K8s 资源 | 「到底部署了什么」 |

> **★ 2026-09-24 实操踩坑**：`helm get values` 只看到 `image.tag` 和 `networkPolicy.enabled`，
> 一度以为 `allowedNamespaces`、`egress` 等配置"丢了" ——
> **其实它们在 chart 的 `values.yaml` 里（chart 默认值），所以不出现在 USER-SUPPLIED 区。**
> 要看全的必须加 `--all`。

**★★ 更要紧的一条：靠 `--set` 打开开关 = 没有固化**

```bash
# ❌ 这样部署，"加固开关"只存在于本次命令行里
helm upgrade damo-app <chart目录> -n damo-app \
  --set image.tag=19 --set networkPolicy.enabled=true --atomic
# → helm get values 里能看到 enabled: true，但它【不在 git 里】

# 后果：将来任何人敲一条不带这个 flag 的 helm upgrade，
#       5 条 NetworkPolicy 会【全部消失】，直接回到裸奔状态，
#       而且 — 和本节开头说的一样 — 【没人会发现，因为服务照常是通的】。
```

**✅ 正确做法：把开关写进 git**（二选一）

```yaml
# 方式 A（推荐）：在 chart 里加一份生产 values，提交进仓库
#   <chart目录>/values-prod.yaml
networkPolicy:
  enabled: true              # ★ 加固开关固化在 git 里
image:
  tag: 19

# 部署时用 -f，而不是 --set
#   helm upgrade damo-app <chart目录> -n damo-app -f <chart目录>/values-prod.yaml --atomic
```

```yaml
# 方式 B：直接把 chart 的 values.yaml 里 enabled 默认改成 true
#   代价：新人 clone 后 helm install 会直接带有默认拒绝，
#         若没同时配好 nodeCidr，可能"装完连不上"，学习曲线变陡。
#   所以更推荐方式 A —— 保留"默认能跑起来"的友好默认，用 -f 覆盖生产配置。
```

> **★ 4.7 的验收标准要加这一条**：
> `helm get values damo-app -n damo-app --all` 里能看到 `networkPolicy.enabled: true`
> —— **这才叫"固化"**。只在 `USER-SUPPLIED VALUES` 区出现，等于没固化。
>
> **面试话术**：「我把加固做成了**开关驱动 + 生产 values 文件**，而且**开关的值提交在 git 里**。
> 因为如果只靠 `helm upgrade --set` 打开，**下一次有人不带这个 flag 升级，策略就全没了** ——
> 这种"悄无声息退回不安全状态"是最难发现的故障。」

> ### ★★ 4.7 实操暴露的一个更普遍的问题：**集群与 Chart 的「漂移」**
>
> 执行 4.7 时自查发现，chart 里**同时缺两样东西**：
> ```
> $ grep -n "externalTrafficPolicy" values.yaml templates/service.yaml   → 无输出
> $ grep -n "timeoutSeconds"        values.yaml templates/deployment.yaml → 无输出
> ```
> 而集群里 `kubectl get svc` 显示 `externalTrafficPolicy: Local` —— 它**只存在于线上**。
>
> **成因**：阶段 4.3（探针 `timeoutSeconds`）与阶段 4.4（Service 的 `Local`）**只改了集群**，
> 前者没同步进 Chart，后者干脆是用 `kubectl patch` 热更新的 —— **从来没进过 Chart**。
>
> **危险在哪**：这会造成「**线上是对的、交付物是错的**」这种最难发现的漂移 ——
> 平时一切正常，**只有在重建的那一刻才暴雷**，而那时你已经不在现场了。
>
> **✅ 对策（本项目从此遵守）**：
> 1. **改完集群立刻同步改 Chart**（"固化"和"变更"成对做，不要留到最后）；
> 2. 用 `kubectl patch` 做热更新只是**过渡手段**，**当天必须补进 values**；
> 3. 每次 4.7 式的固化，跑一遍「三问」自查：
>    **这个状态 ① 在线上吗？② 在 git 吗？③ 被 Helm 管吗？** —— 三个都是"是"才算完成。
>
> **面试话术**：「我做加固时踩过一个坑：**集群改对了、但 Chart 忘了同步**。
> 结果是一份『**线上是对的、交付物是错的**』的状态 —— 平时看不出问题，
> **只有重建那一刻才暴雷**。从那以后我给自己定了规矩：**改动和固化必须成对做**，
> 并且每次都用『在线上 / 在 git / 被声明式工具管』三个问题自查一遍。」

#### 4.7.6 ★★ 别忘了改 Jenkinsfile —— 否则流水线发布反而会「删掉」加固

**这是本节最容易漏、后果最严重的一环。** 2026-09-24 核对仓库时发现：

| 仓库里 | 当时的状态 |
|---|---|
| `chart/damo-app/values.yaml` | 项目 E 原版（无 `externalTrafficPolicy` / `serviceAccount` / `networkPolicy` / `securityContext`） |
| `chart/damo-app/templates/` | 只有 `NOTES.txt` / `_helpers.tpl` / `deployment.yaml` / `service.yaml`（无 `networkpolicy` / `serviceaccount`） |
| `Jenkinsfile` Stage 4 | `helm upgrade --install damo-app chart/damo-app --set image.tag=N --atomic` —— **没有 `-f`** |

**⟹ 下一次流水线发布会发生什么（而且不会报错）：**

1. 用**未加固**的 chart 渲染；
2. Helm 三路合并会**删除**「上一次 release manifest 里有、这次没有」的资源
   → **5 条 NetworkPolicy 全被删**；
3. `externalTrafficPolicy: Local` 被重置回 `Cluster`
   → **规则① 失效、外部访问全断**（§4.4 那个问题原样复现）；
4. SA / `securityContext` 加固消失；
5. **Pod 照常起来，所以 `--atomic` 也救不了** —— 它只在"失败"时回滚，而这里"没失败"。

> ★ **这就是"加固被流水线冲掉"的真实形态**：不是报错，而是**静默退回不安全状态**。
> 而且触发它只需要一次普通的 `git push`。

**✅ 必须同时做两件事**：

**① 把加固后的 Chart 提交到仓库**（让流水线渲染出来的就是加固版）

| 本目录文件 | → 仓库路径 |
|---|---|
| `values.yaml` | `chart/damo-app/values.yaml` |
| `values-prod.yaml` | `chart/damo-app/values-prod.yaml` |
| `deployment.yaml` | `chart/damo-app/templates/deployment.yaml` |
| `service.yaml` | `chart/damo-app/templates/service.yaml` |
| `networkpolicy.yaml` | `chart/damo-app/templates/networkpolicy.yaml` |
| `serviceaccount.yaml` | `chart/damo-app/templates/serviceaccount.yaml` |

**② 给 Jenkinsfile 的 Stage 4 加上 `-f`**（否则开关默认 `false`，策略照样不渲染）

```groovy
helm upgrade --install damo-app chart/damo-app \
  --namespace damo-app \
  -f chart/damo-app/values-prod.yaml \      // ★ 新增：生产 values（加固开关固化在此）
  --set image.tag=${IMAGE_TAG} \
  --atomic --timeout 120s
```

> **为什么 `-f` 这一行是本节的"点睛之笔"**：
> 加固开关 `networkPolicy.enabled` 的**默认值是 `false`**（为了保留"新人 clone 后 install 就能跑"的友好默认）。
> 所以**光把文件提交上去还不够** —— 必须让流水线**显式引用生产 values**。
> **"默认关闭 + 生产文件打开"这个模式，只有在流水线也跟着改之后才成立。**

**③ 顺手在 Stage 5 加一条"加固是否还在"的验证**（成本几乎为零，收益很大）：

```groovy
kubectl get netpol -n ${NAMESPACE}                                              # 期望 5 条
kubectl get svc damo-app -n ${NAMESPACE} -o jsonpath='{.spec.externalTrafficPolicy}'   # 期望 Local
```

> **★ 面试话术（这轮最值钱的一段）**：
> 「加固做完后我做了一件事：**去核对仓库和流水线**。
> 结果发现仓库的 chart 还停在改造前的版本，而 **Jenkins 每次发布会用它重新渲染 ——
> Helm 会把上一次创建的 5 条 NetworkPolicy 全删掉**，而且**不会报错**，
> 因为 Pod 照常起来、`--atomic` 也不认为失败。
> **换句话说：我辛苦做的加固，会被下一次普通的 `git push` 静默抹掉。**
> 所以我把 chart 和 Jenkinsfile 一起提交了，还在发布流水线里加了『加固是否还在』的验证。
> **这让我明白一件事：加固的战场不在集群里，而在交付链路上** ——
> 只在集群里做加固，等于没做。」

> ### ★★ 一个必须先修的坑：这份 Jenkinsfile 本来就编译不过（2026-09-24 实测）
>
> **现象**：把加固后的 chart + Jenkinsfile 提交上去、触发 Jenkins 后，**构建在解析 Jenkinsfile 阶段就直接失败**：
> ```
> WorkflowScript: 130: expecting '}', found '' @ line 130, column 1.
> org.codehaus.groovy.control.MultipleCompilationErrorsException: startup failed
> ```
>
> **原因**：**`stages` 块的右花括号缺失**。
> Stage 5 的 4 个闭合括号只关掉了 `stage` / `steps` / `withCredentials` / `container`，
> 没有关 `stages` —— 于是 `post` 被解析成 `stages` 的**子块**（非法结构），
> 文件读到 EOF 时嵌套深度仍为 **1**。
>
> **★ 关键发现：这个括号在【加固改动之前的原版】里就已经缺失了** ——
> 用同一条检查跑原版，最终深度**同样是 1**。
> **也就是说这个 Jenkinsfile 此前从未成功编译过**（之前对 Stage 4/5 的"验证"
> 很可能是手动执行命令完成的，而不是真的跑通了流水线）。
>
> **定位方法（可复用）**：用配套脚本 **`check-groovy-braces.py`** 逐行追踪花括号嵌套深度
> （它会跳过 `"""` 字符串块与 `//` 注释，避免 JSONPath 的 `{...}` 之类干扰判断）：
> ```bash
> python check-groovy-braces.py <Jenkinsfile>       # 配平→退出码 0；不配平→退出码 1
> python check-groovy-braces.py chart-patch/Jenkinsfile
> ```
> 它的输出会打印最后几个结构行与各自深度，本项目实例：
> ```
> 修复前：118 深度=2 | }            ← Stage 5 闭合完，只剩 pipeline + stages
>         120 深度=3 | post {       ← ❌ post 落在 stages 里面（非法）
>         129 深度=1 | }            ← EOF 还剩 1 层没关
> 修复：在 post 之前补一个右花括号，最终深度 → 0 ✅
> ```
> **排查思路**：找**深度与预期不符**的那一行 —— 像 `post`、`stages` 这类块级关键字
> 应当出现在父块的深度上；它的深度 +1 就说明被解析成了子块。
>
> **★ 教训（很重要）**：**Jenkinsfile 是 Groovy 代码，而我们在本地没法编译它** ——
> 唯一的"编译检查"就是触发一次 Jenkins。所以**改 Jenkinsfile 时，本地至少要做一个
> 括号配平检查**（哪怕只数 `{` 和 `}` 的个数），别指望"看起来没问题"。
> **这次就是靠"改完顺手数了一遍括号"才在提交后立刻定位到的。**
>
> > 另注：修复时我在注释里也刻意避免写 `{` `}` 字符 —— 虽然 Groovy 会正确忽略 `//` 注释，
> > 但**在无法本地编译的前提下，少一个干扰变量就少一次失败构建**。

> ### 💡 动手提示：怎么把 chart-patch 的文件提交到仓库
>
> | 方式 | 可行性 |
> |---|---|
> | **本机 `git` 直接 push** | ✅ **推荐**（2026-09-24 实测可用：本机 Credential Manager 已缓存 GitHub 凭据） |
> | `chart-patch/push-to-repo.sh` 一键脚本 | ✅ 等价（走你自己的凭据） |
> | AI 侧的 GitHub **连接器**写操作 | ❌ 只读，会返回 `403 Resource not accessible by integration` |
>
> **本机 git 流程**（在能访问 GitHub 的机器上）：
> ```bash
> export GIT_TERMINAL_PROMPT=0 GCM_INTERACTIVE=never     # 防交互挂起
> git clone https://github.com/GXEL-xy/damo-app.git /tmp/damo-app && cd /tmp/damo-app
> # 复制 chart-patch 下的 7 个文件到对应位置（映射见上一张表）
> git log -1 --format='%an <%ae>'                        # 查仓库现有作者，对齐身份
> git config user.name "你的名字" && git config user.email "你的邮箱"
> git add -A
> git commit -F - <<'MSG'
> fix/feat: ...
> MSG
> git push --progress --porcelain origin main
> git ls-remote origin refs/heads/main                   # ★ 复核远端
> ```
> ⚠️ **推送若被中断/超时，先 `git ls-remote` 确认远端状态再决定是否重试** ——
> 别凭"输出为空"判定失败（本次就出现过：commit 成功、push 被 SIGTERM，重试才成）。
>
> **✅ 最终结果（2026-09-24 实测通过）**：三个 chart 提交依次推送成功
> （`71ea238` 加固固化 → `4494867` 修 Jenkinsfile 括号 → `a9c75ef` 定稿 capabilities 最小集），
> 之后 **Jenkins 第 44 次构建 `Finished: SUCCESS`**，
> 且 Stage 5 的"加固是否还在"检查项通过 —— **加固真正随流水线发布了**。
> 完整收口清单见 §4.8 末尾。

> **★ 为什么默认 `enabled: false`**：与项目 E 的 HPA 开关同一套设计模式。
> **好处**：① 改造过程可随时 `--set` 打开/关闭，不必改文件；② 团队新人 clone 仓库后 `helm install` 不会因为意外的默认拒绝导致服务起不来；③ 需要启用时一条 `--set` 即可，且有 revision 记录。
> **面试可以讲**：「我给 Chart 加的每一项安全能力都是**默认关闭 + 开关驱动**——因为安全的默认值要考虑"新人第一次部署能不能跑起来"。这不是向安全妥协，而是**让安全能力可以被逐步、可控地启用**。生产环境用 `-f prod-values.yaml` 把它打开并固化。」
>
> **★ 4.7 的面试价值（一句话）**：
> 「**加固如果只做在集群里，就不算做完**——命名空间一重建、或者换个人重新部署，策略就没了，
> 而且**没人会发现，因为服务照常是通的**（默认全通）。
> 所以我最后一步一定是**把手 apply 的策略固化进 Chart、并删掉手 apply 的那批**，
> 让集群状态和 git 里的交付物**收敛到一处真相**。」

### 4.8 阶段 4 完成标准

**A. 手 apply 阶段（4.1–4.6 做完时）**

- [ ] `kubectl -n damo-app get netpol` 显示 **5 条**：`default-deny-ingress` / `allow-damo-app-ingress` / `default-deny-egress` / `allow-egress-dns` / `allow-loadgen-egress`
- [ ] Service 的 `externalTrafficPolicy` = **Local**（★ 否则规则① 的 `ipBlock` 无效，见 §4.4）
- [ ] 外部访问正常：用「自动挑有 Pod 的节点」的写法访问 `:30090` 返回 Build 号（★ 不能再打 master）
- [ ] 两个业务 Pod 各 `READY 1/1`，`RESTARTS` 不再增长
- [ ] `kubectl get endpoints damo-app` 有真实 Pod IP（2 个）
- [ ] **白名单在放行**：`loadgen` 身份访问业务 → 返回页面内容
- [ ] **边界仍在**：跨命名空间（`default` → damo-app）被拒；**damo-app 自身访问自己也被拒**（反直觉点）
- [ ] DNS 解析正常，**外网访问被拒**
- [ ] 全部连通性测试都用 `curl --noproxy '*'` 或 `wget`（代理陷阱，见 §0.6「坑 0」）

**B. Chart 固化阶段（4.7 做完时）—— ★ 以下为 2026-09-24 实测通过的结果**

- [x] `get netpol` 只剩 `damo-app-*` 前缀的 **5 条**，**手 apply 的那批已删除**（收敛到一处真相）
      —— 实测：5 条 AGE **完全一致**（说明是同一次 `helm upgrade` 一次性创建的，即来自 Helm 渲染）
- [x] `helm get manifest damo-app -n damo-app | grep -c "kind: NetworkPolicy"` = **5**（策略在交付物里）
- [x] ★★ **`helm get values damo-app -n damo-app --all` 里能看到 `networkPolicy.enabled: true`**
      —— 开关的值在 **git 里**（用 `-f values-prod.yaml`），**不是靠 `--set` 临时给的**
      （实测输出：`enabled: false`(autoscaling) / `enabled: true`(networkPolicy) / `externalTrafficPolicy: Local`）
- [x] ★★ **`get values ... --all` 里也能看到 `service.externalTrafficPolicy: Local`**
      —— 它**不是手 patch 出来的编外状态**
- [x] ★ **`damo-app-allow-egress-loadtest`（⑤）存在** —— 旧模板里压根没有这条，
      它存在即证明**新 `networkpolicy.yaml` + values 里的 `egress` 段都被正确读取**
- [x] 功能三关：业务可用（`curl $NODE:30090` → `Build #19`）／
      loadgen 链路恢复（`<!DOCTYPE html>`）／
      SA 加固生效（两个 Pod 均 `SA=damo-app-sa automount=false`）
- [x] ★★ **流水线发布也带上加固**（2026-09-24 实测通过，**这才是"固化"的最终证据**）：
      Jenkins **第 44 次构建 `Finished: SUCCESS`** ——
      Stage 4 用 `-f chart/damo-app/values-prod.yaml` 发布成功（`✅ Helm 发布成功: .../damo-app:44`），
      Stage 5 的"加固是否还在"检查项执行通过。
      **★ 意义**：加固不再依赖任何人的手动 `--set` 或手 apply —— **一次普通 `git push` 触发的
      流水线发布，会把加固一起带上**。这正是 §4.7 存在的理由。
- [x] **`values-prod.yaml` 已提交进 GitHub 仓库**（commit `71ea238`，后续 `a9c75ef` 补充 capabilities）
      —— **固化闭环**
- [ ] `helm upgrade --set networkPolicy.enabled=false` 后策略消失、服务不受影响（验证"能一键关掉"）
- [ ] 跨命名空间访问（`default` → damo-app）被拒（§4.4 关④，未单独复测）

> ### 🎉 项目 F 收口（2026-09-24）
>
> 从「**集群里手 apply 的 5 条策略**」走到「**一次 `git push` 就能自动带上加固的流水线发布**」，
> 中间补掉了四个坑，每一个都是真实故障：
>
> | # | 坑 | 关键教训 |
> |---|---|---|
> | 1 | NodePort 源 IP 被 MASQUERADE 成 Pod 网段 → `ipBlock` 永远匹配不到 | **抓包看同一个包的源 IP 有没有被改写**（§4.4） |
> | 2 | 「跳步」导致现象消失且不留痕迹 → 误判出"failsafe 盲区"这个不存在的机制 | **分步做的价值就是"每步只引入一个变量"**（§4.5） |
> | 3 | 仓库 chart 停在项目 E 原版 → **流水线发布会静默删除全部加固** | **加固的战场在交付链路上，不在集群里**（§4.7.6） |
> | 4 | Jenkinsfile 少一个右花括号 → **构建从未成功编译过** | **本地编译不了 Groovy，就只能靠"数括号"+ 一次真构建**（§4.7.6） |
>
> **外加一个反复出现的模式（两次踩到）**：
> **「资源状态 ≠ 服务可用」** —— NetworkPolicy 静默丢包时 Pod 全健康但业务全断；
> nginx worker 全挂时 Pod 仍显示 `1/1 Running`。
> **所以验证一定要用"业务可用性"做判据。**
>
> ---
>
> ### ✅ 最终复核（2026-09-24，流水线第 44 次构建「发布成功后」独立复测）
>
> | 复核项 | 结果 |
> |---|---|
> | `get netpol` | **5 条 `damo-app-*`** ✅ |
> | `svc ... externalTrafficPolicy` | **`Local`** ✅ |
> | Pods | `damo-app-57fdff9c8f-*` **2 个 `1/1 Running`**、**`RESTARTS 0`**、AGE 31m/32m ✅ |
> | `loadgen` → 业务 Service | **`<!DOCTYPE html>`** ✅ |
>
> **★ 注意 Pod 来自【新的 ReplicaSet】`damo-app-57fdff9c8f`（不再是 `7f7d554d66`）、AGE 只有 31 分钟。**
> 这一条同时证明四件事：
> 1. **流水线真的重新部署了**（换了 RS = pod template 变了：`securityContext` + image `:44`）；
> 2. **新 Pod 一次就起来了（`RESTARTS 0`）** → capabilities 最小集**完全正确**（没有反复重启）；
> 3. **加固没被流水线冲掉**（netpol 5 条 + `Local` 都在）；
> 4. **业务可用**（loadgen 拿到页面）。
>
> **换 RS + `RESTARTS 0` + 加固仍在 + 业务可用** —— 这四条合起来，才是"加固真正固化"的完整证据。
> （只看"流水线 SUCCESS"是不够的：它只说明没报错，不说明加固还在。）
>
> **可选最后一项确认**（看 `securityContext` 是否真的写进 Pod）：
> ```bash
> kubectl -n damo-app get pod -l app.kubernetes.io/name=damo-app \
>   -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].securityContext}{"\n"}{end}'
> # 期望：{"allowPrivilegeEscalation":false,"capabilities":{"add":["CHOWN","NET_BIND_SERVICE","SETGID","SETUID"],"drop":["ALL"]},"seccompProfile":{"type":"RuntimeDefault"}}
> ```
- [ ] DNS 解析正常，**外网访问被拒**

---

## 阶段 5：追加加固（可选，至少做一项）

> 前两个阶段解决了「身份」和「网络」两个最核心的问题。这一节是**加分项**——按时间余力选 1–2 项即可。

### 5.1 选项 A：Pod 安全准入（PSA）——先 warn 再 enforce

PSA 有三档：`privileged`（无限制）→ `baseline`（挡最危险的）→ `restricted`（最严）。

```bash
# ★ 第一步只用 warn 模式：只警告不拦截，先看现有 Pod 有多少违规
kubectl label namespace damo-app \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest \
  pod-security.kubernetes.io/audit=baseline \
  --overwrite

# 滚动重启一次，观察警告
kubectl -n damo-app rollout restart deploy/damo-app
kubectl -n damo-app get events --sort-by=.lastTimestamp | grep -i "pod security" | tail -10
# 预期：看到若干 Warning，如
#   - allowPrivilegeEscalation != false
#   - runAsNonRoot != true
#   - seccompProfile 未设置
#   - capabilities 未 drop ALL
```

**看到警告后不要急着 enforce**——先想清楚哪些能修：

| 警告项 | damo-app 能否修 | 怎么修 |
|--------|----------------|--------|
| `allowPrivilegeEscalation != false` | ✅ 能 | securityContext 加 `allowPrivilegeEscalation: false` |
| `capabilities 未 drop ALL` | ✅ 能 | `capabilities: { drop: ["ALL"] }` |
| `seccompProfile 未设置` | ✅ 能 | `seccompProfile: { type: RuntimeDefault }` |
| `runAsNonRoot != true` | ⚠️ 需应用改造 | nginx 官方镜像默认以 root 启动（要绑 80 端口）→ 需换非特权镜像或改端口 |
| `readOnlyRootFilesystem` | ⚠️ 需应用改造 | 需加 emptyDir 挂载 `/var/cache/nginx`、`/var/run` 等可写目录 |

> ### ⚠️★★ 重要：**选项 A（PSA restricted）与选项 B（当前的最小 caps 方案）在当前镜像下互斥**
>
> PSA `restricted` 对 capabilities 的要求是「**必须 drop 掉全部**，
> 且 `add` 里**只允许出现 `NET_BIND_SERVICE` 一项**」。
>
> 而 5.2 的**两轮实测**（2026-09-24）证明：root-master 的 nginx 还需要
> **`CHOWN`**（chown 缓存目录给 worker）+ **`SETGID`/`SETUID`**（降权起 worker）
> —— **这三个是 restricted 不允许加回来的**。
>
> ⟹ **要拿到 `restricted`，就必须走"非 root 镜像 + 非特权端口"那条路**
> （见 5.2 末尾的进阶路线：nginx-unprivileged + 8080 + `runAsNonRoot` +
> `readOnlyRootFilesystem`，那时 `drop: ["ALL"]` 才真的够）。
>
> **这正说明加固是一条【链】，不是一张勾选清单** ——
> 选 A 还是选 B，本质上是在问"你愿意为镜像改造投多少"。
> **面试话术**：「我原本以为 PSA 和 securityContext 是两个独立选项，
> 实测后发现它们**互相约束** —— restricted 只允许加回 `NET_BIND_SERVICE`，
> 而我们的镜像还需要 `CHOWN`/`SETGID`/`SETUID`。
> 所以想要最高档，就必须改造镜像本身。**加固的深度最终受限于应用本身的可改造性。**」

> **正确的加固节奏（面试可讲）**：「我用 **warn 模式先摸底**，看到 damo-app 在 restricted 下有 4 项违规，其中 3 项（`allowPrivilegeEscalation`、`capabilities drop ALL`、`seccompProfile`）改 securityContext 就能满足，我直接补进 Chart；剩下一项 `runAsNonRoot` 需要把 nginx 从 80 端口换到 8080 并换基础镜像，属于应用改造，我评估后排到后续迭代。**先 warn 拿到事实，再决定 enforce 的边界**——直接上 enforce 会让现有工作负载全部被拒，那是事故不是加固。」

### 5.2 选项 B：容器安全上下文（改 Chart）—— ★ 建议先做这一项

**设计原则：渐进式** —— 先做"不影响现有镜像"的项，需要镜像配合的项**注释保留 + 写明原因**。

`values.yaml` 增加（项目 F 的完整版见 `chart-patch/values.yaml`）：

```yaml
# ---- 容器安全上下文（项目 F 加固，渐进式）----
securityContext:
  # ① 禁止提权：容器进程不能获得比父进程更多的权限
  allowPrivilegeEscalation: false
  # ② 丢弃所有 Linux capabilities
  #    ★★ 最小集是【实测】出来的（两轮隔离验证，2026-09-24），别照抄网上：
  #
  #    【第一轮】纯 drop: ["ALL"] → 容器起不来：
  #        nginx: [emerg] chown("/var/cache/nginx/client_temp", 101)
  #               failed (1: Operation not permitted)
  #      ★ 不是"绑 80 端口失败"（NET_BIND_SERVICE）—— 是【chown 失败】。
  #        nginx master 以 root 启动时会把缓存目录 chown 给 worker 用户（uid 101），
  #        这需要 CAP_CHOWN；被 drop 掉就 EPERM。而且它发生在【绑端口之前】，
  #        所以根本走不到 bind 那一步。
  #      ✅ 同轮对照组证明：只加 NET_BIND_SERVICE 【没用】（同样报 chown 错）
  #         → 我一开始的猜测被实测否掉了。**这就是"对照组"的价值。**
  #
  #    【第二轮实测 · 已确认】逐项加回，最小集 = 下面这 4 个：
  #      · CHOWN            → chown 缓存目录给 worker 用户（第一轮的拦路者）
  #      · NET_BIND_SERVICE → 绑定 80 端口
  #      · SETGID / SETUID  → 降权起 worker（切到 nginx 用户）
  capabilities:
    drop: ["ALL"]
    add: ["CHOWN", "NET_BIND_SERVICE", "SETGID", "SETUID"]
  #   ★ 面试话术：**最小权限不是"什么都不给"，而是"只给必需的那一个"。**
  #     而这个"必需的那一个"**必须用实测确定，不能凭网上的说法猜** ——
  #     我一开始猜的是 NET_BIND_SERVICE，实测发现真正拦路的是 CHOWN；
  #     加了 CHOWN 之后又发现 worker 降权失败，还得补 SETGID/SETUID。
  #     最后实测确认 `DAC_OVERRIDE` 【不需要】（加了与不加没区别）。
  #
  #   ⚠️★ 验证陷阱（实测踩到）：**只加 CHOWN 的那个 Pod 状态是 `1/1 Running`，
  #      但它的 worker 全挂了**（master 进程还活着，Pod 不会退出）。
  #      → **资源状态 ≠ 服务可用**；判据必须看【日志里 worker 有没有正常启动】。
  #      这与阶段 4 的 NetworkPolicy 那次是同一类陷阱。
  # ③ seccomp 配置：用运行时默认白名单过滤危险系统调用
  seccompProfile:
    type: RuntimeDefault
  # ④ 只读根文件系统（需要应用镜像支持；nginx 需额外挂载 /var/cache/nginx、/var/run）
  # readOnlyRootFilesystem: true
  # ⑤ 以非 root 运行（需要镜像支持，见 5.1 表格的评估结论）
  # runAsNonRoot: true
  # runAsUser: 101        # nginx 用户 uid
```

模板片段（追加到 `containers:` 里，与 `image:` 同级）：

```yaml
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
```

**验证（改完必须确认 Pod 真的起来了 —— 这是"高风险项"和"无害项"的分水岭）**：

```bash
helm upgrade damo-app <chart目录> -n damo-app \
  -f <chart目录>/values-prod.yaml --atomic --timeout 3m

kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app        # 期望 2 个 1/1 Running
kubectl -n damo-app get pod -l app.kubernetes.io/name=damo-app \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].securityContext}{"\n"}{end}'
# 期望：{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"seccompProfile":{"type":"RuntimeDefault"}}

# ★ 若 CrashLoopBackOff / 起不来 → 先看是不是 EACCES（绑端口失败）
kubectl -n damo-app logs <pod> --tail=20
kubectl -n damo-app describe pod <pod> | tail -20
```

> **★ 为什么把它排在第一个做**：
> ① 风险最低（前三项不影响现有镜像）；
> ② 效果**立刻可见**（`get pod -o jsonpath` 直接看到）；
> ③ 天然带一个"**实测驱动**"的叙事 —— 见上面 `NET_BIND_SERVICE` 那段：
> **先做最"素"的版本，用实测发现到底需不需要那一个 capability。**
>
> **取舍（面试可讲）**：我把前三条**打开**，后两条**注释保留**并在文档里写明原因（需要镜像配合）。
> **"我改了能改的，并且明确记下改不了的为什么改不了"**——这比"全都改了但服务起不来"或"都没改"都好。

### 5.3 选项 C：资源边界（ResourceQuota + LimitRange）

```yaml
# manifests/30-quota-limitrange.yaml
# ============================================================
# 项目 F · 资源边界（可选加固项）
# 目的：防"一个 Pod 申请离谱资源打满节点"，也防"没写 requests 导致调度失衡"
# ⚠️ 注意：一旦创建 ResourceQuota，命名空间内【新建】的 Pod 必须显式声明
#    requests/limits，否则会被拒绝 —— 这是"有破坏性的加固"，
#    必须先确认命名空间内所有工作负载都已声明资源（或靠 LimitRange 兜底）。
# ============================================================
apiVersion: v1
kind: ResourceQuota
metadata:
  name: damo-app-quota
  namespace: damo-app
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: damo-app-limitrange
  namespace: damo-app
spec:
  limits:
    # 默认值：Pod 未声明 resources 时自动补上（防裸 Pod）
    - type: Container
      default:
        cpu: 200m
        memory: 128Mi
      defaultRequest:
        cpu: 50m
        memory: 64Mi
      # 单容器上限：防单个 Pod 申请离谱资源
      max:
        cpu: "2"
        memory: 2Gi
```

```bash
# 应用前的检查：命名空间里现有工作负载是否都声明了 resources？
kubectl -n damo-app get deploy -o json | grep -c '"resources"'
# 若数量少于 Deployment 数，先补全再建 Quota（或先建 LimitRange 兜底）
```

---

## 阶段 6：综合验证与前后对比演示

### 6.1 一次跑完的「安全验收脚本」

把整个改造的验收压缩成一段可反复运行的脚本，**这是你交付给团队的资产**。

`manifests/verify-hardening.sh`：

```bash
#!/usr/bin/env bash
# ============================================================
# 项目 F · 安全加固验收脚本
# 用法：bash manifests/verify-hardening.sh
# 说明：只读检查，不修改集群任何状态，可反复运行
# ============================================================
set -uo pipefail
NS=damo-app
DEPLOYER=system:serviceaccount:damo-app:damo-deployer-sa
POD=$(kubectl -n "$NS" get pod -l app.kubernetes.io/name=damo-app -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)

pass=0; fail=0
chk() { # chk "描述" "期望" "实际"
  if [ "$2" = "$3" ]; then echo "  ✅ $1"; pass=$((pass+1));
  else echo "  ❌ $1  (期望=$2 实际=$3)"; fail=$((fail+1)); fi
}

echo "========== 一、RBAC =========="
echo "[1] 容器内不应存在 token（最小权限的极致）"
r=$(kubectl -n "$NS" exec "$POD" -- sh -c \
      'test -f /var/run/secrets/kubernetes.io/serviceaccount/token && echo EXIST || echo ABSENT' 2>/dev/null)
chk "容器内 token 已移除" "ABSENT" "$r"

echo "[2] 发布者 SA 权限边界"
chk "可更新 damo-app 的 Deployment" "yes" "$(kubectl auth can-i update deployments -n $NS --as=$DEPLOYER)"
chk "不可操作 monitoring 命名空间"  "no"  "$(kubectl auth can-i get deployments -n monitoring --as=$DEPLOYER)"
chk "不可读取全集群 Secret"        "no"  "$(kubectl auth can-i get secrets -A --as=$DEPLOYER)"
chk "不可删除节点"                 "no"  "$(kubectl auth can-i delete nodes --as=$DEPLOYER)"
chk "不可创建命名空间"             "no"  "$(kubectl auth can-i create namespaces --as=$DEPLOYER)"

echo "========== 二、NetworkPolicy =========="
echo "[3] 策略存在性"
n=$(kubectl -n "$NS" get netpol --no-headers 2>/dev/null | wc -l | tr -d ' ')
chk "至少 3 条策略" "3" "$([ "${n:-0}" -ge 3 ] && echo 3 || echo "${n:-0}")"

echo "[4] 业务可用性（放行是否够用）"
code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 http://11.0.1.128:30090/)
chk "外部访问返回 200" "200" "$code"
want=$(kubectl -n "$NS" get deploy damo-app -o jsonpath='{.spec.replicas}')
ready=$(kubectl -n "$NS" get deploy damo-app -o jsonpath='{.status.readyReplicas}')
chk "副本全部 Ready" "$want" "$ready"
ep=$(kubectl -n "$NS" get endpoints damo-app -o jsonpath='{.subsets[0].addresses[*].ip}' | wc -w | tr -d ' ')
chk "Endpoints 有条目" "yes" "$([ "${ep:-0}" -ge 1 ] && echo yes || echo no)"

echo "[5] 网络边界（是否真的拒了不该通的）"
r=$(kubectl -n default run "netchk-$RANDOM" --rm -i --restart=Never --quiet \
      --image=11.0.1.128:30000/busybox:1.36 -- \
      sh -c 'wget -qO- --timeout=3 http://damo-app.damo-app.svc.cluster.local/ >/dev/null 2>&1 && echo OPEN || echo BLOCKED' 2>/dev/null | tail -1)
chk "跨命名空间被拒" "BLOCKED" "$r"
r=$(kubectl -n "$NS" exec "$POD" -- sh -c \
      'wget -qO- --timeout=3 http://www.baidu.com >/dev/null 2>&1 && echo OPEN || echo BLOCKED' 2>/dev/null)
chk "外网出方向被拒（防数据外传）" "BLOCKED" "$r"

echo
echo "========== 结果：$pass 项通过，$fail 项失败 =========="
[ "$fail" -eq 0 ] && echo "🎉 安全加固验收全部通过" || echo "⚠️  有 $fail 项未通过，见上方 ❌"
```

```bash
chmod +x manifests/verify-hardening.sh
bash manifests/verify-hardening.sh
```

### 6.2 整理成 before / after 对照表（README 直接可用）

| 维度 | 改造前（裸奔） | 改造后（最小权限） | 验证命令 |
|------|--------------|------------------|---------|
| **NetworkPolicy 数量** | 0 条 | 4 条（2 条 default-deny + 2 条白名单） | `kubectl -n damo-app get netpol` |
| **容器内 API token** | 存在（可读取） | **不存在** | `kubectl exec <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount/` |
| **跨命名空间访问** | ✅ 通（可从 damo-app 摸到 Prometheus / Loki / Jenkins） | ❌ 拒 | `kubectl -n default run ... wget damo-app...` |
| **外网出方向** | ✅ 通（可外传数据） | ❌ 拒（只放行 DNS） | `kubectl exec <pod> -- wget http://www.baidu.com` |
| **发布身份** | admin kubeconfig（≈ 集群 root） | `damo-deployer-sa`（仅 damo-app 命名空间） | `kubectl auth can-i --list --as=...` |
| **发布权限范围** | 全集群 `*.*` | 9 类资源的有限 verbs + `namespaces` 只读 | `kubectl auth can-i --list -n damo-app --as=...` |
| **kubelet 探针** | 正常（全通，探针靠"收到 RST"通过） | 正常 —— 本集群探针走**本机路由、不进 Calico 策略链**，默认拒绝不影响它（§4.3.2）。<br>⚠️ 别归因给 `timeoutSeconds`：它的 K8s 默认值本来就是 1 秒（§4.3.3）。放行节点网段 + `Local` 是为 **NodePort 外部访问**，不是为探针 | `kubectl get pods` → 2 个 `1/1` |
| **业务可用性** | 正常 | **仍然正常** ★ | `curl http://11.0.1.128:30090/` |

> **★ 最后两行是这张表的灵魂**：加固**没有**牺牲业务可用性。面试时主动强调这一点，因为面试官心里的疑问就是「你搞了这么多安全措施，服务还跑得动吗」。

---

## 7. 故障速查表

按「现象 → 原因 → 排查与解决」组织。**★ 标记的是面试高频考点**。

| 现象 | 原因 | 排查与解决 |
|------|------|-----------|
| **★ Jenkins 构建一开始就失败：`expecting '}', found '' @ line N` / `MultipleCompilationErrorsException`**（★ 2026-09-24 实测踩到） | **Jenkinsfile 的花括号不配平**（Groovy 代码在本地无法编译，只有 Jenkins 会检查）。本项目实测：`stages` 块的右花括号缺失 → `post` 被解析成 `stages` 的子块 → EOF 时嵌套深度仍为 1。**★ 该缺陷在加固改动之前的原版里就已存在** —— 说明这份 Jenkinsfile 此前从未成功编译过 | **逐行追踪花括号嵌套深度**（脚本跳过 `"""` 字符串块与 `//` 注释），看最终深度是否为 0、以及哪一行深度不合预期。本项目实例：`post` 那行深度是 **3**（应为 2）→ 说明它落在 `stages` 里 → 在 `post` 之前补一个右花括号即可。**★ 铁律：改 Jenkinsfile 后本地至少要数一遍 `{` 与 `}` 的个数** —— 别指望"看起来没问题"，因为**你唯一的真实编译检查就是下一次触发 Jenkins** |
| **★★ 集群内 `curl` 全部超时，`ping` 却通；删掉 Policy 后仍然不通**（★ 2026-09-23 实战踩坑，**最阴的一个坑**） | **`curl` 读了 `http_proxy`，把集群内地址的请求错误地发给了外部代理**（本集群宿主机/节点配了 `11.0.1.1:7890` 或 `192.168.5.1:7890`）。`ping` 通是因为 **ICMP 不读代理变量**——这正是"ping 通 curl 不通"这个矛盾组合的成因。**更坑的是：删了 Policy 也不通，会让人误判成"跟 NetworkPolicy 无关、网络本身坏了"** | ① **先看真相**：`curl -v <url> 2>&1 \| head -5`，若出现 `Uses proxy env variable http_proxy == '...'` 就是它；② **判定一律加 `--noproxy '*'`**（比 `export` 更可靠）：`curl --noproxy '*' -s -o /dev/null -w '%{http_code}\n' http://11.0.1.128:30090/` → 期望 `200`；③ **注意大小写**：curl 只读【小写】`http_proxy`/`no_proxy`，大写 `NO_PROXY` 写得再对也没用。本集群实测 shell 覆盖层把 `no_proxy` 改成 `localhost,127.0.0.1,::1,192.168.5.0/24`，**集群网段全丢** → 所有集群内 curl 假超时；④ 根治：在 `~/.bashrc` 末尾（代理客户端启动之后）补 `export no_proxy="...,11.0.1.0/24,10.96.0.0/12,10.244.0.0/16"` 且 `export NO_PROXY="$no_proxy"`；⑤ **替代方案：用 `wget`**（不读代理变量）。详见 §0.6「坑 0」 |
| **★ 加了 NetworkPolicy 完全不生效（流量照样通）** | ① **CNI 不支持**（Flannel 默认不支持）；② 策略的 `podSelector` 没选中目标 Pod；③ 写成了 `ingress: [{}]`（= 允许所有）；④ **踩到了 failsafe 端口**（Calico 无条件放行的一组宿主机端口） | ① 本项目 Calico 由 tigera-operator 部署 → **查 `kubectl -n calico-system get pods` + `kubectl get tigerastatus`**（不是 `kube-system`！查错命名空间会误判成"CNI 没装"）；② `kubectl -n damo-app get pod --show-labels` 与策略的 `matchLabels` 逐字比对；③ `kubectl get netpol -o yaml` 回看有没有多余的空对象；④ `kubectl get felixconfiguration default -o yaml \| grep -A8 failsafe` 换一个不在列表里的端口重测 |
| **★★ 白名单明明写了，但从"节点上" curl 集群地址/NodePort 全部超时；而 Pod 内部访问却正常**（★ 2026-09-23 实测，**本项目最容易自己坑自己的一条**） | **源身份不匹配**：白名单是按 **Pod 标签**（`podSelector`）写的，而从节点上 curl 的**源身份是 host（节点）**，它不在任何 `podSelector` 的覆盖范围内 → 被默认拒绝静默丢包。⚠️ 注意：**"在节点上自测失败"完全不能说明策略写错了** —— 这是测法盲区，不是配置错误 | ① **验证白名单必须用"白名单里明确写了身份"去测**：本项目规则② 写的是 `app=loadgen`，就用 `kubectl run ... --labels="app=loadgen"` 的 Pod 去访问（实测返回 `<!DOCTYPE html>` ✅）；② **对照组**：用不在白名单里的身份（如 `damo-app` 自身标签）测同一地址，应得 `timed out` —— **一正一反才算证明策略精确**；③ 外部访问要用**真实外部客户端**（Windows 宿主机 `curl.exe --noproxy "*"`）验证，不要在节点上自测；④ 若确实需要放行节点 host 流量，那是 **Calico `HostEndpoint`** 的范畴（本案不需要，也建议不引入） |
| **★ 默认拒绝后 Pod 全部 NotReady、反复重启** | **探针流量被策略拦住**——**仅当你的 CNI/探针类型让探针流量需经 CNI 策略判定时**才会发生（**本集群 Calico IPIP 实测不会**，见下一条） | 放行节点网段：`ipBlock: { cidr: 11.0.1.0/24 }` + 应用端口 |
| **★ 默认拒绝已生效，但 Pod 仍 `READY 1/1`、`RESTARTS` 不涨、Endpoints 仍有 IP**（"探针连环挂"没出现；**实测于 2026-09-23 本集群**） | **不是故障，是本集群的正常行为**：`tcpSocket` 探针目标是**本机 Pod**，流量经本机路由直达容器 veth，**不进入 Calico 的 `cali-*` 策略链** → `podSelector: {}` 管不到它 | **不需要修**。改用「连接语义」判定策略是否生效：集群内 `nc -vz ... 80` 应报 **`timed out`**（而非 `refused`）。若确实看到 READY 掉，才按上一条放行节点网段。详见 §4.3.2 |
| **★ 分不清"服务挂了"还是"流量被拦"**（`nc`/`wget` 连不上） | 两种情况的表征不同：**策略拦截 = 静默丢包（超时）**；**端口没人听 = RST（connection refused）**。⚠️ **但超时还有第三个来源：代理（见本表第一条）** | **先排代理（`--noproxy '*'`），再用"放在别处的对照组"一刀切开**：① 本命名空间内 `nc -vz -w 3 damo-app.damo-app.svc.cluster.local 80` → 预期 `timed out`；② 从 `default` 命名空间 `nc -vz` 同一 **Pod IP:80** → 预期 `refused`。**"内超时 / 外 refused"= 策略正在静默丢包（确实生效）**；两边都 `refused` = 策略没生效；两边都超时 = 先查代理，再看 Pod/端口 |
| **★★ 外部 NodePort 访问不通，集群内 Pod 之间却正常**（★ 2026-09-23 抓包定位，**本文档此前写错过，已更正**） | **NodePort 的源 IP 被 MASQUERADE 改写成了节点隧道地址**。实测：Windows 客户端的 SYN 源是 `11.0.1.1`，出 `tunl0` 隧道时被改写成 **`10.244.219.64`（Pod 网段！）** → `ipBlock: 11.0.1.0/24` 匹配不到 → 静默丢弃。**「NodePort SNAT → 节点 IP」的说法在本集群不成立** | 抓包定位：`sudo tcpdump -i any -nn "tcp port 30090 or (host <PodIP> and tcp port 80)"` 在**节点**上抓，看**同一个包的源 IP 有没有变**。修正二选一：**① `externalTrafficPolicy: Local`**（不再 SNAT、源 IP 保留、规则① 才真正有意义；代价：只落到接收节点上的 Pod）；**② 放宽 CIDR 到 `10.244.0.0/16`**（省事但粒度粗）。★ 面试要点：**NodePort 会把客户端身份抹掉 → 这是生产更推荐 Ingress/LoadBalancer 的原因之一** |
| 外部 NodePort 访问不通，但集群内正常（旧结论，仅作对照保留） | 曾以为 NodePort 流量经 kube-proxy **SNAT 后源 IP 变成节点 IP** —— **实测推翻了**，见上一条 | 仅当源 IP 确实保留为客户端 IP（如 `externalTrafficPolicy: Local`）时，才按**客户端所在网段**放行 |
| 默认拒绝 egress 后，Service 域名解析失败 | DNS 出方向被拒 | 放行到 `kube-system/kube-dns` 的 UDP/TCP 53（见 4.6） |
| 加了 egress deny，担心"响应发不回去" | **NetworkPolicy 是有状态的**，入方向已允许连接的响应自动放行 | 不需要额外放行。实测验证：外部 `curl` 仍正常 |
| Service 访问不通，`get endpoints` 是 `<none>` | Pod 不 ready（探针被拒，或应用自身起不来） | 先 `kubectl get pods` 看 READY 列 → 再看 `describe pod` 的探针事件 → 回查放行规则 |
| 策略生效了，但某个"确实该通"的来源不通 | 放行规则粒度不够，或 `podSelector` / `namespaceSelector` 的 AND-OR 语义写错 | **同一个 `from` 元素内是 AND，不同元素之间是 OR**（见第 8 章 Q11）；跨命名空间访问需要显式写 `namespaceSelector` |
| 从 Pod 里 `wget` 报 `bad address` | DNS 解析失败（egress 未放行 53 端口） | 见上；也可先 `nslookup` 确认是 DNS 问题而非网络问题 |
| `kubectl exec` 时看到 token 文件仍存在 | 只改了 values.yaml 没 `helm upgrade`，或 Pod 没有重建 | `helm get manifest` 确认渲染产物含 `automountServiceAccountToken: false`；然后 `kubectl rollout restart deploy/damo-app` |
| 改了 `automountServiceAccountToken: false` 后应用报错 | 应用**确实需要**访问 K8s API（如 Operator、部分 controller） | 说明该应用需要 token——应改成「挂载 token + 最小 Role」，而不是简单关闭。判断依据：看应用日志有没有 `forbidden` / `cannot list` 之类的 API 报错 |
| Jenkins 发布报 `forbidden: User "system:serviceaccount:damo-app:damo-deployer-sa" cannot ...` | 最小 Role 缺了某个 verb 或资源 | 从报错里的 `cannot <verb> <resource>` 精确反推，补进 Role；**不要图省事换成 cluster-admin** |
| Jenkins 发布报 `namespaces is forbidden: cannot create` | 用了 `--create-namespace`，但新 SA 只有 `namespaces` 的 `get` | 去掉 `--create-namespace`——**这正是"少给一个动词"的设计意图** |
| **★ 切换最小权限后，第二天流水线 Stage 4 突然 401/403** | `kubectl create token --duration=24h` 签的 token **已过期** | 重签 token 并更新 Jenkins 凭据；或改用长期 `kubernetes.io/service-account-token` Secret（§3.5 警告框有命令） |
| **★ "明明收权了，旧权限还在"** | Agent Pod 里**仍挂着项目 B 的 `Secret/kubeconfig`**（admin 客户端证书，本集群身份 `kubernetes-admin` @ `kubeadm:cluster-admins`）。`withCredentials` 只是让 helm **改用新凭据**，**并没有移除旧凭据** —— `cat /root/.kube/config` 照样能拿到 admin 身份 | 删 PodTemplate 的 Volume + 删 `Secret/kubeconfig`（§3.5 步骤 6.2）。**⚠️ 不要去删 `clusterrolebinding kubeadm:cluster-admins`** —— master 上你自己用的就是同一份证书，删了会自锁 |
| **★ Stage 5 里 `kubectl auth whoami` 仍显示 `kubernetes-admin`（而不是 `damo-deployer-sa`）** | ① **最常见：Replay 时没有整份替换脚本** —— Replay 默认载入的是**那次构建的旧 Jenkinsfile**，"直接点 Run"跑的还是旧逻辑；② 或 `KUBECONFIG` 环境变量没有透传进 `container('helm')`，kubectl/helm 退回去读挂载的 `/root/.kube/config` | ① 确认 Replay 编辑框里的内容**确实含 `withCredentials`**；② 给命令**显式加 `--kubeconfig "$KUBECONFIG"`**（`helm --kubeconfig ...` / `kubectl --kubeconfig ...`），不依赖环境变量透传；③ 确认 Jenkins 凭据 ID 拼写与 `def DEPLOYER_CRED_ID` 一致 |
| `helm` 报 `secrets is forbidden` | Role 里漏了 `secrets` 权限（Helm release 台账存在 Secret 里） | 补 `secrets` 的 `get/list/watch/create/update/patch/delete` |
| **★ `kubectl apply -f - <<'EOF'` 报 `resource name may not be empty`（报错里 `Name: ""`）** | **heredoc 里的 YAML 缩进被吞了** —— `metadata:` 下的 `name:` 掉到第 0 列 → `metadata` 成了空对象 → 解析不出资源名 | 先 `cat` 回看缩进（`metadata:` 下面必须缩进 2 格）；**更根本的解法：写成 `.yaml` 文件再 `kubectl apply -f xxx.yaml`**，本项目统一这么干。⚠️ 见到 `Name: ""` 别去怀疑 RBAC/权限，就是 YAML 结构问题（§3.5 有完整踩坑记录） |
| 回收旧权限后流水线立刻挂了 | 切换节奏错了（先删旧的，新的还没验证） | **正确节奏**：新增 → 验证 → 切换 → 观察一个完整发布周期 → 回收（见 3.5）。应急：临时恢复旧绑定，重新按节奏走 |
| PSA 设了 `enforce=restricted` 后 Pod 创建被拒 | restricted 要求非 root、只读根文件系统等，现有镜像不满足 | 改回 `warn` 模式，先修镜像与 securityContext，再考虑 enforce |
| 建了 ResourceQuota 后新 Pod 创建失败 | Quota 要求 Pod 显式声明 resources，而工作负载没声明 | 先给所有工作负载补 `requests/limits`（或靠 LimitRange 提供默认值），再建 Quota |
| `kubectl auth can-i` 结果与预期不符 | 忘了 `--as` 的 SA 要带命名空间前缀，或查的不是目标命名空间 | 用完整格式 `system:serviceaccount:damo-app:damo-deployer-sa`，并显式加 `-n <ns>` |
| **★ `kubectl get rolebinding,clusterrolebinding -o json \| grep 'system:serviceaccount:...'` 查不到任何东西** | **写法错**：ServiceAccount 主体在 RBAC 对象里是**结构化字段**（`kind`/`name`/`namespace`），`system:serviceaccount:<ns>:<name>` 这个字符串**只存在于 `--as` 参数和报错信息里**，对象 JSON 里没有 | 改用 jsonpath 按字段查（§2.5 ②），或直接 `kubectl describe clusterrolebinding <名字>`。**⚠️ 注意：这个错法的输出也恰好是空，很容易把"有绑定"误判成"没有绑定"（假阴性）** |
| **★ `kubectl run netcheck --rm -it ...` 报 `error: timed out waiting for the condition`** | **Pod 在超时内没进入 Running**（**不是网络不通**）；而 `--rm` 把失败的 Pod 立刻删掉，现场一并消失，所以只看到这一句 | **去掉 `--rm`** 重跑，然后 `kubectl -n damo-app describe pod netcheck \| tail -20` 看 Events。**本项目最常见的是 `ImagePullBackOff`：`busybox:1.36` 未中转进私有仓库**（§0.5 有中转命令；也可改用 §2.2 方案 A 直接 exec 进 damo-app 容器） |
| 临时 netcheck Pod 一直卡在 `Terminating` | `--rm -it` 的 Pod 需要退出 shell 才会自删 | 另开终端 `kubectl -n <ns> delete pod netcheck --force --grace-period=0`（仅用于测试 Pod） |
| 清理反面教材时漏了 ClusterRoleBinding | 只删了 SA，没删绑定 | `kubectl get clusterrolebinding \| grep over-privileged`，删除后确认；**这条是本项目最容易留下的隐患** |

---

## 8. 自测题（含答案）

**Q1. K8s 安全三层是什么？各回答什么问题？**
A：① **认证（Authentication）**——你是谁（证书 / Token / SA）；② **授权（Authorization）**——你能对什么资源做什么操作（RBAC）；③ **准入控制（Admission）**——你的请求合规吗（Pod 安全准入、ResourceQuota、LimitRange）。三层串联，任一层拒绝即失败。

**Q2. ServiceAccount 和 User 的关键区别是什么？**
A：① User 在 K8s 里**没有对应对象**（认证后只知道名字），SA 是**真实的 K8s 对象**；② SA **属于某个命名空间**，User 是全局的；③ 主体写法上 SA 必须带 `namespace`；④ User 通常由外部认证源管理，SA 由 `kubectl create sa` 创建。

**Q3. K8s 1.24 前后，ServiceAccount token 的机制有什么变化？**
A：1.24 之前，创建 SA 会自动生成**永不过期**的 Secret 并挂载进 Pod——长期凭证，泄漏即永久失守。1.24 起不再自动创建 Secret；Pod 挂载的是 kubelet 通过 **TokenRequest API** 申请的**有有效期（默认 1 小时）、绑定到具体 Pod、Pod 删除即失效**的投影卷。趋势是「**长期静态凭证 → 短期绑定凭证**」。

**Q4. "default SA 没有任何权限"——那它危险在哪？**
A：危险不在于"现在能做什么"，而在于**它是一个有效的集群身份（有一个可用的 token）**。三层含义：① 一旦集群里**任何**角色被错误绑定到 `system:serviceaccounts` 组或 `default` SA，所有 Pod 立刻共享该权限；② 攻击者拿到 token 后可做**信息收集**（探测 API 资源、集群版本、命名空间是否存在）；③ 结合其他疏漏可**横向移动**。所以最彻底的加固不是"少给权限"，而是**根本不给身份**。

**Q5. RBAC 四要素是什么？Role 和 ClusterRole 的区别？**
A：Role / ClusterRole（定义"能干什么"）、RoleBinding / ClusterRoleBinding（定义"谁绑到这个角色"）。区别在**作用域**：Role 属于命名空间，只能在同命名空间内被 RoleBinding 引用；ClusterRole 是集群级，可被 RoleBinding（限定单命名空间生效）或 ClusterRoleBinding（全局生效）引用。ClusterRole 还用于**非命名空间资源**（`nodes`、`namespaces`、`persistentvolumes`）。

**Q6. RoleBinding 绑定 ClusterRole 有什么用途？**
A：**把集群级角色的定义复用到单个命名空间**。例如 `view` 是个 ClusterRole，用 RoleBinding 绑定后，主体只能在指定命名空间内拥有 view 权限。用途：**复用角色定义 + 收敛作用域**。注意 `ClusterRoleBinding` 不能引用 `Role`（不合法）。

**Q7. 为什么说"不写 NetworkPolicy"不等于"安全"？**
A：K8s 默认**全通**——没有任何 NetworkPolicy 时，命名空间内所有 Pod 双向互通，且 Pod 可访问任意外部地址。所以「不写策略」= 「完全不做网络限制」。做网络加固的第一步是 `kubectl get networkpolicy -A` **确认现状**，看到 `No resources found` 才知道自己在裸奔。

**Q8. NetworkPolicy 默认拒绝的 YAML 怎么写？有哪些坑？**
A：
```yaml
spec:
  podSelector: {}              # 选中本命名空间全部 Pod
  policyTypes: [Ingress, Egress]
  # 关键：不写 ingress/egress 段 = 没有允许规则 = 全拒
```
三个坑：① `ingress: [{}]`（含一个空规则对象）= **允许所有来源**，一个括号之差从全拒变全通；② 漏写 `policyTypes: Egress` 会导致出方向不受管控；③ apply 后**必须回看** `-o yaml` 确认结构。

**Q9. 为什么有些说法讲「默认拒绝后第一个挂的是 kubelet 探针」？这对吗？**
A：**要分 CNI 实现看，不能当通用结论 —— 这是我在本集群实测推翻的一条流行说法。**
- **流行说法的逻辑**：探针流量来自节点上的 kubelet，源 IP 是节点 IP，没有 Pod 身份，所以基于 `podSelector`/`namespaceSelector` 的放行覆盖不到它 → 探针被拒 → Pod 不 Ready → Endpoints 为空 → 502。
- **本集群实测（Calico IPIP + tigera-operator，2026-09-23）**：这个链条**不成立**。策略生效期间 Pod 依然 `READY 1/1`、`RESTARTS` 为 0、`get endpoints` 仍有真实 Pod IP。
- **原因**：`tcpSocket` 探针的目标是**本机 Pod 的 IP**，kubelet 从宿主机发起后流量经本机路由直达容器 veth —— **不跨节点、不经 IPIP 封装、不进入 Calico 用于策略执行的 `cali-*` iptables 链**。没进判定链，`podSelector: {}` 自然管不到它。
- **所以"放行节点网段"这条规则的正确归因是**：为了 **NodePort 经 kube-proxy SNAT 后的源 IP**（它才是真正没有 Pod 身份、必须用 `ipBlock` 覆盖的流量），而不是为了探针。
- **正确的生效判据**：不是"Pod 变 NotReady"，而是**连接语义从 `refused` 变成 `timed out`**（静默丢包）。用 `nc -vz -w 3 damo-app.damo-app.svc.cluster.local 80` 观察。
- **如果你的环境确实出现了探针挂**：说明你的 CNI/探针类型让探针流量需要经策略判定（例如换成 `httpGet` 打 Pod IP、或走 Service hairpin 的路径）→ 这时才需要放行节点网段。
- **★ 这题的标准答法（体现深度）**：「这个说法取决于 CNI 实现。我在 Calico IPIP 下实测探针不受影响——探针走本机路径、不进策略链；所以我把节点网段的放行归因于 NodePort 的 SNAT，而不是探针。」

**Q10. NodePort 的流量在 NetworkPolicy 眼里源 IP 是什么？**
A：默认 `externalTrafficPolicy: Cluster` 时，请求经 kube-proxy DNAT 到 Pod，**源 IP 被 SNAT 成接收请求的那个节点的 IP**。所以放行节点网段就能覆盖 NodePort 外部访问。若改成 `Local`，源 IP 保留为客户端真实 IP（好处：可做客户端 IP 审计，代价：流量只转发到本节点的 Pod）。

**Q11. `from` 里 `podSelector` 和 `namespaceSelector` 写在一起和分开写有什么区别？**
A：最阴的语法点。
```yaml
# 写法 A：同一个 from 元素里 → AND（既在该命名空间，又带该标签）
- from:
    - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
      podSelector: { matchLabels: { k8s-app: kube-dns } }

# 写法 B：两个 from 元素 → OR（来自该命名空间的任意 Pod，或本命名空间带该标签的 Pod）
- from:
    - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
    - podSelector: { matchLabels: { k8s-app: kube-dns } }
```
**同一个数组元素内是 AND，不同元素之间是 OR。** 写错会导致策略范围过大（漏放）或过小（服务挂）。

**Q12. NetworkPolicy 是"有状态"的，这意味着什么？**
A：**入方向被允许的连接，其响应包会自动放行**，不需要在 egress 里额外写规则。所以「加了 egress 默认拒绝」不会导致"外部来访问、响应发不回去"。真正被 egress 挡住的是**主动发起的出方向连接**：DNS 查询、调用外部 API、把数据外传（数据渗漏）。

**Q13. 为什么把 NetworkPolicy 写进 Helm Chart，而不是直接 `kubectl apply`？**
A：因为**手 apply 的策略不在交付物里**。一旦命名空间被重建（灾备演练、环境重建、误删），策略就丢了，而且**没人会发现——因为服务还是通的**（默认全通）。写进 Chart 后：① 策略随发布一起生效；② 有 revision 可审计；③ 用 `values` 开关控制，可 `-f prod-values.yaml` 在生产打开。**"政策即代码"不只是把 YAML 存进 git，而是让它能随交付链路自动收敛。**

**Q14. 收紧权限时，正确的操作节奏是什么？**
A：**五步**：① **新增**最小权限身份；② **验证**它能完成全部必要动作；③ **切换**调用方；④ **观察**至少一个完整发布周期；⑤ **回收**旧的高权限配置。**不能先删再建**——权限收紧是不可逆的破坏性变更，一旦删掉旧绑定流水线立刻中断。

**Q15. 为什么加固要用"渐进式"（先 warn 后 enforce）？**
A：因为安全策略的**执行带破坏性**。直接 `pod-security.kubernetes.io/enforce=restricted` 会让所有不满足 restricted 的现有 Pod 无法创建（或无法滚动更新），那是**事故而不是加固**。**先 warn 拿到事实**（几项违规、分别是什么），评估哪些能改、哪些需要应用改造、排期如何，再决定 enforce 的边界。**"先度量、后收紧"** 是安全工程的基本原则。

**Q16. 这个项目为什么"改造不改变业务行为"是硬要求？**
A：因为运维的底线是**不因改造引入故障**。本项目验收标准明确包含「外部 `curl :30090` 返回 200」「副本 2/2 Ready」「最小权限流水线仍能成功发布」三条——**加固的成果不能以牺牲可用性为代价**。如果收权之后服务挂了，那不是"更安全"，而是"更不可用"。面试官最关心的也是这一点：你能不能**在收紧的同时保住业务**。

---

## 9. 面试要点与简历写法

### 10.1 一段话讲完本项目

> "我给集群做了一次安全加固，分两条主线。
> 网络侧：先盘点发现全集群**零条 NetworkPolicy**，实测确认 `damo-app` 这个对外暴露的应用能直连 Prometheus、Loki 甚至 Jenkins——典型的横向移动路径。我按「**先只 deny Ingress，验证后再加 deny Egress**」的分步做法收敛，踩了第一个坑：默认拒绝后**第一个挂的不是外部流量而是 kubelet 探针**，因为探针流量来自节点、没有 Pod 身份，标签类规则覆盖不到它，连锁反应是 Pod 不 Ready → Endpoints 为空 → 502 → 反复重启。放行节点网段后恢复。加 egress 时又验证了一个反直觉的点：**NetworkPolicy 是有状态的，所以 egress 全拒不会打断入方向的响应**，真正被挡的是 DNS——loadgen 立刻报 `bad address`，放行 `kube-system/kube-dns` 的 53 端口后恢复，但外网访问仍然被拒，数据外传通道封住了。最终形成四条放行规则。
> 身份侧：盘点了 SA 现状，发现应用 Pod 用的是 `default` SA 且容器里挂着 API token。我的判断是——**这个静态页应用根本不需要 K8s API，所以不该给它身份**，于是加 `automountServiceAccountToken: false` 把凭证从容器里物理移除。发布侧把 Jenkins 从 admin kubeconfig 换成只够操作 `damo-app` 一个命名空间的最小权限 SA（权限清单是从 `helm upgrade` 实际动作倒推的，还故意不给 `namespaces` 的 create 权限），按「新增→验证→切换→观察→回收」五步节奏切换。最后把这些加固写进 Helm Chart（默认开关关闭、按需 `--set` 打开），并做了一个可反复运行的安全验收脚本。"

### 10.2 高频追问预案

| 追问 | 怎么回答 |
|------|--------------|
| 你的集群安全怎么做的？ | **改造前后对照**：零策略+全通+token 在容器里 → 4 条策略+跨 ns 拒+无 token |
| NetworkPolicy 默认拒绝最容易踩什么坑？ | **实测推翻了一个流行说法**：都说会误杀 kubelet 探针，但 Calico IPIP 下**探针根本不受影响**（Pod 仍 Ready、Endpoints 仍正常）——因为探针走本机路径、不进 `cali-*` 策略链。**真正的坑是"只能靠连接语义判断生效"**：refused → timed out |
| 加了 egress 限流会不会影响服务？ | **不会**——有状态，入方向响应自动放行；有实测 |
| 怎么防数据外传？ | egress 默认拒绝 + 只放行 DNS；实测 `wget baidu.com` 被拒 |
| 为什么不直接给应用一个只读 SA？ | 更彻底：**它不需要身份就不给身份**。不给 token 比"给只读权限"更安全 |
| 收紧权限会不会影响发布？ | 不会，且有证据（F-4）：最小权限 SA 下流水线照样成功发布。**验收标准是"既不多给也不影响业务"** |
| 改权限的节奏是什么？ | 新增→验证→切换→观察→回收，五步；**不能先删再建** |
| 为什么用 Role 不用 ClusterRole？ | **爆炸半径**：纳米级 Role 最多影响一个命名空间。`namespaces` 这类集群级资源只能靠 ClusterRole，所以只给 `get` |
| NetworkPolicy 不生效怎么查？ | 先查 CNI 是否支持（Flannel 不支持）；本项目 Calico 是 **tigera-operator 部署**，组件在 `calico-system`——用 `kubectl get tigerastatus` 看整体健康、`kubectl -n calico-system logs ds/calico-node` 看执行者日志；再用 `kubectl get netpol -o yaml` 回看结构；最后排除 failsafe 端口干扰 |
| `from` 里两个 selector 一起写是什么意思？ | **AND**；分开写在两个数组元素里是 **OR**（Q11） |
| 加固怎么保证不被流水线冲掉？ | 全部写进 Helm Chart（Policy as Code）+ 可反复运行的验收脚本（F-8） |
| 这个项目最难的地方是什么？ | ① **预期落空时的排查**——以为会遇到"探针连环挂"，实测什么都没发生，三次排除后定位到"探针流量不进 CNI 策略链"这个机制（F-5b）；② 收权节奏——必须"先并存再切换"（**讲这个比讲技术细节更能体现工程成熟度**） |
| 资源健康就等于业务可用吗？ | **不等于，这是本项目最反直觉的发现**：默认拒绝后 Pod `READY 1/1`、Endpoints 齐全、零重启——**资源状态完全健康，但业务已全断**。只做 Pod 存活告警的监控会**完整漏掉这次故障**；能发现它的是项目 C 的 `curl` 可用性告警 |

### 10.3 简历写法（可直接用）

> **短版（1 行）**：
> 完成集群安全加固：RBAC 最小权限（应用去 token 化 + 发布者命名空间级最小 Role）+ NetworkPolicy 默认拒绝（Ingress/Egress 双默认拒绝并逐条放行），加固后业务零中断，附可反复运行的验收脚本。

> **长版（3 行，适合放项目详情）**：
> - **网络边界**：盘点发现全集群零 NetworkPolicy，实测 `damo-app` 可横向访问 Prometheus/Loki/Jenkins；按「先 Ingress 后 Egress」分步实施默认拒绝并逐条放行（节点网段覆盖 kubelet 探针与 NodePort SNAT 源、同命名空间压测标签、ingress 命名空间、DNS 出方向），跨命名空间与外网出方向均被拒。
> - **身份权限**：application Pod 采用专用 SA + `automountServiceAccountToken: false`（容器内无 API 凭证）；Jenkins 发布身份从 admin kubeconfig 迁移为命名空间级最小权限 SA（权限清单由 `helm upgrade` 实际动作倒推，`namespaces` 仅授予 `get`），按「新增→验证→切换→观察→回收」节奏完成零中断切换。
> - **工程化**：全部加固以 Helm Chart 开关形式固化（默认关闭、`--set` 启用），沉淀 `verify-hardening.sh` 只读验收脚本，覆盖 RBAC 权限矩阵、业务可用性、跨命名空间与外网阻断等 10 项检查。

### 10.4 与其他项目的叙事串联

```
项目 A  探针 tcpSocket 定版        ──→ 项目 F 的"探针被默认拒绝打挂"能快速定位，因为熟悉探针机制
项目 B  Jenkins 流水线            ──→ 项目 F 要改的正是这条流水线的"发布身份"
项目 C  Prometheus / Grafana      ──→ 项目 F 实测"能从 damo-app 摸到 Prometheus"＝横向移动风险的具体证据
项目 D  Loki 日志                 ──→ 同上；且加固后日志采集不受影响（在 monitoring 命名空间侧）
项目 E  Helm Chart               ──→ 项目 F 的加固必须以 Chart 形式固化，否则被发布冲掉
项目 G  HPA 弹性伸缩              ──→ 项目 F 给 HPA 扩出的新 Pod 套上同一套身份与网络策略
```

**一句话把七个项目串起来**：

> 「A–G 是一条完整的链路：A 让 Pod 活着且能被访问、B 让变更能自动化到达集群、C/D 让我能看见集群在发生什么、E 让交付可审计可回滚、G 让集群能随着负载自己动手、**F 让它动得安全**。这七步合起来，就是一套自建集群的运维闭环——而 F 是唯一一个「不改功能、只改边界」的项目，它证明我知道**能跑起来只是及格线，跑得安全才是交付标准**。」

---

## 10. 学习资源（官方文档）

| 主题 | 资源 |
|------|------|
| **RBAC 授权**（必读） | https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/ |
| 使用 RBAC 鉴权（任务型） | https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding |
| **ServiceAccount 管理** | https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-service-account/ |
| ServiceAccount 管理员指南（token 演进） | https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/service-accounts-admin/ |
| **网络策略**（必读） | https://kubernetes.io/zh-cn/docs/concepts/services-networking/network-policies/ |
| 网络策略声明语法 | https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/policy-resources/network-policy-v1/ |
| **Pod 安全准入（PSA）** | https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-admission/ |
| Pod 安全标准（三种 profile） | https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-standards/ |
| 为 Pod/容器配置 SecurityContext | https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/security-context/ |
| 资源配额（ResourceQuota） | https://kubernetes.io/zh-cn/docs/concepts/policy/resource-quotas/ |
| 准入控制器总览 | https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/ |
| Calico 网络策略文档 | https://docs.tigera.io/calico/latest/network-policy/ |
| `kubectl auth can-i` 参考 | https://kubernetes.io/zh-cn/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/ |
| 练习 | Killercoda 的 RBAC / NetworkPolicy 场景；本地 kind 上复现默认拒绝实验 |
| 书籍 | 《Kubernetes 安全指南》（Kubernetes Security）；《Kubernetes 生产实践》安全章节 |
| 补充（进阶） | OPA/Gatekeeper、Kyverno（策略引擎，做"准入层"的自动化管控） |

---

## 附录 A：文件包清单与操作顺序

### A.1 文件包清单

```
K8s项目F_安全加固/
├── K8s项目F_安全加固实战.md              # 本文档
├── check-groovy-braces.py                # ★ 改 Jenkinsfile 后必跑：花括号配平检查（退出码可用作 CI 门禁）
├── manifests/
│   ├── 10-rbac-damo-deployer.yaml        # 发布者最小权限 SA + Role + 绑定
│   ├── 20-netpol-01-default-deny-ingress.yaml   # 第 1 步：Ingress 默认拒绝
│   ├── 21-netpol-02-allow-ingress.yaml          # 第 2 步：Ingress 白名单（三条规则）
│   ├── 22-netpol-03-default-deny-egress.yaml    # 第 3 步：Egress 默认拒绝
│   ├── 23-netpol-04-allow-egress-dns.yaml       # 第 4 步：放行 DNS（kube-dns 的 53）
│   ├── 24-netpol-05-allow-loadgen-egress.yaml   # 第 5 步：放行 loadgen 的出方向（★ 保住项目 G 的 HPA）
│   ├── 30-quota-limitrange.yaml                 # 可选：资源边界
│   ├── 30-deployer-token-secret.yaml            # §3.5：发布者 SA 的长期 token（可选，作品集推荐）
│   └── verify-hardening.sh                      # ★ 一键验收脚本（只读）
└── chart-patch/                                 # ★ 项目 F 对 Chart 的【全部】改动（不回写项目 E）
    ├── README.md                        # 用法说明：文件 → chart 位置的映射 + 部署/验收/回退
    ├── values.yaml                      # → <chart>/values.yaml（覆盖）：项目 E 原值 + 四项新增
    ├── values-prod.yaml                 # → <chart>/values-prod.yaml（新增）：把加固开关固化进 git
    ├── deployment.yaml                  # → <chart>/templates/deployment.yaml（覆盖）
    ├── service.yaml                     # → <chart>/templates/service.yaml（覆盖）
    ├── networkpolicy.yaml               # → <chart>/templates/networkpolicy.yaml（新增，5 条策略）
    ├── serviceaccount.yaml              # → <chart>/templates/serviceaccount.yaml（新增）
    └── values-fragment.yaml             # 旧版片段（已被 values.yaml 取代，保留仅作参考）
```

> **★ 组织原则（2026-09-24 明确）**：**项目 E 的 `damo-app-chart/` 保持原样、项目 F 不回写它。**
>
> | | 项目 E 的目录 | 项目 F 的 `chart-patch/` |
> |---|---|---|
> | 定位 | 「Helm 化改造」的**历史形态**（独立作品） | 「安全加固」的**全部产物** |
> | 内容 | 没有 `timeoutSeconds`、Service 用默认 `Cluster` 策略 | 四项加固改动 + NetworkPolicy 模板 |
>
> **为什么不回写**：项目 E 当时的形态是它自己的历史状态，不该被后续项目覆盖；
> 项目 F 的改动全部收敛在本目录，**自包含、可单独审阅**，两个项目的文档与交付物都能对得上。
>
> **部署方式**：用本目录的文件覆盖 chart（映射见 `chart-patch/README.md`），
> 再用 `-f values-prod.yaml` 打开加固开关。
>
> **要放进 GitHub 仓库的**：`chart-patch/` 下的文件（`values-fragment.yaml` 除外，它已被取代）。
> **`manifests/` 建议一起提交**——它证明你的加固是**可复现、可验证**的：别人 clone 下来就能 apply、能跑验收脚本。**面试官看到"策略 + 验收脚本"成套出现，会认为你具备工程交付意识。**

### A.2 操作顺序总览

```
阶段 0  执行前检查 6 条（Calico 确认：`calico-system` + `tigerastatus` + `Installation` 封装 + `felixconfiguration` failsafe
        ／ netpol 数量为 0 ／ loadgen 已清理）
   ↓
阶段 2  ★ 先保存「改造前基线」输出 —— 加固后不可补测
   ↓     （零策略 + 全通实测 + 容器内 token 存在）
   ↓
阶段 3  RBAC：反面教材（F-2）→ 应用专用 SA + 去 token → 建最小权限发布者 SA
   ↓     → 权限矩阵验证（F-3）→ Jenkins 五步切换 → 最小权限发布成功（F-4）
   ↓     ★ 关卡：容器内 `ls /var/run/secrets/...` 必须报 No such file
   ↓
阶段 4  网络：记录基线 → 只 deny Ingress → 观察「断了什么」（F-5 / F-5b）
   ↓     ★ 关卡：确认生效判据是「refused → timed out」，而非「Pod 变 NotReady」
   ↓     （实测：本集群探针不受影响，Pod 仍 READY 1/1、Endpoints 仍有 IP）
   ↓     → 放行三条 → 四道关验证（F-5c：外部通/集群内通/Endpoints 有 IP/跨 ns 仍拒）
   ↓     → deny Egress → 观察两个反直觉现象（F-6）→ 放行 DNS
   ↓     → 写进 Chart（开关驱动）
   ↓
阶段 5  可选加固：PSA warn（F-7）/ securityContext / 资源边界
   ↓
阶段 6  跑验收脚本（F-8，与 F-1 配对）→ 整理 before/after 对照表
   ↓
收尾   整理验收输出 → 自绘网络拓扑图 → README 配图 → 提交仓库
```

> **回退预案**：本项目**每一层都可独立回退**——
> - **网络层**：`kubectl delete -f manifests/2*.yaml` 删除全部策略，立刻回到"全通"（业务秒恢复）。
> - **身份层**：`automountServiceAccountToken` 改回 `true` + `helm upgrade`；Jenkins 切换旧凭据（旧的绑定在"回收"前一直保留）。
> - **顺序建议**：如果加固过程中出现业务不可用且 15 分钟内定位不到，**先删网络策略保业务**，再从容排查——**保业务永远优先于保策略**。
