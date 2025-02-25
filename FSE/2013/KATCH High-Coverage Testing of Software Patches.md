source: https://bbs.pediy.com/thread-217255.htm

1.简介

在竞争激烈的软件市场中，现代软件在各个版本之间只有很短的间隔时间。比如，谷歌Chrome浏览器和Firefox浏览器每隔6个周就会推出一个新版本[1]。由于缺乏软件测试，软件漏洞普遍存在于软件的整个生命周期。更糟糕的是，用于修补漏洞的补丁往往自身就存在着漏洞。近期研究表明，所有补丁的近9%存在错误；另一项针对大型商用操作系统的研究则发现，14.8%到24.4%的补丁存在错误，并且其中的22%属于内存相关的漏洞。

判断一个补丁是否具有脆弱性是一项有挑战性的工作。由于针对一个大型程序的完整回归测试将花费大量的时间，因此传统的回归测试技术[4,5]通常是低效的；而且，测试过程不可能检测所有的执行路径。另一种流行的技术是静态分析源代码或二进制代码。基于不同的策略，静态分析可以发现不同类型的漏洞；然而，由于没有运行时信息以及程序输入，静态分析通常有很高的误报率，而且其无法发现运行时漏洞，例如，一个递归函数中由特定输入引起的缓冲区溢出。

相比于回归测试和静态分析，符号执行[8-11]的一个明显优势就是可以全面地测试软件。符号测试工具将符号值假定为输入，而不是获取具体的输入。对于每一个条件分支，符号值性工具产生一个新的约束，并探索一条新的程序路径；而沿着每一条路径，工具都将检查输入是否引起漏洞。

尽管符号执行已被证明在实际的软件测试中表现优异，但其有一个严重的问题就是路径爆炸。程序中可能路径的数目会随着程序规模的增长而呈指数增长，甚至会由于失控的循环迭代而趋近无穷。为了避免针对整个程序的符号执行，以及缓解待分析软件补丁的路径爆炸问题，某些研究者会选择仅分析程序中与补丁相关的部分[12-14]。测试用例被用于补丁所改变的代码。然而，自动产生大量有效的测试用例是十分困难的。

本文中，我们将介绍一种名为“KPSec”的工具，它可以有效探测补丁是否引入了新的漏洞。首先，我们的方法综合运用了控制流图（CFG）和剪枝算法来去除程序中未被补丁影响的部分。对于程序剩下的部分，我们将寻找那些可能引起内存漏洞的函数，并将其标记为安全点。然后，构造生成可能通过安全点的执行路径；对于每一条执行路径，我们使用符号执行来遍历沿途的所有安全点，然后利用多种安全检查来寻找不同类型的内存漏洞。最后，在执行并检查了所有的执行路径之后，我们将报告所发现的安全问题，包括每一个漏洞的类型，位置以及执行路径。KPSec工具的执行流程如图1所示。




图1 KPSec工具的执行流程

本文主要有如下创新：

不同于其他同类工作，KPSec工具无需研发人员构造测试用例，即可判断补丁是否引入了内存相关漏洞；

我们采用了剪枝算法来处理众所周知的路径爆炸问题，因此研发人员能够高效地检测补丁和获取结果报告；

对于补丁所影响的每一个可能引起内存漏洞的函数，我们进行向前/向后遍历分析的方法来搜索所有的相关路径，并结合带有安全检查器的符号执行引擎来发现不同类型的漏洞。

本文按照如下结构组织：我们在第二部分给出方法的简要概述，并在第三部分介绍实现细节；第四部分介绍评估结果，第五部分则讨论相关工作；最后在第六部分进行总结并对未来工作进行展望。

2.解决方案概述

一般来说，当补丁提交到软件存储库时，需要进行测试，以确保所需功能得以正确实现而没有引入任何漏洞。不幸的是，研发人员通常没有提供测试用例来执行补丁相关的每一条路径。

不同于传统的软件补丁测试方法，我们的目标是不依赖于测试用例而自动测试补丁相关的代码。图1展示了KPSec的工作流程：KPSec将程序源代码和补丁视为输入，进而自动检查补丁，最后完成对补丁的分析。补丁检查包括三个主要阶段：预处理，执行路径生成和带有安全检查器的符号执行。

在预处理阶段，我们对源代码和补丁进行预处理。对于补丁来说，由于位于同一个基本块的代码总是一起执行，因此我们从每一个基本块中选择一行代码来代表；对于源代码，我们应用补丁来生成打补丁后的程序，并使用静态分析来搜索其基本块。预处理阶段的输出是补丁所影响的打补丁后程序中的一组基本块。

执行路径生成阶段在基本块集合中搜寻安全敏感的函数和变量，并构建经过这些函数和变量的主要执行路径。

最后一个阶段是带有安全检查器的符号执行。在这个阶段，我们使用符号执行引擎来符号化地执行每条执行路径。每当执行到一个安全敏感的函数或变量，就生成其安全约束。若果安全检查器确定（在该点）不满足安全约束，则认为找到一个安全漏洞。检查器记录漏洞细节，包括类型，位置和执行路径。

我们设计了KPSec的基础结构来自动执行所有三个阶段，而无需修改程序的补丁或者源代码。

3.实现细节

在本小节中，我们基于一个例子来描述KPSec的实现细节，如图2所示。在这个例子中，一个补丁通过增加静态缓冲区大小的方式来修补缓冲区溢出漏洞；然而，缓冲区大小仍然不够大。若szMsg的长度大于30字节，szBuf将溢出。我们将介绍KPSec如何产生主要补丁相关执行路径的关键步骤。


图2 缓冲区溢出例子

A．预处理

预处理有两个主要任务：第一，预处理补丁以去除不相关的行，并保留一行来代表每一个基本块；第二，预处理程序源代码，获取补丁所影响的基本块。

我们首先预处理补丁，以获取影响程序行为的行。不需要记录未执行的行，比如空白行、宏定义以及注释等。结果是获得一组可执行且代表不同基本块的代码行。

接下来，我们应用补丁获得程序源代码的新版本，然后我们使用LLVM工具链将新版本编译生成LLVM字节码，同时伴随生成能够将LLVM字节码重新映射为源代码的LLVM调试信息。之后我们将构建补丁修复的程序的CFG图，图中每个节点代表一个基本块，每条有向线段被用于代表控制流中的一次跳转或分支。通过补丁预处理阶段所获得的代码行集合和LLVM调试信息，我们可以确定补丁是否影响某个基本块。

预处理阶段涉及到我们在KPSec中如何缓解路径爆炸问题；我们仅仅探测补丁相关的基本块中的路径。预处理之后，我们获得了补丁修复程序的CFG图，和补丁相关的基本块。

B．执行路径生成

本阶段的目标是构造生成基于基本块的执行路径，流程如下所述：

第一步，对于每个基本块，我们找出安全敏感的函数和变量。安全敏感的函数是指函数中包含敏感数据；比如，strcpy和memcpy函数就是安全敏感函数，因为它们可能引起缓冲区溢出。安全敏感的变量是指变量直接或间接地对内存进行操作；比如，若一个由malloc函数分配的缓冲区指针没有正确处理，则可能引起内存泄漏。

第二步，对于每个函数和变量，我们进行向前/向后分析来构造数据流图（DFG）。

第三步，通过遍历DFG图来自动构造生成包含安全敏感函数和变量的主要执行路径。

执行路径生成的细节如下。

1)安全点定义：安全点的主要特征是，其参数包含来自文件、用户输入或网络套接字的不可信数据，比如图2所示的补丁中的strcpy函数。我们根据安全函数/变量的类型来对安全点进行分类。

内存管理：内存管理函数包括内存分配函数和内存释放函数。我们以malloc和alloc之类的内存分配函数为例。通常，这些函数的参数是内存复制函数的目标缓冲区。我们跟踪内存分配函数来获取分配缓冲区的大小，然后标记这些缓冲区；当一个函数或程序结束时，我们将检查这些标记的缓冲区是否被释放。类似的，我们也要监控内存释放函数。

内存复制：strcpy，strcat和memcpy之类的内存复制函数，将一个源缓冲区填充/连接到一个目标缓冲区；如果目标缓冲区的大小不足以存放来自源缓冲区的数据，则会发生缓冲区溢出。这类函数是缓冲区溢出攻击最常见的目标。

内存读取：我们经常使用一个偏移指针来访问缓冲区。如果这个指针没有被小心处理的话，将导致越界访问或者未初始化读取（这发生在一段代码未经初始化就访问缓冲区的情况下）。需要注意的是，对于memset函数要特别对待，因为常使用该函数来初始化一个缓冲区指针。

2)数据流图构造：通过CFG图、基本块和安全点，我们进行数据流分析来构造DFG图，并生成所有可能的补丁相关的执行路径。以某一个安全点为起点，我们使用向前/向后分析来构造两种数据流树：前向数据流树和后向数据流树；结合这两种数据流树来构造DFG图。因此，这两棵树有共同的代表安全点的根节点。最后，通过遍历整个DFG图来自动生成所有可能的主要的执行路径。

我们用来构造数据流树的过程与数据流分析是一致的。基于基本块并起始于安全点，我们使用前向分析来跟踪参数，直至到达敏感数据的源头；我们使用后向分析来获取安全点相关的变量集合，然后我们在其他块中继续跟踪这些变量。在数据流树中存在以下几种节点：

根节点：一棵数据流树的根节点，即为预处理阶段所获得的安全点。

子节点：在前向数据流树中，子节点代表可以获得父节点的源节点的一个表达式；在后向数据流树中，父节点是子节点的源节点。如果一个表达式被赋值给一个变量，则我们将这个表达式作为变量的源节点。例如，对于语句int len = strlen(szMsg)，表达式strlen(szMsg)就是变量len的源节点。

叶节点：在前向数据流树中，叶节点代表首次变量赋值；而在后向数据流树中，他指的是执行路径的终点。在变量声明时对变量进行初始化的常量表达式是（数据流图的）源头，比如int x=0。另外，我们将应用来自文件、命令行或用户输入的数据对变量进行初始化的代码语句同样视为源头，比如scanf(“%d”, &x)和*msg = argv[1]。

前向数据流树按照如下步骤构造：

首先，我们初始化一个名为stack-init的<security-point, parent>栈；其中每一个security-point使用静态分析来确定，而parent代表安全点的父节点，并且在开始时被赋值为NULL，这代表当前安全点为根节点。我们同样初始化一个空的集合名为set-processed，这个集合用来存储已经处理的参数；

然后，我们从stack-init中取出一个安全点，将其作为一棵新的数据流树的根节点；从根节点出发，我们根据父节点和子节点之间的关系来寻找其源代码语句。如果源代码语句在set-processed集合中，我们忽略该语句；否则，我们将该语句放入数据流树中作为子节点，并将其放入stack-processed集合中；

对于每个子节点，我们重复进行该流程直到抵达叶节点。

算法1阐述了该流程的细节。
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26

算法1 前向数据流树构造
输入:
stack-init栈, 控制流图
输出:
数据流树 (DFT)
1:  //初始化一个空集，来存放已处理的安全点
2:  make empty set(set − processed)
3:  //处理stack-init中的每一项
4:  while not empty(stack int) do
5:    eachitem = stack int.pop()
6:    if has processed(eachitem) then
7:      continue
8:    end if
9:    if is root(eachitem) then
10:     //构造一棵新的数据流树
11:     root = make tree(eachitem)
12:   end if
13:   for source in find source(eachitem) do
14:     if not orign(source) then
15:       stack init.push(source,eachitem)
16:     end if
17:     eachitem:child = source
18:     set − processed.push(eachitem)
19:   end for
20: end while
21: return root;

构造后向数据流树也是使用类似的方法。前向/后向数据流树的唯一区别就是父节点与子节点之间的关系相反：在前向数据流树中，子节点是父节点的源节点；而在后向数据流树中，父节点是子节点的源节点。对于探索路径来说数据流树不会太大，因为我们只考虑包含安全点的路径。最后，我们将两种数据流树结合起来以生成DFG图，然后通过简单遍历DFG图来生成所有可能的执行路径。对于如图2所示的补丁来说，我们生成一条主路相关的执行路径。

C．带有安全检查器的符号执行

我们的符号执行引擎是基于KLEE[15]的。我们修改KLEE来执行部分符号执行，并实现了五个安全检查器来探测存在的内存相关漏洞。

通常，内存安全漏洞是由函数的不安全使用或者对函数参数（特别是指针）的不恰当处理而引起的。为了获得安全检查器所需要的指针信息，我们定义了一个数据结构用于存储符号执行路径中的指针细节；这个数据结构与SecTAC[16]中用于记录指针信息的符号表类似。我们扩展这个数据结构使其能够记录更多的指针信息，从而能够处理更多类型的内存漏洞。该数据结构如清单1所述。
1
2
3
4
5
6
7
8

typedef struct _pointer_info {
unsigned int uID;
unsigned char uType;
unsigned int uStart;
unsigned int uLength;
bool bInit;
bool bHasfreed;
}pointer_info, *ppointer_info;

清单1 pointer_info结构

pointer_info结构共有六个成员：uID域是唯一资源标识，如果指针指向同一个变量则它们有相同的uID。uType域代表指针的类型，比如int，char或者未定义类型；它也说明了指针分配的位置，比如栈或堆。我们不考虑栈变量相关的内存泄漏问题，因为在其生命周期结束时系统将自动恢复其内存。uStart域存储了数组的开始位置，uBuflen（应该是uLength域吧？？？）域则存储了指针的长度。bInit域作为标志指示了缓冲区是否初始化过。bHasfreed域则是指示指针是否已被释放的标志。对于一条执行路径中的安全函数的每一个指针，我们都将分配一个pointer_info结构，并在符号执行的过程中填充结构体成员的值。

基于指针信息，我们设计了五个检查器来探测五中类型的内存漏洞，包括缓冲区溢出，越界访问，内存泄漏，指针悬挂以及未初始化数据读取。在本小节接下来的部分中，我们将详细介绍每一种检查器。

1)缓冲区溢出和越界访问检查器：当程序试图在缓冲区中存储大于其尺寸的数据时，缓冲区溢出将会发生。攻击者试图覆盖指令指针（IP）、基指针（BP）和其他寄存器，来引起异常、分片故障或者其他的错误。在实际代码中，使用C内存函数（如memcpy、strcpy或sprintf）可能导致缓冲区溢出，而这些函数就是我们在执行路径生成的最后一个阶段要跟踪的安全点。

当符号执行引擎遇到一个不安全的C内存函数时，内存缓冲区溢出检查器获取其参数的详细信息，一般就是将指针存入pointer_info结构。然后，检查器确定函数是否存在内存缓冲区溢出漏洞。在如图2所示的简单补丁片段中，在函数foo中有一个安全点名为strcpy；其安全约束条件是：szBuflen > szMsg.uBuflen，其参数是两个char类型的指针。我们分配两个pointer_info变量来记录szBuf和szMsg指针信息。通过指针关系分析和数据流分析，若szMsg的长度大于30字节，则安全约束不再满足，此时内存缓冲区溢出发生。

内存越界访问检查器与内存缓冲区溢出检查器类似，但是需要检查的对象是一个指针。尽管C/C++已经在语义上对指针进行了很好的定义，当在很多程序中仍存在着可能导致越界内存访问漏洞的代码语句。我们引入一种基于规则的检查器来检测越界内存访问漏洞：在检查器中，一个数组将被视为一个指针，而且一个内存对象与其指针的指向关系在符号执行引擎的路径执行阶段已被记录。检查器将捕获所有通过指针对内存中数据进行存储、分享或访问的代码语句；然后，它将获取指针指向空间的长度和用于访问对象的索引。若索引在对象空间范围之外，则说明检测到一个越界内存访问漏洞。

2)内存泄漏和释放后重用检查器：内存泄漏是最难检测的内存漏洞类型，因为在系统耗尽内存触发内存不足（OOM）终止机制之前，内存泄漏可能不会引起任何问题。远程攻击者可以利用内存泄漏来完成一次拒绝服务攻击。在分配的内存缓冲区未被及时释放时，可能会发生内存泄漏。

在一条代码路径的符号执行过程中，检查器将生成一个已分配堆对象的集合。对指针与堆对象之间关系的分析算法与SecTAC中的指针分析类似。当调用一个内存释放函数（比如C语言中的free函数和C++语言中的delete方法）释放一个堆对象时，检查器将相关的指针从集合中移除。函数返回值、程序里的静态对象和全局堆对象不会导致内存泄漏，因而检查器将它们也从集合中移除。在路径的终点，检查器检测到未被释放和不可达的堆对象。

在内存泄漏检查器的基础上，我们添加两条规则使得检查器能够检测释放后重用漏洞和指针悬挂漏洞。在C/C++语言中，从内存中彻底删除一个对象不会改变任何相关的指针，它们仍将指向内存中的相同位置，然而会呈悬挂状态。如果一个函数仍使用该指针而又没有将其与另一个内存对象相关联，则会存在一个释放后重用漏洞。

检查释放后重用漏洞的规则如下：检查器拦截所有的内存对象操作，并检查相关指针的bHasfreed值是true还是false；如果该对象已被释放，则检查器报告漏洞；然后在路径执行的终点，检查器检查所有已释放的内存对象，检查他们的相关指针是否已被置为NULL；如果一个指针的值不为NULL，则根据悬挂指针的定义它是一个悬挂指针。

3)未初始化数据访问检查器：当程序代码访问一段未初始化的缓冲区时，将发生未初始化数据访问。访问堆栈对象中的未初始化数据，可能会导致特别难以调试的未定义/不确定性的行为。为了降低运行时代价，我们引入检查器来集中检查对堆栈缓冲区中未初始化数据的访问。在分配一段栈/堆缓冲区时，检查器用特殊污染的数据来填充缓冲区，并将指针的bHasinit值设为false；在路径执行期间，检查器捕获所有指针相关的操作，来检查是否存在任何未初始化的数据；若是，即其bHasinit成员的值为false，则检测到一个未初始化数据访问漏洞。

我们同样需要考虑初始化操作，比如memcpy和memset。如果一个指针是这些函数的目标参数，我们将bHasinit值置为true。当检查器遇到一个第三方库函数时，KPSec跳过函数调用，并将对应指针的bHasinit值置为true；因为第三方库函数将为指针赋值。

4.实验评估

A．实验环境

为了测试KPSec工具的有效性，我们从GNU binutils、GNU coreutils和OpenSSL软件上收集了上百个补丁进行评估；它们应用广泛，且（样本）数量足够说明KPSec的有效性。

GNU binutils软件包含大约100,000行C语言代码（LOC）；GNU coreutils包括多种程序（如ls和cut），大概有200,000行C语言代码（LOC）；OpenSSL是一种广泛应用的安全关键系统，包含大概400,000行C语言代码（LOC）。我们从GIT资源库最近的可靠分支中收集所有的补丁。

然后，有很多补丁并不需要检查，比如，补丁的用途是仅仅改变用户手册，改变版权信息，或者为了处理构造错误而更新配置文件。

因此，我们从GNU binutils软件的版本2.22和2.24中选择209个补丁，从GNU coreutils的版本8.17和8.24中选择268个补丁，从OpenSSL的版本1.0.1a和1.0.1b中选择了284个补丁。

我们使用五台型号为Inspur NF5240M3服务器的物理主机来搭建实验环境；每台主机配备Intel Xeon E5-2420 型号的CPU和32GB大小的内存，运行64位CentOS 6.3系统。

B．实验与评估

我们首先测试KPSec工具的性能。一般来说，程序的分析时间与需要执行的路径数目成比例。因为路径总数大致等于2^(if-statements)，我们引入变量λ来代表安全点相关的分支与分支总数之比。我们使用变量λ来对补丁分类，然后在每条路径上分别运行KATCH[17]和KPSec来测量运行时间。在实验中，我们着重关注工具在执行路径和发现漏洞方面的表现；同时，我们为每条路径设置30分钟的超时时间，以忽略那些执行时间无限的路径。结果如图3所示。

图3 KPSec工具性能

补丁的平均大小都很小，λ的值介于0.01和0.13之间。结果显示，当λ小于0.07时，KPSec的执行时间将比KATCH的一半还少。KPSec总的平均执行时间是11.1分钟，而KATCH是22.3分钟。KPSec的执行时间随着λ的增长而显著增长；然后，在KATCH的执行时间和λ值之间没有明显关联。

因为KPSec通过安全点来生成补丁相关的路径，所以执行路径的总数极度依赖λ值的大小；而KATCH的设计思想是通过使用已有的测试用例来实现高覆盖度，因此它需要执行更多的路径，其中一些路径可能并不包含漏洞。

下一步我们研究KPSec工具的漏报率与误报率。在我们收集的补丁中，共存在16个已知漏洞，包括缓冲区溢出，内存泄漏，未初始化数据访问和释放后重用漏洞。作为对比，我们对每个补丁分别运行KATCH和KPSec来估计漏报率和误报率。结果如表1所示；#KV表示我们测试的漏洞数目，#FV表示KPSec和KATCH检测到的漏洞数目，#FN表示漏报的数目。


表1：检测结果总结

相比于KATCH，KPSec的漏报率很低，特别是针对内存泄漏漏洞进行检测的时候。在我们所测试的补丁中，四个内存泄漏漏洞KATCH都无法识别出来。因为KATCH是基于KLEE实现的，所以它不能比KLEE探测出更多的漏洞类型。

结果显示KPSec错过了一个漏洞；该漏洞不是由于安全敏感函数的不当使用而引起的，因此它不在KPSec工具的探测范围之内。

5.相关工作

软件安全已经引起了企业和学术界的广泛关注。在基于符号执行的全程序范围漏洞检测方面，目前有很多进展。KLEE[15]和SecTAC[16]是两款著名的符号执行引擎；这些引擎探索尽可能多的执行路径，然后在每条路径上进行符号执行操作。KLEE综合使用具体和符号执行来搜索程序路径，而SecTAC则重复使用已有的测试用例来构造所测试程序路径的轨迹。尽管全程序范围分析有着路径爆炸的问题，但对于检查补丁来说符号执行是有用的。我们的方法同样使用符号执行来检查补丁，但我们着眼于检测被补丁所影响的代码片段，并且应用了剪枝算法来缓解严重的路径爆炸问题。

Yang et al.提出了DiSE[14]来衡量符号执行，这是一种检测和描述程序改变所带来影响的新技术；DiSE进行全程序范围的符号执行操作，但去除了未被补丁影响的路径。回归验证[5]采用了抽象概念来实现可扩展性，但可能会误报差异性。Le et al.所提出的一个框架Hydrogen，包括一个名为多版本程序间控制流图（MVICFG）的单个程序的多个版本的图示，和跨越MVICFG来检测软件改动和版本变化相关的漏洞的一种需求驱动和路径敏感的符号分析。相比之下，通过探索所有可能的执行路径并进行全路径的符号执行以降低漏报率和误报率，我们的方法主要着眼于检测程序被补丁所影响的部分。Det al.所设计的KATCH，将符号执行与基于静态和动态程序分析的新型启发式方法相结合，生成回归测试用例来分析补丁所导致的变化。KATCH使用已有的测试用例作为起点来合成新的输入，然后使用三种启发式算法（贪婪搜索，活跃路径重生和定义分支）来学习补丁。KATCH有很高的覆盖度，但仅能探索执行路径一个很小的集合。与KATCH不同，我们的方法不需要生成任何测试用例。

UC-KLEE是一种新型可扩展的框架，其使用了一种未限定符号执行的变种方法来检查C/C++代码。UC-KLEE可被用于检查补丁是否引起崩溃；UC-KLEE从用户所选择的一个顶层函数开始，并且该顶层函数所调用的所有函数都将被包含；在探索穷尽了所有执行路径之后，它就可以确定一个函数是否引入了任何崩溃出错。直接调用一个函数可能会漏掉一些前提条件，而这将导致漏报；因此，UC-KLEE同时为用户提供了自动化的启发式方法和接口来手动消除这些错误。与UC-KLEE不同，我们的方法从安全点开始，通过安全点来构建执行路径，然后通过考虑所有的路径条件和安全条件，对每条路径进行符号执行操作来检查多种类型的漏洞。

6.总结及未来工作

在本文中，我们介绍了KPSec，一种用于检查补丁是否导致新的漏洞的工具。我们首先使用静态分析来获取程序中被补丁所影响的部分；然后，从可能导致内存漏洞的函数和变量开始，我们使用剪枝算法来缓解路径爆炸问题，并获取它的主要执行路径；之后，我们将符号执行和检查器相结合，探测所有的路径来寻找漏洞。我们针对来自GNU binutils、GNU coreutils和OpenSSL实际软件的补丁来测试KPSec工具，结果显示KPSec能够快速检测补丁引入的安全漏洞；对于补丁导致的16个已知漏洞，KPSec检测出了其中的15个。总而言之，KPSec在检查软件补丁的漏洞方面是快速有效的。

然而，所提出的方法仍有若干局限性，有待于在后续工作中考虑改善：第一，我们无法处理第三方库函数调用，因为该方法只在源代码模式下有效；因此对于这些函数调用，我们无法获取它们和补丁的关系，进而无法在探测过程中对其进行评估。第二，我们目前的工作集中于传统的内存相关的漏洞，因为这些漏洞难以检测，且它们可能破坏软件系统的安全稳定性；然而，补丁可能引入值得我们注意的其他类型的漏洞，比如数据竞争；很有必要对我们的安全检查器进行扩展，来检测这些类型的漏洞。
