# 第4章 Function语意学（The Sematics of Function）

如果我有一个 Point3d 的指针和对象：
```cpp
Point3d obj;
Point3d *ptr = &obj;
```
当我这样做：
```cpp
obj.normalize();
ptr->normalize();
```
时，会发生什么事呢？其中的 Point3d::normalize() 定义如下：
```cpp
Point3d
Point3d::normalize() const
{
    register float mag = magnitude();
    Point3d normal;
    normal._x = _x / mag;
    normal._y = _y / mag;
    normal._z = _z / mag;
    return normal;
}
```
而其中的 Point3d::magnitude() 又定义如下：
```cpp
float
Point3d::magnitude() const
{
    return sqrt(_x * _x + _y * _y + _z * _z);
}
```
答案是：我不知道！C++ 支持三种类型的 member functions：static、nonstatic 和 virtual，每一种类型被调用的方式都不相同。其间差异正是下一节的主题。不过，我们虽然不能够确定 normalize() 和 magnitude() 两函数是否为 virtual 或 nonvirtual，但可以确定它一定不是 static，原因有而：（1）它直接存取 nonstati数据；（2）它被声明为 const。是的，static member functions 不可能做到这两点。

## 4.1 Member的各种调用方式

回顾历史，原始的 “C with classes” 之支持 nonstatic member functions（请看 [STROUP82]，其中有 C 语言的第一个公开说明）。Virtual 函数是在 20 世纪 80 年代中期被加进来的，并且很明显受到许多质疑（许多质疑至今在 C 族群中仍然存在）。在 [STROUP94] 文献中，Bjarne 写道：

有一种常见的观点，认为 virtual functions 只不过是一种跛脚的函数指针，没什么用 ······ 其意思主要就是，virtual functions 是一种没有效能的形式。

Static member functions 是最后别引入的一种函数类型。它们在 1987 年的 Usenix C++ 研讨会的厂商研习营（Implementor's Wordshop）中被正式提议加入 C++ 中，并由 cfront 2.0 实现出来。

### Nonstatic Member Functions（非静态成员函数）

C++ 的设计准则之一就是：nonstatic member function 至少必须和一般的 nonmember function 有相同的效率。也就是说，如果我们要在以下两个函数之间作选择：
```cpp
float magniude3d(const Point3d *_this) { ... }
float Point3d::magnitude3d() const { ... }
```
那么选择 member function 不应该带来什么额外负担。这是因为编译器内部已将 “member 函数实体” 转换为对等的 “nonmember 函数实体”。

举个例子，下面是 magnitude() 的一个 nonmember 定义：
```cpp
float magnitude3d(const Point3d *_this){
    return sqrt(_this->_x * _this->_x + 
                _this->_y * _this->_y + 
                _this->_z * _this->_z );
}
```
乍见之下似乎 nonmember function 比较没有效率，它间接经由参数取用坐标成员，而 member function 却是直接取用坐标成员。然而实际上 member function 被内化为 nonmember 的形式。下面就是转化步骤：

1. 改写函数的 signature（译注：意指函数原型）以安插一个额外的参数到 member function 中，用以提供一个存取管道，使 class object 得以调用该函数。该额外参数被称为 this 指针：
```cpp
// non-const nonstatic member之增长过程
Point3d
Point3d::magnitude(Point3d *const this)
```
如果 member function 是 const，则变成：
```cpp
// const nonstatic member之扩张过程
Point3d
Point3d::magnitude(const Point3d *const this)
```
2. 将每一个 “对 nonstatic data member 的存取操作” 改为经由 this 指针来存取：
```cpp
{ 
    return sqrt(
        this->_x * this->_x + 
        this->_y * this->_y + 
        this->_z * this->_z);
    )
}
```
3. 将 member function 重新写成一个外部函数。对函数名称进行 “mangling” 处理，使它在程序中称为独一无二的词汇：
```cpp
exter magnitude__7Point3dFv(
    register Point3d *const this);
```
现在这个函数已经被转换好了，而其每一个调用操作也都必须转换。于是：
```cpp
obj.magnitude();
```
变成了：
```cpp
magnitude__7Point3dFv(&obj);
```
而
```cpp
ptr->magnitude();
```
变成了：
```cpp
magnitude__7Point3dFv(ptr);
```
本章一开始所提及的 normalize() 函数会被转化为下面的形式，其中假设已经声明有一个 Point3d copy constructor，而 named returned value(NRV) 的优化也已施行：
```cpp
// 以下描述“named return value函数”的内部转化
// 使用C++伪码
void normalize__7Point3dFv(register const Point3d *const this,
                            Point3d &__result)
{
    register float mag = this->magnitude();

    // default constructor
    __result.Point3d::Point3d();

    __result._x = this->_x/mag;
    __result._y = this->_y/mag;
    __result._z = this->_z/mag;

    return;
}
```

一个比较有效率的做法是直接构架 “normal” 值，像这样：
```cpp
Point3d
Point3d::normalize() const
{
    register float mag = magnitude();
    return Point3d(_x/mag, _y/mag, _z/mag);
}
```
它会被转化为以下的码（我再一次假设 Point3d 的 copy constructor 已经声明好了，而 NRV 的优化也已实施）：
```cpp
// 以下描述内部转化
// 使用C++伪码
void 
normalize__7Point3dFv(register const Point3d *const this,
                        Point3d &__result)
{
    register float mag = this->magnitude();

    // __result用以取代返回值（return value）
    __result.Point3d::Point3d(
        this->_x/mag, this->_y/mag, this->_z/mag);

    return;
}
```
这可以节省 default constructor 初始化所引起的额外负担。

### 名称的特殊处理（Name Mangling）

一般而言，member 的名称前面会加上 class 名称，形成独一无二的命名。例如下面的声明：
```cpp
class Bar { public: int ival; ...};
```
其中的 ival 有可能变成这样：
```cpp
// member经过name-mangling之后的可能结果之一
ival__3Bar
```
为什么编译器要真么做？请考虑这样的派生操作（derivation）：
```cpp
class Foo : public Bar { public: int ival; ...};
```
记住，Foo 对象内部结合了 base class 和 derived class 两者：
```cpp
// C++ 伪码
// Foo 的内部描述
class Foo{
public:
    int ival__3Bar;
    int ival__3Foo;
    ...
}
```
不管你要处理哪一个 ival，通过 “name mangling”，都可以绝对清楚地指出来。由于 member functions 可以被重载化（overloaded），所以需要更广泛的 mangling 手法，以提供绝对独一无二的名称。如果把：
```cpp
class Point{
public:
    void x(float newX);
    float x();
    ...
}
```
转换为：
```cpp
class Point{
public:
    void x__5Point(float newX);
    float X__5Point();
    ...
}
```
会导致两个被重载化（overloaded）的函数实体拥有相同的名称。为了让它们独一无二，唯有再加上它们的参数链表（可以从函数原型中参考得到）。如果把参数类型也编码进去，就一定可以制造独一无二的结果，使我们的两个 x() 函数有良好的转换（单如果你声明 extern "C"，就会压抑 nonmember functions 的 "mangling" 效果）：
```cpp
class Point{
public:
    void x__5PointFf(float newX);
    float x__5PointFv();
    ...
}
```
以上所示的只是 cfront 采用的编码方法。我必须承认，目前的编译器并没有统一的编码方法——虽然不断有一些活动企图导引出这方面的工业标准。当前 C++ 编译器对 name mangling 的做法还没有统一，但我们知道它迟早会统一。

把参数和函数名称编码在一起，编译器于是在不同的被编译模块之间达成了一种有限形式的类型检验。举个例子，如果一个 print 函数被这样定义：
```cpp
void print(const Point3d&) { ... }
```

但意外地被这样声明和调用：
```cpp
// 喔欧：以为是const Point3d&
void print(const Point3d);
```
两个实体如果拥有独一无二的 name mangling，那么任何不正确的调用操作在链接时期就因无法决议（resolved）而失败。有时候我们可以乐观地称此为 “确保类型安全的链接行为” (type-safe linkage)。我说 “乐观地” 是因为它只可以捕捉函数的标记（译注：signature，亦即函数名称+参数数目+参数类型）错误；如果 “返回类型” 声明错误，就没办法检查出来！

当前的编译系统中，有一种所谓的 demangling 工具，用来拦截名称并将其转换回去。使用者可以仍然处于 “不知道内部名称” 的极大幸福之中。然而声明并不是长久以来一直如此轻松，在 cfront 1.1 版，由于该系统未经世故，故总是收藏两种名称（译注：未经 mangled 和经过 mangled 的两种名称）；编译错误消息用的是程序代码函数名称，然而链接器却不，它用的是经过 mangled 的内部名称。

我还记得那些极度苦闷、半带狂怒、红头发、有雀斑的工程师，在一个午后，摇摇晃晃地走进我的办公室，厉声诘问我到底 cfront 对他的程序代码做了些什么手脚。这种与使用者之间的互动关系对我而言很新鲜，所以我的第一个想法是回答他：“没有，当然什么都没有！唔，真的没有。无论如何，我不知道···你为什么不去问 Bjarne 呢？” 我的第二个想法则是平静地询问他问题出在哪里（这使我获得了一些声望，呵呵）。

“这里”，他几近咆哮地向我推来一叠编译结果，并以一种嫌恶的口吻告诉我，链接器说他有一个无法决议（unsolved）的函数：
```cpp
_cpp1_mate44rcmat44
```
或是一般公认很不具亲和性的东西，比如一个 4x4 矩阵类的加法运算符的 mangling 结果：
```cpp
mat44::operator+(const mat44&);
```
原来，这位老兄声明并调用该运算符，但是却忘了定义它！“欧”，他说 “嗯”，他又加了一声。然后他强烈建议我们以后不要把内部名称显示给使用者看。大体来说，我们采纳了他的建议。

### Virtual Member Functions（虚拟成员函数）

如果 normalize() 是一个 virtual member function，那么以下的调用：
```cpp
ptr->normalize();
```
将会别内部转化为：
```cpp
(*ptr->vptr[1])(ptr);
```
其中：
1. vptr 表示由编译器产生的指针，指向 virtual table。它被安插在每一个 “声明有（或继承自）一个或多个 virtual functions” 的 class object 中。事实上其名称也会被 “mangled”，因为在一个复杂的 class 派生体系中，可能存在有多个 vptrs。
2. [1] 是 virtual table slot 的索引值，关联到 normalize() 函数。
3. 第二个 ptr 表示 this 指针

类似的道理，如果 magnitude() 也是一个 virtual function，它在 normalize() 之中的调用操作将被转换如下：
```cpp
// register float mag = magnitude();
register float mag = (*this->vptr[2])(this);
```
此时，由于 Point3d::magnitude() 是在 Point3d::normalize() 中被调用，而后者已经由虚拟机制而决议（resolved）妥当，所以明确地调用 “Point3d 实体” 会比较有效率，并因此压制由于虚拟机制而产生的不必要的重复调用操作：
```cpp
// 明确的调用操作（explicitly invocation）会压制虚拟机制
register float mag = Point3d::magnitude();
```
如果 magnitude() 声明为 inline 函数会更有效率。使用 class scope operator 明确调用一个 virtual function，其决议（resolved）方式会和 nonstatic member function 一样：
```cpp
register float mag = magnitude__7Point3dFv(this);
```
对于以下调用:
```cpp
// Point3d obj;
obj.normalize();
```
如果编译器把它转换为：
```cpp
(*obj.vptr[1])(&obj);
```
虽然语意正确，却没有必要。请回忆那些并不支持多态（polymorphism）的对象（1.3 节）。所以上述经由 obj 调用的函数实体只可以是 Point3d::normalize() “经由一个class object 调用一个 virtual function”，这种操作应该总是被编译器像对待一般的nonstatic member function 一样地加以决议（resolved）：
```cpp
normalize__7Point3dFv(&obj);
```
这项优化工程的另一个利益是，virtual function 的一个 inline 函数实体可以被扩展（expanded）开来，因而提供极大的效率利益。

Virtual functions，特别是它们在继承机制下的行为，将在 4.2 节有比较详细的讨论。

### Static Member Functions（静态成员函数）

如果 Point3d::normalize() 是一个 static member function，以下两个调用操作：
```cpp
obj.normalize();
ptr->normalize();
```
将被转换为一般的 nonmember 函数调用，像这样：
```cpp
// obj.normalize();
normalize__7Point3dSFv();
// ptr->normalize();
normalize__7Point3dSFv();
```
在 C++ 引入 static member function 之前，我想你很少看到下面这种怪异写法[1]:
```cpp
((Point3d*)0)->object_count();
```
[1] Jonathan Shopiro，贝尔实验室的成员，是我所第一位使用这种方法的人。他也是为 C++ 引入 static member functions 的主要倡导者。Static member functions 第一次正式提出，是在 1988 年 Usenix C++ 研讨会的厂商研习会议上由我所作的一个演讲中。当时我的主题是 pointer-to-member functions。我，并不令人讶异地，没有能够让 Tom Cargill 承认多重继承并不太困难；我同时也多少提到了一点 Jonathan 有关于 static member functions 的想法。谢天谢地，他跳上了讲台，对我们大谈其理想。不过在 [STROUP94] 文献中，Bjarne 说他第一次听到 static member functions 提案是来自于 Martion O'Riordan。

其中的 object_count() 只是简单传回 __object_count 这个 static data member。这种习惯是如何演化来的呢？

在引入 static member functions 之前，C++ 语言要求所有的 member functions 都必须经由该 class 的 object 来调用。而实际上，只有当一个或多个 nonstatic data members 在 member function 中被直接存取时，才需要 class object。Class object 提供了 this 指针给这种形式的函数调用使用。这个 this 指针把 “在 member function 中存取的 nonstatic class members” 绑定于 “object 内对应的 members” 之上。如果没有任何一个 members 被直接存取，事实上就不需要 this 指针，因此也就没有必要通过一个 class object 来调用一个 member function。不过 C++ 语言到当前为止并不能够识别这种情况。

这么一来就在存取 static data members 时产生了一些不规则性。如果 class 的设计者把 static data member 声明为 nonpublic（这已知被视为是一种好的习惯），那么他就必须提供一个或多个 member functions 来存取该 member。因此，虽然你可以不靠 class object 来存取一个 static members，但其存取函数却得绑定于一个 class object 之上。


独立于 class object 之外的存取操作，在某个时候特别重要：当 class 设计者希望支持 “没有 class object 存在” 的情况（就像前述的 object_count() 那样）时。程序方法上的解决之道是很奇特地把 0 强制转换为一个 class 指针，因而提供出一个 this 指针实体：
```cpp
// 函数调用的内部转换
object_count((Point3d*) 0);
```
至于语言层面上的解决之道，是由 cfront2.0 所引入的 static member functions。 Static member functions 的主要特性就是它没有 this 指针。以下的次要特性统统根源于其主要特性：
1. 它不能够直接存取其 class 中的 nonstatic members。
2. 它不能够被声明为 const、volatile 或 virtual。
3. 它不需要经由 class object 才被调用——虽然大部分时候它是这样被调用的！

“member selection” 语法的使用是一种符号上的便利，它会被转化为一个直接调用操作：
```cpp
if(Point3d::object_count() > 1) ...
```
如果 class object 是因为某个表达式而获得的，会如何呢？例如：
```cpp
if(foo().object_count() > 1) ...
```
奥，这个表达式仍然需要被评估求值（evaluated）:
```cpp
// 转化，以保存副作用
(void)foo();
if (Point3d::object_count() > 1) ...
```
一个 static member function，当然会被提出于 class 声明之外，并给与一个经过 “mangled” 的适当名称。例如：
```cpp
unsigned int
Point3d::
object_count()
{
    return _object_count;
}
```
会被 cfront 转化为：
```cpp
// 在cfront之下的内部转化结果
unsigned int
object_count_5Point3dSFv()
{
    return _object_count_5Point3d;
}
```
其中 SFv 表示它是个 static member function，拥有一个空白（void）的参数链表（argument list）。

如果取一个 static member function 的地址，获得的将是其在内存中的位置，也就是其地址。由于 static member function 没有 this 指针，所以其地址的类型并不是一个 “指向 class member function 的指针”，而是一个 “nonmember 函数指针”。也就是说：
```cpp
&Point3d::object_count();
```
会得到一个数值，类型是：
```cpp
unsigned int(*)();
```
而不是：
```cpp
unsigned int(Point3d::*)();
```
Static member function 由于缺乏 this 指针，因此差不多等同于 nonmember function。它提供了一个意想不到的好处：成为一个 callback 函数，使我们得以将 C++ 和 C-based X Window 系统结合（请看 [YOUNG95] 中的讨论）。它们也可以成功地应用在线程（threads）函数身上（请看 [SCHMIDT94a]）。

## 4.2 Virtual Member Functions（虚拟成员函数）

我们已经看过了 virtual function 的一般实现模型：每一个 class 有一个 virtual table，内含该 class 之中有作用的 virtual function 的地址，然后每个 object 有一个 vptr，指向 virtual table 的所在。在这一节中，我要走访一组可能的设计，然后根据单一继承、多重继承和虚拟继承等各种情况，从细部上探究这个模型。

为了支持 virtual function 机制，必须首先能够对于多态对象有某种形式的 “执行期类型判断法（runtime type resolution）”。也就是说，以下的调用操作将需要 ptr 在执行期的某些相关信息，
```cpp
ptr->z();
```
如此一来才能够找到并调用 z() 的适当实体。

或许最直接了当但是成本最高的解决方法就是把必要的信息加在 ptr 身上。在这样的策略之下，一个指针（或是一个reference）含有两项信息；
1. 它所参考到的对象的地址（也就是当前它所含有的东西）。
2. 对象类型的某种编码，或是某个结构（内含某些信息，用以正确决议出 z() 函数实例）的地址。

这个方法带来两个问题，第一，它明显增加了空间负担，即使程序并不适用多态（polymorphism）；第二，它打断了与 C 程序间的链接兼容性。

如果这份额外信息不能够和指针放在一起，下一个可以考虑的地方就是把它放在对象本身。但是哪一个对象真正需要这些信息呢？我们应该把这些信息放进可能被继承的每一个聚合体身上吗？或许吧!但请考虑一下这样的 C struct 声明：
```cpp
struct date{ int m, d, y; };
```
严格地说，这符合上述规范。然而事实上它并不需要那些信息。加上那些信息将使 C struct 膨胀并且打破链接兼容性，却没有带来任何明显的补偿利益。

“好吧，” 你说，“只有面对那些明确使用了 class 关键词的声明，才应该加上额外的执行期信息。” 这么做就可以保留语言的兼容性了，不过仍然不是一个够聪明的政策。举个例子，下面的 class 符合新规范：
```cpp
class date { public: int m, d, y; };
```
但实际上它并不需要那份信息。下面的 class 声明虽然不符合新规范，却需要那份信息：
```cpp
struct geom{ public: virtual ~geom(); ... };
```
奥，是的，我们需要一个更好的规范，一个 “以 class 的使用为基础，而不在乎关键词是class 或 struct（1.2 节）” 的规范。如果 class 真正需要那份信息，它就会存在；如果不需要，它就不存在。那么，到底何时才需要这份信息？很明显是在必须支持某种形式之“执行期多态（runtime polymorphism）”的时候。

在 C++ 中，多态（polymorphism）表示 “以一个 public base class 的指针（或reference），寻址出一个 derived class object” 的意思。例如下面的声明：
```cpp
Point *ptr;
```
我们可以指定 ptr 以寻址出一个 Point2d 对象：
```cpp
ptr = new Point2d;  // 原书写为Point2d pt2d = new Point2d; 我想是笔误
```
或是一个 Point3d 对象：
```cpp
ptr = new Point3d;
```
ptr 的多态机能主要扮演一个传送机制（transport mechanism）的角色，经由它，我们可以在程序的任何地方采用一组 public derived 类型。这种多态机制被称为是消极的（passive），可以在编译时期完成——virtual base class 的情况除外。

当被指出的对象真正被使用时，多态也就变成积极的（active）了。下面对于 virtual function 的调用，就是一例：
```cpp
// “积极多态（active polymorphism）”的常见例子
ptr->z();
```
在 runtime type identification(RTTI) 性质于 1993 年被引入 C++ 语言之前，C++ 对 “积极多态（active polymorphism）” 的唯一支持，就是对于 virtual function call 的决议（resolution）操作。有了 RTTI，就能够在执行期查询一个多态的 pointer或多态的 reference 了（RTTI 将在本书 7.3 节讨论）。

```cpp
// “积极多态（active polymorphism）”的第二个例子
if(Point3d *p3d = 
    dynamic_cast<Point3d*>(ptr))
    return p3d->_z;
```
所以，问题已经被区分出来，那就是：欲鉴定哪些 classes 展现多态特性，我们需要额外的执行期信息。一如我所说，关键词 class 和 struct 并不能够帮助我们。由于没有导入如 polymorphic 之类的新关键词，因此识别一个 class 是否支持多态，唯一适当的方法就是看看它是否有任何 virtual function。只要 class 拥有一个 virtual function，它就需要这份额外的执行期信息。

下一个明显的问题是，什么样的额外信息是我们需要存储起来的？也就是说，如果我有这样的调用：
```cpp
ptr->z();
```
其中 z() 是一个 virtual function，那么什么信息才能让我们在执行期调用正确的 z() 实体？我需要知道：
1. ptr 所指对象的真实类型，这可使我们选择正确的 z() 实体；
2. z() 实体位置，以便我能够调用它。

在实现上，首先我可以在每一个多态的 class object 身上增加哪个 members:
1. 一个字符串或数字，表示 class 的类型；
2. 一个指针，指向某表格，表格中带有程序的 virtual functions 的执行期地址。

表格中的 virtual functions 地址如何被构建起来？在 C++ 中，virtual functions（可经由其 class object 被调用）可以再编译时期获知，此外，这一组地址是固定不变的，执行期不可能新增或替换之。由于程序执行时，表格的大小和内容都不会改变，所以其建构和存取皆可以又编译器完全掌握，不需要执行期的任何介入。

然而，执行期备妥那些函数地址，只是解答的一半而已。另一半解答是找到那些地址。以下两个步骤可以完成这项任务：
1. 为了找到表格，每一个 class object 被安插上一个由编译器内部产生的指针，指向该表格。
2. 为了找到函数地址，每一个 virtual function 被指派一个表格索引值。

这些工作都由编译器完成。执行期要做的，只是在特定的 virtual table slot（记录着virtual function 的地址）中激活 virtual function.

一个 class 只会有一个 virtual table。每一个 table 内含其对应的 class object 中所有 active virtual functions 函数实体的地址。这些 active virtual functions 包括：
1. 这个 class 所定义的函数实体。它会改写（overriding）一个可能存在的 base class virtual function 函数实体。
2. 继承自 base class 的函数实体。这是在 derived class 决定不改写 virtual function 时才会出现的情况。
3. 一个 pure_virtual_called() 函数实体，它既可以扮演 pure virtual function 的空间保卫者角色，也可以当做执行期异常处理函数（有时候会用到）。

每一个 virtual function 都被指派一个固定的索引值，这个索引在整个继承体系中保持与特定的 virtual function 的关联。例如在我们的 Point class 体系中：
```cpp
class Point{
public:
    virtual ~Point();
    
    virtual Point& mul(floa t) = 0;
    // ...其它操作

    float x() const { return _x; }
    virtual float y() const { return 0;}
    virtual float z() const { return 0;}
    // ...

protected:
    Point(float x = 0.0);
    float _x;
};
```
virtual destructor 被复制为 slot1，而 mult() 被赋值 slot2. 此例并没有 mult() 的函数定义（译注：因为它是一个 pure virtual function）。所以 pure_virtual_called() 的函数地址会被放在 slot2 中。如果该函数意外地被调用，通常的操作时结束掉这个程序。y() 被赋值 slot3 而 z() 被赋值 slot4。 x() 的 slot 是多少？答案是没有，因为 x() 并非 virtual function。图 4.1 表示 Point 的内存布局和其 virtual table。

<img src = '4.1.png' width = '100%'>

图4.1 Virtual Table的布局：单一继承情况

译注：原书图中对于 Virtual table slot 所指向之对象都加上“”，我恐怕读者误以为它是指向一个字符串（事实上它是指向一个函数实体），所以在本图中全部去掉去“”符号。

```cpp
Point *pp;
...
pp->mult(pp); // => (*pp->__vptr_Point[2])(pp)
```
当一个 class 派生自 Point 时，会发生什么事？例如 class Point2d:
```cpp
class Point2d : public Point{
public:
    Point2d(float x = 0.0, float y = 0.0)
        : Point(x), _y(y) { }
    ~Point2d();
    // 改写base class virtual functions
    Point2d& mult(float);
    float y() const { return _y; }
    // ······其它操作
protected:
    float _y;
};
```
一共有三种可能性：
1. 它可以继承 base class 所声明的 virtual functions 的函数实体。正确地说，是该函数实体的地址会被拷贝到 derived class 的 virtual table 相对应的 slot 之中。
2. 它可以使用自己的函数实体。这表示它自己的函数实体地址必须放在对应的 slot 之中。
3. 它可以加入一个新的 virtual function。这时候 virtual table 的尺寸会增大一个 slot，而新的函数实体地址会被放进该 slot 之中。

Point2d 的 virtual table 在 slot1 中指出 destructor，而在 slot2 中指出mult()（取代 pure virtual function）。它自己的 y() 函数实体地址放在 slot3，继承自 Point 的 z() 函数实体地址则放在 slot4.

类似的情况，Point3d 派生自 Point2d，如下：
```cpp
class Point3d : public Point2d{
public:
    Point3d(float x = 0.0, float y = 0.0, float z = 0.0)
        : Point2d(x, y), _z(z) { }
    ~Point3d();

    // 改写base class virtual functions
    Point3d& mult(float);
    float z() const { return _z; }
    // ······ 其它操作

protected:
    float _z;
};
```
其 virtual table 中的 slot 放置 Point3d 的 destructor，slot2 放置 Point3d::mult(函数地址。slot3 放置继承自 Point2d 的 y() 函数地址，slot4 放置自己的 z() 函数地址。图 4.1 显示 Point3d 的对象布局和其 virtual table。

现在，如果我有这样的式子：
```cpp
ptr->z();
```
那么，我如何有足够的只是在编译时期设定 virtual function 的调用呢?
1. 一般而言，我并不知道 ptr 所指对象的真正类型。然而我知道，经由 ptr 可以存取到该对象的 virtual table。
2. 虽然我不知道哪一个 z() 函数实体会被调用，但我知道每一个 z() 函数的地址都被放在 slot4。

这些信息使得编译器可以将该调用转化为：
```cpp
(*ptr->vptr[4])(ptr);
```
在这个转化中，vptr 表示编译器所安插的指针，指向 virtable; 4 表示 z() 被赋值的 slot 编号（关联到 Point 体系的 virtual table）。唯一一个在执行期才能知道的东西是：slot4 所指的到底是哪一个 z() 函数实体？

在一个单一继承体系中，virtual function 体制的行为十分良好，不但有效率而且很容易塑造出模型来。但是在多重继承和虚拟继承之中，对 virtual functions 的支持就没有那么美好了。

## 多重继承下的Virtual Functions

在多重继承中支持 virtual functions，其复杂度围绕在第二个及后继的 base classes 身上，以及 “必须在执行期调整 this 指针” 这一点。以下面的 class 体系为例：
```cpp
// class体系，用来描述多重继承（MI）情况下支持virtual function是的复杂度
class Base1{
public:
    Base1();
    virtual ~Base1();
    virtual void speakClearly();
    virtual Base1 *clone() const;
protected:
    float data_Base1;
};
class Base2{
public:
    Base2();
    virtual ~Base2();
    virtual void mumble();
    virtual Base2 *clone() const;
protected:
    float data_Base2;
};
class Derived : public Base1, public Base2{
public:
    Derived();
    virtual ~Derived();
    virtual Derived *clone() const;
protected:
    float data_Derived;
}
```

译注：上述例子无法通过 VC5 编译，会出现以下错误信息：

> error C2555: 'Derived::clone' : overriding virtual function differs from 'Base1::clone' only by return type or calling convention
error C2555: 'Derived::clone' : overriding virtual function differs from 'Base2::clone' ongy by return type or calling convention

但这是因为 VC5 未符合 C++ 标准之故。原本 C++ 规定，改写函数（overriding function）的类型，包括函数名称、参数列、返回值类型，都必须和被改写函数（overrided function）相同。本例由于 Derived 派生自 Base1 和 Base2，所以面对 “Base1::clone() 传回 Base1*” 而 “Base2::clone() 传回 Base2*” 这两种情况，Derived::clone() 无所适从。然而时至今日，C++ 标准已针对此项做了修改，为了是容许所谓的虚拟构造函数（virtual constructor）。参见 p.166。

“Derived 支持 virtual functions” 的困难度，统统落在 Base2 subobject 身上。有三个问题需要解决，以此例而言分别是（1）virtual destructor，（2）被继承下来的 Base2::mumble()，（3）一组 clone() 函数实体。让我依次解决每一个问题。

首先，我把一个从 heap 中配置而得的 Derived 对象的地址，指定给一个 Base2 指针：
```cpp
Base2 *pbase2 = new Derived;
```
新的 Derived 对象的地址必须调整，以指向其 Base2 subobject。编译时期会产生以下的码：
```cpp
// 转移以支持第二个base class
Derived *temp = new Derived;
Base2 *pbase2 = temp ? temp + sizeof(Base1) : 0;
```
如果没有这样的调整，指针的任何 “非多态运用”（像下面那样）都将失败：
```cpp
// 即使pbase2被指定一个Derived对象，这也应该没有问题
pbase2->data_Base2;
```
当程序员要删除 pbase2 所指的对象时：
```cpp
// 必须首先调用正确的virtual destructor函数实体
// 然后施行delete运算符
// pbase2可能需要调整，以指出完整对象的起始点
delete pbase2;
```
指针必须被再一次调整，以求再一次指向 Derived 对象的起始处（推测它还指向 Derived 对象）。然而上述的 offset 加法却不能够在编译时期直接设定，因为 pbase2 所指的真正对象只有在执行期才能确定。

一般规则是，经由指向 “第二或后继之 base class” 的指针（或 reference）来调用 derived class virtual function。译注：就像本例的。
```cpp
Base2 *pbase2 = new Derived;
...
delete pbase2;  // invoke derived class's destructor (virtual)
```
该调用操作所连带的 “必要的 this 指针调整” 操作，必须在执行期完成。也就是说，offset 的大小，以及把 offset 加到 this 指针上头的那一小段程序代码，必须由编译器在某个地方插入。问题是，在哪个地方。

Bjarne 原先实施于 cfront 编译器中的方式是将 virtual table 加大，使它容纳此处所需的 this 指针，调整相关事务。每一个 virtual table slot，不再只是一个指针，而是一个聚合体，内含可能的 offset 以及地址。于是 virtual function 的调用操作由：
```cpp
(*pbase2->vptr[1])(pbase2);
```
改变为：
```cpp
(*pbase2->vptr[1].faddr)
```
其中 faddr 内含 virtual function 地址，offset 内含 this 指针调整值。

这个做法的缺点是，它相当于连带处罚了所有的 virtual function 调用操作。不管它们是否需要 offset 的调整。我所谓的处罚，包括 offset 的额外存取及其加法，以及每一个 virtual table slot 的大小改变。

比较有效率的解决方法是利用所谓的 thunk。当我第一次学到这个字眼的时候，我的教授开玩笑地告诉我，thunk 是 knuth的倒拼字，所以他吧这项技术归功于 knuth 博士☺。

译注：Donald E.Knuth，写出经典名著 *The Art of computer Programming* 的那个人。此套书籍被誉为 “the bible of fundamental algorithms”，据说拥有的人很多，看过的人很少☺。目前（1998）出版有:
> Volume 1 : Fundamental Algorithms 3/e
Volume 2 : Seminumerical Algorithms 3/e
Volume 3 : Sorting and Searching 2/e

Thunk 技术初次引进到编译器技术中，我相信是为了支持 ALGOL 独一无二的 pass-by-name 语意。所谓 thunk 是一小段 assembly 码，用来（1）以适当的 offset 值调整 this 指针，（2）跳到 virtual function 去。例如，经由一个 Base2 指针调用 Derived destructor，其相关的 thunk 可能看起来是这个样子：
```cpp
// 虚拟C++码
pbase2_dtor_thunk:
    this += sizeof(base1);
    Derived::~Deribed(this);
```
Bjarne 并不是不知道 thunk 技术，问题是 thunk 只有以 assembly 码完成才有效率可言。由于 cfront 使用 C 作为其程序代码产生语言，所以无法提供一个有效率的 thunk 编译器。

Thunk 技术允许 virtual table slot 继续内含一个简单的指针，因此多重继承不需要任何空间上的额外负担。Slots 中的地址可以直接指向 virtual function，也可以指向一个相关的 thunk（如果需要调整 this 指针的话）。于是，对于那些不需要调整 this 指针的 virtual function（相信大部分是如此，虽然我手上没有数据）而言，也就不需承载效率上的额外负担。

调整 this 指针的第二个额外负担就是，由于两种不同的可能：（1）经由 derived class（或第一个base class）调用，（2）经由第二个（或其后继）base class 调用，同一个函数在 virtual table 中可能需要多笔对应的 slots。例如：
```cpp
Base1 *pbase1 = new Derived;
Base2 *pbase2 = new Derived;

delete pbase1;
delete pbase2;
```
虽然两个 delete 操作导致相同的 Derived destructor，但它们需要两个不同的virtual table slots:
1. pbase1不需要调整 this 指针（因为 Base1 是最左端 base class 之故，它已经指向 Derived 对象的起始处）。其 virtual table slot 需放置真正的 destructor 地址。
2. pbase2 需要调整 this 指针。其 virtual table slot 需要相关的 thunk 地址。

在多重继承之下，一个 derived class 内含 n - 1 个额外的 virtual tables，n 表示其上一层 base classes 的数目（因此，单一继承将不会有额外的virtual tables）。对于本例之 Derived 而言，会有两个 virtual tables 被编译器产生出来：
1. 一个主要实体，与 Base1（最左端 base class）共享。
2. 一个次要实体，与 Base2（第二个 base class）有关。

针对每一个 virtual tables，Derived对象中有对应的 vptr。图 4.2 说明了这一点。vptrs 将在 constructor(s) 中被设立初值（经由编译器所产生出来的码）。

用以支持 “一个 class 拥有多个 virtual tables” 的传统方法是，将每一个 tables 以外部对象的形式产生出来，并给与独一无二的名称。例如，Derived 所关联的两个 tables 可能有这样的名称：
```cpp
vtbl_Derived;  // 主要表格
vtbl_Base2_Derived; // 次要表格
```
于是你将一个 Derived 对象地址指定给一个 Base1 指针或 Derived 指针时，将处理的 virtual table 是主要表格 vtbl_Derived。而当你将一个 Derived 对象地址指定给一个 Base2 指针时，被处理的 virtual table 是次要表格 vtbl_Base2_Derived。

<img src = '4.2.png' width = '100%'>

图4.2 Virtual Table的布局：多重继承情况。
（译注：右下角的三个星号，就是下页的三种个情况）

由于执行期链接器（runtime linkers）的降临（可以支持动态共享函数库），符号名称的链接可能变得非常缓慢，最慢可到······晤···在一部 SparcStation 10 工作站上，每一个毫秒（ms）只处理一个符号名称。为了调节执行期链接器的效率，Sun 编译器将多个 virtual tables 连锁为一个；指向次要表格的指针，可由主要表格名称加上一个 offset 获得。在这样的策略下，每一个 class 只有一个具名的 virtual table。“对于许多 Sun 项目程序代码而言，速度的提升十分明显” [2]。

[2] 语出自 Mike Bail (Sun C++ 编译器的建构者)与我的通信。

稍早我曾写道，有三种情况，第二或后继的 base class 会影响对 virtual functions 的支持。第一种情况是，通过一个 “指向第二个 base class” 的指针，调用 derived class virtual function。例如：
```cpp
Base2 *ptr = new Derived;

// 调用 Derived::~Derived
// ptr必须被向后调整sizeof(Base1)个bytes
delete ptr;
```
从图 4.2 之中，你可以看到这个调用操作的重点：ptr 指向 Derived 对象中的 Base2 subobject；为了能够正确执行，ptr 必须调整指向 Derived 对象的起始处。

第二个情况是第一种情况的变化，通过一个 “指向 derived class” 的指针，调用第二个 base class 中一个继承而来的 virtual function。在此情况下，derived class 指针必须再次调整，以指向第二个 base subobject。例如：
```cpp
Derived *pder = new Derived;

// 调用Base2::mumble()
// pder必须被向前调整sizeof(Base1)个bytes
pder->mumble();
```
第三种情况发生于一个语言扩充性质之下：允许一个 virtual function 的返回值类型有所变化，可能是 base type，也可能是 publicly derived type。这一点可以通过Derived::clone() 函数实体来说明。clone 函数的 Derived 版本传回一个 Derived class 指针，默默地改写了它的两个 base class 函数实体。当我们通过 “指向第二个 base class” 的指针来调用 clone() 时，this 指针的 offset 问题于是诞生：
```cpp
Base2 *pb1 = new Derived;

// 调用Derived* Derived::clone()
// 返回值必须被调整，以指向Base2 subobject
Base2 *pb2 = pb1->clone();
```
当进行 pb1->clone() 时，pb1 会被调整指向 Derived 对象的起始地址，于是 clone() 的 Derived 版会被调用；它会传回一个指针，指向一个新的 Derived 对象；该对象的地址在被指定给 pb2 之前，必须先经过调整，以指向 Base2 subobject。

当函数被认为 “足够小” 的时候，Sun 编译器会提供一个所谓的 “split functions” 技术：以相同算法产生出两个函数，其中第二个在返回之前，为指针加上必要的 offset。于是不论通过 Base1 指针或 Derived 指针调用函数，都不需要调整返回值；而通过 Base2指针所调用的，是另一个函数。

如果函数并不小，“split function” 策略会给予此函数中的多个进入点（entry points）中的一个。每一个进入点需要三个指令，但 Mike Ball 想办法去除了这项成本。对于 OO 没有经验的程序员，可能会怀疑这种 “split function” 的实用性，然而 OO 程序员都会尽量使用小规模的 virtual function 将操作 “局部化”。通常，virtual function 的平均大小是 8 行 [3]。

[3] 一个 virtual function 的平均大小是 8 行，这是我在某时某地听来的说法（不过我忘了是何时何地）。这一说法符合我个人的经验。

函数如果支持多重进入点，就可以不必有许多 “thunks”。如 IBM 就是把 thunk 搂抱在真正被调用的 virtual function 中。函数一开始先（1）调整 this 指针，然后才（2）执行程序员所写的函数码；至于不需调整的函数调用操作，就直接进入（2）的部分。

Microsoft 以所谓的 “address points” 来取代 thunk 策略。即将用来改写别人的那个函数（也就是 overriding function）期待获得的是 “引入该 virtual function 之class”（而非 derived class）的地址。这就是该函数的 “address point”，[MICRO92]对此有完整的讨论。

### 虚拟继承下的 Vritual Functions

考虑下面的 virtual base class 派生体系，从 Point2d 派生出 Point3d:
```cpp
class Point2d{
public:
    Point2d(float = 0.0, float = 0.0);
    virtual ~Point2d();

    virtual void mumble();
    virtual float z();
    // ...
protected:
    float _x, _y;
};

class Point3d : public virtual Point2d{
public:
    Point3d(float = 0.0, float = 0.0, float = 0.0);
    ~Point2d();

    float z();
protected:
    float _z;
};
```
虽然 Point3d 有唯一一个（同时也是最左边的）base class，也就是 Point3d，但 Point3d 和 Point3d 的起始部分并不像 “非虚拟的单一继承” 情况那样一致。这种情况显示于图 4.3。由于 Point2d 和 Point3d 的对象不再相符，两者之间的转换也就需要调整 this 指针。至于在虚拟继承的情况下要消除 thunks，一般而言已经被证明是一项高难度技术。

<img src = '4.3.png'>
图 4.3 Virtual Table 布局：虚拟继承情况。（译注：我个人对于图右下的两个 vtbls 内容感到相当疑惑。至少，mumble() 应该是 Point2d::mumble() 而非 Point3d::mumble()）

当一个 virtual base class 从另一个 virtual base class 派生而来，并且两者都支持 virtual functions 和 nonstatic data members 时，编译器对于 virtual base class 的支持简直就像进了迷宫一样。虽然我手上有一整柜带有答案的例程，并且有一个以上的算法可以决定适当的 offsets 以及各种调整，但这些素材实在太过诡谲迷离，不适合再此处讨论！我的建议是，不要在一个 virtual base class 中声明 nonstatic data members。如果这么做，你会距离复杂的深渊愈来愈近，终不可拔。

## 4.3 函数的效能

在下面这组测试中，我在不同的编译器上计算两个 3D 点，其中用到一个 nonmember friend function、一个 member function，以及一个 virtual member function，并且 Virtual member function 分别在单一、虚拟、多重继承三种情况下执行。下面就是nonmember function:
```cpp
void
cross_product(const Point3d& pA, const Point3d& pB)
{
    Point3d pC;
    pC.x = pA.y * pB.z - pA.z * pB.y;
    pC.y = pA.z * pB.x - pA.x * pB.z;
    pC.z = pA.x * pB.y - pA.y * pB.x;
}
```
main() 函数看起来像这样（调用的是 nonmember function）:
```cpp
main(){
    Point3d pA(1.725, 0.875, 0.478);
    Point3d pB(0.315, 0.317, 0.838);

    for(int iters = 0; iters < 10000000; iters++)
        cross_product(pA, pB);
    return 0;
}
```
如果调用不同形式的函数，当然 main() 就需要改变。表格 4.1 列出测试结果:

表格4.1 函数效率
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|:-:|:-:|
|Inline Member|0.08|4.70|
|Nonmember Friend|4.43|6.13|
|Static Member|4.43|6.13|
|Virtual Member|||
|CC|4.76|6.90|
|NCC|4.63|7.72|
|Virtual Member（多重继承）|||
|CC|4.90|7.06|
|NCC|4.96|8.00|
|Virtual Member（虚拟继承）|||
|CC|5.20|7.07|
|NCC|5.44|8.00|

一如 4.2 节所讨论的，nonmember 或 static member 或 nonstatic member 函数都被转化为完全相同的形式。所以我们毫不惊讶地看到三者的效率完全相同。

但是我们很惊讶的发现，未优化的 inline 函数提高了 25% 左右的效率。而其优化版本的表现简直是奇迹。这是怎么回事？

这项惊人的结果归功于编译器 “将视为不变的表达式（expressions）” 提到循环之外，因此只计算一次。此例显示，inline 函数不只能够节省一般函数调用所带来的额外负担，也提供程序优化的额外机会。

我对 virtual function 的调用时通过一个 reference，而不是通过一个对象，由此我就可以确定调用操作确实经过了虚拟机制。效率的降低程度从 4% 到 11% 不等，其中的一部分反映出 Point3d constructor 对于 vptr 一千万次的设定操作，其它则因为 CC 和 NCC 两者（或至少在 NCC 的 “cfront 兼容模式” 中）使用了 delta-offset（偏移差值）模型来支持 virtual functions。

在该模型中，需要一个 offset 以调整 this 指针，指向放置于 virtual table 中的适当地址。所有像这样的调用形式：
```cpp
ptr->virt_func();
```
都会被转换为：
```cpp
// 调用virtual function
(*ptr->__vptr[index].addr)
    (ptr + ptr->__vptr[index].delta)    // 将this指针的调整之传递过去
```
甚至即使在大部分调用操作中，调整值都是 0（只有在第二或后继的 base class 或 virtual base class 的情况下，其调整值才不是 0）。在这种实现技术之下，不论是单一继承或多重继承，只要是虚拟调用操作，就会消耗相同的成本。当然，在 chunk 模型中，this 指针的调整成本可以被局限于有必要那么做的函数中。

多重继承中的 virtual function 的调用，似乎用掉了较多的成本，这令人感到困惑。当有人期望编译器实现出 thunk 模型，以调整第二个或后继的 base class 的 virtual function，而用来测试的这两个编译器却不支持 thunk 技术，就会得到这种结果。由于 this 指针的调整已经施行于单一继承和多重继承中，故其额外负担不能用来解释这项成本。

当我在单一继承情况下执行这项测试时，我也困惑地发现，每多一层继承，virtual function 的执行时间就有明显的增加。一开始我无法想象其间发生了什么事。然而当我成为这些程序代码的 “最佳男主角” 够久了之后，我终于渐悟了。不管单一继承的深度如何，主循环用以调用函数的码事实上是完全相同的；同样道理，对于坐标值的处理也是完全相同的。期间的不同，同时也是稍早前我没有考虑到的，就是 cross_product() 中出现的局部性 Point3d class object pC。于是 default Point3d constructor 被调用了一千万次。增加继承深度，就多增加执行成本，这一事实上反映出 pC 身上的 constructor 的复杂度。这也能解释为什么多重继承的调用另有一些额外负担。

导入 virtual function 之后，class constructor 将获得参数以设定 virtual table 指针。CC 和 NCC 都不能够把这个设定操作在 “无任何 virtual function 之base class” 建构时优化，所以每多一层继承，就会多增加一个额外的 vptr 设定。此外，不论在 CC 或 NCC，下面这个测试操作会被安插到 constructor中，以回溯兼容于 C++ 2.0：
```cpp
// 在每一个base和derived class constructor中被调用
if(this || this = new(sizeof(*this)))
    /// User code goes here
```
在导入 new 和 delete 运算符之前，承担 class 内存管的唯一方法就是在 constructor 中指定 this 指针。刚才的 if 判断也支持该做法。对于 cfront，“this的指定操作” 的语意回溯兼容性一直到 4.0 版才活得保证（由于各种超越文本范围的神秘理由之故）。有讽刺意味的是，NCC 是 SGI 产品的 “与 cfront 兼容” 模式，所以 NCC 也提供了这种回溯兼容性。除了回溯兼容，不再有任何理由需要在 constructor 中含如刚才的 if 测试。现代的编译器把 new 运算符的调用操作分离开来，就像把一个运算从 constructor 的调用中分离出来一样（6.2 节）。“this 指定操作” 的语意不再由语言来支持。

在这些编译器中，每一个额外的 base class 或额外的单一继承层次，其 constructor 内会加入另一个对 this 指针的测试（就本例而言是不必要的）。若执行这些constructor 一千万次，效率就会因此下降至可以测试的程度。这种效率表现明显反映出一个编译器的反常，而不是对象模型的不正常。

在任何情况下，我想看看是否 construction 调用操作的额外损失会被视为额外花费的效率时间。我以两种不同的风格重写这个函数，都不实用局部对象:
1. 在函数参数中加上一个对象，用于存放加法的结果：
```cpp
void
cross_product(Point3d& pC, const Point3d& pA, const Point3d& pB)
{
    pC.x = pA.y * pB.z - pA.z * pB.y;
    // 其余相同······
}
```
2. 直接在 this 对象中计算结果
```cpp
void Point3d::
cross_product(const Point3d& pB)
{
    x = y * pB.z - z * pB.y;
    // 其余类似，但我们适用对象的x，y，z
}
```
两种情况中，其未优化的执行平均时间为 6.90 秒。

有趣的是这个语言并不提供一个机制，指示是否一个 default constructor 并非必要而应该省略。也就是说，局部性的 pC class object 即使未被我们使用，它还是需要一个constructor——但我们可以经由消除对局部对象的使用，而消除其 constructor 的调用操作。

## 4.4 指向 Member Function 的指针（Pointer-to-Member Funtions）

在第三章中我们已经看到了，取一个 nonstatic data member 的地址，得到的结果是该 member 在 class 布局中的 byte 位置（再加 1）。你可以想象 它是一个不完整的值，在需要被绑定于某个 class object 的地址上，才能够被存取。

取一个 nonstatic member function 的地址，如果该函数是 nonvirtual，则得到的结果是它在内存中真正的地址。然而这个值也是不完全的，它也需要被绑定于某个 class object 的地址时，才能够通过它调用该函数。所有的 nonstatic member functions 都需要对象的地址（以参数 this 指出）。

回顾一下，一个指向 member function 的指针，其声明语法如下：
```cpp
double          // return type
( Point::*      // class the function is member
    pmf)        // name of pointer to member
();             // argument list
```
然后我们可以这样定义并初始化该指针：
```cpp
double (Point::*coord)() = &Point::x;
```
也可以这样指定其值：
```cpp
coord = &Point::y;
```
或调用它，可以这么做：
```cpp
(origin.*coord)();
```
或
```cpp
(ptr->*coord)();
```
这些操作会被编译器转化为：
```cpp
// 虚拟C++码
(coord)(&origin);
```
和
```cpp
// 虚拟C++码
(coord)(ptr);
```
指向 member function 的指针的声明语法，以及指向 “member selection 运算符” 的指针，其作用是作为 this 指针的空间保留者。这也就是为什么 static member function（没有 this 指针）的类型是 “函数指针”，而不是 “指向 member function 之指针”的原因。

使用一个 “member function 指针”，如果并不用于 virtual function、多重继承、virtual base class 等情况的话，并不会比使用一个 “nonmember function 指针” 的成本更高。上述三种情况对于 “member function 指针” 的类型以及调用都太过复杂。事实上，对于那些没有 virtual functions 或 virtual base class，或 multiple base classes 的 classes 而言，编译器可以为它们提供相同的效率。下一节我要讨论为什么 virtual functions 的出现，会使得 “member function 指针” 更复杂化。

### 支持 “指向 Virtual Member Functions” 之指针

注意下面的程序片段：
```cpp
float (Point::*pmf)() = &Point::z;
Point *ptr = new Point3d;
```
pmf，一个指向 member function 的指针，被设值为 Point:: z()（一个 virtual function）的地址。ptr 则被指定以一个 Point3d 对象。如果我们直接经由 ptr 调用 z():
```cpp
ptr->z();
```
则被调用的是 Point3d:: z()。但如果我们从 *pmf* 直接调用 z() 呢？
```cpp
（ptr->*pmf)();
```
仍然是 Point3d::z() 被调用码？也就是说，虚拟机制仍然能够在使用 “指向 member function 之指针” 的情况下运行吗？答案是yes，问题是如何实现呢？

在前一小节中，我们看到了，对一个 nonstatic member function 取其地址，将获得该函数在内存中的地址。然而面对一个 virtual function，其地址在编译时期是未知的，所能知道的仅是 virtual function 在其相关之 virtual table 中的索引值。也就是说，对一个 virtual member function 取其地址，所能获得的只是一个索引值。

例如，假设我们有以下的 Point 声明:
```cpp
class Point
{
public:
    virtual ~Point();
    float x();
    float y();
    virtual float z();
    // ...
};
```
然后取 destructor 的地址：
```cpp
&Point::~Point;
```
得到的结果是 1。取 x() 或 y() 的地址：
```cpp
&Point::x();
&Point::y();
```
得到的则是函数在内存中的地址，因为它们不是 virtual。取z()的地址：
```cpp
&Point::z();
```
得到的结果是 2。通过 pmf 来调用 z()，会被内部转化为一个编译时期的式子，一般形式如下：
```cpp
(*ptr->vptr[(int)pmf])(ptr);
```
对一个 “指向 member function 的指针” 评估求值（evaluated），会因为该值有两种意义而复杂化；其调用操作也将有别于常规调用操作。pmf的内部定义，也就是：
```cpp
float (Point::*pmf)();
```
必须允许该函数能够寻址出 nonvirtual x() 和 virtual z() 两个 member functions，而那两个函数有着相同的原型：
```cpp
// 二者都可以被指定给pmf
float Point::x() { return _x; }
float Point::z() { return 0; }
```
只不过其中一个代表内存地址，另一个代表 virtual table 中的索引值。因此编译器必须定义 pmf 使它能够（1）含有两种数值，（2）更重要的是其数值可以被区别代表内存地址还是 Virtual table 中的索引值。你有好点子吗？

在 cfront2.0 非正式版中，这两个值被内含在一个普通的指针内。cfront 如果识别该值是内存地址还是 virtual table 索引呢？它使用了如下技巧：
```cpp
(((int)pmf)&~127)
    ?       // non-virtual invocation
    (*pmf)(ptr)
    :       // virtual invocation
    (*ptr->vptr[(int)pmf](ptr));
```

一如 Stroustrup 在 [LIPP88] 中所说：

当然啦，这种实现技巧必须假设继承体系中最多只有 128 个 virtual functions。这并不是我们所希望的，但却证明是可行的。然而，多重继承的引入，导致需要更一般化的实现方式，并趁机除去对 virtual functions 的数目限制。

### 在多重继承之下，指向Member Functions的指针

为了让指向 member functions 的指针也能够支持多重继承和虚拟继承，Stroustrup 设计了下面一个结构体（[LIPP88] 中有其原始内容）：
```cpp
// 一般结构，用以支持
// 在多重继承之下指向member functions的指针
struct __mptr{
    int delta;
    int index;
    union{
        ptrtofunc faddr;
        int v_offset;
    };
};
```
它们要表现什么呢？index 和 faddr分别（不同时）带有 virtual table 索引和 nonvirtual member function 地址（为了方便，当 index 不指向 virtual table 时，会被设为 -1）。在该模型之下，像这样的调用操作：
```cpp
(ptr->*pmf)();
```
会变成：
```cpp
(pmf.index < 0)
    ?   // non-virtual invocation
    (*pmf.faddr)(ptr)
    :   // virtual invocation
    (*ptr->vptr[pmf.index](ptr));
```
这种方法所受到的批评是，每个调用操作都得付出上述成本，检查其是否为 virtual 或 nonvirtual。Microsoft 把这项检查拿掉，导入一个它所谓的 vcall thunk。在此策略之下，faddr 被指定的要不就是真正的 member function 地址（如果函数是 nonvirtual 的话），要不就是 vcall thunk 的地址。于是 virtual 或 nonvirtual 函数的调用操作透明化，vcall thunk 会选出并调用相关 virtual table 中的适当 slot。

这个结构体的另一个副作用就是，当传递一个不变值得指针给 member function 时，它需要产生一个临时性对象。举个例子，如果你这么做：
```cpp
extern Point3d foo(const Point3d&, Point3d (Point3d::*)());
void bar(const Point3d& p){
    Point3d pt = foo(p, &Point3d::normal);
    // ...
}
```
其中 &Point3d::normal 的值类似这样：
```cpp
{0, -1, 10727417}
```
将产生一个临时性对象，有明确的初值：
```cpp
// 虚拟C++代码
__mptr temp = {0, -1, 10727417}

foo(p, temp);
```
再回到本节一开始的那个结构体。delta 字段表示 this 指针的 offset 值，而 v_offset 字段放的是一个 virtual (或多重继承中的第二或后继的) base class 的 vptr 位置。如果 vptr 被编译器放在 class 对象的起始处，这个字段就没有必要了，代价则是 C 对象兼容性降低（请回顾 3.4 节）。这些字段只在多重继承或虚拟继承的情况下才有其必要性，有许多编译器在自身内部根据不同的 classes 特性提供多种指向 member functions 的指针形式，例如 Microsoft 就供应了三种风味：
1. 一个单一继承实例（其中带有 vcall thunk 地址或是函数地址）；
2. 一个多重继承实例（其中带有 faddr 和 delta 两个 members）；
3. 一个虚拟继承实例（其中带有四个 members）

### “指向Member Functions之指针”的效率

在下面一组测试中，cross_product() 函数经由以下方式调用：
1. 一个指向 nonmember function 的指针；
2. 一个指向 class member function 的指针；
3. 一个指向 virtual member function 的指针；
4. 多重继承下的 nonvirtual 及 virtual member function call;
5. 虚拟继承下的 nonvirtual 及 virtual member function call;

第一个测试（指向 nonmember function 的指针）以下列方式进行：
```cpp
main(){
    Point3d pA(1.725, 0.875, 0.478);
    Point3d pB(0.315, 0.317, 0.838);
    Point3d* (*pf) (const Point3d&, const Point3d&) = cross_product;
    for(int iters = 0; iters < 10000000; iters++)
        (*pf)(pA, pB);
    return 0;
}
```
第二个测试，“指向 member function 之指针”的声明和调用操作如下：
```cpp
Point3d* (Point3d::*pmf)(const Point3d&) const = 
        &Point3d::cross_product;        // 译注：加不加&，效果一样。
for(int iters = 0; iters < 10000000; iters++)
    (pA.*pmf)(pB);
```
不论在 CC 或 NCC 中，都会把上述操作转化为以下的内部形式，于是以下的函数调用：
```cpp
(pA.*pmf)(pB);      // 译注：原书为(*pA.pmf)(pB);  恐为笔误
```
会被转化为这样的判断:
```cpp
pmf.index < 0
    ? (*pmf.faddr)(&pA + pmf.delta, pB) // 译注：原书少写最后的pB
    : (*pA.__vptr__Point3d[pmf.index].faddr)
    (&pA + pA.__vptr__Point3d[pmf.index].delta, pB);
```
还记得吗，一个 “指向 member function 之指针” 是一个结构，内含三个字段：index、faddr 和 delta。index 若不是内带一个相关 virtual table的索引值，就是以 -1 表示函数是 nonvirtual。faddr 带有 nonvirtual member function 的地址。delta 带有一个可能的 this 指针调整值。表格 4.2 显示测试的结果。

表格4.2 函数指针的效率
||<div style = 'width:80pt'>优化</div>|<div style = 'width:80pt'>未优化</div>|
|-|-|-|
|指向Nonmember Function之指针|(void(*p)|(...))|
||4.30|6.12|
|指向Member Function之指针（PToM):Non-Virtual||
|CC|4.30|6.38|
|NCC|4.89|7.65|
|PToM:多重继承：Nonvirtual|||
|CC|4.30|6.32|
|NCC|5.25|8.00|
|PToM:虚拟继承：Novirtual|||
|CC|4.70|6.84|
|NCC|5.31|8.07|
|指向Member Function之指针（PToM）:Virtual|||
|CC|4.70|7.39|
|NCC|5.64|8.40|
|PToM:多重继承：Virtual|||
|（注意：CC产生出不良的码，导致Segment Faulted）|||
|NCC|5.90|8.72|
|PToM:虚拟继承：Virtual|||
|（注意：CC产生出不良的码，导致Segment Faulted）|||
|NCC|5.84|8.80|

## 4.5 Inline Functions

下面是 Point class 的一个加法运算符的可能实现内容：
```cpp
class Point{
    friend Point
        opeeraotr+(const Point&, const Point&);
}
Point
operator+(const Point& lhs, const Point& rhs)
{
    Point new_pt;
    new_pt._x = lhs._x + rhs._x;
    new_pt._y = lhs._y + rhs._y;
    
    return new_pt;
}
```
理论上，一个比较 “干净” 的做法是使用 inline 函数来完成 set 和 get 函数：
```cpp
// void Point::x(float new_x) { _x = new_x; }
// float Point::x() { return _x; }

new_pt.x(lhs.x() + rhs.x() );
```
由于我们受限只能在上述两个函数中对 _x 直接存取，因此也就将稍后可能发生的 data members 的改变（例如在继承体系中上移或下移）所带来的冲击最小化了。如果把这些存取函数声明为 inline，我们就可以继续保持直接存取 members 的那种高效率——虽然我们兼顾了函数的封装性。此外，加法运算符不再要被声明为 Point 的一个 friend。

然而，实际上我们并不能够强迫将任何函数都变成 inline——虽然 cfront 客户一度曾经发出一封权限极高的修改需求，要求我们加上一个 must_inline 关键词。关键词 inline（或 class declaration 中的 member function 或 friend function 的定义）只是一项请求。如果这项请求被接受，编译器就必须认为它可以用一个表达式（expression
合理地将这个函数扩展开来。

当我说 “编译器相信它可以合理地扩展一个 inline 函数” 时，我的意思是在某个层次上，其执行成本比一般的函数调用及返回机制所带来的负荷低。cfront 有依靠复杂的测试法，通常是用来计算 assignment、function calls、virtual function call 等操作的次数。每个表达式（expression）种类有一个权值，而 inline 函数的复杂度就以这些操作的总和来决定。

一般而言，处理一个 inline 函数，有两个阶段：
1. 分析函数定义，以决定函数的 “intrinsic inline ability”（本质的 inline 能力）。“intrinsic”（本质的、固有的）一词在这里意指“与编译器相关”。

如果函数因其复杂度，或因其建构问题，别判断不可成为 inline，它会被转为一个 static 函数，并在 “被编译模块” 内产生对应的函数定义。在一个支持 “模块个别编译” 的环境中，编译器几乎没有什么权宜之计。理想情况下，链接器会将被产生出来的重复东西清理掉，然而一般来说，目前市面上的链接器并不会将 “随该调用而被产生出来的重复调试信息” 清理掉。UNIX 环境中的 strip 命令可以达到这个目的。

2. 真正的 inline 函数扩展操作时在调用的那一点上。这会带有参数的求值操作（evaluation）以及临时性对象的管理。

同样是在拓展点上，编译器将决定这个调用是否 “不可为 inline”。在 cfront 中，inline 函数如果只有一个表达式（expression），则其第二或后继的调用操作：
```cpp
new_p.x(lhs.x() + rhs.x());
```
就不会被扩展开来。这是因为在 cfront 中它被变成：
```cpp
// 虚拟C++码。建议的inline扩展形式
new_pt.x = lhs._x + x__5PointFV(&rhs);
```
这就完全没有带来效率上的改善！对此，我们唯一能够做的就是重写其内容；
```cpp
// 真恶心：修正inline suppor ☺
new_pt.x(lhs._x + rhs._x);
```
其中的批注，是要让未来读这份码的人知道，我们曾考虑使用 public inline interface，但却必须走回头路。

其它编译器在处理 inline 的扩展时，有像 cfront 这样的束缚码？不！然而，很不幸地，大部分编译器厂商（UNIX 和 PC 都有）似乎认为不值得在 inline 支持技术上做详细的讨论。通常你必须进入到汇编器（assembler）中才能看到是否真正实现了 inline。

### 形式参数（Formal Arguments）

在 inline 扩展期间，到底真正发生了什么事情？奥，是的，每一个形式参数都会被对应的实际参数取代。如果说有什么副作用，那就是不可以只是简单地一一封塞程序中出现的每一个形式参数，因为这将导致对于实际参数的多次求值操作（evaluations）。一般而言，面对 “会带来副作用的实际参数”，通常都需要引入临时性对象。换句话说，如果实际参数是一个常量表达式（constant expression），我们可以在替换之前先完成其求值操作（evaluations）；后继的 inline 替换，就可以把常量直接 “绑” 上去。如果既不是个常量，也不是个带有副作用的表达式，那么就直接代换之。

举个例子，假设我们有以下的简单 inline 函数：
```cpp
inline int
min(int i, int j)
{
    return i < j ? i : j;
}
```
下面是三个调用操作：
```cpp
inline int
bar()
{
    int minval;
    int val1 = 1024;
    int val2 = 2048;

/* (1) */ minval = min(val1, val2);
/* (2) */ minval = min(1024, 2048);
/* (3) */ minval = min(foo(), bar() + 1);
    return minval;
}
```
表示为(1)的那一行会被扩展为：
```cpp
// (1) 参数直接代换
minval = val1 < val2 ? val2 : val2;
```
标示为(2)的那一行直接拥抱常量:
```cpp
// (2) 代换之后，直接使用常量
minval = 1024;
```
标示为(3)的那一行则引发参数的副作用。它需要引入一个临时对象，以避免重复求值（multiple evaluations):
```cpp
// (3) 有副作用，所以导入临时对象
int t1;
int t2;

minval = 
    (t1 = foo(), (t2 = bar() + 1)),
    t1 < t2 ? t1 : t2;
```

### 局部变量 （Local Variables）

如果我们轻微地改变定义，在 inline 定义中加入一个局部变量，会怎样：
```cpp
inline int
min(int i, int j)
{
    int minval = i < j ? i : j;
    return minval;
}
```
这个局部变量需要什么额外的支持或处理吗？如果我们以以下的调用操作：
```cpp
{
    int local_var;
    int minval;
    // ...
    minval = min(val1, val2);
}
```
inline 被扩展开来后，为了维护其局部变量，可能会成为这个样子（理论上这个例子中的局部变量可以被优化，其值可以直接在 minval 中计算）：
```cpp
{
    int local_var;
    int minval;

    // 将inline函数的局部变量处以“mangling”操作
    int __min_lv_minval;
    minval = 
        ( __min_lv_minval = 
            val1 < val2 ? val1 : val2),
            __min_lv_minval;
}
```
一般而言，inline 函数中的每一个局部变量都必须放在函数调用的一个封闭区段中，拥有独一无二的名称。如果 inline 函数以单一表达式（expression）扩展多次，那么每次扩展都需要自己的一组局部变量。如果 inline 函数以分离的多个式子（discrete statements）被扩展多次，那么只需一组局部变量，就可以重复使用（译注：因为它们被放在一个封闭区段中，有自己的 scope）。

inline 函数中的局部变量，再加上有副作用的参数，可能会导致大量临时性对象的产生。特别是如果它以单一表达式（expression）被扩展多次的话。例如下面的调用操作：
```cpp
minval = min(val1, val2) + min(foo(), foo() + 1);
```
可能被扩展为：
```cpp
// 为局部变量产生的临时变量
int __min_lv_minval_00;
int __min_lv_minval_01;
// 为放置副作用值而产生临时变量
int t1;
int t2;

minval = 
    ((__min_lv_minval__00 = 
    val1 < val2 ? val1 : val2),
    __min_lv_minval_00)
    +
    ((__min_lv_minval__01 = t1 = foo()),
        (t2 = foo() + 1),
        t1 < t2 ? t1 : t2),
        __min_lv_minval__01);
```
Inline 函数对于封装提供了一种必要的支持，可以有效存取封装与 class 中的 nonpublic 数据。它同时也是 C 程序中大量使用的 #define（前置处理宏）的一个安全替代品——特别是如果宏中的参数有副作用的话。然而一个 inline 函数如果被调用太多次的话，会产生大量的扩展码，是程序的大小暴涨。

一如我所描述过的，参数带有副作用，或是以一个单一表达式做多重调用，或是在 inline 函数中有多个局部变量，都会产生临时性对象，编译器也许（或也许不）能够把它们移除。此外，inline 中再有 inline，可能会使一个表面上看起来平凡的 inline 却因其连锁复杂性而没办法展开来。这种情况可能发生于复杂 class 体系下的 constructors，或是 object 体系中一些表面上并不正确的 inline  调用所组成的串链——它们每一个都会执行一小组运算，然后对另一个对象发出请求。对于既要安全又要效率的程序，inline 函数提供了一个强而有力的工具。然而，与 non-inline 函数比起来，它们需要更加小心地处理。


[第 3 章 Data 语意学（The Semantics of Data）](The_Semantics_of_Data.md)|[第 5 章 构造、解构、拷贝 语意学（Semantics of Constructor, Destructor, and Copy）](Semantics_of_Constructor_.md)
