---
title: "K8s 项目 E：CI/CD 流水线 Helm 化改造（damo-app Chart 实战）"
date: 2026-10-30T16:40:00+08:00
draft: false
description: "把 damo-app 的裸 YAML 改造成标准 Helm Chart（参数全收敛到 values、标签三处同源、Chart 逐文件解剖），发布环节从 sed + kubectl apply 升级为 helm upgrade --atomic，换来可审计的 revision 与一键 rollback。"
tags: ["k8s Helm"]
categories: [项目]
---

> 📦 **项目文件包**：[下载 K8s项目E_Helm化改造.zip](/files/K8s项目E_Helm化改造.zip)（含 damo-app Chart 全部模板与 Jenkinsfile，可直接拷到集群 master 使用）。本文件是详细操作手册，两者配合。

> **系列定位**：K8s 运维实习项目路线图第 5 站——A 应用容器化 → B CI/CD → C 监控告警 → D 日志收集 → **E Helm 化**。
> **项目定位**：把项目 B 的「sed 替换 `__IMAGE__` + `kubectl apply`」升级为「**Helm Chart 标准化交付**」——`helm upgrade --install --set image.tag=N` 一行发布、`helm rollback` 一键回滚、`--atomic` 失败自动回退。**改造对象是同一套 damo-app 和同一条流水线，改造后行为不变、能力升级。**
> **版本基线**：master 实测 helm **v3.21.3**（helm 3 无 tiller）；Agent 镜像 `dtzar/helm-kubectl:3.19.1`（该仓库 v3 系列最新，**Docker tag 无 v 前缀**；3.x 小版本间 release 台账互通，3.19 读写 3.21 创建的 release 无障碍）／复用项目 B 全部环境（3 节点 v1.35 / containerd 2.x / registry :30000 / damo-app :30090）。
> **文档版本**：v1.1（2026-09-10）· 基于集群实测定版（helm v3.21.3 / tags 16-19 / .helmignore 与 ErrImagePull 两坑入册）· 配套文件包同目录 `damo-app-chart/` + `Jenkinsfile`。
> **权威依据**：Helm 官方文档（helm.sh/docs）、Kubernetes 官方文档。

---

> **📖 怎么用这份文档**
>
> - **前置依赖**：项目 B 已完成（Jenkins 流水线与 `damo-app` 可正常发布）；会读 YAML 即可，不需要 Helm 基础
> - **预计耗时**：3–4 小时
> - **学习路径**：阶段 1 Helm 原理 → 阶段 2 镜像准备 → 阶段 3 逐文件解剖 Chart → 阶段 4 手动跑通 upgrade/rollback → 阶段 5 接回 Jenkins
> - **验收标准**：`helm upgrade` 一条命令完成发布、`helm rollback` 能回退、`helm history` 看得到每个版本用了什么参数

## 目录

- [0. 项目概览](#0-项目概览)
- [阶段 1：先懂再装——Helm 原理](#阶段-1先懂再装helm-原理)
- [阶段 2：镜像准备（helm-kubectl 中转，5 分钟）](#阶段-2镜像准备helm-kubectl-中转5-分钟)
- [阶段 3：Chart 逐文件解剖](#阶段-3chart-逐文件解剖)
- [阶段 4：不依赖 Jenkins，手动跑通全流程](#阶段-4不依赖-jenkins手动跑通全流程)
- [阶段 5：Jenkins 流水线改造](#阶段-5jenkins-流水线改造)
- [阶段 6：亮点演示清单（面试可直接演）](#阶段-6亮点演示清单面试可直接演)
- [7. 故障速查表](#7-故障速查表)
- [8. 自测题（含答案）](#8-自测题含答案)
- [9. 面试要点与简历写法](#9-面试要点与简历写法)
- [10. 学习资源（官方文档）](#10-学习资源官方文档)
- [附录 A：文件包清单与操作顺序](#附录-a文件包清单与操作顺序)

---

## 0. 项目概览

### 0.1 目标与产出

| 产出 | 说明 |
|------|------|
| 标准 Helm Chart | `damo-app-chart/`（Chart.yaml + values.yaml + 4 个模板），参数全覆盖、零硬编码 |
| 改造版流水线 | Stage 4 从 sed+apply 换成 `helm upgrade --install --set image.tag --atomic` |
| 双通道发布语义 | 升级生成 revision，`helm history` 审计、`helm rollback` 一键回退、失败自动回滚 |
| 面试完整故事线 | "我把 CI/CD 的发布环节从 sed 替换升级为 Helm 标准化交付，发布可审计、可回滚、失败自愈" |

### 0.2 改造前后对比（本项目的一句话价值）

| | 项目 B（裸部署版） | 项目 E（Helm 版） |
|---|---|---|
| 发布方式 | `sed` 替换 YAML 里的 `__IMAGE__` → `kubectl apply -f k8s/` | `helm upgrade --install --set image.tag=N --atomic` |
| 版本管理 | 集群里"现在是啥就是啥"，无发布历史 | 每个 revision 持久化，`helm history` 全记录 |
| 回滚 | `kubectl rollout undo`（只回 Pod 模板，不记配置） | `helm rollback <revision>`（整套渲染结果回退） |
| 发布失败 | apply 半途失败 = 集群里是"半新半旧"状态 | `--atomic` 超时/失败**自动回滚**，坏版本永不生效 |
| 参数修改 | 手改 YAML 各处 | 改 values 一处，或 `--set` 临时覆盖 |
| 多环境 | 复制整套 YAML 改参数 | `-f prod-values.yaml` 一份模板多环境 |
| 配置审计 | git 里看"可能被 apply 过的"YAML | `helm get manifest` 看实际渲染生效的完整清单 |

> **为什么说 sed 是脆弱的**：sed 改的是"仓库里的文件"，apply 完集群状态和 git 就可能分叉；多次 sed 的正则还可能误伤。Helm 是"渲染 → 原子提交 → 记录"，三者一体，git 始终是唯一事实来源。

### 0.3 架构与数据流（变化只在 Stage 4）

```
 开发者                    Jenkins (K8s 内)                 三节点 K8s 集群
   │                            │                               │
   │ git push                   │ Poll SCM (每 5 min)           │
   ▼                            ▼                               │
 GitHub ──────────────► Jenkins Master :30080                   │
                                │ 拉起动态 Agent Pod              │
                     ┌─────────────────────────┐                │
                     │  jnlp    控制通道          │                │
                     │  kaniko  构建镜像 ──push──►│ 私有仓库 :30000 │
                     │  helm    helm+kubectl ◄──┼──chart 在 git 仓库里
                     └─────────────────────────┘                │
                                │ helm upgrade --install         │
                                │   --set image.tag=N            │
                                │   --atomic                     │
                                │  ① 渲染 chart → 纯 YAML        │
                                │  ② 对比上一 revision 做补丁      │
                                │  ③ 提交到集群 + 记录 revision    │
                                ▼                               │
                                  kubelet ──pull──► 镜像         │
                                                     ▼          │
                                  damo-app ×2 ──► NodePort :30090│
```

**一句话记住 Helm 的位置**：Helm 不参与运行时（不拉镜像、不重启容器），它只做两件事——**把 Chart 渲染成纯 YAML**、**把"这次渲染的结果"记录成一个版本（revision）**。容器编排仍然 100% 由 K8s 完成。

### 0.4 关键技术决策（面试可讲）

| 决策 | 选择 | 理由 |
|------|------|------|
| Chart 形态 | 手写 Chart（教学） | 每个字段自己写、自己渲染、自己排错；企业对照是 `helm create` 脚手架或社区 Chart（kube-prometheus-stack 等） |
| Chart 分发 | **git 仓库内目录直装**（path install） | 单应用最简形态，Chart 与源码同仓同版本；企业对照：Chart Museum 私仓 / OCI registry（`helm push` 到 30000 也可行，见自测 Q10） |
| 工具容器 | dtzar/helm-kubectl 复合镜像 | 原 kubectl 容器没有 helm 二进制；复合镜像一个容器同时有 helm+kubectl，PodTemplate 只改镜像不改结构 |
| 接管裸资源 | **delete + install**（教学）／annotate+label 迁移（生产正统，阶段 4 讲解） | Helm 3.2+ 对无归属标记的资源拒绝接管（ownership 校验），两条路径各有适用场景 |
| 发布策略 | `upgrade --install --atomic --timeout 120s` | `--install` 通吃首装与迭代；`--atomic` = `--wait` + 失败自动回滚，坏版本进不了"已发布" |
| HPA | Chart 内预埋 `autoscaling.enabled` 开关（默认 false） | 项目 G 弹性伸缩时打开即可，模板已写好分支——Chart 化的另一收益：扩展点提前布局 |

### 0.5 资源与端口现状（无新增）

| 项 | 值 | 说明 |
|----|-----|------|
| NodePort | 30090（不变） | Service 由 Chart 渲染，值与项目 B 一致 |
| 新增镜像 | dtzar/helm-kubectl:3.19.1 → 中转 `11.0.1.128:30000/helm-kubectl:3.19.1` | 唯一新增镜像，走中转铁律 |
| 新增集群对象 | release Secret（`sh.helm.release.v1.damo-app.vN`，helm 自动创建） | revision 的存储本体，阶段 1 讲解 |
| 前置条件 | 项目 B 流水线健康运行；master 上 helm 可用（或用容器执行） | 阶段 4 手动跑通时需要本机 helm，阶段 5 起全部由 Agent 容器执行 |

---

## 阶段 1：先懂再装——Helm 原理

### 1.0 开胃故事：从"逐间砌墙"到"施工图纸"（先建立直觉）

项目 B 的裸部署像**监理拿着一张张零散图纸逐间砌墙**：墙砌完，图纸还摊在桌上（git），但屋里实际什么样没人记账——想撤掉某次"改造"，只能凭记忆反向操作。

Helm 是**总包施工方**：
- **Chart（图纸册）**：不是一张张散图，而是一册带参数表（values.yaml）的标准图纸；
- **渲染（按参数出图）**：开工前按当前参数把图纸册填成确定的施工图（纯 YAML）；
- **Release（施工实例 + 台账）**：每次施工记一页台账（revision）——"第 2 版用了什么参数、渲染出了什么图"，全部存档；
- **upgrade / rollback（按台账施工）**：升级 = 按新参数出图 + 台账记新页；回滚 = 翻出台账旧页原样重砌；
- **--atomic（烂尾自愈）**：施工验收不过（资源没 Ready），自动按台账上一页重砌，**绝不留半成品**。

口诀：**图纸册（Chart）→ 按单出图（渲染）→ 台账记版本（Release）→ 翻账回滚（rollback）→ 验收不过自动返工（--atomic）**。

### 1.1 三大核心概念

| 概念 | 是什么 | 本项目的实例 |
|------|--------|-------------|
| **Chart** | 一组模板 + 一份默认参数，本身只是"图纸册"（tgz 或目录） | `chart/damo-app/` 目录 |
| **Values** | 参数表；Chart 自带默认值，安装时按优先级覆盖 | `values.yaml` + `--set image.tag=N` |
| **Release** | "Chart + 参数"在集群里的一次具体安装实例，**每次 upgrade 产生新 revision** | release 名 `damo-app`，revision 1,2,3… |

三个易混点提前拆雷：

1. **Chart 版本 vs 应用版本**：`Chart.yaml` 里 `version: 0.1.0` 是**模板**的版本（模板改一行就递增）；`appVersion: "1.0"` 是**应用**的语义版本（元信息）。镜像 tag 两者都不是——它由流水线注入 `--set image.tag=${BUILD_NUMBER}`。
2. **Release 名 vs 资源名**：release 是 Helm 记账单位（`helm list` 里的一行）；资源名是渲染产物（由 `_helpers.tpl` 的 fullname 决定）。本项目两者刻意同名（`damo-app`），与项目 B 无缝衔接。
3. **helm upgrade ≠ kubectl apply 的重复执行**：apply 是"期望状态对齐"（无版本概念）；upgrade 是"渲染 → 与上一 revision 对比 → 补丁提交 → 记新 revision"——**配置变更也是一次发布**（改副本数同样 +1 revision）。

### 1.2 Helm 在集群里留下了什么（面试必考：revision 存在哪）

```bash
kubectl -n damo-app get secrets | grep sh.helm.release
# sh.helm.release.v1.damo-app.v1   helm.sh/release.v1   Opaque   1   2m
# sh.helm.release.v1.damo-app.v2   helm.sh/release.v1   Opaque   1   30s
```

- **每个 revision 一个 Secret**（`sh.helm.release.v1.<release>.v<N>`），内容是 base64+gzip 的完整发布记录：**Chart 全文 + 当时的 values + 渲染出的 manifest**。
- `helm rollback` 能工作、`helm get values/manifest` 能审计，靠的就是这套台账——**它存在集群里，不在 Helm 客户端里**，换台机器执行 helm 命令照样能回滚。
- 推论（面试点）：**删掉这些 Secret = release 失忆**，`helm list` 看不到、rollback 无从谈起。清理 release 要用 `helm uninstall`（它会清台账），不要手工删资源。

### 1.3 Helm 3 为什么没有 Tiller（高频概念题）

Helm 2 集群侧有个常驻服务 **Tiller** 代理所有安装操作，且默认权限过大（cluster-admin），成为安全短板；且多租户共用 Tiller 时谁都能动别人的 release。Helm 3 **砍掉 Tiller**：helm 客户端直接拿你的 kubeconfig 调 API Server——**你有多大权限，helm 就只能干多大的事**，权限模型回归 K8s 原生 RBAC。

### 1.4 模板引擎速成（读模板不迷路）

| 语法 | 含义 | 本项目实例 |
|------|------|-----------|
| `{{ .Values.image.tag }}` | 取参数值 | 渲染 `image: "...:N"` |
| `{{ include "damo-app.labels" . }}` | 调用 `_helpers.tpl` 定义的函数 | Deployment/Service 共用同一套标签 |
| `\| nindent 4` | 管道：先缩进再换行（`indent` 不吃首行换行，`nindent` 吃） | 让渲染结果 YAML 缩进正确 |
| `{{- ... }}` | 吃掉符号两边的空白/换行 | 控制渲染结果的整洁 |
| `{{- toYaml .Values.resources \| nindent 12 }}` | 把整块 map（requests/limits）原样转 YAML | 复杂结构参数化 |
| `{{- if ... }}...{{- end }}` | 条件渲染（不满足时整段不出现） | `nodePort` 留空则不写该字段、HPA 开关 |
| `{{ default "x" .Values.a }}` | 空值兜底 | name 函数里 nameOverride 兜底 |

**覆盖优先级**（面试必背）：`--set` ＞ `-f 自定义values文件` ＞ Chart 自带 `values.yaml`。`--set` 的值也会记进该 revision 的台账，`helm get values` 可查。

### 1.5 阶段自测

- [ ] 能不看文档讲完"施工图纸"故事，把 Chart/Values/Release 对号入座
- [ ] 能说出 revision 存在哪（release Secret）、rollback 为什么换台机器也能用
- [ ] 能说清 Chart.yaml 里 version 和 appVersion 的区别；镜像 tag 属于谁
- [ ] 能背出 values 覆盖优先级，并说出 `helm get values` 能查到 --set 的原因
- [ ] 能说出 Helm 3 砍 Tiller 的原因（权限回归原生 RBAC）
- [ ] 能解释"Helm 不参与运行时，只管渲染和记账"

---

## 阶段 2：镜像准备（helm-kubectl 中转，5 分钟）

> 延续项目 B 铁律：国内节点直连 Docker Hub 拉不动，所有镜像先在 master 中转进私有仓库。本项目唯一新增镜像：**dtzar/helm-kubectl**（社区维护的 helm+kubectl 复合镜像，单架构，可直接中转）。

```bash
# 在 master（docker 配了代理）执行
docker pull dtzar/helm-kubectl:3.19.1
docker tag dtzar/helm-kubectl:3.19.1 11.0.1.128:30000/helm-kubectl:3.19.1
docker push 11.0.1.128:30000/helm-kubectl:3.19.1

# 验证（仓库目录应出现 helm-kubectl）
curl -s http://11.0.1.128:30000/v2/_catalog | grep helm-kubectl

# 从 worker 模拟真实拉取路径
sudo crictl pull 11.0.1.128:30000/helm-kubectl:3.19.1
```

**常见报错**：与项目 B 阶段 2 完全同款（代理超时 / insecure-registries 丢失 / manifest unknown），速查表见项目 B 文档第 7 节。

---

## 阶段 3：Chart 逐文件解剖

> 文件包 `damo-app-chart/` 每个文件头部都有完整注释，本节讲"为什么这么设计"。先看结构：

```
damo-app-chart/
├── Chart.yaml            # Chart 元数据（名字/双版本）
├── values.yaml           # ★ 唯一参数来源（默认值=项目 B 现行真实值）
├── .helmignore           # 打包忽略清单
└── templates/
    ├── _helpers.tpl      # 标签/命名函数（Deployment 与 Service 的"共同基因"）
    ├── deployment.yaml   # 2 副本 + 三探针 tcpSocket + __IMAGE__ 不再存在
    ├── service.yaml      # NodePort 30090（沿用端口规划）
    └── NOTES.txt         # 发布成功后打印的"操作小抄"（helm 自动输出）
```

### 3.1 Chart.yaml——双版本元数据

| 字段 | 值 | 要点 |
|------|-----|------|
| `apiVersion: v2` | — | Helm 3 固定 v2 |
| `version: 0.1.0` | 模板版本 | **改模板必须递增**——它进 helm.sh/chart 标签，是"这套 YAML 长什么样"的版本 |
| `appVersion: "1.0"` | 应用版本 | 必须加引号（不加引号 1.0 被解析成 float，helm lint 会拦） |
| 镜像 tag | **不在这里** | tag = 构建号 = 发布时刻的值，由 `--set` 注入并记入 revision |

### 3.2 values.yaml——所有会变的值收敛于此

设计三原则（文件头注释有完整版）：

1. **默认值 = 项目 B 现行真实值**（replicas 2 / NodePort 30090 / 三探针 tcpSocket / requests 100m·64Mi）——参数化不改变行为，升级无缝；
2. **键名分层与 `--set` 点号路径一一对应**：`--set image.tag=3` ↔ `image.tag: "1"`；
3. **`image.pullPolicy: Always` 的理由**：构建号理论上"可复现"（重跑同号构建），Always 保证每次发布核对 digest，杜绝节点旧缓存冒充新版本。

`autoscaling.enabled: false` 是预埋开关：模板里 `replicas` 已用 `if not` 包住——项目 G 打开开关、Chart 渲染出 HPA 并停止渲染 replicas，**扩展点提前布局，这是 Chart 化相比裸 YAML 的隐性收益**。

### 3.3 _helpers.tpl——两处 include，永不失配

| 函数 | 输出 | 被谁引用 |
|------|------|---------|
| `damo-app.name` | `damo-app`（可被 nameOverride 覆盖） | selectorLabels |
| `damo-app.fullname` | Release 名含 Chart 名则取 Release 名 → **`damo-app`**（与 B 无缝） | deployment/service 的 metadata.name |
| `damo-app.selectorLabels` | `app.kubernetes.io/name + instance` | **Deployment 的 pod labels、spec.selector.matchLabels、Service 的 selector——三处同源** |
| `damo-app.labels` | selectorLabels + helm.sh/chart + version + managed-by=Helm | metadata.labels |

> **★ 必讲坑（裸 YAML 时代最高频故障之一）**：Service 的 `selector` 与 Pod 的 `labels` 不一致 → Endpoints 为空 → Service 404。`kubectl get endpoints -n damo-app` 看不到 Pod IP 就查这里。Chart 化后两处来自同一函数，**结构上根除**。
>
> **标签换血注意**：Chart 版标签（`app.kubernetes.io/name=damo-app`）与 B 版标签（`app: damo-app`）不同——因为 Deployment 与 Service 由 helm 同一次 upgrade 一体更新，selector 同步切换，无缝；但**如果单独 apply 一方就会断链**（见 4.2 的接管方案）。

### 3.4 deployment.yaml——三个本质变化

| 变化 | 裸部署版 | Chart 版 |
|------|---------|---------|
| 镜像 | `image: __IMAGE__`（sed 替换） | `image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"`（渲染时求值） |
| 标签 | 手写 `app: damo-app` | `include "damo-app.selectorLabels"`（同源） |
| 端口 | `containerPort: 80` 裸数字 | 命名端口 `http`，探针/Service 引用 name（改端口只改一处） |

探针三件套原样保留 tcpSocket 定版写法（项目 A 教训）；`toYaml .Values.resources` 演示"整块结构参数化"。

### 3.5 service.yaml 与 NOTES.txt

- **nodePort 条件渲染**：`{{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}`——参数为空时整行不出现，交给 K8s 自动分配；这才是"参数化"，渲染出 `nodePort: null` 反而是错的。
- **NOTES.txt**：`helm install/upgrade` 成功后自动打印——发布号、镜像 tag、验证命令、回滚命令。它是"发完发布后给值班人的操作小抄"，写清这些能让任何接手的人 10 秒上手。

### 3.6 .helmignore

Chart 与源码同仓，`helm package` 打包时忽略 .git、Dockerfile、Jenkinsfile、index.html 等——Chart 包只装模板与参数。

---

## 阶段 4：不依赖 Jenkins，手动跑通全流程

> 原则同项目 B：**先在命令行手动把 Helm 的每个动作跑通、每个输出看懂，再交给流水线自动化**。本阶段全部命令在 master 执行（需要本机 helm 3.16+；没有就先 `snap install helm --classic` 或用容器 `docker run --rm -v ~/.kube:/root/.kube -v ./damo-app-chart:/charts 11.0.1.128:30000/helm-kubectl:3.19.1 helm ...`）。

### 4.1 三道体检：lint → template → dry-run

```bash
cd damo-app-chart     # 进入 Chart 目录（文件包上传 master 后）

# ① 静态检查：语法错误、引号缺失、类型问题
helm lint .
# 预期：1 chart(s) linted, 0 chart(s) failed

# ② 本地渲染：不碰集群，看"按当前 values 渲染出的最终 YAML"
helm template damo-app . --set image.tag=9
# 逐段检查：image 行是否为 ...:9、labels 三处是否一致、
# nodePort 是否 30090、resources 是否缩进正确

# ③ 试运行：真的连集群校验（API 版本、字段合法性），但不提交
helm upgrade --install damo-app . -n damo-app --create-namespace \
  --set image.tag=9 --dry-run
```

> **面试点**：CI 里提交前跑 `helm lint` + `helm template`（甚至 `kubeconform` 校验渲染产物），把模板错误挡在集群之外——"流水线左移"的具体实践。

### 4.2 接管裸资源：Helm 的 Ownership 校验（★本项目必踩坑）

直接对项目 B 的存量资源执行 `helm install` 会报错：

```
Error: Unable to continue with install: Deployment "damo-app" in namespace "damo-app"
exists and cannot be imported into a release: invalid ownership metadata;
label validation error: missing key "app.kubernetes.io/managed-by": must be set to "Helm"
```

**为什么**：Helm 3.2+ 防止"误接管"——把别人手 apply 的资源错误地记到 release 台账上，后续 uninstall/rollback 会误伤。接管必须先补"归属标记"（annotation + label）。

**方案 A（推荐，教学环境）：删旧装新**

```bash
kubectl -n damo-app delete deploy/damo-app svc/damo-app
# image.tag 填仓库里实际存在的构建号（curl http://11.0.1.128:30000/v2/damo-app/tags/list 查）
helm upgrade --install damo-app . -n damo-app --create-namespace --set image.tag=19 --atomic --timeout 120s
```

静态页无状态、Pod 删除无损失；镜像在私有仓库秒级重拉。30 秒完成，最干净。

**方案 B（生产正统）：原地迁移（面试加分点）**

```bash
# 补归属标记：告诉 Helm "这个资源从今天起归 release damo-app 管"
kubectl -n damo-app annotate deploy/damo-app svc/damo-app \
  meta.helm.sh/release-name=damo-app \
  meta.helm.sh/release-namespace=damo-app --overwrite
kubectl -n damo-app label deploy/damo-app svc/damo-app \
  app.kubernetes.io/managed-by=Helm --overwrite

# 再 install：Helm 校验标记通过，资源并入 release（不重建、业务零中断）
helm upgrade --install damo-app . -n damo-app --set image.tag=19 --atomic --timeout 120s
```

> 两条路径的取舍：A 牺牲一次重建换干净；B 保留运行中业务零中断，但要小心 label 差异（B 版 `app: damo-app` 标签会在本次 upgrade 被替换为新标签体系，Deployment 滚动更新一遍）。面试答"生产迁移"就讲 B。

### 4.3 发布与验证

> ⚠️ **tag 必须用仓库里真实存在的构建号**：`curl -s http://11.0.1.128:30000/v2/damo-app/tags/list` 先查。本项目集群实测 tags 为 16/17/18/19（流水线从 Build #16 起成功产出），下文以 **19** 为例。

```bash
kubectl -n damo-app get pods,svc                       # 2/2 Running，30090 在位
curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'   # Build #19
kubectl -n damo-app get secrets | grep sh.helm.release       # v1 台账出现
helm list -n damo-app
# NAME       NAMESPACE   REVISION  STATUS    CHART           APP VERSION
# damo-app   damo-app    1         deployed  damo-app-0.1.0  1.0
```

> **★ 实战教训（2026-09-10）**：`STATUS: deployed` 只表示 **manifest 已提交集群**，不等于应用健康——helm 不带 `--wait`/`--atomic` 是"提交即返回"。健康要看 Pod 或让 `--atomic`（隐含 `--wait`）替你等。另：**首装失败时 atomic 的动作是 uninstall**（没有上一版可回滚），现场会被清空，排障要么重跑不带 atomic 保留现场，要么先看 `kubectl get events -n damo-app --sort-by=.lastTimestamp`。

### 4.4 两次升级 + 一次回滚（把台账用起来，集群实测流程）

```bash
# ① 改镜像 = 发布新版本（当前在 19，切到 18——两个真实存在的 tag 之间切换）
helm upgrade damo-app . -n damo-app --set image.tag=18 --atomic --timeout 120s
curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'   # Build #18

# ② 改参数 = 也是一次发布（副本数 2→3）
helm upgrade damo-app . -n damo-app --set replicaCount=3
kubectl -n damo-app get pods                                  # 3 个 Pod
# 注意：replicas=3 意味着台账里存的是 3；rollback 会连副本数一起回退——
# 这正是 Helm"整套渲染结果回退"与 kubectl rollout undo（只回模板）的本质区别

# ③ 看台账
helm history damo-app -n damo-app
# REVISION  STATUS     CHART           DESCRIPTION
# 1         superseded damo-app-0.1.0  Install complete
# 2         superseded damo-app-0.1.0  Upgrade complete
# 3         deployed   damo-app-0.1.0  Upgrade complete

# ④ 回滚到 revision 2（Build #19、2 副本，一步到位）
helm rollback damo-app 2 -n damo-app
curl -s http://11.0.1.128:30090/ | grep -o 'Build #[0-9]*'   # Build #19
kubectl -n damo-app get pods                                  # 2 个 Pod
helm history damo-app -n damo-app                             # 新增 revision 4（rollback 也是发布！）
```

> **★ 面试细节**：rollback 不"删台账"，而是**生成一个内容等于旧版的新 revision**——历史只增不减，任何时刻可审计"谁在什么时候回滚过"。

### 4.5 --atomic 失败自动回滚（演示"坏版本进不了已发布状态"）

```bash
# 故意发一个坏参数：探针打到不存在的端口 → Pod 永远 NotReady → 超时
helm upgrade damo-app . -n damo-app \
  --set probes.readiness.initialDelaySeconds=1 \
  --set-string fakeBad=1 --atomic --timeout 60s \
  --set probes.liveness.initialDelaySeconds=1 \
  --set image.tag=99
# （更直观的坏法：把 values 里探针参数临时改错再 upgrade，演示完改回）

# 预期日志：Upgrade failed...rolling back
helm history damo-app -n damo-app      # 最新 revision 状态 = failed（集群仍是上一版好状态）
curl http://11.0.1.128:30090/          # 依然是 Build #2，业务零感知
```

> **--atomic 的双面性（面试可讲）**：它保证"发布失败不留半成品"，但**失败被自动处理不等于问题被解决**——流水线 Stage 4 报红时必须去看失败原因（大概率是镜像拉不到/探针配置错），atomic 只是止损，不是修复。

### 4.6 阶段 4 完成标准

- [ ] `helm lint` 0 failed；`helm template --set image.tag=9` 渲染产物目检通过
- [ ] 存量裸资源已接管（方案 A 或 B），`helm list` REVISION=1，页面 Build #1
- [ ] 完成两次 upgrade + 一次 rollback，`helm history` 台账完整
- [ ] 亲眼看到 --atomic 失败回滚：history 有 failed 记录、业务页面无感

---

## 阶段 5：Jenkins 流水线改造

### 5.1 改 Jenkins Agent PodTemplate（两处）

Jenkins 管理界面 → System → Clouds → kubernetes → Pod Templates → 原动态 Agent 模板：

| 改动 | 旧值（项目 B） | 新值（项目 E） |
|------|--------------|--------------|
| 容器名 | `kubectl` | `helm`（Jenkinsfile 里 `container('helm')` 与之对应） |
| 镜像 | `11.0.1.128:30000/kubectl:xxx` | `11.0.1.128:30000/helm-kubectl:3.19.1` |
| 其余 | jnlp 留空 Command、kaniko 容器、workspace 卷、SA | **全部不动** |

> 该镜像同时含 helm 与 kubectl（阶段 5 的 Verify 还要用 kubectl），一个容器顶两个，PodTemplate 结构不变。

### 5.2 换 Jenkinsfile（文件包 `Jenkinsfile` 整文件替换仓库里的旧版）

仓库结构同步调整（Chart 与源码同仓，一起版本化）：

```
damo-app/                     # GitHub 仓库
├── index.html
├── Dockerfile
├── Jenkinsfile               # ← 替换为文件包新版（Stage 4/5 改造）
├── chart/
│   └── damo-app/             # ← 文件包 damo-app-chart/ 整目录放这里
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── .helmignore
│       └── templates/...
└── k8s/                      # 保留：作为"改造前"的对照与回退预案（见 7 节速查表）
```

核心变化（Stage 4）：

```groovy
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

| 细节 | 说明 |
|------|------|
| `upgrade --install` | release 不存在则 install，存在则 upgrade——首装与迭代一条命令 |
| `--set image.tag=${IMAGE_TAG}` | 构建号注入，`__IMAGE__` 占位符与 sed 正式退役 |
| `--atomic --timeout 120s` | 流水线级"发布失败自动止损"：Stage 4 报红，集群仍是上一版好状态 |
| `--create-namespace` | 幂等创建 ns，替代裸部署版的 `create ns --dry-run | apply` 组合拳 |

### 5.3 全链路验证

```bash
git add . && git commit -m "feat: helm-ize deployment pipeline" && git push
# 等 Poll SCM（≤5min）或 Jenkins 手动 Build Now

# Console 观察点：
# Stage 3 kaniko push 成功 → Stage 4 出现
#   "Release damo-app has been upgraded. REVISION: N"
# → Stage 5 打印 pods + helm history
```

| # | 验证项 | 命令 | 通过标准 |
|---|--------|------|---------|
| ① | 流水线绿 | Jenkins Build #M Console | 5 个 Stage 全绿 |
| ② | 页面更新 | `curl http://11.0.1.128:30090/` | `Build #M` |
| ③ | 台账推进 | `helm history damo-app -n damo-app` | revision +1，deployed |
| ④ | 归属确认 | `kubectl -n damo-app get deploy damo-app -o yaml \| grep managed-by` | `Helm` |
| ⑤ | 监控无告警 | Grafana :30300 | Pod 重启告警未触发（滚动更新 0 中断） |

---

## 阶段 6：亮点演示清单（面试可直接演）

| # | 演示 | 命令 | 讲述要点 |
|---|------|------|---------|
| 1 | **一行发布** | `helm upgrade damo-app . -n damo-app --set image.tag=N --atomic` | 对比旧版 sed+apply：无占位符、无正则、参数即发布 |
| 2 | **发布审计** | `helm history` → `helm get values damo-app --revision 2` | "我能说出线上任何一个版本当时用了什么参数" |
| 3 | **一键回滚** | `helm rollback damo-app 2` | rollback 也是新 revision，历史只增不减 |
| 4 | **失败自愈** | 故意坏参数 + `--atomic` | history 显示 failed，业务页面无感——"坏版本进不了已发布状态" |
| 5 | **模板即文档** | `helm template . --set image.tag=9` | 现场渲染，labels 三处同源、nodePort 条件渲染 |
| 6 | **release 台账实体** | `kubectl get secrets -n damo-app \| grep sh.helm` | revision 就是 Secret，删台账=失忆（Q7 考点） |

---

## 7. 故障速查表

| 现象 | 原因 | 排查与解决 |
|------|------|-----------|
| **lint 报 `Chart.yaml file is missing`（但文件明明在）** | **① `.helmignore` 写了行尾注释**（helm 只支持整行 `#` 注释，`规则 # 说明` 整行被当规则解析，可误伤加载）；② Windows 传输带入 CRLF/BOM；③ helm 版本过老 | 三层二分：`find . -type f` 查结构 → `/tmp` 放最小 chart 测 helm 本体 → 移走 `.helmignore`/`values.yaml`/清空 templates 逐个二分。修复：`.helmignore` 只留整行注释+规则，文件确认 UTF-8 无 `\r`（`od -c` 查）【2026-09-10 实战实锤】 |
| **upgrade 后 Pod `ErrImagePull: ... not found`** | `--set image.tag=N` 引用了仓库里不存在的 tag（本项目实测：tags 只有 16-19，引用 1 直接拉取失败） | 先 `curl -s http://11.0.1.128:30000/v2/damo-app/tags/list` 确认真实 tag 再发；流水线用 `${BUILD_NUMBER}` 天然不会踩这个 |
| **`STATUS: deployed` 但 Pod 异常** | 不带 `--wait`/`--atomic` 的 upgrade 是"提交即返回"，不保证应用健康 | 看引用是否为空判断：查 `kubectl get pods`；要 helm 替你等就用 `--atomic`（隐含 `--wait`） |
| **install 报 `invalid ownership metadata`** | 接管裸资源未补归属标记（阶段 4.2） | 方案 A：delete+install；方案 B：annotate `meta.helm.sh/release-name/namespace` + label `managed-by=Helm` |
| **Stage 4 报 `helm: not found`** | Agent PodTemplate 容器还是旧 kubectl 镜像 | 换 `11.0.1.128:30000/helm-kubectl:3.19.1`，容器名与 Jenkinsfile `container('helm')` 一致 |
| **`helm: command not found`（master 手动跑）** | 本机未装 helm | snap/helm --classic 安装，或全程用 helm-kubectl 容器执行（-v 挂 kubeconfig 与 chart 目录） |
| **`Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress`** | 上次操作异常中断，release 被锁（pending-upgrade 状态） | `helm history` 确认卡住的 revision → `helm rollback damo-app <上一个好revision>` 解锁；或 `--wait` 别设太短 |
| **upgrade 卡住最后 atomic 回滚** | 新版本 Pod 起不来（探针错/镜像拉不到），`--timeout` 到点自动回退——**这是功能不是故障** | 看 Pod 事件定位根因（`kubectl describe pod`），修好再发；atomic 只止损不修复 |
| **upgrade 后 Service 不通、Endpoints 空** | 单独更新了一方（只 apply 了 Deployment 或只 helm 装了 Service），selector 与 labels 失配 | 两方必须同一次 upgrade 内一体更新；临时验证 `kubectl get endpoints -n damo-app` |
| **`appVersion: 1.0` 报 must be a string** | YAML 把 1.0 解析成 float | 加引号 `"1.0"`（`helm lint` 会拦） |
| **`helm template` 输出缩进错乱/字段丢失** | `nindent`/`indent` 用混、`{{-` 吃换行过度 | 对照文件包模板；渲染目检是硬要求，不目检不提交 |
| **回滚后发现"参数没回去"** | rollback 回的是**渲染结果**，本地 values.yaml 改动不参与 | 想以新参数重新发布 → 再跑一次 `helm upgrade --set`；git 与集群的差异用 `helm diff` 插件看 |
| **`Error: cannot re-use a name that is still in use`** | 同一 ns 有同名 release（残留） | `helm list -n damo-app` 确认；确认无用则 `helm uninstall` 清台账再装 |

---

## 8. 自测题（含答案）

1. **Helm 的 revision 存在哪？为什么换台电脑也能 rollback？**
   存在集群里的 release Secret（`sh.helm.release.v1.<release>.v<N>`，内容含 Chart 全文+当时 values+渲染 manifest）。回滚依据的是集群里的台账，不依赖本地状态。

2. **`version` 和 `appVersion` 的区别？镜像 tag 属于哪个？**
   version=Chart 模板版本（进 helm.sh/chart 标签）；appVersion=应用语义版本（元信息）。镜像 tag 都不属于——它由流水线 `--set image.tag=${BUILD_NUMBER}` 注入，每次发布是一个 values 覆盖值。

3. **为什么裸资源直接 helm install 会被拒绝？生产上怎么迁移？**
   Helm 3.2+ 的 ownership 校验防止误接管。迁移：补 `meta.helm.sh/release-name/namespace` 注解 + `app.kubernetes.io/managed-by=Helm` 标签后再 upgrade --install（零中断）；教学环境可 delete+install。

4. **--atomic 做了什么？它的局限？**
   = `--wait`（等所有资源 Ready）+ 失败/超时自动 rollback。局限：只保证"不留半成品"，不解决根因；失败后必须查原因（事件/日志）。

5. **values 覆盖优先级？`helm get values --revision 2` 为什么能查到 --set 的值？**
   `--set` > `-f` 文件 > chart 默认。因为 --set 的覆盖结果在发布时被并入该 revision 的台账。

6. **rollback 后 `helm history` 为什么是 +1 而不是"回到第 2 行"？**
   rollback 生成一个内容等于旧 revision 的新 revision；历史只增不减，保证审计连续性。

7. **为什么不能手工删 release Secret？**
   台账=release 的存在本体；删除后 helm list 不可见、rollback 无据可依（release 失忆）。清理必须 `helm uninstall`。

8. **Helm 3 为什么砍掉 Tiller？**
   Tiller 常驻集群且默认 cluster-admin，是权限单点与安全短板；Helm 3 让客户端直接用 kubeconfig 调 API，权限回归原生 RBAC。

9. **`nindent` 和 `indent` 的区别？**
   indent 只缩进已有内容；nindent 先补一个换行再缩进——用于 include 片段嵌入时保持 YAML 结构对齐，用错会导致渲染结果缩进错乱。

10. **（进阶）Chart 除了"目录直装"，还有哪些企业级分发方式？**
    ① Chart 仓库（Chart Museum / GitHub Pages 托管 index.yaml，`helm repo add` + `helm install repo/chart`）；② **OCI registry**（Helm 3.8+ GA：`helm push` 到私有 registry，本项目可直接推 `11.0.1.128:30000`，`helm install oci://...`）——与中转镜像的同一套私有仓库体系天然契合。

---

## 9. 面试要点与简历写法

### 9.1 一段话讲完本项目

> "项目 B 里我用 sed 替换 Deployment 里的镜像占位符再 kubectl apply，能用但有三个问题：没有发布历史、回滚粒度粗、失败可能留半成品。项目 E 我把发布环节 Helm 化：应用清单写成标准 Chart（参数全收敛到 values，标签三处同源根除 selector 失配），流水线 Stage 4 换成 `helm upgrade --install --set image.tag=${BUILD_NUMBER} --atomic`——发布生成 revision 可审计、一键 rollback、失败自动回滚到上一版。迁移存量资源时踩了 Helm 的 ownership 校验坑，两种接管方案（原地 annotate 迁移 / delete 重装）我都实际操作过。"

### 9.2 高频追问预案

| 追问 | 怎么回答 |
|------|-----------|
| Helm 和 kubectl apply 区别？ | 0.2 对比表：版本化 vs 状态对齐、atomic vs 半成品 |
| release 存哪里？ | 1.2（Secret + 换机可回滚） |
| Chart 怎么管理多环境？ | 1.4 优先级 + `-f prod-values.yaml`；进阶讲 helm-diff/环境目录 |
| 为什么用 Helm 不用 Kustomize？ | Helm=模板+参数+发布台账（包管理器）；Kustomize=无模板的 overlay 差异化（配置管理器）——两者解决的问题不同，可共存；本项目要的是"发布语义"，选 Helm |
| 迁移存量资源怎么做？ | 4.2 方案 B（annotate+label，亲历过 ownership 报错） |
| --atomic 是万能的吗？ | 4.5：只止损不修复，失败必须看根因 |

### 9.3 简历写法（可直接用）

> 将 CI/CD 流水线发布环节 Helm 化改造：手写应用 Chart（参数全覆盖、标签同源、HPA 扩展点预埋），流水线以 `helm upgrade --install --set image.tag --atomic` 替代 sed+apply，实现发布版本化审计、一键回滚与失败自动回退；完成裸部署资源到 Release 的零中断迁移，沉淀 ownership 校验等 4 类改造故障的排查手册。

---

## 10. 学习资源（官方文档）

| 主题 | 资源 |
|------|------|
| Helm 官方文档 | https://helm.sh/zh/docs/ （Quickstart / Charts / Chart Template Guide 三篇是核心） |
| 模板函数字典 | https://helm.sh/zh/docs/chart_template_guide/functions_and_pipelines/ |
| values 与 --set 语义 | https://helm.sh/zh/docs/chart_template_guide/values_files/ |
| OCI registry 支持 | https://helm.sh/docs/topics/registries/ |
| 练习 | 本地 kind/minikube 上 `helm create` 脚手架对照手写 Chart；Killercoda Helm 场景 |

---

## 附录 A：文件包清单与操作顺序

```
K8s项目E_Helm化改造/
├── K8s项目E_Helm化改造实战.md        # 本文档
├── Jenkinsfile                      # 改造版流水线（替换 GitHub 仓库同名文件）
└── damo-app-chart/                  # Chart（放到 GitHub 仓库 chart/damo-app/）
    ├── Chart.yaml
    ├── values.yaml
    ├── .helmignore
    └── templates/
        ├── _helpers.tpl
        ├── deployment.yaml
        ├── service.yaml
        └── NOTES.txt
```

**操作顺序总览**（细节见各阶段）：

```
阶段 2 中转 helm-kubectl 镜像
   → 阶段 4.1 lint/template/dry-run 三道体检
   → 阶段 4.2 接管裸资源（方案 A 或 B）→ 4.3 发布验证
   → 阶段 4.4 升级×2 + 回滚×1 → 4.5 --atomic 失败演练
   → 阶段 5.1 PodTemplate 换 helm 容器 → 5.2 仓库换 Jenkinsfile + 加 chart/
   → 5.3 git push 全链路验证 → 阶段 6 亮点演示截图
```

> **回退预案**：改造期间任何问题，流水线 Stage 4 临时切回项目 B 的 sed+apply 版 Jenkinsfile（`k8s/` 目录特意保留未删），5 分钟恢复原链路——**改造不锁死退路，这也是工程习惯的一部分**。

---

> **关联文档**：`K8s项目ABCD_四项目串讲总览.md`（四项目协作全景）｜damo-app GitHub 仓库 README 的 Roadmap 第一项（Helm 化）完成后打勾。
