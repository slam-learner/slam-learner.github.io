---
title: Effective C++
date: 2025-06-06 21:51:16
tags: 
- Effective C++
categories:
- C++
---

## 让自己熟悉 C++

### 条款 01 将 C++视作一个语言联邦

C++是一个多重范式编程语言（multi-paradigm programming language）, 同时支持面向过程（procedural）、面向对象（OOP, object-oriented programming）、函数式（functional）、泛型编程（generic）、元编程（meta-programming）的语言。理解 C++，可以将 C++视作由相关语言的联邦而非单一语言。

- C：原始的 C 语言部分
- OOP ：class （包含构造与析构）、封装（encapsulation）、继承（inheritance）、多态（polymorphism）、virtual 函数（动态绑定）
- Template C++，即泛型编程（generic programming），也包含模板元编程（TMP，template meta-programming）
- STL

### 条款 02 尽量以 const、enum、inline 替换 `#define`

尽量少使用宏，尽可能用编译器替代预处理器

- 对于常量，最好用 constexpr 对象或 enums 替换宏
- 对于 function-like macro，最好使用 inline 函数进行替换

### 条款 03 尽可能使用 const

#### Const 指针

Const 在指针左侧——常量指针、底层 const
Const 在指针右侧——指针常量、顶层 const

#### STL 迭代器

STL 如果使用到迭代器而又不通过迭代器修改元素的值，应尽可能使用 const_iterator

#### 函数声明

Const 与函数返回值结合，是限制返回值不会被修改，降低因使用错误而造成的错误

```cpp
// 有理数类
class Rational {...};
const Rational operator* (const Rational& lhs, const Rational& rhs);
```

在这个代码示例中，对两个数值的乘积进行修改其实没有意义，那么为什么还要加上 const 限定返回值呢。

```cpp
if (a * b = c) 
```

假如比较操作时，遗漏掉一个等号，加上 const 限定的版本将会直接产生编译错误，反之则会正常编译通过，浪费在错误中进行排查的时间。另外 [[#条款 18 让接口容易被使用，不易被误用]] 中指出，一个良好的用户自定义类型应尽可能地与内置类型兼容，在上面的例子中，假如 a 和 b 是基础类型（fundamental type），比如 int 类型，上面这个语句本身就无法编译通过，因为 a  * b 的结果是一个右值，它无法被赋值，那么自定义类型也应该与这种行为保持一致。

在函数声明的最后加上 const，则说明这个函数是一个类成员函数，它限制此函数不对成员变量进行修改。

Note:
对于按值返回的用户自定义类型（如 `Rational`），是否加 `const` 是有争议的：

- **优点：** 防止错误赋值
- **缺点：** 阻止移动语义（尤其是在 C++11 后）

现代 C++ 的实践建议是：
> 不要为按值返回的类型加 `const`，除非你明确希望禁止赋值。

Note:
两个成员函数如果常量性不同，将被视作是两个不同的重载

#### 实践技巧：复用 const/non-const 成员函数逻辑

当 `const` 和 `non-const` 成员函数有着实质等价的实现时，令 `non-const` 版本调用 `const` 版本可避免代码重复

```cpp
class Text{
public:
const char& operator[](std::size_t pos) const {
    checkBounds(pos);
    return pText[pos];
}

char& operator[](std::size_t pos) {
    checkBounds(pos);
    return pText[pos];
}

private:
    std::string pText;
}
```

```cpp
class Text{
public:
const char& operator[](std::size_t pos) const {
    checkBounds(pos);
    return pText[pos];
}

char& operator[](std::size_t pos) {
    return const_cast<char&>(
        static_cast<const CTextBlock&>(*this)[pos]
    );
}

private:
    std::string pText;
}
```

### 条款 04 确保对象在使用前已被初始化

- 为内置型对象进行手工初始化，因为 C++不保证初始化他们
- 对象的成员变量初始化动作发生在进入构造函数本体之前，构造函数内的动作更准确的应该叫做赋值动作。不要将赋值和初始化混淆，赋值的方式也能够保证对象带有预期的值，但是效率较低，对象会先被默认构造，然后再调用赋值运算符。

| 行为                  | 发生时机   | 表达式       | 含义           |     |
| ------------------- | ------ | --------- | ------------ | --- |
| 初始化（Initialization） | 对象创建时  | 构造函数初始化列表 | 直接构造成员变量     |     |
| 赋值（Assignment）      | 对象已存在后 | 等号赋值语句    | 调用成员类型的赋值运算符 |     |

赋值

```cpp
class ABEntry {
public:
    ABEntry(const std::string& name, const std::string& address,
            const std::list<PhoneNumber>& phones);
private:
    std::string theName;
    std::string theAddress;
    std::list<PhoneNumber> thePhones;
    int numTimesConsulted;
};

ABEntry::ABEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones) {
    theName = name;
    theAddress = address;
    thePhones = phones;
    numTimesConsulted = 0;
}
```

初始化

```cpp
ABEntry::ABEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones)
    : theName(name),
      theAddress(address),
      thePhones(phones),
      numTimesConsulted(0)  // 内置类型也最好初始化
{}
```

- 如果一个类有多个构造函数，它们可能都需要初始化同一组成员。为了避免重复写初始化列表，可以将“公共初始化逻辑”提取到一个私有函数中，适用于那些赋值和初始化成本相同的成员（如 `int`、`double` 等）。改用它们的赋值操作，并将那些赋值操作移往某个函数（通常是 private) ，供所有构造函数调用。这种做法在“成员变量的初值系由文件或数据库读入”时特别有用。然而，比起经由赋值操作完成的“伪初始化 ”(pseudo-initialization) ，通过成员初值列 (member initialization list) 完成的“真正初始化”通常更加可取。“”

- 无需记住哪些成员必须用初始化列表，只要所有成员都写在初始化列表中，永远不会错。成员变量总是以它们在类中声明的顺序进行初始化，与成员初始化列表中出现的顺序无关，因此应始终按照声明顺序写初始化列表。

- 最后一点：注意不同编译单元中的 non-local static 对象的初始化顺序。函数内的 static 对象被称为 local static 对象，其他 static 对象被称为 non-local static 对象，C++对于“定义于不同编译单元内的 non-local static 对象”的初始化顺序是未定义的，因此常用手法是利用 Singleton 模式将 non-local static 对象替换为 local static 对象（C++11 之后保证线程安全），这个手法的基础在于：C++保证，函数内的 local static 对象会在“该函数被调用期间”“首次遇上该对象之定义式”时被初始化。
Note:

> Static 对象的生成周期从其被构造出来开始直到程序结束为止，他们的析构函数会在 main ()函数结束之后被自动调用。
> 所谓编译单元 (translation unit) 是指产出单一目标文件 (single object file) 的那些源码。基本上它是单一源码文件加上其所含入的头文件 ( #include files) 。

```cpp
// 单例模式或 global-like 使用方式推荐如下
class FileSystem {
public:
    static FileSystem& instance() {
        static FileSystem fs; // ✅ 函数内 static，初始化顺序受保障
        return fs;
    }
};
```

## 构造/析构/赋值运算

### 条款 05 了解 C++默认生成了哪些函数

- 编译器会为 class 生成 default constructor、copy constructor、copy assignment operator、destructor，所有这些函数都是 public 且 inline 的。编译器会尝试隐式声明这些函数，但是否定义（生成代码）要看是否真正被使用（即所谓“隐式声明、按需定义”）。
Note:  

>空类对象并非真的空，它会拥有上述编译器生成的成员函数。并且空类的对象通常至少占用一个字节，以便它们在内存中拥有唯一地址。

- Default 构造函数和析构函数会调用 base class 和 non-static 成员变量的构造函数和析构函数
- 对于拷贝构造函数，如果类内由深拷贝的需求，或者类内有引用成员或者 const 成员，应当自定义拷贝行为
- 编译器生成的析构函数是 non-virtual 的，除非这个 class 的 base class 自身声明有 virtual 析构函数

Note:
C++11 之后，除了以上四个函数外，编译器还会自动生成移动构造函数和移动赋值函数

| 函数                                   | 默认生成条件                  | 说明          |
| ------------------------------------ | ----------------------- | ----------- |
| **默认构造函数** `T()`                     | 没有定义任何构造函数时自动生成         | 用于默认初始化     |
| **析构函数** `~T()`                      | 未定义析构函数时自动生成            | 用于清理资源      |
| **拷贝构造函数** `T(const T&)`             | 未定义任何拷贝/移动构造时自动生成       | 用于按值传递、拷贝对象 |
| **拷贝赋值运算符** `T& operator=(const T&)` | 未定义任何赋值/移动赋值时自动生成       | 用于对象赋值      |
| **移动构造函数** `T(T&&)`                  | 若未定义拷贝/移动构造、析构等，编译器尝试生成 | 用于右值引用转移资源  |
| **移动赋值运算符** `T& operator=(T&&)`      | 条件如上                    | 同样是右值资源转移   |

| 情况            | 默认构造 | 拷贝构造   | 拷贝赋值   | 移动构造       | 移动赋值       | 析构  |
| ------------- | ---- | ------ | ------ | ---------- | ---------- | --- |
| 什么都不写         | ✅    | ✅      | ✅      | ✅（C++11 起） | ✅（C++11 起） | ✅   |
| 显式定义了析构函数     | ✅    | ✅      | ✅      | ❌          | ❌          | ✅   |
| 显式定义了拷贝构造或赋值  | ✅    | ✅      | ✅      | ❌          | ❌          | ✅   |
| 显式定义了移动构造或赋值  | ✅    | ❌（需手写） | ❌（需手写） | ✅          | ✅          | ✅   |
| 显式定义了构造函数（带参） | ❌    | ✅      | ✅      | ✅          | ✅          | ✅   |

现代 C++推荐使用规则：

| 使用目的         | 推荐写法                                    | 说明                    |
| ------------ | --------------------------------------- | --------------------- |
| 明确需要默认行为     | `ClassName() = default;` 等              | 不再依赖隐式生成，更直观          |
| 禁止某个行为       | `ClassName(const ClassName&) = delete;` | 防止拷贝或赋值错误使用           |
| 自己管理资源（RAII） | 显式写析构/拷贝/移动函数                           | 比如用到 `new[]`、文件句柄等    |
| 继承自非平凡基类     | 明确 default / delete 是必要的                | 避免切片或行为不一致            |
| 避免浅拷贝问题      | 显式禁用拷贝，只支持移动                            | 典型于 `std::unique_ptr` |

### 条款 06 若不想使用编译器自动生成的函数，应当明确拒绝

- C++11 中，可以使用 `delete` 关键字显式禁用某个函数
- C++98 中，可以将某个函数声明为 private 且不实现

### 条款 07 为多态基类声明 virtual 析构函数

- 带多态性质的 base class 应该声明 virtual 析构函数，以确保其 derived class 的析构函数被调用。防止指向派生类的基类指针在被释放时只局部销毁了该对象，造成内存泄漏
- 如果一个 class 带有任何 virtual 函数，那么它应该有 virtual 析构函数
- 普通的类无需也不应该有虚析构函数，虚函数是有代价的
- 如果一个类没有被设计为基类，又有被错继承的风险，应将类声明为 `final`（C++11）。

### 条款 08 不要让异常逃离析构函数

- 析构函数不应该抛出异常，如果一个被析构函数调用的函数可能抛出异常，析构函数应捕捉任何异常，然后吞掉异常或者结束程序。
- 如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么 class 应该提供一个普通函数（而非在析构函数中）执行该操作。

### 条款 09 构造/析构函数内不要调用虚函数

- **构造/析构期间即使调用虚函数也不会表现出多态行为。** C++中构造函数的调用顺序为：基类➡派生类，在构造基类期间，派生类还没有被构造完成，所以虚表指针（vptr）仍指向基类的虚表，虚函数调用会调用到基类函数，而非派生类的。因此在基类构造期间，虚函数非虚。类对象的基类构造期间，对象的类型为基类而不是派生类，运行期类型信息也会把对象视作基类。相同道理也适用于析构函数，一旦派生类开始析构，对象内专属于派生类的成员变量就会编程未定义值，C++将它们视作不存在，进入基类的析构函数后，对象就被视作基类对象。

- 解决办法之一是，既然你无法使用 virtual 函数从 base class 向下调用，那就在构造期间令 derived class 将必要的构造信息向上传递至 base class 构造函数

### 条款 10 赋值运算符应返回指向自身的引用

- 赋值操作符应该返回一个指向自身的引用，以便于链式赋值
这个协议不仅适用于标准赋值形式，也适用于所有**赋值**相关运算，例如：

```cpp
class Widget
{
public:
    ...
    Widget& operator+=(const Widget& rhs)  //这个协议适用于
    {                                      //+=、-+、*=等等。
        ...
        return *this；                     
    }

    Widget& operator=(int rhs)             //／此函数也适用，即使
    {                                      //此一操作符的参数类型
        ...                                //不符协定。
        return *this；                   
    }
    ...
}
```

注意 bool 操作符重载的返回值有所不同，请留心，以免无限调用自身。

```cpp
struct Vector2
{
    ....
    bool operator==(const Vector2& other) const  //定义操作符的重载,如果！=，这里做相应修改即可
    {
        return x == other.x && y == other.y;  //不能return *this 否则无限调用自己
    }

    bool operator！=(const Vector2& other) const  
    {
        return ！(*this == other);
    }
};
```

在设计接口时一个重要的原则是，让自己的接口和内置类型相同功能的接口尽可能相似，所以如果没有特殊情况，赋值运算符就应当返回自身的引用。

### 条款 11 赋值运算符中应避免自我赋值

避免自我赋值的常用技术包括以下几种：

- 比较“来源对象”和“目标对象”的地址，如果相同则直接返回
- 精心周到的语句顺序，例如先“记住”原有对象，然后“复制”新值，最后“销毁”原有对象
- 利用 copy-and-swap 技术
- 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确

### 条款 12 复制对象时不要遗漏任何成员

- 类添加成员变量时，不要忘记在拷贝构造函数和赋值操作符中对新加的成员变量进行处理。如果忘记处理，编译器不会给出警告或者报错。
- 如果类有继承行为，那么在为派生类编写拷贝构造函数是要注意复制基类的每一个成员，这些成员往往是私有的，所以无法直接访问，应当让派生类的拷贝构造调用基类的拷贝构造函数来实现成员的复制。

```cpp
//在成员初始化列表显示调用基类的拷贝构造函数
ChildClass::ChildClass(const ChildClass& rhs) : BaseClass(rhs) {  
   // ...
}
```

- 拷贝构造函数和拷贝赋值操作符，他们两个中任意一个不要去调用另一个，这虽然看上去是一个避免代码重复的好方法，但是是荒谬的。原因在于拷贝构造函数在构造一个对象——这个对象在调用之前并不存在；而赋值操作符在改变一个对象——这个对象是已经构造好了的。因此前者调用后者是在给一个还未构造好的对象赋值；而后者调用前者就像是在构造一个已经存在了的对象。如果想要避免重复代码，可以将重复部分提成一个私有函数，两个函数共同进行调用。

```cpp
class MyClass {
public:
    MyClass(const MyClass& rhs) {
        copy_from(rhs);
    }

    MyClass& operator=(const MyClass& rhs) {
        if (this != &rhs) {
            copy_from(rhs);
        }
        return *this;
    }

private:
    void copy_from(const MyClass& rhs) {
        // 拷贝所有成员
        this->a = rhs.a;
        this->b = rhs.b;
        // ...
    }
};
```

## 资源管理

### 条款 13 使用对象管理资源（RAII）

- 为防止资源泄漏，请使用 RAII 对象，它们在构造函数中获得资源并在析构函数中释放资源。
- C++11 之后常使用 unique_ptr 或 shared_ptr 作为资源管理类。
本条款的核心观点在于：以面向流程的方式管理资源（的获取和释放），总是会在各种意外出现时，丢失对资源的控制权并造成资源泄露。以面向过程的方式管理资源意味着，资源的获取和释放都分别被封装在函数中。这种管理方式意味着资源的索取者肩负着释放它的责任，但此时我们就要考虑一下以下几个问题：调用者是否总是会记得释放呢？调用者是否有能力保证合理地释放资源呢？不给调用者过多义务的设计才是一个良好的设计。

首先我们看一下哪些问题会让调用者释放资源的计划付诸东流：

- 一句简单的 `delete` 语句并不会一定执行，例如一个过早的 `return` 语句或是在 `delete` 语句之前某个语句抛出了异常。
- 谨慎的编码可能能在这一时刻保证程序不犯错误，但无法保证软件接受维护时，其他人在 delete 语句之前加入的 return 语句或异常重复第一条错误。

为了保证资源的获取和释放一定会合理执行，我们把获取资源和释放资源的任务封装在一个对象中。当我们构造这个对象时资源自动获取，当我们不需要资源时，我们让对象析构。这便是“Resource Acquisition Is Initialization; RAII”的想法，因为我们总是在获得一笔资源后于同一语句内初始化某个管理对象。无论控制流如何离开区块，一旦对象被销毁（比如离开对象的作用域）其析构函数会自动被调用。

### 条款 14 资源管理类要小心复制行为

- 复制 RAII 对象必须一并复制它所管理的资源，所以资源的 copying 行为决定 RAII 对象的 copying 行为。  
- 如果对想要自行管理 delete（或其他类似行为如上锁/解锁）的类处理复制问题，有以下方案，先创建自己的资源管理类，然后可选择：

> 1. 禁止复制，使用 [[#条款 06 若不想使用编译器自动生成的函数，应当明确拒绝]] 的方法  
> 2. 对复制的资源做引用计数（如 shared_ptr），shared_ptr 支持初始化时自定义删除函数。
> 3. 做真正的深复制  
> 4. 转移资源的拥有权，如 unique_ptr，只保持新对象拥有。

### 条款 15 资源管理类应提供对所管理的原始对象的访问接口

1. 许多 API 需要直接引用资源，而不是通过资源管理类。虽然它们不合规范，但很难避免使用它们。所以资源管理类应该提供访问原始资源的能力。
2. 对原始资源的访问可能经由显式转换或隐式转换。一般而言显式转换比较安全，但隐式转换对客户比较方便。

### 条款 16 成对使用 new 和 delete 时要采用相同形式

- 通过 new 生成对象时，有两件事发生：

> 第一，内存被分配出来（通过名为 operator new 的函数，见条款 49 和条款 51）。
> 第二，针对此内存会有一个（或更多）构造函数被调用。

- 当使用 delete 时，也有两件事发生：针对此内存会有一个（或更多）析构函数被调用，然后内存才被释放（通过名为 operator delete 的函数，见条款 51）。
- 如果调用 new 时使用[]，必须在对应调用 delete 时也使用[]。如果调用 new 时没有使用[]，那么也不该在对应调用 delete 时使用[]。
- 对于数组，不建议使用 typedef 行为，这会让使用者不记得去 delete []。对于这种情况，建议使用 string 或者 vector。

### 条款 17 以独立语句将 new 对象放入智能指针

1. 以独立语句将 new 的对象存储于智能指针中，这样能够保证动态获取的资源一定能被资源管理对象接管，不会造成内存泄漏。
2. 如果不这样做，一旦在【资源申请成功】和【资源管理对象接管资源】之间抛出了异常，就有可能产生难以察觉的资源泄漏。因为异常本身就是意料之外的错误，不容易复现，从而导致资源泄漏无法轻易定位。

说明：
假设有两个函数原型如下：

```cpp
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);
```

如果按照如下方法调用 `processWidget` 函数，则有可能造成资源泄漏：

```cpp
processWidget(std::shared_ptr<Widget>(new Widget()), priority());
```

这是因为编译器在编译 `processWidget` 函数之前会检查即将被传递的各个实参，第二个实参是对 `priority` 函数的简单调用，而第一个实参包含两个部分：

- 执行 `new Widget()` 表达式动态创建 `Widget` 对象。
- 调用 `std::shared_ptr` 类的构造函数并使用 `Widget` 对象的指针作为构造参数。

所以，在调用 `processWidget` 函数之前编译器会做以下三件事情：

- 执行 `new Widget()` 表达式动态创建 `Widget`对象
- 调用 `std::shared_ptr` 类的构造函数并使用`Widget`对象的指针作为构造参数
- 调用 `priority` 函数得到优先级

以何种顺序执行以上三个步骤，C++语句并没有给出严格规定（Java 和 C #中必须按规定的顺序完成参数的核算 ），具体由编译器来决定。

能够明确知道的是执行 `new Widget` 表达式肯定是在调用 `shared_ptr` 构造函数之前，但调用 `priority` 函数则有可能是在第 1、2、3 中任意一步执行。假设调用 `priority` 函数在第 2 步执行，将获得以下执行顺序：

1. 执行 `new Widget()` 表达式动态创建 `Widget`对象
2. 调用 `priority` 函数得到优先级
3. 调用 `std::shared_ptr` 类的构造函数并使用`Widget`对象的指针作为构造参数

如果在调用 `priority` 函数的过程中发生了异常，那么 `new Widget()` 表达式返回的指针会被遗失，就有可能造成资源泄漏。原因是在【资源被创建】和【资源被管理对象接管】之间造成了异常干扰。

**解决办法：**

分离构造 `Widget` 对象的语句，在单独的语句中执行 `new Widget()` 表达式并调用 `shared_ptr` 类的构造函数，最后将智能指针传给 `processWidget` 函数。代码如下：

```cpp
std::shared_ptr<Widget> pw(new Widget());
processWidget(pw, priority());
```

编译器对于**跨越语句的各项操作**没有重新排列的自由，只有在语句内才拥有某种自由度。

因此在上述代码中【（1）执行 `new Widget()` 表达式和（2）调用 `shared_ptr` 类的构造函数】与【对 `priority` 函数的调用】是在不同的语句中，被分隔开来了，所以编译器不得在它们之间任意选择执行次序。

## 设计与声明

### 条款 18 让接口容易被使用，不易被误用

在设计接口时，我们常常会错误地假设，接口的调用者拥有某些必要的知识来规避一些常识性的错误。但事实上，接口的调用者并不总是像正在设计接口的我们一样“聪明”或者知道接口实现的”内幕信息“，结果就是，我们错误的假设使接口表现得不稳定。这些不稳定因素可能是由于调用者缺乏某些先验知识，也有可能仅仅是代码上的粗心错误。接口的调用者可能是别人，也可能是未来的你。所以一个合理的接口，应该尽可能的从语法层面并在编译之时运行之前，帮助接口的调用者规避可能的风险。

如下，设计一个日期类

```cpp
//参考原文：https://blog.csdn.net/hualicuan/article/details/27526033
class Date
{
public:
    Date(const int month, const int day, int const year) 
        : m_month(month)
        , m_day(day)
        , m_year(year)
    {

    }
private:
    int m_day;
    int m_month;
    int m_year;
};
```

错误调用

```cpp
    Date d1(29, 5, 2014);  //调用顺序错乱，应该是 5, 29, 2014
    Date d2(2, 30, 2014);  //传入参数有误，2月没有30号
```

- 使用外覆类型（wrapper）提醒调用者传参错误检查，将参数的附加条件限制在类型本身

当调用者试图传入数字“13”来表达一个“月份”的时候，你可以在函数内部做运行期的检查，然后提出报警或一个异常，但这样的做法更像是一种责任转嫁——调用者只有在尝试过后才发现自己手残把“12”写成了“13”。如果在设计参数类型时就把“月份”这一类型抽象出来，比如使用 enum class（强枚举类型），就能帮助客户在编译时期就发现问题，把参数的附加条件限制在类型本身，可以让接口更易用。

```cpp
struct Day{
    explicit Day (const int day) : m_day (day) {}
    private:
        int m_day;
};

struct Month {
    explicit Month(const int month) : m_month(month) {}
    private:
        int m_month;
};

struct Year {
    explicit Year(const int year) : m_year(year) {}
    private:
        int m_year;
};

class Date {
public:
    Date(const Month &month, const Day &day,  const Year &year) 
        : m_month(month), m_day(day), m_year(year) {}
private:
    Day m_day;
    Month m_month;
    Year m_year;
};
```

类型错误得到预防，但值还是没有得到保障

```cpp
Date d2(2, 30, 2014);  // error,类型错误
Date d3(Day(30), Month(2), Year(2014)); // error,类型错误
Date d4(Month(2), Day(30), Year(2014)); // ok
```

可通过设计对应的类型的值限制来达到

```cpp
struct Month
{
    enum E_MON{JAN = 1, FEC, MAR, APR, MAY, JUN, JUL, AGU, SEP, OCT, NOV, DEC};
    explicit Month(const E_MON month) : m_month(month) {}
private:
    int m_month;
};
```

调用

```cpp
Date d4(Month(Month::E_MON::DEC), Day(30), Year(2014)); //ok
```

- 从语法层面限制调用者不能做的事

接口的调用者往往无意甚至没有意识到自己犯了个错误，所以接口的设计者必须在语法层面做出限制。一个比较常见的限制是加上 `const`，比如在 `operate*` 的返回类型上加上 `const` 修饰，可以防止无意错误的赋值 `if (a * b = c)`。

- 接口应表现出与内置类型的一致性

让自己的类型和内置类型的一致性，比如自定义容器的接口在命名上和 STL 应具备一致性，可以有效防止调用者犯错误。或者你有两个对象相乘的需求，那么你最好重载 `operator*` 而并非设计名为”multiply”的成员函数。

- 从语法层面限制调用者**必须做的事**

**别让接口的调用者总是记得做某些事情**，接口的设计者应在假定他们**总是忘记**这些条条框框的前提下设计接口。比如用智能指针代替原生指针就是为调用者着想的好例子。如果一个核心方法需要在使用前后设置和恢复环境（比如获取锁和归还锁），更好的做法是将设置和恢复环境设置成纯虚函数并要求调用者继承该抽象类，强制他们去实现。在核心方法前后对设置和恢复环境的调用，则应由接口设计者操心。

当方法的调用者（我们的客户）责任越少，他们可能犯的错误也就越少。

请记住：

①好的接口易于正确使用，而难以错误使用。你应该在你的所有接口中为这个特性努力。 ②使易于正确使用的方法包括在接口和行为兼容性上与内建类型保持一致。 ③预防错误的方法包括创建新的类型，限定类型的操作，约束对象的值，以及消除客户的资源管理职责。 ④ `std::shared_ptr` 支持自定义 deleter。这可以防止 cross-DLL 问题，能用于自动解锁互斥体（参见 [[#条款 14 资源管理类要小心复制行为]]）

### 条款 19 设计 class 如同设计一个类型

- 对象该如何创建销毁：包括构造函数、析构函数以及 new 和 delete 操作符的重构需求。
- 对象的构造函数与赋值行为应有何区别：构造函数和赋值操作符的区别，重点在资源管理上。（[[#条款 04 确保对象在使用前已被初始化]]）
- 对象被拷贝时应考虑的行为：拷贝构造函数。
- 对象的合法值是什么？最好在语法层面、至少在编译前应对用户做出监督。
- 新的类型是否应该复合某个继承体系，这就包含虚函数的覆盖问题。
- 新类型和已有类型之间的隐式转换问题，这意味着类型转换函数和非 explicit 函数之间的取舍。
- 新类型是否需要重载操作符。
- 什么样的接口应当暴露在外，而什么样的技术应当封装在内（public 和 private）
- 新类型的效率、资源获取归还、线程安全性和异常安全性如何保证。
- 这个类是否具备 template 的潜质，如果有的话，就应改为模板类。

### 条款 20 优先使用 pass-by-reference-to-const 而不是值传递

函数接口应该以 `const` 引用的形式传参，而不应该按值传参，否则可能会有以下问题：

- 按值传参涉及大量参数的复制，这些副本大多是没有必要的。
- 如果拷贝构造函数设计的是深拷贝而非浅拷贝，那么拷贝的成本将远远大于拷贝某几个指针。
- 对于多态而言，将父类设计成按值传参，如果传入的是子类对象，仅会对子类对象的父类部分进行拷贝，即部分拷贝，而所有属于子类的特性将被丢弃，造成不可预知的错误，同时虚函数也不会被调用。（对象切割）
- 小的类型并不意味着按值传参的成本就会小。首先，类型的大小与编译器的类型和版本有很大关系，某些类型在特定编译器上编译结果会比其他编译器大得多。小的类型也无法保证在日后代码复用和重构之后，其类型始终很小。

总结：

- **尽量以 pass-by-reference-to-const 替换 pass-by-value**。前者通常比较高效，并可避免切割问题（slicing problem）。
- 以上规则**并不适用于内置类型，以及 STL 的选代器和函数对象**。对它们而言，pass-by-value 往往比较适当。

### 条款 21 必须返回对象时，别妄想返回其 reference

在一些函数内部，如果返回局部对象的引用会造成悬垂引用问题，从而导致未定义行为。在 C++11 之前，以值方式返回对象可能会引发拷贝，但是这个开销是不可避免的。C++11 之后，更推荐以值的方式返回，其优点如下：

- **返回值优化（RVO）**：编译器通常会优化掉返回值的拷贝或移动，性能开销很低。
- **安全性好**：返回值不会涉及悬垂引用的问题。
- **代码清晰**：返回值表达的是“函数返回的只是一个副本”，更易于理解和维护。

C++17 之前, RVO 并不是强制的，不过即使没有 RVO，也可以使用 C++11 引入的移动语义节省掉拷贝的开支，而 C++17 之后，RVO 被强制保证，所以以值的形式返回完全不会涉及到拷贝或移动操作，假设某个函数的返回类型的拷贝构造函数和移动构造函数都被删除掉，它被以值的形式返回，由于 RVO 的强制保证，定义某个变量接受这个返回值并不会引发错误，因为这里并不牵涉到拷贝或者移动构造，它相当于是在调用点进行直接构造（direct-initialization）的。

>C++17 中的 **mandatory copy elision**（强制拷贝省略）
> 拷贝省略是一种编译器优化技术，它允许编译器**省略掉某些对象的拷贝或移动操作**，尤其是在函数返回对象或构造临时对象的过程中。
> 在 C++11/14 中，这是一种“**允许的优化**”；  
 >但在 C++17 中，在某些特定情形下，它变成了“**语义强制要求**”：

#### C++17 拷贝省略的情形

 **情形 1：返回局部对象（Named Return Value）**
 在此情形下，C++17 标准强制要求进行返回值优化（RVO）完成拷贝省略，

```cpp
T f () {
    T t;
    return t;
}
```

在这个例子中：

- `t` 是一个**局部变量**；
- 返回语句直接返回 `t`，**类型完全匹配**；
- 没有类型转换或包装；

**情形 2：返回临时对象（Temporary Return Value）**
在此情形下，编译器会执行具名返回值优化（NRVO，Named Return value Optimization），但此优化并没有收到标准的强制要求。

```cpp
T f() {
    return T(); // 直接返回一个临时对象
}
```

这叫做 **临时值返回（temporary expression return）**。

- 这里 `T()` 是一个临时对象；
- 返回语句中没有中间变量或转换；
- 类型完全匹配；
- 所以编译器必须执行 **copy elision**，不会调用拷贝或移动构造函数。

#### 强制 RVO 的前提条件

C++17 中强制 copy elision **只适用于这两种情形**，前提是：

| 条件            | 要求              |     |
| ------------- | --------------- | --- |
| 返回值类型         | 和函数的返回类型完全一致    |     |
| 不发生类型转换       | 没有转换为基类、父类型等    |     |
| 返回的是局部变量或临时对象 | 且是直接 `return` 的 |     |

#### 优化失效的情况

情形 1：返回引用或通过类型转换

```cpp
struct Base {};
struct Derived : Base {};

Base f() {
    Derived d;
    return d; // ❌ 发生类型转换（Derived → Base），不是强制 RVO 情况
}
```

这里返回类型需要进行类型转换，所以不会得到 RVO 保证。

情形 2：return std:: move (t)

```cpp
T f() {
    T t;
    return std::move(t); // ❌ 不强制 RVO，调用移动构造函数
}
```

- `std::move(t)` 会将 `t` 变为右值；
- 这里会尝试调用 **T 的移动构造函数**；
- 所以这个不是 C++17 强制省略的情形（但编译器仍可能优化它，属于“非强制” copy elision）。

情形 3 运行时依赖（在不同的条件分支下返回不同变量）

当编译器无法单纯通过函数来决定返回哪个对象实例时，会禁用（N）RVO。
比如：

```cpp
Obj fun(bool flag) {
  Obj o1;
  Obj o2;
  if (flag) {
    return o1;
  }
  return o2;
}

int main() {
  Obj obj = fun(true);
  return 0;
}
```

情形 4 返回全局变量

当返回的对象不是在函数内创建的时候，是无法进行返回值优化的。

```cpp
Obj g_obj;

Obj fun() {
  return g_obj;
}

int main() {
  Obj obj = fun();
  std::cout << &obj << std::endl;
  return 0;
}
```

情形 5 返回函数参数

与返回全局变量类似，当返回的对象不是在函数内创建的时候，是无法执行返回值优化的。

```cpp
Obj fun(Obj obj) {
  return obj;
}

int main() {
  Obj o;
  Obj obj = fun(o);
  std::cout << "in main " << &obj << std::endl;
  return 0;
}
```

情形 6 返回成员变量

```cpp
struct Wrapper {
   Obj obj;
 };

Obj fun() {
  return Wrapper().obj;
}

int main() {
  Obj obj = fun();
  std::cout << &obj << std::endl;
  return 0;
}
```

情形 7 存在赋值行为

(N) RVO 只能在从返回值创建对象时发送，在现有对象上使用 operator=而不是拷贝/移动构造函数，这样是不会进行 RVO 操作的。

```cpp
Obj fun() {
  Return Obj ();
}

int main() {
  Obj obj;
  obj = fun();
  return 0;
}
```

总结：

- 绝不要返回 pointer 或 reference 指向一个 local stack 对象，或返回 reference 指向一个 heap-allocated 对象，或返回 pointer 或 reference 指向一个 local static 对象而有可能同时需要多个这样的对象。[[#条款 04 确保对象在使用前已被初始化]] 已经为“在单线程环境中合理返回 reference 指向一个 local static 对象”提供了一份设计实例。

### 条款 22 将成员变量声明为 private

对 class 内所有成员变量声明为 `private`，`private` 意味着对变量的封装。但本条款提供的更有价值的信息在于不同的属性控制—— `public`, `private` 和 `protected` ——代表的设计思想。

简单的来说，把所有成员变量声明为 private 的好处有两点。首先，所有的变量都是 private 了，那么所有的 public 和 protected 成员都是函数了，用户在使用的时候也就无需区分，这就是语法一致性；其次，对变量的封装意味着，可以尽量减小因类型内部改变造成的类外外代码的必要改动。

一旦所有变量都被封装了起来，外部无法直接获取，那么所有类的使用者（我们称为客户，客户也可能是未来的自己，也可能是别人）想利用私有变量实现自己的业务功能时，就必须通过我们留出的接口，这样的接口便充当了一层缓冲，将类型内部的升级和改动尽可能的对客户不可见——不可见就是不会产生影响，不会产生影响就不会要求客户更改类外的代码。因此，一个设计良好的类在内部产生改动后，对整个项目的影响只应是需要重新编辑而无需改动类外部的代码。

我们接着说明，`public` 和 `protected` 属性在一定程度上是等价的。一个自定义类型被设计出来就是供客户使用的，那么客户的使用方法无非是两种——用这个类创建对象或者继承这个类以设计新的类——以下简称为第一类客户和第二类客户。那么从封装的角度来说，一个 `public` 的成员说明了类的作者决定对类的第一种客户不封装此成员，而一个 `protected` 的成员说明了类的作者对类的第二种客户不封装此成员。也就是说，当我们把类的两种客户一视同仁了以后，`public`、`protected` 和 `private` 三者反应的即类设计者对类成员封装特性的不同思路——对成员封装还是不封装，如果不封装是对第一类客户不封装还是对第二类客户不封装。

请记住：

- 切记将成员变量声明为 private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供 class 作者以充分的实现弹性。
- Protected 并不比 public 更具封装性。

### 条款 23 优先使用 non-member、non-friend 函数

**一句话总结**：除非函数需要访问对象的内部状态，否则优先设计为非成员函数（而不是成员函数或 friend 函数）。

优点：

- 增加封装性

>封装的目标是**隐藏实现细节**，只暴露必要接口。
>成员函数自动成为 class 接口的一部分（即 `public API`）；
>一旦暴露为成员函数，就表示这个操作是这个类“核心功能”的一部分；
>**非成员函数**不属于类本体，可以在**不破坏类内部状态的前提下**使用类。
>**非成员函数更容易与实现细节解耦**，从而保持类的封装性。

- 提升包裹弹性（Packaging Flexibility）

>非成员函数可以放在：不同的命名空间、独立的头文件或工具库、不受类定义和编译单元的直接限制。
> 非成员函数更容易复用、更方便替换或修改、在多个模块/库进行链接时可以避免依赖地狱。

- 更容易扩展功能

>如果需要为第三方类添加新功能，由于不能修改第三方类的源码，所以只能写非成员函数来扩展功能。而如果依赖成员函数，则意味着必须控制类定义，这在现实中经常做不到。
>非成员函数是 **开放-封闭原则（OCP）** 的体现：对扩展开放、对修改封闭。

```cpp
Namespace utils {
    void print(const ThirdPartyClass& obj);
}
```

举例：

```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);

    // ❌ 错误做法：不应该写为成员函数
    Rational operator*(const Rational& rhs) const {
        return Rational(this->num * rhs.num, this->den * rhs.den);
    }

private:
    int num;
    int den;
};

// ✅ 更好做法：非成员函数（通常写在命名空间中）
Rational operator*(const Rational& lhs, const Rational& rhs) {
    return Rational(lhs.numerator() * rhs.numerator(),
                    lhs.denominator() * rhs.denominator());
}
```

### 条款 24 若函数的所有参数（包括第一个参数）均需类型转换，则其应该设计为非成员函数

C++中，如果把一个函数写成成员函数

```cpp
class A {
public:
    A operator+(const A& other);  // 成员函数，隐含 this 作为第一个参数
};
```

编译器会只允许对非第一个参数做隐式类型转换，但对第一个参数（即 `this` 指向的对象）不做类型转换。

```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    const Rational operator*(const Rational& rhs) const;
};

Rational a(2, 3);
Rational result = a * 2;    // OK：2 被隐式转换为 Rational
result = 2 * a;             // 错！不能隐式将 2 转换为 this 所需的 Rational
```

- `a * 2` 会调用成员函数 `a.operator*(Rational(2))` —— 第二个参数能被转换；
- 但 `2 * a` 中，左边的 2 需要转换成 Rational，并成为成员函数的调用对象 —— 编译器不允许隐式这样做。

使用非成员函数来写上面例子中的运算符重载则可以解决这个问题：

```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    friend const Rational operator*(const Rational& lhs, const Rational& rhs);
};

const Rational operator*(const Rational& lhs, const Rational& rhs) {
    // 实现乘法逻辑
}
```

因此，当一个函数，其所有参数都可能来自不同类型、需要转换时，应使用非成员函数，这样才能获得更大的灵活性和对称性。

### 条款 25 写出一个不抛异常的 swap 函数

`std::swap` 的典型实现如下：

```cpp
namespace std {
    template <class T>
    void swap(T& lhs, T& rhs) {
        T temp(lhs);
        lhs = rhs;
        rhs = temp;
    }
}
```

只要类型 `T` 支持拷贝构造和赋值运算符，`std::swap` 即可完成置换。不过这里涉及到一次拷贝构造和两次赋值，对于一般的类型而言，这样的操作并没有什么问题，但是对于一些特殊的类型，这样的操作不够高效。考虑以下对象：

```cpp
Class WidgetImpl {
public:
 ...
private:
    int a, b, c;
    std::vector<double> v;
}

class Widget {
public：
    Widget(const Widget& rhs);
    Widget& operator=(const Widget& rhs) {
        ...
        *pImpl = *(rhs.pImpl); // 深拷贝
        ...
    }
private:
    WidgetImpl* pImpl;
}
```

`Widget` 对象的置换可以通过直接交换 `pImpl` 指针实现，对 `Widget` 对象直接执行 `std::swap` 会发生以下行为：

1. `T temp = a;` → 调用拷贝构造函数 → 复制整个 `WidgetImpl` 内容
2. `a = b;` → 调用赋值运算符 → 再复制一次
3. `b = temp;` → 第三次复制

实际上总共发生了三次对于 `WidgetImpl` 的深拷贝，而置换实际上只需要 `pImpl` 即可，如果需要让 `std::swap` 实现这样的行为，需要对 `std::swap` 进行偏特化。（ C++规定：通常不允许改变 std 命名空间内的任何东西，但是可以为标准 template（如 swap）制造特化版本，使他专属于我们自己的 class）。类似以下实现：

```cpp
namespace std {
    template <> // 表明是个模板全特化
    void swap<Widget> (Widget& lhs, Widget& rhs) {
        std::swap(lhs.pImpl, rhs.pImpl); // 直接置换pImpl指针,但是由于访问权限会编译失败
    }
}
```

但是这样的代码并不能通过编译，因为 `pImpl` 是 `Widget` 的私有成员，那么如何实现期望的置换呢？答案是在 `Widget` 内部添加一个 `public` 的 `swap` 实现，然后让 `std::swap` 的特化版本调用这个 `public` 成员函数，代码如下：

```cpp
class Widget {
public：
    Widget(const Widget& rhs);
    Widget& operator=(const Widget& rhs) {
        ...
        *pImpl = *(rhs.pImpl); // 深拷贝
        ...
    }

    void swap(Widget& rhs) {
        using std::swap;
        Swap (pImpl, rhs. PImpl);
    }
private:
    WidgetImpl* pImpl;
}

namespace std {
    template <> // 表明是个模板特化
    void swap<Widget> (Widget& lhs, Widget& rhs) { // 为Widget类型提供偏特化（partially specialize）
        lhs.swap(rhs); // 直接置换pImpl指针
    }
}
```

如果 `Widget` 和 `WidgetImpl` 是类模板的话，情况会变得复杂一些，如果直接按照上面代码的想法改写到模板的对应写法，可能会写出类似下面的代码：

```cpp
namespace std {
    template <class T>
    void swap<Widget<T>> (Widget<T>& lhs, Widget<T>& rhs) {
        lhs.swap(rhs);
    }
}
```

但是 C++只允许对类模板（class template）进行部分特化，并不允许对函数模板（function template）进行部分特化。所以这段代码会编译失败。
>Q: 为什么示例是部分特化而不是重载或全特化？
  A: 示例代码尝试为 `std::swap` 提供一个专门用于 `Widget<T>` 的版本，并且 `T` 是模板参数。这正是 “部分特化” 的定义：「保留部分模板参数开放，同时对特定的模板参数形式做出适配。」

如果仍想要在 `std` 命名空间中提供一个针对 `Widget` 的模板 `swap` 函数，一个简单的方法是对 `Widget` 提供重载版本的 `swap` 函数（并非偏特化）。

```cpp
namespace std {
    // std::swap 的重载版本
    template<class T>
    void swap(Widget<T> const& lhs, Widget<T> const& rhs) {
        lhs.swap(rhs);
    }
}
```

重载版本的模板 `swap` 函数的问题在于，`std` 是一个特殊的命名空间，其管理规则也比较特殊，代码可以对 `std` 中的模板进行偏特化，但是不可以添加新的 `templates`，这是未定义行为。所以可行的方法是：提供一个模板 `swap` 函数，但是不放在 `std` 命名空间中，而是放在其他的命名空间中。

```cpp
namespace WidgetStuff {
    ...
    template<class T>
    class Widget {...}; // 提供成员swap函数
    ...
    template<class T>
    void swap(Widget<T>& a, Widget<T>& b) {
        a.swap(b);
    }
}
```

这样处理之后，在任意位置调用 `swap` 函数，根据名称查找法则（name lookup rules；更具体地来说是 argument-dependent lookup（实参依赖查找，ADL）），`swap` 函数会自动调用位于 `WidgetStuff` 命名空间中的 `swap` 函数。所以，如果想要让代码在尽可能多的上下文中使用为自定义类型专门设计的 `swap` 函数，就需要在同一个命名空间中写出成员函数和非成员函数的特化版本 `swap` 函数。

以上讲的是从 swap 的实现者角度出发的内容，但从调用者的角度也有一个很重要的情况值得探讨，假如一个 function template 中需要置换两个对象值：

```cpp
template <class T>
void do_something(T& obj1, T& obj2) {
    ...
    swap(obj1, obj2);
    ...
```

那么这里调用的 `swap` 究竟是哪一个呢？

- 是 `std::swap` 通用版本？
- 是专门为 `T` 类型写在 `std` 里的某个特化版本（可能存在，也可能不存在）？
- 是某个为 `T` 提供的特定 `swap` 函数？（可能存在，也可能在非 `std` 命名空间里）

其实一般来说，调用者的期望行为其实是：
✅ 如果有为 `T` 提供的特定 `swap`，优先用它；  
❎ 如果没有，那就回退到 `std::swap` 通用版本。

那么如何实现上述期望行为呢，答案是：

```cpp
template<typename T>
void do_something(T& obj1, T& obj2)
{
    using std::swap;       // 把 std::swap 引入当前作用域
    ...
    swap(obj1, obj2);      // 调用最合适的 swap
    ...
}
```

上述代码能够实现期望行为的原理在于，C++的 ADL 规则规定：如果 `T` 属于某个命名空间，例如 `WidgetStuff::Widget`，那么编译器在查找 `swap(obj1, obj2)` 时，会首先去 `WidgetStuff` 命名空间里找有没有合适的 `swap`。同时，`using std::swap;` 让 `std::swap` 在当前作用域可见，所以：

- 优先使用 `T` 所在命名空间里的 swap（ ADL 规则 ）
- 如果没有找到，就使用 `std::swap` 的模板实现
- 如果 `std::swap` 为 `T` 做了特化（full specialization），那么使用特化版本

错误写法：

```cpp
std::swap(obj1, obj2);  // ❌ 错误：强行限定只在 std:: 里找
```

这样写会强迫编译器只在 `std` 命名空间进行查找，从而失去了使用更高效 T-specific swap 的机会。  

不过，我们仍需要为 `std::swap<T>` 提供一个特化版本，因为虽然 `std::swap` 不应该被用户“扩展”，但标准库中仍有代码是这样写的：

```cpp
std::swap(obj1, obj2);  // 标准库内部或第三方库写死了 std::swap
```

为了让这些写法也能使用到专门设计的高效版本，仍需要完全特化 `std::swap<T>`。这种行为是 C++标准所允许的（即可以对 `std::swap` 的特定类型做 **全特化**），但不能对它做 **重载或偏特化**。

总结，如果我们是类的作者：

- 定义一个成员 `swap` 函数，高效地交换两个对象。
  - 建议 `noexcept`，因为 `swap` 常被用在强异常安全保障下。
- 在命名空间里定义非成员 `swap(T&, T&)`，调用成员 `swap`。
- 如果是类（非类模板），可以在 `std` 命名空间里全特化 `std::swap<T>`，也调用成员 `swap`。

如果我们是类的使用者，需要使用模板函数交换两个对象，那么应该使用形似下面的代码：

```cpp
template<typename T>
void doSomething(T& a, T& b) {
    using std::swap;
    swap(a, b);  // ✅ 支持 ADL 和 std::swap
}
```

## 实现

### 条款 26 尽可能延后变量定义式的出现时间

如果定义了一个变量而其类型带有一个构造函数或者析构函数，那么当出现变量的定义式时，就需要承受构造成本，当变量离开作用域时，就得承受析构成本，即使这个变量最终并未被使用，仍需耗费这些成本。

```cpp
// this function defines the variable "encrypted" too soon
std::string encryptPassword(const std::string& password)
{
    using namespace std;
    string encrypted;
    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    ...
    return encrypted;
}
```

这个函数中过早的定义了 `encrypted` 变量，如果函数抛出了异常，那么 `encrypted` 变量并不会被使用到，而仍需承受其构造和析构成本，所以应该延后其定义，将其放在抛异常代码的后面。

```cpp
// this function postpones encrypted’s definition until it’s truly necessary
std::string encryptPassword(const std::string& password)
{
    Using namespace std;
    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    string encrypted; // default constructor
    encrypted = password; // assign to encrypted
    encrypt(encrypted);
    return encrypted;
}
```

[[#条款 04 确保对象在使用前已被初始化]] 中说明了使用默认构造函数进行构造之后再进行复制是低效的，所以上面的代码在效率上仍可提高。

```cpp
// finally, the best way to define and initialize encrypted
std::string encryptPassword(const std::string& password)
{
    ... // import std and check length
    string encrypted(password); // define and initialize via copy constructor
    encrypt(encrypted);
    return encrypted;
}
```

还有一点是，如果一个变量只在循环内使用，那么应该是应该在循环外预先定义一个变量并在每次循环时进行赋值，还是该在循环中每次直接定义一个临时变量使用呢？

```cpp
// Approach A: define outside loop      // Approach B: define inside loop
Widget w;
for (int i = 0; i < n; ++i) {           for (int i = 0; i < n; ++i) {
    W = some value dependent on i;          Widget w ( some value dependent on i);
}                                       
    ...                                  ...
}                                       }
```

这里两种做法的成本时：

- A：一次构造、一次析构、n 次赋值
- B：n 次构造、n 次析构

所以，如果 class 的一次赋值成本低于一次构造+一次析构，那么方法 A 是比较高效的，尤其是当 n 很大的时候，否则方法 B 更好。不过方法 A 中 w 的作用域覆盖了整个循环，相比 B 而言，它的可理解性和易维护性要差一些。因此除非（1）明确知道赋值成本低于“构造+赋值”，（2）正在处理代码中效率高度敏感的部分，否则应该使用方法 B。

### 条款 27 尽量少进行类型转换

C++的规则设计旨在保证类型错误不可能发生。理论上，如果程序能顺利编译，就意味着它不会对任何对象执行不安全或无意义的操作。这是一个宝贵的保证，我们不应该轻易放弃它。遗憾的是，类型转换会破坏类型系统。这可能导致各种问题，有些容易识别，有些则极其微妙。

#### 类型转换语法回顾

1. C 风格类型转换：`(T) expression`
2. 函数风格类型转换：`T(expression)`
3. C++风格类型转换：`const_cast` 、`dynamic_cast`、`reinterpret_cast`、`static_cast`

1 和 2 并没有什么差别，只是小括号的位置不同，C++风格的 cast 则各有各的作用：

- `const_cast` 用来移除变量的常量性
- `dynamic_cast` 用来安全完成基类向派生类的向下转型（safe downcasting）。它是唯一无法由旧语法执行的动作，也是唯一可能耗费大量运行成本的转型动作。
- `reinterpret_cast` 会执行低级转型，实际转型结果可能取决于编译器，这意味着它不可一致。例如将一个 pointer to int 转型为一个 int，这个转换在低级代码以外很少见。
- `static_cast` 用来强迫进行隐式转换。例如将 non-const 转换为 const（[[#条款 03 尽可能使用 const]]），将 pointer-to-base 转换为 pointer-to-derived，但是它无法将 const 转换为 non-const，这只有 const_cast 才能做到。

新式类型转换的优点：

- 这种转换容易在代码中识别出来，因而简化“找出类型是在哪里被修改的”（简化问题排查）
- 类型转换的应用范围越窄，编译器越容易判断出出错的运用。

#### 类型转换是有代价的

许多程序员认为，类型转换其实什么都没有做，只是告诉编译器将某种类型视作另一种类型。但这是错误的观点，任何一个类型转换（不论是显示还是隐式调用的类型转换）往往会让编译器产生运行期执行的代码。比如将 int 转换成 double，这种类型转换肯定会消耗一些运行时成本，因为 int 的底层二进制表达方式就和 double 不同。

```cpp
class Base {... } ;
class Derived: public Base {... };
Derived d;
Base* pb = &d;  // 隐式地将 Derived* 转换为 Base*
```

上面的代码建立了一个 Base 指针指向一个 Derived 指针，但有时候两个指针值并不相同，这种情况下会有一个偏移量在运行时作用在 Derived 指针上，用以取得正确的 Base 指针。这个例子说明：单个对象（如 Derived 类型对象）可能拥有多个地址（如通过 Base 指针指向时的地址和通过 Derived 指针指向时的地址），一旦使用多重继承，这种现象几乎必然会出现，即使在单一继承中也有可能出现。需要特别注意的是，偏移量计算只是“有时”需要。不同编译器对对象内存布局和地址计算方式的实现各不相同。因此，即便基于“所了解的内存布局”的强制转换在当前平台有效，也绝不能保证在其他平台同样适用。无数程序员已为此付出过惨痛代价。

#### 类型转换的错误使用

在有些场景下，需要在派生类的 virtual 函数中调用其基类版本。假设我们有一个 `Window` 基类和 `SpecialWindow` 派生类，两者都定义了虚函数 onResize。请注意代码中的类型转换（虽然示例使用新式转换，但改用旧式转换结果不变）。

```cpp
class Window{
public:
    virtual void onResize(){...}
    ...
};

class SpecialWindow:public Window{
public:
    virtual void onResize(){
        static_cast<Window>(*this).onResize(); // 调用基类的实现代码
        ... // 这里进行SpecialWindow的专属行为.
    }
    ...
};
```

上述代码并非在当前对象上调用 `Window::onResize` 后再执行 `SpecialWindow` 的特定操作：它创建 `*this` 对象基类部分的临时副本并在该部分副本上调用 `Window::onResize`，然后才在当前对象上执行 `SpecialWindow` 的特定操作。如果 `Window::onResize` 修改了当前对象（这种可能性很大，因为 onResize 是非 const 成员函数），实际基类部分被修改的将是副本而非当前对象。若 `SpecialWindow::onResize` 修改了当前对象，就会导致当前对象处于无效状态：基类部分未被修改而派生类部分已被修改。

上面代码的正确写法是：

```cpp
void SpecialWindow::onResize(){
    Window::onResize(); //此时才是真正的调用基类部分的onResize实现.
    ...     //同上
}
```

#### 慎用 `dynamic_cast`

许多实现版本的 dynamic_cast 运行效率相当低下。例如至少有一种常见实现方案是基于类名的字符串比较。若对四层单继承体系中的对象执行 dynamic_cast，这种实现方案下每次转换可能需要进行多达四次 strcmp 调用来比较类名。更深的继承层次或多重继承体系的性能开销则更大。某些实现采用这种机制有其原因（主要与支持动态链接有关）。但除了对类型转换保持警惕外，在性能敏感代码中更应慎用 dynamic_cast。在探讨 dynamic_cast 的设计影响之前，有必要指出许多实现版本的 dynamic_cast 运行效率相当低下。例如至少有一种常见实现方案是基于类名的字符串比较。若对四层单继承体系中的对象执行 dynamic_cast，这种实现方案下每次转换可能需要进行多达四次 strcmp 调用来比较类名。更深的继承层次或多重继承体系的性能开销则更大。某些实现采用这种机制有其原因（主要与支持动态链接有关）。但除了对类型转换保持警惕外，在性能敏感代码中更应慎用 dynamic_cast。

(1)何时需要 dynamic_cast?

通常当你想在一个你认定为 derived class 对象上执行 derived class 操作函数时，但是你的手上只有一个指向 base 的指针或引用时，你会想到使用 dynamic_cast 进行转型

(2)如何不做转型，实现上述需求？

通常有两种做法可以解决上述问题：

- 方法一：使用容器，并在其中存储直接指向 derived class 对象的指针 (通常是智能指针)，这样就避免了上述需求。  
- 方法二：在 base class 内提供 virtual 函数做你想对各个派生类想做的事情。这样可以使得你通过 base class 接口处理“所有可能之各种派生类”。

总结：

- 尽可能避免使用类型转换，特别是在性能敏感的代码中要避免 dynamic_cast。如果设计必须使用类型转换，应尝试开发无需转换的替代方案。
- 当必须使用类型转换时，尽量将其隐藏在函数内部。客户端代码可以调用该函数，而不必在自己的代码中直接使用转换。
- 优先使用 C++风格的类型转换，而非旧式风格转换。它们更易于识别，且能更明确地表达转换意图。

### 条款 28 避免返回对象内部数据的句柄

Reference、指针、迭代器系统都是所谓的 handle (句柄，用来获得某个对象)。函数返回一个 handle，随之而来的便是“减低对象封装性”的风险。它也可能导致：虽调用 const 成员函数却造成对象状态被更改的风险。如果返回 handles 指向对象内部成分，则可能带来：常量性上的自相矛盾与悬垂引用问题。

```cpp
class Point{
public:
    Point(int x, int y);
    void setX(int newVal);
    void setY(int newVal);
};

struct RectData{
    Point ul_hc; // 左上角
    Point lr_hc; // 右下角
};

class Rectangle{
public:
    Point&upperLeft() const{return pData->ul_hc;}
    Point&lowerRight() const{return pData->lr_hc;}
    ...
private:
    std::shared_ptr<RectData> pData;
};
```

上面的示例代码中，为了让 `Rectangle` 对象尽可能小，将定义矩形的点放在了辅助类 `RectData` 中并存储其指针，根据 [[#条款 20 优先使用 pass-by-reference-to-const 而不是值传递]]，`Rectangle` 类提供了两个公开函数返回了指向代码底层的 `Point` 对象的引用。这样的设计可以通过编译，但是这是错误的，因为虽然 `upperLeft` 和 `lowerRight` 函数被声明为 `const` 成员函数，客户仍可以通过其修改 `Rectangle` 的内部数据。

```cpp
Point coord1(0, 0);
Point coord2(100, 100);
const Rectangle rec(coord1, coord2); // rec is a const rectangle from (0, 0) to (100, 100)
rec.upperLeft().setX(50); // now rec goes from (50, 0) to (100, 100)
```

因为 `Rectangle` 的内部成员变量只是存储了一个指向实际数据的指针，上面的代码虽然修改掉了内部数据，但是并没有修改内部数据指针指向，所以并不会产生编译错误。

这给我们带来两个教训。首先，数据成员的封装性取决于返回其引用的函数的最低访问级别。在本例中，虽然 `ul_hc` 和 `lr_hc` 本应是 ` Rectangle ` 的私有成员，但由于公有函数返回了它们的引用，实际上它们变成了公有成员。其次，如果 `const` 成员函数返回一个指向存储在对象外部数据的引用，那么该函数的调用者仍然可以修改这些数据（这是 `bitwise constness` 带来的后果——参见 [[#条款 03 尽可能使用 const]]）。

不过，返回成员函数指针的情况并不常见，因此让我们将注意力转回 Rectangle 类及其 `upperLeft` 和 `lowerRight` 成员函数。通过对返回类型添加 const 限定，可以消除我们为这些函数指出的两个问题：

```cpp
class Rectangle {  
public:  
    ...  
    const Point& upperLeft() const { return pData->ul_hc; }  
    const Point& lowerRight() const { return pData->lr_hc; }  
    ...  
};
```

通过这个修改后的设计，客户端可以读取定义矩形的 Points，但不能修改它们。这意味着将 `upperLeft` 和 `lowerRight` 声明为 `const` 不再具有欺骗性，因为它们不再允许调用者修改对象状态。至于封装性问题，我们原本就打算让客户端看到组成 `Rectangle` 的 `Points`，因此这是对封装性的有意识放宽。更重要的是，这种放宽是有限制的：这些函数仅授予读取权限，写入权限仍然被禁止。

即便如此，`upperLeft` 和 `lowerRight` 仍然返回指向对象内部数据的句柄，这可能会在其他方面产生问题。特别地，这可能导致悬垂句柄（dangling handles）——即指向已不复存在的对象组成部分的句柄。这类消失对象最常见的来源是函数返回值。

总结：

- 避免返回对象内部数据的句柄（引用、指针或迭代器）。不返回句柄可以增强封装性，帮助 const 成员函数保持 const 性质，并最大限度地减少悬空句柄的产生。

### 条款 29 努力实现异常安全

假设我们有一个用于表示带背景图的 GUI 菜单类。该类设计用于多线程环境，因此包含一个用于并发控制的互斥锁：  

```cpp
class PrettyMenu {
public:
    void changeBackground(std::istream& imgSrc); // 更改背景图片
private:
    std::mutex mtx;                // 互斥锁
    Image* bgImage;             // 当前背景图
    int imageChanges;           // 背景图修改次数
};
```

以下是该函数的实现：  

```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc) {
    mtx.lock();                  // 互斥锁上锁
    delete bgImage;              // 删除旧图片
    ++imageChanges;              // 更新计数器
    bgImage = new Image(imgSrc); // 安装新图片
    mtx.unlock();                // 释放互斥锁
}
```

从异常安全的角度来看，这个函数简直糟糕透顶。异常安全有两个基本要求，而它一个都不满足：  

1. **不产生资源泄漏**。上述代码违反了这条原则，因为如果"new Image (imgSrc)"抛出异常，unlock 调用将永远不会执行，导致互斥锁被永久持有。  
2. **不允许数据结构被破坏**。如果"new Image (imgSrc)"抛出异常：  
   - BgImage 将指向一个已被删除的对象（空悬指针）  
   - ImageChanges 已被递增，但实际上并未成功安装新图片

这两个要求中，解决资源泄漏的方法比较简单，即使用 [[#条款 14 资源管理类要小心复制行为]]中说明的方法进行资源管理。

```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc) {
    std::unique_lock<std::mutex> lock(mtx);
    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
}
```

异常安全函数的函数必须提供以下三个保证之一：①基本保证：保证对象和数据结构，即保证程序内的任何成员都保持有效状态，但不保证程序状态；举例来说，可以让 `changeBackground` 函数使得一旦有异常被抛出时， `PrettyMenu` 对象可以继续拥有原背景图像，或是令它拥有某个默认的背景图像，但客户无法预期哪一种情况。如果想知道，他们恐怕必须调用个成员函数以得知当时的背景图像是什么。②强烈保证：程序状态不改变；也就是说要么程序改变成功，如果失败的话，保持未调用函数之前的状态不变 ③nothrow 保证：承诺绝对不会抛出异常，也就是说能够保证功能一定会被成功完成（作用于内置类型上的所有操作都可做到）。

我们的每一个函数都应该满足异常安全，问题在于需要满足哪一种保证。最好的情况是给出 `nothrow` 保证，但是我们很难在代码中完全不调用任何一个可能抛出异常的函数，任何使用动态内存的东西（比如 STL 容器）如果无法找到足够的内存满足需求，通常会抛出 `bad_malloc` 异常（条款 49）。总结来说，如果可能的话就让代码提供 `nothrow` 保证，但是对大部分函数来说，需要在基本保证与强烈保证之间抉择。

就 changeBackground 函数而言，实现强保证并非难事。首先，将 `PrettyMenu` 的 `bgImage` 数据成员类型从内置的 `Image*` 指针改为 [[#条款 13 使用对象管理资源（RAII）]] 所述的智能资源管理指针。坦率地说，仅从防止资源泄漏的角度来看，这就是个明智的选择。而这一改动同时能帮助我们实现强异常安全保证，这再次印证了条款 13 的观点：使用对象（如智能指针）管理资源是良好设计的基础。其次，重新调整 `changeBackground` 中的语句顺序，确保在图像实际被修改后才增加 `imageChanges` 计数。作为通用准则，在操作确实完成之前，最好不要改变对象状态来标记操作已完成。

```cpp
class PrettyMenu { ... std::shared_ptr<Image> bgImage; ... }; 

void PrettyMenu::changeBackground(std::istream& imgSrc) {
    std::unique_lock<std::mutex> lock(mtx);
    bgImage.reset(new Image(imgSrc)); // 用"new Image"表达式的结果替换bgImage的内部指针
    ++imageChanges; 
}
```

这里不再需要手动删除旧图像，因为智能指针会在内部自动处理。更重要的是，只有当新图像成功创建后，旧图像的删除操作才会执行。更准确地说，只有 `new Image(imgSrc)` 成功时，`std::shared_ptr::reset` 函数才会被调用。`delete` 操作仅在 `reset` 函数内部使用，因此如果从未进入该函数，`delete` 就永远不会执行。上面的代码几乎足够让 `changeBackground` 函数提供异常安全强保证，但美中不足的地方在于参数 `imgSrc`，如果 `Image` 构造函数抛出异常，有可能 `input stream` 的读取记号 `read marker` 已被移走，而这样的移动对程序其他部分是一种可见的状态改变，所以 `changeBackground` 函数在解决这个问题之前只能说提供了异常安全基本保证。

另外，有一个实现强异常安全保证的通用设计策略，即拷贝-交换原则（copy and swap），其原则很简单：为打算修改的对象拷贝出一个副本，之后在该副本上进行修改，等到所有改变成功之后，再将修改后的副本与原对象进行一个不抛出异常的 swap（[[#条款 25 写出一个不抛异常的 swap 函数]]）。这样的话，假如修改动作产生了异常，swap 函数尚未得到执行，原对象会自然地保持不变。

```cpp
struct PMImpl { 
    std::shared_ptr<Image> bgImage; 
    int imageChanges;
};

class PrettyMenu {
...
private:
    std::mutex mtx;
    std::shared_ptr<PMImpl> pImpl;
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    using std::swap; // see Item 25
    std::unique_lock<std::mutex> lock(mtx);
    std::shared_ptr<PMImpl> pNew(new PMImpl( *pImpl)); // copy data
    pNew->bgImage.reset(new Image(imgSrc)); // modify the copy
    ++pNew->imageChanges;
    swap(pImpl, pNew); // swap the new data into place
} // release the mutex
```

`copy-and-swap` 策略是对状态做出“全有或全无”改变的一个好办法，但是它并不能保证整个函数都有异常安全强保证。假设有一个 `someFunc` 函数使用了 `copy-and-swap` 策略，但函数内还包括对另外两个函数 `f1` 和 `f2` 的调用，若 f 1 或 f 2 未达到强异常安全级别，要使 someFunc 具备强异常安全性将十分困难。例如，假设 `f1` 仅提供基本保证，那么 ` someFunc ` 若要实现强保证，就必须编写代码在调用 `f1` 前记录整个程序状态，捕获 `f1` 的所有异常，然后恢复原始状态。即使 `f1` 和 `f2` 都具备强异常安全性，情况也未必改善。毕竟当 `f1` 执行完成后，程序状态可能已发生任意改变，此时若 `f2` 抛出异常，即使 `f2` 本身未改变任何状态，程序状态也已不同于 `someFunc` 被调用时的初始状态。总结来说，异常安全具有木桶效用。

另外，`copy-and-swap` 策略需要为每个待修改对象创建副本，可能消耗无法或不愿承担的时间与空间成本。强异常安全保证虽极具吸引力，且在实际可行时应予以提供，但并非在所有情况下都具备可行性。当强保证不可行时，必须提供基本保证。实践中会发现，某些函数可以实现强保证，但对其他许多函数而言，效率或复杂度的代价会使强保证难以维系。只要在可行时已尽力提供强保证，那么仅提供基本保证就无可厚非。对多数函数而言，基本保证是完全合理的选择。

总结：

- 异常安全函数即使发生异常也不会泄漏资源、或让数据被破坏。根据安全程度可以分为：基本保证型、强保证型、nothrow 型
- 异常安全强保证往往可以通过 `copy and swap` 实现，但是并非对所有函数都可实现或具备可行性
- 异常安全具有木桶效用

### 条款 30 透彻了解 inline

1. Inline**只是对编译器**的一个**申请**，**不是强制命令**。现代 C++编译器中，inline 与最终函数是否内联没有必然关系，编译器可以对非 `inline` 函数进行内联展开（如果条件合适）。相反，也可以选择**不对 `inline` 函数进行内联**（例如函数太复杂、体积太大等）。现代 C++中，`inline` 的核心语义是允许该函数在多个翻译单元中重复定义（只要定义相同），否则，违反 ODR（One Definition Rule）。
2. Inlining 函数需考虑 object code 的大小；
3. 隐式 inline：将函数定义于 class 定义式内；显示 inline：在其定义式前加上关键字 inline；
4. Inline 函数通常一定被置于头文件内，因为 inlining 大部分情况下都是编译期行为；template 通常也被置于头文件内，因为大部分建置环境都是在编译期完成具现化
5. Template 函数不需要加 inline
6. 慎重决定 inlining 施行范围：将大多数 inlining 限制在小型、被频繁调用的函数身上，以便于日后的调试和二进制升级。

- 编译器通常不对“通过函数指针而进行的调用”实施内联，且需考虑后续代码维护用到函数指针的可能；
- 构造函数和析构函数并不适合用于 inlining，往往会引起代码的膨胀（所不要随便地将构造函数和析构函数的定义体放在类声明中）；
- Inline 函数代码如发生改变，所有用到该 inline 函数的程序都必须重新编译；
- 大部分调试器都不能对 inline 函数进行调试；

总结：

1. 如果 inline 函数不能增强性能，就避免使用它；
2. Inline 修饰符用于解决一些频繁调用的小函数大量消耗栈空间（栈内存）的问题；
3. Inline 函数本身不能是直接递归函数；
4. 将成员函数的定义体放在类声明之中（隐式 inline）虽然能带来书写上的方便，但不是一种良好的编程风格；
5. 关键字 inline 必须与函数定义体放在一起才能使函数成为内联，仅将 inline 放在函数声明前面不起任何作用，即 inline 是一种“用于实现的关键字”；声明前可以加 inline 关键字，但不符合高质量 C++/C 程序设计风格的一个基本原则：声明与定义不可混为一谈，用户没有必要、也不应该知道函数是否需要内联。

### 条款 31 将文件间的编译依存关系降至最低

#### 编译依存关系导致的问题

考虑以下代码：

```cpp
// file: Person.h
#include <string>
#include "Date"
#include "Address"

class Person {
public:
    Person(const std::string& name, const Date& birthday, const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private:
    std::string theName; // implementation detail
    Date theBirthDate; // implementation detail
    Address theAddress; // implementation detail
};
```

这段代码是一个给出了 `Person` 类声明的头文件，这个声明中包含了一些实现细节（成员变量），为了能够通过编译，需要引入 `string`、`Data`、`Address` 类的头文件来提供成员变量的定义，如果这三个头文件中有一个发生了变动，那么编译时每一个引入了变动的头文件以及引入 `Person.h` 头文件的代码都需要重新编译，这样的连串编译依存关系（cascading compilation dependencies）会极大的增加编译时长。

#### 前置声明

如果尝试去掉头文件，只使用前置声明，来解决上述问题，可能会写出下面的代码：

```cpp
namespace std {
    class string; // forward declaration (an incorrect one — see below)
} 

class Date; // forward declaration
class Address; // forward declaration

class Person {
public:
    Person(const std::string& name, const Date& birthday, const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private:
    std::string theName; // implementation detail
    Date theBirthDate; // implementation detail
    Address theAddress; // implementation detail
};
```

上面的代码存在两个问题：

1. `std::string` 并不是一个 `class`，而是 `basic_string<char>` 的类型别名，因此代码中针对 `std::string` 的前置声明并不正确。
2. 通过前置声明的方式引入类型时，该类型是一个 `incomplete type`，编译器无法在编译代码时获取成员变量的实际大小，从而也无法得知 `Person` 类的实际大小，那么当有代码单元中尝试实例化 `Person` 时，编译器在编译该编译单元时将会失败。

`std::string` 由标准库提供，并不可能频繁变动，也不太可能成为编译瓶颈，所以正常引入其头文件即可，而其他的类型则可以通过存储对应类型指针的方式解决 `incomplete type` 导致的问题，因为虽然类型大小无法得知，指针的大小却是固定的，那么 `Person` 类的大小就固定住了，不会再产生编译错误。

```cpp
#include <string> 

class Date; // forward declaration
class Address; // forward declaration

class Person {
public:
    Person(const std::string& name, const Date& birthday, const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private:
    std::string theName; // implementation detail
    Date* theBirthDate; // implementation detail
    Address* theAddress; // implementation detail
};
```

|是否可以用前置声明|情况|
|---|---|
|✅ 可以|类成员是指针或引用|
|✅ 可以|函数参数或返回值是指针或引用|
|❌ 不可以|成员是对象（非指针/引用）|
|❌ 不可以|按值传递或返回对象|
|❌ 不可以|使用对象成员、继承、多态等|

#### PImpl idiom（pointer to implementation 惯用法）

`pImpl` 惯用法是解决上面问题的常用方式，即将所有的实现细节隐藏在一个指针背后。将 `class` 切分成两个，一个只提供接口，另一个负责实现该接口。

```cpp
#include <string> // stl components, shouldn't be forward-declared
#include <memory> // for std::unique_ptr

class PersonImpl; // forward decl
class Date; // forward decl
class Address; // forward decl

class Person { // Person interface
public:
    Person(const std::string& name, const Date& birthday, const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private: // ptr to implementation;
    std::unique_ptr<PersonImpl> pImpl;
}; 
```

一个完整示例

`student.h`

```cpp
#pragma once
#include <memory>

class StudentImpl;

class Student {
public:
    Student(int Id);
    
    /** 因为此处智能指针使用的是 unique_ptr ，它为了保证高效性，
     * 其删除器是自身的一部分，它必须保证 raw pointer 为 complete 对象。
     * 由于编译器默认生成的析构函数是 inline ，
     * 此时 Impl 所指之物仅仅是前置声明，是一个 non-complete 对象，所以会报错。
     * 因此如果使用 unique_ptr 而不是 shared_ptr 实现 Impl 时，
     * 不要使用默认的析构行为，请自行额外实现。
     * 因为shared_ptr不使用自身的 deleter，无需这种保证。
     */
    ~Student();
    
    Student(const Student&) = delete;
    Student& operator=(const Student&) = delete;
    Student(Student&&) = delete;
    Student& operator=(Student&&) = delete;
    void SetId(int Id) const;
    int GetId() const;
private:
    std::unique_ptr<StudentImpl> Impl;
};
```

`student.cpp`

```cpp
#include "Student.h"
class StudentImpl {
public:
    StudentImpl(int mId) {
        Id = mId;
    }

    ~StudentImpl() = default;
    StudentImpl(const StudentImpl&) = default;
    StudentImpl& operator=(const StudentImpl&) = default;
    StudentImpl(StudentImpl&&) = default;
    StudentImpl& operator=(StudentImpl&&) = default;

    void SetId(int mId) {
        Id = mId;
    }

    int GetId() const {
        return Id;
    }

private:
    int Id;
};

Student::Student(int Id): Impl(std::make_unique<StudentImpl>(Id)) {}
Student::~Student() = default;

void Student::SetId(int Id) const {
    Impl->SetId(Id);
}

int Student::GetId() const {
    return Impl->GetId();
}
```

`main.cpp`

```cpp
#include <iostream>  
#include "student.h"

int main()
{
    const Student John(233);
    std::cout << John.GetId() << "\n";

    John.SetId(666);
    std::cout << John.GetId() << "\n";

    return 0;
}
```

#### 分离声明与定义

不论是前置声明还是 pImpl，其实都是在用 `声明的依赖关系` 替代 `定义的依存关系`，这正是编译依存性最小化的本质：让头文件尽可能 `self-sufficient`，如果做不到，则对其他文件内的 `声明`（而非 `定义`）进行依赖。

- **如果可以使用指针或引用，就不要直接定义。** 如果只使用指针或引用，则可以通过前置声明的方式定义出指向该类型的指针或引用；但如果定义某类型的变量，则需要用到该类型的定义式。
- **如果可以，尽量使用 `class声明` 替代 `class定义`**。比如，当声明某一个函数而该函数用到某个 `class` 时，其实并不需要该 `class` 的定义。

```cpp
class Data; // forward decl
Date today();
void clearAppointments(Data d);
```

- **为声明和定义提供不同的头文件。** 为了严格遵守上述准则，需要两个头文件，一个用于声明，一个用于定义。当前，这两个头文件必须保持一致性，如果有个声明被改变了，两个文件都得改变。实际使用时，总是 `#include` 所提供的声明文件而不是手工前置声明若干函数。按照命名惯例，这个只提供声明的头文件常以 `fwd` 结尾，比如 C++标准库中的 `iosfwd` 包含了 `iostream` 各组件的声明，对应的定义则分布在若干不同的头文件内，包括 `sstream`、`streambuf`、`fstream` 和 `<iostream>`

#### Handle class 与 Interface class

像上面所述的如 `Person` 这样使用 `pImpl idiom` 的类，往往被称为 `Handle classes`。`Person` 类所有函数的实现由对应的 `impl类` 提供。

另一种制造 `Handle class` 的方式是，让 `Person` 成为一个抽象基类，称为 `Interface class`，它通常不带成员变量，也没有构造函数，只有一个虚析构函数以及一组纯虚函数，用来描述整个接口。

```cpp
class Person {
public:
    virtual ~Person();
    virtual std::string name() const = 0;
    Virtual std:: string birthDate () const = 0;
    virtual std::string address() const = 0;
    ...
};
```

由于 `Person` 类被做成了抽象类，它无法被示例化，因此客户端在使用 `Person` 类需要持有其指针或引用，并常常调用一个工厂函数或者虚构造函数，通过动态分配获取实际的 `concrete派生类` 对象。

```cpp
class Person {
public:
...
static std::shared_ptr<Person> create(const std::string& name, const Data& birthday,
                                      const Address& addr);
...
};
```

客户端代码：

```cpp
std::string name;
Date dateOfBirth;
Address address;
...
// create an object supporting the Person interface
std::shared_ptr<Person> pp(Person::create(name, dateOfBirth, address));
...
std::cout << pp->name() << " was born on " << pp->birthDate() << " and now lives at "
          << pp->address();
```

`concrete派生类` 代码：

```cpp
class RealPerson: public Person {
public:
    RealPerson(const std::string& name, const Date& birthday, const Address& addr) :
         theName(name), 
         theBirthDate(birthday), 
         theAddress(addr) {}
    virtual ~RealPerson() {}
    std::string name() const; // implementations of these
    std::string birthDate() const; // functions are not shown, but
    std::string address() const; // they are easy to imagine
private:
    std::string theName;
    Date theBirthDate;
    Address theAddress;
};
```

`create` 函数：

```cpp
std::shared_ptr<Person> Person::create(const std::string& name, const Date& birthday, 
                                       const Address& addr) {
    return std::shared_ptr<Person>(new RealPerson(name, birthday,addr));
}
```

`Handle classes` 和 `Interface classes` 是有成本的，`handle class` 的每一个访问都要承受一次通过指针进行间接访问的带家，并且多消耗了一个指针大小的内存。另外，动态内存分配也会带来额外开销，以及产生 `bad_alloc` 异常的可能性。`interface classes` 由于每个函数都是虚函数，因此每次函数调用都会产生一次简介跳跃的代价，并且每个派生类对象都要存储虚函数表，从而产生内存上的消耗。

#### 总结

- 支持编译依存性最小化的一般构想是：相依于声明式，不要相依于定义式。基于此构想的两个手段是 Handle classes 和 Interface classes。
- 程序库头文件应该以完全且仅声明式（full and declaration-only forms）的形式存在。这种做法不论是否涉及 templates 都适用。

## 继承与面向对象设计

### 条款 32 确定 public 继承呈现出泛化关系（is-a）

- Public 继承的意思是：**子类是一种特殊的父类**，这就是所谓的“is-a”关系。
- 在使用 public 继承时，**子类必须涵盖父类的所有特点，必须无条件继承父类的所有特性和接口**。(设计模式中的里氏替换原则: LSP，Liskov Substitution principle)

虽然 public 继承意味着 is-a 的关系看似简单，但有时候如果单纯偏信生活经验，会犯错误。例如：

- 🐧不是🐦

下面的代码为 Bird 类定义了 fly 接口，并让企鹅继承了 Bird 类。

```cpp
class Bird {
public:
virtual void fly(); // birds can fly
...
};

class Penguin: public Bird { // penguins are birds
...
}
```

如果考虑飞行这一特性（或接口），那么企鹅类在继承中就绝对不能用 public 继承鸟类，因为企鹅不会飞，我们要在**编译阶段消除调用飞行接口的可能性**；但如果所关心的接口是下蛋的话，按照我们的法则，企鹅类就可以 public 继承鸟类。

```cpp
class Bird {  //没有声明 fly 函数
public:
    virtual ~Bird();
};

class FlyingBird: public Bird {
public:
    virtual void fly();
    ...
};

class Penguin : public Bird {     //没有声明 fly 函数
public:
    //...
};

inline void Try(){
 Penguin P;
 P.fly();  //编译阶段就会报错！
}
```

- 正方形不是矩形

生活经验告诉我们正方形是特殊的矩形，但这并不意味着在代码中二者可以存在 public 的继承关系，矩形具有长和宽两个变量，但正方形无法拥有这两个变量——没有语法层面可以保证二者永远相等，那就不要用 public 继承。

Note：
除了 is-a (泛化关系)外，类之间还有两个常见的关系：has-a (聚合关系)和 is-implemented-in-terms-of 关系，这些关系将在条款 38 和条款 39 中讨论。

总结：

- Public 继承意味 is-a。适用于 base classes 身上的每一件事情一定也适用于 derived classes 身上，因为每一个 derived class 对象也都是一个 base class 对象。
- 在确定是否需要 public 继承的时候，我们首先要搞清楚**子类是否必须拥有父类每一个特性**，如果不是，则无论生活经验是什么，都不能视作”is-a”的关系。**public 继承关系不会使父类的特性或接口在子类中退化，只会使其扩充。**

### 条款 33 避免继承中的名称遮掩

#### 什么是名称遮掩
>
> 当编译器在 func 的作用域并使用 x 时，它会先在 local 作用域查找是否存在该变量，如果找不到再扩大作用域。显然，编译器会在 local 找到 double x，然后停止查找，这意味着在 local 中使用 x，将总是找到 double x，而非 global 作用域的 int x。这便是我们所说的：名称遮掩（name-hiding）。

```cpp
int x=1;   //全局变量
void func() {
   double x=1.1;  //局部变量
   std::cout<<x<<"\n";
}
```

#### 继承中存在的名称遮掩问题

当继承的类存在多次重载的虚函数时，也会产生名称遮掩问题。在父类中，假如虚函数 `foo()` 被重载了两次，可能是由于参数类型重载（`foo(int)`），也可能是由于 `const` 属性重载 (`foo() const`)。如果子类仅对父类中的 `foo()` 进行了覆写，那么在子类中父类的另外两个实现 (`foo(int)` , `foo() const`)也无法被调用，这就是继承中存在的名称遮盖问题——名称在**作用域级别的遮盖是和参数类型以及是否虚函数无关的**，即使子类重载了父类的一个同名，父类的所有同名函数在子类中都被遮盖。

```cpp
class Base {
public:
   virtual void mf1() = 0;
   virtual void mf1(int);
   virtual void mf2();
   void mf3();
   void mf3(double);
private:
};

class Derived : public Base {
public:
   virtual void mf1();
   void mf3();
};

inline void Try() {
   Derived d;
   int x;
   d.mf1();   //没问题，调用Derived::mf1
   d.mf1(x);  //报错，因为 derived::mf1 遮掩了 base:: mf1 (int) 
   d.mf2();   //没问题，调用Base::mf2
   d.mf3();   //没问题，调用Derived::mf3
   d.mf3(4);  //报错，因为 derived::mf3 遮掩了 base::mf3 （double）
}
```

#### 避免名称遮掩的方式

##### 使用 using 声明

让 base class 内的某些事物可以在 derived class 作用域中可见。注意使用 using 声明式的权限符为 public，注意不要违反继承时的继承权限。对于 public 继承，并不是所有的函数都被继承，因而不是所有的函数都可以进行声明访问。尝试声明无法访问的函数，编译器会自动报错。

```cpp
class Derived : public Base {
public:
   using Base::mf1; // 让Base class内所有名为mf1的东西都在Derived作用域内可见
   using Base::mf3; // 让Base class内所有名为mf3的东西都在Derived作用域内可见
   virtual void mf1();
   void mf3();
};
```

##### 使用转发函数（forwarding function）

如果使用 public 继承，那么如 [[#条款 32 确定 public 继承呈现出泛化关系（is-a）]] 所述，派生类应当保留基类的所有函数，那么就应该将基类的所有重载版本都使用 using 声明进行引入。但是假设 Derived 以 private 形式继承 Base，而 Derived 唯一想继承的 mf 1 是那个无参数版本。那么这里就不应该再使用 using 声明式来引入该函数了，因为 using 声明式会令继承而来的某给定名称之所有同名函数在 derived class 中都可见，这里可以使用转交函数 (forwarding function)进行解决。

```cpp
class Base {
public:
   virtual void mf1()=0;
   virtual void mf1(int);
   virtual void mf2();
   void mf3();
   void mf3(double);
private:
   int x;
};

class Derived : private Base {
public:
   virtual void mf1() { //转交函数（forwarding function）
      Base::mf1();      //暗自成为inline
   }
   ...
};

inline void Try() {
   Derived d;
   int x;
   d.mf1();   //没问题，调用Derived::mf1
   d.mf1(x);  //报错，  Base::mf1()被遮掩了 
}
```

总结：

- 派生类中的名称会隐藏基类中的名称。在公有继承下，这种情况永远不是我们所期望的。
- 要使被隐藏的名称重新可见，可使用 using 声明或转发函数。

### 条款 34 区分接口继承和实现继承

- public 继承其实可以分成：`函数接口（function interfaces）` 继承和 `函数实现（function implementation）` 继承。这意味着 derived class 不仅可以有 base class 函数的声明，还可以有 base class 函数的实现。
- 成员函数的 `接口总是会被继承`。因为条款 32 曾说：public 继承是 is-a 的关系，任何可以作用于 base class 的函数也一定可以作用于 derived class。
- 不同类型的函数代表了**父类对子类实现过程中不同的期望**。
- 在父类中声明纯虚函数，是为了**强制子类拥有一个接口**，并**强制子类提供一份实现**。
- 在父类中声明非纯虚函数，是为了**强制子类拥有一个接口**，并**为其提供一份默认实现**。
- 在父类中声明非虚函数，是为了**强制子类拥有一个接口以及规定好的实现**，并不允许子类对其做任何更改（条款 36 要求我们不得覆写父类的非虚函数）。

> 在这其中，有可能出现问题的是普通虚函数，这是因为父类的默认实现并不能保证对所有子类都适用，因而当子类忘记实现某个本应有定制版本的虚函数时，父类应**从代码层面提醒子类的设计者做相应的检查**，很可惜，普通虚函数无法实现这个功能。一种解决方案是，在父类中**为纯虚函数提供一份实现**，作为需要主动获取的默认实现，当子类在实现纯虚函数时，检查后明确默认实现可以复用，则只需调用该默认实现即可，这个主动调用过程就是在代码层面提醒子类设计者去检查默认实现的适用性。

```cpp
class Airplane {
public:
    virtual ~Airplane() = default;

    //pure-virtual  子类继承函数接口，强制覆写
    virtual void ToString() const = 0;
    //impure-virtual/virtual  子类继承函数接口和默认函数实现，可覆写
    virtual void Fly();
    //non-virtual   子类继承函数接口和默认函数实现，不可覆写
    void DefaultFly();
};

class ModelA : public Airplane { //public 子类继承函数接口
public:
    void ToString() const override;
    void Fly() override;
};
```

- 将纯虚函数、虚函数区分开的并不是在父类有没有实现——纯虚函数也可以有实现，其二者本质区别在于父类对子类的要求不同，前者在于**从编译层面提醒子类主动实现接口**，后者则侧重于**给予子类自由度对接口做个性化适配**。非虚函数则没有给予子类任何自由度，而是要求子类坚定的遵循父类的意志，**保证所有继承体系内能有其一份实现**。  

- 两个常见错误：

> **1. 将所有函数都声明为 non-virtual。**  
> 这会使得 derived class 没有余裕空间进行特化工作。Non-virtual 函数还会带来析构问题，见条款 7。实际上任何 class 如果打算使用多态性质，都会有若干 virtual 函数。如果你关心 virtual 函数的成本，请参考 80-20 法则：一个典型的程序有 80%的执行时间花费在 20%的代码身上。这个法则十分重要，这意味着平均而言你的函数调用中可以有 80%是 virtual，而不冲击程序的大体效率。所以当你担心是否有能力负担 virtual 函数的运行成本时，先关注那举足轻重的 20%代码身上。  
> **2. 将所有函数都声明为 virtual。**  
> 有时候是正确的，比如 interfaces class。然而这也可能是 class 设计者缺乏坚定立场的表现，某些函数就是不该在 derived class 中被重新实现，你就应该把它声明为 non-virtual。

总结：

1. 接口继承和实现继承不同。在 public 继承之下，derived class 总是继承 base class 的接口。
2. Pure virtual 函数只具体指定接口继承。
3. 简朴的（非纯）impure virtual 函数具体指定接口继承及默认实现继承。
4. Non-virtual 函数具体指定接口继承以及强制性实现继承。

### 条款 35 考虑虚函数以外的其他选择

C++的 virtual 函数让我们能方便地实现接口继承与实现继承，但同时也会让我们忽略可能的其他方案。本条款针对于 virtual 函数的功能设计了具有不同优缺点的替代方案。

#### 通过虚函数完成的典型方案

假设有一款游戏涉及到各式角色的健康情况，但不同角色的健康度是不同的，这时候将计算健康度的函数声明为 virtual 是最直观的设计方式。

```cpp
class Character {
public:
    virtual ~Character()=default;
    virtual int CalculateHealthValue() const;
};

class NpcEvil:public Character {
public:
    virtual int CalculateHealthValue() const override;
};
```

这种方式确实非常直观，但从某种意义上说，这种直观性使得我们不会充分考量其他替代方案。为了跳出面向对象设计的思维定式，此条款在这里探讨几种不同的解决思路。

#### 藉由 `Non-Virtual Interface` 手法 （NVI） 实现 Template Method 模式

> 有一个有趣的思想流派**主张 virtual 函数应该几乎总是 private**。这个流派建议：**保留 CalculateHealthValue 为 public 成员函数，但让它成为 non-virtual，并调用一个 private virtual 函数（例如 OnCalculateHealthValue）进行实际工作**

```cpp
class Character {
public:
    virtual ~Character()=default;
    int CalculateHealthValue() const {
        const int Result= OnCalculateHealthValue();
        //...
        return Result;
    }
private:
    virtual int OnCalculateHealthValue()const=0;
};

class NpcEvil:public Character {
private:
    int OnCalculateHealthValue() const override {
        return 100;
    }
};
//这种设计方式：令客户通过 public non-virtual 成员函数间接调用 private virtual 函数，称为 non-virtual interface （NVI）手法。
```

值得注意的一点，**C++ 允许 derived class 覆写 base class 的 private virtual 方法**。看起来诡异，但这是真的。

优点

> NVI 手法的一个优点是可以在调用 private virtual 函数前后做一些额外的事情，其实这也是封装带来的好处。调用之前可以做的工作：锁定互斥器，制造运转日志记录项，验证 class 约束条件，验证函数先决条件等等。调用之后可以做的工作：互斥器解除锁定，验证函数的事后条件，再次验证 class 约束条件等等。但假如没有这一层封装，直接调用 virtual 函数，就没有任何好办法可以做这些事。

缺点

> 在某些 class 继承体系中，virtual 函数必须调用其 base class 的版本，这就导致 virtual 函数必须是 protected 而不能是 private，有些时候 virtual 函数甚至一定得是 public。在这种情况下，non-virtual 成员函数和 virtual 成员函数都是 public 的，NVI 的 wrapper 手法显然就不成立了。

#### 通过 `std::function` 完成策略模式

```cpp
#include <functional>
class Character {
public:
    virtual ~Character() = default;
    //接受一个reference 指向const Character, 并返回int
    using FCalculateHealthValueFunc = std::function<int(const Character&)> ; //注意 const & 不能隐式转化

    explicit Character(FCalculateHealthValueFunc InCalculateHealthValueFunc) :
                       CalculateHealthValueFunc(InCalculateHealthValueFunc) { }

    int CalculateHealthValue() const {
        const int Result = CalculateHealthValueFunc(*this);
        //...
        return Result;
    }

    void SetCalculateHealthValueFunc(FCalculateHealthValueFunc InCalculateHealthValueFunc) {
        CalculateHealthValueFunc = InCalculateHealthValueFunc;
    }

private:
    FCalculateHealthValueFunc CalculateHealthValueFunc;
};

class NpcEvil : public Character {
public:
    explicit NpcEvil(FCalculateHealthValueFunc InCalculateHealthValueFunc)
        : Character(InCalculateHealthValueFunc) {}
};

class FCalculator {
public:
    static double CalculateHealthGM(const Character& InNpcEvil) {
        return 2147483647;
    }

    int CalculateHealthNormal(const Character& InNpcEvil) {
        return 5;
    }

};

int main() {
    //函数指针仍然适用
    NpcEvil John([](const Character&) {
        return 100;
        });
    std::cout << John.CalculateHealthValue() << "\n";

    //使用返回值为 double 的 static member函数
    John.SetCalculateHealthValueFunc(&FCalculator::CalculateHealthGM);
    std::cout << John.CalculateHealthValue() << "\n";

    FCalculator Calculator;

    //使用某对象的 member 函数
    John.SetCalculateHealthValueFunc(std::bind(&FCalculator::CalculateHealthNormal, Calculator, std::placeholders::_1));
    Std:: cout << John.CalculateHealthValue () << "\n";
}
```

总结：

- Virtual 函数的替换方案包括 NVI 手法以及 Strategy 设计模式的多种形式。NVI 手法自身是一个特殊形式的 Template Method 设计模式。
- 将机能从成员函数移到 class 外部函数，带来的一个缺点是，非成员函数无法访问 class 的 non-public 成员。
- `std::function` 对象的行为就像一般函数指针。这样的对象可接纳与给定之目标签名式（target signature）兼容的所有可调用物（callable entities）。

### 条款 36 不要重新定义继承而来的非虚函数

- 如果你的函数**有多态调用的需求**，一定记得把它**设为虚函数**，否则在动态调用（基类指针指向子类对象）的时候是不会调用到子类重载过的函数的，很可能会出错。
- 反之同理，如果一个函数父类没有设置为虚函数，一定不要在子类重载它。
- 原因：多态的动态调用中，只有虚函数是**动态绑定**，非虚函数是**静态绑定**的——指针（或引用）的静态类型是什么，就调用那个类型的函数，和动态类型无关。

```cpp
class B {
public:
    void mf();
    ...
};

class D : public B {
public:
    void mf();
    ...
}

D x;
B* pB = &x;
D* pD = &x;

pB->mf();
pD->mf();
```

上面代码演示了假如 D 类重新定义其 B 类中的非虚函数会发生什么，非虚函数如 `B::mf` 和 `D::mf` 都是静态绑定的，代码中 pB 被定义为一个 pointer-to-B、pD 被定义为一个 pointer-to-D，那么 `pB->mf()` 将会调用 `B::mf`，`pD->mf()` 将会调用 `D::mf`。只有 `mf` 被修改为虚函数，将函数进行动态绑定，两次函数调用才能如预期一样调用到 `D::mf`。

> [[#条款 07 为多态基类声明 virtual 析构函数]] 解释为什么**多态性质的 base classes 应该声明 virtual 析构函数**。如果你在多态性质下的 base class 声明了 non-virtual 函数，那么 derived class 便绝不应该重新定义一个继承而来的 non-virtual 析构函数。但即使你没有定义，[[#条款 05 了解 C++默认生成了哪些函数]] 指出**编译器会默认为你生成它**，所以多态性质的 base classes 都需要 virtual 析构函数。因此就本质而言，[[#条款 07 为多态基类声明 virtual 析构函数]] 只不过是本条款的一个特殊案例，尽管它足够重要到单独成为一个条款。

### 条款 37 不要重新定义继承而来的默认参数值

静态绑定和动态绑定的差异

> 对象的所谓静态类型（static type），就是它在程序中被声明时采用的类型。对象的所谓动态类型（dynamic type），就是指目前所指对象的类型，可以决定一个对象将会有什么样的动态行为。Virtual 函数是动态绑定的，所以调用一个 virtual 函数时，究竟调用那一份函数实现代码，取决于该对象的动态类型。

在继承中：

- 不要更改父类非虚函数的默认参数值，其实**不要重载父类非虚函数的任何东西**，不要做任何改变！(见 [[#条款 36 不要重新定义继承而来的非虚函数]])
- 虚函数不要写默认参数值，子类自然也不要改，**虚函数要从始至终保持没有默认参数值**。

> 默认参数值是属于**静态绑定**的，而**虚函数属于动态绑定**。虚函数在大多数情况是供动态调用，而在动态调用中，子类做出的默认参数改变其实并没有生效，反而会引起误会，让调用者误以为生效了。

```cpp
class B {
public:
    virtual std::string ToString(std::string Text="BBB") const{
        return Text;
    }
};

class D : public B {
public:
    virtual std::string ToString(std::string Text="DDD") const{
        return Text;
    }
};

inline void BTry() {
    D Derived;
    B* BasePointer = &Derived;
    std::cout << BasePointer->ToString() << "\n"; //BBB
}
//BasePointer 静态类型是 B，动态类型是 D。意味着 ToString 的定义取决于 D，而 ToString 的默认参数值却取决于 B。
```

默认参数值属于**静态绑定**的原因是为了提高**运行时效率**。

> 假设默认参数值为动态绑定，编译器就必须要支持某种方式在运行期为 virtual 函数选择适当的默认参数值，这意味着更慢更复杂。

假如你需要重新定义默认参数值的需求

> 1. 替换 virtual 函数。[[#条款 35 考虑虚函数以外的其他选择]] 列出了不少 virtual 函数的替换设计。  
> 2. 如果你真的想让某一个虚函数在这个类中拥有默认参数，那么就把这个虚函数设置成 private，在 public 接口中重制非虚函数，让非虚函数这个“外壳”拥有默认参数值（NVI），当然，这个外壳也是一次性的——在被继承后不要被重载。

总结：

- 绝对不要重新定义一个继承而来的默认参数值，因为默认参数值都是静态绑定，而 virtual 函数——你唯一应该覆写的东西——却是动态绑定。

### 条款 38：通过复合塑造出 has-a 或 is-implemented-in-terms-of 关系

- 两个类的关系除了继承之外，还有“一个类的对象可以作为另一个类的成员”，我们称这种关系为“类的复合”
- Public 继承是一种 is-a 的意义，复合也有它们的意义。复合意味着 has-a（有一个）或 is-implemented-in-terms-of （根据某物实现出）。
- `is-a` 和 `is-implemented-in-terms-of`  的区别：

> 这两种关系其实是在不同领域的表现，如果对象只是你所塑造的世界中的某个物品，某些人物等，那这样的对象就属于应用域部分，如果对象需要负责你所塑造世界的细节部分，是规则的制定者和执行者，那这样的对象就属于实现域部分。当对象处于应用域，它就是 has-a 的关系，当对象处于实现域，它就是 is-implemented-in-terms-of 的关系。  
> 请牢记“is-a”关系的唯一判断法则，一个类的全部属性和接口是否必须**全部**继承到另一个类当中？另一方面，“用一个工具类去实现另一个类”这种情况，是需要对工具类进行**隐藏**的，比如人们并不关心你使用 stack 实现的 queue，所以就藏好所有 stack 的接口，只把 queue 的接口提供给人们用就好了，而红芯浏览器的开发者自然也不希望人们发现 Google Chrome 的内核作为底层实现工具，也需要“藏起来”的行为。

- 什么情况下我们应该用类的复合？  
当某一个类“拥有”另一个类对象作为一个属性（has-a），比如学生拥有铅笔、市民拥有身份证，一个人可以有名字，有地址，有手机号码等。

```cpp
class Address;
class PhoneNumber;
class Person {
public:
    //...
private:
    std::string name;  //复合对象
    Address& Address;
    PhoneNumber& VoiceNumber;
    PhoneNumber& FaxNumber;
};
```

"一个类根据另一个类实现”(is-implemented-in-terms-of )。比如“用 stack 实现一个 queue”，更复杂一点的情况可能是“用一个老版本的 Google Chrome 内核去实现一个红芯浏览器”。再比如用 list 对象实现一个 sets

这里以 list 对象实现一个 sets 为例， **set 成员函数可大量倚赖 list 及标准程序库其他部分提供的机能来完成：**

```cpp
template <typename T>
class Set {
public:
    bool Contains(const T& Item) const;
    void Insert(const T& Item);
    void Remove(const T& Item);
    size_t Size() const;
private:
    std::list<T> Rep;
};

template <typename T>
bool Set<T>::Contains(const T& Item) const {
    bool Result=std::find(Rep.begin(),Rep.end(),Item)!=Rep.end();
    return Result;
}

template <typename T>
void Set<T>::Insert(const T& Item) {
    if(!Contains(Item)) {
        Rep.push_back(Item);
    }
}

template <typename T>
void Set<T>::Remove(const T& Item) {
    typename std::list<T>::iterator It=std::find(Rep.begin(),Rep.end(),Item);
    if (It!=Rep.end()) {
        Rep.erase(Item);
    }
}

template <typename T>
size_t Set<T>::Size() const {
    return Rep.size();
}
//显然，set 和 list 的关系是 is-implemented-in-terms-of，而不单单是 has-a 的关系。
```

总结

- 复合（composition）的意义和 public 继承完全不同。
- 在应用域（application domain），复合意味 has-a （有一个）。在实现域（implementation domain），复合意味 is-implemented-in-terms-of（根据某物实现出）。

### 条款 39：小心谨慎地使用 private 继承

[[#条款 32 确定 public 继承呈现出泛化关系（is-a）]] 中说到**public 继承是一种 is-a**关系。在这种继承体系下，编译器在必要时刻（为了让函数调用成功）会将 derived class 转换为 base class 。

```cpp
class Person {...};
class Student : public Person {...};
void eat(const Person& p);
​
Person p;
Student s;
​
eat(p); // 没问题
eat(s); // 没问题，这里编译器将Student暗自转换为Person类型。
```

Private 继承的两个行为

- 如果 derived class 和 base class 是 **private 继承**，那么从 derived class 到 base class 的**转换将失败**
- 在**private 继承**下，base class 的成员无论是 private、protected 还是 public，继承后**都会变为 private**。

```cpp
class Person {...};
class Student : private Person {...};
void eat(const Person& p);
​
Person p;
Student s;
​
eat(p); // 没问题
eat(s); // 错误！！！s是private继承，编译器无法转换。
```

Private 继承的意义

- private 继承意味着：is-implemented-in-terms-of （根据某物实现出）。Private 继承可以看作纯粹是 `为了实现细节`，它需要的不是类似 public 继承可以向外提供接口，仅仅是为了让 derived class 采用 base class 中已经具备的某种特性。Derived 和 base 之间并没有什么直接意义上的联系。

那么当我们拥有“用一个类去实现另一个类”的需求的时候，如何在类的复合与 private 继承中做选择呢？

- 尽可能用复合，**除非必要，不要采用 private 继承**。
- 当我们需要对工具类的某些方法（虚函数）做重载时，我们应选择 private 继承，这些方法一般都是工具类内专门为继承而设计的调用或回调接口，需要用户自行定制实现。

案例一：能用复合，就不要用 private

> 假设我们需要写一个 Widget（控件）。这个控件需要按某一频率定时检查 Widget 的某些信息，换句话说需要定时地调用某个函数。为了少写新的代码，我们在其他程序中翻到了一个 Timer class。

```cpp
class Timer 
{
public:
    explicit Timer(int tickFrequency);
    virtual void onTick() const;
};
```

> 这个定时器的功能是每隔一段时间就调用一次 onTick 函数。  
> 为了让 Widget 重新定义 virtual 内的 virtual 函数，Widget 必须继承自 Timer。但此时不能使用 public 继承，**因为 Widget 并不是个 Timer**，所以我们必须以 private 形式继承 Timer。

```cpp
class Widget : private Timer 
{
private:
    virtual void onTick() const override;
}
```

> 但**private 继承并不是唯一的选择方案**，我们可以**使用复合**来替代这个方案。  
> 只要在 Widget 内声明一个**嵌套式 private class**, 后者以 public 形式继承 Timer 并重新定义 onTick, 然后在 Widget 内放一个这种类型的对象

```cpp
class Widget 
{
private:  //private的
    class WidgetTimer : public Timer 
    {
    public:
        virtual void onTick() const override;  //本应被目标类覆写的方法在嵌套类中实现，这样TargetClass的子类就无法覆写该方法。
    };
    WidgetTimer timer;
};
```

**该复合设计相比于 private 继承有两个优点：**

- **当 Widget 拥有 derived class 时，你可能同时想阻止 derived class 重新定义 onTick。** 如果是 private 继承（Widget 继承了 Timer），那这个想法就不可能实现。（[[#条款 35 考虑虚函数以外的其他选择]]中指出：derived class 可以重新定义 private virtual 函数，即使派生类中并不能调用它）。但如果 WidgetTimer 是 widget 内部的一个 private 成员并继承 Timer，Widget 的 derived classes 将无法取用 WidgetTimer，因此无法继承它或重新定义它的 virtual 函数。  
- **降低 widget 的编译依存性。** 如果继承 Timer，当 Widget 被编译时 Timer 的定义必须可见，所以定义 widget 的文件必须 `#include "Timer.h"`。如果 WidgetTimer 移出 Widget 之外，而 widget 内含指针指向一个 widgetTimer，widget 可以只带一个简单的 WidgetTimer 前置声明。（对大型系统而言非常重要）关于编译依存性的最小化，详见条款 31 。

案例二：一个使用 private 的极端案例

> 这种情况真是够激进的，只适用于你所处理的 class 不带任何数据时。这样的 `class 不存在任何成员函数或变量`。示例：

```cpp
class Empty {};
​
class DemoWithEmpty {
private:
    int x;
    FEmpty Empty;
};
​
inline void Try(){
    DemoWithEmpty DemoWithEmpty;
    Empty Empty;
    std::cout<<sizeof(Empty)<<"\n";  //1
    std::cout<<sizeof(DemoWithEmpty)<<"\n"; //8
}
```

> 可以看到，一个不含任何成员的 class 的大小居然为 1。因为 C++规定凡是独立对象都必须有非零大小。所以你可以发现 sizeof（Empty）的大小为 1，而且几乎所有的编译器都这样做。至于为什么含一个 int 大小的 class 是 8，这涉及到内存对齐的问题，不必详细讨论。  
> 或许你注意到了，独立对象才需要有非零大小，这意味着继承而来的 Empty class 大小可以不受约束:

```cpp
class FEmpty {};
​
class DemoWithEmpty :private FEmpty{
private:
    int x;
};
    
inline void Try(){
    DemoWithEmpty DemoWithEmpty;
    Empty Empty;
    std::cout<<sizeof(Empty)<<"\n";  //1
    std::cout<<sizeof(DemoWithEmpty)<<"\n"; //4
}
```

> DemoWithEmpty 所用大小正好等于一个 int 的大小，而这种表现就是所谓的 `EBO（empty base optimization）空白基类最优化`。值得注意的是，EBO 一般在单一继承下才可行。

尽管有这些例外情况，让我们回到根本。大部分 class 并非 empty，这很少成为你使用 private 继承的理由。只有当你面对需要访问 base class 的 protected 成员或者覆写 virtual 函数时，private 继承才被纳入考虑。当你审视完所有方案，仍然认为 private 继承是最佳方法，才使用它。

总结：

1. Private 继承意味 is-implemented-in-terms-of（根据某物实现出）。它通常比复合（composition）的级别低。但是当 derived class 需要访问 protected base class 的成员，或需要重新定义继承而来的 virtual 函数时，这么设计是合理的。
2. 和复合（composition）不同，private 继承可以造成 empty base 最优化。这对致力于对象尺寸最小化的程序库开发者而言，可能很重要。

### 条款 40 小心谨慎地使用多重继承

C++社区对多重继承（multiple inheritance MI）持有两类观点。

> 单一继承是好的，但多重继承不值得使用。  
> 单一继承（single inheritance SI）是好的，多重继承更好。

两种观点的比较与选择  
观点一：**多重继承不值得使用**

> 原因：**多重继承可能会引发歧义（ambiguity）行为。** **解决办法：指明调用**

```cpp
// 图书馆可借内容的基类。
class BorrowAbleItem
{
public:
    void checkOut(); // 检查函数
};

// 一个电子小工具类。
class ElectronicGadget
{
private:
    bool checkOut() const; // 检查函数
};

// Mp3播放器类。
class MP3Player :
public : BorrowAbleItem, 
public : ElectronicGadget 
{...};

MP3Player mp;
// 这里会引发歧义，mp对象到底调用的是哪个checkout函数？
mp.checkout();
```

> 疑问：B~class 的 checkOut 函数是 public 的，E~class 的 checkOut 函数是 private 的，理应只有 B~class 的函数是可以调用，那为什么会引发歧义行为？  
> 原因：这与**C++的解析机制**有关（与解析（resolving）重载函数调用的规则相符）。在看到是否有个函数可取用之前，C++会首先确认这个函数是不是此调用的最佳匹配，**找出最佳匹配函数后才检验其可取用性**。在该例中，两个 checkOuts 有相同的匹配程度（因此才造成歧义），没有所谓最佳匹配。因此**ElectronicGadget:: checkOut 的可取用性也就从未被编译器审查**(是不是 public 对该问题也就没有影响, 还没到这一步就错了)。

**解决方法**：如下调用即可

```cpp
// 指定调用BorrowAbleItem的checkOut函数。
mp.BorrowAbleItem::checkOut();
// 错误行为！！！通过最佳匹配检查后发现，这个函数是个private函数。
mp.ElectronicGadget::checkOut();
```

> **原因二：要命的“钻石型多重继承” 解决办法：virtual 继承**

```cpp
class File {...};
class InputFile: virtual public File {...};
class OutputFile: virtual public File {...};
class IOFile: public InputFile, public OutputFile {...};
```

这种继承体系必须面对的一个问题是：**是不是打算让 File class 内的成员变量经过每一条路径被复制？**

总结：

1. 多重继承比单一继承复杂。它可能导致新的歧义性，以及对 virtual 继承的需要。
2. Virtual 继承会增加大小，速度，初始化（赋值）复杂度等等成本。如果 virtual base classes 不带任何数据，将是最具有实用价值的情况。
3. 多重继承的确有正当用途。其中一个情节涉及 **“public 继承某个 Interface class”**  和 **"private 继承某个协助实现的 class"** 的组合。

## 模板与泛型编程

### 条款 41 了解隐式接口和编译器多态

**显式接口**和**运行期多态**

> 面向对象的世界总是以显式接口和运行期多态解决问题。

显式接口的构成**：** 函数名称，参数类型，返回类型，常量性也包括编译器产生的 copy 构造函数和 copy assignment 操作符。( 函数的签名式 )

```cpp
class Widget {
public:
    virtual ~Widget()=default;
    virtual void Normalize()=0;
};
class SpecialWidget:public Widget {
public:
    virtual void Normalize() override {
        std::cout<<"Special"<<"\n";
    }
};
class NormalWidget:public Widget {
public:
    virtual void Normalize() override {
        std::cout<<"Normal"<<"\n";
    }
};
inline void BeginPlayWithWidget(Widget& InWidget) {
    InWidget.Normalize();
}
inline void Try() {
    SpecialWidget SpecialWidget;
    NormalWidget NormalWidget;
    BeginPlayWithWidget(SpecialWidget); //Special
    BeginPlayWithWidget(NormalWidget); //Normal
}
```

可以这样说 BeingPlayWithWidget 函数中的 InWidget

- 由于 InWidget 类型被声明为 Widget，所以 InWidget 必须支持 Widget 接口。我们可以在源码中找到这个接口，看看它们是什么样子，所以我们称之为一个**显式接口**（explicit interface），也就是它在源码中的确可见。
- 由于 Widget 的 BeingPlayWithWidget (或者说某些成员函数)函数是 virtual ，InWidget 将对此函数的调用表现出**运行期多态**（runtime polymorphism），也就是说将**在运行期根据 InWidget 的动态类型决定究竟调用哪个函数。**

**隐式接口**和**编译期多态**

> Templates 及泛型编程的世界，与面向对象的世界有根本的不同。在此世界显式接口和运行期多态仍然存在，但重要性降低。

隐式接口的构成： 有效表达式（valid expression）

```cpp
template <typename T>
inline void BeginPlayWithWidgetTemplate(T& InT) {
    if (InT.Size() > 0 && InT != EClassType::None) {
        InT.Normalize();
        std::cout << InT.Size() << "\n";
        //...
    }
}
```

对 InT 来说

- InT 必须支持哪一种接口，是由函数体中对 InT 的操作决定的。从本例来看，InT 的类型 T 必须要支持 Normalize 、Size、不等比较等操作。看表面可能并非完全正确，但这组操作对于 T 类型的参数来说，是一定要支持的**隐式接口**（implicit interface）。
- 凡是涉及 InT 的任何函数调用，例如 operator> 和 operator != 有可能造成 template 的具现化，使得这些调用得以成功，这样的行为发生在编译器。以**不同的 template 参数具现化 function templates** 会导致调用不同的函数，这便是所谓的**编译期多态**（compile-time polymorphism）。

> 纵使你从未用过 templates，应该不陌生“运行期多态”和“编译期多态”之间的差异，因为它类似于“哪一个重载函数该被调用”（发生在编译期）和“哪一个 virtual 函数该被绑定”（发生在运行期）之间的差异。

隐式接口与显示接口不同，它不基于函数签名式，而是由**有效表达式（valid expression）** 构成

```cpp
template <typename T>
inline void BeginPlayWithWidgetTemplate(T& InT) {
    if (InT.Size() > 0 && InT != EClassType::None) {
        //...
    }
}
```

在该例子中， T 类型的隐式接口有这些约束：

> 1. T 必须提供名叫 Size 的成员函数，该函数返回一个整数值；  
> 2. T 必须支持 operator!= 函数，用来与两个 T 对象，这里假设 EClassType 为 T 类型；  
> 得益于操作符重载（operator overloading）带来的可能性，这两个约束都不需要满足。是的，T 必须支持 size 成员函数，然而这个函数也可能从 base class 继承而得。这个成员函数不需返回一个整数值，甚至不需返回一个数值类型。就此而言，它甚至不需要返回一个定义有 operator>的类型！它唯一需要做的是返回一个类型为 x 的对象，而 x 对象加上一个 int（10 的类型）必须能够调用一个 operator>。这个 operator>不需要非得取得一个类型为 x 的参数不可，因为它也可以取得类型 Y 的参数，只要存在一个隐式转换能够将类型 x 的对象转换为类型 y 的对象！  
> 同样道理，T 并不需要支持 operator!=，因为以下这样也是可以的：operator！=接受一个类型为 x 的对象和一个类型为 Y 的对象，T 可被转换为 x 而 EClassType 的类型可被转换为 Y，这样就可以有效调用 operator !=。

总结：

1. Classes 和 templates 都支持接口（interfaces）和多态（polymorphism）。
2. 对 classes 而言接口是显式的（explicit），**以函数签名为中心**。多态则是通过 virtual 函数发生于**运行期**。
3. 对 template 参数而言，接口是隐式的（implicit），**奠基于有效表达式**。多态则是通过 template 具现化和函数重载解析（function overloading resolution）发生于**编译期**。

### 条款 42 了解 typename 的双重含义

本条款首先提出一个问题：以下 template 声明式中，class 和 typename 有什么不同

```cpp
template<class T>
class Widget;

template<typename T>
class Widget;
```

答案：没有不同。

> **当我们声明 template 类型参数时， class 和 typename 的意义完全相同。**  
> 某些程序员喜欢 class，因为可以少打几个字, 有些人比较喜欢 typename ，因为它**暗示参数并非一定是个 class 类型**。

**然而 C++并不总是把 class 和 typename 视为等价**。有时候你一定得使用 typename。为了解其时机，我们必须先谈谈你可以在 template 内指涉（refer to）的两种名称： (嵌套)从属名称和非从属名称。

```cpp
template<typename T>
void PrintContainer(const T& Container)
{                                           //注意这不是有效的C++代码
    if (Container.Size()>0) {
        T::const_iterator iter(Container.begin()); //取得第一元素的迭代器  注意 iter
        ++iter;                         
        int value=*iter;                    //将该元素复制到某个int，注意 value
        std::cout<<value<<"\n";
    }
}
```

> 在上述代码中强调两个 local 变量：**iter** 和 **value**。  
> iter 的类型是 `T::const_iterator`，实际是什么取决于 template 参数 T。Template 内出现的名称如果依赖于某个 template 参数，我们就称之为**从属名称**（dependent names）。如果从属名称在 class 内呈嵌套状，我们就称之为**嵌套从属名称**（nested dependent names）。T:: const_iterator 就是这样的名称，实际上它还是一个**嵌套从属类型名称**（nested dependent type name），也就是个**嵌套从属名称**并且**指涉是什么类型**。  
> Value 的类型是 int 。它不依赖于任何 template 参数。我们便称之为非从属名称（non-dependent names）。

嵌套从属名称可能导致解析（parsing）困难

> 在我们知道 T 是什么之前，没有任何办法可以知道 T:: const_iterator 是否为一个类型。而当编译器开始解析 template PrintContainer 时，尚未确知 T 是什么东西。 C++有个规则可以解析（resolve）此一歧义状态：**如果解析器在 template 中遭遇一个嵌套从属名称，它便假设这名称不是个类型，除非你告诉它是**。所以缺省情况下嵌套从属名称不是类型。此规则有个例外，稍后我会提到。

再次回顾上述代码:

```cpp
template<typename T>
void PrintContainer(const T& Container)
{                                           
    if (Container.Size()>0) {
        T::const_iterator iter(Container.begin());  //这个名称被假设为非类型
        ......
    }
}
```

> 现在应该很清楚为什么这不是有效的 C++代码了吧。Iter 声明式只有在 T:: const_iterator 是个类型时才合理，但我们并没有告诉 C++说它是，于是 C++假设它不是。若要矫正这个形势，我们必须告诉 C++说 T:: const iterator 是个类型。**只要紧临它之前放置关键字 typename 即可：**

```cpp
template<typename T>
void PrintContainer(const T& Container)      //这是合法的C++代码
{                                           
    if (Container.Size()>0) {
        typename T::const_iterator iter(Container.begin());  //ok
        ......
    }
}
```

一般性规则很简单：**任何时候当你想要在 template 中指涉一个嵌套从属类型名称，就必须在紧临它的前一个位置放上关键字 typename。**（再提醒一次，很快我会谈到一个例外。）

**typename 只被用来验明嵌套从属类型名称**；其他名称不该有它存在。例如下面这个 function template，接受一个容器和一个“指向该容器”的选代器：

```cpp
template<typename C>                 //允许使用"typename"（或"class"）
void f(const C&container，           //不允许使用"typename"
typename C::iterator iter)；        //一定要使用"typename"
```

> 上述的 C 并不是嵌套从属类型名称（它并非嵌套于任何“取决于 template 参数”的东西内），所以声明 container 时并不需要以 typename 为前导，但 C:: iterator 是个嵌套从属类型名称，所以必须以 typename 为前导。

“Typename 必须作为嵌套从属类型名称的前缀词”这一规则的例外是：**typename 不可以出现在 base classes list 内的嵌套从属类型名称之前，也不可在 member initialization list（成员初值列）中作为 base class 修饰符。**

```cpp
class Message {
public:
    Message() = default;
    explicit Message(std::string InText): Text(std::move(InText)) {}

    void SetText(std::string InText) {
        Text = std::move(InText);
    }

    std::string GetText() {
        return Text;
    }

private:
    std::string Text;
};

template <typename T>
class MessageContainer {
public:
    typedef T ElementType;
};

template <typename T>
class Printer : private MessageContainer<T>::ElementType {//base classes list不使用typename
public:
    //使用 typename 表明是一个类型，而不是变量
    //使用 typedef 给过长的类型起别名，方便。
    typedef typename MessageContainer<T>::ElementType ElementType; 

    explicit Printer(): MessageContainer<T>::ElementType() { //mem.init.list中不使用typename

        std::cout << typeid(ElementType).name() << "\n"; //class Message

    }

    void Log(std::string Text) {
        ElementType::SetText(Text);
        std::cout << ElementType::GetText() << "\n";
    }
};

inline void TryWithPrinter() {
    Printer<Message> Printer;
    Printer.Log("HelloWorld");
}
```

总结：

1. 声明 template 参数时，前缀关键字 class 和 typename 可互换。
2. 请使用关键字 typename 标识**嵌套从属类型名称**；但 **不得在 base class lists（基类列）** 或 **member initialization lists（成员初值列）** 内以它作为 base class 修饰符。

### 条款 43 学习处理模板化基类内的名称

从一个例子入手，假设我们要设计游戏中人物的相关列表，比如 buff 列表，物品列表等等，一个显而易见的设计是：

```cpp
#include <iostream>
#include <set>

class Buff {
public:
    virtual ~Buff() = default;
    virtual void Start() = 0;
    virtual void End() = 0;
    virtual void OnTick() = 0;
};

class RedBuff : public Buff {
public:
    virtual void Start() override {}
    virtual void End() override {}
    virtual void OnTick() override {}
};

class BlueBuff : public Buff {
public:
    virtual void Start() override {}
    virtual void End() override {}
    virtual void OnTick() override {}
};

template <typename T>
class Container {
public:
    void Add(T Item) {
        std::cout << "Add" << "\n";
        Data.insert(Item);
    }

    void Clear() {
        std::cout << "Clear" << "\n";
        Data.clear();
    }

    size_t Size() { return Data.size(); }

private:
    std::set<T> Data;
};

template <typename T>
class PlayerContainer : public Container<T> {
public:
    void RemoveAll() {
        Clear();  // 在此无法访问 Clear 函数,找不到标识符
    }

    void ShowAll() {
        //...
    }
};

int main() {
    PlayerContainer<Buff*> PlayerBuffContainer;
    PlayerBuffContainer.Clear();

    RedBuff* RedBuffOne = new RedBuff;
    BlueBuff* BlueBuffOne = new BlueBuff;
    PlayerBuffContainer.Add(RedBuffOne);
    PlayerBuffContainer.Add(BlueBuffOne);

    std::cout << PlayerBuffContainer.Size() << "\n";
}
```

运行此代码，出现错误

```cpp
Clear:找不到标识符!
```

而出错的原因在于：

> 当编辑器遭遇 class template PlayerContainer 时，其实并不知道它究竟继承哪个 class。当然它继承的是 Container ，但其中的 T 是一个 template 参数，**不到后来的具现化，是无法确切知道它是什么**。而如果不知道 T 是什么，就不清楚 class Container 看起来像什么——更确切地说是没办法知道它是否有个 Clear 函数。  
> 例如，如果有以下**特化版** class Container （模板全特化）

```cpp
template<>                 //一个全特化的Container
class Container<Buff*> {   //它和一般的template相同，区别只在于它删掉了void Clear() 函数
public:
 void Add(T Item) {
     std::cout << "Add" << "\n";
     Data.Insert (Item);
 }
 
 size_t Size() {
     return Data.size();
 }
};

//等价于
template<typename T=Buff*>
class Container<T> {
public:
  .....
};
```

> 现在，再让我们考虑 derived class PlayerContainer:

```cpp
template <typename T>
class PlayerContainer : public Container<T> {
public:
 void RemoveAll() {
     Clear();   //如果T==Buff,这个函数不存在
 }

 void ShowAll() {
     //...
 }
};
```

> 正如注释所言，**当 base class 被指定为 Container 时，这段代码将不合法！**因为该版本的 Container template 类**被特化**，其中并**不存在 Clear 函数**，且由于编译器会**优先考虑特化版本**，意味着 Container 使用 Buff 具现化时类中只存在 Add、Size 函数，并未提供 Clear 函数。  
> 这正是前面所说，为什么 C++ 拒绝在 PlayerContainer 访问 Clear 函数的原因：它知道 base classes templates 有可能被特化，而那个**特化版本可能不提供和一般性 template 相同的接口。因此它往往拒绝在 base classes templates 寻找继承而来的名称**。因此它往往拒绝在 templatized base classes（模板化基类，本例的 Container）内寻找继承而来的名称（本例的 Clear）。

所以，**我们必须有某种办法令 C++不进入 templatized base classes 观察的行为失效**。幸运的是，我们有三个解决办法：

> 1. 在 base class template 函数调用动作之前**加上 this->**。This 指针可以访问所有成员函数。

```cpp
template <typename T>
class PlayerContainer : public Container<T> {
public:
    void RemoveAll() {
      this -> Clear();   //成立，假设Clear将被继承
    }
 ...
};
```

> **2. 使用 using 声明式**。可以告诉编译器进入 base class 作用域寻找函数。

```cpp
template <typename T>
class PlayerContainer : public Container<T> {
public:
    using Container<T>::Clear; //告诉编译器，请他假设Clear位于base class内
    void RemoveAll() {
       Clear();   
    }
 ...
};
```

（虽然 using 声明式在这里或在条款 33 都可有效运作，但两处解决的问题其实不相同。这里的情况并不是 base class 名称被 derived class 名称遮掩，而是编译器不进入 base class 作用域内查找，于是我们通过 using 告诉它，请它那么做。）  

> **3. 明确指出被调用函数位于 base class 内。 (不推荐)**

```cpp
template <typename T>
class PlayerContainer : public Container<T> {
public:
    void RemoveAll() {
      Container<T>::Clear();   //成立，假设Clear将被继承
    }
 ...
};
```

但这往往是最不让人满意的一个解法，因为**如果被调用的是 virtual 函数**，上述的明确资格修饰（explicit qualification）会**关闭"virtual 绑定行为”**  
从名称可视点的角度来看，上述每一个解法做的事情都相同：**对编译器承诺 base class template 的任何特化版本都支持其泛化版本所提供的接口。如果承诺未被保证，编译器仍然会报错。**

总结：

1. **可在 derived class templates 内通过 this-> 指涉 base class templates 内的成员名称，或藉由一个明白写出的 base class 资格修饰符完成。**

### 条款 44 将与参数无关的代码抽离 templates

- **templates 是节省时间和避免代码重复的奇方妙法。**

> 你不再需要键入 20 个类似的 classes 并且每一个都带有 20 个成员函数，你只需要键入一个 class template，留给编译器去具现化那 20 个你需要的相关 classes 即可，而且对于 20 个函数中未被调用的，编译器不会自动生成。这样的技术是不是很伟大，呵呵。

- 但这也很容易使得**代码膨胀**（code bloat），templates 产出码带着重复，或者几乎重复的代码，数据，或者两者。你可以通过：**共性与变形分析**（commonality and variability analysis）来避免代码膨胀。

> 这个概念其实你早在使用，即使你从未写过一个 templates。当你编写某个函数时，你明白其中某些部分的实现码和另一个函数的实现码实质相同，你会很单纯的重复它们吗？当然不，你会抽出这两个函数相同的部分，放进第三个函数中，然后令原先两个函数调用这个新函数。也就是说：你分析了两个函数的共性和变形，把公共的部分搬到一个新的函数中去，变化的部分保留在原来的函数不动。对于 class 也是这个道理，如果你明白某些 class 和另一个 class 具有相同的部分，你也会把共性搬到一个新的 class。

- Templates 的优化思路也是如此，以相同的方式避免重复，但其中有个窍门。在 non-template 代码中，重复很明确。然而**在 template 代码中，重复是隐晦的，** 毕竟只存在一份 template 代码，所以你必须自己去感受 template 具现化时可能发生的重复。

造成代码膨胀的一个典型的例子： **template class 成员依赖 template 参数值**

```cpp
// 典型例子
template <typename T, std::size_t n>
class SquareMatrix {
  public:
    void invert();
};

SquareMatrix<double, 5> m1;
SquareMatrix<double, 10> m2;
```

> 会具现两份非常相似的代码，因为除了一个参数 5，一个参数 10，其他都完全一样

**改进** ：使用带参数的函数

```cpp
template <typename T>
class SquareMatrixBase {
  protected:            // protected 保证只有本类/子类本身可以调用
    void invert(std::size_t n);
}

template<typename T, std::size_t n>
class SquareMatrix  : private SquareMatrixBase<T> { // private继承，derived 和base不是is-a关系，base只是帮助实现derived 
  private:
    using SquareMatrixBase<T>::invert;  // derived class 会掩盖template base class的函数

  public:
    inline void invert() { this->invert(n);}
}
```

> 如上，SquareMatrixBase 只对“矩阵元素对象的类型”参数化，不对矩阵的尺寸参数化。因此对于某给定元素类型，所有矩阵共享同一个 SquareMatrixBase 类。  
> SquareMatrixBase:: invert 只是企图成为“避免派生类代码重复”的一种方法，所以它**用 protected 替换 public**。调用它而造成的额外成本应该是 0 (因此派生类的 invert 调用基类版本的 invert 时是 inline 调用)。这里函数使用 `this->`，否则**模板化基类的函数名称会被派生类掩盖**。注意这里是**private 继承**，说明了这里的基类只是为了帮助派生类的实现，不是为了表现 SquareMatrixBase 和 SquareMatrix 的 is-a 关系。

目前为止一切都好，但还有一些问题没有解决：

> SquareMatrixBase:: invert 如何知道该操作什么数据？  
> 虽然它从参数中知道矩阵尺寸，但它如何知道哪个矩阵的数据在哪儿？想必只有派生类知道。  
> 派生类如何联络其基类做逆运算动作？

解决办法： **令 SquareMatrixBase 存储一个指针，指向矩阵数值所在的内存**

```cpp
template <class T>
class SquareMatrixBase {
protected:
    SquareMatrixBase(std::size_t n, T* pMem) : size(n), pData(pMem) {}
    void SetDataPtr(T* ptr) { pData = ptr; }
    ...
private:
    std::size_t size;
    T* pData;
}

template <class T, size_t n>
class SquareMatrix : private SquareMatrixBase<T> {
public:
    SquareMatrix() : SquareMatrixBase<T>(n, data) {}
    ...
private:
    T data[n*n];
}
```

这类类型的对象不需要动态分配内存，但对象自身可能非常大。另一种做法是把每一个矩阵的数据放进 heap (也就是通过 new 来分配内存)

```cpp
template<typename T, std::size_t n>
class SquareMatrix : private SquareMatrixBase<T>{
public:
    SquareMatrix () :  
         SquareMatrixBase<T>(n, 0),             //将基类的数据指针设为null
         pData(new T[n * n])   // 为矩阵内容分配内存, 将指向该内存的指出存储起来
     {this->setDataPtr(pData.get();)}

private:
    std::unique_ptr<T[]> pData;
}
```

这个条款只讨论由 non-type template parameters (非类型模板参数)带来的膨胀，其实 type parameters (类型参数)也会导致膨胀。

> 1. 比如在很多平台上，int 和 long 有相同的二进制表述，所以 vector< int>和 vector< long>的成员函数可能完全相同。  
> 2. 同样的，大多数平台上，所有指针类型都有相同的二进制表述，因此凡模板持有指针者 (比如 list< int>、list< const int >等)往往应该对每一个成员使用唯一一份底层实现。  
> 3. 也就是说，如果你实现成员函数而它们操作强类型指针（T），你应该令它们调用另一个无类型指针 (void )的函数，由后者完成实际工作。

总结:

1. Templates 生成多个 classes 和多个 functions，所以任何 template 代码都不该与某个造成膨胀的 template 参数产生相依关系。
2. 因非类型模板参数（non-type template parameters）而造成的代码膨胀，往往可以消除，做法是以函数参数或 class 成员变量替换 template 参数。
3. 因类型参数（type parameters）而造成的代码膨胀，往往可以降低，做法是让带有完全相同二进制表述（binary representations）的具现类型（instantiation types）实现代码共享。

### 条款 45 运用成员函数模板接受所有兼容类型
