假设有如下所示的 PostgreSQL 表和数据

```sql
CREATE TABLE foo
(
    id   INTEGER,
    name VARCHAR
);

CREATE TABLE bar
(
    id     INTEGER,
    name   VARCHAR,
    foo_id INTEGER
);

INSERT INTO foo (id, name) VALUES (1, 'foo1');
INSERT INTO foo (id, name) VALUES (2, 'foo2');

INSERT INTO bar (id, name, foo_id) VALUES (1, 'bar1', 1);
INSERT INTO bar (id, name, foo_id) VALUES (2, 'bar2', 1);
INSERT INTO bar (id, name, foo_id) VALUES (3, 'bar3', 2);
```

具体的数据为

```
// foo 表
+--+----+
|id|name|
+--+----+
|1 |foo1|
|2 |foo2|
+--+----+

// bar 表
+--+----+------+
|id|name|foo_id|
+--+----+------+
|1 |bar1|1     |
|2 |bar2|1     |
|3 |bar3|2     |
+--+----+------+
```

现在的问题是下面的 `UPDATE ... SET ... FROM ... WHERE ...` 语句会使用 `bar` 的哪一行数据更新 `foo` 的数据

```sql
UPDATE foo AS f
SET name = b.name
FROM bar AS b
WHERE f.id = b.foo_id;
```

先来看看更新后 `foo` 的数据，看起来 `foo` 的数据都使用 `f.id = b.foo_id` 时最小 `bar.id` 的数据来更新。

```
+--+----+
|id|name|
+--+----+
|1 |bar1|
|2 |bar3|
+--+----+
```

再来看看[官方文档](https://www.postgresql.org/docs/18/sql-update.html)是怎么描述的

> When a `FROM` clause is present, what essentially happens is that the target table is joined to the tables mentioned in the ***`from_item`*** list, and each output row of the join represents an update operation for the target table. When using `FROM` you should ensure that the join produces at most one output row for each row to be modified. In other words, a target row shouldn't join to more than one row from the other table(s). If it does, then only one of the join rows will be used to update the target row, but which one will be used is not readily predictable.

官方文档提到了 *joined to* 但是没有明确说是那种类型的 *join*，但是在 [7.2.1. The FROM Clause](https://www.postgresql.org/docs/18/queries-table-expressions.html#QUERIES-FROM) 说明有效的 *join* 类型

```
T1 { [INNER] | { LEFT | RIGHT | FULL } [OUTER] } JOIN T2 ON boolean_expression
T1 { [INNER] | { LEFT | RIGHT | FULL } [OUTER] } JOIN T2 USING ( join column list )
T1 NATURAL { [INNER] | { LEFT | RIGHT | FULL } [OUTER] } JOIN T2
```

时提到 `INNER JOIN` 是 *join* 的默认类型

> The words `INNER` and `OUTER` are optional in all forms. `INNER` is the default; `LEFT`, `RIGHT`, and `FULL` imply an outer join.

因此 `foo` 表和 `bar` 表连接后

```sql
select f.id as "f.id", f.name as "f.name", b.id as "b.id", b.name as "b.name", b.foo_id as "b.foo_id"
from foo as f inner join bar as b on f.id = b.foo_id;
```

的结果为

```
+----+------+----+------+--------+
|f.id|f.name|b.id|b.name|b.foo_id|
+----+------+----+------+--------+
|1   |foo1  |1   |bar1  |1       |
|1   |foo1  |2   |bar2  |1       |
|2   |foo2  |3   |bar3  |2       |
+----+------+----+------+--------+
```

但是具体使用 `bar` 的哪一行数据进行更新是难以预测的，因此在使用 `FROM` 时应保证连接后最多为待修改行生成一个输出行，比如在这个例子中可能需要删掉 *bar.foo_id = 1* 的某一行来满足这个条件。
