---
title: "shardingspare源码"
date: 2020-12-22T21:39:16+08:00
draft: false
---
# shardingsphere源码解析

# 使用方法

导入包

```java
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core</artifactId>
    <version>${shardingsphere.version}</version>
</dependency>
```


配置路由规则 比如两库两表


```java

// Configure actual data sources
Map<String, DataSource> dataSourceMap = new HashMap<>();

// Configure the first data source
BasicDataSource dataSource1 = new BasicDataSource();
dataSource1.setDriverClassName("com.mysql.jdbc.Driver");
dataSource1.setUrl("jdbc:mysql://localhost:3306/ds0");
dataSource1.setUsername("root");
dataSource1.setPassword("");
dataSourceMap.put("ds0", dataSource1);

// Configure the second data source
BasicDataSource dataSource2 = new BasicDataSource();
dataSource2.setDriverClassName("com.mysql.jdbc.Driver");
dataSource2.setUrl("jdbc:mysql://localhost:3306/ds1");
dataSource2.setUsername("root");
dataSource2.setPassword("");
dataSourceMap.put("ds1", dataSource2);

// Configure order table rule
ShardingTableRuleConfiguration orderTableRuleConfig = new ShardingTableRuleConfiguration("t_order", "ds${0..1}.t_order${0..1}");

// Configure database sharding strategy
orderTableRuleConfig.setDatabaseShardingStrategy(new StandardShardingStrategyConfiguration("user_id", "dbShardingAlgorithm"));

// Configure table sharding strategy
orderTableRuleConfig.setTableShardingStrategy(new StandardShardingStrategyConfiguration("order_id", "tableShardingAlgorithm"));

// Omit t_order_item table rule configuration ...
// ...
    
// Configure sharding rule
ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
shardingRuleConfig.getTables().add(orderTableRuleConfig);

// Configure database sharding algorithm
Properties dbShardingAlgorithmrProps = new Properties();
dbShardingAlgorithmrProps.setProperty("algorithm-expression", "ds${user_id % 2}");
shardingRuleConfig.getShardingAlgorithms().put("dbShardingAlgorithm", new ShardingSphereAlgorithmConfiguration("INLINE", dbShardingAlgorithmrProps));

// Configure table sharding algorithm
Properties tableShardingAlgorithmrProps = new Properties();
tableShardingAlgorithmrProps.setProperty("algorithm-expression", "t_order${order_id % 2}");
shardingRuleConfig.getShardingAlgorithms().put("tableShardingAlgorithm", new ShardingSphereAlgorithmConfiguration("INLINE", tableShardingAlgorithmrProps));

// Create ShardingSphereDataSource
DataSource dataSource = ShardingSphereDataSourceFactory.createDataSource(dataSourceMap, Collections.singleton(shardingRuleConfig), new Properties());
```

使用 ShardingSphereDataSource

ShardingSphereDataSourceFactory 负责 创建 ShardingSphereDataSource , 后者实现了标准的JDBC DataSource 接口. 
可以做到不影响 本地 JDBC和ORM框架的使用 比如JPA或者是Mybatis

```java
DataSource dataSource = ShardingSphereDataSourceFactory.createDataSource(dataSourceMap, Collections.singleton(shardingRuleConfig), new Properties());
String sql = "SELECT i.* FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id WHERE o.user_id=? AND o.order_id=?";
try (
        Connection conn = dataSource.getConnection();
        PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setInt(1, 10);
    ps.setInt(2, 1000);
    try (ResultSet rs = ps.executeQuery()) {
        while(rs.next()) {
            // ...
        }
    }
}
```

# 源码解析

核心模块如下
![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086501659080.jpg)

## 支持分片路由的数据源ShardingDataSource

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086501734429.jpg)

DataSource 很简单只有 getConnection 相关的两个方法

所以核心路由算法根据下面的实现可知主要在 ShardingConnection 

```java
  @Override
    public final ShardingConnection getConnection() {
        return new ShardingConnection(getDataSourceMap(), shardingContext, getShardingTransactionManagerEngine(), TransactionTypeHolder.get());
    }
```

而连接后创建支持路由的Statement

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086501858229.jpg)

比如 ShardingPreparedStatement 就是支持路由的 PreparedStatement

对于该 statement 的所有set 方法都会在 AbstractShardingPreparedStatementAdapter 进行缓存

```java
    
    // 缓存参数设置;
    private void setParameter(final int parameterIndex, final Object value) {
        if (parameters.size() == parameterIndex - 1) {
            parameters.add(value);
            return;
        }
        for (int i = parameters.size(); i <= parameterIndex - 1; i++) {
            parameters.add(null);
        }
        parameters.set(parameterIndex - 1, value);
    }
    // 重放参数设置;
    protected final void replaySetParameter(final PreparedStatement preparedStatement, final List<Object> parameters) {
        setParameterMethodInvocations.clear();
        addParameters(parameters);
        for (SetParameterMethodInvocation each : setParameterMethodInvocations) {
            each.invoke(preparedStatement);
        }
    }
   
```

在执行阶段在进行路由、设置变量、结果合并

ShardingPreparedStatement#executeQuery
```java
    // 返回ResultSet的查询操作 
    @Override
    public ResultSet executeQuery() throws SQLException {
        ResultSet result;
        try {
            clearPrevious();
            //1. 分片路由
            shard();
            //2. 初始化线程池 重放statement
            initPreparedStatementExecutor();
            //3. 合并结果;
            MergeEngine mergeEngine = MergeEngineFactory.newInstance(connection.getShardingContext().getDatabaseType(), 
                    connection.getShardingContext().getShardingRule(), routeResult, connection.getShardingContext().getMetaData().getTable(), preparedStatementExecutor.executeQuery());
            //4.合并结果集
            result = getResultSet(mergeEngine);
        } finally {
            clearBatch();
        }
        currentResultSet = result;
        return result;
    }
    // 任何 DML: INSERT, UPDATE or DELET 或者 没有返回的 DDL
     @Override
    public int executeUpdate() throws SQLException {
        try {
            clearPrevious();
            shard();
            initPreparedStatementExecutor();
            return preparedStatementExecutor.executeUpdate();
        } finally {
            clearBatch();
        }
    } 
```

### 分片路由

ShardingPreparedStatement#shard
```java
   private void shard() {
        routeResult = shardingEngine.shard(sql, getParameters());
    }   
```

路由算法实现

```java
@RequiredArgsConstructor
public abstract class BaseShardingEngine {

    private final ShardingRule shardingRule;

    private final ShardingProperties shardingProperties;

    private final ShardingMetaData metaData;

    private final SPIRoutingHook routingHook = new SPIRoutingHook();

    /**
     * Shard.
     *
     * @param sql SQL
     * @param parameters parameters of SQL
     * @return SQL route result
     */
    public SQLRouteResult shard(final String sql, final List<Object> parameters) {
        // 1. 拷贝参数
        List<Object> clonedParameters = cloneParameters(parameters);
        // 2. 执行路由
        SQLRouteResult result = executeRoute(sql, clonedParameters);
        // 3. 根据路由
        result.getRouteUnits().addAll(HintManager.isDatabaseShardingOnly() ?
            convert(sql, clonedParameters, result) :  // 只根据数据库路由, 给每个数据源关联sql和参数
            rewriteAndConvert(clonedParameters, result)  // 其他路由.
        );
        // 4. 如果配置了打印sql, 则打印之;
        if (shardingProperties.getValue(ShardingPropertiesConstant.SQL_SHOW)) {
            boolean showSimple = shardingProperties.getValue(ShardingPropertiesConstant.SQL_SIMPLE);
            SQLLogger.logSQL(sql, showSimple, result.getSqlStatement(), result.getRouteUnits());
        }
        return result;
    }

    protected abstract List<Object> cloneParameters(List<Object> parameters);

    /**
     *  对于{@link PreparedQueryShardingEngine} 路由到 {@link org.apache.shardingsphere.core.route.PreparedStatementRoutingEngine}
     */
    protected abstract SQLRouteResult route(String sql, List<Object> parameters);
    
    // 上面第2步 将会执行该函数
    private SQLRouteResult executeRoute(final String sql, final List<Object> clonedParameters) {
        // 1. 开始路由钩子
        routingHook.start(sql);
        try {
            // 2. 执行路由
            SQLRouteResult result = route(sql, clonedParameters);
            // 3. 路由成功钩子
            routingHook.finishSuccess(result, metaData.getTable());
            return result;
            // CHECKSTYLE:OFF
        } catch (final Exception ex) {
            // 路由失败钩子;
            // CHECKSTYLE:ON
            routingHook.finishFailure(ex);
            throw ex;
        }
    }

    /**
     * 只根据数据库路由, 给每个数据源关联sql和参数
     */
    private Collection<RouteUnit> convert(final String sql, final List<Object> parameters, final SQLRouteResult sqlRouteResult) {
        Collection<RouteUnit> result = new LinkedHashSet<>();
        // 对于每个数据源, 在关联本次的sql和参数
        for (RoutingUnit each : sqlRouteResult.getRoutingResult().getRoutingUnits()) {
            result.add(new RouteUnit(each.getDataSourceName(), new SQLUnit(sql, parameters)));
        }
        return result;
    }
    // 重写数据库和表
    private Collection<RouteUnit> rewriteAndConvert(final List<Object> parameters, final SQLRouteResult sqlRouteResult) {
        // 1. 重写引擎
        SQLRewriteEngine rewriteEngine = new SQLRewriteEngine(shardingRule, sqlRouteResult.getSqlStatement(), parameters, sqlRouteResult.getRoutingResult().isSingleRouting());
        // 2. 路由参数重写器
        ShardingParameterRewriter shardingParameterRewriter = new ShardingParameterRewriter(sqlRouteResult);
        // 3. 重写器;
        Collection<SQLRewriter> sqlRewriters = new LinkedList<>();
        // 3.1 路由重写器
        sqlRewriters.add(new ShardingSQLRewriter(shardingRule, sqlRouteResult, sqlRouteResult.getOptimizeResult()));
        // 3.2 加密重写器
        if (sqlRouteResult.getSqlStatement() instanceof DMLStatement) {
            sqlRewriters.add(new EncryptSQLRewriter(shardingRule.getEncryptRule().getEncryptorEngine(), (DMLStatement) sqlRouteResult.getSqlStatement(), sqlRouteResult.getOptimizeResult()));
        }
        // 4. 初始化重写引擎
        rewriteEngine.init(Collections.<ParameterRewriter>singletonList(shardingParameterRewriter), sqlRewriters);
        Collection<RouteUnit> result = new LinkedHashSet<>();
        // 5. 对于之前每个路由结果, 关联数据源和重写后的sql
        for (RoutingUnit each : sqlRouteResult.getRoutingResult().getRoutingUnits()) {
            result.add(new RouteUnit(each.getDataSourceName(), rewriteEngine.generateSQL(each)));
        }
        return result;
    }
}

```

如上
1. 执行路由算得到路由结果 RouteResult
2. 使用重写引擎重写sql


#### 执行路由算法 

对于 PreparedQueryShardingEngine 路由算法 实现为PreparedStatementRoutingEngine

```java
public final class PreparedQueryShardingEngine extends BaseShardingEngine {
    
    private final PreparedStatementRoutingEngine routingEngine;
    
    public PreparedQueryShardingEngine(final String sql, final ShardingRule shardingRule, final ShardingProperties shardingProperties,
                                       final ShardingMetaData metaData, final DatabaseType databaseType, final ParsingResultCache cache) {
        super(shardingRule, shardingProperties, metaData);
        routingEngine = new PreparedStatementRoutingEngine(sql, shardingRule, metaData, databaseType, cache);
    }
    
    @Override
    protected List<Object> cloneParameters(final List<Object> parameters) {
        return new ArrayList<>(parameters);
    }
    
    @Override
    protected SQLRouteResult route(final String sql, final List<Object> parameters) {
        // 路由引擎执行;
        return routingEngine.route(parameters);
    }
}

```

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086502194429.jpg)

```java

public final class PreparedStatementRoutingEngine {
    
    private final String logicSQL;
    private final ShardingRouter shardingRouter;
    private final ShardingMasterSlaveRouter masterSlaveRouter;
    private SQLStatement sqlStatement;
    
    public PreparedStatementRoutingEngine(final String logicSQL, final ShardingRule shardingRule,
                                          final ShardingMetaData shardingMetaData, final DatabaseType databaseType, final ParsingResultCache parsingResultCache) {
        this.logicSQL = logicSQL;
        // 1. 仅数据库路由 ? DatabaseHintSQLRouter : ParsingSQLRouter;
        shardingRouter = ShardingRouterFactory.newInstance(shardingRule, shardingMetaData, databaseType, parsingResultCache);
        // 2. 主从路由
        masterSlaveRouter = new ShardingMasterSlaveRouter(shardingRule.getMasterSlaveRules());
    }
    
    public SQLRouteResult route(final List<Object> parameters) {
        // 1. 解析sql;
        if (null == sqlStatement) {
            sqlStatement = shardingRouter.parse(logicSQL, true);
        }
        // 2. 路由
        return masterSlaveRouter.route(shardingRouter.route(sqlStatement, parameters));
    }
 }   
```

PreparedStatementRoutingEngine 将会
1. 通过 ShardingRouter 解析 statement 
2. 再通过 ShardingMasterSlaveRouter 路由


##### PreparedStatementRoutingEngine 解析出 statement


首先会调用 ParsingSQLRouter#parse

```java
    public SQLStatement parse(final String logicSQL, final boolean useCache) {
        parsingHook.start(logicSQL);
        try {
            // 分片路由解析器
            SQLStatement result = new ShardingSQLParseEntry(databaseType, shardingRule, shardingMetaData.getTable(), parsingResultCache).parse(logicSQL, useCache);
            parsingHook.finishSuccess(result, shardingMetaData.getTable());
            return result;
            // CHECKSTYLE:OFF
        } catch (final Exception ex) {
            // CHECKSTYLE:ON
            parsingHook.finishFailure(ex);
            throw ex;
        }
    }
```

ShardingSQLParseEntry 的基类实现 SQLParseEntry#parse , 支持缓存

```java
    public final SQLStatement parse(final String sql, final boolean useCache) {
        Optional<SQLStatement> cachedSQLStatement = getSQLStatementFromCache(sql, useCache);
        if (cachedSQLStatement.isPresent()) {
            return cachedSQLStatement.get();
        }
        // sql 解析引擎去解析
        SQLStatement result = getSQLParseEngine(sql).parse();
        if (useCache) {
            parsingResultCache.put(sql, result);
        }
        return result;
    }
```
实际的解析引擎

```java
    @Override
    protected SQLParseEngine getSQLParseEngine(final String sql) {
        return new SQLParseEngine(ShardingParseRuleRegistry.getInstance(), databaseType, sql, shardingRule, shardingTableMetaData);
    }
```

解析引擎实现

```java
public final class SQLParseEngine {
    
    private final SQLParserEngine parserEngine;
    
    private final SQLSegmentsExtractorEngine extractorEngine;
    
    private final SQLStatementFillerEngine fillerEngine;
    
    private final SQLStatementOptimizerEngine optimizerEngine;
    
    public SQLParseEngine(final ParseRuleRegistry parseRuleRegistry, final DatabaseType databaseType, final String sql, final BaseRule rule, final ShardingTableMetaData shardingTableMetaData) {
        DatabaseType trunkDatabaseType = DatabaseTypes.getTrunkDatabaseType(databaseType.getName());
        parserEngine = new SQLParserEngine(parseRuleRegistry, trunkDatabaseType, sql);
        extractorEngine = new SQLSegmentsExtractorEngine();
        fillerEngine = new SQLStatementFillerEngine(parseRuleRegistry, trunkDatabaseType, sql, rule, shardingTableMetaData);
        optimizerEngine = new SQLStatementOptimizerEngine(rule, shardingTableMetaData);
    }
    
    /**
     * Parse SQL.
     *
     * @return SQL statement
     */
    public SQLStatement parse() {
        // 1. 对sql解析得到语法树
        SQLAST ast = parserEngine.parse();
        // 2. 参数索引;
        Map<ParserRuleContext, Integer> parameterMarkerIndexes = ast.getParameterMarkerIndexes(); //
        // 3. 解析出每个片段;
        Collection<SQLSegment> sqlSegments = extractorEngine.extract(ast, parameterMarkerIndexes);
        // 4. 每个片段进行填充后得到该 SQLStatement
        SQLStatement result = fillerEngine.fill(sqlSegments, parameterMarkerIndexes.size(), ast.getSqlStatementRule());
        // 5. 优化该 statement
        optimizerEngine.optimize(ast.getSqlStatementRule(), result);
        return result;
    }
}

```


第三步填充算法如下:

```java
public final class SQLStatementFillerEngine {
    
    private final ParseRuleRegistry parseRuleRegistry;
    
    private final DatabaseType databaseType;
    
    private final String sql;
    
    private final BaseRule rule;
    
    private final ShardingTableMetaData shardingTableMetaData;
    
    /**
     * Fill SQL statement.
     *
     * @param sqlSegments SQL segments
     * @param parameterMarkerCount parameter marker count
     * @param rule SQL statement rule
     * @return SQL statement
     */
    @SneakyThrows
    public SQLStatement fill(final Collection<SQLSegment> sqlSegments, final int parameterMarkerCount, final SQLStatementRule rule) {
        // 1. 根据规则拿到最终聚合的 SQLStatement
        SQLStatement result = rule.getSqlStatementClass().newInstance();
        result.setLogicSQL(sql);
        result.setParametersCount(parameterMarkerCount);
        result.getSQLSegments().addAll(sqlSegments);
        // 2. 迭代填充每一个 segment, 去填充到 statement(result)
        for (SQLSegment each : sqlSegments) {
            // 2.1 每个片段找填充器;
            Optional<SQLSegmentFiller> filler = parseRuleRegistry.findSQLSegmentFiller(databaseType, each.getClass());
            // 2.2 填充该statement;
            if (filler.isPresent()) {
                // 2.2.1
                doFill(each, result, filler.get());
            }
        }
        return result;
    }

    /**
     *  根据填充器和规则类型 设置 填充器;
     */
    @SuppressWarnings("unchecked")
    private void doFill(final SQLSegment sqlSegment, final SQLStatement sqlStatement, final SQLSegmentFiller filler) {
        // 1.  根据填充器和规则类型 设置 填充器
        // 1.1 路由规则
        if (filler instanceof ShardingRuleAware && rule instanceof ShardingRule) {
            ((ShardingRuleAware) filler).setShardingRule((ShardingRule) rule);
        }
        // 1.2 加密规则;
        if (filler instanceof EncryptRuleAware && rule instanceof EncryptRule) {
            ((EncryptRuleAware) filler).setEncryptRule((EncryptRule) rule);
        }
        // 1.3 路由表元数据;
        if (filler instanceof ShardingTableMetaDataAware) {
            ((ShardingTableMetaDataAware) filler).setShardingTableMetaData(shardingTableMetaData);
        }
        // 2. 让填充器去填充 statement
        filler.fill(sqlSegment, sqlStatement);
    }
}

```

填充器有如下实现: 
![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086502519213.jpg)
![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086502572751.jpg)

##### PreparedStatementRoutingEngine 根据SQLStatement, 用 ParsingSQLRouter 路由

对于解析出来的 statemet 
在进行路由 ParsingSQLRouter#route

```java
    @Override
    public RoutingResult route() {
        if (isDMLForModify(sqlStatement) && !sqlStatement.getTables().isSingleTable()) {
            throw new SQLParsingException("Cannot support Multiple-Table for '%s'.", sqlStatement);
        }
        return generateRoutingResult(  // 3. 封装路由结果;
                    getDataNodes(      // 2. 执行路由, 拿到数字节点;
                        shardingRule.getTableRule(logicTableName)   // 1. 根据逻辑表名拿到 TableRule
                    )
        );
    }
```


ShardingRule#getTableRule 获取可用表名 

```java
    public TableRule getTableRule(final String logicTableName) {
        // 根据逻辑表名查询到 TableRule
        Optional<TableRule> tableRule = findTableRule(logicTableName);
        if (tableRule.isPresent()) {
            return tableRule.get();
        }
        // 广播表 绑定所有数据源
        if (isBroadcastTable(logicTableName)) {
            return new TableRule(shardingDataSourceNames.getDataSourceNames(), logicTableName);
        }
        //
        if (!Strings.isNullOrEmpty(shardingDataSourceNames.getDefaultDataSourceName())) {
            return new TableRule(shardingDataSourceNames.getDefaultDataSourceName(), logicTableName);
        }
        throw new ShardingConfigurationException("Cannot find table rule and default data source with logic table: '%s'", logicTableName);
    }
    
```

路由到DataNode

```java
    private Collection<DataNode> getDataNodes(final TableRule tableRule) {
        if (shardingRule.isRoutingByHint(tableRule)) {
            return routeByHint(tableRule);
        }
        // 条件路由
        if (isRoutingByShardingConditions(tableRule)) {
            return routeByShardingConditions(tableRule);
        }
        return routeByMixedConditions(tableRule);
    }
```

条件路由详情: 

```java
    private Collection<DataNode> routeByShardingConditions(final TableRule tableRule) {
        return optimizeResult.getRouteConditions().getRouteConditions().isEmpty() ?
                route(tableRule, Collections.<RouteValue>emptyList(), Collections.<RouteValue>emptyList())
                : routeByShardingConditionsWithCondition(tableRule);
    }
    
    private Collection<DataNode> routeByShardingConditionsWithCondition(final TableRule tableRule) {
        Collection<DataNode> result = new LinkedList<>();
        for (RouteCondition each : optimizeResult.getRouteConditions().getRouteConditions()) {
            Collection<DataNode> dataNodes = route(tableRule,
                    // 根据数据库规则中的列 能够匹配 rule里面的哪些 RouteValue
                    getShardingValuesFromShardingConditions(shardingRule.getDatabaseShardingStrategy(tableRule).getShardingColumns(), each),
                    // 根据表结构规则中的列 能够匹配 rule里面的哪些 RouteValue
                    getShardingValuesFromShardingConditions(shardingRule.getTableShardingStrategy(tableRule).getShardingColumns(), each));
            reviseInsertOptimizeResult(each, dataNodes);
            result.addAll(dataNodes);
        }
        return result;
    }
    
```

应用路由算法
StandardRoutingEngine#getShardingValuesFromShardingConditions

```java
    
    private List<RouteValue> getShardingValuesFromShardingConditions(final Collection<String> shardingColumns, final RouteCondition routeCondition) {
        List<RouteValue> result = new ArrayList<>(shardingColumns.size());
        // 迭代每个规则中的路由变量, 并匹配路由列;
        for (RouteValue each : routeCondition.getRouteValues()) {
            // 逻辑表对应的物理表;
            Optional<BindingTableRule> bindingTableRule = shardingRule.findBindingTableRule(logicTableName);
            // 物理表包含该逻辑表名;
            if ((logicTableName.equals(each.getTableName()) || // 逻辑表就是物理表
                    // 任意一个物理表的逻辑表对得上 并且 分片列对得上;
                    bindingTableRule.isPresent() && bindingTableRule.get().hasLogicTable(logicTableName)) && shardingColumns.contains(each.getColumnName())) {
                result.add(each);
            }
        }
        return result;
    }
    
```
#### 重写sql


对于如下demo
```java
   
    @Test
    public void assertRewriteForTableName() {
        List<Object> parameters = new ArrayList<>(2);
        parameters.add(1);
        parameters.add("x");
        // 待重写的分词
        selectStatement.getSQLSegments().add(new TableSegment(7, 13, "table_x"));
        selectStatement.getSQLSegments().add(new TableSegment(31, 37, "table_x"));
        selectStatement.getSQLSegments().add(new TableSegment(47, 53, "table_x"));
        // 路由结果重写参数, parameters
        routeResult = new SQLRouteResult(selectStatement);
        routeResult.setRoutingResult(new RoutingResult());
        routeResult.setOptimizeResult(new OptimizeResult(new RouteConditions(Collections.<RouteCondition>emptyList())));
        // 原始sql;
        selectStatement.setLogicSQL("SELECT table_x.id, x.name FROM table_x x WHERE table_x.id=? AND x.name=?");
        // 创建引擎;
        SQLRewriteEngine rewriteEngine = createSQLRewriteEngine(parameters);
        assertThat(getSQLBuilder(rewriteEngine).toSQL(null, tableTokens), is("SELECT table_1.id, x.name FROM table_1 x WHERE table_1.id=? AND x.name=?"));
    }
    
```
创建引擎 

```java
 
    private SQLRewriteEngine createSQLRewriteEngine(final List<Object> parameters) {
        // 1. 重写引擎 引用到 shardingRule 的重写规则， 根据 statement 中的 segments 生成 tokens
        SQLRewriteEngine result = new SQLRewriteEngine(shardingRule, routeResult.getSqlStatement(), parameters, routeResult.getRoutingResult().isSingleRouting());
        Collection<SQLRewriter> sqlRewriters = new LinkedList<>();
        // 2. 路由重写器;
        sqlRewriters.add(new ShardingSQLRewriter(shardingRule, routeResult, routeResult.getOptimizeResult()));
        // 3. 加密重写器
        if (routeResult.getSqlStatement() instanceof DMLStatement) {
            sqlRewriters.add(new EncryptSQLRewriter(shardingRule.getEncryptRule().getEncryptorEngine(), (DMLStatement) routeResult.getSqlStatement(), routeResult.getOptimizeResult()));
        }

        // 4. 根据路由规则重写参数和tokens
        result.init(Collections.<ParameterRewriter>singletonList(new ShardingParameterRewriter(routeResult)), sqlRewriters);
        // 返回结果
        return result;
    }
}
```

SQLRewriteEngine#init 实现将参数重写器和sql重写器应用，并写入sqlBuilder,最后可以通过 SQLBuilder#toSQL 输出

```java
    /**
     * Initialize SQL rewrite engine.
     * 
     * @param parameterRewriters parameter rewriters
     * @param sqlRewriters SQL rewriters
     */
    public void init(final Collection<ParameterRewriter> parameterRewriters, final Collection<SQLRewriter> sqlRewriters) {
        //  重写参数
        for (ParameterRewriter each : parameterRewriters) {
            each.rewrite(parameterBuilder);
        }
        if (sqlTokens.isEmpty()) {
            baseSQLRewriter.appendWholeSQL(sqlBuilder);
            return;
        }
        //
        baseSQLRewriter.appendInitialLiteral(sqlBuilder);
        // 迭代所有的tokens
        for (SQLToken each : sqlTokens) {
            // 交给每个重写器重写, 结果会通过 sqlBuilder 输出;
            for (SQLRewriter sqlRewriter : sqlRewriters) {
                sqlRewriter.rewrite(sqlBuilder, parameterBuilder, each);
            }
            // 生成列名
            baseSQLRewriter.rewrite(sqlBuilder, parameterBuilder, each);
        }
    }
    
```


SQLBuilder#toSQL 实现如下:

```java
    public String toSQL(final RoutingUnit routingUnit, final Map<String, String> logicAndActualTables) {
        StringBuilder result = new StringBuilder();
        for (Object each : segments) {
            // 重写表名
            if (each instanceof Alterable) {
                result.append(((Alterable) each).toString(routingUnit, logicAndActualTables));
            } else {
                result.append(each);
            }
        }
        return result.toString();
    }
```
# 相关概念 


## BindingTableRule

代表相关表具有相同的分片规则, 相关表join分配到单库而不用笛卡尔积

[standard-route](https://shardingsphere.apache.org/document/legacy/4.x/document/en/features/sharding/principle/route/#standard-route)

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086502722203.jpg)

# 相关连接

[shardingsphere java-api](https://shardingsphere.apache.org/document/current/en/user-manual/shardingsphere-jdbc/usage/sharding/java-api/)


[shardingsphere 4.x](https://shardingsphere.apache.org/document/legacy/4.x/document/en/overview/)

[shardingsphere 5.x](https://shardingsphere.apache.org/document/current/en/overview/)
