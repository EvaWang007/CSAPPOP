# 👻Fucking Tech!!!
下面是按你一面暴露问题整理的**二面冲刺复习计划**。我把“git 的 lbase”按最可能的 **git rebase** 处理。

# 一、你的薄弱点分组

## A. C++ 核心底层

重点：

```text
1. 智能指针
2. 内存泄露
3. 线程池维护机制
```

这是最优先的部分，因为 C++ 客户端二面很容易继续深挖。

---

## B. 网络与系统

重点：

```text
1. DNS 解析
2. QUIC
3. HTTP/HTTPS/TLS
```

你已经学过 TCP/UDP、HTTP/HTTPS，接下来要把它们整理成面试表达。

---

## C. 数据库 / 索引

重点：

```text
1. 索引底层实现机制
2. 索引太多怎么办
```

面试官可能是从数据库性能角度问，也可能是在问数据结构中的索引思想。

---

## D. Python / Git 工具链

重点：

```text
1. Python 内存处理机制
2. git rebase
```

---

## E. AI Coding / Agent 工程

重点：

```text
1. 如何测评 Cursor 等大模型工作效果
2. Cursor / Codex / Claude Code 对比
3. Skill / MCP / Agent
```

这部分和你的 ROS Agent 项目强相关，要准备成项目亮点。

Cursor 官方强调 Agent 可以把想法转成代码，支持 Rules、Skills、MCP、hooks 等扩展能力；Codex 是 OpenAI 的 coding agent，可读写和运行代码，也支持 IDE 与云端任务；Claude Code 官方定位是能理解代码库、跨文件修改、运行测试并自动化开发任务的 agentic coding system。([Cursor][1])

---

# 二、三天复习计划

## Day 1：C++ 必杀区

### 上午：智能指针

必须掌握：

```text
unique_ptr
shared_ptr
weak_ptr
引用计数
循环引用
RAII
move 语义
make_shared
shared_ptr 线程安全边界
```

重点面试题：

```text
1. shared_ptr 的引用计数在哪里？
2. shared_ptr 是否线程安全？
3. weak_ptr 解决什么问题？
4. unique_ptr 为什么不能拷贝？
5. make_shared 和直接 new 的区别？
6. shared_ptr 循环引用怎么解决？
```

你要准备一个简化版手写 `shared_ptr`。

---

### 下午：C++ 内存泄露

必须掌握：

```text
1. new 后忘记 delete
2. 异常路径提前 return
3. 容器中保存裸指针
4. 循环引用
5. 线程未释放资源
6. 文件/socket fd 未关闭
7. 回调/lambda 捕获 shared_ptr 导致生命周期延长
```

检测工具：

```text
Valgrind
AddressSanitizer
LeakSanitizer
智能指针
RAII 封装
```

面试表达：

> C++ 内存泄露本质是对象生命周期管理失败。工程上用 RAII、智能指针、容器托管资源，配合 ASan/Valgrind 检测，并重点关注异常路径、循环引用、线程资源和 fd/socket 释放。

---

### 晚上：线程池维护机制

必须掌握：

```text
任务队列
worker 线程
mutex
condition_variable
stop flag
优雅退出
异常捕获
队列满后的拒绝策略
```

能画出：

```text
submit(task)
  ↓
push 到 queue
  ↓
notify_one
  ↓
worker 被唤醒
  ↓
pop task
  ↓
execute
```

必写伪代码：

```text
ThreadPool
- vector<thread> workers
- queue<function<void()>> tasks
- mutex
- condition_variable
- bool stop
```

---

## Day 2：网络 + 数据库索引 + Python/Git

### 上午：DNS + QUIC

DNS 你要能讲清：

```text
1. DNS 是应用层协议
2. 域名 → IP
3. 浏览器缓存 / 系统缓存 / hosts / 本地 DNS
4. 根 DNS / 顶级域 DNS / 权威 DNS
5. DNS 通常使用 UDP 53
6. ping 域名时先 DNS 解析，再 ICMP
```

QUIC 你要能讲清：

```text
QUIC = 基于 UDP 的安全可靠多路复用传输协议
HTTP/3 基于 QUIC
```

重点：

```text
1. 为什么基于 UDP？
2. 如何解决 TCP 队头阻塞？
3. 为什么支持连接迁移？
4. 和 TCP + TLS 的区别？
```

面试版：

> QUIC 基于 UDP，因为 UDP 足够简单，便于在用户态实现连接管理、TLS 加密、可靠传输、拥塞控制和多路复用。相比 TCP，QUIC 可以减少握手延迟，支持连接迁移，并缓解 HTTP/2 在 TCP 丢包时的队头阻塞问题。

---

### 下午：索引实现机制

这里重点准备数据库索引。

必须掌握：

```text
1. 索引是什么？
2. 为什么能加速查询？
3. B+ 树为什么适合数据库索引？
4. 哈希索引适合什么？
5. 聚簇索引 / 非聚簇索引
6. 联合索引
7. 最左前缀原则
8. 覆盖索引
9. 回表
10. 索引失效
```

“索引太多怎么办”要这样答：

```text
索引不是越多越好。
索引会提升查询速度，但会增加写入、更新、删除成本。
因为每次数据变化都要维护索引结构。
索引还会占用磁盘和内存。
所以要根据慢查询、查询频率、区分度、联合索引覆盖情况来优化。
删除低频、重复、低区分度索引。
```

面试版：

> 索引底层常见实现是 B+ 树。它通过有序结构减少磁盘 I/O，使查询从全表扫描变成树高范围内的查找。索引太多会增加写入成本、占用存储，并影响优化器选择，因此需要根据慢查询、字段区分度、查询模式和覆盖索引情况做取舍。

---

### 晚上：Python 内存机制 + Git rebase

Python 内存：

```text
1. 引用计数
2. 垃圾回收 GC
3. 循环引用
4. 分代回收
5. 小对象池
6. GIL 和内存管理关系
7. del 只是减少引用，不一定立即释放内存给操作系统
```

Git rebase：

```text
git rebase = 把当前分支的提交“搬到”另一个 base 后面
```

对比 merge：

```text
merge：保留分叉历史，生成 merge commit
rebase：重写提交历史，让提交线性
```

常用：

```bash
git checkout feature
git rebase main
```

注意：

```text
不要随便 rebase 已经推送并被多人共享的公共分支
```

---

## Day 3：AI Coding / Agent 项目表达 + 模拟面试

### 上午：Cursor / Codex / Claude Code 对比

你要从这几个维度讲：

```text
1. 定位
2. 上下文能力
3. 工具调用能力
4. 自动化程度
5. 测试与验证
6. 适合场景
```

简版对比：

| 工具          | 定位                                      | 适合场景              |
| ----------- | --------------------------------------- | ----------------- |
| Cursor      | AI IDE，适合边写边改、快速理解代码、规则/技能驱动开发          | 日常开发、项目内迭代        |
| Codex       | OpenAI coding agent，可读写运行代码，支持 IDE/云端任务 | 自动修复、并行任务、PR/功能开发 |
| Claude Code | agentic coding system，可理解代码库、跨文件修改、运行测试 | 大代码库重构、复杂任务规划     |

Codex 官方称它可以读、编辑、运行代码，帮助构建功能、修复 bug、理解陌生代码；Claude Code 官方文档强调它能理解整个代码库并跨文件和工具完成任务；Cursor 官方则突出 IDE 内 agentic development，以及 Rules/Skills 等定制能力。([OpenAI 开发者][2])

---

### 下午：如何测评 AI Coding 工具效果

你要准备一套评估框架：

```text
1. 任务完成率
2. 一次通过率
3. 编译通过率
4. 单元测试通过率
5. 回归测试通过率
6. 修改范围是否可控
7. 是否引入副作用
8. 代码风格一致性
9. 人工 review 成本
10. 平均完成时间
```

面试版：

> 我不会只看模型“写没写出来”，而是用工程指标评估，比如编译通过率、测试通过率、任务完成率、diff 范围、回归问题数量、人工 review 成本和完成时间。对于 Agent 项目，还会看工具调用成功率、工具选择准确率、上下文压缩后任务成功率和 trace 可解释性。

---

### 晚上：Skill / MCP / Agent

必须能区分：

```text
Agent：能基于目标自主规划、调用工具、观察结果并迭代的系统
MCP：连接 Agent 和外部工具/数据源的标准协议
Skill：可复用的任务能力包/操作流程/上下文指令
Tool Calling：LLM 生成结构化工具调用，Host 执行并回填结果
```

MCP 官方定义是连接 AI 应用与外部系统的开放标准，可接入数据源、工具和工作流；其 specification 也强调 MCP 为 LLM 应用和外部数据源/工具提供标准化集成方式。([模型上下文协议][3])

你的项目表达：

> 我在 ROS Agent 项目中把 ROS2 topic、service、传感器快照、导航命令等能力抽象成工具，并通过 MCP/tool calling 机制让 LLM 以结构化方式调用。Host 负责工具筛选、权限控制、结果回填和 tracing，MCP Server 负责执行具体 ROS 操作。Skill 则用于沉淀可复用工作流，比如导航调试、传感器检查、日志分析。

---

# 三、优先级排序

## P0：必须会

```text
C++ 智能指针
C++ 内存泄露
线程池
DNS
QUIC
索引 B+ 树
MCP / Tool Calling / Agent
AI Coding 评估指标
```

## P1：尽量会

```text
Python 引用计数/GC
git rebase
Cursor/Codex/Claude Code 对比
HTTP/2/HTTP/3
```

## P2：补充

```text
Skill 设计
上下文压缩
工具调用 tracing
安全工具调用
```

---

# 四、二面回答模板

你可以按这个结构准备每个知识点：

```text
1. 是什么
2. 为什么需要
3. 底层原理
4. 工程问题
5. 项目中怎么用
6. 面试官可能追问什么
```

---

# 五、建议你接下来的复习顺序

```text
1. C++ 智能指针
2. C++ 内存泄露
3. C++ 线程池
4. DNS + QUIC
5. 数据库索引
6. Python 内存机制
7. git rebase
8. AI Coding 工具评估
9. Cursor/Codex/Claude Code 对比
10. Skill / MCP / Agent
```

最关键的是：**C++、网络、索引是基础分；AI Coding/Agent 是你的项目加分项。**

[1]: https://cursor.com/?utm_source=chatgpt.com "Cursor: The best way to code with AI"
[2]: https://developers.openai.com/codex/ide?utm_source=chatgpt.com "Codex IDE extension"
[3]: https://modelcontextprotocol.io/docs/getting-started/intro?utm_source=chatgpt.com "What is the Model Context Protocol (MCP)?"
