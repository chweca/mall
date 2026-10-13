# HomeMall 部署与验证说明

> 项目类型：基于 [macrozheng/mall](https://github.com/macrozheng/mall) 开源电商项目的云原生部署与 DevOps 实践。
> 记录日期：2026-10-13。
> 本文基于个人实验室**已经完成过的部署**，不是适用于任意环境的“一键安装脚本”。所有私有 IP、域名、账号和仓库地址都需要按部署环境调整。
> **安全原则：不在 Git 仓库中提交真实密码、Token、私钥、Kubeconfig 或完整 Secret 明文。**

## 1. 项目部署目标

把现有 Spring Boot 商城项目容器化，通过私有镜像仓库管理镜像，用 Helm 管理 Kubernetes 资源，并借助 Jenkins 实现可人工控制的 CI/CD 发布。

核心步骤：

```text
源码（GitHub）
    ↓
Jenkins Controller 读取 Jenkinsfile 并调度
    ↓
git-test Agent：显式 Checkout → Maven 编译 → Docker 构建
    ↓
Harbor：保存版本化镜像
    ↓
DEPLOY_TO_K8S ?（默认为 false）
    ├── false：完成 CI，不修改 K8s
    └── true：切换 k8s-master Agent
                ↓
            unstash 获取 Helm Chart
                ↓
            helm upgrade 发布到 Kubernetes
                ↓
            Rollout、Pod、Service、浏览器验证
```

本项目已验证的范围：`mall-admin` 后端和 `admin-web` 管理前端的 CI/CD 镜像构建及部署；Chart 还包含 `mall-portal`、`mall-app-web` 等资源及外部依赖配置，但不等于已把上游全部服务、依赖一键部署完毕。

## 2. 实验环境与组件职责

| 组件 | 作用 | 本次实验记录 |
| --- | --- | --- |
| PVE / Rocky Linux | 提供多台虚拟机 | 自建实验环境 |
| Jenkins Controller | 读取 Jenkinsfile、调度流水线 | `10.10.10.35` |
| Jenkins Agent `git-test` | Git、Maven、Docker 构建与推送 | `10.10.10.60` |
| Jenkins Agent `k8s-master` | 执行 Helm / kubectl | Kubernetes 控制节点 |
| Kubernetes | 承载应用，Service、Ingress、Deployment 等 | kubeadm v1.30，`homemall-dev` |
| Harbor | 保存 Docker 镜像 | `10.10.10.10`，项目 `homemall` |
| Traefik | Ingress Controller，处理 HTTP 入口 | 由已有集群预先提供 |
| MySQL、Redis、MinIO 等 | 业务依赖 | **需要事先安装、配置与备份** |
| Helm | 管理应用配置和发布历史 | Release `homemall` |

> 这些 IP 是**实验室内部地址**，不能直接用于互联网或其他人的环境。所有组件是否对公网开放，需要独立进行安全检查。

### 2.1 为什么分成两个 Agent？

- **Controller（控制端）**：负责调度，不负责运行所有编译任务。
- **git-test（构建端）**：拉源码、执行 Maven、构建 Docker 镜像并推送 Harbor。
- **k8s-master（发布端）**：只需要 Helm Chart 和集群访问权限；无需重复克隆整个 GitHub 后端仓库。

每台 Agent 有自己的 **Workspace（工作空间）**。构建端的文件不会自动出现在部署端，所以使用 `stash` 暂存 Helm Chart，再通过 `unstash` 在部署端取回。

## 3. 仓库与关键配置归属

| 文件 / 目录 | 作用 |
| --- | --- |
| `Jenkinsfile` | 定义 CI/CD 阶段、Agent 和部署开关 |
| `mall-admin/pom.xml` | 后端 Maven 构建依赖与配置 |
| `mall-admin/Dockerfile` | 生成后端运行镜像（以实际仓库版本为准） |
| 前端仓库 `chweca/mall-admin-web` 中的 `Dockerfile` | 前端构建并打包静态站点为 Nginx 镜像 |
| `deploy/helm/homemall-chart/Chart.yaml` | Chart 元数据 |
| `deploy/helm/homemall-chart/values.yaml` | 镜像、端口、副本、服务地址等默认值 |
| `deploy/helm/homemall-chart/templates/` | Deployment、Service、Ingress、HPA、PDB 等资源模板 |
| `mall-portal/src/main/resources/application-prod.yml` | Spring Boot 生产环境配置及环境变量占位符 |

后端仓库：[chweca/mall](https://github.com/chweca/mall/tree/homemall/dockerization)（分支 `homemall/dockerization`）。前端构建仓库：[chweca/mall-admin-web](https://github.com/chweca/mall-admin-web)。

## 4. 部署前置条件：不能跳过的检查

**本节是部署前清单，不代表执行下方命令就能自动创建全部依赖。**

1. Kubernetes 集群可访问，节点运行正常，容器运行时与 Pod 网络正常。
2. 已安装 Traefik Ingress Controller；相关域名解析到正确的入口。
3. Harbor 可以从构建 Agent 推送，也可以从 Kubernetes 节点拉取；若为 HTTP 私有仓库，要在容器运行时做相应的可信配置。
4. `homemall-dev` 命名空间存在；应用所需的 MySQL、Redis、MinIO 等后端服务和存储已就绪。
5. 根据 Chart 模板创建了正确名称和键名的 Kubernetes Secret，并配置最小必要访问权限；**不能把 Secret 的真实值提交到仓库**。
6. Jenkins Controller 能读取 GitHub 中的 Jenkinsfile；两个 Agent 已注册且具有对应工具/权限。发布端只授予需要的 Kubernetes 操作权限。
7. 有数据库和持久化数据的备份/恢复预案。Helm 回滚不能自动回滚数据库内容。

### 4.1 Chart 引用的 Secret 名称与键名

**这里只列出键名，不展示或创建真实密码。** 根据本仓库当前 Chart：

| Secret | 必要的 key | 使用方 |
| --- | --- | --- |
| `harbor-regcred` | 镜像仓库拉取凭据，由 `imagePullSecrets` 引用 | 应用 Pod |
| `mysql-app-secret` | `MYSQL_APP_PASSWORD` | `mall-admin`、`mall-portal` |
| `redis-secret` | `REDIS_PASSWORD` | `mall-admin`、`mall-portal` |
| `minio-secret` | `MINIO_ACCESS_KEY`、`MINIO_SECRET_KEY` | `mall-admin` |
| `druid-login-secret` | `DRUID_LOGIN_PASSWORD` | `mall-portal` |
| `rabbitmq-secret` | `username`、`password` | `mall-portal` |

`imagePullSecrets` 使用的是 Docker Registry 类型凭据，与普通 `Opaque` Secret 的组织形式不同。`rabbitmq-secret` 的 Kubernetes 密码值也必须与 RabbitMQ **实际已有账号**一致；修改 Secret 本身不一定会修改数据库/持久化卷中已存在的 RabbitMQ 用户口令。

部署前可以**只查询资源和键名**，不要使用 `kubectl get secret -o yaml` 分享其值：

```bash
kubectl get namespace homemall-dev
kubectl get secret -n homemall-dev \
  harbor-regcred mysql-app-secret redis-secret minio-secret \
  druid-login-secret rabbitmq-secret
```

> 如果这些资源缺失，先在安全渠道准备凭据，再由授权人员按实际环境创建；**不要直接照抄实验室的账号、密码或业务地址**。对缺失的 Secret，不要启动相应的新镜像。

### 4.2 检查现有运行状况（纯查询）

```bash
kubectl get nodes
kubectl get deploy,svc,ingress -n homemall-dev
kubectl get pods -n homemall-dev -o wide
helm list -n homemall-dev
helm history homemall -n homemall-dev
```

阅读顺序：**集群正常 → 资源存在 → Pod 状态正常 → Helm Release 与历史版本明确**。

## 5. Helm Chart：先审核、再模拟、后发布

### 5.1 核心理解

```text
values.yaml：提供默认参数
       ↓
templates/：引用这些参数生成资源定义
       ↓
Helm Release：记录一次发布及其修订历史
       ↓
Kubernetes：实际管理 Deployment、Service、Ingress 等资源
```

`helm upgrade` 是对整个 Release 进行升级，不应把它当成只改一个镜像的命令。因此，本项目发布前检查了 GitHub Chart 与本地 Chart 的差异，并比较了旧、新的渲染清单。

### 5.2 在已有 Release 环境中模拟一次升级

以下命令适用于**已有 `homemall` Release 且依赖均准备好**的集群。使用 `:21` 仅演示本次已经发布的镜像标签，不代表应再次执行真实升级。

```bash
# 本地渲染：不修改集群
helm template homemall deploy/helm/homemall-chart \
  -n homemall-dev \
  --is-upgrade \
  --set-string mallAdmin.image.tag=21 \
  --set-string adminWeb.image.tag=21 \
  > /tmp/homemall-rendered.yaml

# 服务器端模拟：不提交本次升级
helm upgrade homemall deploy/helm/homemall-chart \
  -n homemall-dev \
  --set-string mallAdmin.image.tag=21 \
  --set-string adminWeb.image.tag=21 \
  --dry-run=server --hide-secret \
  > /tmp/homemall-dryrun.txt
```

- `helm template`：生成用于审查的资源 YAML。
- `--set-string`：将指定镜像标签以字符串形式覆盖 Chart 默认值。
- `--dry-run=server`：请求 Kubernetes API 参与模拟升级，但**不提交资源更新**。
- `--hide-secret`：隐藏模拟输出中的 Secret 对象；输出仍可能包含非 Secret 的内部配置，谨慎传播。

**注意：** `helm template` 或 `dry-run` 成功，不等于新镜像能启动。重新部署前还应检查 `helm get values`、实际 Secret、镜像仓库可达性与业务依赖。

### 5.3 变更审查原则

本次上线前做过以下核对：

- 本地 Chart 与 GitHub Chart 的源码差异仅涉及 4 个文件，人工确认主要是空格/空行。
- 比较 `helm get manifest` 与指定镜像 `:19` 的 `helm template` 结果，忽略纯空白行及尾随空格后，仅发现前后端两处镜像标签变化。
- 因 `helm plugin list` 未安装 Diff 插件，使用 `diff` 和 Helm 原生命令完成比较。

这是**当时特定版本、特定参数**的审查结果，不能自动推广到所有未来构建。每次有新 Chart 变更时都需要重新审查。

## 6. Jenkins CI/CD：真实配置和操作入口

### 6.1 Jenkins Job 设置

Jenkins 项目名称：`homemall-cicd`。流水线来源为 **Pipeline script from SCM**，Git 仓库使用后端 `chweca/mall`，分支为 `*/homemall/dockerization`，`Script Path` 为 `Jenkinsfile`。

首次验证时通过 Jenkins 网页**手动运行**；本次实践不宣称已经配置 GitHub Webhook 自动触发。

### 6.2 Jenkinsfile 的关键逻辑（节选）

```groovy
pipeline {
    agent { label 'git-test' }

    options {
        skipDefaultCheckout(true)
    }

    parameters {
        booleanParam(
            name: 'DEPLOY_TO_K8S',
            defaultValue: false,
            description: 'Deploy to Kubernetes'
        )
    }

    stages {
        // 省略构建阶段：详见仓库根目录 Jenkinsfile
        stage('Deploy to Kubernetes') {
            when {
                beforeAgent true
                expression { params.DEPLOY_TO_K8S }
            }
            agent { label 'k8s-master' }
            steps {
                unstash 'helm-chart'
                sh '''
                    helm upgrade homemall backend/deploy/helm/homemall-chart \
                      -n homemall-dev \
                      --set-string adminWeb.image.tag=${BUILD_NUMBER} \
                      --set-string mallAdmin.image.tag=${BUILD_NUMBER}

                    kubectl rollout status deployment/admin-web \
                      -n homemall-dev --timeout=120s
                    kubectl rollout status deployment/mall-admin \
                      -n homemall-dev --timeout=120s
                '''
            }
        }
    }
}
```

- `skipDefaultCheckout(true)`：关闭 Declarative Pipeline 自动检出源码；**显式 `git` 步骤仍正常执行**。
- `stash`：在 `git-test` 的 Workspace 暂存 Helm Chart 文件；`unstash`：在 `k8s-master` Workspace 恢复 Chart。
- `DEPLOY_TO_K8S=false`：默认只执行 CI。网页上勾选为 `true` 只影响**本次构建参数**，不会修改 Jenkinsfile。
- `BUILD_NUMBER`：使用 Jenkins 构建编号作为前、后端镜像标签；重新构建将产生新编号。

### 6.3 手动触发（带发布风险）

```text
Jenkins → homemall-cicd → Build with Parameters
       ↓
DEPLOY_TO_K8S 不勾选：只做 CI
DEPLOY_TO_K8S 勾选：CI 成功后执行 CD（真实修改 Kubernetes）
       ↓
Build → Pipeline Overview → Console Output
```

**不能把勾选发布参数当成无风险操作。** 会触发镜像构建、Harbor 推送和 Kubernetes 滚动更新。正式上线前应备份并记录基线版本，最好设置审批、告警和回滚预案。

## 7. 发布后验证：按层检查

### 7.1 Jenkins 与 Harbor

1. Jenkins 流水线执行成功；`Deploy to Kubernetes` 阶段确实执行而非 `Skipped`。
2. Harbor 中检查 `mall-admin:${BUILD_NUMBER}`、`admin-web:${BUILD_NUMBER}` 镜像标签是否存在。

### 7.2 Kubernetes 镜像、Pod 就绪和发布历史

```bash
# 确认当前 Deployment 使用的镜像标签
kubectl get deployment mall-admin admin-web -n homemall-dev \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[*].image'

# 确认就绪、更新和可用副本
kubectl get deployment mall-admin admin-web -n homemall-dev

# 确认 Helm 修订记录
helm history homemall -n homemall-dev | tail -5
```

**2026-10-13 本次实测结果：**

| 项目 | 结果 |
| --- | --- |
| Jenkins 构建 | `#21` SUCCESS |
| 运行镜像 | `mall-admin:21`、`admin-web:21` |
| Deployment | `mall-admin 2/2`、`admin-web 1/1`，UP-TO-DATE 和 AVAILABLE 符合预期 |
| Helm 发布 | Revision **33 → 34**，Revision 34 为 `deployed` |

### 7.3 从前端容器访问后端 Service

```bash
kubectl exec -n homemall-dev deployment/admin-web -- \
  wget -qO- http://mall-admin:8080/actuator/health
```

本次实际返回：

```json
{"status":"UP","groups":["liveness","readiness"]}
```

这个测试证明当前请求能够经过 Kubernetes 内部网络和 `mall-admin` Service 到达健康的后端应用；`groups` 字段只是说明存在这两个健康分组，并不意味着分别测试了两个接口。

### 7.4 浏览器验证

在允许访问的网络中打开实际部署的商城网页，进行登录、商品列表等基础操作（按环境和测试账号权限）。本次发布后，人工检查网页访问正常。**并未据此声称全量业务功能、性能、安全和故障演练已经通过。**

## 8. 失败后的排查和回滚原则

诊断顺序：

```text
Jenkins Controller 能获取 Jenkinsfile 吗？
      ↓
Agent 上的 Checkout、Maven、Docker 是否成功？
      ↓
Harbor 是否存在正确镜像 Tag？
      ↓
CD 阶段是否执行了 unstash / helm upgrade？
      ↓
Helm Release 当前 Revision 是什么？
      ↓
Deployment 镜像、Pod READY、事件/日志如何？
      ↓
HTTP 健康检查和浏览器业务是否恢复？
```

**本次真实遇到的两类 Git 故障：**

- Controller 下载 GitHub 缓慢：以 `jenkins` 系统用户对比直连与经 Mihomo 的 Git 下载，代理路径明显更快；配置 Jenkins 用户访问 GitHub 的 Git 代理后，后续 SCM 获取成功。
- 切换到 `k8s-master` 后再次自动 Git Fetch：Jenkins Declarative Pipeline 默认 Checkout 与已设计的 `stash/unstash` 重复；配置 `skipDefaultCheckout(true)` 后，后续 `#21` 完成发布。

**回滚只在确认需要时执行：**

```bash
# 先只读查看历史，明确稳定的回滚目标
helm history homemall -n homemall-dev

# 示意：在充分确认目标版本、业务影响和备份后执行；此命令会修改运行环境
# helm rollback homemall <目标Revision> -n homemall-dev
```

**注意：** Revision 33 是本次发布之前的稳定基线，但 Helm 历史可能在后续持续变化，不应把 `33` 写死在通用回滚脚本里。回滚 Helm 主要恢复 Helm 管理的资源配置，不能自动恢复 MySQL 数据、镜像仓库或 RabbitMQ 内部持久化账号。

## 9. 安全与复现边界

- 为 Spring Boot 配置文件中的部分口令使用 `${ENV_VAR}` 占位符，并通过 Deployment `env.valueFrom.secretKeyRef` 注入；这不自动创建 Secret，也不保证已运行容器同步更新环境变量。
- 旧 Git 提交、旧容器镜像中可能包含曾硬编码的凭据；需要审计并轮换仍有效的凭据，不能仅删除最新文件中的密码。
- 发布账户应使用最小权限，Jenkins Agent 访问 Docker Socket、Harbor、Kubernetes 的权限要单独评估。
- 该 Chart 引用了实验室固定 IP、服务名、Secret、存储和内部 DNS，因此当前仓库**尚不能声称是可移植的一键部署方案**。
- 上游代码归原作者及贡献者；这里重点展示本人的部署改造、集成和可验证的运维实践。

## 10. 参考文档

- [项目首页](../README.md)
- [系统架构](architecture.md)
- [Helm Chart](../deploy/helm/homemall-chart/)
- [Jenkinsfile](../Jenkinsfile)
- [上游业务项目](https://github.com/macrozheng/mall)
