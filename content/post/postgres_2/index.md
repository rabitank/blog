---
title: "从 Postgres 开始学数据库（二）"
description: schema 修改、时区与时间函数、去重聚合、子查询、并发与事务、存储过程
date: 2026-09-30T12:00:00+08:00
image: 
math: 
license: 
comments: true
draft: true
categories: ["数据库"]
tags: ["postgres", "sql", "数据库"]
build:
    list: always
---

# schema 模式

**ALTER TABLE**
当dt 模式需要更改时, 使用alter来修改dt的schema.
数据库拥有自动处理和转换dt的能力,也支持运行时修改,但是不保证立即执行.

```sql
ALTER TABLE fav DROP COLUMN oops;  -- 删除列/属性
ALTER TABLE post ALTER COLUMN content TYPE TEXT; --修改列类型
ALTER TABLE fav ADD COLUMN howmuch INTEGER; -- 添加列

```

同样也可以修改约束
```
drop constraint
```


## 执行sql文件

```
\i sql.sql
```

## 时区问题

使用TIMESTAMPTZ, 带时区的时间戳, 以避免记录时刻时需要时区转换
NOW() 则是获取当前时刻, 类型同样是TSTZ

```sql
SHOW timezone; -- 查看数据库时区
```
默认Asia/Shanghai, 也就是东八区的北京时间(+08, 相对UTC)

## DEFAULT
你可以指定列的默认值或者默认函数
```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- 不允许为空, 默认值使用NOW填充
update_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
```

UTC 格林尼治时间, 和时区无关(一般等于英国)
使用此作为基准时间.

查看可用时区
```sql
SELECT * FROM pg_timezone_names; -- 这是数据库自带的一个伪表
```

### 转换
Now() 返回 TSTZ类型, 有几种方法将其转换为其他类型
```sql

NOW() ::DATE -- 糖

CAST(NOW() as DATE)

```

**INTERVAL**
可以把字符串转化为时间端.
```
SELECT NOW(), NOW() - INTERVAL '2 days', (NOW() - INTERVAL '2 days')::DATE;

              now              |           ?column?            |    date
-------------------------------+-------------------------------+------------
 2026-08-02 17:18:18.999109+08 | 2026-07-31 17:18:18.999109+08 | 2026-07-31
```

**DATE_TRUNC**
截断时间戳到某个粒度/单位,丢弃这之后的数据
```sql
SELECT DATE_TRUNC('day', NOW()), DATE_TRUNC('day', NOW() + INTERVAL '1 day');
       date_trunc       |       date_trunc
------------------------+------------------------
 2026-08-02 00:00:00+08 | 2026-08-03 00:00:00+08
```

选择今天发生的事情
```sql
SELECT id, contnent FROM comment WHERE created_at >= DATE_TRUNC('day', NOW())
AND created_at < DATE_TRUNC('day', NOW()) + INTERVAL'1 day');

SELECT id, contnent FROM comment WHERE created_at ::DATE = NOW()::DATE
-- 两种方法, 且注意 = 表示相同, 第二种方法更慢,存在全局扫描
```

## 消除垂直重复

对**SELECT返回结果集**去重/消除垂直重复
DISTINCT: 返回不重复的行
DISTINCT ON: 对部分列进行唯一性去重, **`DISTINCT ON` 必须与 `ORDER BY` 配合使用**，且 `ORDER BY` 的**开头**必须与 `DISTINCT ON` 的表达式一致。
GROUP BY: 去重,通过一些聚合函数 COUNT(), MAX(), SUM()等

```sql
SELECT DISTINCT model FROM racing;  -- 对model 列去重

SELECT DISTINCT ON (model) make, model FROM racing ORDER BY model, make;;  -- 对make, model结果中的model列去重, 当多个 `make` 对应同一个 `model` 时，**只有“排在第一个”的那个 `make` 会被留下，其他全部被丢弃**。

SELECT COUNT(abbrev), abbrev FROM pg_timezone_names GROUP BY abbrev;
-- 对abbrev出现进行计数并聚合, 这里如果同时使用了聚合函数COUNT(abbrev)和abbrev ,那GROUP BY是必须的
```

WHERE 聚合前过滤 + 消除 + **having** 类似于GROUP 后的WHERE 过滤

```
SELECT COUNT(abbrev) AS ct, abbrev FROM pg_timezone_names 
WHERE is_dst = 't' 
GROUP BY abbrev
HAVING COUNT(abbrev) > 10;
-- result
 ct | abbrev
----+--------
 23 | EEST
 20 | CDT
 37 | CEST
 27 | EDT
 12 | MDT
``` 

## 子查询/嵌套查询
```sql
select content from comment where account_id = (select id from account where email='ed@umich.edu')

-- it account_id = 7
select account_id from account where email='ed@umich.edu'  -- result 7
```
即将两次查询合到一起。然而数据库只会对单次查询进行优化，对于子查询里的多条查询语句，数据库不会缓存而是每次迭代都按子查询描述的做。
where，having一类的都可以用子查询代替，但是低效。
```sql
SELECT ct,abbrev from (
	select count(abbrev) as ct, abbrev from pg_timezone_names where is_dst = 't' GROUP BY abbrev
) as zap
where ct > 10;
```
上面这段sql中, where ct> 10 与 子语句可以合为接在最后的having count(abbrev) > 10

# 并发性
读取与写入,或者说一条指令必须原子性地发生
数据库使用锁技术

## Compound 语句/追加/复合/后缀

在一条语句中做更多的事情以保证效率和并发

```sql
.... returning * -- 执行操作后要求返回整条记录

insert into fav (post_id, account_id, howmuch) values (1, 1, 1)
returning *;
```

### on conflict ... do ...
非常常用， 因为并发操作时无法确认服务器数据状态。
在指定的属性组上发生冲突时放弃操作并, do ...
```sql
insert into fav (post_id, account_id) values (1, 1)
	on conflict (post_id, account_id)
	do update set howmuch = fav.howmuch + 1 -- 注意这里的上下文是在冲突发生之后
returning *;
```

## 事务
事务（Transaction）是把多条SQL语句打包成一个**不可分割的执行单元**。在 PostgreSQL 中，默认每条SQL都是自动提交的单语句事务，但用 `BEGIN` 可以开启一个显式事务。
### begin , rollback, commit

- **`BEGIN`（或 `START TRANSACTION`）**：**开启事务**。从此处开始，**你之后执行的所有修改（增删改）都只在当前会话的“工作区”中生效，其他用户看不到，且数据并未真正写入磁盘。** 类似于COW机制。
    
- **`COMMIT`**：**提交事务**。将工作区的所有更改**永久写入**磁盘。一旦提交，数据就真正固化了，无法通过回滚撤销。
    
- **`ROLLBACK`**：**回滚事务**。放弃工作区的所有更改，数据库状态恢复到 `BEGIN` 之前的样子。常用于发现操作有误时及时止损。

###  SELECT ... FOR UPDATE —— 行级写锁

当你在查询语句末尾加上 `FOR UPDATE`，被查询选中的**所有行**都会被锁定。
- **其他事务的行为**：
    - 其他事务的 `SELECT`（普通查询）不受影响，依然能读到数据。
    - 但其他事务的 `UPDATE`、`DELETE`，甚至另一个 `SELECT ... FOR UPDATE` **都会被阻塞**，必须等到当前事务提交或回滚后，才能继续执行。

### SELECT ... FOR UPDATE OF —— 锁定“特定表”的行

`FOR UPDATE OF` 是 `FOR UPDATE` 的进阶版，主要用于 **多表联查（JOIN）** 时，**精准指定**只锁定其中某一个（或某几个）表的行，而其他表的数据不加锁。

```sql
begin;
select xxxxxxxxxxxxxxxxx for update of dt -- 请求锁住dt选中行进行 update 语义操作
-- time pass
rollback -- 放弃这次更新操作并解锁


begin;
xxxxxxxxxxxxxxxxx for update of dt 
-- time pass
commit -- 提交这次更新到数据区并解锁

```

- **`NOWAIT`**：如果行被其他事务锁定，**立即报错**返回，而不是等待。适用于“抢不到就放弃”的场景。
    
```sql
    SELECT * FROM products WHERE id = 1 FOR UPDATE NOWAIT;
```
     
- **`SKIP LOCKED`**：跳过所有被锁定的行，只返回当前未被锁定的行。适用于“谁空闲就处理谁”的任务队列。
    
```sql
    SELECT * FROM task_queue WHERE status = 'ready' FOR UPDATE SKIP LOCKED;
```

### 发生错误
事务中发生错误指令后事务终止, 此后的指令都将失效不再执行直到事务块被显式结束。终止后的事务块释放了锁。


# 存储过程

存储过程就是一段命名sql语句块， 可以通过名字进行调用。
数据库自带了一些，比如xxx_help之类的过程用于返回帮助说明。
用户可以自定义创建，配合触发器进行调用以自动化操作。
除此之外，存储过程作为语句块可被数据库编译以加速执行。
一个配合触发器的例子
```sql
create or replace function trigger_set_timestamp)_
returns trigger as $$
begin 
	new.updated_at = now(); -- assign now() timestamp to dt 'new' updated_at attr
	return new
end;
$$ language plpgsql;

create trigger set_timestamp
before update on fav
for each row
execute procedure trigger_set_timestamp();


```
1. **`new.updated_at = now();`**
    - `NEW` 是 PostgreSQL 触发器里的**特殊变量**，代表“即将被插入或更新的那行数据”的内存镜像。
    - 这行代码把该行里的 `updated_at` 字段赋值为**当前的系统时间戳**。
    - **关键理解**：此时修改的是内存里的 `NEW` 行，**还没有真正写入磁盘**。
2. **`return new;`**
    - 对于 `BEFORE` 触发器（前置触发器），**必须返回 `NEW`**，否则后续的 INSERT/UPDATE 操作会因拿不到数据而报错。
    - 返回 `NEW` 相当于把修改后的内存行“递交”给真正的 DML 语句去执行。
严格来说，PL/pgSQL 中 `returns trigger` 叫“触发器函数”，它不能像普通函数那样被 `SELECT` 直接调用，只能由触发器触发执行。
- **`before update on fav`**：在 `fav` 表的 **UPDATE 操作执行之前** 触发。
- **`for each row`**：如果一条 UPDATE 语句影响了 10 行，这个触发器就会被执行 10 次（逐行处理）。
- **`execute procedure ...`**：指定要调用的触发器函数。
