---
title: CREATE INDEX
summary: CREATE INDEX 在 TiDB 中的使用概况
category: reference
---

# CREATE INDEX

`CREATE INDEX` 语句用于在已有表中添加新索引，功能等同于 `ALTER TABLE .. ADD INDEX`。包含该语句提供了 MySQL 兼容性。

## 语法图

**CreateIndexStmt:**

![CreateIndexStmt](/media/sqlgram/CreateIndexStmt.png)

**CreateIndexStmtUnique:**

![CreateIndexStmtUnique](/media/sqlgram/CreateIndexStmtUnique.png)

**Identifier:**

![Identifier](/media/sqlgram/Identifier.png)

**IndexTypeOpt:**

![IndexTypeOpt](/media/sqlgram/IndexTypeOpt.png)

**TableName:**

![TableName](/media/sqlgram/TableName.png)

**IndexColNameList:**

![IndexColNameList](/media/sqlgram/IndexColNameList.png)

**IndexOptionList:**

![IndexOptionList](/media/sqlgram/IndexOptionList.png)

**IndexOption:**

![IndexOption](/media/sqlgram/IndexOption.png)

## 示例

{{< copyable "sql" >}}

```sql
CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, c1 INT NOT NULL);
```

```
Query OK, 0 rows affected (0.10 sec)
```

{{< copyable "sql" >}}

```sql
INSERT INTO t1 (c1) VALUES (1),(2),(3),(4),(5);
```

```
Query OK, 5 rows affected (0.02 sec)
Records: 5  Duplicates: 0  Warnings: 0
```

{{< copyable "sql" >}}

```sql
EXPLAIN SELECT * FROM t1 WHERE c1 = 3;
```

```
+---------------------+----------+------+-------------------------------------------------------------+
| id                  | count    | task | operator info                                               |
+---------------------+----------+------+-------------------------------------------------------------+
| TableReader_7       | 10.00    | root | data:Selection_6                                            |
| └─Selection_6       | 10.00    | cop  | eq(test.t1.c1, 3)                                           |
|   └─TableScan_5     | 10000.00 | cop  | table:t1, range:[-inf,+inf], keep order:false, stats:pseudo |
+---------------------+----------+------+-------------------------------------------------------------+
3 rows in set (0.00 sec)
```

{{< copyable "sql" >}}

```sql
CREATE INDEX c1 ON t1 (c1);
```

```
Query OK, 0 rows affected (0.30 sec)
```

{{< copyable "sql" >}}

```sql
EXPLAIN SELECT * FROM t1 WHERE c1 = 3;
```

```
+-------------------+-------+------+-----------------------------------------------------------------+
| id                | count | task | operator info                                                   |
+-------------------+-------+------+-----------------------------------------------------------------+
| IndexReader_6     | 10.00 | root | index:IndexScan_5                                               |
| └─IndexScan_5     | 10.00 | cop  | table:t1, index:c1, range:[3,3], keep order:false, stats:pseudo |
+-------------------+-------+------+-----------------------------------------------------------------+
2 rows in set (0.00 sec)
```

{{< copyable "sql" >}}

```sql
ALTER TABLE t1 DROP INDEX c1;
```

```
Query OK, 0 rows affected (0.30 sec)
```

{{< copyable "sql" >}}

```sql
CREATE UNIQUE INDEX c1 ON t1 (c1);
```

```
Query OK, 0 rows affected (0.31 sec)
```

## 表达式索引

TiDB 不仅能将索引建立在表中的一个或多个列上，还可以将索引建立在一个表达式上。当查询涉及表达式时，表达式索引能够加速这些查询。

考虑以下查询：

{{< copyable "sql" >}}

```sql
SELECT * FROM t WHERE lower(name) = "pingcap";
```

如果建立了如下的表达式索引，就可以使用索引加速以上查询：

{{< copyable "sql" >}}

```sql
CREATE INDEX idx ON t ((lower(name)));
```

维护表达式索引的代价比一般的索引更高，因为在插入或者更新每一行时都需要计算出表达式的值。因为表达式的值已经存储在索引中，所以当优化器选择表达式索引时，表达式的值就不需要再计算。因此，当查询速度比插入速度和更新速度更重要时，可以考虑建立表达式索引。

表达式索引的语法和限制与 MySQL 相同，是通过将索引建立在隐藏的虚拟生成列 (generated virtual column) 上来实现的。因此所支持的表达式继承了虚拟生成列的所有[限制](/reference/sql/generated-columns.md#局限性)。目前，建立了索引的表达式只有在 `FIELD` 子句、`WHERE` 子句和 `ORDER BY` 子句中时，优化器才能使用表达式索引。后续将支持 `GROUP BY` 子句。

## 相关 session 变量

和 `CREATE INDEX` 语句相关的全局变量有 `tidb_ddl_reorg_worker_cnt`，`tidb_ddl_reorg_batch_size` 和 `tidb_ddl_reorg_priority`，具体可以参考 [TiDB 特定系统变量](/reference/configuration/tidb-server/tidb-specific-variables.md#tidb_ddl_reorg_worker_cnt)。

## MySQL 兼容性

* 不支持 `FULLTEXT`，`HASH` 和 `SPATIAL` 索引。
* 不支持降序索引 （类似于 MySQL 5.7）。
* 默认无法向表中添加 `PRIMARY KEY`，在开启 `alter-primary-key` 配置项后可支持此功能，详情参考：[alter-primary-key](/reference/configuration/tidb-server/configuration-file.md#alter-primary-key)。

## 另请参阅

* [ADD INDEX](/reference/sql/statements/add-index.md)
* [DROP INDEX](/reference/sql/statements/drop-index.md)
* [RENAME INDEX](/reference/sql/statements/rename-index.md)
* [ADD COLUMN](/reference/sql/statements/add-column.md)
* [CREATE TABLE](/reference/sql/statements/create-table.md)
* [EXPLAIN](/reference/sql/statements/explain.md)
