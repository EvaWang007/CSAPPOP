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

# Redis缓存+DateBase读数据机制

***🥑先查 Redis（缓存），没命中再查 fakeDB（模拟数据库）***


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






















