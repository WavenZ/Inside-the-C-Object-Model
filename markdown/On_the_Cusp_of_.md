# 第7章 站在对象模型的尖端 On the Cusp of the Ojbect Model

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

每一个函数会产生出一个 exception 表格，它描述与函数相关的各区域、任何必要的善后码（cleanup code，被 local class object destructor 调用），以及 catch 子句的位置（如果某个区域是在 try 区段之中）。

### 当一个实际对象在程序执行时被丢出，会发生什么事

当一个 exception 被丢出时，exception object 会被产生出来并通常放置在相同形式的 exception 数据堆栈中。从 throw 端传染给 catch 子句是 exception object 的地址、类型描述器（或是一个函数指针，该函数会传回与该 exception type 有关的类型描述器对象），以及可能会有的 exception object 描述器（如果有人定义它的话
。

考虑一个 catch 子句如下：
```cpp
catch (exPoint p)
{
    // do something
    throw;
}
```
以及一个 exception object，类型为 exVertex，派生自 exPoint。这两种类型都吻合，于是 catch 子句会作用起来。那么 p 会发生什么事？

1. p 将以 exception object 作为初值，就像是一个函数参数一样，这意味着如果定义有（或由编译器合成出）一个 copy constructor 和一个 destructor 的话，它们都会实施于 local copy 身上。
2. 由于 p 是一个 object 而不是一个 reference，当其内容被拷贝的时候，这个 exception object 的 non-exPoint 部分会被切掉（sliced off）。此外，如果为了 exception 的继承而提供有 virtual functions，那么 p 的 vptr 会被设为 exPoint 的 virtual table；exception object 的 vptr 不会被拷贝。

当这个 exception 被再丢出一次时，会发生什么事情呢？p 现在是繁殖出来的 object？还是从 throw 端产生的原始 exception object？p 是一个 local object，在 catch 子句的末端将被摧毁。丢出 p 需得产生另一个临时对象，并意味着丧失原来的 exception 的 exVertex 部分。原来的 exception object 被再一次丢出；任何对 p 的修改都会被抛弃。

像下面这样的一个 catch 子句：
```cpp
catch (exPoint &rp)
{
    // do something
    throw;
}
```
则是参考到真正的 exception object。任何虚拟调用都会被决议（resolved）为 instances active for exVertex，也就是 exception object 的真正类型。任何对此 object 的改变都会被繁殖到下一个 catch 子句中。

最后，这里提出一个有趣的谜题。如果我们由下面的 throw 操作：
```cpp
exVertex errVer;

// ...
mumble()
{
    // ...
    if(mumble_cond){
        errVer.fileName("mumble()");
        throw errVer;
    }
}
```
究竟是真正的 exception errVer 被繁殖，抑或是 errVer 的一个复制品被构造于 exception stack 之中并被繁殖？答案是一个复制品被构造出来，全局性的 errVer 并没有被繁殖。这意味着在一个 catch 子句中对于 exception object 的任何改变都是局部性的，不会影响 errVer。只有在一个 catch 子句评估完毕并且知道它不会再丢出 exception 之后，真正的 exception object 才会被摧毁。

在某一次对 PC C++ 编译器的评论当中（请参考 [HORST95]），Cay Horstmann 测量了 EH 所带来的效益和大小的额外负担。Cay 编译器并执行一个测试程序，产生并摧毁大量的 local objects——它们有自己的 constructors 和 destructors。两个程序都不发生实际的 exceptions，它们之间的差异只在于其中一个 main() 内有个 catch(...) 子句。下面是它他测量了 Microsoft、Borland、Symantec 编译器后的结果。首先，程序大小有异：

表格 7.1 对象大小（两种情况：有或没有 exception handling）
||没有EH|有EH|百分比|
|-|-|-|-|
|Borland|86,822|89,510|3%|
|Microsoft|60,146|67,071|13%|
|Symantec|69,786|74,826|8%|

其次，执行速度也有异：

表格 7.2 执行速度（两种情况：有或没有 exception handling）
||没有EH|有EH|百分比|
|-|-|-|-|
|Borland|78秒|83秒|6%|
|Microsoft|83秒|87秒|5%|
|Symantec|94秒|96秒|4%|

与其他语言特性作比较，C++ 编译器支持 EH 机制所付出的代价最大。某种程度上是由于其执行期的天性以及对底层硬件的以来，以及 UNIX 和 PC 两种平台对于执行速度和程序大小有着不同的取舍优先状态之故。

## 7.3 执行期类型识别（Runtime Type Identification, RTTI）

在 cfront 中，用以表现出一个程序的所谓 “内部类型体系”，看起来像这样：
```cpp
// 程序层次结构的根类（root class）
class node{ ... };

// root of the 'type' subtree: basic types,
// 'derived' types : pointers, arrays,
// functions, classes, enums ...
class type : public node { ... };

// two representantions for functions
class fct : public type { ... };
class gen : public type { ... };
```
其中 gen 是 generic 的简写，用来表现一个 overloaded function。

于是只要你有一个变量，或是类型为 type* 的成员（并知道它代表一个函数），你就必须决定其特定的 derived type 受否为 fct 或是 gen。在 2.0 之前，除了 destructor 之外唯一不能够被 overloaded 的函数就是 conversion 运算符，例如：
```cpp
class String{
public:
    operator char*();
    // ...
};
```
在 2.0 导入 const member function 之前，conversion 运算符不能够被 overloaded，因为它们不使用参数。直到引进了 const member functions 后，情况才有所变化。现在，像下面这样的声明就有可能了：
```cpp
class String{
public:
    // ok with Release 2.0
    operator char*();
    operator char*() const;
    // ...
};
```
也就是说，在 2.0 版之前，以一个 explicit cast 来存取 derived object 总是安全（而且比较快速）的，像下面这样：
```cpp
typedef type *ptype;
typedef fct *pfct;

simplify_conv_op(ptype pt)
{
    // ok : conversion operators can only be fcts
    pfct pf = pfct(pt);
    // ...
}
```
在 const member functions 引入之前，这份码是正确的。请注意其中甚至有一个批注，说明这样的转型的安全性。但是在 cosnt member functions 引进之后，不论程序批注或程序代码都不对了。程序代码之所以失败，非常不幸是因为 String class 声明的改变。因为 char* conversion 运算符现在被内部视为一个 gen 而不是一个 fct。

下面这样的转型形式：
```cpp
pfct f = pfct(pt);
```
被称为 downcast (向下转型)，因为它有效地把一个 base class 转换至继承结构的末端，变成其 derived classes 中的某一个。Downcast 有潜在性的风险，因为它遏制了类型系统的作用，不正确的使用可能会带来错误的解释（如果它是一个 read 操作）或腐蚀掉程序内存（如果它是一个 write 操作）。在我们的例子中，一个指向 gen object 的指针被不正确地转型为一个指向 fct object 的指针 pf。所有后续对 pf 的使用都是不正确的（除非只是检查它是否为 0，或只是把它拿来和其它指针相比较）。

### Type-Safe Downcast（保证安全的向下转型操作）

C++ 被吹毛求疵的一点就是，它缺乏一个保证安全的 downcast（向下转型操作）。只有在 “类型真的可以被适当转型” 的情况下，你才能够执行 downcast（请看 [BUDD91]）。一个 type-safe downcast 必须在执行期对指针有所查询，看看它是否指向它所展现（表达）之 object 的真正类型。依此，欲支持 type-safe downcast，在 object 空间和执行时间上都需要一些额外负担:
1. 需要额外的空间以储存类型信息（type information），通常是一个指针，指向某个类型信息节点。
2. 需要额外的时间以决定执行期的类型（runtime type），因为，正如其名所示，这需要在执行期才能决定。

这样的机制面对下面这样平常的 C 结构，会如何影响其大小、效率、以及链接兼容性呢？
```cpp
char *winnie_tbl[] = { "rumbly in my tummy", "oh, bother" };
```
很明显，它所导致的空间和效率上的不良报应甚为可观。

冲突发生在两组使用者之间：

1. 程序员大量使用多态（polymorphism）,并因而需要正统而合法的大量downcast操作。
2. 程序员使用内建数据类型以及非多态设备，因而不受各种额外负担所带来的报应。

理想的解决方案是，为两派使用者提供正统而合法的需要——虽然或许得牺牲一些设计上的纯度与优雅性。你知道要怎么做吗？

C++ 的 RTTI 机制提供一个安全的 downcast 设备，但只对那些展现 “多态（也就是使用继承和动态绑定）” 的类型有效。我们如何分辨这些？编译器能否光看 class 的定义就决定这个 class 用以表现一个独立的 ADT 或是一个支持多态的可继承子类型（subtype）？当然，策略之一就是导入一个新的关键词，优点是可以清楚地识别出支持新特性的类型，缺点则是必须翻新旧程序。

另一个策略是经由声明一个或多个 virtual functions 来区别 class 声明。其优点是透明化地将旧有程序转换过来，只要重新编译就好。缺点则是可能会将一个其实并非必要的 virtual function 强迫导入继承体系的 bas class 身上。毫无疑问你还可以想出更多策略，不过，目前所说的这一个正式 RTTI 机制所支持的策略。在 C++ 中，一个具备多态性质的 class（所谓的polymorphism class），正是内涵着继承而来（或直接声明）的 virtual functions。

从编译器的角度来看，这个策略还有其他优点，就是大量降低额外负担。所有 polymorphism classes 的 objects 都维护了一个指针（vptr），指向virtual function table。只要我们把与该 class 相关的 RTTI object 地址放进 virtual table中（通常放在第一个 slot），那么额外负担就降低为：每一个 class object 只多花费一个指针。这个指针只需被设定一次，它是被编译器静态设定，而不是在执行期由 class constructor 设定（vptr 才是这么设定）。

### Type-Safe Dynamic Cast（保证安全的动态转型）

dynamic_cast 运算符可以在执行期决定真正的类型。如果 downcast 是安全的（也就是说，如果 base type pointer 指向一个 derived class object），这个运算符会传回被适当转型过的指针。如果 downcast 不是安全的，这个运算符会传回 0.下面就是我们如何重写我们版本的 cfront downcast（当然啦，现在的 pt 实际类型可能是 fct，也可能是gen。比较受欢迎的程序方法是使用 virtual function。在此法中，参数的真正类型被封装起来。程序比较清晰也比较容易扩充，得以处理更多类型）：
```cpp
typedef type *ptype;
typedef fct *pfct;

simplify_conv_op(ptype pt)
{
    if(pfct pf = dynamic_cast<pfct>(pt)){
        // ... process pf
    }
    else { ... }
}
```
什么是 dynamic_cast 的真正成本呢？pfct 的一个类型描述其会被编译器产生出来。由 pt 指向之 class object 类型描述器必须在执行期通过 vptr 取得。下面就是可能的转换：
```cpp
// 取得pt的类型描述器
((type_info*)(pt->vptr[0]))->_type_descriptor;
```
type_info 是 C++ Standard 所定义的类型描述器的 class 名称，该 class 中放置着待索求的类型信息。virtual table 的以一个 slot 内含 type_info object 的地址；此 type_info object 与 pt 所指之 class type 有关（请看 1.1 节的图 1.3）.
这两个类型描述器被交给一个 runtime library 函数，比较之后告诉我们是否吻合。很明显这比 static_cast 昂贵得多，但却安全得多（如果我们把一个 fct 类型 “downcast” 为一个 gen 类型的话）。

最初对 runtime cast 的支持提议中，并未引进任何关键词或额外的语法。下面这样的转型操作：
```cpp
// 最初对runtime cast的提议语法
pfct pf = pfct(pt);
```
究竟是 static 还是 dynamic，必须视 pt 是否指向一个多态 class object 而定。贝尔实验室中的我们这一伙认为这样子很棒，但是 C++ 标准委员会想的是另一套。他们的评论，就我所知，认为这么一来一个代价昂贵的 runtime 操作与一个简单的 static cast 太相像了。也就是说，当我们看到这个 cast 操作，无法知道 pt 是否指向多态对象（polymorphic object），也就无法知道这个 cast 操作是执行于编译时期或是执行期。这当然是事实，然而，virtual function call 不也同样吗？或许 C++ 标准委员也应该引进一种新语法和关键词，以便区分：
```cpp
pt->foobar();
```
另一个是静态决议的 function call，或是一个通过虚拟机制的调用操作！

### References 并不是 Pointers

程序执行中对一个 class 指针类型施以 dynamic_cast 运算符，会获得 true 或 false:
1. 如果传回真正的地址，表示这个 object 的动态类型被确认了，一些与类型有关的操作现在可以施行于其上。
2. 如果传回 0，表示没有指向任何 object，以为应该以另一种逻辑施行于这个动态类型未确定的 object 身上。

dynaimc_cast 运算符也适用于 reference 身上。然而对于一个 non-type-safe cast，其结果不会与施行于指针的情况相同。为什么？一个 reference 不可以像指针那样 “把自己设为 0 便代表了 “no object” ”；若将一个 reference 设为 0，会引起一个临时性对象（拥有被参考到的类型）被产生出来，该临时对象的初值为 0，这个 reference 然后被设定称为该临时对象的一个别名（alias）。因此当 dynamic_cast 运算符施行于一个 reference时，不能够提供对等于指针情况下的那一组 true/false。取而代之的是，会发生下列事情：
1. 如果 reference 真正参考到适当的 derived class（包括下一层或下下一层或下下下一层或...），downcast 会被执行而程序可以继续进行。
2. 如果 reference 并不真正是某一种 derived class，那么，由于不能够传回 0，遂丢出一个 bad_cast exception。

下面是重新实现后的 simplify_conv_op 函数，参数改为一个 reference:
```cpp
simplify_conv_op(const type &rt)
{
    try{
        fct &rf = dynamic_cast<fct&>(rt);
        // ...
    }
    catch(bad_cast){
        // ... mumble ...
    }
}
```
其中执行的操作十分理想地表现出某种 exception failure，而不只是简单（一如从前）的控制流程。

### Typeip 运算符

使用 typeid 运算符，就有可能以一个 reference 达到相同的执行期替代路线（runtime "altenative pathway”）：
```cpp
simplify_conv_op(const type &rt)
{
    if(typeid(rt) == typeid(fct))
    {
        fct &rf = static_cast<fct&>(rt);
        // ...
    }
    else { ... }
}
```
虽然我必须说，在这里，一个明显较佳的实现策略是在 gen 和 fct classes 中都引进一个 virtual function。

typeid 运算符传回一个 const refernce，类型为 type_info。先前测试中出现的 equality（等号）运算符，其实是一个被 overloaded 的函数：
```cpp
bool
type_info:
operator==(const type_info&) const;
```
如果两个 type_info objects 相等，这个 equality 运算符就传回 true。

type_info object 由什么组成？C++ Standard（Section 18.5.1）中对type_info 的定义如下（译注：你可以在 Visual C++ 的 typeinfo.h 中找到类似的定义）：
```cpp
class type_info{
public:
    virtual ~type_info();
    bool operator==(const type_info&) const;
    bool operator!=(const type_infO&) const;
    bool before(const type_info&) const;
    const char* name() const;   // 译注：传回class原始名称
private:
    // prevent memberwise init and copy
    type_info(const type_info&);
    type_info& operator=(const type_info&);;

    // data members
};
```
编译器必须提供最小量信息是 class 的真实名称、以及在 type_info  objects 之间的某些排序算法（这就是 before() 函数的目的）、以及某些形式的描述器，用来表现 explicit class type 和这个 class 的任何 subtypes。在描述 exception handling 的原始文章（ [KOENIG90b] ）中，曾建议实现出一种描述器：编码后的字符串（译注）。其他策略请见 [SUN94a] 和 [LENKOV92]。

译注：Microsoft 的 Visual C++ 就是采用编码后的字符串作为描述器。所以 Visual C++ 的 typeinfo.h 中对 type_info 的定义，就比上述的 C++ Standard 所定义的还多一个 member:
```cpp
class type_info{
public:
    const char* name() const;   // 传回class原始名称
    const char* raw_name() const;   // 传回cass名称的编码字符串
    ...
};
```
下面是以 Visual C++ 完成的一个小范例，用以检验编码后的 class 名称：
```cpp
// building : c1 -GR typeid.cpp
#include <iostream.h>
#include <typeinfo.h>

class B { ... };
class D : public B { ... };

void main()
{
    B *pb = new B;
    D *pd = new D;

    cout << "pb's type name = " << typeid(pb).name() << endl;
    cout << "pd's type name = " << typeid(pd).name() << endl;
    cout << "pb's type rawname = " << typeid(pb).raw_name() << endl;
    cout << "pd's type rawname = " << typeid(pd).raw_name() << endl;
}
```
执行结果为：
```
pb's type name = class B * (class B 的原始名称)
pd's type name = class D * (class D 的原始名称)
pb's type rawname = .PAVB@@ (class B 的源码名称)
pd's type rawneme = .PAVD@@ (class D 的编码名称)
```
虽然 RTTI 提供的 type_info 对于 exception handling 的支持来说是必要的，但对于 exception handling 的完整支持而言，还不够。如果再加上额外的一些 type_info derived classes，就可以在 exception 发生时提供有关于指针、函数及类等等的更详细信息。例如 MetaWare 就定义了以下的额外类：
```cpp
class Pointer_type_info: public type_info { ... };
class Member_pointer_info : public type_info { ... };
class Modified_type_info: public type_info { ... };
class Array_type_info : public type_info { ... };
class Func_type_info : public type_info { ... };
class Class_type_info : public type_info { ... };
```

并允许使用者取用它们。不幸的是，那些 derived class 的大小以及命名习惯都没有一个标准，各家编译器大有不同。

虽然我早说过 RTTI 只适用于多态类（polymorphic classes），事实上 type_info objects 也适用于内建类型，以及非多态的使用者自定类型。这对于 exception handling 的支持由必要。例如：
```cpp
int ex_errno;
...
throw ex_errno;
```
其中 int 类型也有他自己的 type_info object。下面就是使用法：
```cpp
int *ptr;
...
if(typeid(ptr) == typeid(int*))
    ...
```
在程序中使用 typeid(expression)，像这样：
```cpp
int ival;
...
typeid(ival) ...;
```
或是使用 typeid(type)，像这样：
```cpp
typeid(double) ...;
```
会传回一个 const type_info&。这与先前使用多态类型（polymorphic types）的差异在于，这时候的 type_info object 是静态取得，而非执行期取得。一般的实现策略是在需要时才产生 type_info object，而非程序一开头就产生之。

## 7.4 效率有了，弹性呢

传统的 C++ 对象模型提供有效率的执行期支持。这份效率，再加上与 C 之间的兼容性，造成了 C++ 的广泛被接受度。然而，在某些领域方面，像是动态共享函数库（dynamically shared libraries）、共享内存（shared memory）、以及分布式对象（distrubuted object）方面，这个对象模型的弹性还是不够。

### 动态函数共享库（Dynamic Shared Libraries）

理想中，一个动态链接的 shared library 应该像 “突然造访” 一样。也就是说，当应用程序下一次再执行时，会透明化地取用新的 library 版本。新的 library 问世不应该对旧的应用程序产生侵略性，应用程序不应该需要为此重新建造（building）依次。然而，在目前的 C++ 对象模型中，如果新版 library 中的 class object 布局有所变更，上述的 “library 无侵略性” 说法便有待商榷了。这是因为 class 的大小及其每一个直接（或继承而来）的 members 的偏移量（offset）都在编译时期就已经固定（虚拟继承的members 除外）。这虽然带来效率，却在二进制层面（binary level）影响了弹性。如果 object 布局改变，应用程序就必须重新编译。[GOLD94] 和 [PALAY92] 两篇文章描述了前人的许多有趣的努力，希望把 C++ 对象模型推至更具 “突然造访” 的能力。当然，这会丧失部分的执行期速度优势和大小优势。

### 共享内存（Shared Memory）

当一个 shared library 被加载，它在内存中的位置由 runtime linker 决定，一般而言与执行中的行程（process）无关。然而，在 C++ 对象模型中，当一个动态的 shared library 支持一个 class object，其中含有 virtual functions（被放在 shared memory 中），上述说法便不正确。问题并不在于 “将该 object 放置于 shared memory中” 的那个行程，而在于“想要经由这个 shared object 附着并调用一个 virtual function” 的第二个或更后记的行程。除非 dynamic shared library 被放置于其安全相同的内存位置上，就像当初加载这个 shared object 的行程一样，否则 virtual function 会死得很难看，可能的错误包括 segment fault 或 bus error。病灶出在每一个 virtual function 在 virtual table 中的位置已经被写死了。目前的解决方法时属于程序层面，程序员必须保证让跨越行程的 shared libraries 在相同的座落地址（在 SGI 中，使用者可以根据所谓的 so-location 档，指定每一个 shared library 的精确位置）。至于编译器系统层面上的解决方法，势必得牺牲原本得 virtual table 实现模型所带来得高效率。

Coommon Object Request Borker Architecture（CORBA）、Component Object Model（COM）、以及System Object Model（SOM）都企图定义出分布式、二进制层面得对象模型，并且与任何程序语言无关（请见 [MOWBRAY95] 对 CORBA 的详细讨论，及对 SOM 和 COM 的次要讨论。至于 C++ 导向的讨论，请见 [HAM95]）的 SOM、[BOX95] 的 COM、以及 [VINOS93] 和 [VINOS94] 的 CORBA）。这些努力有可能在未来将 C++ 对象模型推往更高的弹性（经由更多的间接性），但我们也知道那需要赔上更多的执行期速度与效率。

当我们的电算环境的需求更加进化（试想想 Web programming、Java applets）时，传统的 C++ 对象模型，带着它那 “高效率” 和 “与C兼容” 的特性，可能会不断累加束缚。然而，到了哪个时候，对象模型（Object Model）将证明 C++ 的全面适用性，不论在各式各样的操作系统、各式各样的硬件驱动程序、基因工程、以及我自己目前专注的 3D 计算机绘图和动画上。

译注：如果您对于 Component Object Model（COM）感兴趣，我推荐两本理想的书籍。一本是 Essential COM(Don Box/Addison Wesley/1998)。另一本是 Inside COM（Dale Rogerson/Microsoft Press/1996）。Essential COM 的 1、2 两章把软件组件（components）的本质、问题所在、以及 COM 的解决之道解释得非常好，带着读者以一般的、纯粹的 C++ 语言（不借助于任何工具）完成一个 COM 程序结构；第 3 章以后得内容略嫌艰涩，可辅以 Inside COM。Inside COM 全书清爽建议，但最好必须先读过 Essential COM的前两章，你才会由扎实身后的基础去接受它。

[第 6 章 执行期语意学（Runtime Semantics）](Runtime_Semantics.md)
