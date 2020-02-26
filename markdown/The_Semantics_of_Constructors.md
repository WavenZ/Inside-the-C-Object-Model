# 第 2 章 构造函数语意学（The Semantics of Constructors）

译注：本章大量出现以下英文术语

> implicit  : 暗中的、隐含的（通常意指并非在程序源代码中出现的）<br>
> explicit   : 明确的（通常意指程序源代码中所出线的）<br>
> trivial    : 没有用的<br>
> nontrivial : 有用的<br>
> memberwise : 对每一个 member 施以······<br>
> bitwise    : 对每一个 bit 施以······<br>
> semantics  : 语意<br>

关于 C++，最常听到的一个抱怨就是，编译器背着程序员做了太多事情。Conversion 运算符就是最常被引用的一个例子。Jerry Schwarz, iostream 函数库的建筑师，就曾经说过一个故事。他说他最早的意图是支持一个 iostream class object 的纯量测试（scalar test），像这样：
```cpp
    if ( cin ) ...
```
为了让 cin 能够求得一个真假值，Jerry 首先为它定义一个 conversion 运算符：operator()。在良好行为如上者，这的确可以正确运行，但是在下面这种情况的程序设计中，它的行为就势必令人大吃一惊了：
```cpp
    // 喔欧：应该是cout，而不是cin
    cin << intVal;
```
这个粗心的程序员要的应该是 cout 而不是 cin。Class 层次结构的 “type-safe” 天性应该能够捕捉这类输出运算符的错误运用。然而，带有几分唯物主义色彩的编译器，比较喜欢找到一个正确的诠释（如果有的话），而不是把程序标示为错误就算了！在此例中，内建的左移位运算符（left shift opertor，<<）只有在“cin 可改变为和一个整数值同义”时才适用。编译器会检查可以使用的各个 conversion 运算符，然后它找到了 opertor int()，那正是它要的东西。左移位运算符现在可以操作了：如果没有成功，至少也是合法的：
```cpp
    // 喔欧：不完全是程序员想要的
    int temp = cin.operator int();
    tem << intVAl;
```
Jerry 如何解决这个意想不到的行为？他以 opertor void*() 取代operator int()。这种错误有时候被称为 “Schwarz Error”。虽然这种错误只能算是一种“不足”，但缺乏一个 implicit class conversion 机制却实在是强烈的败笔。String class 的最初实例（引述于 [STROUP94] p.83）可以说是一种推动力量：如果没有 implicit conversion 的支持，String 函数库必须将每一个拥有字符串参数的 C runtime library 函数都复制一份[1]。

[1] 有趣的是，标准的 C++ library string class 并不提供一个implicit conversion 运算子：它提供的是一个具名实体（named instance），使用者必须明确调用才行。

在不少程序员之间，存在着一种忐忑不安的情绪，认为一个编译器暗中施行的 “user-defined conversion运算符” 可能不会导致 Schwarz Error。事实上关键词 explicit 之所以被导入这个语言，就是为了提供给程序员一种方法，使他们能够制止“单一参数的 constructor” 被当做一个 conversion 运算符。虽然他们很容易从 Schwarz Error 的故事中获得安慰，但是 conversion 运算符实际上很难在一种可预期的良好行为模式下使用，Conversion 运算符的引用应该是明智的，而其测试应该是严酷的，并且在程序已出现不寻常活动的第一个症状时就发出疑问。

问题在于编译器太过于按照字面意义来解释你的意图，而没有在背后为你多做点什么事情——虽然简单，要让程序员相信他们被 Schwarz Error 盯上是颇为困难的。“背后活动”比较可能发生在 “memberwise initialization” 或是所谓的 “named return value optimization”（NRV）身上，在这一章中，我要挖掘编译器对于 “对象构造过程” 的干涉，以及对于 “程序形式” 和 “程序效率” 上的冲击。

## 2.1 Default Constructor的建构操作

C++ Annotated Reference Manual（ARM）[ELLIS90] 中的 Section 12.1 告诉我们：“default construtors...在需要的时候被编译器产生出来”。关键字眼是“在需要的时候”。被谁需要？做什么事情？看看下面这段程序代码：
```cpp
class Foo {public : int val; Foo *pnext; };
void foo_bar()
{
    // 喔欧：程序要求bar's members都被清为0
    Foo bar;
    if (bar.val || bar.pnext)
        // ... do something
    // ...
}
```
译注：以下文字的原文多次使用 implementation 这个字眼，许多时候它是指 “C++ 实现器”，也就是指 C++ 编译器，这种情况下我会将它直接译为编译器。

在这个例子中，正确的程序语意是要求 Foo 有一个 default constructor，可以将它的两个 members 初始化为 0. 上面这段码可曾符合 ARM 所说的“在需要的时候”？答案是 no。其间的差别在于一个是程序员的需要，一个是编译器的需要。程序如果有需要，那是程序员的责任；本例要承担责任的是设计 class Foo 的人[2]。是的上述程序片段并不会合成出一个 default constructor。

[2]Global objects 的内存会在程序激活的时候被清为0. local objects 配置于程序的堆栈中，heap objects 配置于自由空间中，都不一定会被清为0，它们的内容将是内存上次被使用后的遗迹。

那么，什么时候才会合成出一个 default constructor 呢？当编译器需要它的时候！此外，被合成出来的 constructor 只执行编译器所需的行动，也就是说，即使有需要为 class Foo 合成一个 default constructor，那个 constructor 也不会将两个 data members val 和 pnext 初始化为 0. 为了让上一段码正确执行，class Foo 的设计者必须提供一个明显的 default constructor，将两个 members 适当地初始化。

C++ Standard 已经修改了 ARM 中的说法，虽然行为事实上仍然是相同的。C++ Standard [ISO-C++95] 的 Sectin 12.1 这么说：

> 对于 class X，如果没有任何 user-declared constructor，那么会有一个 default construtor 被暗中（implicitly）声明出来······一个被暗中声明出来的 default constructor 将是一个trivial（浅薄而无能，没啥用的）constructor······

C++ Standard 然后开始一一叙述在什么样的情况下这个 implicit default constructor 会被视为 trivial。一个 nontrivial default constructor 在 ARM 的术语中就是编译器所需要的的那种，必要的话会有编译器合成出来。下面的四个小节分别探讨 nontrivial default constructor 的四种情况：

### “带有Default Constructor”的Member Class Object

如果一个 class 没有任何 constructor，但它内含一个 member object，而后者有 default constructor，那么这个 class 的 implicit default constructor 就是 “nontrivial”，编译器需要为此 class 合成一个 default constructor。不过这个合成操作只有在 constructor 真正需要被调用时才会发生。

于是出现了一个有趣的问题，在 c++ 各个不同的编译模块中（译注：原文为 compilation model，是否为 complation module 之误？不同的编译模块意指不同的档案），编译器如何避免合成出多个 default constructor（譬如说一个是为 A.C 档合成，另一个是为 B.C 档合成）呢？解决方法是把合成的 default constructor、copy constructor、destructor、assignment copy operator 都以 inline 方式完成。一个 inline 函数有静态链接（static linkage），不会被档案以外者看到。如果函数太复杂，不适合做成 inline，就会合成出一个 explicit non-inline static 实体（inline 函数将在 4.5 节有比较详细的说明）。

举个例子，在下面的程序片段中，编译器为 class Bar 合成一个 default constructor：
```cpp
class Foo { public : Foo(), Foo(int) ...};
class Bar { public: Foo foo; char *str; }; // 译注：不是继承，是内含！

void foo_bar()
{
    Bar bar;    // Bar:foo必须在此处初始化
                // 译注：Bar::foo是一个member object，而其class Foo
                //       拥有default constructor，符合本小节主题
    if ( str ) { } ...
}
```
被合成的 Bar default constructor 内含必要的代码，能够调用 class Foo 的 default constructor 来处理 member object Bar::foo，但它并不产生任何码来初始化 Bar::bar。是的，将 Bar::foo 初始化是编译器的责任，将 Bar::str 初始化则是程序员的责任，被合成的 default constructor 看起来可能像这样[3]：

[3]为了简化我们的讨论，这些例子都省略掉隐含的this指针

```cpp
// Bar的default constructor可能会被这样合成
// 被member foo调用class Foo的default constructor
inline
Bar::Bar()
{
    // C++伪码
    foo.Foo::Foo();
}
```
再一次请你注意，被合成的 default constructor 只满足编译器的需要，而不是程序的需要。为了让这个程序片段能够正确执行，字符指针 str 也需要被初始化。让我们假设程序员经由下面的 default constructor 提供了str的初始化操作：
```cpp
// 程序员定义的default constructor
Bar::Bar() { str = 0; } 
```
现在程序的需求获得满足了，但是编译器还需要初始化 member object foo。由于 default constructor 已经被明确地定义出来，编译器没办法合成第二个。“奥，伤脑筋”，你可能会这样说，编译器会采取什么行动呢？

编译器的行动是：“如果 class A 内含一个或一个以上的 member class objects，那么 class A 的每一个 constructor 必须调用每一个 member classes 的 default constructor ”。编译器会扩张已存在的 constructors，在其中安插一些码，使得 user code 在执行之前，先调用必要的 default constructor。延续前一个例子，扩张后的 constructor 可能像这样：
```cpp
// 扩张后的default constructor
// C++伪码
Bar:Bar()
{
    foo.Foo::Foo(); // 附加上的compiler code
    str = 0;        // explicit user code
}
```

如果有多个 class member objects 都要求 constructor 初始化操作，将如何呢？C++ 语言要求以 “member objects 在 class 中的声明次序”来调用各个 constructor。这一点由编译器完成，它为每一个 constructor 安插程序代码，以“member声明次序”调用每一个 member 所关联的 default constructors。这些码将被安插在 explicit user code 之前。举个例子，假设我们有以下三个 classes：
```cpp
class Dopey { public: Dopey(); ... };
class Sneezy { public: Snezzy(int); Sneezy(); ... };
class Bashful { public: Bashful(); ....}
```
以及一个class Snow_White:
```cpp
class Snow_White{
public:
    Dopey dopey;    // 译注：dopey、sneezy和bashful是三个member objects
    Sneezy sneezy;
    Bashful bashful;
    // ...
private:
i   int mumble;
};
```
如果 Snow_White 没有定义 default constructor，就会有一个 nontrivial constructor 被合成出来，依序调用 Doepy、Sneezy、Bashful 的 default constructor。然而如果 Snow_White 定义了下面这样的 default constructor：
```cpp
// 程序员所写的default constructor
Snow_White:Snow_White() : sneezy(1024)
{
    mumble = 2048;
}
```
它会被扩张为：
```cpp
// 编译器扩张后的default constructor
// C++伪码
Snow_White:Snow_White() : sneezy(1024)
{
    // 插入member class object
    // 调用其constructor
    dopey.Dopey::Doepy();
    sneezy.Sneezy::Sneezy(1024);
    bashful.Bashful::Bashful();

    // explicit user code
    mumble = 2048;
}
```
2.4 节将讨论“调用 implicit default constructors”和“调用明确条列于 member initialization list 中的 constructors”之间的互动关系。

### “带有 Default Constructor” 的 Base Class

类似的道理，如果一个没有任何 constructors 的 class 派生自一个“带有 default constructor”的 base class，那么这个 derived class 的 default constructor 会被视为 nontrivial，并因此需要被合成出来。它将调用上一层 base classes 的 default constructor (根据它们的声明次序)。对一个后继派生的 class 而言，这个合成的 constructor 和一个“被明确提供的 default constructor”没有什么差异。

如果设计者提供多个 constructors，但其中都没有 default constructor 呢？编译器会扩张现有的每一个 constructors，将“用以调用所有必要之 default constructors”的程序代码加进去。它不会合成一个新的 default constructor，这是因为其他“有 user 所提供的 constructors”存在的缘故。如果同时亦存在着“带有 default constructors”的 member class objects，那些 default constructor 也会被调用——在所有的 base class constructor 都被调用之后。

### “带有一个 Virtual Function” 的 Class

另有两种情况，也需要合成出 default constructor：

1. class 声明（或继承）一个 virtual function.
2. class 派生自一个继承串链，其中有一个或更多的 virtual base classes.

不管哪一种情况，由于缺乏由 user 声明的 constructor，编译器会详细记录合成一个 default constructor 的必要信息，以下面这个程序片段为例：
```cpp
class Widget{
public:
    virtual void flip() = 0;
    // ...
};

void flip(const Widget& widget) { widget.flip(); }
// 假设Bell和Whistle都派生自Widget
void foo()
{
    Bell b;
    Whistle w;

    flip(b);
    flip(w);
}
```

下面两个扩张操作会在编译其间发生：

1. 一个 virtual function table (在 cfront 中被称为 vtbl)会被编译器产生出来，内放 class 的 virtual functions 地址。
2. 在每一个 class object 中，一个额外的 pointer member（也就是 vptr）会被编译器合成出来，内含相关的 class vtbl 的地址。

此外，widget.flip() 的虚拟引发操作（virtual invocation）会被重新改写，以便用 widget 的 vptr 和 vtbl 中的 flip() 条目：
```cpp
// widget.flip()的虚拟引发操作（virtual invocation）的转变
(*widget.vptr[1])(&widget))
```
其中：
1. [1] 表示 flip() 在 virtual table 中的固定索引；
2. &widget 代表要交给“被调用的某个 flip() 函数实体”的 this 指针。

为了让这个机制发挥功效，编译器必须为每一个 Widget（或其派生类之）object 的 vptr 设定初值，放置适当的 virtual table 地址。对于 class 所定义的每一个 constructor，编译器会安插一些码来做这样的事情（请看 5.2 节）。对于那些未声明任何 constructors 的 classes，编译器会为它们合成一个 default constructor，以便正确地初始化每一个 class object 的 vptr。

### “带有一个 Virtual Base Class” 的 Class

Virtual base class 的实现法在不同的编译器之间有极大的差异。然而，每一种实现法的共通点在于必须使 virtual base class 在其每一个 derived class object 中的位置，能够于执行期准备妥当。例如下面这段程序代码中：
```cpp
class X { public: int i; };
class A : public virtual X { public: int j; };
class B : public virtual X { public: double d; };
class C : public A, public B { public :int k; };

// 无法在编译时期决定（resolve）出pa->X::i的位置
void foo(const A* pa) { pa->i = 1024; }

main()
{
    foo(new A);
    foo(new C);
    // ...
}
```
编译器无法固定住 foo() 之中“经由 pa 而存取的 X::i ”的实际偏移位置，因为 pa 的真正类型可以改变。编译器必须改变“执行存取操作”的那些码，使 X::i 可以延迟至执行期才决定下来。原先 cfront 的做法是靠“在 derived class object 的每一个 virtual base classes 中安插一个指针”完成。所有“经由 refernce 或 pointer 来存取一个 virtual base class” 的操作都可以通过相关指针完成。在我的例子中，foo() 可以被改写如下，以符合这样的实现策略：
```cpp
// 可能的编译器转变操作
void foo(const A* pa) { pa->__vbcX->i = 1024; }
```
其中，__vbcX 表示编译器所产生的指针，指向virtual base class X.

正如你所臆测的那样，__vbcX（或编译器所作出的某个什么东西）是在 class object 建构期间被完成的。对于 class 所定义的每一个 constructor，编译器会安插那些“允许每一个 virtual base class 的执行期存取操作”的码。如果 class 没有声明任何 constructors，编译器必须为它合成一个 default constructor.

### 总结：

有四种情况，会导致“编译器必须为未声明 constructor 之 classes 合成一个 default constructor”。C+++ Standard 把那些合成物称为 implicit nontrivial default constructor。被合成出来的 constructor 只能满足编译器（而非程序）的需要。它之所以能够完成任务，是借着“调用 member object 或 base class 的 default constructor” 或是“为每一个 object 初始化其 virtual function 机制或 virtual base class 机制”而完成。至于没有存在那四种情况而又没有声明任何 constructor 的classes，我们说它们拥有的是 implicit trivial default constructors，它们实际上并不会被合成出来。

在合成的 default constructor 中，只有 base class subobjects 和 member class objects 会被初始化。所有其它的 nonstatic data member，如整数、整数指针、整数数组等等都不会被初始化。这些初始化操作对程序而言或许有需要，但对编译器则并非必要。如果程序需要一个“把某指针设为 0” 的 default constructor，那么提供它的人应该是程序员。

C++新手一般有两个常见的误解：
1. 任何 class 如果没有定义 default constructor，就会被合成一个来。
2. 编译器合成出来的 default constructor 会明确设定“class 内每一个 data member 的默认值”

如你所见，没有一个是真的！

## 2.2 Copy Constructor的建构操作

有三种情况，会以一个 object 的内容作为另一个 class object 的初值。最明显的一种情况当然就是对一个 object 做明确的初始化操作，像这样：
```cpp
class X { ... };
X x;

// 明确地以一个object的内容作为另一个class object的初值
X xx = x;
```
另两种情况是当 object 被当做参数交给某个函数时，例如：
```cpp
extern void foo(X x);

void bar()
{
    X xx;

    // 以xx作为foo()第一个参数的初值（不明显的初始化操作）
    foo(xx);

    // ...
}
```
以及当函数传回一个 class object 时，例如：
```cpp
X
foo_bar()
{
    X xx;
    // ...
    return xx;
}
```
假设 class 设计者明确定义了一个 copy constructor（这是一个 constructor，有一个参数的类型是其 class type），像下面这样：
```cpp
// user-defined copy constructor的实例
// 可以是多参数形式，其第二个参数及后继参数以一个默认值供应之

X::X(const X& x);
Y::Y(const Y& y, int = 0);
```
那么在大部分情况下，当一个 class object 以另一个同类实体作为初值时，上述的 constructor 会被调用。这可能会导致一个暂时性的 class object 的产生或程序代码的蜕变（或两者都有）。

### Default Memberwise Initialization

如果 class 没有提供一个 explicit copy constructor 又当如何？当 class object 以“相同 class 的另一个 object”作为初值时期内部是以所谓的 default memberwise initialization 手法完成的，也就是把每一个内建的或派生的 data member（例如一个指针或一数目组）的值，从某个 object 拷贝一份到另一个 object 身上。不过它并不会拷贝其中的 member class object，而是以递归的方式实行 memberwise initialization 。例如，考虑下面这个 class 声明：
```cpp
class String{
public:
    // ... 没有explicit copy constructor
private:
    char *str;
    int len;
};
```
一个 String object 的 default memberwise initialization 发生在这种情况之下：
```cpp
String noun("book");
String verb = noun;
```
其完成方式就好像个别设定每一个 members 一样：
```cpp
// 语意相等
verb.str = noun.str;
verb.len = noun.len;
```
如果一个 String object 被声明为另一个 class 的 member，像这样：
```cpp
class Word{
public:
    // ... 没有explicit copy constructor
private:
    int _occurs;
    String _word;   // 译注：String object成为class Word的一个member!
};
```
那么一个 Word object 的 default memberwise initialization 会拷贝其内建的 member _occurs，然后再于 String member object _word 身上递归实施 memberwise initialization。

这样的操作实际上如何完成？ARM 告诉我们：

> 从概念上而言，对于一个class X，这个操作是被一个 copy constructor 实现出来······

其中主要的字眼是“概念上”。这个注释又紧跟着一些解释：

> 一个良好的编译器可以为大部分 class objects 产生 bitwise copies，因为它们有 bitwise copy semantics······

也就是说，“如果一个 class 未定义出 copy constructor，编译器就自动为它产生出一个”这句话不对，而是应该像 ARM 所说：

> Default constructors 和copy constructors 在必要的时候才由编译器产生出来。

这个句子中的“必要”意指当 class 不展现 bitwise copy semantics 时。C++ Standard 仍然保留了 ARM 的意义，但是将相关讨论更形式化如下（括号内是我的批注）：

> 一个 class object 可以从两种方式复制得到，一种是被初始化（也就是我们这里所关心的），另一种是被指定（assignment，第5章讨论之）。从概念上而言，这两个操作分别是以 copy constructor 和 copy assignment operator 完成的。

就像 default constructor 一样，C++ Standard 上说，如果 class 没有声明一个 copy constructor，就会有隐含的声明（implicitly declared）或隐含的定义（implicitly defined）出现。和以前一样，C++ Standard 把 copy constructor 区分为 trivial 和 nontrivial 两种。只有 nontrivial 的实体才会被合成于程序之中。决定一个 copy constructor 是否为 trivial 的标准在于 class 是否展现出所谓的“bitwise copy semantics”。下一节我将说明“class 展现出 bitsise copy semantics” 这句话是什么意思。

### Bitwise Copy Semantics（位逐次拷贝）

在下面的程序片段中：
```cpp
#include "Word.h"

Word noun("book");

void foo()
{
    Word verb = noun;
    // ...
}
```
很明显 verb 是根据 noun 来初始化。但是在尚未看过 class Word 的声明之前，我们不可能预测这个初始化操作的程序行为。如果 class Word 的设计者定义了一个 copy constructor，verb 的初始化操作会调用它。但如果该 class 没有定义 explicit copy constructor，那么是否会有一个编译器合成的实体被调用呢？这就得视该 class 是否展现 “bitwise copy semantics” 而定。举个例子，已知下面的 class Word 声明：
```cpp
// 以下声明展现了bitwise copy semantics
class Word{
public:
    Word(const char*);
    ~Word() {delete [] str;}
    // ...
private:
    int cnt;
    char *str;
};
```
这种情况下并不需要合成一个 default copy constructor，因为上述声明展现的"default copy semantics"，而 verb 的初始化操作也就不需要以一个函数调用收场[4].

[4] 当然，程序的执行将因为 class Word 如此宣告而灾情惨重。如今 local object verb 和 global object noun 都指向相同的字符串。在退出 foo() 之前，local object verb 会执行 destructor，于是字符串被删除，global object noun 从此指向一堆无意义之物。member str 的问题只能够靠“由 class 设计者实现出一个 explicit copy constructor 以改写 default memberwise initialization” 或是靠“不允许完全拷贝”解决之。不过这和“是否有一个 copy constructor 被编译器合成出来”没有关系。

然而，如果 class World 是这样声明：
```cpp
// 以下声明并未展现出bitwise copy semantics
class Word{
public:
    Word(const String&);
    ~Word();
    // ...
private:
    int cnt;
    String str;
};
```
其中String声明了一个 explicit copy constructor：
```cpp
class String{
public:
    String(const char*);
    String(const String&);
    ~String();
    // ...
};
```
在这个情况下，编译器必须合成出一个 copy constructor 以便调用 member class String object 的 copy constructor:
```cpp
// 一个被合成出来的copy constructor
// C++伪码
inline Word::word(const Word& wd)
{
    str.String::String(wd.str);
    cnt = wd.cnt;
}
```
有一点很值得注意：在这被合成出来的 copy cosntructor 中，如整数、指针、数组等等的 nonclass member 也都会被复制，正如我们所期待的一样。

### 不要 Bitwise Copy Semanatics！

什么时候一个 class 不展现出 “bitwise copy semantics” 呢？有四种情况：

1. 当 class 内含一个 member object 而后来的 class 声明有一个 copy constructor 时（无论是被 class 设计者明确地声明，就像前面的 String 那样；或是被编译器合成，像 class Word 那样）。
2. 当 class 继承自一个 base class 而后面存在有一个 copy constructor 时（再次强调，不论是被明确声明或是被合成而得）。
3. 当 class 声明了一个或多个 virtual functions 时。
4. 当 class 派生自一个继承串链，其中有一个或多个 virtual base classes 时。

前两种情况中，编译器必须将 member 或 base class 的 “copy constructor调用操作” 安插到被合成的 copy constructor 中。前一节 class Word 的 “合成而得的 copy constructor” 正足以说明情况 1。情况 3 和 4 有点复杂，是我接下来要讨论的题目。

### 重新设定 Virtual Table 的指针

回忆编译期间的两个程序扩张操作（只要有一个 class 声明了一个或多个 virtual functions 就会如此）：
1. 增加一个 virtual function table(vtbl)，内含每一个有作用的 virtual function 的地址。
2. 将一个执行 virtual function table 的指针（vptr），安插在每一个 class object 内。

很显然，如果编译器对于每一个新产生的 class object 的 vptr 不能成功而正确地设好其初值，将导致可怕的后果。因此，当编译器导入一个 vptr 到 class 之中时，该 class 就不在展现 bitwise semantics 了。现在，编译器需要合成出一个 copy constructor，以求将 vptr 适当地初始化，下面是个例子。

首先，我定义两个 classes，ZooAnimal 和 Bear:
```cpp
class ZooAnimal{
public:
    ZooAnimal();
    virtual ~ZooAnimal();

    virtual void animate();
    virtual void draw();
    // ...
private:
    // ZooAnimal的animate()和draw()
    // 所需要的数据
};

class Bear : public ZooAnimal{
public:
     Bear();
    void animate();     // 译注：虽未明写virtual，它其实是virtual
    void draw();        // 译注：虽未明写virtual，它其实是virtual
    virtual void dance();
    // ...
private:
    // Bear的animate()和draw()和dance()
    // 所需要的数据
};
```
ZooAnimal class object 以另一个 ZooAnimal class object 作为初值，或 Bear class object 以另一个 Bear class object 作为初值，都可以直接靠 “bitwise copy semantics” 完成（除了可能会有的 pointer member 之外。为了简化，这种情况被我剔除）。举个例子：
```cpp
Bear yogi;
Ber winnie = yogi;
```
yogi 会被 default Bear constructor 初始化。而在 constructor 中，yogi 的 vptr被设定指向 Bear class 的 virtual table（靠编译器安插的码完成）。因此，把 yogi 的 vptr 指拷贝给 winnie 的 vptr 是安全的。

译注：让我译下面这张图来补充说明yogi和winnie的关系：
<img src = "virtual_table_for_Bear.png" width = '80%'>

当一个 base class object 以其 derived class 的 object内容做初始化操作时，其 vptr 复制操作也必须保证安全，例如：
```cpp
ZooAnimal franny = yogi;    // 译注：这会发生切割（slices）行为
```
franny 的 vptr 不可以被设定指向 Bear class 的 virtual table (但如果 yogo 的 vptr 被直接 “bitwise copy” 的话，就会导致此结果)，否则当下面程序片段中的 draw() 被调用而 franny 被传进去时，就会“炸毁”（blow up）[5]：
```cpp
void draw(const ZooAnimal& zoey) { zoey.draw(); }
void foo(){
    // franny的vptr指向ZooAnimal的virtual table，
    // 而非Bear的virtual table（彼由yogo的vptr指出）
    ZooAnimal franny = yogo;

    draw(yogi);     // 调用Bear::draw()
    draw(frannt);   // 调用ZooAnimal::draw(0)
}
```
[5] 通过 franny 调用 virtual function draw()，调用的是 ZooAnimal 实体而非 Bear 实体（甚至虽然 franny 是以 Bear object yogi 作为初值），因为 franny 是一个 ZooZnimal object。事实上，yogi 中的 Bear 部分已经在 franny 初始化时被切割掉了。如果 franny 被声明为一个 refernnce（或如果它是一个指针，而其值为 yogi 的地址），那么经由 franny 所调用的 draw() 才会是 Bear 的函数实体。这已在 1.3 节讨论过。

译注：让我再以下面这张图来补充说明yogi和franny的关系：
<img src = "franny.png" width = "80%">

也就是说，合成出来的ZooAnimal copy constructor会明确设定object的vptr指向ZooAnimal class的virtual table，而不是直接从右手边的class object中将其vptr现值拷贝过来。

### 处理 Virtual Base Class Subobject

Virtual base class 的存在需要特别处理。一个 class object 如果以另一个 object 作为初值，而后者有一个 virtual base class subobject，那么也会使 “bitwise copy semantics” 失效。

每一个编译器对于虚拟继承的支持承诺，都表示必须让 “derived class object” 中的 virtual base class subobject 位置”在执行期就准备妥当。维护“位置的完整性”是编译器的责任。“Bitwise copy sematics” 可能会破坏这个位置，所以编译器必须在它自己合成出来的 copy constructor 中作出仲裁。举个例子，在下面的声明中，ZooAnimal 成为 Raccoon 的一个 virtual base class：
```cpp
class Raccoono : public virtual ZooAnimal{  // 译注：raccoon，浣熊
public:
    Raccoon() { /* 设定private data初值 */}
    Raccoon(int val) { /* 设定private data初值 */ }
    // ...
private:
    // 所有必要的数据
};
```
编译器所产生的代码（用以调用 ZooAnimal 的 default constructor、将 Raccoon 的 vptr 初始化，并定位出 Raccoon 中的 ZooAnimal subobject）被安插在两个 Raccoon constructor 之内，成为其先头部队。

那么 “memberwise初始化” 呢？奥，一个 virtual base class 的存在会使 bitwise copy semantics 无效。其次，问题并不发生于 “一个 class object 以另一个同类的 object 作为初值” 之时，而是发生于 “一个 class object 以其 derived classes 的某个 object 作为初值” 之时。例如，让 Raccoon object 以一个 RedPanda object 作为初值，而 RedPanda 声明如下：
```cpp
class RedPanda : public Raccoon{ // 译注：RedPanda，大熊猫
public:
    Redpanda() { /* 设定private data初值 */ }
    RedPanda(int val) { /* 设定private data初值 */ }
    // ...
private:
    // 所有必要的数据
};
```
我再强调一次，如果以一个 Raccoon object 作为另一个 Raccoon object 的初值，那么 “bitwise copy” 就绰绰有余了：
```cpp
// 简单的bitwise copy就足够了
Raccoon rocky；
Raccoon little_critter = rocky;
```

然而如果企图以一个 RedPanda object 作为 little_criter 的初值，编译器必须判断 “后续当程序员企图存取其 ZooAnimal subobject 时是否能够正确地执行” （这是一个理性的程序员所期望的）：
```cpp
// 简单的bitwise copy还不够
// 编译器必须明确地将little_critter的
// virtual base class pointer/offset初始化

RedPanda little_red;
Raccoon little_critter = little_red;
```
在这种情况下，为了完成正确的 little_critter 初值设定，编译器必须合成一个 copy constructor，安插一些码以设定 virtual base class pointer/offset 的初值（或只是简单地确定它没有被抹掉），对每一个 members 执行必要的 memberwise 初始化操作，以及执行其它的内存相关工作（3.4 节对于 virtual base classes 有更详细的讨论）。

译注：让我以下面这张图来补充说明 little_red 和 little_criter 的关系：

<img src = "little_red.png" width = "%80">

在下面的情况中，编译器无法知道是否 “bitwise copy semantics” 还保持着，因为它无法知道（没有流程分析）Raccoon 指针是否指向一个真正的 Raccoon object，或是指向一个 derived class object:
```cpp
// 简单的bitwise copy可能够用，也可能不够用
Raccoon *ptr;
Raccoon little_critter = *ptr;
```
这里有一个有趣的问题：当一个初始化操作存在并保持着 “bitwise copy semantics” 的状态时，如果编译器能够保证 object 有正确而相等的初始化操作，是否它应该压抑 copy constructor 的调用，以使其所产生的程序代码优化？至少在合成的 copy constructor 之下，程序副作用的可能性是零，所以优化似乎是合理的。如果 copy constructotr 是由 class 设计者提供的呢？这是一个颇有争议的题目，我将在下一届结束前回来讨论之。

让我做个总结：我们已经看过四种情况，在那些情况下 class 不再保持 “bitwise copy semantics”，而且 default copy constructor 如果未被声明的话，会被视为nontrivial。在这四种情况下，如果缺乏一个已声明的 copy constructor，编译器为了正确处理 “以一个 class object 作为另一个 class object 的初值”，必须合成出一个 copy constructor。下一节将讨论编译器调用 copy constructor 的策略，以及这些策略如何影响我们的程序。

## 2.3 程序转化语意学（Program Transformation Semantics）

已知下面程序片段：
```cpp
#include "X.h"

X foo()
{
    X xx;
    // ...
    return x;;
}
```
一个人可能会做出以下假设：
1. 每次 foo() 被调用，就传送回 xx 的值。
2. 如果 class X 定义了一个 copy constructor，那么当 foo() 被调用时，保证该 copy constructor 也会被调用。

第一个假设的真实性，必须视 class X 如何定义而定。第二个假设的真实性，虽然也有部分必须视 class X 如何定义而定，但最主要还是视你的 C++ 编译器所提供的的进取性优化程度（degree of aggressive optimization）而定。你甚至可以假设在一个高品质的 C++ 编译器中，上述两点对于 class X 的 nontrivial definitions 都不正确。以下小节将讨论其原因。

### 明确的初始化操作

已知有这样的定义：
```cpp
X x0;
```
下面有三个定义，每一个都明显地以 x0 来初始化其 class object：
```cpp
void foo_bar(){
    X x1(x0);       // 译注：定义了x1
    X x2 = X0;      // 译注：定义了x2
    X x3 = X(x0);   // 译注：定义了x3
    // ...
}
```
必要的程序转化有两个阶段：
1. 重写每一个定义，其中的初始化操作会被剥除。（译注：这里所谓的“定义”是指上述的x1，x2，x3 三行；在严谨的 C++ 用法中，“定义”是指“占用内存”的行为）。
2. class 的 copy constructor 调用操作会被安插进去。

举个例子，在明确的双阶段转化之后，foo_bar() 可能看起来像这样：
```cpp
// 可能的程序转换
// C++伪码
void foo_bar(){
    X x1;   // 译注：定义被重写，初始化操作被剥除
    X x2;   // 译注：定义被重写，初始化操作被剥除
    X x3;   // 译注：定义被重写，初始化操作被剥除

    // 编译器安插X copy construction的调用操作
    x1.X::X(x0);
    x2.X::X(x0);
    x3.X::X(x0);
    // ...
}
```
其中的：
```cpp
    x1.X::X(x0);
```
就表现出对以下的 copy constructor 的调用：
```cpp
    X::X(const X& xx);
```

## 参数的初始化（Argument Initialization）

C++ Standard (Sectin 8.5) 说，把一个 class object 当做参数传给一个函数（或是作为一个函数的返回值），相当于以下形式的初始化操作：
```cpp
    X xxx = arg;
```
其中，xx 代表形式参数（或返回值）而 arg 代表真正的参数值。因此，若已知这个函数：
```cpp
    void foo(X x0);
```
下面这样的调用方式：
```cpp
    X xx;
    // ...
    foo(xx);
```
将会要求局部实体（local instance）x0 以 memberwise 的方式将 xx 当做初值。在编译器实现技术上，有一种策略是导入所谓的暂时性 object，并调用 copy constructor 将它初始化，然后将该暂时性 object 交给函数。例如将前一段程序代码转换如下：
```cpp
// C++伪码
// 编译器产生出来的暂时对象
X __temp0;

// 编译器对copy constructor的调用
__temp0.X::X( xx );
// 重新改写函数调用操作，以便使用上述的暂时对象
foo( __temp0 );
```
然而这样的转换只做了一半功夫而已。你看出残留问题了吗？问题出在 foo() 的声明。暂时性 object 先以 class X 的 copy constructor 正确地设定了初值，然后再以 bitwise 方式拷贝到 x0 这个局部实体中。奥，真讨厌，foo() 的声明因而也必须被转化，形式参数必须从原先的一个 class X object 改变为一个 class X refernce，像这样：
```cpp
 void foo(X& x0);
```
其中 class X 声明了一个 destructor，它会在 foo() 函数完成之后被调用，对付那个暂时性的 object。

另一种实现方式是以“拷贝建构”（copy construct）的方式把实际参数直接建构在其应该的位置上，该位置视参数活动范围的不同记录于程序堆栈中。在函数返回之前，局部对象（local object）的 destructor(如果有定义的话)会被执行。Borland C++ 编译器就是使用此法，但它也提供一个编译选项，用以指定前一种做法，以便和其早期版本兼容。

### 返回值的初始化（Return Value Initialization）

已知下面这个函数定义：
```cpp
X bar()
{
    X xx;
    // 处理 xx ...
    return xx;
}
```
你可能会问 bar() 的返回值如何从局部对象 xx 中拷贝过来？Stroustrup 在 cfront 中的解决方式是一个双阶段转化：
1. 首先加上一个额外参数，类型是 class object 的一个 refernce。这个参数将用来放置被“拷贝构建（copy constructed）”而得的返回值。
2. 在 return 指令之前安插一个 copy constructor 调用操作，以便将欲传回之 object 的内容当做上述新增参数的初值。

真正的返回值是什么？最后一个转化操作会重新改写函数，使他不传回任何值。根据这样的算法，bar() 转换如下：
```cpp
// 函数转换
// 以反映出copy constructor的应用
// C++伪码
void bar(X& __result)   // 译注：加上一个额外参数
{
    X xx;
    // 编译器所产生的default constructor调用操作
    xx.X::X();

    // ...处理xx

    // 编译器所产生的copy constructor调用操作
    __result.X::XX(xx);
    return;
}
```
现在编译器必须转换每一个 bar() 调用操作，以反映其新定义。例如：
```cpp
X xx = bar();
```
将被转换为下列两个指定句：
```cpp
// 注意，不必施行default constructor
X xx;
bar(xx);
```
而：
```cpp
bar().memfunc();    // 译注：执行bar()所传回之X class object的memberfunc()
```
可能被转化为：
```cpp
// 编译器所产生的暂时对象
X __temp0;
( bar(__temp0), __temp0).memfunc();
```
同样道理，如果程序声明了一个函数指针，像这样：
```cpp
X (*pf)();
pf = bar;
```
它也必须被转化为：
```cpp
void ( *pf ) ( X& );
pf = bar;
```

### 在使用者层面做优化（Optimization at the User Level）

我相信始作俑者是 Jonathan Shopiro!他对于像 bar() 这样的函数，最先提出“程序员优化”的观念：定义一个“计算用”的 constructor。换句话说程序员不再写：
```cpp
X bar(cosnt T& y, const T& z){
    X xx;
    // ...以y和z来处理xx
    return xx;
}
```
那会要求 xx 被 “memberwise” 地拷贝到编译器所产生的 __result 之中。Jonathan 定义另一个 constructor，可以直接计算 xx 的值：
```cpp
X bar(const T& y, const T& z)
{
    return X(y, z);
}
```
于是当 bar() 的定义被转换之后，效率会比较高：
```cpp
// C++伪码
void bar(X& __result)
// 译注：上行是否应为bar(X& __result, const T& y, const T& z)
{
    __result.X::X(y, z);
    return;
}
```
__result 被直接计算出来，而不是经由 copy constructor 拷贝而得！不过这种解决方法受到了某种批评，怕那些特殊用途的 constructor 可能会大量扩散。在这个层面上，class 的设计是以效率考虑居多，而不是以“支持抽象化”为优先。

### 在编译器层面做优化（Optimization at the Compiler Level）

在一个如 bar() 这样的函数中，所有的 return 指令传回相同的具名数值（译注：named value，我想作者指 p.64 的 xx），因此编译器有可能自己做优化，方法是以 result 参数取代 named return value。例如下面的 bar() 定义：
```cpp
X bar(){
    X xx;
    // ..处理xx
    return xx;
}
```
编译器把其中的xx以__result取代：
```cpp
void bar(X& __result){
    // default constructor被调用
    // C++ 伪码
    __result.X::X();
    
    // ...直接处理__result

    return;
}
```
这样的编译器优化操作，有时候被称为 Named Return Value（NRV）优化，在 ARM Section 12.1.1.c (300~303 页) 中有所描述。NRV 优化如今被视为是标准 C++ 编译器的一个义不容辞的优化操作——虽然其需求其实超越了正式标准之外。为了对效率的改善有所感觉，请你想想下面的码：
```cpp
class test{
    friend test foo(doule);
public:
    test()
        { memset(arr, 0, 100 * sizeof(double));}
private:
    double array[100];
};
```
同时请考虑一下函数，它产生、修改，并传回一个 test class object:
```cpp
test
foo(double val)
{
    test local;

    local.array[0] = val;
    local.array[99] = val;

    return local;
}
```
有一个main()函数调用上述foo()函数一千万次：
```cpp
main()
{
    for(int cnt = 0; cnt < 10000000; cnt++)
        { test t = foo(double(cnt));}
    return 0;
}
// 译注：整个程序的意思是重复循环10000000此，每次产生一个testobject;
// 每个test object配置一个拥有100个double的数组；所有的元素都设初值为0
// 只有#0和#99元素以循环计数器的之作为初值
```
这个程序的第一个版本不能实施 NRV 优化，因为 test class 缺少一个 copy constructor。第二个版本加上一个 inline copy constructor 如下：
```cpp
inline
test::test(const test& t)
{
    memcpy(this, &t, sizeof(test));
}
// 译注：别忘了class test的声明中加一个member function如下
// public:
// inline test(cosnt test& t);
```
这个 copy constructor 的出现激活了 C++ 编译器中的 NRV 优化。NRV 优化的执行并不通过另外独立的优化工具完成。下面是测试时间表（除了效率的改善，另一个值得注意的或许是不同编译器间的效率差异）：

Named Return Value(NRV) 优化
||未实施 NRV|实施 NRV|实施 NRV+-O（译注：-O 表示优化）|
|-|-|-|-|
|CC|1:48.52|46.73|46.05|
|NCC|3:00.57|1:33.48|1:32.36|

虽然 NRV 优化提供了重要的效率改善，它还是饱受批评。其中一个原因是，优化由编译器默默完成，而它是否真的被完成，并不十分清楚（因为很少有编译器会说明其实现程度，或是否实现）（译注见下页）。第二个原因是，一旦函数变得非常复杂，优化也就变得比较难以实行。在 cfront 中，只有当所有的 named return 指令句发生于函数的 top level 时，优化才施行。如果导入 “a nested local block with a return statement”，cfront 就会静静地将优化关闭。许多程序员主张对这种情况应该以 “特殊化的constructor” 策略取代之。

译注：我猜或许 Microsoft Visual C++ 就是 Lipman 所说的那种编译器。我把前述的程序范例以 VC++ 5.0 编译后执行，在 Pentium 133, 96MB RAM 的机器上得结果如下。加上 inline copy constructor 之后，按理应实施 NRV 优化，结果效率反而较差。由于不知 Lippman 在何种硬设备上测试 CC 和 NCC，所以我的这些数据不应该拿来和上一页的结果比较，只宜就 NRV 之实施与未实施两种情况比较之。

Named Return Value(NRV)优化
||未实施 NRV|实施 NRV|实施 NRV+-O（译注：-O 表示优化）|
|-|-|-|-|
|VC|2:21|2:46.4|2:20.8|

译注：欲得知程序所费时间，可在 main() 的 for loop 先后调用 printlocaltime()，而 printlocaltime() 可设计如下（应用了 C Runtime Library 中的时间函数）：
```cpp
#include <stdio.h>  // for printf()
#include <time.h>   // for time() and localtime()
void printlocaltime(void)
{
    struct tm *timeptr;
    time_t secsnow;

    time(&secnoe);
    timeptr = localtime(&secsnow);
    print("The data is %d-%d-19%02d\n", // 进入21世纪，这招就不行了☺
        (timeptr->tm_mon) + 1,
        (timeptr->tm_mday),
        (timeptr->tm_year));
    printf("Local time is %02d:%02d:%02d\n",
        timeptr->tm_hour,
        timeptr->tm_min,
        timeptr->tm_sec);
}

```

上述两个批评主要关心的是编译器可能在施行优化时失败。第三个批评则是从相反的方向出发。某些程序员真的不喜欢应用程序被优化，你知道他们抱怨什么吗？想象你已经摆好了你的 copy constructor 的阵势，使你的程序 “以 copying 方式产生出一个 object 时”，对称地调用 destructor，例如：
```cpp
void foo()
{
    // 这里希望有一个copy constructor
    X xx = bar();
    // ...
    // 这里调用destructor
}
```
在此情况下，对称性被优化给打破了：程序虽然比较快，却是错误的。难道编译器因为压抑了 copy constructor 的调用而在这里出毛病吗？也就是说，难道 copy constructor 在 “object 是经由 copy 而完成其初始化”的情况下，一定要被调用吗？

这样的需求在许多程序中可能会被征以严格的“效率税”。例如，虽然下面三个初始化操作在语意上相等：
```cpp
X xx0(1024);
X xx1 = X(1024);
X xx2 = (X)1024;
```
但是在第二行和第三行中，语法明显地提供了两个步骤的初始化操作：
1. 将一个暂时性的 object 设以初值 1024;
2. 将暂时性的 object 以拷贝建构的方式作为 explicit object 的初值。

换句话说，xx0 是被单一的 constructor 操作设定初值：
```cpp
// C++伪码
xx0.X::X(1024);
```
而 xx1 和 xx2 却调用两个 constructor，产生一个暂时性 object，并针对该暂时性 object 调用 class X 的 destructor：
```cpp
// C++伪码
X __temp0;
__temp0.X::X(1024);
xx1.X::X(__temp0);
__temp0.X::~X();
```

C++标准委员会已经讨论过 “剔除 copy constructor 调用操作” 的合法性。在我下笔时刻，委员会还没有最后的决议。然而，按照 Josee Lajoie（委员会副主席，同时也是Core Language group 主席）的看法，NRV 优化非常重要，不应该驳回。很明显这项讨论已经逐渐导致多少带点神秘气息的两种情况：是否 copy cosntructor 的剔除在 “拷贝 static object 和 local object” 时也应该成立？例如，已知下面程序片段：
```cpp
Thing outer;
{
    // 可以不加考虑innser吗
    Thing inner(outer);
}
```
inner 应该从 outer 中拷贝构造起来，或是 inner 可以简单地被忽略？这样的问题可以很类似地套问在有关 static extern objects 拷贝构造的操作身上。依照 Josee 的看法，要消除 static objects 的 copy constructor，几乎可以肯定是不被允许的。至于 automatic objects 如 inner，则尚未决议。

一般而言，面对 “以一个 class object 作为另一个 class object 的初值”的情形，语言允许编译器有大量的自由发挥空间。其利益当然是导致机器码产生时有明显的效率提升缺点则是你不能够安全地规划你的 copy constructor 的副作用，必须视其执行而定。

### Copy cosntrucotr：要还是不要？

已知下面的 3D 坐标点类：
```cpp
class Point3d{
public:
    Point3d(float x, float y, float z);
    // ...
private:
    float _x, _y, _z;
};
```
这个 class 的设计者应该提供一个 explicit copy constructor 吗？

上述 class 的 default copy cosntructor 被视为 trivial。它既没有任何 member（或 base）class Objects 带有 copy constructor，也没任何的 virtual base class 或 virtual function。所以，默认情况下，一个 Point3d class object 的 “memberwise” 初始化操作会导致 “bitwise copy”。这样的效率很高，但安全吗？

答案是 yes。三个坐标成员是以数值来存储。bitwise copy 既不会导致 memory leak，也不会产生 address aliasing，因此它即快速又安全。

那么，这个 class 的设计者应该提供一个 explicit copy constructor 吗？你讲如何回答这个问题？答案当然很明显是 no。没有任何理由要你提供一个 copy constructor 函数实体，因为编译器自动为你实施了最好的行为。比较难以回答的是，如果你被问及是否预见 class 需要大量的 memberwise 初始化操作，例如以传值（by value）的方式传回 objects？如果答案是 yes，那么提供一个 copy constructor 的 explicit inline 函数实体就非常合理——在“你的编译器提供 NRV 优化”的前提下。

例如，Point3d 支持下面一组函数：
```cpp
Point3d operator+(const Point3d&, const Point3d&);
Point3d operator-(const Point3d&, const Point3d&);
Point3d operator*(cosnt Point3d&, int);
```
所有那些函数都能够良好地符合 NRV template：
```cpp
{
    Point3d result;
    // 计算result
    return result;
}
```
实现 copy constructor 的最简单方法像这样：
```cpp
Point3d::Point3d(const Point3d &rhs)
{
    memcpy(this, &rhs, sizeof(Point3d));
};
```
然而不管使用 memcpy() 或 memset()，都只有在 “class不含任何由编译器产生的内部 members” 时才能有效运行。如果 Point3d class 声明一个或一个以上的 virtual functions，或内含一个 virtual base class，那么使用上述函数将会导致那些 “编译器产生的内部members” 的初值被改写。例如，已知下面声明：
```cpp
class Shape{
public:
    // 喔欧：这会改变内部的vptr
    Shape() { memset(this, 0, sizeof(Shape));}
    virtual ~Shape();
    // ...
}
```
编译器为此 constructor 扩张的内容看起来像是：
```cpp
// 扩张后的constructor
// C++伪码
Shape::Shape()
{
    // vptr必须在使用者的代码执行之前先设定妥当
    __vptr__Shape = __vtbl__Shape;

    // 喔欧：memset会将vptr清为0
    memset(this, 0, sizeof(Shape));
};
```
如你所见，若要正确使用 memset() 和 memcpy() ，需要掌握某些 C++ Object Model 的语意学知识！

### 摘要

copy constructor 的应用，迫使编译器多多少少对你的程序代码做部分转化。尤其是当一个函数以传值（by value）的方式传回一个 class object，而该 class 有一个 copy constructor（不论是明确定义出来的，或是合成的）时。这将导致深奥的程序转化——不论在函数的定义或使用上。此外编译器也将 copy cosntructor 的调用操作优化，以一个额外的第一参数（数值被直接存放于其中）取代 NRV。程序员如果了解那些转换，以及 copy constructor 优化后的各种可能状态，就比较能够控制他们的程序的执行效率。

## 2.4 成员们的初始化队伍（Member Initialization List）

当你写下一个 constructor 时，你有机会设定 class members 的初值。要不是经由 member initialization list，就是在 constructor 函数本身之内。除了四种情况，你的任何选择其实都差不多。

本节之中，我首先要澄清何时使用 initialization list 才有意义，然后解释 list 内部的真正操作是什么，然后我们再来看看一些微妙的陷阱。

下列情况中，为了让你的程序能够被顺序编译，你必须使用 member initialization list：
1. 当初始化一个 reference member 时；
2. 当初始化一个 const member 时；
3. 当调用一个 base class 的 constructor，而它拥有一组参数时；
4. 当调用一个 member class 的 constructor，而它拥有一组参数时。

在这四种情况中，程序可以被正确编译执行，但是效率不彰。例如：
```cpp
class Word{
    String _name;
    int _cnt;
public:
    // 没有错误，只不过太天真...
    word(){
        _name = 0;
        _cnt = 0;
    }
};
```
在这里，Word constructor 会先产生一个暂时性的 String object，然后将它初始化，再以一个 assignment 运算符将暂时性 object 指定给 _name，然后再摧毁那个暂时性 object。这是故意的吗？不太可能。编译器会产生一个警告码？我不知道！以下是constructor 可能的内部扩张结果：
```cpp
// C++ 伪码
Word::word(/* this pointer goes here */)
{
    // 调用String的default constructor
    _name.String::String();

    // 产生暂时性对象
    String temp = String(0);

    // "memberwise"地拷贝_name
    _name.String::operator=(temp);

    // 摧毁暂时性对象
    temp.String::~String();

    _cnt = 0;
}
``` 
对程序代码反复审查并修正之，得到一个明显更有效率的实现方法：
```cpp
// 较佳的方式
Word::Word : _name(0)
{
    _cnt = 0;
}
```
它会被扩张成这个样子：
```cpp
// C++伪码
Word::Word( /* this pointer goes here */)
{
    // 调用String(int) constructor
    _name.String::String(0);
    _cnt = 0;
}
```
顺带一提，陷阱最有可能发生在这种形式的 template code 中：
```cpp
template<class type>
foo<type>::foo(type t)
{
    // 可能是（也可能不是）个好主意
    // 视type的真正类型而定
    _t = t;
}
```
这会引导某些程序员十分积极进取地坚持所有的 member 初始化操作必须在 member initialization list 中完成，甚至即使是一个行为良好的 member 如 _cnt：
```cpp
// 坚持此种写码风格
Word::World()
    : _cnt(0), _name(0)
    {}
```
在这里我们不禁要提出一个合理的问题：member initialization list 中到底会发生什么事情？许多 C++ 新手对于 list 的语法感到迷惑，他们误以为它是一组函数调用。当然它不是！

编译器会一一操作 initialization list，以适当次序在 constructor 之内安插初始化操作，并且在任何 explicit user code 之间。例如，先前的 Word constructor 被扩充为：
```cpp
// C++伪码
Word::Word( /* this pointer goes here */ )
{
    _name.String::String(0);
    _cnt = 0;
}
```
嗯·····嗯，它看起来很像是在 constructor 中指定 _cnt 的值。事实上，有一些微妙的东西要注意：list 中的项目次序是由 class 中的 members 声明次序决定，不是由 initialization list 中的排列次序决定。在本例的 Word class 中，_name 被声明于 _cnt 之前，所以它的初始化也比较早。

“初始化次序” 和 “initialization list 中的项目排列次序” 之间的外观错乱，会导致下面意想不到的危险：
```cpp
class X{
    int i;
    int j;
public:
    // 喔欧，你看出问题了吗？（译注）
    X(int val)
        : j(val), i(j)
    {}
};
```
译注：上述程序代码看起来像是要把 j 设初值为 val，再把 i 设初值为 j。问题在于，由于声明次序的缘故，initialization list 中的 i(j) 其实比 j(val) 更早执行。但因为 j 一开始未有初值，所以 i(j) 的执行结果 i 无法预知其值。稍后我会列出一个完整的范例。

这个“臭虫”的困难度在于它很不容易被观察出来。编译器应该发出一个警告消息，但是到目前为止我只知道有一个编译器（g++, GNU c++ 编译器[6]）做到这一点。我建议你总是把一个 member 的初始化操作和另一个放在一起（如果你真觉得有必要的话），放在 constructor 之内，像这样：
```cpp
// 比较受到喜爱的方式
X::X(int val)
    : j(val)
{
    i = j;
}
```

译注：现在我把有陷阱的写法和更改后的写法放到一个程序中去，检验其结果：
```cpp
#include <stdio.h>

class X{
public:
    int i;
    int j;
public:
    // 喔欧，你看出问题了吗
    X(int val)
        : j(val), i(j)  // 有陷阱的写法
    {}
};

class Y{
public:
    int i;
    int j;
public:
    Y(int val)
        : j(val)
    { i = j; }      // 修改后的写法
};

main()
{
    X x(3);
    Y y(5);

    printf("x.i = %d x.j = %d \n", x.i, x.j);
    printf("y.i = %d y.j = %d \n", y.i, y.j);

    return 0;
}
```
执行结果如下：
```cpp
F:\lippman\prog\inilist.02>inilist
x.i = -2124198216  x.j = 3
y.i = 5  y.j = 5
```
果然 x.i 的指不是我们所期望的3.

这里还有一个有趣的问题，initialization list 中的项目被安插到 constructor 中，会继续保存声明的次序码？也就是说，已知：
```cpp
// 一个有趣的问题：
X::X(int val)
    : j(val)
{
    i = j;
}
```
j 的初始化操作会安插在 explict user assignment 操作（i = j）之前或之后？如果声明次序继续被保存，则这份码大大不妙（译注：因为势必要先将 i 初始化再将 j 初始化）。然而这份码其实是正确的，因为 initialization list 的项目被放在 explicit user code 之前。

另一个常见的问题是，是否你能够像下面这样，调用一个 member function 以设定一个member 的初值：
```cpp
// X::xfoo()被盗用，这样好吗
X::X(int val)
    : i(xfoo(val)), j(val)
    {}
```
其中的 xfoo() 是 X 的一个 member function。答案是 yes，但是······唔·····之所以加上但是，是因为我要给你一个忠告：请使用 “存在于 constructor 体内的一个member” ，而不要使用 “存在于 member initialization list 中的 member”，来为另一个 member 设定初值。你并不知道 xfoo() 对 X object 的依赖性有多高，如果你把 xfoo() 放在 constructor 体内，那么对于 “到底是哪一个 member 在 xfoo() 执行时被设立初值” 这件事，就可以确保不会发生模棱两可的情况。

Member function 的使用是合法的（当然我们必须不考虑它所用到的 members 是否已初始化），这是因为和此 object 相关的 this 指针已经被建构妥当，而 constructor 大约被扩充为：
```cpp
// C++伪码：constructor被扩充后的结果
X::X( /* this poiter, */ int val)
{
    i = this->xfoo(val);
    j = val;
}
```
最后，如果一个 derived class member function 被调用，其返回值被当做 base class constructor 中的一个参数，将会如何：
```cpp
// 调用FooBar::fval()可以吗
class FooBar : public X{            // 译注：base class是X
    int _fval;
public:
    int fval() { return _fval; }    // 译注：derived class member function
    FooBar(int val)
        : _fval(val),
        X(fval())       // 译注：fval()作为base class constructor的参数
    {}
    ...
}
```
你认为如何？这是一个好主意吗？下面是它可能的扩张结果：
```cpp
// C++ 伪码
FooBar::FooBar( /* this pointer goes here */)
{
    // 喔欧：实在不是一个好主意
    X::X(this, this->fval());
    _fval = val;
}
```
它的确不是一个好主意（后继数章对于 base class 和 virtual base class 在 member initialization list 中的初始化程序会有比较详细的说明）。

简略地说，编译器会对 initialization list 一一处理并可能重新排序，以反映出 members 的声明次序。它会安插一些代码到 constructor 体内，并置于任何 explicit user code 之前。



[第 1 章 关于对象（Object Lessons）](Object_Lessons.md)|[第 3 章 Data 语意学（The Semantics of Data）](The_Semantics_of_Data.md)
