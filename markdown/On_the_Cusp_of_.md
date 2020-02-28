# 第 7 章 站在对象模型的尖端 On the Cusp of the Ojbect Model

译注：本章大量沿用两个原文词汇：instantiate（动词）和 instantiation（名词）。你不容易在一般的字典上查到这两个词。牛津计算机字典上对于instantiation 的解释是：

1. The creation of a particular instance of an object class, generic unit, or template.
2.  The application of a parameterized abstract data type to a particular set of parameters.

在本章中应采用第一个解释。有些时候我会把 instantiation 译为 “实现” 或 “具现” —— “具体实现出一个实体（instance）” 的意思。

这一章我要讨论三个著名的 C++ 语言扩充性质，它们都会影响 C++ 对象。它们分别是 template、exception handling(EH) 和 runtime type identification(RTTI)。EH（以及 RTTI——可想象成是 EH 的一个副作用）对于这本书所谈到的其它语言性质而言，算是一个特例，因为我没有机会真正实现它。我的讨论是以 [CHASE94]、[LAJOIE94a]、[LAJOIE94b]、[LENKOV92]以及 [SUN94a] 为基础的。

# 7.1 Template

C++ 程序设计的风格及习惯，自从 1991 年的 cfront 3.0 引入 templates 之后就深深地改变了。原本 template 被视为是对 container class 如 List 和 Arrays 的一项支持，但现在它已经成为通用程序设计（也就是Standard Template Library, STL）的基础。它也被用于属性混合（如内存配置策略，[BOOCH93]）或互斥（mutual exclusion）机制（适用于线程同步化控制）的参数化技术之中。它甚至被使用于一项所谓的 template metaprograms 技术：class expression templates 将在编译时期而非执行期被评估（evaluated），因而带来重大的效率提升（请参考 [VELD95]）。

然而，如果我说 template 是令程序员最挫败感的一个主题，恐怕也是真的。错误消息可能远在离真正问题相距十万八千里处就产生了，编译时间提高了不少，而程序员开始极端害怕修改一个内有多重相依关系的 .H 文件，特别是如果他正在和 “臭虫” 奋战的时候。程序大小像气球一样地膨胀也是常有的事。犹有进者，template 的所有行为都超越了一般程序人员的理解能力，那些人但求能够完成手上的工作就好。如果上述问题持续存在，他们可能会认为 template 是一个障碍而不是一个帮助。你很容易看到一个被命名的 template 专家的人，被一群项目成员团团围住，要求解决问题并将产生出来的 template 优化。

这一节的焦点放在 template 的语意上面，我们将讨论 templates 在编译系统中 “何时”、“为什么” 以及 “如何” 发挥其功能。下面是有关 template 的三个主要讨论方向：
1. template 的声明。基本上来说就是当你声明一个 template class、template class member function 等等时，会发生什么事情。
2. 如果 “具现（instantiates）” 出 class object 以及 inline nonmember，以及 member template functions，这些是 “每一个编译单位都会拥有一份实体” 的东西。
3. 如何 “局限（instantiates）”出 nonmember 以及 member template functions，以及 static template class members，这些都是 “每一个可执行文件中只需要一份实体” 的东西。这也就是一般而言 template 所带来的的问题。

我使用 “具现”（instantiation）这个字眼来表示 “行程（process）将真正的类型和表达式绑定到 template 相关形式参数（format parameters）上头” 的操作。举个例子，下面是一个 template function：
```cpp
template<class Type>
Type
min(const Type &t1, const Type &t2) { ... }
```
用法如下：
```cpp
min(1.0, 2.0);
```
于是行程就把 Type 绑定为 double 并产生 min() 的一个程序文字实体（并适当施以 “mangling” 手术，给他一个独一无二的名称），其中 t1 和 t2 的类型都是 double。

### Template 的 “具现” 行为（Template Instantiation）

考虑下面的 template Point class:
```cpp
template <class Type>
class Point
{
public:
    enum Status{ unallocated, normalized};

    Point(Type x = 0.0, Type y = 0.0, Type z = 0.0);
    ~Point();

    void* operator new(size_t);
    void operator delete(void*, size_t);
    // ...
private:
    static Point<Type> *freeList;
    static int chunkSize;
    Type _x, _y, _z;
};
```
首先，当编译器看到 template class 声明时，它会作出什么反应？在实际程序中，什么反应也没有！也就是说，上述的 static data members 并不可用。nested enum 或其 enumerators 也一样。

虽然 enum Status 的真正类型在所有的 Point instantiation 中都一样，其 enumerators 也是，但它们每一个都只能通过 template Point class 的某个实体来存取或操作。因此我们可以这样写：
```cpp
// ok :
Point<float>::Status s;
```
但不能这样写：
```cpp
// error :
Point::Status s;
```
即使两种类型抽象地来说是一样的（而且，最理想情况下，我们希望这个 enum 只有一个实体被产生出来。如果不是这样，我们可能会想要把这个 enum 抽出到一个 nontemplate base class 中，以避免多份拷贝）。

同样的道理，freeList 和 chunkSize 对程序而言也还不可用。我们不能够写：
```cpp
// error :
Point::freeList;
```
我们必须明确地指定类型，才能使用 freeList:
```cpp
// ok :
Point<float>::freeList;
```

像上面这样使用 static member，会使其中一份实体于 Point class 的 float instantiation 在程序中产生关联。如果我们写：
```cpp
// ok : 另一个实体（instance）
Point<double>::freeList;
```
就会出现第二个 freeList 实体，与 Point class 的 double instantiation 产生关联。

如果我们定义一个指针，指向特定的实体，像这样：
```cpp
Point<float> *ptr = 0;
```
这一次，程序中什么也没有发生。为什么呢？因为一个指向 class object 的指针，本省并不是一个 class object，编译器不需要知道与该 class 有关的任何 members 的数据或 object 布局数据。所以将 “Point 的一个 float 实体” 具现也就没有标有。在 C++ Standard 完成之前，“声明一个指针指向某个 template class” 这件事情并非未被强制定义，编译器可以自行决定要或不要将 template “具现” 出来。cfront 就是这么做的（这使得某些程序员大感困窘）！如今 C++ Standard 已经禁止编译器这样做。

如果不是 pointer 而是 reference，又如何？假设：
```cpp
const Point<float> &ref = 0;
```
是的，它真的会具现出一个 “Point 的 float 实体” 来。这个定义的真正语意会被扩展为：
```cpp
// 内部扩展
Point<float> temporary(float(0));
const Point<float> &ref = temporary;
```
为什么呢？因为 reference 并不是无物（no object）的代名词。0 被视为整数，必须被转换为以下类型的一个对象：
```cpp
Point<float>
```
如果没有转换的可能，这个定义就是错误的，会在编译时被挑出来。

所以，一个 class object 的定义，不论是由编译器暗中地做（像稍早程序代码中出现过的 temporary），或是有程序员像下面这样明确地做：
```cpp
const Point<float> origin;
```
都会导致 template class 的 “具现”，也就是说，float instantiation 的真正对象布局会被产生出来。回顾先前的 template 声明，我们看到 Point 有三个 nonstatic members，每一个的类型都是 Type。Type 现在被绑定为 float，所以 origin 的配置空间必须足够容纳三个 float 成员。

然而，member functions（至少对于那些未被使用过的）不应该被 “实体” 化，只有在 member functions 被使用的时候，C++ Standard 才要求它们被 “具现” 出来。当前的编译器并不精确遵循这项要求。之所以有使用者来主导 “具现”（instantiation）规则，有两个主要原因：
1. 空间和时间效率的考虑。如果 class 中有 100 个 member functions，但你的程序只针对某个类型使用其中两个，针对另一个类型使用其中五个，那么将其它 193 个函数都 “具现” 将会花费大量的时间和空间。
2. 尚未实现的机能。并不是一个 template 具现出来的所有类型就一定能够完整支持一组 member functions 所需要的所有运算符。如果只 “具现” 那些真正用到的 member functions，template 就能够支持那些原本可能会造成编译时期错误的类型（types）。

举个例子，origin 的定义需要调用 Point 的 default constructor 和 destructor，那么只有这两个函数需要被 “具现”。类似的道理，当程序员写：
```cpp
Point<float>*p = new Point<float>;
```
时，只有（1）Point template 的 float 实例，（2）new 运算符，（3）default constructor 需要被 “具现” 化。有趣的是，虽然 new 运算符是这个 class 的一个 implicity static member，以至于它不能够直接处理其中任何一个 nonstatic members，但它还是依赖真正的 template 参数类型，以为它的第一参数 size_t 代表 class 的大小。

这些函数在什么时候 “具现” 出来呢？当前流行两种策略：
1. 在编译时期。那么函数将 “具现” 于 origin 和 p 存放的那个文件中。
2. 在链接时候。那么编译器会被一些辅助工具重新激活。template 函数实体可能被放在这个文件中、别的文件中、或一个分离的储存位置上。

稍后的小节将对函数的 “具现” 化作更详细的讨论。

[GARGILL95] 曾经提到有趣的一点，在 “int 和 long 一致”（或 “double 和 long double 一致”）的结构之中，两个类型具现操作：
```cpp
Point<int> pi;
Point<long> pi;
```
应该产生一个还是两个实体呢？目前我知道的所有编译器都产生两个实体（可能有两组完整的 member functions）。C++ Standard 并未对此有什么强制规定。

### Template的错误报告（Error Reporting within a Template）

考虑下面的 template 声明：
```cpp
(1) tempalte<class T>
(2) class Mumble
(3) {
(4) public$:
(5)     Mumble(T t = 1024)
(6)         : _t(t)
(7)     {
(8)         if(tt != t)
(9)             throw ex ex;
(10)    }
(11)private:
(12)    T tt;
(13)}
```
这个 Mumble template class 的声明内含一些既露骨又潜沉的错误：

1. L4：使用 $ 字符是不对的。这项错误有两方面。第一，$ 并不是一个可以合法用于标识符的字符；第二，class 声明中只允许有 public、protected、private 三个卷标（labels），$ 的出现使 public$ 不成为 public。第一点是词汇（lexical）上的错误，第二点则是造句/解析（syntactic/parsing）上的错误。
2. L5：t 被初始化为常量 1024，或许可以，也或许不可以，视 T 的真实类型而定。一般而言，只有 template 的各个实体才诊断得出来。
3. L6：_t 并不是哪一个 member 的名称，tt 才是。这种错误一般会在 “类型检验” 这个阶段被找出来。是的，每一个名称必须绑定于一个定义身上，要不就会产生错误。
4. L8：!= 运算符可能已定义好，但也可能还没有，视 T 的真正类型而定。和第 2 点一样，只有 template 的各个实体才诊断得出来。
5. L9：我们意外地键入 ex 两次。这个错误会在编译时期的解析（parsing）阶段被发现。C++ 语言中一个合法的句子不允许一个标识符紧跟在另一个标识符之后。
6. L13：我们忘记以一个分号作为 class 声明的结束。这项错误也会在编译时期的语句分析（parsing）阶段被发现。

在一个 nonmember class 声明中，这六个既露骨又潜沉的错误会被编译器挑出来。但 template class 却不同。举个例子，所有与类型有关的检验，如果牵涉到 template 参数，都必须延迟到真正的具现操作（instantiation）发生，才得为之。也就是说，L5 和 L8 的潜在错误（上述 2，4 两项）会在每个具现操作（instantiation）发生时被检查出来并记录之，其结果将因不同的实际类型而不同。

于是，如果
```cpp
Memble<int> mi;
```
则 L5 和 L8 是正确的，而如果：
```cpp
Memble<int*> pmi;
```
那么 L8 正确而 L5 错误，因为你不能够将一个整数常量（除了 0）指定给一个指针。面对这样的声明：
```cpp
class SmallInt{
public:
    SmallInt(int);
    // ...
}
```
由于其 != 运算符并未定义，所以下面的句子：
```cpp
Mumble<SmallInt> smi;
```
会造成 L8 错误，而 L5 正确。当然，下面的例子：
```cpp
Mumble<SmallInt*> psmi;
```
又造成 L8 正确而 L5 错误。

那么，什么样的错误会在编译器处理 template 声明时被标示出来？这里有一部分和 template 的处理策略有关。cfront 对 template 的处理是完全解析（parse）但不做类型检验；只有在每一个具现操作（instantiation）发生时才做类型检验。所以在一个 parsing 策略之下，所有词汇（lexing）错误和解析（parsing）错误都会在处理 template 声明的过程中被标示出来。

语汇分析器（lexical analyzer）会在 L4 捕捉到一个不合法的字符，解析器（parser）会这样标示它：
```cpp
public$:    // caught
```
标示这是一个不合法的卷标（label）。解析器（parser）不会把 “对一个未命名的 member 做出参考操作” 视为错误：
```cpp
_t(t)   // not caught
```
但它会抓出 L9 “ex 出现两次” 以及 L13 “缺少一个分号” 这两种错误。

在一个十分普遍地替代策略中（例如 [BALL92a] 中所记录），template 的声明被收集成为一系列的 “lexical tokens”，而 parsing 操作延迟，直到真正有具现操作（instantiation）发生时才开始。每当看到一个 instantiation 发生，这组 token 就被推往 parser，然后调用类型检验等等。面对先前出现的那个 template 声明，“lexical tokenizing” 会指出什么错误吗？事实上很少，只有 L4 所使用的不合法字符挥被指出。其余的 template 声明都被解析为合法的 tokens 并被收集起来。

目前的编译器，面对一个 template 声明，在它被一组实际参数具现之前，只能施行以有限的错误检查。tempalte 中那些与语法无关的错误，程序员可能认为十分明显，编译器却让它通过了，只有在特定实体被定义之后，才会发出抱怨。这是目前实现技术上的一个大问题。

Nonmember 和 member template functions 在具现行为（instantiation）发生之前也一样没有做到完全的类型检验。这导致某些十分露骨的 template 错误声明竟然得以通过编译。例如下面的 template 声明：
```cpp
tempalte<class type>
class Foo
{
public:
    Foo();
    type val();
    void val(type v);
private:
    type _val;
};
```

> @zl : 哪里由露骨的错误？

不论是 cfront 或 Sun 编译器或 Borland 编译器，都不会对以下程序代码产生怨言：
```cpp
// 目前各家编译器都不会显示出以下定义的语句合法而语意错误：
// (a) bogus_member不是class的一个member function
// (b) dbx 不是 class的一个data member
template<class type>
double Foo<type>::bogus_member() { return this->dbx; }
```
再说一次，这是编译器设计者自己的决定。Template facility 并没有说或不允许对 template 声明的类型部分有更严酷的检验。当然，像这样的错误是可以被发现并标示出来的，只不过没有人去做罢了。

### Template 中的名称决议方式（Name Resolution within a Template）

你必须能够区分以下两种意义。一种是 C++ Standard 所谓的 “scope of the template definition”，也就是 “定义出 template” 的程序。另一种是 C++ Standard 所谓的 “scope of the template instantiation”，也就是 “具现出 template” 的程序。

第一种情况举例如下：
```cpp
// scope of the template definition
extern double foo(double);

template<class type>
class ScopeRules
{
public:
    void invariant(){
        _member = foo(_val);
    }
    type type_dependent(){
        return foo(_member);
    }
    // ...
private:
    int _val;
    type _member;
};
```
第二种情况举例如下：
```cpp
// scope of the template instantiation
extern int foo(int);
// ...
ScopeRules<int> sr0;
```
在 ScopeRules template 中有两个 foo() 调用操作。在 “scope of template definition” 中，只有一个 foo() 函数声明位于 scope 之内。然而在 “scope of template innstantiation” 中，两个 foo() 函数声明都位于 scope 之内。如果我们有一个函数调用操作：
```cpp
// scope of the template instantiation
sr0.invariant();
```
那么，在 invariant() 中调用的究竟是哪一个 foo() 函数实体呢？
```cpp
// 调用的是哪一个foo()函数实体
_member = foo(_val);
```
在调用操作的那一点上，程序中的两个函数实体是：
```cpp
// scope of the template declaration
extern double foo(double);

// scope of the tempalte instantiation
extern int foo(int);
```
而 _val 的类型是 int。那么你认为选中的是哪一个呢？（提示：除了瞎猜之外，唯一能够正确回答的方法，就是知道正确答案☺）。结果，被选中的是直觉之外的那一个：
```cpp
// scope of the template declaration
extern double foo(double);
```
Template 之中，对于一个 nonmember name 的决议结果是根据这个 name 的使用是否与 “用以具现出该 template 的参数类型” 有关而决定的。如果其使用互不相关，那么就以 “scope of the template declaration” 来决定 name。如果其使用互有关联，那么就以 “scope of the template instantiation” 来决定 name。在第一个例子中，foo() 与用以具现ScopeRules 的参数类型无关：
```cpp
// the resolution of foo() is not
// dependent on the template argument
_member = foo(_val);
```
这是因为 _val 的类型是 int；_val 是一个 “类型不会变动” 的 template class member。也就是说，被用来具现出这个 template 的真正类型，对于_val 的类型并没有影响。此外，函数的决议结果只和函数的原型(signature)有关，和函数的返回值没有关联。因此，_member 的类型并不会影响哪一个 foo() 实体被选中。foo() 的调用与 template 参数毫无关联！所以调用操作必须根据 “scope of the template declaration” 来决议。在此 scope中，只有一个 foo() 候选者（注意，这种行为不能够以一个简单的宏扩展——比如使用一个 #define宏——重现之）。

让我们另外看看 “与类型相关”（type-dependant）的用法：
```cpp
sr0.type_dependent();
```
这个函数的内容如下：
```cpp
return foo(_member);
```
它究竟会调用哪一个 foo()呢?

这个例子很清楚地与 template 参数有关，因为该参数将决定 _member 的真正类型。所以这一次 foo() 必须在 “scope of the template instantiation” 中决议，本例中这个 scope 有两个 foo() 函数声明。由于 _member 的类型在本例中为 int，所以应该是 int 版的 foo() 出线。如果 ScopeRules 是以 double 类型具现出来，那么就应该是 double 版的 foo() 出线。如果 ScopeRules 是以 unsigned int 或 long 类型具现出来，那么 foo() 调用操作就暧昧不明。最后，如果 ScopeRules 是以某一个 class 类型具现出来，而该 class 没有针对 int 或 double 实现出 conversion 运算符，那么 foo() 调用操作会被标示为错误。不管如何改变，都是由 “scope of the template instantiation” 来决定，而不是由 “scope of the template declaration” 决定。

这意味着一个编译器必须保持两个 scope contexts:
1. “scope of the tempalte declaration”，用以专注于一般的 template class。
2. “scope of the template instantiation”，用以专注于特定的实体。

编译器的决议（resolution）算法必须决定哪一个才是合适的 scope，然后在搜寻适当的 name。

### Member Function 的具现行为（Member Function Instantiation）

对于 template 的支持，最困难莫过于 template function 的具现（instantiation）。目前的编译器提供了两种策略：一个是编译时期策略，程序代码必须在 program text file 中备妥可用；另一个是链接时期策略，有一些 meta-compilation 工具可以导引编译器的具现行为（instantiation）。

下面是编译器设计者必须回答三个主要问题：

1. 编译器如何找出函数的定义？

答案一致是包含 template program text file，就好像它是个 header 文件一样。Borland 编译器就是遵循这个策略。另一种方法是要求一个文件命名规则，例如，我们可以要求，在 Point.h 文件中发现的函数声明，其 template program text 一定要放置于文件 Point.C 或 Point.cpp 中，以此类推。cfront 就是遵循这个策略。Edison Design Group 编译器对此两种策略都支持。

2. 编译器如何能够只具现出程序中用到的 member functions？

解决方法之一就是，根本忽略这项要求，把一个已经具现出来的 class 的所有 member functions 都产生出来。Borland 即使这么做的——虽然它也提供 #progmas 让你压制（或具现出）特定实体。另一种策略就是仿真链接操作，检测看看哪一个函数真正需要，然后只为它（们）产生实体。cfront 就是这么做的，Edison Design Group 编译器对此两种策略都支持。

3. 编译器如何阻止 member definitions 在多个 .o 文件中都被具现呢？

解决方法之一就是产生多个实体，然后从链接器中提供支持，只留下其中一个实体，其余都忽略。另一个方法就是由使用者来导引 “仿真链接阶段” 的具现策略，决定哪个实体（instances）才是所需求的。

目前，不论编译时期或链接时期的具现（instantiation）策略，其弱点就是，当 template 实体被产生出来时，有时候会大量增加编译时间。很显然，这将是 template functions 第一次具现时的必要条件。然而当那些函数被并非必要地在此具现，或是当 “决定那些函数是否需要再具现” 所花的代价太大时，编译器的表现令人失望！

C++ 支持 tempalte 的原始意图可以想见是一个由使用者导引的自动具现机制（use-directed automatic instantiation mechanism），既不需要使用者的介入，也不需要相同文件有多次的具现行为。但是这已被证明是非常难以达成的任务，比任何人此刻所能想象的还要难（请参考 [STROUP94]）。ptlink，随着 cfront 3.0 版所附的原始具现工具，提供了一个由使用者执行的自动具现机制（use-driven automatic instantiation mechanism），但它实在太复杂了，即使是久经世故的人也没法一下子了解。

Edison Design Group 开发出一套第二代的 deirected-instantiation 机制，非常接近于（我所知的）template facility 原始涵义。它的主要过程如下：

1. 一个程序的程序代码别编译时，最初并不会产生任何 “template 具现体”。然而，相关信息已经被产生于 object files 之中。
2. 当 object files 被链接在一块儿时，会有一个 prelinker 程序被执行起来。它会检查 object files，寻找 template实体 的相互参考以及对应的定义。
3. 对于每一个 “参考到 template实体” 而 “该实体却没有定义” 的情况，prelinker 将该文件视为与另一个文件（在其中，实体已经具现）同类。以这种方法，就可以将必要的程序具现操作指定给特定的文件。这些都会注册在 prelinker 所产生的 .ii 文件中（放在磁盘目录 ii_file）。
4. prelinker 重新执行编译器，重新编译每一个 “.ii 文件曾被改变过” 的文件。这个过程不断重复，直到所有必要的具现操作都已完成。
5. 所有的 object files 被链接成一个可执行文件。

这种 directed-instantiation 体制的主要成本在于，程序第一次被编译时的 .ii 文件设定时间。次要成本测试必须针对每一个 “compile afterwards” 执行 prelinker，以确保所有被参考到的 templates 都存在有定义。在最初的设定以及成功地第一次链接之后，重新编译操作包含以下程序：
1. 对于每一个将被重新编译的 program text file，编译器检查其对应的 .ii 文件。
2. 如果对应的 .ii 文件列出一组要被具现（instantiated）的 templates，那些 templates（而且也只有那些 templates）会在此次编译时被具现。
3. prelinker 必须执行起来，确保所有被参考到的 templates 已经被定义妥当。

以我的观点，出现某种形式的 automated template 机制，是 “对程序员友善的 C++ 编译系统” 的一个必要组件。虽然大家也公认，目前没有任何一个系统是完美的。作为一个程序开发者，我不会使用（也不会推荐）一个没有这种机制的编译系统。

不幸的是，没有任何一个机制是没有 bugs 的。Edision Design Group 的编译器使用了一个由 cfront 2.0 引入的算法 [KOENIG90a]，针对程序中的每一个 class 自动产生一个 virtual table 的单一实体（在大部分情况下）。例如下面的 class 声明：
```cpp
class PrimitiveObject : public Geometry
{
public:
    virtual ~PrimitiveObject();
    virtual void draw();
};
```
如果它被含入于 15 个或 45 个程序源码中，编译器如何能够确保只有一个 virtual table 实体被产生出来呢？产生 15 份或 45 份实体倒还容易些!

Andy Koenig 以下面的方法解决这个问题：每一个 virtual function 的地址都被放置于 active classes 的 virtual table 中 [1]。如果取得函数地址，则表示 virtual function 的定义必定出现在程序的某个地点；否则程序就无法链接成功。此外，该函数只能有一个实体，否则也是链接不成功。那么，就被 virtual table 放在定义了该 class 之第一个 non-inline、nonpure virtual function 的文件中吧。以我们的例子而言，编译器会将 virtual table 产生在储存着 virtual destructor 的文件之中。

[1] C++ Standard 已经放松了对这一点的要求

不幸的是，在 template 之中，这种单一定义并不一定为真。在 template 所支持的 “将模块中的每一样东西都编译” 的模型下，不只是多个定义可能被产生，而且链接器也放任让多个定义同时出现，它只要选择其中一个而将其余都忽略也就是了。

好吧，真是有趣，但 Edison Design Group 的 automatic instantiation 机制做什么事呢？考虑下面这个 library 函数：
```cpp
void foo(const Point<float> *ptr)
{
    ptr->virtual_func();
}
```
virtual function call 被转换为类似这样的东西：
```cpp
// C++伪码
// ptr->virtual_func();
(*ptr->__vbtl__Point<float>[2])(ptr);
```
于是导致具现（instantiation）出 Point class 的一个 float 实体及其 virtual_func()。由于每一个 virtual function 的地址被放置于 table 之中，如果 virtual table 被产生出来，每一个 virtual function 也都必须被具现（instantiated）。这就是为什么 C++ Standard 有下面的文字说明的缘故：

> 如果一个 virtual function 被具现（instantiated）出来，其具现点紧跟在其 class 的具现点之后。

然而，如果编译器遵循 cfront 的 virtual table 实现体制，那么在 “Point 的 float 实体有一个 virtual destructor 定义被具现出来之前，这个 table 不会被产生。除非在这一点上，并没有明确使用 virtual destructor 以担保其具现行为（instantiation）。

Edision Design Group 的 automatic template 机制并不明确它自己的编译器对于第一个 non-inline、nonpure virtual function 的隐晦使用，所以并没有把它标示于 .ii 文件中。结果，链接器反而回头抱怨下面这个符号没有出现：
```cpp
__vtbl_Point<float>
```
并拒绝产生一个可执行文件。奥，真是麻烦！Automatic instantiation 在此失效！程序员必须明确地强迫将 destructor 具现出来。目前的编译系统是以 #pragma 指令来支持此需求。然而 C++ Standard 也已经扩充了对 template 的支持，允许程序员明确地要求在一个文件中将整个 class template 具现出来：
```cpp
tempalte class Point3d<float>;
```
或是针对一个 template class 的个别 member function:
```cpp
template float Point3d<float>::X() const;
```
或是针对某个个别 template function:
```cpp
template Point3d<float> operaotor+(
    const Point3d<float>&, const Poin3d<float>&);
```
在实现层面上，template instantiation 似乎拒绝全面自动化。甚至虽然每一件工作都做对了，产生出来的 object files 的重新编译成本仍然可能太高——如果程序十分巨大的话！以手动方式先在个别的 object module 中完成预先具现操作（pre-instantiation），虽然沉闷，却是唯一有效率的方法。

## 7.2 异常处理（Exception Handling)

欲支持 exception handling，编译器的主要工作就是找出 catch 字句，以处理被丢出来的 exception。这多少需要追踪程序堆栈中的每一个函数的当前作用区域（包括追踪函数中的 local class objects 当时的情况）。同时，编译器必须提供某种查询 exception objects 的方法，以知道其实际类型（这直接导致某种形式的执行期类型识别，也就是 RTTI）。最后，还需要某种机制用以管理被丢出的 object，包括它的产生、储存、可能的解构（如果有相关的 destructor）、清理（clean up）以及一般存取。也可能有一个以上的 objects 同时起作用。一般而言，exception handling 机制需要与编译器所产生的数据解构以及执行期的一个 exception library 紧密合作。在程序大小和执行速度之间，编译器必须有所抉择：

1. 为了维持执行速度，编译器可以在编译时期建立起用于支持的数据结构。这会使程序膨胀的大小，但编译器可以几乎忽略这些结构，直到有个 exception被丢出来。
2. 为了维护程序大小，编译器可以在执行期建立起用于支持的数据结构。这会影响程序的执行速度，但意味着编译器只有在必要的时候才建立那些数据结构（并且可以抛弃之）。

根据 [CHASE94] 所言，Modula-3 Report 竟然将一个原本为了维护执行速度而偏好的做法变成了一个制度，它赞成 “在 exceptional case 上花费 10000 个指令，以节省正常情况中的一个指令”。但这样的交易并非永远畅行无碍。最近在 Tel Aviv 的一场研讨会中，我与 Shay Bushinsky 交谈，他是 “Junior” 项目的开发者。“Junior” 是一个国际象棋程序，在 1994 年冬季的全球计算机国际象棋程序大赛中与 IBM 的深蓝（Deep Blue）得分相同，并列第三。令人惊讶的是，这个程序在一个 Pentium 个人计算机上运行（Deep Blue 则使用了 256 颗 CPU）。他告诉我当他们在并有 exception handling 的 Borland 编译器上重新编译 “Junior” 之后，程序虽然没有任何改变，内存却不够运用了。结果，他们只好回头找一套旧版的 Borland 编译器。对于 “Junior” 而言，他并不要一个提及庞大但是没有增加多少攻击能力的新版本。

过去还有一种错误的看法，认为由于 exception handling 的出现才导致 cfront 的灭绝，因为不可能提供一个既可接受而又强固的 exception handling 机制，且没有支持程序代码产生器（和链接器）。UNIX Software Laboratory（USL）当初搁置了由 HP 移交出来的 exception handing C-generating implementation 大约有一年之久（请看 [LENKOV92]）。USL 最后终于已知赞成取消掉 cfront 4.0（及任何更高版本）的开发计划。

### Exception Handling 快速检阅

C++ 的 exception handling 由三个主要的词汇组件构成：

1. 一个 throw 子句。它在程序某处发出一个 exception。被丢出去的 exception 可以是内建类型，也可以是使用者自定类型。
2. 一个或多个 catch 子句。每一个 catch 子句都是一个 exception handler。它用来表示说，这个字句准备处理某种类型的 exception，并且在封闭的大括号区段中提供实际的处理程序。
3. 一个 try 区段。它被围绕以一些列的叙述句（statements），这些叙述句可能会引发 catch 子句起作用。

当一个 exception 被丢出去时，控制权会从函数调用中被释放出来，并讯号一个吻合的 catch 字句。如果都没有吻合者，那么默认的处理例程 terminate() 会被调用。当控制权被放弃后，堆栈中的每一个函数调用也就被推离（popped up）。
这个程序称为 unwinding the stack。在每一个函数被推之前，函数的 local class objects 的 destructor 会被调用。

Exception handling 中比较不那么直观的就是它对于那些似乎没什么事做的函数所带来的影响。例如下面这个函数：
```cpp
(1) Point*
(2) mumble()
(3) {
(4)     Point *pt1, *pt2;
(5)     pt1 = foo();
(6)     if( !pt1 )
(7)         return 0;
(8)     Point p;
(9)     pt2 = foo();
(10)    if( !pt2 )
(11)        return pt1;
(12)
(13)    ...
(14) }
```
如果有一个 exception 在第一调用 foo()（L5）时被丢出，那么这个 mumble() 函数会被推出程序堆栈。由于调用 foo() 的操作并不在一个 try 区段之内，也就是不需要尝试和一个 catch 子句吻合。这里也没有任何 local class objects 需要解构。然而如果有一个 exception 在第二次调用foo()（L11）时被丢出，exception handling 机制就必须在 “从程序堆栈中 “unwinding” 这个函数” 之前，先调用 p 的 destructor。

在 exception handling 之下，L4-L8 和 L9-L16 被视为两块语意不同的区域，因为当 exception 被丢出来时，这两块区域有不同的执行期语意。而且，欲支持 exception handling，需要额外的一些 “登记” 操作与数据。编译器的做法有两种，一种是把两块区域以个别的 “将被摧毁之 local objects” 链表（已在编译时期设妥）联合起来，另一种做法是让两块区域共享同一个链表，该链表会在执行期扩大或缩小。

在程序员层面，exception handling 也改变了函数在资源管理上的语意。例如，下面的函数中含有对一块共享内存的 locking 和 unlocking 操作，虽然看起来和 exceptions 没有什么关联，但是 exception handling 之下并不保证能够正确运行：
```cpp
void 
mumble(void *arena)
{
    Point *p = new Point;
    smLock(arena);  // function call

    // 如果有一个exception在此发生，问题就来了
    // ...

    smUnLock(arena);    // function call
    delete p;
}
```
本例之中，exception handling 机制把整个函数视为单一区域，不需要操作 “将函数从程序堆栈中 ‘unwinding’” 的事情。然而从语意上来说，在函数被推离堆栈之前，我们需要 unlock 共享内存，并 delete p，让函数成为 “exception proof” 的最明确（但不是最有效率）的方法就是安插一个 default catch 子句，像这样：
```cpp
void mumble(void *arena)
{
    Point *p;
    p = new Point;
    try{
        smLock(arena);
        // ...
    }
    catch(...){
        smUnLock(arena);
        delete p;
        throw;
    }
    smUnLock(arena);
    delete p;
}
```

这个函数现在有了两个区域：
1. try block 以外的区域，在那里，exception handling 机制除了 “pop” 程序堆栈之外，没有其他事情要做。
2. try block 以内的区域（以及它所联合的 default catch 子句）。

请注意，new 运算符的调用并非在 try 区段内。这是我的错误吗？如果 new 运算符或是 Point constructor 在配置内存之后发生一个 exception，那么内存既不会被 unlocking，p 也不会被 delete（这两动作都在 catch 区段内）。这是正确的语意吗？

是的，它是。如果 new 运算符丢出一个 exception，那么就不需要配置 heap中的内存，Point constructor 也不需要被调用。所以也没有理由调用 delete 运算符。然而如果是在 Point constructor 中发生 exception，此时内存已经配置完成，那么 Point 之中任何构造好的合成物或子对象（subobject，也就是一个 member class object 或 base class object）都将自动被解构掉，然后 heap 内存也会被释放掉。不论哪种情况，都不需要调用 delete 运算符。

类似的道理，如果一个 exception 是在 new 运算符执行过程中被丢出，arena 所指向的内存就绝不会被 locked，因此，也没有必要 unlock 之。

处理这些资源管理的问题，我的一个建议方法就是，将资源需求封装于一个 class object 之内，并由 destructor 来释放资源（然而如果资源必须被索求、被释放、再被索求、再被释放······许多次的时候，这种风格会变得有点累赘）：
```cpp
void mumble(void *arena)
{
    auto_ptr<Point> ph(new Point);
    SMLock sm(arena);

    // 如果这里丢出一个exception，现在就没有问题了
    // ...

    // 不需要明确地unlock和delete
    // local destructor在这里被调用
    // sm.SMLock::~SMLock();
    // ph.auto_ptr<Point>::~auto_ptr<Point>()
}
```
从 exception handling 的角度来看，这个函数现在有三个区段：
1. 第一个区段是 auto_ptr 被定义之处。
2. 第二个区段是 SMLock 被定义之处。
3. 上述两个定义之后的整个函数。

如果 exception 是在 auto_ptr constructor 中被丢出的，那么就没有 active local objects 需要被 EH 机制摧毁。然而如果 SMLock constructor 中丢出一个 exception，则 auto_ptr object 必须在 “unwinding” 之前被摧毁。至于在第三个区段中，两个 local objects 当然都必须被摧毁。

支持 EH，会使那些拥有 member class subobjects 或 base class subobjects（并且它们也都有 constructor）的 classes 的 constructor 更复杂。一个 class 如果被部分构造，则其 destructor 必须只施行于那些已被构造的 subobjects 和（或）member objects 身上。例如，假设 class X 有 member objects A、B 和 C，都各有一对 constructor 和 destructor，如果 A 有 constructor 丢出一个 exception，不论 A 或 B 或 C 都不需要调用其 destructor。如果 B 的 constructor 丢出一个exception，则 A 的 destructor 必须被调用，但 C 不用。处理所有这些意外事故，是编译器的责任。

同样的道理，如果程序员写下：
```cpp
// class Point3d : public Point2d{ ... }
Point3d *cvs = new Point3d[512];
```
会发生两件事：
1. 从 heap 中配置足以给 512 个 Point3d objects 所用的内存。
2. 如果成功，先是 Point2d constructor，然后是 Point3d constructor，会施行于每一个元素身上。

如果 #27 元素的 Point3d constructor 丢出一个 exception，会怎样呢？对于 #27 元素，只有 Point2d destructor 需要调用执行。对于前 26 个元素，Point3d destructor 和 Point2d destructor 都需要起而执行，然后内存必须被释放回去。

### 对 Exception Handling 的支持

当一个 exception 发生时，编译系统必须完成以下事情：
1. 检验发生 throw 操作的函数
2. 决定 throw 操作是否发生在 thy 区段中；
3. 若是，编译系统必须把 exception type 拿来和每一个 catch 子句比较；
4. 如果比较吻合，流程控制应该交到 catch 子句手中；
5. 如果 throw 的发生并不在 try 区段中，或没有一个 catch 子句吻合，那么系统必须（a）摧毁所有的 active local objects，(b)从堆栈中将当前函数 “unwind” 掉，（c）进行到程序堆栈中的下一个函数中去，然后重复上述步骤 2-5.

### 决定 throw 是否发生在一个 try 区段中

还记得吗，一个函数可以被想象成是好几个区域：
1. try 区段以外的区域，而且没有 active local objects。
2. try 区段以外的区域，但是一个（以上）的 active local objects 需要解构。
2. try 区段以内的区域。

编译器必须标示出以上各区域，并使它们对执行期的 exception handling 系统有所作用。一个很棒的策略就是构造出 program counter-range 表格。

回忆一下，program counter（译注：在 Intel CPU 中为 EIP 缓存器）内含下一个即将执行的程序指令。好，为了在一个内含 try 区段的函数中标示出某个区域，可以把 program counter 的起始值和结束值（或是起始值和范围）储存在一个表格中。

当 throw 操作发生时，当前的 program counter 值被拿来于对应的 “范围表格” 进行比较，以决定当前作用中的区域是否在一个 try 区段中。如果是，就需要找出相关的 catch 子句（稍后我们再来看这一部分）。如果这个 exception 无法被处理（或者它被再次丢出），当前的这个函数会从程序堆栈中被推出 (popped)，而 program counter 会被设定为调用端地址，然后这样的循环再重新开始。

### 将 exception 的类型和每一个 catch 子句的类型做比较

对于每一个被丢出来的 exception，编译器必须产生一个类型描述器，对 exception 的类型进行编码。如果那是一个 derived type，则编码内容必须包括其所有的 base class 的类型信息。只编进 public base class 的类型是不够的的，因为这个 exception 可能被一个 member function 捕捉，而在一个 member function 的范围（scope）之中，在 derived class 和 nonpublic base class 之间可以转换。

类型描述器（type descriptor）是必要的，因为真正的 exception 是在执行期被处理，其 object 必须由自己的类型信息。RTTI 正是因为支持 EH 而获得的副产品。我将在 7.3 节讨论 RTTI。

编译器还必须为每一个 catch 子句产生一个类型描述器。执行期的 exception handler 会对 “被丢出之 object 的类型描述器” 和 “每一个 cause 子句的类型描述器” 进行比较，直到找到吻合的一个，或是直接堆栈已经被 “unwound” 而 terminate() 已被调用。

每一个函数会产生出一个 exception 表格，它描述与函数相关的
