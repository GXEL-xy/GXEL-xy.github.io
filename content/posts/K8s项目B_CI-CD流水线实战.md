---
title: "K8s 项目 B：Jenkins CI/CD 流水线实战"
date: 2026-09-01T13:43:35+08:00
draft: false
description: "在三节点 K8s 集群上部署 Jenkins Master + 动态 Agent + 自建私有镜像仓库，用 Kaniko 实现代码提交→自动构建→推送→自动部署→访问验证的 CI/CD 全自动链路。"
tags: ["k8s CI/CD"]
categories: [项目]
---

> 📦 **项目文件包**：[下载 K8s项目B_CI-CD流水线.zip](/files/K8s项目B_CI-CD流水线.zip)（含全部 YAML、damo-app 源码、Jenkinsfile、部署脚本，可直接拷到集群 master 使用）。本文件是详细操作手册，两者配合。

> **项目定位**：在自建三节点 K8s 集群（v1.35 / containerd / Ubuntu 24.04）上，部署 **Jenkins Master + 动态 Agent**，自建**私有镜像仓库 registry**，实现「**代码提交 → 自动构建镜像（Kaniko）→ 推送私有仓库 → 自动部署到 K8s → 访问验证**」全自动链路。
>
> **面试价值**：这是运维/DevOps 实习 JD 中出现频率最高的能力。做完这个项目，你能回答"CI/CD 全链路怎么落地""为什么用 Kaniko 不用 Docker-in-Docker""动态 Agent 和静态从节点区别""私有仓库怎么配"等高频面试题，且全部有动手实证。
>
> **环境假设**（请先对照检查，如有不同以实际为准）：
>
> | 项目 | 假设值 | 检查命令 |
> |------|--------|---------|
> | 集群版本 | v1.35 | `kubectl version` |
> | 节点数 | 1 master + 2 worker | `kubectl get nodes` |
> | 容器运行时 | containerd **2.x**（kubelet 用它拉镜像；⚠️ 2.x 的 registry 配置归 `io.containerd.cri.v1.images` 插件，用 config_path + hosts.toml，见 2.3） | `containerd --version` + `kubectl get nodes -o wide` |
> | 节点 docker | 三节点额外装了 docker（**K8s 不用它**，仅可辅助测试/备选构建） | `docker version` |
> | 节点系统 | Ubuntu 24.04 | `cat /etc/os-release` |
> | 外网 | 可访问 Docker Hub / GitHub | 手动验证 |
> | 存储 | local-path StorageClass 已装（项目 A 装过） | `kubectl get sc` |
> | 节点 IP | 示例 `11.0.1.128`（master） | `kubectl get nodes -o wide` |

---

## 目录

- [0. 项目全景与架构](#0-项目全景与架构)
- [阶段 1：源码与 Git 仓库准备](#阶段-1源码与-git-仓库准备)
- [阶段 2：自建私有镜像仓库 registry](#阶段-2自建私有镜像仓库-registry)
- [阶段 3：部署 Jenkins Master](#阶段-3部署-jenkins-master)
- [阶段 4：Jenkins 核心配置（Cloud + 动态 Agent + 凭据）](#阶段-4jenkins-核心配置cloud--动态-agent--凭据)
- [阶段 5：流水线应用（Jenkinsfile + Pipeline Job）](#阶段-5流水线应用jenkinsfile--pipeline-job)
- [阶段 6：全链路验证与面试演示](#阶段-6全链路验证与面试演示)
- [7. 故障速查表](#7-故障速查表)
- [8. 自测题（含答案）](#8-自测题含答案)
- [9. 面试要点与简历写法](#9-面试要点与简历写法)
- [10. 进阶（可选加分项）](#10-进阶可选加分项)
- [11. 资源清单（官方文档）](#11-资源清单官方文档)
- [附录 A：全部 YAML 汇总](#附录-a全部-yaml-汇总)
- [附录 B：YAML 落地为文件（推荐做法）](#附录-byaml-落地为文件推荐做法)
- [附录 C：Jenkinsfile 全文](#附录-cjenkinsfile-全文)

---

## 0. 项目全景与架构

### 0.1 项目路线图位置

```
项目 0 环境搭建（三节点 kubeadm 集群）     ✅ 已完成
项目 A 应用容器化部署（WordPress+MySQL）   ✅ 已完成
项目 B CI/CD 流水线（本文件）             ⬅️ 当前
项目 C 监控告警（Prometheus+Grafana）      ⏳ 下一步
加分项：故障演练 / GitOps(ArgoCD) / HA 加固
```

### 0.2 架构图

```
┌────────────────────────────────────────────────────────────────────┐
│                       你的本地电脑（浏览器）                          │
│   1. 访问 Jenkins 管理页 http://<NODE_IP>:30080                     │
│   2. 访问 damo-app 页面   http://<NODE_IP>:30090                    │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
┌──────────────────────────────▼─────────────────────────────────────┐
│                  三节点 K8s 集群（v1.35, containerd）                │
│                                                                     │
│  ┌─────────────┐   GitHub 公开仓库（damo-app 源码）                  │
│  │   Jenkins    │◄──── Poll SCM 每 5 分钟轮询（webhook 不可达原因）   │
│  │   Master     │      git clone                                    │
│  │ NodePort:    │                                                   │
│  │   30080      │──► 按需拉起动态 Agent Pod ──┐                     │
│  └─────────────┘                            │                      │
│                                             ▼                      │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │          Agent Pod（一次性，构建完销毁）                    │       │
│  │  ┌───────────┐  ┌──────────────┐  ┌────────────────┐    │       │
│  │  │  jnlp     │  │   kaniko     │  │    kubectl     │    │       │
│  │  │ 控制器/调度│  │ git clone +  │  │ apply/rollout  │    │       │
│  │  │           │  │ 镜像构建     │  │ 部署到目标NS    │    │       │
│  │  └───────────┘  └──────┬───────┘  └───────┬────────┘    │       │
│  │        共享 workspace 卷（三容器共用源码目录）             │       │
│  └────────────────────────┼────────────────────────────────┘       │
│                           │ docker push                             │
│                           ▼                                         │
│  ┌──────────────────────────────┐   拉取镜像(containerd)            │
│  │ 私有仓库 registry:2          │◄────────────────────────┐        │
│  │ NodePort: 30000              │                         │        │
│  └──────────────────────────────┘                         │        │
│  ┌─────────────────────────────────────────────────────────┴──┐    │
│  │  damo-app 命名空间：Deployment + Service(NodePort 30090)    │    │
│  │  每次构建生成新 tag → 滚动更新 → 页面显示 Build #N           │    │
│  └────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
```

### 0.3 关键技术决策（面试可讲）

| 决策点 | 本项目选择 | 为什么（面试话术） |
|--------|-----------|------------------|
| 构建工具 | **Kaniko** | ① kubelet 的运行时是 containerd，K8s 拉镜像走 containerd，与节点 docker 无关；② 若挂宿主机 docker.sock 构建，Agent Pod 只能调度到装了 docker 的节点（nodeSelector），且等于把宿主机 root 权限交给构建，安全风险大；③ DinD 需要 privileged 特权容器、每构建起一个 daemon 开销大。Kaniko 在 Pod 内**非特权**构建、直接推送 registry、与节点环境完全解耦，是 K8s 原生最佳实践 |
| 镜像仓库 | **自建 registry:2** | 私有镜像仓库是生产标配（代码不能外泄）；避免 Docker Hub 匿名拉取限流；顺便练习 containerd 私有源配置 |
| CI 服务器 | **Jenkins** | 运维 JD 出现频率最高；插件生态成熟；Kubernetes Plugin 提供动态 Agent |
| Agent 模式 | **动态 Agent** | 构建任务来时按需拉起 Pod，用完销毁，资源零闲置；比传统静态从节点省资源，是 K8s 时代标准做法 |
| 触发方式 | **Poll SCM 轮询** | 本集群是内网 VM，GitHub webhook 无法回调（无公网 IP）；轮询简单可靠。生产环境用 webhook 或 GitOps（见进阶） |
| 部署方式 | **kubectl apply + 动态替换 tag** | 直观展示部署原理；进阶可换 Helm/ArgoCD |

### 0.4 端口规划（避开项目 A 已用端口）

| 服务 | 端口 | 说明 |
|------|------|------|
| registry（NodePort） | **30000** → 5000 | 私有镜像仓库 |
| Jenkins（NodePort） | **30080** → 8080 | Jenkins 管理界面 |
| damo-app（NodePort） | **30090** → 80 | 流水线部署的演示应用 |
| （项目 A 已占用 32541） | — | ingress-nginx NodePort |

### 0.5 全程变量定义（已按你的实际环境填好）

```bash
# ========== 以下变量在 master 节点上执行 ==========
export NODE_IP=11.0.1.128                       # 你的 master 节点 IP（已确认）
export GIT_REPO=https://github.com/GXEL-xy/damo-app.git   # 你的 GitHub 仓库（已确认）
export K8S_VERSION=v1.35                        # 集群版本
```

> ⚠️ 本文所有 `<NODE_IP>` 占位符均指上面这个 `NODE_IP`（`11.0.1.128`）。

---

## 阶段 1：源码与 Git 仓库准备

> **干什么**：准备一个极简演示应用（nginx 静态页）的完整源码仓库，包含 Dockerfile、K8s 部署清单、Jenkinsfile，推送到 GitHub。
> **为什么**：CI/CD 必须有"源代码仓库"作为起点；用独立轻量应用避免引入 WordPress 复杂度，聚焦流水线本身。
> **产出**：GitHub 公开仓库 `damo-app`，含 4 个文件。

### 1.1 创建 GitHub 仓库

1. 浏览器打开 https://github.com/new
2. 仓库名：`damo-app`，**Public**（公开仓库才能让 Jenkins 匿名 clone，省去配置凭据）
3. 不要勾选"Add a README"（保持空仓库，避免冲突）
4. 点 Create repository

> 备选（国内访问 GitHub 慢）：用 Gitee 创建同名公开仓库，后面所有 `github.com` 地址换成 `gitee.com` 即可，流程完全一样。

### 1.2 写演示应用源码

在**任意一台电脑**上新建目录（或在 GitHub 网页端直接在线创建文件，选"Add file"更省事）：

```
damo-app/
├── index.html          # 静态页面，显示构建号（验证滚动更新的关键）
├── Dockerfile          # 构建 nginx 镜像
├── k8s/
│   ├── deployment.yaml # Deployment（image 含占位符，由流水线替换）
│   └── service.yaml    # NodePort 30090 暴露
└── Jenkinsfile         # 流水线定义（阶段 5 再创建，可先留空占位）
```

**1.2.1 `index.html`**（页面显示构建号，每次流水线构建后内容变化，肉眼可见验证滚动更新）：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>Demo App - CI/CD Pipeline</title>
  <style>
    body { font-family: "Microsoft YaHei", sans-serif; background: #0f172a; color: #e2e8f0;
           display: flex; height: 100vh; margin: 0; align-items: center; justify-content: center; }
    .card { text-align: center; padding: 48px; border-radius: 16px; background: #1e293b;
            box-shadow: 0 20px 60px rgba(0,0,0,.5); }
    h1 { font-size: 42px; margin: 0 0 12px; }
    .build { color: #38bdf8; font-size: 22px; font-weight: bold; }
    .meta { color: #94a3b8; font-size: 14px; margin-top: 24px; }
  </style>
</head>
<body>
  <div class="card">
    <h1>🚀 CI/CD Demo App</h1>
    <p class="build">Build #__BUILD_NUMBER__</p>
    <p>代码提交 → 自动构建 → 自动部署，全链路已验证 ✅</p>
    <p class="meta">部署时间：__BUILD_TIME__</p>
  </div>
</body>
</html>
```

> `__BUILD_NUMBER__` / `__BUILD_TIME__` 是占位符，流水线构建时用 `sed` 替换成真实构建号和构建时间（见 5.1）。

**1.2.2 `Dockerfile`**：

```dockerfile
# 基础镜像：官方 nginx alpine 版（体积小）
FROM nginx:1.27-alpine

# 把静态页面复制进 nginx 默认站点目录
COPY index.html /usr/share/nginx/html/index.html

# 容器启动时把构建号替换进页面（也可以在流水线里 sed，二选一）
# 这里不做，交给流水线在构建镜像前处理（见 Jenkinsfile）
EXPOSE 80
```

**1.2.3 `k8s/deployment.yaml`**（注意 `image` 行的占位符 `__IMAGE__`，流水线构建时动态替换）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: damo-app
  namespace: damo-app
  labels:
    app: damo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: damo-app
  template:
    metadata:
      labels:
        app: damo-app
    spec:
      containers:
        - name: damo-app
          # 占位符：流水线用 sed 替换为 11.0.1.128:30000/damo-app:<构建号>
          image: __IMAGE__
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          # 静态页应用，探针用 tcpSocket 最简单（项目 A 的血泪教训）
          startupProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30
          readinessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
```

**1.2.4 `k8s/service.yaml`**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: damo-app
  namespace: damo-app
spec:
  type: NodePort          # 用 NodePort 直接访问，不需要 Ingress（简化）
  selector:
    app: damo-app
  ports:
    - port: 80            # Service 端口
      targetPort: 80      # 容器端口
      nodePort: 30090     # 节点端口（避开项目 A 的 32541）
```

**1.2.5 推送 GitHub**：

```bash
# 在源码目录里执行（任意电脑，装好 git）
cd damo-app
git init
git add .
git commit -m "init: demo app source"
git branch -M main
git remote add origin https://github.com/GXEL-xy/damo-app.git
git push -u origin main
```

> 验证：浏览器打开 https://github.com/GXEL-xy/damo-app 能看到 4 个文件即成功。

### 1.3 完成标准（阶段 1）

| 检查项 | 通过标准 |
|--------|---------|
| GitHub 仓库 | `damo-app` 公开仓库存在，含 index.html / Dockerfile / k8s/ / Jenkinsfile 4 项 |
| 源码可 clone | `git clone https://github.com/GXEL-xy/damo-app.git` 成功 |
| 本地构建验证（可选） | 有 docker 的机器 `docker build -t damo-app:test .` 成功 |

---

## 阶段 2：自建私有镜像仓库 registry

> **干什么**：在集群里部署 registry:2（轻量私有仓库），暴露 NodePort 30000；三节点 containerd 配置信任该 http 仓库。
> **为什么**：流水线构建出的镜像要有个"中转站"；生产环境私有仓库是必备基础设施（代码/镜像不公开）；顺带练习 containerd 私有源配置（运维高频技能）。
> **怎么做**：写 3 个 YAML → apply → 改 3 个节点的 containerd 配置 → 验证。
> **怎么验证**：集群内拉取/推送测试 + 部署一个测试 Pod 成功。

### 2.1 为什么 containerd 要特殊配置

containerd 默认**只从 https 且受信任的仓库拉取镜像**。我们的 registry:2 默认走 **http**（没配 TLS 证书），所以 containerd 会拒绝拉取，报 `http: server gave HTTP response to HTTPS client`。必须在每个节点的 `/etc/containerd/config.toml` 里告诉 containerd："这个地址是 http 的，我信任它"。

### 2.2 部署 registry

> **文件化**：registry 的 4 个资源（Namespace / PVC / Deployment / Service）已写成一个文件 **`manifests/registry.yaml`**（项目文件包已提供，见「📦 项目文件包」；文件内容见下方）。部署 = 一条 `kubectl apply -f` 命令，不要再用 heredoc 管道。

**2.2.1 资源文件内容**（`manifests/registry.yaml`，4 个资源用 `---` 分隔）：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: registry
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-data
  namespace: registry
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path      # 项目 A 装过；没有就 kubectl get sc 看实际名字
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
        - name: registry
          image: registry:2.8.3
          ports:
            - containerPort: 5000
          env:
            # 允许删除镜像（生产常用，便于清理测试镜像）
            - name: REGISTRY_STORAGE_DELETE_ENABLED
              value: "true"
          volumeMounts:
            - name: data
              mountPath: /var/lib/registry
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: registry-data
---
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: registry
spec:
  type: NodePort
  selector:
    app: registry
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30000
```

**2.2.2 应用文件创建资源**（master 上执行，一条命令装全部）：

```bash
# 方式一（推荐）：用项目文件包里的文件
kubectl apply -f manifests/registry.yaml

# 方式二：没有文件包时，把上面 YAML 保存为 registry.yaml 再 apply
#   vi registry.yaml        （粘贴上面的内容）
#   kubectl apply -f registry.yaml
```

> `kubectl apply -f 文件名.yaml` 会一次创建文件里的全部资源（`---` 分隔的多个文档）。比 `cat <<'EOF' | kubectl apply -f -` 的 heredoc 管道方式好在：**文件可留档、可 git 版本管理、可反复 apply、可 diff 审查**（见附录 B）。

**2.2.3 验证 registry 起来了**：

```bash
kubectl get pods -n registry -o wide          # 应为 Running
kubectl get svc -n registry                   # NodePort 30000
curl http://11.0.1.128:30000/v2/              # 返回 {} 即正常（任何节点 IP 都行）
```

> `curl http://<NODE_IP>:30000/v2/` 返回 `{}`（或 200），说明 registry API 通了。

### 2.3 三节点 containerd 配置 insecure-registry

> **必须在 3 个节点（master + 2 worker）上都执行**。因为 Deployment 的 Pod 可能被调度到任意节点，那个节点必须能拉取私有仓库镜像。

**2.3.1 备份原配置**（每个节点执行）：

```bash
sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak.$(date +%F)
```

**2.3.2 确认 containerd 版本（决定配置方式，⚠️ 必做）**：

```bash
# 看 containerd 版本
containerd --version

# 看 config.toml 里 CRI 插件的真实路径（决定用哪段配置）
grep -n 'cri"\]' /etc/containerd/config.toml | head -5
```

判断结果（**三个节点版本应一致**）：

| grep 输出 | containerd 版本 | 用哪段配置 |
|-----------|----------------|-----------|
| `io.containerd.cri.v1.images` / `io.containerd.cri.v1.runtime` | **2.x**（v2.0 起 CRI 拆分） | **2.3.3-A（推荐 config_path + hosts.toml）** |
| `io.containerd.grpc.v1.cri` | 1.7 | 2.3.3-B |
| 无输出（精简配置） | 按版本 | 用对应方式 |

> ⚠️ **containerd 2.x 的关键变化**：① CRI 插件从 1.7 的单个 `io.containerd.grpc.v1.cri` 拆分为 `io.containerd.cri.v1.runtime`（容器生命周期）、`io.containerd.cri.v1.images`（镜像管理）、`io.containerd.cri.v1.sandbox`；② **registry 镜像仓库配置归 `io.containerd.cri.v1.images` 管**；③ 老式的 `registry.mirrors` / `registry.configs` 已**废弃**，官方推荐 `config_path` + `hosts.toml` 方式。用错插件路径或废弃写法都可能被静默忽略，表现为"配了还是拉不了 http 仓库"。

**2.3.3-A  containerd 2.x 配置（官方推荐：config_path + hosts.toml）**：

每个节点两步——**Step 1** 在 config.toml 里启用 config_path：

```bash
# 追加（如果 config.toml 已存在 [plugins."io.containerd.cri.v1.images".registry] 段，改用 vi 在其中加 config_path 行）
cat >> /etc/containerd/config.toml <<'EOF'

# ===== CI/CD 私有仓库 (containerd 2.x, config_path 方式) =====
[plugins."io.containerd.cri.v1.images".registry]
  config_path = "/etc/containerd/certs.d"
EOF
```

**Step 2** 创建 hosts.toml（目录名 = 仓库地址，`11.0.1.128` 换成你的 master IP）：

```bash
sudo mkdir -p /etc/containerd/certs.d/11.0.1.128:30000
sudo tee /etc/containerd/certs.d/11.0.1.128:30000/hosts.toml <<'EOF'
server = "http://11.0.1.128:30000"

[host."http://11.0.1.128:30000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
EOF
```

> hosts.toml 是 containerd 每个仓库一个文件的声明式配置：目录名即仓库地址，`skip_verify = true` 跳过 TLS 校验（http 仓库必需），`capabilities` 声明能力（pull/resolve 够用；push 不经 containerd，Kaniko 直推）。

**2.3.3-A'  备选（containerd 2.x 兼容写法，mirrors/configs 已废弃，不推荐）**：

```bash
cat >> /etc/containerd/config.toml <<'EOF'

[plugins."io.containerd.cri.v1.images".registry.mirrors]
  [plugins."io.containerd.cri.v1.images".registry.mirrors."11.0.1.128:30000"]
    endpoint = ["http://11.0.1.128:30000"]

[plugins."io.containerd.cri.v1.images".registry.configs]
  [plugins."io.containerd.cri.v1.images".registry.configs."11.0.1.128:30000".tls]
    insecure_skip_verify = true
EOF
```

> ⚠️ 该写法在 2.x 中已标记 DEPRECATED（仅未设 config_path 时生效），且与 config_path 同时配置会报错。**优先用 2.3.3-A 的 hosts.toml 方式**；此段仅供排查兼容问题时对照。

**2.3.3-B  containerd 1.7 配置**（mirrors/configs 在 1.7 是正常配置方式，非废弃）：

```bash
cat >> /etc/containerd/config.toml <<'EOF'

# ===== CI/CD 私有仓库 (containerd 1.7) =====
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."11.0.1.128:30000"]
    endpoint = ["http://11.0.1.128:30000"]

[plugins."io.containerd.grpc.v1.cri".registry.configs]
  [plugins."io.containerd.grpc.v1.cri".registry.configs."11.0.1.128:30000".tls]
    insecure_skip_verify = true
EOF
```

> ⚠️ 注意：如果 config.toml 里**已经存在**对应 registry 段（含 mirrors/configs/config_path），不要直接 `>>` 追加（会产生重复键、互相覆盖导致不生效）。此时用 `sudo vi /etc/containerd/config.toml` 人工编辑，在已有段内补子段/行。判断：`grep -n "registry\|certs.d" /etc/containerd/config.toml` 有输出即已存在。

**2.3.4 重启 containerd 并验证**（每个节点执行）：

```bash
sudo systemctl restart containerd
sudo systemctl status containerd --no-pager | head -5   # active (running)
```

**2.3.5 端到端测试 registry（推 + 拉）**——你的节点装了 docker，测试比 ctr 更直接：

**Step 1：让 docker daemon 信任 http 私有仓库**（每个装了 docker 的节点执行）：

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "insecure-registries": ["11.0.1.128:30000"]
}
EOF
sudo systemctl restart docker
```

> ⚠️ docker daemon 与 kubelet/containerd 完全独立，重启 docker **不影响集群**（K8s 不用它）。放心重启。

**Step 2：推送测试镜像**（任意一个节点）：

```
配置docker代理
# 1. 创建目录（如果不存在）
sudo mkdir -p /etc/systemd/system/docker.service.d

# 2. 写入新代理配置（NO_PROXY 已更新）
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf <<'EOF'
[Service]
Environment="HTTP_PROXY=http://11.0.1.1:7890"
Environment="HTTPS_PROXY=http://11.0.1.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,11.0.1.128,10.244.0.0/16"
EOF

# 3. 重载 systemd 并重启 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
或者
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://docker.xuanyuan.me",
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run"
  ]
}
配置国内镜像仓库。
```

```bash
docker pull nginx:1.27-alpine
docker tag nginx:1.27-alpine 11.0.1.128:30000/test:latest
docker push 11.0.1.128:30000/test:latest
# 输出出现 "test:latest digest: ..." 即推送成功
```

**Step 3：验证 containerd（CRI）能拉取私有镜像**（**三个节点都执行**）：

> ⚠️ **验证工具要用 `crictl`，不要用 `ctr`**：`ctr` 是 containerd 底层 CLI，**不读 CRI 插件（`io.containerd.cri.v1.images`）的 registry 配置**，且默认用 HTTPS 访问仓库（对 http 仓库会报 `server gave HTTP response to HTTPS client`），无法验证我们配的东西。`crictl` 走 CRI 接口（ImageService），与 kubelet 拉镜像完全同路径，才是正确验证方式。若未安装：`sudo apt install -y cri-tools`。

```bash
# ① crictl 配置 CRI 端点（一次性）
sudo crictl config runtime-endpoint unix:///run/containerd/containerd.sock
sudo crictl config image-endpoint   unix:///run/containerd/containerd.sock

# ② 用 crictl 拉私有镜像（验证 2.3.3 配置是否生效）
sudo crictl pull 11.0.1.128:30000/test:latest
# 成功 = 输出 "Image is up to date" 或 "Pulled image"
```

**Step 3'（可选）：终极验证——直接部署一个测试 Pod**（最贴近真实场景，只在 master 执行）：

```bash
kubectl run test-pull --image=11.0.1.128:30000/test:latest --restart=Never -n default
kubectl get pod test-pull -w        # 变为 Running 即拉取成功
kubectl delete pod test-pull        # 验证完删除
```

> 若 Step 3 的 crictl 报 `http: server gave HTTP response to HTTPS client`，说明 CRI 配置没生效：① 确认 2.3.2 的版本/路径判断正确（2.x 用 `io.containerd.cri.v1.images` + config_path + hosts.toml，1.7 用 `io.containerd.grpc.v1.cri`）；② 确认 hosts.toml 文件已创建且路径/内容正确；③ `sudo systemctl restart containerd` 已执行。

**2.3.6 简化验证**（每个节点）：

```bash
curl -s http://11.0.1.128:30000/v2/ && echo " <- registry API OK"
```

### 2.4 完成标准（阶段 2）

| 检查项 | 通过标准 |
|--------|---------|
| registry Pod | `kubectl get pods -n registry` → Running |
| registry API | 任意节点 `curl http://<NODE_IP>:30000/v2/` → `{}` |
| docker 推送测试 | 节点 `docker push 11.0.1.128:30000/test:latest` 成功 |
| containerd 拉取测试 | 三节点 `sudo crictl pull 11.0.1.128:30000/test:latest` 成功（**用 crictl 不用 ctr**，ctr 不读 CRI 配置） |
| 版本方式正确 | 2.x → `io.containerd.cri.v1.images` + config_path/hosts.toml；1.7 → `io.containerd.grpc.v1.cri`；三节点一致 |
| 三节点配置 | 3 个节点的 config.toml 都有 mirrors 段，containerd active (running) |
| 端口占用 | `ss -lntp | grep 30000` 无冲突（若 30000 被占，改 Service nodePort） |

---

## 阶段 3：部署 Jenkins Master

> **干什么**：把 Jenkins（LTS 版）以 Deployment 形式部署进集群，PVC 持久化配置，NodePort 30080 供浏览器访问。
> **为什么**：Jenkins 跑在集群里，才能天然地通过 ServiceAccount 免密访问 K8s API、按需拉起 Agent Pod；这也是"把 CI 基础设施容器化"的标准姿势（面试亮点）。
> **怎么做**：RBAC → PVC → Deployment → Service → 网页初始化。
> **怎么验证**：浏览器能打开 Jenkins 登录页。

### 3.1 资源文件（一个文件装下 Jenkins 全部资源）

> **文件化**：Jenkins 的 6 个资源（Namespace / ServiceAccount / ClusterRoleBinding / PVC / Deployment / Service）已写成一个文件 **`manifests/jenkins.yaml`**（项目文件包已提供，内容见下方）。部署 = 一条 `kubectl apply -f` 命令。

> Jenkins 需要权限：① 通过 K8s API 创建/管理 Agent Pod（Kubernetes Plugin）；② 部署应用。教学环境直接给 cluster-admin，**生产必须最小权限**（面试会被问，见 9 节）。

**资源文件内容**（`manifests/jenkins.yaml`，6 个资源用 `---` 分隔）：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: jenkins
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-home
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      securityContext:
        fsGroup: 1000          # Jenkins 容器内用户 UID 1000，保证 PVC 可写
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-jdk17
          ports:
            - containerPort: 8080
            - containerPort: 50000   # agent 连接端口（jnlp）
          env:
            - name: TZ
              value: Asia/Shanghai   # 时区（日志时间正确）
            # 初始管理员密码自动生成到 /var/jenkins_home/secrets/initialAdminPassword
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2
              memory: 2Gi
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-home
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      nodePort: 30080
    - name: jnlp
      port: 50000
      targetPort: 50000
      nodePort: 31000    # Agent 通过此端口回连 Jenkins
```

**应用文件创建资源**（master 上执行，一条命令装全部）：

```bash
# 方式一（推荐）：用项目文件包里的文件
kubectl apply -f manifests/jenkins.yaml

# 方式二：没有文件包时，把上面 YAML 保存为 jenkins.yaml 再 apply
#   vi jenkins.yaml        （粘贴上面的内容）
#   kubectl apply -f jenkins.yaml
```

> ⚠️ **Agent 回连端口 31000 很关键**：动态 Agent Pod 需要连接 Jenkins Master 的 50000 端口（jnlp 协议）。Agent 和 Master 在同一个集群内，**其实走集群内 DNS（jenkins.jenkins.svc.cluster.local:50000）最稳**，NodePort 31000 是备选。PodTemplate 配置时 Jenkins 会自动填，见 4.3。

**3.2 等待 Jenkins 就绪**：

```bash
kubectl rollout status deployment/jenkins -n jenkins
kubectl get pods -n jenkins -o wide    # Running
```

### 3.3 Jenkins 网页初始化

1. **浏览器访问**：`http://11.0.1.128:30080/`
2. **取初始密码**（master 上执行）：
   ```bash
   kubectl exec -it deploy/jenkins -n jenkins -- cat /var/jenkins_home/secrets/initialAdminPassword
   ```
3. 粘贴密码 → **Install suggested plugins**（推荐插件，网络约 3-10 分钟，耐心等）
4. 创建第一个管理员用户（自己起个账号密码，例如 admin / 你的密码）
5. 实例配置：Jenkins URL 保持默认 `http://11.0.1.128:30080/`（浏览器访问地址），保存完成

### 3.4 确认已装插件（后面要用）

进入 **系统管理 → 插件管理 → 已安装**，确认以下插件存在（推荐安装通常自带）：
`Kubernetes`、`Pipeline`、`Git`、`Credentials Binding`、`Blue Ocean`（有则加分）。

缺哪个装哪个：**系统管理 → 插件管理 → 可选插件** → 搜索名字 → 勾选 → 安装。Kubernetes 插件如果没装，**必须现在装**（阶段 4 全靠它）。

### 3.5 完成标准（阶段 3）

| 检查项 | 通过标准 |
|--------|---------|
| Jenkins Pod | Running，`rollout status` 成功 |
| 登录页 | 浏览器 `http://<NODE_IP>:30080/` 可打开 |
| 初始化 | 管理员账号创建成功，进入主界面 |
| 插件 | Kubernetes / Pipeline / Git 已安装 |

---

## 阶段 4：Jenkins 核心配置（Cloud + 动态 Agent + 凭据）

> **干什么**：配置 Kubernetes Cloud（让 Jenkins 能调 K8s API 拉起 Agent Pod）、配置 kubeconfig 凭据、配置 Agent PodTemplate（jnlp + kaniko + kubectl 三容器）。
> **为什么**：这是"动态 Agent"的核心。配好后 Jenkins 构建时会自动创建 Agent Pod，跑完自动删除——这就是 K8s 版 CI 与裸机 Jenkins 的本质区别。
> **怎么做**：UI 操作为主，步骤多但每步都有验证点。
> **怎么验证**：测试连接成功 + 手动触发一次空构建看到 Agent Pod 自动创建。

### 4.1 准备 kubeconfig 凭据（给 Agent 里的 kubectl 用）

Agent Pod 里的 kubectl 容器需要访问集群 API 来 `kubectl apply`。两种方式，**推荐方式 A**：

**方式 A（推荐）：用集群 admin 的 kubeconfig**

在 master 节点生成（admin 凭据权限最全，教学够用）：

```bash
# master 节点执行
kubectl config view --raw --minify | sed 's|server:.*|server: https://kubernetes.default.svc.cluster.local:443|' > /tmp/kubeconfig-jenkins
cat /tmp/kubeconfig-jenkins
```

> 关键点：把 `server` 改成 `https://kubernetes.default.svc.cluster.local:443`——这是集群内访问 apiserver 的地址。Agent Pod 在集群内，走这个地址最稳（不依赖节点 IP 和证书 SAN）。
> 如果 sed 没替换成功（server 行格式不同），手动改最后一行的 server 值。

然后存进 K8s Secret（在 master 执行，Jenkins 集群内可以读）：

```bash
kubectl create secret generic kubeconfig -n jenkins \
  --from-file=config=/tmp/kubeconfig-jenkins
```

**方式 B（可选）：用 Jenkins 自己的 ServiceAccount token**

```bash
# 获取 jenkins SA 的 token（v1.24+ 需要手动创建）
kubectl -n jenkins create token jenkins --duration=87600h > /tmp/sa-token
echo "https://kubernetes.default.svc.cluster.local:443" > /tmp/sa-server
kubectl create secret generic kubeconfig -n jenkins \
  --from-file=token=/tmp/sa-token --from-file=server=/tmp/sa-server
```

> 方式 B 更安全但 kubectl 容器里要用 token 拼 kubeconfig，脚本复杂些。**本文按方式 A 继续**。

### 4.2 配置 Kubernetes Cloud（Jenkins UI）

路径：**系统管理（Manage Jenkins）→ 系统配置（System）→ 最下方 Cloud 区域 → Kubernetes**（如果没有，说明 Kubernetes 插件没装成功，回到 3.6）

点 **Add a new cloud → Kubernetes**，填写：

| 字段 | 填写值 | 说明 |
|------|--------|------|
| Name | `k8s-cluster` | 自定义名称 |
| Kubernetes URL | `https://kubernetes.default.svc.cluster.local` | Jenkins 在集群内，走集群 DNS 访问 apiserver |
| Kubernetes namespace | `jenkins` | Agent Pod 创建在 jenkins 命名空间 |
| Credentials | （见下） | 选「Kubernetes Service Account」方式，无需手动填 |
| Jenkins URL | `http://jenkins.jenkins.svc.cluster.local:8080` | Agent 回连 Master 的集群内地址（**重要**） |
| Jenkins tunnel | 留空 | 用上面的 URL 即可，不走 NodePort 隧道 |

**Credentials 设置**：
点 Credentials 旁边的 **Add → Jenkins**：
- Kind 选 **Kubernetes Service Account**（使用 Jenkins 自己挂载的 SA token，无需复制粘贴）
- ID 填 `k8s-sa`
- 保存

然后回到 Cloud 配置，Credentials 选刚建的 `k8s-sa`。

**测试连接**：点 **Test Connection**，出现 `Connected to Kubernetes` 绿色提示 = 成功。
（如果报错，见 7 节故障速查表「Test Connection 失败」）

> ⚠️ **实测踩坑（2026-08-28）**：Test Connection 报 `PKIX path building failed ... unable to find valid certification path`（TLS 证书验证失败）时——因为 **Kubernetes Service Account 凭据类型不会自动带 CA 证书**，Jenkins 无法验证 apiserver 证书。两种解法：
> - **快**：勾 Cloud 配置里的 **"禁用 HTTPS 证书检查"**（Disable HTTPS certificate check），跳过证书验证（实验环境 OK，生产不推荐）
> - **稳**：凭据改用 **Secret file 类型**，上传完整 kubeconfig（含 CA 证书），Jenkins 用它连接 apiserver

### 4.3 配置 Agent PodTemplate（三容器）

在刚才的 Cloud 配置里，往下滚到 **Kubernetes Pod Template**，点 **Add Pod Template**：

**4.3.1 基本信息**：

| 字段 | 值 |
|------|-----|
| Name | `ci-agent` |
| Labels | `ci-agent`（Pipeline 用 `agent { label 'ci-agent' }` 匹配） |
| Namespace | `jenkins` |
| 容器列表 | 添加下面 3 个容器 |

**4.3.2 容器 1：jnlp（控制器容器，必须有）**

| 字段 | 值 |
|------|-----|
| Name | `jnlp` |
| Docker image | `jenkins/inbound-agent:jdk17` |
| Command / Arguments | **留空（默认）** ⚠️ |

> jnlp 容器是 Jenkins 和 Agent 之间的"信使"，负责接收任务指令、调度其他容器跑命令。**每个 PodTemplate 必须有它**。
>
> ⚠️ **血泪教训（2026-08-28 实测踩坑）**：jnlp 容器的 Command/Arguments **绝对不能覆盖**！Kubernetes 插件会自动向 jnlp 容器注入连接参数（JENKINS_URL、JNLP_SECRET 等），镜像自带的 jenkins-agent 入口负责建立与 Master 的连接。一旦把 Command/Args 改成 `sleep 999d`，容器虽然 Running 但**永远不会连接 Jenkins** → Agent 永远离线（构建日志报 `All nodes of label 'ci-agent' are offline`）。**保持默认即可**。

**4.3.3 容器 2：kaniko（镜像构建）**

| 字段 | 值 |
|------|-----|
| Name | `kaniko` |
| Docker image | `gcr.io/kaniko-project/executor:debug` |
| 命令（Command） | `/busybox/sh` |
| 参数（Args） | `-c` `sleep 999d` |
| Working directory | **留空（默认）** |
| 资源请求 | cpu 200m / memory 256Mi |

> **为什么用 debug 版 + sleep**：executor 正式版是纯静态二进制（没有 shell），无法在 Pod 里常驻；debug 版内置 busybox，可以 `sleep` 挂起让 Jenkins 容器操作。Command/Args 覆盖默认入口，否则容器一启动就跑构建直接退出。
>
> **镜像源备选**（gcr.io 拉不动时）：`quay.io/kaniko-project/executor:debug`，或自行配置镜像加速。

**4.3.4 容器 3：kubectl（部署动作）**

| 字段 | 值 |
|------|-----|
| Name | `kubectl` |
| Docker image | `alpine/k8s:1.35.0` |
| 命令（Command） | `/bin/sh` |
| 参数（Args） | `-c` `sleep 999d` |
| Working directory | **留空（默认）** |
| 资源请求 | cpu 100m / memory 128Mi |

> ⚠️ **镜像选择（2026-08-28 实测踩坑）**：
> - ❌ `bitnami/kubectl:1.35` —— Docker Hub 上**该 tag 不存在**（Bitnami 已调整镜像策略），直接 ImagePullBackOff/not found
> - ❌ `registry.k8s.io/kubectl:v1.35.0` —— 官方镜像能拉，但**没有 /bin/sh**（纯静态二进制），配了 `Command: /bin/sh` 会容器启动即 Error
> - ✅ **`alpine/k8s:1.35.0`** —— alpine 基础系统 + kubectl，自带 /bin/sh，可支撑 `sleep 999d`，实测可用
>
> 版本号尽量和集群一致（1.35.x），避免 kubectl 版本比集群旧出现兼容告警。

**4.3.5 挂载 kubeconfig（关键步骤）**

目标：让 **kubectl 容器**能读到 `/root/.kube/config`。

在 PodTemplate 里添加 **Volumes**（最下方 Pod Template Volumes → Add Volume）：

| 字段 | 值 |
|------|-----|
| Type | `Secret` |
| Secret name | `kubeconfig`（4.1 创建的） |
| Mount path | `/root/.kube/` |

> 注意：Secret volume 挂载到 Pod 层，**所有容器都能看到**。路径挂到 `/root/.kube/` 后，kubectl 容器默认就会读 `/root/.kube/config`，无需额外 KUBECONFIG 环境变量。挂载点选 `/root/.kube`（目录）而不是 `/root/.kube/config`（文件），因为 kubeconfig Secret 里的 key 名是 `config`，挂目录才能变成 `config` 文件。

**4.3.6 共享 Workspace（关键原理，别配错）**

Kubernetes Plugin 默认给 Pod 挂一个 **workspace 卷**（emptyDir），挂载路径是 `/home/jenkins/agent/workspace/<Job名>`，**所有容器共享**。Jenkins 执行 `sh` 步骤时默认工作目录（cwd）就是 workspace 路径——所以 Jenkinsfile 里**全部用相对路径**（`--context .`、`k8s/deployment.yaml`），不要写死 `/workspace` 之类不存在的绝对路径。

> ⚠️ 常见坑：如果给容器配了 Working directory `/workspace` 而 workspace 卷其实在 `/home/jenkins/agent/workspace`，容器会进到一个空目录，Kaniko 会报 `failed to resolve source metadata`。所以 4.3.3/4.3.4 的 Working directory 必须留空。

**4.3.7 保存并验证**

点 **Save**。然后做个快速验证：

1. 进 **系统管理 → 节点管理（Nodes）→ 左侧"新建节点"不用管 → 看「Clouds」区域**
2. 点 `k8s-cluster` 旁的小箭头 → **Provision**（手动拉起一个 Agent 试试）
3. 到 master 执行 `kubectl get pods -n jenkins`，应该能看到一个名字类似 `default-xxxx` 的 Pod（由 PodTemplate 生成，含 3 个容器）
4. 等它 Online（绿色），再手动删掉/等它超时回收：
   ```bash
   kubectl delete pods -n jenkins -l jenkins=slave
   ```
   （或用 UI 右上角断开）

> 如果 Provision 后 Pod 一直 Pending 或容器起不来，按 7 节排障。

### 4.4 完成标准（阶段 4）

| 检查项 | 通过标准 |
|--------|---------|
| Test Connection | 返回 Connected to Kubernetes |
| Agent Pod 可拉起 | Provision 后 `kubectl get pods -n jenkins` 出现 3 容器 Pod |
| kubeconfig 挂载 | Agent Pod 的 kubectl 容器 `kubectl get nodes` 能列出集群节点 |

---

## 阶段 5：流水线应用（Jenkinsfile + Pipeline Job）

> **干什么**：写 Jenkinsfile（声明式流水线），创建 Pipeline Job 关联 GitHub 仓库，实现：clone → Kaniko 构建镜像 → 推送私有仓库 → kubectl 部署 → 验证。
> **为什么**：Jenkinsfile 即"流水线即代码"（Pipeline as Code），放仓库里版本管理，是生产标准姿势。
> **怎么做**：先写 Jenkinsfile 推仓库，再在 Jenkins 建 Job，手动触发一次。
> **怎么验证**：构建成功，damo-app 部署出来，页面显示 Build #1。

### 5.1 编写 Jenkinsfile（推送到 GitHub）

在 `damo-app` 仓库根目录创建 `Jenkinsfile`（阶段 1 的 4 个文件之一，现在补全），内容如下：

```groovy
// ================= 全局变量（已按你的实际环境填好） =================
def REGISTRY = "11.0.1.128:30000"          // ← 私有仓库地址
def APP_NAME = "damo-app"
def NAMESPACE = "damo-app"
def GIT_BRANCH = "main"

pipeline {
    // 匹配 4.3 配置的 PodTemplate 标签
    agent { label 'ci-agent' }

    environment {
        // 本次构建的完整镜像地址：11.0.1.128:30000/damo-app:<构建号>
        IMAGE = "${REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}"
        // 页面显示的构建时间
        BUILD_TIME = sh(script: "date '+%Y-%m-%d %H:%M:%S'", returnStdout: true).trim()
    }

    stages {
        // ---------- Stage 1: 拉取代码 ----------
        stage('1. Checkout') {
            steps {
                // 从 Job 的 SCM 配置里拉代码（git clone 到共享 workspace）
                checkout scm
                echo "代码已拉取，构建号: ${env.BUILD_NUMBER}"
            }
        }

        // ---------- Stage 2: 渲染页面占位符 ----------
        stage('2. Render Index') {
            steps {
                // 把 index.html 里的 __BUILD_NUMBER__ / __BUILD_TIME__ 替换成真实值
                sh """
                    sed -i 's/__BUILD_NUMBER__/${env.BUILD_NUMBER}/g' index.html
                    sed -i 's/__BUILD_TIME__/${BUILD_TIME}/g' index.html
                """
            }
        }

        // ---------- Stage 3: Kaniko 构建镜像 ----------
        stage('3. Build Image') {
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                            --context . \
                            --dockerfile Dockerfile \
                            --destination ${IMAGE} \
                            --insecure-registry ${REGISTRY} \
                            --skip-tls-verify
                    """
                }
            }
        }

        // ---------- Stage 4: 部署到 K8s ----------
        stage('4. Deploy to K8s') {
            steps {
                container('kubectl') {
                    sh """
                        # 创建/确保命名空间
                        kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        # 把 deployment.yaml 里的 __IMAGE__ 占位符替换为本次构建的镜像地址
                        sed -i 's|__IMAGE__|${IMAGE}|g' k8s/deployment.yaml
                        # 应用全部清单
                        kubectl apply -f k8s/
                        # 等待滚动更新完成（120 秒超时）
                        kubectl rollout status deployment/${APP_NAME} -n ${NAMESPACE} --timeout=120s
                    """
                }
            }
        }

        // ---------- Stage 5: 验证 ----------
        stage('5. Verify') {
            steps {
                container('kubectl') {
                    sh """
                        kubectl get pods -n ${NAMESPACE} -o wide
                        echo "---- 访问验证 ----"
                        echo "curl http://<NODE_IP>:30090/ 应显示 Build #${env.BUILD_NUMBER}"
                    """
                }
            }
        }
    }

    // ---------- 结果通知（打印在构建日志尾部） ----------
    post {
        success {
            echo "✅ 构建部署成功: ${IMAGE}"
            echo "访问: http://<NODE_IP>:30090/"
        }
        failure {
            echo "❌ 构建失败，检查上述日志"
        }
    }
}
```

**关键点讲解（面试要能讲）**：

| 行 | 作用 |
|----|------|
| `agent { label 'ci-agent' }` | 让构建跑在 4.3 配置的动态 Agent Pod 上 |
| `container('kaniko')` / `container('kubectl')` | 指定步骤在 Pod 的哪个容器里执行（三容器协作） |
| `--insecure-registry` + `--skip-tls-verify` | 告诉 Kaniko 目标是 http 私有仓库，跳过 TLS 校验 |
| `--context .` | Kaniko 从当前工作目录（= 共享 workspace）找 Dockerfile 和源码 |
| `sed -i 's|__IMAGE__|...|'` | 部署清单的动态 tag 注入（每次构建换新 tag，强制滚动更新） |
| `imagePullPolicy: Always` | 保证 containerd 每次都拉取新 tag（同 tag 会有缓存） |
| `${env.BUILD_NUMBER}` | Jenkins 内置变量，每次构建自增，天然作为镜像 tag |

**推送到 GitHub**：

```bash
cd damo-app
git add Jenkinsfile
git commit -m "feat: add pipeline"
git push origin main
```

### 5.2 创建 Pipeline Job

1. Jenkins 首页 → **新建任务（New Item）**
2. 名称：`damo-app-pipeline`，类型选 **流水线（Pipeline）**，点 OK
3. 在 **流水线（Pipeline）** 区域：
   - **定义（Definition）**：`Pipeline script from SCM`
   - **SCM**：`Git`
   - **仓库 URL**：`https://github.com/GXEL-xy/damo-app.git`（公开仓库无需凭据）
   - **分支**：`*/main`
   - **脚本路径**：`Jenkinsfile`（默认就是，保持）
4. 点 **保存**

**5.2.1 添加轮询触发**（webhook 替代方案）：

进入 Job → **配置（Configure）** → 找到 **构建触发器（Build Triggers）** → 勾选 **Poll SCM** → 日程填：

```
H/5 * * * *
```

> 含义：每 5 分钟检查一次 GitHub，有新提交就触发构建。这是 cron 格式：分 时 日 月 周。
> 为什么不用 GitHub webhook：Jenkins 在内网 VM，GitHub 无法反向访问（无公网 IP）。面试说清楚这一点是加分项。
> 进阶：用 Gitee webhook 或 Jenkins 公网中转（见第 10 节）。

**5.2.2 保存后再等一次轮询**，或直接手动触发。

### 5.3 手动触发第一次构建

1. 进入 `damo-app-pipeline` Job → 点左侧 **立即构建（Build Now）**
2. 立即去 master 看 Agent Pod 是否被拉起：
   ```bash
   kubectl get pods -n jenkins -w
   # 应看到 default-xxxx 的 3 容器 Pod 出现，跑完自动消失
   ```
3. 回到 Jenkins 点 **#1** → **Console Output** 看日志，逐步应有：
   ```
   [Pipeline] stage (1. Checkout)
   ...
   INFO[000x] Pushed image to 11.0.1.128:30000/damo-app:1
   ...
   deployment.apps/damo-app created (或 configured)
   deployment "damo-app" successfully rolled out
   ✅ 构建部署成功
   ```
4. 浏览器访问验证：
   ```bash
   curl http://11.0.1.128:30090/        # 页面显示 Build #1
   ```
   浏览器打开 `http://11.0.1.128:30090/` 应看到深色卡片页面，显示 **Build #1**。

### 5.4 首次构建常见卡点（先自查）

| 现象 | 大概率原因 |
|------|-----------|
| Agent Pod 一直 Pending | 资源不足（低配 VM）：`kubectl describe pod -n jenkins <agent名>` 看事件；或镜像拉取慢（gcr.io） |
| kaniko 报 `failed to push` | 私有仓库地址写错/未配 `--insecure-registry`；registry Pod 没起来 |
| kubectl 报 `unable to load root certificate` / 403 | kubeconfig 没挂载对/SA 权限不足，回 4.1 重做 |
| `deployment "damo-app" not found` | 第一次是 created，第二次以后才是 configured，属正常 |
| 页面显示旧 Build 号 | 浏览器缓存；或 `imagePullPolicy` 没设 Always（回看 1.2.3） |

### 5.5 完成标准（阶段 5）

| 检查项 | 通过标准 |
|--------|---------|
| Agent 动态伸缩 | 构建时 `kubectl get pods -n jenkins` 出现 Agent Pod，结束后自动删除 |
| 镜像入库 | `curl http://<NODE_IP>:30000/v2/damo-app/tags/list` 返回构建号列表 |
| 部署成功 | `kubectl get deploy,pods -n damo-app` 正常，rollout status 通过 |
| 页面验证 | `curl http://<NODE_IP>:30090/` 显示 Build #1 |

---

## 阶段 6：全链路验证与面试演示

> **干什么**：模拟真实开发流程——改一行代码 push，验证 Jenkins 自动检测、自动构建、自动部署，页面自动更新到 Build #2。再演示回滚。
> **为什么**：这是整个项目的"高光时刻"，也是面试官最爱看的闭环证明。
> **怎么做**：改代码 → push → 等轮询 → 观察自动流水线 → 访问验证 → 回滚演示。
> **怎么验证**：不点任何按钮，页面从 Build #1 自动变 Build #2。

### 6.1 全自动链路演示（改代码 → 自动发布）

1. 修改 `index.html`，比如把标题改成 `CI/CD Demo App v2`：
   ```bash
   cd damo-app
   sed -i 's/CI\/CD Demo App/CI\/CD Demo App v2/' index.html
   git add index.html
   git commit -m "feat: v2 title"
   git push origin main
   ```
2. **什么都不用做**，等最多 5 分钟（Poll SCM 周期）。期间可以观察：
   - Jenkins Job 页面：出现 **#2** 构建，且"触发原因"显示 `Started by an SCM change`
   - master 上：`watch -n1 "kubectl get pods -n jenkins"` 看到 Agent Pod 自动拉起
3. 构建结束后刷新 `http://11.0.1.128:30090/`：
   - 页面变成 **Build #2**，标题 v2
   - 证明：**Git push → Jenkins 自动检测 → 构建新镜像 → 滚动更新 → 用户无感知访问新版本**，全链路闭环 ✅
4. 看滚动更新过程（可选，演示用）：
   ```bash
   # 构建进行中时在 master 执行，能看到新旧 Pod 交替
   kubectl get pods -n damo-app -w
   ```
   （会看到 2 个旧 Pod 逐个被新 Pod 替换，Ready 后旧 Pod 删除——这就是 Deployment 滚动更新）

### 6.2 回滚演示（面试必杀技）

```bash
# 查看发布历史
kubectl rollout history deployment/damo-app -n damo-app
# 回滚到上一个版本（Build #1 的镜像）
kubectl rollout undo deployment/damo-app -n damo-app
kubectl rollout status deployment/damo-app -n damo-app
# 验证：页面标题变回旧版本
curl http://11.0.1.128:30090/
```

> 面试讲法：滚动更新出问题时，运维能一条命令回滚，K8s 的 `rollout undo` 配合不可变镜像 tag（每次构建新 tag）就是生产级发布策略。

### 6.3 60 秒面试演示脚本（对着终端念）

```
1. 打开 GitHub 仓库 → 修改 index.html 标题
2. git push
3. 打开 Jenkins → 显示 #N 构建，触发原因 SCM change
4. 切到终端 → kubectl get pods -n jenkins -w（Agent 动态拉起/销毁）
5. 切到终端 → curl http://<NODE_IP>:30090/（页面已更新，显示新 Build 号）
6. 补一句：回滚用 kubectl rollout undo
```

### 6.4 演示要点（面试回答"项目亮点"）

| 亮点 | 怎么说 |
|------|--------|
| 全链路 | "我搭了一条完整的 CI/CD 链路：git push → Jenkins 自动检测 → Kaniko 构建镜像 → 推送私有 registry → kubectl 滚动部署 → 访问验证，全程无人值守" |
| 动态 Agent | "构建时 K8s 按需拉起 Agent Pod，跑完自动销毁，集群资源零闲置" |
| Kaniko 选型 | "集群没有 docker daemon，用 Kaniko 在 Pod 内非特权构建，比 DinD 安全" |
| 私有仓库 | "自建 registry + containerd 私有源配置，模拟生产镜像管理" |
| 发布策略 | "每次构建用 BUILD_NUMBER 做不可变 tag + imagePullPolicy: Always + rollout undo 回滚" |

---

## 7. 故障速查表

按出现频率排序。排障方法论延续项目 A 的四件套：`get` → `describe` → `logs` → `events`。

### 7.1 Jenkins 侧

| 现象 | 原因 | 排查与解决 |
|------|------|-----------|
| 浏览器打不开 30080 | Service 没起好 / NodePort 冲突 / 防火墙 | `kubectl get svc -n jenkins`；`ss -lntp \| grep 30080`；Ubuntu `sudo ufw status`，放行或关闭 |
| Test Connection 报 `Connection timed out` | Jenkins 访问 apiserver 地址错误 / 网络 | 确认 URL 是 `https://kubernetes.default.svc.cluster.local`；`kubectl exec -it deploy/jenkins -n jenkins -- wget -qO- --no-check-certificate https://kubernetes.default.svc.cluster.local/version` |
| Test Connection 报 401/403 | SA token 无效 / RBAC 不足 | 确认 3.1 的 SA + ClusterRoleBinding 已 apply；凭据选 Kubernetes Service Account 而非手动 token |
| Test Connection 报 `PKIX path building failed` | 凭据没带 CA 证书，无法验证 apiserver 证书 | 勾 **禁用 HTTPS 证书检查**（快）或凭据换 **Secret file 类型上传 kubeconfig**（稳，见 4.2 提示） |
| 构建日志 `doesn't have label 'ci-agent'` | Pod Template 没保存/没生效，Jenkins 不认识该 label | 确认 Configure System → Clouds → Pod Template 已 **Save**（最底部）；Labels 精确为 `ci-agent` 无空格 |
| 构建日志 `All nodes of label 'ci-agent' are offline` | Agent Pod 起了但 jnlp 没连上 Master | 看 `kubectl get pods -n jenkins`：镜像拉取失败（ImagePullBackOff）或 jnlp 容器问题（见下两行）|
| Agent Pod `ImagePullBackOff: ... not found` | 镜像 tag 不存在 | `crictl pull` 逐个验证；bitnami/kubectl:1.35 已确认不存在 → 用 alpine/k8s:1.35.0 |
| Pod 一直 `ContainerCreating` 不动 | 某个容器镜像拉不下来（tag 不存在或网络被墙） | `kubectl get events -n jenkins --sort-by=.lastTimestamp \| tail -15` 看卡在哪个镜像；实战踩坑：jnlp 镜像 `dkl17` 拼错（应为 jdk17）、`alpine/k8s:v1.35.0` 多了 v（该仓库 tag 不带 v，registry.k8s.io 才带 v） |
| **Pod 10 秒一轮"创建→Running→Error→删除"无限循环** | **非 jnlp 容器启动即退出**（跑镜像默认 entrypoint：kaniko 的 executor、alpine/k8s 默认命令都执行完就退）→ Pod 永不 Ready → Reaper 删 Pod 重建 | **核心规则：jnlp 容器 Command/Args 必须留空；其余容器必须显式 Command=`sleep`、Args=`infinity`**。证据：Jenkins 日志 `Reaper: Containers kaniko,kubectl were terminated`；Events 里 `Killing - Stopping container jnlp` |
| 构建 `Still waiting... 'ci-agent-xxx' is offline` | Pod 创建成功但 jnlp 连不回 Master | Cloud 配置 **Jenkins 通道（tunnel）** 填 `11.0.1.128:31000`（jnlp 的 NodePort→50000）；验证：Jenkins 日志出现 `Accepted JNLP4-connect connection` 即成功 |
| `'Jenkins' doesn't have label 'ci-agent'`（连 Pod 都不创建） | Cloud 静默失效：连续多次 UI 配置改动/保存异常后，Kubernetes Cloud 与 apiserver 的连接卡死 | 先重启 Jenkins（`kubectl rollout restart deployment/jenkins`）；仍不行则逐项检查 Pod Template 的名称/标签字段 |
| PodTemplate 保存报 `Oops! A problem occurred` | 新版 Kubernetes 插件对部分字段组合（如 jnlp 镜像留空）保存异常 | 换显式值（jnlp 镜像填 `jenkins/inbound-agent:jdk17`）；持续报错看 Jenkins 日志 `kubectl logs -n jenkins -l app=jenkins \| grep -A 30 exception` |
| useSecurity=true 但无痕窗口仍不要求登录 | securityRealm 被改成 `SecurityRealm$None`（免密排障期间的遗留），不只是 useSecurity 的问题 | 确认：`kubectl exec ... -- grep securityRealm config.xml`；修复：init.groovy.d 放置 Groovy 脚本强制 `new HudsonPrivateSecurityRealm(false)` + `createAccount()` + 重启，账号建好**必须删脚本**；写文件进容器用 `kubectl exec -i ... sh -c 'cat > xxx' < 本地文件`（kubectl cp 依赖容器 tar，可能失败） |
| Agent Pod `Error`（启动即失败） | kubectl 容器用无 shell 的镜像（registry.k8s.io/kubectl 官方镜像没有 /bin/sh）+ Command /bin/sh | kubectl 镜像换 `alpine/k8s:1.35.0`（自带 shell）|
| Agent 一直 offline 但容器 Running | **jnlp 容器 Command/Args 被覆盖成 sleep**（默认 jenkins-agent 进程没跑）| jnlp 容器 Command/Arguments 清空恢复默认（见 4.3.2 血泪教训）|
| Agent Pod 一直 Pending | 节点资源不足 / 镜像拉取慢 | `kubectl describe pod -n jenkins <agent>` 看 Events（Insufficient cpu/memory 或 ImagePullBackOff）；低配 VM 建议给 kaniko 镜像预热 `crictl pull` |
| Agent Pod 起来又一直 Waiting / 连不上 | Jenkins URL 配错，Agent 回连失败 | 确认 Cloud 的 Jenkins URL 是 `http://jenkins.jenkins.svc.cluster.local:8080`；`kubectl logs -n jenkins <agent> -c jnlp` 看报错 |
| jnlp 容器日志 `Connection refused` | 50000 端口没暴露或防火墙 | 确认 Service 有 jnlp 端口 50000；集群内走 DNS 通常不需要 NodePort |
| 构建一直排队不执行 | 没有匹配 label 的 Agent / 资源不足 | Job 的 agent label 必须和 PodTemplate Labels 一致（`ci-agent`）；看队列：Jenkins → 构建队列 |

### 7.2 构建阶段

| 现象 | 原因 | 排查与解决 |
|------|------|-----------|
| kaniko 报 `Get "https://index.docker.io/v2/": dial tcp ... connection refused`（解析出被污染 IP） | **kaniko 在 Pod 内直连 Docker Hub 拉基础镜像被墙**（国内网络常态，与节点 crictl 被墙同源） | **基础镜像私有化**：master 上 `docker pull nginx:1.27-alpine → tag → push 11.0.1.128:30000/nginx:1.27-alpine`，Dockerfile 的 FROM 改为私有仓库地址（GitHub 网页编辑最快）。一劳永逸，构建不再依赖外网 |
| kaniko 报 `failed to resolve source metadata` | 工作目录不对，找不到 Dockerfile | 确认容器 Working directory 留空（cwd 默认即共享 workspace）；kaniko 用 `--context .`；Dockerfile 在仓库根目录 |
| kaniko 报 `failed to push ... connection refused` | registry 地址不通 | `kubectl get pods -n registry`；从 Agent 容器测 `wget http://11.0.1.128:30000/v2/` |
| kaniko 报 TLS / certificate 错误 | 仓库 http 但没加 `--insecure-registry` | 确认 Jenkinsfile 的 kaniko 命令带 `--insecure-registry ${REGISTRY} --skip-tls-verify` |
| `sed` 报 Permission denied | 只读挂载 / 权限 | 确认 kaniko/kubectl 容器没把 Working directory 设到只读路径；正常 cwd（workspace）可写 |
| kubectl 报 `The connection to the server ... was refused` | kubeconfig server 地址不对 | 确认 kubeconfig 里 server 是 `https://kubernetes.default.svc.cluster.local:443` |
| kubectl 报 `Forbidden` | SA 权限不足 | 确认 ClusterRoleBinding 存在：`kubectl get clusterrolebinding jenkins-admin` |
| Stage 4 一直等 rollout | 新镜像拉不下来（ImagePullBackOff） | `kubectl describe pod -n damo-app`；多半是 containerd 没配 registry mirrors（回 2.3）或 tag 不存在 |

### 7.3 镜像与运行时侧

| 现象 | 原因 | 排查与解决 |
|------|------|-----------|
| Pod 事件 `ErrImagePull: http: server gave HTTP response to HTTPS client` | containerd 没配 http 仓库，**或 2.x 用了 1.7 的旧路径/废弃写法**（配置被静默忽略） | ① 确认版本：`containerd --version`；② 2.x 用 `io.containerd.cri.v1.images` + config_path + hosts.toml（2.3.3-A），1.7 用 `io.containerd.grpc.v1.cri`（2.3.3-B）；③ 三节点 `systemctl restart containerd` |
| `docker push` 报 `http: server gave HTTP response to HTTPS client` | docker daemon 没信任 http 仓库 | 节点改 `/etc/docker/daemon.json` 加 `insecure-registries`（见 2.3.5 Step 1）后 `systemctl restart docker` |
| Pod 事件 `ImagePullBackOff: manifest unknown` | 镜像 tag 不存在 | `curl http://<NODE_IP>:30000/v2/damo-app/tags/list` 看实际 tag |
| 页面一直是旧 Build 号 | 浏览器缓存 / 没配 Always | 强制刷新 Ctrl+F5；确认 deployment.yaml `imagePullPolicy: Always` |
| registry 数据丢了 | PVC 没挂好 | `kubectl get pvc -n registry` 看 Bound；local-path 的 PVC 节点绑定性（Pod 要调度到同节点，多副本会挂） |
| gcr.io 镜像拉不下来（超时） | 国内网络访问 gcr.io 不稳定 | 把 kaniko 镜像换成 `quay.io/kaniko-project/executor:debug` 或配镜像加速器；在节点 `crictl pull` 预热 |

### 7.4 实战排障实录（2026-08-31，一次构建跑通前的 8 连坑）

> 真实排障时间线，每个坑都对应上表一行。面试被问"你遇到过什么印象最深的故障"时，讲这条故事线。

| # | 现象 | 根因 | 修复 |
|---|------|------|------|
| 1 | Pod 卡 `ContainerCreating`，`kubectl logs -c jnlp` 报 waiting | jnlp 镜像 tag 拼写错误 `jenkins/inbound-agent:dkl17` | 改 `jdk17`；顺带发现 UI 镜像留空会触发保存 Oops，显式填写更稳 |
| 2 | Events：`alpine/k8s:v1.35.0 not found` | tag 规则混淆：alpine/k8s 不带 v，registry.k8s.io 带 v | 去 v；work2 直连 Docker Hub 被墙 → **master docker 中转私有仓库**（pull→tag→push） |
| 3 | jnlp 日志无连接记录 | Cloud 未配 Jenkins 通道 | tunnel 填 `11.0.1.128:31000`；日志出现 `Accepted JNLP4-connect` 即通 |
| 4 | Pod 10 秒一轮创建→1/3 Error→删除循环 | kaniko/kubectl 容器跑默认 entrypoint 启动即退，Reaper 删 Pod | 非jnlp 容器显式 `sleep infinity`（**jnlp 必须留空**，两者规则相反） |
| 5 | `'Jenkins' doesn't have label 'ci-agent'`，连 Pod 都不建 | 连续配置改动后 Cloud 静默失效 | 重启 Jenkins 恢复 |
| 6 | kaniko `dial tcp 31.13.83.2:443 refused`（DNS 污染 IP） | Pod 内直连 Docker Hub 拉基础镜像被墙 | 基础镜像私有化 + Dockerfile 改 FROM 私有仓库地址 |
| 7 | useSecurity=true 但无痕窗口不弹登录 | securityRealm 已是 `SecurityRealm$None` | init.groovy.d 脚本强制重建 realm + 创建管理员 + 重启 |
| 8 | 脚本进容器后账号没建成 | kubectl cp 失败（目录不存在）且失败分支日志未被发现 | `kubectl exec -i ... < 本地文件` 写入；grep 脚本 println 验证 |

**方法论沉淀**：
- Agent 类问题两大金标准：`kubectl get events -n jenkins --sort-by=.lastTimestamp`（K8s 侧）+ Jenkins 日志 grep `JNLP4-connect`/`provisioning`（Jenkins 侧）。
- Pod 反复重建 ≠ 容器重启（RESTARTS=0 时是 Pod 被插件杀掉重建），先分清再排障。
- 一次只改一个变量，改完立即验证；连续 UI 改动后 Cloud 状态可疑就重启。

---

## 8. 自测题（含答案）

**Q1：你们的 CI/CD 链路完整流程是什么？**
A1：git push 到 GitHub → Jenkins Poll SCM 检测到变更 → 拉起动态 Agent Pod（jnlp+kaniko+kubectl）→ checkout 源码 → Kaniko 构建镜像 → 推送私有 registry（11.0.1.128:30000）→ kubectl apply 部署清单（动态替换镜像 tag）→ rollout status 等待完成 → curl 验证页面。全程无人值守，约 3-5 分钟。

**Q2：为什么用 Kaniko 而不是 Docker-in-Docker（DinD）或挂 docker.sock？**
A2：三层原因：① kubelet 的运行时是 containerd，K8s 拉镜像走 containerd，节点 docker 与集群无关，挂 docker.sock 并没有"复用运行时"的好处；② 挂 docker.sock 等于把宿主机的 root 权限（docker daemon 有 root 能力）交给构建，且 Agent Pod 必须 nodeSelector 调度到装了 docker 的节点，与 Pod 的可移植性冲突；③ DinD 需要 privileged 特权容器，安全风险大，且每个构建要起一个 docker daemon、镜像层浪费严重。Kaniko 以非特权方式在 Pod 内构建镜像、直接推送 registry，与节点环境完全解耦，是 K8s 环境的标准做法。

**Q3：动态 Agent 和传统 Jenkins 从节点有什么区别？**
A3：传统从节点是常驻的虚拟机/容器，空闲也占资源。K8s 动态 Agent 是构建任务来临时由 Kubernetes Plugin 按需创建 Pod，构建完自动销毁，资源零闲置，且 Agent 环境通过 PodTemplate 完全可编程（镜像、资源、卷都是声明式的）。

**Q4：私有仓库为什么用 http 也能拉？生产应该怎么做？**
A4：开发环境用 http + containerd 的 mirrors 配置（insecure_skip_verify）省去证书管理。生产必须：registry 配 TLS 证书（或自签 CA 下发到所有节点）+ htpasswd 认证 + Deployment 配 imagePullSecret，镜像仓库不放公网。

**Q5：为什么用 Poll SCM 而不用 webhook？**
A5：Jenkins 在内网 VM，GitHub 无法反向访问（没有公网 IP），webhook 回调不到。Poll SCM 每 5 分钟轮询一次，内网环境最可靠。生产环境用公网 Git 平台 + webhook，或 GitOps（ArgoCD 监听仓库）实现秒级触发。

**Q6：每次构建用 BUILD_NUMBER 做镜像 tag 有什么好处？**
A6：不可变 tag（immutable tag）。每个版本对应唯一镜像，可以精确回滚（rollout undo 到指定 revision）、审计发布记录、避免 `latest` 覆盖导致无法回滚的问题。

**Q7：滚动更新过程中，用户会不会感知到中断？**
A7：不会。Deployment 默认 RollingUpdate：先起新 Pod，Ready 后摘流量（readinessProbe 通过），再逐步销毁旧 Pod，全程 Service 只把流量导给 Ready 的 Pod。我们的 damo-app 有 tcpSocket readinessProbe，新 Pod 就绪前不会接流量。

**Q8：镜像 tag 每次变，为什么还要 imagePullPolicy: Always？**
A8：containerd/kubelet 对镜像有本地缓存，如果 tag 相同（比如误用 latest）不会重新拉取。用唯一 tag + Always 是双保险，保证每次部署都是构建出的新镜像。

**Q9：三节点的 containerd 为什么都要配置 registry mirrors？**
A9：Deployment 的 Pod 可能被调度到任意节点，该节点的 containerd 必须能解析并拉取私有仓库镜像。只配 master 会导致 Pod 被调度到 worker 时 ImagePullBackOff。

**Q10：如果 Jenkins Master 挂了，你的 CI/CD 还能发布吗？**
A10：不能，Jenkins 是单点。这是单体 CI 的局限，也是生产引入 GitOps（ArgoCD）的动机：GitOps 里部署由集群内的 controller 监听仓库自动完成，CI 挂掉不影响已发布版本的回滚和收敛。目前项目 B 阶段 Jenkins 是教学单点，进阶部分有 ArgoCD 方案。

---

## 9. 面试要点与简历写法

### 9.1 简历项目写法（STAR 结构）

> **K8s CI/CD 流水线实战**（2026.08-2026.09）
>
> **项目背景**：为自建三节点 K8s 集群（v1.35, containerd）搭建自动化发布链路，替代手动 kubectl 部署。
>
> **核心工作**：
> - 在集群内部署 Jenkins（LTS）并配置 Kubernetes Plugin，实现构建时按需拉起三容器 Agent Pod（jnlp/kaniko/kubectl），构建完自动销毁；
> - 自建 registry:2 私有镜像仓库（NodePort 30000），配置三节点 containerd 私有源信任（mirrors + insecure_skip_verify）；
> - 编写声明式 Jenkinsfile 流水线：git clone → 页面渲染 → Kaniko 非特权构建镜像 → 推送私有仓库 → 动态替换镜像 tag 并 kubectl apply → rollout 验证，实现 git push 全自动发布（Poll SCM 每 5 分钟）；
> - 验证滚动更新与 kubectl rollout undo 回滚，沉淀故障速查表与排障文档。
>
> **项目成果**：代码提交后 3-5 分钟自动上线，页面构建号可见可审计；掌握 K8s 原生镜像构建（Kaniko）、动态 Agent、私有仓库、滚动发布全栈技能。

### 9.2 高频追问及回答

| 追问 | 回答要点 |
|------|---------|
| 流水线里 kaniko 和 kubectl 为什么是两个容器？ | 单一职责：kaniko 只构建推送镜像，kubectl 只执行部署；共享 workspace 卷交换产物。出问题只看对应容器日志，互不污染 |
| imagePullSecrets 用过吗？ | 目前私有仓库无认证（开发环境），进阶方案里有 htpasswd + imagePullSecret 的完整配置（见 10.1） |
| 怎么保证构建环境一致？ | PodTemplate 声明式定义镜像版本（如 alpine/k8s:1.35.0），构建环境版本固定，杜绝"在我机器上是好的" |
| 构建很慢怎么优化？ | Kaniko 用 `--cache` 分层缓存；镜像预热 `crictl pull`；资源充足可提升 Agent 并发 |
| 多项目怎么做？ | 一个 PodTemplate + 多个 Pipeline Job（不同仓库），或 Jenkins Shared Library 抽取公共逻辑 |

### 9.3 简历技能标签（可按实际勾选）

`Jenkins` · `CI/CD` · `Docker/Kaniko` · `私有镜像仓库` · `Kubernetes` · `kubectl` · `Git` · `滚动发布` · `Linux` · `containerd`

---

## 10. 进阶（可选加分项）

### 10.1 registry 加认证 + imagePullSecret（生产级）

```bash
# 1. 生成密码文件（在任意机器，需 docker 或 htpasswd 命令）
mkdir -p /tmp/reg-auth && cd /tmp/reg-auth
docker run --rm --entrypoint htpasswd registry:2.8.3 \
  -Bbn admin '你的密码' > htpasswd
kubectl create secret generic reg-auth -n registry \
  --from-file=htpasswd

# 2. registry Deployment 挂载认证（加 volume + env）
#    volumeMounts: /auth 挂载 reg-auth
#    env: REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm
#         REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd
#         REGISTRY_HTTP_SECRET=<随机串>

# 3. containerd 三节点加认证配置（config.toml 的 configs 段加 auth）
#    [plugins."io.containerd.grpc.v1.cri".registry.configs."11.0.1.128:30000".auth]
#      username = "admin"
#      password = "你的密码"

# 4. Deployment 加 imagePullSecret
#    kubectl create secret docker-registry regcred -n damo-app \
#      --docker-server=11.0.1.128:30000 --docker-username=admin --docker-password='你的密码'
#    deployment.yaml 的 spec.template.spec 加:
#      imagePullSecrets:
#        - name: regcred

# 5. Kaniko 推送需要凭据：挂 /kaniko/.docker/config.json（含 auth 的 docker login 结果）
```

> ⚠️ 改动较大，建议项目 B 主线跑通、演示存档后再做；面试时"讲得出方案"即可，不一定全做。

### 10.2 webhook 触发（替代轮询）

内网没有公网 IP，可用：① 内网 GitLab/Gitea（webhook 走内网可达）；② 公网跳板（frp/ngrok 映射 30080）；③ 平台级方案：GitHub Actions 推送镜像到仓库后，用 `kubectl rollout` 或 ArgoCD 监听镜像更新（见 10.3）。

### 10.3 GitOps 预告（项目 B+ 的方向）

ArgoCD：集群内 controller 监听 Git 仓库，仓库里声明"我要跑什么"，集群自动收敛到声明状态。CI（Jenkins）只负责构建镜像推仓库，CD（ArgoCD）负责部署——"Git 是唯一事实来源"。这是现代 DevOps 面试高频词，项目 C 之后可作为加分项。

### 10.4 更多演示

- 构建失败模拟：故意在 Jenkinsfile 写错命令 → 观察失败 Stage、post failure 日志、Agent 依然自动销毁
- 并发构建：同时触发 2 次构建 → 观察 Jenkins 拉起 2 个 Agent Pod
- 资源限制演练：把 Agent request 调大 → Pod Pending → 观察排队队列

### 10.5 备选构建方案：挂载 docker.sock（经典 Jenkins 教程方式）

你的节点装了 docker，如果想体验 Jenkins 经典教程的 `docker build` 方式，可以切换。**主线不推荐**（原因见 0.3 决策表：权限过大 + nodeSelector 绑定节点 + 与 containerd 运行时无关）。真要做，配置要点如下：

1. **Agent 容器镜像**：把 kaniko 容器换成带 docker CLI 的镜像（如 `docker:27-cli`），PodTemplate 里 Command/Args 覆盖为 `sleep 999d`
2. **PodTemplate 加 nodeSelector**：`kubernetes.io/hostname: worker1`（或任一装了 docker 的节点）——docker.sock 只在有 docker 的节点存在
3. **PodTemplate 加 Host Path Volume**：
   | 字段 | 值 |
   |------|-----|
   | Type | Host Path |
   | Host path | `/var/run/docker.sock` |
   | Mount path | `/var/run/docker.sock` |
4. **Jenkinsfile 构建段改为**：
   ```groovy
   stage('3. Build & Push') {
       steps {
           container('docker') {
               sh """
                   docker build -t ${IMAGE} .
                   docker push ${IMAGE}
               """
           }
       }
   }
   ```
5. 注意：docker daemon 需要信任 http 仓库（2.3.5 Step 1 的 daemon.json 已在节点配好）

**面试对比话术**："docker.sock 方案更接近 Jenkins 经典教程，但等于把宿主机 root 权限交给流水线、Agent 被绑定到特定节点；Kaniko 无特权、随处可调度、镜像构建与运行时解耦，所以我主线用 Kaniko。"

---

## 11. 资源清单（官方文档）

| 主题 | 官方资源 |
|------|---------|
| Jenkins Kubernetes Plugin | https://plugins.jenkins.io/kubernetes/ （含 PodTemplate 全部配置说明） |
| Jenkins Pipeline 语法 | https://www.jenkins.io/doc/book/pipeline/syntax/ |
| Kaniko 官方 | https://github.com/GoogleContainerTools/kaniko （`--insecure-registry` 等标志说明） |
| registry 官方 | https://distribution.github.io/distribution/ |
| containerd CRI 配置 | https://github.com/containerd/containerd/blob/main/docs/cri/registry.md （mirrors/configs 结构） |
| Deployment 滚动更新 | https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/ |
| Poll SCM / cron 语法 | https://www.jenkins.io/doc/book/pipeline/syntax/#triggers |
| Jenkins 镜像 | https://hub.docker.com/r/jenkins/jenkins |
| Kubernetes Plugin 视频（B 站） | 搜索「Jenkins Kubernetes 动态 agent」 |
| 练习平台 | Killercoda 的 Jenkins/K8s 场景、Katacoda 替代品 |

---

## 附录 A：资源文件清单与一键部署

（所有 YAML 均已落成文件，位于项目文件包 `K8s项目B_CI-CD流水线/`；正文各阶段含文件完整内容。部署一律用 `kubectl apply -f 文件`，不用 heredoc。）

```bash
# ================= 阶段 2：registry（一个文件 4 个资源） =================
kubectl apply -f manifests/registry.yaml

# ================= 阶段 3：Jenkins（一个文件 6 个资源） =================
kubectl apply -f manifests/jenkins.yaml

# ================= 阶段 4：kubeconfig 凭据（master 执行） =================
kubectl config view --raw --minify | sed 's|server:.*|server: https://kubernetes.default.svc.cluster.local:443|' > /tmp/kubeconfig-jenkins
kubectl create secret generic kubeconfig -n jenkins --from-file=config=/tmp/kubeconfig-jenkins

# ================= 阶段 5：damo-app 清单（在 damo-app/k8s/，由流水线 apply；也可手动先试） =================
kubectl apply -f damo-app/k8s/     # deployment.yaml + service.yaml（注意：deployment 里 image 是 __IMAGE__ 占位符，手动部署前先替换为 11.0.1.128:30000/damo-app:v1）
```

**文件清单**：

```
K8s项目B_CI-CD流水线/
├── manifests/registry.yaml      # 阶段2：registry 全部资源（NS+PVC+Deploy+SVC）
├── manifests/jenkins.yaml       # 阶段3：jenkins 全部资源（NS+SA+RBAC+PVC+Deploy+SVC）
├── damo-app/k8s/deployment.yaml # 阶段5：damo-app Deployment（__IMAGE__ 占位符）
├── damo-app/k8s/service.yaml    # 阶段5：damo-app Service（NodePort 30090）
└── scripts/deploy.sh            # 一键：apply registry + jenkins + 等待就绪
```

**一键部署脚本**（`bash scripts/deploy.sh`，等价于上面阶段 2 + 3）：

```bash
#!/bin/bash
# 项目B 一键部署脚本（在 master 节点执行）
set -e
cd "$(dirname "$0")/.."
echo "==> 部署 registry"
kubectl apply -f manifests/registry.yaml
echo "==> 部署 jenkins"
kubectl apply -f manifests/jenkins.yaml
echo "==> 等待就绪"
kubectl rollout status deployment/registry -n registry
kubectl rollout status deployment/jenkins -n jenkins
echo "==> 完成"
```

---

## 附录 B：YAML 落地为文件（推荐做法）

> **背景**：本项目所有 YAML 已落成文件（见项目文件包 `manifests/` 与 `damo-app/k8s/`），创建资源一律 `kubectl apply -f 文件.yaml`。本附录解释为什么坚持"文件化"以及背后的原理，方便你在生产里管理更多清单。

### B.1 原理：heredoc 的两种收尾

```bash
# 写法 1：保存到文件（把管道换成重定向 >）
cat > registry.yaml <<'EOF'
...yaml 内容...
EOF

# 写法 2：直接 apply（不落盘）
cat <<'EOF' | kubectl apply -f -
...yaml 内容...
EOF
```

> 两者只是收尾不同：`> 文件名` 写文件，`| kubectl apply -f -` 进管道。正文所有 YAML 想存文件，把每一处结尾的 `| kubectl apply -f -` 改成 `> 文件名.yaml` 即可。

### B.2 推荐做法：按阶段分文件（master 节点执行）

> 💡 项目文件包 `K8s项目B_CI-CD流水线/` 已直接提供这两个文件（`manifests/registry.yaml`、`manifests/jenkins.yaml`），**无需手动创建**，拷到 master 解压即用。下面内容保留，供无文件包时参考生成（每个文件用 `---` 分隔多个资源，`kubectl apply -f` 一次应用整个文件）：

```bash
mkdir -p ~/k8s-projects/cicd && cd ~/k8s-projects/cicd
```

**文件 1：`registry.yaml`（阶段 2 全部资源）**

```bash
cat > registry.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: registry
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-data
  namespace: registry
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
        - name: registry
          image: registry:2.8.3
          ports:
            - containerPort: 5000
          env:
            - name: REGISTRY_STORAGE_DELETE_ENABLED
              value: "true"
          volumeMounts:
            - name: data
              mountPath: /var/lib/registry
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: registry-data
---
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: registry
spec:
  type: NodePort
  selector:
    app: registry
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30000
EOF
```

**文件 2：`jenkins.yaml`（阶段 3 全部资源）**

```bash
cat > jenkins.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: jenkins
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-home
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      securityContext:
        fsGroup: 1000
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-jdk17
          ports:
            - containerPort: 8080
            - containerPort: 50000
          env:
            - name: TZ
              value: Asia/Shanghai
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2
              memory: 2Gi
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-home
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      nodePort: 30080
    - name: jnlp
      port: 50000
      targetPort: 50000
      nodePort: 31000
EOF
```

**应用（一条命令装全部）**：

```bash
kubectl apply -f registry.yaml
kubectl apply -f jenkins.yaml
```

### B.3 文件落地后的常用操作（顺便学的运维技能）

```bash
kubectl get -f registry.yaml            # 查看文件定义的资源状态
kubectl diff -f jenkins.yaml            # 对比"文件声明的状态"和"集群当前状态"的差异（apply 前审查神器）
kubectl delete -f jenkins.yaml          # 按文件删除全部资源（注意会删 PVC，谨慎）
```

> 这三个命令是"声明式管理"的日常姿势：`apply -f` 让集群收敛到文件声明的状态，`diff` 预览差异，`delete -f` 批量清理。面试问"YAML 怎么管理"就答这套。

### B.4 本项目所有 YAML 文件清单（落地后应长这样）

```
~/k8s-projects/cicd/
├── registry.yaml      # 阶段2：registry 全部资源
├── jenkins.yaml       # 阶段3：jenkins 全部资源
└── kubeconfig-secret  # 阶段4：kubectl create secret 命令生成，无 YAML 文件
```

> damo-app 的 deployment.yaml / service.yaml 在 GitHub 仓库 `k8s/` 目录里（由流水线 apply），不在这里。

---

## 附录 C：Jenkinsfile 全文

（见 5.1，`11.0.1.128` 已是你的 master IP，可整体复制）

```groovy
def REGISTRY = "11.0.1.128:30000"          // master IP（已确认）
def APP_NAME = "damo-app"
def NAMESPACE = "damo-app"

pipeline {
    agent { label 'ci-agent' }
    environment {
        IMAGE = "${REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}"
        BUILD_TIME = sh(script: "date '+%Y-%m-%d %H:%M:%S'", returnStdout: true).trim()
    }
    stages {
        stage('1. Checkout') {
            steps { checkout scm }
        }
        stage('2. Render Index') {
            steps {
                sh """
                    sed -i 's/__BUILD_NUMBER__/${env.BUILD_NUMBER}/g' index.html
                    sed -i 's/__BUILD_TIME__/${BUILD_TIME}/g' index.html
                """
            }
        }
        stage('3. Build Image') {
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                            --context . \
                            --dockerfile Dockerfile \
                            --destination ${IMAGE} \
                            --insecure-registry ${REGISTRY} \
                            --skip-tls-verify
                    """
                }
            }
        }
        stage('4. Deploy to K8s') {
            steps {
                container('kubectl') {
                    sh """
                        kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        sed -i 's|__IMAGE__|${IMAGE}|g' k8s/deployment.yaml
                        kubectl apply -f k8s/
                        kubectl rollout status deployment/${APP_NAME} -n ${NAMESPACE} --timeout=120s
                    """
                }
            }
        }
        stage('5. Verify') {
            steps {
                container('kubectl') {
                    sh "kubectl get pods -n ${NAMESPACE} -o wide"
                }
            }
        }
    }
    post {
        success {
            echo "✅ 构建部署成功: ${IMAGE}"
            echo "访问: http://<NODE_IP>:30090/"
        }
        failure { echo "❌ 构建失败，检查上述日志" }
    }
}
```

---

> **文档版本**：v1.0（2026-08-27）· 适配 K8s v1.35 / containerd / Ubuntu 24.04 / Jenkins LTS / Kaniko / registry 2.8
> **下一步**：跑通后开始项目 C（监控告警 Prometheus + Grafana）。
