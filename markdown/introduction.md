# 第0章 导读（译者的话）

## 合适的读者

很不容易三言两语就说明此书的适当读者。作者 Lippman 参与设计了全世界第一套 C++ 编译器 cfront，这本书就是一位伟大的 C++ 编译器设计者在向你阐述他如何处理各种 explicit（明白出现于 C++ 程序代码）和 implicit（隐藏于程序代码背后）的 C++ 语意。

对于 C++ 程序老手，这必然是一本让你大呼过瘾的绝妙好书。

C++ 老手分两类。意中人把语言用的熟烂，OO 观念也有。另一种人不但如此，还对于台面下的机制，如编译器合成的 default constructor 啦、object 的内存布局啦等等有莫大的兴趣。本书对于第二类老手的吸引器自不待言，至于第一类老手，或许你没那么大的刨根究底的兴趣，不过我还是极力推荐你阅读此书。了解 C++ 对象模型，绝对有助于你在语言本身以及面向对象观念两方面的层次提升。

你需要喜喜推敲每一个句子，每一个例子，囫囵吞枣是完全没有用的。作者是 C++ 大师级人物，并且参与开发了第一套 C++ 编译器，他的解说以及诠释确实是鞭辟入里，你无比在看过每一小段之后，融会贯通，把它的思想观念化为己有，再接续另一小节。但阅读次序并不需要按照书中的章节排列。

## 阅读次序

我个人认为，第 1，3，4 章最能带给读者迅速而且最大的帮助，这些都是经常引起程序员困惑的主题。作者在这些章节中有不少示意图（我自己也加了不少）。你或许可以从这三张挑着看起。

其他章节比较晦涩一些（我的感觉），不妨“视可而择之”。

当然，这都是十分主观的认定。客观的意见只有一个：你可以随你的兴趣与需求，从任一章看是看起，各章之间没有必然关联性。

## 翻译风格

太多朋友告诉我，他们阅读中文计算机书籍，不论是著作或译作，最大的阅读困难子啊与一大堆没有标准译名的技术名词或习惯用语（至于那些误谬不知所云的奇怪作品当然本就不在考虑之列）。其实，就算国家相关机构有统一译名（或曾有过，谁知道？），流通于工业界与学术界之间的还是原文名词与术语。

对于工程师，我希望我缩写的书和我所译的书能够让各位读来通体顺畅；对于学生，我还是希望多发挥一些引导的力量，引导各位多使用、多认识原文术语和专有名词，不要说出像“无模式对话盒”这种奇怪的话。

由于本书读者定位之故，我决定保留大量的原文技术名词与术语。我清楚地知道，在我们的技术领域里，研究人员或工程师如何使用这些词汇。

当然，有些中文译名较普及，也较贴切，我并不排除使用。其间的挑选与决定，不可避免地带了点个人色彩。

下面是本书的原文名词（按字母排序）及其意义：

|英文名词|原文名词（或）及其含义|
|-|-|
|access level|访问级。就是 C++ 的 public、private、protected 三种等级|
|access section|访问区段。就是 class 中的 public、private、protected 三种段落|
|alignment|边界调整，调整至某些 types 的倍数。其结果视不同的机器而定。例如 32 位机器通常调整至 4 的倍数|
|bind|绑定，将程序中的某个符号真正附着（决议）至一块实体上|
|chain|串链|
|class|类|
|class hierarchy|class 体系，class 层次结构|
|compositon|组合。通常与继承（inheritance）一同讨论|
|concrete inheritance|具体继承（相对于抽象继承）|
|constructor|构造函数|
|data member|数据成员（亦或被称为 member variable）|
|declaration, declare|声明|
|definition, define|定义|
|derived|派生|
|encapsulation|封装|
|explicit|明确的（通常指 C++ 代码中明确出现的）|
|hierarchy|体系，层次结构|
|implement|实现（动词）|
|implementation|实现品、实现物。本书有时候指 C++ 编译器。大部分时候是指 class member function 的内容|
|implicit|隐含的、暗喻的（通常指未出现在 C++ 程序代码中的）|
|inheritance|继承|
|inline|内联（C++ 的一个关键词）|
|instance|实体（有些书籍译为“案例”或实例，极不妥当）|
|layout|布局。本书常常出现这个字，意指 object 在内存中的数据分布情况|
|mangle|名称切割重组（C++ 对于函数名称的一种处理方式）|
|member function|成员函数，亦或被称为 function member|
|members|成员，泛指 data members 和 member functions|
|object|对象（根据 class 的声明而完成的一份占有内存的实体）|
|offset|偏移位置|
|operand|操作数|
|operator|运算符|
|overhand|额外负担（因某种设计，而导致的额外成本）|
|overload|重载|
|overloaded function|重载函数|
|override|改写（对 virtual function 的重新设计）|
|paradigm|典范（请参考 #22 页）|
|pointer|指针|
|polymorphism|多态（“面向对象”最重要的一个性质）|
|programming|程序设计、程序化|
|reference|参考、参用（动词）|
|resolve|决议。函数调用时链接器所进行的一种操作，将符号与函数实体产生关联。如果你调用 func() 而链接时找不到 func() 实体，就会出现“unresolved externals”链接错误|
|slot|表格中的一格（一个元素）；条孔；条目；条格|
|subtype|子类型|
|type|类型，类别（指的是 int、float 等内建类型，或 C++ classed 等自定类型）|
|virtual|虚拟|
|virtual function|虚拟函数|
|virtual inheritance|虚拟继承|
|virtual table|虚拟表格（为实现虚拟机制而设计的一种表格，内放 virtual functions 的地址）|

有时候考虑到上下文的因素，面对同一个名词，在译与不译之间，我可能会有不同的选择。例如，面对“ pointer ”，我会译为“指针”,但由于我并未将 reference 译为“参考”（实在不对味），所以如果原文是 “the manipulation of a pointer or reference in C++ ······”，为了中英对等或平衡的缘故，我不会把它译为 “C++ 中对于指针和 reference 的操作行为······”，我会译为 “C++ 中对于 pointer 和 reference 的操作行为······”。

## 译注

书中有一些译注。大部分译注，如果够短的话，会被我直接放在括号之中，接续本文。较长的译注，则被我安排在被注文字的段落下面（紧临，并加标示）。

## 原书错误

这本书虽说质地极佳，制作的严谨度却不及格！有损 Lippman 的大师地位。

属于“作者笔误”之类的错误，比较无伤大雅，例如少了一个 ； 符号，或是多了一个 ； 符号，或是少了一个 } 符号，或是多了一个 ） 符号等等。比较严重的错误，是程序变量名称或函数名称或 class 名称与文字叙述不一致，甚或是图片中对于 object 布局的画法，与程序代码中的声明不一致。这两种错误都会严重损费读者的心神。

只要是被我发现的错误，都已被我修正。一下是错误更正列表：

实例：L5 表示第 5 行，L-9 表示倒数第 9 行。页码所示为原书页码。
|页码|原文位置|原文内容|应修改为|
|-|-|-|-|
|p.35|最后一行|Bashful(),|Bashful();|
|p.57|表格第二行|1.32.26|1：32.36|
|p.61|L1|memcpy...程序代码最后少了一个)||
|p.61|L10|Shape()...程序代码最后少了一个}||
|p.64|L-9|程序代码最后多了一个;||
|p.78|最后四行码|==|似乎应为=|
|p.84|图3.1b说明|struct Point3d|class Point3d|
|p.87|L-2|virtual...程序代码最后少了一个;||
|p.87|全页多处|pc2_2（不符合命名意义）|pc1_2（符合命名意义）|
|p.90|图3.2a说明|Vptr placement and end of class|Vptr placement of class|
|p.91|图3.2(b)|__vptr__has_vrts|__vptr__has_virts|
|p.92|码L-7|class Vertex2d|class Vertex3d|
|p.92|码L-6|public Point2d|public Point3d|
|p.93|图3.4说明|Vertex2d的对象布局|Vertex3d的对象布局|
|p.92-p.94||符号名称混用，前后误谬不符|已全部更改过|
|p.97|码L2|public Point3d, public Vertex|配合图3.5ab，应调整次序为public Vertex, public Point3d|
|p.99|图3.5(a)|符号与书中程序代码多处不符|已全部更改过|
|p.100|图3.5(b)|符号与书中程序代码多处不符|已全部更改过|
|p.100|L-2|?pv3d + ...最后多了一个)||
|p.106|L16|ptld::y|pt2d::_y|
|p.107|L10|&3d_point:: z|&Point3d:: z|
|p.108|L6|&3d_point:: z|&Point3d:: z|
|p.108|L-6|int d::\*dmp, d\*pd|int Derived::*dmp, Derived *pd|
|p.109|L1|d *pd|Derived *pd|
|p.109|L4|int b2::*bmp=&b2::val2;|int Base2::*bmp = &Base2::val2;|
|p.110|L2|不符合稍早出现的程序代码|把pt3d改为Point3d|
|p.115|L1|magnitude()|magnitude3d()|
|p.126|L12|Point2d pt2d = new Point2d;|ptr = new Point2d;|
|p.136|图4.2右下|Derived::~close()|Derived::close()|
|p.138|L-12|class Point3d...最后少一个{||
|p.140|程序代码|没有与文字中的class命名一致|所有的pt3d改为Point3d|
|p.142|L-7|if(this...程序代码最右边少一个)||
|p.143|程序代码|没有与文字中的class命名一致|所有的pt3d改为Point3d|
|p.145|L-6|pointer:: z()|Point:: z()|
|p.147|L1|pointer::\*pmf|Point::\*pmf|
|p.147|L5|point::x()|Point::x()|
|p.147|L6|point:: z()|Point:: z()|
|p.147|中段码L-1|程序代码最后缺少一个)||
|p.148|中段码L1|(ptr->*pmf)函数最后少一个;||
|p.148|中段码L-1|(*ptr->vptr[...函数最后少一个)||
|p.150|L-7|pA.__vptr__pt3d...最后少一个;||
|p.152|L4|point new_pt;|Point new_pt|
|p.156|L7|{|}|
|p.160|L11, L12|Abstract_Base|Abstract_base|
|p.162|L-3|Abstract_base函数最后少一个;||
|p.166|中，码L3|Point1 local1 = ...|Point local2 = ...|
|p.166|中，码L4|Point2 local2;|Point local2;|
|p.174|中，码L-1|Line::Line()函数最后多了一个;||
|p.174|中下，码L-1|Line()::Line()函数最后多了一个;||
|p.175|中上，码L-1|Line()::~Line()函数最后多了一个;||
|p.182|中下，码L6|Point3d::Point3d()|PVertex::PVertex()|
|p.183|上，码L9|Point3d::Point3d()|PVertex::PVertex()|
|p.185|上，码L3|y = 0.0 之前缺少 float||
|p.186|中下，码L6|缺少一个 return||
|p.187|中，码L3|const Point3d &p|const Point3d &p3d|
|p.204|下，码L3|缺少一个 return 1;||
|p.208|中下，码L2|new Pvertex;|new PVertex;|
|p.219|上，码L1|__nw(5 * sizeof(int));|__new(5 * sizeof(int));|
|p.222|上，码18|//new(ptr_array...程序缺少了一个;||
|p.224|中，码L1|Point2w ptr = ...|Point2d *ptw = ....|
|p.224|下，码L5|operator new()|函数定义多一个;|
|p.225|上，码L2|Point2w ptw = ...|Point2w *pt2 = ...|
|p.226|下，码L1|Point2w p2w = ...|Point2w *p2w = ...|
|p.229|中，码L1|c.operator==(a + b);|c.operator=(a + b);|
|p.232|中下，码L2|x xx;|X xx;|
|p.232|中下，码L3|x yy;|X yy;|
|p.232|下，码L2|struct x_1xx;|struct X_1xx;|
|p.232|下，码L3|struct x_1yy;|struct X_1yy;|
|p.233|码L2|struct x__0__Q1;|struct X__0__Q1;|
|p.233|码L3|struct x__0__Q2;|struct X__0__Q2;|
|p.233|中，码|if 条件句的最后多了一个;||
|p.253|码L-1|foo() 函数最后多了一个;||

## 推荐

我个人翻译过不少书籍，每一本都精挑细选后才动手（品质不厚的原文书，译它做啥？！）。在这么些译文中，我从来不做直接而露骨的推荐。好的书籍自然而然会得到识者的欣赏。过去我译的那些明显具有实用价值的书籍，总有相当数量的读者有强烈的需求，所以我从不担心没有足够的人来为好书散播口碑。但 Lippman 的这本书不一样，它可能不会为你带来明显而立即的实用性，它可能因此在书肆中蒙上一层灰（其原文书我就没听说过多少人读过），枉费我从众多原文书中挑出这本好书。我担心听到这样的话：

对象模型？呵，我会写 C++ 程序，写的一级棒，这些属于编译器层面的东西，于我何有哉！

对象模型是深层结构的只是，关系到“与语言无关、与平台无关、跨网络可执行”软件组件（software component）的基础原理。也因此，了解 C++ 对象模型，是学习目前软件组件三大规格（COM、CORBA、SOM）的基础。

如果你对软件组织（software component）没有兴趣，C++ 对象模型也能够使你对虚拟函数、虚拟继承、虚拟接口有脱胎换骨的新认识，或是对于各种 C++ 写法所带来的效率利益有通盘的认识。

我因此要大声地说：有经验的 C++ programmer 都应该看看这本书。

如果您对 COM 有兴趣，我也要同时推荐你看另一本书：Essential COM, Don Box著，Addison Wesley 1998 出版（COM 本质论，侯捷译，碁峰 1998）。这也是一本论述非常清楚的书籍，把 COM 的由来（为什么需要 COM、如何使用 COM）以循序渐进的方式阐述得非常深刻，是我所看过最理想的一本 COM 基础书籍。


[第 \ 章 本立道生 前言](/markdown/preface.md)|[第 1 章 关于对象（Object Lessons）](/markdown/Object_Lessons.md)

