# 第6章 执行期语意学（Runtime Semantics）

想象一下我们有下面这个简单的式子：
```cpp
if(yy == xx.getValue()) ...
```
其中 xx 和 yy 定义为：
```cpp
X xx;
Y yy;
```
class Y 定义为：
```cpp
class Y{
public:
    Y();
    ~Y();
    bool operator==(const Y&) const;
    // ...
};
```
class X 定义为：
```cpp
class X{
public:
    X();
    ~X();
    operator Y() const;     // 译注：conversion 运算符
    X getValue();
    // ...
};
```
真的简单，不是吗？好，让我们看看本章一开始的那个表达式该如何处理。

首先，让我们决定 equality（等号）运算符所参考到的真正实体。在这个例子中，它被决议（resolves）为 “被 overloaded 的 Y 成员实体”。下面是该式子的第一个转换：
```cpp
// resolution of intended operator
if (yy.operator==(xx.getValue()))
```
Y 的 equality（等号）运算符需要一个类型为 Y 的参数，然而 getValue() 传回的却是一个类型为 X 的 object。若非有什么方法可以把一个 X object 转换为一个 Y object，那么这个式子就算错！本例中 X 提供一个 conversion 运算符，把一个 X object 转换为一个 Y object。它必须施行于 getValue() 的返回值身上。下面是该式子的第二次转换：
```cpp
// conversion of getValue()'s return value
if (yy.operator==(xx.getValue().operator Y()))
```
到目前为止所发生的一切都是编译器根据 class 的隐含语意，对我们的程序代码所做的 “增胖” 操作。如果我们需要，我们也可以明确地写出那样的式子。不，我并没有建议你那么做，不过如果你那么做，会使编译器速度稍微快一些。

虽然程序的语意是正确的，但其教育性却尚不能说是正确的。接下来我们必须产生一个临时对象，用来放置函数调用所传回的值：
1. 产生一个临时的 class X object，放置 getValue() 的返回值：
```cpp
X temp1 = xx.getValue();
```
2. 产生一个临时的 class Y object，放置 operator Y() 的返回值：
```cpp
Y temp2 = temp1.operator Y();
```
3. 产生一个临时的 int object，放置 equality（等号）运算符的返回值：
```cpp
int temp3 = yy.operator == (temp2);
```
最后，适当的 destructor 将被施行于每一个临时性的 class object 身上。这导致我们的式子被转换为以下形式：
```cpp
// C++伪码
// 以下是条件句if(yy == xx.getValue()) ...的转换
{
    X temp1 = xx.getValue();
    Y temp2 = temp1.operator Y();
    int temp3 = yy.operator == (temp2);

    if(temp3) ...

    temp2.Y::~Y();
    temp1.X::~X();
}
```
唔，似乎不少呢！这是 C++ 的一件困难事情：不太容易从程序代码看出表达式的复杂度。本章之中，我要带领各位看看执行期所发生的一些转换。我会在 6.3 节回到 “临时性生成物” 这个主题上。

## 6.1 对象的构造和解构（Object Construction and Destruction）

一般而言，constructor 和 destructor的安插都如你所预期：
```cpp
// C++伪码
{
    Point point;
    // point.Point::Point() 一般而言会被安插在这里
    ...
    // point.Point::~Point() 一般而言会被安插在这里
}
```
如果一个区段（译注：以{}括起来的区域）或函数中有一个以上的离开点，情况会稍微混乱一些。Destructor 必须被放在每一个离开点（当时 object 还存活）之前，例如：
```cpp
{
    Point point;
    // constructor 在这里行动
    switch(int(point.x())){
        case -1:
            // mumble;
            // destructor 在这里行动
            return;
        case 0:
            // mumble;
            // destructor 在这里行动
            return;
        case 1:
            // mumble;
            // destructor 在这里行动
        default:
            // mumble;
            // destructor 在这里行动
            return;
    }
    // destructor 在这里行动
}
```
在这个例子中，point 的 destructor 必须在 switch 指令的四个出口的 return 操作前被生成出来。另外也很有可能在这个区段的结束符号（右大括号）之前被生成出来——即使程序分析的结果发现绝不会进行到哪里。

同样的道理，goto 指令也可能需要许多个 destructor 调用操作。例如下面的程序片段：
```cpp
{
    if(cache)
        // 检查cache；如果吻合就传回1
        return 1;   // 译注：原书少了这一行，应为作者笔误

    Point xx;
    // xx的constructor 在这里行动

    while(cvs.iter(xx))
        if(xx == value)
            goto found;
    // xx的destructor 在这里行动
    return 0;

found:
    // cache item
    // xx的destructor 在这里行动
    return 1;
}
```
Destructor 调用操作必须被放在最后两个 return 指令之前，但是却不必被放在最初的 return 之前，那当然是因为那时 object 尚未被定义出来！

一般而言我们会把 object 尽可能放置在使用它的那个程序区段附近，这样做可以节省不必要的对象产生操作和摧毁操作。以本例而言，如果我们在检查 cache 之前就定义了 Point object，那就不够理想。这个道理似乎非常明显，但许多 pascal 或 C 程序员使用 C++ 的时候，仍然习惯把所有的 objects 放在函数或某个区段的起始处。

### 全局对象（Global Objects）

如果我们有以下程序片段：
```cpp
Matrix identity;

main()
{
    // identity 必须在此被初始化
    Matrix m1 = identity;
    ...
    return 0;
}
```
C++ 保证，一定会在 main() 函数中第一次用到 identity 之前，把 identity 构造出来，而在 main() 函数结束之前把 identity 摧毁掉。像 identity 这样的所谓 global object 如果有 constructor 和 destructor 的话，我们说它需要静态的初始化操作和内存释放操作。

C++ 程序中所有的 global objects 都被放置在程序的 data segment 中。如果明确指定给他一个值，object 将以该值为初值。否则 object 所配置到的内存内容为 0.因此在下面这段码中：
```cpp
int v1 = 1024;
int v2;
```
v1 和 v2 都被配置于程序的 data segment，v1 值为 1024，v2 值为 0（这和 C 略有不同，C 并不自动设定初值）。在 C 语言中一个 global object 只能够被一个常量表达式（可在编译时期求其值的那种）设定初值。当然，constructor 并不是常量表达式。虽然 class object 在编译时期可以被放置于 data segment 中并且内容为 0，但 constructor 一直要到程序激活（startup）时才会实施。必须对一个 “放置于 program data segment 中的 object 的初始化表达式” 做评估（evaluate），这正是为什么一个 object 需要静态初始化的原因。

> @zl: C语言中整型全局变量不是默认初始化为 0 吗？

当 cfront 还是唯一的 C++ 编译器，而且跨平台移植性比效率的考虑更重要的时候，有一个可移植但成本颇高的静态初始化（以及内存释放）方法，我把它称为 munch。cfront 的束缚是，它的解决方案必须在每一个 UNIX 平台上——从 Cray 到 VAX 到 Sun 到 UNIX PC（AT&T 曾短暂地推过这项产品）——都有效。因此不论是相关的 linker 或 object-file format，都不能预先做任何假设。由于有这样的限制，下面这些 munch 策略就浮现出来了：

1. 为每一个需要静态初始化的档案产生一个 _sti() 函数，内带必要的 constructor 调用操作或 inline expansions。例如前面所说的 identity 对象会在 matrix.c 中产生出下面的 _sti() 函数（译注：我想 sti 就是 static initialization 的缩写）：
```cpp
__sti__matrix_c__identity(){
    // C++伪码
    identity.Matrix::Matrix();  // 译注：这就是static initialization
}
```
其中 matrix_c 是文件名编码，_identity 表示文件中所定义的第一个 static object（译注：原书写的是 nonstatic object，不过我想应该是 static object）在 __sti 之后附加上这两个名称，可以为可执行文件提供一个独一无二的识别符号（Andy Koenig 和 Bjarne 两人努力设计出这种编译体制，以解决 Jim Coplien 所提出的名称冲突的困扰）。
2. 类似情况，在每一个需要静态的内存释放操作（static dellocation）的文件中，产生一个 _std() 函数（译注：我想 std 就是 static dellocation 的缩写），内带必要的 destructor 调用操作，或是其 inline expansions。在我们的例子中会有一个 _std() 函数被产生出来，针对 identity 对象调用 Matrix destructor。
3. 提供一组 runtime library “munch” 函数：一个 _main() 函数（用以调用可执行文件中的所有 _sti() 函数），以及一个 exit() 函数（以类似方法调用所有的 _std() 函数）。

译注：我把上面的意思整理于下：
```cpp
int main()
{
    _main();    // {
                //   __sti_xxx();
                //   __sti_xxx();
                //   __sti_xxx();
                // } // 对所有的global objects做static initialization动作
    ...
    ...
    ...
    exit();     // {
                //   __std_xxx();
                //   __std_xxx();
                //   __std_xxx();
                // } // 对所有的global objects做static deallocation动作
}
```
cfront 在你的程序中安插一个 _main() 函数调用操作，作为 main() 函数的第一个指令。这里的 exit() 和 C library 的 exit() 不同，为了链接前者，在 cfront 的 CC 命令中必须先指定 C++ standard library。一般而言这样就可以了，但对于不同的平台上的 cfront，仍然可能存在某些变化，例如 HP 工作站上的编译系统最初就拒绝拨 出 munch exit() 函数，所持理由如今我已经忘记了（谢天谢地），不过我记得当初是十分令人失望的。这样的绝望来自于使用者发现他或者她的 static destructors 并没有被调用起来。

最后一个需要解决的问题是，如何收集一个程序中各个 object files 的 _sti() 函数和 _std() 函数。还记得吗，它必须视可移植的——虽然移植性限制在 UNIX 平台。花一点时间想想，你如何解决这个问题。这不算是技术上的挑战，但在当时，cfront（也就代表 C++）若要成功地流行于各 pigtail，必须依靠它。

我们的解决方法是使用 nm 命令。nm 会倾印出 object file 的符号表格项目（symbol table entries）。一个可执行文件系由 .o 文件产生出来，nm 将施行于可执行文件身上。其输出被导入（“piped into”）munch 程序中（我印象中是 Rob Murray 写出 munch 程序，其它有贡献的人士我就不记得了）。munch 程序会 “用力咀嚼” 符号表格中的名称，搜寻以 __sti 或 __std 开头的名称（是的，值得安慰的是那些目标是以 __sti 或 __std 开头，例如 __sti_ha_fooled_you），然后把函数名称加到一个 sti() 函数和 std() 函数的跳离表格（jump table）中。接下来它把这个表格写到一个小的 program text 文件中，然后，听来或许诡异，CC 命令被重新激活，将这个内含表格的文件加以编译。然后整个可执行文件被重新链接。_main() 和 exit() 于是在各个表格上走访以便，轮流调用每一个项目（代表一个函数地址）。

这个做法可以解决问题，但似乎离正统的计算机科学远了一些。在 System V 1.0 版的时代，其修订版 (patch) 中有一个比较快速的变种方法（我记得是 Jerry Schwarz 完成的）。修订版（patch）假设可执行文件是 System V COFF(Common Object File Format) 格式，于是它检验可执行文件并找出那些 “有着 __link nodes” 并 “内带一个指针，指向 _sti() 函数和 _std() 函数” 的文件，将它们统统串链在一起。接下来它把链表的根源设为一个全局性的 _head object（定义于新的 patch runtime library 中）。这个 patch library 内带另一种不同的 _main() 函数和 exit() 函数，将以 _head 为起始的链表走访一遍。最后，针对 Sun、BSD 以及 ELF 的其它 patch libraries 终于也由各个使用者团体捐赠出来，用以和各式各样的 cfront 版本搭配。

当特定平台上的 C++ 编译器开始出现时，更有效率的方法也就有可能随之出现，因为个平台上有可能扩充链接器和目标文件格式（object file format）,以求直接支持静态初始化和内存释放操作。例如，System V 的 Executable and Linking Fromat(ELF) 被扩充以增加支持 .init 和 .fini 两个 sections（译注），这两个 sections 内带对象所需要的信息，分别对应于静态初始化和释放操作。编译器特定（Implementation-specific）的 starup 函数（通常名为 crt0.o）会完成平台特定（platform-specific）的支持（分别针对静态初始化和释放操作的支持）。

译注：所谓 section 就是 16 位时代所说的 segment，例如 code segment 或 data segment 等等。System V 的 COFF 格式对不同目的的 sections （放置不同的信息）给与不同的命名，例如 .text section、.idata section、.edata section、.src 等等。每一个 section名称都以字符 "."开头。不过有时候我们仍然沿用过去的习惯用语，如 data segment（本章稍早曾出现过）或 code segment。

cfront 2.0 版之前并不支持 nonclass object 的静态初始化操作：也就是说，C 语言的限制仍然残留着。所以，像下面这样的例子，每一个初始化操作都被标示为不合法：
```cpp
extern int i;

// 全部都要求静态初始化（static initialization）
// 在2.0版之前的C和C++中，这些都是不合法的
int j = i;
int *pi= new int(i);
double sal = compute_sal(get_employee(i));
```
支持 “nonclass objects 的静态初始化”，在某种程度上是支持 virtual base classes 的一个副产品。virtual base classes 怎么会扯进这个主题呢？哦，以一个 derived class 的 pointer 或 reference 来存取 virtual base class subobject，使一种 nonconstant expression，必须在执行期才能加以评估求值。例如，尽管下列程序片段在编译器时期可知：
```cpp
// constant expression
Vertex3d *pv = new PVertex;
Point3d *p3d = pv;
// 译注：此处所使用的class继承体系系采用第5章#211页的描述
```
其 virtual base class Point 的 subobject 在每一个 derived class 中的位置却可能会变动，因此不能够在编译时期设定下来。下面的初始化操作：
```cpp
// Point是Point3d的一个virtual base class
// pt的初始化操作需要
// 某种形式的执行期评估（runtime evaluation）
Point *pt = p3d;
```
需要编译器提供内部扩充，以支持 class object 的静态初始化（至少涵盖 class objects 的指针和 refernces）。例如：
```cpp
// Initial support of virtual base class conversion
// requires non-constant initialization support
Point *pt = p3d->vbcPoint;
```
提供必要的支持以涵盖所有的 nonclass objects，并不需要走太远的路。

使用被静态初始化的 objects 有一些缺点。例如，如果 exception handling 被支持，那些 objects 将不能够被放置于 try 区段之中。这对于被静态调用的 constructors 可能是特别无法接受的，因为任何的 throw 操作将必然触发 exception handing library 默认的 terminate() 函数。另一个缺点是为了控制 “需要跨越模块做静态初始化” objects 的相依顺序而扯出来的复杂度（请参考 [SCHWARZ89]，其中有对该问题的第一次讨论，以及如今被称为 Schwarz counters 的东西。如果你需要更广泛的讨论文献，请参考 [CARROLL95]）。我建议你根本就不要用那些需要静态初始化的 global objects（虽然这项建议几乎普遍地不为 C 程序员所接受）。

### 局部静态对象（Local Static Objects）

假设我们有以下程序片段：
```cpp
const Matrix&
identity(){
    static Matrix mat_identity;
    // ...
    return mat_identity;
}
```
Local static class object 保证了什么样的语意？

1. mat_identity 的 constructor 必须只能施行一次，虽然上述函数可能会被调用多次。
2. mat_identity 的 destructor 必须只能施行一次，虽然上述函数可能会被调用多次。

编译器的策略之一就是，无条件地在程序起始（startup）时构造出对象来。然而这会导致所有的 local static class objects 都在程序起始时被初始化，即使它们所在的那个函数从不曾被调用过。因此，只在 identity() 被调用时才把 mat_identity 构造起来，是比较好的做法（现在的 C++ Standard 已经强制要求这一点）。我们应该怎么做呢？

以下即使我在 cfront 之中的做法。首先，我导入一个临时性对象以保护 mat_identity 的初始化操作。第一次处理 identity() 时，这个临时对象被评估为 false，于是 constructor 会被调用，然后临时对象被改为 true。这样就解决了构造的问题。而在相反的一端，destructor 也需要有条件地施行于 mat_identity 身上，但只有在 mat_identity 已经被构造起来时才算数。要判断 mat_identity 是否被构造起来，很简单。如果那个临时对象为 true，就表示构造好了。困难的是，由于 cfront 产生 C 码，mat_identity 对函数而言仍然是 local，因此我没办法在静态的内存释放函数（static deallocation function）中存取它。奥，伤脑筋！解决的方法有点诡异，结构化语言避之唯恐不及：我取出 local object 的地址。（当然啦，由于 object 是 static，其地址在 downstream component 中将会转换到程序内用来放置 global object 的 data segment 中）下面是 cfront 的输出（经过轻微的修润）：
```cpp
// 被产生出来的临时对象，作为戒护之用
static struct Matrix *__0__F3 = 0;

// C++的refernce在C中是以pointer来代替
// identity()的名称会被mangled

struct Matrix*
identity_Fv()
{
    // __1反映出语汇层面的设计，
    // 使得C++得以支持这样的码：
    // int val;
    // int f() { int val;
    //            return val + ::val;}
    // 最后一行会变成：
    // ... return __lval + val;

    static struct Matrix __1mat_identity;

    // 如果临时性的保护对象已被设立，那就什么也别做。否则
    // （a）调用constructor：__ct__6MatrixFv
    // （b）设定保护对象，使它指向目标对象
    __0__F3
        ? 0
        : (__ct__6MatrixFv(&__1mat_identity),
          (__0__F3 = (&__lmat_identity)));
    ...   
}
```
最后，destructor 必须在 “与 text program file（也就是本例中的 stat_0.c）有关联的静态内存释放函数（static deallocation function）” 中被有条件地调用：
```cpp
char __std__stat_0_c_j()
{
    __0__F3
        ? __dt__6MatrixFv(__0__F3, 2)
        : 0;
    ...
}
```
请记住，指针的使用是 cfront 所特有的；然而条件式解构则是所有编译器都需要的。在我下笔之时，C++ 委员会似乎正酝酿要改变 destructor 对于 local class objects 的语意。新的规则要求编译单位中的 static local class objects 必须被摧毁——以构造的相反次序摧毁。由于这些 objects 是在需要时才被构造（例如每一个含有 static local class objects 的函数第一次被进入时），所以编译时无法预期其集合以及顺序。为了支持新的规则，可能需要对被产生出来的 static class objects 保持一个执行期链表。

### 对象数组（Array of Objects）

假设我们有下列的数组定义：
```cpp
Point knots[10];
```
需要完成什么东西呢？如果 Point 既没有定义一个 cosntructor 也没有定义一个 destructor，那么我们的工作不会比建立一个 “内建（build-in）类型所组成的数组” 更多，也就是说，我们只需配置足够的内存以储存 10 个连续的 Point 元素。

然而 Point 的确定义了一个 default destructor，所以这个 destructor 必须轮流施行于每一个元素之上。一般而言这是经由一个或多个 runtim library 函数达成。在 cfront 中，我们使用一个被命名为 vec_new() 的函数，产生出以 class objects 构造而成的数组。比较晚近的编译器，包括Borland、Microsoft 和 Sun，则是提供两个函数，一共用来处理 “没有virtual base class” 的 class，另一个用来处理 “内带 virtual base class” 的 class。后一个函数通常被称为 vec_vnew()。函数类型通常如下（当然在各平台上可能会有些许差异存在）：
```cpp
void *
vec_new(
    void *array,            // 数组起始地址
    size_t elem_size,       // 每一个class object的大小
    int elem_count,         // 数组中的元素数目
    void (*constructor)(void*),
    void (*destructor)(void*, char)
)
```
其中的 constructor 和 destructor 参数是这个 class 的 default constructor 和 default destructor 的函数指针。参数 array 带有的若不是具名数组（本例为 knots）的地址，就是 0. 如果是 0，那么数组将经由应用程序的 new 运算符，被动态配置于 heap 中。Sun 把 “由 class objects 所组成的具名数组” 和 “动态配置而来的数组” 的处理操作分为两个 library 函数：_vector_new2 和 _vector_con，它们各自拥有一个 virtual base class 函数实体。

参数 elem_size 表示数组中的元素数目（我将在 6.2 节讨论 new 和 delete 时再回到这个主题来）。在 vec_new() 中，constructor 施行于 elem_count 个元素之上。对于支持 exception handling 的编译器而言， destructor 的提供是必要的。下面是编译器可能针对我们的 10 个 Point 元素所做的 vec_new() 调用操作：
```cpp
Point knots[10];
vec_new(&knotes, sizeof(Point), 10, &Point::Point, 0);
```
如果 Point 也定义了一个 destructor，当 knots 的声明结束时，该 destructor 也必须施行于那 10 个 Point 元素身上。我想你不会惊讶，这是经由一个类似的 vec_delete()（或是一个 vec_vdelete()——如果 classes 拥有 virtual base classes 的话）的 runtime library 函数完成（Sun 对于 “具名数组” 和 “动态配置而来的数组”，处理方式不同）的，其函数原型如下：
```cpp
void *
vec_delete(
    void *array,                // 数组起始地址
    size_t elem_size,           // 每一个class object的大小
    int elem_count,             // 数组中的元素数目
    void (*destructor)(void*, char)
)
```
有些编译器会另外增加一些参数，用以传递其它数值，以便能够有条件地导引 vec_delete() 的逻辑。在 vec_delete() 中，destructor 被施行于elem_count 个元素身上。

如果程序员提供一个或多个明显初值给一个由 class objects 组成的数组，像下面这样，会如何：
```cpp
Point knots[10] = {
    Point(),
    Point(1.0, 1.0, 0.5),
    -1.0
};
```
对于那些明显获得初值的元素，vec_new() 不再有必要。对于那些尚未被初始化的元素，vec_new() 的施行方式就像面对 “由 class elements 组成的数组，而该数组没有 explicit initialization list” 一样。因此上一个定义很可能被转换为：
```cpp
Point knots[10];

// C++伪码

// 明确地初始化前3个元素
Point::Point(&knots[0]);
Point::Point(&knots[1], 1.0, 1.0, 0.5);
Point::Point(&knots[2], -1.0, 0.0, 0.0);

// 以vec_new初始化后7个元素
vec_new(&knots + 3, sizeof(Point), 7, &Point::Point, 0);
```
### Default Constructors 和数组

如果你想要在程序中取出一个 constructor 的地址，这是不可以的。当然啦，这是编译器在支持 vec_new() 时该做的事情。然而，经由一个指针来激活 constructor，将无法（不被允许）存取 default argument values。This has always resulted in less that first class handling of the initialization of an array of class objects(译注：很抱歉，我未能充分明了这句话的意思)。

举个例子，在 cfront 2.0 之前，声明一个由 class objects 所组成的数组，意味着这个 class 必须没有声明 constructors 或一个 default constructor（没有参数那种）。一个 constructor 不可以取一个或一个以上的默认参数值。这是违反直觉的，会导致以下的大错。下面是 cfront 1.0中对复数函数库（complex library）的声明，你能看出其中的错误吗？
```cpp
class complex{
    complex(double = 0.0, double = 0.0);
    ...
}
```
在当前的语言规则下，此复数函数库的而使用者没办法声明一个由 complex class objects 组成的数组。显然我们在语言的一个陷阱上被绊倒了。在 1.1版，我们修改的是 class library；然而在 2.0 版，我们修改了语言本身。

再一次地，让我们花点时间想想，如何支持以下句子：
```cpp
complex::complex(double = 0.0, double = 0.0);
```
当程序员写出：
```cpp
complex c_array[10];
```
时，而编译器最终需要调用：
```cpp
vec_new(&c_array, sizeof(complex), 10,
        &complex::complex, 0);
```
默认的参数如何能够对 vec_new() 而言有用？

很明显，有多种可能的实现方法。cfront 所采用的方法是产生一个内部的 stub constructor，没有参数。在其函数内调用由程序员提供的 constructor，并将 default 参数值明确地指定过去（由于 constructor 的地址已被取得，所以他不能够成为一个 inline）：
```cpp
// 内部产生的stub constructor
// 用以支持数组的构造
complex::complex()
{
    complex(0.0, 0.0);
}
```
编译器自己又一次违反了一个明显的语言规则：class 如今支持了两个没有带参数的 constructor。当然，只有当 class objects 数组真正被产生出来时，stub 实体才会别产生以及被使用。

## 6.2 new和delete运算符

运算符 new 的使用，看起来似乎是个单一运算，像这样：
```cpp
int *pi = new int(5);
```
但事实上它是由以下两个步骤完成：
1. 通过适当的 new 运算符函数实体，配置所需的内存：
```cpp
// 调用函数库中的new运算符
int *pi = __new(sizeof(int));
```
2. 给配置得来的对象设立初值：
```cpp
*pi = 5;
```
更进一步，初始化操作应该在内存配置成功（经由 new 运算符）后才执行：
```cpp
// new运算符的两个分离步骤
// given: int *pi = new int(5)

// 重写声明
int *pi;
if(pi == __new(sizeof(int)))
    *pi = 5;    // 译注：成功了才初始化
```
delete 运算符的情况类似。当程序员写下：
```cpp
delete pi;
```
时，如果 pi 的值是 0，C++ 语言会要求 delete 运算符不再有操作。因此编译器必须为此调用构造一层保护膜：
```cpp
if(pi != 0)
    __delete(pi);
```
请注意 pi 并不会因此被自动清除为 0，因此像这样的后续行为：
```cpp
// 喔欧：没有良好的定义，但是合法
if(pi && *pi == 5) ...
```
虽然没有良好的定义，但是可能（也可能不）被评估为真。这是因为对于 pi 所指之内存的变更或再使用，可能（也可能不）会发生。

pi 所指对象之生命会因 delete 而结束。所以后继任何对 pi 的参考操作就不再保证有良好的行为，并因此被视为是一种不好的程序风格。然而，把 pi 继续当做一个指针来用，仍然是可以的（虽然其使用受到限制），例如：
```cpp
// ok : pi仍然指向合法空间
// 甚至即使储存于其中的object已经不再合法
if (pi == sentinel) ...
```
在这里，使用指针 pi 和使用 pi 所指之对象，其差别在于哪一个的声明已经结束了。虽然该地址上的对象不再合法，但地址本身却仍然代表一个合法的程序空间。因此 pi 能够继续被使用，但只能在受限制的情况下，很像一个 void* 指针的情况。

以 constructor 来配置一个 class object，情况类似。例如：
```cpp
Point3d *origin = new Point3d;
```
被转换为：
```cpp
Point3d *origin;
// C++伪码
if(origin = __new(sizeof(Point3d)))
    origin = Point3d::Point3d(origin);
```
如果实现出 exception handling，那么转换结果可能会更复杂些：
```cpp
// C++伪码
if(origin = __new(sizeof(Point3d))){
    try{
        origin = Point3d::Point3e(origin);
    }
    catch( ... ){
        // 调用delete library function以
        // 释放因new而配置的内存
        __delete(origin);

        // 将原来的exception上传
        throw;
    }
}
```
在这里，如果以 new 运算符配置 object，而其 constructor 丢出一个 exception，配置得来的内存就会别释放掉。然后 exception 再被丢出去（上传）。

Destructor 的应用极为类似。下面的式子：
```cpp
delete origin;
```
会变成：
```cpp
if(origin != 0){
    // C++伪码
    POint3d::~Point3d(origin);
    __delete(origin);
}
```
如果在 exception handling 的情况下，destructor 应该被放在一个 try 区段中。exception handler 会调用 delete 运算符，然后再一次丢出该 exception。

一般的 library 对于 new 运算符的实现操作都很直接了当，但有两个精巧之处值得斟酌（请注意，以下版本并未考虑 exception handling）：
```cpp
extern void*
operator new(size_t size)
{
    if(size == 0)
        size = 1;
    void *last_alloc;
    while(!(last_alloc = malloc(size)))
    {
        if(_new_handler)
            (*_new_handler)();
        else
            return 0;
    }
    return last_alloc;
}
```
虽然这样写是合法的：
```cpp
new T[0];
```
但语言要求每一次对 new 的调用都必须返回一个独一无二的指针。解决该问题的传统方法是传回一个指针，指向一个默认为 1 byte 的内存区块（这就是为什么程序代码中的 size 被设为 1 的原因）。这个实现技术的另一个有趣之处是，它允许使用者提供一个属于自己的 _new handler() 函数。这正是为什么每一次循环都调用 _new_handler() 之故。

new 运算符实际上总是以标准的 C malloc() 完成，虽然并没有固定一定的这么做不可。相同的情况，delete 运算符也总是以标准的 C free() 完成：
```cpp
extern void
operator delete(void *ptr_)
{
    if(ptr)
        free((char*)ptr);
}
```
### 针对数组的 new 语意

当我们这么写：
```cpp
int *p_array = new int [5];
```
时，vec_new() 不会真正被调用，因为它的主要功能是把 default constructor 施行于 class objects 所组成的数组的每一个元素身上。倒是 new 运算符函数会被调用：
```cpp
int *p_array = (int*)__new(5 * sizeof(int));
```
相同的情况，如果我们写：
```cpp
// struct simple_aggr { float f1, f2;};
simple_aggr *p_aggr = new simple_aggr[5];
```
vec_new() 也不会被调用。为什么呢？因为 simple_aggr 并没有定义一个 constructor 或 destructor，所以配置数组以及清除 p_aggr 数组的操作，只是单纯地获得内存和释放内存而已。这些操作由 new 和 delete 运算符来完成就绰绰有余了。

然而如果 class 定义了一个 default constructor，某些版本的 vec_new() 就会别调用，配置并构造 class objects 所组成的数组。例如这个算式：
```cpp
Point3d *p_array = new Point3d[10];
```
通常会被编译为：
```cpp
Point3d *p_array;
p_array = vec_new(0, sizeof(Point3d), 10,
                &Point3d::Point3d,
                &Point3d::~Point3d);
```
还记得吗，在个别的数组元素构造过程中，如果发生 exception，destructor 就会传递给 vec_new()。只有已经构造妥当的元素才需要 destructor 的施行，因为它们的内存已经被配置出来了，vec_new() 有责任在 exception 发生的时候把那些内存释放掉。

在 c++ 2.0 版之前，将数组的真正大小提供给程序的 delete 运算符，是程序员的责任。因此，如果我们原先写下：
```cpp
int arrat_size = 10;
Point3d* p_array = new Point3d[array_size];
```
那么我么就必须对应地写下：
```cpp
delete [array_size] p_array;
```

在 2.1 版中，这个语言有了一些修改，程序员不再需要在 delete 时指定数组元素的数目，因此我们现在可以这样写：
```cpp
delete [] p_array;
```
然而为了向下兼容，两种形式都可以接受。支持这种新形式的第一个编译器当然就是 cfront，有 Jonathan Shopiro 完成任务。这项技术支持首先需要知道的是指针所指的内存空间，然后施其中的元素数目。

寻找数组维度给 delete 运算符的效率带来极大的影响，所以才导致这样的妥协：只有中括号出现时，编译器才寻找数组的维度，否则它便假设只有单独一个 objects 要被删除。如果程序员没有提供必须的中括号，像这样：
```cpp
delete p_array;     // 喔欧
```
那么就只有第一个元素会被解构。其它的元素仍然存在——虽然其相关的内存已经被要求归还了。

各家编译器之间存在一个有趣的差异，那就是元素数目如果被明显指定，是否会被拿去利用。在 Jonathan 的原始版本中，他优先采用使用者（程序员）明确指定的值。下面是他所写的程序代码的虚拟版本（pseudo-version），附带注释：
```cpp
// 首先检查是否最后一个被配置的项目（__cache_key）
// 是当前要被delete的项目
// 
// 如果是，就不需要做搜寻操作了
// 如果不是，就寻找元素数目

int elem_count = _cache_key == pointer
    ? ((_cache_key = 0), __cache_cout)
    :   // 取出元素数目

// num_elem: 元素数目，将传递给vec_new()
// 对于配置于heap中的数组，只有针对以下形式，才会设定一个值：
// delete [10] ptr;
// 否则cfront会传-1以表示取出
if(num_elem == -1)
    // prefer explicit user size if choice!
    num_elem = ans;
```
然而几乎晚近所有的 C++ 编译器都不考虑程序员的明确指定（如果有的话）：
```cpp
x.c", line3:warning(467)
    delete array size expression ignored (anachronism)
    foo() { delete [12] pi;}
```
为什么 Jonathan 优先采用程序员所指定的值，而新近的编译器却不这么做呢？因为这个性质刚被导入的时候，没有任何程序代码会不 “明确指定数组大小”。 时代演化到 cfront 4.0 的今天，我们已经这个习惯贴上 “落伍” 的标记，并且产生一个类似的警告消息。

应该如何记录元素数目？一个明显的方法就是为 vec_new() 所传回的每一个内存区块配置一个额外的 word，然后把元素数目包藏在那个 word 之中。通常这种被包藏的数值成为所谓的 cookie（小甜饼）。然而，Jonathan 和 Sun 编译器决定维护一个 “联合数组（associative array)”，放置指针及大小。Sun 也把 destructor 的地址维护于此数组之中，请看 [CLAM93].

cookie 策略有一个普遍引起忧虑的话题，那就是如果一个坏指针应该被交给 delete_vec()，取出来的 cookie 自然是不合法的。一个不合法的元素数目和一个坏的起始地址，会导致 destructor 以非预期的次数被施行于一段非预期的区域。然而在 “联合数组” 的政策之下，坏指针的可能结果就只是取出错误的元素数目而已。

在原始编译器中，有两个主要函数用来储存和取出所谓的 cookie：
```cpp
// array_key是新数组的地址
// mustn't either be 0 or already entered
// elem_count is the count; it may be 0

typedef void* pv;
extern int __insert_new_array(PV array_key, ine elem_count);

// 从表格中取出（并去除）array_key
// 若不是传回elem_count，就是传回-1
extern int __remove_old_array(PV array_key);
```
下面是 cfront 中的 vec_new() 原始内容经过修润后的一份呈现，并附加批注：
```cpp
PV __vec_new(PV ptr_array, int elem_count,
            int size, PV construct)
{
    // 如果ptr_array是0，从heap之中配置数组
    // 如果ptr_array不是0，表示程序员写的是：
    // T array[count]
    // 或
    // new(ptr_array) T[0];

    int alloc = 0;  // 我们要在vec_new中配置吗
    int array_sz = elem_count * size;

    if(alloc = ptr_array == 0)
        // 全局运算符new
        ptr_array = PV(new char[array_sz]);
    // 在exception handling之下，
    // 将丢出exception bad_alloc
    if(ptr_array == 0)
        return 0;
    
    // 把数组元素数目放到cache中
    int status = __insert_new_array(ptr_array, elem_count);
    if(status == -1){
        // 在exception handling 之下将丢出exception
        // 将丢出exception bad_alloc
        if(alloc)
            delete ptr_array;
        return 0;
    }

    if(construct){
        register char* elem = (char*)ptr_array;
        register char* lim = elem + array_sz;
        // PF是一个typedef，代表一个函数指针
        register PF fp = PF(constructor);
        while(elem < lim){
            // 通过fp调用constructor作用于
            // “this”元素上（由elem指出）
            (*fp)((void*)elem);
            // 前进到下一个元素
            elem += size;
        }
    }
    return PV(ptr_array);
}
```
vec_delete() 的操作差不多，但其行为并不总是 C++ 程序员所预期或需求

例如，已知下面两个 class 声明：
```cpp
class Point{
public:
    Point();
    virtual ~Point();
    // ...
};

class Point3d : public Point{
public:
    Point3d();
    virtual ~Point3d();
    // ...
};
```
如果我们配置一个数组，内带 10 个 Point3d objects，我们会预期 Point 和 Point3d 的 constructor 被调用各 10 次，每次作用于数组中的一个元素：
```cpp
// 完全不是个好主意
Point *ptr = new Point3d[10];
```
而当我们 delete “由 ptr 所指向的 10 个 Point3d 元素” 时，会发生什么事情呢？很明显，我们需要虚拟机制的帮助，以获得预期的 Point destructor 和 Point3d destructor 各 10 次的呼唤（每一次作用于数组中的一个元素）：
```cpp
// 喔欧：这并不是我们所要的
// 只有Point::~Point被调用······
delete[] ptr;
```
施行于数组上的 destructor，如我们所见，是根据交给 vec_delete() 函数之 “被删除的指针类型的 destructor”——本例中正是 Point destructor。这很明显并非我们所希望。此外，每一个元素的大小也一并被传递过去。这就是 vec_delete() 如何迭代走过每一个数组元素的方式。本例中被传递过去的是 Point class object 的大小而不是 Point3d class object 的大小。整个运行过程非常不幸的失败了，不只是因为执行期错误的 destructor，而且自从第一个元素之后，该 destructor 即被施行于不正确的内存区块中（译注：因为元素的大小不对）。

程序员应该怎样做才好？最好就是避免以一个 base class 指针指向一个 derived class objects 所组成的数组——如果 derived class object 比其 base 大的话（译注：通常如此）。如果你真的一定要这样子写程序，解决之道在于程序员层面，而非语言层面：
```cpp
for(int ix = 0; ix < elem_count; ++ix)
{
    Point3d *p = &((Point3d*)ptr)[ix];      // 译注：原书为Point *p = ..
    delete p;                               // 恐为笔误
}                           
```
基本上，程序员必须迭代走过整个数组，把 delete 运算符实施与每一个元素身上。以此方式，调用操作将是 virtual，因此，Point3d 和 Point 的 destructor 都会施行于数组的每一个 objects 身上。

### Place Operator new 的语意

> @zl : C++ primer中的定位new。

有一个预先定义好的重载的（overloaded）new 运算符，称为 place operator new。它需要第二个参数，类型为 void*。调用方式如下：
```cpp
Point2w = *ptw = new(arena) Point2w;
```
其中 arena 指向内存中的一个区块，用以放置新产生出来的 Point2w object。这个预先定义好的 place operator new 的实现方法兼职是出乎意料的平凡，它只要将“获得的指针（译注：上面的arena）”所指的地址传回即可：
```cpp
void *
operator new(size_t, void* p)
{
    return p;
```
如果他的作用只是传回第二个参数，那么它有什么价值呢？也就是说，为什么不简单地这么写算了（这不就是实际所发生的操作吗）：
```cpp
Point2w *ptw = (Point2w*)arena;
```
唔，事实上这只是所发生的操作的一半而已。另外一半无法有程序员产生出来。想想这些问题：
1. 什么是使 placement new operator 能够有效的另一半扩充（而且是 “arena 的明确指定操作（explicit assignment）” 所没有提供的）？
2. 什么是 arena 指针的真正类型？该类型暗示了什么？

Placement new operator 所扩充的另一半边将是 Point2w constructor 自动实施于 arena 所指的地址上：
```cpp
// C++伪码
Point2w *ptw = (Point2w*)arena;
if(ptw != 0)
    ptw->Point2w::Point2w();
```
这正是使 placement operator new 威力如此强大的原因。这一份码决定objects 被放置在哪里；编译系统保证 object 的 constructor 会施行于其上。

然而却有一个轻微的不良行为。你看得出来吗？下面是个有问题的程序片段：
```cpp
// 让arena成为全局性定义
void foobar(){
    Point2w *p2w = new(arena) Point2w;
    // ... do it ...
    // ... now manipulate a new object ...
    p2w = new(arena)Point2w
}
```
如果 placement operator 在原已存在的一个 object 上构造新的 object，而该现有的 object 有一个 destructor，这个 destructor 并不会被调用。调用该 destructor 的方法之一是将那个指针 delete 掉。不过在此例中如果你像下面这样做，绝对是个错误：
```cpp
// 以下并不是实施destructor的正确方法
delete p2w;
p2w = new(arena)Point2w;
```
是的，delete 运算符会发生作用，这的确是我们所期待的。但是它也会释放由 p2w 所指的内存，这却不是我们所希望的，因为下一个指令就要用到 p2w 了。因此，我们应该明确地调用 destructor 并保留储存空间，以便再使用[1]。

[1] Standard C++ 以一个 placement operator delete 矫正了这个错误，它会对 object 实施 destructor，但不释放内存，所以就不必再直接调用 destructor 了。

```cpp
// 施行destructor的正确方法
p2w->~Point2w;
p2w = new(arena) Point2w;
```
剩下的唯一问题是一个设计上的问题：在我们的例子中对 placement operator 的第一次调用，会将新 object 构造于原已存在的 object 之上？还是会构造于全新地址上？也就是说，如果我们这样写：
```cpp
Point2w *p2w = new(arena) Point2w;
```
我们如何知道 arena 所指的这块区域是否需要先解构？这个问题在语言层面上并没有解答。一个合理的习俗是令执行 new 的这一端也要负责执行 destructor 的责任。

另一个问题关系到 arena 所表现的真正指针类型。C++ Standard 说它必须指向相同类型的 class，要不就是一块 “新鲜” 内存，足够容纳该类型的 object。注意，derived class 很明显并不在被支持之列。对于一个 derived class，或是其他没有关联的类型，其行为虽然并非不合法，却也未经定义。

“新鲜” 的储存空间可以这样配置而来：
```cpp
char *arena = new char[sizeof(Point2w)];
```
相同类型的 object 则可以这样获得：
```cpp
Point2w *arena = new Point2w;
```
不论是哪一种情况，新的 Point2w 的储存空间的确是覆盖了 arena 的位置，而此行为已在良好控制之下。然而，一般而言，placement new operator 并不支持多态（polymorphism）。被交给 new 的指针，应该适当地指定一块预先配置好的内存。如果 derived class 比其 base class 大，例如：
```cpp
Point2w *p2w = new(arena)Point3w;
```
Point3w 的 cosntructor 将会导致严重的破坏。

Placement new operator 被引入 C++ 2.0 时，最晦涩隐暗的问题就是下面这个由 Jonathan Shopiro 提出的问题：
```cpp
struct Base{ int j; virtual void f();};
struct Derived : Base { void f();};
void fooBar(){
    Base b;
    b.f();              // Base::f()被调用
    b.~Base();
    new(&b) Derived;    // 1
    b.f();              // 哪一个f()被调用
}
```
由于上述两个 classes 有相同的大小，故把 derived object 放在为 base class 而配置的内存中是安全的。然而，要支持这一点，或许必须放弃对于“经由 objects 静态调用所有 virtual functions（比如 b.f()）” 通常都会有的优化处理。如果，placement new operator 的这种使用方式在 Standard C++ 中未能获得支持（请看 C++ Standard 3.8 节）。于是上述程序的行为没有明确定义：我们不能够斩钉截铁地说哪一个 f() 函数实体会被调用。尽管大部分使用者可能以为调用的是 Derived::f()，但大部分编译器调用却是 Base::f()。

## 6.3 临时性对象（Temporary Objects）

如果我们有一个函数，形式如下：
```cpp
T operator+(const T&, const T&);
```
以及两个 T objects，a 和 b，那么：
```cpp
a + b;
```
可能会导致一个临时性对象，以放置传回的对象。是否会导致一个临时性对象，视编译器的进取型（aggressiveness）以及上述操作发生时的程序上下关系（program context）而定。例如下面这个片段：
```cpp
T a, b;
T c = a + b;
```
编译器会产生一个临时性对象，放置 a + b 的结果，然后再使用 T 的 copy constructor，把该临时性对象当做 c 的初始值。然而比较更可能的转换是直接以拷贝构造的方式，将 a+b 的值放到 c 中（2.3 节对于加法运算符的转换曾有讨论），于是就不需要临时性对象，以及对其 constructor 和 destructor 的调用了。

此外，视 operator+() 的定义而定，named return value(NRV) 优化（请看 2.3 节）也可能实施起来。这将导致直接在上述 c 对象中求表达式结果，避免执行 copy constructor 和具名对象（named object）的 destructor。

三种方式所获得的 c 对象，结果都一样。其间的差异在于初始化的成本。一个编译器可能给我们任何保证码？严格地说没有。C++ Standard 允许编译器对于临时性对象的产生有完全的自由度：
```
在某些环境下，由 processor 产生临时性对象是有必要的，亦或是比较方便的。这样的临时性对象由编译器来定义。（C++ Standard，12.2 节）
```
理论上，C++ Standard 允许编译器厂商有完全的自由度。但实际上，由于市场的竞争，几乎保证任何表达式（expression）如果有这种形式：
```cpp
T c = a + b;
```
而其中的加法运算符被定义为：
```cpp
T operator+(const T&, const T&);
```
或
```cpp
T T::operator+(const T&)
```
那么实现时根本不产生一个临时性对象。

然而请你注意，意义相当的 assignment 叙述句（statement）：
```cpp
c = a + b;
```
不能够忽略临时性对象。相反，它会导致下面的结果：
```cpp
// C++伪码
// T temp = a + b
T temp;
temp.operator+(a, b);   // (1) 译注：原书为a.operator+(temp, b);误！
// c = temp
c.operator=（temp);     // (2)
temp.T::~T();
```
标示为 (1) 的那一行，未构造的临时对象被赋值给 operator+()。这意思是要不是 “表达式的结果被 copy constructed 至临时对象中”，就是 “以临时对象取到 NRV”。在后者中，原本要施行于 NRV 的 constructor，现在将施行于该临时对象中。

不管哪一种情况，直接传递 c（上例赋值操作的目标对象）到运算符函数中是有问题的。由于运算符函数并不为其外加参数调用一个 destructor（它期望一块 “新鲜的” 内存），所以必须再次调用之前先调用 destructor。然而，“转换” 语意将被用来将下面的 assignment 操作：
```cpp
c = a + b;  // c.operator=(a, b)
```
取代为其 copy assginment 运算符的隐含调用操作，以及一系列的 destructor 和 copy construction：
```cpp
// C++伪码
c.T::~T();
c.T::T(a + b);
```
copy constructor、destructor 以及 copy assignment operator 都可以由使用者供应，所以不能够保证上述两个操作会导致相同的语意。因此，以一连串的 destructor 和 copy constructor 来取代 assignment，一般而言是不安全的，而且会产生临时对象。所以这样的初始化操作：
```cpp
T c = a + b;
```
总是比下面的操作更有效率地被编译器转换：
```cpp
c = a + b;
```
第三种运算形式是，没有出现目标对象：
```cpp
a + b;  // no target
```
这时候有必要产生一个临时对象，以防止运算后的结果。虽然看起来有点怪异，但这种情况实际上在子表达式（subexpressions）中十分普遍，例如，如果我们这样写：
```cpp
String s("Hello"), t("Word"), u("!");
```
那个不论：
```cpp
String v;
v = s + t + u;
```
或
```cpp
printf("%s\n", s + t);
```
都会产生一个临时对象，与 s+t 相关联。

最后一个表达式带来一些秘教式的论题，那就是 “临时对象的生命期”。这个论题颇值得深入探讨。在 Standard C++ 之前，临时对象的生命（也就是说它的 destructor 何时实施）并没有明确指定，而是由编译厂商自行决定。换句话说，上述的 printf() 并不保证安全，因为它的正确性与 s + t 何时被摧毁有关。

（本例的一个可能性是，String class 定义了一个 conversion 运算符如下：
`String::operator const char*() { return _str; }`
其中 _str 是一个 private member addressing storage，在 String object 构造是配置，在其 destructor 中被释放）。

因此，如果临时对象在调用 printf() 之前就被解构了，经由 conversion 运算符交给它的地址就是不合法的。真正的结果视底部的 delete 运算符在释放内存时的进取性而定。某些编译器可能会把这块内存标示为 free，不以人任何方式改变其内容。在这块内存被其他地方宣称主权之前，只要它还没有被 delete 掉，它就可以被使用。虽然对于软件工程而言这不足以作为模范，但像这样在内存被释放之后又再被使用，并非罕见。事实上 malloc() 的许多编译器会提供一个特殊的调用操作：
```cpp
malloc(0);
```
它是用来保证上述行为的。

例如，下面是对于该算式的一个可能的 pre-Standard 转换。虽然在 pre-Standard 语言定义中是合法的，但却可能造成重大灾难：
```cpp
// C++伪码：pre-Standard的合法转换
// 临时性队对象被摧毁的太快（太早）了

String temp1 = operator+(s, t);
const char *temp2 = temp1.operator const char*();

// 喔欧：合法但是有欠考虑，太多轻率
temp1.~String();

// 这时候并未定义temp2指向何方
printf("%s\n", temp2);
```
另一种（比较被喜欢的转换方式是在调用 printf() 之后实施 String destructor）。在 C++ Standard 之下，这正是该表示式的必须转换方式。标准规格上这么说：
```
临时性对象的被摧毁，应该是对完整表达式（full-expression）求值过程中的最后一个步骤。该完整表达式造成临时对象的产生（Section 12.2）。
```
什么事一个完成表达式（full-expression）？正费事地说，它是被涵括的表达式中最外围的那个。下面这个式子：
```cpp
// terriary full expression with 5 sub-expressions
((objA > 1024) && (objB > 1024))
    ? objA + objB : foo(objA, objB);
```
一共有五个子算式（subexpressions），内带一个 “？：完整表达式” 中。任何一个子表达式所产生的任何一个临时对象，都应该在完整表达式被求值完成后，才可以毁去。

当临时性对象是根据程序的执行期语意有条件地被产生出来时，临时性对象的生命规则就显得有些复杂了。举个例子，像这样的表达式：
```cpp
if( s + t || u + v)
```
其中的 u + v 自算式只有在 s + t 别评估为 false 时，才会开始被评估。与第二个子算式有关的临时对象必须被摧毁。但是，很明显地，不可以被无条件地摧毁。也就是说，我们希望只有在临时性独享被产生出来的情况下才去摧毁它。

在讨论临时对象的声明规则之前，标准编译器将临时对象的构造和解构附着于第二个子算式的评估程序中。例如，对于以下的 class 声明：
```cpp
class X{
public:
    X();
    ~X();
    operator int();
    X foo();
private:
    int val;
};
```
以及对于 class X 的两个 objects 的条件测试：
```cpp
main(){
    X xx;
    Y yy;
    if(xx.foo() || yy.foo())
    ;
    return 0;
}
=
```
cfront 对于 main() 产生出以下的转换结果（已经过轻微的修润和注释）：
```cpp
int main(void){
    struct X __1xx;
    struct X __1yy;

    int __0_result;

    // name_mangled default constructor
    // X:X( X *this)
    __ct__1xFv(&__1xx);
    __ct__1xFv(&__1yy);
    {
        // 被产生出来的临时性对象
        struct X __0__Q1;
        struct X __0__Q2;
        int __0__Q3;

        /* 每一端变成一个附逗点的表达式，
         * 有着下列次序：
         * 
         * tempQ1 = xx.foo();
         * tempQ3 = tempQ1.operator int();
         * tempQ1.X::~X();
         * tempQ3;
         */
        
        // __opi__1xFv ==> X::operator int()
        if(((
            __0_Q3 = __opi__1xFv(((
            __0_Q1 = foo_1xFv(&__1xx)), (&__0__Q1)))),
            __dt__1xFv(&__0__Q1, 2)), __0__Q3)
        ||| (((
            __0_Q3 = __opi__1xFv(((
            __0_Q1 = foo_1xFv(&__1xx)), (&__0__Q2)))),
            __dt__1xFv(&__0__Q2, 2)), __0__Q3))
        {
            __0_result = 0;
            __dt_1xFv(&__1yy, 2);
            __dt_1xFv(&__1xx, 2);
        }
        return __0_result;
    }
}
```
把临时性对象的 destructor 放在每一个子算式的求值过程中，可以免除 “努力跟踪第二个子算式是否真的需要被评估”。然而在 C++ Standard 的临时对象生命规则中，这样的策略不再被允许。临时性对象在完整表达式尚未评估完全之前，不得被摧毁。也就是说，某些形式的条件测试现在必须被安插进来，以决定是否要摧毁和第二算式有关的临时对象。

临时性对象的声明规则有两个例外。第一个例外发生在表达式被用来初始化一个 object 时。例如：
```cpp
bool verbose;
...
String progNameVersion = 
    !verbose
        ? 0
        : progName + progVersion;
```
其中 progName 和 progVersion 都是 String objects。这时候会产生出一个临时对象，放置加法运算符的运算结果：
```cpp
String operator+(const String&, const String&);
```
临时对象必须根据 verbose 的测试结果有条件地解构。在临时对象的声明规则之下，它应该在完整的 “?:表达式” 结果评估之后尽快地摧毁。然而，如果 progNameVersion 的初始化需要调用一个 copy constructor：
```cpp
// C++伪码
progNameVersion.String::String(temp);
```
那么临时性独享的解构（在 “？：完整表达式” 之后）当然就不是我们所期望的。C++ Standard 要求说：

> ······凡含有表达式执行结果的临时性对象，应该存留到 object 的初始化操作完成为止。

甚至即使每一个人都坚守 C++ Standard 中的临时对象生命规则，程序员还是由可能让一个临时对象在他们的控制中被摧毁。其间的主要差异在于这时候的行为有明确的定义。例如，在新的临时对象生命规则中，下面这个初始化操作保证失败：
```cpp
// 喔欧：不是个好主意
const char *progNameVersion = 
    progName + progVersion
```
其中 progName 和 progVersion 都是 String objects。产生出来的程序代码看起来像这样：
```cpp
// C++ pseudo Code
String temp;
operator+(temp, progName, progVersion);
progNameVersion = temp.String::operator char*();
temp.String::~String();
```
此刻 progNameVersion 指向未定义的 heap 内存！

临时性对象的声明规则的第二个例外是 “当一个临时性对象被一个 reference 绑定” 时，例如：
```cpp
const String &space = "";
```
产生出这样的程序代码：
```cpp
// C++ pseudo Code
String temp;
temp.String::String(" ");
const String &space = temp;
```
很明显，如果临时性对象现在被摧毁，那个 reference 也就差不多没什么用了。所以规则上说：

> 如果一个临时性对象被绑定于一个 reference，对象将残留，直到被初始化之 reference 的声明结束，或直到临时对象的声明范畴（scope）结束——视哪一种情况先到达而定。


### 临时性对象的迷思（深化、传说）

有一种说法是，由于当前的 C++ 编译器会产生临时性对象，导致程序的执行比较没有效率，因此在工程界或科学界，C++ 只能成为 FORTRAN 以外可怜的第二选择。更有人认为，这种效率上的不彰足以掩盖 C++ 在 “抽象化” 上的贡献（例如 [BUDGE92]）。相反的论点则请参考 [NACK94])。发表于 The journal of C Language 上的 [BUDGE94] 对此有过一份有趣的研究。

在 FORTRAN-77 和 C++ 的一场比较之中，Kent Budge 和其助手分别以两种语言写了一个复数测试程序（在 FORTRAN 中复数是内建类型，在 C++ 中它是一个具体类，有两个 members，一个代表实数，一个代表虚数。Standard C++ 已经把复数类放在标准链接库中）。C++ 程序实现 inline 运算符如下：
```cpp
friend complex operator+(complex, complex);
```
请注意，被传递的 class objects（内带有 constructors 和 destructors）是以 by value 而非 by refernce 的方式传递，像这样：
```cpp
friend complex operator+(const complex&, const complex&);
```
一般而言并非是一种好的 C++ 程序风格。除了 “copy by value possibly large class objects” 这个主题之外，每一个正式参数的局部实体都被 copy constructed 和 destructed，并且可能导致临时对象的诞生。在这个测试例程中，作者生成把正式参数转换为 const reference 并不会明显改变效率。这是因为每一个函数是 inline 之故。

测试程序看起来像这样：
```cpp
void func(complex *a, const complex *b,
        const complex *c, int N)
{
    for(int i = 0; i < N; i++)
        a[i] = b[i] + c[i] - b[i] * c[i];
}
```
其中对于复数的加法、减法、乘法和 assignment 运算符，都是 inline 函数。C++ 码产生出五个临时对象：
1. 一个临时对象，用来放置 b[i] + c[i]。
2. 一个临时对象，用来放置 b[i] * c[i].
3. 一个临时对象，用来放置上述两个临时对象的相减结果。
4. 两个临时对象，分别用来放置上述第一个临时对象和第二个临时对象，为的是完成第三个临时对象。

测试结果发现，FORTRAN-77 的码果然快达两倍。他们的第一个假设是把责任归咎于测试对象。为了验证，他们以手工方式把 cfront 中介输出码中的而所有临时对象一一消除。一如预期，效率增加了两倍，几乎和 FORTRAN-77 相当。

测试并没有就此停下来，Budge 和其助手以另一个方法来实验。这一次他们采用反聚合（disaggregated）手法。也就是说，他们把那些临时对象拆开为一对一对的临时性 double 变量。结果发现，这种做法对于效率的提升，和先前 “消除所有临时对象” 一样。于是他们写下这样的心得：

> 我们所测试的编译系统很明显能够消除内建（build-in）类型的局部变量，但对于 class 类型的局部变量就行不通。这是 C++ back-ends（而不是 C++ front-end）的限制。这似乎是很普遍的情况，因为在 Sun CC、GNU g++  以及 HP CC 等编译器中都会发生。[BUDGE94]

这篇文章分析了被产生出来的 assembly 码，并表示效率降低的原因是由于程序中存在大量的堆栈存取操作（以读写个别的 class members）。经过反聚合（disaggregated），并将个别的 members 放到缓存器中，就能够达到几乎两倍的效率。这使他们写下了这样的讨论：

> 加上适度的努力，反聚合（disaggregation）大有可为。但是一般的C++编译器并没有把它视为一个重要的优化关键。

这份研究对于良好的优化提供了一个具有说服力的证明。目前存在有一些优化工具，的确把临时对象的一些成分放进了缓存器中。当编译器厂商把他们的焦点从语言特性的支持（与 Standard C++ 比较）转移到实现技术的优劣上时，如反聚合（disaggregation）这般的优化操作就会更普遍了。

[第 5 章 构造、解构、拷贝 语意学（Semantics of Constructor, Destructor, and Copy）](Semantics_of_Constructor_.md)|[第 7 章 站在对象模型的尖端（On the Cusp of the Object Model）](On_the_Cusp_of_.md)
