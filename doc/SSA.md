# Static single assignment form
*静态单一赋值形式*
[链接](https://en.wikipedia.org/wiki/Static_single_assignment_form)

在编译器设计中，静态单一赋值形式（通常缩写为SSA形式或简称为SSA）是中间表示（IR）的属性，它要求每个变量只分配一次，并且每个变量在使用之前定义。原始IR中的存在变量以版本划分，新变量通常由原始名称在文本中用下标表示，以便每个定义都有自己的版本。在SSA形式中，use-def链是显式的，每个包含一个元素。

SSA由Barry K. Rosen，Mark N. Wegman和F. Kenneth Zadeck于1988年提出。[1] Ron Cytron，Jeanne Ferrante和IBM的前三位研究人员开发了一种算法，可以有效地计算SSA形式。[2]

可以期望在Fortran或C的编译器中找到SSA，而在函数式语言编译器中，例如Scheme，ML和Haskell的编译器，通常使用连续传递样式（CPS）。 SSA在形式上等同于除了非本地控制流之外的良好行为的CPS子集，当CPS用作中间表示时不会发生这种情况。因此，以一个方式制定的优化和转换立即适用于另一个。

## Benefits

SSA的主要用途来自于它如何通过简化变量的属性来同时简化和改进各种编译器优化的结果。 例如，考虑这段代码：

```
 y := 1
 y := 2
 x := y
```

人类可以看到第一个赋值是不必要的，并且在第三行中使用的y的值来自y的第二个赋值。 程序必须执行到达定义分析以确定这一点。 但如果该程序采用SSA形式，则这些都是即时的：

 ```
 y1 := 1
 y2 := 2
 x1 := y2
 ```

通过使用SSA启用或强大增强的编译器优化算法包括：

- 常量折叠
- value范围传播[3]
- 稀疏条件常数传播
- 无用代码消除
- 全局变量编号
- 部分冗余消除
- 力量减少
- 注册分配

## Converting to SSA

将普通代码转换为SSA形式主要是用新变量替换每个赋值的目标，并将变量的每个用法替换为到达该点的变量的“版本”。 例如，请考虑以下控制流程图：

![SSA_example1.1](/home/cyoung/CLionProjects/implement_llvm/doc/image/SSA_example1.1.png)

更改“x <-- x - 3”左侧的名称，并将x的以下用法更改为该新名称将使程序保持不变。 这可以在SSA中通过创建两个新变量来利用：x1和x2，每个变量只分配一次。 同样，为所有其他变量提供可区分的下标产生：

![SSA_example1.2](/home/cyoung/CLionProjects/implement_llvm/doc/image/SSA_example1.2.png)

很清楚每个用途所指的定义，除了一种情况：底部块中y的两个使用都可以指y1或y2，这取决于控制流采用的路径。

要解决此问题，会在最后一个块中插入一个特殊语句，称为Φ（Phi）函数。 该语句将通过“选择”y1或y2生成y的新定义，称为y3，具体取决于过去的控制流程。

![SSA_example1.3](/home/cyoung/CLionProjects/implement_llvm/doc/image/SSA_example1.3.png)

现在，最后一个块可以简单地使用y3，并且将以任一方式获得正确的值。不需要x的Φ函数：只有x的一个版本，即x2到达这个位置，所以没有问题（换句话说，Φ（x2，x2）= x2）。

给定一个任意的控制流图，很难说出插入Φ函数的位置以及哪些变量。这个一般性问题有一个有效的解决方案，可以使用一个称为优势边界的概念来计算（见下文）。

Φ功能未在大多数机器上实现为机器操作。编译器可以简单地通过在存储器（或相同的寄存器）中使用与产生Φ功能的输入的任何操作的目的地相同的位置来实现Φ功能。然而，当同时操作推测性地产生对Φ功能的输入时，这种方法不起作用，如在宽问题机器上可能发生的那样。通常，广泛发布机器具有在这种情况下由编译器用于实现Φ功能的选择指令。

根据Kenny Zadeck [4]的说法，Φ函数最初被称为伪函数，而SSA是在20世纪80年代在IBM Research开发的。 Φ功能的正式名称仅在作品首次发表在学术论文中时才被采用。

### Computing minimal SSA using dominance frontiers(边界计算)

首先，我们需要一个dominator的概念：我们说如果不通过A第一个就不可能到达B，则节点A严格控制控制流图中的节点B.这很有用，因为如果我们到达B，我们就知道A中的任何代码都已运行。如果A严格支配B或A = B，我们说A支配B（B由A支配）。

现在我们可以定义支配边界：如果A不严格支配B，则节点B处于节点A的支配边界，但确实支配B的一些直接前任。（可能节点A是B的直接前身。然后，因为任何节点都支配自身而节点A占主导地位，节点B处于节点A的支配边界。）从A的角度来看，这些节点是其他控制路径（不通过A）最早出现的节点。

优势边界捕获我们需要Φ函数的精确位置：如果节点A定义了某个变量，那么该定义和该定义单独（或重新定义）将到达每个节点A占主导地位。只有当我们离开这些节点并进入支配边界时，我们必须考虑引入同一变量的其他定义的其他流量。此外，控制流程图中不需要其他Φ功能来处理A的定义，我们可以做到这一点。

计算优势边界集[5]的一种算法是：

```
for each node b
    if the number of immediate predecessors of b ≥ 2
        for each p in immediate predecessors of b
            runner := p
            while runner ≠ idom(b)
                add b to runner’s dominance frontier set
                runner := idom(runner)
```

注意：在上面的代码中，节点n的前一个前导是控制转移到节点n的任何节点，而idom（b）是立即支配节点b（单个集合）的节点。

有一种有效的算法可以找到每个节点的优势边界。 该算法最初在Cytron等人中描述。 同样有用的是Andrew Appel的书“Java中的现代编译器实现”第19章（剑桥大学出版社，2002年）。 有关详细信息，请参阅该文章。

Rice大学的Keith D. Cooper，Timothy J. Harvey和Ken Kennedy在他们的题为“简单，快速优势算法”的论文中描述了一种算法。[5] 该算法使用精心设计的数据结构来提高性能。

## Variations that reduce the number of Φ functions

“最小”SSA插入所需的最少数量的Φ函数，以确保每个名称仅被赋值一次，并且原始程序中每个名称的引用（使用）仍然可以引用唯一名称。 （需要后一个要求以确保编译器可以在每个操作中记下每个操作数的名称。）

然而，其中一些Φ功能可能已经死亡。 因此，最小SSA不一定产生特定过程所需的最少数量的Φ功能。 对于某些类型的分析，这些Φ功能是多余的，可能导致分析效率降低。

### Pruned SSA

修剪的SSA形式基于简单的观察：仅在Φ函数之后“活”的变量需要Φ函数。 （这里，“实时”表示该值沿着从所讨论的Φ函数开始的某个路径使用。）如果变量不是实时的，则不能使用Φ函数的结果，并且Φ函数的赋值为死。

修剪的SSA形式的构造使用Φ函数插入阶段中的实时变量信息来确定是否需要给定的Φ函数。如果原始变量名称不在Φ功能插入点处，则不插入Φ功能。

另一种可能性是将修剪视为死代码消除问题。然后，只有当输入程序中的任何用途被重写到它时，或者如果它将用作另一个Φ函数中的参数时，Φ函数才是有效的。输入SSA表单时，每次使用都会被重写为最接近它的定义。然后，Φ函数将被认为是实时的，只要它是支配至少一个用途的最近定义，或至少一个活Φ的参数。

### Semi-pruned SSA

半修剪的SSA形式[6]试图减少Φ函数的数量而不会产生计算实时变量信息的相对高的成本。 它基于以下观察：如果变量在进入基本块时永远不会存在，则它永远不需要Φ函数。 在SSA构造期间，省略任何“块局部”变量的Φ函数。

计算块局部变量集是一种比完全实时变量分析更简单，更快速的过程，使得半修剪SSA形式比修剪SSA形式更有效。 另一方面，半修剪的SSA形式将包含更多的Φ函数。

## Converting out of SSA form

SSA格式通常不用于直接执行（尽管可以解释SSA [7]），并且它经常在“另一个IR之上”使用它与之保持直接对应。这可以通过将SSA“构造”为一组函数来实现，这些函数在现有IR的部分（基本块，指令，操作数等）与其SSA对应物之间进行映射。当不再需要SSA格式时，可以丢弃这些映射函数，只留下现在优化的IR。

在SSA表单上执行优化通常会导致纠缠的SSA-Web，这意味着存在Φ指令，其操作数并非都具有相同的根操作数。在这种情况下，使用颜色输出算法来解决SSA问题。朴素算法沿着每个前任路径引入一个副本，这导致不同根符号的源被放入Φ而不是Φ的目的地。有多种算法可以用较少的副本从SSA中出来，大多数使用干扰图或一些近似它来进行复制合并。

## Extensions

SSA格式的扩展可以分为两类。

重命名方案扩展会更改重命名标准。 回想一下，SSA表单在为每个变量赋值时重命名。 替代方案包括静态单一使用形式（在使用时在每个语句中重命名每个变量）和静态单一信息形式（在赋值时重命名每个变量，并在后支配边界处重命名）。

特定于功能的扩展保留变量的单一赋值属性，但包含新的语义以模拟其他功能。 一些特定于功能的扩展模拟高级编程语言功能，如数组，对象和别名指针。 其他特定于功能的扩展模拟了低级架构功能，如推测和预测。