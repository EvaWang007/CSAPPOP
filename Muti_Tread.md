# 多线程池的实现原理
线程池类：
```
class ThreadPool {
public:
    ThreadPool(size_t);
    template<class F, class... Args>
    auto enqueue(F&& f, Args&&... args) 
        -> std::future<typename std::result_of<F(Args...)>::type>;
    ~ThreadPool();
private:
    // need to keep track of threads so we can join them
    std::vector< std::thread > workers;//线程池里面的线程们
    // the task queue
    std::queue< std::function<void()> > tasks;//任务队列，存放要执行的函数
    
    // synchronization
    std::mutex queue_mutex;//互斥锁，保证多个线程抢任务时不会乱套
    std::condition_variable condition;//负责唤醒线程
    bool stop;
};
```
线程池构造函数（线程池的运作机理所在）
```
inline ThreadPool::ThreadPool(size_t threads)//线程池构造函数，参数threads是池子里开启的线程数量
    :   stop(false)
{
    for(size_t i = 0;i<threads;++i)
        workers.emplace_back(
            [this]
            {
                for(;;)
                {
                    std::function<void()> task;

                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock,
                            [this]{ return this->stop || !this->tasks.empty(); });
                        if(this->stop && this->tasks.empty())
                            return;
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }

                    task();
                }
            }
        );
}
```
🦫来解释一下这个运作原理：

workers是线程容器，调用emplace_back创建一个线程，task是不断从队列queue里面弹出的函数，this指针指向线程池本身，

线程通过this来看自己跑的是哪个函数。在线程进入队列试图判断之前先看锁的状态:和如果有锁就等着说明其他线程正在队列里面操作，

没有锁就进入队列看：这个函数已经停止或者函数队列空了线程就wait，如果函数已经完成且队列还没空，说明这个线程手上的函数执行完毕了，线程就退出，取下一个函数。

除此之外，每当一个线程领取到函数之后 this->tasks.pop();这个加锁的大括号(这里插入一个🥑)就结束了,锁释放;真正的函数执行在括号外task()


🥑says:锁是怎么加上去的，为什么要加锁

***锁的状态：随作用域“自动开启与关闭”***
   
在 C++ 中，std::unique_lock 采用的是一种叫 RAII（资源获取即初始化）的机制。

加锁时刻：当执行到 std::unique_lock<std::mutex> lock(this->queue_mutex); 这一行时，线程会去尝试“抓”那把锁。如果别的线程正抓着，它就停在这行等，直到抓到为止。

解锁时刻：这把锁会在它所属的大括号 { } 结束时，自动释放

我们可以看到以上代码中，每个线程[this]进去的时候首先会看到**锁代码**' std::unique_lock<std::mutex> lock(this->queue_mutex)`，这个锁的作用域就是它所在的这个大括号。

任何一个线程想去检查队列、取任务，必须先拿到这把“锁”。如果锁被别人拿着，它必须在门外等着。（不然就可能出现两个线程都在对队列进行操作，万一对同一片区域操作就会报段错误）

这个**锁代码**过了之后，线程才能进队列顺利查看判断，接收函数即this->tasks.pop();

这个时候其实函数是没有执行的，我们发现紧跟着一个大括号括回，代表锁的作用域结束了，锁释放。

真正线程对函数的执行是`task()`，这个时候锁已经在上面释放了，也就是说，在一个线程执行函数的时候别的线程可以同时去队列里面找函数。完美！










