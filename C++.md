# C++ 基本语法
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

## define const typedef volatile
define是纯替换，const是常量声明（指针或者普通值），后面不能再做修改；typedef是把c++中的一些类型或者自己定义的结构体换个短一点的名字后面类型声明方便一些。

编译器为了提速，常把变量值缓存到 CPU 寄存器里。但有些变量（如硬件状态、多线程共享标志）的值可能在编译器不知情的情况下被外部改变。
而volatile的作用：强制要求每次读取该变量时，必须老老实实地从 物理内存（RAM） 中读取，而不是使用寄存器里的旧备份

## 虚函数，析构函数，构造函数
   虚函数是在基类中定义（只有~函数名()），在派生类（子类）中会被重写的，它的目的就是为了在使用基类指针调用某些函数的时候，实际上我们希望它根据子类的不同情况调用子类的函数；
   
   **析构函数**就属于上述情况之一，顺序是子类→基类，当我们删除一个基类指针是和希望首先调用其派生类的析构函数再调用基类的析构函数确保不同的派生类的资源都可以正确释放；
   
   **构造函数**是在对象创建的时候调用的,顺序是基类→子类，如果子类没有创建好的话不可能重写构造函数


# C++编译过程
:process: 
***:love: 静态链接库和动态链接库***
1. 静态链接库（ Static Link Linbrary)
假设你有一个数学库 math.a，里面包含 add() 和 sub() 函数。

你写了 main.cpp 调用了 add()。

链接器把 math.a 中关于 add() 的二进制指令“复制”进 main.exe。

结果：你的 main.exe 即使脱离了 math.a 也能独立运行，因为它内部已经自带了那份代码。

2. 动态链接库 (Dynamic Link Library)
后缀名：.dll (Windows), .so (Linux)

运作方式（下馆子）：
在链接阶段，链接器不会拷贝具体代码，而是在 .exe 中保留一个“引用标记”。直到程序**运行（Runtime）**时，操作系统才会去内存或磁盘中寻找对应的库文件并加载它。

结合例子：
同样的数学库，这次是 math.so。

你编译 main.cpp。

链接器在 main.exe 里写下一行：“运行的时候请去找一个叫 math.so 的库，调用里面的 add() 函数。”

结果：如果你的电脑里没有 math.so，双击 main.exe 会报错：“找不到 .dll/.so 文件”



  
   
