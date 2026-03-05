# CSAPP Chapter6
## Q1:现代的计算机架构在传统的的冯·诺伊曼架构上做出了什么改进?
  原始冯·诺依曼结构将指令和数据混合存储。现代 CPU 在内部 L1 Cache 层面采用了哈佛结构:

  指令缓存（L1i）与数据缓存（L1d）分离：CPU 拥有独立的指令总线和数据总线（以行的形式存储的，一般是64bit）。

  并行取指：这允许 CPU 在同一时刻既能读取下一条要执行的指令，又能读写当前指令需要的数据，互不干扰
  
  (L2 Cache和L3 Cache指令缓存和数据缓存是共享的，这和架构设计有关不同的CPU型号架构设计可能不同)

## Q2：Cache的读写策略是什么？
   写通策略：Cache行一旦被改写，会立即同步一路往下更新到主存

   写回策略：Cache行被改写后只是被标记，在该行即将被逐出之前检查该标记，需要的话再写回主存（但是存在缓存一致性的问题）

   MESI协议：Modified, Exclusive, Shard, Invalid（多核系统中不同CPU核之间的缓存之间的交流方式，比如一个数据在核A的缓存中被修改了它怎么通知其他的缓存）
   Modified: The local processor has modified the cache line. This also implies it is the only copy in any cache.
   
   Exclusive: The cache line is not modified but known to not be loaded into any other processor’s cache.   
   
   Shared: The cache line is not modified and might exist in another processor’s cache.  
   
   Invalid: The cache line is invalid, i.e., unused.

  ⚔️CPU设计的关键很大程度在**分支预测**上，CPU 内部有一个极其聪明的分支预测器。它会像老司机一样总结经验：

  静态预测：比如往回跳的（循环）通常预测为“跳转”，往前跳的（if）通常预测为“不跳转”。

  动态预测：CPU 会维护一张表，记录过去几次这个地方是怎么跳的。

  如果过去 10 次循环都跳回了开头，第 11 次它会果断预测“继续跳回。
  
  理解了分支预测，你就知道为什么**“有序数组”比“无序数组”处理起来快得多**：

 有序数组：if (data[i] < 128) 的结果通常是连续的一堆“真”，然后连续的一堆“假”。CPU 预测器非常容易抓到规律，几乎 100% 命中。

无序数组：结果忽真忽假，随机分布。CPU 预测器会反复被打脸，流水线不断重启，速度极慢

## Q3:CPU是怎么读出所需要的数据的？（逻辑链条）
翻译阶段：CPU 给出虚拟地址（VA）→MMU 查找 TLB/页表→ MMU算出物理地址（虚拟页号对应的物理页号+页内偏移）。

缓存阶段：Cache Hit：直接拿走，这是最理想的情况（1ns）。Cache Miss：去 RAM 找。

主存阶段：RAM Hit：把包含该数据的一块（Cache Line，通常 64B）搬进 Cache，再给 CPU。RAM Miss（缺页）：去磁盘找。OS 从磁盘找到包含该数据的页面。将这个页面拷贝到 RAM 的一个空闲页帧里。更新页表：把原本指向磁盘的那个条目（Valid=0），改成指向刚填入 RAM 的这个物理页帧号（Valid=1）

外存阶段：Disk Read：OS 介入，把 4KB 的整页从磁盘搬进 RAM（耗时极长，ms 级别）。替换/写入：如果 RAM 满了，用 LRU 踢走一页，放入新页，并更新页表。

  
