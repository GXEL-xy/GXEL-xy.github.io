---
title: "K8s 实战项目 A：WordPress + MySQL 应用容器化部署"
date: 2026-08-26T22:46:29+08:00
draft: false
description: "在三节点集群上完成 Dockerfile→镜像构建推送→配置管理→MySQL 有状态应用→WordPress 无状态应用→Ingress 网络暴露→验证排障的完整 K8s 部署链路。"
tags: ["k8s基础项目"]
---

> **目标**：在三节点集群上完成「Dockerfile → 镜像构建推送 → 配置管理 → 有状态应用(MySQL) → 无状态应用(WordPress) → 网络暴露(Ingress) → 验证排障」完整链路。
> **组织方式**：按实施顺序分 **4 个阶段**，每个阶段 = 干什么 / 为什么 / 怎么做 / 怎么验证。
> **选型理由**：WordPress + MySQL 是覆盖 K8s 知识点最全的组合——MySQL 用 StatefulSet/PV/PVC（有状态 + 存储），WordPress 用 Deployment/ConfigMap/Secret/Ingress（无状态 + 配置 + 网络）。
> **前置条件**：三节点集群健康已验证（`kubectl get nodes` 3/3 Ready）。
> **权威依据**：Kubernetes 官方文档（kubernetes.io/zh-cn/docs）、Docker 官方文档（docs.docker.com）。

---

## 一、项目总览

### 1.1 架构

```
                         ┌────────────────────────────┐
 用户浏览器 ──HTTP──▶   │  Ingress (nginx-ingress)   │
                         │  http://blog.example.com   │
                         └─────────────┬──────────────┘
                                       │
                         ┌─────────────▼──────────────┐
                         │ Service: wordpress (ClusterIP)│
                         └─────────────┬──────────────┘
                                       │
                         ┌─────────────▼──────────────┐
                         │ Deployment: wordpress      │
                         │   ├─ Pod 1 (node1)         │
                         │   └─ Pod 2 (node2)         │  ← 滚动更新/自愈
                         └─────────────┬──────────────┘
                                       │ 访问 MySQL（集群内 DNS）
                         ┌─────────────▼──────────────┐
                         │ Headless Service: mysql    │
                         └─────────────┬──────────────┘
                                       │
                         ┌─────────────▼──────────────┐
                         │ StatefulSet: mysql (Pod 1) │
                         │   └─ PVC ──▶ StorageClass  │  ← 数据持久化
                         └────────────────────────────┘
```

### 1.2 覆盖知识点清单（与技能清单对应）

| 技能点 | 本项目体现 | 所在阶段 |
|--------|-----------|---------|
| Docker 镜像工程化 | 多阶段构建、镜像瘦身、私有仓库 | 阶段 2 |
| ConfigMap / Secret | wp 配置参数化、数据库密码 | 阶段 1/2 |
| 存储 PV / PVC / StorageClass | MySQL 持久化、动态供给 | 阶段 1 |
| StatefulSet | MySQL 有状态部署、Headless Service | 阶段 1 |
| Deployment | WordPress 滚动更新、探针、资源限制 | 阶段 2 |
| Service | ClusterIP、Headless、NodePort 调试 | 阶段 1/2 |
| Ingress | 域名路由、TLS 可选 | 阶段 3 |
| 排障四件套 | 全程用于定位问题 | 全程 |

### 1.3 目录结构（建议固定，面试可展示）

```
~/k8s-projects/wordpress/
├── docker/                      # 镜像构建相关
│   ├── Dockerfile
│   └── .dockerignore
├── k8s/                         # 所有 YAML 清单
│   ├── 00-namespace.yaml
│   ├── 01-configmap.yaml
│   ├── 02-secret.yaml
│   ├── 03-storageclass.yaml
│   ├── 04-mysql-statefulset.yaml
│   ├── 05-wordpress-deployment.yaml
│   ├── 06-service.yaml
│   └── 07-ingress.yaml
├── README.md                    # 架构图 + 部署步骤
└── troubleshooting.md           # 故障演练记录
```

### 1.4 实施流程总览（先看这个）

> 不要按 Step 1→9 线性走。**按依赖倒序 + 每阶段可验证**推进，每阶段结束都有一个能演示的结果。

```
阶段 0 · 环境地基检查 ──▶ 阶段 1 · MySQL 数据库先行 ──▶ 阶段 2 · WordPress 应用上线 ──▶ 阶段 3 · Ingress 暴露 + 全链路验证
```

**为什么是这个顺序（面试会问）**：

| 原则 | 说明 |
|------|------|
| 依赖倒置 | 先部署被依赖的（MySQL），再部署依赖它的（WordPress），否则 WordPress 一启动就连接失败 |
| 风险前置 | 最容易失败的环节（存储、私有仓库拉镜像）放最前面做，失败成本最低；最后只剩"锦上添花"步骤 |
| 配置先行 | ConfigMap/Secret 先定义好，Deployment 直接引用，避免改配置反复重启 Pod |

### 1.5 实施 Check 清单（可打印，边做边勾）

| ☐ | 阶段 | 检查项 | 命令/操作 | 通过标准 |
|----|------|--------|----------|---------|
| ☐ | 0 | 节点就绪 | `kubectl get nodes -o wide` | 3/3 Ready |
| ☐ | 0 | StorageClass | `kubectl get sc` | 存在且 provisioner 正常 |
| ☐ | 0 | Ingress Controller | `kubectl get pods -n ingress-nginx` | controller Running |
| ☐ | 0 | 工具链 | `docker`/`kubectl`/`helm` 各 `--version` | 均可用 |
| ☐ | 1 | Namespace | `kubectl get ns wordpress` | Active |
| ☐ | 1 | Secret | `kubectl get secret -n wordpress` | mysql-secret 存在 |
| ☐ | 1 | PVC 绑定 | `kubectl get pvc -n wordpress` | data-mysql-0 **Bound** |
| ☐ | 1 | MySQL 可连 | `kubectl exec -it mysql-0 -- mysql -uroot -p... -e "SHOW DATABASES;"` | 能列出库 |
| ☐ | 1 | 持久化演示 | `kubectl delete pod mysql-0` 后重建 | 数据仍在 |
| ☐ | 2 | 配置生效 | `kubectl get cm,secret -n wordpress` | 两者存在且内容正确 |
| ☐ | 2 | 双副本运行 | `kubectl get pods -n wordpress -o wide` | 2 Running 且不同节点 |
| ☐ | 2 | Service 端点 | `kubectl get endpoints -n wordpress` | 有 Pod IP |
| ☐ | 2 | 滚动更新 | `kubectl set image` + `rollout status` | 0 中断完成 |
| ☐ | 2 | 自愈 | `kubectl delete pod -l app=wordpress` | 自动重建 Running |
| ☐ | 3 | Ingress 规则 | `kubectl get ingress -n wordpress` | 规则存在 |
| ☐ | 3 | 域名访问 | `curl --resolve blog.example.com:80:<节点IP> http://blog.example.com/` | 返回 WordPress 页面 |
| ☐ | 3 | 数据库建表 | `kubectl exec -it mysql-0 -- mysql -e "USE wordpress; SHOW TABLES;"` | wp_posts 等表存在 |
| ☐ | 3 | 文档沉淀 | README.md + troubleshooting.md | 完成 |

> 每完成一阶段就 `kubectl get all` 全览一次并把结果记进 `troubleshooting.md`。中途停下也能随时讲"我做到哪、验证了什么"。

---

## 二、阶段 0：环境地基检查

### 2.1 本阶段四要素

| 要素 | 内容 |
|------|------|
| **干什么** | 确认集群健康、StorageClass 就绪、Ingress Controller 已装、工具链齐全 |
| **为什么** | 这三件事是所有后续步骤的"基础设施服务"：没 SC 则 PVC 永远 Pending；没 Ingress 则阶段 3 无法做；工具缺了中途才发现会打断节奏 |
| **怎么做** | 见 2.2-2.5 四步 |
| **怎么验证** | 3/3 Ready；SC 出现且 local-path-provisioner Pod 正常；ingress-nginx controller Running |

### 2.2 确认集群健康

```bash
kubectl get nodes -o wide

# 系统组件全部 Running（coredns / calico / kube-proxy）
kubectl get pods -A | grep -v Running

# DNS 验证（K8s 网络模型核心）
kubectl run test --image=busybox --restart=Never -- sleep 3600
kubectl exec test -- nslookup kubernetes.default.svc.cluster.local
kubectl delete pod test
```

### 2.3 检查/安装 StorageClass

```bash
kubectl get sc
# 云厂商托管集群一般有默认 SC（如 alicloud-disk-*、standard）→ 直接跳过
# 自建集群通常没有 → 安装 local-path-provisioner

kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

# 验证
kubectl get sc
# local-path   rancher.io/local-path   Delete   Immediate
kubectl get pods -n local-path-storage
```

> **local-path 与 NFS 的选择**：测试环境用 local-path（5 分钟装好，数据在节点本地）；生产多节点共享存储用 NFS Provisioner（数据可跨节点迁移）。本项目先用 local-path，阶段 3 末尾给出 NFS 升级路径。

### 2.4 安装 Ingress Controller（nginx-ingress）

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

kubectl get pods -n ingress-nginx -w        # 等 controller Running
kubectl get svc -n ingress-nginx            # 记录外部端口（NodePort）
```

### 2.5 确认工具链

```bash
docker --version
kubectl version --client
helm version
```

### 2.6 阶段 0 完成标准

- [ ] `kubectl get nodes` → 3/3 Ready
- [ ] `kubectl get sc` → local-path 存在且正常
- [ ] `kubectl get pods -n ingress-nginx` → controller Running
- [ ] docker / kubectl / helm 可用

---

## 三、阶段 1：MySQL 数据库先行

### 3.1 本阶段四要素

| 要素 | 内容 |
|------|------|
| **干什么** | 创建 Namespace → Secret（密码）→ StorageClass 声明 → StatefulSet（含 Headless Service + volumeClaimTemplates） |
| **为什么** | MySQL 是 WordPress 的强依赖，必须先有库；且 StatefulSet+PV 是技术含量最高的部分，先啃硬骨头 |
| **怎么做** | 按 3.2 → 3.3 → 3.4 → 3.5 顺序执行（StatefulSet 引用了 Secret，顺序不能反） |
| **怎么验证** | PVC **Bound**；能进 MySQL；**删 Pod 数据不丢**（关键演示） |

### 3.2 步骤 1：创建 Namespace

```yaml
# k8s/00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
  labels:
    app: wordpress
    env: prod
```

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl get ns wordpress
```

> **为什么用 Namespace**：资源隔离 + 权限边界（RBAC 按 ns 授权）+ 清理方便（`kubectl delete ns wordpress` 一键删除全部）。

### 3.3 步骤 2：创建 Secret（数据库密码）

```yaml
# k8s/02-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress
type: Opaque
stringData:                  # stringData 自动帮你 base64，写起来更安全直观
  MYSQL_ROOT_PASSWORD: "RootPass@2026"
  MYSQL_PASSWORD: "WpPass@2026"
  WORDPRESS_DB_PASSWORD: "WpPass@2026"
```

```bash
kubectl apply -f k8s/02-secret.yaml
# 记住：Secret 只是 base64 编码，不是加密！
kubectl -n wordpress get secret mysql-secret -o yaml
echo -n "V3BQYXNzQDIwMjY=" | base64 -d   # 可逆，生产要用 KMS/Vault
```

### 3.4 步骤 3：声明 StorageClass（如无默认 SC）

```yaml
# k8s/03-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-local
provisioner: rancher.io/local-path   # 使用 local-path 的 provisioner
reclaimPolicy: Delete                # 删除 PVC 时是否删数据：Delete/Retain
volumeBindingMode: Immediate
```

> **reclaimPolicy 面试点**：`Delete` = PVC 删除时 PV 和数据一起删（测试环境）；`Retain` = 保留数据，需手动处理（生产数据库环境）。

### 3.5 步骤 4：MySQL StatefulSet（核心）

**为什么 MySQL 用 StatefulSet 而不是 Deployment**：

| 对比 | Deployment | StatefulSet |
|------|-----------|-------------|
| Pod 名称 | 随机后缀（web-abc123） | 有序稳定（mysql-0） |
| 网络标识 | 不稳定 | 稳定 DNS（mysql-0.mysql） |
| 存储 | 共享 PVC 或临时 | **每副本独立 PVC（volumeClaimTemplates）** |
| 启动/停止顺序 | 并行 | 有序（0→1→2） |
| 适用 | 无状态应用 | 数据库/消息队列等有状态应用 |

```yaml
# k8s/04-mysql-statefulset.yaml（含 Headless Service + StatefulSet）
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  clusterIP: None              # Headless：无 VIP，直接暴露 Pod IP
  selector:
    app: mysql
  ports:
    - port: 3306
      name: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: wordpress
spec:
  serviceName: mysql              # 必须关联 Headless Service
  replicas: 1                     # 生产主从需 ≥2，本项目 1 足够
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          args:
            - --character-set-server=utf8mb4
            - --collation-server=utf8mb4_unicode_ci
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              value: wordpress
            - name: MYSQL_USER
              value: wpuser
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          readinessProbe:           # 就绪探针：mysqladmin ping 通了才接流量
            exec:
              command:
                - sh
                - -c
                - mysqladmin ping -h127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:            # 存活探针：失败重启
            exec:
              command:
                - sh
                - -c
                - mysqladmin ping -h127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD
            initialDelaySeconds: 60
            periodSeconds: 10
  volumeClaimTemplates:            # 每个副本自动生成独立 PVC（StatefulSet 特有）
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-local
        resources:
          requests:
            storage: 5Gi
```

> **Headless Service 作用**：`mysql-0.mysql.wordpress.svc.cluster.local` 直接解析到 Pod IP。有状态应用的每个副本都有独立稳定 DNS。WordPress 在同一 ns 用 `mysql` 就能连上。

### 3.6 步骤 5：验证 MySQL

```bash
kubectl apply -f k8s/04-mysql-statefulset.yaml
kubectl -n wordpress get sts,pods,pvc -o wide

# 预期结果
# sts/mysql       1/1
# pod/mysql-0     1/1 Running
# pvc/data-mysql-0   Bound

# 测试连接（进 Pod 执行）
kubectl -n wordpress exec -it mysql-0 -- mysql -uroot -p'RootPass@2026' -e "SHOW DATABASES;"

# 验证持久化（关键演示！）
kubectl -n wordpress delete pod mysql-0    # 删掉 Pod
kubectl -n wordpress get pod mysql-0       # StatefulSet 自动重建
kubectl -n wordpress exec -it mysql-0 -- mysql -uroot -p'RootPass@2026' -e "SHOW DATABASES;"
# 数据还在 → 因为 PVC 是独立的，Pod 重建只是重新挂载
```

### 3.7 阶段 1 完成标准

- [ ] `kubectl get pvc -n wordpress` → data-mysql-0 **Bound**
- [ ] `kubectl exec -it mysql-0 -- mysql ...` → 能 SHOW DATABASES
- [ ] 删除 mysql-0 后自动重建，**数据仍在**（记进 troubleshooting.md）

---

## 四、阶段 2：WordPress 应用上线

### 4.1 本阶段四要素

| 要素 | 内容 |
|------|------|
| **干什么** | 镜像准备（官方镜像或自研）→ ConfigMap（非敏感参数）→ Deployment（引用 CM/Secret + initContainer + 探针 + 资源限制）→ Service |
| **为什么** | 配置先于应用；initContainer 等 MySQL 就绪避免 CrashLoopBackOff（本项目最大坑之一）；探针保证流量只打到健康 Pod |
| **怎么做** | 按 4.2 → 4.3 → 4.4 → 4.5 顺序执行 |
| **怎么验证** | 双副本 Running 且跨节点；Endpoints 有 Pod IP；滚动更新 0 中断；删 Pod 自动重建 |

### 4.2 步骤 1：镜像准备（两条路径）

**路径 A：直接用官方镜像（本项目主路径）**

WordPress 官方镜像已含完整 PHP + Apache，直接引用即可，无需自己构建：

```yaml
# 在 4.4 的 Deployment 中引用
image: wordpress:6.5
```

**路径 B：自研应用镜像（技能展示，面试加分）**

> 面试官问"你会写 Dockerfile 吗"时展示。核心是**多阶段构建**：构建阶段有编译工具链（几百 MB），运行阶段只拷产物（几 MB）。

```dockerfile
# ===== 阶段 1：构建 =====
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .

# ===== 阶段 2：运行（只拷贝构建产物） =====
FROM alpine:3.20
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/myapp .
USER appuser                    # 非 root 运行（安全最佳实践）
EXPOSE 8080
ENTRYPOINT ["./myapp"]
```

```dockerfile
# .dockerignore（减小构建上下文）
.git
node_modules
*.md
docker/
k8s/
```

**镜像瘦身检查清单**：

| 手段 | 效果 | 示例 |
|------|------|------|
| 多阶段构建 | 体积减 80%+ | 如上 |
| 小基础镜像 | 300MB→50MB | `alpine` / `distroless` 替代 `ubuntu` |
| `.dockerignore` | 减少构建上下文 | 排除 .git、node_modules |
| 合并 RUN 指令 | 减少层数 | `RUN apt update && apt install -y x && rm -rf /var/lib/apt/lists/*` |

**推送私有仓库 + imagePullSecret（面试必考）**：

```bash
# 起本地 registry（轻量，实验/内网用）
docker run -d --name registry \
  -p 5000:5000 \
  -v /opt/registry:/var/lib/registry \
  --restart=always \
  registry:2

# 打 tag 并推送
docker tag wordpress:6.5 <registry-ip>:5000/wordpress:6.5
docker push <registry-ip>:5000/wordpress:6.5

# 创建 imagePullSecret（kubelet 拉镜像时的认证凭据）
kubectl -n wordpress create secret docker-registry regcred \
  --docker-server=<registry-ip>:5000 \
  --docker-username=admin \
  --docker-password=yourpassword

# Deployment 中引用
# spec.template.spec.imagePullSecrets:
#   - name: regcred
```

> ⚠️ **自建集群拉私有仓库报 ImagePullBackOff 的经典坑**：K8s 默认用 HTTPS 访问 registry，本地 registry 是 HTTP。每个节点都要配置 containerd 信任该非安全仓库（`/etc/containerd/config.toml` 设 insecure registry，`systemctl restart containerd`），或 docker daemon.json 配 `insecure-registries`。

### 4.3 步骤 2：创建 ConfigMap（非敏感配置）

```yaml
# k8s/01-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-config
  namespace: wordpress
data:
  WP_DEBUG: "false"
  WORDPRESS_DB_HOST: mysql            # 对应 Headless Service 名（阶段 1）
  WORDPRESS_DB_NAME: wordpress
  WORDPRESS_DB_USER: wpuser
  WORDPRESS_DB_PORT: "3306"
```

```bash
kubectl apply -f k8s/01-configmap.yaml
```

**ConfigMap/Secret 注入方式对比**：

| 方式 | 用法 | 何时用 |
|------|------|--------|
| 环境变量 | `envFrom` / `env.valueFrom` | 应用读环境变量 |
| 文件挂载 | `volumeMounts` + `volumes.configMap` | 应用读配置文件 |
| 命令行参数 | `command` 引用 | 少用 |

> **面试点**：ConfigMap/Secret 更新后，**已运行的 Pod 不会自动拿到新值**（环境变量方式需重启 Pod）。部署时用 `kubectl rollout restart deployment/wordpress` 让新配置生效。

### 4.4 步骤 3：WordPress Deployment（核心）

```yaml
# k8s/05-wordpress-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 2                      # 双副本：演示滚动更新 + 负载
  selector:
    matchLabels:
      app: wordpress
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # 更新时最多多出 1 个 Pod
      maxUnavailable: 0            # 更新时不允许少于期望副本（0 中断）
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      initContainers:              # 初始化容器：等 MySQL 就绪再启动主容器
        - name: wait-for-mysql
          image: busybox:1.36
          command:
            - sh
            - -c
            - until nc -z mysql 3306; do echo "waiting mysql..."; sleep 2; done
      containers:
        - name: wordpress
          image: wordpress:6.5
          imagePullPolicy: IfNotPresent
          envFrom:
            - configMapRef:
                name: wordpress-config
          env:
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: WORDPRESS_DB_PASSWORD
          ports:
            - containerPort: 80
              name: http
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          startupProbe:              # 启动保护（实测最终方案）：最多 50s，期间 liveness 暂停
            tcpSocket:
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 10
            timeoutSeconds: 5
          readinessProbe:          # 就绪探针：端口在监听即接流量
            tcpSocket:
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 6
            timeoutSeconds: 5
          livenessProbe:           # 存活探针：端口在监听即存活
            tcpSocket:
              port: 80
            initialDelaySeconds: 60  # 给足冷启动时间
            periodSeconds: 10
            failureThreshold: 6
            timeoutSeconds: 5
```

> ⚠️ **本项目实测踩坑（必读，完整演进史）**：kubelet 的 httpGet 探针默认 `timeoutSeconds: 1`，WordPress 冷启动页面（PHP 冷启动 + 连 MySQL + 渲染）响应常超 1s → liveness 误杀 → 反复 Killing。演进过程：① `timeoutSeconds: 5` → 仍被杀（页面 >5s）；② 加 `startupProbe` + timeout 10 → 探针跟随 301 到 install.php 仍挂起；③ 探针 port 误写 NodePort 32541 → connection refused；④ **最终方案：三个探针全部改 `tcpSocket`（只管端口监听，不依赖页面响应）** → 稳定 1/1。排查线索：`describe` 事件 `context deadline exceeded` = 探针超时；`connection refused` = 端口没监听（进程没起或探针 port 写错）。方法论：**探针管"进程存活"（tcpSocket），页面健康交给 curl/监控**。

**三种探针对比（面试必问）**：

| 探针 | 失败后果 | 用途 |
|------|---------|------|
| livenessProbe | 重启容器 | 应用假死（进程在但无响应） |
| readinessProbe | 摘除流量（不重启） | 应用启动慢、依赖未就绪 |
| startupProbe | 同 liveness | 启动期保护（避免慢启动被误杀） |

### 4.5 步骤 4：创建 Service

```yaml
# k8s/06-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  type: ClusterIP               # 集群内访问，外部经 Ingress（阶段 3）
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: http
```

```bash
kubectl apply -f k8s/06-service.yaml
# 验证 Service 与 Endpoints（排障关键！）
kubectl -n wordpress get svc,endpoints
# ENDPOINTS 列有 Pod IP → selector 匹配正确
# ENDPOINTS 为空 → selector 或标签写错了（高频坑）
```

**Service 类型速查**：

| 类型 | 访问方式 | 适用 |
|------|---------|------|
| ClusterIP | 集群内 VIP | 内部服务间调用 |
| NodePort | 每个节点 IP:30000-32767 | 测试环境对外 |
| LoadBalancer | 云厂商 LB | 公有云生产 |

### 4.6 步骤 5：部署 + 滚动更新/回滚/自愈演示

```bash
# 部署
kubectl apply -f k8s/01-configmap.yaml -f k8s/05-wordpress-deployment.yaml -f k8s/06-service.yaml
kubectl -n wordpress get deploy,rs,pods -o wide

# 验证滚动更新（面试演示）
kubectl -n wordpress set image deployment/wordpress wordpress=wordpress:6.6
kubectl -n wordpress rollout status deployment/wordpress   # 等待完成
kubectl -n wordpress rollout history deployment/wordpress  # 查看发布历史
kubectl -n wordpress rollout undo deployment/wordpress     # 回滚到上一个版本

# 验证自愈
kubectl -n wordpress delete pod -l app=wordpress   # 干掉一个 Pod
kubectl -n wordpress get pods                       # 立即重建新 Pod
```

### 4.7 阶段 2 完成标准

- [ ] `kubectl get pods -n wordpress -o wide` → 2 Running 且分布在**不同节点**
- [ ] `kubectl get endpoints -n wordpress` → 有 Pod IP
- [ ] `kubectl rollout status` → 滚动更新 0 中断完成
- [ ] 删除一个 Pod → 自动重建（自愈）

---

## 五、阶段 3：Ingress 暴露 + 全链路验证

### 5.1 本阶段四要素

| 要素 | 内容 |
|------|------|
| **干什么** | Ingress 资源（域名路由）→ 全链路验证 → 写 `troubleshooting.md` 和 README |
| **为什么** | 暴露层只依赖前面所有东西，最后做；文档是项目转化为简历产出物的关键环节 |
| **怎么做** | 按 5.2 → 5.3 → 5.4 → 5.5 → 5.6 顺序执行 |
| **怎么验证** | curl 返回 WordPress 安装页（全链路打通）；MySQL 出现 wp_ 表；文档完成 |

### 5.2 步骤 1：创建 Ingress 资源

```yaml
# k8s/07-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress
  namespace: wordpress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 50m   # 上传限制（WordPress 常见）
spec:
  ingressClassName: nginx
  rules:
    - host: blog.example.com          # 域名（本地测试用 --resolve 或改 hosts）
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wordpress
                port:
                  number: 80
```

```bash
kubectl apply -f k8s/07-ingress.yaml
kubectl -n wordpress get ingress
```

### 5.3 步骤 2：域名访问验证（无需真实 DNS 的技巧）

```bash
# 查看 ingress-nginx 的 NodePort
kubectl get svc -n ingress-nginx

# 方式 1：Host 头模拟
curl -H "Host: blog.example.com" http://<任一节点IP>:<ingress-nginx NodePort>/ -v

# 方式 2：--resolve（推荐，等价于临时 DNS）
curl --resolve blog.example.com:80:<节点IP> http://blog.example.com/
```

**通过标准**：返回 WordPress 安装页 → **全链路打通**（浏览器 → Ingress → Service → Deployment → MySQL）。

### 5.4 步骤 3：全链路验证清单

| 验证项 | 命令 | 通过标准 |
|--------|------|---------|
| 全部资源 | `kubectl -n wordpress get all` | 无异常状态 |
| Pod 分布 | `kubectl get pods -n wordpress -o wide` | wordpress 分布在 ≥2 节点 |
| 持久化 | 删除 mysql-0 后数据仍在 | 见 3.6 |
| 滚动更新 | `rollout status` | 0 中断完成 |
| 自愈 | 删 Pod 自动重建 | 秒级恢复 |
| 网络链路 | curl Host 头访问 | 返回 WordPress 页面 |
| 数据库建表 | `kubectl exec -it mysql-0 -- mysql -e "USE wordpress; SHOW TABLES;"` | wp_posts 等表存在 |

### 5.5 步骤 4：排障四件套 + 高频故障速查

**排障四件套（贯穿全程）**：

```
kubectl get <资源>              # 看状态：Pending/Running/CrashLoopBackOff
kubectl describe <资源>         # 看事件：为什么失败（Events 段最关键）
kubectl logs <pod>              # 看应用日志（-f 实时、--previous 看上次崩溃）
kubectl get events --sort-by=.lastTimestamp   # 全集群事件
```

**高频故障速查**：

| 现象 | 原因 | 排查与解决 |
|------|------|-----------|
| **Pending** | 资源不足/调度失败 | `describe pod` 看 Events；`kubectl get nodes` 看资源；检查污点 |
| **ImagePullBackOff** | 镜像拉取失败 | `describe pod` 看原因；私有仓库检查 imagePullSecret、insecure-registry、镜像 tag |
| **CrashLoopBackOff** | 启动即崩溃 | `kubectl logs pod --previous`；常见：连 DB 失败、密码错、配置错 |
| **Service 不通** | Endpoints 为空 | `kubectl get endpoints`；selector 标签不匹配是头号原因 |
| **MySQL 连不上** | DNS/网络/认证 | Pod 内 `nslookup mysql`、`nc -z mysql 3306`；看 wp-config 参数 |
| **PVC Pending** | 无 StorageClass | `kubectl get pvc` 看事件；确认 SC 存在且 provisioner 正常 |
| **readiness 探针失败** | 应用慢/路径错 | `describe pod` 看探针日志；调整 initialDelaySeconds 或探针路径 |
| **Pod 反复 Killing（探针误杀）** | **httpGet 探针默认 timeoutSeconds=1**，页面响应超 1s 被误判；页面 301→install.php 挂起、探针 port 误写 NodePort 也会叠加 | **最终方案：三个探针改 `tcpSocket`（只管端口）**；演进：timeoutSeconds 5 → startupProbe → tcpSocket（详见 4.4 踩坑说明） |
| **探针端口写错** | 探针 port 误写 NodePort（如 32541）而非 containerPort（80） | 事件显示 `Get "http://PodIP:32541/"` + refused → 探针 port 改回 80 |
| **探针跟随 301 挂起** | GET / 被 WordPress 301 到 install.php，kubelet 跟随后该页执行 >timeout | 日志只见 301 无 install.php 完成记录 → 改 tcpSocket 探针 |
| **浏览器 502 / nginx 503** | 502：浏览器走代理/HTTPS-First 或 Ingress 无匹配 Host；503：Service 无 ready 端点 | curl 对照定位：curl 通 = 浏览器侧问题（代理/缓存）；`get endpoints` 空 = 修 Pod |
| **apply Ingress 报 webhook 超时** | ingress-nginx admission webhook 的 **caBundle 与证书不匹配**（TCP/TLS 均正常，curl -k 握手成功即证明 controller 正常） | 快速绕过：`kubectl delete validatingwebhookconfiguration ingress-nginx-admission`；根治：`helm uninstall` 后重新 `helm install`（create/patch job 是 post-install hooks，**upgrade 不会触发**） |
| **WordPress 维护模式** | WordPress 自动更新进入维护模式，更新被中断残留 | 查 `.maintenance`（`find /var/www/html -name ".maintenance"`）；无 PVC 时重建容器即恢复镜像原始文件；长期：wp-config 关自动更新 |

### 5.6 步骤 5：文档沉淀（项目转产出物）

```bash
mkdir -p ~/k8s-projects/wordpress
# README.md：架构图 + 部署步骤 + 验证截图（面试展示用）
# troubleshooting.md：每次故障演练记录（现象 → 排查 → 根因 → 修复 → 复盘）
```

**README 模板要点**：项目一句话介绍、架构图、四阶段 Check 清单勾选结果、各验证演示的截图/录屏、遇到的坑与解决。

### 5.7 阶段 3 完成标准（= 项目 A 交付）

- [ ] `kubectl get ingress -n wordpress` → 规则存在
- [ ] `curl --resolve blog.example.com:80:<节点IP> http://blog.example.com/` → 返回 WordPress 页面
- [ ] MySQL 中 wp_posts 等表存在
- [ ] README.md + troubleshooting.md 完成
- [ ] **NFS 升级路径（可选）**：生产环境把 local-path 换成 NFS Provisioner，数据跨节点共享

---

