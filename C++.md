# C++ Review
## 指针 引用
   定义，参数传递
## 变量
   全局变量，静态全局变量，局部变量，静态局部变量
   想象一下，如果你有 10 个 .cpp 文件都要用同一个全局变量 global_count：

  方法 A（笨办法）：在每个 .cpp 里都手写一遍 extern int global_count;。

  风险：万一哪天你想把变量类型改成 long，你得手动去改 10 个地方。改漏一个，链接器就会报错，或者发生难以排查的内存越界。

  方法 B（专业办法）：在 vars.h 里写一次 extern int global_count;，然后在 10 个 .cpp 里直接 #include "vars.h"。

   好处：一处修改，全局同步。这体现了编程中的 DRY (Don't Repeat Yourself) 原则
## struct class union
   前者默认public继承，后者默认private继承
```
   struct HeroStruct {
    // 默认是 public（公开的）
    std::string name;
    int hp;

    void attack() {
        std::cout << name << " 发起攻击！" << std::endl;
    }
};

// 使用
HeroStruct s;
s.name = "阿古朵"; // 直接访问，合法
```
```
class HeroClass {
    // 默认是 private（私有的）
    std::string name;
    int hp;

public: // 必须显式声明 public 才能在外部访问
    void setName(std::string n) { name = n; }
    void attack() {
        std::cout << name << " 发起攻击！" << std::endl;
    }
};

// 使用
HeroClass c;
// c.name = "李白"; // 错误！编译报错，因为 name 是私有的
c.setName("李白");   // 正确，通过公开接口访问
```
而union是所有成员完全重叠，起始地址相同，每次只能存一个值，sizeof(union)是最大的的一个成员变量的大小
  
   
