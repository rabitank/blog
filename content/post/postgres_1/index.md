---
title: "从 Postgres 开始学数据库（一）"
description: 安装、SQL 命令与增删改查、字段类型、主键索引与数据库设计（键、JOIN、多对多）
date: 2026-09-30T11:55:00+08:00
image: 
math: 
license: 
comments: true
draft: false
categories: ["数据库"]
tags: ["postgres", "sql", "数据库"]
build:
    list: always
---

## 安装
下载即可`
手动添加psql到环境变量
注意先以psql -U postgres 登录超级用户
```sql
CREATE USER "count" WITH PASSWORD "password"
CREATE DATABASE adatabase WITH OWNER 'count';
```
创建好用户和database, 推荐用户名使用windows账户名这样不用指定登录用户

>  关键字可以不用大写来着..
# SQL
## 命令
```
\q 退出
\i sql_cmds_file 执行sql脚本
\dt 查看datatable
\d+ datatable 查看模式
```


## Sql操作

注意单引号和双引号有区别, 字符串值必须使用单引号包裹

### 创建
```sql
CREATE TABLE users(
 name VARCHAR(128),
 email VARCHAR(128)
);
```
### 插入
```sql
INSERT INTO users (name, email) VALUES ('Chuck', 'csev@umich.edu');
```
### 删除
```sql
-- 如无WHERE则会删除整个dt
DELETE FROM users WHERE email=' csev@umich.edu ';
```
### 更新
```sql
UPDATE users SET name='Ekalous' WHERE email='csev@ekalos.edu';
```

>  where 隐含了迭代查找,每个匹配的都会执行

### 查询 (Retrieving)/选择
```sql
# SELECT 选择行, * 表任意
SELECT * FROM dt WHERE attr=value
```

### 排序:
```sql
对select 返回结果 进行排序
SELECT * FROM users ORDER BY attr
SELECT * FROM users ORDER BY attr DESC # 降序
```
### 匹配/选择:
LIKE实际上就是某种正则, `%` 为通配
>  LIKE 无法利用上cache的index, 需要整个扫描一遍
```sql
SELECT * FROM users WHERE name LIKE '%e%';
```

### 限定

**LIMIT**
查询语句后加上限制, LIMIT只查找限定数量的记录
```sql
query sql + LIMIT number;
```

**OFFSET**
加上offset则可以跳过n条数据,获取后面的
以上加在排序查询语句后面很有用
>  从 0 开始, 
>  OFFSET 1 返回从第二条开始的记录
```sql
query sql + OFFSET number;
```
### 计数
COUNT(), 不是sql命令而是聚合函数
```sql
# 计算at = val的记录或者说行有多少条
SELECT COUNT(*) FROM dt WHERE at=val;
```

### 整个删除dt

```sql
DROP TABLE dt;
```
## 字段

### 字符串
- CHAR(n):
	- 分配**固定**长子串,存储短字符,一般小于64字符(存固定长guid就很完美),最好存满
- VARCHAR(n):
	- 可变长字符串,存长字符串,64~128长一般(只是视频推荐用法), 使用字符长计数压缩,没有空余空间
- TEXT:
	- 文本字段,无限制长度,可以很大.
	- 不能索引和排序, WHERE不能用, 但是可以用LIKE

> 关于字符集- Character
> Character 8~32 bit占位, CHAR, VARCHAR, TEXT 都存储的字符集而不只是字节

### 字节
- BYTEA(n):
	- 8-32 bytes 信息存储, 指定字节数目, 最大到255字节

### 整数
- SMALLINT: +-30000
- INTEGER 20亿
- BIGINT 10^18
### 浮点
- REAL: 32bits, 7位精度
- DOUBLE PRECISION 64bits 14位精度 10^308
- NUMERIC:特殊小数,模拟十进制以精确表示货币

### 日期
- TIMESTAMP: YYYY-MM-DD HH:MM:DD 时间戳 64bits,实际记录内容位BD 4713年开始的秒 (+TZ会统一时区)
- DATA: YYYY-MM-DD
- TIME: HH:MM:SS 


## 自动主码
一般来说对于接收插入的dt, 希望添加id并以添加顺序作为值, id即成为主键/码
Postgres提供SERIAL来自动化这个操作(非标准)

```sql
CREATE  TABLE users (
id SERIAL,
name VARCHAR(128),
email VARCHAR(128) UNIQUE, -- 声明at 值应该不重复
PRIMARY KEY(id) -- 声明主键
)
```

## Index索引
索引是指定主键的dt后,数据库生成的记录数据位置的缓存加速访问结构

树索引: 对数据分块建立索引, 树结构存储块位置, 适合前缀匹配
hash索引:MD5, SHA-1, SHA-256等hash算法. 只适用精确匹配(唯一标识符)

具体哪种索引由数据库决定

## LoadFromCsv
psql命令
```sql
\copy dt(atr, atr, atr) FROM 'csvpath.csv' WITH DELIMITER ',' CSV;
```

# 数据库设计

## 键
这里的键在视频教程中类似于attribute或者教材中的'码'
当然这里没有约束
- 主键:用于查找确认的唯一属性或者属性组,一般和内容无关的id数字, id命名
- 逻辑键:普通的有含义的属性,可以通过逻辑键建立索引,通过unique声明来构建这种快速查找和判断机制
- > title varchar(128) UNIQUE
- 外键: 其他关系的主键,一般用 xxx_id 命名
原则
- 面向修改, 不要重复数据
- 使用整数作为主键和引用
- 为每一个dt添加特殊id
- 设计时围绕存储主体思考

belong to, A 属于B, 会在A中加入B外键.

**UNIQUE声明**
唯一性声明,会拒绝加入重复的声明过唯一的属性或者属性组合
```sql
create table genre(
id SERIAL,
name varchar(128) unique, -- 声明流派名是唯一的
primary key(id)
);
```

```sql
create table track(
id SERIAL,
title varchar(128),
len integer,
rating integer,
count integer,
album_id integer references album(id) on delete cascade,
genre_id integer references genre(id) on delete cascade,
UNIQUE(title, album_id),  -- 声明title album_id组合是唯一的
primary key(id)
);
```

**索引**
对主键和唯一性声明会建立索引
这里数据库自动创建了btree索引
```
索引：
    "track_pkey" PRIMARY KEY, btree (id)
    "track_title_album_id_key" UNIQUE CONSTRAINT, btree (title, album_id)
```

**联级删除规则 外键声明**
声明属性时使用references + dt(attri) 来声明外键引用.
delete cascade 说明当外键数据删除时本条记录也同步删除
```
artist_id INTERGER REFERENCES artist(id) ON DELETE CASCADE
```

### 几种联级
Defalut / RESTICT: 不允许在未删除引用外键的情况下删除被引用的记录
CASCADE: 联级, 实际上类似父子行关系, 被引用记录变化后也会跟着一起
SET NULL: 被引用记录删除后引用部分(外键列)置空
>  置空需要列的类型允许, 如INTEGER NULL才能允许列值为空
## 链接

**JOIN ON**
JOIN, 组合并链接两个dt, 可添加θ
所谓的θ链接, 即组合两个dt后ON后面添加条件进行限定.
(JOIN未指定情况下就是 INNER JOIN)

```sql
SELECT album.title, artist.name FROM album JOIN artist ON album.artist_id = artist.id;
```
JOIN可以连续嵌套
```sql
SELECT * 
FROM album 
	JOIN  artist ON  ****
	JOIN genre ON ****
	JOIN track ON ****
```

**CROSS JOIN**
即卡笛尔积,不要使用它,纯组合来的

## 多对多关系设计

多对多设计不能使用单个表表示, 比如在track后面跟所属父级id之类的.
而是使用中间表来专门表示多对多关系.

>  此外创建dt需要从边缘向中心创建,因为外键需要在主键创建后才能被创建


主键(主码) 使用属性组

```sql
CREATE TABLE member(
	student_id INTEGER REFERENCES student(id) ON DELETE CASCADE,
	course_id INTEGER REFERENCES course(id) ON DELETE CASCADE,
	role INTEGER,  -- 描述这种组合关系种类
	PRIMARY KEY(student_id, course_id)
)
```

查询实现
```sql
SELECT student.name, member.role, course.title
FROM student
JOIN member ON member.student_id = student.id
JOIN course ON member.course_id  = course.id
ORDER BY course.title, member.role DESC, student.name;
```
