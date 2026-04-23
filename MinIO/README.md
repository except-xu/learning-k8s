# MinIO部署和使用实践

MinIO是一个高性能、兼容Amazon S3协议的开源对象存储系统，专为AI/ML、分析型和大规模数据密集型场景设计。它以速度、可扩展性和云原生特性著称，并在 Kubernetes 环境中表现尤为突出。  

## MinIO简介

### **MinIO 的核心特性：**

**1. 高性能对象存储（High Performance）**

MinIO 专为高吞吐、低延迟的数据访问而设计，适用于 AI/ML、数据湖、分析型工作负载等需要快速读写的场景。

**2. 完全兼容 S3 API**

它提供与 Amazon S3 兼容的 API，使现有 S3 工具、SDK 和应用无需修改即可使用。

**3. 云原生设计（Native to Kubernetes）**

MinIO 是少数真正为 Kubernetes 原生构建的对象存储系统之一，可在任何云、任何 K8s 发行版、私有云或边缘环境运行。

**4. 开源（AGPLv3）**

MinIO 采用 GNU AGPLv3 许可证，完全开源，社区可自由使用、修改和再分发。

**5. 面向 AI 与数据密集型场景优化**

官方明确指出 MinIO 针对 AI/ML、分析型任务和大规模数据管道进行了优化，是现代数据基础设施的重要组件。

**6. 软件定义、跨平台**

MinIO 不依赖特定硬件，可在裸机、虚拟机、公有云、私有云和边缘设备上运行。

### MinIO 的典型使用场景

- AI/ML 模型与训练数据存储
- 大规模日志、事件、对象数据湖
- 云原生应用的持久化对象存储
- 替代 Amazon S3 的本地或混合云方案
- **与 vLLM、SGLang、RAG 系统集成作为模型仓库**



MinIO的GitHub地址：官方仓库 **[MinIO](https://github.com/minio/minio)**  ，该仓库已于2026年2月归档，但仍可作为社区版源码参考。



## 部署MinIO

使用Helm在Kubernetes上安装MinIO的核心流程是：添加官方Helm仓库 → 配置values → 安装 → 验证。

### 示例 values.yaml

需要注意，如下values示例文件中使用了名为"openebs-minio-localpv"的存储类，它由OpenEBS提供，且面向MinIO的需求进行的特定优化。如果使用的是其它存储类，请注意变更。

```yaml
mode: standalone

rootUser: "admin"
rootPassword: "magedu.com"

replicas: 1

persistence:
  enabled: true
  size: 50Gi
  storageClass: "openebs-minio-localpv"    # 使用指定的存储类（由OpenEBS提供）

resources:
  requests:
    cpu: 500m
    memory: 2Gi

service:
  type: ClusterIP
  port: 9000

consoleService:
  type: ClusterIP
  port: 9001

## -----------------------------
## MinIO API Service (9000)
## -----------------------------
service:
  type: ClusterIP
  port: 9000

## -----------------------------
## MinIO Console Service (9001)
## -----------------------------
consoleService:
  type: LoadBalancer     # 通过LoadBalancer暴露Console，如无需要，可将其修改为ClusterIP
  port: 9001
  annotations: {}        # 如需指定LB类型可在此添加


## -----------------------------
## Ingress for MinIO API (9000)
## -----------------------------
ingress:
  enabled: false   # 如需开放API，可改为true
  ingressClassName: nginx
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
  hosts:
    - minio.magedu.com.com
  paths:
    - /
  tls: []


## ------------------------------------
## Ingress for MinIO Console (9001)
## ------------------------------------
consoleIngress:
  enabled: true      # 使用Ingress开放MinIO Console
  ingressClassName: nginx
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
  hosts:
    - minio-console.magedu.com
  paths:
    - /
  tls: []
```



### 部署过程

1. 添加MinIO 官方 Helm 仓库

   Helm需要从MinIO官方仓库拉取 chart，相关的命令如下：

   ```bash
   # 添加仓库
   helm repo add minio https://charts.min.io/
   
   # 更新仓库索引
   helm repo update
   
   # 确认添加结果
   helm repo list
   ```

2. 准备自定义 values.yaml（推荐）

   我们需要通过values.yaml配置存储、访问密钥、持久化、服务暴露方式等关键参数，具体可参考前一节示例的内容。

   - 设置访问密钥：rootUser、rootPassword
   - 配置持久化：persistence.enabled=true
   - 配置存储大小：persistence.size=50Gi
   - 配置 Service 类型：ClusterIP 或 LoadBalancer

3. 安装 MinIO

   接下来使用helm命令部署MinIO到Kubernetes集群即可，下面示例命令是指定的目标名称空间为minio。

   ```bash
   # 创建helm release
   helm install minio minio/minio -n minio -f values.yaml --create-namespace
   
   # 确认Pod启动完成
   kubectl get pods -n minio
   ```

4. 验证 MinIO 服务是否正常运行

   接下来，我们要确保MinIO已成功启动并可访问，一个简单的方式为使用“kubectl get svc -n minio”命令进行。

5. 访问 MinIO 控制台

   根据前面示例文件中的定义，我们可通过Ingress中定义的主机名“minio-console.magedu.com”或直接使用minio-console的LoadBalancer IP进行服务访问。默认的用户名和密码分别为admin/magedu.com。



### 生产环境额外建议

1. 多节点分布式MinIO（高可用）

- 设置 `mode: distributed`
- 至少 4 个 MinIO 实例
- 配置多个卷：`volumesPerServer: 4`

2. 使用TLS

- 提供证书 secret：`tls.enabled=true`
- 绑定证书：`tls.certSecret=your-cert-secret`

3. 与 LLM 推理平台集成

你在课程中可以强调：
- MinIO 作为 **模型仓库（Model Store）**
- vLLM / SGLang / Triton 通过 S3 API 拉取模型
- 支持版本化、热更新、蓝绿模型切换



## 使用示例

mc是管理MinIO的“瑞士军刀”  ，可用于管理 MinIO、S3、兼容 S3 的对象存储，功能远比 AWS CLI 更强。



#### 一、主机管理（连接MinIO）

1. 添加主机（最常用）

```bash
mc alias set myminio http://minio.example.com testkey magedu.com
```

2. 查看主机列表

```bash
mc alias list
```

3. 删除主机

```bash
mc alias remove myminio
```



#### 二、Bucket（桶）管理

1. 列出所有 bucket

```bash
mc ls myminio
```

2. 创建 bucket

```bash
mc mb myminio/mybucket
```

3. 删除 bucket

```bash
mc rb myminio/mybucket
```

4. 查看 bucket 信息

```bash
mc stat myminio/mybucket
```

5. 设置 bucket 为公开

```bash
mc anonymous set public myminio/mybucket
```

6. 设置 bucket 为私有

```bash
mc anonymous set private myminio/mybucket
```



#### 三、对象（文件）管理

1. 上传文件

```bash
mc cp ./file.txt myminio/mybucket/
```

2. 下载文件

```bash
mc cp myminio/mybucket/file.txt ./
```

3. 删除对象

```bash
mc rm myminio/mybucket/file.txt
```

4. 递归上传目录

```bash
mc cp --recursive ./data/ myminio/mybucket/
```

5. 同步目录（非常常用）

```bash
mc mirror ./data/ myminio/mybucket/
```



#### 四、用户与权限管理

1. 创建用户

```bash
mc admin user add myminio newuser newpassword
```

2. 删除用户

```bash
mc admin user remove myminio newuser
```

3. 查看用户

```bash
mc admin user info myminio newuser
```

4. 绑定策略

```bash
mc admin policy attach myminio readwrite --user newuser
```

5. 查看策略

```bash
mc admin policy info myminio readwrite
```



#### 五、MinIO集群管理（admin命令）

1. 查看集群信息

```bash
mc admin info myminio
```

2. 查看节点状态

```bash
mc admin health myminio
```

3. 查看磁盘状态

```bash
mc admin storage info myminio
```

4. 重启 MinIO 节点

```bash
mc admin service restart myminio
```

#### 六、监控与日志

1. 查看实时日志

```bash
mc admin console myminio
```

2. 查看 Prometheus 指标

```bash
mc admin prometheus generate myminio
```

#### 七、复制与备份（非常常用）

1. 同步两个 MinIO 之间的数据

```bash
mc mirror myminio/mybucket backupminio/mybucket
```

2. 双向同步

```bash
mc mirror --watch myminio/mybucket backupminio/mybucket
```

#### 八、诊断

1. 检查连接

```bash
mc admin info myminio
```

2. 检查权限

```bash
mc admin policy info myminio readwrite
```

3. 检查桶策略

```bash
mc anonymous get myminio/mybucket
```

























