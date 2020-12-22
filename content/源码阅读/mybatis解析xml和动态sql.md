---
title: "mybatis解析xml和动态sql"
date: 2020-12-22T21:39:16+08:00
draft: false
tags: ["mybatis"]
---
#### mybatis 如何解析xml文件

support.DaoSupport#afterPropertiesSet
MapperFactoryBean#checkDaoConfig
总之， MapperFactoryBean该工厂bean初始化后就会注册到 MapperRegistry
org.apache.ibatis.binding.MapperRegistry#addMapper
在触发解析xml
MapperAnnotationBuilder#parse
MapperAnnotationBuilder#loadXmlResource
MapperBuilderAssistant#addMappedStatement

### 动态sql解析

DynamicSqlSource#getBoundSql

根据参数解析动态sql生成 BoundSql

```JAVA
  public BoundSql getBoundSql(Object parameterObject) {
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings == null || parameterMappings.isEmpty()) {
      boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    // check for nested result maps in parameter mappings (issue #30)
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
      String rmId = pm.getResultMapId();
      if (rmId != null) {
        ResultMap rm = configuration.getResultMap(rmId);
        if (rm != null) {
          hasNestedResultMaps |= rm.hasNestedResultMaps();
        }
      }
    }

    return boundSql;
  }
```


参数映射 

SqlSourceBuilder#parse
```JAVA

  public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    // 
    String sql = parser.parse(originalSql);
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }

```
GenericTokenParser#parse


解析器源码如下
核心逻辑为迭代解析每一个表达式，并使用org.apache.ibatis.parsing.TokenHandler#handleToken 解析出表达式的值;

```JAVA
public class GenericTokenParser {

  private final String openToken;
  private final String closeToken;
  private final TokenHandler handler;

  public GenericTokenParser(String openToken, String closeToken, TokenHandler handler) {
    this.openToken = openToken;
    this.closeToken = closeToken;
    this.handler = handler;
  }

  public String parse(String text) {
    if (text == null || text.isEmpty()) {
      return "";
    }
    // search open token
    int start = text.indexOf(openToken, 0);
    if (start == -1) {
      return text;
    }
    char[] src = text.toCharArray();
    int offset = 0;
    final StringBuilder builder = new StringBuilder();
    StringBuilder expression = null;
    // 循环迭代从offset开始解析每一个表达式
    while (start > -1) {

      // 1. 找到了起始词汇但是其前缀为转译符号: 跳过解析该词汇
      if (start > 0 && src[start - 1] == '\\') {
        // this open token is escaped. remove the backslash and continue.
        builder.append(src, offset, start - offset - 1).append(openToken);
        offset = start + openToken.length();
      // 2. 解析动态变量
      } else {
        // found open token. let's search close token.
        if (expression == null) {
          expression = new StringBuilder();
        } else {
          expression.setLength(0);
        }
        // 2.1 追加变量前缀部分
        builder.append(src, offset, start - offset);
        // 2.2 跳过变量起始符号比如 ${
        offset = start + openToken.length();
        // 2.3 从offset开始定位结尾词 }
        int end = text.indexOf(closeToken, offset);
        // 2.4 生成表达式
        while (end > -1) {
          // 跳过带转译符号的结尾字符
          if (end > offset && src[end - 1] == '\\') {
            // 追加表达式
            // this close token is escaped. remove the backslash and continue.
            expression.append(src, offset, end - offset - 1).append(closeToken);
            // 跳过
            offset = end + closeToken.length();
            end = text.indexOf(closeToken, offset);
          // 找到表达式
          } else {
            expression.append(src, offset, end - offset);
            offset = end + closeToken.length();
            break;
          }
        }
        // 2.5 无法生成表达式 直接追加
        if (end == -1) {
          // close token was not found.
          builder.append(src, start, src.length - start);
          offset = src.length;
        // 2.6 生成表达式, 交给handler 处理;
        } else {
          builder.append(handler.handleToken(expression.toString()));
          // 更新当前解析到的位置
          offset = end + closeToken.length();
        }
      }
      // 3 迭代处理下一个表达式;
      start = text.indexOf(openToken, offset);
    }

    // 最后发现还有没有解析完的字符 直接追加到结果;
    if (offset < src.length) {
      builder.append(src, offset, src.length - offset);
    }
    return builder.toString();
  }
}

```

TokenHandler 有如下实现

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086504698645.jpg)

1. PropertyParser.VariableTokenHandler 
    从 Properties 解析变量值, 并支持默认值;
2. ParameterMappingTokenHandler
   1. 将 每个表达式通过 handleToken 转换为占位符 ？
   2. 缓存 ParameterMapping
   3. StaticSqlSource
3. 动态sql 
    1. DynamicSqlSource
    




### 动态sql


```java
public class DynamicSqlSource implements SqlSource {

  private final Configuration configuration;
  private final SqlNode rootSqlNode;

  public DynamicSqlSource(Configuration configuration, SqlNode rootSqlNode) {
    this.configuration = configuration;
    this.rootSqlNode = rootSqlNode;
  }

  @Override
  public BoundSql getBoundSql(Object parameterObject) {
    // 1. 解析上下文;
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    // 2. 执行动态sql, 并将结果输出到context
    rootSqlNode.apply(context);
    // 3. 将动态sql输出为sql模版
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    // 4. 去除变量生成占位符;
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    // 4. 生成参数绑定后的sql
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
      boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
  }

}

```

其中 SqlNode 的实现有： 

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086504847702.jpg)

### 循环节点实现 ForEachSqlNode

主要是通过apply函数执行

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086504956449.jpg)

demo

```java
  @Test
  public void shouldIterateOnceForEachItemInCollection() throws Exception {
    // 1. 参数
    final HashMap<String, String[]> parameterObject = new HashMap<String, String[]>() {{
      put("array", new String[]{"one", "two", "three"});
    }};
    final String expected = "SELECT * FROM BLOG WHERE ID in (  one = ? AND two = ? AND three = ? )";
    // 2. 创建动态sql
    DynamicSqlSource source = createDynamicSqlSource(
        new TextSqlNode("SELECT * FROM BLOG WHERE ID in"),
        new ForEachSqlNode(new Configuration(),mixedContents(new TextSqlNode("${item} = #{item}")), "array", "index", "item", "(", ")", "AND"));
        
    // 3. 生成绑定后的sql
    BoundSql boundSql = source.getBoundSql(parameterObject);
    assertEquals(expected, boundSql.getSql());
    assertEquals(3, boundSql.getParameterMappings().size());
    assertEquals("__frch_item_0", boundSql.getParameterMappings().get(0).getProperty());
    assertEquals("__frch_item_1", boundSql.getParameterMappings().get(1).getProperty());
    assertEquals("__frch_item_2", boundSql.getParameterMappings().get(2).getProperty());
  }

```

创建动态sql的函数
```java
  private DynamicSqlSource createDynamicSqlSource(SqlNode... contents) throws IOException, SQLException {
    createBlogDataSource();
    final String resource = "org/apache/ibatis/builder/MapperConfig.xml";
    final Reader reader = Resources.getResourceAsReader(resource);
    SqlSessionFactory sqlMapper = new SqlSessionFactoryBuilder().build(reader);
    Configuration configuration = sqlMapper.getConfiguration();
    MixedSqlNode sqlNode = mixedContents(contents);
    return new DynamicSqlSource(configuration, sqlNode);
  }

```


那么如上<3>获取 SqlSource#getBoundSql
实际调用如下:

org.apache.ibatis.scripting.xmltags.DynamicSqlSource#getBoundSql


```java
public class DynamicSqlSource implements SqlSource {

  private final Configuration configuration;
  private final SqlNode rootSqlNode;

  public DynamicSqlSource(Configuration configuration, SqlNode rootSqlNode) {
    this.configuration = configuration;
    this.rootSqlNode = rootSqlNode;
  }

  @Override
  public BoundSql getBoundSql(Object parameterObject) {
    // 1. 解析上下文;
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    // 2. 执行动态sql
    rootSqlNode.apply(context);
    // 3. 转化为静态sql
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    // 4. 获取映射处理后的sql
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    // 5. 生成参数绑定后的sql
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    // 5. 设置其他变量
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
      boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
  }

}

```
如上
1. 创建了 DynamicContext 
    1. 所有子节点会在自己apply函数执行后面, 通过调用其 append 方法追加解析后的内容:
    
    ```java
     public void appendSql(String sql) {
            sqlBuilder.append(sql);
            sqlBuilder.append(" ");
     }
    ```
    
2. apply 阶段会通过如下 MixedSqlNode 触发每个节点apply

org.apache.ibatis.scripting.xmltags.MixedSqlNode#apply

```java
public class MixedSqlNode implements SqlNode {
  private final List<SqlNode> contents;

  public MixedSqlNode(List<SqlNode> contents) {
    this.contents = contents;
  }

  @Override
  public boolean apply(DynamicContext context) {
    // 迭代每一个node 输出到context
    for (SqlNode sqlNode : contents) {
      sqlNode.apply(context);
    }
    return true;
  }
}

```

其中 ForEachSqlNode#apply 最为复杂

```java
  /**
   * 执行动态 sql 并生成到 contents
   * @param context
   * @return
   */
  @Override
  public boolean apply(DynamicContext context) {
    Map<String, Object> bindings = context.getBindings();
    // 1. 在对象 bindings 上执行表达式  collectionExpression 并获取后者的迭代器;
    final Iterable<?> iterable = evaluator.evaluateIterable(collectionExpression, bindings);
    // 2. 直接迭代完成;
    if (!iterable.iterator().hasNext()) {
      return true;
    }
    boolean first = true;
    // 2. 输出起始词;
    applyOpen(context);
    int i = 0;
    // 3. 执行迭代;
    for (Object o : iterable) {

      DynamicContext oldContext = context;
      // 3.1 前缀处理, 封装context, 覆写appendSql方法
      // 第一个或者是没有前缀, 直接输出;
      if (first || separator == null) {
        context = new PrefixedContext(context, "");
      // 输出前缀
      } else {
        context = new PrefixedContext(context, separator);
      }
      // 3.2 建索引;
      int uniqueNumber = context.getUniqueNumber();
      // Issue #709
      if (o instanceof Map.Entry) {
        @SuppressWarnings("unchecked") 
        Map.Entry<Object, Object> mapEntry = (Map.Entry<Object, Object>) o;
        applyIndex(context, mapEntry.getKey(), uniqueNumber);
        applyItem(context, mapEntry.getValue(), uniqueNumber);
      } else {
        applyIndex(context, i, uniqueNumber);
        applyItem(context, o, uniqueNumber);
      }
      /**
       * 3.3 变量替换 封装 appendSql
       *
       *  apply 后的回溯过程
       *  {@link TextSqlNode#apply(DynamicContext)}
       *     {@link FilteredDynamicContext#appendSql(java.lang.String)}
       *        {@link PrefixedContext#appendSql(String)}
       *          {@link DynamicContext#appendSql(java.lang.String)}
       *   修改并输出当前item的变量表达式, 到父容器;
       */
      contents.apply(new FilteredDynamicContext(configuration, context, index, item, uniqueNumber));
      if (first) {
        first = !((PrefixedContext) context).isPrefixApplied();
      }
      context = oldContext;
      // 位移
      i++;
    }
    // 4. 追加结束词
    applyClose(context);
    context.getBindings().remove(item);
    context.getBindings().remove(index);
    return true;
  }

```


注意上面使用 PrefixedContext、FilteredDynamicContext 两次封装实现前缀和列表变量绑定功能, 它们都是通过 覆写 appendSql方法实现子问题的解append到父问题;


FilteredDynamicContext#appendSql
```java

    @Override
    public void appendSql(String sql) {
      // 构建该sql的解析器, 可以修改循环里面的变量名称
      GenericTokenParser parser = new GenericTokenParser("#{", "}", new TokenHandler() {
        @Override
        public String handleToken(String content) {
          // 给原始变量加上索引标示;
          String newContent = content.replaceFirst("^\\s*" + item + "(?![^.,:\\s])", itemizeItem(item, index));
          if (itemIndex != null && newContent.equals(content)) {
            newContent = content.replaceFirst("^\\s*" + itemIndex + "(?![^.,:\\s])", itemizeItem(itemIndex, index));
          }
          // 输出替换后的变量
          return new StringBuilder("#{").append(newContent).append("}").toString();
        }
      });

      // 回溯: 输出上有context
      delegate.appendSql(parser.parse(sql));
    }
```

ForEachSqlNode.PrefixedContext#appendSql

```java
    @Override
    public void appendSql(String sql) {
      if (!prefixApplied && sql != null && sql.trim().length() > 0) {
        delegate.appendSql(prefix);
        prefixApplied = true;
      }
      delegate.appendSql(sql);
    }

```


### 总结

综上分析了: 
1. mybatis 解析xml文件的时机和位置
1. Mybatis的解析器
2. 动态sql如何使用解析器
    1. 组合模式 MixedSqlNode
    2. 迭代器 ForEachSqlNode
        1. 通过代理context实现PrefixedContext、FilteredDynamicContext实现了前缀和列表变量绑定的
        
        

