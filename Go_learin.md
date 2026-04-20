## Go的工程化使用

### 1. 泛型
    接口interface，可以满足在接收端未知传输数据类型的情况下，自动进行类型推导
```
    package main

import "fmt"

// 【1. 定义约束】：使用 ~ 表示支持该类型的衍生类型
type Number interface {
    ~int | ~int64 | ~float64
}

// 【2. 编写泛型函数】：T 是类型参数，它必须满足 Number 约束
func GetMax[T Number](list []T) T {
    var max T = list[0]
    for _, v := range list {
        if v > max {
            max = v
        }
    }
    return max
}

func main() {
    // 【3. 调用】：Go 会自动推导类型（实例化）
    ints := []int{1, 5, 2}
    floats := []float64{3.14, 1.1, 5.5}

    fmt.Println(GetMax(ints))   // 输出: 5
    fmt.Println(GetMax(floats)) // 输出: 5.5
}
```
🐑 ***explains:函数GetMax的输入类型为通过接口Number约束的类型T,输入类型是一个由类型T元素构成的list，输出是一个T类型的数据，这个接口Number在type处被定义***

T Number（准入证）：这就像是在门口设个保安。只有满足 Number 接口要求的类型（比如 int 或 float64）才拿得到这张准入证 T。

(list []T)（输入参数）：既然 T 已经确定了身份，那么输入的必须是一个由 T 这种元素组成的切片（Slice）。

T（输出结果）：既然是求最大值，返回的东西当然也得是 T 类型


### 2. 断言(assertion)
```
package main

import "fmt"

func main() {
    var i interface{} = "Hello, Go!"

    // 尝试将 i 断言为 string 类型
    s, ok := i.(string)
    if ok {
        fmt.Println("断言成功:", s)
    } else {
        fmt.Println("断言失败")
    }

    // 尝试将 i 断言为 int 类型
    n, ok := i.(int)
    if ok {
        fmt.Println("断言成功:", n)
    } else {
        fmt.Println("断言失败")
    }
}
```
s, ok := i.(string)

s (Value)：这是结果变量。

如果断言成功：s 会存储转换后的具体值（即 "Hello, Go!"）。

如果断言失败：s 会被赋予该类型的零值（比如 string 的零值是 ""，int 的零值是 0）。

ok (Boolean)：这是成功标志。

它只负责告诉你：这次“开箱”到底成没成功。成功就是 true，失败就是 false。

i.(string)：这是动作本身。

意思是：“我认为接口 i 里面装的是 string，请帮我验证并提取。”

***断言的存在可以让接收端更加方便的进行接口拼接，自动识别接口所传输的数据，并以此进行代码段的执行***



### 3. Channel
```
package main

import (
    "context"
    "fmt"
    "time"
)

// 模拟一个复杂的 AI 处理任务
func callLLM(id int, query string, resultChan chan<- string) {
    // 模拟不同的模型响应速度不同
    duration := time.Duration(id*2) * time.Second 
    time.Sleep(duration)
    
    resultChan <- fmt.Sprintf("模型 %d 的结果: 对于 '%s' 的回答完毕", id, query)
}

func main() {
    // 1. 定义一个带缓冲的信道，防止协程阻塞无法退出（内存泄露）
    resChan := make(chan string, 3)
    
    // 2. 创建一个带超时的上下文 (Context)，这是后端控制生命周期的标准做法
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    // 3. 并发启动 3 个 Agent 任务
    query := "如何设计 AUV 的控制系统？"
    for i := 1; i <= 3; i++ {
        go callLLM(i, query, resChan)
    }

    // 4. 深度使用 Select：多路复用监听
    select {
    case res := <-resChan:
        // 谁跑得快，我们就拿谁的结果
        fmt.Println("【成功】取到最快结果:", res)
    case <-ctx.Done():
        // 如果 3 秒到了，resChan 还没收到数据，ctx.Done 会触发
        fmt.Println("【超时】所有模型响应太慢，执行熔断逻辑")
    }

    fmt.Println("主程序结束")
}
```

建立一个 Channel 就像在内存里开辟一个“仓库”，必须使用内置函数 make。

语法：ch := make(chan 类型)

例子：ch := make(chan int) —— 建立一个专门装整数的通道。

2. 操作数据：用 <- (箭头符号)
<- 是 数据流动 的方向标识。你可以根据箭头指向哪，来判断是“存”还是“取”。

A. 发送数据（存入）
箭头指向 Channel，表示把数据塞进去。

Go
ch <- 100  // 把数字 100 送进管道 ch
B. 接收数据（取出）
箭头从 Channel 指出来，表示把数据拿出来。

Go
data := <-ch  // 从管道 ch 拿出一个东西，赋给变量 data

🦫 ***asks ` resChan := make(chan string, 3)` why there is a 3 here???***
假设你写了 ch := make(chan int, 3)：

发送第 1、2、3 个数据时：发送方（协程）不会阻塞。数据被放在缓冲区里，发送方代码继续往下跑。

发送第 4 个数据时：此时“暂存架”满了。如果还没有人来取走任何一个数据，发送方协程就会在这里停住（阻塞），直到架子腾出空间。

接收数据时：只要架子上有东西，接收方可以直接拿走。如果架子空了，接收方就会停住（阻塞），直到有人往里放东西

***这是一个缓冲设置！！！***






### 4.defer机制
//defer标记
当你写下 defer 时，你实际上是在告诉编译器：“先别急着执行这行代码，把它存进一个栈里，等我这个函数准备 return 的时候再回头跑它。”
func demo() {
    defer fmt.Println("我是最后执行的") // 就像设置了一个定时炸弹
    fmt.Println("我是先执行的")
}
// 输出：
// 我是先执行的
// 我是最后执行的












### 5. G(goroutine) M(machine) P(processor)机制
```
package main

import (
	"context"
	"fmt"
	"math/rand"
	"time"
)

// 任务结果结构体
type CheckResult struct {
	ID   int    // 电机编号
	Msg  string // 状态信息
}

func main() {
	// 1. 创建通信信道 (Channel)
	// 缓冲区设为 3，确保 3 个协程发完数据后都能正常退出，不阻塞 M
	resultChan := make(chan CheckResult, 3)

	// 2. 设定 Context 超时 (控制整个 select 的阻塞时长)
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	fmt.Println("开始并行检查 3 个推进器电机...")

	// 3. 开启协程 (Goroutines)
	// 此时调度器 (Scheduler) 会将这 3 个 G 分配给 P，由 M 运行
	for i := 1; i <= 3; i++ {
		go func(motorID int) {
			// 模拟不同电机的响应耗时 (随机 1-3 秒)
			scanTime := time.Duration(rand.Intn(2000)+500) * time.Millisecond
			time.Sleep(scanTime)

			// 协程间通信：将结果塞入管道
			resultChan <- CheckResult{
				ID:  motorID,
				Msg: fmt.Sprintf("耗时 %v, 状态正常", scanTime),
			}
		}(i)
	}

	// 4. Select 阻塞等待机制
	// 主协程运行到这里会停下，直到 resultChan 有数据或者 ctx 超时
	select {
	case firstRes := <-resultChan:
		// 谁先跑完就跑这个 case
		fmt.Printf("【收到首个反馈】推进器 %d 检测完毕: %s\n", firstRes.ID, firstRes.Msg)

	case <-ctx.Done():
		// 如果 2 秒内没有任何协程完成，或者没拿到最快的，就跑这个 case
		fmt.Println("【超时警告】动力系统检测响应过慢，进入安全模式")
	}

	fmt.Println("主程序流程结束。")
}
```
***🍎Core功能实现：主函数创建了缓冲为3的传输CheckResult类型数据的通道resultChan，for循环开启协程func运行，先运行完的协程把结果塞入管道，select阻塞等待机制，先传输出结果的协程把结果写入firstRes并输出否则就进入安全模式***

🥦分析：
##关于 firstRes := <-resultChan：

resultChan 里的第一个结果被**主协程（Main Goroutine）**从通道中取出来，并赋值给了 firstRes 变量。

##关于“塞入”和“取出”：

协程是“生产者”，它们负责往管道里塞（Send）。

主函数（select）是“消费者”，它负责从管道里取（Receive）

在 Go 语言中，这其实是由三个部分组成的：

func(motorID int) { ... }：
这是一个没有名字的函数定义。在 C++11 之后，这被称为 Lambda 表达式。你没有给它取名（比如不叫 checkMotor），而是直接定义了它的逻辑。

##go 关键字：
它告诉 Go 调度器：“嘿，不要在当前线程直接运行后面这个函数，而是给它创建一个新的 Goroutine (G)，把它扔到队列里去异步执行。”

最后那个 (i)：
这代表立即调用。因为函数定义好了之后，必须传参并运行它。这里的 (i) 就是把当前循环的变量 i 传给函数内部的 motorID

主协程：发令枪响，通过 go 派出了 3 个匿名函数工兵。

主协程：立刻跑到 select 终点站坐下，闭上眼睛阻塞（休息）。

匿名工兵 (G1, G2, G3)：在不同的 CPU 核心上拼命跑。

第一个抵达的工兵：把“接力棒”（数据）塞进 resultChan。

调度器：感应到管道有货，拍醒主协程。

主协程：睁眼，拿到接力棒（赋值给 firstRes），执行打印，然后潇洒离开，不再管后面的人

🦌 finds out:

G (Goroutine)：你开启的那些任务。

M (Machine)：真正的物理线程（由内核管理）。

P (Processor)：处理器上下文（调度的关键）。

它的运行机制是：

绑定：一个 M（工人）必须绑定一个 P（工作证）才能运行 G（任务）。

本地队列：每个 P 都有一个自己的“待办清单”（Local Queue），存放着还没运行的 G。这样 M 拿任务时不需要加锁，效率极高。

任务窃取 (Work Stealing)：如果某个 M 特别快，把自己 P 里的任务干完了，它会去别的 P 那里**“偷”**一半任务过来帮着干。这保证了多核 CPU 不会被浪费。


***Go 提倡 “通过通信来共享内存”***

同步机制：Channel 不仅仅是传数据的，它还自带“阻塞”属性。当一个协程 <-ch 拿不到数据时，调度器会把它挂起（不占用 CPU），直到有数据进入。

解耦：生产者协程不需要知道消费者是谁，只要往管道里塞数据即可。这大大降低了并发编程中死锁（Deadlock）的概率。

***GMP机制的优势***
极度轻量：一个 C++ 线程默认栈大小是 1MB 或 8MB，而一个 Goroutine 初始只需 2KB。这意味着你可以在普通笔记本上轻松开启 100 万个协程而不崩溃。

用户态管理：Goroutine 的切换不需要进入操作系统内核（Kernel），而是在用户态完成。这比 C++ 的线程上下文切换快得多。







