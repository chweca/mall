# HomeMall 云原生工程：真实故障与排障复盘

> 项目定位：基于 [macrozheng/mall](https://github.com/macrozheng/mall) 的个人 Kubernetes / DevOps 工程实践。
> 记录范围：2026-09 ～ 2026-10 的实际故障与发布前风险验证。
> 本文只陈述有历史操作记录支持的结论；没有保存原始完整日志的地方，会明确写为「历史复盘，待补原始日志」。
>
> **阅读提示：** 优先看 1～4 节四个代表性案例。此文用于工程复盘，不是让当前运行的 HomeMall 重新执行破坏性实验。

## 0. 排障方法与证据级别

遇到故障时按这个顺序处理：

```text
业务现象（用户看到什么）
          ↓
判断所属层次（入口、Service、Pod、应用、依赖、CI/CD）
          ↓
收集状态 / Event / 日志 / HTTP 证据
          ↓
提出可检验的原因假设
          ↓
实施最小修复
          ↓
验证故障恢复，记录预防措施
```

**证据级别**：

- **A：有历史命令输出、配置或发布记录支撑。** 可以作为公开 GitHub 故障案例。
- **B：旧记录有现象和解决结论，但需要补充原始日志或确切命令。** 文中标注，不凭空编造截图。
- **C：仅为演练、机制验证或事前风险分析，不应冒充已发生的生产事故。** 放在附录。

## 1. K8s Pod 无法稳定启动：从 ImagePullBackOff 到 CrashLoopBackOff

**类型：真实故障；证据等级：B（旧复盘有结论，待补完整 Events/应用日志）**
**涉及层：Harbor 镜像仓库 → kubelet 拉镜像 → 容器启动 → Spring Boot → MySQL。**

### 现象

早期 `mall-admin` 部署曾依次遇到：

1. **`ImagePullBackOff`**：节点无法正常拉取私有 Harbor 镜像。
2. 拉取问题解决后，出现 **`CrashLoopBackOff`**：容器能启动进程，但应用随后退出并反复重启。
3. 历史应用报错包含：`Public Key Retrieval is not allowed`。

**注意：** 这是两个不同层次的问题，不能看到 Pod 起不来就一概判断为 Kubernetes 网络故障。

### 排查逻辑

```text
kubectl get pod 发现异常状态
           ↓
ImagePullBackOff？→ 看 Pod Events、Harbor 仓库地址和 imagePullSecrets
           ↓
镜像能拉下来，但 CrashLoopBackOff？→ 看 Spring Boot 容器日志
           ↓
日志出现 JDBC 连接错误
           ↓
检查 MySQL 连接配置、容器环境变量、服务名和 JDBC URL
```

用于复盘的检查命令（只读；`<POD>` 请替换为实际名称）：

```bash
kubectl get pods -n homemall-dev
kubectl describe pod <POD> -n homemall-dev
kubectl logs <POD> -n homemall-dev --previous
```

- `describe` 重点关注 `Events`、镜像和 Secret 引用；
- `logs --previous` 用来查看上一次已经退出的容器日志；
- `Public Key Retrieval is not allowed` 是 JDBC/MySQL 认证连接相关报错，不是 Harbor 拉取问题。

### 历史修复与结论

- 检查并配置私有仓库拉取认证（`imagePullSecrets`），解决镜像拉取阶段问题。
- 将正确的 Spring Boot 数据源/Redis 运行配置注入 Pod，核对 JDBC 地址；历史修复记录包含 `allowPublicKeyRetrieval=true`。
- 通过 Pod 状态和应用访问恢复验证。

**仍需补证据：** 最初失败的 Pod Events 全文、修复前后 JDBC URL 差异、对应 Helm Revision。不要把本节简化成“仅添加 `allowPublicKeyRetrieval=true` 就解决全部问题”。更正式的环境还应评估 MySQL TLS 与账户认证的安全配置。

**★ 重点记忆：`ImagePullBackOff` 先看镜像拉取；`CrashLoopBackOff` 先看应用进程退出原因，两者不同。**

---

## 2. readinessProbe 故障：Pod Running，但无法进入 Service 可用后端

**类型：主动故障注入与恢复演练；证据等级：A**
**阶段：Gate 10（2026-10-05）**
**涉及层：Helm Deployment → kubelet Probe → Pod Ready 状态 → Endpoint/EndpointSlice → Service。**

### 现象与证据

为验证 readinessProbe 的作用，曾将 `mall-admin` 的检查路径故意改错：

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness-test-error  # 演练用错误路径
    port: 8080
```

历史记录显示：

- 新 Pod 进程仍可运行，处于 `Running`；
- 容器为 `0/1`、`Ready=false`；
- Endpoint/EndpointSlice 中该 Pod 的就绪状态为 `False`；
- 其他健康副本仍为 `Ready=true`。

**根因：** kubelet 请求了不存在的 HTTP 健康检查路径，检查失败，故不会把该 Pod 标记为可接收服务流量。

### 修复与验证

修改文件：`/root/homemall-chart/templates/mall-admin-deployment.yaml`（该阶段原始实验位置）。

将路径改回：

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: {{ .Values.mallAdmin.service.port }}
```

历史复盘确认修改后滚动更新恢复成功，后端可用副本恢复正常。当前 GitHub Helm Chart 仍保留正确路径。

复盘查询：

```bash
kubectl get deployment mall-admin -n homemall-dev
kubectl describe pod <POD> -n homemall-dev
kubectl get endpointslice -n homemall-dev
```

**★ 重点记忆：`Running` 只代表容器进程运行，`Ready` 才是 Service 是否应向 Pod 分发流量的重要条件。**

---

## 3. MinIO 图片访问问题：后端可上传，不代表浏览器可访问

**类型：真实功能故障；证据等级：A**
**阶段：Gate 12（2026-10-09～10）**
**涉及层：浏览器 → 公网反向代理/FRP → Traefik → Service/EndpointSlice → MinIO。**

### 两个独立的问题

1. **历史图片访问异常：** 旧 OSS 图片在带页面 `Referer` 的真实 GET 请求下可能返回 `403`，即使商城商品 API 能返回 `200 + JSON`，图片仍无法展示。
2. **新图片地址错误：** 后端曾返回 MinIO 内网地址（`10.10.10.36:9000`）给浏览器。K8s 内部连通不等于外部浏览器能够访问该内网地址。

### 关键思路：区分内网上传地址与公网展示地址

```text
管理后台选择图片
        ↓
mall-admin 使用 MINIO_ENDPOINT 上传到 MinIO（内网）
        ↓
MinIO 保存图片对象
        ↓
mall-admin 返回 MINIO_PUBLIC_URL 拼出的图片 URL
        ↓
浏览器通过 img.chweca.com 请求图片
        ↓
Nginx → FRP → Traefik → minio-external Service → EndpointSlice → MinIO
```

实际改造后，两个配置承担不同职责：

```text
MINIO_ENDPOINT    = http://10.10.10.36:9000  # 后端上传用
MINIO_PUBLIC_URL  = http://img.chweca.com     # 返回给浏览器
```

`MINIO_ACCESSKEY` / `MINIO_SECRETKEY` 来自 Kubernetes Secret，而不是写在公开的 Helm Values 中。

### 最终验证

- 新图片上传和后台/商城显示成功；
- 经公共域名访问代表性的图片 URL 成功；
- 旧图片恢复展示，见下一节。

**注意边界：** 图片公网链路在个人实验环境中验证过，不意味着所有访问控制、TLS 和高可用指标达到生产要求。

**★ 重点记忆：服务端 SDK 的上传地址和客户端浏览器的下载地址，可能是两个完全不同的地址。**

---

## 4. 旧 OSS 图片迁移：先迁对象，再改数据库 URL

**类型：真实数据修复；证据等级：A**
**阶段：Gate 12（2026-10-09～10）**

### 问题

历史商品主图 URL 仍指向原作者 OSS。即使自己的 MinIO、Ingress 全部部署好，数据库里旧地址不变，浏览器仍会继续请求旧 OSS。

### 实施原则

```text
统计数据库旧 URL
       ↓
识别需要迁移的文件（去重）
       ↓
备份数据库
       ↓
下载旧对象、检查是真实图片
       ↓
上传对象到自己 MinIO
       ↓
通过 img.chweca.com 测试读取
       ↓
UPDATE 数据库图片 URL
       ↓
重新查询剩余旧 URL 数量 + 浏览器确认
```

### 实际结果

- 涉及 `pms_product.pic` 的 **37 条商品主图记录**；
- 去重后是 **22 个不同的旧 OSS 图片 URL / 文件**；
- 22 个旧图片文件完成下载并上传到 MinIO；
- 37 条记录的图片 URL 更新，旧前缀剩余匹配数量为 0；
- 实际商城商品主图展示恢复。

**边界：** 这是商品主图范围的迁移，不等于品牌图、广告图、商品详情富文本等全部历史资源已迁移，也没有声称逐个文件完成 HTTP 校验和验证。

**★ 重点记忆：先迁移并验证对象可读取，再修改数据库引用；否则页面可能出现更大范围的坏图。**

---

## 5. Jenkins #18：Controller 获取 Jenkinsfile 时 Git Fetch 卡住

**类型：真实故障；证据等级：A**
**涉及层：Jenkins Controller → 运行 Jenkins 服务的 Linux 用户 → Git HTTPS → GitHub。**

### 现象

构建控制台长时间仅显示 `Started by user`，没有出现 `[Pipeline] Start of Pipeline`。Controller 上可观察到 `git fetch` 和 `git-remote-https` 进程。人为中止后日志显示 `InterruptedException`；它反映的是终止时被中断，不是网络问题的原始原因。

### 对照证据

| 测试 | 观察 |
| --- | --- |
| Jenkins 用户直接浅克隆 | 60 秒只下载约 7%，显示约 137 KiB/s，随后受 timeout 限制结束 |
| 同一用户通过 Mihomo 代理浅克隆 | 完成 891 个对象、10.60 MiB，传输显示约 16.83 MiB/s |
| `curl -x` 访问 GitHub | 代理路径返回 HTTP 200 |

### 修复

为 Jenkins Controller 上的 `jenkins` Linux 用户配置 GitHub 专用 Git 代理：

```bash
runuser -u jenkins -- git config --global \
  http.https://github.com.proxy http://10.10.10.3:7890
```

通过 `git config --get-urlmatch http.proxy https://github.com/chweca/mall.git` 验证配置匹配。随后 Jenkins #19 Pipeline 成功。

**审慎结论：** 对照实验强烈支持 GitHub 数据下载路径是主要影响因素；没有抓包证明 #18 当时所有连接的实际代理选择。成功验证意味着修复有效，但不等于从根本上证明互联网直连为何缓慢。

---

## 6. Jenkins #20：切换部署 Agent 后默认 Checkout 重复拉取 GitHub

**类型：真实故障；证据等级：A**
**涉及层：Declarative Pipeline → Agent Workspace → 自动 SCM Checkout → stash/unstash。**

### 现象与证据

Deploy 阶段报错包含：

```text
Remote call to k8s-master
ERROR: Error cloning remote repo 'origin'
curl 28 Failed to connect to github.com port 443: Connection timed out
```

`k8s-master` Agent 的 Git Fetch 发生在 `unstash` 前。该阶段原本只需要一份 Helm Chart，而不是整个 GitHub 仓库。

### 机制与修复

```text
git-test 主动 Checkout 后端仓库
          ↓
stash 保存 Chart
          ↓
切换 k8s-master Agent
          ↓
默认自动 SCM Checkout → GitHub 超时
          ↓
修复：关闭默认自动 Checkout
          ↓
unstash 恢复 Chart → 执行 Helm
```

Jenkinsfile 提交 `304e4d7` 添加：

```groovy
options {
    skipDefaultCheckout(true)
}
```

**关键区别：** 该选项只禁止 Jenkins 默认自动检出；Jenkinsfile 中显式写出的 `git` 步骤继续执行，Controller 仍需获取 Jenkinsfile。

### 验证

- Jenkins #21 Pipeline `SUCCESS`；
- Kubernetes `mall-admin:21` / `admin-web:21`；
- Deployment `2/2` 与 `1/1` Ready；
- 后端 `/actuator/health` 返回 `UP`；
- Helm Revision **34** `deployed`；浏览器访问正常。

---

## 7. 发布前风险审查：Helm Chart Diff 与 Dry-run

**类型：发布前预防性检查，不属于生产故障；证据等级：A**

### 原因

Helm 升级的是整个 Release。即使只修改两个镜像 tag，也不能默认认为其他资源完全不会变化。

### 实际做法

1. 检查当前 Helm 值与 Deployment：旧版 `mall-admin:gate12-public-url`、`admin-web:17` 一致；用户覆盖的 Values 为 `[]`。
2. 对比 GitHub Chart 与 `k8s-master` 本地 Chart：4 个文件存在空白字符差异，没有发现业务配置差异。
3. 对 GitHub Chart 执行 `helm upgrade --dry-run=server --hide-secret`，返回码为 `0`。
4. 比较 `helm get manifest` 与 `helm template`；原始文本其他差异行数为 `7`，在使用 `diff -uBZ` 排除空白行和行尾空格后，**其他非空白差异行为 0**。
5. 预期实质镜像改动：`mall-admin` 和 `admin-web` 更新为构建版本；正式 CD 后成为 `:21`。

**★ 重点记忆：`--dry-run` 测试能否模拟升级；Diff 复核改动范围；上线后还需要 Rollout、HTTP 和浏览器验证。**

---

## 8. 安全整改中的“险些出错”：变量外置但运行环境未补齐

**类型：配置发布风险，避免了一次不安全更新；不应描述为实际故障事故。**

### 关键风险

`mall-portal/src/main/resources/application-prod.yml` 中的数据库、Redis、Druid、RabbitMQ 密码改为 `${...}` 环境变量引用后，**旧 Pod 并不会自动获得这些变量**。如果直接切换新镜像而 Kubernetes Deployment 还未设置所需环境变量，可能无法正常连接依赖组件。

处理路径：源码外置 → 核对环境变量名称 → Helm Deployment `env.valueFrom.secretKeyRef` 接入 Secret → 验证 Secret 名称和 Key → 再发布镜像。

此外：旧 Git 历史与旧镜像里的旧凭据不会自动消失；RabbitMQ 持久化数据中的内部账号也未必因修改 Secret 自动更新。凭据轮换是独立任务，不可用“已经使用 Secret”代替。

---

## 9. 哪些 Gate 内容不要冒充真实故障？

| 旧阶段项目 | 合适的展示方式 | 原因 |
| --- | --- | --- |
| Gate 5：NFS PVC 从 Pending 到 Bound | 存储机制 / 实验验证 | 仅凭状态变化不能认定发生过存储事故 |
| Gate 6：Headless Service vs ClusterIP | 技术设计说明 | 这是服务发现机制，不是故障 |
| Gate 9：Actuator / Prometheus 指标 | 可观测性实践 | 未收集到明确的故障告警闭环证据 |
| Gate 10：HPA 扩缩容、PDB/Drain | 高可用和节点维护演练 | 是主动验证可靠性，不代表真实宕机事故 |
| PVE/SSH 间歇性断线 | 待补证据的故障候选 | 曾复现异常，但最终根因未充分证实，不应声称已根治 |
| Kubernetes `ImagePullBackOff` 细节 | 待补日志的历史案例 | 有复盘结论，仍值得补原始 Event 和应用日志 |

## 10. 面试最值得讲的四个案例

1. **Pod `Running` 为什么不能保证业务可用？** — 用 Gate 10 readinessProbe、EndpointSlice 解释。
2. **为什么 Java 上传成功，浏览器却看不到图片？** — 用 MinIO 内网 ENDPOINT / 公网 PUBLIC_URL 解释。
3. **为什么 Jenkins 切换 Agent 后又去 GitHub 下载？** — 用自动 Checkout、`stash/unstash` 解释。
4. **发布前怎样避免不小心改变其他资源？** — 用 Helm Release、Chart Diff、Dry-run 和 Rollout 解释。

建议讲述顺序：**现象（10秒）→ 调用链（15秒）→ 证据（20秒）→ 修复和验证（15秒）**。不要求背长命令，需能讲清楚组件职责和因果关系。

## 11. 后续补证据清单

- [ ] Gate 5：从旧终端记录中补 Harbor 拉取失败的 Pod Events、相关 Secret 引用与修复输出。
- [ ] Gate 5：补 JDBC 异常日志、Spring Boot 实际读取的连接配置（去除真实口令）。
- [ ] Gate 10：选取 readiness 错误路径、Pod `0/1`、Endpoint `Ready=false`、修复后就绪的脱敏截图（已有详细阶段笔记）。
- [ ] Gate 12：整理浏览器旧 OSS 403、MinIO 图片 URL 前后变化、22/37 迁移验证的脱敏截图。
- [ ] Jenkins：整理 #18、#20 报错与 #21 成功界面，隐藏不需要公开的个人信息。
- [ ] 部署说明中追加故障文档链接，并核对引用文件路径。

> 来源：本项目 Gate 10 和 Gate 12 的阶段复盘记录、2026-10-13 Jenkins #18/#20/#21 实际终端日志、GitHub `homemall/dockerization` 分支上的 Jenkinsfile 与 Helm Chart。本文对旧记录欠缺的原始输出明确标注了「待补证据」。
