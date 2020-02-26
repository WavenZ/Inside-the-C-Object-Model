# 深度探索C++对象模型 Inside The C++ Object Model

Stanley B.Lippman 著

侯捷 译

华中科技大学出版社
Pearson Education（培生教育出版社）

# 本立道生（侯捷译序）

对于传统的结构化（sequential）语言，我们向来没有太多的疑惑，虽然在函数调用的背后，也有着堆栈建立、参数排列、返回地址、堆栈清除等等幕后机制。但函数调用时那么地自然而明显，好像只是夹带着一个包裹，从程序的第一个地点跳到另一个地点去执行。

但是对于面向对象（Object Oriented）语言，我们的疑惑就多了，究其因，这种语言的编译器为我们（程序员）做了太多的服务：构造函数、解构函数、虚拟函数、继承、多态......有时它为我们合成一些额外的函数（或运算符），有时候它又扩张我们所写的函数，放进更多的操作，有时候它还会为我们的 objects 添油加醋，放进一些奇妙的东西，使你面对 sizeof 的结果大惊失色。

存在我心里头一直有个疑惑：计算机程序最基础的形式，总是脱离不了一行行的循环执行模式，为什么 OO（面向对象）语言却能够“自动完成”这么多事情呢？另一个疑惑是：威力强大的 polymorphism（多态），其底层机制究竟如何？

如果不了解编译器对我们所写的 C++ 代码做了什么手脚，这些困惑永远解不开。

这本书解决了过去令我百思不解的诸多疑惑。我要向所有已具备 C++ 多年程序设计经验的同好们大力推荐这本书。

这本书同时也是越向组件软件（component-ware）基本精神的跳板。不管你想学习 COM（Component Object Model）或 CORBA（Common Object Request Broker Architecture），或是 SOM（System Object Model），了解 C++ ObjectModel，将使你更清楚软件组件（components）设计上的难点与应用之道，不但我自己在学习 COM 的道路上有此强烈的感受，Essential COM（COM 本质论，侯捷译，碁峰 1998）的作者 Don Box 也在他的书中推崇 Lippman 的这一本卓越的书籍。

是的，这当然不会是一本轻松的书籍。某些章节（例如 3，4 两章）可能给你立即的享受——享受面对底层机制有所体会与掌控的快乐；某些章节（例如 5，6，7 三章）可能带给你短暂的痛苦——痛苦于艰难深涩、难以吞咽的内容。这些快乐与痛苦，其实就是我翻译此书时的心情写照。无论如何，我希望透过我的译笔，把这本难得的好书带到更多人面前，引领大家见识C++底层构造的技术之美。

侯捷 2001.03.20 于新竹
jjhou@jjhou.com


请注意：本书特点，作者 Lippman 在其前言中有很详细的描述，我不再多言，翻译用词与心得，记录在第 0 章（译者的话）之中，对您或有导读之功。

请注意：原文本有大大小小约 80-90 个笔误，有的无伤大雅，有的对阅读顺畅影响甚巨（如前后文所用符号不一致，内文与图形所用符号不一致——甚至因而导致图片的文字解释不正确），我已在第 0 章（译者的话）列出我所找到的所有错误。此外，某些场合我还会再错误出现之处再加注，表示原文内容为何。这么做不是画蛇添足，也不为彰显什么，我知道有些读者会拿着原文书和中译文对照着看，我把原书加注出来，可免读者怀疑是否我打错字或是译错了。另一方面也是为了文责自负······唔······万一 Lippman 是对的而 LLHou 错了呢？！我虽有相当把握，还是希望明白摊开来让读者检验。


## 前言（Stanlcy B.Lippman）

差不多有 10 年之久，我在贝尔实验室（Bell Laboratories）埋首于 C++ 的实现任务，最初的工作是在 cfront 上面（Bjarne Stroustrup 的第一个 C++ 编译器），从 1986 年的 1.1 版到 1991 年 9 月的 3.0 版，然后转移到 Simlifier（这是我们内部的命名），也就是 Foundation 项目中的 C++ 对象模型部分。在 Simplifier 设计期间，我开始酝酿这本书。

Foundation 项目是什么？在 Bjarne 的领导下，贝尔实验室中的一个小组探索者以 C++ 完成大规模程序设计时的种种问题的解决之道。 Foundation 项目是我们为了构造大系统而努力定义的一种新的开发模型；我们只使用 C++ ，并不提供多种语言的解决方案。这是个令人兴奋的工作，一方面是因为工作本身，一方面是因为工作伙伴：Bjarne、Andy Koenig、Rob Murray、Martin Carroll、Judy Ward、Steve Buroff、Peter Juhl，以及我自己。 Barbara Moo 管理我们这一群人（Bjarne 和 Andy 除外）。Barbara Moo 常说管理一个软件团队，就像放牧一群骄傲的猫。

我们把 Foundation 想象成一个核心，在那上面，其他人可以为使用者铺设一层真正的开发环境，把他整修为他们所期望的 UNIX 或 Smalltalk 模型，私底下我们把它称为 Grail （传说中耶稣最后的晚餐所用的圣杯），人人都想要，但是从来没人找到过！

Grail 使用一个由 Rob Murray 发展出来并命名为 ALF 的面向对象层次结构，提供一个永久的、以语意为基础的表现法。在 Grail 中，传统编译器被分解为数个各自分离的可执行文件， parser 负责建立程序的 ALF 表现法。其它每一个组件（比如 type checking、simplification、code generation）以及工具（比如 browser）都在程序的一个ALF表现体上操作（并可能加以扩展）。Simplifier 是编译器的一部分，处于 type checking 和 code generation 之间。Simplifier 这个名称是由 Bjarne 所提倡的，它原本是 cfront 的一个阶段（phase）。

在 type checking 和 code generation 之间，Simplifier 做什么事情呢？它用来转换内部的程序表现。有三种转换风味是任何对象模型都需要的：

**1.与编译器息息相关的转换（Implementation-dependent transformations）**

这是与特定编译器有关的转换。在 ALF 之下，这意味着我们所谓的 “tentative” nodes。例如，当 parser 看到这个表达式：
```cpp
    fct();
```
它并不知道是否 (a) 这是一个函数调用操作，或者 (b) 这是 overloaded call operator 在 class object fct 上的一种应用。默认情况下，这个式子所代表的是一个函数调用，但是当 (b) 的情况出现时，Simplifier 就要重写并转换 call subtree 。

**2.语言语意转换（Language semantics transformations）**

这包括 constructor/destructor 的合成与扩展、memberwise 初始化、对于 menberwise copy 的支持、在程序代码中安插 conversion operators、临时性对象，以及对 constructor/destructor 的调用。

**3.程序代码和对象模型的转换（Code and object model transformations）**

这包括对 virtual functions、virtual base class 和 inheritance 的一般支持、 new 和 delete 运算符、class objects 所组成的数组、local static class instances、带有非常量表达式（nonconstant expression）之 global object 的静态初始化操作。我对 Simplifier 所规划的一个目标是：提供一个对象模型体系，在其中，对象的实现是一个虚拟接口，支持各种对象模型。

最后两种类型的转换构成了本书的基础。这意味着本书是为编译器设计者而写的吗？不是，绝对不是！这本书是由一位编译器设计者针对中高级 C++ 程序员所写的。隐藏在本书背后的假设是，程序员如果了解 C++ 对象模型，就可以写出比较没有错误倾向而且比较有效率的代码。

### 什么是C++对象模型

有两个概念可以解释 C++ 对象模型：

1.语言中直接支持面向对象程序设计的部分。
2.对于各种支持的底层实现机制。

语言层面的支持，涵盖于我的 C++ primer 一书以及其他许多 C++ 书籍当中。至于第二个概念，则几乎不能够于当前任何读物中发现，只有 [ELLIS90] 和 [STROUP94] 勉强有一些蛛丝马迹。本书主要专注于 C++ 对象模型的第二个概念。本书语言遵循 C++ 委员会于 1995 冬季会议中通过的 Standard C++ 草案（除了某些细节，这份草案应该能够反映出该语言的最终版本）。

C++ 对象模型的第一个概念是一种“不变量”。例如，C++ class 的完整 virtual functions 在编译时期就固定下来了，程序员没有办法在执行期动态增加或取代其中某一个。这使得虚拟调用操作得以有快速的派送（dispatch）结果，付出的成本则是执行期的弹性。

对象模型的底层实现机制，在语言层面上是看不出来的——虽然对象模型的语意可以使得某些实现品（编译器）比其他实品更接近自然。例如，virtual function calls，一般而言是通过一个表格（内含 virtual functions 地址）的索引而决议得知。一定要使用如此的 virtual table 吗？不，编译器可以自由引进其他任何变通做法。如果使用 virtual table，那么其布局、存取方法，产生时机以及数百个细节也都必须决定下来，而所有决定也都由每一个实现品（编译器）自行取舍。不过，既然说到这里，我也必须明确告诉你，目前所有的编译器对于 virtual function 的实现法都是使用各个 class 专属的 virtual table，大小固定，并且在程序执行前就已经构造好了。

如果 C++ 对象模型的底层机制并未标准化，那么你可能会问：何必探讨它呢？主要的理由是，我的经验告诉我，如果一个程序员了解底层实现模型，他就能够写出效率较高的代码，自信心也比较高。一个人不应该用猜的方式，或是等待某大师的宣判，才确定“合适提供一个 copy constructor 而何时不需要”。这类问题的解答应该来自于我们对于对象模型的了解。

写这本书的第二个理由是为了消除我们对于 C++ 语言（及其对面向对象的支持）的各种错误认识。下面一段话节录自我收到的一封信，来信者希望将 C++ 引入其程序环境中：


> 我和一群人一起工作，他们过去不曾写过（或完全不熟悉） C++ 和 OO 。其中一位工程师从 1985 年就开始写 C 了，他强烈地认为 C++ 只对那些 user-type 程序才好用，对 server 程序却不理想。他说如果要写一个快速而有效率的数据库引擎，应该使用 C 而非 C++。他认为 C++ 庞大又迟缓。


C++ 当然并不是天生地庞大又迟缓，但我发现这似乎成为 C 程序员的一个共识。然而，光是这么说并不足以使人信服，何况我又被认为是 C++ 的“代言人”。这本书就是企图极尽可能将各式各样的 Object facilities（如 inheritance、virtual functions、指向 class menbers 的指针······）所带来的额外负荷说个清楚。

除了我个人回答这封信，我也把此信寄给 HP 公司的 Steve Vinoski；先前我曾与他讨论过 C++ 的效率问题。以下节录自他的响应：

> 过去数年我听过太多与你的同时类似的看法。许多情况下，这种看法是源于对 C++ 事实真相的缺乏了解。就在上周，我才和一位朋友闲聊，他在一家 IC 制造厂服务，他说他们不使用 C++，因为“它在你的背后做事情”。我继续追问，于是他说根据他的了解，C++ 调用 malloc() 和 free() 而不让程序员知道。这当然不是真的。这是一种所谓的迷思与传说，引导出类似于你的同事的看法······
<br>
在抽象性和实际性之间找出平衡点，需要只是、经验，以及许多思考。 C++ 的使用需要付出许多心力，但是我的经验告诉我，这项投资的报酬率相当高。

我喜欢把这本书想象成我对那一封读者来信的回答。是的，这本书是一个知识陈列库，帮助大家去除围绕在 C++ 四周的迷思与传说。

如果 C++ 对象模型的底层机制会因为实现品（编译器）和时间的变动而不同，我如何能够对于任何特定主题提供一般化的讨论呢？静态初始化（Static initialization）可为此提供一个有趣的例子。

已知一个 class X 有着 constructor，像这样：
```cpp
class X
{
    friend istream& operator>>(istream&, X&);
public:
    X(int sz = 1024) { ptr = new char[sz]; }
    ...
private:
    char *ptr;
};
```
而一个 class X 的 global object 的声明，像这样：
```cpp
X buf;
main(){
    // buf 必须在这个时候构造起来
    cin >> setw(1024) >> buf;
    ...
}
```
C++ 对象模型保证，X constructor 将在 main() 之前便把 buf 初始化。然而它并没有说明这是如何办到的。答案是所谓的静态初始化（static initialization），实际做法则有赖于开发环境对此的支持属于哪一层级。

原始的 cfront 实现品不单只是假想没有环境支持，它也假想没有明确的目标平台，唯一能够假想的平台就是 UNIX 及其衍化的一些实体。我们的解决之道也因此只专注在 UNIX 身上：我们使用 nm 命令。CC 命令（一个 UNIX shell script）产生出一个可执行文件，然后我们把 nm 施行于其上，产生出一个新的 .c 文件。然后编译这个新的 .c 文件，再重新链接出一个可执行文件（这就是所谓的 munch solution）。这种做法是以编译器时间来交换移植性。

接下来是提供一个“平台特定”解决之道：直接验证并穿越 COFF-based 程序的可执行文件（即所谓的 patch solution），不再需要 nm、compile、relink。COFF 是 Common Object File Format 的缩写，是 System V pre-Release 4 UNIX 系统所发展出来的格式。这两种解决方案都属于程序层面，也就是说，针对每一个需要静态初始化的 .c 文件，cfront会 产生出一个 sti 函数，执行必要的初始化操作。不论是 patch solution 或是 munch solution，都会去寻找以 sti 开头的函数，并且安排它们以一种未被定义的次序执行起来（经由安插在 main() 之后第一行的一个 library function_main() 执行之）（译注：本书第 6 章对此有详细说明）。

System V COFF-specific C++ 编译器与 cfront 的各个版本平行发展，由于瞄准了一个特定平台和特定操作系统，该编译期因而能够影响链接器特地为它修改：产生出一个新的 .ini section，用以收集需要静态初始化的 objects。链接器的这种扩充方式，提供了所谓的 environment-based solution，那当然更在 program-based solution 层面之上。

至此，任何以 cfront program-based solution 为基础的一般化（泛型）操作将令人迷惑。为什么？因为 C++ 已经成为主流语言，它已经接收了更多更多的 environment-based solutions。这本书如何维护其间的平衡呢？我的策略是：如果在不同的 C++ 编译器上有重大的实现技术差异，我就至少讨论两种做法。但如果 cfront 之后的编译器实现模型只是解决 cfront 原本就已理解的问题，例如对虚拟继承的支持，那么我就阐述历史的演化。当我说到“传统模型”时，我的意思是 Stroustrup 的原始构想（反映在 cfront 身上），它提供一种实现模式，在今天所有的商业化实现品上仍然可见。

## 本书组织

第 1 章，关于对象（Objects Lessons），提供以对象为基础的观念背景，以及由 C++ 提供的面向对象程序设计典范（paradigm。译注：关于 paradigm 这个字请参阅本书#22页的译注）。本章包括对于对象模型的一个大略浏览，说明目前普及的工业产品，但没有对多重继承和虚拟继承有太靠近的观察（那是第 3 章和第 4 章的重头戏）。

第 2 章，构造函数语意学（The Semantics of Constructors），详细讨论 constructor 如何工作。本章谈到 constructors 何时被编译器合成，以及给你的程序效率带来什么样的意义。

第 3 章至第 5 章是本书的重要题材。在这里，我详细地讨论了 C++ 对象模型的细节。第 3 章，Data 语意学（The Semantics of Data），讨论 data menbers 的处理。第 4 章，Function 语意学（The Semantics of Funtion)，专注于各式各样的 member functions，并特别详细地讨论如何支持 virtual functions。第5章，构造、解构、拷贝语意学（Semantics of Construction, Destruction, and Copy），讨论如何支持 class 模型，也讨论到 object 的生命期。每一章都有测试程序以及测试数据。我们对效率的预测将拿来和实际结果做比较。

第 6 章，执行期语意学（Run time Semantics），检视执行期的某些对象模型行为，包括临时对象的声明及其死亡，以及对 new 运算符和 delete 运算符的支持。

第7章，在对象模型的尖端（On the Cusp of the Object Model），专注于 exception handing、template support、runtime type identification。

## 预定的读者

这本书可以扮演家庭教师的角色，不过它定位在中级以上的 C++ 程序员，而非 C++ 新手。我尝试提供足够的内容，使它能够被任何有点 C++ 基础（例如读过我的 C++ primer 并有一些实际经验）的人接受。理想的读者是，曾经有过数年的 C++ 程序经验，希望进一步了解“底层做些什么事情”的人。书中某些部分甚至对于 C++ 高手也具有吸引力，比如临时性对象的产生，以及 named return value (NRV) 优化的细节等等。在与本书素材相同的各个公开演讲场合中，我已经证实了这些材料的吸引力。

## 程序范例及其执行

本书的程序范例主要有两个目的：
1.为了提供书中所谈的C++对象模型各种概念的具体说明。
2.提供测试，以测量各种语言性质的相对成本。

无论哪一种意图，都只是为了展现对象模型。举例而言，虽然我在书中有大量的举例，但我并非建议一个真实的 3D graphic library 必须以虚拟继承的方式来表现一个3D点（不过你可以在 [POKOR94] 中发现作者 Pokorny 的确在这么做）。

书中所有的程序测试都在一部 SGI Indigo2xL 上编译执行，使用 SGI 5.2 UNIX 操作系统中的 CC 和 NCC 编译器。CC 是 cfront 3.0.1 版（它会产生出 C 码，再由一个 C 编译器重新编译为可执行文件）。NCC 是 Edison Design Group 的 C++ front-end 2.19 版，内含一个由 SGI 供应的程序代码产生器。至于时间测量，是采用 UNIX 的 timex 命令针对一千万次迭代测试所得的平均值。

虽然在 xL 机器上使用这两个编译器，对读者而言可能觉得有些神秘，我却觉得对这本书的目的而言，很好。不论是 cfront 或现在的 Edison Design Group's C++ front-end（Bjarne 称其为“cfront 的儿子”），都与平台无关，它们是一种一般化的编译器，被授权给 34 家以上的计算机制造商（其中包括 Gray、SGI、Intel）和软件开发厂商（包括Centerline 和 Novell，后者是原先的 UNIX 软件实验室）。效率的测量并非为了对目前市面上的各家编译系统进行评比，而只是为了提供 C++ 对象模型之各种特性的一个相对成本测量。至于商业评比的效率数据，你可以在几乎任何一本计算机杂志的计算机产品检验报告中获得。

## 致谢

略

## 参考书目

略

[第 0 章 导读（译者的话）](introduction.md)
