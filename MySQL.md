## InnoDB存储查找原理
可以。我们用一个**真实查询例子**把 InnoDB 的索引结构讲透。你先记住一句话：

**在 InnoDB 里，表数据本身就是一棵按主键组织的 B+Tree；普通索引也是 B+Tree，但普通索引的叶子节点不存完整行，只存“索引列 + 主键值”。**

---

### 1. 准备一张表

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

### 2. InnoDB 中的表不是“数组”，而是 B+Tree

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

### 3. 聚簇索引：叶子节点直接存完整行数据

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

### 4. 例子一：按主键查找

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

### 5. 例子二：按普通索引查找

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
***🐦TIPS:当查询需要的字段已经全部在二级索引里时，就不需要回表。***

比如：

SELECT id, age FROM user WHERE age = 20;

idx_age(age) 的叶子节点里本来就有：

age + id

所以它可以直接返回：

id    age
2     20
3     20

不需要再去主键索引里查完整行。

---

### 6. 二级索引为什么不直接存完整行？

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

### 7. 例子三：覆盖索引，不需要回表

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

### 8. 例子四：联合索引的查找过程

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

### 8.1 查询 name 和 age

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

### 8.2 只查 name

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

### 9. 例子五：范围查询是怎么走索引的？

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

### 10. 例子六：没有索引时会发生什么？

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

### 11. 重点：InnoDB 的 B+Tree 节点里到底存什么？

> 11.1 主键索引

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

> 11.2 二级索引

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

> 11.3 联合索引

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

### 12. 一条查询的完整对比

下面这几条 SQL，执行路径不同。

---

> SQL 1：主键查询

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

> SQL 2：普通索引查询完整行

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

> SQL 3：普通索引覆盖查询

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

> SQL 4：联合索引查询

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

> SQL 5：没有索引

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

### 13. 为什么 B+Tree 适合数据库索引？

B+Tree 有几个特点特别适合数据库。

🧁>第一，树高很低

InnoDB 一个页默认 16KB，一个索引页可以存很多索引项。因此即使有几百万、几千万行数据，B+Tree 的高度也可能只有 3 到 4 层。

也就是说一次查询可能只需要：

```text
根节点 Page
内部节点 Page
叶子节点 Page
```

几次页访问就能定位数据。

---

🧁> 第二，叶子节点有序，适合范围查询

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

🧁> 第三，减少磁盘 I/O

数据库最怕随机磁盘 I/O。B+Tree 的每个节点都是页，InnoDB 以页为单位读写磁盘。

所以一次查询不是读一条记录，而是读一个页：

```text
读入一个 16KB 页
页里可能包含很多条记录
```

这能提高局部性。

---

### 14. 你可以这样记住 InnoDB 索引结构

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

### 15. 最后用一个完整类比帮你理解

你可以把 InnoDB 索引想象成图书馆。

> 聚簇索引

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

> 二级索引

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

### 16. 一句话总结

**InnoDB 的索引结构核心是：主键索引的叶子节点存完整行，二级索引的叶子节点存索引列和主键值；所以按主键查询可以直接拿到数据，按普通索引查询完整行通常需要先查二级索引，再通过主键回到聚簇索引查完整行，这就是回表。**


对，你的理解已经非常接近了，但需要把一句话修正得更严谨：

> **无论是聚簇索引还是二级索引，它们的非叶子节点存储的都是“索引键 + 子页指针”；叶子节点本身就是一个个 Page，这些 Page 里面存放多条记录。区别在于：聚簇索引叶子 Page 里存完整行数据；二级索引叶子 Page 里存二级索引列 + 主键值。**

你刚才说的“叶子节点存储的是 Page 里面包含一段在这个索引下方的数据记录”，方向对，但更准确地说应该是：

```text
B+Tree 的叶子节点 ≈ 一个 InnoDB Page
Page 里面包含多条记录
```

不是“叶子节点里面再存一个 Page”，而是：

```text
叶子节点本身就是由 Page 承载的。
```

---

### 物理存储结构
> 1. 聚簇索引的结构

假设表是：

```sql
CREATE TABLE user (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    city VARCHAR(50),
    INDEX idx_name(name)
) ENGINE=InnoDB;
```

数据：

```text
id    name     age    city
1     Alice    18     Paris
2     Bob      20     London
3     Cindy    22     Berlin
4     David    25     Paris
5     Eva      30     Rome
```

聚簇索引是 `PRIMARY(id)`。

它的逻辑结构可以理解为：

```text
PRIMARY(id) B+Tree

                  根 Page
              [id=3] -> 子页指针
             /                \
            /                  \
  叶子 Page 100              叶子 Page 101
  id=1 完整行                id=3 完整行
  id=2 完整行                id=4 完整行
                             id=5 完整行
```

更具体地说：

```text
聚簇索引非叶子 Page：

Page 1
--------------------------------
key: id=3  -> child_page=101
--------------------------------
```

聚簇索引叶子 Page：

```text
Page 100
------------------------------------------------
id=1, name=Alice, age=18, city=Paris
id=2, name=Bob,   age=20, city=London
------------------------------------------------

Page 101
------------------------------------------------
id=3, name=Cindy, age=22, city=Berlin
id=4, name=David, age=25, city=Paris
id=5, name=Eva,   age=30, city=Rome
------------------------------------------------
```

所以对于聚簇索引：

```text
非叶子节点：主键 id + 子页指针
叶子节点：主键 id + 完整行数据
```

---

> 2. 二级索引的结构

现在看二级索引：

```sql
INDEX idx_name(name)
```

它也是一棵 B+Tree。

但是它的内容不是完整行，而是：

```text
name + 主键 id
```

逻辑结构类似：

```text
idx_name(name) B+Tree

                  根 Page
            [name='Cindy'] -> 子页指针
             /                         \
            /                           \
  叶子 Page 200                       叶子 Page 201
  name=Alice, id=1                    name=Cindy, id=3
  name=Bob,   id=2                    name=David, id=4
                                      name=Eva,   id=5
```

更具体地说：

```text
二级索引非叶子 Page：

Page 2
-------------------------------------
key: name='Cindy' -> child_page=201
-------------------------------------
```

二级索引叶子 Page：

```text
Page 200
--------------------
name=Alice, id=1
name=Bob,   id=2
--------------------

Page 201
--------------------
name=Cindy, id=3
name=David, id=4
name=Eva,   id=5
--------------------
```

所以对于二级索引：

```text
非叶子节点：二级索引键 name + 子页指针
叶子节点：二级索引键 name + 主键 id
```

如果是联合索引：

```sql
INDEX idx_name_age(name, age)
```

那么它的叶子 Page 存：

```text
name + age + 主键 id
```

非叶子 Page 也按照：

```text
name + age
```

来组织导航。

---

> 3. 非叶子节点到底存什么？

你说“非叶子节点存储的都是索引，比如 id、name 等”，这句话基本对，但要补一个关键点：

🐦‍⬛**非叶子节点不只是存索引值，还要存“指向下一层 Page 的指针”。**

否则它不知道下一步该去哪个 Page。

所以非叶子节点更准确是：

```text
索引键 + 子页指针
```

例如聚簇索引：

```text
id=100 -> child_page=8
id=300 -> child_page=9
id=500 -> child_page=10
```

表示：

```text
小于 100 的去某个左侧页
100 到 300 的去 child_page=8
300 到 500 的去 child_page=9
大于 500 的去 child_page=10
```

二级索引也是类似，只不过 key 换成了二级索引列。

例如 `idx_name(name)`：

```text
name='Cindy' -> child_page=200
name='Frank' -> child_page=201
name='Tom'   -> child_page=202
```

---

### 4. 叶子节点到底存什么？

叶子节点，也就是叶子 Page，存的是一批有序记录。

但是不同索引的叶子 Page 记录内容不同。

***🥬聚簇索引叶子 Page***

```text
PRIMARY(id) 的叶子 Page 存完整行：

id=1, name=Alice, age=18, city=Paris
id=2, name=Bob,   age=20, city=London
id=3, name=Cindy, age=22, city=Berlin
```

所以主键查找可以直接拿完整数据。

---

### 二级索引叶子 Page

```text
idx_name(name) 的叶子 Page 存：

name=Alice, id=1
name=Bob,   id=2
name=Cindy, id=3
```

它不包含完整的 `age`、`city` 等字段。

所以：

```sql
SELECT * FROM user WHERE name = 'Bob';
```

会先查二级索引：

```text
name=Bob -> id=2
```

然后回表：

```text
PRIMARY(id) -> id=2 -> 完整行
```

---

### 5. 标准表述

> **无论是聚簇索引还是二级索引，它们本质上都是 B+Tree。B+Tree 的非叶子 Page 存储“索引键 + 子页指针”，用于导航到下一层 Page；叶子 Page 存储真正的索引记录。对于聚簇索引，叶子 Page 中的记录就是完整行数据；对于二级索引，叶子 Page 中的记录是“二级索引列 + 主键值”，如果查询字段没有被覆盖，就需要根据主键值回到聚簇索引查完整行。**


---

### 6. 用一张对比表记住

```text
索引类型       非叶子 Page 存什么                  叶子 Page 存什么
-----------------------------------------------------------------------
聚簇索引       主键值 + 子页指针                    主键值 + 完整行数据

二级索引       二级索引列 + 子页指针                二级索引列 + 主键值

联合索引       联合索引列 + 子页指针                联合索引列 + 主键值
```

例如：

```text
PRIMARY(id)

非叶子 Page:
id -> child_page

叶子 Page:
id + name + age + city
```

```text
idx_name(name)

非叶子 Page:
name -> child_page

叶子 Page:
name + id
```

```text
idx_name_age(name, age)

非叶子 Page:
name + age -> child_page

叶子 Page:
name + age + id
```

---

### 7. 最关键的区别

你可以这样抓主线：

```text
所有索引都是 B+Tree。
所有 B+Tree 都由 Page 组成。
所有非叶子 Page 都负责导航。
所有叶子 Page 都存实际索引记录。
```

但是：

```text
聚簇索引的叶子记录 = 完整行
二级索引的叶子记录 = 索引列 + 主键
```

所以：

```text
按主键查：
PRIMARY 非叶子 Page -> PRIMARY 叶子 Page -> 完整行

按二级索引查：
二级索引非叶子 Page -> 二级索引叶子 Page -> 主键值
                                     |
                                     v
                           PRIMARY 非叶子 Page
                                     |
                                     v
                           PRIMARY 叶子 Page -> 完整行
```

---

### 8. 最后一句话总结

**你的理解基本正确，只需要把“叶子节点存储的是 Page”改成“B+Tree 的叶子节点由 Page 承载，Page 里面存多条有序记录”。聚簇索引的叶子 Page 存完整行，二级索引和联合索引的叶子 Page 存索引列加主键值；非叶子 Page 存索引键加子页指针，用来导航到下一层 Page。**



## SQL最左索引原则🤖
SQL 的**最左索引原则**，更准确叫 **最左前缀原则**，主要针对**联合索引**。

一句话：

> 对于联合索引 `(a, b, c)`，MySQL 使用索引时，通常必须从最左边的字段 `a` 开始，按照索引定义的顺序连续使用，不能跳过中间字段。

---

# 1. 先看什么是联合索引

假设有一张用户表：

```sql
CREATE TABLE user (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    city VARCHAR(50),
    gender VARCHAR(10),
    INDEX idx_name_age_city(name, age, city)
) ENGINE=InnoDB;
```

这里有一个联合索引：

```sql
INDEX idx_name_age_city(name, age, city)
```

这个索引不是分别给 `name`、`age`、`city` 建了三个索引，而是建立了一棵按照下面顺序排序的 B+Tree：

```text
先按 name 排序
name 相同，再按 age 排序
age 相同，再按 city 排序
```

所以这个联合索引的逻辑顺序可以理解为：

```text
name    age    city      id
Alice   18     Beijing   1
Alice   18     Paris     2
Alice   20     London    3
Bob     18     Paris     4
Bob     22     Berlin    5
Cindy   20     Rome      6
David   25     Paris     7
```

也就是说，联合索引 `(name, age, city)` 本质上是按照字典序排列的：

```text
(name, age, city)
```

这就像英文词典：

```text
先按第一个字母排
第一个字母相同，再按第二个字母排
第二个字母相同，再按第三个字母排
```

---

# 2. 什么是最左前缀？

对于联合索引：

```sql
(name, age, city)
```

它可以支持这些前缀：

```text
name
name, age
name, age, city
```

也就是说，它可以高效支持：

```sql
WHERE name = 'Alice';
```

```sql
WHERE name = 'Alice' AND age = 18;
```

```sql
WHERE name = 'Alice' AND age = 18 AND city = 'Paris';
```

因为这些查询都是从最左边的 `name` 开始，并且没有跳过中间字段。

但是它通常不能很好支持：

```sql
WHERE age = 18;
```

或者：

```sql
WHERE city = 'Paris';
```

或者：

```sql
WHERE age = 18 AND city = 'Paris';
```

因为这些条件没有从联合索引最左边的 `name` 开始。

---

# 3. 为什么必须从最左边开始？

因为联合索引的排序方式决定了它的查找能力。

联合索引 `(name, age, city)` 的排序是：

```text
先 name
再 age
再 city
```

所以在这棵 B+Tree 里，`age` 不是全局有序的，`city` 也不是全局有序的。

看这个索引数据：

```text
name    age    city
Alice   18     Beijing
Alice   18     Paris
Alice   20     London
Bob     18     Paris
Bob     22     Berlin
Cindy   20     Rome
David   25     Paris
```

你看 `age` 这一列：

```text
18
18
20
18
22
20
25
```

它不是整体有序的。

因为索引首先按 `name` 排序，只有在 `name` 相同的情况下，`age` 才是有序的。

所以如果你查询：

```sql
WHERE age = 18
```

MySQL 没办法直接在 `(name, age, city)` 这棵索引树里快速定位所有 `age = 18` 的记录。因为 `age = 18` 分散在不同 `name` 分组下面：

```text
Alice, 18
Bob,   18
```

所以最左列 `name` 非常关键。

---

# 4. 具体走一遍：能用完整联合索引的情况

现在执行：

```sql
SELECT *
FROM user
WHERE name = 'Alice'
  AND age = 18
  AND city = 'Paris';
```

索引是：

```sql
idx_name_age_city(name, age, city)
```

因为查询条件完整匹配了：

```text
name -> age -> city
```

所以 MySQL 可以高效使用整个联合索引。

查找过程大致是：

```text
第一步：在 idx_name_age_city 这棵 B+Tree 中查 name = 'Alice'

第二步：在 name = 'Alice' 的范围内继续查 age = 18

第三步：在 name = 'Alice' 且 age = 18 的范围内继续查 city = 'Paris'

第四步：找到对应的二级索引记录

第五步：如果 SELECT * 需要完整行，就根据二级索引叶子节点里的主键 id 回表
```

图示：

```text
联合索引 idx_name_age_city：

name    age    city      id
Alice   18     Beijing   1
Alice   18     Paris     2   ← 命中
Alice   20     London    3
Bob     18     Paris     4
Bob     22     Berlin    5
Cindy   20     Rome      6
```

对于这条 SQL：

```sql
WHERE name = 'Alice'
  AND age = 18
  AND city = 'Paris'
```

可以理解为在索引中定位这个三元组：

```text
('Alice', 18, 'Paris')
```

这是最理想的情况。

---

# 5. 只用最左一列的情况

执行：

```sql
SELECT *
FROM user
WHERE name = 'Alice';
```

这也能使用联合索引。

因为 `(name, age, city)` 的最左前缀包含 `name`。

查找过程：

```text
第一步：在索引里定位 name = 'Alice' 的起点

第二步：沿着叶子节点向后扫描所有 name = 'Alice' 的记录

第三步：得到这些 id

第四步：如果需要完整行，再回表
```

索引中：

```text
name    age    city      id
Alice   18     Beijing   1   ← 命中
Alice   18     Paris     2   ← 命中
Alice   20     London    3   ← 命中
Bob     18     Paris     4
Bob     22     Berlin    5
```

它可以直接找到 `Alice` 这一段，因为 `name` 是联合索引的第一列。

所以 `(name, age, city)` 可以当作一个单列索引 `name` 来用。

---

# 6. 使用前两列的情况

执行：

```sql
SELECT *
FROM user
WHERE name = 'Alice'
  AND age = 18;
```

这也能使用联合索引的前两列：

```text
name, age
```

过程是：

```text
第一步：定位 name = 'Alice'

第二步：在 Alice 这一段里定位 age = 18

第三步：找到所有满足 name = 'Alice' AND age = 18 的记录

第四步：如果 SELECT *，再回表
```

索引中：

```text
name    age    city      id
Alice   18     Beijing   1   ← 命中
Alice   18     Paris     2   ← 命中
Alice   20     London    3
Bob     18     Paris     4
```

因为在 `name = Alice` 的范围内，`age` 是有序的，所以可以继续用 `age` 做索引定位。

---

# 7. 跳过中间列会怎样？

执行：

```sql
SELECT *
FROM user
WHERE name = 'Alice'
  AND city = 'Paris';
```

索引是：

```sql
(name, age, city)
```

这条 SQL 有 `name` 和 `city`，但跳过了中间的 `age`。

这时候 MySQL 通常只能高效使用 `name` 这一列来缩小范围，不能像完整匹配一样直接用 `city` 定位。

为什么？

因为索引排序是：

```text
name -> age -> city
```

在 `name = Alice` 的范围内，记录是先按 `age` 排序，再按 `city` 排序：

```text
name    age    city
Alice   18     Beijing
Alice   18     Paris
Alice   20     London
Alice   22     Paris
Alice   30     Rome
```

如果不知道 `age`，那么 `city = Paris` 在 `Alice` 分组里不一定连续。

所以执行逻辑大概是：

```text
第一步：用索引定位 name = 'Alice' 这一段

第二步：扫描 Alice 这一段中的多条记录

第三步：逐条判断 city 是否等于 Paris
```

也就是说：

```text
name 可以用于索引定位
city 可能只能作为过滤条件
```

这就是“跳过中间列，后面的列通常不能继续用于精确定位”。

---

# 8. 完全不使用最左列会怎样？

执行：

```sql
SELECT *
FROM user
WHERE age = 18;
```

索引是：

```sql
(name, age, city)
```

这条 SQL 没有 `name`，只有 `age`。

由于 `age` 不是联合索引的最左列，所以这条 SQL 通常不能有效利用这个联合索引进行快速定位。

原因是索引数据是这样排的：

```text
name    age
Alice   18
Alice   20
Bob     18
Bob     22
Cindy   18
David   30
```

`age = 18` 分散在不同 `name` 下面：

```text
Alice 18
Bob   18
Cindy 18
```

如果没有 `name`，MySQL 很难直接找到所有 `age = 18` 的连续区间。

所以如果你经常查询：

```sql
WHERE age = 18
```

应该单独给 `age` 建索引：

```sql
INDEX idx_age(age)
```

或者根据业务查询模式重新设计联合索引：

```sql
INDEX idx_age_name(age, name)
```

---

# 9. 范围查询会影响后续列使用

这是最左前缀原则中非常重要的一点。

假设索引还是：

```sql
INDEX idx_name_age_city(name, age, city)
```

执行：

```sql
SELECT *
FROM user
WHERE name = 'Alice'
  AND age > 18
  AND city = 'Paris';
```

很多初学者会以为这条 SQL 可以完整用到：

```text
name, age, city
```

但实际上要更细。

因为 `age > 18` 是范围查询。

一般情况下，联合索引在遇到范围条件后，后面的列就很难继续用于精确定位。

也就是说：

```text
name 可以用
age 可以用作范围扫描
city 通常不能继续用于缩小索引定位范围
```

执行过程类似：

```text
第一步：定位 name = 'Alice'

第二步：在 Alice 范围内找到 age > 18 的起点

第三步：向后扫描所有 age > 18 的索引记录

第四步：扫描过程中再判断 city 是否等于 Paris
```

索引中：

```text
name    age    city
Alice   18     Beijing
Alice   18     Paris
Alice   20     London
Alice   22     Paris
Alice   30     Rome
Bob     18     Paris
```

对于：

```sql
WHERE name = 'Alice'
  AND age > 18
  AND city = 'Paris'
```

它会先锁定：

```text
Alice 且 age > 18
```

这段范围：

```text
Alice   20   London
Alice   22   Paris    ← city 满足
Alice   30   Rome
```

然后再过滤 `city = 'Paris'`。

所以记住：

> 联合索引中，如果前面字段使用了范围查询，后面的字段通常不能继续用于索引的精确定位。

常见范围条件包括：

```sql
>
<
>=
<=
BETWEEN
LIKE 'abc%'
```

其中 `LIKE 'abc%'` 本质上也是范围查询。

---

# 10. 等值查询顺序和索引顺序不是一回事

比如索引是：

```sql
INDEX idx_name_age_city(name, age, city)
```

执行：

```sql
SELECT *
FROM user
WHERE age = 18
  AND city = 'Paris'
  AND name = 'Alice';
```

虽然 SQL 里条件顺序写成了：

```text
age -> city -> name
```

但是 MySQL 优化器会重排条件。

这条 SQL 依然可以使用：

```text
name, age, city
```

因为逻辑上它包含了：

```text
name = 'Alice'
age = 18
city = 'Paris'
```

也就是说，最左前缀原则关注的是**联合索引定义中的字段顺序**，不是你 SQL 里 WHERE 条件的书写顺序。

下面两条 SQL 在索引使用上通常等价：

```sql
WHERE name = 'Alice'
  AND age = 18
  AND city = 'Paris';
```

```sql
WHERE city = 'Paris'
  AND age = 18
  AND name = 'Alice';
```

关键是索引定义：

```sql
(name, age, city)
```

而不是 WHERE 里面谁写在前面。

---

# 11. ORDER BY 也受最左前缀影响

最左前缀不仅影响 `WHERE`，也影响 `ORDER BY`。

索引：

```sql
INDEX idx_name_age_city(name, age, city)
```

这条 SQL 可以较好利用索引排序：

```sql
SELECT *
FROM user
WHERE name = 'Alice'
ORDER BY age, city;
```

因为在 `name = Alice` 的范围内，索引天然按照：

```text
age, city
```

排序。

但这条就不一定好：

```sql
SELECT *
FROM user
WHERE name = 'Alice'
ORDER BY city;
```

因为索引在 `name = Alice` 内部的排序是：

```text
age -> city
```

不是单独按 `city` 排序。

所以可能需要额外排序，也就是 `filesort`。

---

# 12. 用一个完整流程带你走一遍

我们现在用这个联合索引：

```sql
INDEX idx_name_age_city(name, age, city)
```

查询：

```sql
SELECT *
FROM user
WHERE name = 'Alice'
  AND age = 18
  AND city = 'Paris';
```

## 第一步：理解索引排序

联合索引叶子节点大概是：

```text
name    age    city      id
Alice   18     Beijing   1
Alice   18     Paris     2
Alice   20     London    3
Bob     18     Paris     4
Bob     22     Berlin    5
Cindy   20     Rome      6
```

它按 `(name, age, city)` 排序。

---

## 第二步：从最左列 name 开始定位

MySQL 先找：

```text
name = Alice
```

这样可以定位到这一段：

```text
Alice   18     Beijing   1
Alice   18     Paris     2
Alice   20     London    3
```

---

## 第三步：继续用 age 缩小范围

在 `Alice` 这一段中，再找：

```text
age = 18
```

范围缩小为：

```text
Alice   18     Beijing   1
Alice   18     Paris     2
```

---

## 第四步：继续用 city 精确定位

在 `Alice + age=18` 这一段中，再找：

```text
city = Paris
```

最后命中：

```text
Alice   18     Paris     2
```

---

## 第五步：判断是否需要回表

因为 SQL 是：

```sql
SELECT *
```

联合索引叶子节点里只有：

```text
name, age, city, 主键 id
```

如果表里还有其他字段，比如 `gender`、`email`、`created_at`，那就需要根据 `id = 2` 回到聚簇索引查完整行。

流程是：

```text
idx_name_age_city
      |
      v
找到 name='Alice', age=18, city='Paris' 对应的 id=2
      |
      v
PRIMARY(id)
      |
      v
查完整行
```

如果 SQL 是：

```sql
SELECT id, name, age, city
FROM user
WHERE name = 'Alice'
  AND age = 18
  AND city = 'Paris';
```

那就不需要回表，因为这几个字段都在联合索引里。

这就是覆盖索引。

---

# 13. 常见情况总结

假设联合索引是：

```sql
INDEX idx_abc(a, b, c)
```

下面是各种查询是否能较好使用索引：

```text
查询条件                         索引使用情况
---------------------------------------------------------
WHERE a = 1                      可以用 a

WHERE a = 1 AND b = 2            可以用 a, b

WHERE a = 1 AND b = 2 AND c = 3  可以用 a, b, c

WHERE b = 2                      通常不能有效用 idx_abc

WHERE c = 3                      通常不能有效用 idx_abc

WHERE b = 2 AND c = 3            通常不能有效用 idx_abc

WHERE a = 1 AND c = 3            通常只能用 a，c 用作过滤

WHERE a = 1 AND b > 2 AND c = 3  通常用 a, b 范围，c 用作过滤

WHERE a > 1 AND b = 2            通常用 a 范围，b 难以继续精确定位
```

---

# 14. 记忆口诀

可以这样记：

```text
从左到右，连续匹配；
中间不能断；
遇到范围，后面变弱；
等值顺序可调整；
排序也要看索引顺序。
```

更口语一点：

> 联合索引就像电话簿，先按姓排，再按名排。你知道姓，可以快速找；你知道姓和名，更快；但你只知道名，不知道姓，就很难直接定位。

---

# 15. 最核心总结

**最左前缀原则指的是：对于联合索引 `(a, b, c)`，MySQL 使用索引时通常要从最左列 `a` 开始，按顺序连续使用 `a -> b -> c`，不能跳过中间列。原因是联合索引在 B+Tree 中是按 `(a, b, c)` 的字典序排列的，只有在前一列确定的情况下，后一列才是有序可定位的。**

所以：

```sql
INDEX(a, b, c)
```

可以高效支持：

```sql
WHERE a = ?
WHERE a = ? AND b = ?
WHERE a = ? AND b = ? AND c = ?
```

但不适合单独支持：

```sql
WHERE b = ?
WHERE c = ?
WHERE b = ? AND c = ?
```

除非 MySQL 使用特殊优化，或者优化器认为扫描索引比扫描全表更划算，但这已经不是典型意义上的“完整命中最左前缀”了。



## InnoDB的读写机制
⛓️ 需要满足四个原则：A（原子性) C(一致性) I(隔离性) D(持久性)
                       ⬇️
     不满足的时候可能出现的问题：幻读，脏读，不可重复读
                       ⬇️
     采用事务级隔离实现：MySQL/InnoDB 常说的四个事务隔离级别是：
```
| 隔离级别               | 中文名  | 解决的问题         | 仍可能出现的问题                                        |
| ------------------ | ---- | ------------- | ----------------------------------------------- |
| `READ UNCOMMITTED` | 读未提交 | 基本不解决         | 脏读、不可重复读、幻读                                     |
| `READ COMMITTED`   | 读已提交 | 解决脏读          | 不可重复读、幻读                                        |
| `REPEATABLE READ`  | 可重复读 | 解决脏读、不可重复读    | 标准 SQL 中可能幻读；InnoDB 中通过 MVCC/Next-Key Lock 做了处理 |
| `SERIALIZABLE`     | 串行化  | 解决脏读、不可重复读、幻读 | 并发性能最低                                          |
```
                       ⬇️
事务级别隔离的设置用户可以自己设置，底层实现依赖InnoDB的以下机制：***MVCC，Read view，Undo log，next-key lock***这个是SQL自动实现的


你这个问题问到核心了。你现在的理解有一半是对的，但有一个关键误区：

> **Read View 不是数据本身的快照，不是把表复制了一份。**
> **Read View 是一个“可见性判断规则”。它告诉当前事务：哪些事务的修改我能看见，哪些事务的修改我不能看见。**

真正保存“数据不同形态”的，是：

```text
Undo Log 版本链
```

而不是 Read View。

所以三者关系是：

```text
Undo Log：保存历史版本
MVCC：让一行数据可以有多个版本，并根据规则读取合适版本
Read View：判断当前事务应该看到哪个版本
```

你可以先记住这一句话：

> **MVCC = 多版本数据 + 可见性判断。Undo Log 提供多版本，Read View 提供可见性判断。**

---

# 1. 先不用数据库术语，先用一个生活类比

假设有一个文档：

```text
余额 = 1000
```

后来有人修改成：

```text
余额 = 800
```

又有人修改成：

```text
余额 = 600
```

如果数据库只保留最新值，那么所有事务都只能看到：

```text
余额 = 600
```

但是事务隔离要求不同事务可能看到不同时间点的数据。

比如事务 A 是很早开始的，它应该看到：

```text
余额 = 1000
```

事务 B 是后来开始的，它应该看到：

```text
余额 = 800
```

事务 C 是最新开始的，它应该看到：

```text
余额 = 600
```

这就需要数据库同时保留多个版本：

```text
最新版本：余额 = 600
上一个版本：余额 = 800
再上一个版本：余额 = 1000
```

这就是 **MVCC，多版本并发控制**。

---

# 2. MVCC 到底做了什么？

MVCC 全称是：

```text
Multi-Version Concurrency Control
多版本并发控制
```

它解决的问题是：

> 在并发场景下，读操作不一定要等写操作完成，写操作也不一定要阻塞普通读操作。

没有 MVCC 的话，可能是这样：

```text
事务 T1 正在修改一行数据
事务 T2 想读这行数据
T2 必须等待 T1 提交或回滚
```

这会导致并发性能很差。

有了 MVCC 后，可以这样：

```text
T1 正在修改最新版本
T2 如果不应该看到 T1 的修改，就去读旧版本
```

所以 MVCC 的核心价值是：

```text
读写并发
读不阻塞写
写不阻塞普通快照读
```

注意，我说的是**普通快照读**，不是 `SELECT ... FOR UPDATE` 这种当前读。

---

# 3. 一行数据在 InnoDB 中不是只有你看到的字段

假设你建表：

```sql
CREATE TABLE account (
    id INT PRIMARY KEY,
    balance INT
) ENGINE=InnoDB;
```

你看到的是：

```text
id    balance
1     1000
```

但 InnoDB 内部还会给每一行加隐藏字段。你现在重点理解两个：

```text
DB_TRX_ID    最近一次修改这行的事务 id
DB_ROLL_PTR  指向 Undo Log 中旧版本的指针
```

所以真实结构可以粗略理解成：

```text
id    balance    DB_TRX_ID    DB_ROLL_PTR
1     1000       10           null
```

含义是：

```text
这行当前版本是事务 10 创建或最后修改的
它没有更旧版本
```

---

# 4. 更新一行时，MVCC 发生了什么？

初始状态：

```text
id=1, balance=1000, DB_TRX_ID=10
```

事务 T20 执行：

```sql
BEGIN;

UPDATE account
SET balance = 800
WHERE id = 1;
```

InnoDB 不会简单粗暴地只把 1000 改成 800。

它会做两件事：

## 第一步：把旧版本写入 Undo Log

```text
Undo Log 中保存旧版本：
id=1, balance=1000, DB_TRX_ID=10
```

## 第二步：当前行变成新版本

```text
当前行：
id=1, balance=800, DB_TRX_ID=20, DB_ROLL_PTR -> 旧版本
```

于是这行数据形成了一个版本链：

```text
当前版本：
balance = 800, DB_TRX_ID = 20
        |
        | DB_ROLL_PTR
        v
旧版本：
balance = 1000, DB_TRX_ID = 10
```

如果后面又有事务 T30 修改：

```sql
UPDATE account
SET balance = 600
WHERE id = 1;
```

版本链就变成：

```text
当前版本：
balance = 600, DB_TRX_ID = 30
        |
        v
旧版本：
balance = 800, DB_TRX_ID = 20
        |
        v
更旧版本：
balance = 1000, DB_TRX_ID = 10
```

这条链就是 MVCC 能够读取历史版本的基础。

---

# 5. 那 Read View 是什么？

现在关键来了。

假设一行数据有多个版本：

```text
balance = 600, 由事务 30 修改
balance = 800, 由事务 20 修改
balance = 1000, 由事务 10 修改
```

当事务 T100 来读这行数据时，它要回答一个问题：

> 我到底应该看到哪个版本？

这时候就需要 Read View。

Read View 可以理解为：

```text
当前事务读数据时生成的一份“事务可见性名单”
```

它不是数据副本，而是类似这样一份判断规则：

```text
在我生成 Read View 的那一刻：

哪些事务已经提交？
哪些事务还没提交？
哪些事务是在我之后才开始的？
```

然后根据这些信息判断：

```text
版本 30 对我可见吗？
版本 20 对我可见吗？
版本 10 对我可见吗？
```

如果当前最新版本不可见，就沿着 Undo Log 往旧版本找。

---

# 6. 你的理解应该改成这样

你原来的理解是：

> Read View 是一个事务对数据操作过程中，对数据不同形态的快照，这个快照对其他事务专属可见。

更准确的说法应该是：

> **Read View 是某个事务在进行快照读时生成的“可见性视图”。它不是保存数据不同形态的快照，而是保存当前活跃事务等信息，用来判断 Undo Log 版本链中的哪些数据版本对当前事务可见。**

也就是说：

```text
数据不同形态：由 Undo Log 版本链保存
能不能看到某个形态：由 Read View 判断
整个机制：叫 MVCC
```

---

# 7. 用一个具体事务例子完整走一遍

假设初始数据：

```text
account
id    balance
1     1000
```

内部版本：

```text
balance=1000, DB_TRX_ID=10
```

现在有两个事务。

---

## 7.1 事务 T100 开始读取

事务 T100：

```sql
BEGIN;

SELECT balance
FROM account
WHERE id = 1;
```

假设它读到：

```text
1000
```

在 `REPEATABLE READ` 下，这次普通 `SELECT` 会生成一个 Read View。

你可以理解为：

```text
T100 的 Read View：
在我读数据这一刻，哪些事务对我可见，哪些不可见。
```

假设此时还没有其他事务修改，所以它看到版本：

```text
balance=1000, DB_TRX_ID=10
```

---

## 7.2 事务 T200 修改数据并提交

另一个事务 T200：

```sql
BEGIN;

UPDATE account
SET balance = 800
WHERE id = 1;

COMMIT;
```

现在版本链变成：

```text
当前版本：
balance=800, DB_TRX_ID=200
        |
        v
旧版本：
balance=1000, DB_TRX_ID=10
```

---

## 7.3 T100 再次读取

事务 T100 再次执行：

```sql
SELECT balance
FROM account
WHERE id = 1;
```

你可能以为最新值已经是 800，所以 T100 应该看到 800。

但在 `REPEATABLE READ` 下，T100 仍然看到：

```text
1000
```

为什么？

因为 T100 复用第一次 SELECT 时生成的 Read View。

这个 Read View 会判断：

```text
balance=800 这个版本是事务 200 修改的
事务 200 是在我的 Read View 之后提交的
所以对我不可见
```

于是 InnoDB 沿着 `DB_ROLL_PTR` 去 Undo Log 找旧版本：

```text
balance=1000
```

这个旧版本对 T100 可见，于是返回 1000。

完整过程是：

```text
T100 读取当前行
    |
    v
看到当前版本 balance=800, DB_TRX_ID=200
    |
    v
用 Read View 判断：T200 对我不可见
    |
    v
沿 Undo Log 找旧版本
    |
    v
找到 balance=1000, DB_TRX_ID=10
    |
    v
判断可见
    |
    v
返回 1000
```

这就是 MVCC 的工作过程。

---

# 8. 所以 MVCC 不是“复制一份表”

这是很多人初学时最大的误区。

MVCC 不是这样：

```text
事务 T100 开始时，数据库复制一份 account 表给它
```

如果真这么做，成本太高了。

MVCC 实际是这样：

```text
表里保存最新版本
旧版本放在 Undo Log 里
每个读事务用 Read View 判断该看哪个版本
```

所以它很像：

```text
当前数据 + 历史版本链 + 可见性规则
```

---

# 9. Read View 里面到底有什么？

你不需要一开始背所有字段，但为了真正理解，我给你讲清楚。

一个 Read View 里大致关心这些信息：

```text
creator_trx_id：创建这个 Read View 的事务 id

m_ids：创建 Read View 时，系统中还活跃的事务 id 列表

min_trx_id：活跃事务列表中的最小事务 id

max_trx_id：创建 Read View 时，系统将要分配给下一个事务的 id
```

你可以理解为：

```text
m_ids：我拍照时还没提交的事务名单
min_trx_id：这些未提交事务中最小的 id
max_trx_id：在我拍照之后才会出现的事务 id 边界
```

Read View 的作用是判断某个版本的 `DB_TRX_ID` 是否可见。

---

# 10. Read View 怎么判断版本是否可见？

假设一个数据版本的修改事务 id 是 `trx_id`。

Read View 判断规则可以简化为：

## 情况一：这个版本是我自己改的

```text
trx_id == creator_trx_id
```

可见。

因为事务当然能看到自己修改的数据。

例如：

```sql
BEGIN;

UPDATE account SET balance = 900 WHERE id = 1;

SELECT balance FROM account WHERE id = 1;
```

自己改成 900，自己当然能读到 900。

---

## 情况二：这个版本在 Read View 生成前已经提交

```text
trx_id < min_trx_id
```

通常可见。

因为这个版本很早之前就存在了，在我生成 Read View 的时候它已经是稳定数据。

---

## 情况三：这个版本是 Read View 之后才出现的

```text
trx_id >= max_trx_id
```

不可见。

因为这是“未来事务”产生的版本。

---

## 情况四：这个版本的事务在 m_ids 活跃列表中

```text
trx_id in m_ids
```

不可见。

因为在我生成 Read View 的那一刻，这个事务还没提交。

---

## 情况五：不在活跃列表中，并且不是未来事务

```text
min_trx_id <= trx_id < max_trx_id
且 trx_id 不在 m_ids 中
```

通常可见。

因为它说明这个事务在我生成 Read View 时已经提交了。

---

# 11. 用具体数字看 Read View

假设当前系统里有这些事务：

```text
事务 100：正在执行
事务 200：正在执行
事务 300：正在执行
```

现在事务 150 执行普通 SELECT，生成 Read View：

```text
creator_trx_id = 150
m_ids = [100, 200, 300]
min_trx_id = 100
max_trx_id = 301
```

现在它读取某一行，看到版本链：

```text
版本 A：balance=600, DB_TRX_ID=300
版本 B：balance=800, DB_TRX_ID=200
版本 C：balance=1000, DB_TRX_ID=90
```

判断过程：

## 看版本 A

```text
DB_TRX_ID = 300
300 在 m_ids 中
说明事务 300 在我生成 Read View 时还没提交
所以不可见
```

继续找旧版本。

## 看版本 B

```text
DB_TRX_ID = 200
200 在 m_ids 中
说明事务 200 在我生成 Read View 时还没提交
所以不可见
```

继续找旧版本。

## 看版本 C

```text
DB_TRX_ID = 90
90 < min_trx_id 100
说明它在我生成 Read View 前已经提交
所以可见
```

于是最终返回：

```text
balance = 1000
```

这就是 Read View + Undo Log 版本链的完整判断流程。

---

# 12. RC 和 RR 的区别，其实就是 Read View 什么时候生成

这是你理解隔离级别的关键。

## READ COMMITTED

```text
每条 SELECT 都生成新的 Read View
```

也就是说：

```sql
BEGIN;

SELECT ...;  -- Read View 1

SELECT ...;  -- Read View 2

SELECT ...;  -- Read View 3
```

每次查询都看“这条语句开始时已经提交的数据”。

所以在 RC 下，如果另一个事务在两次 SELECT 中间提交了修改，第二次 SELECT 就能看到。

因此：

```text
RC 可以防止脏读
但可能出现不可重复读
```

---

## REPEATABLE READ

```text
一个事务内第一次普通 SELECT 生成 Read View
后续普通 SELECT 复用同一个 Read View
```

也就是说：

```sql
BEGIN;

SELECT ...;  -- 生成 Read View 1

SELECT ...;  -- 继续使用 Read View 1

SELECT ...;  -- 继续使用 Read View 1
```

所以在 RR 下，即使其他事务提交了修改，当前事务后续普通 SELECT 仍然看旧快照。

因此：

```text
RR 可以防止不可重复读
```

---

# 13. 具体对比：RC 和 RR 为什么结果不同？

初始：

```text
id=1, balance=1000
```

---

## 13.1 READ COMMITTED

事务 T1：

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;

SELECT balance FROM account WHERE id = 1;
```

第一次 SELECT 生成 Read View A，返回：

```text
1000
```

事务 T2：

```sql
BEGIN;

UPDATE account SET balance = 800 WHERE id = 1;

COMMIT;
```

此时版本链：

```text
balance=800, DB_TRX_ID=T2
        |
        v
balance=1000
```

事务 T1 第二次读：

```sql
SELECT balance FROM account WHERE id = 1;
```

RC 下第二次 SELECT 生成新的 Read View B。

这个新 Read View 认为：

```text
T2 已经提交
balance=800 对我可见
```

所以返回：

```text
800
```

结果：

```text
第一次读：1000
第二次读：800
```

这就是不可重复读。

---

## 13.2 REPEATABLE READ

事务 T1：

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;

SELECT balance FROM account WHERE id = 1;
```

第一次 SELECT 生成 Read View A，返回：

```text
1000
```

事务 T2：

```sql
BEGIN;

UPDATE account SET balance = 800 WHERE id = 1;

COMMIT;
```

事务 T1 第二次读：

```sql
SELECT balance FROM account WHERE id = 1;
```

RR 下第二次 SELECT 继续使用 Read View A。

Read View A 认为：

```text
T2 的修改对我不可见
```

于是沿 Undo Log 找旧版本：

```text
1000
```

结果：

```text
第一次读：1000
第二次读：1000
```

这就是可重复读。

---

# 14. Read View 是“专属可见”吗？

你说：

> 这个快照对其他事务专属可见。

这里需要修正一下。

更准确是：

> **Read View 是当前事务自己用来判断数据版本可见性的规则。它不是让某份数据“对其他事务专属可见”，而是让当前事务决定自己能看到哪些版本。**

也就是说：

```text
T1 有 T1 的 Read View
T2 有 T2 的 Read View
T3 有 T3 的 Read View
```

同一个数据版本：

```text
balance=800
```

可能对 T1 不可见，但对 T2 可见。

例如：

```text
T1 的 Read View 很早创建，所以看不到 balance=800
T2 的 Read View 较晚创建，所以能看到 balance=800
```

所以不是数据版本“属于某个事务”，而是：

```text
每个事务用自己的 Read View 判断某个版本是否可见
```

---

# 15. MVCC 的完整执行流程

现在把整个流程总结成一个算法。

普通 SELECT 时，InnoDB 大概做：

```text
1. 找到这行数据的最新版本。

2. 查看这个版本的 DB_TRX_ID。

3. 用当前事务的 Read View 判断这个 DB_TRX_ID 是否可见。

4. 如果可见：
       返回这个版本。

5. 如果不可见：
       通过 DB_ROLL_PTR 找到 Undo Log 中的上一个版本。

6. 继续判断上一个版本是否可见。

7. 一直向前找，直到找到一个可见版本，或者找不到。
```

可以写成伪代码：

```python
def consistent_read(row, read_view):
    version = row.latest_version

    while version is not None:
        trx_id = version.DB_TRX_ID

        if read_view.is_visible(trx_id):
            return version.data

        version = version.undo_log_previous_version

    return None
```

这就是 MVCC 的核心。

---

# 16. 为什么 MVCC 可以让读不阻塞写？

假设 T2 正在修改：

```sql
UPDATE account SET balance = 800 WHERE id = 1;
```

这时最新版本是：

```text
balance=800
```

但还没提交。

T1 普通 SELECT 来读。

如果没有 MVCC，T1 可能要等 T2 提交或回滚。

有了 MVCC，T1 可以判断：

```text
T2 未提交，对我不可见
```

然后沿 Undo Log 读旧版本：

```text
balance=1000
```

所以 T1 不需要等 T2。

这就是：

```text
读不阻塞写
```

反过来，T1 普通 SELECT 也通常不会阻塞 T2 的 UPDATE。

这就是：

```text
写不阻塞普通读
```

当然，如果是：

```sql
SELECT ... FOR UPDATE;
```

那就是当前读，会加锁，不是普通 MVCC 快照读了。

---

# 17. MVCC 能不能解决所有并发问题？

不能。

MVCC 主要解决的是：

```text
普通 SELECT 的一致性读
```

也就是快照读。

但这些操作依然需要锁：

```sql
UPDATE ...
DELETE ...
SELECT ... FOR UPDATE
INSERT ...
```

因为它们要操作当前最新数据。

所以 InnoDB 的隔离不是只靠 MVCC，而是：

```text
快照读：MVCC + Undo Log + Read View
当前读：锁机制
```

---

# 18. 当前读为什么不用普通 Read View？

比如：

```sql
SELECT *
FROM account
WHERE id = 1
FOR UPDATE;
```

这句话的意思不是“给我一个历史快照”。

它的意思是：

> 我要拿到当前最新的数据，并且我要锁住它，准备修改。

所以它不能只用旧版本。

如果它读旧版本并加锁，就没有意义了。因为你要保护的是当前真实数据。

所以当前读要读当前最新已提交版本，并加锁。

这就是为什么在 RR 下可能出现：

```sql
BEGIN;

SELECT balance FROM account WHERE id = 1;
-- 普通 SELECT，看到 1000

-- 另一个事务把 balance 改成 800 并提交

SELECT balance FROM account WHERE id = 1 FOR UPDATE;
-- 当前读，可能看到 800
```

这不是矛盾，而是因为：

```text
普通 SELECT：快照读，看 Read View
FOR UPDATE：当前读，看最新并加锁
```

---

# 19. 最容易混淆的三个概念

## 19.1 Undo Log 不是 Read View

Undo Log 是历史版本数据。

```text
balance=1000
balance=800
balance=600
```

这些旧版本靠 Undo Log 保存。

---

## 19.2 Read View 不是历史数据

Read View 是判断规则。

它只回答：

```text
版本 600 可不可见？
版本 800 可不可见？
版本 1000 可不可见？
```

---

## 19.3 MVCC 是整个机制

MVCC 包含：

```text
隐藏事务字段
Undo Log 版本链
Read View 可见性判断
一致性读流程
```

所以：

```text
Undo Log + Read View + 隐藏字段 + 读取算法 = MVCC
```

---

# 20. 最后用一句话把它彻底串起来

假设当前版本链是：

```text
balance=600, trx_id=300
        |
        v
balance=800, trx_id=200
        |
        v
balance=1000, trx_id=100
```

某个事务 T150 有一个 Read View：

```text
我只能看到 trx_id <= 150 之前已经提交的版本
```

那么它读数据时：

```text
看 balance=600，trx_id=300，不可见
看 balance=800，trx_id=200，不可见
看 balance=1000，trx_id=100，可见
返回 balance=1000
```

这就是 MVCC。

---

# 21. 你现在应该这样理解

你可以把自己的理解修正成这版：

> **InnoDB 中，每次更新数据时不会简单覆盖旧数据，而是通过 Undo Log 保留旧版本，并在当前记录中用隐藏字段指向旧版本，形成版本链。MVCC 的作用就是让普通 SELECT 在不加锁的情况下，根据当前事务的 Read View 判断版本链中哪个版本对自己可见。Read View 不是数据快照，而是事务在某个时刻生成的可见性快照；RC 下每条 SELECT 生成一个新的 Read View，RR 下一个事务内复用同一个 Read View，所以 RR 能实现可重复读。**

最短总结：

```text
Undo Log 保存旧版本；
Read View 判断版本可见性；
MVCC 让事务读取“自己该看到的版本”，而不是永远读取最新版本。
```
对，你这个理解已经**非常准确**了。可以再稍微修正成更标准的表述：

> **MVCC 是 InnoDB 为数据维护多个历史版本的机制。每次事务修改一行数据时，InnoDB 会通过隐藏字段和 Undo Log 形成版本链，记录“这行数据由哪个事务修改、当前版本是什么、旧版本在哪里”。这样后续事务在读取时，可以根据自己的可见性规则选择读取某个历史版本；事务回滚时，也可以根据 Undo Log 恢复旧值。**

> **Read View 是事务在执行一致性读时创建的可见性视图。它不是数据副本，而是一套判断规则，用来告诉当前读事务：哪些事务的修改对我可见，哪些事务的修改对我不可见。**

你可以这样记：

```text
MVCC = 多版本机制
Undo Log = 保存旧版本
Read View = 判断哪个版本可见
```

---

# 1. 你的理解拆开看

你说：

> MVCC 是给每个数据创建改动操作版本链。

基本正确，但更严谨一点是：

> **不是每个数据天然都有很长的版本链，而是当数据被更新时，InnoDB 会通过 Undo Log 逐步形成版本链。**

比如最开始只有一行：

```text
id=1, balance=1000
```

事务 T10 修改：

```sql
UPDATE account SET balance = 800 WHERE id = 1;
```

InnoDB 内部会变成：

```text
当前版本：
balance=800, DB_TRX_ID=T10
        |
        v
Undo Log 旧版本：
balance=1000
```

事务 T20 再修改：

```sql
UPDATE account SET balance = 600 WHERE id = 1;
```

版本链变成：

```text
当前版本：
balance=600, DB_TRX_ID=T20
        |
        v
旧版本：
balance=800, DB_TRX_ID=T10
        |
        v
更旧版本：
balance=1000
```

所以你说的“哪个事务对其进行了修改、改成什么样子、指向旧数据的指针”是对的。

---

# 2. Undo Log 在这里具体负责什么？

Undo Log 主要负责两件事：

```text
1. 支持事务回滚
2. 支持 MVCC 读取历史版本
```

比如当前版本是：

```text
balance=600
```

如果事务 T20 回滚，就可以根据 Undo Log 把它恢复成：

```text
balance=800
```

如果另一个事务 T5 很早之前开始，它不应该看到 T10、T20 的修改，也可以通过 Undo Log 找到它应该看到的旧版本：

```text
balance=1000
```

所以 Undo Log 是 MVCC 的物质基础。

---

# 3. Read View 的理解也基本正确

你说：

> Read View 是读事务在对数据进行读取的时候根据自身情况创建的，它告诉这个读事务哪些改动是可以被它查看的。

这个说法也对，但可以更标准：

> **Read View 是事务执行快照读时生成的可见性判断规则。它记录生成 Read View 那一刻系统中事务的状态，例如哪些事务还活跃、哪些事务已经提交，从而判断某个数据版本是否对当前事务可见。**

它不保存：

```text
balance=1000
balance=800
balance=600
```

这些具体数据。

它保存的是类似这种判断依据：

```text
事务 T10 的修改可见
事务 T20 的修改不可见
事务 T30 是未来事务，不可见
```

然后 InnoDB 根据这个规则去版本链里找合适版本。

---

# 4. 用一个完整例子把你的理解确认一遍

初始数据：

```text
id=1, balance=1000
```

事务 T1 开始：

```sql
BEGIN;

SELECT balance FROM account WHERE id = 1;
```

假设在 RR 隔离级别下，T1 第一次普通 `SELECT` 创建 Read View。

T1 看到：

```text
balance=1000
```

然后事务 T2 修改并提交：

```sql
BEGIN;

UPDATE account SET balance = 800 WHERE id = 1;

COMMIT;
```

现在版本链是：

```text
当前版本：
balance=800, DB_TRX_ID=T2
        |
        v
旧版本：
balance=1000
```

T1 再次读取：

```sql
SELECT balance FROM account WHERE id = 1;
```

InnoDB 会这样做：

```text
1. 先看到当前最新版本 balance=800
2. 查看这个版本的 DB_TRX_ID = T2
3. 用 T1 的 Read View 判断 T2 是否可见
4. 发现 T2 的修改对 T1 不可见
5. 沿着 Undo Log 找旧版本
6. 找到 balance=1000
7. 判断旧版本可见
8. 返回 balance=1000
```

所以 T1 两次读取结果都是：

```text
1000
```

这就是 RR 下的可重复读。

---

# 5. MVCC 和 Read View 的分工

你可以用这个图记：

```text
一行数据的版本链：

最新版本：
balance=600, trx_id=30
        |
        v
旧版本：
balance=800, trx_id=20
        |
        v
更旧版本：
balance=1000, trx_id=10
```

Read View 做的事情是：

```text
trx_id=30 的版本，我能不能看？
不能。

trx_id=20 的版本，我能不能看？
不能。

trx_id=10 的版本，我能不能看？
能。

所以返回 balance=1000。
```

所以：

```text
Undo Log 负责“有哪些版本”
Read View 负责“哪个版本对我可见”
MVCC 负责“根据版本链和可见性规则完成一致性读取”
```

---

# 6. 还要补一个重要细节：自己修改的数据自己可见

如果一个事务自己修改了数据，即使按照 Read View 看起来这个修改发生在事务过程中，它自己也能看到。

例如：

```sql
BEGIN;

SELECT balance FROM account WHERE id = 1;
-- 看到 1000

UPDATE account SET balance = 900 WHERE id = 1;

SELECT balance FROM account WHERE id = 1;
-- 看到 900
```

因为：

```text
事务自己的修改，对自己永远可见。
```

所以 Read View 的规则不是简单的“事务开始后发生的都不可见”，还要加上：

```text
自己改的版本可见。
```

---

# 7. RC 和 RR 的差别还是 Read View 的创建时机

你的理解再往前推一步，就能理解 RC 和 RR。

## READ COMMITTED

```text
每条 SELECT 都创建新的 Read View
```

所以：

```text
第一次 SELECT 看一个快照
第二次 SELECT 看另一个新快照
```

如果中间有其他事务提交，第二次就能看到。

---

## REPEATABLE READ

```text
一个事务内第一次普通 SELECT 创建 Read View
后续普通 SELECT 复用这个 Read View
```

所以：

```text
第一次 SELECT 和第二次 SELECT 使用同一套可见性规则
```

结果就稳定了。

---

# 8. 最终标准表述

你可以把自己的话整理成下面这版：

> **MVCC 是 InnoDB 的多版本并发控制机制。每次事务修改数据时，InnoDB 会通过隐藏字段 `DB_TRX_ID` 记录修改该版本的事务 ID，通过 `DB_ROLL_PTR` 指向 Undo Log 中的旧版本，从而形成版本链。这个版本链既可以用于事务回滚，也可以用于普通 SELECT 的一致性读。Read View 是事务执行快照读时生成的可见性规则，它根据当前活跃事务列表等信息判断某个版本是否对当前事务可见。如果最新版本不可见，InnoDB 就沿着 Undo Log 版本链向前查找，直到找到一个可见版本。**

最短理解就是：

```text
MVCC 让一行数据拥有多个版本；
Undo Log 保存旧版本；
Read View 决定当前事务能看到哪个版本；
InnoDB 根据 Read View 在版本链中找到可见版本返回。
```



   








