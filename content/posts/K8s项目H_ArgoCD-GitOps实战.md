---
title: "K8s 项目 H：Argo CD 与 GitOps 实战"
date: 2026-10-30T16:46:00+08:00
draft: false
description: "用 Argo CD 把交付从 Jenkins push 模型升级为 pull 模型：Git 声明集群状态、Application 持续协调、配置漂移自动纠偏、回滚就是 git revert，并厘清 Jenkins 与 Argo CD 各自的写入边界。"
tags: ["k8s GitOps"]
categories: [项目]
---

> 📦 **项目文件包**：[下载 K8s项目H_ArgoCD-GitOps.zip](/files/K8s项目H_ArgoCD-GitOps.zip)（含 Application / AppProject manifests、安装说明与预检脚本）。本文件是详细操作手册，两者配合。

> **系列定位**：K8s 运维实习项目路线图第 8 站——A 应用容器化 → B CI/CD → C 监控告警 → D 日志收集 → E Helm 化 → F 安全加固 → G 弹性伸缩 → **H GitOps**。
> **项目定位**：把交付模型从「**Jenkins 推（push）**」升级为「**Argo CD 拉（pull）**」——Jenkins 只负责构建镜像（CI），集群的期望状态由 Git 声明、由 Argo CD 持续协调（CD）。
> **冲刺计划原话是「可只做原理，不实操」**，但本文档**两层都给全**：
> - **第 1 层（必做，2 小时）**：阶段 1 原理 + 阶段 7 的「与 Jenkins 分工」——**面试能讲清"Poll SCM 是内网妥协 → GitOps 是终局"，就足够**；
> - **第 2 层（可选，+3 小时）**：阶段 2–6 的完整实操——在集群里真装一个 Argo CD，跑通「改 Git 自动同步 → 手动改集群被自动纠偏 → 一键回滚」三个演示。
>
> **强烈建议做第 2 层**，理由很实际：A–G 七个项目你都有**可点开的证据**（GIF、截图、脚本）。GitOps 如果只停在"能讲原理"，面试官追问一句「你实际用过吗」就会露底。而 Argo CD 的 UI **自带极其适合截图的拓扑图和 diff 视图**——它是你所有项目里**最上镜**的一个。
> **版本基线**：K8s v1.35｜Argo CD v2.x（`stable` 分支清单）｜复用项目 B/C/E/F 全部环境（3 节点 / containerd 2.x / registry `11.0.1.128:30000` / damo-app 由 Helm 管理 / GitHub 仓库 `GXEL-xy/damo-app` 公开可访问）。
> **前置条件**：① 项目 E 已完成（`chart/damo-app` 在 GitHub 仓库里）；② 项目 G/F 的加固已完成（本项目的 Application 需要引用 `values.yaml` 的现状）；③ 项目 F 的 loadgen 已清理。
> **预计耗时**：原理 2 小时；实操 +2–3 小时（含镜像中转）。
> **权威依据**：Argo CD 官方文档（argo-cd.readthedocs.io）、OpenGitOps 原则（opengitops.dev）、Kubernetes 官方文档。

---

> **📖 怎么用这份文档**
>
> - **前置依赖**：项目 B/E/G 已完成（有 Jenkins 构建、有 Chart、有 HPA）；一个公开 GitHub 仓库
> - **预计耗时**：6–8 小时
> - **学习路径**：阶段 1 push 与 pull 的区别 → 阶段 2 镜像中转 → 阶段 3 装 Argo CD → 阶段 4 建 Application → 阶段 5 首次同步 → 阶段 6 三个核心演示 → 阶段 7 与 Jenkins 分工
> - **验收标准**：Application 显示 Synced/Healthy；改 Git 后自动同步；手改集群被自动纠偏；`git revert` 能完成回滚

## 目录

- [0. 项目概览](#0-项目概览)
- [阶段 1：先懂再做——push 与 pull 的本质区别](#阶段-1先懂再做push-与-pull-的本质区别)
- [阶段 2：镜像准备（Argo CD 全家桶中转）](#阶段-2镜像准备argo-cd-全家桶中转)
- [阶段 3：安装 Argo CD](#阶段-3安装-argo-cd)
- [阶段 4：注册仓库与创建 Application](#阶段-4注册仓库与创建-application)
- [阶段 5：首次同步与验证](#阶段-5首次同步与验证)
- [阶段 6：GitOps 三个核心演示（本项目的精华）](#阶段-6gitops-三个核心演示本项目的精华)
- [阶段 7：与 Jenkins 分工——CI/CD 的最终形态](#阶段-7与-jenkins-分工cicd-的最终形态)
- [8. 故障速查表](#8-故障速查表)
- [9. 自测题（含答案）](#9-自测题含答案)
- [10. 面试要点与简历写法](#10-面试要点与简历写法)
- [11. 学习资源（官方文档）](#11-学习资源官方文档)
- [附录 A：文件包清单与操作顺序](#附录-a文件包清单与操作顺序)
- [附录 B：GitOps 对照表与 Argo CD 命令速查](#附录-bgitops-对照表与-argo-cd-命令速查)

---

## 0. 项目概览

### 0.1 目标与产出

| 产出 | 说明 |
|------|------|
| **GitOps 原理认知** | 能讲清 push 与 pull 的本质区别、为什么 pull 更适合集群、poll SCM 与 webhook 的关系 |
| **可运行的 Argo CD** | 装在 `argocd` 命名空间，UI 通过 NodePort `30400` 访问，admin 密码已改 |
| **两个 Application** | `damo-app`（Helm 源，指向 `chart/damo-app`）——Git 是唯一事实来源的实证 |
| **三个核心演示** | ① 改 Git → 自动同步；② 手改集群 → 自动纠偏（self-heal）；③ 一键回滚 |
| **CI/CD 分工蓝图** | Jenkins 只做 CI（构建推镜像），CD 交给 Argo CD；镜像 tag 传递的两种方案对比 |
| 面试完整故事线 | "我把交付从 Jenkins push 模型升级为 Argo CD pull 模型：Jenkins 只负责构建推镜像，集群状态由 Git 声明、Argo CD 持续协调，漂移自动纠偏、回滚就是 git revert" |

### 0.2 一句话记住 GitOps

> **Git 是唯一的事实来源（Single Source of Truth）；集群状态是 Git 的投影；任何人手动改集群，都会被自动纠偏抹掉。**

而 Argo CD 是这个理念的**具体实现**：一个跑在集群里的「**状态收敛器**」——它不停地比较「**Git 里声明的**」和「**集群里实际的**」，不一致就动手拉平。

### 0.3 架构与数据流（push vs pull）

```
┌──────────────────── 现状（项目 B–G）：push 模型 ─────────────────────┐
│                                                                      │
│  开发者 ──push──► GitHub ──轮询──► Jenkins（在集群内）                │
│                                      │                               │
│                                      │ ★ Jenkins 持有集群凭据         │
│                                      │   主动向 API Server 推送变更    │
│                                      ▼                               │
│                              kubernetes API Server                   │
│                                      │                               │
│                                集群实际状态                           │
│                                                                      │
│  问题：① 集群凭据在 CI 系统里（凭据外扩）；                            │
│        ② 只有"变更时"才收敛 → 有人手动改了集群，没人知道；              │
│        ③ 内网无法收 webhook → 只能轮询（有延迟、有开销）               │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────── 本项目（项目 H）：pull 模型 ─────────────────────┐
│                                                                      │
│  开发者 ──push──► GitHub（唯一事实来源）                              │
│                       ▲                                              │
│                       │ ★ Argo CD 主动拉（pull）读取 Git 声明          │
│                       │   （不需要 GitHub 能访问集群！）               │
│  ┌────────────────────┴─────────────────────────────┐                │
│  │  Argo CD（跑在集群内，argocd 命名空间）             │                │
│  │  ┌────────────────────────────────────────────┐  │                │
│  │  │ 持续协调循环（每 3 分钟 + Git webhook 触发）  │  │                │
│  │  │  ① 拉取 Git 里声明的期望状态                 │  │                │
│  │  │  ② 读取集群里的实际状态（API Server）         │  │                │
│  │  │  ③ diff：不一致？→ OutOfSync                │  │                │
│  │  │  ④ automated + selfHeal → 自动 apply 拉平    │  │                │
│  │  └────────────────────────────────────────────┘  │                │
│  └────────────────────┬─────────────────────────────┘                │
│                       │ 下发（用集群内 SA，凭据不出集群）               │
│                       ▼                                              │
│                集群实际状态 ◄── 有人手动改？下一轮就被纠偏回去           │
└──────────────────────────────────────────────────────────────────────┘
```

**这张图的价值在于左侧是"我原来的做法"，右侧是"我演进后的做法"**——面试时把这个演进讲出来，比直接讲 Argo CD 好用得多。

### 0.4 关键技术决策（面试可讲）

| 决策 | 选择 | 理由 |
|------|------|------|
| **是否真的实操** | ✅ 做（虽然计划说"可只做原理"） | A–G 都有可点开的证据；GitOps 只讲原理经不起追问。且 Argo CD 的 UI 是最上镜的素材 |
| **仓库认证** | **公开仓库，不配凭据** | `GXEL-xy/damo-app` 是公开仓库，Argo CD 免认证就能拉。**好处：少一个凭据要管**（呼应项目 F 的最小权限思路）。私有仓库场景见 4.4 |
| **Application 的 source 类型** | **Helm Chart 目录**（`path: chart/damo-app`） | Argo CD **原生支持渲染 Helm Chart**，不需要额外插件。这样项目 E 的手写 Chart **一行不用改**就被 GitOps 接管——**Chart 化的收益再次兑现** |
| **参数注入方式** | 用 `helm.parameters` 传 `image.tag`，版本仍由构建号决定 | 保留"镜像 tag 由流水线决定"的语义；不引入 Argo CD Image Updater（那是另一个组件，属于进阶） |
| **同步策略** | `automated` + `prune: true` + `selfHeal: true` | **selfHeal 是本项目的灵魂**——没有它就没有"自动纠偏"这个演示；`prune` 让 Git 里删掉的资源在集群里也被删（否则 Git 里删了、集群里还在，Git 就不是唯一事实来源了） |
| **是否建命名空间** | ❌ 不用 `CreateNamespace=true` | 命名空间已存在（项目 E 建的）；且开启它需要给 Argo CD 的 controller 补 `namespaces create` 权限——**延续项目 F 的最小权限思路，能不给就不给** |
| **HPA 与 GitOps 的冲突** | 加 `ignoreDifferences: /spec/replicas` | ★ **这是本项目最"高级"的一个点**：项目 G 让 HPA 管副本数，而 GitOps 的 selfHeal 会试图把副本数拉回 Git 的值——**两者抢控制权**。正确做法是让 GitOps 显式忽略运行时字段，见 6.5 |
| **不装什么** | ❌ 不装 ApplicationSet、不接 SSO/Dex、不装 Image Updater、不用 Argo Rollouts | 每一项都是独立主题。**先跑通"Git 声明 → 自动收敛"这条主链路**，其余按需扩展。面试被问到能说出"我知道它们是什么、为什么这一版不做"就够了 |

### 0.5 资源与端口现状

| 项 | 值 | 说明 |
|----|-----|------|
| 新增命名空间 | `argocd` | Argo CD 全部组件装在这里 |
| **新增 NodePort** | **30400** → `argocd-server` 的 443 | ⚠️ 端口规划：30000/30080/31000/30090/30091/30093/30300/32541 已占用，取 30400 |
| 新增镜像 | **3 个**：argocd、redis、dex | **全部必须中转**到 `11.0.1.128:30000`（铁律）。清单见阶段 2 |
| 新增集群对象 | 约 10 个（Deployment ×2~3、StatefulSet ×0、Service ×2、Secret、ConfigMap、RBAC 若干、CRD ×3） | Argo CD 是"中型"组件，比 Prometheus 轻、比 Loki 重 |
| Application CRD | `applications.argoproj.io`、`appprojects.argoproj.io`、`applicationsets.argoproj.io` | 由 install.yaml 一起装 |
| 资源占用估计 | 约 500m CPU / 1Gi 内存（3 节点集群可承受） | 若节点吃紧，可给 `argocd-repo-server` 降 requests |

### 0.6 执行前检查清单（10 分钟）

> **⚠️ 执行环境先说清楚（别弄混）**
> **集群跑在另一台电脑上**（那台机器里的 3 台 VM，11.0.1.128/129/130）。
> 写文档的这台机器**永远连不上 11.0.1.x**——那是那台宿主内部的 VM 网络，不是本机所在的局域网。
> 所以本项目的操作方式固定为：**本机写文档/脚本 → 拷到那台机器 → 在那台机器的 master 上执行 → 输出贴回**。
> 检查「那台电脑本身能不能上 GitHub」也很重要，但那是宿主的事；**Argo CD 需要的是「Pod 能出网」**，见第 ⑦ 条。

> **💡 已脚本化（可选）**：同目录下的 **`h-preflight.sh`** 一键跑完下面全部内容，并把结果落盘成 `h-baseline-<时间戳>/`（含 `helm-values.yaml`、`resources-before.yaml`），阶段 5 直接拿它逐项对账。
> 传到那台机器后在 master 上跑：`bash h-preflight.sh`
> **懒得传文件就用下面的「粘贴版」**——一段全选粘贴到 master 终端，效果等价，只是不留基线文件。

**粘贴版（一次全选，粘到 master 终端）**

```bash
echo "=== 1 节点 ==="; kubectl get nodes -o wide
echo "=== 2 release ==="; helm list -n damo-app
echo "=== 3 ★ 生产开关（接管前基线，务必截图）==="; helm -n damo-app get values damo-app
echo "=== 4 ★ 对象计数（阶段5必须一个不少）==="
kubectl -n damo-app get hpa; kubectl -n damo-app get netpol
echo "HPA=$(kubectl -n damo-app get hpa --no-headers 2>/dev/null | wc -l)  NETPOL=$(kubectl -n damo-app get netpol --no-headers 2>/dev/null | wc -l)"
echo "=== 5 业务 ==="; kubectl -n damo-app get pods -o wide
for ip in 11.0.1.129 11.0.1.130; do printf '%s:30090 → ' "$ip"; curl -s --noproxy '*' --max-time 5 "http://$ip:30090/" | grep -o 'Build #[0-9]*' || echo FAIL; done
echo "=== 6 loadgen ==="; kubectl -n damo-app get deploy loadgen 2>&1 | tail -2
echo "=== 7 端口 30400 ==="; kubectl get svc -A -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name,NP:.spec.ports[*].nodePort' --no-headers | grep -w 30400 || echo "30400 空闲 ✓"
echo "=== 8 ★ Pod 出网 GitHub（Argo CD 命脉）==="
# ★ 不要用 kubectl run -i --rm：超时会把 Pod 删掉、证据一起消失（实测踩过）
kubectl run h-netcheck --image=11.0.1.128:30000/busybox:1.36 --restart=Never --command -- sleep 600 >/dev/null
if kubectl wait --for=condition=Ready pod/h-netcheck --timeout=90s; then
  kubectl exec h-netcheck -- sh -c 'nslookup github.com >/dev/null 2>&1 && echo DNS_OK || echo DNS_FAIL; nc -z -w 8 github.com 443 >/dev/null 2>&1 && echo TCP443_OK || echo TCP443_FAIL'
else
  echo "Pod 没起来 —— 看这里（多半是镜像拉不到）："; kubectl describe pod h-netcheck | tail -20
fi
kubectl delete pod h-netcheck --wait=false >/dev/null 2>&1
echo "=== 9 私有仓库镜像 ==="; curl -s --noproxy '*' --max-time 8 http://11.0.1.128:30000/v2/_catalog
echo "=== 10 资源余量 ==="; kubectl top nodes; df -h /var/lib/containerd
echo "=== 体检结束 ==="
```

**下面是分解版**——想知道每条在查什么，就照着读。

```bash
# ① 确认项目 E 的 Chart 在 GitHub 上可访问（★ 这是 Argo CD 的数据源）
curl -sI https://raw.githubusercontent.com/GXEL-xy/damo-app/main/chart/damo-app/Chart.yaml | head -1
# 期望：HTTP/2 200
# 若 404，先确认分支名（main 还是 master）与路径：
git ls-remote --heads https://github.com/GXEL-xy/damo-app.git

# ② 确认命名空间与 release 现状
kubectl get ns damo-app
helm list -n damo-app

# ②.5 ★★ 记录"当前生效的生产开关"——接管时必须原样复现（否则会静默丢失，见 4.2.1）
helm -n damo-app get values damo-app
# 期望能看到：autoscaling.enabled=true、networkPolicy.enabled=true
# 同时确认这些对象此刻真的存在（同步后要能对得上）：
kubectl -n damo-app get hpa
kubectl -n damo-app get netpol
# 期望：1 个 HPA（damo-app）+ 5 条 NetworkPolicy
# ★ 把这两条的输出【截图/存文件】——它是"接管前基线"，阶段 5 要逐项对比

# ③ 记录当前状态（GitOps 接管前的基线，后面要对比）
kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.replicas}'; echo
kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app
curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'

# ④ 确认项目 F 的 loadgen 已清理
kubectl -n damo-app get deploy loadgen
# 期望：NotFound 或 READY 0/0

# ⑤ 查端口占用（30400 是否空闲）
kubectl get svc -A | grep -E "3[0-2][0-9]{3}" | sort
# 确认输出里没有 30400

# ⑥ 确认节点资源余量（Argo CD 要占约 500m CPU / 1Gi 内存）
kubectl top nodes
# 期望：三个节点的 CPU% 都在 50% 以下
```

> ⚠️ **第 ① 条是本项目的关键前置**：Argo CD 的 pull 模型要求**集群能主动访问 GitHub**（不需要 GitHub 反向访问集群）。你的内网 VM 已经能 clone GitHub（项目 B 的 Jenkins 就是这么拉代码的），所以这条路是通的——**而这正是 pull 模型相对 webhook 的最大优势：它把"网络可达性"的要求从"外部能进来"变成"内部能出去"，后者在内网环境容易得多。**

---

## 阶段 1：先懂再做——push 与 pull 的本质区别

> **本阶段是"必做"的那 2 小时。** 即使不做实操，这一节的内容也足够支撑面试。

### 1.0 开胃故事：快递送货 vs 出门取件（先建立直觉）

**push 模型（项目 B–G 的做法）= 快递送货上门。**

Jenkins 是快递员，手里有你家的钥匙（集群凭据）。你在 GitHub 下单（提交代码），他轮询到有新订单（`pollSCM`），就自己开门进屋，把东西摆好。
- 问题一：**钥匙在快递员手里**。他被抢（Jenkins 被攻破），你家就完了。
- 问题二：**他只在有订单时才来**。你自己把沙发挪了位置他不会管——**集群会慢慢偏离"应该的样子"，而且没人知道**。
- 问题三：**你家门牌要能被外面看到**。如果是没有公网地址的城中村（内网环境），快递员根本找不到你，只能改成"每 5 分钟自己去门口看一眼有没有包裹"（**这就是 poll SCM 的本质：一种无奈的对策**）。

**pull 模型（GitOps 的做法）= 你雇了个管家，他自己去驿站取件。**

Argo CD 是住在家里的管家。他知道"家里应该是什么样"（Git 里的声明），每 3 分钟自己出门去驿站看一眼（拉取 Git），回来对照家里的实际情况，**不一致就立刻摆回原样**。
- 关键一：**钥匙不出门**。管家用家里的身份（集群内 SA）干活，**GitHub 不需要能访问集群**——网络要求从"外面能进来"变成"自己能出去"。
- 关键二：**他不是只在有订单时才来**。你手动把沙发挪了，他下一轮就搬回去（**self-heal 自动纠偏**）。
- 关键三：**"应该怎么样"写在纸上，谁都能查、能改、能回滚**。想撤掉上次的改动？`git revert` 一下，管家自然就把家里改回去了。

> **口诀**：**push 是"我推给你"，pull 是"我来对齐"；push 靠凭据外扩 + 事件触发，pull 靠状态声明 + 持续协调。**
> **而"持续协调"这四个字，就是 GitOps 相对 CI/CD 脚本化部署最本质的升级**——部署不再是一个"动作"，而是一个**持续维持的不变量**。

### 1.1 GitOps 的四条原则（OpenGitOps 官方定义）

| # | 原则 | 含义 | 本项目的体现 |
|---|------|------|-------------|
| 1 | **声明式（Declarative）** | 系统期望状态被**声明**出来，而不是靠一串命令去"做到" | Helm Chart + `values.yaml`（项目 E 的成果） |
| 2 | **版本化且不可变（Versioned and Immutable）** | 期望状态存在版本控制系统里，**历史可追溯、内容不可篡改** | Git 里的 `chart/damo-app`，每次提交就是一个版本 |
| 3 | **自动拉取（Pulled Automatically）** | 软件代理**自动从源拉取**期望状态 | Argo CD 每 3 分钟轮询 Git（或 webhook 立即触发） |
| 4 | **持续协调（Continuously Reconciled）** | 代理**持续观察实际状态并驱使其向期望状态收敛** | `selfHeal: true`——手动改集群被自动纠偏 |

> **面试高价值点**：很多人把 GitOps 等同于"用 Git 管 YAML"。**但第 3、4 条才是分水岭**——如果你只是把 YAML 存在 Git 里、然后让 Jenkins 在发布时 `kubectl apply`，那还是 **push 模型**（原则 3、4 都没满足）。**只有"代理主动拉取 + 持续纠偏"才叫 GitOps。** 这句话能立刻分辨出"读过文章"和"真的理解"。

### 1.2 Argo CD 的内部架构（四个组件，各管一件事）

```
┌─────────────────── argocd 命名空间 ───────────────────┐
│                                                        │
│  [argocd-server]              [argocd-repo-server]     │
│   · 提供 Web UI 与 API          · 负责拉 Git 仓库        │
│   · 处理 kubectl/CLI 请求        · 负责渲染（Helm/Kustomize/    │
│   · 对外暴露（本项目 NodePort    │   plain YAML）         │
│     30400 → 443）              · ★ 有本地缓存（fetch 不必每次拉全量）│
│           │                            ▲               │
│           │                            │ 请求渲染       │
│           ▼                            │               │
│  [argocd-application-controller]───────┘               │
│   · ★ 核心：持续协调循环                                │
│   · 比较 desired(Git) vs live(cluster)                 │
│   · 计算 diff、决定 Sync/OutOfSync                      │
│   · 执行 apply / prune（用 SA 权限）                     │
│   · 单副本 StatefulSet（要保证一致性）                    │
│           │                                            │
│           ▼                                            │
│  [argocd-redis]                                        │
│   · 缓存（controller 与 server 共享状态/锁）              │
│   · 不是必须持久化（丢了重建即可）                        │
│                                                        │
│  [argocd-dex-server]（可选）                            │
│   · 只在使用 SSO/OIDC 登录时需要                          │
│   · 本项目用本地 admin 账号 → 可以不部署（进阶减法）      │
└────────────────────────────────────────────────────────┘
```

| 组件 | 一句话职责 | 挂了会怎样 |
|------|-----------|-----------|
| **application-controller** | **协调器**：算 diff、执行同步 | **核心挂掉** → 不再自动纠偏（但集群里已部署的东西照常运行） |
| **repo-server** | **渲染器**：拉 Git、渲染 Helm/Kustomize | 无法获取期望状态 → 全部 Application 报错 |
| **server** | **门面**：UI/API | 看不到界面，但自动同步仍在工作（**这个区分很能体现理解深度**） |
| **redis** | **缓存** | 性能下降、可能重建状态 |
| **dex** | SSO 登录（可选） | 只影响 SSO 登录，不影响同步 |

> **★ 面试加分点**：「Argo CD 的 UI 挂了不影响自动同步」——因为 `argocd-server` 只是门面，真正干活的是 `application-controller`。这说明我理解**控制面组件的职责边界**，而不是把它当成一个黑盒。

### 1.3 与 Jenkins 的职责边界（这是全系列最重要的一张对照表）

| 维度 | Jenkins（项目 B–G） | Argo CD（项目 H） | 结论 |
|------|-------------------|------------------|------|
| **模型** | push（主动推送到集群） | **pull**（从 Git 拉取期望状态） | 互补 |
| **触发** | 代码提交 → 构建 | **持续协调**（不依赖事件） | 不是替代关系 |
| **集群凭据位置** | **CI 系统里**（凭据外扩，风险面大） | **集群内**（SA，凭据不出集群） | pull 更安全 |
| **网络要求** | 需要能"推送"到集群（内网要靠轮询妥协） | 只需集群能**出网拉 Git** | pull 对内网友好 |
| **擅长** | 构建、测试、扫描、推镜像（**有状态、有中间产物**） | 部署、漂移纠偏、回滚、多环境一致性（**声明式、无中间产物**） | **分工而非取代** |
| **回滚** | `helm rollback`（集群里的 revision） | **`git revert`**（Git 历史本身就是回滚依据） | 回滚语义统一到 Git |
| **审计** | Jenkins 构建历史 | **Git 提交历史 + Argo CD 的 Sync 历史** | 二者都有价值 |
| **有人手动改了集群** | 不知道（下次发布才覆盖） | **3 分钟内自动纠偏** | pull 的核心优势 |
| **引入新组件** | 已有 | 新增（~500m CPU/1Gi 内存） | 有成本 |

> **定位结论（一定要记牢）**：
> **Jenkins 是"流水线工人"，Argo CD 是"看家管家"。** 工人负责把砖头造出来（构建镜像）、把图纸画好（更新 Git 里的声明）；管家负责保证房子**始终**和图纸一致。
> **不要用 Jenkins 做持续协调**（它没有"持续"这个概念），**也不要用 Argo CD 做构建**（它不做 CI，没有构建产物、没有测试报告）。**它们的交集只有一处**：谁来把"新镜像 tag"写进 Git——这就是阶段 7 要讲的。

### 1.4 四个易混概念的一次性澄清

| 易混 | 区别 |
|------|------|
| **CI vs CD** | CI = 构建 + 测试（产出**制品/镜像**）；CD = 部署（让**集群状态**等于期望）。**本项目把 CD 从 Jenkins 挪到 Argo CD** |
| **GitOps vs "YAML 存 Git"** | GitOps 必须满足「自动拉取 + 持续协调」；只是把 YAML 放 Git 里再 `kubectl apply` 仍是 push 模型 |
| **Argo CD vs Flux** | 都是 pull 模型 GitOps 工具。Argo CD 有**更成熟的 UI 与多集群管理**、声明单位是 `Application` CRD；Flux 更"K8s 原生"（纯 CRD，无 UI 强依赖）。面试可讲"我选了 Argo CD 因为它的 UI 对团队上手更友好" |
| **GitOps vs Helm** | 完全正交：Helm 解决「**怎么把一份模板渲染成多环境清单**」，GitOps 解决「**谁在什么时候把渲染结果同步到集群**」。本项目里**两者叠加使用**：Argo CD 调用 Helm 渲染，再同步（这正是 0.4 决策表里选 Helm source 的原因） |

### 1.5 阶段自测（不做实操也必须能答）

- [ ] 能用"快递送货 vs 雇个管家"讲清 push 与 pull 的区别
- [ ] 能说出 GitOps 四原则，并解释为什么"只是把 YAML 存 Git"不算 GitOps
- [ ] 能说出 poll SCM 的本质妥协，以及 pull 模型为什么对内网更友好
- [ ] 能说出 Argo CD 四个组件的职责，并指出哪个是核心
- [ ] 能解释"Argo CD 的 UI 挂了会不会影响自动同步"
- [ ] 能说清 Jenkins 与 Argo CD 的分工边界（谁做构建、谁做协调）
- [ ] 能解释 Helm 与 GitOps 是正交而非替代关系

---

## 阶段 2：镜像准备（Argo CD 全家桶中转）

> 延续铁律：**master 上 docker 拉 → tag → push 到私有仓库 → 各节点 containerd 从私有仓库拉**。

### 2.1 先下载 install.yaml（不要凭记忆列镜像）

```bash
# 在 master 上
mkdir -p ~/argocd-install && cd ~/argocd-install
curl -LO https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ★ 从清单里"提取"出真实需要的镜像列表（版本随发行变化，绝不硬编码）
grep -E '^\s+image:' install.yaml | sed 's/.*image: *//' | tr -d '"' | sort -u
```

**预期输出（形如，实际版本以你下载到的为准）**：

```
ghcr.io/dexidp/dex:v2.x.x
public.ecr.aws/docker/library/redis:7.x-alpine
quay.io/argoproj/argocd:v2.x.x
```

> **★ 这个"从清单提取镜像"的做法本身就是方法论**：不凭记忆写版本号，而是**让清单告诉你需要什么**。面试时如果被问「你怎么知道要中转哪些镜像」，这个回答比背清单更有说服力。

### 2.2 逐个中转（可脚本化）

```bash
# 在 master 上执行（docker 已配代理）
REGISTRY=11.0.1.128:30000

# ① argocd 本体
docker pull quay.io/argoproj/argocd:v2.13.0            # ← 换成你提取到的版本
docker tag  quay.io/argoproj/argocd:v2.13.0 ${REGISTRY}/argocd:v2.13.0
docker push ${REGISTRY}/argocd:v2.13.0

# ② redis
docker pull public.ecr.aws/docker/library/redis:7.2-alpine   # ← 以实际提取到的为准
docker tag  public.ecr.aws/docker/library/redis:7.2-alpine ${REGISTRY}/redis:7.2-alpine
docker push ${REGISTRY}/redis:7.2-alpine

# ③ dex（若你决定保留 dex；纯 admin 登录可跳过并删掉对应 Deployment）
docker pull ghcr.io/dexidp/dex:v2.41.1                 # ← 以实际提取到的为准
docker tag  ghcr.io/dexidp/dex:v2.41.1 ${REGISTRY}/dex:v2.41.1
docker push ${REGISTRY}/dex:v2.41.1

# ④ 验证
curl -s http://11.0.1.128:30000/v2/_catalog | tr ',' '\n' | grep -E 'argocd|redis|dex'
```

> ⚠️ **三个镜像源分别在三个不同的仓库**（quay.io / public.ecr.aws / ghcr.io）——这正是"中转铁律"的价值所在：**节点只需要认识一个私有仓库**，上游有几个源、能不能直连都不重要。
>
> **如果某个镜像拉不动**（GitHub Container Registry 在国内经常不稳）：可先在某台能上网的机器上 `docker save` 成 tar，再 `scp` 到 master 后 `docker load`，然后 tag + push。**这条路要会走**——它是"镜像中转"的兜底方案。

### 2.3 改写 install.yaml 里的镜像地址

```bash
cd ~/argocd-install

# 用 sed 批量替换三处（注意：先备份原文件）
cp install.yaml install.yaml.orig
sed -i 's|quay.io/argoproj/argocd:|11.0.1.128:30000/argocd:|g' install.yaml
sed -i 's|public.ecr.aws/docker/library/redis:|11.0.1.128:30000/redis:|g' install.yaml
sed -i 's|ghcr.io/dexidp/dex:|11.0.1.128:30000/dex:|g' install.yaml

# ★ 核对：不应再有上游仓库地址残留
grep -E '^\s+image:' install.yaml | sed 's/.*image: *//' | tr -d '"' | sort -u
# 期望：三行，全部以 11.0.1.128:30000/ 开头

# ★ 再做一次反向检查：有没有漏掉的
grep -cE 'quay\.io|public\.ecr\.aws|ghcr\.io' install.yaml
# 期望：0
```

---

## 阶段 3：安装 Argo CD

### 3.1 安装（两条命令）

```bash
# ① 创建命名空间
kubectl create namespace argocd

# ② ★ 用 server-side apply 安装（不要用默认的 client-side apply，原因见下方）
kubectl apply --server-side=true --force-conflicts -n argocd -f ~/argocd-install/install.yaml

# ③ 等待全部就绪（首次会拉镜像，可能需要 1-3 分钟）
kubectl -n argocd get pods -w
# 等到全部 Running（server / repo-server / application-controller / redis / dex）
```

> ### ⚠️ 为什么必须加 `--server-side=true`（实测踩到的坑）
>
> **现象**：`kubectl apply -f install.yaml` 前面几十个对象全部 `created`，最后报：
> ```
> The CustomResourceDefinition "applicationsets.argoproj.io" is invalid:
> metadata.annotations: Too long: may not be more than 262144 bytes
> ```
>
> **机制**：client-side apply 会把**整个对象的 JSON** 塞进
> `kubectl.kubernetes.io/last-applied-configuration` 这个注解里存起来（下次 apply 用它做三方合并）。
> 而 `applicationsets.argoproj.io` 这个 CRD 带完整的 OpenAPI schema，**序列化后超过 256KB**，
> 于是注解本身超限 → 整个对象被 API Server 拒绝。
>
> **解法**：`--server-side=true`。Server-Side Apply 把合并逻辑挪到 API Server，
> **不再写 `last-applied-configuration` 注解**，因此没有这个大小限制。
>
> **★ 三个连带要点（面试可讲）**：
> 1. **它不是"没装上"**：出错的只有那一个 CRD，其余对象都已 created。
>    重跑 server-side apply 是**幂等**的，已存在的对象会被 update，不会重复创建。
> 2. `--force-conflicts` 是必要的：同一个对象先被 client-side 创建过，
>    字段管理器（field manager）不同会报 `conflict`，`--force-conflicts` 让 server-side 接管所有权。
> 3. **这正是声明式工具的一个真实边界**：`kubectl apply` 不是"万能幂等"，
>    遇到超大 CRD（Argo CD ApplicationSet、Prometheus Operator、部分 Istio CRD 都有这问题）必须换 SSA。
>    判断方法：报错里出现 `last-applied-configuration` 或 `annotations: Too long` 就是它。
>
> **核对三个 CRD 是否齐全**：
> ```bash
> kubectl get crd | grep argoproj
> # 期望三行：applications / applicationsets / appprojects
> ```

**组件清单与预期状态**：

| 对象 | 类型 | 预期 |
|------|------|------|
| `argocd-server` | Deployment | 1/1 Running |
| `argocd-repo-server` | Deployment | 1/1 Running |
| `argocd-application-controller` | **StatefulSet** | 1/1 Running（注意是 StatefulSet，不是 Deployment） |
| `argocd-redis` | Deployment | 1/1 Running |
| `argocd-dex-server` | Deployment | 1/1 Running（若保留） |

```bash
# 确认全部就绪
kubectl -n argocd get deploy,statefulset
kubectl -n argocd get pods

# CRD 已装（Application 才有地方放）
kubectl get crd | grep argoproj
# 期望：applications.argoproj.io / applicationsets.argoproj.io / appprojects.argoproj.io
```

> **常见卡点**：`argocd-application-controller` 是 **StatefulSet**，如果你 `kubectl get deploy` 找不到它不要慌——这是设计使然（要保证单副本一致性，用 StatefulSet 有稳定标识）。

### 3.2 暴露 UI：把 `argocd-server` 改成 NodePort 30400

```bash
# 方式一：直接 patch（快速，但状态不在 Git 里）
kubectl -n argocd patch svc argocd-server -p '{
  "spec": {
    "type": "NodePort",
    "ports": [
      {"name":"http","port":80,"protocol":"TCP","targetPort":8080},
      {"name":"https","port":443,"protocol":"TCP","targetPort":8080,"nodePort":30400}
    ]
  }
}'
```

> **⚠️ 注意一个反直觉的点**：Argo CD 的 `argocd-server` **只有 8080 一个容器端口**（同时支持 HTTP 和 HTTPS，由它自己判断）。所以上面 `targetPort` 都指向 8080，`port` 才有 80/443 之分。如果写成 `targetPort: 443`，连接会被拒。
>
> **方式二（推荐，能进 Git）**：把 Service 定义写进一个 Git 管理的目录（或用 Helm/Kustomize），让 Argo CD 自己管自己的 Service。但这属于"自举（bootstrap）"话题，本项目为控制复杂度用方式一，并在文档中记录取舍。

```bash
kubectl -n argocd get svc argocd-server
# 期望：TYPE = NodePort，PORTS 里有 443:30400/TCP

# 验证外部可达（注意是 HTTPS，证书自签 → 用 -k 跳过校验）
curl -sk -o /dev/null -w '%{http_code}\n' https://11.0.1.128:30400/
# 期望：200 或 404（都说明服务起来了）
```

### 3.3 拿到初始密码并登录

```bash
# 初始管理员密码存在 Secret 里（用户名固定为 admin）
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

# ★ 记录这个密码，浏览器打开 https://11.0.1.128:30400
#   用户名 admin / 密码 上一步的输出
#   浏览器会警告证书不受信任（自签）→ 点"高级 → 继续访问"
```

### 3.4 改密码（必做）+ 可选的 CLI 安装

```bash
# ===== 改密码 =====
# 方式一：用 UI（左侧 User Info → Update Password）
# 方式二：用 CLI
#   在能联网的机器上下载 argocd CLI：
#     curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
#     chmod +x argocd && sudo mv argocd /usr/local/bin/
#   然后：
#     argocd login 11.0.1.128:30400 --username admin --password <初始密码> --insecure
#     argocd account update-password

# ===== 改完密码后，删除初始密码 Secret（重要）=====
kubectl -n argocd delete secret argocd-initial-admin-secret
# ★ 为什么删：这个 Secret 明文存着初始密码，任何能读 argocd 命名空间 Secret 的身份
#   都能拿到它。删掉不影响已登录的会话，也符合项目 F 的"最小暴露面"原则。
```

---

## 阶段 4：注册仓库与创建 Application

### 4.1 公开仓库：免凭据直接接（本项目选择）

因为 `GXEL-xy/damo-app` 是**公开仓库**，Argo CD **不需要任何凭据**就能拉取。这也是本项目「不新增凭据」的原因（呼应项目 F 的最小权限思路：**能不新增的凭据就不新增**）。

**但有一个前提要验证**：Argo CD 的 `repo-server` 要能**从集群内访问到 GitHub**。

```bash
# 验证 repo-server 能出网（这是 pull 模型的生命线）
REPO_POD=$(kubectl -n argocd get pod -l app.kubernetes.io/name=argocd-repo-server -o jsonpath='{.items[0].metadata.name}')
kubectl -n argocd exec "$REPO_POD" -- sh -c \
  'git ls-remote --heads https://github.com/GXEL-xy/damo-app.git 2>&1 | head -3'
# 期望：能看到 refs/heads/main（或 master）
# 若超时/失败 → 就是集群出网问题，不是 Argo CD 配置问题（★ 这个区分很重要）
```

> **★ 排错思路（面试可讲）**：Argo CD 报「仓库不可达」时，第一件事不是去翻 Argo CD 配置，而是**在 repo-server Pod 里直接 `git ls-remote` 试一次**。这一下就能把问题分成两类：**网络问题**（出网不通）还是 **Argo CD 配置问题**（认证/路径错）。**用"在故障组件内部做最小复现"来切分问题，是排障的通用手法。**

### 4.1.1 ⚠️ 实测踩到的坑：TCP 通 ≠ HTTPS 通，repo-server 要单独配代理

本项目实测遇到的真实报错（`Application` 状态 `Unknown`，`operationState.phase=Error`）：

```
ComparisonError: Failed to load target state: failed to generate manifest for source 1 of 1:
rpc error: code = Unknown desc = failed to list refs:
Get "https://github.com/GXEL-xy/damo-app.git/info/refs?service=git-upload-pack":
context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

而在 repo-server 里直接跑 `git ls-remote` 报的是另一句：

```
fatal: unable to access 'https://github.com/GXEL-xy/damo-app.git/':
Recv failure: Connection reset by peer
```

**★ 三条关键判断**：

| 观察到什么 | 说明什么 |
|---|---|
| 之前用 busybox 的 `nc -z github.com 443` 返回**通** | **TCP 三次握手成功 ≠ HTTPS 可用**。握手在明文阶段就完成了，TLS 的 SNI 是握手之后才发的——中间设备完全可以"让你连上，再把你重置" |
| 报错是 `Connection reset by peer` 而不是 `timed out` | **主动重置**（有设备在干预），不是路由不通。这两种语义要分开看（项目 F 的判断阶梯在这里同样适用） |
| `argocd-repo-server-network-policy` 的 `policyTypes` 只有 `Ingress` | **不是 NetworkPolicy 拦的**。先排除这条，别在策略上白花时间 |

**解法**：给 `argocd-repo-server` **单独**配代理。

```bash
# ① 先从这台机器的 docker 配置里找出正在用的代理（docker 能拉镜像，说明它一定配了）
cat /etc/systemd/system/docker.service.d/*.conf
# 本项目实测值为：http://11.0.1.1:7890（11.0.1.1 是 VMware NAT 网关，代理跑在宿主机上）

# ② 配给 repo-server（★ NO_PROXY 必须含集群内部网段，否则访问 argocd-server/redis 也会绕代理）
kubectl -n argocd set env deploy/argocd-repo-server \
  HTTP_PROXY=http://11.0.1.1:7890 \
  HTTPS_PROXY=http://11.0.1.1:7890 \
  NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,10.244.0.0/16,11.0.1.0/24,.svc,.cluster.local

kubectl -n argocd rollout status deploy/argocd-repo-server --timeout=180s
```

```bash
# ③ 复验（这一步过了才算真通）
kubectl -n argocd exec deploy/argocd-repo-server -- \
  git ls-remote https://github.com/GXEL-xy/damo-app.git 2>&1 | head -3
# 期望：<sha>  HEAD / <sha>  refs/heads/main
```

**★ 四个值得记住的点**：

1. **只配 `repo-server` 就够**：`argocd-server` 只处理 UI/API，`application-controller` 走集群内 API 和 repo-server 的 gRPC，**它们都不需要出外网**。按需授权，不搞"全组件都配代理"。
2. **`NO_PROXY` 不是可选项**：漏了 `10.96.0.0/12`（Service CIDR）和 `.svc`，repo-server 访问 `argocd-redis`、`argocd-server` 都会绕去代理，直接连不上——**从"拉不到 Git"变成"整个 Argo CD 不工作"**。
3. **代理地址不要猜**：从 `docker.service.d` 里读，那是这台机器上**已经被验证可用**的地址（项目 B 拉镜像就靠它）。这比"问宿主机代理开在哪"可靠。
4. **等价于给 repo-server 开了条外网专线**：以后要接内网 Git（如自建 Gitea），把它的地址加进 `NO_PROXY`，否则会绕代理绕丢。

> **面试价值**：这道题把「网络分层排障」讲透了——
> **`nc` 通只能证明 L3/L4 通；git 要的是 L7 通。** 报错语义（`reset` vs `timeout`）指向不同的拦截方式；
> 先用 NetworkPolicy 排除集群内因素，再从 docker 配置里找到"已知可用"的代理地址复用，
> 最后只给**真正需要出网的那一个组件**配代理。**每一步都在缩小范围，而不是乱试。**

### 4.2 Application 定义（写文件再 apply）

`manifests/argocd-application-damo-app.yaml`：

```yaml
# ============================================================
# 项目 H · Argo CD Application：把 damo-app 的 Helm Chart 交给 GitOps 管
#
# 设计要点（面试可讲）：
# ① source 用 Helm Chart 目录（path: chart/damo-app）——Argo CD 原生渲染 Helm，
#    项目 E 的手写 Chart 一行不用改；
# ② image.tag 通过 helm.parameters 注入，保留"镜像 tag 由构建号决定"的语义；
# ③ syncPolicy 开 automated + prune + selfHeal：
#    - prune：Git 里删掉的资源，集群里也删（否则 Git 就不是唯一事实来源）
#    - selfHeal：有人手动改集群 → 自动纠偏（★ 这是本项目的灵魂）
# ④ 不用 CreateNamespace（命名空间已存在，且开启需给 controller 补
#    namespaces create 权限——延续项目 F 的最小权限思路，能不给就不给）；
# ⑤ ignoreDifferences 忽略 /spec/replicas ——
#    ★ 解决"项目 G 的 HPA"与"GitOps selfHeal"抢副本数控制权的问题（见 6.5）。
# ============================================================
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: damo-app
  namespace: argocd                       # ★ Application 对象本身放在 argocd 命名空间
  labels:
    app.kubernetes.io/part-of: damo-app
  finalizers:
    # 删除 Application 时级联删除其管理的资源（不加则删 App 留资源，形成孤儿）
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/GXEL-xy/damo-app.git
    targetRevision: HEAD                  # 或写 main/master 明确指向分支
    path: chart/damo-app                  # ★ Helm Chart 所在目录
    helm:
      releaseName: damo-app               # 与项目 E 的 release 名一致 → 无缝接管
      valueFiles:
        # ★★★ 顺序即优先级：后加载的覆盖先加载的（同 helm -f 的语义）
        - values.yaml            # chart 默认值：HPA / NetworkPolicy 全关
        - values-prod.yaml       # ★ 生产开关：NetworkPolicy=true + autoscaling=true
      parameters:
        # ★ 镜像 tag 由这里注入（Git 里看到的数值就是当前部署的版本）
        - name: image.tag
          value: "19"

  destination:
    server: https://kubernetes.default.svc   # 集群内自身；多集群时换成别的 server
    namespace: damo-app

  syncPolicy:
    automated:
      prune: true        # Git 里删了 → 集群里也删
      selfHeal: true     # 集群漂移 → 自动纠偏（★ 演示核心）
    syncOptions:
      - ApplyOutOfSyncOnly=true   # 只 apply 有差异的资源，减少无谓 patch
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # ★ 让 GitOps 与 HPA 和平共处：副本数是运行时状态，不由 Git 管
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

```bash
kubectl apply -f manifests/argocd-application-damo-app.yaml

# 查看状态
kubectl -n argocd get applications
# 期望：
# NAME       SYNC STATUS   HEALTH STATUS
# damo-app   OutOfSync     Missing      ← 首次是 OutOfSync/Missing，正常（还没同步）

kubectl -n argocd get application damo-app -o jsonpath='{.status.sync.status}{"\n"}'
kubectl -n argocd get application damo-app -o jsonpath='{.status.health.status}{"\n"}'
```

### 4.2.1 ⚠️ 接管前必须知道：`valueFiles` 漏一个，加固就静默消失

这是开工前核对 Chart 时发现的一个**真实陷阱**，也是整个项目 H 最容易翻车、最没人提醒的地方。

**事实**：`chart/damo-app/values.yaml` 里两个开关都是**关**的：

```yaml
autoscaling:
  enabled: false        # HPA 不开
networkPolicy:
  enabled: false        # NetworkPolicy 不开
```

真正的开关在 **`values-prod.yaml`**（项目 F 建的、项目 G 追加的），里面写着 `networkPolicy.enabled: true` 和 `autoscaling.enabled: true`。
项目 F/G 部署时用的是 `helm upgrade ... -f chart/damo-app/values-prod.yaml`，所以集群里**现在是有 5 条 NetworkPolicy + 1 个 HPA 的**。

**如果 Application 里只写 `valueFiles: [values.yaml]` 会发生什么**：

| 渲染结果 | 集群现状 | `prune: true` 的行为 | 后果 |
|---------|---------|---------------------|------|
| 无 HPA | 有 1 个 HPA | **删除它** | 项目 G 的弹性伸缩能力消失 |
| 无 NetworkPolicy | 有 5 条策略 | **全部删除** | 服务回到裸奔（项目 F 的加固归零） |
| Deployment 带 `replicas: 2` | 副本数由 HPA 管 | 写回 2 | 副本数锁死，6.5 的"控制权冲突"演示直接没了 |

**最阴险的地方是：没有任何报错。** Application 会显示 `Synced` + `Healthy`，UI 一片绿色，
但集群已经从「加固 + 弹性」退化成「裸奔 + 固定 2 副本」。**绿色不等于正确。**

> **★ 根因（和项目 F §4.7.5 是同一个道理，只是被 GitOps 放大了）**：
> 「**开关不在 git 里，就等于它会悄悄消失。**」
> 项目 F 的版本是「下次升级时丢」；GitOps 的版本更狠——**接管那一刻就丢**，
> 而且是**一次性、批量、无声地把前两个项目的成果全部回滚**。
>
> **面试价值**：这题能直接区分「会敲 Argo CD 命令」和「真的理解声明式接管」——
> 「接管一个已有工作负载时，第一件事不是点 Sync，而是**把当前 release 的 values 完整复现到 Application 里**，
> 否则 diff 里那些'看起来是 Git 多出来的东西'，其实是你自己丢掉的配置。」
> 一句话：**接管前先 `helm get values`，别让模板默认值冒充生产配置。**

**落地做法**：`valueFiles` 按 Helm 的加载顺序写两行，后者覆盖前者：

```yaml
      valueFiles:
        - values.yaml          # 默认值（先加载）
        - values-prod.yaml     # ★ 生产开关（后加载，覆盖上面的 false）
```

> **同步前自检一句**（推荐养成习惯）：
> ```bash
> # 对比"Helm 实际生效的 values" 与 "Argo CD 要渲染的 values"，两者必须一致
> helm -n damo-app get values damo-app
> ```
> 若这里能看到 `autoscaling.enabled: true` / `networkPolicy.enabled: true`，
> 而你的 Application 里没有 `values-prod.yaml`，**先停下来改 Application，再同步**。

### 4.3 用 UI 观察「同步前的 diff」（★ 核心机制）

浏览器打开 `https://11.0.1.128:30400`：

```
1. 左侧 Applications → 看到 damo-app 卡片
   · SYNC STATUS: OutOfSync（黄色）
   · HEALTH: Missing（因为还没同步）
2. 点开卡片 → 顶部有 APP DIFF 按钮
3. ★ 关键：界面上会同时显示【Git 声明的期望状态】与【集群里的实际状态】并高亮差异
   · 右侧（Live/集群）里现有 Deployment/Service 是项目 E 手装的那份
   · 左侧（Desired/Git）是要拉平到的目标
4. 注意：Deployment 里 `replicas` 字段【不会】出现在 diff 里 ——
   因为 ignoreDifferences 把它排除了（这是 HPA 与 GitOps 共存的关键）
```

### 4.4 私有仓库场景（了解即可，本项目不需要）

| 认证方式 | 配置方式 | 适用 |
|---------|---------|------|
| **HTTPS + Token** | UI：Settings → Repositories → Connect repo，填 username + PAT（**不要填账号密码**） | 大多数私有仓库 |
| **SSH 私钥** | 填 SSH URL + 私钥（Argo CD 支持 `sshPrivateKey`） | 需要部署密钥的场景 |
| **GitHub App** | 配置 App ID + Installation ID + 私钥 | 组织级、权限最细 |
| **CLI 一次性** | `argocd repo add <url> --username u --password p` | 快速试验 |

> **安全提醒（呼应项目 F）**：仓库凭据同样要**最小权限**——GitHub 的 fine-grained token 只给 `Contents: Read` 就够（Argo CD 只需要读代码，永远不写）。**"部署工具只需要读权限"这条原则，对 Git 仓库和对集群都成立。**

---

## 阶段 5：首次同步与验证

### 5.1 触发同步

**方式一：UI 点击（推荐，演示友好）**

```
damo-app 卡片 → 顶部 SYNC 按钮 → 弹窗里勾选需要的资源 → 点 SYNCHRONIZE
（会看到实时进度：资源逐个变成绿色）
```

**方式二：kubectl 打注解（注意：这个注解是"刷新"，不是"同步"）**

```bash
# argocd.argoproj.io/refresh 只让 Application 立刻重新拉 Git 并重算 diff，
# 【不会】直接执行同步。但本项目开了 automated，
# 所以"重算后发现 OutOfSync"会立刻引发自动同步 —— 效果上等价于"马上同步"。
# ★ 想表达"直接执行同步"这个语义，应该用下面这行（sync 注解）：
kubectl -n argocd patch application damo-app --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
#   ↑ refresh：重新比对（hard = 连 repo-server 缓存一起清）
#   ↓ sync：指定要同步到的 revision，直接触发一次同步操作
# kubectl -n argocd patch application damo-app --type merge \
#   -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```

> **★ 一个容易说错的细节（面试会被抓）**：
> `refresh` 是**读**操作（重新比对），`sync` 是**写**操作（apply 到集群）。
> 在 `automated` 模式下两者看起来差不多（刷新后自愈会跟着同步），
> 但在**手动同步**的应用里，打 `refresh` 注解只会让状态从 Unknown 变成 OutOfSync，**集群不会有任何变化**。
> 分清这一点，说明你不是只会照抄命令。

**方式三：CLI**

```bash
argocd app sync damo-app --insecure
argocd app wait damo-app --insecure
```

### 5.2 五道验证（第 ④ 道是最容易被跳过、后果最重的一道）

```bash
# ① Application 状态变绿
kubectl -n argocd get applications
# 期望：damo-app   Synced   Healthy

# ② 集群里的资源归属变成了 Argo CD
kubectl -n damo-app get deploy damo-app -o jsonpath='{.metadata.labels}'; echo
# 期望：labels 里出现 app.kubernetes.io/instance=damo-app、argocd.argoproj.io/instance=damo-app

kubectl -n damo-app get deploy damo-app -o jsonpath='{.metadata.annotations}'; echo
# 期望：能看到 Argo CD 的相关注解

# ③ 业务正常（关键：GitOps 接管不能影响业务）
curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'
kubectl -n damo-app get pods -l app.kubernetes.io/name=damo-app

# ④ ★★ 基线对账：接管后 HPA 与 NetworkPolicy 必须【一个都不少】（最易漏、也最致命）
kubectl -n damo-app get hpa
kubectl -n damo-app get netpol
# 期望：hpa 里 damo-app 在列（TARGETS 有值）；netpol 仍是 5 条
# ★★ 若这里变空，说明 valueFiles 漏了 values-prod.yaml（见 4.2.1）——
#    此时 Application 依然显示 Synced/Healthy，【绿色不代表正确】
# ★ 顺手确认 Deployment 里没有 replicas 字段（副本数归 HPA 管）：
kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.replicas}'; echo
# 期望：能读出值（HPA 写入的当前值），但 Git 侧的清单里【不出现】该字段
# 对比方式：curl 原始渲染产物，确认里面没有 replicas:
curl -s https://raw.githubusercontent.com/GXEL-xy/damo-app/main/chart/damo-app/templates/deployment.yaml \
  | grep -n "replicas" | head -5
# 期望：只在 {{- if not .Values.autoscaling.enabled }} 守卫内出现

# ⑤ ★ 与 Helm 台账的关系（这里有个坑，见 5.3）
helm list -n damo-app
```

### 5.3 ⚠️ 一个必须提前知道的坑：Helm 台账会被"顶掉"

**现象**：GitOps 接管后，`helm list -n damo-app` 可能**看不到 release 了**，或者 release 状态异常。

**原因**：Argo CD 渲染 Helm Chart **不通过 helm CLI**，而是用 Go 版 Helm SDK 渲染出纯 YAML，然后直接 `kubectl apply` 这些 YAML——**它不创建 `sh.helm.release.v1.*` 那套 revision Secret**。所以：

| 维度 | 项目 E（Helm CLI 发布） | 项目 H（Argo CD 渲染 Helm） |
|------|----------------------|--------------------------|
| 渲染方式 | `helm template` 内部逻辑 | Helm SDK（同样是 Helm 的模板引擎） |
| 写入集群 | `kubectl apply` + **写 release Secret** | 直接 apply **纯 YAML** |
| 版本台账 | `sh.helm.release.v1.damo-app.vN` | **Argo CD 自己的 Sync 历史** |
| 回滚方式 | `helm rollback <revision>` | **`git revert` 或 Argo CD 的 History 回滚** |

> **★ 这个坑面试常被追问**：「Helm 和 Argo CD 都能渲染 Chart，但它们的**"记账"方式不同**——Helm CLI 会在集群里写 release Secret 作为版本台账，Argo CD 则把版本台账留在 Git（提交历史）和自身的 Sync 历史里。所以**切到 GitOps 之后，`helm rollback` 不再是回滚的正确入口，应该用 `git revert`**。
> 这也解释了为什么**不该让 Helm 和 Argo CD 同时写同一批资源**——两边都在争'谁来记账'。我的做法是让 Argo CD 独占这批资源的写权，Helm CLI 只用于**本地渲染验证**（`helm template`）和 Chart 的 lint。
> 如果确实需要保留 `helm list` 可见，可以让 Argo CD 以 `helm template | kubectl apply` 的方式……但那是把 GitOps 又降级回 push 脚本了，得不偿失。」

**正确的分工结论**：

| 用途 | 用什么 |
|------|--------|
| 渲染验证、本地调试 | `helm template` / `helm lint`（**只读，不写集群**） |
| 集群的实际变更 | **只由 Argo CD 执行** |
| 回滚 | **`git revert`** 或 Argo CD UI 的 History → Rollback |
| 查"现在集群里是什么" | `kubectl get -o yaml` 或 Argo CD UI 的 Live 视图 |

---

## 阶段 6：GitOps 三个核心演示（本项目的精华）

> 这三个演示是**整个项目 H 的面试价值所在**。即使时间紧，也至少完成 ① 和 ②。

### 6.1 演示①：改 Git → 自动同步（Git 是唯一事实来源）

**目标**：展示"我只要改 Git，集群自己会跟上"。

```bash
# 在本地克隆仓库（若还没有）
git clone https://github.com/GXEL-xy/damo-app.git
cd damo-app

# ★ 改什么才"看得见、又不会打架"？—— 选参数的三个条件：
#   ① 能被 kubectl 直接读到（可验证）；
#   ② 不在 HPA 管辖范围（replicas 被 ignoreDifferences 忽略了，改它【看不到】效果）；
#   ③ 改了不影响业务可用性。
#   ⛔ 不要改副本数：项目 G 的 HPA 管着它，而且 6.5 的 ignoreDifferences 会把它忽略掉，
#      改完 Argo CD 既不报错也不动作 —— 那是"看起来失败"，会白白浪费一次演示。
#   ✅ 推荐改 probes.readiness.periodSeconds（5 → 8）：探针参数不归任何控制器管，
#      改完 describe 里一眼能看到，且不影响服务可用。

vim chart/damo-app/values.yaml
# 找到：probes → readiness → periodSeconds: 5
# 改为：periodSeconds: 8

git add chart/damo-app/values.yaml
git commit -m "chore: bump readiness probe period for demo"
git push
```

```bash
# 观察 Argo CD 自动跟上（Poll 周期默认 3 分钟；想快就手动 refresh）
watch -n 10 'kubectl -n argocd get application damo-app -o custom-columns="NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,REV:.status.sync.revision"'
# 时间线：
#   T+0      SYNC: Synced
#   T+~30s   SYNC: OutOfSync   ← 检测到 Git 有新提交
#   T+~60s   SYNC: Synced      ← 自动 apply 完成
#   REV 列从旧的 commit sha 变成新的 sha

# 验证集群真的变了
kubectl -n damo-app describe deploy damo-app | grep -i "readiness\|period"
# 期望：Period 从 5 变成 8 秒
```

> **加速技巧**：默认轮询周期是 3 分钟，演示时太长。可以在 UI 里点 `REFRESH`（硬刷新）立即触发；或者改 `argocd-cm` 的 `timeout.reconciliation`（**不建议低于 60s**，会对 Git 产生较多请求）。**更"生产"的做法是配 Git webhook**——但你的集群是内网、GitHub 无法回调，所以**这个场景恰好又回到了 poll 与 webhook 的取舍**（面试时把这一点讲出来，能把项目 B 的叙事和项目 H 串起来）。

### 6.2 演示②：手动改集群 → 自动纠偏（self-heal，★ 最震撼的演示）

**目标**：展示"漂移会被自动抹掉"——这是 pull 模型相对 push 模型**无法用其他手段替代**的能力。

```bash
# 确认当前状态
kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.template.spec.containers[0].image}'; echo

# ★ 手动制造"漂移"：把镜像换成别的版本（模拟运维误操作/应急改配置）
kubectl -n damo-app set image deploy/damo-app \
  "damo-app=11.0.1.128:30000/damo-app:18"
# 立刻观察：Pod 会开始滚动更新到 tag 18

# 记录现象
kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
# 期望：...:18  ← 漂移已经产生

# ★ 不要动任何东西，等 Argo CD 发现并纠偏（最多 3 分钟；或 UI 点 REFRESH 加速）
watch -n 10 'kubectl -n damo-app get deploy damo-app -o jsonpath="{.spec.template.spec.containers[0].image}"; echo; kubectl -n argocd get application damo-app -o custom-columns="SYNC:.status.sync.status"'

# 时间线：
#   T+0      image: ...:18   SYNC: OutOfSync  ← 漂移被检测到
#   T+~60s   image: ...:19   SYNC: Synced     ← ★ 被自动改回 Git 里的值

# 确认最终状态
kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
# 期望：回到 ...:19 —— 漂移被抹平

curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'
# 期望：Build #19（回到 Git 声明的版本）
```

> **★ 这个演示的意义（一定要在面试里讲透）**：
> 「push 模型下，**有人手动改了集群，没有任何机制会告诉你**——直到下一次发布才可能被覆盖，甚至可能永远留着（因为这个字段恰好不在发布的范围里）。监控只能告诉你"有人改了"，但不会自动恢复。
> **GitOps 的 selfHeal 把这个能力变成了默认行为**：集群状态只要偏离 Git 声明，就会在轮询周期内被拉回来。**它把"配置漂移"从一个需要人盯的运维问题，变成了一个被自动维持的不变量。**」

**如果想演示「不纠偏」的对照组**（很有说服力）：

```bash
# 临时关掉 selfHeal，重复上面的漂移操作
kubectl -n argocd patch application damo-app --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"selfHeal":false}}}}'

kubectl -n damo-app set image deploy/damo-app "damo-app=11.0.1.128:30000/damo-app:18"
sleep 180
kubectl -n damo-app get deploy damo-app -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
# 期望：仍然是 :18 —— 漂移【保持】了（因为 selfHeal 关了）
# 但 SYNC 会一直是 OutOfSync —— 说明 Argo CD 知道，只是不动手

# 恢复 selfHeal（然后看着它把漂移抹掉）
kubectl -n argocd patch application damo-app --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"selfHeal":true}}}}'
```

### 6.3 演示③：回滚 = `git revert`（回滚语义统一到 Git）

```bash
# 场景：刚才那次"探针 period 5→8"的改动要撤销
cd damo-app

# 查看历史
git log --oneline -5
# 找到那次 commit，例如 a1b2c3d chore: bump readiness probe period for demo

# 方式一：revert（生成一个"反向"的新提交，历史只增不减，可审计）
git revert a1b2c3d --no-edit
git push
# 等 Argo CD 同步 → 集群自动回到 period=5

# 方式二：用 Argo CD 的历史回滚（UI）
#   Application 详情 → HISTORY AND ROLLBACK → 选中某个历史 revision → ROLLBACK
#   注意：这会把集群拉回那个 Git revision 对应的状态，但【不会改 Git】——
#         所以下一轮协调会又把它拉回 Git 的 HEAD（除非你同时改了 Git）
#   ★ 这个细节很关键：GitOps 里"回滚"的权威动作永远是改 Git
```

> **★ 面试必答**：「GitOps 里回滚怎么做？」
> **答**：「**`git revert`**。因为 Git 是唯一事实来源，回滚就是"提交一个把状态改回旧版的新提交"。Argo CD 的 UI 里有 History 和 Rollback 按钮，**但它的作用是应急止血**（快速把集群拉回某个历史状态），**不是权威回滚动作**——因为它不改 Git，下一轮协调会把你拉回来。**权威回滚必须落在 Git 上**，这样才保证了'任何时刻 Git 都能解释集群为什么是这个样子'。
> 对比 Helm 的 `helm rollback`：那是把集群拉回某个 revision，**Git 里没有任何记录**——过一段时间就没人知道做过这次回滚了。」

### 6.4 三句总结（背下来）

| 演示 | 一句话总结 | 对应的 GitOps 原则 |
|------|-----------|------------------|
| ① 改 Git 自动同步 | **"我只改 Git，集群自己对上了"** | 自动拉取（Pulled Automatically） |
| ② 手改被纠偏 | **"人为的漂移撑不过一个协调周期"** | 持续协调（Continuously Reconciled） |
| ③ `git revert` 回滚 | **"回滚就是一个提交，历史只增不减"** | 版本化且不可变 |

### 6.5 ★ 进阶：HPA 与 GitOps 的控制权冲突（本项目最高阶的一个点）

这个冲突是**项目 G 与项目 H 的交界处**，能讲清楚它，说明你的知识是成体系的而不是孤立的。

**冲突是什么**：项目 G 让 **HPA** 管理 `Deployment.spec.replicas`（因为是运行时状态）；而 GitOps 的 `selfHeal` 会试图让集群等于 **Git 里的声明**（Git 里没有 replicas，或有一个固定值）。结果两者**抢同一个字段的控制权**：

```
   HPA:      "流量涨了 → replicas 应该 6"
   Argo CD:  "Git 里没写 replicas / 写的是 2 → 拉回 2"
                    ↑ 每 15s 一轮              ↑ 每 3min 一轮
   → 现象：副本数反复横跳、Application 长期 OutOfSync
```

**三种解法与取舍**：

| 解法 | 怎么做 | 取舍 | 本项目 |
|------|-------|------|--------|
| **① ignoreDifferences**（推荐） | 在 Application 里加 `ignoreDifferences: /spec/replicas`（已在 4.2 的 YAML 里写好） | 简单明确；**让 GitOps 显式声明"这个字段不归我管"** | ✅ 采用 |
| **② `RespectIgnoreDifferences=true` 同步选项** | 配合 ① 使用，保证同步时也不重置该字段 | 更严格；建议与 ① 一起加 | ✅ 已在 syncOptions 里留位 |
| **③ 不用 HPA，副本数完全由 Git 管** | 关掉 `autoscaling.enabled`，副本数进 values.yaml | 简单但**放弃弹性**——流量涨了只能靠人工改 Git（改 Git 也是一次提交，比 HPA 慢得多） | ❌ 不采用（项目 G 的价值不能丢） |

> **★ 一句话总结（面试金句）**：
> 「**GitOps 管"应该是什么"，HPA 管"此刻需要几个"。** 副本数是**运行时状态**而不是**期望状态**，所以它天然不该由 Git 管——`ignoreDifferences` 就是在 Application 里**显式地把这块控制权让给 HPA**。
> **这个冲突的本质是"声明式系统遇到运行时自治"**——把边界划清楚，两个系统就能共存；划不清楚，就会出现两个控制器互相打架的经典问题。」

---

## 阶段 7：与 Jenkins 分工——CI/CD 的最终形态

> **本阶段是"必做"部分的第二块**（即使不做实操也要能讲）。它把项目 B（Jenkins）和项目 H（Argo CD）缝合起来，形成完整的交付链路。

### 7.1 目标架构：CI 与 CD 的清晰分界

```
                                    ┌─────────────────────────────┐
 开发者 push 代码                    │  GitHub（唯一事实来源）       │
   │                                │   · 应用源码 index.html      │
   │                                │   · Dockerfile              │
   ▼                                │   · chart/damo-app/（声明）  │
 ① 代码提交 ──────────────────────► │   · Jenkinsfile             │
                                    └──────┬──────────────┬───────┘
                                           │              │
                       ② 轮询到变更（pull）  │              │ ③ Argo CD 拉取声明（pull）
                                           ▼              ▼
                            ┌──────────────────┐   ┌──────────────────────┐
                            │ Jenkins（CI 域）  │   │ Argo CD（CD 域）      │
                            │  · 构建镜像       │   │  · 渲染 Helm Chart    │
                            │  · 单元测试        │   │  · 计算 diff          │
                            │  · 推送到私有仓库  │   │  · 持续协调 / selfHeal│
                            │  ★ 不再 deploy！   │   │  ★ 唯一的写入者       │
                            └────────┬─────────┘   └──────────┬───────────┘
                                     │ ④ push 镜像             │ ⑤ apply
                                     ▼                        ▼
                            ┌──────────────────┐   ┌──────────────────────┐
                            │ registry:2       │   │ 集群实际状态          │
                            │ 11.0.1.128:30000 │   │  damo-app :30090     │
                            └──────────────────┘   └──────────────────────┘
                                     ▲
                                     └── ⑥ kubelet 从私有仓库拉取镜像
```

**关键原则：每个资源只允许一个"写入者"。**

| 资源 | 唯一写入者 | 说明 |
|------|-----------|------|
| 镜像（registry 里的 tag） | **Jenkins** | CI 的产出 |
| 集群里的 K8s 资源 | **Argo CD** | CD 的产出。Jenkins **不再**执行 kubectl/helm apply |
| Git 里的 chart/values | 开发者（人或 Jenkins 的自动提交） | 见 7.2 |
| `replicas` 字段 | **HPA**（项目 G） | 通过 `ignoreDifferences` 让给 HPA |

> **★ 为什么"两个都写"是危险的**：如果 Jenkins 既推镜像又 `helm upgrade`，Argo CD 也在同步同一个 Deployment，**会出现两者互相覆盖、状态永远收敛不了的"神仙打架"**。所以本项目明确：**Jenkins 的 Stage 4/5 被移除（或改为只更新 Git），部署完全交给 Argo CD。**
>
> **这个"拆除"动作本身就是很好的面试材料**：「我从项目 B 一路演进到 H，最后**主动删掉了 Jenkinsfile 里的部署 Stage**——不是因为它坏了，而是因为**部署的职责应该收敛到一个持续协调的控制器上**，而不是散落在一次性的构建脚本里。」

### 7.2 镜像 tag 怎么从 Jenkins 传到 Argo CD（两种方案对比）

这是 CI/CD 分工里**唯一需要设计的地方**——镜像 tag 是动态的（构建号），而 Git 是静态声明，怎么把动态值送进去？

| 方案 | 做法 | 优点 | 缺点 | 本项目 |
|------|------|------|------|--------|
| **A. Jenkins 回写 Git**（推荐起步） | Jenkins 构建成功后，用脚本改 `chart/damo-app/values.yaml`（或 `argocd-application.yaml`）里的 `image.tag`，然后 `git commit && git push`；Argo CD 拉到新提交后自动同步 | 简单直观；**Git 里能看到"谁在什么时候部署了什么"**；不需要新组件 | Jenkins 要有 Git 写权限（**新增一个凭据**）；会产生"机器人提交"，需要约定 commit message 规范避免触发死循环 | ✅ 先讲这个 |
| **B. Argo CD Image Updater** | 部署 Image Updater 组件，它主动查 registry 有没有新 tag，符合策略就**自动回写 Git** | Jenkins **不需要 Git 写权限**；CI 与 CD 完全解耦 | **新增一个组件**；需要配 registry 凭据；策略配置有学习成本 | 🔜 进阶（了解即可） |
| **C. 不传 tag，用 `latest` + `Always`** | 让 Argo CD 只声明"用 latest" | 最简单 | ❌ **反模式**：Git 无法说明"当前部署的是哪个版本"；`latest` 会被覆盖导致不可复现；**违背 GitOps"版本化且不可变"原则** | ❌ 明确不做 |

**方案 A 的 Jenkinsfile 改造（把部署 Stage 换成"回写 Git"）**：

```groovy
// 项目 B/E 的 Stage 4/5 被替换为 ↓
stage('4. Update image tag in Git') {
    steps {
        container('helm') {
            withCredentials([usernamePassword(
                credentialsId: 'git-github',
                usernameVariable: 'GIT_USER',
                passwordVariable: 'GIT_TOKEN'
            )]) {
                sh """
                    set -e
                    git config user.email "jenkins@ci.local"
                    git config user.name  "jenkins-ci"

                    # 只改声明里的镜像 tag（单点修改，最小差异）
                    sed -i "s|^  tag: .*|  tag: \\"${IMAGE_TAG}\\"|" chart/damo-app/values.yaml
                    # 若用 Application 的 helm.parameters 注入，则改 manifests/argocd-application.yaml

                    git add chart/damo-app/values.yaml
                    git commit -m "ci: bump image.tag to ${IMAGE_TAG} [skip ci]" || echo "no change"
                    git push origin HEAD:main
                """
            }
        }
    }
}

// 说明：
//  · commit message 里的 [skip ci] 是约定，避免"机器人提交"再次触发流水线形成死循环
//  · 这里【不再】执行任何 kubectl/helm apply —— 部署由 Argo CD 完成
//  · 镜像 push 仍在 Stage 3（kaniko），保持不变
```

> ⚠️ **"机器人提交触发死循环"是这个方案的经典坑**：Jenkins 提交 → poll SCM 检测到变更 → 再次触发构建 → 又提交…… 三种防法：① commit message 里带 `[skip ci]`，并在 Jenkinsfile 开头判断（`if (env.GIT_COMMIT_MSG.contains('[skip ci]')) { currentBuild.result='NOT_BUILT'; return }`）；② 用 `pollSCM` 的“忽略某些提交”能力（需要插件）；③ 直接改成方案 B（Image Updater）。

### 7.3 改造后的完整链路（一句话版本）

```
开发者 push 代码
  → Jenkins 轮询到 → kaniko 构建镜像 → 推到私有仓库
  → Jenkins 把新 tag 回写到 Git（一个 commit）
  → Argo CD 轮询到 Git 变化 → 渲染 Chart → 计算 diff
  → 自动 apply → 滚动更新 → 页面显示新 Build 号
  → 全过程零人工干预，且"为什么是现在这个版本"在 Git 里有完整答案
```

### 7.3.1 ★ 实测记录：从 #47 失败到 #48 成功（这次改造的真实过程）

**改造后 Jenkinsfile 的关键差异**（`git show 355a4d0` 可复核）：

| Stage | 项目 E/F 版本 | 项目 H 版本 |
|---|---|---|
| 1-3 | Checkout / Render / Kaniko Build | **完全不变** |
| 4 | `helm upgrade --install ... --atomic` | `sed` 改 `values-prod.yaml` 的 tag → `git commit` → `git push` |
| 5 | `kubectl get pods` + `helm history` | 只打印交接说明 |
| 凭据 | `kubeconfig-damo-deployer`（集群写权限） | **`github-damo-app`（只能写一个仓库的 Contents）** |

**★ 安全收益（这次改造最大的隐性价值）**：Jenkins 的 PodTemplate 里原本挂着一个 `kubeconfig` Secret（`volume-0`），意味着**任何能改 Jenkinsfile 的人都能对集群做任何事**。改造后 Stage 5 不再有任何 `kubectl`/`helm` 调用，这个凭据可以彻底删掉——**攻击面从"整台集群"缩小到"一个仓库的一个文件"**。这是项目 F「最小权限」思路在 CI 侧的延续，而且是那次改造时没预见到的额外收益。

**构建 #47：失败在最后一步**

```
+ git push https://GXEL-xy:****@github.com/GXEL-xy/damo-app.git HEAD:main
fatal: unable to access 'https://github.com/GXEL-xy/damo-app.git/':
  Recv failure: Connection reset by peer
ERROR: script returned exit code 128
```

诡异之处：**同一个构建里 `git fetch` 是成功的**，日志里还明明写着 `Setting http proxy: 11.0.1.1:7890`。

**根因**：那行 `Setting http proxy` 是 **Jenkins 的 Git 插件在它自己执行的 fetch 命令上注入的**（等价于 `git -c http.proxy=...`），它**不会变成容器里的环境变量**。所以插件自己的 fetch 走代理、成功；而我们在 `sh` 里手写的 `git push` 是裸命令，不继承任何代理设置 → 直连 GitHub → 被重置。

**修法**（commit `7b06b11`）：

```sh
export HTTP_PROXY=http://11.0.1.1:7890
export HTTPS_PROXY=http://11.0.1.1:7890
export NO_PROXY=localhost,127.0.0.1,11.0.1.128,10.244.0.0/16
```

> **★ 与 §4.1.1 是同一类故障**：报错都是 `Connection reset by peer`（**主动重置**，不是 `timeout` 那种静默丢包）。这两个字眼的区别，就是"有人在 TLS 层干预"和"包没到"的分水岭。三次踩到（repo-server、Jenkins push、更早的 docker pull）本质都是**同一件事：这台机器出外网必须走 `11.0.1.1:7890`，而每一层容器/进程的代理配置都是独立的。**

**构建 #48：全链路打通**

```
---- 改动前 ----   39:  tag: "46"
---- 改动后 ----   39:  tag: "48"
已回写 Git，等待 Argo CD 拉取
CI 完成：镜像 48 已推送，tag 已回写 Git
Finished: SUCCESS
```

Git 侧多了一个提交 `eafa770 ci: bump image.tag to 48 [skip ci]`，然后：

| 验证项 | 结果 |
|---|---|
| Argo CD `REV` | `eafa7709aa...`（正是那条回写提交） |
| Argo CD `SYNC` | `Synced` |
| Deployment 镜像 | `11.0.1.128:30000/damo-app:48` |
| 业务页面 | `Build #48`（两个 worker 节点都是） |
| Jenkins 是否碰过集群 | **否**（Stage 5 无任何 kubectl/helm） |

**死循环防护验证**：

```
#48  SUCCESS   13:13:33
#47  FAILURE   13:10:06
#46  SUCCESS   01:02:11     ← 一个月前
```

从 #48 完成到复查隔了 25 分钟（poll 每 5 分钟一轮，跑了 5 轮），**构建号没有再增长**——说明「忽略 `[skip ci]` 提交」真的生效了，机器人提交不会自己触发自己。

> **★ 一个需要知道的警告**：日志里有
> `DEPRECATED: Message exclusion during polling is deprecated for Pipeline jobs. Use the Pipeline SCM trait instead.`
> 这条配置（配在 Job 的 SCM → Additional Behaviours 里）在 **Pipeline job 上已标记为废弃**，本项目实测**仍然生效**，但未来版本可能失效。
> **更可靠的做法**是在 Jenkinsfile 里自己判断（不依赖插件配置）：读 `git log -1 --pretty=%B`，命中 `[skip ci]` 就中止。
> 取舍：插件配置**零代码**、但要理解"它可能在升级后失效"；Jenkinsfile 判断**代码可审计**、但要处理"如何优雅地终止 pipeline"（`error()` 会把构建标红，需要配合 `currentBuild.result='NOT_BUILT'` 或 `when` 条件）。**本项目先用插件配置，并在文档里记下这条退路。**

### 7.4 阶段 7 完成标准

- [x] 能画出 7.1 的架构图并讲清"每个资源只有一个写入者"
- [x] 能说清镜像 tag 传递的三种方案与取舍（A/B/C）
- [x] 能说清为什么 `latest` + `Always` 是反模式
- [x] 能说出"机器人提交死循环"的成因与至少一种防法
- [x] 能解释为什么**主动删掉 Jenkins 的部署 Stage** 是一个正确的演进而不是退步
- [x] ★ 能说出删掉部署 Stage 后**顺带得到的两个收益**：Jenkins 不再持有 kubeconfig（攻击面收敛）、构建失败不会污染运行环境
- [x] ★ 能解释"为什么插件 fetch 能过代理、我手写的 git push 不能"（注入的是命令行参数，不是环境变量）

---

## 8. 故障速查表

按「现象 → 原因 → 排查与解决」组织。**★ 标记的是面试高频考点**。

| 现象 | 原因 | 排查与解决 |
|------|------|-----------|
| **★ Application 报 `repository not found` / `failed to list refs`** | ① 集群**出网不通**（最常见）；② repoURL 写错；③ 私有仓库缺凭据 | **先在 repo-server Pod 里 `git ls-remote <url>` 最小复现**——通了就是 Argo CD 配置问题，不通就是网络问题（这一刀切分法最有价值） |
| **★★ `ComparisonError: failed to list refs ... context deadline exceeded` 或 `Recv failure: Connection reset by peer`** | **repo-server 出不了 HTTPS**。★ 注意 `nc -z github.com 443` 通**不代表** git 能用——TCP 握手成功后 TLS 被按 SNI 重置，是典型的中间设备阻断 | 给 repo-server 单独配代理（见 4.1.1）：`kubectl -n argocd set env deploy/argocd-repo-server HTTPS_PROXY=... NO_PROXY=...`。**只配 repo-server 就够**，server/controller 不走外网 |
| `kubectl exec` 进 repo-server 报 `exec: "curl": executable file not found` | 该镜像精简，没有 curl | 用 `git ls-remote <url>` 代替（镜像里必有 git），或用 `kubectl debug` 挂一个带工具的容器 |
| Application 一直 `OutOfSync`，但点 Sync 又说成功 | ① 集群里有 Argo CD 不认识的字段被反复抹掉（如被 webhook/controller 改写）；② **HPA 在改 replicas**（★ 本项目场景） | `ignoreDifferences` 忽略运行时字段（本项目的 `/spec/replicas`）；或用 `RespectIgnoreDifferences=true` |
| **★ 副本数反复横跳、SYNC 长期 OutOfSync** | **HPA 与 GitOps 抢 `replicas` 控制权** | 见 6.5：加 `ignoreDifferences: /spec/replicas`——**显式把这块控制权让给 HPA** |
| **★★ 同步后 HPA / NetworkPolicy 集体消失，但 Application 显示 `Synced` + `Healthy`** | `helm.valueFiles` 只写了 `values.yaml`，漏了 **`values-prod.yaml`** → `autoscaling.enabled` / `networkPolicy.enabled` 回落 `false`，`prune: true` 把项目 F/G 建的 6 个对象全删了（**无任何报错**） | 见 4.2.1。`helm -n damo-app get values damo-app` 是接管前的基准；补上 `values-prod.yaml` 后重新 sync 即可重建 |
| Pod `ImagePullBackOff`，镜像 tag 不存在 | Application 里 `helm.parameters` 的 `image.tag` 指向了仓库里没有的 tag | `curl http://11.0.1.128:30000/v2/damo-app/tags/list` 核对；改 `image.tag` 后重新 sync |
| **★★ `apply install.yaml` 报 `CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long`** | client-side apply 把整个对象塞进 `last-applied-configuration` 注解，而该 CRD 的 schema 序列化后 >256KB | 改用 `kubectl apply --server-side=true --force-conflicts -n argocd -f install.yaml`（SSA 不写该注解）。见 3.1。**其余对象其实已创建成功，重跑幂等** |
| 组件 Pod `ImagePullBackOff`（Argo CD 自身） | 三个镜像没中转全，或 sed 替换漏了一处 | `grep -E '^\s+image:' install.yaml \| sort -u` 逐个核对；`grep -c -E 'quay\.io\|public\.ecr\.aws\|ghcr\.io' install.yaml` 应为 0 |
| `argocd-server` 的 NodePort 连不上 | ★ 把 `targetPort` 写成了 443（实际容器只暴露 8080） | 改回 `targetPort: 8080`；`kubectl -n argocd get svc argocd-server -o yaml` 核对 |
| UI 打不开、浏览器报证书错误 | Argo CD 默认用自签证书 | 正常现象，点"高级 → 继续访问"，或 curl 时加 `-k`。生产应配正规证书或放在反代后面 |
| 忘记 admin 密码 | 已按 3.4 删除了 `argocd-initial-admin-secret` | `argocd account update-password`（需已登录），或用 `kubectl -n argocd patch secret argocd-secret` 重置 bcrypt 哈希（**注意：是 bcrypt 哈希，不是明文**） |
| 删除了 Application，集群里的资源还在 | Application 缺 `resources-finalizer.argocd.argoproj.io` finalizer | 删除 Application 时要加 finalizer 才会级联删除。**⚠️ 生产慎用级联删除**——先确认这些资源确实不该留 |
| Git 里删了资源，集群里还在 | `prune` 没开（默认 `false`） | 在 `syncPolicy.automated` 里加 `prune: true`；单次同步时也可勾选 "Prune" |
| 改 Git 后很久没同步（>3 分钟） | 轮询周期是 3 分钟；或 repo-server 缓存 | UI 点 `REFRESH`（硬刷新）；生产配 Git webhook 立即触发。**注意：本项目内网环境 GitHub 无法回调，所以轮询是合理的妥协**（与项目 B 的 poll SCM 同源原因） |
| Jenkins 回写 Git 后触发死循环构建 | 机器人提交又触发 poll SCM | commit message 加 `[skip ci]` + 配「忽略某些提交」；或改用 Image Updater（7.2 方案 B）。**实测**：Job SCM 的 Additional Behaviours 里配 `Polling ignores commits with certain messages = .*\[skip ci\].*` 可用，但日志会警告 `DEPRECATED ... for Pipeline jobs`，升级后可能失效，退路见 7.3.1 |
| **★★ Jenkins 里手工 `git push` 报 `Recv failure: Connection reset by peer`，但同一构建的 `git fetch` 成功** | **插件注入的代理只作用于它自己的命令**：日志里的 `Setting http proxy: 11.0.1.1:7890` 是 Git 插件在 fetch 上加了 `-c http.proxy=`，**不会成为容器环境变量**，手写命令不继承 | 在 `sh` 里显式 `export HTTP_PROXY/HTTPS_PROXY/NO_PROXY`。见 7.3.1。★ 与 4.1.1 同源：`reset` = 主动干预，`timeout` = 没到 |
| `helm list` 里看不到 release 了 | Argo CD 用 Helm SDK 渲染 + 直接 apply 纯 YAML，**不写 release Secret** | 这是预期行为（见 5.3）。**回滚改用 `git revert`**；Helm CLI 只用于本地 `template`/`lint` |
| 改动 `values.yaml` 后 sync 了但集群没变 | ① Argo CD 缓存未刷新；② 改的参数没被模板引用；③ 改了 `image.tag` 但 `pullPolicy` 是 `IfNotPresent` 导致节点用旧镜像 | ① UI 硬刷新；② `helm template` 本地验证该参数确实渲染进产物；③ 项目 E 的 `values.yaml` 用的是 `Always`，一般不会有这个问题 |
| Argo CD 与 Helm CLI 同时改同一批资源，状态混乱 | 两个"写入者"在争同一份资源 | **确立单一写入者**：集群变更只由 Argo CD 执行；Helm CLI 降级为本地渲染验证工具（见 5.3） |
| `argocd-application-controller` 一直不是 Running | 它是 **StatefulSet** 不是 Deployment，可能被误查；或资源不足 | `kubectl -n argocd get statefulset`；`kubectl -n argocd describe pod argocd-application-controller-0` 看 Events |
| 集群资源紧张，Argo CD 起来后业务变慢 | Argo CD 占用约 500m CPU / 1Gi 内存 | 给 `argocd-repo-server` 降 `requests`；或临时把 `argocd-dex-server` 缩到 0（不用 SSO 时） |
| Application 的 `dest-namespace` 不存在导致同步失败 | 没用 `CreateNamespace=true` 但目标 ns 不存在 | 手工先 `kubectl create ns`；或给 controller 补 `namespaces` create 权限后开启该选项（**与项目 F 的最小权限取舍要权衡**） |

---

## 9. 自测题（含答案）

**Q1. push 模型和 pull 模型最本质的区别是什么？**
A：**push 是"我把变更推给你"，pull 是"我来对齐你的声明"**。具体差异有三层：① **凭据位置**——push 把集群凭据放在 CI 系统里（外扩），pull 用集群内 SA（不出集群）；② **触发方式**——push 依赖事件（提交触发），pull 是**持续协调**（不依赖事件）；③ **网络要求**——push 需要"外部能推送到集群"，pull 只需"集群能出网拉 Git"，**后者在内网环境容易得多**。所以 poll SCM 本质上是 push 模型在网络受限下的妥协，而 pull 是绕开这个限制的架构选择。

**Q2. GitOps 的四条原则是什么？为什么"把 YAML 存 Git"不算 GitOps？**
A：声明式、版本化且不可变、**自动拉取**、**持续协调**。因为 GitOps 的分水岭在第 3、4 条：如果只是把 YAML 放 Git，然后由 Jenkins 在发布时 `kubectl apply`，那仍然是"事件触发的一次性动作"，**没有"代理主动拉取"，也没有"持续纠偏"**——只是把配置文件挪了个位置，本质还是 push 模型。

**Q3. Argo CD 有哪些核心组件？哪个是核心？UI 挂了会影响同步吗？**
A：`application-controller`（**核心**，执行协调循环、算 diff、apply）、`repo-server`（拉 Git、渲染 Helm/Kustomize）、`server`（UI/API，门面）、`redis`（缓存）、`dex`（SSO，可选）。**UI（server）挂了不影响自动同步**——真正干活的是 controller。这个区分说明我理解控制面组件的职责边界，而不是把它当黑盒。

**Q4. GitOps 与 Helm 是什么关系？会互相替代吗？**
A：**完全正交，且叠加使用**。Helm 解决「**怎么把一份模板渲染成多环境的清单**」（模板 + 参数 + 渲染），GitOps 解决「**谁在什么时候把渲染结果同步到集群并能持续纠偏**」（协调 + 漂移检测）。本项目里 Argo CD **调用 Helm 引擎**渲染 Chart，然后同步——所以项目 E 的手写 Chart 一行没改就被 GitOps 接管。

**Q5. Argo CD 渲染 Helm Chart 与 `helm install` 有什么不同？**
A：**渲染结果相同，但"记账方式"不同**。`helm install/upgrade` 会写 `sh.helm.release.v1.*` Secret 作为版本台账，支持 `helm rollback`；Argo CD 用 Helm SDK 渲染成**纯 YAML** 直接 apply，**不写 release Secret**，版本台账留在 **Git 提交历史**和 Argo CD 自己的 Sync 历史里。**推论：切到 GitOps 后回滚要用 `git revert` 而不是 `helm rollback`；并且不该让两者同时写同一批资源**（争"谁来记账"）。

**Q6. 为什么说"选 Helm 还是 Kustomize"和"选 Jenkins 还是 Argo CD"是两个不同维度的问题？**
A：前者是**渲染层**（模板参数化 vs overlay 差异化），后者是**交付层**（push 一次性动作 vs pull 持续协调）。两者可以任意组合：Helm + Argo CD、Kustomize + Argo CD、Helm + Jenkins 都成立。把不同层的问题混在一起思考，是初学者最常见的概念混乱。

**Q7. `selfHeal` 和 `prune` 分别解决什么问题？**
A：`selfHeal`：**集群状态偏离 Git 声明时自动纠偏**——把"配置漂移"从需要人盯的运维问题变成被自动维持的不变量。`prune`：**Git 里删除的资源，集群里也删掉**——否则 Git 里已经没了、集群里还在，**"Git 是唯一事实来源"就成了空话**。两者都需要，缺一个都会破坏 GitOps 的不变量语义。

**Q8. GitOps 里回滚怎么做？为什么 Argo CD 的 Rollback 按钮不是权威动作？**
A：**权威动作是 `git revert`**——提交一个把状态改回旧版的新提交，历史只增不减、可审计。Argo CD 的 History → Rollback 是**应急止血**（快速把集群拉回某个历史状态），**因为它不改 Git**，下一轮协调就会把集群拉回 Git 的 HEAD。对比 Helm 的 `helm rollback`：它改集群但不留 Git 记录，过一阵就没人知道做过这次回滚了。

**Q9. GitOps 和 HPA 会冲突吗？为什么？怎么解决？**
A：**会，冲突在 `Deployment.spec.replicas` 这一个字段上**。HPA 认为它是**运行时状态**（随负载变），GitOps 的 selfHeal 认为它应该等于 **Git 里的声明**。两者同时控制会导致副本数反复横跳、Application 长期 OutOfSync。解法是 `ignoreDifferences: /spec/replicas` —— **在 Application 里显式声明"这个字段不归我管"，把控制权让给 HPA**。本质是「**GitOps 管"应该是什么"，HPA 管"此刻需要几个"**」，两个声明式系统必须把字段边界划清楚。

**Q10. `latest` + `imagePullPolicy: Always` 为什么是 GitOps 的反模式？**
A：因为违背了 GitOps 的第二条原则「**版本化且不可变**」。如果 Git 里只写 `latest`，那么：① **Git 无法回答"当前部署的是哪个版本"**——唯一事实来源解释不了集群状态；② 上游重新推同一个 `latest` 会导致**部署不可复现**；③ 回滚无从下手（没有可回退的确定版本）。正确做法是**用不可变的唯一 tag**（如构建号），让 Git 里的每一个提交都精确对应一个可复现的部署。

**Q11. Jenkins 和 Argo CD 如何分工？为什么"两个都部署"很危险？**
A：**Jenkins 做 CI**（构建、测试、推镜像，有状态、有中间产物），**Argo CD 做 CD**（部署、持续协调、漂移纠偏，声明式、无中间产物）。**"两个都部署"危险在于两个控制器争同一批资源的写权**——会互相覆盖、状态永远收敛不了。原则是「**每个资源只允许一个写入者**」：镜像由 Jenkins 写，集群资源由 Argo CD 写，`replicas` 由 HPA 写。

**Q12. 镜像 tag 怎么从 Jenkins 传到 Argo CD？三种方案与取舍？**
A：① **Jenkins 回写 Git**（推荐起步）：构建后改 `values.yaml` 的 `image.tag` 并提交，Argo CD 拉到后自动同步。优点是简单、Git 里有完整部署记录；缺点是 Jenkins 需要 Git 写权限（新增凭据），且要防"机器人提交"死循环（`[skip ci]` 约定）。② **Argo CD Image Updater**：新组件主动查 registry 并回写 Git，CI 与 CD 完全解耦，但多一个组件要维护。③ **`latest` + Always**：❌ 反模式，违背不可变原则。

**Q13. 你的集群是内网、GitHub 无法回调，为什么 Argo CD 还能用？**
A：因为 **pull 模型的网络要求是"集群能出网拉 Git"，不是"Git 能推送到集群"**。方向和 push 模型相反——内网 VM 能访问 GitHub（项目 B 的 Jenkins 一直这么做），所以 Argo CD 完全可用。**这正是 pull 模型对内网友好的根本原因**，也是我说"poll SCM 是 push 模型在网络受限下的妥协，而 GitOps 是绕开这个限制的架构选择"的由来。

**Q14. 如果 Git 里的 Chart 写错了（比如引用了不存在的镜像 tag），会发生什么？**
A：Argo CD 会**尝试同步并失败**（Pod `ImagePullBackOff`），于是显示 `OutOfSync` + `Degraded`/`Progressing`，并且**按 `syncPolicy.retry` 的策略反复重试**（本项目配了 limit 5 次、指数退避）。此时集群会处于"半更新"的中间态。**防护措施**：① 项目 E 的 `helm lint` + `helm template` 在 CI 里先验证；② Argo CD 有 `--dry-run` 能力的 Sync Windows / PreSync hooks（进阶）；③ 更彻底的是**在 CI 里用 `kubectl apply --dry-run=server` 或 kubeconform 校验渲染产物**，把错误挡在 Git 之前——**这是"流水线左移"在 GitOps 场景下的对应做法**。

**Q15. 为什么本项目不装 ApplicationSet / Argo Rollouts / Image Updater？**
A：因为它们各自解决**不同的问题**，而我这一版的目标是**跑通主链路并讲清原理**：`ApplicationSet` 解决"几百个 Application 的批量生成"，`Argo Rollouts` 解决"金丝雀/蓝绿发布策略"，`Image Updater` 解决"自动回写 tag"。**在只有 1 个应用、集群资源有限的场景下引入它们，是复杂度而非能力**——增加了要维护的组件、要理解的抽象，却换不来实际收益。**我选择"知道它们是什么、为什么这一版不做"，而不是为了简历堆组件。** 这个取舍本身就体现判断力。

---

## 10. 面试要点与简历写法

### 11.1 一段话讲完本项目

> 「我把交付模型从**push 升级成了 pull**。原来（项目 B–G）是 Jenkins 持有集群凭据、构建完直接 `helm upgrade` 推到集群——这条路有三个问题：凭据在 CI 系统里外扩、只有变更时才动作所以**手动改集群没人知道**、内网收不到 webhook 只能轮询。
> 项目 H 我在集群里部署了 Argo CD，让它做**持续协调**：期望状态由 Git 里的 Helm Chart 声明（项目 E 的手写 Chart 一行没改就被接管，因为 Argo CD 原生渲染 Helm），Argo CD 每 3 分钟拉一次 Git、和集群实际状态做 diff，不一致就自动拉平。
> 我做了三个演示：① 改 Git 后集群自动同步；② **手动把镜像改成旧版本，Argo CD 在下一轮协调里自己改了回来**——这是 push 模型无法用其他手段替代的能力，它把"配置漂移"从运维问题变成了被自动维持的不变量；③ 回滚用 `git revert`，历史只增不减。
> 过程中处理了两个有价值的边界问题：一是 **Helm 与 GitOps 的"记账权"冲突**——Argo CD 用 Helm SDK 渲染纯 YAML、不写 release Secret，所以切过来之后 `helm rollback` 不再是回滚入口，Helm CLI 降级为本地渲染验证工具，集群变更的唯一写入者是 Argo CD；二是 **HPA 与 GitOps 抢 `replicas` 控制权**，我用 `ignoreDifferences` 显式把这个字段让给 HPA——本质是划清"GitOps 管应该是什么、HPA 管此刻需要几个"的边界。
> 最后我把 Jenkinsfile 里的部署 Stage **主动删掉了**：不是因为它坏了，而是部署职责应该收敛到一个持续协调的控制器上，而不是散落在一次性的构建脚本里。Jenkins 只做 CI（构建推镜像 + 回写镜像 tag 到 Git），CD 全部交给 Argo CD。」

### 11.2 高频追问预案

| 追问 | 怎么回答 |
|------|---------------------|
| GitOps 和"用 Git 管配置"有什么区别？ | Q2：分水岭是「自动拉取 + 持续协调」两条原则 |
| 你们内网环境怎么用 GitOps？ | Q13：pull 要求的是"集群能出网"，方向与 push 相反 |
| Argo CD 和 Jenkins 会冲突吗？ | Q11：**每个资源只能有一个写入者**——所以我删掉了 Jenkins 的部署 Stage |
| GitOps 里回滚怎么做？ | Q8：`git revert` 是权威动作；Argo CD 的 Rollback 只是应急止血 |
| Helm 和 Argo CD 有什么冲突？ | Q5：记账方式不同（release Secret vs Git 提交历史）；**不让两者同时写同一批资源** |
| GitOps 和 HPA 冲突吗？ | Q9：`ignoreDifferences /spec/replicas`，把运行时字段让给 HPA |
| `selfHeal` 怎么验证？ | **开 selfHeal 时漂移被自动纠正；关掉后漂移会保留** —— 有对照组才算证明机制 |
| 为什么不用 `latest`？ | Q10：违背不可变原则，Git 无法解释集群状态、不可复现、无法回滚 |
| 为什么不用 Flux？ | 两者都是 pull 模型；Argo CD 的 UI 与多集群管理更成熟，团队上手更友好；Flux 更纯粹 CRD 驱动 |
| 为什么不装 ApplicationSet / Rollouts？ | Q15：**在 1 个应用、资源有限的集群里引入它们是复杂度而非能力** |
| 你集群现在是什么状态？ | "HPA 管副本、Argo CD 管其余、Jenkins 只管构建" |
| 这个项目最难的地方是什么？ | ① **Helm 与 GitOps 的记账权冲突**（helm list 里 release 消失，一开始以为装错了）；② **HPA 与 selfHeal 抢 replicas**（现象是副本数横跳，本质是两个声明式系统的字段边界问题）——**这两个都是"两个机制在同一处重叠"的典型，讲它们比讲安装步骤有价值得多** |

### 11.3 简历写法（可直接用）

> **短版（1 行）**：
> 在集群部署 Argo CD 落地 GitOps 交付模型：以 Git 为唯一事实来源、Argo CD 持续协调，实现变更自动同步与集群漂移自动纠偏，回滚统一为 `git revert`；Jenkins 收敛为纯 CI（构建推镜像 + 回写 tag），确立「每个资源单一写入者」的分工边界。

> **长版（3 行，适合放项目详情）**：
> - **GitOps 落地**：部署 Argo CD（本地镜像仓库中转 argocd/redis/dex，NodePort 30400 暴露），以 Application CRD 接管既有 Helm Chart（`automated + prune + selfHeal`），跑通「改 Git 自动同步」「集群漂移自动纠偏」「`git revert` 回滚」三条主链路，并保留 self-heal 关闭的对照组验证纠偏机制真实生效。
> - **机制边界处理**：识别并解决两处控制权冲突——Helm CLI 与 Argo CD 的 release 记账权冲突（Argo CD 渲染纯 YAML 不写 release Secret，故回滚入口从 `helm rollback` 迁移为 `git revert`，Helm CLI 降级为渲染校验工具）；HPA 与 selfHeal 的 `spec.replicas` 争用（以 `ignoreDifferences` 将运行时字段控制权显式让给 HPA）。
> - **CI/CD 分工**：将 Jenkins 收敛为纯 CI（kaniko 构建 + 推私有仓库 + 回写镜像 tag 至 Git，`[skip ci]` 防循环），移除其部署阶段，确立「镜像由 Jenkins 写、集群资源由 Argo CD 写、副本数由 HPA 写」的单一写入者原则。

### 11.4 把八个项目串成一条完整叙事（这是最终要交付给面试官的）

```
项目 A  容器化 + 探针定版        ── 让应用能活着被访问
   ↓
项目 B  CI/CD 流水线（push）      ── 让变更能自动化到达集群
   ↓
项目 C  监控告警                 ── 让我能看见集群在发生什么
   ↓
项目 D  日志收集                 ── 让我能定位"为什么"
   ↓
项目 E  Helm 化                 ── 让交付可审计、可回滚、失败自愈
   ↓
项目 F  安全加固                 ── 让变更在边界内发生（RBAC + 网络策略）
   ↓
项目 G  HPA 弹性伸缩             ── 让集群随负载自己动手
   ↓
项目 H  Argo CD / GitOps        ── 让集群状态永远等于声明（持续协调）
```

**串讲金句（背下来）**：

> 「A–H 是一条完整的演进链，而且**每一步都在修上一步暴露的问题**：
> A 教会我"能用"的定义是"Pod 活着且探针不误杀"；
> B 让变更自动化了，但**凭据外扩、漂移无人知、内网只能轮询**；
> C/D 给我装上了眼睛和耳朵，于是我能**用证据而不是猜测**来排障；
> E 把发布从"sed 替换"升级为"有台账、可回滚、失败自愈"，**并顺手给 G 预埋了 HPA 的开关**；
> F 让我意识到"能跑起来只是及格线"，于是把权限和网络边界收到最小；
> G 让集群能根据负载自己动手，但引入了一个新问题——**副本数变成运行时状态，而当时的交付模型只有一次性动作**；
> H 就是对这个问题的回答：**把交付从"一次性动作"升级为"持续维持的不变量"**。
> 所以这八个项目不是八个孤立的技术点，而是**一个自建集群从"能跑"到"能管"的完整闭环**——这也是我认为自己可以承担运维岗位的原因：我不只是会装组件，我知道每个组件是为了解决什么问题被引入的。」

---

## 11. 学习资源（官方文档）

| 主题 | 资源 |
|------|------|
| **Argo CD 官方文档**（首选） | https://argo-cd.readthedocs.io/ |
| Argo CD 快速入门 | https://argo-cd.readthedocs.io/en/stable/getting_started/ |
| Application 规格说明 | https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/ |
| **自动同步与 self-heal** | https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/ |
| **ignoreDifferences**（HPA 共存必读） | https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/ |
| 同步选项（syncOptions） | https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/ |
| Helm 支持说明 | https://argo-cd.readthedocs.io/en/stable/user-guide/helm/ |
| 私有仓库认证 | https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/ |
| 资源健康状态判定 | https://argo-cd.readthedocs.io/en/stable/operator-manual/health/ |
| **OpenGitOps 四原则**（权威定义） | https://opengitops.dev/ |
| Argo CD Image Updater（进阶） | https://argocd-image-updater.readthedocs.io/ |
| ApplicationSet（进阶） | https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/ |
| Argo Rollouts（发布策略，进阶） | https://argoproj.github.io/rollouts/ |
| Flux（对比学习） | https://fluxcd.io/docs/ |
| 练习 | Argo CD 官方 "Getting Started" 的本地 kind 集群演练；Killercoda 的 Argo CD 场景 |
| 书籍 | 《GitOps and Kubernetes》(Manning)；《Kubernetes 生产实践》交付相关章节 |

---

## 附录 A：文件包清单与操作顺序

### A.1 文件包清单

```
K8s项目H_ArgoCD-GitOps/
├── K8s项目H_ArgoCD_GitOps实战.md        # 本文档
├── h-preflight.sh                       # ★ 开工体检脚本（跑完落盘 h-baseline-<时间戳>/）
├── manifests/
│   ├── argocd-application-damo-app.yaml # ★ Application 定义（含 ignoreDifferences + values-prod.yaml）
│   └── argocd-server-nodeport.md        # NodePort 30400 的 patch 说明与取舍
└── install/
    └── README.md                        # 镜像中转 + install.yaml 改写的步骤清单
```

> **要放进 GitHub 仓库的**：`manifest/argocd-application-damo-app.yaml`（建议放到仓库的 `gitops/` 目录）——**它就是"集群期望状态"的可审计声明**，值得进 Git。
> 更进一步（进阶做法）：把 Argo CD 自己也用 Application 管起来（**自举 / bootstrap**），这样"连 Argo CD 自己的配置也是 GitOps 管的"。**知道这个概念就够了，本项目不展开**。

### A.2 操作顺序总览

```
阶段 0  ★ 跑 h-preflight.sh（GitHub 可达 / Pod 出网 / 30400 空闲 / HPA+NetPol 基线）
   ↓     基线落盘到 h-baseline-<时间戳>/，阶段 5 用它逐项对账
   ↓
阶段 1  ★ 只做原理也必须读完这 2 小时（尤其 1.3 与 Jenkins 的分工边界）
   ↓
阶段 2  下载 install.yaml → 从清单提取镜像 → 逐个中转 → sed 改写 → 反向核对（H-1）
   ↓
阶段 3  create ns argocd → apply install.yaml → 等就绪 → NodePort 30400
   ↓     → 取初始密码 → 登录 → 改密码 → 删 argocd-initial-admin-secret（H-2）
   ↓
阶段 4  验证 repo-server 能出网 → apply Application（★ 记得带 values-prod.yaml）→ UI 看 OutOfSync + APP DIFF（H-3）
   ↓
阶段 5  首次 Sync → 五道验证（★ 第④道：HPA/NetPol 一个不少）→ 同步成功 + 资源拓扑图（H-4）
   ↓     ★ 关卡：Synced / Healthy，且 curl :30090 仍正常
   ↓
阶段 6  ★ 三个演示：改 Git 自动同步（H-5，录 GIF）
   ↓     → 手改集群自动纠偏（H-6，录 GIF）+ 关掉 selfHeal 的对照组（H-7）
   ↓     → git revert 回滚（H-9）
   ↓     → 进阶：HPA 与 GitOps 共存验证（H-10）
   ↓
阶段 7  ★ 只做原理也必须读完（CI/CD 分工）。实操：改 Jenkinsfile（删部署 Stage + 回写 tag）
   ↓     → 自绘 CI/CD 分工架构图（H-8）
   ↓
收尾   整理验收输出 → 自绘架构图放 README 首位 → 提交仓库
```

> **回退预案**：本项目**完全可逆且不影响业务**——
> - **临时停手**：`kubectl -n argocd patch application damo-app --type merge -p '{"spec":{"syncPolicy":null}}'` 关掉自动同步，集群保持现状不动。
> - **彻底退出 GitOps**：`kubectl -n argocd delete application damo-app --cascade=orphan`（**orphan = 保留集群资源**，只删 Application 对象），然后回到项目 E 的 `helm upgrade` 部署模式即可。
> - **⚠️ 不要用** `--cascade=foreground/background`** 删 Application**——那会**级联删除 damo-app 的所有资源**（因为它带着 `resources-finalizer.argocd.argoproj.io`），业务会直接中断。
> - **一句话原则**：**先想清楚"这个 Application 删掉会带走什么"，再动手。**

---

## 附录 B：GitOps 对照表与 Argo CD 命令速查

### B.1 三个模型的一次性对照（这是本项目的知识总结）

| 维度 | 手工 kubectl（项目 A） | Helm + Jenkins（项目 E） | Argo CD（项目 H） |
|------|---------------------|------------------------|------------------|
| 期望状态声明 | ❌ 直接敲命令 | ✅ Chart + values | ✅ Chart + values（**在 Git 里**） |
| 谁执行变更 | 人 | Jenkins（push） | **Argo CD（pull）** |
| 版本台账 | 无 | release Secret（`helm history`） | **Git 提交历史 + Argo CD Sync 历史** |
| 回滚方式 | 凭记忆反向操作 | `helm rollback <rev>` | **`git revert`** |
| 漂移检测 | ❌ 不知道 | ❌ 不知道 | ✅ **持续 diff** |
| 漂移纠偏 | ❌ 手动 | ❌ 无 | ✅ **selfHeal 自动** |
| 网络要求 | 人能连集群 | CI 能推到集群 | **集群能出网拉 Git** |
| 凭据位置 | 人的 kubeconfig | **CI 系统里**（外扩） | **集群内 SA**（不外扩） |
| 审计能力 | 无 | 构建日志 | **Git 历史（谁、何时、改了什么、为什么）** |
| 引入成本 | 0 | 中 | 中（+一个组件 ~500m CPU/1Gi） |

### B.2 Argo CD 常用命令速查（CLI）

```bash
# ===== 登录 =====
argocd login 11.0.1.128:30400 --username admin --password '<密码>' --insecure

# ===== 查看 =====
argocd app list                                    # 列出所有 Application 及状态
argocd app get damo-app                            # 详情（含 sync/health 与资源树）
argocd app history damo-app                        # 同步历史（相当于发布台账）
argocd app diff damo-app                           # ★ 命令行看 diff（等价于 UI 的 APP DIFF）
argocd app manifests damo-app                      # 查看当前渲染出的 manifest（排查渲染问题用）
argocd app resources damo-app                      # 列出管理的所有资源及状态

# ===== 同步 =====
argocd app sync damo-app                           # 手动同步
argocd app sync damo-app --prune                   # 同步并清理 Git 里已删除的资源
argocd app sync damo-app --dry-run                 # ★ 只预览不执行（安全的预演）
argocd app sync damo-app --resource apps:Deployment:damo-app   # 只同步单个资源
argocd app wait damo-app                           # 等待同步完成

# ===== 回滚（应急用；权威动作是 git revert）=====
argocd app rollback damo-app <revision-id>         # 从 history 里选一个 revision 回滚

# ===== 仓库管理 =====
argocd repo list
argocd repo add https://github.com/GXEL-xy/damo-app.git
argocd repo add https://github.com/org/private.git \
  --username <user> --password <PAT>               # 私有仓库（PAT 只给 Contents: Read）

# ===== 刷新（触发重新比对，演示提速用）=====
argocd app get damo-app --refresh                  # 软刷新
argocd app get damo-app --hard-refresh             # ★ 硬刷新（清缓存重新拉 Git）

# ===== 不用 CLI 时的等价 kubectl 操作 =====
kubectl -n argocd get applications
kubectl -n argocd get application damo-app -o jsonpath='{.status.sync.status}'
kubectl -n argocd get application damo-app -o jsonpath='{.status.health.status}'
kubectl -n argocd patch application damo-app --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'   # 硬刷新
```

### B.3 Application 关键字段速查

| 字段 | 作用 | 本项目取值 |
|------|------|-----------|
| `spec.project` | 属于哪个 AppProject（权限边界） | `default` |
| `spec.source.repoURL` | Git 仓库地址 | `https://github.com/GXEL-xy/damo-app.git` |
| `spec.source.targetRevision` | 分支/tag/commit | `HEAD`（或 `main`） |
| `spec.source.path` | 清单所在目录 | `chart/damo-app` |
| `spec.source.helm.releaseName` | Helm release 名 | `damo-app`（与项目 E 一致） |
| `spec.source.helm.parameters` | 覆盖 values 的参数 | `image.tag = "19"` |
| `spec.destination.server` | 目标集群 API | `https://kubernetes.default.svc`（本集群） |
| `spec.destination.namespace` | 目标命名空间 | `damo-app` |
| `spec.syncPolicy.automated.prune` | Git 删了集群也删 | `true` |
| `spec.syncPolicy.automated.selfHeal` | 漂移自动纠偏 | `true` |
| `spec.syncPolicy.syncOptions` | 同步行为开关 | `ApplyOutOfSyncOnly`、`PrunePropagationPolicy` |
| `spec.ignoreDifferences` | 忽略的字段（交给其他控制器管） | `/spec/replicas`（**让给 HPA**） |
| `metadata.finalizers` | 删除级联行为 | `resources-finalizer.argocd.argoproj.io` |
