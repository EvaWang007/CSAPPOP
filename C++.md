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

   In conclusion,析构函数可以（最好）写成虚函数，而构造函数不能是虚函数。


# C++编译过程
:avocado: 
***:heart: 静态链接库和动态链接库***
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

# C++面向对象特性：继承&多态
## 继承
   子类会继承到基类的部分特性，继承方式如下：
   
   public：外部可调用。

   private：基类的“小秘密”，子类不可见。

   protected：专门为继承设计的。外人看不见，但自家（子类）可以随便用
   
   继承最强大的地方在于***代码复用***。
## 多态
   本质：接口与实现分离。基类定义了一个接口，而子类提供这个接口针对不同的子类有不同的实现。

   解耦：代码不再依赖于具体的类，而是依赖于抽象的基类。增加新子类时，原有代码（如 main 函数逻辑）不需要修改

   动态绑定（针对运行期多态）：通过虚函数表（vtable），程序在运行时才决定调用哪个函数，而不是编译时
### 重载(Overload)
      在同一个作用域内，函数名相同但参数个数、类型或顺序不同
```
   class Calculator {
public:
    int add(int a, int b) { return a + b; }
    // 重载：函数名相同，参数类型不同
    double add(double a, double b) { return a + b; } 
};
```
### 重写(Override)
    派生类重新定义基类中被声明为 virtual 的函数。这是实现动态多态的基础。其中在基类中被定义为纯虚函数的函数在子类中必须被重写。
```
class Animal {
public:
    virtual void speak() { std::cout << "未知叫声" << std::endl; }
};

class Dog : public Animal {
public:
    // 重写：基类是 virtual，函数签名完全一致
    void speak() override { std::cout << "汪汪！" << std::endl; } 
};
```
### 隐藏(Hide)
    当子类定义了与父类同名但不符合重写条件的函数时，父类的函数会被“遮住”，子类调用的时候只能用子类写的那个看不到父类那个了。
```
class Base {
public:
    void fun(int a) { std::cout << "Base fun" << std::endl; }
};

class Derived : public Base {
public:
    // 隐藏：函数名相同，但参数不同，或者基类函数没有 virtual
    // 此时 Base 里的 fun(int) 在 Derived 作用域内被屏蔽了
    void fun(std::string s) { std::cout << "Derived fun" << std::endl; }
};

// 使用时：
Derived d;
d.fun("hello"); // 正确
// d.fun(10);   // 错误！编译器找不到 fun(int)，因为它被 fun(string) 隐藏了
```

## C++语言特性
### 左值和右值
:banana:左值：指表达式结束后依然存在的持久对象。

:apple: 右值：表达式结束就不再存在的临时对象。

以下用比较直观的例子介绍使用效果：
首先是一个一般的左值引用（引用拷贝）
```
class MyString {
    char* data;
public:
    // 拷贝构造函数：参数是左值引用 (const MyString& )
    MyString(const MyString& other) {
        // 1. 看到别人有数据，我也先申请一块同样大的内存
        data = new char[strlen(other.data) + 1];
        // 2. 一个字符一个字符地复制过去
        strcpy(data, other.data);
        std::cout << "执行了深拷贝，好累啊！\n";
    }
};
```
如果那个“旧对象”是一个临时对象（右值），比如一个函数返回的临时结果。它马上就要被销毁了。既然它要死了，我们为什么还要辛苦复制呢？
直接把它的内存地址抢过来不就行了？所以接下来使用右值引用：
```
class MyString {
    char* data;
public:
    // 移动构造函数：参数是右值引用 (MyString&& )
    MyString(MyString&& other) {
        // 1. 直接把你的内存地址给我，我不申请新空间了
        this->data = other.data;
        // 2. 把你的指针抹掉（置空），防止你自杀（析构）时把我的数据带走
        other.data = nullptr; 
        std::cout << "执行了移动语义，直接抢了指针，真快！\n";
    }
};
```

### 虚函数和纯虚函数
虚函数：被virtual关键字修饰的成员函数

纯虚函数：被virtual修饰且=0。纯虚函数是在基类中声明的虚函数，它在基类中没有定义，但要求任何派生类都要定义自己的实现方法。

:warning:虚函数必须实现，否则编译器会报错!!!

***虚函数的实现机制 ——→虚函数表***
每个类（基类和各个子类）都有自己独一无二的一张表。这张表就像是一个“索引清单”，记录了当前这个类应该执行哪些函数。

当子类继承基类时，会发生以下三个步骤的“演变”：

步骤一：拷贝（Copy）
子类的虚函数表在初始状态下，是直接拷贝基类的虚函数表。此时，:arrow:如果子类什么都不做，它的表里存的还是父类函数的地址。

步骤二：替换（Override/重写）
一旦子类重写了某个虚函数，编译器就会把子类虚函数表中原本指向父类函数的那个地址，**替换（覆盖）**成子类自己实现的函数地址。

步骤三：保留（Keep）
对于子类没有重写的虚函数，表里的地址依然**指向父类的实现**

比如这个Dog类和Cat类都继承自Animal基类
<img width="1548" height="466" alt="image" src="https://github.com/user-attachments/assets/3483df5e-e12a-45b9-bb84-a896c383eb4f" />

煮一个:nut:
当你写 Animal* p = new Dog(); 通过Dog类实例化一个Dog对象时，发生了什么？

对象里藏了线索：在内存中，new Dog() 创建出的那个对象块，头部存着一个指针（vptr）。

线索指向 Dog 表：这个 vptr 坚定地指向 Dog 的虚函数表，而不是 Animal 的。

基类指针去“顺藤摸瓜”：当你调用 p->speak() 时，代码并不关心 p 是什么类型，它直接去 p 指向的对象头部取走 vptr。

精准打击：既然 vptr 指向的是 Dog 的表，查表得到的自然就是 &Dog::speak。

这就是为什么同一个 Animal* 指针，指着猫就查猫的表，指着狗就查狗的表。如果没有重写，就会被带回到父类:arrow:处。



    

   
   
   



  
   
