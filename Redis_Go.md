# Redis 五大数据存储结构 => String    Hashmap   Set   zSet   List
```
//**************1. Sring 操作****************
	// SET key value EX 10s (设置10秒过期)
	err := rdb.Set(ctx, "user:name", "EvaWang", 10*time.Second).Err()
	if err != nil {
		fmt.Printf("详细信息: %v\n", err) // 查看具体的报错文字
		panic(err)                    // 直接抛出错误，程序会停止运行。所以要放在打印后面
	}

	// GET key
	val, err := rdb.Get(ctx, "user:name").Result()
	if err != nil {
		panic(err)
	}
	fmt.Println("String 结果:", val)

	// INCR (计数器自增)
	rdb.Incr(ctx, "page_view")

	/*关于 err 的返回逻辑，我们需要从 Go 语言的习惯和 go-redis 的设计两个角度来精确理解：

	  1. 每个操作都有 err
	  在 go-redis 中，几乎 每一个 通过 rdb 发出的命令（Set, Get, LPush, ZAdd 等）都会返回一个错误对象。这是为了保证系统的稳定性——因为分布式系统中，网络请求随时可能失败。

	  不过，go-redis 的写法稍有不同，它采用了“链式调用”：

	  rdb.Set(...).Err(): 只想要错误信息，不关心返回值（比如 Set 操作成功了通常只返回 "OK"）。

	  rdb.Get(...).Result(): 既要拿到的值，也要错误信息

	  2.err 的返回值
	  操作顺利时：err 返回的是 nil（空指针/零值）。

	  操作不顺利时：err 返回的是一个 error 对象，里面包含具体的文字描述

	//************2. List 操作***********
	// LPUSH: 从左侧插入数据
	rdb.LPush(ctx, "tasks", "task_1", "task_2")

	// RPOP: 从右侧弹出数据
	task, err := rdb.RPop(ctx, "tasks").Result()
	fmt.Println("弹出的任务:", task)
	if err != nil {
		fmt.Printf("详细信息: %v\n", err)
		panic(err)
	}

	// LRANGE: 获取指定范围内的元素 (0 到 -1 表示全部)
	allTasks, err := rdb.LRange(ctx, "tasks", 0, -1).Result()
	if err != nil {
		fmt.Printf("详细信息: %v\n", err)
		panic(err)
	}
	fmt.Println("List 剩余任务:", allTasks)

	//************3. Hash 操作***********
	// HSET: 设置字段
	rdb.HSet(ctx, "user:1001", "name", "Xinli", "age", "20").Err()
	if err != nil {
		fmt.Printf("详细信息: %v\n", err)
		panic(err)
	}

	// HGETALL: 获取该 Hash 下的所有键值对
	userMap, _ := rdb.HGetAll(ctx, "user:1001").Result()
	fmt.Println("Hash 用户信息:", userMap["name"], userMap["age"])

	//************4. Set 操作***********
	// 1. SAdd: 添加关注的人 (哪怕运行多次，"Golang" 也只会出现一次)
	rdb.SAdd(ctx, "user:Eva:following", "Golang", "Robotics", "DeepLearning", "Redis")

	// 2. SIsMember: 判断是否关注了某个话题
	isFollowing, err := rdb.SIsMember(ctx, "user:Eva:following", "C++").Result()
	if err != nil {
		fmt.Println("查询出错:", err)
	}
	fmt.Printf("Eva 关注了 C++ 吗? %v\n", isFollowing) // 结果应该是 false

	// 3. SMembers: 获取所有关注的话题
	allFollowing, _ := rdb.SMembers(ctx, "user:Eva:following").Result()
	fmt.Println("Eva 的所有关注:", allFollowing)

	// 4. 集合运算示例 (交集)
	rdb.SAdd(ctx, "user:Xinli:following", "Robotics", "Guitar", "C++")

	// SInter: 计算两个用户的共同爱好
	common, _ := rdb.SInter(ctx, "user:Eva:following", "user:Xinli:following").Result()
	fmt.Println("共同爱好:", common) // 应该输出 [Robotics]
	/*唯一性：同一个会员（元素）不能加进来两次，如果尝试添加重复元素，集合会自动忽略它。这就像一个兴趣小组，如果你已经加入了，就不能再加入一次了。
	  自动去重：如果你向 Set 中添加 10 次 "apple"，最后里面也只有 1 个 "apple"。

	  极速查询：判断某个元素是否在集合中（SIsMember）的速度极快，无论集合里有 10 个还是 10 万个元素，耗时几乎一样。

	  集合运算：这是 Set 的杀手锏。它可以直接在 Redis 服务器端计算两个集合的交集、并集、差集

	//************5. zSet 操作***********
	zsetKey := "robot:efficiency:rank"

	// 1. ZAdd: 添加成员及其分数
	// 注意：Member 必须唯一，但 Score 可以重复
	err = rdb.ZAdd(ctx, zsetKey,
		redis.Z{Score: 88.5, Member: "Robot_A"},
		redis.Z{Score: 92.0, Member: "Robot_B"},
		redis.Z{Score: 75.0, Member: "Robot_C"},
		redis.Z{Score: 95.5, Member: "Robot_D"},
	).Err()

	if err != nil {
		fmt.Printf("❌ 写入 ZSet 失败: %v\n", err)
		return
	}

	// 2. ZIncrBy: 给某个成员加分 (比如 Robot_A 优化了算法，分数提升)
	rdb.ZIncrBy(ctx, zsetKey, 5.0, "Robot_A")

	// 3. ZRevRangeWithScores: 获取前 3 名 (按分数从高到低)
	// 0 是第一名，2 是第三名
	fmt.Println("🏆 机器人性能排行榜 (前三名):")
	top3, err := rdb.ZRevRangeWithScores(ctx, zsetKey, 0, 2).Result()
	if err != nil {
		fmt.Printf("❌ 读取排行榜失败: %v\n", err)
		return
	}

	for i, z := range top3 {
		fmt.Printf("第 %d 名: %s, 效率得分: %.1f\n", i+1, z.Member, z.Score)
	}

	// 4. ZRank: 查看某个特定机器人的具体排名 (从低到高排在第几)
	rank, _ := rdb.ZRank(ctx, zsetKey, "Robot_C").Result()
	fmt.Printf("\n🤖 Robot_C 在所有机器人中排名(从低到高): 第 %d 位\n", rank+1)

	// 5. ZScore: 获取指定成员的分数
	score, _ := rdb.ZScore(ctx, zsetKey, "Robot_D").Result()
	fmt.Printf("🎯 Robot_D 的最终得分是: %.1f\n", score)

}
```

# Redis 缓存 + DateBase读数据机制 => 数据读写的原子性和一致性

***🥑先查 Redis（缓存），没命中再查 fakeDB（模拟数据库），***


先读redis里面的
```
type User struct {
	ID   int64  `json:"id"`
	Name string `json:"name"`
	Age  int    `json:"age"`
}

// 模拟数据库
var fakeDB = map[int64]User{
	1: {ID: 1, Name: "Eva", Age: 24},
	2: {ID: 2, Name: "Xinli", Age: 22},
}

type CacheService struct {
	ctx context.Context
	rdb *redis.Client
}

func NewCacheService(addr string) *CacheService {
	rdb := redis.NewClient(&redis.Options{
		Addr: addr,
		DB:   0,
	})
	return &CacheService{
		ctx: context.Background(),
		rdb: rdb,
	}
}

func (s *CacheService) Close() error {
	return s.rdb.Close()
}

func userStringKey(id int64) string { return fmt.Sprintf("cache:user:string:%d", id) }
func userHashKey(id int64) string   { return fmt.Sprintf("cache:user:hash:%d", id) }
func recentKey(id int64) string     { return fmt.Sprintf("cache:user:recent:%d", id) }
func interestKey(id int64) string   { return fmt.Sprintf("cache:user:interest:%d", id) }
func rankKey(day string) string     { return fmt.Sprintf("cache:rank:%s", day) }
```

:🫀***Q & A***🫀
1. 缓存在哪里  
`GetUserByID` 里这行就是“读缓存”：
```go
raw, err := s.rdb.Get(s.ctx, key).Result()
```
这里的 `s.rdb` 是 `redis.NewClient(...)` 创建的 Redis 客户端，所以缓存=Redis。

2. DB 在哪里  
同一个函数里这段是“读 DB”：
```go
u, ok := fakeDB[id]
```
`fakeDB` 是你定义的内存 map：
```go
var fakeDB = map[int64]User{...}
```
它只是“假数据库”，用于教学演示，不是真 MySQL/PostgreSQL。

3. 为什么是“先缓存再 DB”  
因为代码顺序就是：
1. 先 `Get` Redis。  
2. 如果 `err == nil`，直接返回（缓存命中）。  
3. 如果是 `redis.Nil`（key 不存在），才执行 `fakeDB[id]`（回源 DB）。  
4. DB 查到后再 `Set(..., 30*time.Second)` 回写 Redis。

你可以把它理解成：  
Redis 是“近处小仓库”，`fakeDB` 是“远处总仓库”；先看小仓库，没有再去总仓，并把结果补回小仓库。




```
// -------------------- 1) String: 缓存读写 + 过期 --------------------
// Cache-Aside 模式：先读缓存，miss 再读 DB 并回填缓存
func (s *CacheService) GetUserByID(id int64) (User, error) {
	key := userStringKey(id)

	raw, err := s.rdb.Get(s.ctx, key).Result()
	if err == nil {
		var u User
		if unmarshalErr := json.Unmarshal([]byte(raw), &u); unmarshalErr != nil {
			return User{}, unmarshalErr
		}
		fmt.Println("[String] cache hit")
		return u, nil
	}

	if !errors.Is(err, redis.Nil) {
		return User{}, err
	}

	// 缓存未命中 -> 读 DB
	u, ok := fakeDB[id]
	if !ok {
		return User{}, fmt.Errorf("db: user %d not found", id)
	}

	b, _ := json.Marshal(u)
	// 设置 30 秒过期
	if err := s.rdb.Set(s.ctx, key, b, 30*time.Second).Err(); err != nil {
		return User{}, err
	}
	fmt.Println("[String] cache miss, load from db and set ttl=30s")
	return u, nil
}

// 写 DB 后删除旧缓存（常见做法）
func (s *CacheService) UpdateUserName(id int64, newName string) error {
	u, ok := fakeDB[id]
	if !ok {
		return fmt.Errorf("db: user %d not found", id)
	}
	u.Name = newName
	fakeDB[id] = u

	// 删除旧缓存，保证下一次读取拿到新值
	return s.rdb.Del(s.ctx, userStringKey(id)).Err()
}

// -------------------- 2) Hash: 结构化缓存 + 过期 --------------------
func (s *CacheService) SaveUserHash(u User) error {
	key := userHashKey(u.ID)

	fields := map[string]interface{}{
		"id":   u.ID,
		"name": u.Name,
		"age":  u.Age,
	}
	if err := s.rdb.HSet(s.ctx, key, fields).Err(); err != nil {
		return err
	}
	// Hash 需要单独设置过期
	return s.rdb.Expire(s.ctx, key, 30*time.Second).Err()
}

func (s *CacheService) GetUserHash(id int64) (map[string]string, error) {
	key := userHashKey(id)
	return s.rdb.HGetAll(s.ctx, key).Result()
}

// -------------------- 3) List: 最近行为队列 + 过期 --------------------
func (s *CacheService) AddRecentAction(id int64, action string) error {
	key := recentKey(id)
	pipe := s.rdb.TxPipeline()

	pipe.LPush(s.ctx, key, action)        // 新行为放左边
	pipe.LTrim(s.ctx, key, 0, 19)         // 只保留最近20条
	pipe.Expire(s.ctx, key, 24*time.Hour) // 整个列表一天过期

	_, err := pipe.Exec(s.ctx)
	return err
}

func (s *CacheService) GetRecentActions(id int64) ([]string, error) {
	return s.rdb.LRange(s.ctx, recentKey(id), 0, -1).Result()
}

// -------------------- 4) Set: 去重兴趣标签 + 过期 --------------------
func (s *CacheService) AddInterest(id int64, tags ...string) error {
	key := interestKey(id)
	if err := s.rdb.SAdd(s.ctx, key, tags).Err(); err != nil {
		return err
	}
	return s.rdb.Expire(s.ctx, key, 7*24*time.Hour).Err()
}

func (s *CacheService) HasInterest(id int64, tag string) (bool, error) {
	return s.rdb.SIsMember(s.ctx, interestKey(id), tag).Result()
}

// -------------------- 5) ZSet: 排行榜 + 过期 --------------------
func (s *CacheService) AddScore(day string, member string, delta float64) error {
	key := rankKey(day)
	pipe := s.rdb.TxPipeline()

	pipe.ZIncrBy(s.ctx, key, delta, member)
	pipe.Expire(s.ctx, key, 48*time.Hour) // 日榜保留48小时

	_, err := pipe.Exec(s.ctx)
	return err
}

func (s *CacheService) TopN(day string, n int64) ([]redis.Z, error) {
	return s.rdb.ZRevRangeWithScores(s.ctx, rankKey(day), 0, n-1).Result()
}

func main() {
	svc := NewCacheService("localhost:6379")
	defer svc.Close()

	// 1) String 缓存读写演示
	u, err := svc.GetUserByID(1)
	if err != nil {
		panic(err)
	}
	fmt.Println("user:", u)

	_ = svc.UpdateUserName(1, "Eva_New")
	u2, _ := svc.GetUserByID(1) // 会 miss 一次后回填
	fmt.Println("user after update:", u2)

	// 2) Hash 演示
	_ = svc.SaveUserHash(u2)
	h, _ := svc.GetUserHash(1)
	fmt.Println("hash:", h)

	// 3) List 演示
	_ = svc.AddRecentAction(1, "login")
	_ = svc.AddRecentAction(1, "view_course")
	actions, _ := svc.GetRecentActions(1)
	fmt.Println("recent actions:", actions)

	// 4) Set 演示
	_ = svc.AddInterest(1, "Go", "Redis", "Go") // Go 自动去重
	has, _ := svc.HasInterest(1, "Go")
	fmt.Println("has Go:", has)

	// 5) ZSet 演示
	day := time.Now().Format("20060102")
	_ = svc.AddScore(day, "user:1", 10)
	_ = svc.AddScore(day, "user:2", 25)
	_ = svc.AddScore(day, "user:1", 5)
	top, _ := svc.TopN(day, 3)
	fmt.Println("top rank:", top)
}
```

# Redis 分布式锁+发布/订阅 => 并发竞争☄️数据安全⚖️

🥇**分布式锁**：

多个操作同时进行的时候，每个操作都获取一个自己独特的Value，然后执行SetNX机制（即有锁就失败，没有锁就把自己的锁往里放。

注意这里锁的名字是统一的，但是每个操作都有自己独特的Value值，这个值去抢同一个锁），保证操作的时候不会“打架”；

当操作完之后实行Lua脚本，保证***是谁放的Value谁才能删***保证即使任务超时锁过期了操作依然不会被篡改

🥈**发布/订阅**：

以下面这个机器人代码为例子：
```
package main

import (
	"context"
	"fmt"
	"github.com/redis/go-redis/v9"
	"time"
)

var ctx = context.Background()

func main() {
	rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})

	// 定义频道名称
	const droneChannel = "drone:status:updates"

	// 1. 【订阅者逻辑】启动一个后台协程来听消息
	go func() {
		pubsub := rdb.Subscribe(ctx, droneChannel)
		defer pubsub.Close()

		fmt.Println("🛰️  监控系统已上线，正在监听状态...")

		// 用管道（Channel）的方式接收消息
		ch := pubsub.Channel()
		for msg := range ch {
			fmt.Printf("📬 【收到广播】来自频道 [%s]: %s\n", msg.Channel, msg.Payload)
		}
	}()

	// 稍微等一下，确保订阅者已经准备好
	time.Sleep(time.Millisecond * 500)

	// 2. 【发布者逻辑】在主线程发送消息
	fmt.Println("🚀 传感器检测到异常，准备广播...")
	
	messages := []string{
		"警告：水深超过 500 米",
		"状态：推进器转速异常",
		"指令：立即上浮",
	}

	for _, m := range messages {
		// 向频道发送消息
		err := rdb.Publish(ctx, droneChannel, m).Err()
		if err != nil {
			fmt.Println("广播失败:", err)
		}
		time.Sleep(time.Second) // 每秒发一条
	}
}
```

这个订阅者协程pubsub负责监听该订阅者的消息，这个消息的路径来源是droneChannel，这个管道链接了订阅者和发布者；

发布者其实是最后的for函数，负责遍历message中的内容并送入管道droneChannel；

最后这个pubsub使用go内部的channel管道收到的消息printf出去；

🐷tell the difference:

***Redis Channel：像“电台广播”***

机制：它是解耦的。发布者只管往 Redis 里的某个 Key（频道名）扔消息，它根本不知道谁在听。

连接性：订阅者必须保持长连接。如果网络断了，断开期间的消息就永远错过了。

主要目的：实现分布式系统之间的消息同步。

***Go Channel：像“接力棒”***

机制：它是为了解决 Goroutine（协程） 之间的同步。

阻塞性：在 Go 里，如果管道满了或没准备好，发送动作会卡住。这是一种强大的“流量控制”手段。

主要目的：实现单一程序内部的并发安全通信

***In Conclusion⚗️***

***不要通过共享内存来通信，而要通过通信来共享内存***。使用 chan 类型让你能用 range 循环优雅地处理消息。

异步缓冲：go-redis 在后台帮你维护了一个缓冲区。当 Redis 的消息像潮水一样涌来时，它先存在 Go Channel 里，让你的处理逻辑慢慢消化。


🧠 ***Puts them together!!! Volia~~🔥***
```
package main

import (
	"context"
	"fmt"
	"github.com/google/uuid"
	"github.com/redis/go-redis/v9"
	"time"
)

var ctx = context.Background()

func main() {
	rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	
	taskLockKey := "underwater:pipe:repair:lock"
	statusChannel := "underwater:task:broadcast"
	myID := "Robot-Eva-007"

	// --- 1. 订阅者：其他机器人或控制中心在听广播 ---
	go func() {
		pubsub := rdb.Subscribe(ctx, statusChannel)
		ch := pubsub.Channel()
		for msg := range ch {
			fmt.Printf("📡 [基地收到广播]: %s\n", msg.Payload)
		}
	}()

	// --- 2. 分布式锁：尝试获取维修权 ---
	// 抢地盘、贴封条
	lockValue := uuid.New().String()//随机生成的ID，每个进程特有
	success, _ := rdb.SetNX(ctx, taskLockKey, lockValue, 10*time.Second).Result()

	if success {
		fmt.Printf("✅ %s: 抢锁成功！进入维修区域。\n", myID)

		// --- 3. 发布/订阅：维修过程中不断发报 ---
		for i := 20; i <= 100; i += 40 {
			time.Sleep(1 * time.Second)
			message := fmt.Sprintf("🤖 %s 正在维修中，进度：%d%%", myID, i)
			
			// 通过频道告诉所有人进度
			rdb.Publish(ctx, statusChannel, message)
		}

		// --- 4. 释放锁：维修结束，认准身份证 ---
		luaRelease := `
			if redis.call("get", KEYS[1]) == ARGV[1] then
				return redis.call("del", KEYS[1])
			else
				return 0
			end
		`
		rdb.Eval(ctx, luaRelease, []string{taskLockKey}, lockValue)
		fmt.Printf("🔓 %s: 维修完成，撤离并释放锁。\n", myID)
		rdb.Publish(ctx, statusChannel, "📢 维修区域已空出，下一台请进。")

	} else {
		fmt.Printf("❌ %s: 抢锁失败，维修区已有机器人，原地待命。\n", myID)
	}

	time.Sleep(2 * time.Second) // 等待广播发完
}
```

原则1：***先***准备好订阅端，***后***准备好发布端

原则2：进入对数据库的实际操作/发布操作之前要先抢锁

原则3：每个操作都有自己独特的lockValue,它们试图被塞进同一个锁taskLockKey，锁是一样的，一次只能有一把钥匙lockValue插上***(SetNX)***

********************************************************************************************

在刚才的发布for代码（实际的机器人操作）前面加一个锁限制，只有塞锁操作SetNX成功才能进入发布for流程发布消息；

同时在for代码结束之后加Lua机制，保证上锁和解锁的是一个ID否则继续等待这个锁的释放；

这个机制可以防止多个机器人进入发布流程导致redis的操作混乱


以下是一个抢仓库的资源代码
```
mport (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"math/rand"
	"sync"
	"time"

	"github.com/redis/go-redis/v9"
)

// -------------------- 分布式锁 --------------------

type DistributedLock struct {
	rdb   *redis.Client
	key   string
	value string        // 锁的持有者标识（唯一值）
	ttl   time.Duration // 锁自动过期时间
}

func NewDistributedLock(rdb *redis.Client, key string, ttl time.Duration) *DistributedLock {
	return &DistributedLock{
		rdb:   rdb,
		key:   key,
		value: fmt.Sprintf("%d-%d", time.Now().UnixNano(), rand.Int63()),
		ttl:   ttl,
	}
}

// Acquire: SET key value NX PX ttl
// NX=仅当key不存在时设置；PX=毫秒级过期
func (l *DistributedLock) Acquire(ctx context.Context) (bool, error) {
	return l.rdb.SetNX(ctx, l.key, l.value, l.ttl).Result()
}

// 用 Lua 做“比较后删除”
// 只有 value 一致（自己加的锁）才允许删，避免误删别人的锁
var unlockScript = redis.NewScript(`
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
`)

func (l *DistributedLock) Release(ctx context.Context) (bool, error) {
	n, err := unlockScript.Run(ctx, l.rdb, []string{l.key}, l.value).Int()
	if err != nil {
		return false, err
	}
	return n == 1, nil
}

func (l *DistributedLock) WithLock(
	ctx context.Context,
	maxRetry int,
	retryInterval time.Duration,
	fn func() error,
) error {
	for i := 0; i < maxRetry; i++ {
		ok, err := l.Acquire(ctx)
		if err != nil {
			return err
		}
		if ok {
			defer l.Release(ctx)
			return fn()
		}
		time.Sleep(retryInterval)
	}
	return errors.New("acquire lock timeout")
}

// -------------------- 发布订阅 --------------------

func startSubscriber(ctx context.Context, rdb *redis.Client, wg *sync.WaitGroup) {
	wg.Add(1)
	go func() {
		defer wg.Done()

		pubsub := rdb.Subscribe(ctx, "order_events")
		defer pubsub.Close()

		// 等待订阅建立
		if _, err := pubsub.Receive(ctx); err != nil {
			fmt.Println("[SUB] 订阅失败:", err)
			return
		}

		ch := pubsub.Channel()
		for {
			select {
			case msg := <-ch:
				if msg == nil {
					return
				}
				fmt.Println("[SUB] 收到事件:", msg.Payload)
			case <-ctx.Done():
				return
			}
		}
	}()
}

// -------------------- 业务：并发抢库存 --------------------

func buyOnce(ctx context.Context, rdb *redis.Client, workerID int) error {
	lock := NewDistributedLock(rdb, "lock:stock:item:1001", 3*time.Second)

	return lock.WithLock(ctx, 40, 80*time.Millisecond, func() error {
		stock, err := rdb.Get(ctx, "stock:item:1001").Int()
		if err == redis.Nil {
			return errors.New("库存key不存在")
		}
		if err != nil {
			return err
		}

		if stock <= 0 {
			fmt.Printf("[worker-%d] 库存不足，抢购失败\n", workerID)
			return nil
		}

		newStock, err := rdb.Decr(ctx, "stock:item:1001").Result()
		if err != nil {
			return err
		}

		event := map[string]interface{}{
			"worker_id":  workerID,
			"action":     "buy_success",
			"left_stock": newStock,
			"time":       time.Now().Format(time.RFC3339),
		}
		b, _ := json.Marshal(event)

		if err := rdb.Publish(ctx, "order_events", b).Err(); err != nil {
			return err
		}

		fmt.Printf("[worker-%d] 抢购成功，剩余库存=%d\n", workerID, newStock)
		return nil
	})
}

func main() {
	rand.Seed(time.Now().UnixNano())
	ctx := context.Background()

	rdb := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
		DB:   0,
	})
	defer rdb.Close()

	if err := rdb.Ping(ctx).Err(); err != nil {
		panic("Redis 连接失败: " + err.Error())
	}

	// 初始化库存=5
	if err := rdb.Set(ctx, "stock:item:1001", 5, 0).Err(); err != nil {
		panic(err)
	}

	// 启动订阅者
	subCtx, cancelSub := context.WithCancel(ctx)
	var subWG sync.WaitGroup
	startSubscriber(subCtx, rdb, &subWG)
	time.Sleep(200 * time.Millisecond) // 给订阅建立一点时间

	// 10个并发worker抢购
	var wg sync.WaitGroup
	for i := 1; i <= 10; i++ {
		wg.Add(1)
		workerID := i
		go func() {
			defer wg.Done()
			if err := buyOnce(ctx, rdb, workerID); err != nil {
				fmt.Printf("[worker-%d] 执行失败: %v\n", workerID, err)
			}
		}()
	}

	wg.Wait()
	time.Sleep(300 * time.Millisecond) // 等订阅消息打印完
	cancelSub()
	subWG.Wait()

	finalStock, _ := rdb.Get(ctx, "stock:item:1001").Int()
	fmt.Println("最终库存:", finalStock)
}
```

解析：

1. 程序启动与 Redis 连接  
在 [`main.go:526`](/home/evawang/code/go_redis/main.go:526) 到 [`main.go:538`](/home/evawang/code/go_redis/main.go:538)：  
创建 `rdb`，`Ping` 检查 Redis 可用，不可用就直接 `panic`。

2. 初始化共享资源（库存）  
在 [`main.go:540`](/home/evawang/code/go_redis/main.go:540) 到 [`main.go:543`](/home/evawang/code/go_redis/main.go:543)：  
把 `stock:item:1001` 设置为 `5`，表示总库存 5。

3. 启动订阅端（消息接收方）  
在 [`main.go:546`](/home/evawang/code/go_redis/main.go:546) 到 [`main.go:549`](/home/evawang/code/go_redis/main.go:549)，调用 [`startSubscriber`](/home/evawang/code/go_redis/main.go:456)：  
开启 goroutine 订阅 `order_events`。  
之后任何 `Publish("order_events", ...)` 的消息都会在这里打印出来。

4. 启动 10 个并发 worker 抢购  
在 [`main.go:551`](/home/evawang/code/go_redis/main.go:551) 到 [`main.go:562`](/home/evawang/code/go_redis/main.go:562)：  
每个 worker 都执行 [`buyOnce`](/home/evawang/code/go_redis/main.go:487)。

5. 每个 worker 先抢“同一把锁”  
在 [`main.go:488`](/home/evawang/code/go_redis/main.go:488) 和 [`main.go:490`](/home/evawang/code/go_redis/main.go:490)：  
锁 key 固定是 `lock:stock:item:1001`，所以大家竞争的是同一临界区。  
`NewDistributedLock` 会给每个 worker 生成唯一 `value`（持有者标识）[`main.go:405`](/home/evawang/code/go_redis/main.go:405)。  
`WithLock` 内部重试抢锁（最多 40 次，每次间隔 80ms）[`main.go:434`](/home/evawang/code/go_redis/main.go:434)。

6. 抢到锁后才进入扣库存逻辑  
在 [`main.go:491`](/home/evawang/code/go_redis/main.go:491) 到 [`main.go:507`](/home/evawang/code/go_redis/main.go:507)：  
先读库存，`stock<=0` 就失败返回；否则 `DECR` 扣减 1。  
因为有锁保护，避免了并发下“同时读到同一库存”导致超卖。

7. 扣减成功后发布事件给订阅端  
在 [`main.go:509`](/home/evawang/code/go_redis/main.go:509) 到 [`main.go:519`](/home/evawang/code/go_redis/main.go:519)：  
组装 JSON 事件，`Publish` 到 `order_events`。  
订阅协程会收到并打印（见第 3 步）。

8. 业务函数结束时释放锁（安全释放）  
在 [`main.go:446`](/home/evawang/code/go_redis/main.go:446) 会 `defer l.Release(ctx)`。  
`Release` 用 Lua 校验“锁里的 value 是不是我自己的”，是才删锁 [`main.go:418`](/home/evawang/code/go_redis/main.go:418) 到 [`main.go:432`](/home/evawang/code/go_redis/main.go:432)。  
这是防止误删别人锁的关键。

9. 主流程收尾  
在 [`main.go:564`](/home/evawang/code/go_redis/main.go:564) 到 [`main.go:570`](/home/evawang/code/go_redis/main.go:570)：  
等待所有 worker 结束，关闭订阅，最后读取并打印最终库存。

你可以把整个业务抽象成一句话：  
“多个并发请求先竞争同一把 Redis 分布式锁，抢到锁的请求才能安全扣库存；扣减成功后通过 Pub/Sub 广播事件给监听方，实现并发安全 + 异步通知。”






























