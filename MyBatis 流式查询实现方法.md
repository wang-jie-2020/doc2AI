
---
## 😀 默认助手

MyBatis 原生没有“游标”查询 API。大结果集通常用 `Cursor<T>` 流式读取，避免一次把所有结果装入内存：

```java
@Mapper
public interface UserMapper {
    @Select("SELECT id, name FROM user")
    Cursor<User> scanUsers();
}
```

```java
@Transactional(readOnly = true)
public void processUsers() {
    try (Cursor<User> cursor = userMapper.scanUsers()) {
        for (User user : cursor) {
            process(user);
        }
    }
}
```

注意：

- 必须在数据库连接和事务仍然打开时遍历，并及时关闭 `Cursor`；不要在事务结束后返回它。
- JDBC 驱动可能需要额外设置才能真正流式读取。例如 MySQL 通常要给连接加 `useCursorFetch=true`，并设置正数 `fetchSize`。
- 大批量处理也可以用 `ResultHandler<T>`；它逐条回调，但同样要注意事务、连接和驱动的 fetch size 配置。
- 如果你指的是存储过程的数据库游标，配置方式不同；需说明数据库类型和 MyBatis 版本。


---
## 😀 默认助手

MyBatis 的 `Cursor<T>` 和 `ResultHandler<T>` 适合“结果很多、逐条处理”的查询；普通分页查询更适合 `LIMIT/OFFSET` 或基于游标的分页。下面是常见场景。

**1. 导出大量数据**

例如导出几百万条订单，逐条写入 CSV，避免把所有订单一次性加载到 `List`：

```java
@Mapper
public interface OrderMapper {
    @Select("SELECT id, created_at, amount FROM orders ORDER BY id")
    @Options(fetchSize = 500)
    Cursor<OrderRow> scanOrders();
}
```

```java
@Transactional(readOnly = true)
public void exportOrders(Writer writer) throws IOException {
    try (Cursor<OrderRow> rows = orderMapper.scanOrders()) {
        for (OrderRow row : rows) {
            writeCsvLine(writer, row);
        }
    }
}
```

适用于报表导出、数据备份、生成离线文件。文件写入和查询都可能很慢，因此应控制连接占用时间。

**2. 批量迁移或修复数据**

例如逐条读取旧表记录，转换后写入新表。查询保持流式，写入按批次提交：

```java
@Transactional(readOnly = true)
public void migrate() {
    List<NewRecord> batch = new ArrayList<>(500);

    try (Cursor<OldRecord> rows = oldMapper.scanAll()) {
        for (OldRecord row : rows) {
            batch.add(convert(row));

            if (batch.size() == 500) {
                newMapper.insertBatch(batch);
                batch.clear();
            }
        }

        if (!batch.isEmpty()) {
            newMapper.insertBatch(batch);
        }
    }
}
```

实际项目中，查询事务和写入事务往往需要分开设计，避免长事务；还要考虑失败后从哪个位置继续处理。

**3. 大数据集逐条计算**

例如为历史订单计算统计值或生成搜索索引：

```java
try (Cursor<OrderRow> rows = orderMapper.scanOrders()) {
    for (OrderRow row : rows) {
        searchIndexer.index(row);
    }
}
```

适用于无需保留全量结果、每条记录可以独立处理的任务。若处理速度很慢，长时间占用数据库连接可能成为瓶颈，可改为分批分页读取。

**4. `ResultHandler`：边查边回调**

如果不需要在业务代码里持有 `Cursor`，可以让 MyBatis 每取到一条记录就调用处理器：

```java
mapper.scanOrders(resultContext -> {
    OrderRow row = resultContext.getResultObject();
    process(row);
});
```

适用于轻量逐条处理。它同样需要查询期间保持连接可用；不要在回调中执行可能导致长时间阻塞的操作。

**选择建议：**
- 全量遍历、需要清晰控制关闭时机：用 `Cursor<T>`。
- MyBatis 查询回调即可完成处理：用 `ResultHandler<T>`。
- 任务需要断点续跑、重试或不宜长时间占用连接：按主键分批查询，例如 `WHERE id > #{lastId} ORDER BY id LIMIT #{size}`。
- 驱动配置很重要：例如 MySQL 通常要启用 `useCursorFetch=true`，并设置正数 `fetchSize`；具体行为取决于 JDBC 驱动和数据库。
