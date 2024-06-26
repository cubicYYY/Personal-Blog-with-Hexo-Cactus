---
title: '<译>Tai-Danae Bradley: 米田视角→米田嵌入→米田引理'
tags:
  - Math
  - Translation
  - Yoneda Lemma
categories:
  - Computer Science and Math
  - Category Theory
permalink: yoneda/
date: 2024-04-08 02:38:20
updated: 2024-04-14 1:04:42
---


# 米田视角 → 米田嵌入 → 米田引理

*注：本标题为译者添加。*
原作者： [Tai-Danae Bradley](https://www.math3ma.com/about)
翻译： CubicY
部分图片与启发来源：[Bartosz Milewski](https://bartoszmilewski.com/2015/09/01/the-yoneda-lemma/trackback/)

## <a name="prelude"></a>译者序

米田引理(Yoneda Lemma)往往是新手在学习范畴论时遇到的第一个大障碍，犹如五十音图之于日语初学者。然而就译者本人看来，其难度并非来自于思维上的复杂，而更多来自于符号滥用（让人不禁想问：数学和计算机科学最大难题是否都是 Naming）。尽管一些概念嵌套层数过多对思考确实造成了阻碍，但是通过无歧义的符号与生动的图示可以大大降低心智负担。本系列作者在这两点上都做得十分出色，故译者产生了将其译为中文的想法。一些个人的理解将以“译者注”的形式放置在文章各处。尽管如此，在此仍然强烈推荐读者阅读原文。

为了避免歧义，将本文使用的部分记号与约定列举在此：
- 我们说“在X意义下等价”(up to X)有如下几种等价表述：
  - 对象间如果只相差一个X关系就可以转换为对方，那么我们认为他们是相同的
  - 对象间最多相差一个X的关系
  - 不考虑X带来的差异时相同
  - （如果X是一种等价关系）按照X关系划分不同等价类，如果对象在一个等价类里则视为相同
  例如：“在同构意义下等价”指的是在这种视角下，如果两个不同的对象至多只是相差一个同构关系，那么我们认为他们就是一个对象；"up to permutations"可理解为“不考虑置换带来的差异时相同”；5, 8, 11在模3的意义下等价。
- $ \mathbf{Set}^{ \mathbf{C} } $ 表示一个函子范畴，其中的函子是 $ \mathbf{C} \rightarrow { \mathbf{Set} } $. 为了方便书写（$\mathbf{C}$ 所指代的范畴没有什么歧义时），本文有时会如此简写：
  $$ \mathbf{Set} := \mathbf{Set}^{ \mathbf{C} } 
  $$ $$ \mathbf{Set}^{op} := \mathbf{Set}^{ \mathbf{C}^{op} } $$ 这一记号有时在其他文章中记作 $\mathbf{Funct}(\mathbf{C},\mathbf{Set})$
- 对于范畴C中的态射我们记作 $ \text{hom}_\mathbf{C}(X, Y) $，类似地，对 $\mathbf{C}$ 没有歧义时简写为$ \text{hom}(X, Y) $
  - 形如 $ \text{hom}(A, -): \mathbf{Set}^{ \mathbf{C} }$ 的记号代表一个函子，它将A发送到“所有从A出发的态射构成的集合”；这一记号有时在其他文章中记作 $h_A$
  - 形如 $ \text{hom}(-, A): \mathbf{Set}^{ \mathbf{C}^{op} } $ 的记号代表一个函子，它将A发送到“所有以A为目标的态射构成的集合”，这一记号有时在其他文章中记作 $h^A$
  - 形如 $ \text{hom}(-, -):  \mathbf{C} \rightarrow \mathbf{Set}^{ \mathbf{C}^{op} } 或 \mathbf{C}^{op} \rightarrow \mathbf{Set}^{ \mathbf{C} } $ 的记号将C对应到一个自然变换范畴，对于每个Y(或X)有$ h^Y(X)=h_X(Y)=\text{hom}(X, Y) $是新范畴中的对象，所有Y(或X)的取值对应的新对象构成新范畴的全部元素，这一记号有时在其他文章中记作 $h^{(-)}$ 或 $h$ 。根据本文对Yoneda嵌入的定义，本文中按 $ \mathbf{C} \rightarrow \mathbf{Set}^{ \mathbf{C}^{op} }$阐释。
  - $h^{(-)}(f)_X$ 意为对于C中的态射 $f: A \rightarrow A'$，$h^{(-)}(f)$ 是一个自然变换，当其限制在每个分量(component) $ X $ 上时有 $ \text{hom}(X, A) \rightarrow \text{hom}(X, A')$
- 许多文章会直接将 co-Yoneda引理 称作 Yoneda引理，co-Yoneda嵌入 称作 Yoneda嵌入 。为了避免歧义，本文规定如下（但本质上没有区别，不需要过于担心带来会混乱）：
  - Yoneda嵌入： $\mathscr{Y}: \mathbf{C} \rightarrow \mathbf{Set}^{ \mathbf{C}^{op} }$
  - Yoneda引理： $\mathbf{Nat}(\text{hom}(−,X),F)\cong F(X).$
  - co-Yoneda嵌入： $\mathscr{co-Y}: \mathbf{C}^{op} \rightarrow \mathbf{Set}^{ \mathbf{C} }$
  - co-Yoneda引理： $\mathbf{Nat}(\text{hom}(X,-),F)\cong F(X).$
译者推荐尽量使用英文的术语（例如，使用"presheaf"而非“预层”这一称呼）以减少歧义。如果读者愿意，也可以使用平假名$よ$来命名Yoneda嵌入（可能比较有趣吧）。

这里有几条锦囊也许能帮你快速抓住核心，如有错漏欢迎斧正！如果在重新阅读时有晕头转向之感（尤其是第三部分），不妨试着阅读一番（**如果是初次阅读不妨跳过，可以看完文章再回头看**）：

- **笔者认为，对于Yoneda引理（更准确的说，是推论 1）最精炼的总结是：“人是一切社会关系的总和。”**
- > 你在一个粒子加速器工作。你想要理解某个粒子，你所能做的就是将其他粒子甩到它上面，看看会发生什么。如果你了解你的神秘粒子对所有可能的测试粒子种类以及所有可能的测试能量大小是如何做出响应的，那么你就已经知道关于你的神秘粒子的一切了。
  > You work at a particle accelerator. You want to understand some particle. All you can do is throw other particles at it and see what happens. If you understand how your mystery particle responds to all possible test particles at all possible test energies, then you know everything there is to know about your mystery particle.
  *Source: [https://mathoverflow.net/a/3223](https://mathoverflow.net/a/3223)*
- 熟悉群论的读者可能会想到凯莱定理(Cayley's theorem)，也即所有群 G 同构于在 G 上的对称群(即 Sym(G))的子群。事实上，可以认为这个定理是米田引理的一个特例。文末会提到如何构造一个只有单个对象、态射对应群元素的范畴并在其上应用Yoneda引理以证明之。
  - 在群作用语言下的等效表述：如果群作用子 $\sigma_g: G\rightarrow \text{Aut}_{\text{set} }(S)$ (也就是每个元素对应到一个自同构上)是单射的（也就是单同态），群作用 $G\times S\rightarrow S$在某个集合 S（S 可以来自 G 本身）上是忠实的
  - 为什么**这样的**群作用是忠实的(faithful)？只需要关注单位元在集合 S 中对应的元素 u：此时在不同的自同构下 u 一定被发送(send)到不同元素上。（参见[Chapter 7: Group actions, symmetric group and Cayley’s theorem | Essence of Group Theory](https://www.youtube.com/watch?v=sNX3txN9zc4&)）
  - 接下来你会看到米田引理证明过程中也是通过关注恒等映射这样的“单位元”被发送至何处（这单位元其实如同一个“凝结核”，确定了其被发送至何方后将引发连锁反应，使得所有其他对象的去向被一一确定，后文会详细阐述），从而得出米田嵌入的忠实性。这不是巧合。
- 译者这里给出一个例子帮助读者梳理第一推论想表达的：假设现有Int(I), Double(D), Float(F)三种类型，这个定理告诉我们Float类型的意义就是如下几句陈述的总和：
  - 对于任何F->I的函数，我们可以：
    - 将f: F->I转化为f': I->I
    - 将f: F->D转化为f': I->D
    - 将f: F->F转化为f': I->F
    以及
    - 将g: I->I转化为g': F->I
    - 将g: I->D转化为g': F->D
    - 将g: I->F转化为g': F->F
    六种转化的具体方式确定下来。
  - 对于任何F->D的函数，我们可以...
  - 对于任何F->F的函数，我们可以...
  （如果你觉得怪怪的，那也许是因为协变与逆变的小问题）
  例如，对于函数downcast: Double->Float，我们能将取整函数floorF: Float->Int转为取整函数floorD: Double->Int.
  所以对于如下三个代码片段，他们是等价的，可以互相转换：

  ```java
  //A
  Double a
  Int c = floorD(a)
  ```

  ```java
  //B
  Double a
  Float b = downcast(a)
  Int c = floorF(b)
  ```

  ```java
  //C
  Double a
  Int c = floorF(downcast(a))
  ```

  B与C显然是等价的。
  A转B是一种扩展，这种扩展的意义是：Float这种类型可以通过在Double和Int间定义这些函数的转换关系创造出来。
  B转A是一种缩减，这种缩减的意义是：我们不需要Float类型的具体中间值，也能运算出结果！
  所以Float这一类型，或者说任何特定类型的本质都是在定义“涉及该特定类型的类型转换”之间的一种特定变换关系（例如Float类型将指导如何复合floorF和downcast得到floorD）.
  而这种“涉及该特定类型的类型转换”之间的变换关系，和“各种类型的类型转换”是同构的。而众所周知范畴论从不要求相等，只要求同构，
  **所以一种类型即代表了一种对应该类型的特定变换关系，这种对应该类型的特定变换关系即定义了该类型。**
- （这里是基于co-Yoneda的叙述，但反正本质上没有区别）一种[来自Bartosz Milewski的理解](https://bartoszmilewski.com/2015/09/01/the-yoneda-lemma/trackback/)：任何数据类型 T 都可以被一个函数 f 所取代。f 接受一个T的“处理器”handler（这个handler会把T转成目标类型） 并执行其余计算——即是续延（Continuation）。
  - Haskell表述： ```forall x . (t -> x) -> F x ≅ F t```
  - 我们可以把函子看成一种广义的容器（参见：[Functors are Containers](https://bartoszmilewski.com/2014/01/14/functors-are-containers/trackback/)），所以左边就是把t转成x，然后变成F x这一步只是把x装起来变成可以装x类型的容器而已（我们并不关心这个F对x做了什么，我们关心的是“F对x的确做了些什么”这个事实本身——所以把F x当作一个容器好了，只需要把x放进去然后盖上盖子，不去理睬里面发生了什么）。假设容器一开始只能装x类型，但是如果对所有可能的类型x，“装入t类型为F t”这事都能通过handler: t->x转化为“装入x类型为F x”，那么这个容器事实上就是可以装t类型的。
  - 译者的理解是，如果不把`F x`看作容器，而是看成类型构造器，那么可以这么解释：基于t构造新类型的实质，就是需要从每个t类型值构造新的F t类型值（所以在这种视角下，没有具体的转换细节，我们只知道有一个新类型F t带来新的关系，这些新的关系从旧关系导出）。对任意可能的类型x，每个t类型值都对应一种自己的*转换方式*，使得各个handler: t->x（某种意义上是“关系”，之所以说是“某种意义”是因为这只是单向的，还有反方向）分别转化为一个值v，v的类型是基于x构造的新类型F x。 **或者说，每个类型x都可以要求一个能把t值转化为x值的handler，而后用构造器F构造新类型时，就可以从handler这些旧有关系导出新类型F t与其他新类型F x之间的新关系；因为这些handler把类型t的运算细节“包办”了。** 对于任意类型x，任意的handler都有这种*转换方式*（共同构成每个x的自然变换）。因此，应用类型构造器F时，类型t的每个值都会指导形如(t->...)的handler该如何转换为(F t->F...)，而这些自然变换构成了t类型存在的全部意义。所有和t有关的运算都可以视为一种类型构造，因此类型t的意义只有在运算中才会得到显现，一个不参与运算（不构造新类型）的类型没有任何存在价值，这符合直觉。

- 分别取定F和A，会发现Yoneda引理这个同构关系对于F、t都是自然同构（参见[【范畴论】第5讲，Yoneda引理](https://www.bilibili.com/video/BV1AN4y1V7mH)）

每日更新！

## <a name="1"></a>1/3：米田视角(The Yoneda Perspective)

*本部分翻译自 [https://www.math3ma.com/blog/the-yoneda-perspective](https://www.math3ma.com/blog/the-yoneda-perspective)*

[Dan Piponi](http://blog.sigfpe.com/2006/11/yoneda-lemma.html?m=1) 说这是“数学界细碎(trivial)玩意中最难的”；[nLab](https://ncatlab.org/nlab/show/Yoneda+lemma) 认为它是“初等的，深刻的，核心的”；而 [Emily Riehl](http://www.math.jhu.edu/~eriehl/context.pdf) 则提名它为“可以说是范畴论中最重要的结果”。然而，正如 [Tom Leinster](http://www.maths.ed.ac.uk/~tl/categories/yoneda.ps) 所指出的，“许多人觉得它非常令人困惑。”
他们谈论的是什么？
**正是米田引理(Yoneda lemma)。**

![yoneda](yoneda.jpg)

“但……”你问道，“米田引理是啥？如果这还只是一个引理，那么……天哪……那定理又是什么？”

在回答问题之前，我想通过“悠闲地漫步”(leisurely stroll)于其中的两个推论来诱导出这个结果。实际上，话说在前头，我没有在范畴学领域的任何部分“漫步”的资质（译者注：作者的自谦）。的确，有资格为我们展示 Yoneda 引理这颗闪闪发光的精切钻石的，按理说是范畴学家们，而我并不是范畴学家（译者注：依然是自谦）。不过，我就是喜欢跟你们讲述一些我喜欢的事物，而Yoneda引理恰恰是我喜欢的事物。因此，我们还是在这相遇了！

现在，我可以简单地告诉你在我看来这些推论是什么，也可以简单地把引理甩给你。但正如 [Joseph Maher](http://www.math.csi.cuny.edu/~maher/) 曾说的：

> 数学是喜剧的反面，是反笑话(anti-joke)\*。我们先告诉你笑点，然后费力地向你解释为什么这个笑点是对的。

所以，首先我要告诉你们一个笑点——关于 Yoneda 引理两个推论的总结。[我之前在这个博客上提到过它](http://www.math3ma.com/mathema/2016/10/6/the-sierpinski-space-and-its-special-property)，但我在这个系列文章中的目标是将这个总结锚定在一个更正规、更严谨的数学基础上——也就是费力地解释为什么它是正确的笑点。它是这样一个概念：
> 数学对象完全由它们与其他对象的关系所决定。

让我们称其为*Yoneda视角*。简而言之，它表明如果你想理解对象（集合、群、拓扑空间等等），那么用[Barry Mazur](http://www.math.harvard.edu/~mazur/preprints/when_is_one.pdf)的话来说，你会想要理解“它们与同类的、所有其他对象之间的关系网”。我们已经在几篇帖子中探讨了该理念——[The Most Obvious Secret in Mathematics](http://www.math3ma.com/mathema/2016/9/12/the-most-obvious-secret-in-mathematics)和[The Sierpinski Space and its Special Property](http://www.math3ma.com/mathema/2016/10/6/the-sierpinski-space-and-its-special-property)——所以我就不在这详细展开了。（如果你还没看过，请务必去看一看！你可以认为这些帖子是本文的前传。）但是我想提一嘴，Yoneda 视角启发了一种观点，一些数学家——还有这个博客\*\*也越来越多地采纳了这种观点，也即
> 数学对象的性质比它的定义更重要。

为什么要采用这种观点？因为很容易脱口而出地背诵那些定义：笛卡尔乘积是...，集合生成的自由群是...，商拓扑是...然而，定义并不总能说明一切。积是生来就带有*到达它*或*源于它*的映射的吗？如果一个自由群的生成集位于另一个群内，这两个群在某种程度上存在联系吗？商空间上的连续函数是什么样子的？这些问题都研究对象的属性——而正是属性定义了对象的特征。一旦我们拓宽视角，从对象所存在的范畴这一视角检视对象，答案就会水落石出。

我想表达的是什么含义呢？

假设你想探索对象 X 的性质。现在想象一下，把自己带入另一个对象 Y 的视角。然后问：“从 Y 的视点(vantage point)看起来，X 是什么样的？” 然后现在走到另一个对象 Z 旁，同样设身身处地问：“现在呢？从 Z 这里看，X 是什么样？”一直这么做下去，直到你站在范畴中每个对象的角度都剖析了 X 一遍。最后，你就会收集到大量关于 X 的信息。

就像[这个片段](https://www.youtube.com/watch?v=vwF7AHTQaoc)。从一侧看，片段里的这座雕塑像一头大象；从另一个角度看，它又看起来像两只长颈鹿。但这两个角度都没有给出对雕塑完整的描述。为了真正理解这座雕塑，我们应该从所有可能的视点来观看它。范畴理论也有同样的观点：更多的视点提供了更多的信息。注意，划重点：Yoneda 引理说明

> **所有**的视点会给出**所有**的信息。

这就是上文所述 Yoneda视角 的本质，也是思维清晰的数学家如此重视态射、交换图、普遍性质等的原因之一（不知道你注意到了吗？）。这些都是关于关系的！当然，这只是一种直觉。实际数学形式是什么样的？那两个推论是什么？我所说的“…物体完全被…决定”究竟是什么意思？什么是“两个物体之间的关系”？我们如何看待“范畴中*所有对象的视点*”？Yoneda 引理*具体*是如何捕捉到这一点的？

这些是我们[下周](#2)要回答的问题。
顺带一提，想复习一下范畴论吗？可以从这几个地方开始：
[What is Category Theory, Anyway?](http://www.math3ma.com/mathema/2017/1/17/what-is-category-theory-anyway)
[What's a Category?](http://www.math3ma.com/mathema/2017/1/23/what-is-a-category)
[What's a Functor?](http://www.math3ma.com/mathema/2017/1/31/what-is-a-functor-part-1)
[What's a Natural Transformation?](http://www.math3ma.com/mathema/2017/2/6/what-is-a-natural-transformation)

---

\*: 所以数学家是 cocomedian? (那[comathematician](https://www.instagram.com/p/BLwhFLFgTbE/?taken-by=math3ma)是什么?)（译者注：co-这一词根前缀在范畴论中被大量应用，用来表达“对偶的”“反方向的”等意味。哇，这岂不是可以说明 coconut 其实就是 nut？）
\*\*: 参见 [Exhibit A](http://www.math3ma.com/mathema/2017/1/17/what-is-category-theory-anyway) 和 [Exhibit B](http://www.math3ma.com/mathema/2016/9/12/the-most-obvious-secret-in-mathematics) 和 [Exhibit C](http://www.math3ma.com/mathema/2016/10/6/the-sierpinski-space-and-its-special-property) 和 [Exhibit D](https://twitter.com/math3ma/status/894700378792611840) 和 [Exhibit E](http://www.math3ma.com/mathema/2017/2/22/crumbs), 作为例证

## 2/3：<a name="2"></a>米田嵌入(The Yoneda Embedding)

*本部分翻译自 [https://www.math3ma.com/blog/the-yoneda-embedding](https://www.math3ma.com/blog/the-yoneda-embedding)*

[上周](#1)我们开始探讨Yoneda引理。然而，我们并没有直接陈述Yoneda引理本身（因为缺乏动机），而是“悠闲地漫步”于其推论之一———Yoneda视角，也就是：

> 一个对象完全由其与其他对象的“关系”确定，

也就是：

> 从范畴中的每个对象的视角看，对象是什么“样子”。

但这让我们不禁想问，“实际数学形式是什么样的？那两个推论是什么推论？”在这篇文章中，我们会尽量找出答案。首先，让我们将这三个抽象叙述背后的具体数学概念具体化：

- "...两个对象之间的*关系*..."
- "...范畴中*每个对象的视点*..."
- "...一个对象*完全*由...*表征(characterized)*..."

我们将逐一解密这些叙述方式。接下来，设 Set 表示集合范畴，C 为任意范畴。

---

> “两个对象之间的关系”是指一个态射(morphism)。

假如在 C 中，两个对象 X 和 Y 它们之间存在一个态射，那么我们就说他们“存在关系”。例如，如果X是一个具有[离散拓扑](http://www.math3ma.com/the-back-pocket/2016/8/26/comparing-topologies)的拓扑空间，那么从 X 到 Y 就有很多关系——也就是有很多从 X 到 Y 的连续函数——对于任何空间Y。事实上，从一个离散空间出发的*所有*映射都是连续的。

![](discrete.jpg)

而反观域范畴，其中的对象之间的关系就非常少了——因为在具有不同特征(characteristics)的域之间不存在域同态！

---

> "...范畴中*每个对象的视点*..." 编码在一个函子(functor)中

要“从 C 中所有对象的视点”分析一个对象 X ，我们需要一种方法来跟踪 X 与 C 中所有对象的关系网。这个“关系网”指的就是所有*从Z到X*和*从X到Z*的态射集合，也就是这些集合：
$$\text{hom}(Z,X)\ 与\ \text{hom}(X,Z)\ \text{,对于范畴}\mathbf{C}\text{中的任意对象}Z\text{.}$$
注意到，我们希望C中的*每个*Z都对应一个不同的集合。一个可行的办法是通过[反变函子(contravariant functor)](https://www.math3ma.com/blog/what-is-a-functor-part-2) $\text{hom}(-,X): \mathbf{C}^{op}\rightarrow \mathbf{Set}$，它将Z发送到集合$\text{hom}(Z,X)$，并将态射$f:Z\rightarrow W$发送到其[拉回(pullback)](https://www.math3ma.com/blog/the-sierpinski-space-and-its-special-property) $f^*$，$f^*$则定义为“与f前复合(precomposing)”这一操作（译者注：前复合、后复合的解释参见[What are post-composition and pre-composition in category theory?](https://math.stackexchange.com/questions/3889401/what-are-post-composition-and-pre-composition-in-category-theory)，同时请注意区分 $f^*$ 与 $f_*$ 这两个记号以免混淆前复合与后复合）。类似的，对于C中的所有Z，集合$\text{hom}(X,Z)$都在[协变函子(covariant functor)](https://www.math3ma.com/blog/what-is-a-functor-part-1) $\text{hom}(X,-):\mathbf{C}\rightarrow\mathbf{Set}$的像(image)中。

![](network.jpg)

---

> “一个对象‘完全由...确定’”意味着你知道它是在同构的意义下等价(up to isomorphism)的。

说“一个对象X完全由...确定”意味着X作为由省略号处内容所表征的对象，其[在同构的意义下](https://www.math3ma.com/blog/up-to-isomorphism)是唯一的。比如在本部分第一段中，省略号处对应的是“它们与其他对象的关系”。（尽管通常来说省略号处是一个[普适性质](https://jeremykun.com/2013/05/24/universal-properties/)。这很正常。根据[Yoneda视角](#1)，这两种填在省略号处的内容没啥区别！（译者注：确实不知道怎么翻译该句"the two addendums go hand in hand"，此处的两种addendums可能指的是追加在省略号处的可以是“对象带有的普适性质/对象的定义蕴含的属性”，也可以是上文的“对象与其他对象的关系”，而这两种答案本质上是一样的））重点在于，如果Y与X以*相同*的方式与C中的*所有*其他对象联系——也就是说，如果从整个范畴的视点看，Y看起来就像X一样——那么X和Y一定是同构的。

例如，假设X和Y是拓扑空间，令$\cdot$表示单点空间，$I$和$S^1$表示单位区间和圆。那么我们有，

- X和Y具有相同的势(cardinality)当且仅当 $\text{hom}(\bullet,X) \cong \text{hom}(\bullet,Y)$
- X和Y具有相同的路径空间(path space)当且仅当 $\text{hom}(I,X) \cong \text{hom}(I,Y)$
- X和Y具有相同的（自由）[环空间(loop space)](https://en.wikipedia.org/wiki/Loop_space)当且仅当 $\text{hom}(S^1,X) \cong \text{hom}(S^1,Y)$

![](probe.jpg)

根据路径空间和环空间的定义，后两点成立。（它们分别是从$I$和$S^1$到$X$的所有连续函数的空间。）而至于第一点，只是因为一个形如$\bullet \to X$的映射其实就是在$X$中选择一个点。我们甚至可以说$\bullet \to X$是$X$的一个“$\bullet$塑造的元素”。类似地，一个路径$I \to X$可以被看作是$X$的一个“$I$塑造的元素”，而一个环$S^1 \to X$则是一个“$S^1$塑造的元素。” 本质上，我们使用$\bullet$、$I$和$S^1$来探测(probe)$X$和$Y$. 为了洞悉全貌，我们必须用*所有*空间探测它们——也即从*所有*空间的视点观察它们。

---

在确立这些数学形式之后，“一个对象X完全由X与其他对象的关系所确定”这个说法现在可以明确为两个要点：
> **要点 #1.** 关于X的一切我们需要知道的东西都被编码在$\text{hom}(-,X)$中。实际上，对象X**表示**了函子$\text{hom}(-,X)$. 

> **要点 #2.** 当且仅当它们所表示的函子$\text{hom}(-,X)$和$\text{hom}(-,Y)$同构时，X和Y才是同构的。

我们现在先考虑第一点，之后再重新审视第二点。所以我们真的能够像第一点说的那样用一个函子来确定一个对象吗？显然存在一个指派
$$X \mapsto \text{hom}(-, X)$$
这是因为范畴 $\mathbf{C}$ 中的任何对象 $X$ 都对应于一个函子 $\text{hom}(-,X)$，那么这个函子存在于... 呃... 在哪呢？$\text{hom}(-,X)$ 位于哪里？它也存在于一个范畴中！正如我们很久以前提到的，存在一个范畴 $\mathbf{Set}^{ \mathbf{C}^{op} }$ ，它的对象是从 $\mathbf{C}^{op}$ 到 $\mathbf{Set}$ 的函子，而其态射是自然变换。因此，存在一个函子 $\mathscr{Y}: \mathbf{C} \rightarrow \mathbf{Set}^{ \mathbf{C}^{op} }$，它将对象 $X$ 映射到 $\text{hom}(-, X)$，将一个态射 $f: X \rightarrow Y$ 映射到自然变换 $f_*: \text{hom}(-, X) \rightarrow \text{hom}(-, Y)$（你自己应该能验证）。这个自然变换的每个分量都是通过与 $f$ 后合成(postcomposing)得到的。在范畴 $\mathbf{Set}^{ \mathbf{C}^{op} }$ 中的函子被称为预层(presheaf)，我们感兴趣的预层，即形式为 $\text{hom}(-,X)$ 的预层，被称为**可表示函子**(representable functors)。但是我们需要证明这个命名是否合理： $X$ 真正地、忠实地且最大限度地*表示*函子 $\text{hom}(-, X)$ 吗？

答案是肯定的，但有一个条件：当 $\mathscr{Y}$ 将 $X$ 发送到预层范畴时，它必须*保留* $X$ 与 $\mathbf{C}$ 中对象之间的关系。换句话说，对于 $X$ 和 $Y$ 之间的每个关系，应恰好存在*一个*关系（自然变换）在$\text{hom}(-, X)$ 和 $\text{hom}(-, Y)$ 之间。更正式地说，对于 $\mathbf{C}$ 中的每对 $X$ 和 $Y$，由 $f \mapsto f_*$ 定义的函数 
$$\text{hom}(X, Y) \rightarrow \mathbf{Nat}(\text{hom}(-, X), \text{hom}(-, Y))$$
应该是一个双射。（记号 $\mathbf{Nat}(-,-)$ 表示从 [留空] 到 [留空] 的自然变换集合（译者注：留空即为在该处使用 $-$ 处填入的对象填空）） 。如果 $\mathscr{Y}$ 满足这个条件，则称其为全忠实(fully faithful)*的并且称它将范畴 $\mathbf{C}$ **嵌入**到 $Set^{C^{op} }$ 中。

![](embed.jpg)

但问题来了：

这个把 $f$ 映射到 $f_*$ 的函数
$$\text{hom}(X, Y) \rightarrow \mathbf{Nat}(\text{hom}(-, X), \text{hom}(-, Y))$$
真的是一个双射吗？

单射性是显然的：如果 $f, g: X \rightarrow Y$ 是不同的态射，则它们的推出(pushforward) $f_*$ 和 $g_*$ 显然是不同的。但，满射性呢？给定任意的自然变换 $\eta: \text{hom}(-, X) \rightarrow \text{hom}(-, Y)$，是否真的存在一个态射 $f: X \rightarrow Y$ 使得 $\eta = f_*$？也就是说，
***每个*可表示函子之间的自然变换，是否都来自于表示了它们的对象之间的态射？**

可能存在大量的自然变换 $\text{hom}(-, X)$ 和 $\text{hom}(-, Y)$！似乎不能指望其中的每个都来自于一个 $X \rightarrow Y$ 的态射。

然而，奇怪的是，真就是这样的！

> 还真是！

对于每个自然变换 $\eta: \text{hom}(-, X) \rightarrow \text{hom}(-, Y)$，存在唯一的态射 $f: X \rightarrow Y$，使得 $\eta = f_*$. 

而**这**正是Yoneda引理直接导出的。

结果是，$\mathscr{Y}$ 将 $\mathbf{C}$ 全忠实地嵌入到 $\mathbf{Set}^{ \mathbf{C}^{op} }$ 中。（这是上面提到的“要点 #1”的形式化表述。）因此，$\mathscr{Y}$ 被称为**Yoneda嵌入**。

但是，态射 $X \rightarrow Y$ 与自然变换 $\text{hom}(-, X) \rightarrow \text{hom}(-, Y)$ 之间是双射的这一事实仅仅是Yoneda引理导出的一个*结果*。正如我们[下周](#3)将看到的，Yoneda引理其实是一个更强的结论！它是关于**任意**函子 $F: \mathbf{C}^{op} \rightarrow \mathbf{Set}$ 的自然变换 $\text{hom}(-, X) \rightarrow F$ 的结论。

---

\* 更一般地，对于任意函子 $F: \mathbf{C} \rightarrow \mathbf{D}$，存在一个函数 $\text{hom}\_{\mathbf{C} }(X, Y) \rightarrow \text{hom}_{\mathbf{D} }(F(X), F(Y))$，由 $f \mapsto F(f)$ 给出。如果这个映射是单射，我们说 $F$ 忠实(faithful)；如果它是满射，我们说 $F$ 全射(full)；如果它是双射，我们说 $F$ 全忠实(fully faithful)（译者注：相当于此时我们只关心态射在F作用下的变化情况）。以下是一个方便的[表格](http://www.math3ma.com/mathema/2017/7/30/naming-functors)，用于命名其他类型的函子。

(译者注：我将这个有趣的表格直接附在此处↓)
![](naming.jpg)

## 3/3：<a name="3"></a>米田引理(The Yoneda Lemma)

*本部分翻译自 [https://www.math3ma.com/blog/the-yoneda-lemma](https://www.math3ma.com/blog/the-yoneda-lemma)*

欢迎来到关于Yoneda引理的第三部分，也是最后一部分！在过去的几周里，我们慢慢揭示了[Yoneda视角](#1)背后的数学，即一个对象完全由其与其他对象的关系所决定这一范畴论里的原则(maxim)。

[上周](#2)，我们将这一原则分成了两点：

> **要点 #1.** 关于X的一切我们需要知道的东西都被编码在$\text{hom}(-,X)$中。实际上，对象X**表示**了函子$\text{hom}(-,X)$. 

> **要点 #2.** 当且仅当它们所表示的函子$\text{hom}(-,X)$和$\text{hom}(-,Y)$同构时，X和Y才是同构的。

注意到第一点其实是在以不那么严谨的方式表达Yoneda嵌入 $\mathscr{Y}: \mathbf{C} \rightarrow \mathbf{Set}^{op}$ （它将一个对象 $X$ 映射到函子 $\text{hom}(-,X)$）是全忠实的。换句话说，从 $\text{hom}(X,Y)$（一组态射的集合）到 $\mathbf{Nat}(\text{hom}(-,X), \text{hom}(-,Y))$（一组自然变换的集合）的函数将 $f$ 映射到 $f_*$，而这是一个双射。（此处 $f_*$ 将任一个态射 $g:Z\rightarrow X$ 映射到这样的复合： $f\circ g:Z\rightarrow Y$（译者注：也就是与f后复合））。

这意味着对于每个 $f:X\rightarrow Y$，存在*恰好一个*由 $f$ 自身构造出(cooked up)的自然变换 $\text{hom}(-,X)\rightarrow \text{hom}(-,Y)$. 反过来，如果 $\eta:\text{hom}(-,X)\rightarrow \text{hom}(-,Y)$ 是任一自然变换，则存在*恰好一个*由 $\eta$ 自身得到的态射 $X\rightarrow Y$. 

![](bijection.jpg)

现在，有一个简单但至关重要的观察发现：注意到集合 $\text{hom}(X,Y)$ 位于函子 $\text{hom}(-,Y):\mathbf{C}^{op} \rightarrow \mathbf{Set}$ 的像中！"但是，" 你会问， "这重要在哪呢？"

因为，这让我们可以这样重新表述上周得到的结果：

> 对于范畴 $\mathbf{C}$ 中的任意对象 $X$，自然变换 $\text{hom}(-,X) \rightarrow \text{hom}(-,Y)$ 与集合 $\text{hom}(X,Y)$ 中的元素双射（一一对应）。

相当简洁，对吧？你知道真正让人惊叹的是什么吗？这个结论不仅对于形如 $\text{hom}(-,Y)$ 的函子成立；它对于从 $\mathbf{C}^{op}$ 到 $\mathbf{Set}$ 的**所有**函子都成立。

**所有！**

*这*就是所谓Yoneda引理。

### Yoneda引理
> 对于任意函子 $F: \mathbf{C}^{op} \rightarrow \mathbf{Set}$ 和范畴 $\mathbf{C}$ 中的任意对象 $X$，自然变换 $\text{hom}(-,X) \rightarrow F$ 与集合 $F(X)$ 中的元素一一对应。也就是说， 
> $$ \mathbf{Nat}(\text{hom}(-,X), F) \cong F(X) $$

你明白这玩意的重要性了吗？自然变换 $\text{hom}(-,X) \rightarrow F$ 的集合似乎可能是超\~级\~大\~的，一大片浓密的未知、难以驯服和没用的杂草丛。"但是，" Yoneda引理告诉我们，"*不是这样的！*" 把F限制在特定对象 $X$ 上应用会得到一个集合F(X)，而*只有*从该集合中元素“烹饪”而成（构造而来）的自然变换，才是真实存在的自然变换。

那么这个“烹饪”用到的食谱是什么呢？

给定 $c \in F(X)$，定义 $\eta: \text{hom}(-,X) \rightarrow F$，通过声明 $\eta _Y: \text{hom}(Y,X) \rightarrow F(Y)$ 为一个将映射 $g: Y \rightarrow X$ 发送到在 $F(Y)$ 中的元素 $Fg(c)$的态射，其中 $Fg$ 表示在 $F$ 作用下映射 $g$ 对应的像。

![](F_bijection.jpg)

反过来说，任意自然变换 $\eta: \text{hom}(-,X) \rightarrow F$ 只取决于 $F(X)$ 中的一个元素，即 $\eta_X(\text{id}_X)$. 这里 $\eta_X$ 表示态射 $\text{hom}(X,X) \rightarrow F(X)$，而 $\text{id}_X$ 是 $X$ 上的单位态射（译者注：单位态射即为恒等态射 $\text{id}_X \in \text{hom}(X,X)$ ，这里即是序言部分所说“单位态射”起到“凝结核”效果的体现：虽然单位态射是我们唯一已知一定存在的态射，但其他所有态射的去向都被这个单位态射的去向固定了）。

我们还是要检查这些指派(assignment)（译者注：应该指的是上面两种对偶的说法）是否真的彼此互为逆（以及 $\eta$ 是否真的是一个自然变换）。但正如Tom Leinster[曾经说过](https://arxiv.org/abs/1612.09375)的，“理解了问题就等于快知道答案了……只有一种向前进的办法。”

Yoneda引理的一个直接推论是[上周讨论的内容](#2)，这里再次强调：

### Yoneda引理第一推论（要点＃1）

> **推论1：** Yoneda嵌入 $\mathscr{Y} : \mathbf{C} \rightarrow \mathbf{Set}^{\mathbf{C} }$ 是全忠实的。

映射 $\text{hom}(X,Y) \mapsto \text{Nat}(\text{hom}(-,X), \text{hom}(-,Y))$ 的单射性是显而易见的（如果 $f \neq g$，则 $f_* \neq g_*$）。Yoneda引理则给出满射性。为了证明这一点，将 $F = \text{hom}(-,Y)$ 代入引理的陈述中，那么我们得到一个双射
$$ \mathbf{Nat}(\text{hom}(-,X), \text{hom}(-,Y)) \cong \text{hom}(X,Y) $$
现在设 $\eta : \text{hom}(-,X) \rightarrow \text{hom}(-,Y)$ 是任意的自然变换。我们需要证明存在一个态射 $f : X \rightarrow Y$ 使得 $\eta = f_*$. 这里的" $\eta = f_*$ " 意味着对于范畴 $\mathbf{C}$ 中的每个对象 $W$ 和任何映射 $g : W \rightarrow X$，都有
$$\eta_W(g) = f \circ g$$
那么映射 $f$ 应该是什么呢？实际上只有一种选择！根据Yoneda引理，$\eta$ 恰好完全取决于一个态射 $\eta_X(\text{id}_X) : X \rightarrow Y$. 所以我们就选这个！也就是说，
$$\text{令} f := \eta_X(\text{id}_X)$$
现在我们只需要验证这样是否可行。但根据定义，这样的陈述“$\eta$ 是一个自然变换”意味着对于范畴 $\mathbf{C}$ 中的任意一对对象 $Z$、$W$ 和任何映射 $g : W \rightarrow Z$，我们有等式 $\eta_W \circ g^* = g^* \circ \eta_Z$，也就是说，对于任何 $h : Z \rightarrow X$，都有
$$\eta_W(h \circ g) = \eta_Z(h) \circ g$$

![](natural.jpg)


由于这对于*所有* $Z$ 和 $h$ 都成立，所以对于特例： $Z=X$ 且 $h=\text{id}\_X \in \text{hom}(X,X)$ 仍然成立。
现在，自然性条件给出的正是我们想要的：对于所有 $g : W \rightarrow X$，都有 $\eta_W(g) = fg$. 因此 $\eta = f_*$. 

因此，一个对象 $X$ 本质上(effectually)与其可表示函子等效。到目前为止，我们总是专注于反变函子 $\text{hom}(-,X)$，但其实存在Yoneda嵌入的反变版本（以及对应于*协变*函子版本 $F$ 的Yoneda引理版本）（译者注：即序章部分所述co-Yoneda嵌入与co-Yoneda引理），因此有一个类似的“推论1”，其中 $\text{hom}(-,X)$ 被替换为 $\text{hom}(X,-)$. 但本质是相同的——$X$ 是由其与其他对象的关系决定的。

更好的是，它是*完全地*由其与其他对象的关系决定的。这个“完全”一词是由提到过的要点二给出的，实际上它就是Yoneda引理的第二推论。

### Yoneda引理第二推论（要点＃2）

> 推论2：$X \cong Y \Leftrightarrow \text{hom}(-,X) \cong \text{hom}(-,Y)$. 

在一个方向（译者注：$\Rightarrow$）上它显然成立，因为 $Y$ 是一个函子：如果 $X$ 和 $Y$ 是同构的，那么 $\text{hom}(-,X)$ 和 $\text{hom}(-,Y)$ 也是同构的；而另一个方向上（译者注：$\Leftarrow$）也成立则是因为 $Y$ 是全忠实的。有个一般性的事实：如果 $F : \mathbf{C} \rightarrow \mathbf{D}$ 是一个全忠实的函子，并且如果 $F(X) \cong F(Y)$，那么 $X \cong Y$. （证明见本部分脚注。*）

我们上周看到了这个推论的一个例子：两个拓扑空间 $X$ 和 $Y$ 有相同的势当且仅当 $\text{hom}(\bullet, X) \cong \text{hom}(\bullet, Y)$；它们有相同的路径空间当且仅当 $\text{hom}(I, X) \cong \text{hom}(I, Y)$；它们有相同的环空间当且仅当 $\text{hom}(S^1, X) \cong \text{hom}(S^1, Y)$；等等。用各种空间探测 $X$ 和 $Y$ 会给出更多信息。用*所有*空间来探测则可以给出*所有*信息。

![](probe2.jpg)

当然，观察从 $X$ *出发*的映射同样能提供有效信息。例如，$X$ 是[连通的](http://www.math3ma.com/mathema/2015/9/7/on-connectedness-intuitively)当且仅当每个映射 $X \rightarrow \{0,1\}$ 都是常值映射。有趣的是，如果我们考虑由Sierpinski拓扑赋予的相同集合 $\{0,1\}$，那么（[正如我们之前所见](http://www.math3ma.com/mathema/2016/10/6/the-sierpinski-space-and-its-special-property)）集合 $\text{hom}(X, \{0,1\})$ 将得以捕获 $X$ 上的全部拓扑。此外，从 $X$ 到[Eilenberg-MacLane](https://en.wikipedia.org/wiki/Eilenberg%E2%80%93MacLane_space)空间的映射在同伦的意义下(up to homotopy)取决于 $X$ 的同调群。另一方面，从 n维球面 到 $X$ 的映射的同伦类塑造了 $X$ 的同伦群。

如此着重强调态射是Yoneda视角（或者说是Yoneda引理）的结果；*一切的关键在于关系！*

![](summary.jpg)

我想再用一个例子收尾本文。Yoneda引理有时被描述为群论中[Cayley定理](https://en.wikipedia.org/wiki/Cayley%27s_theorem)的推广。确实是这样的。我们可以使用Yoneda引理来*证明*Cayley定理。

---

### 一个对Cayley定理的证明

任意群 $G$ 都可以被视为一个范畴，记作 $\mathbf{BG}$，其中有唯一对象 $\bullet$，每个群元素对应一个态射。从 $\mathbf{BG}^{op}$ 到 $\mathbf{Set}$ 的函子 $F$ 被称为右$G$-集合。它将 $\bullet$ 发送到一个集合 $X$，将一个态射（即群元素） $g$ 发送到右乘以 $g$ 的函数 $X \rightarrow X$. 特别地，当 $F = \text{hom}(-,\bullet)$ 时，集合 $\text{hom}(\bullet,\bullet)$ 就是 $G$ ，其自身也被视为右$G$-集合。然后根据Yoneda引理，我们有一个双射：

![](cayley1.jpg)

右边简单来说就是群 $G$ 中的所有元素。但左边呢？首先注意到自然变换 $\text{hom}(-,\bullet) \rightarrow \text{hom}(-,\bullet)$ 就只是 $G$-等变函数 $G \rightarrow G$. 

但它们是哪些 $G$-等变函数呢？根据推论1中的双射，它们是由 $G$ 中的元素构造而成的。简而言之，集合 $\mathbf{Nat}(\text{hom}(\bullet,\bullet), \text{hom}(\bullet,\bullet))$ 就是由所有函数 $f_g : G \rightarrow G$ 构成的，其定义为 $x \mapsto xg$. 而这些*恰好*是由一个固定元素的乘法得到的 $G$ 的自同态！

![](cayley2.jpg)

因此左边实际上是所有 $G$ 上的置换的一个子群。而且，根据Yoneda引理，这个子群同构于群 $G$ 本身。这就是Cayley定理。（编辑于2022年12月：上面的图表有些误导。请参阅[这个StackExchange帖子](https://math.stackexchange.com/questions/4598810/yoneda-lemma-cayley-s-theorem-and-example-with-cyclic-group-mathbbz-6)。）

（译者注：简而言之，在上文构造的范畴中，根据Yoneda引理第一推论，$\text{hom}(-,\bullet) \rightarrow \text{hom}(-,\bullet) \cong \text{hom}(\bullet,\bullet)$. $\text{hom}(-,\bullet) \rightarrow \text{hom}(-,\bullet)$ 由态射 $\text{hom}(\bullet,\bullet) \rightarrow \text{hom}(\bullet,\bullet)$ 决定，而 $\text{hom}(\bullet,\bullet) \cong G$，因此该态射也即G到自身的一个置换。故该自然变换范畴同构于G上的置换群的一个子群，而由于 $\text{hom}(\bullet,\bullet) \cong G$, 第一推论将给出其同构于G.）

---

### 延伸阅读

若想深入了解Yoneda引理，我强烈推荐Tom Leinster的[*Basic Category Theory*](https://www.amazon.com/Category-Cambridge-Studies-Advanced-Mathematics/dp/1107044243)以及他阐释得非常清晰的[*The Yoneda Lemma: What's it All About?*](http://www.maths.ed.ac.uk/~tl/categories/yoneda.ps)。我还推荐Emily Riehl的[*Category Theory in Context*](http://www.math.jhu.edu/~eriehl/context.pdf)（她的例子特别丰富），以及对于一些真正深奥的数学，可以参考[nLab](https://ncatlab.org/nlab/show/Yoneda+lemma)。在这些链接中，你会注意到Yoneda引理还有第三个经典的推论，而我们在本系列中并未涵盖。也许未来的文章中会有！

如果你喜欢“用其他对象探测对象”的想法，你会很高兴知道这是*广义点哲学*的一部分。想了解更多可以参考Leinster的[*Doing Without Diagrams*](http://www.maths.ed.ac.uk/~tl/elements.pdf)和William Lawvere的[*An Elementary Theory of the Category of Sets*](http://www.tac.mta.ca/tac/reprints/articles/11/tr11.pdf)（2005年版本）。

最后，还有一个很干练灵性的结果叫做密度定理（例如[这里](http://www.math.jhu.edu/~eriehl/context.pdf)的定理6.5.8），它告诉我们每个从 $\mathbf{C}^{op}$ 到 $\mathbf{Set}$ 的函子 $F$ 实际上都是由被表示的函子 $\text{hom}(-,X)$ *构建*而成的。严谨地说，每个这样的 $F$ 都是某些 $\text{hom}(-,X)$ 的[余极限](https://en.wikipedia.org/wiki/Limit_(category_theory))。这确实是一个惊艳的成果（背后有精彩的数学！比如Kan扩展），但我会推迟相关的讨论——我们还没有讨论过[余极限或极限](http://www.math3ma.com/mathema/2018/1/2/limits-and-colimits-part-1)！至少目前为止。

---

\*证明：设 $h : F(X) \rightarrow F(Y)$ 是具有逆 $h^{-1}$ 的同构。由于 $F$ 是全忠实的，存在唯一的态射 $f : X \rightarrow Y$ 使得 $Ff = h$ 。类似地，存在唯一的态射 $g : Y \rightarrow X$ 使得 $Fg = h^{-1}$ 。因此，
$$ \text{id}\_{F(X)} = h^{-1} \circ h = Fg \circ Ff = F(fg). $$
但 $F(\text{id}\_X)$ 映射到 $ \text{id}\_{F(X)} $ ，因此由于 $F$ 是忠实的，有 $fg = \text{id}_X$ 。类似的论证可证明 $gf = \text{id}\_Y$ 。因此， $f$ 是一个同构。

## 译后记 <a name="postlude"></a>

译者的知识水平非常有限，难免有一些错误，希望读者见谅，也欢迎斧正。比如在序言部分对引理的解读是否有失偏颇乃至存在误导？
此外，这是我第一次正经翻译文章，所以也期盼在结构、句式、术语上的翻译建议`:)`
例如，本文将"vantage point"翻译为“视点”，将"cooked up from"翻译为“由……构造而来”，"gives rise to"翻译为“取决于”，不知是否有更优的选择。
欢迎交流探讨\^_\^

## 译者推荐阅读
如果你喜欢可视化或偏直觉的阐释，请务必读一读：
[Category Theory for Programmers: The Preface (by Bartosz Milewski)](https://bartoszmilewski.com/2014/10/28/category-theory-for-programmers-the-preface/trackback/) （本文对应章节：[The Yoneda Lemma](https://bartoszmilewski.com/2015/09/01/the-yoneda-lemma/trackback/)）
偏数学方向的推荐：
[代数学方法：第一卷（李文威 著）](https://www.wwli.asia/downloads/books/Al-jabr-1.pdf)
偏计算机方向的推荐：
[Categorical Logic and Type Theory (by B. Jacobs)](https://people.mpi-sws.org/~dreyer/courses/catlogic/jacobs.pdf)