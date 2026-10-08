
---
## 😀 默认助手

 结论：没有一个适用于所有 JDBC 驱动的批量配置。应先确认“数据库类型、实际 JDBC 驱动、驱动版本”，再查该版本的官方连接属性；只有驱动文档明确提供批量改写或批量优化参数时才配置。

1. **先确认运行时实际使用的驱动**

   不要只看数据库名称。MySQL Connector/J、MariaDB Driver、PostgreSQL Driver，即使连接的数据库相近，配置项也可能完全不同。

   ```java
   try (Connection connection = dataSource.getConnection()) {
       DatabaseMetaData metaData = connection.getMetaData();

       System.out.println(metaData.getDatabaseProductName());
       System.out.println(metaData.getDatabaseProductVersion());
       System.out.println(metaData.getDriverName());
       System.out.println(metaData.getDriverVersion());
       System.out.println(metaData.getURL());
   }
   ```

   同时检查项目依赖，例如：

   ```xml
   <dependency>
       <groupId>com.mysql</groupId>
       <artifactId>mysql-connector-j</artifactId>
   </dependency>
   ```

   重点确认实际运行时版本，而不是只看代码中声明的版本。连接池、其他依赖可能带入了不同版本。

2. **查对应驱动版本的官方配置文档**

   在驱动文档中搜索这些关键词：

   ```text
   batch
   executeBatch
   addBatch
   rewrite
   bulk
   prepared statement
   connection properties
   ```

   常见情况如下：

   | 驱动 | 常见批量相关配置 |
   |---|---|
   | MySQL Connector/J | `rewriteBatchedStatements=true` |
   | PostgreSQL JDBC | `reWriteBatchedInserts=true` |
   | Oracle JDBC | 通常依赖 JDBC batch 或数组绑定，没有一个通用的 URL 开关 |
   | SQL Server JDBC | 需要根据具体 Microsoft JDBC Driver 版本查官方文档，不能直接套用 MySQL 参数 |

   例如 MySQL：

   ```properties
   spring.datasource.url=jdbc:mysql://localhost:3306/test?rewriteBatchedStatements=true
   ```

   或：

   ```yaml
   spring:
     datasource:
       hikari:
         data-source-properties:
           rewriteBatchedStatements: true
   ```

   `rewriteBatchedStatements` 是 MySQL Connector/J 的配置，不是 MyBatis 的配置，也不是 JDBC 标准配置。

3. **确认配置是否真的被当前驱动支持**

   配置名称必须同时匹配：

   - JDBC 驱动厂商；
   - 驱动版本；
   - 使用的连接方式；
   - 使用的数据库操作类型。

   例如：

   ```text
   rewriteBatchedStatements=true
   ```

   写在 PostgreSQL 或 Oracle 的连接串里没有意义。某些驱动或连接池可能忽略未知参数，程序未必会直接报错，因此不能仅凭应用正常启动判断配置生效。

4. **通过数据库端行为验证，而不是只看 MyBatis 日志**

   MyBatis 日志显示的是逻辑 SQL 调用，不一定代表驱动实际发出的网络内容。验证时可以：

   - 开关批量配置分别执行相同数据量；
   - 查看数据库通用日志、审计日志或 `performance_schema`；
   - 观察是否出现一条多值 `INSERT`；
   - 比较执行耗时、网络往返次数和数据库 CPU；
   - 使用数据库端执行统计，而不是只依赖 P6Spy 日志。

   例如 MySQL 开启配置后，批量插入更可能被改写为：

   ```sql
   INSERT INTO user(name, age)
   VALUES (?, ?), (?, ?), (?, ?);
   ```

   但具体改写结果仍由 Connector/J 版本和 SQL 类型决定。

5. **区分“必须配置”和“性能优化配置”**

   `ExecutorType.BATCH` 正常工作的最低条件是：

   ```java
   try (SqlSession session =
            sqlSessionFactory.openSession(ExecutorType.BATCH, false)) {

       UserMapper mapper = session.getMapper(UserMapper.class);

       for (User user : users) {
           mapper.insert(user);
       }

       session.flushStatements();
       session.commit();
   }
   ```

   JDBC 驱动即使没有额外批量配置，也可以执行 `addBatch()` 和 `executeBatch()`。驱动参数通常影响的是“是否进一步合并 SQL、减少网络交互、提升性能”，而不是决定 MyBatis batch 是否有效。
