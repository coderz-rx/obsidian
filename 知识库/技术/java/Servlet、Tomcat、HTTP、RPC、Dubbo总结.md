# Servlet / Tomcat / HTTP / RPC / Dubbo 总结

> 本文把通用知识点与本项目（magic_mirror）实际的 Dubbo 配置结合，便于对照理解。
> 相关配置文件：
> - `magic_mirror/src/main/resources/dubbo/dubbo.xml`（总装）
> - `magic_mirror/src/main/resources/dubbo/provider/provider.xml` + `provider/beans.xml`（提供端）
> - `magic_mirror/src/main/resources/dubbo/consumer/consumer.xml`（消费端）

---

## 一、Servlet

Servlet 是 **Java EE 的一套 Web 接口规范**，不是实现类。

- 生命周期：`init()` → `service()` → `destroy()`；**单实例、多线程**，成员变量非线程安全。
- 作用：定义如何接收 HTTP 请求、处理并返回响应。
- Filter、Listener 属于 Servlet 体系的扩展能力。

---

## 二、Tomcat（Servlet 容器）

Tomcat 是 Servlet 规范的实现，本质是**一个 Java 进程（JVM 实例）**。

### 组件层级

```
Server(一个Tomcat进程)
 └─ Service(Connector + Engine)
     └─ Engine
         └─ Host(虚拟主机，区分域名)
             └─ Context(一个Web应用 / war包)
```

### Connector

监听端口，接收 TCP 连接，解析 HTTP；内部 3 类线程：

1. **Acceptor**：接收 TCP 连接
2. **Poller**：NIO 多路复用，监听 IO 可读事件
3. **Worker 线程池**：处理 HTTP 请求，执行 Servlet 逻辑

### 线程池差异

Tomcat 自定义 `TaskQueue`，与 JDK 原生 `ThreadPoolExecutor` 逻辑不同：

- **Tomcat**：先扩容到 `maxThreads`，再入队列。
- **JDK 原生**：先入队列（队列满）再扩容。

> 注意：Tomcat 所有 Worker 线程都是 JVM 内的 Java 线程；一个 JVM 进程可以有大量线程，共享堆、文件句柄等进程资源。

---

## 三、HTTP vs RPC

1. **HTTP**：应用层标准协议，基于 TCP。一般由网关 / 前端调用，走 Tomcat + Servlet 链路，JSON 文本，跨语言友好，报文冗余大。
2. **RPC**：远程过程调用，思想是像调用本地方法一样调用远端服务；**Dubbo 是 Java 生态的 RPC 框架**。

> **区分**：HTTP 可以作为 RPC 的传输载体；但 Dubbo 默认用私有 TCP 协议，**自带 Netty 网络服务，独立端口、独立 IO / 业务线程池，不走 Tomcat**。
>
> 同一应用同时提供 HTTP + Dubbo：两套线程池隔离，HTTP 线程打满不会影响 Dubbo；但 **CPU、堆内存、文件句柄这类进程全局资源共享**，全局资源耗尽两者都会受影响。

**本项目印证**：`provider.xml` 中单独声明了 Dubbo 协议与端口，与 Web 容器端口完全独立：

```xml
<dubbo:protocol id="dubbo" name="dubbo" port="${dubbo.magicMirror.provider.port}"
                threads="${dubbo.magicMirror.provider.threads}"/>
```

`port=20881`、`threads` 为 Dubbo 自己的业务线程池大小——这就是「独立端口、独立线程池」的具体体现。

---

## 四、Dubbo（RPC 框架）

### 五大核心角色

Provider（服务提供者）、Consumer（消费者）、Registry（注册中心）、Monitor（监控）、Container（Spring 容器）。

### 注册 & 服务发现

1. **Provider 启动**：读取配置里注册中心地址，主动建立长连接，上报元数据（接口、IP、端口、版本、分组），维持心跳。
2. **Consumer 启动**：连接注册中心，订阅目标接口，拿到节点列表**本地内存缓存**。
3. **节点上下线**：注册中心推送变更，消费者更新本地缓存。

> **重点**：
> - 注册中心**只存元数据，不转发业务流量**；RPC 请求是 Consumer 直连 Provider。
> - 注册中心宕机：存量 RPC 调用可依靠本地缓存继续执行；只是无法注册新服务、感知节点变更。
> - 部署：生产注册中心独立集群部署；Provider / Consumer 可同机或分开部署。

### 注册中心协议：`zookeeper` vs `availability`（本项目重点）

本项目两种协议混用，需要特别理解：

| 对比项 | `zookeeper` | `availability` |
|---|---|---|
| 来源 | Dubbo 开源原生 | 去哪儿内部扩展（自定义 SPI 实现） |
| `address` | **必须**写（ZK 的 IP:port，常用占位符注入） | **留空**，地址由内部寻址服务自动解析 |
| 寻址方式 | 应用直连指定 ZK 集群 | 内部「寻址中心」按 **环境 + group** 动态返回该连的 ZK |
| 隔离依据 | address + group | **group 为主** |
| 底层存储 | ZooKeeper | 最终仍落到 ZooKeeper |

**`availability` 不写地址，地址从哪来？**

1. 应用启动走到 `availability` SPI 扩展。
2. 扩展不直连 ZK，而是先问内部「注册寻址中心」（入口由框架 / 环境变量注入，业务无需写）。
3. 寻址中心按 **环境（prod / beta / 机房）+ group** 返回真正的 ZK 地址列表。
4. 扩展再用返回的地址连 ZK 做注册 / 订阅。

> 核心价值：**同一份代码配置，换环境部署不用改 ZK 地址**，靠 `group` 做环境隔离，降低连错环境风险。
> 本项目 group 随环境变化：prod 为 `hotelSearch-galaxy-open-prod`，beta/dev 为 `hotelSearch-galaxy-open-noah-${ENV_CODE}`。

**本项目配置对照**：

- 自家统一体系内的服务 → `availability`（地址留空）：
  - provider 注册：`<dubbo:registry id="provider" protocol="availability" group="${dubbo.magicMirror.provider.group}" address=""/>`
  - 消费 sirius：`<dubbo:registry id="sirius" protocol="availability" address="" group="${dubbo.consumer.sirius.group}"/>`
- 跨团队 / 独立部署的外部服务 → `zookeeper`（显式地址）：
  - `qhotel-room-info`、`qhotel-hotel-cluster`、`mars`，均带 `address="${...zk}"`

### 负载均衡策略

1. **random**（默认随机）：最常用，适合节点性能相近场景。
2. **roundRobin 轮询**：按顺序分发，适合节点性能均匀。
3. **leastActive 最少活跃**：优先选请求少的节点，适合节点性能差异大。
4. **consistentHash 一致性哈希**：相同参数路由到同一节点，适合缓存场景。

### 集群容错策略

- **FailOver**（默认）：失败自动重试其他节点，**适合读请求；计费扣费这类写操作禁用，防止重复执行**。
- **FailFast**：失败直接报错，不重试，适合写 / 支付 / 计费等幂等难保证场景。
- **FailSafe**：失败仅打日志，不抛异常，非核心日志 / 埋点场景。
- **FailBack**：后台异步重试，适合消息通知类。

**本项目印证（幂等与重试的刻意设计）**：

- 消费端读类接口用 `cluster="failover" retries="1"`（如 `submitOrderRemoteAsync`、`queryHotelTreeInfoAppService`）。
- 提供端拉黑类写服务（`beans.xml`）**统一 `retries="0"`**——拉黑写操作非幂等，服务端明确关闭重试避免重复写，正好对应「写操作禁用 FailOver」的原则。

### RPC 超时

- **常见原因**：业务阻塞（锁、慢 SQL）、流量突增资源耗尽、网络抖动、下游依赖超时、线程池打满、超时配置过小。
- **排查路径**：监控 → 看 RT / GC / CPU / 线程池 → `jstack` 抓栈 → 查看日志。
- 重试要注意幂等，非幂等写接口禁止重试。

**本项目印证（超时按接口特性分级）**：

| 接口 | 侧 | timeout | 说明 |
|---|---|---|---|
| `BanHotelService` | provider | 100ms | 典型纯缓存 / 内存查询，要快 |
| 多数拉黑写服务 | provider | 10s | 写类操作留足时间 |
| `WrapperOnlineOfflineService` | provider | 11s | 上下线包装操作 |
| `hotelInfoServiceFacade` | consumer | 3s | 查 mars 酒店信息 |
| `submitOrderRemoteAsync` | consumer | 14s | 提交订单异步 |
| `queryHotelTreeInfoAppService` | consumer | 600s | 全量 / 离线大数据量接口，超时给得极宽 |

---

## 五、本项目 Dubbo 配置组装关系

```
dubbo.xml
├── provider/provider.xml   → 注册中心(availability) + dubbo协议/端口/线程 + 全局默认(filter/executes/retries)
│     └── import beans.xml   → 对外暴露的 8 个拉黑领域服务(retries=0, timeout 按需)
└── consumer/consumer.xml    → 4 个注册中心(availability + zookeeper 混用) + 4 个外部服务引用
```

- **Provider 侧**：对外提供「拉黑相关能力」（Ban* 系列服务），全局挂 `filter="qaccesslogprovider"` 做访问日志。
- **Consumer 侧**：依赖「下单 / 房型异常 / 酒店树 / mars 酒店信息」等外部服务。
- 两端共用同一个 `dubbo.xml` 组装的 Spring / Dubbo 上下文。

通用参数约定：

- `check="false"`：启动时不校验提供方是否存在，避免下游未就绪导致本应用无法启动。
- `version="1.0"`：接口版本，配合 group 做隔离与灰度。

---

## 六、选型总结

- **HTTP**：对外、前端 / 第三方调用、跨语言，通用性强。
- **Dubbo**：内网同语言微服务、高 QPS 低 RT、需要完善微服务治理（注册发现、负载均衡、容错、灰度）。
