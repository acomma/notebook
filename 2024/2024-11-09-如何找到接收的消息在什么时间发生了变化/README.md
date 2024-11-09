## 问题描述

我们有一个系统在不停接收其他系统发来的消息，当前时刻发送的消息和前一时刻发送消息可能相同也可能不同。我们的任务是在消息发生变化时把这些消息查询出来，但是有一个要求，突然的变化不算变化，即消息的内容需要连续出现两次才是有效的消息，举个例子，假设依次收到的消息是 AAB 则 B 是无效的消息，此时消息未发生变化；假设依次收到的消息是 AABB 则消息发生了变化，从 A 变成了 B；假设依次收到的消息是 AABCC 则 B 是无效的消息，但是消息发生了变化，从 A 变成了 C 而不是从 A 变成了 B 再从 B 变成了 C；假设依次的收到的消息是 AABAACC 则 B 是无效的消失，但是消息发生了变化，从 A 变成了 C 而不是从 A 变成 B 再变成 A 再变成了 C。

## 问题分析

下面我们创建一张消息表 `message` 来模拟这种情况

```sql
CREATE TABLE message (
    id SERIAL4,
    sender VARCHAR,
    content VARCHAR,
    create_at TIMESTAMPTZ,
    PRIMARY KEY (id)
);

INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:11');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:12');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'B', '2024-11-09 11:11:13');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'B', '2024-11-09 11:11:14');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'B', '2024-11-09 11:11:15');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'C', '2024-11-09 11:11:16');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:17');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:18');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'A', '2024-11-09 11:11:19');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'D', '2024-11-09 11:11:20');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'D', '2024-11-09 11:11:21');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'D', '2024-11-09 11:11:22');
INSERT INTO message(sender, content, create_at) VALUES ('Bob', 'E', '2024-11-09 11:11:23');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'A', '2024-11-09 11:11:24');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'A', '2024-11-09 11:11:25');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'B', '2024-11-09 11:11:26');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'A', '2024-11-09 11:11:27');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'A', '2024-11-09 11:11:28');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'C', '2024-11-09 11:11:29');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'C', '2024-11-09 11:11:30');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'D', '2024-11-09 11:11:31');
INSERT INTO message(sender, content, create_at) VALUES ('Alice', 'D', '2024-11-09 11:11:32');
```

解决问题的核心是找到一种方法能够返回连续出现两次及以上的记录；然后在这个新的结果集上比较每一条消息和前一条消息的内容是否一致，如果不一致则发生了变化。

## 方法一

对于任意一条记录来说，它的内容连续出现两次及以上的条件是它的内容要么和前一条记录的内容一样，要么和后一条记录的内容一样。在 PostgreSQL 中可以通过 `LAG` 函数获取前一条记录的内容，通过 `LEAD` 函数获取后一条记录的内容。

```sql
SELECT *,
       LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content,
       LEAD(content) OVER(PARTITION BY sender ORDER BY create_at) AS next_content
FROM message
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+------------+------------+
|id|sender|content|create_at          |prev_content|next_content|
+--+------+-------+-------------------+------------+------------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |A           |
|2 |Bob   |A      |2024-11-09 11:11:12|A           |B           |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |B           |
|4 |Bob   |B      |2024-11-09 11:11:14|B           |B           |
|5 |Bob   |B      |2024-11-09 11:11:15|B           |C           |
|6 |Bob   |C      |2024-11-09 11:11:16|B           |A           |
|7 |Bob   |A      |2024-11-09 11:11:17|C           |A           |
|8 |Bob   |A      |2024-11-09 11:11:18|A           |A           |
|9 |Bob   |A      |2024-11-09 11:11:19|A           |D           |
|10|Bob   |D      |2024-11-09 11:11:20|A           |D           |
|11|Bob   |D      |2024-11-09 11:11:21|D           |D           |
|12|Bob   |D      |2024-11-09 11:11:22|D           |E           |
|13|Bob   |E      |2024-11-09 11:11:23|D           |null        |
|14|Alice |A      |2024-11-09 11:11:24|null        |A           |
|15|Alice |A      |2024-11-09 11:11:25|A           |B           |
|16|Alice |B      |2024-11-09 11:11:26|A           |A           |
|17|Alice |A      |2024-11-09 11:11:27|B           |A           |
|18|Alice |A      |2024-11-09 11:11:28|A           |C           |
|19|Alice |C      |2024-11-09 11:11:29|A           |C           |
|20|Alice |C      |2024-11-09 11:11:30|C           |D           |
|21|Alice |D      |2024-11-09 11:11:31|C           |D           |
|22|Alice |D      |2024-11-09 11:11:32|D           |null        |
+--+------+-------+-------------------+------------+------------+
```

我们只需要找到 `content` 要么等于 `prev_content` 要么等于 `next_content` 的记录，这些记录就是连续出现两次及以上的记录

```sql
SELECT id, sender, content, create_at
FROM (
    SELECT *,
           LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content,
           LEAD(content) OVER(PARTITION BY sender ORDER BY create_at) AS next_content
    FROM message
) AS t
WHERE content = prev_content OR content = next_content
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+
|id|sender|content|create_at          |
+--+------+-------+-------------------+
|1 |Bob   |A      |2024-11-09 11:11:11|
|2 |Bob   |A      |2024-11-09 11:11:12|
|3 |Bob   |B      |2024-11-09 11:11:13|
|4 |Bob   |B      |2024-11-09 11:11:14|
|5 |Bob   |B      |2024-11-09 11:11:15|
|7 |Bob   |A      |2024-11-09 11:11:17|
|8 |Bob   |A      |2024-11-09 11:11:18|
|9 |Bob   |A      |2024-11-09 11:11:19|
|10|Bob   |D      |2024-11-09 11:11:20|
|11|Bob   |D      |2024-11-09 11:11:21|
|12|Bob   |D      |2024-11-09 11:11:22|
|14|Alice |A      |2024-11-09 11:11:24|
|15|Alice |A      |2024-11-09 11:11:25|
|17|Alice |A      |2024-11-09 11:11:27|
|18|Alice |A      |2024-11-09 11:11:28|
|19|Alice |C      |2024-11-09 11:11:29|
|20|Alice |C      |2024-11-09 11:11:30|
|21|Alice |D      |2024-11-09 11:11:31|
|22|Alice |D      |2024-11-09 11:11:32|
+--+------+-------+-------------------+
```

在这个结果的基础上比较每条记录的内容是否和前一条记录的内容一致，不一致则发生了变化。这里分两步来进行计算，首先找到每条记录的前一条记录

```sql
SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
FROM (
    SELECT id, sender, content, create_at
    FROM (
        SELECT *,
               LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content,
               LEAD(content) OVER(PARTITION BY sender ORDER BY create_at) AS next_content
        FROM message
    ) AS t
    WHERE content = prev_content OR content = next_content
) AS consecutive
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+------------+
|id|sender|content|create_at          |prev_content|
+--+------+-------+-------------------+------------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |
|2 |Bob   |A      |2024-11-09 11:11:12|A           |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |
|4 |Bob   |B      |2024-11-09 11:11:14|B           |
|5 |Bob   |B      |2024-11-09 11:11:15|B           |
|7 |Bob   |A      |2024-11-09 11:11:17|B           |
|8 |Bob   |A      |2024-11-09 11:11:18|A           |
|9 |Bob   |A      |2024-11-09 11:11:19|A           |
|10|Bob   |D      |2024-11-09 11:11:20|A           |
|11|Bob   |D      |2024-11-09 11:11:21|D           |
|12|Bob   |D      |2024-11-09 11:11:22|D           |
|14|Alice |A      |2024-11-09 11:11:24|null        |
|15|Alice |A      |2024-11-09 11:11:25|A           |
|17|Alice |A      |2024-11-09 11:11:27|A           |
|18|Alice |A      |2024-11-09 11:11:28|A           |
|19|Alice |C      |2024-11-09 11:11:29|A           |
|20|Alice |C      |2024-11-09 11:11:30|C           |
|21|Alice |D      |2024-11-09 11:11:31|C           |
|22|Alice |D      |2024-11-09 11:11:32|D           |
+--+------+-------+-------------------+------------+
```

然后再比较当前记录的内容是否和前一条记录的内容一样，如果不一样则是发生了变化

```sql
SELECT *
FROM (
    SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
    FROM (
        SELECT id, sender, content, create_at
        FROM (
            SELECT *,
                   LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content,
                   LEAD(content) OVER(PARTITION BY sender ORDER BY create_at) AS next_content
            FROM message
        ) AS t
        WHERE content = prev_content OR content = next_content
    ) AS consecutive
) AS preved
WHERE prev_content IS NULL OR content != prev_content
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+------------+
|id|sender|content|create_at          |prev_content|
+--+------+-------+-------------------+------------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |
|7 |Bob   |A      |2024-11-09 11:11:17|B           |
|10|Bob   |D      |2024-11-09 11:11:20|A           |
|14|Alice |A      |2024-11-09 11:11:24|null        |
|19|Alice |C      |2024-11-09 11:11:29|A           |
|21|Alice |D      |2024-11-09 11:11:31|C           |
+--+------+-------+-------------------+------------+
```

为了更清晰地表达我们的意图和逻辑下面使用 `WITH` 改写上面的嵌套查询

```sql
WITH consecutive AS (
    SELECT id, sender, content, create_at
    FROM (
        SELECT *,
               LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content,
               LEAD(content) OVER(PARTITION BY sender ORDER BY create_at) AS next_content
        FROM message
    ) AS t
    WHERE content = prev_content OR content = next_content
),
preved AS (
    SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
    FROM consecutive
),
changed AS (
    SELECT *
    FROM preved
    WHERE prev_content IS NULL OR content != prev_content
)
SELECT * FROM changed ORDER BY id;
```

### 连续出现三次及以上的记录

对于任意一条记录来说，它的内容连续出现三次及以上的条件是它的内容要么和前两条记录的内容一样，要么和后两条记录的内容一样，要么和前一条和后一条的内容都一样。`LAG` 函数的完整定义为 `lag(value anyelement [, offset integer [, default anyelement ]])`，它在分区内当前行的之前 `offset` 个位置的行上计算，`offset` 的默认值为 1。可以将 `offset` 的值设为 2 从而得到在当前记录前面第二条的记录。`LEAD` 函数是类似的。

```sql
SELECT id, sender, content, create_at
FROM (
    SELECT *,
           LAG(content, 1) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content_1,  -- 当前行前面第一行的内容
           LAG(content, 2) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content_2,  -- 当前行前面第二行的内容
           LEAD(content, 1) OVER(PARTITION BY sender ORDER BY create_at) AS next_content_1, -- 当前行后面第一行的内容
           LEAD(content, 2) OVER(PARTITION BY sender ORDER BY create_at) AS next_content_2  -- 当前行后面第二行的内容
    FROM message
) AS t
WHERE (content = prev_content_1 AND content = prev_content_2) -- 当前行内容与它前面两行内容一样
   OR (content = next_content_1 AND content = next_content_2) -- 当前行内容与它后面两行内容一样
   OR (content = prev_content_1 AND content = next_content_1) -- 当前行内容与它前面一行和后面一行内容一样
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+
|id|sender|content|create_at          |
+--+------+-------+-------------------+
|3 |Bob   |B      |2024-11-09 11:11:13|
|4 |Bob   |B      |2024-11-09 11:11:14|
|5 |Bob   |B      |2024-11-09 11:11:15|
|7 |Bob   |A      |2024-11-09 11:11:17|
|8 |Bob   |A      |2024-11-09 11:11:18|
|9 |Bob   |A      |2024-11-09 11:11:19|
|10|Bob   |D      |2024-11-09 11:11:20|
|11|Bob   |D      |2024-11-09 11:11:21|
|12|Bob   |D      |2024-11-09 11:11:22|
+--+------+-------+-------------------+
```

## 方法二

方法一如果需要连续出现更多次的记录则需要更多的 `LAG` 和 `LEAD` 函数，如果把连续出现的次数作为一个参数则需要动态的拼接查询语句。下面来实现一种不需要动态拼接查询语句即可实现连续出现任意次数的需求。

为了比较前后两条消息是否有变化，需要使用 `LAG` 函数获取当前消息的前一条消息

```sql
SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
FROM message
ORDER BY id;
```

接下来是判断当前消息相对于前一条消息是否有变化，需要用到 `CASE ... WHEN ... THEN ... ELSE ... END` 表达式，0 表示没有变化，1 表示有变化

```sql
SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
FROM (
    SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
    FROM message
) AS preved
ORDER BY id;
```

如果只是把有变化的查询出来只需要在上面的查询的基础上添加 `is_change = 1` 这个条件即可，但是我们的要求是连续两次变化才算变化。为了实现这个需求，一种直观的方式是根据消息内容分组，统计每个分组内的数量，排除数量为 1 的分组。但是这种做法无法排出交叉地出现的 1 内容，比如 AABCCBCC，此时 B 无法被排除。因此需要定义新的分组 ID，根据新的分组 ID 重新分组。我们可以利用已经计算出来的 `is_change` 字段，在它之上进行求和来得到新的分组 ID

```sql
SELECT *, SUM(is_change) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
FROM (
    SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
    FROM (
        SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
        FROM message
    ) AS preved
) AS content_change
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+------------+---------+--------+
|id|sender|content|create_at          |prev_content|is_change|group_id|
+--+------+-------+-------------------+------------+---------+--------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |1        |1       |
|2 |Bob   |A      |2024-11-09 11:11:12|A           |0        |1       |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |1        |2       |
|4 |Bob   |B      |2024-11-09 11:11:14|B           |0        |2       |
|5 |Bob   |B      |2024-11-09 11:11:15|B           |0        |2       |
|6 |Bob   |C      |2024-11-09 11:11:16|B           |1        |3       |
|7 |Bob   |A      |2024-11-09 11:11:17|C           |1        |4       |
|8 |Bob   |A      |2024-11-09 11:11:18|A           |0        |4       |
|9 |Bob   |A      |2024-11-09 11:11:19|A           |0        |4       |
|10|Bob   |D      |2024-11-09 11:11:20|A           |1        |5       |
|11|Bob   |D      |2024-11-09 11:11:21|D           |0        |5       |
|12|Bob   |D      |2024-11-09 11:11:22|D           |0        |5       |
|13|Bob   |E      |2024-11-09 11:11:23|D           |1        |6       |
|14|Alice |A      |2024-11-09 11:11:24|null        |1        |1       |
|15|Alice |A      |2024-11-09 11:11:25|A           |0        |1       |
|16|Alice |B      |2024-11-09 11:11:26|A           |1        |2       |
|17|Alice |A      |2024-11-09 11:11:27|B           |1        |3       |
|18|Alice |A      |2024-11-09 11:11:28|A           |0        |3       |
|19|Alice |C      |2024-11-09 11:11:29|A           |1        |4       |
|20|Alice |C      |2024-11-09 11:11:30|C           |0        |4       |
|21|Alice |D      |2024-11-09 11:11:31|C           |1        |5       |
|22|Alice |D      |2024-11-09 11:11:32|D           |0        |5       |
+--+------+-------+-------------------+------------+---------+--------+
```

接下来使用新的分组 ID 统计每个分组的数量

```sql
SELECT *, COUNT(*) OVER(PARTITION BY sender, group_id) AS group_count
FROM (
    SELECT *, SUM(is_change) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
    FROM (
        SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
        FROM (
            SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
            FROM message
        ) AS preved
    ) AS content_change
) AS grouped
ORDER BY id;
```

这条语句执行的结果如下表所示

```text
+--+------+-------+-------------------+------------+---------+--------+-----------+
|id|sender|content|create_at          |prev_content|is_change|group_id|group_count|
+--+------+-------+-------------------+------------+---------+--------+-----------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |1        |1       |2          |
|2 |Bob   |A      |2024-11-09 11:11:12|A           |0        |1       |2          |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |1        |2       |3          |
|4 |Bob   |B      |2024-11-09 11:11:14|B           |0        |2       |3          |
|5 |Bob   |B      |2024-11-09 11:11:15|B           |0        |2       |3          |
|6 |Bob   |C      |2024-11-09 11:11:16|B           |1        |3       |1          |
|7 |Bob   |A      |2024-11-09 11:11:17|C           |1        |4       |3          |
|8 |Bob   |A      |2024-11-09 11:11:18|A           |0        |4       |3          |
|9 |Bob   |A      |2024-11-09 11:11:19|A           |0        |4       |3          |
|10|Bob   |D      |2024-11-09 11:11:20|A           |1        |5       |3          |
|11|Bob   |D      |2024-11-09 11:11:21|D           |0        |5       |3          |
|12|Bob   |D      |2024-11-09 11:11:22|D           |0        |5       |3          |
|13|Bob   |E      |2024-11-09 11:11:23|D           |1        |6       |1          |
|14|Alice |A      |2024-11-09 11:11:24|null        |1        |1       |2          |
|15|Alice |A      |2024-11-09 11:11:25|A           |0        |1       |2          |
|16|Alice |B      |2024-11-09 11:11:26|A           |1        |2       |1          |
|17|Alice |A      |2024-11-09 11:11:27|B           |1        |3       |2          |
|18|Alice |A      |2024-11-09 11:11:28|A           |0        |3       |2          |
|19|Alice |C      |2024-11-09 11:11:29|A           |1        |4       |2          |
|20|Alice |C      |2024-11-09 11:11:30|C           |0        |4       |2          |
|21|Alice |D      |2024-11-09 11:11:31|C           |1        |5       |2          |
|22|Alice |D      |2024-11-09 11:11:32|D           |0        |5       |2          |
+--+------+-------+-------------------+------------+---------+--------+-----------+
```

在上面结果的基础上排除分组数量为 1 的记录

```sql
SELECT id, sender, content, create_at
FROM (
    SELECT *, COUNT(*) OVER(PARTITION BY sender, group_id) AS group_count
    FROM (
        SELECT *, SUM(is_change) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
        FROM (
            SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
            FROM (
                SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
                FROM message
            ) AS preved
        ) AS content_change
    ) AS grouped
) AS counted
WHERE group_count > 1
ORDER BY id;
```

此时我们得到的记录中已经没有突然变化的内容，在此之上重新计算每个消息的前一个消息

```sql
SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
FROM (
    SELECT id, sender, content, create_at
    FROM (
        SELECT *, COUNT(*) OVER(PARTITION BY sender, group_id) AS group_count
        FROM (
            SELECT *, SUM(is_change) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
            FROM (
                SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
                FROM (
                    SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
                    FROM message
                ) AS preved
            ) AS content_change
        ) AS grouped
    ) AS counted
    WHERE group_count > 1
) AS continued
ORDER BY id;
```

接下来就是比较当前消息和前一个消息是否一样，如果不一样则消息发生了变化

```sql
SELECT *
FROM (
    SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
    FROM (
        SELECT id, sender, content, create_at
        FROM (
            SELECT *, COUNT(*) OVER(PARTITION BY sender, group_id) AS group_count
            FROM (
                SELECT *, SUM(is_change) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
                FROM (
                    SELECT *, CASE WHEN content = prev_content THEN 0 ELSE 1 END AS is_change
                    FROM (
                        SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
                        FROM message
                    ) AS preved
                ) AS content_change
            ) AS grouped
        ) AS counted
        WHERE group_count > 1
    ) AS continued
) AS re_preved
WHERE (prev_content IS NULL OR content != prev_content)
ORDER BY id;
```

最终我们得到的结果如下表所示

```text
+--+------+-------+-------------------+------------+
|id|sender|content|create_at          |prev_content|
+--+------+-------+-------------------+------------+
|1 |Bob   |A      |2024-11-09 11:11:11|null        |
|3 |Bob   |B      |2024-11-09 11:11:13|A           |
|7 |Bob   |A      |2024-11-09 11:11:17|B           |
|10|Bob   |D      |2024-11-09 11:11:20|A           |
|14|Alice |A      |2024-11-09 11:11:24|null        |
|19|Alice |C      |2024-11-09 11:11:29|A           |
|21|Alice |D      |2024-11-09 11:11:31|C           |
+--+------+-------+-------------------+------------+
```

为了更清晰地表达我们的意图和逻辑下面使用 `WITH` 改写上面的嵌套查询

```sql
WITH preved AS (
    SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
    FROM message
),
grouped AS (
    SELECT *, SUM(CASE WHEN content = prev_content THEN 0 ELSE 1 END) OVER(PARTITION BY sender ORDER BY create_at) AS group_id
    FROM preved
),
counted AS (
    SELECT *, COUNT(*) OVER(PARTITION BY sender, group_id) AS group_count
    FROM grouped
),
continued AS (
    SELECT id, sender, content, create_at
    FROM counted
    WHERE group_count > 1
),
re_preved AS (
    SELECT *, LAG(content) OVER(PARTITION BY sender ORDER BY create_at) AS prev_content
    FROM continued
),
changed AS (
    SELECT *
    FROM re_preved
    WHERE prev_content IS NULL OR content != prev_content
)
SELECT * FROM changed ORDER BY id;
```

上面的方法可以通过调节每个分组中的记录数即可实现消息必须连续出现任意次数的需求，比如改为 `group_count > 2`，就可以实现消息必须连续出现三次及以上的需求。

### 使用全局行号和局部行号的差作为分组的 ID

```sql
WITH rowed AS (
    SELECT *,
           ROW_NUMBER() OVER(PARTITION BY sender ORDER BY create_at) AS global_row_num, -- 全局行号
           ROW_NUMBER() OVER(PARTITION BY sender, content ORDER BY create_at) AS local_row_num -- 局部行号
    FROM message
),
grouped AS (
    SELECT *, global_row_num - local_row_num AS group_id
    FROM rowed
),
counted AS (
    SELECT *, COUNT(*) OVER(partition by sender, group_id) AS group_count
    FROM grouped
)
SELECT * FROM counted WHERE group_count > 1 ORDER BY id;
```

或者

```sql
WITH grouped AS (
    SELECT *, ROW_NUMBER() OVER(PARTITION BY sender ORDER BY create_at) - ROW_NUMBER() OVER(PARTITION BY sender, content ORDER BY create_at) AS group_id
    FROM message
),
counted AS (
    -- 计算连续记录的出现次数
    SELECT sender, content, group_id, COUNT(*) AS consecutive_count
    FROM grouped
    GROUP BY sender, content, group_id
    HAVING COUNT(*) > 1 -- 筛选出连续出现两次及以上的记录
)
SELECT m.id, m.sender, m.content, m.create_at
FROM message AS m
    JOIN counted AS c ON m.sender = c.sender AND m.content = c.content
    JOIN grouped AS g ON m.id = g.id AND g.group_id = c.group_id
ORDER BY m.id;
```
