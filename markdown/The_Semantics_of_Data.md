# 第 3 章 Data 语意学（The Semantics of Data）

前些时候我收到一封来自法国的电子邮件，发信人似乎有些迷惘也有些烦乱。他志愿（要不就是被选派）为他的项目团队提供一个“永恒的” library。在做准备工作的时候，他写出以下的码并打印它们的 sizeof 结果：
```cpp
class X{ };
class Y : public virtual X { };
class Z : public virtual X { };
class A : public Y, public Z { };
```

译注：X,Y,Z,A 的继承关系如下图所示:

<img src = 'xyza.png' width = '30%'>

上述 X,Y,Z,A 中没有任何一个 class 内含明显的数据，其间只表示了继承关系。所以发现者认为每一个 class 的大小都应该是 0.当然不对！即使是 class X 的大小也不为 0：
```cpp
sizeof X 的结果为1
sizeof Y 的结果为8
sizeof Z 的结果为8
sizeof A 的结果为12
```

译注：以下是我在 Visual C++ 5.0 上的执行结果
```cpp
sizeof X 的结果为1
sizeof Y 的结果为4
sizeof Z 的结果为4
sizeof A 的结果为8    
```

原因将在 p.86 的译注和 p.87 的正文中解释。

让我们一次看看每一个 class 的声明，并看看他们为什么获得上述结果。

一个空的 class 如：
```cpp
// size X == 1
class X { };
```

事实上并不是空的，它有一个隐晦的 1 byte，那是被编译器安插进去的一个 char。这使得这个 class 的两个 objects 得以在内存中配置独一无二的地址：
```cpp
X a, b;
if(&a == &b) cerr << "yipes!" << endl;
```
另来信读者感到惊讶和沮丧的，我怀疑是 Y 和 Z 的 sizeof 结果：
```cpp
// sizeof Y == sizeof Z == 8
class Y : public virtual X {};
class Z : public virtual X {};
```
在来信者的机器上， Y 和 Z 的大小都是 8.这个大小和机器有关，也和编译器有关。事实上Y和Z的大小受到三个因素的影响：

1. 语言本身所造成的额外负担（overhead） 当语言支持 virtual base class 时，就会导致一些额外负担。在 derived class 中吗，这个额外负担反映在某种形式的指针身上，它或者指向 virtual base class subobject，或者指向一个相关表格；表格中存放的若不是 virtual base class subobject 的地址，就是其偏移量（offset）。在来信者的机器上，指针是 4 bytes（我将在 3.4 节讨论 virtual base class）。
2. 编译器对于特殊情况所提供的优化处理 Virtual base class X subobject 的 1 byte 大小也出现在 class Y 和 Z 身上。传统上它被放在 derived class 的固定（不变动）部分的尾端。某些编译器会对 empty virtual base class 提供特殊支持（以下第 3 点之后的一段文字对此有比较详细的讨论）。来信读者所使用的编译器，显然并未提供这项特殊处理。
3. Alignment 的限制 class Y 和 Z 的大小截止目前为 5 bytes。在大部分机器上，群聚的结构体大小会受到 alignment 的限制，使它们能够更有效率地在内存中被存取。在来信者的机器上，alignment 是 4 bytes，所以 class Y 和 Z 必须填补 3 bytes。最终得到的结果就是 8 bytes。

译注：alignment 就是将数值调整到某数的整数倍。在 32 位计算机上，通常 alignment 位 4 bytes（32 位），以使 bus 的 “运输量” 达到最高效率。

译注：我以下图表现上述的 X，Y，Z 对象布局：
<img src = 'empty.png' width = '80%'>

Empty virtual base class 已经成为 C++ OO 设计的一个特有术语了。它提供一个 virtual interface, 没有任何数据。某些新近的编译器（译注）对此提供了特殊处理（请看 [SUN94a]）.在这个策略之下，一个 empty virtual base class 被视为 derived class object 最开头的一部分，也就是说它并没有花费任何的额外空间。这就节省了上述第 2 点的 1 byte（译注：因为既然有了 members，就不需要原本为了 empty class 而安插的一个 char），也就不需要第 3 点所说的 3 bytes的填补。只剩下第一点所说的额外负担。在此模型下，Y 和 Z 的大小都是 4 而不是 8.

译注：Visual C++ 就是上述这一类型的编译器。我以下图来表现 Visual C++ 对于 class X, Y, Z 的对象布局：
<img src = 'vc.png' width = '80%'>

编译器之间的潜在差异正说明了 C++ 对象模型的演化。这个模型为一般情况提供了解决之道。当特殊情况逐渐被挖掘出来时，种种启发（尝试错误）法于是被引入，提供优化的处理。如果成功，启发法于是就提升为普遍的策略，并跨越各种编译器而合并。它被视为标准（虽然它并不被规范为标准），久而久之也就成了语言的一部分。Virtual function table 就是一个好例子，另一个例子是第 2 章讨论过的 “named return value(NRV) 优化”。

那么，你期望 class A 的大小是什么呢？很明显，某种程度上必须视你所使用的编译器而定。首先，请你考虑那种并未特别处理 empty virtual base class 的编译器。如果我们忘记 Y 和 Z 都是“虚拟派生”自 class X,我们可能会回答 16，毕竟 Y 和 Z 的大小都是8。然而当我们对 class A 施以 sizeof 运算符时，得到的答案竟然是 12。到底是怎么回事？

记住，一个 virtual base class subobject 只会在 derived class 中存在一份实体，不管它在 class 继承体系中出现了多少次！class A 的大小由下列几点决定：
1. 被大家共享的唯一一个 class X 实体，大小为 1 byte。
2. Base class Y 的大小，减去 “因 virtual base class X 而配置” 的大小，结果是4 bytes。Base class Z 的算法亦同。加起来是 8 bytes。
3. class A 自已的大小：0 byte
4. class A 的 alignment 数量（如果有的话）。前述三项总合，表示调整前的大小是 9 bytes。class A 必须调整至 4 bytes 边界，所以需要填补 3 bytes。结果是 12 bytes。

现在如果我们考虑那种 “特别对 empty virtual base class 做了处理” 的编译器呢？一如前述，class X 实体的那 1 byte 将被拿掉，于是额外的 3 bytes 填补额也不必了，因此 classA 的大小将是 8 bytes。注意，如果我们在 virtual base class X 中放置一个（以上）的 data members，两种编译器（“有特殊处理”者和“没有特殊处理”者）就会产生出完全相同的对象布局。

C++ Standard 并不强制规定如 “base class subobjects 的排列次序” 或 “不同存取层级的 data members 的排列次序” 这种琐碎细节。它也不规定 virtual functions 或 virtual base classes 的实现细节。C++ Standard 只说；那些细节由各家厂商自定。我在本章以及全书中，都会区分 “C++ Standard” 和 “当前的 C++ 实现标准” 两种讨论。

在这一章中，class 的 data members 以及 class heirarchy 是中心议题。一个 class 的 data members，一般而言，可以表现这个 class 在程序执行时的某种状态。 Nonstatic data members 放置的是 “个别的 class object” 感兴趣的数据，static data members 则放置的是 “整个 class” 感兴趣的数据。

C++ 对象模型尽量以空间优化和存取速度优化的考虑来表现 nonstatic data members，并且保持和 C 语言 struct 数据配置的兼容性。它把数据直接存放在每一个 class object 之中。对于继承而来的 nonstatic data members（不管是 virtual 或 nonvirtual base class）也是如此。不过并没有强制定义其间的排列顺序。至于 static data members，则被放置在程序的一个 global data segment 中，不会影响个别的 class object 的大小。在程序之中，不管该 class 被产生出多少个 objects（经由直接产生或间接派生），static data members 永远只存在一份实体（译注：甚至即使该 class 没有任何 object 实体，其 static data members 也已存在）。但是一个 template class 的 static data members 的行为稍有不同，7.1 节有详细的讨论。

每一个 class object 因此必须由足够的大小以容纳它所有的 nonstatic data members。有时候其值可能令你吃惊（正如那位法国来信者），因为它可能比你想象的还大，原因是：
1. 由编译器自动加上的额外 data members，用以支持某些语言特性（主要是各种 virtual 特性）。
2. 因为 alignment（边界调整）的需要。

## 3.1 Data Member的绑定（The Binding of a Data Member）

考虑下面这段程序代码：
```cpp
// 某个foo.h头文件，从某处含入
extern float x;

// 程序员的Point3d.h文件
class Point3d
{
public:
    Point3d(float, float, float);
    // 问题：被传回和被设定的x是哪一个x呢
    float X() const { return x; }
    void X(float new_x) const { x = new_x; }
    // ...
private:
    float x, y, z;
};
```
如果我问你 Point3d::X() 传回哪一个 x？是 class 内部的那个 x，还是外部（extern）的那个 x？今天每个人都会回答我是内部那一个。这个答案是正确的，但并不是从过去以来一直都是正确！

在 C++ 最早的编译器上，如果在 Point3d::X() 的两个函数实例中对 x 进行参阅（取用）操作，这操作将会指向 global x object！这样的绑定结果几乎普遍地不在大家的预期之中，并因此导出早期 C++ 的两种防御性程序设计风格：

1. 把所有的 data members 放在 class 声明起头处，以确保正确的绑定：
```cpp
class Point3d
{
    // 防御性程序设计风格 #1
    // 在class声明起头处先放置所有的data member
    float x, y, z;
public:
    float X() const { return x; }
    // ... etc. ...
};
```
2. 把所有的 inline functions，不管大小都放在 class 声明之外：
```cpp
class Point3d
{
public:
    // 防御性程序设计风格 #2
    // 把所有的inlines都移到class之外
    Point3d();
    float X() const;
    void X(flaot) const;
    // ... etc. ...
};
inline float Point3d::
X() const
{
    return x;
}
// ... etc. ...
```
这些程序设计风格事实上到今天还存在，虽然它们的必要性已经自从 C++ 2.0 之后（伴随着 C++ Refernce Manual 的修订）就消失了。这个古老的语言规则被称为 “member rewriting rule”，大意是 “一个 inline 函数实体，在整个 class 声明未被完全看见之前，是不会被评估求值（evaluated）的”。C++ Standard 以 “member scope resolution rules” 来精炼这个 “rewriteing rule”，其效果是，如果一个 inline 函数在 class 声明之后立刻被定义的话，那么就还是对其评估求值（evaluate）。也就是说，当一个人写下这样的码：
```cpp
extern int x;

class Point3d
{
public:
    // 对于函数本身的分析将延迟直至
    // class声明的右大括号出现才开始
    float X() const { return x; }
    // ...
private:
    float x;
};

// 事实上，分析在这里进行
```
时，对 member functions 本身的分析，会直到整个 class 的声明都出现了才开始。因此，在一个 inline member function 躯体之内的一个 data member 绑定操作，会在整个 class 声明完成之后才发生。

然而，这对于 member function 的 argument list 并不为真。Argument list 中的名称还是会在它们第一次遭遇时被适当地决议（resolved）完成。因此在 extern 和 nested type names 之间的非直觉绑定操作还是会发生。例如在下面的程序片段中，length 的类型在两个 member function signatures 中都决议（resolve）为 global typdef，也就是 int。当后续再有 length 的 nested typedef 声明出现时，C++ Standard 就把稍早的绑定标示为非法：
```cpp
typedef int length;

class Point3d
{
public:
    // 喔欧：length被决议（resolved)为global
    // 没问题：_val被决议（resulved)为Point3d::_val
    void mumble(length val) { _val = val; }
    length mumble() { return _val; }
    // ...
private:
    // length 必须在“本class对它的第一个参考操作”之前被看见
    // 这样的声明将使先前的参考操作不合法
    typedef float length;
    length _val;
    // ...
};
```
上述这种语言状况，仍然需要某种防御性程序风格：请始终把 “nested type声明” 放在 class 的起始处。在上述例子中，如果把 length 的 nested typedef 定义于 “在 class 中被参考” 之前，就可以确保非直觉绑定的争取性。

## 3.2 Data Member的布局（Data Member Layout）

已知下面一组 data members:
```cpp
class Point3d{
public:
    // ...
private:
    float x;
    static List<Point3d*> *freeList;
    float y;
    static const int chunkSize = 250;
    float z;
};
```
NonStatic data members 在 class object 中的排列顺序将和其被声明的顺序一样，任何中间介入的 static data members 如 freelist 和 chunkSize 都不会被放进对象布局之中。在上述例子中，每一个 Point3d 对象是由三个 float 组成，次序是 x，y，z. static data members 存放在程序的 data segment 中，和个别的 class objects 无关。

C++ Standard 要求，在同一个 access section（也就是 private、public、protected等区段）中，members 的排列只需符合 “较晚出现的 members 在 class object 中有较高的地址” 这一条件即可（请看 C++ Standard 9.2 节）。也就是说，各个 members 并不一定得连续排列。什么东西可能会介于被声明的 members 之间呢？members 的边界调整（alignment）可能就需要填补一些 bytes。对于 C 和 C++ 而言，这的确是真的，对当前的 C++ 编译器实现情况而言，这也是真的。

编译器还可能会合成一些内部使用的 data members，以支持整个对象模型。vptr 就是这样的东西，当前所有的编译器都把它安插在每一个 “内含 virtual function 之 class” 的 object 内。vptr 会放在什么位置呢？传统上它被放在所有明确声明的 members 的最后， 不过如今也有一些编译器把 vptr 放在一个 class object 的最前端。C++ Standard 秉持先前所说的“对于布局所持的放任态度”。允许编译器把那些内部产生出来的members 自由放在任何位置上，甚至放在那些被程序员声明出来的 members 之间。

C++ Stadard 也允许编译器将多个 access sections 之中的 data members 自由排列，不必在乎它们出现在 class 声明中的次序。也就是说，下面这样的声明：
```cpp
class Point3d{
public:
    // ...
private:
    float x;
    static List<Point3d*> *freeList;
private:
    float y;
    static const int chunkSize = 250;
private:
    float z;
};
```
其 class object 的大小和组成都和我们先前声明的哪个相同，但是 members 的排列次序则视编译器而定。编译器可以随便把 y 或 z 或什么东西放为第一个，不过就我所知，当前没有任何编译器会这么做。

当前各家编译器都是把一个以上的 access sections 连锁在一起，依照声明的次序，称为一个连续区块。Accesss sections 的多寡并不会招来额外负担。例如在一个 section 中声明 8 个 members，或是在 8 个 sections 中总共声明 8 个 members，得到的 object大小是一样的。

下面这个 template function，接受两个 data members，然后判断谁先出现在 class object 之中。如果两个 members 都是不同的 access sections 中的第一个被声明者，此函数就可以用来判断哪一个 section 先出现（如果你对 class member 的指针并不熟悉，请参考 3.6 节）：
```cpp
template< class class_type,
          class data_type1,
          class data_type2 >
char*
access_order(
    data_type1 class_type::*mem1;
    data_type2 class_type::*mem2; )
{
    assert(mem1 != mem2);
    return 
        mem1 < mem2
            ? "member 1 occurs first"
            : "member 2 occurs first";
}
```
上述函数可以这样被调用：
```cpp
access_order(&Point3d::z, &Point3d::y);
```
于是 class_type 会被绑定为 Point3d，而 data_type1 和 data_type2 会被绑定为float。

## 3.3 Data Member 的存取

已知下面这段程序代码：
```cpp
Point3d origin;
origin.x = 0.0;
```
你可能会问 x 的存取成本是什么？答案视 x 和 Point3d 如何声明而定。x 可能是个 static member，也可能是个 nonstatic member。Point3d 可能是个独立（非派生）的 class，也可能是从另一个单一的 base class 派生而来；虽然可能性不高，但它甚至可能是从多重继承或虚拟继承而来。下面数节将依次检验每一种可能性。

在开始之前，让我先抛出一个问题。如果我们有两个定义，origin 和 pt：
```cpp
Point3d origin, *pt = &orgin;
```
我用它们来存取 data members，像这样：
```cpp
origin.x = 0.0;
pt->x = 0.0;
```
通过 origin 存取和通过 pt 存取，由什么重大差异吗？如果你的回答是 yes，请你从 class Point3d 和 data member x 的角度来说明差异的发生因素。我会在这一节结束前重返这个问题并提出我的答案：

### Static Data Members

Static data members，按其字面意义，被编译器提出于 class 之外，一如我在 1.1 节所说，并被视为一个 global 变量（但只在 class 生命范围之内可见）。每一个 member 的存取许可（译注：private 或 protected 或 public），以及与 class 的关联，并不会导致任何空间上或执行时间上的额外负担——不论是在个别的 class objects 或是在 static data member 本身。

每一个 static data member 只有一个实体，存放在程序的 data segment 之中。每次程序参阅（取用）static member，就会被内部转化为对该唯一的 extern 实体的直接参考操作。例如：
```cpp
// origin.chunkSize == 250;
Point3d::chunkSize == 250;  // 译注：我想作者的意思可能是要说
                            // Point3d::chunkSize == 250;
// pt->chunkSize == 250;
Point3d::chunkSize == 250;  // 译注：我想作者的意思可能是要说
                            // Point3d::chunkSize == 250;
```
从指令执行的观点来看，这是 C++ 语言中 “通过一个指针和通过一个对象来存取 member，结论完全相同” 的唯一一种情况。这是因为 “经由 member selection operators（译注：也就是'.'运算符）对一个 static data member 进行存取操作”只是语法上的一种便宜行事而已。member 其实并不在 class object 之中，因此存取 static members 并不需要通过 class object。

但如果 chunkSize 是一个从复杂继承关系中继承而来的 member，又当如何？或许它是一个 “virtual base class 的 virtual base class”（或其他同等复杂的继承结构）的 member 也说不定。哦，那无关紧要，程序之中对于 static members 还是只有唯一一个实体，而其存取路径仍然是那么直接。

如果 static data member 的存取是经由函数调用（或其他某些语法）而别存取呢？举个例子，如果我们写：
```cpp
foobar().chunkSize == 250;  // 译注：我想作者的意思可能是要说
                            // foobar().chunkSize = 250
```
调用 foobar() 会发生什么事情？在 C++ 的准标准（pre-Standard）规格中，没有人知道会发生什么事，因为 ARM 并未指定 foobar() 是否必须被求值（evaluated）。cfront 的做法是简单地把它丢掉。但 C++ Standard 明确要求 foobar() 必须被求值（evaluated），虽然其结果并无用处。下面是一种可能的转化：
```cpp
// foobar().chunkSize == 250;   // 译注：我想作者的意思是要说
                                // foobar().chunksize = 250;

// evaluate expression, discarding result (void) foobar();
Point3d.chunkSize == 250;       // 译注：我想作者的意思是要说
                                // Point3d.chunkSize = 250;
```
若取一个 static data member 的地址，会得到一个指向其数据类型的指针，而不是一个指向其 class member 的指针，因为 static member 并不内含在一个 class object 之中。例如：
```cpp
&Point3d::chunkSize;
```
会获得类型如下的内存地址：
```cpp
const int*
```
如果有两个 classes，每一个都声明了一个 static member freeList，那么当它们都被放在程序的 data segment 时，就会导致名称冲突。编译器的解决方式是暗中对每一个 static data member 编码（这种手法有个很美的名称：name-mangling)，以获得一个独一无二的程序识别代码。有多少个编译器，就有多少种 name-mangling 做法!通常不外乎是表格啦、语法措辞啦等等。任何 name-mangling 做法都有两个要点：
1. 一种算法，推导出独一无二的名称。
2. 万一编译系统（或环境工具）必须和使用者交谈，那些独一无二的名称可以轻易被推导回到原来的名称。
 
### Nonstatic Data Members

Nonstatic data members 直接存放在每一个 class object之中，除非经由明确的（explicit）和暗喻的（implicit）class object，没办法直接存取它们，只要程序员在一个 member function 中直接处理一个 nonstatic data member，所谓 “implicit class object” 就会发生。例如下面这段码：
```cpp
Point3d
Point3d::translate(const Point3d& pt){
    x += pt.x;
    y += pt.y;
    z += pt.z;
}
```
表面上所看到的对于 x，y，z 的直接存取，事实上是经由一个 “implicit class object” （由 this 指针表达）完成。事实上这个函数的参数是：
```cpp
// member function的内部转化
Point3d
Point3d::translate(Point3d *const this, const Point3d &pt){
    this->x += pt.x;
    this->y += pt.y;
    this->z += pt.z;
}
```
Member functions 在本书的第 4 章由比较详细的讨论。

欲对一个 nonstatic data member 进行存取操作，编译器需要把 class object 的起始地址加上 data member 的偏移量（offset）。举个例子，如果：
```cpp
origin._y = 0.0;
```
那么地址 &origin._y 将等于：
```cpp
&origin + (&Point3d::_y - 1);
```
请注意其中的 -1 操作。指向 data member 的指针，其 offset 只总是被加上1，这样可以使编译器系统区分出 “一个指向 data member 的指针，用以指出 class 的第一个member” 和 “一个指向 data member 的指针，没有指出任何 member” 两种情况。 “指向 data members 的指针” 将在 3.6 节有比较详细的讨论。

每一个 nonstatic data member 的偏移量（offset）在编译器时期即可获知，甚至如果 member 属于一个 base class subobject（派生自单一或多重继承串链）也是一样。因此，存取一个 nonstatic data member，其效率和存取一个 C struct member 或一个 nonderived class 的 member 是一样的。

现在让我们看看虚拟继承。虚拟继承将为 “经由 base class subobject 存取 class members” 导入一层新的间接性，譬如：
```cpp
Point3d *pt3d;
pt3d->_x = 0.0
```
其执行效率在 _x 是一个 struct member、一个 class member、单一继承、多重继承的情况下都完全相同。但如果 _x 是一个 virtual base class 的 member，存取速度会比较慢一点。下一节我会验证 “继承对于 member 布局的影响”。在我们尚未进行到那里之前，请会议本节一开始的一个问题：以两种方法存取x坐标，像这样：
```cpp
origin.x = 0.0;
pt->x = 0.0;
```
“从origin存取” 和 “从pt存取” 有什么重大的差异？答案是 “当 Point3d 是一个derived class，而在其继承结构中有一个 virtual base class，并且被存取的 member（如本例的 x）是一个从该 virtual base class 继承而来的 member 时，就会有重大的差异”。这时候我们不能够说 pt 必然指向哪一种 class type（因此我们也就不知道编译时期这个 member 真正的 offset 位置），所以这个存取操作必须延迟至执行期，经由一个额外的间接导引，才能够解决。但如果使用 origin，就不会有这些问题，其类型无疑是 Point3d class，而即使它继承自 virtual base class，members 的 offset 位置也在编译时期就固定了。一个积极进取的编译器甚至可以静态地经由 origin 就解决掉了 x 的存取。


## 3.4 “继承”与Data Member

在 C++ 继承模型中，一个 derived class object 所表现出来的额东西，是其自己的members 加上其 base class(es) members 的总和。至于 derived class members 和 base class(es) members 的排列次序并未在 C++ Standard 中强制指定：理论上编译器可以自由安排之。在大部分编译器上头，base class members 总是先出现，但属于 virtual base class 的除外（一般而言，任何一条规则一旦碰上 virtual base class  就没辙儿，这里亦不例外。）

了解了这种继承模型之后，你可能会问，如果我为 2D（二维）或 3D（三维）坐标点提供两个抽象数据类型如下：
```cpp
// supporting abstract data types
class Point2d{
public:
    // constructor(s)
    // operations
    // access functions
private:
    float x, y;
};
class Point3d{
public:
    // constructor(s)
    // operations
    // access functions
private:
    float x, y, z;
};
```
这和 “提供两层或三层继承结构，每一层（代表一个维度）是一个 class，派生自较低维层次” 有什么不同？下面各小节的讨论将涵盖 “单一继承且不含 virtual functions”、“单一继承并含 virtual functions”、“多重继承”、“虚拟继承” 等四种情况。图 3.1a 就是 Point2d 和 Point3d 的对象布局图，在没有 virtual functions 的情况下（如本例），它们和 C struct 完全一样。

<img src = '3.1a.png' width = '60%'>

图 3.1a 个别 structs 的数据布局

### 只要继承不要多态（Inheritance without Polymorphism）

想象一下，程序员或许希望，不论是 2D 或 3D 坐标点，都能够共享同一个实体，但又能够继续使用 “与类型性质相关（所谓 type-specific)” 的实体。我们有一个设计策略，就是从 Point2d 派生出一个 Point3d，于是 Point3d 将继承 x 和 y 坐标的一切（包括数据实体和操作方法）。带来的影响则是可以共享 “数据本身” 以及 “数据的处理方法”，并将其局部化。一般而言，具体继承（concrete inheritance，译注：相对于虚拟继承virtual inheritance）并不会增加空间或存取时间上的额外负担。
```cpp
class Point2d{
public:
    Point2d(float x = 0.0, float y = 0.0)
        : _x(x), _y(y) { };
    float x() { return _x; }
    float y() { return _y; }

    void x(float newX) { _x = newX; }
    void y(float newY) { _y = newY; }

    void operator+=(const Point2d& rhs){
        _x += rhs.x();
        _y ++ rhs.y();
    }
    // ... more members
protected:
    float _x, _y;
};

// inheritance from concrete class
class Point3d : public Point2d{
public:
    Point3d(float x = 0.0, float y = 0.0, float z = 0.0)
        : Point2d(x, y), _z(z) { };
    
    float z() { return _z; }
    void z(float newZ) { _z = newZ; }
    void operator+=(const Point3d& rhs){
        Point2d::operator+=(rhs);
        _z += rhs.z();
    }
    // ... more members
protected:
    float _z;
};
```
这样设计的好处就是可以把管理 x 和 y 坐标的程序代码局部化，此外这个设计可以明显表现出两个抽象类之间的紧密关系。当这两个 classes 独立的时候，Point2d object 和 Point3d object 的声明和使用都不会有所改变。所以这两个抽象类的使用者不需要知道objects 是否为独立的 classes 类型，或是彼此之间有继承的关系。如 3.1b 显示 Point2d 和 Point3d 继承关系的实物布局，其间并没有声明 virtual 接口。

<img src = '3.1b.png' width = '80%'>

图 3.1b 单一继承而且没有 virtual function 时的数据布局

把两个原本独立不相干的 classes 凑成一对 “type/subtype”，并带有继承关系，会有什么易犯的错误呢？经验不足的人可能会重复设计一些相同操作的函数。以我们例子中的 constructor 和 operator+= 为例，它们并没有被做成 inline 函数（也可能是编译器为了某些理由没有支持 inline member functions）。Point3d object 的初始化操作或加法操作，将需要部分的 Point2d object 和部分的 Point3d object 作为成本。一般而言，选择某些函数做成 inline 函数，是设计 class 时的一个重要课题。

第二个易犯的错误是，把一个 class 分解为两层或更多层，有可能会为了 “表现 class 体系之抽象化” 而膨胀所需空间。C++ 语言保证 “出现在 derived class 中的 base class subobject 有其完整原样性”，正是重点所在。这似乎有点难以理解！最好的解释方法就是彻底了解一个例程，让我们从一个具体的class开始：
```cpp
class Concrete{
public:
    // ...
private:
    int val;
    char c1;
    char c2;
    char c3;
};
```
在一部32位机器中，每一个 Concrete class object 的大小都是 8 bytes，细分如下：

1. val占用 4 bytes；
2. c1、c2 和 c3各占用 1 byte；
3. alignment（调整到 word 边界）需要 1 bytes。

现在假设，经过某些分析之后，我们决定一个更逻辑的表达方式，把 Concrete 分裂为三层结构：
```cpp
class Concrete1{
public:
    // ...
private:
    int val;
    char bit1;
};
class Concrete2 : public Concrete1{
public:
    // ...
private:
    char bit2;
};
class Concrete3 : public Concrete2{
public:
    // ...
private:
    char bit3;
};
```
从设计的观点来看，这个结构可能比较合理。但从效率的观点来看，我们可能会受困于一个事实：现在 Concrete3 object 的大小是 16 bytes，比原先的设计多了一倍。

> @zl : 我在我的电脑上对此进行了试验。用 g++ 8.2.0 编译器时，得到 Concrete1、Concete2、Concrete3 object 的大小均为 8 bytes；用 Visual studio 2019 编译器时，得到 Concrete1、Concete2、Concrete3 object 的大小分别为 8 bytes、12 bytes、16 bytes。


怎么回事，还记得 “base class subobject 在 derived class 中的原样性”吗？让我们仔细观察这一继承结构的内部布局，看看到底发生了什么事。

Concrete1 内含两个 members：val 和 bit1，加起来是 5 bytes，而一个 Concrete1 object 实际用掉 8 bytes，包括填补用的 3 bytes，以使 object 能够符合一部机器的word 边界。不论是 C 或 C++ 都是这样。一般而言，边界调整（alignment）是由处理器（processor）来决定的。

到目前为止没什么需要抱怨的。但这种典型的布局会导致轻率的程序员犯下错误。Concrete2 加入唯一一个 nonstatic data member bit2，数据类型为 char。轻率的程序员以为它会和 Concrete1 捆绑在一起，占用原本用来填补空间的 1 tyte；于是 Concrete2 object 的大小为 8 bytes，其中 2 bytes用于填补空间。

然而 Concrete2 的 bit2 实际上却是被放在填补空间所用的 3 bytes 之后。于是其大小变成 12 bytes，不是 8 bytes。其中有 6 bytes 浪费在填补空间上。相同的道理使得 Concrete3 object 的大小是 16 bytes，其中 9 bytes 用于填补空间。

“真是愚蠢”，我们那位纯真小甜甜这么说。许多读者以电子邮件、电话或是用嘴巴也对我这么说。你可了解为什么这个语言有这样的行为？

译注：下图可说明 Concrete1、Concrete2、Concrete3 的对象布局
<img src = 'concrete.png' width = '80%'>

让我们声明以下一组指针：
```cpp
Concrete2 *pc2;
Concrete1 *pc1_1, *pc1_2;
```
其中 pc1_1 和 pc1_2  两者都可以指向前述三种 classes object。下面这个指定操作：
```cpp
*pc1_2 = *pc1_1;
```
应该执行一个默认的 “memberwise” 复制操作（复制一个个的 members），对象是被指的 object 的 Concrete1 那一部分。如果 pc1_1 实际指向一个 Concrete2 object 或 Concrete3 object，则上述操作应该将复制内容指定给其 Concrete1 subobject.

然而，如果 C++ 语言把 derived class members（也就是 Concrete2::bit2 或Concrete3::bit3）和 Concrete1 subobject 捆绑在一起，去除填补空间，上述那些语意就无法保留了，那么下面的指定操作：
```cpp
pc1_1 = pc2;    // 译注：令pc1_1指向Concrete2对象

// 喔欧：derived class subobject被覆盖掉
// 于是其bit2 member现在有了一个并非预期的数值
*pc1_2 = *pc1_1;
```
就会将 “被捆绑在一起、继承而得的” members 内容覆盖掉。程序员必须花费极大的心力才能找出这个“臭虫”!

译注：让我以图形解释。如果 “base class subobject 在 derived class 中的原样性”受到破坏，也就是说，编译器把 base clas object 原本的填补空间让出来给 derived class members 使用，像这样：

<img src = 'concrete1.png' width = '80%'>

那么当发生 Concrete1 subobject 的复制操作时，就会破坏 Concrete2 members。

<img src = 'concrete2.png' width = '80%'>

### 加上多态（Adding Polymorphism）

如果我们要处理一个坐标点，而不打算在乎它是一个 Point2d 或 Point3d 实例，那么我需要在继承关系中提供一个 virtual function 接口。让我们看看如果这么做，情况会有什么改变：
```cpp
// 译注：以下Point2d声明请与#101页的声明做比较
class Point2d{
public:
    Point2d(float x = 0.0, float y = 0.0)
        : _x( x ), _y( y ) { };
    
    // x和y的存取函数与前一版相同
    // 由于对不同维度的点，这些函数操作固定不变，所以不必设计为virtual

    // 加上z的保留空间（当前什么也没做）
    virtual float z() { return 0.0; }   // 译注：2d点的z为0.0是合理的
    virtual void z(flaot) { }

    // 设定以下的运算符为virtual
    virtual void
    operator += (const Point2d& rhs){
        _x += rhs.x();
        _y += rhs.y();
    }
    // ... more members
protected:
    float _x, _y;
};
```
只有当我们企图以多态的方式（polymorphically）处理 2d 或 3d 坐标点时，在设计之中导入一个 virtual 接口才显得合理。也就是说，写下这样的码：
```cpp
void foo(Point2d &p1, Point2d &p2){
    // ...
    p1 += p2;
    // ...
}
```
其中 p1 和 p2 可能是 2d 也可能是 3d 坐标点。这并不是先前任何设计所能支持的。这样的弹性，当然正是面向对象程序设计的中心。这样的弹性，势必给我们的 Point2d class 带来空间和存取时间的额外负担：

1. 导入一个和 Point2d 有关的 virtual table，用来存放它所声明的每一个 virtual functions 的地址。这个 table 的元素数目一般而言是被声明的 virtual functions 的数目，在加上一个或两个 slots（用以支持 runtime type identification）。
2. 在每一个 class object 中导入一个 vptr，提供执行期的链接，使每一个 object 能够找到相应的 virtual table。
3. 加强 constructor，使它能够为 vptr 设定初值，让它指向 class 所对应的 virtual table。这可能意味着在 dirived class 和每一个 base class 的 constructor 中，重新设定 vptr 的值。其情况视编译器的优化的积极性而定。第 5 章对此有比较详细的讨论。
4. 加强 destructor，是它能够抹消 “指向 class 之相关 virtual table” 的 vptr。要知道，vptr 很可能已经在 derived class destructor 中被设定为 derived class 的virtual table 地址。记住，destructor 的调用次序是反向的：从 derived class 到 base class。一个积极的优化编译器可以压抑那些大量的指定操作。

这些额外负担带来的冲击程度视 “被处理的 Point2d objects 的数目和生命期” 而定，也视 “对这些 objects 做多态程序设计所得的权益” 而定。如果一个应用程序知道它所能使用的 Point objects 只限于二位坐标点或三维坐标点，那么这种设计所带来的额外负担可能变得令人无法接受[1]。

[1] 我不知道是否有哪个产品系统真正使用了一个多态的 Point 类别体系。

以下是新的Point3d声明：
```cpp
// 译注：以下的Point3d声明请与#101页的声明做比较
class Point3d : public Point2d{
public:
    Point3d(float x = 0.0, float y = 0.0, float z = 0.0)
        : Point2d(x, y), _z(z) { };
    
    float z() { return _z; }
    void z(float new Z) { _z = newZ; }

    void operator += (const Point2d& rhs){
        // 译注：注意上行是Point2d&而非Point3d&
        Point2d::operator += (rhs);
        _z += rhs.z();
    }
    // ... more members
protected:
    float _z;
}
```
译注：上述新的（与 p.101 比较）Point2d 和 Point3d 声明，最大一个好处是，你可以把 operator+= 运用在一个 Point3d 对象和一个 Point2d 对象身上：
```
Point2d p2d(2.1, 2.2);
Point3d p3d(3.1, 3.1, 3.3);
p3d += p2d;
```
得到的p3d新值将是(5.2, 5.4, 3.3);

虽然 class 的声明语法没有改变，但每一件事情都不一样了：两个 z() member function 以及 operator+=() 运算符都成了虚拟函数：每一个 Point3d class object 内含一个额外的 vptr member（继承自 Point2d）；多了一个 Point3d virtual table；此外，每一个 virtual member function 的调用也比以前复杂了（第 4 章对此有详细说明）。

目前在 C++ 编译器那个领域里有一个主要的讨论题目：把 vptr 放置在 class object 的哪里会更好？在 cfront 编译器中，它被放在 class object 的尾端，用以支持下面的继承类型，如图 3.2a 所示：
```cpp
struct no_virts{
    int d1, d2;
};

class has_virts : public no_virts{
public:
    virtual void foo();
    // ...
private:
    int d3;
};

no_virts *p = new has_virts;
```

<img src = 'virts.png' width = '80%'>

图 3.2a Vptr 被放在 class 的尾端

把 vptr 放在 class object 的尾端，可以保留 base class C struct 的对象布局，因而允许在 C 程序代码中也能使用。这种做法在 C++ 最初问世时，被许多人采用。

到了 C++2.0，开始支持虚拟继承以及抽象基类，并且由于面向对象典范（OO paradigm）的兴起，某些编译器开始把 vptr 放到 class object 的起头处（例如 Martin O'Riordan, 他领导 Microsoft 的第一个 C++ 编译器产品，就十分主张这种做法）。请看图 3.2b 的图解说明。

<img src = 'virts1.png' width = '80%'>

图 3.2b Vptr 放在 class 的前端

把 vptr 放在 class object 的前端，对于 “在多重继承之下，通过指向 class members 的指针调用 virtual function”，会带来一些帮助（请参考 4.4 节）。否则，不仅 “从 class object 起始点开始量起” 的 offset 必须在执行期备妥，甚至于 class vptr 之间的 offset 也准备妥当。当然，vptr 放在前端，代价就是丧失了 C 语言兼容性。这种丧失有多少意义？有多少程序会从一个 C struct 派生出一个具多态性质的 class呢？当前我手上并没有什么统计数据可以告诉我这一点。

图 3.3 显示 Point2d 和 Point3d 加上了 virtual function 之后的继承布局。注意此图是把 vptr 放在 base class 的尾端。

<img src = 'point_virtual_func.png' width = '80%'>

图3.3 单一继承并含虚拟函数情况下的数据布局

### 多重继承（Multiple Inheritance）

单一继承提供了一种 “自然多态（natural polymorphism” 形式，是关于 classes 体系中的 base bype 和 derived type 之间的转换。请看图 3.1b、图 3.2a 或图 3.3，你会看到 base class 和 derived class 的 object 都是从相同的地址开始，其间差异只在于 derived object 比较大，用以多容纳它自己的 nonstatic data members。下面这样的指定操作：
```cpp
Point3d p3d;
Point2d *p = &p3d;
```
把一个 derived class object 指定给 base class（不管继承深度有多深）的指针或 reference。该操作并不需要编译器去调停或修改地址。它很自然地可以发生，而且提供了最佳执行效率。

图 3.2b 把 vptr 放在 class object 的起始处。如果 base class 没有 virtual function 而 derived class 有（译注：正如图 3.2b），那么单一继承的自然多态（natural polymorphism）就会被打破。在这种情况下，把一个 derived object 转换为其 base 类型，就需要编译器的介入，用以调整地址（因 vptr 插入之故）。在既是多重继承又是虚拟继承的情况下，编译器的介入更有必要。

多重继承既不像单一继承，也不容易模塑其模型。多重继承的复杂度在于 derived class 和其上一个 base class 乃至于上上一个 base class.... 之间的 “非自然” 关系。例如，考虑下面这个多重继承所获得的 class Vertex3d:

译注：原书的 p92~p94 有很多前后不一致的地方，以及很多 “本身虽没有错误却可能误导读者思想” 的叙述。程序代码和图片说明也不相符，简直一团乱！我已将其全部更正。如果您拿着原文书对照此中译本看，请不要乍见之下对我产生误会。

```cpp
class Point2d{
public:
    // ...(译注：拥有virtual接口。所以Point2d对象之中会有vptr)
protected:
    float _x, _y;
};

class Point3d : public Point2d{
public:
    // ...
protected:
    float _z;
};

class Vertex{
public:
    // ...(译注：拥有virtual接口，所以Vertex对象之中会有vptr)
protected:
    Vertex *next;
};

class Vertex3d :    // 译注：原书误把Vertex3d写为Vertex2d
    public Point3d, public Vertex{  // 译注：原书误把Point3d写为Point2d
public:
    // ...
protected:
    float mumble;
};
```
译注：至此，Point2d、Point3d、Vertex、Vertex3d的继承关系如下：

<img src = 'vertex3d.png' width = '40%'>

多重继承的问题主要发生于 derived class objects 和其第二或后继的 base class objects 之间的转换；不论是直接转换如下：
```cpp
extern void mumble(const Vertex&);
Vertex3d v;
...
// 将一个Vertex3d转换为一个Vertex.这是“不自然的”
mumble(v);
```
或是经由其所支持的 virtual function 机制转换。因支持 “virtual function 之调用操作” 而引发的问题将在 4.2 节讨论。

对一个多重派生对象，将其地址指定给 “最左端（也就是第一个）base class 的指针”，情况将和单一继承时相同，因为二者都指向相同的其实地址。需付出的成本只有地址的指定操作而已（图 3.4 显示出多重继承的布局）。至于第二个或后继的 base class 的地址指定操作，则需要将地址修改过：加上（或减去，如果 downcast 的话）介于中间的 base class subobjects(s) 大小，例如：
```cpp
Vertex3d v3d;
Vertex *pv;
Point2d *p2d;   // 译注：原书命名为*pp，不符合命名原则，改为*p2d较佳
Point3d *p3d;   // 译注：Point3d的定义请看图3.3和#109页
```
那么下面这个指定操作：
```cpp
pv = &v3d;
```
需要这样的内部转化：
```cpp
// 虚拟C++码
pv = (Vertex*)(((char*)&v3d) + sizeof(Point3d));
```
而下面的指定操作：
```cpp
p2d = &v3d;
p3d = &v3d;
```
都只需要简单地拷贝其地址就行了。如果有两个指针如下：
```cpp
Vertex3d *pv3d;     // 译注：原书命名为*p3d，不符合命名规则，改为*pv3d较佳
Vertex *pv;
```
那么下面的指定操作：
```cpp
pv = pv3d;
```
不能够只是简单地被转换为：
```cpp
// 虚拟C++码
pv = (Vertex*)((char*)pv3d) + sizeof(Point3d);
```
因为如果 pv3d 为 0，pv 将获得 sizeof(Point3d) 的值。这是错误的！所以，对于指针，内部转换操作需要有一个条件测试：
```cpp
// 虚拟C++码
pv = pv3d
    ? (Vertex*)((char*)pv3d) + sizeof(Point3d)
    : 0;
```
至于 reference，则不需要针对可能的 0 值做防卫，因为 reference 不可能参考到“无物”（no object）。

<img src = 'vertex3d1.png' width = '80%'>

图3.4 数据布局:多重继承（Multiple Inheritance）

译注：原书的图 3.4 只画出 Vertex2d，没有画出 Vertex3d。虽然其中的 Vertex2d 对象布局图 “可能” 是正确的（我们并没有在书中看到其声明码），但我相信这其实是 Lippman 的笔误，因为它与书中的许多讨论没有关系。所以我把真正与书中讨论有关的 Vertex3d 的对象布局画于译本的图 3.4，如上。

C++ Standard 并未要求 Verted3d 中的 base classes Point3d 和 Vertex 有特定的排列次序。原始的 cfront 编译器是根据声明次序来排列它们。因此 cfront 编译器制作出来的 Vertex3d 对象，将可被视为是一个 Point3d subobject（其中又有一个Point2d subobject）加上一个 Vertex subobject，最后再加上 Vertex3d 自己的部分。目前各编译器仍然是一次方式完成多重 base classes 的布局（但如果加上虚拟继承，就不一样了）。

某些编译器（例如 MetaWare）设计有一种优化技术，只要第二个（后后继）base class 声明了一个 virtual function，而第一个 base class 没有，就把多个 base classes 的次序进行调换。这样可以在 derived class object 中少产生一个 vptr。这项优化技术并未得到全球各厂商的认可，因此并不普及。

如果要存取第二个（或后继）base class 中的一个 data member，将会是怎样的情况？需要付出额外的成本吗？不，members 的位置在编译时就固定了，因此存取 members 只是一个简单的 offset 运算，就像单一继承一样简单——不管是经由一个指针、一个 reference 或是一个 object 来存取。

### 虚拟继承（Virtual Inheritance）

多重继承的一个语意上的副作用就是，它必须支持某种形式的 “shared subobject 继承”。一个典型的例子是最早的 iostream libray:
```cpp
// pre-standard iostream implementation
class ios { ... };
class istream : public ios { ... };
class ostream : public ios { ... };
class iostream :
    public istream, public ostream { ... };
```
译注：下图可表现iostream的继承体系图，左为多重继承，右为虚拟多重继承。

<img src = 'iostream.png' width = '80%'>

不论是 istream 或 ostream 都内含一个 ios subobject，然而在 iostream 的对象布局中，我们只需要单一一份 ios subobject 就好。语言层面的解决方法是导入所谓的虚拟继承：

```cpp
class ios { ... };
class istream : public virtual ios { ... };
class ostream : public virtual ios { ... };
class iostream : 
    public istream, public ostream { ... };
```
一如其语意所呈现的复杂度，要在编译器中支持虚拟继承，实在是困难度颇高。在上述 iostream 例子中，实现技术的挑战在于，要找到一个足够有效的方法，将 istream 和 ostream 各自维护的一个 ios subobject，折叠成为一个由 iostream 维护的单一 ios subobject，并且还可以保存 base class 和 derived class 的指针（以及 reference）之间的多态指定操作（polymorphism assignments）。

一般的实现方法如下所述。Class 如果内含一个或多个 virtual base class subobjects，像 istream 那样，将被分割为两部分：一个不变局部和一个共享局部。不变局部中的数据，不管后继如何演化，总是拥有固定的 offsoet (从 object 的开头算起)，所以这一部分数据可以被直接存取。至于共享局部，所表现的就是 virtual base class subobject。这一部分的数据，其位置会因为每次的派生操作而有变化，所以它们只可以被间接存取。各家编译器实现技术之间的差异就在于间接存取的方法不同。以下说明三种主流策略。下面是 Vertex3d 虚拟继承的层次结构[2]:
```cpp
class Point2d{
public:
    ...
protected:
    float _x, _y;
};

class Vertex : public virtual Point2d{
public:
    ...
protected:
    Vertex *next;
};

class Point3d : public virtual Point2d{
public:
    ...
protected:
    float _z;
};

class Vertex3d :
    public Vertex, public Point3d
    // 译注：原书上一行的两个classes次序相反。为与图3.5ab配合，故改之。
{
public:
    ...
protected:
    float mumble;
};
```
[2] 这个层次结构是 [POKOR94] 所倡议的，那是一本很好的 3D Graphics 教科书，使用 C++ 语言。

译注：下图可表现 Point2d、Point3d、Vertex、Vertex3d 的继承体系：

<img src = 'vertex3d2.png' width = '60%'>   

一般的布局策略是先安排好 derived class 的不变部分，然后再建立其共享部分。

然而，这中间存在着一个问题：如何能够存取 class 的共享部分呢？cfront 编译器会在每一个 derived class object 中安插一些指针，每个指针指向一个 virtual base class。要存取继承得来的 virtual base class members，可以使用相关指针间接完成。举个例子，如果我们有以下的 Point3d 运算符：
```cpp
void
Point3d::
operator ++ (const Point3d &rhs)
{
    _x += rhs._x;
    _y += rhs._y;
    _z += rhs._z;
};
```
在 cfront 策略之下，这个运算符会被内部转换为：
```cpp
// 虚拟C++码
__vbcPoint2d->_x += rhs.__vbcPoint2d->_x;   // 译注：vbc意为virtual base class
__vbcPoint2d->_y += rhs.__vbcPoint2d->_y;
_z += rhs._z;
```
而一个 derived class 和一个 base class 的实例之间的转换，像这样：
```cpp
Point2d *p2d = pv3d;    // 译注：原书为Vertex *pv = pv3d； 恐为笔误
```
在 cfront 实现模型之下，会变成：
```cpp
// 虚拟C++码
Point2d *p2d = pv3d ? pv3d->__vbcPoint2d : 0;
// 译注：原书为 Vertex *pv = pv3d ? pv3d->__vbcPoint2d : 0;
// 恐为笔误（感谢黄俊达先生与刘东岳先生来信指导）
```

这样的实现模型有两个主要的缺点：
1. 每一个对象必须针对其每一个 virtual base class 背负一个额外的指针，然而理想上我们却希望 class object 有固定的负担，不因为其 virtual base classes 的数目而有所变化。想想看这该如何解决？
2. 由于虚拟继承串链的加长，导致间接存取层次的增加。这里的意思是，如果我有三层虚拟衍化，我就需要三次间接存取（经由三个 virtual base class 指针）。然而理想上我们却希望有固定的存取时间，不因为虚拟衍化的深度而改变。

MetaWare 和其它编译器到今天仍然使用 cfront 的原始实现模型来解决第二个问题，它们经由拷贝操作取得所有的 nested virtual base class 指针，放到 derived class object 之中。这就解决了 “固定存取时间” 的问题，虽然付出了一些空间上的代价。MetaWare 提供一个编译时期的选项，允许程序员选择是否要产生双重指针。图 3.5a 说明这种 “以指针指向 base class” 的实现模型。

至于第一个问题，一般而言有两个解决方法。Microsoft 编译器引入所谓的 virtual base class table。每一个 class object 如果有一个或多个 virtual base classes，就会由编译器安插一个指针，指向 virtual base class table。至于真正的virtual base class 指针，当然是放在该表格中。虽然此法已行之有年，但我并不知道是否有其它任何编译器使用此法。说不定 Microsoft 对此法提出专利，以至别人不能使用它。

<img src = 'vertex3d3.png' width = '100%'>

图 3.5a 虚拟继承，使用 Pointer Strategy 所产生的数据布局（译注：原书图 3.5a 把Vertex 写为 Vertex2d，与书中程序代码不符，所以我全部改为 Vertex）

第二个解决方法，同时也是 Bjarne 比较喜欢的方法（至少当我还和他共事于 Foundation 项目时），是在 virtual function table 中放置 virtual base class 的 offset（而不是地址）。图 3.5b 显示这种 base class offset 实现模型。我在 Foundation 项目中实现出这种方法，将 virtual base class offset 和 virtual function entries 混在在一起。在新近的 Sun 编译器中，virtual function table 可经由正值或负值来索引。如果是正值，很显然就是索引到 virtual functions;如果是复值，则是索引到 virtual base class offsets。在这样的策略之下，Point3d 的 operator+= 运算符必须被转换为以下形式（为了可读性，我没有做类型转换，同时我也没有先执行对效率有帮助的地址预先计算操作）：
```cpp
// 虚拟C++代码
(this + __vptr__Point3d[-1])->_x +=
    (&rhs + rhs.__vptr__Point3d[-1])->_x;
(this + __vptr__Point3d[-1])->-y +=
    (&rhs + rhs.__vptr__Point3d[-1])->_y
_z += rhs._z;
```

虽然在此策略之下，对于继承而来的 members 做存取操作，成本会比价昂贵，不过该成本已经被分散至 “对 member 的使用” 上，属于局部性成本。Derived class 实体和 base class 实体之间的转换操作，例如：
```cpp
Point2d *p2d = pv3d; // 译注：原书为Vertex *pv = pv3d; 恐为笔误
```
在上述实现模型下将变成：
```cpp
// 虚拟C++码
Point2d *p2d = pv3d ? pv3d + pv3d->__vptr__Point3d[-1] : 0;
// 译注：上一行原书为：
// Vertex *pv = pv3d ? pv3d + pv3d->__vptr__Point3d[-1] : 0;
// 恐为笔误（感谢黄俊达先生与刘东岳先生来信指导）
```
上述每一种方法都是一种实现模型，而不是一种标准。每一种模型都是用来解决 “存取 shared subobject 内的数据（其位置会因每次派生操作而有变化）” 所引发的问题。由于对 virtual base class 的支持带来额外的负担以及高度的复杂性，每一种实现模型多少有点不同，而且我想还会随着时间而进化。

经由一个非多态的 class object 来存取一个继承而来的 virtual base class 的 member，像这样：
```cpp
Point3d origin;
...
origin._x;
```
可以被优化为一个直接存取操作，就好像一个经由对象调用的 virtual function 调用操作，可以在编译时期被决议（resolved）完成一样。在这次存取以及下一次存取之间，对象的类型不可以改变，所以 “virtual base class subobject 的位置会变化” 的问题在这种情况下就不再存在了。

一般而言，virtual base class 最有效的一种运用形式就是：一个抽象的 virtual base class，没有任何 data members。

<img src = 'vertex3d4.png' width = '80%'>

图 3.5b 虚拟继承，使用 Virtual Table Offset Strategy 所产生的数据布局（译注：原书图 3.5b 把 Vertex 写为 Vertex2d，与书中程序代码不符，所以我全部改为 Vertex）

## 3.5 对象成员的效率（Object Member Efficiency）

下面数个测试，旨在测试聚合（aggregation）、封装（encapsulation），以及继承（inheritance）所引发的额外负担的程度。所有测试都是以个别局部变量的加法、减法、赋值（assign）等操作的存取成本为依据。下面就是个别的局部变量：
```cpp
float pA_x = 1.725, pA_y = 0.875, pA_z = 0.478;
float pB_x = 0.315, pB_y = 0.317, pB_z = 0.838;
```
每个表达式需执行一千万次，如下所述（当然啦，一旦坐标点的表现方式有变化，运算语法也就得随之变化）：
```cpp
for(int iters = 0; iters < 10000000; iters++)
{
    pB_x = pA_x - pB_z;
    pB_y = pA_y + pB_x;
    pB_z = pA_z + pB_y;
}
```
我们首先针对三个 float 元素所组成的局部数组进行测试：
```cpp
enum fussy { x, y, z};
for(int iters = 0; iters < 10000000; iters++)
{
    pB[ x ] = pA[ x ] - pB[ z ];
    pB[ y ] = pA[ y ] + pB[ x ];
    pB[ z ] = pA[ z ] + pB[ y ];
}
```
第二个测试是把同样的数组元素转换为一个 C struct 数据抽象类型，其中的成员皆为 float，成员名称是 x，y，z:
```cpp
for(int iters = 0; iters < 10000000; iters++)
{
    pB.x = pA.x - pB.z;
    pB.y = pA.y + pB.x;
    pB.z = pA.z + pB.y;
}
```
更深一层的抽象化，是作出数据封装，并使用 inline 函数。坐标点现在以一个独立的 Point3d class 来表示。我尝试两种不同形式的存取函数，第一，我定义一个 inline 函数，传回一个 refernce，允许它出现在 assignment 运算符的两端：
```cpp
class Point3d{
public:
    Point3d(float xx = 0.0, float yy = 0.0, float zz = 0.0)
        : _x(xx), _y(yy), _z(zz) { }
    float& x() { return _x; }
    float& y() { return _y; }
    float& z() { return _z; }
private:
    float _x, _y, _z;
};
```
那么真正对每一个坐标元素的存取操作应该是像这样：
```cpp
for(int iters = 0; iters < 10000000; iters++)
{
    pB.x() = pA.x() - pB.z();
    pB.y() = pA.y() + pB.x();
    pB.z() = pA.z() + pB.y(); 
}
```
我所定义的第二种存取函数形式是，提供一对 get/set 函数：
```cpp
float x() { return _x; }    // 译注：此即get函数
void x(float newX)          // 译注：此即set函数
        { _x = newX; }
```
于是对每一个坐标值的存取操作应该像这样：
```cpp
pB.x(pA.x() - pB.z());
```
表格 3.1 列出两种编译上述各种测试的结果。只有当两个编译器的效率有明显差异时，我才会把两者分别列出。

表格3.1 不断加强抽象化程度之后，数据的存取效率：
||<div style = "width:80pt">优化</div>|<div style = "width:80pt">未优化</div>|
|-|:-:|:-:|
|个别的局部变量|0.80|1.42|
|局部数组|||
|CC|0.80|2.55|
|NCC|0.80|1.42|
|struct之中有public成员|0.80|1.42|
|class之中有inline Get函数|||
|CC|0.80|2.56|
|NCC|0.80|3.10|
|class之中有inline Get & Set函数|||
|CC|0.80|1.74|
|NCC|0.80|2.87|
这里所显示的重点在于，如果把优化开关打开，“封装”就不会带来执行期的效率成本。使用 inline 存取函数亦然。

我很奇怪为什么在 CC 之下存取数组，几乎比 NCC 慢两倍，尤其是数组存取所牵扯的只是 C 数组，并没有用到任何复杂的 C++ 特性。以为程序代码产生（code generation）专家将这种反常现象解释为 “一种奇行怪癖 ······ 与特定的编译器有关”。或许是真的，但它发生在我正用来开发软件的编译器身上耶！我决定挖掘其中秘密。叫我 “爱挑毛病的乔治” 吧，如果你喜欢的话！如果你对这个题目不感兴趣，请直接跳往下一个主题。

在下面的 assembly 语言输出片段中，l.s 表示加载（load）一个单精度浮点数，s.s 表示存储（store）一个单精度浮点数，sub.s 表示将两个单精度浮点数相减。下面是两种编译器的 assembly 语言输出结果，它们都加载两个值，将某一个减去另一个，然后储存其结果。在效率较差的 CC 编译器中，每一个局部变量的地址被计算并放进一个缓存器之中（addu 表示无正负号的加法）：
```cpp
// CC assembler output
# 13 pB[ x ] = pA[ x ] - pB[ z ];
    addu    $25, $sp, 20
    l.s     $f4, 0($25)
    addu    $24, $sp, 8
    l.s     $f6, 8($24)
    sub.s   $f8, $f4, $f6
    s.s     $f8, 0($24)
```
而在 NCC 编译器的 assembly 输出中，加载 (load) 步骤直接计算地址：
```cpp
// NCC assembler output
# 13 pB[ x ] = pA[ x ] - pB[ z ];
    l.s     $f4, 20($sp)
    l.s     $f6, 16($sp)
    sub.s   $f8, $f4, $f6
    s.s     $f8, 8($sp)
```
如果局部变量被存取多次，CC 策略或许比价有效率。然而对于单一存取操作，把变量地址放到一个缓存器中很明显地增加了表达式的成本。不论哪一种编译器，只要把优化开关打开，两段码就会变得相同，在其中，循环内的所有运算都会以缓存器内的数值来执行。

让我下一个结论：如果没有把优化开关打开，就很难猜测一个程序的效率表现，因为程序代码潜在性地受到专家所谓的 “一种奇行怪癖 ······ 与特定编译器有关” 的魔咒影响。在你开始 “程序代码层面的优化操作” 以加速程序的运行之前，你应该先却是地测试效率，而不是靠着推论与常识判断。

在下一个测试中，我首先要介绍 Point 抽象化的一个三层单一继承表示法，然后再介绍 Point 抽象化的一个虚拟继承表达法。我要测试直接存取和 inline 存取（多重继承并不适用于这个模型，所以我决定放弃它）。三层单一继承表达法如下：
```cpp
class Point1d { ... };                     // 维护x
class Point2d : public Point1d { ... };    // 维护y
class Point3d : public Point3d { ... };    // 维护z
```
“单层虚拟继承” 是从 Point1d 中虚拟派生出 Point2d；“双层虚拟继承” 则又从 Point2d 中虚拟派生出 Point3d。表格 3.2 列出了两种编译器的测试结果。同样地，只有当两种编译器的效率有明显不足时，我才会把两者分别列出。

表格 3.2 在继承模型之下的数据存取
||<div style = "width:80pt">优化<div>|<div style = "width:80pt">未优化<div>|
|-|:-:|:-:|
|单一继承|||
|直接存取|0.80|1.42|
|使用inline函数|||
|CC|0.80|2.55|
|NCC|0.80|3.10|
|虚拟继承（单层）|||
|直接存取|1.60|1.94|
|使用 inline 函数|||
|CC|1.60|2.75|
|NCC|1.60|3.30|
|虚拟继承（双层）|||
|直接存取|||
|CC|2.25|2.74|
|NCC|3.04|3.68|
|使用 inline 函数|||
|CC|2.25|3.22|
|NCC|2.50|3.81|

单一继承应该不会影响测试的效率，因为 members 被连续存储于 derived class object 中，并且其 offset 在编译时期就已知了。测试结果一如预期，和表格 3.1 中的抽象数据类型结果相同。这一结果在多重继承的情况下应该也是相同的，但我不能确定。

其次，值得注意的是，如果把优化关闭，以常识来判断，我们说效率应该相同（对于 “直接存取” 和 “inline 存取” 两种做法）。然而实际上却是 inline 存取比较慢。我们再次得到教训：程序员如果关心其程序效率，应该实际测试，不要光凭推论或常识判断或假设。另一个需要注意的是，优化操作并不一定总是能够有效运行，我不只一次以优化方式来编译一个已通过编译的正常程序，却以失败收场。

虚拟继承的效率令人失望！两种编译器都没能识别出对 “继承而来的 data member ptld::_x” 的存取是通过一个非多态对象（因而不需要执行期的间接存取）。两个编译器都会对 ptld::_x (及双层虚拟继承中的pt2d::_y) 产生间接存取操作，虽然其在 Point3d 对象中的位置早在编译时期就固定了。“间接性” 压抑了 “把所有运算都移往缓存器执行” 的优化能力。但是间接性并不会严重影响非优化程序的执行效率。

## 3.6 指向Data Members的指针（Pointer to Data Members)

指向 data members 的指针，是一个优点神秘但颇有用处的语言特性，特别是如果你需要详细调查 class members 的底层布局的话。这样的调查可用以决定 vptr 是放在 class 的起始处或是尾端。另一个用途展现于 3.2 节，可用来决定 class 中的 access section 的次序。一如我曾说过，那是一个神秘但有时候有用的语言特性。

考虑下面的 Point3d 声明。其中有一个 virtual function，一个 static data member，以及三个坐标值：
```cpp
class Point3d{
public:
    virtual ~Point3d();
    // ...
protected:
    static Point3d origin;
    float x, y, z;
};
```
每一个 Point3d class object 含有三个坐标值，依次为 x,y,z，以及一个 vptr。至于 static data member origin，将被放在 class object 之外。唯一可能因编译器不同而不同的是 vptr 的位置。C++ Standard 允许 vptr 被放在对象中的任何位置：在起始处，在尾端，或是在各个 members 之间。然而实际上，所有编译器不是把 vptr 放在对象的头部，就是放在对象的尾部。

那么，取某个坐标成员的地址，代表什么意思？例如，以下操作所得到的值代表什么：
```cpp
&Point3d::z;    // 译注：原书的&3d_point::z; 应为笔误
```
上述操作将得到 z 坐标在 class object 中的偏移量（offset）。最低限度其值将是 x 和 y 的大小总和，因为 C++ 语言要求同一个 access level 中的 members 的排列次序应该和其声明次序相同。

然而 vptr 的为位置就没有限制。不过容我再说一次，实际上 vptr 不是放在对象的头部，就是放在对象的尾部。在一部 32 位机器上，每一个 float 是 4 bytes，所以我们应该期望刚才获得的值要不是 8，就是 12（在 32 位机器上一个 vptr 是 4bytes）。

然而，这样的期望却还少 1 byte。对于 C 和 C++ 程序员而言，这多少算是个有点年代的错误了。

如果 vptr 放在对象的尾部，则三个坐标值在对象布局中的 offset 分别是 0, 4, 8.如果 vptr 放在对象的起头，则三个坐标值在对象布局中的 offset 分别是 4, 8, 12. 然而你若去取 data members 的地址，传回的值总是多 1，也就是 1, 5, 9 或 5, 9, 13 等等。你知道为什么 Bjarne 决定要这么做吗？

译注：如何取 `&Point3d::z` 的值并打印出来？以下是示范做法：
```cpp
printf("&Point3d::x = %p\n", &Point3d::x); // 结果VC5:4, BCB3:5
printf("&Point3d::y = %p\n", &Point3d::y); // 结果VC5:8, BCB3:9
printf("&Point3d::z = %p\n", &Point3d::z); // 结果VC5:C, BCB3:D

```
注意，不可以这么做：
```cpp
cout << "&Point3d::x = " << &Point3d::x << endl;
cout << "&Point3d::y = " << &Point3d::y << endl;
cout << "&Point3d::z = " << &Point3d::z << endl;
```
否则会得到错误消息：
> error C2679:binary'<<':no operator defined which takes a right-hand operand of type 'float Point3d:: *'(or there is no acceptable conversion)(new behavior; please see help)

我使用的编译器是 Microsoft Visual C++ 5.0。为什么执行结果并不如书中所说增加 1 呢？原因可能是 Visual C++ 做了特殊处理，其道理与本章一开始对于 empty vitual base class 的讨论相近！

问题在于，如果区分一个 “没有指向任何 data member ” 的指针和一个指向 “第一个 data member” 的指针？考虑这样的例子：
```cpp
float Point3d::*p1 = 0;
float Point3d::*p2 = &Point3d::x;
// 译注：Point::*的意思是：“指向Point3d data member”的指针类型

// 喔欧：如何区分
if(p1 == p2){
    cout << "p1 & p2 contain the same value -- ";
    cout << " they must address the same member!" << endl;
}
```
为了区分 p1 和 p2，每一个真正的 member offset 之都被加上 1.因此，不论编译器或使用者都必须记住，在真正使用该值以指出一个 member 之前，请先减掉 1.

认识 “指向 data members 的指针” 之后，我们发现，要解释：
```cpp
&Point3d::z;    // 译注：原书的&3d_point::z; 应为笔误
```
和
```cpp
&origin.z
```
之间的差异，就非常明确了。鉴于 “取一个 nonstatic data member 的地址，将会得到它在 class中的 offset”，取一个 “绑定与真正 class object 身上的 data member” 的地址，将会得到该 member 在内存中的真正地址。把:
```cpp
&origin.z
```
所得的结果减（译注：原文为加，错误）z 的偏移值（相对于 origin 起始地址），并加1（译注：原文为减，错误），就会得到 origin 起始地址。上一行的返回值类型应该是：
```cpp
float*
```
而不是
```cpp
float Point3d::*
```
由于上述操作所参考的是一个特定实例，所以取一个 static data member 的地址，意义也相同。

在多重继承之下，若要将第二个（或后继）base class 的指针和一个 “与 derived class object 绑定” 之 member 结合起来，那么将会因为 “需要加入 offset 值” 而变得相当复杂。例如，假设我们有：
```cpp
struct Base1 { int val1; }
struct Base2 { int val2; }
struct Derived : Base1, Base2 { ... };

void func1(int Derived::*dmp, Derived *pd)
{
    // 期望第一个参数得到的是一个“指向derived class之member”的指针
    // 如果传进来的却是一个“指向base class之member”的指针，会怎样呢
    pd->*dmp;
}

void func2(Derived *pd)
{
    // bmp将成为1
    int Base2::*bmp = &Base2::val2;
    // 喔欧，bmp == 1,
    // 但是在Derived中，val2 == 5
    func1(bmp, pd);
}
```
当 bmp 被作为 func1() 的第一个参数时，他的值就必须因介入的 Base1 class 的大小而调整，否则 func1() 中这样的操作：
```cpp
pd->*dmp;
```
将存取到 Base1::val1，而非程序员所以为的 Base2::val2. 要解决这个问题，必须：
```cpp
// 经由编译器内部转换
func1(bmp + sizeof(Base1), pd);
```
然而，一般而言，我们不能够保证 bmp 不是 0，因此必须特别留意之：
```cpp
// 内部转换
// 防范 bmp == 0
func1(bmp ? bmp + sizeof(Base1) : 0, pd);
```

译注：我实际写了一个小程序，打印上述各个 member 的 offset 值：
```cpp
printf("&Base1::val1 = %p \n", &Base1::val1);        // (1)
printf("&Base2::val2 = %p \n", &Base2::val2);        // (2)
printf("&Derived::val1 = %p \n", &Derived::val1);    // (3)
printf("&Derived::val2 = %p \n", &Derived::val2);    // (4)
```
经过 Visual C++ 5.0 编译后，执行结果竟然都是 0。(1)(2)(3) 都是 0 是可以理解的（为什么不是 1？可能是因为 Visual C++ 有特殊处理：稍早 p.131 中我的另一个译注曾有说明）。但为什么 (4) 也是 0，而不是 4？是否编译器已经内部处理过了呢？很可能（我只能如此猜测）。

如果我把Derived的声明改为：
```cpp
struct Derived : Base1, Base2 {int vald; };
```
那么：
```cpp
printf("&Derived::vald = %p \n", &Derived::vald);
```
将得到8，表示vald的前面的确有val1和val2.

### “指向Members的指针”的效率问题

下面的测试企图获得一些测试数据，让我们了解，在 3D 坐标点的各种 class 的实现方式之下，使用 “指向 members 的指针” 所带来的影响。一开始的两个例子并没有继承关系，第一个例子是要取得一个 “已绑定的 member” 的地址：
```cpp
float *ax = &pA.x;
```
然后施以赋值（assignment）、加法、减法操作如下：
```cpp
*bx = *ax - *bz;
*by = *ay + *bx;
*bz = *az + *by;
```
第二个例子则是针对三个 members，取得 “指向 data member 之指针” 的地址：
```cpp
float Point3d::*ax = &Point3d::x;
```
而赋值（assignment）、加法和减法等操作，都是使用 “指向 data member 之指针” 语法，把数值绑定到对象 pA 和 pB 中：
```cpp
pB.*bx = pA.*ax - pB.*bz;
pB.*by = pA.*ay + pB.*bx;
pB.*bx = pA.*az + pB.*by;
```
回忆 3.5 节中的直接存取操作，平均时间是 0.8 秒（当优化开启时）或 1.42 秒（当优化关闭时）。现在再执行者两个测试，结果列于表格 3.3 中。

表格3.3存取 Nonstatic Data Member
||<div style = "width:80pt">优化|<div style = "width:80pt">未优化</div>|
|-|:-:|:-:|
|直接存取（请参考3.5节）|0.80|1.42|
|指针指向已绑定的Member|0.80|3.04|
|指针指向Data Member|||
|CC|0.80|5.34|
|NCC|4.04|5.34|

未优化的结果正如预期。也就是说，为每一个 “member 存取操作” 加上一层间接性（经由已绑定的指针），会使执行时间多出一倍不止。以 “指向 member 的指针” 来存取数据，再一次几乎用掉了双倍时间。要把 “指向 member 的指针” 绑定到 class object 身上，需要额外地把 offset 减 1。更重要的是，当然，优化可以使所有三种个策略的效率变得已知，唯 NCC 编译器除外。你不妨注意一下，在这里，NCC 编译器所产生的码在优化情况下有着令人震惊的低效率，这反映出它所产生出来的 assembly 码有着可怜的优化操作，这和 C++ 程序代码如何表现并无直接关系——要知道，我曾检验过 CC 和 NCC 产生出来的未优化 assembly 码，两者完全一样！

下一组测试要看看 “继承” 对于 “指向 data member 的指针” 所带来的效率冲击。在第一个例子中，独立的 Point class 被重新设计为一个三层单一继承体系，每一个 class 有一个 member:
```cpp
class Point { ... };                    // float x;
class Point2d : public Point { ... };   // float y;
class Point3d : public Point2d { ... }; // float z;
```
第二个例子仍然是三层单一继承体系，但导入一层虚拟继承：Point2d 虚拟派生自 Point。结果，每次对于 Point:x 的存取，将是对一个 virtual base class data member 的存取。最后一个例子，实用性很低，几乎纯粹是好奇心的驱使：我加上第二层虚拟继承，使 Point3d 虚拟派生自 Point2d。表格 3.4 显示测试结果。注意：
由于 NCC 优化的效率在各项测试中的结果都是一致的，我已经把它从表格中剔除了。

表格 3.4 “指向 Data Member 的指针” 存取方式
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|没有继承|0.80|5.34|
|单一基层（三层）|0.80|5.34|
|虚拟继承（单层）|1.60|5.44|
|虚拟继承（双层|3.4|5.51|

由于被继承的 data members 是直接存放在 class object 之中，所以继承的引入一点也不会影响这些码的效率。虚拟继承所带来的主要冲击是，它妨碍了优化的有效性。为什么呢？在两个编译器中，每一层虚拟继承都导入一个额外层次的间接性。在两个编译器中，每次存取 Point::x，像这样：
```cpp
pB.*bx
```
会被转换为：
```cpp
&pB->__vbcPoint + (bx - 1)
```
而不是转换最直接的：
```cpp
&pB + (bx - 1)
```
额外的间接性会降低“把所有的处理都搬移到缓存器中执行”的优化能力。

[第 2 章 构造函数语意学（The Semantics of Constructors）](/markdown/The_Semantics_of_Constructors.md)|[第 4 章 Function 语意学（The Semantics of Function）](/markdown/The_Semantics_of_Function.md)
