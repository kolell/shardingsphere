# ShardingSphere 技术架构详细分析

## 架构层次图

```
┌─────────────────────────────────────────────────────────────────┐
│                        应用层 (Application Layer)                  │
├─────────────────────────────────────────────────────────────────┤
│  ShardingSphere-JDBC  │              ShardingSphere-Proxy       │
├─────────────────────────────────────────────────────────────────┤
│                      核心处理层 (Core Layer)                      │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │
│  │ Parsing │ │ Routing │ │Rewriting│ │Executing│ │ Merging │    │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘    │
├─────────────────────────────────────────────────────────────────┤
│                     功能增强层 (Feature Layer)                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │
│  │ Sharding│ │ReadWrite│ │ Encrypt │ │ Shadow  │ │Discovery│    │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘    │
├─────────────────────────────────────────────────────────────────┤
│                    基础设施层 (Infrastructure Layer)               │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │
│  │   SPI   │ │Metadata │ │ Context │ │ Config  │ │Executor │    │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘    │
├─────────────────────────────────────────────────────────────────┤
│                      数据库层 (Database Layer)                    │
│        MySQL    │    PostgreSQL    │    Oracle    │    ...      │
└─────────────────────────────────────────────────────────────────┘
```

## 关键组件详细分析

### 1. SQL处理引擎

#### 解析阶段 (shardingsphere-sql-parser)
```java
// 解析器门面模式实现
public interface DatabaseTypedSQLParserFacade {
    Class<? extends SQLLexer> getLexerClass();
    Class<? extends SQLParser> getParserClass();
    String getDatabaseType();
}
```

**特点**:
- 基于ANTLR4的多方言解析器
- 支持135种语法规则文件
- 统一的AST抽象模型
- 可扩展的方言支持

#### 路由阶段 (shardingsphere-infra-route)
```java
// 路由引擎抽象
public interface SQLRouter<T extends SQLStatement> {
    RouteContext createRouteContext(SQLStatementContext<T> sqlStatementContext, 
                                   List<Object> parameters, 
                                   ShardingSphereSchema schema);
}
```

**路由策略**:
- 标准路由: 基于分片键的精确路由
- 复合路由: 多分片键组合路由
- 提示路由: 强制路由指定
- 广播路由: 广播到所有节点
- 单播路由: 随机选择一个节点

#### 改写阶段 (shardingsphere-infra-rewrite)
```java
// SQL改写上下文
public final class SQLRewriteContext {
    private final ShardingSphereSchema schema;
    private final SQLStatementContext<?> sqlStatementContext;
    private final String sql;
    private final List<Object> parameters;
    
    public void generateSQLTokens() {
        // 生成SQL改写标记
    }
}
```

**改写类型**:
- 分片改写: 表名、分页等
- 加密改写: 敏感数据处理
- 影子改写: 影子表路由
- 优化改写: SQL优化

### 2. 功能特性模块

#### 数据分片算法
```java
// 分片算法接口
public interface ShardingAlgorithm extends TypedSPI {
    Collection<String> doSharding(Collection<String> availableTargetNames, 
                                 ShardingValue shardingValue);
}
```

**内置算法**:
- 精确分片算法: `PreciseShardingAlgorithm`
- 范围分片算法: `RangeShardingAlgorithm` 
- 复合分片算法: `ComplexKeysShardingAlgorithm`
- 提示分片算法: `HintShardingAlgorithm`

#### 读写分离策略
```java
// 负载均衡算法
public interface ReadQueryLoadBalanceAlgorithm extends TypedSPI {
    String getDataSource(String name, String writeDataSourceName, 
                        List<String> readDataSourceNames);
}
```

**负载均衡算法**:
- 轮询: `RoundRobinReplicaLoadBalanceAlgorithm`
- 随机: `RandomReplicaLoadBalanceAlgorithm`
- 权重: `WeightReplicaLoadBalanceAlgorithm`

### 3. 扩展机制

#### SPI机制
```java
// SPI加载器
public final class ShardingSphereServiceLoader {
    public static <T> Collection<T> getSingletonServiceInstances(Class<T> service) {
        return TypedSPIRegistry.getRegisteredServices(service);
    }
}
```

**扩展点类型**:
- 算法扩展: 分片、加密、负载均衡算法
- 规则扩展: 自定义规则类型
- 治理扩展: 配置中心、注册中心
- 存储扩展: 数据库方言、协议

### 4. 治理能力

#### 配置管理
```java
// 配置中心接口
public interface ConfigurationRepository {
    String get(String key);
    void persist(String key, String value);
    void watch(String key, DataChangedEventListener listener);
}
```

**支持的配置中心**:
- ZooKeeper
- etcd  
- Apollo
- Nacos
- Consul

#### 注册中心
```java
// 实例注册
public interface RegistryRepository {
    void persistEphemeral(String key, String value);
    List<String> getChildrenKeys(String key);
    void watch(String key, DataChangedEventListener listener);
}
```

### 5. 事务管理

#### 分布式事务
```java
// 事务管理器
public interface ShardingSphereTransactionManager {
    void begin();
    void commit();
    void rollback();
    TransactionType getTransactionType();
}
```

**事务类型**:
- LOCAL: 本地事务
- XA: XA分布式事务
- BASE: 柔性事务

**事务管理器**:
- Atomikos
- Narayana
- Bitronix
- Seata

## 性能优化机制

### 1. 连接池优化
- 数据源复用
- 连接池隔离
- 懒加载机制

### 2. SQL优化
- 解析结果缓存
- 路由结果缓存
- 执行计划缓存

### 3. 并行执行
- 多数据源并行查询
- 结果集流式归并
- 内存控制机制

### 4. 链路追踪
- OpenTracing集成
- SkyWalking支持
- Jaeger兼容

## 部署架构模式

### 单机模式
```
Application -> ShardingSphere-JDBC -> Multiple Databases
```

### 集群模式  
```
Multiple Applications -> ShardingSphere-Proxy Cluster -> Multiple Databases
                              |
                         Configuration Center
                              |
                         Registry Center
```

### 混合模式
```
Java Applications -> ShardingSphere-JDBC ─┐
                                          ├─> Multiple Databases
Other Applications -> ShardingSphere-Proxy ┘
```

该架构分析展示了ShardingSphere的完整技术栈和设计理念，体现了其作为Database Plus产品的技术深度和广度。