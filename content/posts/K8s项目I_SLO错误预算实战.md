---
title: "K8s 项目 I：SLO 与错误预算实战"
date: 2026-11-01T15:40:00+08:00
draft: false
description: "把 damo-app 的告警从资源阈值改造成用户视角的 SLO：用 ingress-nginx 七层指标定义成功率 SLI，按 SRE Workbook 四窗口法配燃烧率告警（14.4x / 6x / 1x），并用真实演练证明 5 分钟故障触发 P1、短抖动不叫醒人。含冷启动期长窗口误报的两次修正。"
tags: ["k8s SLO"]
categories: [项目]
---

> 📦 **项目文件包**：[下载 K8s项目I_SLO错误预算.zip](/files/K8s项目I_SLO错误预算.zip)（含 ingress 入口、流量发生器、Prometheus 配置（14 条记录规则 + 5 条告警）、Grafana 大盘与验收脚本）。本文件是详细操作手册，两者配合。


> **系列定位**：K8s 运维项目路线图第 9 站——A 容器化 → B CI/CD → C 监控告警 → D 日志 → E Helm → F 安全 → G 弹性 → H GitOps → **I SLO**。
> **它补的是哪块**：C 和 D 解决了「把数据采上来」，本项目解决「**用数据判断好坏**」。
> **版本基线**：Kubernetes v1.35 ／ Prometheus（项目 C 部署）／ Grafana 11.1.0 ／ ingress-nginx（helm 部署，NodePort 32541）。
> **权威依据**：Google SRE Workbook《Alerting on SLOs》、Google SRE Book《Service Level Objectives》、Prometheus 官方文档（Recording rules / Alerting rules）。

---

> **📖 怎么用这份文档**
>
> - **前置依赖**：项目 C 已完成（Prometheus + Alertmanager + Grafana 在 `monitoring` 命名空间可访问）；`damo-app` 有稳定流量（用 G 项目的 loadgen，或起一个 `while curl` 循环）；ingress-nginx 已暴露 metrics 端口。
> - **预计耗时**：3–4 小时（其中 1 小时是等告警触发，可以并行做别的事）。
> - **学习路径**：阶段 1 先把 SLI/SLO/燃烧率的概念吃透 → 阶段 2 接数据源 → 阶段 3 写记录规则 → 阶段 4 配告警 → 阶段 5 建大盘 → 阶段 6 注入故障验证。**阶段 1 不能跳**，否则后面全是抄配置。
> - **验收标准**：`40-verify.sh` 五关全过；演练能亲眼看到「快烧告警触发、短抖动不触发」两种结果。

## 目录

- [0. 项目概览](#0-项目概览)
- [阶段 1：先懂再做——SLI / SLO / 错误预算](#阶段-1先懂再做sli--slo--错误预算)
- [阶段 2：补入口、造流量、接数据源](#阶段-2补入口造流量接数据源)
- [阶段 3：定义 SLI 与记录规则](#阶段-3定义-sli-与记录规则)
- [阶段 4：多窗口燃烧率告警](#阶段-4多窗口燃烧率告警)
- [阶段 5：Grafana 错误预算大盘](#阶段-5grafana-错误预算大盘)
- [阶段 6：故障演练（本项目核心演示）](#阶段-6故障演练本项目核心演示)
- [7. 故障速查表](#7-故障速查表)
- [8. 自测题（含答案）](#8-自测题含答案)
- [9. 面试要点与简历写法](#9-面试要点与简历写法)
- [10. 学习资源（官方文档）](#10-学习资源官方文档)
- [附录 A：文件包清单与操作顺序](#附录-a文件包清单与操作顺序)

---

## 0. 项目概览

### 0.1 目标与产出

| 产出 | 说明 |
|---|---|
| **SLI 数据源** | Prometheus 新增 `ingress-nginx` 抓取任务，拿到七层入口的请求数与状态码 |
| **记录规则** | 5 个时间窗口的成功率 + 5 个燃烧率 + 错误预算剩余，全部预计算 |
| **告警规则** | 快烧（14.4x）/ 中烧（6x）/ 慢烧（1x）+ 预算不足 + **SLI 无数据** 共 5 条 |
| **Grafana 大盘** | 当前 SLI、预算剩余、多窗口燃烧率、状态码分布、P95 延迟、活跃告警 |
| **故障演练记录** | 一次「快烧触发 + 短抖动不触发」的对照实验，带时间线 |

### 0.2 要解决的问题：阈值告警答不了老板的问题

项目 C 做完之后，你有一堆这样的告警：`NodeMemoryHigh`、`NodeDiskHigh`、`PodCrashLooping`、`DeploymentReplicasShort`。

然后发生这样一段对话：

> **老板**：这个月系统稳不稳？
> **你**：CPU 平均 45%，内存 60%，磁盘 30%，节点都健康。
> **老板**：我问的是——**用户有没有打不开过**？

你答不上来。因为那些指标回答的是「**机器忙不忙**」，不是「**用户好不好**」。

更麻烦的是反向情况：

- CPU 只有 20%，但数据库连接池耗尽 → 用户 100% 报错，**一条告警都不响**。
- CPU 90% 持续半小时，但缓存命中率高、响应仍然 50ms → **告警响了一整夜**，人爬起来发现没事。

阈值告警的三个盲区：

1. **不知道「多严重」**：CPU 90% 和 91% 是两个告警，但它们对用户的影响可能完全一样（都没影响）。
2. **不知道「影响了多少人」**：一个节点挂了，如果它上面没有业务 Pod，用户毫无感知；如果它是唯一入口，全站不可用——同一个 `NodeDown` 告警。
3. **会疲劳**：一天 50 条告警，第 51 条真事故就没人看了。

SLO 解决的就是这三个问题：**用「用户成功请求的比例」来定义好坏，用「还能坏多久」来决定要不要叫人。**

### 0.3 架构与数据流

```
用户请求
   │
   ▼
ingress-nginx ──(/metrics, 10254)──▶ Prometheus
   │                                   │
   ▼                                   ├─ 记录规则：slo:sli_ratio:* / slo:burn_rate:*
damo-app                               │
                                       ▼
                                 Alertmanager ──▶ 邮件 / Webhook
                                       │
                                       ▼
                                    Grafana（预算曲线）
```

关键点：**SLI 在 ingress 层算，不在应用层算**。

- 应用层要埋点（改代码、加 SDK），而且你只能看到「自己的服务」，看不到 nginx 的 502/503。
- 节点层（node-exporter）根本不知道 HTTP 是什么。
- ingress 层天然有 `status` 和 `duration`，是「用户视角」的最低成本实现。

### 0.4 关键决策（面试可讲）

| 决策 | 选了什么 | 为什么不选另一个 |
|---|---|---|
| **SLI 层级** | ingress-nginx | 应用埋点要改代码且盖不住 5xx 网关错误；node-exporter 算不出成功率 |
| **SLO 目标** | 99.5%（30 天） | 99.9% 在单机练习环境几乎必然违规（30 天只允许 43 分钟不可用），会让告警变成噪声；99% 又太宽松，故障了也不报警 |
| **统计窗口** | 30 天滚动 | 7 天波动太大（一次故障就把预算打穿）；90 天反馈太慢，改进了也看不出来 |
| **告警方式** | 多窗口燃烧率 | 单窗口（只看 5m）会被抖动误报；单阈值（错误率 >1%）无法区分「短暂尖峰」和「持续恶化」 |
| **与 C 的关系** | **并存，不替换** | SLO 告警回答「用户受了多大影响」，阈值告警回答「是哪个资源出的问题」。前者决定要不要叫人，后者决定叫人来查什么 |
| **长窗口告警的准入门槛** | 抓取点数 ≥ 窗口的 50% | 用请求数做门槛会被流量大小误导（实测 8762 个请求只覆盖 0.9% 的时间） |

### 0.5 前置条件

```bash
# 1. C 的监控栈在跑
kubectl -n monitoring get pods | grep -E "prometheus|alertmanager|grafana"

# 2. ingress-nginx 的 metrics Service 存在（名字可能带 release 前缀）
kubectl -n ingress-nginx get svc | grep metrics
# 预期类似：ingress-nginx-controller-metrics   ClusterIP   10.96.x.x   10254/TCP

# 3. damo-app 有 Ingress（没有 Ingress 就没有 SLI 数据源，阶段 2.0 会补）
kubectl -n damo-app get ingress
# 如果为空，跳到阶段 2.0
```

如果第 2 条查不到 metrics Service，说明 helm 安装时 `controller.metrics.enabled=false`，需要：

```bash
helm upgrade ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx \
  --reuse-values --set controller.metrics.enabled=true
```

---

## 阶段 1：先懂再做——SLI / SLO / 错误预算

### 1.0 开胃故事：两个运维的对话

> **甲**：「我们昨天出了个事故。」
> **乙**：「多大？」
> **甲**：「不知道，反正监控上一片红。」
>
> **乙**：「我们上周也有一次，**0.3% 的请求失败了 4 分钟**，烧掉了当月 8% 的错误预算，我按流程记了个工单，没叫人。」
> **甲**：「……你怎么知道是 0.3%？」

差别不在工具，在**度量方式**。乙能说出「多少用户受影响、还剩多少容忍度」，甲只能说「一片红」。

SLO 就是把「一片红」翻译成「0.3% 的请求、4 分钟、8% 预算」的那套语言。

### 1.1 四个词，一次说清

| 词 | 全称 | 一句话 | 例子 |
|---|---|---|---|
| **SLI** | Service Level **Indicator** | 你**实际测到**的数字 | 过去 5 分钟成功率 99.72% |
| **SLO** | Service Level **Objective** | 你**给自己定**的目标 | 30 天内成功率 ≥ 99.5% |
| **SLA** | Service Level **Agreement** | 你**对外承诺**（带赔偿条款） | 月度可用性 <99.9% 赔付 10% |
| **错误预算** | Error Budget | `1 - SLO`，允许失败的额度 | 0.5% 的请求，约 **3.6 小时**全站不可用 |

三个容易搞错的点：

1. **SLO 一定比 SLA 严**。SLA 是「赔钱的线」，SLO 是「内部警戒线」，两者之间是缓冲带。
2. **错误预算是「可以花掉的钱」，不是「不能碰的红线」**。每月预算不花完，说明你发布太保守了——这也是 SLO 的一个反直觉用途：**推动变更速度**。
3. **99.5% 不等于「每 200 个请求挂 1 个」这么简单**，因为它是按时间窗口累计的。30 天 99.5% 的预算换算成「全站完全不可用」的时间是 `30 × 24 × 60 × 0.005 = 216 分钟`。注意这是极端换算——现实中是「部分请求失败」，预算消耗得更慢。

### 1.2 为什么「CPU > 80%」不是好告警

拿三个场景对比：

| 场景 | CPU | 用户感受 | 阈值告警 | SLO 告警 |
|---|---|---|---|---|
| A | 90% | 全部 200 OK，P95 50ms | **响**（误报） | 不响 |
| B | 20% | 数据库挂了，100% 500 | **不响**（漏报） | **响** |
| C | 85% | 10% 请求超时 | 响（但说不清多严重） | 响，且能说「烧了 15% 预算」 |

阈值告警监控的是**原因**，SLO 监控的是**结果**。原因有成百上千个，结果只有一个：**用户有没有成功**。

那阈值告警还要不要？要。因为 SLO 告警只说「用户挂了」，不说「为什么挂」。正确的分工是：

- **SLO 告警负责叫醒你**（判断：这事够不够严重）。
- **阈值/资源告警负责定位**（判断：去看哪个组件）。

### 1.3 燃烧率：把「坏了多少」换算成「还能坏多久」

定义：

```
燃烧率 = 当前错误率 / 预算允许的错误率
       = (1 - SLI) / (1 - SLO)
```

代入 SLO = 99.5%：

```
燃烧率 = (1 - SLI) / 0.005
```

几个关键值（背下来）：

| 燃烧率 | 含义 | 30 天预算耗尽时间 |
|---|---|---|
| **1x** | 正好按预算速度烧 | 30 天 |
| **2x** | 快一倍 | 15 天 |
| **6x** | 6 小时烧掉 5% 预算 | 5 天 |
| **14.4x** | **1 小时烧掉 2% 预算** | 约 2 天 |
| **200x** | 全站 100% 失败 | 3.6 小时 |

为什么 14.4 这个数字这么怪？因为它是**倒推**出来的：

```
想让「1 小时烧掉 2% 的 30 天预算」成立：
  1 小时占 30 天的比例 = 1 / 720
  需要的燃烧率 = 2% / (1/720) = 0.02 × 720 = 14.4
```

同理 6x 对应「6 小时烧 5%」：`0.05 / (6/720) = 0.05 × 120 = 6`。

### 1.4 多窗口多燃烧率（SRE Workbook 的四窗口法）

单窗口会出两种错：

- **只看 5 分钟**：一次 30 秒抖动就让燃烧率飙到 20x → 误报。
- **只看 1 小时**：故障刚开始的 4 分钟内看不出异常 → 迟报。

解法是**两个窗口同时超阈值才报**：

| 严重度 | 长窗口 | 短窗口 | 燃烧率阈值 | 预算消耗 | 该做什么 |
|---|---|---|---|---|---|
| **P1 快烧** | 1h | 5m | **14.4x** | 2% / 1h | 立刻处理，打电话 |
| **P2 中烧** | 6h | 30m | **6x** | 5% / 6h | 当天处理，发消息 |
| **P3 慢烧** | 3d | 6h | **1x** | 10% / 3d | 记工单，工作日看 |

两个窗口的分工：

- **长窗口**保证「这事是持续的，不是一秒钟的毛刺」。
- **短窗口**保证「现在还在发生」——避免故障已经恢复了、告警还在响。

### 1.5 阶段自测

1. SLO 99.5%、窗口 30 天，错误预算换算成「全站不可用」是多少分钟？
2. 当前 5 分钟成功率 98%，燃烧率是多少？
3. 为什么「1h 烧 2%」对应的燃烧率是 14.4 而不是 2？

<details>
<summary>答案</summary>

1. `30 × 24 × 60 × 0.005 = 216` 分钟（3.6 小时）。
2. `(1 - 0.98) / 0.005 = 4x`，意味着按这个速度 7.5 天烧完预算。
3. 因为燃烧率是**相对**「30 天匀速用完」的倍数：1 小时只占 30 天的 1/720，要在这 1 小时里烧掉 2%，速度就得是 `0.02 × 720 = 14.4` 倍。

</details>

---

## 阶段 2：补入口、造流量、接数据源

### 2.0 先补一个七层入口（这一步不做，后面全是空）

**先说清楚为什么。**

你的 damo-app 现在是通过 `NodePort 30090` 直接暴露的，**没有 Ingress**。而 ingress-nginx 的指标只统计「经过它转发的请求」——没有 Ingress，`nginx_ingress_controller_requests{namespace="damo-app"}` 永远是空的，SLI 无从算起。

这不是工具的限制，是 SLO 的定义决定的：**你只能对「经过某个入口的流量」承诺可用性**。所以做 SLO 的第一步，是先明确「我的入口在哪」。

```bash
kubectl apply -f 05-damo-app-ingress.yaml
kubectl -n damo-app get ingress
curl -s -o /dev/null -w "%{http_code}
" -H "Host: damo.example.com" http://11.0.1.128:32541/
```

返回 `200` 就通了。三个细节：

- `ssl-redirect: "false"`：没配证书时 ingress 默认会把 http 308 跳到 https，然后失败。
- `host: damo.example.com` 是假域名，靠请求头 `Host` 匹配，不需要真实 DNS。
- 想换域名就同步改后面所有命令里的 `Host`。

> **正规做法**：把 `templates/ingress.yaml` 加进 Chart，用 `ingress.enabled` 开关控制，推 Git 让 Argo CD 同步。**手工 apply 的 Ingress 不在 Git 里，违反 H 项目「Git 是唯一事实来源」**——面试聊到 GitOps 时这一点要能主动承认，并说出「应该改成 Chart 模板」。

### 2.1 造持续流量（而且必须经过 Ingress）

SLI 是「请求比例」，分母为 0 时整个公式是空的。更关键的是：**流量必须走 Ingress**。G 项目的 loadgen 是集群内直连 Service 的，ingress-nginx 根本看不见，用它压测不会产生任何 SLI。

```bash
bash 06-slo-traffic.sh
```

脚本在 **`default`** 命名空间起一个 busybox Pod，每 0.2 秒带 `Host: damo.example.com` 打一次 ingress，约 5 req/s。

> **为什么不在 `damo-app` 里起**：项目 F 给 `damo-app` 做了 **egress 默认拒绝**，在里面发起的出站请求会被静默丢包——现象是 `nc` 和 `wget` 都 `timed out`（注意不是 `refused`），DNS 却是通的。而且语义上「外部用户」不该待在业务命名空间里，搬到 `default` 才是对的。

验证（1 分钟后有值）：

```promql
sum(rate(nginx_ingress_controller_requests{namespace="damo-app"}[5m]))
```

≈ 5 就对了。停止：`kubectl -n default delete pod slo-traffic`。

### 2.2 看一眼 ingress 指标长什么样

```bash
kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller-metrics 10254:10254 &
curl -s http://127.0.0.1:10254/metrics | grep -E "^nginx_ingress_controller_requests|^nginx_ingress_controller_request_duration"
```

你会看到：

```
nginx_ingress_controller_requests{namespace="damo-app",ingress="damo-app",status="200",...} 18432
nginx_ingress_controller_requests{namespace="damo-app",ingress="damo-app",status="503",...} 27
nginx_ingress_controller_request_duration_seconds_bucket{...,le="0.1"} 17230
```

两个宝贝：

- `status` 标签 → 直接算成功率。
- `duration_seconds_bucket` → 直接算 P95/P99。

而且它是 **counter**（只增不减），所以必须用 `rate()`，不能直接读值。

> **★ 如果 `get svc` 里压根没有 metrics Service**（很可能，因为 ingress-nginx **v1.12 起 `--enable-metrics` 默认 false**，chart 也不会建这个 Service）：
>
> 两件事要分开做，缺一不可：
>
> ```bash
> # ① Pod 侧暴露端口
> kubectl -n ingress-nginx patch deploy ingress-nginx-controller --type=json \
>   -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--enable-metrics=true"}]'
> kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller --timeout=180s
>
> # ② 让 Prometheus 的服务发现找得到（role: endpoints 找的是 Service 背后的 Endpoint）
> kubectl apply -f 07-ingress-metrics-svc.yaml
> kubectl -n ingress-nginx get endpoints ingress-nginx-controller-metrics
> ```
>
> 验证时**一定要 exec 到最新的 Pod**——滚动更新期间旧 Pod 还在列表里，`jsonpath '{.items[0].name}'` 很可能拿到旧的，那个 Pod 没有指标：
>
> ```bash
> POD=$(kubectl -n ingress-nginx get pod --sort-by=.status.startTime -o name | tail -1 | cut -d/ -f2)
> kubectl -n ingress-nginx exec "$POD" -- curl -s http://localhost:10254/metrics | grep "^nginx_ingress_controller_requests" | head -2
> ```

### 2.3 应用新的 Prometheus 配置

`10-prometheus-config-slo.yaml` 是「项目 C 的配置 + 两处增量」，直接覆盖：

```bash
# 先备份现有配置（出问题能立刻回滚）
kubectl -n monitoring get cm prometheus-config -o yaml > prometheus-config.backup.yaml

kubectl apply -f 10-prometheus-config-slo.yaml
```

**注意**：ConfigMap 挂载进 Pod 是**异步更新**的，通常 30–60 秒才同步到容器内。等一会儿再 reload：

```bash
kubectl -n monitoring port-forward svc/prometheus 9090:9090 >/dev/null 2>&1 &
sleep 60
curl -X POST http://127.0.0.1:9090/-/reload
curl -s http://127.0.0.1:9090/-/ready      # 返回 "Prometheus Server is Ready."
```

如果 reload 报错，先看日志：

```bash
kubectl -n monitoring logs deploy/prometheus --tail=30
```

### 2.4 验证抓取成功

```bash
curl -s 'http://127.0.0.1:9090/api/v1/query?query=up{job="ingress-nginx"}' | head -c 500
```

预期 `"value":[...,"1"]`。如果为 0 或空，看第 7 章前两条。

### 2.5 阶段完成标准

- [ ] Ingress 存在，`curl -H "Host: damo.example.com" :32541` 返回 200
- [ ] 流量 Pod 在跑，`nginx_ingress_controller_requests` 有约 5 req/s
- [ ] `up{job="ingress-nginx"}` 返回 1
- [ ] `status` 标签能看到 200
- [ ] Prometheus Targets 页面里 `ingress-nginx` 是 UP

---

## 阶段 3：定义 SLI 与记录规则

### 3.1 SLI 表达式怎么写

**成功率（非 5xx 占比）**：

```promql
sum(rate(nginx_ingress_controller_requests{namespace="damo-app",status!~"5.."}[5m]))
/
sum(rate(nginx_ingress_controller_requests{namespace="damo-app"}[5m]))
```

三个细节，每个都有坑：

1. **`status!~"5.."` 而不是 `status=~"2.."`**：4xx（比如用户访问不存在的页面）是客户端错误，不该算服务不可用。用「排除 5xx」比「只要 2xx」更符合直觉。
2. **必须用 `rate()`**：counter 只增不减，直接读值算比例毫无意义。
3. **分子分母都要 `sum()`**：不聚合的话，不同 status 会变成多条时间序列，比例算不出来。

**为什么用记录规则而不是直接在告警里写？**

- 直接写在告警里 → 每次评估（15s）都要扫一遍原始数据，5 条告警 × 5 个窗口 = 25 次重复计算。
- 记录规则 → 每 30s 算一次存成新指标，告警和 Grafana 都读这个结果，**一份计算多处复用**。

### 3.2 记录规则逐条讲

文件里（`slo-rules.yml`）一共 12 条记录规则，分三类：

**A. 五个时间窗口的成功率**

```yaml
- record: slo:sli_ratio:5m    # 短窗口：快速发现
- record: slo:sli_ratio:30m   # 配中烧
- record: slo:sli_ratio:1h    # 配快烧
- record: slo:sli_ratio:6h    # 配慢烧的短边
- record: slo:sli_ratio:3d    # 慢烧的长窗口
```

**B. 五个燃烧率**

```yaml
- record: slo:burn_rate:1h
  expr: (1 - slo:sli_ratio:1h) / 0.005
```

注意 `0.005` 就是 `1 - 0.995`。这里硬编码了 SLO 目标——Prometheus 没有「变量」概念，改 SLO 就要改这些表达式。**这是它不如 SLO 专用框架（如 Sloth、Pyrra）的地方**，但手工写一遍才真的懂它在算什么。

**C. 预算剩余**

```yaml
- record: slo:budget:remaining_ratio
  expr: 1 - ((1 - slo:sli_ratio:3d) / 0.005)
```

含义：`1 - 已消耗比例`。完全没失败时是 1，预算用完是 0，**全挂了会变成负数**（这很正常，说明超额消耗）。

> 为什么用 3d 而不是 30d 算预算？因为 `rate(...[30d])` 每次评估都要扫 30 天的原始样本，代价太大。用 3d 是做近似——真要精确可以用分层记录规则（1h 的 `increase` 再 `sum_over_time`）。**面试如果被追问性能，这就是加分点。**

### 3.3 验证记录规则

```bash
kubectl -n monitoring port-forward svc/prometheus 9090:9090 >/dev/null 2>&1 &
curl -s 'http://127.0.0.1:9090/api/v1/rules?type=record' | grep -o 'slo:[a-z_:0-9]*' | sort -u
```

或者直接在 Prometheus UI 的 Graph 页输入：

```promql
slo:sli_ratio:5m      # 应该接近 1（比如 1 或 0.999）
slo:burn_rate:1h      # 应该接近 0
slo:budget:remaining_ratio   # 应该接近 1
```

**如果全都是空**：规则没加载。检查 `prometheus.yml` 里 `rule_files` 的第二行路径、以及 ConfigMap 是否真的更新进了 Pod：

```bash
kubectl -n monitoring exec deploy/prometheus -- ls -l /etc/prometheus/
# 应该看到 rules.yml 和 slo-rules.yml 两个文件
```

### 3.4 阶段完成标准

- [ ] `slo:sli_ratio:5m` 有值且接近 1
- [ ] `slo:burn_rate:1h` 有值（正常时接近 0）
- [ ] `slo:budget:remaining_ratio` 有值且接近 1
- [ ] `/etc/prometheus/` 里能看到 `slo-rules.yml`

---

## 阶段 4：多窗口燃烧率告警

### 4.1 三个窗口怎么选

| 告警 | 表达式 | for | 语义 |
|---|---|---|---|
| `SLOBurnRateFast` | `1h > 14.4 and 5m > 14.4` | 2m | 按当前速度，1 小时烧掉 2% 预算 |
| `SLOBurnRateMedium` | `6h > 6 and 30m > 6` | 15m | 6 小时烧掉 5% |
| `SLOBurnRateSlow` | `3d > 1 and 6h > 1` | 1h | 正好按预算速度消耗 |
| `SLOBudgetLow` | `剩余 < 20%` | 10m | 兜底，不管速度 |
| `SLONoData` | `absent(slo:sli_ratio:5m)` | 10m | **监控监控本身** |

`and` 是 PromQL 的集合运算：两边都有的标签组合才会输出。因为两边都只输出一条无标签序列，效果就是「两个条件同时为真」。

### 4.2 为什么 for 不能省

`for: 2m` 表示「表达式持续为真 2 分钟」才 firing。它过滤的是「瞬时毛刺」，和多窗口的「短窗口」是两个不同层次的过滤：

- **短窗口（5m）**：平滑掉几秒级的抖动。
- **for（2m）**：过滤掉「刚满足又掉下去」的边界情况。

两个都要有，否则会出现「告警响一下又自己消失」，运维管这叫 flapping。

### 4.3 验证告警已加载

```bash
curl -s 'http://127.0.0.1:9090/api/v1/rules?type=alert' | grep -o '"SLO[A-Za-z]*"' | sort -u
```

预期看到 5 条：`SLOBurnRateFast`、`SLOBurnRateMedium`、`SLOBurnRateSlow`、`SLOBudgetLow`、`SLONoData`。

UI 上在 **Alerts** 页也能看到，状态应该是 `Inactive`（绿色）。

### 4.4 和 C 的阈值告警并存

这里有个常见误解：**做了 SLO 就要把阈值告警删掉**。

不要删。原因在 1.2 节说过：SLO 负责「叫不叫人」，阈值负责「叫人来查什么」。一条完整的告警链路是：

```
SLOBurnRateFast（用户在挂）
   ↓ 人来了，但不知道查什么
   → 看一眼 alerts 页面：NodeDown / PodCrashLooping / DeploymentReplicasShort
     某个组件在红 → 直奔那里
```

如果只有 SLO 告警，你只知道「坏了」，不知道「哪坏了」，MTTR 反而变长。

### 4.5 阶段完成标准

- [ ] 5 条 SLO 告警在 Prometheus Alerts 页可见
- [ ] C 的 4 条阈值告警仍然可见（没被覆盖）
- [ ] `SLONoData` 在正常有流量时是 Inactive


### 4.6 ★ 冷启动：新接入指标时，长窗口告警不可信

这一节是真实踩出来的（2026-11-01 实测）。

**现象**：指标接入 32 分钟后做了一次 30 秒的故障演练，预期「一条告警都不发」，实际 `SLOBurnRateSlow` 和 `SLOBudgetLow` 双双 firing。

**原因**：3 天窗口里只有 32 分钟数据（约 960 个请求），30 秒故障产生约 15 个 503。`rate()` 不会告诉你「窗口没填满」，它按现有样本外推：

```
SLI    = (960 - 15) / 960 = 98.4%
燃烧率 = (1 - 0.984) / 0.005 = 3.1x     ← > 1，慢烧成立
预算   = 1 - 3.1 = -2.1                 ← < 0.2，预算告警成立
```

**第一次修正是错的**：用请求数做门槛。

```yaml
slo:requests:count_3d > 5000
```

结果 `count_3d` 实测 **8762**，直接越过门槛——因为 DNS 缓存生效后流量从 0.5 涨到 3.6 req/s，40 分钟就攒够 8762 个请求，可时间只覆盖了 `40/4320 = 0.9%`。

> **教训：请求数会随流量大小浮动，判断窗口覆盖必须看时间。**

**第二次修正（正确）**：用抓取点数。

```promql
slo:coverage:3d = count_over_time(up{job="ingress-nginx"}[3d])
slo:coverage:6h = count_over_time(up{job="ingress-nginx"}[6h])
```

满窗点数 = 窗口时长 ÷ 抓取间隔（15s）。

| 窗口 | 满窗点数 | 门槛 | 相当于覆盖 | 什么时候够用 |
|---|---|---|---|---|
| 5m | 20 | 不需要 | — | 起来就有 |
| 1h | 240 | 不需要 | — | 40 分钟即可判断 |
| 6h | 1440 | **700** | 3 小时 | 接入 3 小时后 |
| 3d | 17280 | **8000** | 33 小时 | 接入约 1.5 天后 |

**这条经验解释了一个常见疑问**：为什么生产环境做 SLO 都建议「先观察一周再开长窗口告警」？不是工具慢，是**窗口需要时间填满**。

还有一个操作层面的坑：**任何「改完规则立刻验证」的脚本，都要留 30 秒给记录规则跑第一轮**，否则你会把「还没算出来」误判成「没生效」。

---

## 阶段 5：Grafana 错误预算大盘

### 5.1 导入

```bash
# 方式一：UI 导入
#   Grafana → Dashboards → New → Import → 上传 20-grafana-dashboard.json

# 方式二：curl 导入（把 admin 密码换成你的）
curl -s -X POST http://11.0.1.128:30300/api/dashboards/db \
  -H "Content-Type: application/json" \
  -u admin:你的密码 \
  -d "{\"dashboard\": $(cat 20-grafana-dashboard.json), \"overwrite\": true}"
```

导入后如果面板显示 `No data`，先确认数据源是 Prometheus（Uid 保持默认即可）。

### 5.2 六个面板怎么读

| 面板 | 看什么 | 异常信号 |
|---|---|---|
| 当前成功率 SLI（5m） | 绿=达标，橙=跌破 99%，红=跌破 99.5% 线 | 掉到 99% 以下基本等于在烧预算 |
| 30 天错误预算剩余 | Gauge 指针位置 | 低于 50% 说明这个月已经不太平 |
| 活跃的 SLO 告警 | 有没有 firing | 有数据 = 正在告警 |
| 燃烧率多窗口 | 四条线（5m/1h/6h/3d）+ 虚线阈值 | **5m 先动、1h 后动**是正常现象 |
| 请求速率（按状态码） | 堆叠面积，看 5xx 那一块 | 5xx 面积变大是 SLI 下降的直接原因 |
| P95 延迟 | 单线 | 延迟 SLO 的第二阶段可以用它 |

### 5.3 阶段完成标准

- [ ] 六个面板都有数据（不是 No data）
- [ ] 成功率为 1.000（或极接近）
- [ ] 预算剩余接近 100%
- [ ] 燃烧率都在 1x 以下

---

## 阶段 6：故障演练（本项目核心演示）

### 6.1 场景 A：快烧

**开始前确认两件事**：`kubectl -n damo-app get pod slo-traffic` 还在 Running；`curl -H "Host: damo.example.com" :32541` 还返回 200。流量一断，SLI 分母为 0，演练就白做了。

**再算一笔账**：要是故障只持续 3 分钟，1h 窗口的燃烧率能不能超过 14.4？

```
1h 窗口总请求 ≈ 60 分钟的流量
故障 3 分钟，期间 100% 失败
错误率 = 3 / 60 = 5%
燃烧率 = 5% / 0.5% = 10x    ← 小于 14.4，不触发！
```

要让 `1h > 14.4`，需要错误率 > `14.4 × 0.5% = 7.2%`，也就是 `0.072 × 60 = 4.32` 分钟的全量失败。

**所以演练必须持续 5 分钟**，3 分钟不够。这个计算本身就是面试素材——说明你知道「告警延迟」不是 bug，是窗口设计的必然结果。

执行：

```bash
bash 30-fault-injection.sh fast
```

脚本做三件事：`scale --replicas=0` → 倒计时 300 秒 → 恢复。

**预期时间线**（自己记录，作为产出）：

| 时刻 | 现象 | 怎么看 |
|---|---|---|
| T+0 | 缩容到 0 | `kubectl get pods` 变空 |
| T+30s | 503 出现 | Grafana 状态码面板 503 面积顶满 |
| T+1m | 5m SLI → 0，5m 燃烧率 → 200x | Grafana 燃烧率面板的 5m 线垂直上去 |
| T+4.5m | 1h 燃烧率 > 14.4 | 1h 线缓慢爬过红色虚线 |
| T+6.5m | **SLOBurnRateFast firing** | Alertmanager / 邮件 |
| T+300s | 恢复副本 | 503 消失，SLI 回升 |
| T+1h | 1h 燃烧率才回落到阈值下 | **这就是告警不会立刻恢复的原因** |

### 6.2 场景 B：对照组（同样故障，时间更短）

```bash
bash 30-fault-injection.sh slow
```

同样是 100% 失败，但只持续 30 秒。

**预期**：

- 5m 燃烧率瞬间飙到 20x（因为 30 秒 / 5 分钟 = 10% 错误率）。
- 1h 燃烧率只有约 1.7x（30 秒 / 3600 秒 = 0.8% 错误率）。
- **结果：没有任何 SLO 告警 firing。**

这个对照证明了两件事：

1. 如果只看 5m 单窗口，这 30 秒就会误报。
2. 双窗口设计的目的就是「**短抖动不叫醒人，持续恶化才叫醒人**」。

### 6.3 把两次演练写成一张表（面试直接讲）

| 场景 | 故障时长 | 5m 燃烧率峰值 | 1h 燃烧率峰值 | 告警 |
|---|---|---|---|---|
| A | 5 分钟 | 200x | 16.7x | **SLOBurnRateFast 触发** |
| B | 30 秒 | 20x | 1.7x | 不触发 |

一句话总结：**故障的「严重程度」不是看峰值，是看「持续多久 × 影响多大」。**

### 6.4 阶段完成标准

- [ ] 场景 A 触发 `SLOBurnRateFast`（已实测，见 6.5）
- [ ] 场景 B 不触发任何 SLO 告警（**需等场景 A 的 1h 窗口滚干净再做**，约 55 分钟后）
- [ ] 两次演练的燃烧率峰值都记录下来了
- [ ] 恢复后预算剩余明显下降（场景 A 大约消耗 2% 左右）


### 6.5 真实演练记录（2026-11-01 实测）

**为什么不用 `scale --replicas=0`**：G 项目的 HPA 还在（min 2 / max 6），缩到 0 会在 15 秒内被改回 2，故障持续不到 4.32 分钟，演练必然失败——表面看是「没告警」，实际是「故障根本没发生」。

所以改用「**Ingress 后端指向不存在的 Service**」，nginx 立刻返回 503。这条路不碰 Deployment，HPA 管不着，而且它模拟的是「网关配置错误」这类更常见的真实故障。

| 时刻 | 动作 / 现象 | 数据 |
|---|---|---|
| 15:30:05 | 注入：`backend.service.name` → `not-exist` | — |
| 15:30:15 | 503 出现，200 那条掉到 0 | Grafana 状态码面板 |
| 15:31 | 5m SLI → 0 | `burn_rate:5m` = **200x** |
| 15:34:30 | 1h 燃烧率越过 14.4 | 错误率 7.2% 的分界线 |
| 15:35:06 | 恢复 Ingress | 503 消失，SLI 回升 |
| 15:36:30 | **`SLOBurnRateFast` firing** | 邮件：`1h 燃烧率 23.1x`，critical |
| 15:38 | 复查 | Fast firing；Medium pending（6h 覆盖不足，已按 4.6 加门槛） |

**告警邮件原文（摘要）**：

```
[FIRING:1] SLOBurnRateFast (critical damo-app-availability)

Labels:
  alertname = SLOBurnRateFast
  severity  = critical
  slo       = damo-app-availability
Annotations:
  summary     = damo-app 错误预算快速燃烧（1h 燃烧率 23.1x）
  description = 快烧阈值 14.4x 已持续 2 分钟，按此速度 30 天预算将在数小时内耗尽。
                先看 5xx 的 status 分布与 ingress 日志。
```

**为什么是 23.1x 而不是 16.7x**：1 小时窗口当时只有 40 分钟数据，5 分钟故障占 `5/40 = 12.5%`（换算 25x）；等窗口填满才是 `5/60 = 8.3%` → 16.7x。两者都远超 14.4，结论不变，但**这解释了「新接指标头几个小时 SLO 数值偏高」的原因**。

**两个必须知道的现象**：

1. **恢复之后告警不会消失**。15:35 网页就恢复了，Fast 还在 firing——因为 1 小时窗口里还留着那 5 分钟的失败样本，要等它滚出去（约 55 分钟）燃烧率才降下来。SLO 告警衡量的是「过去一小时的用户感受」，不是「此刻的状态」。
2. **同一场故障，不同窗口给出不同结论**。1h 报 16.7x（快烧），6h 报约 2.8x（不到中烧线 6x）。这不是矛盾，正是多窗口的意义：**不同窗口对应不同的响应级别**。

---

## 7. 故障速查表

> 按「现象」查。★ 是本项目最容易卡住的地方。

| 现象 | 根因 | 排查与解决 |
|---|---|---|
| ★ `up{job="ingress-nginx"}` 无数据 | metrics Service 不存在 / 端口名不叫 `metrics` / namespace 写错 | `kubectl -n ingress-nginx get svc,endpoints`；确认 Service 有 `metrics` 端口；确认 helm 的 `controller.metrics.enabled=true` |
| 抓到 target 但 `up=0` | 端口通了但 `/metrics` 路径 404，或 Service 指向了错误端口 | `kubectl -n ingress-nginx get endpoints <svc>` 看端口号是不是 10254 |
| ★ `slo:sli_ratio:5m` 为空 | 没有流量（分母为 0）/ `namespace` 标签不是 `damo-app` | 先查原始指标 `nginx_ingress_controller_requests` 是否非空；再看它的 `namespace` 标签实际值 |
| 记录规则全部为空 | `rule_files` 路径没生效 / ConfigMap 没同步进 Pod | `kubectl exec deploy/prometheus -- ls /etc/prometheus/`；确认有 `slo-rules.yml`；再 `curl -X POST /-/reload` |
| reload 报 `failed to reload config` | YAML 语法错误（常见于 `{{ }}` 里嵌套引号） | 看 `kubectl logs deploy/prometheus` 的具体行号；用 `promtool check rules` 本地校验 |
| ★ 演练做完没告警 | 故障时长不够（<4.32 分钟），1h 窗口被稀释 | 按 6.1 的算账：至少 5 分钟全量失败 |
| 告警一直在 firing 不恢复 | 1h 窗口还包含故障期间的样本 | 正常现象，等窗口滚出去（最多 1 小时） |
| `slo:budget:remaining_ratio` 是负数 | 窗口内失败率超过 0.5%，预算超额消耗 | 不是 bug，是「这个月已经超支」的正确表达 |
| Alertmanager 收不到 | 告警规则里的 label 与 route 不匹配 | `kubectl -n monitoring exec deploy/alertmanager -- cat /etc/alertmanager/alertmanager.yml`；本项目沿用 C 的 `severity` 路由即可 |
| Grafana 面板 No data | 数据源 Uid 不匹配 / 时间范围选错 | 面板编辑里看 Query inspector；把时间范围调到 `Last 6 hours` |

---

## 8. 自测题（含答案）

1. SLI 和 SLO 的区别是什么？各自举一个例子。
2. 为什么 4xx 不算「服务不可用」？
3. 燃烧率 1x 是什么含义？14.4x 呢？
4. 为什么必须用「长窗口 + 短窗口」两个条件，只用一个会怎样？
5. SLO 99.5%、窗口 30 天，错误预算折合成「全站不可用」是多少分钟？
6. 100% 失败持续 5 分钟，1h 燃烧率大约多少（假设此前全部成功）？
7. 有了 SLO 告警，为什么还要保留 C 的阈值告警？
8. `SLONoData` 这条告警在防什么？

<details>
<summary>参考答案</summary>

1. SLI 是**测出来的**（过去 5 分钟成功率 99.72%），SLO 是**定下来的目标**（30 天 ≥99.5%）。SLI 是事实，SLO 是承诺。
2. 4xx 是客户端的问题（路径写错、没权限），服务端是正常工作的。把它算成失败会让 SLO 被爬虫和扫描器污染。
3. 1x = 正好按预算速度消耗，30 天用完预算；14.4x = 1 小时烧掉 2% 的 30 天预算，约 2 天耗尽。
4. 只用短窗口：30 秒抖动就报警（误报）；只用长窗口：故障开始后 4 分钟内看不出来（迟报）。两个都超才报 = 既持续、又正在发生。
5. `30×24×60×0.005 = 216` 分钟（3.6 小时）。
6. 错误率 `5/60 ≈ 8.3%`，燃烧率 `8.3%/0.5% ≈ 16.7x`。
7. 因为 SLO 只说「用户在挂」，不说「哪里挂了」。阈值告警负责定位根因，两者是「叫醒」和「指路」的分工。
8. 防「静默失败」：如果 ingress 抓取挂了或业务没流量，SLI 会变成空值，所有燃烧率告警都不会触发——**看起来一切正常，其实监控已经瞎了**。

</details>

---

## 9. 面试要点与简历写法

### 9.1 一句话介绍

> 我把 damo-app 的告警从「资源阈值」改成了「用户视角的 SLO」：用 ingress-nginx 的七层指标定义成功率 SLI，按 SRE Workbook 的四窗口法配燃烧率告警，并做了一次对照演练证明——5 分钟的持续故障会触发 P1 告警，30 秒的同强度抖动不会。

### 9.2 三个值得展开的决策

| 决策 | 为什么 | 结果 |
|---|---|---|
| **SLI 放在 ingress 层** | 用户感受到的是七层结果；应用层埋点盖不住网关 5xx，node 层根本不懂 HTTP | 零代码改动拿到「用户视角成功率」 |
| **99.5% 而不是 99.9%** | 单机练习环境 99.9% 必然违规（30 天只允许 43 分钟），告警会变成噪声 | 一个月只报一次真实故障 |
| **双窗口而不是单窗口** | 单窗口 5m 会被抖动误报，单窗口 1h 会迟报 4 分钟 | 演练证实：短抖动不报、持续故障必报 |

### 9.3 高频追问

| 追问 | 怎么答 |
|---|---|
| SLO 和 SLA 什么关系？ | SLA 是赔钱线，SLO 是内部警戒线，SLO 必须比 SLA 严，中间是缓冲带 |
| 预算花不完说明什么？ | 说明发布太保守，可以加快变更节奏——SLO 不只用来踩刹车，也用来踩油门 |
| 99.9% 到底多严？ | 30 天只能有 43 分钟全站不可用，约等于每月一次 40 分钟的发布事故都不允许 |
| 告警太多怎么办？ | 先分级（P1 电话 / P2 消息 / P3 工单），再用燃烧率替代「资源阈值」做主告警，资源告警降级为定位手段 |
| 你怎么知道告警是准的？ | 做了对照演练：5 分钟故障触发、30 秒抖动不触发，两个场景的燃烧率峰值都记录下来 |
| Prometheus 算 30 天窗口会不会很慢？ | 会。所以预算用 3d 近似，生产上应该用分层记录规则（1h increase 再 sum_over_time） |

---

## 10. 学习资源（官方文档）

| 主题 | 链接 |
|---|---|
| SRE Workbook：基于 SLO 的告警 | https://sre.google/workbook/alerting-on-slos/ |
| SRE Book：Service Level Objectives | https://sre.google/sre-book/service-level-objectives/ |
| Prometheus Recording rules | https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/ |
| Prometheus Alerting rules | https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/ |
| ingress-nginx 指标说明 | https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/ |
| PromQL 基础（rate/irate 区别） | https://prometheus.io/docs/prometheus/latest/querying/functions/#rate |

---

## 附录 A：文件包清单与操作顺序

```
K8s项目I_SLO错误预算/
├── K8s项目I_SLO错误预算实战.md
└── manifests/
    ├── 05-damo-app-ingress.yaml        # 给 damo-app 补七层入口（SLI 的前提）
    ├── 06-slo-traffic.sh               # 造经过 Ingress 的持续流量
    ├── 10-prometheus-config-slo.yaml   # ConfigMap：C 的配置 + ingress-nginx job + slo-rules.yml
    ├── 20-grafana-dashboard.json       # 6 个面板的大盘
    ├── 30-fault-injection.sh           # 演练：fast（5 分钟）/ slow（30 秒）
    └── 40-verify.sh                    # 五关验收
```

**操作顺序**：

```bash
# 0. 补入口 + 造流量
kubectl apply -f 05-damo-app-ingress.yaml
bash 06-slo-traffic.sh

# 1. 备份 + 覆盖配置
kubectl -n monitoring get cm prometheus-config -o yaml > prometheus-config.backup.yaml
kubectl apply -f 10-prometheus-config-slo.yaml

# 2. 等 60 秒让 ConfigMap 同步，再热加载
kubectl -n monitoring port-forward svc/prometheus 9090:9090 >/dev/null 2>&1 &
sleep 60 && curl -X POST http://127.0.0.1:9090/-/reload

# 3. 验收
bash 40-verify.sh

# 4. 导入大盘（UI 或 curl）

# 5. 演练
bash 30-fault-injection.sh fast
```

**回退**：

```bash
kubectl apply -f prometheus-config.backup.yaml
kubectl -n monitoring rollout restart deploy/prometheus
```

回退后 SLO 的记录规则和告警全部消失，C 的原有告警不受影响——因为它们是两个独立的规则文件。
