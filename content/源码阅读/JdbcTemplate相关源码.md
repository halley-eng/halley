
# JdbcTemplate 相关源码

### 数据行映射到对象 BeanPropertyRowMapper#mapRow

```java

	/**
	 * Extract the values for all columns in the current row.
	 * <p>Utilizes public setters and result set meta-data.
	 * @see java.sql.ResultSetMetaData
	 */
	@Override
	public T mapRow(ResultSet rs, int rowNumber) throws SQLException {
		// 1. 构建并初始化BeanWrapper
		BeanWrapperImpl bw = new BeanWrapperImpl();
		initBeanWrapper(bw);

		// 2. 构建实例;
		T mappedObject = constructMappedInstance(rs, bw);
		// 3. 封装到 Spring 的BeanWrapper
		bw.setBeanInstance(mappedObject);

		// 4. 循环解析每一列到BeanWrapper;
		ResultSetMetaData rsmd = rs.getMetaData();
		int columnCount = rsmd.getColumnCount();
		Set<String> populatedProperties = (isCheckFullyPopulated() ? new HashSet<>() : null);

		for (int index = 1; index <= columnCount; index++) {
			// 4.1 从 metadata 查列名;
			String column = JdbcUtils.lookupColumnName(rsmd, index);
			// 4.2 转换成字段名称;
			String field = lowerCaseName(StringUtils.delete(column, " "));
			// 4.3 找到对应哪个字段;
			PropertyDescriptor pd = (this.mappedFields != null ? this.mappedFields.get(field) : null);
			if (pd != null) {
				try {
					// 从结果集中解析出字段值;
					Object value = getColumnValue(rs, index, pd);
					if (rowNumber == 0 && logger.isDebugEnabled()) {
						logger.debug("Mapping column '" + column + "' to property '" + pd.getName() +
								"' of type '" + ClassUtils.getQualifiedName(pd.getPropertyType()) + "'");
					}
					// 设置属性值到BeanWrapper
					try {
						bw.setPropertyValue(pd.getName(), value);
					}
					// 类型不匹配 1. 基本类型且允许空值则仅打印日志  2.否则抛出异常
					catch (TypeMismatchException ex) {
						if (value == null && this.primitivesDefaultedForNullValue) {
							if (logger.isDebugEnabled()) {
								logger.debug("Intercepted TypeMismatchException for row " + rowNumber +
										" and column '" + column + "' with null value when setting property '" +
										pd.getName() + "' of type '" +
										ClassUtils.getQualifiedName(pd.getPropertyType()) +
										"' on object: " + mappedObject, ex);
							}
						}
						else {
							throw ex;
						}
					}
					// 聚合已经填充的字段名称;
					if (populatedProperties != null) {
						populatedProperties.add(pd.getName());
					}
				}
				catch (NotWritablePropertyException ex) {
					throw new DataRetrievalFailureException(
							"Unable to map column '" + column + "' to property '" + pd.getName() + "'", ex);
				}
			}
			else {
				// No PropertyDescriptor found
				if (rowNumber == 0 && logger.isDebugEnabled()) {
					logger.debug("No property found for column '" + column + "' mapped to field '" + field + "'");
				}
			}
		}

		if (populatedProperties != null && !populatedProperties.equals(this.mappedProperties)) {
			throw new InvalidDataAccessApiUsageException("Given ResultSet does not contain all fields " +
					"necessary to populate object of " + this.mappedClass + ": " + this.mappedProperties);
		}

		return mappedObject;
	}

```




### 执行纯sql


JdbcTemplate#execute(java.lang.String)


```java
	@Override
	public void execute(final String sql) throws DataAccessException {
		if (logger.isDebugEnabled()) {
			logger.debug("Executing SQL statement [" + sql + "]");
		}

		/**
		 * 1. 回调封装SQL 和 statement的具体执行过程; (参数映射和结果映射)
		 * Callback to execute the statement.
		 */
		class ExecuteStatementCallback implements StatementCallback<Object>, SqlProvider {
			@Override
			@Nullable
			public Object doInStatement(Statement stmt) throws SQLException {
				stmt.execute(sql);
				return null;
			}
			@Override
			public String getSql() {
				return sql;
			}
		}
		// 2. execute 封装获取连接和异常处理
		execute(new ExecuteStatementCallback(), true);
	}
```



### 带结果映射的查询

封装列表处理

```java
	@Override
	public <T> List<T> query(String sql, RowMapper<T> rowMapper) throws DataAccessException {
		// RowMapperResultSetExtractor 执行 列表处理;
		// RowMapper 执行单行映射;
		return result(query(sql, new RowMapperResultSetExtractor<>(rowMapper)));
	}
```

封装结果映射逻辑和实际执行

```JAVA
	@Override
	@Nullable
	public <T> T query(final String sql, final ResultSetExtractor<T> rse) throws DataAccessException {
		Assert.notNull(sql, "SQL must not be null");
		Assert.notNull(rse, "ResultSetExtractor must not be null");
		if (logger.isDebugEnabled()) {
			logger.debug("Executing SQL query [" + sql + "]");
		}
		// 1. 封装sql、执行过程、结果映射
		/**
		 * Callback to execute the query.
		 */
		class QueryStatementCallback implements StatementCallback<T>, SqlProvider {
			@Override
			@Nullable
			public T doInStatement(Statement stmt) throws SQLException {
				ResultSet rs = null;
				try {
					// 执行sql
					rs = stmt.executeQuery(sql);
					// 结果映射
					return rse.extractData(rs);
				}
				finally {
					JdbcUtils.closeResultSet(rs);
				}
			}
			@Override
			public String getSql() {
				return sql;
			}
		}
		// 2. execute 封装获取数据库连接的过程;
		return execute(new QueryStatementCallback(), true);
	}
```



### 带参数映射的查询

```JAVA

	@Override
	@Nullable
	public <T> T queryForObject(String sql, RowMapper<T> rowMapper, @Nullable Object... args) throws DataAccessException {
		List<T> results = query(sql, args, new RowMapperResultSetExtractor<>(rowMapper, 1));
		return DataAccessUtils.nullableSingleResult(results);
	}

```

封装参数

```JAVA
	@Deprecated
	@Override
	@Nullable
	public <T> T query(String sql, @Nullable Object[] args, ResultSetExtractor<T> rse) throws DataAccessException {
		return query(sql, newArgPreparedStatementSetter(args), rse);
	}
    
    protected PreparedStatementSetter newArgPreparedStatementSetter(@Nullable Object[] args) {
		return new ArgumentPreparedStatementSetter(args);
	}

```

由此可见实际参数被封装为

```JAVA

/**
 * 封装 将多个参数设置到  PreparedStatement 的过程
 * Simple adapter for {@link PreparedStatementSetter} that applies a given array of arguments.
 *
 * @author Juergen Hoeller
 * @since 3.2.3
 */
public class ArgumentPreparedStatementSetter implements PreparedStatementSetter, ParameterDisposer {

	@Nullable
	private final Object[] args;


	/**
	 * Create a new ArgPreparedStatementSetter for the given arguments.
	 * @param args the arguments to set
	 */
	public ArgumentPreparedStatementSetter(@Nullable Object[] args) {
		this.args = args;
	}

	/**
	 * 迭代参数并设置值;
	 * @param ps the PreparedStatement to invoke setter methods on
	 * @throws SQLException
	 */
	@Override
	public void setValues(PreparedStatement ps) throws SQLException {
		if (this.args != null) {
			for (int i = 0; i < this.args.length; i++) {
				Object arg = this.args[i];
				doSetValue(ps, i + 1, arg);
			}
		}
	}

	/**
	 * 设置单个参数
	 * Set the value for prepared statements specified parameter index using the passed in value.
	 * This method can be overridden by sub-classes if needed.
	 * @param ps the PreparedStatement
	 * @param parameterPosition index of the parameter position
	 * @param argValue the value to set
	 * @throws SQLException if thrown by PreparedStatement methods
	 */
	protected void doSetValue(PreparedStatement ps, int parameterPosition, Object argValue) throws SQLException {
		if (argValue instanceof SqlParameterValue) {
			SqlParameterValue paramValue = (SqlParameterValue) argValue;
			StatementCreatorUtils.setParameterValue(ps, parameterPosition, paramValue, paramValue.getValue());
		}
		else {
			StatementCreatorUtils.setParameterValue(ps, parameterPosition, SqlTypeValue.TYPE_UNKNOWN, argValue);
		}
	}

	@Override
	public void cleanupParameters() {
		StatementCreatorUtils.cleanupParameters(this.args);
	}

}

```





```JAVA
	@Override
	@Nullable
	public <T> T query(String sql, @Nullable PreparedStatementSetter pss, ResultSetExtractor<T> rse) throws DataAccessException {
        // 封装statement的创建过程
		return query(new SimplePreparedStatementCreator(sql), pss, rse);
	}
```

statement 被 封装为 PreparedStatementCallback 对象, 其包含 参数映射、sql执行、结果解析等功能;

```JAVA
	@Nullable
	public <T> T query(
			PreparedStatementCreator psc, @Nullable final PreparedStatementSetter pss, final ResultSetExtractor<T> rse)
			throws DataAccessException {

		Assert.notNull(rse, "ResultSetExtractor must not be null");
		logger.debug("Executing prepared SQL query");

		return execute(psc, new PreparedStatementCallback<T>() {
			@Override
			@Nullable
			public T doInPreparedStatement(PreparedStatement ps) throws SQLException {
				ResultSet rs = null;
				try {
					// 1. 将参数填充都statement;
					if (pss != null) {
						pss.setValues(ps);
					}
					// 2. 执行sql
					rs = ps.executeQuery();
					// 3. 结果映射;
					return rse.extractData(rs);
				}
				finally {
                   // 4. 关闭statement; 
					JdbcUtils.closeResultSet(rs);
					if (pss instanceof ParameterDisposer) {
						((ParameterDisposer) pss).cleanupParameters();
					}
				}
			}
		}, true);
	}

```

### 批量更新



```java
	@Override
	public int[] batchUpdate(String sql, final BatchPreparedStatementSetter pss) throws DataAccessException {
		if (logger.isDebugEnabled()) {
			logger.debug("Executing SQL batch update [" + sql + "]");
		}

		int[] result = execute(sql, (PreparedStatementCallback<int[]>) ps -> {
			try {
				// 1. 批次大小;
				int batchSize = pss.getBatchSize();
				InterruptibleBatchPreparedStatementSetter ipss =
						(pss instanceof InterruptibleBatchPreparedStatementSetter ?
						(InterruptibleBatchPreparedStatementSetter) pss : null);
				// 2. 如果执行批次更新, 则批量设置值, 并一次更新
				if (JdbcUtils.supportsBatchUpdates(ps.getConnection())) {
					// 2.1  迭代批次
					for (int i = 0; i < batchSize; i++) {
						// 2.1.1 给第i个批次的statement 设置变量;
						pss.setValues(ps, i);
						if (ipss != null && ipss.isBatchExhausted(i)) {
							break;
						}
						// 2.1.2 将该statement加入批次;
						ps.addBatch();
					}
					// 2.2 执行批次
					return ps.executeBatch();
				}
				// 3. 如果连接不支持批量执行, 则每次执行, 并统一返回结果;
				else {
					List<Integer> rowsAffected = new ArrayList<>();
					for (int i = 0; i < batchSize; i++) {
						// 2.1 给该statement设置值;
						pss.setValues(ps, i);
						if (ipss != null && ipss.isBatchExhausted(i)) {
							break;
						}
						// 2.2  执行该statement;
						rowsAffected.add(ps.executeUpdate());
					}
					int[] rowsAffectedArray = new int[rowsAffected.size()];
					for (int i = 0; i < rowsAffectedArray.length; i++) {
						rowsAffectedArray[i] = rowsAffected.get(i);
					}
					return rowsAffectedArray;
				}
			}
			finally {
				if (pss instanceof ParameterDisposer) {
					((ParameterDisposer) pss).cleanupParameters();
				}
			}
		});

		Assert.state(result != null, "No result array");
		return result;
	}
```

demo 如下 JdbcTemplateTests#testBatchUpdateWithPreparedStatement

```java

	@Test
	public void testBatchUpdateWithPreparedStatement() throws Exception {
		final String sql = "UPDATE NOSUCHTABLE SET DATE_DISPATCHED = SYSDATE WHERE ID = ?";
		final int[] ids = new int[] {100, 200};
		final int[] rowsAffected = new int[] {1, 2};

		given(this.preparedStatement.executeBatch()).willReturn(rowsAffected);
		mockDatabaseMetaData(true);

		BatchPreparedStatementSetter setter = new BatchPreparedStatementSetter() {
			@Override
			public void setValues(PreparedStatement ps, int i) throws SQLException {
				ps.setInt(1, ids[i]);
			}
			@Override
			public int getBatchSize() {
				return ids.length;
			}
		};

		JdbcTemplate template = new JdbcTemplate(this.dataSource, false);

		int[] actualRowsAffected = template.batchUpdate(sql, setter);
		assertTrue("executed 2 updates", actualRowsAffected.length == 2);
		assertEquals(rowsAffected[0], actualRowsAffected[0]);
		assertEquals(rowsAffected[1], actualRowsAffected[1]);

		verify(this.preparedStatement, times(2)).addBatch();
		verify(this.preparedStatement).setInt(1, ids[0]);
		verify(this.preparedStatement).setInt(1, ids[1]);
		verify(this.preparedStatement).close();
		verify(this.connection, atLeastOnce()).close();
	}
```


