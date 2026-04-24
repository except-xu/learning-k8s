# KEDA部署与实践



KEDA (Kubernetes Event-Driven Autoscaler) 是为Kubernetes设计的开源、轻量级且功能强大的事件驱动自动伸缩器。它于2023年8月正式从云原生计算基金会（CNCF）毕业，成为一个成熟稳定的项目。简单来说，KEDA的核心价值在于让Kubernetes中的应用能够根据业务事件（例如消息队列中的任务数量、数据库中的待处理记录等）自动进行扩缩容，而不仅仅是依赖CPU或内存使用率。



## KEDA简介

Kubernetes原生的水平Pod自动伸缩器（HPA）功能强大，但它主要基于CPU、内存等资源指标进行扩缩容。这种方式存在一个“被动”和“滞后”的问题：应用必须等到CPU或内存飙升后，HPA才会触发扩容，这在流量突增时可能导致服务被“冲垮”。KEDA 的出现正是为了解决这个问题，它作为 HPA 的增强和补充，而非替代品。

- 从“被动响应”到“主动伸缩”：KEDA 像一个敏锐的“工头”，时刻监控着各种外部事件源。一旦发现任务积压（例如 Kafka 队列中出现了新消息），它会立即通知 HPA 进行扩容，实现“有活就干”的主动伸缩模式。
- 简化复杂指标的实现：虽然 HPA 也支持通过 prometheus-adapter 等方式使用自定义指标，但实现过程相对繁琐。KEDA 内置了数十种常见的“Scaler”（伸缩器），可以直接对接各种第三方应用，大大简化了配置。

### KEDA 如何工作？

KEDA 的核心架构包含两个主要组件：

1. KEDA Operator（操作员）：负责监听用户定义的伸缩规则（通过 ScaledObject 资源），并根据事件源的状态来管理目标工作负载的副本数。
2. Metrics Server（指标服务器）：它将 KEDA 从各种事件源获取的数据，转换成 Kubernetes HPA 能够识别的外部指标（External Metrics）格式。

**工作流程如下**：

1. 用户通过 ScaledObject 自定义资源定义伸缩规则，指定目标工作负载（如 Deployment）和触发事件（如 Kafka 队列长度）。
2. KEDA Operator 持续轮询指定的事件源，获取事件数据（如消息积压量）。
3. KEDA Metrics Server 将这些事件数据暴露为外部指标。
4. KEDA 会自动创建并管理一个 HPA 资源，该 HPA 读取 KEDA 提供的外部指标。
5. HPA 根据指标值（如队列长度超过阈值）自动调整目标工作负载的副本数，实现扩容或缩容。

### KEDA 的核心特性

- 事件驱动：支持根据消息队列（Kafka, RabbitMQ 等）、数据库（MySQL, PostgreSQL 等）、监控系统（Prometheus）等多种外部事件源进行伸缩。
- 缩容到零 (Scale to Zero)：当没有事件触发时，KEDA 可以将应用副本数缩减到 0，实现资源的极致节省，非常适合间歇性工作的服务。
- 丰富的 Scaler 目录：内置了数十种开箱即用的 Scaler，覆盖了绝大多数常见场景。如果内置的不满足需求，还支持开发自定义 Scaler。
- 支持 Cron 定时伸缩：可以根据预设的时间表（如业务高峰期前）进行定时扩容或缩容。
- 自动管理 HPA：用户只需关注业务事件，KEDA 会在后台自动创建和管理 HPA，简化了运维复杂度。



### KEDA和HPA

KEDA 和 HPA 的核心区别在于它们的触发机制和伸缩能力。简单来说，KEDA 是 HPA 的增强版，它让 Kubernetes 的伸缩能力从“被动”走向了“主动”。最核心的区别可以概括为以下两点：

1. 触发指标不同
   - HPA (水平 Pod 自动伸缩器)：主要依赖 CPU 和内存 等资源利用率指标。它是一种“被动”的伸缩方式，只有当 Pod 的负载（如 CPU）升高后，HPA 才会触发扩容。
   - KEDA (事件驱动自动伸缩器)：依赖 外部事件 作为触发源，例如 Kafka 消息队列的积压数量、数据库中的待处理任务数、Prometheus 监控的自定义业务指标等。这是一种“主动”的伸缩方式，只要有任务到来，就能立即触发扩容，无需等待 CPU 升高。

2. 伸缩范围不同
   - HPA：通常无法将副本数缩减到 0。这意味着即使在完全没有负载的空闲时段，也至少会有一个 Pod 在运行，会持续消耗资源。
   - KEDA：支持 缩容到零 (Scale to Zero)。当没有任何事件需要处理时，KEDA 可以将 Pod 副本数降为 0，实现资源的极致节省。当新事件到来时，再迅速从 0 扩容。



**协作关系**：KEDA如何增强HPA？

KEDA并非要取代 HPA，而是作为HPA的补充和增强。它们通常协同工作：

- KEDA负责“感知”：KEDA监听各种外部事件源，并将这些事件数据转换成 HPA 能够识别的“外部指标 (External Metrics)”。
- HPA负责“执行”：KEDA会自动创建并管理一个HPA对象。这个 HPA 读取 KEDA 提供的外部指标，并根据指标值来实际执行Pod副本数的扩缩容操作。

我们可以把KEDA想象成一个敏锐的“工头”，它时刻监控着任务队列。一旦发现新任务，它就立刻告诉“施工队”HPA 需要增加人手。而 HPA 就是那个负责具体增减人手的“施工队”。



## 部署和使用

### 部署

部署 KEDA 最通用且推荐的方式是使用 Helm，它适用于大多数 Kubernetes 环境（如自建集群、TKE 等）。如果你使用的是云厂商的托管 Kubernetes 服务（如 Azure AKS），通常也支持通过云平台的插件或加载项直接启用。下面我们基于Helm来按需部署。

1. 添加 KEDA Helm 仓库

   首先，我们运行如下命令将 KEDA 的官方 Helm 仓库添加到本地环境中：

   ```bash
   helm repo add kedacore https://kedacore.github.io/charts
   helm repo update
   ```

2. 安装 KEDA

   接下来，我们使用Helm安装KEDA到指定的命名空间中，若名称空间不存在则要自动创建：

   ```bash
   helm install keda kedacore/keda --namespace keda --create-namespace --create-namespace
   ```

   > 💡 国内环境特别提示：
   > 如果你在国内环境部署，可能会遇到镜像拉取失败的问题。这时，你可以创建一个values.yaml文件，将镜像仓库地址替换为可用的镜像源（如Docker Hub的镜像同步仓库），然后再使用 -f values.yaml 参数进行安装。

3. 验证部署

   安装完成后，检查 Pod 状态以确保所有组件都已正常运行：

   ```bash
   kubectl get pods -n keda
   ```

   需要确保 keda-operator、keda-operator-metrics-apiserver 等组件处于 Running 状态。



**部署后的关键注意事项：**
部署只是第一步，为了让KEDA稳定运行，我们需要注意以下几点：

1. **避免与现有HPA冲突**
  KEDA 底层是通过创建和管理 HPA 来工作的，因此，**不要**对同一个工作负载（Deployment/StatefulSet）同时手动创建 HPA 和 KEDA 的 ScaledObject，这将会导致两个控制器争夺控制权，引发不可预测的伸缩行为。
2. **身份验证配置**
  KEDA 需要访问外部事件源（如 AWS SQS, Azure Service Bus, Kafka 等）。在生产环境中，建议使用工作负载身份 (Workload Identity) 或Pod Identity来安全地配置KEDA与云资源的认证，而不是将密钥硬编码在配置文件中。
3. **版本兼容性**
  在安装前，建议检查KEDA版本与当前Kubernetes集群版本的兼容性。例如，较新的KEDA版本可能不再支持旧版Kubernetes API，或者在升级集群前需要确认KEDA是否需要先行升级。



### 核心使用逻辑

KEDA 的使用逻辑非常清晰，它遵循 Kubernetes 的声明式 API 理念。用户不需要编写复杂的脚本或程序来监控和伸缩，而是只需要声明“想要什么”，KEDA 就会负责“如何实现它”。这个逻辑可以分解为三个核心要素：
1. 目标 (Target)：也就是想让哪个目标对象进行伸缩？它通常指向Kubernetes的一个Deployment 或 StatefulSet。
2. 触发器 (Trigger)：根据什么来伸缩？这是 KEDA 的灵魂。触发器定义了外部事件源，比如 Kafka 队列的积压消息数、Prometheus 的某个业务指标、AWS SQS 的队列长度等。
3. 规则 (Rules)：具体怎么伸缩？也就是定义伸缩的边界和阈值，例如：最小/最大副本数是多少？触发扩容的阈值是多少？缩容的冷却时间是多久？



**逻辑总结**：通过创建一个名为 ScaledObject 的自定义资源，将目标、触发器和规则这三者绑定在一起。KEDA 接收到这个声明后，便会自动化地完成后续的监控和伸缩工作。



### ScaledObject 

ScaledObject 是 KEDA 中最核心的自定义资源定义（CRD）。简单来说，如果说KEDA是Kubernetes的“自动伸缩引擎”，那么ScaledObject就是“控制面板”。用户不需要编写复杂的代码或脚本，只需要编写一个 ScaledObject 的 YAML 文件，告诉 KEDA 具体的需求，KEDA 就会自动完成剩下的所有工作。

ScaledObject是我们与KEDA交互的唯一接口，所有的伸缩逻辑都在这一个文件中定义。



#### ScaledObject 结构

一个标准的 ScaledObject 通常包含以下几个关键部分：
1. 元数据 (Metadata)：定义资源的名称和命名空间。
2. 伸缩目标 (scaleTargetRef)：这是KEDA需要控制的对象。
- name: Deployment或StatefulSet的名字。
- kind: 资源类型，通常是Deployment。
3. 伸缩边界 (Scaling Bounds)：控制副本数的上下限，防止资源耗尽或服务不可用。
- minReplicaCount: 最小副本数。KEDA 的强大之处在于支持 0，即没有任务时完全停止服务以节省成本。
- maxReplicaCount: 最大副本数。用于防止因指标异常导致的无限扩容。
4. 触发器 (Triggers)：这是 ScaledObject 的灵魂，支持配置一个或多个触发器。
- type: 指定事件源类型（如 prometheus, kafka, aws-sqs, rabbitmq, cron 等）。
- metadata: 具体的配置参数，如队列名称、服务器地址、阈值等。
5. 高级配置 (Advanced Settings)
- pollingInterval: KEDA 检查事件源的频率（秒）。
- cooldownPeriod: 缩容前的冷却时间（秒），防止在阈值附近频繁抖动。



#### 示例

下面是一个基于 Prometheus 指标进行伸缩的 ScaledObject 完整示例：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-app-scaler          # 1. 定义名称
  namespace: default
spec:
  # 2. 指定要伸缩的目标应用
  scaleTargetRef:
    name: my-app-deployment    # 对应的 Deployment 名称
    kind: Deployment           # 资源类型

  # 3. 定义伸缩边界
  minReplicaCount: 1           # 最少保留 1 个副本（也可以设为 0）
  maxReplicaCount: 10          # 最多扩展到 10 个副本

  # 4. 高级参数
  pollingInterval: 15          # 每 15 秒检查一次指标
  cooldownPeriod: 300          # 缩容前等待 300 秒（5分钟）

  # 5. 定义触发器
  triggers:
  - type: prometheus           # 使用 Prometheus 作为触发源
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: http_requests_per_second
      threshold: '100'         # 目标阈值：每个 Pod 处理 100 QPS
      query: sum(rate(http_requests_total{job="my-app"}[5m]))
```



#### 关键特性与最佳实践

1. 自动管理 HPA：我们不需要手动创建HPA，在应用ScaledObjec 后，KEDA会自动在后台创建一个 HPA 资源。如果直接修改 HPA，也可能会被KEDA覆盖，因此请始终通过修改ScaledObject来管理伸缩。
2. 多触发器支持：我们可以在一个ScaledObject中配置多个触发器。例如，既监控 CPU 使用率，又监控 Kafka 队列长度。KEDA 会综合评估，只要任一触发器满足条件，就会触发扩容。
3. 避免冲突：不要对同一个 Deployment 同时使用原生的 HPA 和 KEDA 的 ScaledObject。这会导致两个控制器互相竞争，产生不可预测的震荡行为。
4. 身份验证分离：如果触发器需要认证（如连接 AWS SQS 或 RabbitMQ），建议使用 TriggerAuthentication 资源来管理敏感信息（如密码、密钥），而不是直接写在 ScaledObject 的 metadata 中，以提高安全性。



总结来说，ScaledObject 是 KEDA 的“大脑”，它将复杂的监控和伸缩逻辑封装在一个简单的 YAML 文件中，让你能够轻松实现事件驱动的自动伸缩。













