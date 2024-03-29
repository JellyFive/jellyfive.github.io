## C/C++程序内存分配

讲的比较简单。

### 内存分配区域

C/C++编译的程序占用的内存一般是以下几个部分：

1. 栈区 stack：由编译器自动分配和释放，存放函数的参数值、局部变量
2. 堆区 heap：由程序员自己分配和释放，若不释放在程序结束的时候可能会被 OS 回收
3. 全局区/静态区 static：编译器编译时分配内存，全局变量和静态变量在一起，程序结束后由 OS 释放
   1. 初始化的全局变量和静态变量在一个区域
   2. 未初始化的全局变量和静态变量在相邻的另一个区域
4. 常量区：常量字符串，程序结束后由 OS 释放
5. 代码区：存放函数的二进制代码

**举个例子**

```c
int a = 10; // 全局初始化区
char *p1; // 全局初未始化区

main(){
  int b; // 栈区
  char s[] = 'abc'; // 'abc'在常量区，s在栈区
  char *p2; // 栈
  char *p3 = '123'; // '123\n'在常量区，p3在栈区
  static int c = 1; // 全局初始化区
  p1 = (char *)malloc(10); // 分配的10字节的区域在堆区
  strcpy(p1, '123'); // '123\n'在常量区,且编译器可能会把它与p3指向的'123'优化成一个地址
}
```

**补充：**

如果使用堆容易发生内存泄露，且如果频繁使用 malloc 和 free 容易产生内存碎片

- 内存泄漏：是指程序中己动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。

- 内存碎片：已经释放的但无法再继续使用的内存，是由于申请和释放的时间不协调导致的，无法避免只能尽量减少。（操作系统的分页机制也会导致内存碎片）

### 上述变量之间的区别

**按存储区域**

- 全局变量、静态全局变量、静态局部变量存放在内存的全局区

- 局部变量存放在栈区

**按作用域**

- 全局变量在整个工程文件内有效

- 静态全局变量只在定义它的文件内有效

- 静态局部变量只在定义它的函数内有效，且程序仅分配一次内存，函数返回后该变量不会消失

- 局部变量在定义它的函数内有效，但是函数返回后会失效

- 全局变量和静态变量如果没有初始化，编译器会初始化为 0，局部变量未初始化则值不可知

- 静态局部变量与全局变量共享全局数据区，但静态局部变量只在定义它的函数内可见

- 把局部变量转变成静态变量是改变了在内存中的存储方式，也就是改变了生存期

- 把全局变量转变成静态变量是改变了作用域，限制了适用范围

**举个例子**

```c
#include <stdio.h>

void func();
int n = 1; //全局变量
int  main(void)
{
    static int a; // 静态局部变量,但静态局部变量只在定义它的函数中可见,并且只初始化一次
    int b = -10; // 局部变量
    printf("main:   a=%d,   b=%d,   n=%d\n",a,b,n);  // main: a=0, b=-10, c=1

    b += 4; // b = -6
    func(); // func: a=4, b=10, n=13
    printf("main:   a=%d,   b=%d,   n=%d\n",a,b,n); // main: a=0, b=-6, n=13

    n += 10; // n=23
    func(); // func: a=6, b=10, n=35
    printf("main:   a=%d,   b=%d,   n=%d\n",a,b,n); // main: a=0; b=-6, n=35
}
void func()
{
    static int a = 2; // 静态局部变量
    int b = 5; // 局部变量
    a += 2;
    n += 12;
    b += 5;
    printf("func:   a=%d,   b=%d,   n=%d\n",a,b,n);
}
```

## 什么是栈帧

_先看一下“过程”的定义_

**过程：**提供了一种封装代码的方式，用一组指定的参数和一个可选的返回值实现了某种功能，使得可以在程序中不同的地方调用这个函数。不同的编程语言过程的形式多样，在 C 语言中可以将过程称为**函数调用**。

_下面这句很重要！！_

- **C 语言的过程调用机制的一个关键特性是使用了栈数据结构提供的先进后出的内存管理原则**。

_再看一下栈帧的好几种定义_

- C 语言中，**每个栈帧对应一个未运行完的函数**，栈帧中保存了该函数的返回地址和局部变量等信息，栈帧大小不一致。因为在函数调用的时候会上下文切换，这里就需要有一个保存当前上下文的机制。

- 栈帧也叫过程活动记录，是编译器用来实现过程/函数调用的一种数据结构。

- 栈帧是函数执行的环境。

_总的来说_

函数在调用的时候在栈空间上开辟一段空间以供函数使用，这块空间称为栈帧。每个函数的每次调用都有自己独立的栈帧。

接下来从汇编的角度来描述栈帧。

### 运行时栈

下述汇编指令都是基于 x86-64。

运行时栈上存放函数信息的位置都是一个栈帧。

**涉及几个名词：**参数构造区、返回地址、栈指针，主要从下图来解释一下。（会包含一些汇编的指令及描述）

**这里需要知道一下。**x86-64 的栈是向低地址方向增长，栈指针%rsp 指向栈顶，也就是运行时栈的结束位置。%rbp 指向栈底，位置不变。%esp 表示采用的 32 位系统，64 为为%rsp。（涉及到汇编指令，后面用例子来描述）

**补充的一个重要的关于寄存器的点。**x86-64 的 CPU 包含了 16 个 64 位的通用目的寄存器。其实就是用来存储整数数据和指针的，可以搜一下图，每一个寄存器都用来存放一类信息，比如后面会用到的：返回值(%rax)、第一个参数(%rdi)等等。

#### 关于运行时栈

常说的进程栈其实是进程的运行时栈，运行时栈主要用来传递参数(寄存器不够用时)、存储程序的返回信息（寄存器不够用时）、保存寄存器中的值（临时存储一下，待寄存器需要的时候再复制回去）、以及存储局部变量。

结合栈增长方向，栈指针减小就是在为没有指定初始值的数据在栈上分配空间，增加栈指针是在释放空间。

下图主要描述的是过程 P 调用过程 Q，后面将过程理解为函数调用。后续的几个小节都是围绕这个图展开。

![](https://gitee.com/jellyfive/blog-image/raw/master/image/20230408183319.png)

从图中可以看到 Q 在执行的时候，P 的信息以及向上追溯到 P 的函数调用过程都是被暂时挂起的。Q 只需为自己的局部变量分配新的存储空间，或者去调用其他的函数，当 Q 返回的时候这些存储空间会被释放。（这里其实是一个先进后出的过程，可以回想一下栈的数据结构）

#### 转移控制

将控制从函数 P 转移到函数 Q 只需要把 PC（程序计数器，正在执行的指令的位置,寄存器表示为%rip)）设置为 Q 的起始位置。当从 Q 返回的时候，处理器需要知道从哪里还原 P 的现场，也就是说处理器需要记录它回到 P 时继续执行的地址（地址 A）。

汇编指令`call`会把 A 压入栈中（地址 A 也就是上图的**返回地址**，是紧跟在`call`指令后面的那条指令的地址），并将 PC 设置为 Q 的起始地址。

换句话说，调用者在调用完函数之后需要继续执行下一条指令，返回地址就用来存放下一条指令的地址。

汇编指令`ret`会从栈中弹出地址 A，并把 PC 设置为 A。

**举个例子**

multstroe 和 main 函数的反汇编代码。（objdump 可以反汇编 c 代码）

![](https://gitee.com/jellyfive/blog-image/raw/master/image/20230408193938.png)

1. `callq`指令的地址为 0x400563，从 a）图看到指明了此时%rip 和%rsp 的值
2. 调用`callq`后，返回地址 0x400568 被压入栈，且%rip 指向了 0x40050，也就是 mulstore 的第一条指令
3. multstore 指定直到遇到`retq`，会将返回地址弹出并跳转到%rip，继续执行 main 函数

#### 数据传送

大部分数据传输都是通过寄存器如%rdi %rsi 实现的。过程 P 调用过程 Q 时，必须先把参数复制到适当的寄存器中。可以通过通用寄存器最多传递**6 个整型参数**（为啥是 6 个？从 16 个通用寄存器里面可以看到原因，从寄存器名字也能看出名字是有特殊含义的）

如果一个函数有大于 6 个的整型参数，超出的部分需要通过栈来传递，也就是图中的**“参数构造区”**。如果 P 调用 Q 有 n 个参数(n>7)，第 1~6 个参数存放在对应的通用寄存器中，参数 7~n 存放在函数 P 的栈帧结构中。

通过栈传递参数的时候，所有数据大小对向 8 的倍数对齐。（为什么是 8 后面可以再总结一个文档，这里不赘述）

**举个例子**

这里也涉及到了不同字节数的整型和指针的存储，每个指针是 8 字节。

![](https://gitee.com/jellyfive/blog-image/raw/master/image/20230408194033.png)

从图中可以看出前 6 个参数实际上是通过寄存器传递的，后面两个 a4 和\*a4p 通过栈传递。`movq 16(%rsp),%rax`用于获取 a4p。这里也涉及到一个知识点：函数参数是从右向左压栈。

#### 栈上局部存储

大部分时候再函数调用都不需要超过寄存器大小的本地存储区域，但也有一些必须要存储在内存的局部数据，这部分体现为图中的**局部变量**。

对于局部数据需要存在内存中的情况：

- 寄存器不足以存放所有本地数据

- 对一个局部变量使用地址运算符“&”，因此需要为其产生地址

- 某些局部变量是数组或结构，也可以理解为数组，通过引用的方式访问的。

**举个例子**

结合数据传送部分的例子举个例子。

![](https://gitee.com/jellyfive/blog-image/raw/master/image/20230408194112.png)

图b)中是call_proc的汇编代码，其中2~15行都是在调用proc做准备，包括为局部参数和函数调用的参数建立栈帧。从这个函数能看到，有x1~x4四个局部变量，然后在调用proc的时候用到了这四个局部变量的地址，因此需要为他们分配内存。

3~6 行是在为变量赋值。7,10,12,14 是生成对应的位置的指针。从图 33 看到，在栈上为 x1~x4 分配了内存，这个是按照字节大小排列的。然后参数 7 和参数 8 表示要传递给 proc 的两个局部变量。**（为啥是两个呢？因为其余的 6 个参数是存在寄存器中的，这个不要忘了）**

调用 proc 的时候会，就会执行 proc 的汇编代码。当程序返回 call_proc 的时候，代码会取出 4 个局部变量并计算(x1+x2)\*(x3+x4)，程序结束的时候释放栈帧。

#### 寄存器中的局部存储空间

寄存器组是唯一被所有函数调用共享的资源，需要保证调用者调用另一个函数的时候，被调用者不会覆盖调用者要使用到的寄存器值。

寄存器%rbp %rbx %r12~%r15 被划分为被调用者保存寄存器。就是说函数 P 调用函数 Q 的时候，函数 Q 需要保存这些寄存器的值，保证在 Q 返回到 P 的时候这些值与被调用前是一样的，在图中表示为**被保存的寄存器**。

这部分不过多举例。

### 举个例子

```c
// main.c
#include <stdio.h>

int ADD(int x, int y)
{
    int z = 0;
    z = x + y;
    return z;
}

int main()
{
    int a = 10;
    int b = 20;
    int ret = ADD(a, b);
    printf("result: %d\n", ret);

    return 0;
}
```

将上述代码通过 objdump 反汇编得到汇编代码，关于编译链接这里不讲述，汇编代码也只描述需要的部分。

```text
gcc -c -o main.o main.c // 先生成目标文件
objdump -s -d main.o > main.o.txt // 再用objdump反汇编
```

一些指令：

- `push %rbp` : 当前%rbp 存放的是调用者栈帧的栈底指针，由于一个时刻只能有一个栈帧结构，因此需要将调用者栈帧的栈底指针压入栈，也就是保存旧的%rbp

- `mov %rsp,%rbp`：现在执行新的函数，因此%rbp 应该保存新的栈帧的栈底，也就是说调用者栈帧的栈顶是新的栈帧的栈底（该栈底保存的是调用者栈帧的栈底地址）

- `pop %rbp`：对栈顶元素出栈，栈顶存放的是调用者栈帧的栈底地址，这一指令就是把这个栈底地址赋值给%rbp，从而恢复调用者栈帧的栈底。（栈的先进后出结构）

```c
_main:                                  ## @main
## %bb.0:
	pushq	%rbp // 把rbp压入栈
	movq	%rsp, %rbp // 把rsp赋值给rbp
	subq	$32, %rsp // 为后面的参数分配空间，减小栈指针
	movl	$0, -4(%rbp)
	movl	$10, -8(%rbp) // 传入参数10
	movl	$20, -12(%rbp) // 传入参数20
	movl	-8(%rbp), %edi // 将a的值移到寄存器%edi
	movl	-12(%rbp), %esi // 将b的值移到寄存器%esi
	callq	_ADD       // 调用ADD函数
	movl	%eax, -16(%rbp) //将eax寄存器的值赋值给主函数中的ret，返回值
	movl	-16(%rbp), %esi
	leaq	L_.str(%rip), %rdi
	movb	$0, %al
	callq	_printf
	xorl	%ecx, %ecx
	movl	%eax, -20(%rbp)         ## 4-byte Spill
	movl	%ecx, %eax
	addq	$32, %rsp
	popq	%rbp
	retq
	.cfi_endproc
```

示意图，示意图没有按照字节来排序，只是为了表明栈指针的存储。如下：

![](https://gitee.com/jellyfive/blog-image/raw/master/image/20230408194243.png)

```c
_ADD:                                   ## @ADD
	pushq	%rbp  //保存旧的rbp
	movq	%rsp, %rbp // 把当前rsp的地址赋值给rbp作为新的栈底
	movl	%edi, -4(%rbp) // 参数x
	movl	%esi, -8(%rbp) // 参数y
	movl	$0, -12(%rbp) // 变量z
	movl	-4(%rbp), %eax
	addl	-8(%rbp), %eax // x+y
	movl	%eax, -12(%rbp) // z=x+y
	movl	-12(%rbp), %eax // 函数返回值存在eax寄存器中
	popq	%rbp // 释放栈空间
	retq // 把返回地址出栈并跳转返回地址对应的指令
```

示意图如下：

![](https://gitee.com/jellyfive/blog-image/raw/master/image/20230408194303.png

**rsp 寄存器的值一直在变（随着局部变量、临时变量等的加入），rbp 寄存器的值在本次函数栈帧中一直保持不变，且始终都是被调函数栈帧的栈底。**

## 最后，一个进程在内存中是怎样映射的呢

假设一个函数的调用顺序为：`main(...) ->; func_1(...) ->; func_2(...) ->; func_3(...)`，当其被操作系统调入内存运行，对应的进程在内存中的映射为：

![](https://gitee.com/jellyfive/blog-image/raw/master/image/20230408194321.png)

随着函数调用层数的增加，函数栈帧是一块块地向内存低地址方向延伸的。

### 参考资料

《深入理解计算机系统》原书第三版，第 3 章，3.7 过程
