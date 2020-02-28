# 第5章 构造、解构、拷贝语意学 Semantics of Construction, Destruction, and Copy

考虑下面这个 abstract base class 声明：
```cpp
class Abstract_base{
public:
    virtual ~Abstract_base() = 0;
    virtual void interface() const = 0;
    virtual const char*
        mumble() const { return _mumble; }
protected:
    char* _mumble;
};
```
你看出什么问题了没有？虽然这个 class 被设计为一个抽象的 base class（其中有 pure virtual function，使得 Abstract_base 不可能拥有实体），但它仍然需要一个明确的构造函数以初始化其 data member _mumble。如果没有这个初始化操作，其 derived class 的局部性对象 _mumble 将无法决定初值，例如：
```cpp
class Concrete_derived : public Abstract_base{
public:
    Concrete_derived();
    // ...
};
void foo()
{
    // Abstract_base::_mumble 未被初始化
    Concrete_derived trouble;
    // ...
}
```
你可能会争辩说，也许 Abstract_base 的设计者试图让其每一个 derived class 提供 _mumble 的初值。然而如果是这样，derived class 的唯一要求就是 Abstract_base 必须提供一个带有唯一参数的 protected constructor：
```cpp
Abstract_base::
Abstract_base(char *mumble_value = 0)
    : _mumble(mumble_value)
    { }
```
一般而言，class 的 data member 应该被初始化，并且只在 constructor 中或是在 class 的其它 member functions 中指定初值。其它任何操作都将破坏封装性质，使 class 的维护和修改更加困难。

当然你也可能争辩说设计者的错误并不在于未提供一个 explicit constructor，而是他不应该在抽象的 base class 中声明 data members。这是比较强而有力的论点（把 interface 和 implementation 分离），但它并不是行遍天下皆有理，因为将 “被共享的数据” 抽取出来放在 base class 中，毕竟是一种正当的设计。

### 纯虚拟函数的存在（Presence of a Pure Virtual Function）

C++ 新手常常很惊讶地发现，一个人竟然可以定义和调用（invoke）一个 pure virtual function; 不过它只能被静态地调用（invoked statically）,不能经由虚拟机制调用。例如，你可以合法地写下这段码：
```cpp
// ok : 定义pure virtual function
//      但只可能被静态地调用（invoked statically）

inline void
Abstract_bae::interface() const
                    // 译注： 原书缺少const，应为笔误
{       // 译注：请注意，先前曾声明这是一个pure virtual const function
    // ...
}
inline void
Concrete_derived::interface() const
                    // 译注： 原书缺少const，应为笔误
{
    // ok : 静态调用（static invocation）
    Abstract_base::interface();
                // 译注：请注意，我们竟然能够调用一个pure virtual function
    // ...
}
```
要不要这样做，全由 class 设计者决定。唯一的例外就是 pure virtual destructor:  class 设计者一定要定义它。为什么？因为每一个 derived class destructor 会被编译器加以扩展，以静态调用的方式调用其 “每一个 virtual base class” 以及 “上一层 base class” 的 destructor。因此，只要缺乏任何一个 base class destructors 的定义，就会导致链接失败。

译注：以 VC5 而言，如果我们没有定义 Abstract_base class 的 pure virtual destructor，虽然可以通过编译，但链接时会出现以下错误：

> error LINK2001: unresolved external symbol "public: virtual __thiscall
Abstract_base::~Abstract_base(void)"(??1Abstract_base&&UAE&XZ)

> @zl:我在 g++ 8.2.0 和 visual studio 2019 进行测试，在抽象类中不定义 pure virtual destructor，编译和链接都不会出错！

你可能会争辩说，难道对一个 pure virtual destructor 的调用操作，不应该在 “编译器扩展 derived class 的 destructor” 时压抑下来吗？不！class 设计者可能已经真的定义了一个 pure virtual destructor（就像上一个例子中定义了 Abstract_base::interface() 那样）。这样的设计是以 C++ 语言的一个保证为前提：继承体系中每一个 class object 的 destructor 都会被调用。所以编译器不能够压抑这个调用操作。

你也可能争辩说，难道编译器不应该有足够的数据“合成”（如果 class 设计者忘记定义或不知道要定义）一个 pure virtual destructor 的函数定义码？不！编译器的确没有足够知识，因为编译器对一个可执行文件采取 “分离编译模型” 之故。是的，开发环境可以提供一个设备，在链接时找出 pure virtual destructor 不存在的事实 体，然后重新激活编译器，赋予一个特殊指令（directive），以合成一个必要的函数实体；但是我不知道目前是否有任何编译器这么做。

一个比较好的替代方案就是，不要把 virtual destructor 声明为 pure。

### 虚拟规格的存在（Presence of a Virtual Specification）

如果你决定把 Abstract_base::mumble() 设计为一个virtual function，那将是一个糟糕的选择，因为其函数定义内容并不与类型有关，因而几乎不会被后继的 derived class 改写。此外，由于它的 non-virtual 函数实体是一个 inline 函数，如果常常被调用的话，效率上的报应是在不轻。

然而，编译器难道不能够经由分析，知道该函数只有一个实体存在于 class 层次体系之中？果然如此的话，难道它不能够把调用操作转换为一个静态调用操作（static invocatin），以允许该调用操作的 inline expansion？如果 class 层次体系陆续被加入新的 classes，带有这个函数的新实体，又当如何？是的，新的 class 会破坏优化！该函数现在必须被重新编译（或是产生第二个——也就是多态——实体，编译器将通过流程分析决定哪一个实体要被调用）。不过，函数可以以二进制形式存在于一个 library 中。欲挖掘出这样的相依性，有可能需要某种形式的 persistent program database 或 library manager。

一般而言，把所有的成员函数都声明为 virtual function，然后再靠编译器的优化操作把非必要的 virtual invocation 去除，并不是好的设计观念。

### 虚拟规格中const的存在

决定一个 virtual function 是否需要 const，似乎是件琐屑的事情。但当你真正面对一个 abstract base class 时，却不容易做决定。做这件事情，意味着得假设 subclass 实体可能被无穷次数地调用。不把函数声明为 const，意味着该函数不能够获得一个 const reference 或 const pointer。比较令人头大的是，声明一个函数为 const，然后才发现实际上其 derived instance 必须修改某一个 data member，我不知道有没有一直的解决方法，我的想法很简单，不再用 const 就是了。

### 重新考虑class的声明

由前面的讨论可知，重新定义 Abstract_base 如下，才是比较适当的一种假设：
```cpp
class Abstract_base{
public:
    virtual ~Abstract_base();       // 译注：不再是pure virtual
    virtual void interface() = 0;   // 译注：不再是const
    const char* mumble() const { return _mumble; }  // 译注：不再是virtual
protected:
    Abstract_base(char* pc = 0);    // 新增一个带有唯一参数的constructor
    char *_memble;
};
```

## 5.1 “无继承”情况下的对象构造

考虑下面这个程序片段：
```cpp
(1) Point global;
(2)
(3) Point foobar()
(4) {
(5)     Point local;
(6)     Point *heap = new Point;
(7)     *heap = local;
(8)     // ... stuff ...
(9)     delete heap;
(10)    return local;
(11) }
```
L1, L5, L6 表现出三种不同的对象产生方式：global 内存配置、local 内存配置和 heap 内存配置。L7 把一个 class object 指定给另一个，L10 设定返回值，L9 则明确地以 delete 运算符删除 heap object。

一个 object 的生命，是该 object 的一个执行期属性。local object 的生命从 L5 的定义开始，到 L10 为止。global object 的生命和整个程序的生命相同。heap object 的生命从它被 new 运算符配置出来开始，到它被 delete 运算符摧毁为止。

下面是 Point 的第一次声明，可以写成 C 程序。C++ Standard 说这是一种所谓的 Plain Of Data 的声明形式：
```c
typedef struct
{
    float x, y, z;
}Point;
```
如果我们以 C++ 来编译这段码，会发生什么事？观念上，编译器会为 Point 声明一个 trivial constructor、一个 trivial destructor、一个 trivial copy constructor，以及一个trivial copy assignment operator。但实际上，编译器会分析这个声明，并为它贴上 Plain Ol' Data 卷标。

当编译器遇到这样的定义：
```cpp
    Point glocal;
```
时，观念上 Point 的 trivial constructor 和 destrucotr 都会被产生并调用，constructor 在程序起始（startup）处被调用而 destructor 在程序的 exit() 处被调用（exit() 系由系统产生，放在 main() 结束之前）。然而，事实上那些 trivial members 要不是没被定义，就是没被调用，程序的行为一如它在 C 中的表现一样。

唔，只有一个小小的例外。在 C 之中，global 被视为一个 “临时性的定义”，因为它没有明确的初始化操作。一个 “临时性的定义” 可以在程序中产生多次。那些实例会被链接器折叠起来，只留下单独一个实体，放在程序 data segment 中一个 “特别保留给未初始化之 global objects 使用” 的空间。由于历史的缘故，这块空间被称为 BSS，这是 Block Started by Symbol 的缩写，是 IBM 704 assembler 的一个 pseudo-op。

C++ 并不支持 “临时性的定义”，这是因为 class 构造行为的隐含应用之故。虽然大家公认这个语言可以判断一个 class objects 或是一个 Plain Ol' Data，但似乎没有必要搞得那么复杂。因此，global 在 C++ 中被视为完全定义（它会阻止第二个或更多个定义）。C 和 C++ 的一个差异就在于，BSS data segment 在 C++ 中相对地不重要。C++ 的所有全局对象都被当做 “初始化过的数据” 来对待。

foobar() 函数中的 L5，有一个 Point object local，同样也是既没有被构造也没有被解构。当然啦，Point object local 如果没有先经过初始化，可能会成为一个潜在的程序臭虫——万一第一次使用它就需要其初值的话（如 L7）。至于 heap object 在 L6 的初始化操作：
```cpp
Point *heap = new Point;
```
会被转换为对 new 运算符（由 library 提供）的调用：
```cpp
Point *heap = __new(sizeof(Point));
```
再一次容我强调，并没有 default cosntructor 施行于 new 运算符所传回的 Point object 身上。L7 对此 object 有一个赋值（赋值，assign）操作，如果 local 曾被适当地初始化过，一切就没有问题：
```cpp
(7) *heap = local;
```
事实上这一行会产生编译警告如下：
> warning, line 7 : local is used before being initialized.

观念上，这样的指定操作会触发 trivial copy assignment operator 进行拷贝搬运操作。然而实际上此 object 是一个 Plain Ol' Data，所以赋值操作（assignment）将只是像 C 那样的纯粹位搬移操作。L9 执行一个 delete操作：
```cpp
(9) delete heap;
```
会被转换为对 delete运算符（由library提供）的调用：
```cpp
__delete(heap);
```
观念上，这样的操作会触发 Point 的 trivial destructor。但一如我们所见，destructor 要不是没有被产生就是没有被调用。最后，函数以传值（by value）的方式将 local 当做返回值传回，这在观念上会出发 trivial copy cosntructor，不过实际上 return 操作只是一个简单的位拷贝操作，因为对象是一个 Plain Ol' Data.

### 抽象数据类型（Abstract Data Type）

以下是 Point 的第二次声明，在 public 接口之下多了 private 数据，提供完整的封装性，但没有提供任何 virtual function:
```cpp
class Point{
public:
    Point(float x = 0.0, float y = 0.0, float z = 0.0)
        : _x(x), _y(y), _z(z) { }
    // no copy constructor, copy operator
    // or destructor defined ...

    // ...
private:
    float _x, _y, _z;
};
```
这个经过封装的 Point class，其大小并没有改变，还是三个连续的 float，是的，不论 private、public 存取层，或是 member function 的声明，都不会占用额外的对象空间。

我们并没有为 Point 定义一个 copy constructor 或 copy operator，因为默认的位语意（default bitwise semantics。译注：请看第 2 章，#51 页）已经足够。我们也不需要提供一个 destructor，因为程序默认的内存管理方法也已足够。

对于一个 global 实体：
```cpp
Point global;   // 实施Point::Point(0.0, 0.0, 0.0);
```
现在有了 default constructor 作用于其上。由于 global 被定义在全局范畴中，其初始化操作将延迟到程序激活（starup）时才开始（6.1 节对此有详细讨论）。

如果要对 class 中的所有成员都设定常量初值，那么给与一个 explicit initiaization list 会比较高效（比起意义相同的 constructor 的 inline expansion 而言）。甚至在 local scope 中也是如此（不过在 local scope 中这可能稍微有些不够直观。你可以在 5.4 节看到一些讨论）。举个例子，考虑下面这个程序片段：
```cpp
void mumble()
{
    Point local1 = { 1.0, 1.0, 1.0 };
                    // 译注：原书为Point1，应为笔误
    Point local2;   // 译注：原书为Point2，应为笔误

    // 相当于一个inline expansion
    // explicit initialization会稍微快一些
    local2._x = 1.0
    local2._y = 1.0
    local2._z = 1.0
}
```
local1 的初始化操作会比 local2 的高效。这是因为当函数的 activation record 被放进程序堆栈时，上述 initialization list 中的常量就可以被放进 local1 内存中了。

Explicit initialization list 带来三项缺点：
1. 只有当 class members 都是public时，此法才奏效。
2. 只能指定常量，因为它们在编译时期就可以被评估求值（evaluated）。
3. 由于编译器并没有自动施行之，所以初始化行为的失败可能性会比较高一些。

好了，explicit initialization list 所带来的效率优点能够补偿其软件工程上的缺点吗？一般而言，答案是 no。然而在某些特殊情况下有不一样。例如，或许你以手工打造了一些巨大的数据结构如调色盘（color palette），或是你正要把一堆常量数据（比如以 Alias 或 SoftImage 软件产生出来的复杂集合模型中的控制点和节点）倾倒给程序，那么 explicit initialization list 的效率会比 inline constructor 好得多，特别是对全局对象（global object）而言。

在编译器层面，会有一个优化机制用来识别 inline constructor，后者简单地提供一个 member-by-member 的常量指定操作。然后编译器会抽取出那些值，并且对待它们就好像是 explicit initialization list 所供应的一样，而不会把 constructor 扩展成为一系列的 assignment 指令。

于是，local Point object 的定义：
```cpp
{
    Point local;
    // ...
}
```
现在被附加上 default Point constructor 的 inline expansion：
```cpp
{
    // inline expansion of default constructor
    Point local;
    lcoal._x = 0.0; local._y = 0.0; local._z = 0.0;
    // ...
}
```

L6 配置出一个 heap Point object：
```cpp
(6) Point *heap = new Point;
```
现在则被附加一个 “对 default Point constructor 的有条件调用操作”：
```cpp
// C++伪码
Point *headp = __new(sizeof(Point))
if(heap != 0)
    heap->Point::Point();
```
然后才又被编译器进行 inline expansion 操作。至于把 heap 指针指向 local object：
```cpp
(7) *heap = local;
```
则保持着简单的位拷贝操作。以传值方式传回 local object，情况也是一样：
```cpp
(10) return local;
```
L9 删除 heap 所指之对象：
```cpp
(9) delete heap;
```
该操作并不会导致 destructor 被调用，因为我们并没有明确地提供一个 destructor 函数实体。

观念上，我们的 Point class 有一个相关的 default copy constructor、copy operator 和 destructor，然而它们都是无关痛痒的（trivial），而且编译器实际上根本没有产生它们。

### 为继承做准备

我们的第三个 Point 声明，将为 “继承性质” 以及某些操作的动态决议（dynamic resolution）做准备。当前我们限制对 z 成员进行存取操作：
```cpp
class Point{
public:
    Point(float x = 0.0, float y = 0.0)
        : _x(x), _y(y) { }
    // no destructor, copy constructor, or
    // copy operator defined ...

    virtual float z();
    // ...
protected:
    float _x, _y;
};
```

再一次容我强调，我并没有定义一个 copy constructor、copy operator、destructor。我们的所有 members 都以数值来储存，因此在程序层面的默认语意之下，行为良好。某些人可能会争辩说，virtual functions 的导入应该总是附带着一个 virtual destructor 的声明，但那么做在这个例子中对我们并无好处。

virtual functions 的引入促使每一个 Point object 拥有一个 virtual table pointer。这个指针提供给我们 virtual 接口的弹性，代价是：每一个 object 需要额外的一个 word 空间。有什么重大影响吗？视应用个情况而定！在 3D 模型应用领域中，如果你要表现一个复杂的集合形状，有着 60 个 NURB 表面，每个表面有 512 个控制点，那么每个 object 多负担 4 个 bytes 将导致大约 200,000 个 bytes 的额外负担。这可能有意义，也可能没有意义，必须视它对多态（polymorphism）设计所带来的实际效益的比例而定，只有在完成之后，你才能评估要不要避免之。

除了每一个 class object 多负担一个 vptr 之外，virtual function 的引入也引发编译器对于我们的 Point class 产生膨胀作用：

1. 我们所定义的 constructor 被附加了一些码，以便将 vptr 初始化。这些码必须被附加在任何 base class constructors 的调用之后，但必须在任何由使用者（程序员）供应的码之前。例如，下面就是可能的附加结果（以 Point 为例）：
```cpp
// C++ 伪码：内部膨胀
Point*
Point::Point(Point* this,
                float x, float y)
            : _x(x), _y(y)
{
    // 设定object的virtual table pointer (vptr)
    this->_vptr_Point = __vtbl__Point;

    // 扩展member initialization list
    this->_x = x;
    this->_y = y;

    // 传回this对象
    return this;
}
```
2. 合成一个 copy constructor 和一个 copy assignment operator，而且其操作不再是 trivial（但 implicit destructor 仍然是 trivial）。如果一个 Point object 被初始化或一一个 derived class object 赋值，那么以位为基础（bitwise）的操作可能给 vptr 带来非法设定。
```cpp
// C++伪码
// copy constructor的内部合成
inline Point*
Point::Point(Point* this, const Point &rhs)
{
    // 设定object的virtual table pointer(vptr)
    this->__vptr_Point = __vtbl__Point;

    // 将rhs坐标的连续位拷贝到this对象，
    // 或是经由member assignment提供一个member······

    return this;
}
```
编译器在优化状态下可能会把 object 的连续内容拷贝到另一个 object 身上，而不会实现一个精确地 “以成员为基础（memberwise。译注：请参考第 2 章，#49 页）” 的赋值操作。C++ Standard 要求编译器尽量延迟 nontrivial members 的实际合成操作，直到真正遇到其使用场合为止。

为了方便起见，我把 foobar() 再次列于此：
```cpp
(1) Point global;
(2)
(3) Point foobar()
(4) {
(5)     Point local;
(6)     Point *heap = new Point;
(7)     *heap = local;
(8)     // ... stuff ...
(9)     delete heap;
(10)    return local;
(11) }
```
L1 的 global 初始化操作、L6 的 heap 初始化操作以及 L9 的 heap 删除操作，都还是和稍早的 Point 版本相同，然而 L7 的 memberwise 赋值操作：
```cpp
*heap = local;
```
很有可能触发 copy assignment operator 的合成，及其调用操作的一个 inline expansion（内联扩展）：以 this 取代 heap 而以 rhs 取代 local。

最戏剧性的冲击发生在以传值方式传回 local 的那一行 (L10)。由于 copy constructor 的出现，foobar() 很有可能被转化为下面这样（2.3 节曾有过详细的讨论）：
```cpp
// C++ 伪码：foobar()的转化
// 用以支持copy constructor

Point foobar(Point& __result)
{
    Point local;
    local.Point::Point(0.0, 0.0);

    // heap的部分与前相同······

    // copy constructor的应用
    __result.Point::Point(local);

    // local对象的destructor将在这里执行
    // 调用Point定义的destructor；
    // local.Point::~Point();

    return;
}
```
如果支持 named return value(NRV) 优化，这个函数会进一步被转化为：
```cpp
// C++伪码：foobar()的转化，
// 以支持named return value（NRV）优化

Point foobar(Point& __result)
{
    __result.Point::Point(0.0, 0.0);

    // heap的部分与前相同······

    return;
}
```

一般而言，如果你的设计之中有许多函数都需要以传值方式（by value）传回一个 local class object，例如像如下形式的一个算术运算：
```cpp
T operator+(const T&, const T&)
{
    T result;
    // ······真正的工作在此
    return result;
}
```
那么提供一个 copy constructor 就比较合理——甚至即使 default memberwise 语意已经足够。它的出现会触发 NRV 优化。然而，就像我在前一个例子中有所展现的那样，NRV 优化后将不再需要调用 copy constructor，因为运算结果已经被直接置于 “将被传回的object” 体内了。

## 5.2 继承体系下的对象构造

当我们定义一个 object 如下：
```cpp
T object;
```
时，实际上会发生什么事情呢？如果 T 有一个 constructor（不论是由 user 提供或是由编译器合成），它会被调用。这很明显，比较不明显的是，constructor 的调用真正伴随了什么？

Constructor 可能内带大量的隐藏码，因为编译器会扩充每一个 constructor，扩充程度视 class T 的继承体系而定。一般而言编译器所做的扩充操作大约如下:
1. 记录在 member initialization list 中的 data members 初始化操作会被放进 constructor 的函数本身，并以 members 的声明顺序为顺序。
2. 如果一个 member 并没有出现在 member initialization list 之中，但它有一个 default constructor，那么该 default constructor 必须被调用。
3. 在那之前，如果 class object 有 virtual table pointer(s)，它（们）必须被设定初值，指向适当的 virtual table(s).
4. 在那之前，所有上一层的 base class constructors 必须被调用，以 base class 的声明顺序为顺序（与 member initialization list 中的顺序没关联）：
（1）如果 base class 被列于 member initializatin list 中，那么任何明确指定的参数都应该传递进去。
（2）如果 base class 没有被列于 member initialization list 中，而它有 default constructor（或 default memberwise copy costructor），那么就调用之。
（3）如果 base class 是多重继承下的第二或后继的 base class，那么 this 指针必须有所调整。
5. 在那之前，所有 virtual base class constructors 必须被调用，从左到右，从最深到最浅：
（1）如果 class 被列于 member initialization list 中，那么如果有任何明确指定的参数，都应该传递进去。若没有列于 list 之中，而 class
有一个 default constructor，也应该调用之。
（2）此外，class 中的每一个 virtual base class subobject 的偏移量（offset）必须在执行期可被存取。
（3）如果 class object 是最底层（most-derived）的 class，其 constructors 可能被调用；某些用以支持这个行为的机制必须被放进来。

在这一节中，我要从 “C++ 语言对 classes 所保证的语意” 这个角度来探讨 constructors 扩充的必要性。我再次以 Point 为例，并为它增加一个 copy constructor、一个 copy operator、一个 virtual destructor 如下：
```cpp
class Point{
public:
    Point(float x = 0.0, float y = 0.0);
    Point(const Point&);                // 译注：copy constructor
    Point& operator=(const Point&);     // 译注：copy assignment operaotor

    virtual ~Point();;                  // 译注：virtual destructor
    // ...
protected:
    float _x, _y;
};
```
在我开始介绍并一步步走过以 Point 为根源的继承体系之前，我要带你很快地看看 Line class 的声明和扩充结果，它由 _begin 和 _end 两个点构成：
```cpp
class Line{
    Point _begin, _end;
public:
    Line(float = 0.0, float = 0.0, float = 0.0, float = 0.0);
    Line(const Point&, const Point&);

    draw();
    // ...
};
```
每一个 explicit constructor 都会被扩充以调用其两个 member class objects 的  constructor。如果我们定义 constructor 如下：
```cpp
Line::Line(const Point& begin, const Point& end)
    : _end(end), _begin(begin)
{ }
```
它会被编译器扩充并转换为：
```cpp
// C++伪码：Line constructor的扩充
Line*
Line::Line(Line* this,
            const Point& begin, const Point& end)
{
    this->_begin.Point::Point(begin);
    this->_end.Point::Point(end);
    return this;
}
```
由于 Point 声明了一个 copy constructor、一个 copy opaertor，以及一个 destructor（本例为 virtual），所以 Line class 的 implicit copy constructor、copy operator 和 destructor 都将有实际功能（nontrivial)。

当程序员写下：
```cpp
Line a;
```
时，implicit Line destructor 会被合成出来（如果 Line 派生自 Point，那么合成出来的 destructor 将会是 virtual。然而由于 Line 只是内带 Point objects 而非继承自 Point，所以被合成出来的 destructor 只是 nontrivial 而已）。在其中，它的member class objects 的 destructors 会被调用（以其构造的相反顺序）：
```cpp
// C++伪码：合成出来的Line destructor
inline void
Line::~Line(Line *this)
{
    this->_end.Point::~Point();
    this->_begin.Point::~Point();
}
```
当然，如果 Point destructor 是 inline 函数，那么每一个调用操作会在调用地点被扩展开来。请注意，虽然 Point destructor 是 virtual，但其调用操作（在containint class destructor 之中）会被静态地决议出来（resolved statically）。

类似的道理，当一个程序员写下：
```cpp
Line b = a;
```
时，implicit Line copy constructor 会被合成出来，成为一个 inline public member。

最后，当程序员写下：
```cpp
a = b;
```
时，implicit copy assignment operator 会被合成出来，成为一个 inline public member。

最近当我改写 cfront 时，我注意到在产生 copy opaertor 的时候，并没有用如下的条件句筛选：
```cpp
if(this == &rhs) return *this;
```
于是 cfront 会做多余的拷贝操作，像这样：
```cpp
Line *p1 = &a;
Line *p2 = Ya;
*p1 = *p2
```
我发现并不是只要 cfront 才如此。Borland 也缺少这项筛选，而我怀疑大部分编译器也都如此。在一个由编译器合成而来的 copy operator 中，上述的重复操作虽然安全却累赘，因为并没有伴随任何的资源释放行动。在一个由程序员供应的 copy operator 中忘记检查自我指派（赋值）操作是否失败，是新手极易陷入的一项错误，例如：
```cpp
// 使用者供应的copy assignment operator
// 忘记提供一个自我拷贝时的筛选

String&
String::operator=(const String &rhs){
    // 这里需要筛选（在释放资源之前）
    delete[] str;
    str = new char[strlen(rhs.str) + 1];
}
```
有多次我想为 cfront 加上一个警告消息，说 “在一个 copy operator 之中，面对自我拷贝一个筛选操作；但却有一个 delete operator 对应某个 member 操作”。这项意图并没有实现，但我仍然认为像这样的一个警告信息对程序员是有帮助的。

### 虚拟继承（Virtual Inheritance）

考虑下面这个虚拟继承（继承自我们的 Point):
```cpp
class Point3d : public virtual Point{
public:
    Point3d(float x = 0.0, float y = 0.0, float z = 0.0)
        : Point(x, y), _z(z) { }
    Point3d(const Point3d& rhs)
        : Point(rhs), _z(rhs._z) { }
    ~Point3d();
    Point3d& operator=(const Point3d&);

    virtual float z() { return _z; }
    // ...
protected:
    float _z;
};
```
传统的 “constructor 扩充现象” 并没有用，这是因为 virtual base class 的 “共享性” 之故：
```cpp
// C++伪码：
// 不合法的constructor扩充内容
Point3d*
Point3d::Point3d(Point3d* this,
                float x, float y, float z)
{
    this->Point::Point(x, y);
    this->__vptr_Point3d = __vtbl_Point3d;
    this->__vptr_Point3d_Point =
        __vtbl_Point3d_Point;
    this->_z = rhs._z;
    return this;
}
```

你看得出上面的 Point3d constructor 扩充内容有什么错误码？

试着现象以下三种类派生情况：
```cpp
class Vertex : virtual public Point { ... };
class Vertex3d : public Point3d, public Vertex { ... };
class PVertex : public Vertex3d { ... };
```
Vertex 的 constructor 必须也调用 Point 的 constructor。然而，当 Point3d 和 Vertex 同为 Vertex3d 的 subobjects 时，它们对 Point constructor 的调用操作一定不可以发生，取而代之的是，作为一个最底层的 class，Vertex3d 有责任将 Point 初始化。而更往后（往下）的继承，则是 PVertex（不再是 Vertex3d）来负责完成 “被共享之Point subobject” 的构造。

传统的策略如果要支持 “好，现在将 virtual base class 初始化······啊，现在不要······”，会导致 constructor 中有更多的扩充内容，用以指示 virtual base class constructor 应不应该被调用。constructor 的函数本身因而必须条件式地测试传进来的参数，然后决定调用或不调用相关的 virtual base class constructors.下面就是Point3d 的 constructor 扩充内容：
```cpp
// C++伪码
// 在virtual base class情况下的constructor扩充内容
Point3d*
Point3d::Point3d(Point3d *this, bool __most_derived,
                float x, float y, float z)
{
    if(__most_derived != false)
        this->Point::Point(x, y);

    this->__vptr_Point3d = __vtbl_Point3d;
    this->__vptr_Point3d_Point =
        __vtbl_Point3d_Point;
    this->_z = rhs._z;
    return this;
}
```
在更深层的继承情况下，例如 Vertex3d，当调用 Point3d 和 Vertex 的 constructor时，总是会把 __most_derived 参数设为 false，于是就压制了两个 constructors 中对 Point constructor 的调用操作。
```cpp
// C++伪码
// 在virtual base class情况下的constructor扩充内容
Vertex3d*
Vertex3d::Vertex3d(Vertex3d *this, bool __most_derived,
                float x, float y, float z)
{
    if(__most_derived != false)
        this->Point::Point(x, y);

    // 调用上一层base classes
    // 设定__most_derived为false

    this->Point3d::Point3d(false, x, y, z);
    this->Vertex::Vertex(false, x, y);

    // 设定vptrs
    // 安插user code

    return this;
}
```
这样的策略得以保持语意的正确无误。举个例子，当我们定义：
```cpp
Point3d origin;
```
时，Point3d constructor 可以正确地调用其 Point virtual base class subobject。而当我们定义：
```cpp
Vertex3d cv;
```
时，Vertex3d constructor 正确地调用 Point constructor。Point3d 和 Vertex 的constructor 会做每一件该做的事情——对 Point 的调用操作除外。如果这个行为是正确的，那么什么事错误的呢？

我想许多人已经注意到了某种状态。在这种状态中，“virtual base class constructors 的被调用” 有着明确的定义：只有当一个完整的 class object 被定义出来（例如 origin）时，它才会被调用；如果 object 只是某个完整 object 的 subobject，它就不会被调用。

以这一点为杠杆，我么可以产生更有效率的 constructors。某些新近的编译器把每个 constructor 分裂为二，一个针对完整的 object，另一个针对 subobject。“完整object” 版无条件地调用 virtual base constructors，设定所有的 vptrs 等等。“subobject” 版则不调用 virtual base constructors，也可能不设定 vptrs 等等。我将在下一节讨论对 vptr 的设定。constructor 的分裂可带来程序速度的提升，不幸的是我并没有真正用过这一类编译器，所以没有测试数据可以提供给各位。然而在 Foundation项目中，Rob Murray 完成了 cfront 的 C output 优化，可省略掉非必要的条件测试，并设定 vptr，并且他记录到一份颇为可观的速度提升结果。

### vptr初始化语意学（The Semantics of the vptr Initialization）

当我们定义一个 PVertex object 时，constructor 的调用顺序是：
```cpp
Point(x, y);
Point3d(x, y, z);
Vertex(x, y, z);
Vertex3d(x, y, z);
PVertex(x, y, z);
```
假设这个继承体系中的每一个 class 都定义了一个 virtual function size()，该函数负责传回 class 的大小。如果我们写：
```cpp
PVertex pv;
Point3d p3d;

Point *pr = &pv;
```
那么这个调用操作：
```cpp
pt->size();
```
将传回 PVertex 的大小，而：
```cpp
pt = &p3d;
pt->size();
```
将传回 Point3d 的大小。

更进一步，我们假设这个继承体系中的每一个 constructors 内带一个调用操作，像这样：
```cpp
Point3d::Point3d(float , float y, float z)
    : _x(x), _y(y), _z(z)
{
    if (spyOn)
        cerr << "Within Point3d::Point3d()"
            << "size: " << size() << endl;
}
```
当我们定义 PVertex object 时，前述的五个 constructor 会如何？每一次 size() 调用会被决议为 PVertex::size() 吗（毕竟那是我们正在构造的东西）？或者每次调用会被决议为 “当前正在执行之 constructor 所对应之 class” 的 size() 函数实体？

C++ 语言规则告诉我们，在 Point3d constructor 中调用的 size() 函数，必须被决议为 Point3d::size() 而不是 PVertex::size()。更一般地，在一个 class(本例为 Point3d) 来调用一个 virtual function，其函数实体应该是在此 class（本例为Point3d）中有作用的那个。由于各种 constructor 的调用顺序之故，上述情况是必要的。

constructor 的调用顺序是：由根源而末端（bottom up），由内而外（inside out）。当 base class constructor 执行时，derived 实体还没有被构造出来。在 PVertex constructor 执行完毕之前，PVertex 并不是一个完整的对象；Point3d constructor 执行之后，只有 Point3d subobject 构造完毕。

这意味着，当每一个 PVertex base class constructor 被调用时，编译系统必须保证有适当的 size() 函数实体被调用。怎样才能办到这一点呢？

如果调用操作限制必须在 constructor (或 destructor) 中直接调用，那么答案十分明显：将每一个调用操作以静态方式决议之，千万不要用到虚拟机制。如果是在 Point3d constructor 中，就明确调用 Point3d::size()。

然而如果 size() 之中又调用一个 virtual function，会发生什么事情呢？这种情况下，这个调用也必须决议为 Point3d 的函数实体。而在其他情况下，这个调用时纯正的 virtual，必须经由虚拟机制来决定其归向。也就是说，虚拟机制本身必须知道是否这个调用源自于一个 constructor 之中。

另一个我们可以采取的方法是，在 constructor（或 destructor）内设立一个标志，说：“嗨，不，这次请以静态方式来决议”。然后我们就可以以标志值作为判断依据，产生条件式的调用操作。

这的确可行，虽然感觉起来优点不够优雅和有效率。就算是一种 “hack” 行为吧！我们甚至可以写一个批注来为自己开脱：
```cpp
// yuck!!! fix the language semantics!
```
这个解决方法感觉起来比较像是我们的第一个设计策略失败后的一个对策，而不是釜底抽薪的方法。根本的解决之道是，在执行一个 constructor 时，必须限制一组 virtual functions 候选名单。

想一想，什么事决定一个 class 的 virtual functions 名单的关键？答案是 virtual table。Virtual table 如何被处理？答案是通过 vptr。所以，为了控制一个 class 中有所作用的函数，编译系统只要简单地控制住 vptr 的初始化操作和谁当操作即可。当然，设定 vptr 是编译器的责任，任何程序员都不必操心此事。

vptr 初始化操作应该如何处理？本质而言，就其这得视 vptr 在 constructor 之中 “应该在何时被初始化” 而定。我们有三种选择：
1. 在任何操作之前。
2. 在 base class constructor 调用操作之后，但是在程序员提供的码或是 “member 
initialization list 中所列的 members 初始化操作”之前。
3. 在每一件事情发生之后。

答案是 2. 另两种选择没什么价值。如果你不相信，请在 1 或 3 策略下试试看。策略 2 解决了 “在 class 中限制一组 virtual functions 名单” 的问题。如果每一个 constructor 都一直等待到其 base class constructor 执行完毕之后才设定其对象的 vptr，那么每次他都能够调用正确的 virtual function 实体。

令每一个 base class constructor 设定其对象的 vptr，使它指向相关的 virtual table 之后，构造中的对象就可以严格而正确地变成 “构造过程中所幻化出来的每一个 class” 的对象。也就是说，一个 PVertex 对象会先形成一个 Point 对象、一个 Point3d 对象、一个 Vertex 对象、一个 Vertex3d 对象，然后才成为一个 PVertex 对象。在每一个 base class constructor 中，对象可以与 constructor’s class 的完整对象作比较。对于对象而言，“个体发生学” 概括了 “系统发生学”。constructor 的执行算法通常如下：
1. 在 derived class constructor 中，“所有 virtual base classes” 及 “上一层base class” 的 constructors 被调用。
2. 上述完成之后，对象的 vptr(s) 被初始化，指向相关的 virtual table(s).
3. 如果有 member initialization list 的话，将在 constructor 体内扩展开来。这必须在 vptr 被设定之后才进行，以免有一个 virtual member function 被调用。
4. 最后，执行程序员所提供的码。
例如，已知下面这个由程序员定义的 PVertex constructor：
```cpp
PVertex::PVertex(float x, float y, float z)
    : _next(0), Vertex3d(x, y, z),
    Point(x, y)
{
    if(spyOn)
        cerr << "With PVertex::PVertex()"
                            // 译注：原书为Point3d::Point3d(),应为笔误
            << "size: " << size() << endl;
}
```
它很可能被扩展为：
```cpp
// C++伪码
// PVertex constructor的扩展结果
PVertex*
PVertex::PVertex(PVertex* this,
                bool __most__derived,
                float x, float y, float z)
{
    // 条件式地调用virtual base constructor
    if(__most__derived != false)
        this->Point::Point(x, y);

    // 无条件地调用上一层base
    this->Vertex3d::Vertex3d(x, y, z);

    // 将相关的vptr初始化
    this->__vptr_PVertex = __vtbl_PVertex;
    this->__vptr_Point__PVertex = 
          __vtbl_Point__PVertex;
    // 程序员所写的码
    if(spyOn)
        cerr << "Within PVertex::PVertex()" 
                            // 译注：原书为Point3d::Point3d()，应为笔误
        // 经由虚拟机制调用
        << (*this->__vptr__PVertex[3].faddr)(this)
        << endl;
    // 传回被构造的对象
    return this;
}
```
这就完美地解决了我们所说的有关限制虚拟机制的问题。但是，这真是一个完美的解答吗？假设我们的 Point constructor 定义为：
```cpp
Point::Point(float x, float y)
    : _x(x), _y(y) { }
```
我们的 Point3d constructor 定义为：
```cpp
Point3d::Point3d(float x, float y, float z)
    : Point(x, y), _z(z) { }
```
更进一步，假设我们的 Vertex 和 Vertex3d cosntructor 有类似的定义。你是否能够看出我们的解决方法并不完美——即使我们完美地解决了我们的问题？

下面是 vptr 必须被设定的两种情况：
1. 当一个完整的对象被构造起来时。如果我们声明一个 Point 对象，Point constructor 必须设定其 vptr。
2. 当一个 subobject constructor 调用了一个 virtual function（不论是直接调用或间接调用）时。

如果我们声明一个 PVertex 对象，然后由于我们对其 base class constructors 的最新定义，其 vptr 将不再需要在每一个 base class constructor 中被设定。解决之道是把 constructor 分裂为一个完整的 object 实体和一个 subobject 实体。在 subobject 实体中，vptr 的设定可以省略（如果可能的话）。

知道了这些之后，你应该能够回答下面的问题了吧：在 class 的 constructor 的 member initialization list 中调用该 class 的一个虚拟函数，安全吗？就实际而言，将该函数运行于其 class’s data member 的初始化行动中，总是安全的。这是因为，正如我们所见，vptr 保证能够在 member initialization list 被扩展之前，由编译器正确设定好。但是在语意上这可能是不安全的，因为函数本身可能还得以来未被设立初值的 members。所以我并不推荐这种做法。然而，从 vptr 的整体角度来看，这是安全的。

何时需要供应参数给一个 base class constructor？这种情况下在 “class 的constructor 的 member initialization list 中” 调用该 class 的虚拟函数，仍然安全吗？不！此时 vptr 若不是尚未被设定好，就是被设定指向错误的 class。更进一步地，该函数所存取的任何 class‘s data members 一定还没被初始化。

## 5.3 对象复制语意学（Object Copy Semantics）

当我们设计一个 class，并以一个 class object 指定给另一个 class object 时，我们有三种选择：
1. 什么都不做，因此得以实施默认行为。
2. 提供一个 explicit copy assignment operator。
3. 明确地拒绝把一个 class object 指定给另一个 class object。

如果要选择第 3 点，不准将一个 class object 指定给另一个 class object，那么只要将 copy assignment operator 声明为 private，并且不提供其定义即可。把它设为private，我们就不再允许于任何地点（除了在 member functions 以及此 class 的friends 之中）进行赋值（assign）操作。不提供其函数定义，则一旦某个 member function 或 friend 企图影响一份拷贝，程序在链接时就会失败。一般认为这和链接器的性质有关（也就是说并不属于语言本身的性质），所以不是很令人满意。

> @zl: C++11提供delete的方式来explicitly删除成员函数，例如：
Point(const Point& rhs) = delete;

在这一节，我要验证 copy assignment operator 的语意，以及它们如何被模塑出来。再一次容易强调，我利用 Point class 来帮助讨论：
```cpp
class Point{
public:
    Point(float x  = 0.0, float y = 0.0);
    // ... (没有virtual function)
protected:
    float _x, _y;
};
```
没有什么理由需要禁止拷贝一个 Point object。因此问题就变成了：默认行为是否足够？如果我们要支持的只是一个简单的拷贝操作，那么默认行为不但足够而且有效率，我们没有理由再自己提供一个 copy assignment operator。

只有在默认行为所导致的语意不安全或不正确时，我们才需要设计一个 copy assignment operator（关于 memberwise copy 及其潜在陷阱，[lipp91c] 有完整的讨论）。好，默认的 memberwise copy 行为对于我们的 Point object 不安全码？不正确码？不，由于坐标都内带数值，所以不会发生 “别名话（aliasing）” 或 “内存泄漏（memory leak）”。如果我们自己提供一个 copy assignment operator，程序反倒会执行的比较慢。

如果我们不对 Point 供应一个 copy assignment operaotor，而光是依赖默认的 memberwise copy，编译器会产生出一个实体吗？这个答案和 copy constructor 的情况一样：实际上不会！由于此 class 已经有了 bitwise copy 语意，所以 implicit copy assignment operaotor 被视为毫无用处，也根本不会被合成出来。

一个 class 对于默认的 copy assignment operator，在以下情况不会表现出 bitwise copy语意：
1. 当class内带一个 member object，而其 class 有一个 copy assignment operator时。
2. 当一个 class 的 base class 有一个 copy assignment operator 时。
3. 当一个 class 声明了任何 virtual functions（我们一定不能够拷贝右端 class object 的 vptr 地址，因为它可能是一个 derived class object）。
4. 当 class 继承自一个 virtual base class（不论此 base class 有没有 copy operator）时。

C++ Standard 上说 copy assignment operators 并不会表示 bitwise copy semantics 是 nontrivial。实际上，只有 nontrivial instances 才会被合成出来。

于是，对于我们的 Point class，这样的赋值（assign)操作：
```cpp 
Point a, b;
...
a = b;
```
由 bitwise copy 完成，把 Point b 拷贝到 Point a，其间并没有 copy assignment operator 被调用。从语意上或从效率上考虑，这都是我们所需要的。注意，我们还是可能提供一个 copy constructor，为的是把 name return value（NRV）优化打开。copy constructor 的出现不应该让我们认定也一定要提供一个 copy assignment operaotr。

现在我要导入一个 copy assignment operator，用以说明该 operator 在继承之下的行为：
```cpp
inline
Point&
Point::operator=(const Point& p)
{
    _x = p._x;
    _y = p._y;
    // 译注：原书有误，这里缺少return *this;
}
```
现在派生一个 Point3d class（请注意是虚拟继承）：
```cpp
class Point3d : virtual public Point{
public:
    Point3d(float x = 0.0, float = 0.0, float z = 0.0);
    // ...
protected:
    float _z;
};
```
如果我们没有为 Point3d 定义一个 copy assignment operator，编译器就必须合成一个（因为前述的第二项和第四项理由）。合成而得的东西可能看起来像这样：
```cpp
// C++伪码：被合成de copy assignment operator
inline Point3d&
Point3d::operator=(Point3d* const this, const Point3d &p)
{
    // 调用 base class的函数实体
    this->Point::operator=(p);

    // memberwise copy the derived class members
    _z = p._z;
    return *this;
}
```
copy assignment operator 有一个非正交性情况（nonorthogonal aspect，意指不够理想、不够严谨的情况），那就它缺乏一个 member assignment list（也就是平行于member initialization list 的东西）。因此我们不能够写：
```cpp
// C++伪码，以下性质并不支持
inline Point3d&
Point3d::operator=(const Point3d &p3d)
    : Point(p3d), z(p3d._z)
    { }
```
我们必须写成以下两种形式，才能调用 base class 的 copy assignment operator:
```cpp
Point::operator=(p3d);
```
或
```cpp
(*(Point*)this) = p3d;
````
缺少 copy assignment list，看来或许是一件小事，但如果没有它，编译器一般而言就没有办法压抑上一层 base class 的 copy operator 被调用。例如，下面是个 Vertex copy operator，其中 Vertex 也是虚拟继承自 Point:
```cpp
// class Vertex : virtual public Point
intline Vertex&
Vertex::operator=(const Vertex &v)
{
    this->Point::operator=(v);
    _next = v._next;
    return *this;
}
```
现在让我们从 Point3d 和 Vertex 中派生出 Vertex3d，下面是 Vertex3d 的 copy assignment operator：
```cpp
inline Vertex3d&
Vertex3d::operator=(const Vertex3d &v)
{
    this->Point::operator=(v);
    this->Point3d::operator=(v);
    this->Vertex::operator=(v);
    ...
}
```
编译器如何能够在 Point3d 和 Vertex 的 copy assignment operators 中压抑 Point的 copy assignment operators 呢？编译器不能够重复传统的 constructor 解决方案（附加上额外的参数）。这是因为，和 constructor 以及 destructor不同的是，“取copy assignment operator地址” 的操作时合法的。因此，下面这个例子是毫无瑕疵的合法程序代码（虽然它也毫无瑕疵地推翻了我们希望把 copy assignment operator 做得更灵巧的企图）：
```cpp
typedef Point3d& (Point3d::*pmfPoint3d)(const Point3d&);

pmfPoint3d pmf = &Point3d::operator=;
(x.*pmf)(x);
```
然而我们无法支持它，我们仍然需要根据其独特的继承体系，安插任何可能数目的参数给 copy assignment operator。这一点在我们支持由 class objects（内带 virtual base classes）所组成的数组的配置操作时，也是证明是很有问题的（请看 6.2 节的讨论）。

另一个方法是，编译器可能为 copy assignment operator 产生分化函数（split functions），以支持这个 class 成为 most-derivd class 或成为中间的 base class。如果 copy assignment operator 被编译器产生的话，那么 “split function解决方案” 可以说是定义明确。但如果它是被 class 设计者所完成的，那就不能算是定义明确。例如，一个人如何分化像下面这样的函数呢（特别是当 init_bases() 为 virtual时）：
```cpp
inline Vertex3d&
Vertex3d::operator=(const Vertex3d &v)
{
    init_bases(v);
    ...
}
```
事实上，copy assignment operator 在虚拟继承情况下行为不佳，需要小心地设计和说明。许多编译器甚至并不尝试取得正确的语意，它们在每一个中间（调停用）的 copy assignment operator 中调用每一个 base class instance，于是造成 virtual ase class copy assignment operator 的多个实体被调用。我知道 Cfront、Edison Design Group 的前端处理器、Barland C++ 4.5 以及 Symantec 最新版 C++ 编译器都这么做。我猜想你的编译器也是如此。C++ Standard 是怎么说这件事的呢？

```
我们并没有规定那些代表 virtual base class 的 subobjects 是否该被 “隐喻定义（implicitly defined）的 copy assignment operator” 指派（赋值，assign）内容一次以上。（C++ Standard, Section 12.8）
```

如果使用一个以语言为基础的解决方法，那么应该为 copy assgnment operator 提供一个附加的 “member copy list”。简单地说，任何解决方案如果是以程序操作为基础，将导致较高的复杂度和较大的错误倾向。一般公认，这是语言的一个弱点，也是一个人应该小心检验其程序代码的地方（当他使用 virtual base classes 时）。

有一种方法可以保证 most-derived class 会引发（完成）virtual base class subobject 的 copy 行为，那就是在 derived class 的 copy assignment operaotr 函数实体的最后，明确调用哪个 operator，像这样：
```cpp
inline Vertex3d&
Vertex3d::operator=(const Vertex3d &v)
{
    this->Point3d::operator=(v);
    this->Vertex::operator=(v);
    // must place this last if your compiler does
    // not suppress intermediate class invocations
    this->Point::operator=(v);
    ...
}
```
这并不能够省略 subobject 的多重拷贝，但却可以保证语意正确。另一个解决方案要求把 virtual subobject 拷贝到一个分离的函数中，并根据 call path 条件化地调用它。

我建议尽可能不要允许一个 virtual base class 的拷贝操作。我甚至提供一个比较奇怪的建议：不要再任何 virtual base class 中声明数据。

## 5.4 对象的功能（Object Efficiency）

在以下的效率测试中，对象构造和拷贝所需的成本是以 Point3d class 声明为基准，以简单形式逐渐到复杂形式，包括 Plain Ol's Data、抽象数据类型（Abstract Data Type, ADT）、单一继承、多重继承、虚拟继承。以下函数是测试的主角：
```cpp
Point3d lots_of_copies(Point3d a, Point3d b)
{
    Point3d pC = a;

    pC = b; // (1)
    b = a;  // (2)
    return pC;
}
```
它内带四个 memberwise 初始化操作，包括两个参数、一个返回值以及一个局部对象 pC。它也内带两个 memberwise 拷贝操作，分别是标示为 (1) 和 (2) 那两行的 pC 和 b。main() 函数如下：
```cpp
main()
{
    Point3d pA(1.725, 0.875, 0.478);
    Point3d pB(0.315, 0.317, 0.838);
    Point3d pC;
    for(int iters = 0; iters < 10000000; iters ++)
        pC = lots_of_copies(pA, pB);
    
    return 0;
}
```
在最初的两个程序中，数据类型是一个 struct 和一个拥有 public 数据的 class：
```cpp
struct Point3d{ float x, y, z; };
class Point3d{ public: float x, y, z; };
```
对 pA 和 pB 的初始化操作时通过 explicit initialization list 来进行的：
```cpp
Point3d pA = {1.725, 0.875, 0.478};
Poitn3d pB = {0.315, 0.317, 0.838};
```
这两个操作表现出 bitwise copy 语意，所以你应该会预期它们的执行有最好的效率。结果如下：

“memberwise” 初始化操作和拷贝操作（Initialization and Copy):
Public Data members
Bitwise Copy Semantics
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|CC|5.05|6.39|
|NCC|5.84|7.22|

CC 的效率比较好，是因为 NCC 循环中多产生了六个额外的汇编语言指令。这个额外负担并不会反映出任何特定的 C++ 语意，这两个编译器的 “中间 C 输出” 大致是相等的。

下一个测试，唯一的改变是数据的封装以及 inline 函数的使用，以及一个 inline constructor，用以初始化每一个 object。class 仍然展现出 bitwise copy 语意，所以尝试告诉我们，执行期的效率应该相同。事实上则是有一些距离：

“memberwise” 初始化操作和拷贝操作（Initialization and Copy）:
Private Data Members:
Inline Access and Inline Construction
Bitwise Copy Semantics
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|CC|5.18|6.52|
|NCC|6.00|7.33|

我曾经想过，效率上的差异并不是因为 lots_of_copies()，而是因为 main() 函数中的 class object 初始化操作。所以我修改 struct 的初始化操作如下，并复制 inline class constructor 的扩展部分：
```cpp
mian()
{
    Point3d pA;
    pA.x = 1.725; pA.y = 0.875; pA.z = 0.478;

    Point3d pB;
    pB.x = 0.315; pB.y = 0.317; pB.z = 0.838;

    // ······ 其余相同
}
```
它们现在成为一种封装后的 class 表现形式。我发现两次执行时间都增加了：

“memberwise” 初始化操作和拷贝操作（Initialization and Copy）：
Public Data Members:
Individual Member Initialzation
Bitwise Copy Semantics

||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|CC|5.18|6.52|
|NCC|5.99|7.33|

经由 constructor 的 inline expansion，坐标成员的初始化操作带来以下两个指令的汇编语言码：一个指令把常量值加载到缓存器中，另一个指令进行真正的存储操作：
```
# 20 pt3d pA(1.725, 0.875, 0.478);
li.s $f4, 1.7250000238418579e+00
s.s. $f4, 76($sp)
# etc
```
坐标成员若经由 explicit initialization list 来做初始化操作，会得到单一指令，因为常量值被预先加载好了：
```
$$7:
    .float 1.7250000238418579e+00
    # etc.
```
在封装和未封装过的两种 Point3d 声明之间，另一个差异是关于下一行的语意：
```cpp
Point3d pC;
```
如果使用 ADT 表示法,pC 会以其 default constructor 的 inline expansion 自动进行初始化——甚至虽然就此例而言，没出初始化也很安全。从某一个角度来说，虽然这些差异是在很小，但它们扮演警告的角色，警告说 “封装加上 inline 支持，完全相当于 C 程序中的直接数据存取”。从另一个角度来说，这些差异并不具有什么意义，因此也就没有理由放弃 “封装” 特性在软件工程的利益。它们是一些你得记在心里以备特殊情况下能够排上用场的东西。

下一个测试，我把 Point3d 的表现法切割为三个层次的单一继承：
```cpp
class Point1d { };   // _x
class Point2d : public Point1d { };  // _y
class Poitn3d : public POint2d { };  // _z
```
我没有使用任何 virtual functions。由于 Point3d 仍然展现出 bitwise copy 的语意，所以额外的单一继承不应该影响 “memberwise 对象初始化或拷贝操作” 的成本。这可以由测试结果显示出来：

“memberwise” 初始化操作和拷贝操作（Initialization and Copy):
单一继承：
Protected Members : Inline Access
Bitwise Copy Semantics
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|CC|5.18|6.52|
|NCC|6.26|7.33|

下面的多重继承，一般公认是比较高明的设计。由于其 member 的分布，它完成了任务（至少提供了一个测试例程序☺）：
```cpp
class Point1d { };   // _x
class Point2d { };   // _y
class Point3d
    : public Point1d, public Point2 { };    // _z
```
由于 Point3d class 仍然显现出 bitwise copy 语意，所以额外的多重继承关联不应该在 memberwise 的对象初始化操作或拷贝操作上增加成本。除了 CC 优化情况之外（它的效率稍微高一些），执行的结果的确验证了我的想法：

“memberwise” 初始化操作和拷贝操作（Initialization and Copy)：
多重继承：
Protected Members: Inline Access
Bitwise Copy Semantics
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|CC|5.06|6.52|
|NCC|6.26|7.33|

截止目前的所有测试，所有版本的差异都是以 “初始化三个 local objects” 为中心，而不是以 “memberwise initialization 和 copy 的消耗” 为中心。这些操作的完成都是一成不变的，因为到目前为止的所有那些陈述都支持 “bitwise copy” 语意。然而一旦导入虚拟继承语意，情况就改变了。下面这样的单层虚拟继承：
```cpp
class Point1d { ... };
class Point2d : public virtual Point1d { ... };
class Point3d : public Point2d { ... };
```
不再允许 class 拥有 bitwise copy 语意（第一层虚拟继承不允许之，第二层继承则更加复杂）。合成型的 inline copy constructor 和 copy assignment operator 于是被产生出来，并被派生用场，这导致效率成本上一个重大增加：

“memberwise” 初始化和拷贝操作（Initialization and Copy）：
虚拟基层（一层）：
被合成的Inline Copy Constructor
被合成的Inline Copy Operator
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|CC|15.59|26.45|
|NCC|17.29|23.93|

为了更实在地了解这些数据，让我回到先前的陈述：有着封装味道并加上一个 virtual function 的 class 声明。回忆一下，这种情况并不允许 bitwise copy 语意存在。合成型的 inline copy constructor 和 copy assignment operator 于是被产生出来，并且被派上用场。效率成本的增加虽不如预期，比起 bitwise copy 却也有大约 40%~50%。如果这个函数时程序员供应一个 non-inline 函数，成本仍然较高：

“memberwise” 初始化操作和拷贝操作（Initialization and Copy）:
抽象数据类型：虚拟函数：
被合成的Inline Copy Constructor
被合成的Inline Copy Operator
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|CC|8.34|9.94|
|NCC|7.67|13.05|

下面的测试是采用其他有着 bitwise copy 语意的表现方式，去掉 inline 合成型 memberwise copy constructor 和 copy assignment operator。这一次反映出来的额是对象构造和拷贝的成本增加，原因是继承体系的复杂度增加了。

“memberwise” 初始化操作和拷贝操作（Initialization and Copy）
被合成的Inline Copy Constructor
被合成的Inline Copy Operator
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|单一继承|||
|CC|12.69|17.47|
|NCC|10.35|17.74|
|多重继承|||
|CC|14.91|21.51|
|NCC|12.39|20.39|
|虚拟继承|||
|CC|19.90|29.73|
|NCC|19.31|26.80|

## 5.5 解构语意学（Semantics of Destruction）

如果 class 没有定义 destructor，那么只有在 class 内带的 member object (或是class 自己的 base class) 拥有 destructor 的情况下，编译器才会自动合成出一个来。否则，destructor 会被视为不需要，也就不需被合成（当然更不需要被调用）。例如，我们的 Point，默认情况下并没有被编译器合成出一个 destructor——甚至虽然它拥有一个 virtaul function:
```cpp
class Point{
public:
    Point(float x = 0.0, float y = 0.0);
    Point(const Point&);
    virtual float z();

    // ...
private:
    float _x, _y;
};
```
类似的道理，如果我们把两个 Point 对象组合成一个 Line class：
```cpp
class Line{
public:
    Line(const Point&, const Point&);
    // ...

    virtual draw();
    // ...
protected:
    Point _begin, _end;
};
```

Line 也不会用于一个被合成出来的 destructor，因为 Point 并没有 destructor。

当我们从 Point 派生出 Point3d（甚至即便是一种虚拟派生关联）时，如果我们没有声明一个 destructor，编译器就没有必要合成一个 destructor。

不论 Point 或 Point3d，都不需要 destructor，为它们提供 destructor 反而不符合效率。你应该拒绝那种被我称为 “对称策略” 的奇怪想法：“你已经定义了一个 constructor，所以你以为提供一个 destructor 也是天经地义的事”。事实上，你应该因为 “需要” 而非 “感觉” 来提供 destructor，更不要因为你不确定是否需要一个 destructor，于是就提供它。

为了决定 class 是否需要一个程序层面的 destructor（或是 constructor），请你想想一个 class object 的声明在哪里结束（或开始）？需要什么操作才能保证对象的完整？这是你写程序时比较需要了解的（或是你的 class 使用者比较需要了解的）。这也是 constructor 和 destructor 什么时候起作用的关键。举个例子，已知：
```cpp
{
    Point pt;
    Point *p = new Point3d;
    foo(&pt, p);
    ...
    delete p;
}
```
我们看到，pt 和 p 在作为 foo() 函数的参数之前，都必须先初始化为某些坐标值。这时候需要一个 constructor，是否使用者必须明确地提供坐标值。一般而言，class 的使用者没办法检验一个 local 变量或 heap 变量以知道它们是否被初始化。把 constructor 想象为程序的一个额外负担是错误的，因为它们的工作有其必要性。如果没有它们，抽象化（abstraction）的使用就会有错误的倾向。

当我们明确地 delete 掉 p 时会如何呢？有任何程序上必须处理的吗？是否需要在delete 之前这怎么做：
```cpp
p->x(0); p->y(0);
```
不，当然不需要。没有任何理由说明在 delete 一个对象之前先得将其内容清楚干净。你也不需要归还任何资源。在结束 pt 和 p 的生命之前，没有任何 “class 使用者层面” 的程序操作时绝对必要的，因此，也就不一定需要一个 destructor。

然而请考虑我们的 Vertex class，它维护一个由紧邻的 “顶点” 所形成的链表，并且当一个顶点的声明结束时，在链表上来回移动以完成删除操作。如果这（或其它语意）正是程序员所需要的，那么这就是 Vertex destructor 的工作。

当我们从 Point3d 和 Vertex 派生出 Vertex3d 时，如果我们不供应一个 explicit Vertex3d destructor，那么我们还是希望 Vertex destructor 被调用，以结束一个 Vertex3d object。因此，编译器必须合成一个 Vertex3d destructor，其唯一任务就是调用 Vertex destructor。如果我们提供一个 Vertex3d destructor，编译器会扩展它，使它调用 Vertex destructor（在我们所供应的程序代码之后）。一个由程序员定义的 destructor 被扩展的方式类似 constructors 被扩展的方式，但顺序相反：

1. 如果 object 内带一个 vptr，那么首先重设（reset）相关的 virtual table。
2. destructor 的函数本身现在被执行，也就是说 vptr 会在程序员的码执行前被重设（reset）。
3. 如果 class 拥有 members class object，而后者拥有 destructors，那么它们会以其声明顺序相反顺序被调用。
4. 如果有任何一层（上一层）nonvirtual base classes 拥有 destructor，它们会以其声明顺序的相反顺序被调用。
5. 如果有任何 virtual base classes 拥有 destructor，而当前讨论的这个 class 是最尾端（most-derived）的 class，那么它们会以其原来的构造顺序的相反顺序被调用。。

译注：这个顺序似乎有点问题，请参考下页说明。

就像 constructor 一样，目前对于 destructor 的一种最佳实现策略就是维护两份destructor 实体：
1. 一个 complete object 实体，总是设定好 vptr(s)，并调用 virtual base class destructor。
2. 一个 base class subobject 实体；除非在 destructor 函数中调用一个 virtual function，否则它绝不会调用 virtual base class destructors 并设定 vptr。

一个 object 的生命结束于其 destructor 开始执行之前。由于每一个 base class destructor 都轮番地被调用，所以 derived object 实际上变成了一个完整的 object。例如一个 PVertex 对象归还其内存空间之前，会一次变成一个 Vertex3d 对象、一个 Vertex 对象、一个 Point3d 对象，最后成为一个 Point 对象。当我们在 destructor 中调用 member functions 时，对象的蜕变会因为 vptr 的重新设定（在每一个 destructor 中，在程序员所供应的码执行之前）而受到影响。在程序中施行 destructors 的真正语意将在第 6 章详述。

译注：上页的 destructor 扩展形式似乎应为 2,3,1,4,5，才符合 constructor 的相反顺序。重新整理于下：
1. destructor 的函数本身首先被执行。
2. 如果 class 拥 有member class objects，而后者拥有 destructors，那么它们会以其声明顺序的相反顺序被调用。
3. 如果 object 内带一个 vptr，则现在被重新设定，指向适当之 base class 的 virtual table.
4. ······同稍早所述
5. ······同稍早所述

[第 4 章 Function 语意学（The Semantics of Function）](The_Semantics_of_Function.md)|[第 6 章 执行期语意学（Runtime Semantics）](Runtime_Semantics.md)
