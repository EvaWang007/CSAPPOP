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

:kiwi:TIPS:如何判断结构体是否相等？***能否用 memcmp 函数判断结构体相等？NO!NO!NO***
Struct的内存分布原则：

1. 成员对齐规则：
每个成员相对于结构体首地址的偏移量（Offset），必须是该成员大小（或 #pragma pack 指定的值）的整数倍。

2. 结构体整体对齐规则
结构体的总大小必须是其最大成员大小的整数倍。如果不足，编译器会在末尾填充（Padding）字节。

3. 嵌套对齐规则
如果结构体 A 中包含结构体 B，则 B 作为一个整体，其对齐起始位置应该是 B 内部最大成员大小的整数倍。

所以将占用空间大的变量放在前面，可以有效减少 Padding，缩小结构体体积。

所以，加了padding的结构体会很不一样！！！很多初学者尝试用 memcmp(&s1, &s2, sizeof(MyStruct))。
这是错误的！ 因为：

***Padding 的值是不确定的***：结构体中间填充的字节里可能是随机的垃圾数据，即使两个结构体的数组成员完全一样，memcmp 也可能因为 Padding 不同而返回“不相等”。

含有指针：如果结构体里有指针，memcmp 比较的是地址，而不是指针指向的内容。

:sunshine:正确做法 A：重载 operator== (推荐)
这是最稳妥、最符合 C++ 工程实践的方法，自己手写重载一个运算符==，对结构体中的每个部分进行比较，如下所示：
```
struct Student {
    int id;
    std::string name;
    Date birthday;
    std::vector<int> scores;

    // 重载 == 运算符
    // const: 保证比较过程中不会修改对象成员
    // & : 引用传递，避免大对象拷贝开销
    bool operator==(const Student& other) const {
        // 逐个成员进行逻辑比较
        return id == other.id &&
               name == other.name &&
               birthday == other.birthday && // 这里会调用 Date 的 ==
               scores == other.scores;       // std::vector 自带了 == 重载
    }

    // 习惯上通常也会顺手把 != 也写了
    bool operator!=(const Student& other) const {
        return !(*this == other); // 直接复用 == 的逻辑
    }
};
```

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

煮一个:nut:：
当你写 Animal* p = new Dog(); 通过Dog类实例化一个Dog对象时，发生了什么？

对象里藏了线索：在内存中，new Dog() 创建出的那个对象块，头部存着一个指针（vptr）。

线索指向 Dog 表：这个 vptr 坚定地指向 Dog 的虚函数表，而不是 Animal 的。

基类指针去“顺藤摸瓜”：当你调用 p->speak() 时，代码并不关心 p 是什么类型，它直接去 p 指向的对象头部取走 vptr。

精准打击：既然 vptr 指向的是 Dog 的表，查表得到的自然就是 &Dog::speak。

这就是为什么同一个 Animal* 指针，指着猫就查猫的表，指着狗就查狗的表。如果没有重写，就会被带回到父类:arrow:处。

刚才我们讨论的是单继承的情况，就是这个子类只继承于一个基类；现在我们讨论多继承的情况，一个子类继承于好几个基类比如：class Derived : public Base1, public Base2

子类对象会按照继承顺序，在内存中依次排列 Base1 的部分和 Base2 的部分。具体示例如下：假设Base1里面有函数f1被子类Derived重写，除此之外子类新增一个f3，Base2里面有一个f2被重写：
```
此时 Derived 对象的内部长这样：

vptr1 (指向 Derived 的第一张虚表)

[0]: &Derived::f1 (重写了 Base1)

[1]: &Derived::f3 (子类新增的，挂在第一张表后面)

Base1 的成员变量

vptr2 (指向 Derived 的第二张虚表)

[0]: &Derived::f2 (重写了 Base2)

Base2 的成员变量

Derived 自己的成员变量

```
<img width="827" height="269" alt="image" src="https://github.com/user-attachments/assets/33b4da8c-5bc2-4e80-b0cc-9e656c1dd550" />

:attention:C++ 必须保证：当你把子类指针强制转换为任何一个父类指针时，该指针指向的内存布局都必须符合那个父类的预期

## C++容器STL
### 数据存储容器
***1.序列式容器***

按元素插入顺序存储,元素位置与插入顺序相关,不自动排序。

vector(***慢删快查***):动态数组,内存连续,支持随机访问([]或at()),优点尾插/尾删效率高(O(1)),适合频繁访问元素的场景。缺点是中间插入/删除效率低(O(n)),扩容时可能重新分配内存。

deque(***慢删快查***):双端队列,内存分段连续,支持首尾高效操作。优点头插/头删、尾插/尾删效率均为O(1),可随机访问,适合场景需要在两端频繁操作的场景(如实现队列、栈)。

list(***慢查快删***):双向链表,元素通过指针连接,内存不连续。优点任意位置插入/删除效率高(O(1),只需修改指针)。缺点不支持随机访问(访问元素需遍历,O(n)),内存开销较大。

array:固定大小数组(C++11新增),编译时确定大小,内存连续,比原生数组更安全(支持边界检查),但大小不可变。

***2.关联式容器（key和val）***:
元素的存储和辨认都是依靠键(key),支持快速查找(通常O(log n)),分为有序和无序两类。

**有序**：

set（元素自动键值的升序排列，key=value），

map(也是按照键值升序但是key不一定等于value）,

以上二者的底层实现都是基于：红黑树 (Red-Black Tree)。

代价：每次插入都会进行“旋转”来保持树的平衡，所以插入速度比哈希表慢。

威力：支持范围查找（比如查找所有 key 在 10 到 100 之间的元素），这是哈希表做不到的。

严格弱序：要求 Key 必须实现 < 运算符

**无序**

无序关联容器 (unordered_set / map)底层实现：

**哈希表 (Hash Table)**:取决于哈希函数的映射。它通过“哈希函数”将你的 Key 转化成一个数字（索引），直接去对应的内存地址拿东西。

性能瓶颈：如果哈希函数选得不好，会产生“哈希冲突”，导致性能退化到 O(N)，哈希表为了减少“哈希冲突”（两个不同的 Key 算出同一个索引），通常会预留大量的空位（桶）。

对比：链表的内存利用率很高（除了多出的指针），而哈希表往往需要 1.5 到 2 倍于数据的空间来维持高效率


### 迭代器
本质：它的存在让算法可以不关心容器的底层实现：无论你是数组、链表还是树，只要你有迭代器，算法就能遍历你。

:peace:迭代器的基本用法(四个核心操作)：

*it: 解引用（获取指向的数据）。

it++: 向后移动（指向下一个元素）。

it == container.end(): 边界判断（end() 指向最后一个元素之后的“空位置”）。

container.begin(): 起点（指向第一个元素）。

一个vector的迭代器示例：
```
int main() {
    std::vector<int> v = {10, 20, 30};

    // 1. 获取迭代器
    std::vector<int>::iterator it = v.begin();

    // 2. 遍历
    for (; it != v.end(); ++it) {
        std::cout << *it << " "; // 输出：10 20 30
    }
}
```
BUT:::warning: 
     迭代器虽然好用，但它是极其脆弱的“快照”。一旦容器的结构发生变化（增删），迭代器持有的底层指针可能就失效了，比如deque 中间插入会搬移元素。由于 deque 迭代器里记录了具体内存块（node）的地址，一旦元素搬移或者中控表重排，这个迭代器里的四个指针（cur, first, last, node）就会全部指向错误的地方。
```
std::vector<int> v = {1, 2};
auto it = v.begin(); 
v.push_back(3); // 触发扩容，原有内存被释放，重新开辟空间
// std::cout << *it; // 崩了！it 指向的是已经被销毁的老内存地址
```

迭代器的典型应用：删除元素
```
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v = {1, 2, 2, 3, 4, 2, 5};
    
    // 目标：删除所有等于 2 的元素
    for (auto it = v.begin(); it != v.end(); ) {
        if (*it == 2) {
            // ***重要：必须用返回值更新迭代器，不能执行 it++，这个删掉之后返回的it就是指向删掉后面的那个位置了***
            it = v.erase(it); 
        } else {
            ++it; // 只有没删除时才手动递增
        }
    }
}
```
container.erase(it) 执行后，迭代器 it 会立即失效。但 erase 会返回一个指向被删元素下一个位置的有效迭代器。

:avocado:says ：
迭代器失效的方式很多，当容器的结构发生改变的时候，之前生成的旧的的迭代器就有可能发生失效

:peanuts: says yes I know some cases like this:

<img width="1404" height="164" alt="image" src="https://github.com/user-attachments/assets/d4e28008-94b6-4eea-bf4e-38710a8c4cac" />
<img width="1532" height="130" alt="image" src="https://github.com/user-attachments/assets/4f91dc82-f6e8-43ee-bdda-cfba3e0e5a84" />

:banana: says:刚才提到的都是序列式容器，现在来看看关联式容器，就没有那么容易失效，为什么呢，这很好理解，关联式容器是链式存储，插入元素的时候只要
迭代器指的那个地方没出事，就能顺着找到每一个（包括新插入的）,但是也不是完全不出问题
迭代器有其生命终点，主要有以下三种情况会导致失效：

A. 元素被删除（这是最常见的）
如果你删除了迭代器指向的那个特定节点，该节点的内存被回收，迭代器自然废了。

B. 容器被销毁
如果整个 map 或 set 的生命周期结束（比如出了作用域），内部所有节点被释放，所有迭代器全部失效。

C. clear() 操作
调用 m.clear() 会销毁所有节点，迭代器全部失效


### priority_queue的底层实现原理

在 C++ STL 中，priority_queue 默认底层是基于 std::vector（动态数组） 实现的，逻辑上它是一棵完全二叉树。

***:orange:tells you why use vector instead of list:***

随机访问能力：计算父子索引需要 O(1) 的定位能力，链表需要O(N)。

内存连续性：对于 CPU 缓存极其友好，读取速度快。（vector的存储几乎是在一块连续的内存上易于存取，很少出现缺页）

尾部操作高效：堆的插入总是先放在最后，vector::push_back 是平均 O(1)

以下是一个基于vector的简单大顶堆(Heap)逻辑实现：
```
lass MaxHeap {
private:
    std::vector<int> heap;

    // 【核心动作1：上浮】 插入时使用
    void heapifyUp(int index) {
        while (index > 0) {
            int parent = (index - 1) / 2;
            if (heap[index] > heap[parent]) {
                std::swap(heap[index], heap[parent]);
                index = parent; // 继续向上比对
            } else {
                break;
            }
        }
    }

    // 【核心动作2：下沉】 弹出堆顶时使用
    void heapifyDown(int index) {
        int size = heap.size();
        while (true) {
            int left = 2 * index + 1;
            int right = 2 * index + 2;
            int largest = index;

            // 检查左孩子是否更大
            if (left < size && heap[left] > heap[largest]) {
                largest = left;
            }
            // 检查右孩子是否更大
            if (right < size && heap[right] > heap[largest]) {
                largest = right;
            }

            if (largest != index) {
                std::swap(heap[index], heap[largest]);
                index = largest; // 继续向下调整
            } else {
                break; // 已经比子节点都大了，停止
            }
        }
    }

public:
    void push(int val) {
        heap.push_back(val);      // 1. 先插到数组末尾
        heapifyUp(heap.size() - 1); // 2. 向上调整
    }

    void pop() {
        if (heap.empty()) return;
        // 1. 用最后一个元素覆盖堆顶
        heap[0] = heap.back();
        heap.pop_back();
        // 2. 向下调整，维持最大堆性质
        if (!heap.empty()) {
            heapifyDown(0);
        }
    }

    int top() { return heap[0]; }
    bool empty() { return heap.empty(); }
};
```

:skyrocket:到这里应该会好奇deque和vector的区别：

***vector:***

一块连续的内存区域，随机访问极快：一次简单的加法运算O(1)所以CPU 预取效率最高（总是会命中）

但是扩容代价大：需要开辟新空间并整体搬迁，改动元素也很麻烦，需要整块的移动后面部分

***deque***

deque 的数据空间确实是不连续分布的，但它不像链表那样碎片化。

它的结构分为两层：

中控表 (Map)：这是一个连续的数组，里面存储的不是数据，而是指向各个缓冲区（Buffer）的指针。

缓冲区 (Buffer)：这是真正存数据的地方。每个缓冲区是一块连续的、固定大小的内存。

因此CPU预取较慢一点，因为需要先查中控表，再查缓冲区。但是数据改动增加快只需要在中控表加个指针，再开个新块即可。











    

   
   
   



  
   
