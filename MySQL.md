## InnoDB存储查找原理
可以。我们用一个**真实查询例子**把 InnoDB 的索引结构讲透。你先记住一句话：

**在 InnoDB 里，表数据本身就是一棵按主键组织的 B+Tree；普通索引也是 B+Tree，但普通索引的叶子节点不存完整行，只存“索引列 + 主键值”。**

---

# 1. 准备一张表

假设有这样一张用户表：

```sql
CREATE TABLE user (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    city VARCHAR(50),
    INDEX idx_age(age),
    INDEX idx_name_age(name, age)
) ENGINE=InnoDB;
```

表里有这些数据：

```text
id    name     age    city
1     Alice    18     Paris
2     Bob      20     London
3     Cindy    20     Berlin
4     David    25     Paris
5     Eva      30     Rome
6     Frank    35     Paris
```

这张表有三个索引：

```text
PRIMARY KEY(id)          主键索引 / 聚簇索引
idx_age(age)             普通二级索引
idx_name_age(name, age)  联合二级索引
```

---

# 2. InnoDB 中的表不是“数组”，而是 B+Tree

很多初学者会以为表在磁盘上是这样：

```text
row1
row2
row3
row4
row5
```

但 InnoDB 不是这样理解表的。

在 InnoDB 里，一张表的数据是按主键组织成一棵 B+Tree。

也就是说，对于上面的 `user` 表，真正的数据组织大概是：

```text
聚簇索引 PRIMARY(id)

                 [3]
              /       \
           /             \
     [1, 2]              [3, 4, 5, 6]
      |                       |
      v                       v
叶子节点存完整行          叶子节点存完整行

id=1 Alice 18 Paris      id=3 Cindy 20 Berlin
id=2 Bob   20 London     id=4 David 25 Paris
                         id=5 Eva   30 Rome
                         id=6 Frank 35 Paris
```

真实结构比这个复杂，节点是以 **Page，页** 为单位组织的。InnoDB 默认页大小是 16KB。一个页里可以存很多条记录。

你可以把它理解成：

```text
B+Tree 节点 = 一个或多个 InnoDB Page
Page 里存多条索引记录
```

---

# 3. 聚簇索引：叶子节点直接存完整行数据

主键索引也叫 **聚簇索引 Clustered Index**。

对于 InnoDB 来说，主键索引非常特殊：

```text
主键索引的叶子节点 = 完整数据行
```

例如：

```text
PRIMARY(id) 的叶子节点内容：

id=1 -> id=1, name=Alice, age=18, city=Paris
id=2 -> id=2, name=Bob,   age=20, city=London
id=3 -> id=3, name=Cindy, age=20, city=Berlin
id=4 -> id=4, name=David, age=25, city=Paris
id=5 -> id=5, name=Eva,   age=30, city=Rome
id=6 -> id=6, name=Frank, age=35, city=Paris
```

所以你可以把 InnoDB 表理解成：

```text
表 = 主键索引 B+Tree
```

不是“表 + 主键索引”，而是“表数据本身就挂在主键索引的叶子节点上”。

---

# 4. 例子一：按主键查找

执行：

```sql
SELECT * FROM user WHERE id = 4;
```

InnoDB 的查找过程大概是：

```text
第一步：从 PRIMARY(id) 的根节点开始
第二步：判断 id=4 应该走哪条分支
第三步：进入对应的叶子页
第四步：在叶子页中找到 id=4
第五步：因为主键索引叶子节点存完整行，所以直接返回整行数据
```

图示：

```text
查询条件：id = 4

                 [3]
              /       \
       id < 3           id >= 3
        |                  |
        v                  v
     [1, 2]          [3, 4, 5, 6]
                         |
                         v
              找到 id=4 的完整行：

              id=4, name=David, age=25, city=Paris
```

所以这个查询非常快，因为它只需要走一棵树。

大致过程是：

```text
PRIMARY B+Tree
    |
    v
找到叶子节点
    |
    v
直接拿到完整行
```

---

# 5. 例子二：按普通索引查找

现在执行：

```sql
SELECT * FROM user WHERE age = 20;
```

因为表上有普通索引：

```sql
INDEX idx_age(age)
```

InnoDB 会走 `idx_age`。

但是注意：`idx_age` 是二级索引，不是聚簇索引。二级索引的叶子节点不存完整行。

它存的是：

```text
age + 主键 id
```

所以 `idx_age` 的结构大概是：

```text
idx_age(age)

age=18 -> id=1
age=20 -> id=2
age=20 -> id=3
age=25 -> id=4
age=30 -> id=5
age=35 -> id=6
```

更像这样：

```text
二级索引 idx_age 的叶子节点：

age    id
18     1
20     2
20     3
25     4
30     5
35     6
```

执行：

```sql
SELECT * FROM user WHERE age = 20;
```

查找过程是：

```text
第一步：在 idx_age 这棵 B+Tree 中查 age=20
第二步：找到两条二级索引记录：
        age=20, id=2
        age=20, id=3

第三步：因为 SELECT * 需要完整行，但 idx_age 里没有 name、city 等完整数据
第四步：拿 id=2 回到 PRIMARY(id) 聚簇索引查完整行
第五步：拿 id=3 回到 PRIMARY(id) 聚簇索引查完整行
第六步：返回两行数据
```

图示：

```text
查询条件：age = 20

先查二级索引 idx_age：

idx_age:
age=18 -> id=1
age=20 -> id=2   命中
age=20 -> id=3   命中
age=25 -> id=4
age=30 -> id=5
age=35 -> id=6

得到主键 id=2, id=3

然后回到 PRIMARY(id)：

PRIMARY:
id=2 -> id=2, name=Bob,   age=20, city=London
id=3 -> id=3, name=Cindy, age=20, city=Berlin
```

这个“拿二级索引里的主键，再去主键索引查完整行”的过程，就叫：

```text
回表
```

---

# 6. 二级索引为什么不直接存完整行？

你可能会问：为什么 `idx_age` 不直接把完整行也存进去？这样不是不用回表了吗？

原因是空间成本太高。

假设表里有这些索引：

```sql
INDEX idx_age(age),
INDEX idx_city(city),
INDEX idx_name_age(name, age)
```

如果每个二级索引都存完整行，那么同一行数据会被复制很多份：

```text
PRIMARY(id) 里存完整行
idx_age 里也存完整行
idx_city 里也存完整行
idx_name_age 里也存完整行
```

这会导致：

```text
1. 磁盘空间暴涨
2. 插入变慢
3. 更新变慢
4. 页分裂更多
5. 缓存命中率下降
```

所以 InnoDB 选择：

```text
聚簇索引叶子节点存完整行
二级索引叶子节点只存索引列 + 主键值
```

这是一种空间和查询效率之间的权衡。

---

# 7. 例子三：覆盖索引，不需要回表

现在执行：

```sql
SELECT id, age FROM user WHERE age = 20;
```

注意这次不是 `SELECT *`，而是只查：

```text
id, age
```

而 `idx_age` 里面本来就有：

```text
age + id
```

所以查询过程变成：

```text
第一步：查 idx_age
第二步：找到 age=20 的记录
第三步：直接从 idx_age 里拿到 age 和 id
第四步：返回结果
```

不需要回到主键索引查完整行。

图示：

```text
idx_age 叶子节点：

age    id
18     1
20     2   命中，已经有 age 和 id
20     3   命中，已经有 age 和 id
25     4
30     5
35     6
```

返回：

```text
id    age
2     20
3     20
```

这种情况叫：

```text
覆盖索引 Covering Index
```

也就是：

```text
查询需要的字段，索引本身已经全部包含了，不需要回表。
```

所以这条 SQL：

```sql
SELECT id, age FROM user WHERE age = 20;
```

通常比这条 SQL 快：

```sql
SELECT * FROM user WHERE age = 20;
```

因为前者可以覆盖索引，后者大概率需要回表。

---

# 8. 例子四：联合索引的查找过程

表里还有一个联合索引：

```sql
INDEX idx_name_age(name, age)
```

这个索引的排序规则是：

```text
先按 name 排序
name 相同，再按 age 排序
最后存主键 id
```

假设数据在 `idx_name_age` 里的逻辑顺序是：

```text
name     age    id
Alice    18     1
Bob      20     2
Cindy    20     3
David    25     4
Eva      30     5
Frank    35     6
```

如果有重复 name，例如：

```text
name     age    id
Tom      18     7
Tom      20     8
Tom      25     9
```

联合索引里会按：

```text
先 name，再 age，再主键 id
```

排序。

---

## 8.1 查询 name 和 age

执行：

```sql
SELECT * FROM user WHERE name = 'Bob' AND age = 20;
```

可以完整使用联合索引：

```text
idx_name_age(name, age)
```

查找过程：

```text
第一步：在 idx_name_age 中查 name='Bob'
第二步：在 name='Bob' 的范围内继续查 age=20
第三步：找到对应主键 id=2
第四步：回到 PRIMARY(id) 查完整行
```

图示：

```text
idx_name_age:

name     age    id
Alice    18     1
Bob      20     2   命中
Cindy    20     3
David    25     4
Eva      30     5
Frank    35     6

得到 id=2

回表：
PRIMARY(id) 找 id=2 的完整行
```

---

## 8.2 只查 name

执行：

```sql
SELECT * FROM user WHERE name = 'Bob';
```

也可以使用 `idx_name_age`，因为联合索引满足**最左前缀原则**。

联合索引是：

```text
(name, age)
```

所以可以支持：

```sql
WHERE name = 'Bob'
WHERE name = 'Bob' AND age = 20
WHERE name = 'Bob' AND age > 18
```

但通常不能很好支持：

```sql
WHERE age = 20
```

因为 `age` 不是联合索引最左边的列。

你可以把联合索引理解成一本字典：

```text
先按 name 排序，再按 age 排序
```

如果你知道 name，很容易在这个字典里定位。

如果你只知道 age，不知道 name，就很难直接利用这本字典的整体排序。

---

# 9. 例子五：范围查询是怎么走索引的？

执行：

```sql
SELECT * FROM user WHERE age BETWEEN 20 AND 30;
```

因为有：

```sql
INDEX idx_age(age)
```

InnoDB 可以在 `idx_age` 中定位到 `age=20` 的位置，然后沿着叶子节点向后扫描，直到 `age > 30` 停止。

```text
idx_age 叶子节点有序：

age=18 -> id=1
age=20 -> id=2   开始
age=20 -> id=3
age=25 -> id=4
age=30 -> id=5
age=35 -> id=6   停止
```

查到的主键有：

```text
id=2
id=3
id=4
id=5
```

然后如果是：

```sql
SELECT * FROM user WHERE age BETWEEN 20 AND 30;
```

需要完整行，所以会回表：

```text
id=2 -> 回 PRIMARY 查完整行
id=3 -> 回 PRIMARY 查完整行
id=4 -> 回 PRIMARY 查完整行
id=5 -> 回 PRIMARY 查完整行
```

但是如果是：

```sql
SELECT id, age FROM user WHERE age BETWEEN 20 AND 30;
```

就可以直接用 `idx_age` 返回，不需要回表。

---

# 10. 例子六：没有索引时会发生什么？

执行：

```sql
SELECT * FROM user WHERE city = 'Paris';
```

但表上没有：

```sql
INDEX idx_city(city)
```

此时 InnoDB 不能通过 `city` 快速定位，只能扫描聚簇索引的叶子节点。

也就是：

```text
PRIMARY(id) 叶子节点：

id=1 Alice 18 Paris    检查 city，是 Paris，命中
id=2 Bob   20 London   检查 city，不命中
id=3 Cindy 20 Berlin   检查 city，不命中
id=4 David 25 Paris    检查 city，是 Paris，命中
id=5 Eva   30 Rome     检查 city，不命中
id=6 Frank 35 Paris    检查 city，是 Paris，命中
```

这就是全表扫描。

注意，InnoDB 的全表扫描并不是扫描“无序文件”，而是扫描：

```text
聚簇索引 PRIMARY(id) 的叶子节点
```

所以严格说，InnoDB 的表扫描本质上是在扫主键 B+Tree 的叶子节点。

---

# 11. 重点：InnoDB 的 B+Tree 节点里到底存什么？

## 11.1 主键索引

对于：

```sql
PRIMARY KEY(id)
```

叶子节点存：

```text
id + 完整行数据
```

例如：

```text
id=4 -> id=4, name=David, age=25, city=Paris
```

所以：

```sql
SELECT * FROM user WHERE id = 4;
```

一次索引查找就够。

---

## 11.2 二级索引

对于：

```sql
INDEX idx_age(age)
```

叶子节点存：

```text
age + 主键 id
```

例如：

```text
age=20 -> id=2
age=20 -> id=3
```

所以：

```sql
SELECT * FROM user WHERE age = 20;
```

需要：

```text
先查 idx_age
再用 id 回表查 PRIMARY
```

---

## 11.3 联合索引

对于：

```sql
INDEX idx_name_age(name, age)
```

叶子节点存：

```text
name + age + 主键 id
```

例如：

```text
name=Bob, age=20 -> id=2
```

如果查询：

```sql
SELECT id, name, age FROM user WHERE name = 'Bob' AND age = 20;
```

它需要的字段都在联合索引里：

```text
id
name
age
```

所以可以覆盖索引，不需要回表。

但如果查询：

```sql
SELECT id, name, age, city FROM user WHERE name = 'Bob' AND age = 20;
```

因为 `city` 不在 `idx_name_age` 中，所以需要回表。

---

# 12. 一条查询的完整对比

下面这几条 SQL，执行路径不同。

---

## SQL 1：主键查询

```sql
SELECT * FROM user WHERE id = 4;
```

路径：

```text
PRIMARY(id)
    |
    v
直接找到完整行
```

是否回表：

```text
不需要
```

性能：

```text
非常快
```

---

## SQL 2：普通索引查询完整行

```sql
SELECT * FROM user WHERE age = 20;
```

路径：

```text
idx_age(age)
    |
    v
找到 id=2, id=3
    |
    v
PRIMARY(id)
    |
    v
查完整行
```

是否回表：

```text
需要
```

性能：

```text
如果命中行少，通常很快；
如果命中行很多，回表成本会很高。
```

---

## SQL 3：普通索引覆盖查询

```sql
SELECT id, age FROM user WHERE age = 20;
```

路径：

```text
idx_age(age)
    |
    v
直接返回 id, age
```

是否回表：

```text
不需要
```

性能：

```text
通常比 SELECT * 快
```

---

## SQL 4：联合索引查询

```sql
SELECT id, name, age FROM user WHERE name = 'Bob' AND age = 20;
```

路径：

```text
idx_name_age(name, age)
    |
    v
直接返回 id, name, age
```

是否回表：

```text
不需要，因为覆盖索引
```

---

## SQL 5：没有索引

```sql
SELECT * FROM user WHERE city = 'Paris';
```

路径：

```text
扫描 PRIMARY(id) 的叶子节点
    |
    v
逐行判断 city 是否等于 Paris
```

是否回表：

```text
不涉及回表，因为直接扫的是聚簇索引叶子节点，本身就是完整行。
```

性能：

```text
数据量大时很慢
```

---

# 13. 为什么 B+Tree 适合数据库索引？

B+Tree 有几个特点特别适合数据库。

## 第一，树高很低

InnoDB 一个页默认 16KB，一个索引页可以存很多索引项。因此即使有几百万、几千万行数据，B+Tree 的高度也可能只有 3 到 4 层。

也就是说一次查询可能只需要：

```text
根节点 Page
内部节点 Page
叶子节点 Page
```

几次页访问就能定位数据。

---

## 第二，叶子节点有序，适合范围查询

比如：

```sql
SELECT * FROM user WHERE age BETWEEN 20 AND 30;
```

B+Tree 可以先定位到 `age=20`，然后沿着叶子节点顺序扫描。

```text
20 -> 20 -> 25 -> 30
```

这比哈希索引更适合范围查询。

---

## 第三，减少磁盘 I/O

数据库最怕随机磁盘 I/O。B+Tree 的每个节点都是页，InnoDB 以页为单位读写磁盘。

所以一次查询不是读一条记录，而是读一个页：

```text
读入一个 16KB 页
页里可能包含很多条记录
```

这能提高局部性。

---

# 14. 你可以这样记住 InnoDB 索引结构

最重要的是这张表：

```text
索引类型        叶子节点存什么                      是否存完整行
----------------------------------------------------------------
主键索引        主键 + 完整行数据                    是
二级索引        索引列 + 主键值                      否
联合索引        多个索引列 + 主键值                  否
```

所以：

```text
按主键查：
主键 B+Tree -> 完整行

按普通索引查 SELECT *：
普通索引 B+Tree -> 主键值 -> 主键 B+Tree -> 完整行

按普通索引查部分字段：
如果字段都在普通索引里 -> 覆盖索引 -> 不回表

没有索引：
扫描聚簇索引叶子节点 -> 逐行判断
```

---

# 15. 最后用一个完整类比帮你理解

你可以把 InnoDB 索引想象成图书馆。

## 聚簇索引

```text
主键索引 = 按书号排列的书架
书架上直接放着完整的书
```

你知道书号：

```text
id = 4
```

直接去书架拿书。

---

## 二级索引

```text
普通索引 = 按作者/年龄/名字做的一本目录
目录里不放完整的书，只写书号
```

你知道作者：

```text
age = 20
```

先查目录：

```text
age=20 -> 书号 id=2
age=20 -> 书号 id=3
```

然后拿书号去主书架找完整的书。

这个过程就是回表。

---

# 16. 一句话总结

**InnoDB 的索引结构核心是：主键索引的叶子节点存完整行，二级索引的叶子节点存索引列和主键值；所以按主键查询可以直接拿到数据，按普通索引查询完整行通常需要先查二级索引，再通过主键回到聚簇索引查完整行，这就是回表。**
