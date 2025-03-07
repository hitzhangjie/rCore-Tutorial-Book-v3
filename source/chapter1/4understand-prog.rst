理解应用程序和执行环境
==================================

.. toctree::
   :hidden:
   :maxdepth: 5



本节导读
-------------------------------

在前面几节，我们进行了大量的实验。接下来是要消化总结和归纳理解的时候了。
本节主要会进一步归纳总结执行程序和执行环境相关的基础知识：

 - 物理内存与物理地址
 - 函数调用与栈
 - 调用规范
 - 程序内存布局
 - 执行环境

这些知识与编译器有直接的关系，而且操作系统的执行过程，会涉及到这些知识。如果读者已经了解上述基础知识，可直接跳过本节，进入下一节学习。

.. _term-physical-address:
.. _term-physical-memory:


物理内存与物理地址
----------------------------
物理内存是计算机体系结构中一个重要的组成部分。在存储方面，CPU 唯一能够直接访问的只有物理内存中的数据，它可以通过访存指令来达到这一目的。
从 CPU 的视角看来，可以将物理内存看成一个大字节数组，而物理地址则对应于一个能够用来访问数组中某个元素的下标。与我们日常编程习惯不同的
是，该下标通常不以 0 开头，而通常以 ``0x80000000`` 开头。简言之， CPU 可以通过物理地址来 **逐字节** 地访问物理内存中保存的
数据。

值得一提的是，当 CPU 以多个字节（比如 2/4/8 或更多）为单位访问物理内存（事实上并不局限于物理内存）中的数据时，就有可能会引入端序和
地址对齐的问题。由于这并不是重点，我们在这里不展开说明。


.. _function-call-and-stack:

函数调用与栈
----------------------------

从汇编指令的级别看待一段程序的执行，假如 CPU 依次执行的指令的物理地址序列为 :math:`\{a_n\}`，那么这个序列会符合怎样的模式呢？

.. _term-control-flow:

其中最简单的无疑就是 CPU 一条条连续向下执行指令，也即满足递推公式 :math:`a_{n+1}=a_n+L`，这里我们假设该平台的指令是定长的且均为 
:math:`L` 字节（常见情况为 2/4 字节）。但是执行序列并不总是符合这种模式，当位于物理地址 :math:`a_n` 的指令是一条跳转指令的时候，
该模式就有可能被破坏。跳转指令对应于我们在程序中构造的 **控制流** (Control Flow) 的多种不同结构，比如分支结构（如 if/switch 语句）
和循环结构（如 for/while 语句）。用来实现上述两种结构的跳转指令，只需实现跳转功能，也就是将 pc 寄存器设置到一个指定的地址即可。

.. _term-function-call:

另一种控制流结构则显得更为复杂： **函数调用** (Function Call)。我们大概清楚调用函数整个过程中代码执行的顺序，如果是从源代码级的
视角来看，我们会去执行被调用函数的代码，等到它返回之后，我们会回到调用函数对应语句的下一行继续执行。那么我们如何用汇编指令来实现
这一过程？首先在调用的时候，需要有一条指令跳转到被调用函数的位置，这个看起来和其他控制结构没什么不同；但是在被调用函数返回的时候，我们
却需要返回那条跳转过来的指令的下一条继续执行。这次用来返回的跳转究竟跳转到何处，在对应的函数调用发生之前是不知道的。比如，我们在两个不同的
地方调用同一个函数，显然函数返回之后会回到不同的地址。这是一个很大的不同：其他控制流都只需要跳转到一个 *编译期固定下来* 的地址，而函数调用
的返回跳转是跳转到一个 *运行时确定* （确切地说是在函数调用发生的时候）的地址。


.. image:: function-call.png
   :align: center
   :name: function-call


对此，指令集必须给用于函数调用的跳转指令一些额外的能力，而不只是单纯的跳转。在 RISC-V 架构上，有两条指令即符合这样的特征：

.. list-table:: RISC-V 函数调用跳转指令
   :widths: 20 30
   :header-rows: 1
   :align: center

   * - 指令
     - 指令功能
   * - :math:`\text{jal}\ \text{rd},\ \text{imm}[20:1]`
     - :math:`\text{rd}\leftarrow\text{pc}+4`

       :math:`\text{pc}\leftarrow\text{pc}+\text{imm}`
   * - :math:`\text{jalr}\ \text{rd},\ (\text{imm}[11:0])\text{rs}`
     - :math:`\text{rd}\leftarrow\text{pc}+4`
       
       :math:`\text{pc}\leftarrow\text{rs}+\text{imm}`

.. _term-source-register:
.. _term-immediate:
.. _term-destination-register:

.. note::

   **RISC-V 指令各部分含义**

   在大多数只与通用寄存器打交道的指令中， rs 表示 **源寄存器** (Source Register)， imm 表示 **立即数** (Immediate)，
   是一个常数，二者构成了指令的输入部分；而 rd 表示 **目标寄存器** (Destination Register)，它是指令的输出部分。rs 和 rd 
   可以在 32 个通用寄存器 x0~x31 中选取。但是这三个部分都不是必须的，某些指令只有一种输入类型，另一些指令则没有输出部分。


.. _term-pseudo-instruction:

从中可以看出，这两条指令在设置 pc 寄存器完成跳转功能之前，还将当前跳转指令的下一条指令地址保存在 rd 寄存器中，即 :math:`\text{rd}\leftarrow\text{pc}+4` 这条指令的含义。
（这里假设所有指令的长度均为 4 字节）
在 RISC-V 架构中，
通常使用 ``ra`` 寄存器（即 ``x1`` 寄存器）作为其中的 ``rd`` 对应的具体寄存器，因此在函数返回的时候，只需跳转回 ``ra``  所保存的地址即可。事实上在函数返回的时候我们常常使用一条
**伪指令** (Pseudo Instruction) 跳转回调用之前的位置： ``ret`` 。它会被汇编器翻译为 ``jalr x0, 0(x1)``，含义为跳转到寄存器 
``ra`` 保存的物理地址，由于 ``x0`` 是一个恒为 ``0`` 的寄存器，在 ``rd`` 中保存这一步被省略。

总结一下，在进行函数调用的时候，我们通过 ``jalr`` 指令
保存返回地址并实现跳转；而在函数即将返回的时候，则通过 ``ret`` 伪指令回到跳转之前的下一条指令继续执行。这样，RISC-V的这两条指令就实现了函数调用流程的核心机制。

由于我们是在 ``ra`` 寄存器中保存返回地址的，我们要保证它在函数执行的全程不发生变化，不然在 ``ret`` 之后就会跳转到错误的位置。事实上编译器
除了函数调用的相关指令之外确实基本上不使用 ``ra`` 寄存器。也就是说，如果在函数中没有调用其他函数，那 ``ra`` 的值不会变化，函数调用流程
能够正常工作。但遗憾的是，在实际编写代码的时候我们常常会遇到函数 **多层嵌套调用** 的情形。我们很容易想象，如果函数不支持嵌套调用，那么编程将会
变得多么复杂。如果我们试图在一个函数 :math:`f` 中调用一个子函数，在跳转到子函数 :math:`g` 的同时，ra 会被覆盖成这条跳转指令的
下一条的地址，而 ra 之前所保存的函数 :math:`f` 的返回地址将会 `永久丢失` 。 

.. _term-function-context:
.. _term-activation-record:

因此，若想正确实现嵌套函数调用的控制流，我们必须通过某种方式保证：在一个函数调用子函数的前后，``ra`` 寄存器的值不能发生变化。但实际上，
这并不仅仅局限于 ``ra`` 一个寄存器，而是作用于所有的通用寄存器。这是因为，编译器是独立编译每个函数的，因此一个函数并不能知道它所调用的
子函数修改了哪些寄存器。而站在一个函数的视角，在调用子函数的过程中某些寄存器的值被覆盖的确会对它接下来的执行产生影响。因此这是必要的。
我们将由于函数调用，在控制流转移前后需要保持不变的寄存器集合称之为 **函数调用上下文** (Function Call Context) 。

.. _term-save-restore:

由于每个 CPU 只有一套寄存器，我们若想在子函数调用前后保持函数调用上下文不变，就需要物理内存的帮助。确切的说，在调用子函数之前，我们需要在
物理内存中的一个区域 **保存** (Save) 函数调用上下文中的寄存器；而在函数执行完毕后，我们会从内存中同样的区域读取并 **恢复** (Restore) 函数调用上下文
中的寄存器。实际上，这一工作是由子函数的调用者和被调用者（也就是子函数自身）合作完成。函数调用上下文中的寄存器被分为如下两类：

.. _term-callee-saved:
.. _term-caller-saved:

- **被调用者保存** (Callee-Saved) 寄存器，即被调用的函数保存的寄存器，即由被调用的函数来保证在调用它前后，这些寄存器保持不变；
- **调用者保存** (Caller-Saved) 寄存器，被调用的函数可能会覆盖这些寄存器，所以需要发起调用的函数来保存这些寄存器。

从名字中可以看出，函数调用上下文由调用者和被调用者分别保存，其具体过程分别如下：

- 调用者：首先保存不希望在函数调用过程中发生变化的 **调用者保存寄存器** ，然后通过 jal/jalr 指令调用子函数，返回之后恢复这些寄存器。
- 被调用者：在被调用函数的起始，先保存函数执行过程中被用到的 **被调用者保存寄存器** ，然后执行函数，最后在函数退出之前恢复这些寄存器。

.. _term-prologue:
.. _term-epilogue:

我们发现无论是调用函数还是被调用函数，都会因调用行为而需要两段匹配的保存和恢复寄存器的汇编代码，可以分别将其称为 **开场** (Prologue) 和 
**结尾** (Epilogue)，它们会由编译器帮我们自动插入，来完成相关寄存器的保存与恢复。一个函数既有可能作为调用者调用其他函数，也有可能作为被调用者被其他函数调用。


.. chyyuu

  对于这个函数而言，如果在执行的时候需要修改被调用者保存寄存器，而必须在函数开头的开场和结尾处进行保存；对于调用者保存寄存器则可以没有任何顾虑的随便使用，因为它在约定中本就不需要承担保证调用者保存寄存器保持不变的义务。

.. note::

   **寄存器保存与编译器优化**

   这里值得说明的是，调用者和被调用者实际上只需分别按需保存调用者保存寄存器和被调用者保存寄存器的一个子集。对于调用函数而言，在调用子函数的时候，即使子函数修改了 **调用者保存寄存器** ，编译器在调用函数中插入的代码会恢复这些寄存器；而对于被调用函数而言，在其执行过程中没有
   使用到的被调用者保存寄存器也无需保存。编译器作为寄存器的使用者自然知道在这两个场景中，分别有哪些值得保存的寄存器。
   从这一角度也可以理解为何要将函数调用上下文分成两类：可以在尽可能早的时候优化掉一些无用的寄存器保存与恢复。

.. chyyuu 最好有个例子说明

.. _term-calling-convention:


调用规范
----------------


**调用规范** (Calling Convention) 约定在某个指令集架构上，某种编程语言的函数调用如何实现。它包括了以下内容：

1. 函数的输入参数和返回值如何传递；
2. 函数调用上下文中调用者/被调用者保存寄存器的划分；
3. 其他的在函数调用流程中对于寄存器的使用方法。

调用规范是对于一种确定的编程语言来说的，因为一般意义上的函数调用只会在编程语言的内部进行。当一种语言想要调用用另一门编程语言编写的函数
接口时，编译器就需要同时清楚两门语言的调用规范，并对寄存器的使用做出调整。

.. note::

   **RISC-V 架构上的 C 语言调用规范**

   RISC-V 架构上的 C 语言调用规范可以在 `这里 <https://riscv.org/wp-content/uploads/2015/01/riscv-calling.pdf>`_ 找到。
   它对通用寄存器的使用做出了如下约定：

   .. list-table:: RISC-V 寄存器功能分类
      :widths: 20 20 40
      :align: center
      :header-rows: 1

      * - 寄存器组
        - 保存者
        - 功能
      * - a0~a7
        - 调用者保存
        - 用来传递输入参数。其中的 a0 和 a1 还用来保存返回值。
      * - t0~t6
        - 调用者保存
        - 作为临时寄存器使用，在函数中可以随意使用无需保存。
      * - s0~s11
        - 被调用者保存
        - 作为临时寄存器使用，保存后才能在函数中使用。

   剩下的 5 个通用寄存器情况如下：

   - zero(x0) 之前提到过，它恒为零，函数调用不会对它产生影响；
   - ra(x1) 是调用者保存的，不过它并不会在每次调用子函数的时候都保存一次，而是在函数的开头和结尾保存/恢复即可。虽然  ``ra`` 看上去和其它被调用者保存寄存器保存的位置一样，但是它确实是调用者保存的。
   - sp(x2) 是被调用者保存的。这个是之后就会提到的栈指针寄存器。
   - gp(x3) 和 tp(x4) 在一个程序运行期间都不会变化，因此不必放在函数调用上下文中。它们的用途在后面的章节会提到。

   更加详细的内容可以参考 Cornell 大学的 `课件内容 <http://www.cs.cornell.edu/courses/cs3410/2019sp/schedule/slides/10-calling-notes-bw.pdf>`_ 。

.. _term-stack:
.. _term-stack-pointer:
.. _term-stack-frame:

之前我们讨论了函数调用上下文的保存/恢复时机以及寄存器的选择，但我们并没有详细说明这些寄存器保存在哪里，只是用“内存中的一块区域”草草带过。实际上，
它更确切的名字是 **栈** (Stack) 。 sp(x2) 常用来保存 **栈指针** (Stack Pointer)，它指向内存中栈顶地址。在 
RISC-V 架构中，栈是从高地址向低地址增长的。在一个函数中，作为起始的开场代码负责分配一块新的栈空间，即将 ``sp`` 
的值减小相应的字节数即可，于是物理地址区间 :math:`[\text{新sp},\text{旧sp})` 对应的物理内存的一部分便可以被这个函数用来进行函数调用上下文的保存/恢复
，这块物理内存被称为这个函数的 **栈帧** (Stackframe)。同理，函数中的结尾代码负责将开场代码分配的栈帧回收，这也仅仅需要
将 ``sp`` 的值增加相同的字节数回到分配之前的状态。这也可以解释为什么 ``sp`` 是一个被调用者保存寄存器。

.. figure:: CallStack.png
   :align: center

   函数调用与栈帧：如图所示，我们能够看到在程序依次调用 a、调用 b、调用 c、c 返回、b 返回整个过程中栈帧的分配/回收以及 ``sp`` 寄存器的变化。
   图中标有 a/b/c 的块分别代表函数 a/b/c 的栈帧。

.. _term-lifo:

.. note::

   **数据结构中的栈与实现函数调用所需要的栈**

   从数据结构的角度来看，栈是一个 **后入先出** (Last In First Out, LIFO) 的线性表，支持向栈顶压入一个元素以及从栈顶弹出一个元素
   两种操作，分别被称为 push 和 pop。从它提供的接口来看，它只支持访问栈顶附近的元素。因此在实现的时候需要维护一个指向栈顶
   的指针来表示栈当前的状态。

   我们这里的栈与数据结构中的栈原理相同，在很多方面可以一一对应。栈指针 ``sp`` 可以对应到指向栈顶的指针，对于栈帧的分配/回收可以分别
   对应到 ``push`` / ``pop`` 操作。如果将我们的栈看成一个内存分配器，它之所以可以这么简单，是因为它回收的内存一定是 *最近一次分配* 的内存，
   从而只需要类似 ``push`` / ``pop`` 的两种操作即可。

在合适的编译选项设置之下，一个函数的栈帧内容可能如下图所示：

.. figure:: StackFrame.png
   :align: center

   函数栈帧中的内容

它的开头和结尾分别在 sp(x2) 和 fp(s0) 所指向的地址。按照地址从高到低分别有以下内容，它们都是通过 ``sp`` 加上一个偏移量来访问的：

- ``ra`` 寄存器保存其返回之后的跳转地址，是一个调用者保存寄存器；
- 父亲栈帧的结束地址 ``fp`` ，是一个被调用者保存寄存器；
- 其他被调用者保存寄存器 ``s1`` ~ ``s11`` ；
- 函数所使用到的局部变量。

因此，栈上 ``fp`` 信息实际上保存了一条完整的函数调用链，通过适当的方式我们可以实现对函数调用关系的跟踪。

至此，我们基本上说明了函数调用是如何基于栈来实现的。不过我们可以暂时先忽略掉这些细节，因为我们现在只是需要在初始化阶段完成栈的设置，也就是
设置好栈指针 ```sp`` 寄存器，编译器会帮我们自动完成后面的函数调用相关机制的代码生成。麻烦的是， ``sp`` 的值也不能随便设置，至少我们需要保证它指向合法的物理内存，
而且不能与程序的其他代码、数据段相交，因为在函数调用的过程中，栈区域里面的内容会被修改。如何保证这一点呢？此外，之前我们还提到我们编写的
初始化代码必须放在物理地址 ``0x80020000`` 开头的内存上，这又如何做到呢？事实上，这两点都需要我们接下来讲到的程序内存布局的知识。

程序内存布局
----------------------------

.. _term-section:
.. _term-memory-layout:

在我们将源代码编译为可执行文件之后，它就会变成一个看似充满了杂乱无章的字节的一个文件。但我们知道这些字节至少可以分成代码和数据两部分，在
程序运行起来的时候它们的功能并不相同：代码部分由一条条可以被 CPU 解码并执行的指令组成，而数据部分只是被 CPU 视作可读写的内存空间。事实上
我们还可以根据其功能进一步把两个部分划分为更小的单位： **段** (Section) 。不同的段会被编译器放置在内存不同的位置上，这构成了程序的 
**内存布局** (Memory Layout)。一种典型的程序相对内存布局如下所示：

.. figure:: MemoryLayout.png
   :align: center

   一种典型的程序相对内存布局

在上图中可以看到，代码部分只有代码段 ``.text`` 一个段，存放程序的所有汇编代码。而数据部分则还可以继续细化：

.. _term-heap:

- 已初始化数据段保存程序中那些已初始化的全局数据，分为 ``.rodata`` 和 ``.data`` 两部分。前者存放只读的全局数据，通常是一些常数或者是
  常量字符串等；而后者存放可修改的全局数据。
- 未初始化数据段 ``.bss`` 保存程序中那些未初始化的全局数据，通常由程序的加载者代为进行零初始化，即将这块区域逐字节清零；
- **堆** （heap）区域用来存放程序运行时动态分配的数据，如 C/C++ 中的 malloc/new 分配到的数据本体就放在堆区域，它向高地址增长；
- **栈** （stack）区域不仅用作函数调用上下文的保存与恢复，每个函数作用域内的局部变量也被编译器放在它的栈帧内，它向低地址增长。

.. note::

   **局部变量与全局变量**

   在一个函数的视角中，它能够访问的变量包括以下几种：
   
   - 函数的输入参数和局部变量：保存在一些寄存器或是该函数的栈帧里面，如果是在栈帧里面的话是基于当前 sp 加上一个偏移量来访问的；
   - 全局变量：保存在数据段 ``.data`` 和 ``.bss`` 中，某些情况下 gp(x3) 寄存器保存两个数据段中间的一个位置，于是全局变量是基于 
     gp 加上一个偏移量来访问的。
   - 堆上的动态变量：本体被保存在堆上，大小在运行时才能确定。而我们只能 *直接* 访问栈上或者全局数据段中的 **编译期确定大小** 的变量。
     因此我们需要通过一个运行时分配内存得到的一个指向堆上数据的指针来访问它，指针的位宽确实在编译期就能够确定。该指针即可以作为局部变量
     放在栈帧里面，也可以作为全局变量放在全局数据段中。

我们可以将常说的编译流程细化为多个阶段（虽然输入一条命令便可将它们全部完成）：

.. _term-compiler:
.. _term-assembler:
.. _term-linker:
.. _term-object-file:

1. **编译器** (Compiler) 将每个源文件从某门高级编程语言转化为汇编语言，注意此时源文件仍然是一个 ASCII 或其他编码的文本文件；
2. **汇编器** (Assembler) 将上一步的每个源文件中的文本格式的指令转化为机器码，得到一个二进制的 **目标文件** (Object File)；
3. **链接器** (Linker) 将上一步得到的所有目标文件以及一些可能的外部目标文件链接在一起形成一个完整的可执行文件。

每个目标文件都有着自己局部的内存布局，里面含有若干个段。在链接的时候，链接器会将这些内存布局合并起来形成一个整体的内存布局。此外，每个目标文件
都有一个符号表，里面记录着它需要从其他文件中寻找的外部符号和能够提供给其他文件的符号，通常是一些函数和全局变量等。在链接的时候汇编器会将
外部符号替换为实际的地址。


.. note::

    本节内容部分参考自：

    - `RISC-V C 语言调用规范 <https://riscv.org/wp-content/uploads/2015/01/riscv-calling.pdf>`_
    - `Notes from Cornell CS3410 2019Spring <http://www.cs.cornell.edu/courses/cs3410/2019sp/schedule/slides/10-calling-notes-bw.pdf>`_
    - `Lecture from Berkeley CS61C 2018Spring <https://inst.eecs.berkeley.edu/~cs61c/sp18/lec/06/lec06.pdf>`_
    - `Lecture from MIT 6.828 2020 <https://pdos.csail.mit.edu/6.828/2020/lec/l-riscv-slides.pdf>`_
