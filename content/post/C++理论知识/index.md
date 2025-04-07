---
title: C++理论知识
# description: Welcome to Hugo Theme Stack
slug: CPPlilunzhishi 
date: 2025-04-07 00:00:00+0000
# image: filesystem.jpg
categories:
    - 面试经验
    - 理论知识
tags:
    - C++
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
## 指针和引用

指针是占用内存地址的独立变量，内存中存储的就是一个地址。而引用是某个变量的别名，不占用独立内存空间。

指针定义时可以不初始化，但是最好初始化，指向的内容可以变，存在空指针。引用定义时必须初始化，绑定后不可改变，不存在空引用。

使用指针访问地址内容时需要星号解引用，引用不需要。

指针一般用于动态内存分配，数组操作以及函数参数传递。引用通常用于函数传参，操作符重载和创建别名。

## 数据类型

short至少16位、int至少与short一样长、long至少32位、long long至少64位。一般int就是32位和long一样。为了避免歧义可以使用uint32_t/int32_t

## 关键字

### const关键字用于修饰只读变量

必须定义时就赋初值。

对于指针：
1. 底层const是const int *a = 地址或者int const *a =地址，指针可以指向别的地址，但是a的内容不能变
2. 顶层const是int *const a = 地址，指针只能指向这个地址，但是地址存放的内容可以变

常量引用：const int &a = b引用的值是常量，不能通过引用修改。

常量成员函数：
```cpp
class Hero {
public:
    int getBlood() const {...}
};
```
大括号之前写const，表示不会修改对象的非静态成员变量，但是可以修改静态成员变量。只能调用其他常量成员函数，不能调用非const成员函数（避免事实修改）

常量对象：const Hero hero(...);必须在创建对象时调用构造函数，使用初始化列表。创建后不可修改成员变量，只能调用常量成员函数。

常引用参数：void foo(const string& str)确保参数不会被修改，可以绑定右值。

常量指针参数：也分为底层和顶层，保护指针指向地址不变或指针地址指向值不变。

### const和constexpr

const表示只读，而constexpr表示常量。

• const的值可以由函数返回值等在运行时初始化，而constexpr必须在编译时由常量表达式初始化

• const可以修饰成员函数，表示不改变非静态成员变量的值，但是不能修饰普通函数；constexpr可以修饰函数，意思时能在编译期调用并返回函数值，注意不能动态分配内存，必须非常简单。
```cpp
constexpr int factorial(int n) {
    return (n <= 1) ? 1 : n * factorial(n - 1); // C++14起支持递归[1,6](@ref)。
}
constexpr int result = factorial(5); // 编译期计算结果为120[1,4](@ref)。• constexpr可以将一些计算提前到编译期做，避免运行时开销。constexpr还可以用于检查一个表达式是否是常量表达式
```
### static关键字定义静态变量和函数

静态变量：static修饰的局部变量或全局变量，生命周期和进程一样长，不会因离开函数作用域而销毁，默认初始化为0。可以用于计数函数调用次数、在递归函数中缓存中间结果等。

全局静态变量和全局变量的区别在于全局静态变量不能在其他文件中通过extern关键字引用。

静态函数是static修饰的普通函数，作用域仅限当前文件，不可被其他文件通过extern声明。

静态成员变量是类内用static修饰的成员变量，属于类而不是对象。所有实例共享同一份数据，必须在类外单独初始化。

静态成员函数是类中使用static修饰的成员函数，静态成员函数不能直接访问非静态成员或调用非静态成员函数，可以通过类名调用而需要创建类的示例.

静态成员或静态成员函数可以通过实例调用，但是不合适。1. 容易混淆静态和非静态成员2.最好使用类名调用

const和static不能一起使用，因为静态成员函数不含有this指针，而const成员函数必须具体到某一实例。

### Mutable

const函数不修改对象成员，但是某些成员如计数器等要求修改，则可以用，mutable修饰，在const中可以修改这些成员。

### define和typedef

• define只是字符串替换，没有类型检查，作用于编译的预处理阶段，可以防止头文件重复引用（写头文件的开头结尾有#ifndef和#endif）。不分配内存，有多少次使用就有多少次替换

• typedef为类型定义别名，作用于编译处理阶段，有对应的数据类型，在静态区分配空间，程序运行过程中内存中只有一个拷贝

### define和inline

• inline用于定义内联函数，在调用函数处将函数展开，因此能够避免调用函数带来的压栈、跳转等开销

• 必须是小的函数体，不能存在循环，必须减少条件判断

• inline先编译再插入，在编译阶段。define在预处理阶段展开

### define和const

const和define都可以用于定义常量

• const在编译阶段、define在预处理阶段

• const定义的常量放在内存中，define直接替换

• const常量有类型，define没有

### using和typedef

都可以定义别名

typedef int Integer

using Integer = int建议使用using，因为可读性更强，更简洁。

using还能用来引入命名空间，引入基类成员。

通过using namespace ...将命名空间内所有成员引入当前作用域，因此不需要前缀。但是最好不要在头文件中使用，以免导致命名冲突。过度使用会降低可读性，不好确定成员是哪个命名空间的。

通过using std::cout可以引入特定成员，减少命名冲突风险。

（要不跳过吧）通过在派生类中声明using Base::Base 继承基类构造函数，避免重复定义。在派生类中使用Base::func可以将基类中private的func提升为派生类的public成员。当派生类定义同名成员时可以显式保留基类成员。

## 命名空间

命名空间通过将代码元素封装在特定作用域内避免变量的命名冲突。

例如定义两个命名空间，可以通过两个冒号解析

```cpp
namespace NS1 {int *a};
namespace NS2 {int *a};
NS1::a;
NS2::a;
```

### new和malloc都可以用于动态内存分配

• new分配对象大小的内存后调用构造函数初始化对象，返回对象指针；malloc需要指定分配大小，并且返回void指针

• 空间不足时，new抛出异常，而malloc返回空指针

• new必须用delete释放，调用析构函数；malloc必须用free释放，不调用析构函数

• malloc在堆上分配空间，new在自由存储区分配（允许自定义内存池，但一般也是堆）。

malloc在底层使用malloc分配，但是会额外完成构造逻辑，通过重载new可以将自由存储区指向其他区域。

### delete和free

• delete会调用析构函数，free仅释放内存，可能导致资源泄漏，如未关闭文件等。

• delete可以正确释放new直接创建的数组，free必须手动循环释放数组的每一项

### extern

声明在其他文件定义的非静态变量或非静态函数

i++, ++i

i++返回的是i，++i返回的是i+1。

i++创建了一个临时对象保存i，并返回临时对象。

### std::atomic

确保对变量的读写、修改等操作不可分割，操作过程中不会被其他线程打断。

### struct和class

struct和class的功能几乎完全一样，只是默认行为不同。struct默认成员、继承方式都是公有的，class默认私有。

## 函数指针

指针指向的是函数的地址（代码段），允许在运行时动态选择要调用的函数。

int (*func)(int, int)

如果省略括号就会编程返回指针的函数

可以用于实现1. 回调函数机制，将函数指针作为参数传递2.根据不同参数选择不同函数处理

## 强制类型转换

四种强制类型转换的关键字：static_cast，dynamic_cast，reinterpret_cast和const_cast

• static_cast用于截断或扩展整数、上行转换（将派生类转换为基类）
    ◦ 编译时检查，无运行开销
    ◦ 下行转换不安全，因为没有动态类型检查
    ◦ 支持void*和整数之间的互换

• dynamic_cast用于动态类型转换，可以将基类指针转换为派生类指针
    ◦ 运行时有类型检查，转换失败返回空指针，引用的话抛出异常
    ◦ 通过虚函数表获取对象的实际类型信息，检查目标类型是否是实际类型

• const_cast用于去除const属性，但是如果对象是真正的常量，则会导致未定义行为。编译器也有可能将const优化成字面量，导致修改无效。

• reinterpret_cast用于无关类型的转换，如int转为char，基于内存的位重新解释，绕过类型系统，高风险。

static是按类型规则逻辑处理数据，但reinterpret是物理上重新解释内存

## C++内存管理

### 堆和栈

栈是由编译器自动管理，存放局部变量、函数参数以及返回地址，通过移动栈指针分配内存，栈上变量的生命周期和所在函数相同。

堆由用户手动管理，存放动态分配数据，生命周期由用户控制。

栈分配得块，因为进程本身就有一段连续的段空间，仅需调整栈指针就可以分配和回收。每个线程有独立的栈，栈的容量较小。

堆不是先进后出的，因此每次从堆中分配内存时需要先通过空闲内存块链表搜索合适大小的内存块，如果没有再移动堆顶指针。可能还会触发碎片整理。同一个进程的不同线程共享堆空间，因此可能需要同步机制。

### 内存分区
栈

堆

全局/静态：bss未初始化，data初始化

只读数据段.rodata

代码段

### 内存泄漏

内存泄漏是程序分配内存后失去了对该内存的控制，因此无法释放。

• 堆内存泄漏：通过malloc，new等动态分配的对象忘记释放

• 系统资源泄漏：程序使用系统分配的资源，如socket等，却没有释放或关闭。

• 对象泄漏：没有正确定义析构函数或没有将基类的析构函数定义为虚函数，这样导致基类指针指向的派生类对象会调用基类的析构函数，从而无法正确释放派生类资源

防止内存泄漏的方法有：将内存的分配和释放封装在类中，从而避免忘记释放；使用智能指针。

### 智能指针

智能指针用来避免内存泄漏，生命周期后会自动释放资源，不需要用户手动释放。

3种智能指针：
• 独占智能指针unique_ptr：一个独占指针唯一拥有资源，别的指针不能指向他管理的内存。没有额外开销，性能接近于原生指针
• 共享智能指针shared_ptr：允许多个指针指向同一个对象，利用计数器机制管理。只有最后一个指针销毁时才释放。但是可能导致循环引用
• 弱引用指针weak_ptr：不增加引用计数，因此不会导致循环引用，例如父节点通过shared_ptr引用子节点，子节点通过weak_ptr引用父节点

### 野指针和悬浮指针

野指针是指向无效或位置内存地址的指针，使用野指针会导致不可预知的行为。

1. 未初始化的指针，其值为随机内存地址，如栈和堆中的垃圾地址

2. 释放后未置空的指针，内存值已不可用，如释放了参数指针

3. 返回局部变量的指针（变量在函数作用域外被销毁）

4. 指针越界访问（数组越界、分配内存之外等）

悬空指针是指向已释放内存的指针，是野指针的一种。

### 内存对齐

内存对齐是指数据在内存中的存储起始地址是某个值的倍数。

结构体中可能包含不同类型的变量，变量按照声明的顺序放置，第一个成员和整个结构体的地址相同。

出于CPU访问效率的考虑，变量的起始地址应当在长度的整数倍上，比如4字节的int，起始地址应该是4字节的整数倍。

### 乱序执行

• CPU层面，允许不共享数据的指令并行执行

• 编译器层面允许调整无数据以来的指令顺序、将循环体展开为多个迭代以减少分支预测、分支预测优化。

单线程场景可以提升执行效率，多线程场景可能导致数据竞争，线程同步关系颠倒可能导致逻辑错误

可以采用原子类型std::atomic保证操作的原子性、插入内存屏障指令限制某些重排序。

## stl

string实际不属于stl，而是C++标准库的独立组件
```cpp
// 构造
string s1; // 空，默认构造函数
string s2("Hello"); // 默认构造函数
string s3(5, 'c'); // 默认构造函数
string s4(s2); // 拷贝构造

// 子串
string s5 = s2.substr(0,3); // 起始，偏移量
getline(istream &in, string &s);size(), length()返回字符个数
empty()判断是否空
capacity()返回分配的存储容量，不含'\0'
resize(int len, char c)就是truncate
clear()清空字符串内容，但是保留容量
reserve()预分配内存减少扩容开销
```

[]通过下标访问，不检查越界。

at()通过下标访问并检查越界

front(), back()访问首尾字符

c_str()返回以\0结尾的c风格字符串

+= / append()追加字符串或字符

push_back()追加字符

insert(pos, str)在pos处插入str

replace(pos, len, str)替换从pos开始的len个字符为str

erase(pos, len)删除从pos开始的len个字符

find(str, pos)从pos开始查找str，返回首次出现的位置

rfind(str, pos)反向查找

find_first_of(str)返回包含str任意字符的首个位置

查找失败返回npos（-1的无符号形式）

==和<>直接比较（字典序）

compare(str)返回0（相等），正数（大于），负数（小于）

begin(), end(), rbegin(), rend()等迭代器

## 程序编译过程
• 源代码
• 预处理
    ◦ 宏替换、包含头文件（将头文件内容插入源文件）、删除注释
• 编译
    ◦ 生成汇编代码.s文件。词法分析、语法分析、语义分析、中间代码生成与优化
• 汇编
    ◦ 将汇编代码翻译为机器指令，生成目标文件.o，有代码段、数据段、符号表
• 链接
    ◦ 链接器将多个目标文件和库文件合并为可执行文件
    ◦ 跨文件符号解析、地址重定位
    ◦ 静态链接将库代码拷贝到可执行文件、动态链接在运行时加载共享库

## 初始化

int a = 1; int a(1)和int a = {1}的区别

int a = 1是复制初始化，编译器先将右值转换为目标类型对象，然后调用拷贝构造函数初始化左侧对象。

int a (1)调用了int类的构造函数，1是参数s。

int a = {1}是列表初始化，可以避免隐式类型转换带来的窄化，因为编译器会检查是否窄化。

## 右值引用用来实现移动语义和完美转发

左值是占用一定内存空间的，可以取地址的值。右值是不占用地址空间的，一般是立即数等。

左值的生命周期取决于变量类型，而右值一般执行完所在语句后就死亡。

右值引用，例如int && a = 1；可以将1这个右值的生命周期延长到作用域结束。

右值引用可以实现移动语义，也就是将资源从一个所有者移动到另一个所有者，而不需要拷贝。例如某个函数的返回值是一个对象，而函数的返回值是一个右值，因为函数结束后，变量所在作用域就结束了。因此本来调用函数的语句执行完后返回值就死亡了，是一个右值。然而使用右值引用可以延长其生命周期，可以用其初始化另一个对象。该类具有移动构造函数的情况下，会把对象的成员直接转移给新对象，而不是调用拷贝构造函数复制。

完美转发一般用于模板函数，可以在传递参数时原样转发，而不是构造一个新的参数。通过std::forward实现

移动语义只适合实现了移动构造函数的类，对于int，double等还是会复制

```cpp
int&& rv = 42;          // 绑定右值
std::string s = "hello";
std::string&& s_rv = std::move(s);  // 转换为右值引用

class MyString {
public:
    // 移动构造函数
    MyString(MyString&& other) noexcept 
        : data(other.data), size(other.size) {
        other.data = nullptr;  // 原对象资源置空
        other.size = 0;
    }
private:
    char* data;
    size_t size;
};

std::vector<MyString> vec;
MyString tmp("test");
vec.push_back(std::move(tmp));  // 调用移动构造函数而非拷贝

void process(int& x) { /* 处理左值 */ }
void process(int&& x) { /* 处理右值 */ }

template <typename T>
void relay(T&& arg) {
    process(std::forward<T>(arg));  // 保持arg的原始类型
}

int main() {
    int a = 10;
    relay(a);        // 调用process(int&)
    relay(20);       // 调用process(int&&)
}
```
## 面向对象特性

CPP的面向对象特性包括 封装、继承、多态

### 封装

将数据和对数据的操作捆绑在一个类中，通过访问控制隐藏内部实现，对外暴露有限接口。作用：数据安全，避免外部直接修改或访问私有成员；模块化，类便于复用和维护；简化使用，用户无需了解具体实现
访问权限

public - 公有成员可以被外部访问

protected - 保护成员只能被类内部、友元函数和派生类函数访问

private - 私有成员只能被类内部访问

友元：允许非成员函数或其他类访问protected和private。单向的，不具有传递性。

### 构造函数和析构函数

构造函数在对象创建时自动调用初始化数据成员，与类名完全相同，没有返回值，可以重载。

构造函数不能是虚函数。

• 默认构造函数
    ◦ 无参数时自动构造，不会初始化int等内置类型。string有自己的构造函数所以初始化为空字符串
    ◦ 如果没有定义任何构造函数则会生成默认构造函数，如果有构造函数则需要自己定义不带参数的构造函数
    ◦ 同时定义无参构造函数和全缺省构造函数会产生冲突（a(){}和a(int a  = 0){}）

• 带参数构造函数
    ◦ 自己定义的构造函数

• 拷贝构造函数
    ◦ 使用一个对象构造一个新对象
    ◦ 浅拷贝：对指针成员仅复制指针地址，若原对象销毁则指针内容会释放
        ▪ 需要自定义深拷贝函数，创建空间并将值复制过来

• 移动构造函数
    ◦ 通过资源转移将原对象的成员转移到自身，不需要拷贝。原对象不再拥有资源，可以安全删除。
    ◦ 参数必须是右值引用，如果要处理左值需要用std::move显式转换

析构函数在对象生命周期结束时释放资源，包括局部对象离开作用域、delete删除对象。函数名为波浪号加类名，不允许带参数，不可重载。自动调用。默认析构函数不会释放资源，因此必须自己定义析构函数。基类的析构函数需要定义为虚函数。

构造顺序：基类构造函数 - 成员构造函数 - 自己的构造函数

析构顺序反向：自己析构函数 - 成员析构函数 - 基类析构函数

构造函数不能是虚函数，基类的析构函数需要设为虚函数

构造函数不能是虚函数，因为虚函数有虚函数表动态绑定，而虚函数表由对象的虚指针指向。虚指针在构造函数中初始化，因此构造函数无法使用虚指针。另一方面，虚函数的语义是根据对象的实际类型实现运行时多态，而构造函数正是用于确定对象的具体类型，因此有语义冲突。

基类的析构函数需要是虚函数，否则通过基类指针或引用释放派生类对象时会调用基类的析构函数，导致无法正确释放资源。但是如果是子类指针指向则可以正确析构。

### 继承

允许派生类继承基类的非私有属性和方法。作用：派生类可以复用基类代码减少重复实现、派生类可以扩展基类代码实现更多功能

• 继承方式

    ◦ public：表达特例关系，例如学生是一种人，因此学生具有人的所有成员

        ▪ 成员属性保持不变

    ◦ protected：可以用于实现多级继承中的中间层，既不对外暴露成员，又允许再次继承

        ▪ public和protected变为protected

    ◦ private：实现新功能，例如用链表实现栈，但是不暴露链表接口，只保留栈的接口

        ▪ public和protected变为private

    ◦ 三种继承方式，派生类都不能访问基类的private成员

### 多重继承

允许一个派生类同时继承多个基类，基类构造函数的调用顺序取决于派生类声明时的排列顺序，而不是参数列表顺序。析构函数的调用顺序于构造函数相反。

允许组合不同基类的功能，例如基类手机、电脑等和基类输出电池容量（纯接口类）。

• 二义性

    ◦ 多个基类有重名成员，需要用两个冒号指明哪个基类

• 菱形继承

    ◦ 多个中间基类来自同一个基类，则派生类继承多个相同成员副本

    ◦ 确保中间基类使用虚继承，则派生类仅保留一份副本

        ▪ 虚继承就是继承时加virtual关键字，虚继承的用处就是解决菱形继承
```cpp
class A {
    int a;
    int b;
}
class B : virtual public A {}
class C : virtual public A {}
class D : public B, public C {}
```
• 最好用组合代替多重继承，避免深层次多重继承导致难以维护

派生类会调用基类构造函数和析构函数

创建派生类对象时会调用基类构造函数。如果有默认基类构造函数会隐式调用，否则必须显式调用。

```cpp
class A {
    int a;
public:
    A(int x) : a(x) {}
};
class B : public A {
    int b;
public:
    B(int x) : b(0) {A(x);}
    B(int x, int y) : b(y) {A(x);}
};
```

构造函数调用顺序：基类构造函数 - 派生类成员构造函数 - 派生类构造函数

析构时会自动调用派生类的析构函数，再自动调用基类的析构函数。注意，如果用基类指针指向派生类，而基类的析构函数未声明为虚函数，则导致派生类析构函数未调用。因此最好将基类的析构函数声明为虚函数。

## 多态

同一操作作用于不同对象表现不同特性，分为编译时多态和运行时多态

• 编译时多态：无运行时开销，但无法根据运行情况调整行为
    ◦ 函数/运算符重载/模板
        ▪ 函数重载：同一作用域中定义多个同名函数，参数列表不同
        ▪ 运算符重载：自定义类对象的运算操作
```cpp
// 左操作数是当前类的对象，则用成员函数重载
class vector {
private:
    int x;
    int y;
public:
    vector operator+(const vector &v) {
        // 隐含this->x
        return vector(x+v.x, y+v.y);
    }
};
// 左操作数不是当前类对象，用友元函数重载。注意友元关系无法继承
class array {
private:
    int *s;
    int size;
public:
    friend ostream &operator<<(ostream &os, const array &arr) {
        for (int i = 0; i < arr.size; ++i) {
            os << arr.s[i]<<" ";
        }
    }  
};
```
        ▪ 模板：泛化参数类型从而定义可以处理多种数据类型的函数或类
```cpp
template <typename T>
class stack {
    T data[100];
// 函数操作
};

T add(T a, T b) {
// 如果a是整型，b是浮点就会编译失败，注意这是编译时多态，不会等到运行时
    return a + b;
}
```
• 运行时多态：允许程序通过对象的实际类型调用对用的函数
    ◦ 重写
        ▪ 派生类重写并覆盖基类的函数，根据对象实际类型调用该对象的函数。
        ▪ 基类的函数必须声明为虚函数，参数列表、返回类型等声明必须完全一致，但是访问权限可以不同（变得更开放）
        ▪ 不能重写private函数（根本看不见），静态方法不能重写为非静态方法
        ▪ 重写时可以在大括号前写override，但是也可以不写
            • 不写override则编译器不会检查签名是否一致，若不一致不会报错，而是将重写的函数视为新函数（重定义），导致基类函数隐藏，无法使用多态（基类指针指向派生类对象，只能使用基类函数， 派生类指针指向派生类对象，只能使用派生类函数，但可以用::显式调用基类函数。将基类函数声明为virtual，则为省略override的重写）。

不将基类函数声明为虚函数，则为重定义。

基类指针可以指向派生类，但派生类指针不能指向基类

在派生类对象的内存布局中，基类部分在派生类部分前部，因此基类指针可以访问派生类对象中属于基类的成员而不会越界。

通过虚函数机制，基类指针可以调用派生类重写的虚函数。

因此，基类析构函数不设置为虚函数的话，基类指针无法调用派生类的析构函数。

派生类指针无法指向基类对象，因为基类对象不包含派生类特有的成员，因此会有越界风险。不允许隐式转换，强制转换可能导致运行时错误。

### 虚函数

#### 虚函数表存放

虚函数表指针和对象一起存储，放在对象的起始地址。

虚函数表在编译时生成，本身放在只读数据段。

虚函数内容放在代码段。

构造函数会隐式初始化对象的虚函数表指针。

通过基类指针调用虚函数时程序会通过对象的虚函数表指针找到虚函数表，根据函数在表中的偏移量调用正确的派生函数。

派生类会继承基类的虚函数表，若重写虚函数则替换虚表中的地址。

虚函数表是所有的类共享，还是每个类有自己的虚函数表

每个类有自己的虚函数表，编译器会为每个包含虚函数的类单独生成虚函数表，同一类的不同对象共享一个虚表，不同的类不共享一张虚表。

纯虚函数

纯虚函数没有函数体，只有=0声明。含有纯虚函数的基类是虚基类（抽象类），虚基类无法实例化，只能用于派生类，派生类必须要实现否则派生类也是虚基类。用于提供一种接口规范。

```cpp
class A{
public:
    virtual void func() {
        // 实现，允许重写
    }
    virtual void func2() = 0;
    // 纯虚函数，必须重写
}
```
不能被声明为虚函数

1. 构造函数

2. 普通非成员函数（只能重载，不能重写）

3. 静态成员函数（无法访问虚函数表（虚函数指针在对象首部））

4. 友元函数（友元函数不能继承）

5. 内联成员函数（内联在编译时就展开了，而重写是运行时的多态）

## 空类的大小

空类是不包含非静态成员变量、虚函数或虚基类的类

空类会默认生成默认构造函数、拷贝构造函数、默认析构函数、赋值运算符和取地址运算符。

空类可以用来作为接口占位，具体逻辑由不同子类完成。

• 默认为1字节，因为每个对象必须有唯一的地址

• 当空类作为基类时，编译器可能会用空基类优化，就是将占位字节和派生类成员合并。但是如果继承了多个空基类，则不能这样优化

• 如果包含虚函数，则需存储虚函数表指针，大小是指针的大小（64位系统8字节）

• 空类作为其他类成员则可能需要内存对齐

• 静态成员变量存放在数据段，成员函数存放在代码段，因此不影响类大小。

## 运算符重载

重载运算符本质上是重载函数，只是函数名为operator+或其他运算符。

使用运算符比函数调用的形式更直观。

注意：

1. C++中作用域解析::， 成员访问.，三目条件运算符?:等不可重载

2. 重载运算符不能改变优先级，比如+的优先级低于*

3. 不能改变运算符的操作数，二元运算符必须有两个参数

4. 重载的运算符必须至少有一个操作数是用户自定义类型，也就是int double等内置类型不能重载

5. 需要注意操作符在语义上要直观，相关的操作符也要重载，比如重载了==，也应该重载!=

## 段错误Segmentation Fault

段错误的本质是程序运行访问了非法地址或违反了访问权限，导致被操作系统强行终止。

• 访问未分配的内存（释放指针后又使用）

• 越界访问

• 修改只读数据

MMU检测到非法访问后，向操作系统发送错误信号sigsegv，进程终止并生成core文件，这叫做核心转储。

要注意指针的使用和边界检查、是否栈过深等。

## lambda表达式

lambda表达式是一种匿名函数工具，对于比较简短的逻辑，可以写一个lambda表达式代替定义一个函数。

lambda表达式的完整形式是[捕获列表](参数列表)mutable->返回类型{函数体}

捕获列表表示能够访问哪些外部变量，=：按值捕获所有外部变量、&：按引用捕获所有外部变量、x, &y：x按值，y按引用、this：捕获当前对象的成员变量

mutable：允许修改按值捕获的变量部分，默认是const不能改变

-> int 返回值是int，也可以不写，让编译器推导

既然捕获了所有变量，就可以不写参数列表。

## 多线程、条件变量condition variable和mutex

条件变量是实现线程间同步的工具，引入condition_variable头文件，和mutex配合使用。
通过wait等待，通过notify_one和notify_all唤醒
cv.wait(lock)或者cv.wait(lock, 一个lambda表达式谓词)。前者：释放lock并等待notify；后者：释放lock并等待notify，只有在lambda表达式为真才唤醒。
std::thread用来创建线程，接收函数名、lambda表达式。线程也是一种对象。thread不可复制，可以移动。这是为了防止多个线程对象管理同一线程资源造成竞争。
```cpp
#include <iostream>
#include <thread>
using namespace std;

void hello(int val) {
    cout<<"hello, "<<val<<endl;
}

// 线程参数按值传递，线程在构造函数调用时立即启动
thread t1(hello, 1);
thread t2([](){cout<<"lambda"<<endl;});
class MyClass {
public:
    void func(int val) {
        cout<<"MyClass: "<<val<<endl;
    }
};
// 绑定对象函数
MyClass obj;
thread t3(MyClass::func, &obj, 34);
int main() {
    // join的意思是主线程等待该线程完成再继续进行
    t1.join();
    t2.join();
    t3.join();
    return 0;
}
```
thread的join方法是让主线程阻塞等待该线程完成，用来同步。detach方法是将子线程从主线程的管理分离，在后台独立运行。由C++库自己管理回收。detach后无法用原来的thread对象管理线程。
thread在销毁之前必须被join或者detach，必须保证线程函数运行期间线程对象有效或者detach
### lock_guard和unique_lock

lock_guard创建就上锁，作用域结束析构时候解锁，不能中途手工解锁，也不能复制

unique_lock允许创建时不上锁而是延迟上锁，可以随时手动加锁解锁(unique_lock<mutex> lock(mtx); lock.unlock(); lock(lock);)，析构时自动释放，不可复制可以移动，条件变量必须配合unique_lock使用

### 互斥锁和条件变量程序例子

多线程交替打印数字

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <condition_variable>
using namespace std;

mutex mtx;
condition_variable cv;
bool flag = true;

void print_odd() {
    // 打印奇数
    for (int i = 1; i < 100; i += 2) {
        unique_lock<mutex> lock(mtx);
        // 在使用条件变量之前必须持有锁，否则对条件变量的检查不是原子的
        cv.wait(lock, [](){return flag;});
        cout<<"odd: "<<i<<endl;
        flag = false;
        cv.notify_one();
    }
}

void print_even() {
    // 打印偶数
    for (int i = 2; i <= 100; i += 2) {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [](){return !flag;});
        cout<<"even: "<<i<<endl;
        flag = true;
        cv.notify_one();
    }
}
int main() {
    thread t1(print_odd);
    thread t2(print_even);
    t1.join();
    t2.join();
    return 0;
}
```
生产者消费者问题

只用了一个cv，可能导致无效唤醒。因此可以用两个条件变量，分别专门唤醒消费者和生产者。

```cpp
#include <iostream>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <queue>
#include <thread>
using namespace std;

const int TOTAL_ITEMS = 20;
const int BUFFER_SIZE = 5;
queue<int> q;
atomic<int> done = false;
mutex mtx;
condition_variable cv;

void producer() {
    for (int i = 0; i < TOTAL_ITEMS; ++i) {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [](){return q.size() < BUFFER_SIZE;});
        q.push(i);
        cout<<"producer: "<<i<<" q.size: "<<q.size()<<endl;
        cv.notify_one();
    }
    done = true;
    cv.notify_all();
}

void consumer() {
    while (true) {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [](){return !q.empty() || done;});
        if (q.empty() && done) {
            break;
        }
        cout<<"consumer: "<<q.front()<<" q.size"<<q.size()<<endl;
        q.pop();
        cv.notify_one();
    }
}
int main() {
    thread t1(producer);
    thread t2(consumer);
    t1.join();
    t2.join();
    return 0;
}
```
读写锁

C++17的shared_mutex可以直接实现读写锁（写者优先）

```cpp
#include <iostream>
#include <shared_mutex>
#include <thread>
#include <vector>
using namespace std;

class ThreadSafeResource {
private:
    int data = 0;
    shared_mutex sm;
public:
    void read() {
        shared_lock<shared_mutex> l (sm);
        cout<<"read: "<<data<<endl;
    }
    void write(int d) {
        unique_lock<shared_mutex> l(sm);
        data = d;
        cout<<"write: "<<data<<endl;
    }
};

int main() {
    ThreadSafeResource TSR;
    vector<thread> v;
    for (int i = 0; i < 5; ++i) {
        // 创建5个读者线程
        v.emplace_back([&](){TSR.read();});
    }
    // 创建写者线程
    for (int i = 0; i < 5; ++i) {
        v.emplace_back([&](){TSR.write(i);});
    }
    for (int i = 0; i < 10; ++i) {
        v[i].join();
    }
    return 0;
}
```
C++11中，使用条件变量和mutex实现读写锁。

```cpp
#include <iostream>
#include <mutex>
#include <condition_variable>
#include <vector>
#include <thread>
using namespace std;
class RWL {
private:
    bool write_active;
    int nr_reader;
    mutex mtx;
    condition_variable cv;
public:
    RWL() : nr_reader(0), write_active(false) {}
    void readLock() {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [this](){return !write_active;});
        nr_reader++;
    }
    void readUnlock(){
        unique_lock<mutex> lock(mtx);
        nr_reader--;
        if (nr_reader == 0) {
            cv.notify_one();
        }
    }
    void writeLock() {
        unique_lock<mutex> lock(mtx);
        // 不能有其他写者，不能有读者
        cv.wait(lock, [this](){return !write_active && nr_reader == 0;});
        write_active = true;
    }
    void writeUnlock() {
        unique_lock<mutex> lock(mtx);
        write_active = false;
        cv.notify_all();
    }
};
int main() {
    RWL rwl;
    int data = 0;
    vector<thread> v;
    for (int i = 0; i < 5; ++i) {
        // 5个读线程
        v.emplace_back([&, i]{
            rwl.readLock();
            cout<<"reader "<<i<<": "<<data<<endl;
            rwl.readUnlock();
        });
    }
    // 5个写线程
    for (int j = 0; j < 5; ++j) {
        v.emplace_back([&, j]{
            rwl.writeLock();
            data = j;
            cout<<"writer "<<j<<": "<<data<<endl;
            rwl.writeUnlock();
        });
    }
    for (int k = 0; k < 10; ++k) {
        v[k].join();
    }
    return 0;
}
```

## 原子变量

对原子变量的读写修改时不可中断的，两个线程同时递增一个原子变量不会导致出错，这是硬件上的CPU指令保证的。

## 函数名是地址

函数名是函数的地址，在代码中会隐式转换成指向函数的指针

## STL

STL包括算法、容器和迭代器、仿函数、适配器和分配器。

• 容器是存放数据的各种数据结构，包括vector，deque、set、map等

• 算法通过迭代器操作容器数据，包括sort、find等常用算法

• 迭代器是访问容器元素的的抽象或者广义上的指针，重载了*、->和++等运算符

• 仿函数是重载了小括号的对象或者结构体，可以像调用函数一样使用这种对象。

    ◦ 内置的仿函数有算数运算（plus<T>，multiplies<T>等）、关系和逻辑运算（less<T>, equal_to<T>等，用于排序和条件判断）

    ◦ sort(v.begin(), v.end(), greater<int>())可以实现降序排列，也可以自定义一个bool cmp(int a, int b) {return a>b;}然后调用sort(v.begin(), v.end(), cmp);

• 适配器通过容器实现更高级的功能，例如stack和queue都是借助deque容器实现的

• 分配器用于管理内存的动态分配和释放

STL的优点在于具有高可重用性（采用模板类和模板函数，更通用）、高性能（精心设计的低复杂度算法，利用红黑树等复杂底层结构）、高移植性（使用STL编写的模块可以方便移植到其他项目，而不需要顺带移植很多实现函数）。

### 常用容器

• pair<T1, T2> p

    ◦ 定义为一个struct，通过pair.first和pair.second访问两个成员

    ◦ 用来操作关联容器，例如可以将一个pair插入map

• vector

    ◦ 在堆中分配了一段连续的内存空间存放元素

    ◦ 迭代器first，end。first是起始元素位置，end是最后一个元素之后的位置。还有rbegin和rend。

    ◦ 扩容：vector.capacity()表示不分配新内存的情况下最多可以保存的元素个数（预分配的大小），vector.size()表示当前已经存储的元素个数，capacity永远大于等于size，相等时就会扩容。扩容就是动态
    
    申请一段新的连续空间并把数组迁移过去
    
        ▪ 固定扩容是每次在原capacity的基础上增加固定的容量，比如每次都加20，优点是浪费较少，缺点是可能需要多次扩容
    
        ▪ 加倍扩容是每次将capacity翻倍，则需要预留较多空间，但是减少迁移开销
    
        ▪ 注意扩容会使得原来的迭代器失效
    
    ◦ resize：改变当前容器中含有元素的数量，如果resize(len)中的len<=capacity则只调整size=len，如果len >capacity则将size和capacity均设置为len。新增元素调用默认构造函数。
    
    ◦ reserve(len)改变capacity，如果len < capacity就不做任何改变，len>capacity就扩容，包括迁移
    
    ◦ emplace_back直接在容器的内存空间内使用对象的构造函数，无需临时对象，而pushback需要先构造一个临时对象再拷贝到容器尾部，因此emplaceback更高效
    
    ◦ 操作
```cpp
    vector<int> v1;
    vector<int> v2(5, 10);// 5个10
    int arr[] = {1, 2, 3};
    vector<int> v3(arr, arr+3);// 迭代器或指针范围初始化
    vector<int> v4(v3); // 拷贝构造
    vector<int> v5 = {1, 2, 3, 4}; // 列表初始化
    vector<int> v6 {1, 2, 3, 4}; // 等号可以省略

    v1.push_back(4); // 尾部插入
    v1.emplace_back(5); // 尾部插入，但是emplace_back更高效
    v1.insert(v1.begin()+1, 10); // 把10插入到第二个位置
    v1.pop_back(); // 删除尾部最后一个元素
    v1.erase(v1.begin()+1); // 删除第二个元素
    v1.erase(v1.begin()+1, v1.begin()+3); // 删除第2，3个元素（左闭右开）
    v1.clear(); // 删除所有元素

    v1.at(1); // 检查越界
    v1[2]; // 不检查越界

    v1.resize(5); // 调整大小
    v1.resize(7, 42); // 调整大小并将 新增元素 初始化为42
    
    v1.reserve(100); // 预留空间
    v1.shrink_to_fit(); // 释放预留的未使用空间
    
    v1.size(); // 元素个数
    v1.capacity(); // 预留空间可以存放的元素个数

    for (int i = 0; i < v1.size(); ++i) {
        // 传统遍历
    }

    for (auto it = v1.begin(); it != v1.end(); ++it) {
        // 迭代器遍历
    }

    for (int i : v1) {
        // 简洁写法
    }

    sort(v1.begin(), v1.end()); // 升序遍历
    sort(v1.begin(), v1.end(), greater<int>()); // 降序遍历• list 环状双向链表
```
◦ 在任意位置插入删除元素不会影响其他元素的迭代器，迭代器只支持++和--，不能下标或跳跃（i+5）访问
```cpp
    vector<int> v = {1,5,5,6,8};
    list<int>l1;
    list<int>l2(5, 10); // 5个10
    list<int>l3(v.begin(), v.end()); // 其他容器的迭代器范围初始化
    list<int>l4(l3); // 拷贝构造

    l1.push_back(1); // 尾部插入 1
    l1.push_front(2); // 头部插入 2 1
    l1.insert(l1.begin(), 3); // 3 2 1
    l1.pop_back(); // 尾部删除 3 2
    l1.pop_front(); // 头部删除 2
    l1.erase(l1.begin()); //删除迭代器位置

    l1.sort(); // 必须用成员函数sort，不能用sort(l1.begin(), l1.end())，因为迭代器不支持随机访问
    l2.sort(); 
    l1.merge(l2); // 合并两个有序的链表，将l2合并到l1
    l1.reverse(); // 反转链表
    l1.unique(); // 删除链表相邻重复元素，因此需要先排序，否则只删除相邻的
    l1.unique([](int a, int b){return abs(a-b<5);}); // 删除相邻绝对值相差小于5的• list和vector
    ```

◦ list是双向链表，vector是数组

◦ list只能顺序访问，vector可以随机访问

◦ vector插入会导致扩容和迁移、迭代器失效，list不会

◦ vector只有扩容时申请内存，list每次插入都需申请内存

• deque双端数组（队列）

◦ 支持快速随机访问，但是没有vector快

◦ 内部是分段的连续空间，迭代器比vector复杂，为了提高效率可以先将内容复制到vector，排序后再拷贝回deque

◦ 底层由多个固定大小的连续内存块组成，通过一个中控数组管理这些缓冲区的指针

◦ 头尾的插入和删除可能涉及到分配新缓冲区，O1，中间的插入删除可能涉及缓冲区迁移 ，On

```cpp
    deque<int> dq;
    dq.push_front(1); //也可以emplace，同样原理emplace性能好，
                        //push只适合加入现成对象，emplace适合用参数构造

    dq.push_back(2);
    dq.front();
    dq.back();
    dq.pop_front();
    dq.at(0);
    dq[0];
    deque<int> dq2(5, 10);
    deque<int> dq3(dq2);
    deque<int> dq4(dq.begin(), dq.end());
    ```
• stack和queue
    ◦ 是基于deque的容器适配器,push 和 pop 操作均为 O(1)（访问首尾），不能通过迭代器访问
```cpp
#include <queue>
#include <stack>
stack<int> s;
s.push(1);      // 入栈
s.pop();        // 出栈
int top = s.top(); // 获取栈顶元素
bool isEmpty = s.empty(); // 判断栈是否为空
size_t size = s.size();    // 返回元素数量

std::queue<int> q；
q.push(10);         // 入队元素 10
q.emplace(20);      // 直接构造元素 20
q.pop();  // 删除队首元素
int head = q.front();  // 获取队首元素
int tail = q.back();   // 获取队尾元素
if (!q.empty()) {
    std::cout << "队列元素数量：" << q.size();
}
if (!q.empty()) { // 不检查空可能导致未定义行为
    int val = q.front();
    q.pop();
}```
• heap和priority_queue

◦ 堆是一种完全二叉树，大根堆是每个节点值一定大于子节点，小根堆是每个节点值一定小于子节点。

◦ heap一般配合vector使用

```cpp
    vector<int> v{5,8,3,7,1};
    // 默认生成大根堆
    make_heap(v.begin(), v.end());
    make_heap(v.begin(), v.end(), greater<int>()); // 小根堆

    // 向堆插入元素
    v.push_back(99);
    push_heap(v.begin(), v.end());
    push_heap(v.begin(), v.end(), greater<int>()); // greater<int>() 需要保持一致
    // 将堆顶元素移动到尾部
    pop_heap(v.begin(), v.end());
    v.pop_back();

    // 将堆转化为有序序列
    sort_heap(v.begin(), v.end());
    ```
    
    ◦ priority_queue默认底层容器是vector，自动维护堆结构（注意不是顺序结构），没有迭代器
    
    ◦ 堆的调整算法是Ologn

• map和set

    ◦ map和set底层采用红黑树实现，红黑树是一种不严格的平衡二叉搜索树，保证最长路径不超过最短路径的两倍，而AVL是严格的二叉平衡树。红黑树的旋转操作比较简单，适合有大量插入删除操作的场景，比如vma的管理
    
    ◦ 都是C++的关联容器，只通过接口访问元素，底层都是红黑树实现
    
    ◦ set判断一个元素是否存在，mao因是个好
    
    ◦ 插入删除查找时间Ologn，可以iter遍历 On
    
    ◦ map不可修改键，可以修改值；set不可修改键
    
    ◦ 默认按键升序排列，键唯一
```cpp
map<int, string> m;
m.insert(make_pair(1, "Alice"));  // 推荐，避免临时对象构造[6](@ref)
m[2] = "Bob";     

set<int> s;
s.insert(10);  

auto it = m.find(1);              // 返回迭代器，未找到返回`end()`[1,6](@ref)
if (m.count(1) > 0) { ... } 

m.erase(1);                       // 按键删除，返回删除元素数量[6](@ref)
s.erase(s.begin());  

for (auto it = m.begin(); it != m.end(); ++it) {
    cout << it->first << ": " << it->second << endl;
}

for (const auto& kv : m) { ... }  // 自动解引用为键值对[7](@ref)• unordered_map和unordered_set
```
◦ 哈希表实现，无序关联容器
    ◦ 插入查找删除复杂度O1
## vector是线程安全的吗
vector不是线程安全的，多个线程同时修改线程会导致数据竞争。需要锁机制保护。

线程不安全的情况有：同时修改容器结构，如push_back、insert、erase等；混合读写操作，如一个线程读、另一个线程写同一个元素，或者一个线程读、另一个线程修改容器结构

## 泛型编程

泛型编程是通过模板机制允许在编写代码时不指定具体的数据类型，而是在编译时根据具体类型生成特定代码，避免重复编程。

类模板是创建通用类的机制，使用template<T>声明，类内部用T表示通用类型参数，函数模板用于定义通用函数，也是声明然后用T。但是类模板实例化时必须指定类型参数（例如stack<int> s），不能自动推导，函数模板可以根据调用的参数类型自动推导（例如add(3, 5)会生成一个int版本的add）

全特化是为模板的所有参数指定具体类型，也就是没有用泛型编程，偏特化是部分指定模板参数或者添加类型修饰，如指针和引用。偏特化只适用于类模板，不适合函数模板。

```cpp
// 偏特化
template <typename T>
class Stack<T*> {  // 特化为指针类型
private:
    T** data;
public:
    void push(T* item) { /* 指针特殊处理 */ }
};
```
## 类型推导

auto可以让编译器在编译期推导出变量的类型。必须马上初始化，定义多个变量时不能有二义性，不能用作函数参数，类中不能用作非静态成员变量（非静态成员变量的初始化发生在对象创建时，即构造函数执行阶段，不是编译期。而静态成员需要在类外定义的时候初始化），不能定义数组，可以定义指针，不能推导模板参数。单纯使用auto时推导会忽略引用和cv（const， volatile）限定，也就是推导出对象的原始类型。但是使用auto&或者auto*的时候会保留引用和cv限定

```cpp
int main() {
    int x = 10;
    const int& crx = x;
    auto a = crx;       // a 是 int（忽略引用和 const）
    a = 20;             // 合法：a 是普通 int
    // crx = 20;        // 非法：crx 是 const 引用
}

int main() {
    int x = 10;
    const int& crx = x;
    auto& b = crx;      // b 是 const int&（保留引用和 const）
    // b = 20;          // 非法：b 是 const 引用
}
```
decltype用于推导表达式类型，但是不会计算表达式

## nullptr

nullptr用来代替NULL，因为NULL就是0。

void foo(char *c) {}

void foo(int x) {}参数是指针和整数时重载会有二义性

## C/CPP编译流程

编译流程是预处理、编译、汇编、链接

• 预处理：展开define、包含头文件、条件编译、处理特殊符号等与井号相关的，输出.i文本文件

• 编译：词法分析、语法分析、语义分析、中间代码生成，并且优化一些处理，如展开循环等。输出.s汇编语言文件

• 汇编：将汇编代码转换成机器指令，生成符号表记录函数和变量地址等，输出.o可重定位目标文件

• 链接：通过调整代码和数据的相对位置生成最终的内存布局，将静态链接嵌入可执行文件，动态链接仅记录映射信息，在运行时加载共享库。输出可执行文件

## 静态链接 vs 动态链接

静态链接将库文件嵌入可执行文件中，因此每个程序包含副本，占用空间大。动态链接共享库的副本没节省空间。静态链接更新需要重新编译整个程序，但是动态链接只需要替换库文件。静态链接运行更快，动态链接需要运行时加载，静态链接没有外部依赖，动态链接需要保证库文件存在且版本兼容。

## undefined reference
属于链接阶段错误，表示编译器在链接时找不到函数或变量的具体实现。可能是缺少某些库文件或者目标文件，或者库之间有依赖关系而连接顺序错误，或者函数变量声明与定义不一致