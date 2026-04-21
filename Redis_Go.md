# Redis 五大数据存储结构 String  Hashmap  Set  zSet  List
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
