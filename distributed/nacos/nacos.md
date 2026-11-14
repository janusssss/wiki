# Nacos 知识点

## 一、Nacos 是什么

阿里开源的动态服务发现与配置管理平台（Dynamic Naming and Configuration Service）。

- 核心能力：**服务注册与发现** + **动态配置管理**
- 定位：微服务架构的基础设施，Spring Cloud Alibaba 的关键组件
- 协议：兼容 OpenAPI，2.x 客户端使用基于 gRPC 的长连接
- 协议特点：开源（Apache 2.0），Java 实现，可独立部署

## 二、两大核心模块

### 1. 配置管理（Config Service）

- 集中管理应用配置，改配置无需重启，实时生效
- 配置三元组定位：`Namespace + Group + DataId`
  - `Namespace`：环境隔离（如 dev / test / prod）
  - `Group`：业务分组，默认 `DEFAULT_GROUP`
  - `DataId`：单个配置文件，如 `application.yaml`
- 客户端通过**长轮询**（Long Polling，约 30s 超时）感知配置变更
- 配置变更历史与一键回滚

### 2. 服务注册发现（Naming Service）

- 微服务启动时向 Nacos 注册自身实例（IP + 端口 + 权重 + 元数据）
- 消费者从 Nacos 拉取服务实例列表并做负载均衡
- 支持权重负载均衡、同集群优先就近访问、元数据路由
- 临时实例（Ephemeral）与持久实例（Persistent）两种

## 三、一致性模型（重点）

| 模块 | 一致性协议 | 说明 |
|------|-----------|------|
| Naming（临时实例） | **Distro**（AP） | 阿里自研，各节点分摊数据写入，允许短暂不一致 |
| Naming（持久实例） | **JRaft**（CP） | Raft 的 Java 实现 |
| Config | **JRaft**（CP） | 选主、日志复制保证配置一致性 |

设计思路：临时实例重可用性（服务挂了不该阻塞整个注册中心），配置重一致性（配置写错会出事故）。

## 四、健康检查

- **临时实例**：客户端定时发送心跳（默认 5s 一次），15s 未收到标记不健康，30s 未收到剔除实例
- **持久实例**：由 Nacos 服务端主动探测（HTTP/TCP）

## 五、部署模式

| 模式 | 说明 |
|------|------|
| 单机 | 内置 Derby 数据库，适合开发测试 |
| 集群 | 推荐外接 MySQL（多节点共享），3 节点起步 |
| 仲裁模式 | 官方容器化部署方案，节点不存数据只做选举 |

⚠️ 2.x 集群注意：客户端 gRPC 长连接端口 = 服务端主端口 + 10000，防火墙要把 `8848` 和 `9848` 都放行（gRPC 端口默认偏移 +1000，即 `8848+1000=9848`，Nacos 2.x 官方文档说明）。

## 六、客户端使用（OpenAPI 示例）

```bash
# 注册实例
curl -X POST "http://127.0.0.1:8848/nacos/v1/ns/instance" \
  -d "serviceName=demo&ip=127.0.0.1&port=8080"

# 发现实例
curl "http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=demo"

# 发布配置
curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs" \
  -d "dataId=demo.yaml&group=DEFAULT_GROUP&content=key:value"

# 拉取配置
curl "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=demo.yaml&group=DEFAULT_GROUP"
```

## 七、Nacos 1.x vs 2.x

| 特性 | 1.x | 2.x |
|------|-----|-----|
| 通信 | HTTP 长轮询 | gRPC 长连接（推拉结合） |
| 配置变更通知 | 延迟最高 ~30s | 毫秒级推送 |
| 服务端内部 | HTTP 同步 | gRPC 流式同步 |
| 性能 | 一般 | 大幅提升 |

1.x 客户端在 2.x 服务端上仍可兼容使用。

## 八、常见坑

1. **集群脑裂**：集群节点数必须为奇数，且跨节点互通；Distro 协议下节点下线数据会自动同步恢复
2. **MySQL 兼容性**：Nacos 官方支持 MySQL 5.6.5+，8.0 需要在 JDBC URL 加 `allowPublicKeyRetrieval=true&useSSL=false`
3. **命名规范**：DataId 推荐 `${prefix}-${spring.profile.active}.${file-extension}` 格式
4. **鉴权**：2.x 默认关闭鉴权，生产环境务必开启（修改 `application.properties` 中 `nacos.core.auth.enabled=true`），否则任何人都能改配置
5. **心跳超时**：容器化部署注意系统时间同步，时钟漂移会导致实例被误剔除

## 九、同类组件对比

| 组件 | 特点 | 适用场景 |
|------|------|---------|
| Nacos | 配置 + 注册二合一，AP/CP 混合，Java 生态 | Spring Cloud Alibaba 首选 |
| Consul | Go 实现，CP，内置 KV + 服务网格集成 | 多云、Service Mesh |
| etcd | Go 实现，CP，纯 KV | K8s 底层，自研体系 |
| ZooKeeper | CP，老牌，ZAB 协议 | 大数据生态（Hadoop/Kafka） |
| Eureka | AP，仅注册发现，已停止新功能开发 | 老 Spring Cloud 项目 |

## 十、总结

Nacos 的核心价值 = **配置中心 + 注册中心二合一**，开箱即用，管理界面友好。
Java 微服务选型基本绕不开；非 Java 生态可选 Consul 或 etcd。

## 十一、常见面试问题

### Q1：Nacos 和 Eureka 有什么区别？

- **一致性模型**：Eureka 是纯 AP（永远不拒绝服务）；Nacos 临时实例 AP（Distro）、持久实例 CP（JRaft）
- **功能**：Eureka 只有注册发现；Nacos 配置 + 注册二合一
- **健康检查**：Eureka 靠客户端心跳 + 自我保护机制；Nacos 临时实例心跳 + 服务端主动探测持久实例
- **维护状态**：Eureka 1.x 已停止新功能开发；Nacos 持续迭代

### Q2：为什么 Nacos 临时实例用 AP 而不用 CP？

注册中心的场景下，**可用性比强一致更重要**。如果网络分区时 CP 导致注册中心不可用，所有服务都发现不了彼此，整个系统瘫痪；而 AP 只是短暂拿到过期的实例列表，消费端重试或熔断可以兜住。配置数据则相反，写错配置后果严重，所以用 CP。

### Q3：Nacos 的 Distro 协议了解吗？

Distro 是 Nacos 自研的 AP 分布式协议，核心思想：
- 每个节点负责一部分数据（按 hash 分片），**写请求路由到负责该数据的节点**
- 负责节点把数据**同步给其他所有节点**（每节点都有全量副本）
- 节点间定时心跳校验（默认 5s），发现丢数据就从邻居全量拉取
- 节点下线不影响整体可用性，新节点上线自动同步数据

和 Raft 的区别：Raft 是选主 + 日志复制，写必须走 Leader；Distro 无选主，多节点都能写自己分片的数据，吞吐更高但不保证强一致。

### Q4：Nacos 集群挂了一台还能用吗？

- **临时实例**：能。Distro 协议下每个节点都有全量数据副本，注册和发现不受影响
- **配置读取**：能，本地有缓存，且其他节点仍在
- **配置写入**：JRaft 需要多数派存活，3 节点挂 1 台仍可写，挂 2 台不可写

### Q5：Nacos 1.x 和 2.x 的区别？为什么换 gRPC？

1.x 用 HTTP 长轮询做配置变更通知，客户端发起、30 秒超时窗口，延迟高且服务端压力大。
2.x 改为 gRPC 长连接，服务端**主动推送**变更，毫秒级生效；同时服务端节点间也用 gRPC 流式同步，性能大幅提升。
兼容性：1.x 客户端可直接连 2.x 服务端。

### Q6：配置热更新的原理？应用侧要注意什么？

服务端 → gRPC 推送（或 1.x 长轮询响应）→ 客户端更新本地缓存快照 → 触发监听器回调。
应用侧注意：
- Spring Cloud 需要 `@RefreshScope` 或监听 `EnvironmentChangeEvent`，否则 Bean 里缓存的老值不会变
- 推送只保证"新配置已到达进程"，业务逻辑如何生效要自己设计

### Q7：Nacos 怎么保证配置的安全性？

- 开启鉴权：`nacos.core.auth.enabled=true`，默认用户名密码 `nacos/nacos`，生产必须改
- 配置加密：结合 Jasypt 等对敏感项（数据库密码、AK/SK）加密后再存入
- namespace/group 做权限隔离（2.x 支持细粒度权限控制）
- 配置变更历史 + 回滚做操作审计

### Q8：Nacos 集群部署的要点？

- 节点数奇数起步（3/5），JRaft 需多数派存活
- 外接 MySQL（Derby 仅单机用），注意 MySQL 8 要加 `allowPublicKeyRetrieval=true`
- 2.x 注意 gRPC 端口：主端口 +1000（如 8848 → 9848），防火墙都要放行
- 节点间时钟同步，避免心跳误判剔除实例

### Q9：为什么客户端和服务端之间要保持长连接？断线了怎么办？

长连接（gRPC）让服务端能**秒级推送**配置变更和实例变更，不用轮询。
断线时客户端自动重连（默认退避策略），重连成功后**全量拉取**一次数据做补偿；同时本地缓存保证服务端完全不可用时仍能用旧配置启动（`failover` 机制，本地有 `failover` 目录可放应急配置）。
