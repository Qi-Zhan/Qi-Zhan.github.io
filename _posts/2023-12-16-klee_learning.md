---
layout: post
title: "符号执行, KLEE 与 LLVM"
categories: cs
author:
- 詹奇
toc:
  beginning: true
---

最近在尝试使用 KLEE 科研，这里记录一些学习笔记。

## 符号执行一瞥

众所周知，一般的测试都是基于具体的输入，观察程序的行为。
而符号执行的基本思想是，将程序的输入符号化，然后通过符号执行引擎运行程序，得到程序的执行路径，然后通过约束求解器求解路径上的约束，得到具体的输入。具体的知识和例子可参考 [1]，不再赘述。一篇我认为总结的很好的综述。由于我们的目的是进一步的了解符号执行和 KLEE, 所以我们也会偏重总结 KLEE 对于符号执行里的经典问题的选择。

### 执行引擎

符号执行的核心是对于一段具体的代码，究竟该如何执行它，也就是执行引擎。我在看 KLEE 的时候，发现它认为自己是 dynamic symbolic execution(DSE), 但奇怪的是，很多文献将 DSE 和 concolic execution(CE) 混在一起谈论，
但在我的理解下 KLEE 的实现还是比较接近传统符号执行的，CE 在我的认知下是 DART[4] 采用的方法，每次都用具体输入运行然后解约束生成下一个具体的输入的方法，很明显和 KLEE 的做法不一致。
那难道 KLEE 不是 DSE？然而其官网上第一句话就是 KLEE is a dynamic symbolic execution engine built on top of the LLVM compiler infrastructure.

这个问题我想了很久，最终在 [2] [3] 这两个资料下得到了解答，符号执行这个词在发展过程中经过了太多的使用。已经需要进一步分类才能区分主流方法了。下面是我根据各种资料再次总结的符号执行的分类：

#### 动态符号执行(DSE)

动态符号执行混合了具体的程序执行和符号的程序执行，以此来提升运行效率和解决与环境交互的问题。而 DSE 又能再次进行分类：

1. "Offline" DSE, 也就是著名的 concolic execution(CE). 它们依赖具体的程序执行来驱动符号的执行。即，**CE 每次的执行都是具体的值的运行**，同时维持一个符号化的约束列表(通过插桩等方法记录约束和具体值的走向)，在真正执行完后，尝试翻转约束列表里面的一些约束得到新的约束然后解得新的具体的程序输入。这也解释了为什么称为离线，因为是在每次具体执行结束之后才求解得到新输入。[5] 的实验内容很好的解释了该如何实现以及有什么好处，推荐感兴趣的同学完成。CE 本质上更像是测试，所以和环境交互并没有太大困难，只是会丢失一些符号状态和相应约束。除了 DART，还有 SAGE, PEX (Microsoft), CUTE (UIUC), CREST (Berkeley) 等工具。

2. "Online" DSE, 也就是 EXE(KLEE), 还有 SPF (NASA Java), Cloud9 这些工具采用的方法，也是我们主要关心的。程序的初始值是符号值的输入，运行程序时如果目标都是具体值就正常执行，如果有一个符号值就采用 expr tree 符号化执行。在遇到分支指令时，fork 出新的状态然后在两边添加对应的约束。在执行终止时求解约束得到具体的测试输入。

3. Selective Symbolic Execution. 交替进行具体执行和符号执行，可以聚焦于感兴趣的代码符号执行，例如 S$$^2$$E.

这也就解决了我的主要疑问，DART 和 KLEE 代表的是 DSE 下的两类方法，由此我认为将 DSE 和 CE 视为同一个东西是不太合理的，只要具体的运行代码了，无论是具体化还是符号化都应当视作 DSE.

#### 静态符号执行(SSE)

既然有 DSE, 那肯定就有 SSE，这也是一开始我很疑惑的地方。都是符号环境下运行，普通的 SSE 和以 KLEE 为代表 DSE 区别在哪里？从 [6] 中讲的内容，我个人理解 SSE 更好的理解方式是当成正向的验证策略。基于 hoare 逻辑的验证求的是最弱前条件，然后利用求解器检验当前的前条件能否推出最弱前条件。那么如果我们换一个思路，从前往后来尝试验证，那就可以通过程序的语义来(具体见 [6])抽象代码收集约束并最终完成**前向**的验证。
SSE 的考量中没有路径爆炸的问题(?)，因为它总是像经典的静态分析一样合并路径，通过一个符号表达式来抽象所有的路径。

[7] 中也有一个不错的总结：SSE 是 statement-based. DSE 是 path-based.
DSE 每次在 **一条路径** 上探索程序，为每条路径生成约束。
而 SSE 遍历程序，将整个语句转换成公式，这里的公式则代表了任一路径的性质。可以将漏洞描述成

#### 纯符号执行(Pure SE)

纯符号执行枚举程序的路径收集约束然后检测这些条件能否满足。这好像是在分离逻辑验证用的比较多，我就不太懂了 :).

#### 分类总结

综上所述，**我最终的理解是**，总是运行具体的输入且通过收集路径约束得到下一个具体输入的就是 CE, 这是 DSE 的一种形式，另一种 DSE 则是在符号化基础上解释程序，遇到分支则 fork 出新状态，最后通过约束求解器求解约束得到具体的输入。而 DSE 与 SSE 的最大区别在于 DSE 上按照测试的路子一条路径一个状态(path based)。而 SSE 的视角不同，是按照静态分析的路子尝试抽象(statement-based)，力图只分析一条合并的路径(?)。**感觉平时我们讨论的最广泛的符号执行其实都是指动态符号执行。**

> 虽说费劲心思查了好久资料想这个问题在实践上并没有什么用，而且实践上很多方法就是把各种符号执行技术混起来用的，但尝试搞清楚总是有些裨益 :(

### 内存模型

内存模型同样符号执行中重要且不易处理的问题。
符号执行过程中，除了遇到那些显然的指令(算法运算)，对于指针和数组取址的处理就涉及到了一个大难题：如何建模内存。这听起来是个很简单的事情，直觉上来看把它当做一个大数组就行了，然而当指针和下标是符号值的情况下，这个问题就变得复杂了。

对于数组的下标，一种简单的方法是假设这个符号值可以是有界集合中的任意一个下标，也就是每个情况 fork 一个新状态。而如 KLEE 一般会利用 SMT 里面的 array 相关 theory, 直接将数组作为约束的一等公民(first class)，将数组和相关操作编码进了约束求解器里面。

而对于符号化地址的处理则更加麻烦。原则上来说一个符号化的地址可以指向内存的任何一个地方，但如果真要这么考虑状态实在是太多了。所以实践中有些方法采用 address concretization, 地址具体化的方法。而 KLEE 则是将每一个内存的对象都映射到一个数组上，这样就将一个平坦的地址空间映射到了一个分段的地址空间上。

具体的处理我们在后面讨论 KLEE 源码中内存相关操作时再详细介绍。

### 与环境交互

环境的交互(系统环境与应用环境)同样也是符号执行遇到的难题。简单来说就是遇到难以符号执行的代码时该怎么处理，例如打开一个文件的返回值，调用没有源码的库函数/底层函数等等。
早期的一些工作 DART, EXE 等选择不处理，也就是放弃对这些函数的符号执行，采用具体的输入来让程序继续运行下去。

而 KLEE 则是自己实现了一套基本的符号文件系统来抽象与文件系统交互的过程。

### 缓解路径爆炸

当程序中有循环时，符号执行会产生大量的路径(每个 while 语句理论上都可以有无穷路径).
所有实践中使用的符号执行工具都会遇到路径爆炸的问题，而多数工具都有自己的或启发式，或经验的方法来缓解。

一些很自然的方法包括：

* eager evaluation 经常性的检查约束能否满足而不是到最后再次求解一个庞大的约束，对无解的状态直接剪枝。
* summary 对函数和循环做摘要，生成一个直接的映射关系就不用重复计算了。这个技巧在经典的静态分析中也很常见。
* Interpolation 没咋看懂
* state merge 状态合并。

例如在对于下面这段代码：

```c
void foo(int x, int y) { 
  if (x < 5) 
    y = y * 2; 
  else 
    y = y * 3; 
  return y;
}
```

按照一般的情况会生成两个状态 $$\pi = \alpha_x < 5, y = 2 * \alpha_x$$; $$\pi = \alpha_x \ge 5, y = 3 * \alpha_x$$, 利用 SMT 的表示能力 ite(if-then-else) 我们可以将这两个状态合并成一个状态 $$ \pi = \text{true}, y = \text{ite}(\alpha_x < 5, 2 * \alpha_x, 3 * \alpha_x)$$.

对于类似的情况状态合并总是可以做的，但这也显示是一个 trade-off, 因为这加剧了求解器的负担。这也就诞生了很多启发式的方法，比较有名的是 Veritesting, 它根据简单和困难的语句来考虑是否合并，那些系统调用，也是经典的 DSE 和 SSE 混合使用的例子。

### 约束求解

约束的求解并不是我目前太关心的问题，简单总结一下我在 [1] 中看到的一些 KLEE 使用的一些优化策略。

* 重写/简化表达式
* implied value concretization, 对于可以推断得到的符号值，直接替换为具体值
* KLEE 还有一个比较有趣的，称为 counterexample caching 的方法。通过一个 cache, 约束集合映射到一组具体的赋值。当不可满足集合(SMT 无解)在 cache 里面且是我们要求的 S 的子集时，显然 S 也不可满足。同样如果可满足集合是 S 的超集，那 S 也可满足，即使是 S 的子集，也可以优先代入试试看。

## KLEE

OK，我们现在进入 KLEE 源码分析阶段。希望能在解释清楚 KLEE 大致流程的同时，**能和前面的理论级别的阐释**对应。
安装过程采用了 KLEE 提供的 Docker 镜像，十分方便，不再赘述。

### 主循环

KLEE 的主循环出奇的简单且符合预期：

```cpp
// lib::Core::Executor 
// void Executor::run(ExecutionState &initialState)
while (!states.empty() && !haltExecution) {
    // 由当前使用的 searcher 选择下一个要 run 的状态
    ExecutionState &state = searcher->selectState();
    // 当前状态的指令
    KInstruction *ki = state.pc;
    // 指令往前偏移
    stepInstruction(state);
    // 执行指令
    executeInstruction(state, ki);
    // 更新状态集合，例如增加新产生的状态，删除已经结束的状态
    updateStates(&state);
}
```

当然这里面最重要的就是 `executeInstruction` 了，真正的解释执行每一个 LLVM IR, 约 **1.3k** LOC, 我简化了大部分代码：

```cpp
void Executor::executeInstruction(ExecutionState &state, KInstruction *ki) {
  Instruction *i = ki->inst;
  switch (i->getOpcode()) {
    // 算数表达式，最典型且最易处理的情况
    case Instruction::Add: {
      ref<Expr> left = eval(ki, 0, state).value;
      ref<Expr> right = eval(ki, 1, state).value;
      // 找到 ki 对应的 target index, 赋一个 AddExpr
      bindLocal(ki, state, AddExpr::create(left, right));
    }
    // cast 转类型指令略过
    // 控制流指令, 最经典的 fork 处理，一边增加 condition 为 true 的约束，一边增加 condition 为 false 的约束
    case Instruction::Br: { ... }
    // ** 内存相关指令 **
    case Instruction::Alloca: {
      // 一些与 LLVM 的交互得到要分配的内存大小
      executeAlloc(state, size, true, ki);
    }
    // 这个 executeMemoryOperation 后面重点讨论
    case Instruction::Load:
    case Instruction::Store: {
      executeMemoryOperation(...);
    }
    // GEP 只计算地址，不参与内存运行，怪不得看起来比较 trivial
    // <https://llvm.org/docs/LangRef.html#getelementptr-instruction>
    case Instruction::GetElementPtr: {
      ...
    }
  }    
}
```

eval 函数如我们前面所介绍的，常量就从常量池里面取，变量就从当前栈帧的局部变量里面取。其中的 `index` 建立与我想在后面介绍的构建系统相关。

```cpp
const Cell& Executor::eval(KInstruction *ki, unsigned index, 
                           ExecutionState &state) const {
  int vnumber = ki->operands[index];
  if (vnumber < 0) { // 常量
    unsigned index = -vnumber - 2;
    return kmodule->constantTable[index];
  } else { // 变量
    unsigned index = vnumber;
    StackFrame &sf = state.stack.back();
    return sf.locals[index];
  }
}
```

### 状态管理

KLEE 中的状态，也就是一个个被 fork 出的分支路径，在 `/klee/Execution-State.h` 中, 主要包含两类 objects:

* AddressSpace: 包含了当前状态所有 objects 的元数据，包括全局，局部和堆上的对象。`AddressSpace::resolveOne` 用于解析一个地址，返回一个 `ObjectState` 对象。
* ConstraintManager: 记录约束。

### 内存相关操作

简单的符号执行在上文讨论的函数 `executeInstruction` 中已经是自明的了。但是对于内存相关的操作，例如 `Load` 和 `Store` 就需要进一步的处理了。这里也能对应前文对于 KLEE 内存模型的讨论，也是我接下来工作比较关心的部分。下面的内容大量参考 [8], 一篇介绍 KLEE 内部实现的文章。

#### executeMemoryOperation

KLEE 中两个与内存相关的类是 `MemoryObject` 和 `ObjectState`, 定义在 `lib/Core/Memory.h` 中。

MemoryObject 用来表示一个有基址和大小的 object, 在 `executeMemoryOperation` 中 KLEE 自动确保这样的访问是合法的，MemoryObject 提供了一些方便的方法来实现这一点。如果说 MemoryObject 关注的是 Object 的空间一致性，那么 ObjectState class 的作用就是用来真正访问状态里的内存值。

如我们上面看到的， Load 和 Store 指令的实现都来自`executeMemoryOperation`. 这个函数的实现混合了 我们前面提到的 AddressSpace, MemoryObject::getBoundsCheckOffset, ObjectState, 如果出现了 overflow 就会报错并终止当前状态。

### 构建过程

KLEE 是基于 LLVM IR 的，所以这里的构建过程指的是 KLEE 是如何通过 wrapper 的方式将一个项目的字节码(.bc 文件) 转换成适合自身处理的代码。例如前面的 `eval` 函数里面的表的建立，各种 IR 相关信息的获取。这个部分在符号执行的技术上是不太重要的，主要是工程实践相关，但如果我们也想做类似的基于某一种 IR 的进一步操纵，看看如何包装还是有一定参考价值的。

经过我一番寻找，这里面最重要的函数应该是 `lib:module:KModule::manifest`:

```cpp
void KModule::manifest(InterpreterHandler *ih, bool forceSourceOutput) {
  /* Build shadow structures */
  /* 把所有指令搞个表，trivial */
  infos = std::unique_ptr<InstructionInfoTable>(
      new InstructionInfoTable(*module.get()));

  std::vector<Function *> declarations;

  for (auto &Function : *module) {
    // wrapper KFunction, 里面有 wrapper KInstruction
    //
    // 这里面完成了我感兴趣的 local 的绑定和计算
    auto kf = std::unique_ptr<KFunction>(new KFunction(&Function, this));
    functionMap.insert(std::make_pair(&Function, kf.get()));
    functions.push_back(std::move(kf));
  }
  /* Compute various interesting properties */
  ...
}
```

## LLVM

可以看出，KLEE 的本质是在符号基础上解释 LLVM IR。我感觉多了解一些 LLVM 的使用对于现在和今后科研都能有所帮助。简单的 LLVM 介绍也不再赘述，只列出我感兴趣的(大概)更深入的内容。

### 一切皆 Value

这是一开始让我感到有些奇怪的点，LLVM 的 Value Class 的描述:

```text
This is a very important LLVM class. It is the base class of all values computed by a program that may be used as operands to other values. Value is the super class of other important classes such as Instruction and Function. All Values have a Type. Type is not a subclass of Value. Some values can have a name and they belong to some Module. Setting the name on the Value automatically updates the module's symbol table.

Every value has a "use list" that keeps track of which other Values are using this Value. A Value can also have an arbitrary number of ValueHandle objects that watch it and listen to RAUW and Destroy events. See llvm/IR/ValueHandle.h for details.
```

原来这个 use list 是会追踪全部 value 的，而 无论是指令，或者函数都是 Value.

To be continued...

## 杂项

* [wllvm](https://github.com/travitch/whole-program-llvm) 和 [gllvm](https://github.com/SRI-CSL/gllvm) 可以基于项目的 makefile 生成整个项目的 LLVM IR，这样可以方便的使用 KLEE 进行符号执行。
* 记得加参数 [--disable-verify](https://github.com/klee/klee/issues/937), 否则无法加载 bitcode 文件。
* --optimize 说不定是一个很重要的参数。

## 参考资料

[1] [A Survey of Symbolic Execution Techniques.](https://arxiv.org/pdf/1610.00502.pdf)

[2] <https://ece.uwaterloo.ca/~agurfink/stqam.w19/assets/pdf/W05-DSE.pdf>

[3] <https://alastairreid.github.io/RelatedWork/notes/symbolic-execution/>

[4] [DART](https://dl.acm.org/doi/10.1145/1065010.1065036)

[5] [MIT lab](https://css.csail.mit.edu/6.858/2023/labs/lab3.html)

[6] [CMU program analysis ch 13](https://cmu-program-analysis.github.io/2023/index.html)

[7] [Enhancing Symbolic Execution with Veritesting](https://softsec.kaist.ac.kr/~sangkilc/papers/avgerinos-icse14.pdf)

[8] [KLEE Internal](https://github.com/angea/pocorgtfo/blob/master/contents/articles/18-08.pdf)
