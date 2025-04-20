- [Lua stack 简介](#lua-stack-简介)
  - [Lua 整体执行流程](#lua-整体执行流程)
  - [Lua stack 与寄存器](#lua-stack-与寄存器)
  - [Lua 寄存器分配](#lua-寄存器分配)
  - [加深理解](#加深理解)

# Lua stack 简介

**声明：以下 Lua stack 分析基于 Lua 5.4.7 。并且分析 Lua 脚本代码的栈结构，不包括 C closure ，但总体结构是一致的，只是细节有所不同。**

## Lua 整体执行流程

开始之前，先来看一段 C 代码。

```c
# include <stdio.h>

static int foo(int a, int b)
{
    return a + b;
}

int main()
{
    int a = 1;
    int b = 2;
    int ret = foo(a, b);
    printf("%d\n", ret);
    return 0;
}
```

接着编译且反汇编。这里使用 `O0` 确保编译器生成的机器代码指令与源代码尽量一一对应，便于观察。

```shell
$ gcc test.c -c -O0 -g -Wall -mno-red-zone
$ objdump -dS test.o
```

反汇编后输出的汇编代码如下所示。

```asm
Disassembly of section .text:

0000000000000000 <foo>:
# include <stdio.h>

static int foo(int a, int b)
{
   0:   f3 0f 1e fa             endbr64 
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   48 83 ec 08             sub    $0x8,%rsp
   c:   89 7d fc                mov    %edi,-0x4(%rbp)
   f:   89 75 f8                mov    %esi,-0x8(%rbp)
    return a + b;
  12:   8b 55 fc                mov    -0x4(%rbp),%edx
  15:   8b 45 f8                mov    -0x8(%rbp),%eax
  18:   01 d0                   add    %edx,%eax
}
  1a:   c9                      leave  
  1b:   c3                      ret    

000000000000001c <main>:

int main()
{
  1c:   f3 0f 1e fa             endbr64 
  20:   55                      push   %rbp
  21:   48 89 e5                mov    %rsp,%rbp
  24:   48 83 ec 10             sub    $0x10,%rsp
    int a = 1;
  28:   c7 45 f4 01 00 00 00    movl   $0x1,-0xc(%rbp)
    int b = 2;
  2f:   c7 45 f8 02 00 00 00    movl   $0x2,-0x8(%rbp)
    int ret = foo(a, b);
  36:   8b 55 f8                mov    -0x8(%rbp),%edx
  39:   8b 45 f4                mov    -0xc(%rbp),%eax
  3c:   89 d6                   mov    %edx,%esi
  3e:   89 c7                   mov    %eax,%edi
  40:   e8 bb ff ff ff          call   0 <foo>
  45:   89 45 fc                mov    %eax,-0x4(%rbp)
    printf("%d\n", ret);
  48:   8b 45 fc                mov    -0x4(%rbp),%eax
  4b:   89 c6                   mov    %eax,%esi
  4d:   48 8d 05 00 00 00 00    lea    0x0(%rip),%rax        # 54 <main+0x38>
  54:   48 89 c7                mov    %rax,%rdi
  57:   b8 00 00 00 00          mov    $0x0,%eax
  5c:   e8 00 00 00 00          call   61 <main+0x45>
    return 0;
  61:   b8 00 00 00 00          mov    $0x0,%eax
}
  66:   c9                      leave  
  67:   c3                      ret
```

上述输出可以清晰的看见 `main` 和 `foo` 函数左侧的机器指令和右侧的汇编代码，以及它们对应的 C 代码。其中 `main` 函数中的 `sub $0x10,%rsp` 和 `foo` 函数中的 `sub $0x8,%rsp` 是为局部变量（函数的参数也是局部变量）分配栈空间。

我们再来看上述 C 代码对应的 lua 代码。

```lua
local function foo(a, b)
    return a + b
end

local a = 1
local b = 2
local ret = foo(a, b)
print(ret)
```

接着使用 Lua 编译器执行 `luac -l test.lua` 查看输出的 Lua 虚拟机指令中的函数调用。

```
main <test.lua:0,0> (12 instructions at 0x555fe0945cc0)
0+ params, 6 slots, 1 upvalue, 4 locals, 1 constant, 1 function
        1       [1]     VARARGPREP      0
        2       [3]     CLOSURE         0 0     ; 0x555fe0946000
        3       [5]     LOADI           1 1
        4       [6]     LOADI           2 2
        5       [7]     MOVE            3 0
        6       [7]     MOVE            4 1
        7       [7]     MOVE            5 2
        8       [7]     CALL            3 3 2   ; 2 in 1 out
        9       [8]     GETTABUP        4 0 0   ; _ENV "print"
        10      [8]     MOVE            5 3
        11      [8]     CALL            4 2 1   ; 1 in 0 out
        12      [8]     RETURN          4 1 1   ; 0 out

function <test.lua:1,3> (4 instructions at 0x555fe0946000)
2 params, 3 slots, 0 upvalues, 2 locals, 0 constants, 0 functions
        1       [2]     ADD             2 0 1
        2       [2]     MMBIN           0 1 6   ; __add
        3       [2]     RETURN1         2
        4       [3]     RETURN0
```

虽然 C 是静态编译型语言而 Lua 是动态解释型语言，但都作为编程语言，本身就存在着一丝相似性，并且 Lua 是由 C 实现的，可以通过类比 C 中相似的概念，去探讨 Lua 中这些概念对应的实现细节。

下面先来类比看一下 Lua 整体执行的过程。

C 的执行过程通常是：`源代码 -> 词法分析 -> 语法分析 -> Intermediate Representation -> 生成 CPU 指令 -> 生成可执行文件 -> 由操作系统执行`。

[The implementation of Lua 5.0](https://lua.org/doc/jucs05.pdf) 提到：
> The Lua compiler uses no intermediate representation. It emits instructions for the virtual machine “on the fly” as it parses a program.

作为类比 Lua 的执行过程是：`源代码 -> 词法分析 -> 语法分析 -> 生成虚拟机指令 -> 由解释器执行` 。
从 Lua 实现的视角来看，执行过程是：`code -> lua_load -> luaX_next 词法分析 -> luaY_parser 语法分析 -> LClosure 虚拟机指令 -> lua_callk/lua_pcallk/lua_resume -> luaV_execute` 执行虚拟机指令 。

下面来探讨关于 Lua stack 的一些实现细节。

## Lua stack 与寄存器

继续类比 C stack ，根据 CSAPP Chapter3 可知，C stack 是一段连续的地址空间，并且栈向低地址处方向增长，即栈顶的地址比栈上其他元素地址都小。


接着上述反汇编代码，如下所示，寄存器 `%rsp` 中存放的便是栈顶地址，通过 `sub $0x10,%rsp` 为局部变量分配栈空间，其中变量 `a` 的地址是 `-0xc(%rbp)` 而 `b` 的地址是 `-0x8(%rbp)` 。这里使用的是减法，也印证栈是向低地址处方向增长。

```
int main()
{
  1c:   f3 0f 1e fa             endbr64 
  20:   55                      push   %rbp
  21:   48 89 e5                mov    %rsp,%rbp
  24:   48 83 ec 10             sub    $0x10,%rsp
    int a = 1;
  28:   c7 45 f4 01 00 00 00    movl   $0x1,-0xc(%rbp)
    int b = 2;
  2f:   c7 45 f8 02 00 00 00    movl   $0x2,-0x8(%rbp)
```

如下所示，使用 `call` 指令调用 `foo` 函数。 `call` 指令会把下一条指令地址压入栈中，并跳转到目标地址执行指令（设置 PC 寄存器），也就是 `foo` 函数包含的指令的起始地址。

```
    int ret = foo(a, b);
  36:   8b 55 f8                mov    -0x8(%rbp),%edx
  39:   8b 45 f4                mov    -0xc(%rbp),%eax
  3c:   89 d6                   mov    %edx,%esi
  3e:   89 c7                   mov    %eax,%edi
  40:   e8 bb ff ff ff          call   0 <foo>
```

如下所示，`foo` 函数中，也是通过 `sub $0x8,%rsp` 为局部变量分配栈空间，最后调用 `ret` 指令，返回到 `main` 函数中。 `ret` 指令就是弹出栈顶的地址，并跳转到该地址继续执行指令。

```
static int foo(int a, int b)
{
   0:   f3 0f 1e fa             endbr64 
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   48 83 ec 08             sub    $0x8,%rsp
   c:   89 7d fc                mov    %edi,-0x4(%rbp)
   f:   89 75 f8                mov    %esi,-0x8(%rbp)
    return a + b;
  12:   8b 55 fc                mov    -0x4(%rbp),%edx
  15:   8b 45 f8                mov    -0x8(%rbp),%eax
  18:   01 d0                   add    %edx,%eax
}
  1a:   c9                      leave  
  1b:   c3                      ret 
```

函数栈 stack 的基本作用就是为局部变量分配存储空间，当函数返回时，回收此栈空间，也就回收了局部变量占用的空间。
栈这种数据结构的基本性质是后进先出，而调用函数时，为被调用的函数分配新的栈空间，被调用函数执行完毕返回时，也就把相应的栈空间回收。刚好与函数调用时的栈空间的分配与回收相对应：后分配的栈空间，先回收。
此外 C stack 还保存了函数执行完后返回的地址，隐式形成了函数调用链。

```
main <test.lua:0,0> (12 instructions at 0x555fe0945cc0)
0+ params, 6 slots, 1 upvalue, 4 locals, 1 constant, 1 function
        1       [1]     VARARGPREP      0
        2       [3]     CLOSURE         0 0     ; 0x555fe0946000
        3       [5]     LOADI           1 1
        4       [6]     LOADI           2 2
        5       [7]     MOVE            3 0
        6       [7]     MOVE            4 1
        7       [7]     MOVE            5 2
        8       [7]     CALL            3 3 2   ; 2 in 1 out
        9       [8]     GETTABUP        4 0 0   ; _ENV "print"
        10      [8]     MOVE            5 3
        11      [8]     CALL            4 2 1   ; 1 in 0 out
        12      [8]     RETURN          4 1 1   ; 0 out

function <test.lua:1,3> (4 instructions at 0x555fe0946000)
2 params, 3 slots, 0 upvalues, 2 locals, 0 constants, 0 functions
        1       [2]     ADD             2 0 1
        2       [2]     MMBIN           0 1 6   ; __add
        3       [2]     RETURN1         2
        4       [3]     RETURN0
```

以 C stack 类比，来看上述 Lua 代码对应的指令集输出。
其中最左侧一列是函数包含的指令序号，从 1 开始。其中 `6 slots` 表示占用栈上 6 个元素，从 0 到 5 。
main 就是 `main` 函数包含的指令列表，而 `function <test.lua:1,3>` 则是 `foo` 函数包含的指令列表。
在上述 Lua main chunk 代码中：

* 第 `5` 号指令，将函数 `foo` 的值从索引 `0` 移动到寄存器索引 `3` 中，为调用 `foo` 做准备。
* 第 `6` 和 `7` 号指令，分别将变量 `a` 和 `b` 移动到寄存器索引 `4` 和 `5` 处。
* 第 `8` 号指令调用 `foo` 函数。
* 第 `11` 号指令调用 `print` 函数。

此时便形成了函数 `foo` 的栈，其栈底是 `foo` 函数的值，接着是函数参数。 `CALL` 指令执行完毕后，虚拟机会把返回值压入栈上，然后弹出 `foo` 函数所有占用的其余的栈空间，如下所示：

* 第 `8` 号 `CALL` 指令后有注释 `2 in 1 out` 表示压入 2 参数，返回 1 个结果。 `foo` 函数栈的栈底由 `foo` 函数的值占用，由于只返回 1 个结果，因此函数执行完毕后，`foo` 函数的值所处的栈索引会替换为返回结果的值。
* 第 `9` 号指令将 `print` 函数的值移动到寄存器索引 `4` ，也就是之前变量 `a` 占用的空间，表示 `foo` 函数占用的空间已被回收。
* 第 `10` 号指令将 `foo` 函数的返回结果，从寄存器索引 `3` 移动到索引 `5` 处，为接下来函数调用做准备。寄存器索引 `3` 起初是 `foo` 函数的栈底，接着存放了 `foo` 函数返回的唯一的值。

用图示来说明上述栈空间。在执行第 `8` 号指令，调用 `foo` 函数时的栈空间如下所示。

* `main` 函数中的索引 `0` 到 `2` 是局部变量，按顺序是 `foo, a, b` 。
* `main` 函数中的索引 `3` 和 `4` 此时就是 `foo` 函数中的索引 `0` 和 `1` 。
* `foo` 函数中的索引 `0` 和 `1` 是函数参数 `a` 和 `b` 。索引 `2` 是计算后函数返回的结果。

```
main
 |
 f  0  1  2  3  4  5   <- main 函数栈空间的数组索引，f 表示 main 函数值
[0][1][2][3][4][5][6]  <- Lua stack 栈空间数组
          f  0  1  2   <- foo  函数栈空间的数组索引，f 表示 foo 函数值
          |
         foo
```

`foo` 函数执行完毕后返回结果，且栈空间被回收，此时栈空间如下所示。

* `foo` 函数执行完毕后，栈空间被回收，返回的结果值保存在之前 `foo` 函数值所处的索引处。

```
main
 |
 f  0  1  2  3  4  5   <- main 函数栈空间的数组索引，f 表示 main 函数值
[0][1][2][3][4][5][6]  <- Lua stack 栈空间数组
          r            <- r 表示 foo 返回的结果值
```

[The implementation of Lua 5.0](https://lua.org/doc/jucs05.pdf) 提到从 Lua5.0 开始，开始使用 register-based virtual machine 。如下所示：
> * This register-based machine also uses a stack, for allocating activation records, wherein the registers live.
> * When Lua enters a function, it preallocates from the stack an activation record large enough to hold all the function registers.
> * All local variables are allocated in registers.

以 C 类比，C stack 有 push/pop 指令进行栈操作，而 Lua stack 是函数调用前就预先分配好空间。
**再次说明：这里主要谈及 Lua 代码中进行函数调用的 Lua stack 细节，不包括在宿主代码（比如 C 代码）通过 Lua C API 进行函数调用时的 Lua stack 细节。无论是 Lua 代码还是宿主代码，关于 Lua stack 的整体概念是一致的，只是细节有些许不同**。

来看看 Lua stack 实现的细节，主要是两方面：数据和调用链。先看 `lua_State` 结构：

* Lua stack 也是数组，起始地址是 `L->stack` ，结尾是 `L->stack_last` 。而 `L->top` 指向栈顶，表示当前栈上第一个空闲槽位。
* Lua 函数调用链由 `L->ci` 表示，这是一个双向链表，`L->ci` 表示当前被调用的函数。通过 `L->ci->previous` 便可追溯完整的调用链。

```c
struct lua_State {
  ...
  StkIdRel top;    /* first free slot in the stack */
  CallInfo *ci;    /* call info for current function */
  StkIdRel stack_last;  /* end of stack (last element + 1) */
  StkIdRel stack;  /* stack base */
  ...
};
```

Lua 虚拟机为 Lua 代码中的函数调用生成 `OP_CALL` 指令（当然还存在其它的 CALL 指令，这里先忽略）。下面是一些实现细节：

* `luaD_precall` 进行函数调用前的准备工作，而 `luaD_poscall` 在函数执行结束后进行收尾工作。
* `luaD_precall` 调用 `prepCallInfo` 函数分配一个 `ci` 表示此函数调用，并链到调用链上，具体分配逻辑见 `next_ci` 。并且还会预先分配栈空间，见 `case LUA_VLCL` 分支中的代码。
* 在适当的时机调用 `luaD_growstack` 去扩容 Lua stack 。
* 在适当的时机调用 `luaD_shrinkstack` 去收缩栈，在收缩栈的同时会调用 `luaE_shrinkCI` 销毁不用的 `ci` 来收缩调用链。
* `luaD_poscall` 调用 `moveresults` 函数将函数返回结果压入栈，并回收不用的栈空间。此外，还执行 `L->ci = ci->previous;` 将自身从调用链上移除。

C stack 与 Lua stack 不同的是，C stack 使用的是进程划分好的一片地址空间，应用程序无需关注扩容和收缩。而 Lua stack 使用的是在堆上动态分配的数组，必然由 Lua VM 管理此数组空间，空间不足时进行扩容，空间过大时进行收缩。
仔细想，这里就引入了一个问题。 Lua stack 无论是进行扩容还是收缩时，栈上元素的地址会发生变更，此前旧的地址无效，必须采用新的地址。 Lua 通过 `relstack` 和 `correctstack` 函数解决地址变更问题。
Lua 一个很巧妙的设计是不会通过指针直接引用栈上元素的地址，想持有栈上的元素只会记录元素相对于栈起始地址的偏移，具体见 `savestack` 和 `restorestack` 。只要栈上该位置的元素所占用的未被回收，无论栈的地址如何变化，相对栈起始地址的偏移是不会变的，因此可以安全的持有此偏移值。

上述这些内容，不再去谈及具体的函数实现，有了整体的认识后，相关的细节自行去阅读或搜索，应该都可以找到答案。特别是就具体的细节与 AI 进行交互，也能有所收获。

## Lua 寄存器分配

接着简要解释说明前面引用 [The implementation of Lua 5.0](https://lua.org/doc/jucs05.pdf) 中的内容。

> * When Lua enters a function, it preallocates from the stack an activation record large enough to hold all the function registers.
> * All local variables are allocated in registers.

通常 Lua 在编译阶段生成虚拟机指令时就完成了寄存器的分配（再次提示这里仅涉及 Lua 代码，不包括 C API 去修改栈）。变长函数参数以及变长返回值是个例外，必须得等到执行时才确定函数参数的个数及返回值的个数。由于寄存器的分配与虚拟机指令的生成息息相关，细节又很多，下文简化讨论，仅简单描述局部变量的寄存器分配的概念。

[The implementation of Lua 5.0](https://lua.org/doc/jucs05.pdf) 提到 Lua 语法分析器是手写的 recursive descent parser 。递归下降的解析方式的特点是直接为每个语法规则编写解析函数。解析局部变量的函数则是 lparser.c 文件中的 `localstat` 函数。


例如编写 `local a, b, c` 声明局部变量时，语法解析器便会来到 `localstat` 函数。对于每个局部变量，会调用 `new_localvar` 记录此变量。解析完成后，会再调用 `adjustlocalvars` ，其中一个重点就是设置局部变量的寄存器索引 `var->vd.ridx = reglevel++;` 。
函数 `luaY_nvarstack` 返回需占用寄存器的局部变量数量，也即是局部变量可使用的下一个寄存器索引，由返回值可知，函数局部变量的寄存器索引从 0 开始。通过 `local var_name <const>` 声明的局部变量不包括在内，因为它不需要占用寄存器。

```c
/*
** Start the scope for the last 'nvars' created variables.
*/
static void adjustlocalvars (LexState *ls, int nvars) {
  FuncState *fs = ls->fs;
  int reglevel = luaY_nvarstack(fs);
  int i;
  for (i = 0; i < nvars; i++) {
    int vidx = fs->nactvar++;
    Vardesc *var = getlocalvardesc(fs, vidx);
    var->vd.ridx = reglevel++;
    var->vd.pidx = registerlocalvar(ls, fs, var->vd.name);
  }
}
```

该怎么理解呢？首先 Lua 中寄存器的概念本质就是栈，而 Lua 中栈本质就是数组，为局部变量分配寄存器就是从栈上分配槽位并返回索引标识此槽位。虽然每个函数使用的栈空只是是 Lua 一整个栈数组的一部分，但是每个函数的局部变量的寄存器索引从 0 开始。

```lua
local a = 1
local b, c <const> = a + a ^ 2 ^ 3 + _G.xxx[a * a] + 10, 20
local d = 16 + c
```

如上述 Lua 代码对应的指令集如下所示，其中局部变量 `v1, a, b` 对应的寄存器索引如下：

* `a` 的寄存器索引是 `0` ，对应指令是 ` 2 [1] LOADI 0 1` 。
* `b` 的寄存器索引是 `1` ，对应指令是 `14 [2] ADDI  1 1 10`
* `c` 使用 `<const>` 修饰，不占用寄存器。
* `d` 的寄存器索引是 `2` ，对应指令是 `16 [3] LOADI 2 36` ，注意这里解析时就读取 `c` 的值，然后进行常量表达式折叠计算得到 `36` 。

```
main <test.lua:0,0> (17 instructions at 0x55cff53b7cc0)
0+ params, 4 slots, 1 upvalue, 3 locals, 3 constants, 0 functions
        1       [1]     VARARGPREP      0
        2       [1]     LOADI           0 1
        3       [2]     POWK            1 0 0   ; 8.0
        4       [2]     MMBINK          0 0 10 0        ; __pow 8.0
        5       [2]     ADD             1 0 1
        6       [2]     MMBIN           0 1 6   ; __add
        7       [2]     GETTABUP        2 0 1   ; _ENV "_G"
        8       [2]     GETFIELD        2 2 2   ; "xxx"
        9       [2]     MUL             3 0 0
        10      [2]     MMBIN           0 0 8   ; __mul
        11      [2]     GETTABLE        2 2 3
        12      [2]     ADD             1 1 2
        13      [2]     MMBIN           1 2 6   ; __add
        14      [2]     ADDI            1 1 10
        15      [2]     MMBINI          1 10 6 0        ; __add
        16      [3]     LOADI           2 36
        17      [3]     RETURN          3 1 1   ; 0 out
```

通过上面例子，可以得到以下 Lua 的设计原则，后续可根据这些原则去品鉴具体的实现。

* 局部变量的寄存器索引从 `0` 开始连续分配，函数中 `adjustlocalvars` 对 `var->vd.ridx` 赋值也体现了索引是连续的（注 ridx 变则 register index 的缩写）。
* 完成对表达式的解析后会回收临时使用的寄存器，以支持后续重复使用未被局部变量占用的寄存器，见 `b` 的初始化表达式对应的指令集中使用到了寄存器索引 `3` ，但最终 `b` 使用的寄存器是索引 `1` 。
* 在 `struct FuncState` 中定义的 `freereg` 表示下一个可被使用的寄存器索引。此值增大时表示分配新的寄存器，减小时表示回收不用的寄存器，后续再增大时便复用了之前已用过的寄存器。
* 分配寄存器的函数是 `luaK_reserveregs` 而回收寄存器的函数是 `freereg` 。每次分配寄存器都会调用 `luaK_checkstack` 检查是否超出上限，并将占用的寄存器数量保存在 `maxstacksize` 中。**在执行阶段进行函数调用时，便会根据 `maxstacksize` 预先分配栈空间**。
* 当完成对表达式的解析后，便调用 `luaK_exp2nextreg` 将表达式的值分配到寄存器中。比如 `local a = b` 解析完 `b` 后，便会先分配一个新的寄存器，然后生成 `OP_MOVE` 指令，将 `b` 的值拷贝到新的寄存器索引对应的栈空间中。

局部变量必然是存放在栈上，按照 Lua 的概念便是在寄存器中。实际过程是为表达式的值分配寄存器存放结果，只是这时如果有关联的局部变量，则把此寄存器索引与此局部变量关联，此时便完成了对局部变量使用的寄存器的分配。
这个前提便是 Lua 解析器是顺序解析局部变量的声明以及对应的初始化表达式的。例如，当解析 `local a = 1, 2, 3, 4, 5` 时，解析器先是会分配 5 个寄存器去分别存放 `1-5` ，接着发现仅 1 个局部变量，此时便会回收 4 个不用的寄存器，见 `localstat` 函数调用 `adjust_assign` 中的 `fs->freereg += needed;  /* remove extra values */` 语句（这里并未调用 `freereg` 函数，而是直接减去不用的寄存器数量，注 `needed` 这时是负值）。

总结一下，本文先是提到了 Lua 代码的总体执行流程，然后描述了运行时 Lua stack 作为数组如何为函数调用提供服务，最后简要描述了 Lua 是如何在生成指令时就完成了对栈空间的分配。如果感兴趣，可以自行阅读对应代码，获取更多细节。

## 加深理解

下面举一个小例子来看 Lua 是如何应用内部的栈实现来加深理解。来看 `luaL_traceback` 是如何打印堆栈的，在 Lua 代码中常用的 `debug.traceback` 便是调用此函数。函数主体结构如下：不断的调用 `lua_getstack` 直到无法再获取到栈信息。

```c
while (lua_getstack(L1, level++, &ar)) {
  ...
  lua_getinfo(L1, "Slnt", &ar);
  ...
}
```

在 `lua_getstack` 函数中便能看见调用链，回想一下前面提到的调用链。
当将传入 `level=0` 时，此时获取的 ci 就是 `L->ci` ，即 `debug.traceback` 函数本身，传入 `level=1` 时得到的是 `L->ci->previous` ，即调用 `debug.traceback` 的函数。传入 level 的值越大，便能追溯到调用的源头。`L->base_ci` 是创建虚拟机时默认的 ci ，也就是调用链的源头。

```c
LUA_API int lua_getstack (lua_State *L, int level, lua_Debug *ar) {
  int status;
  CallInfo *ci;
  if (level < 0) return 0;  /* invalid (negative) level */
  lua_lock(L);
  for (ci = L->ci; level > 0 && ci != &L->base_ci; ci = ci->previous)
    level--;
  if (level == 0 && ci != &L->base_ci) {  /* level found? */
    status = 1;
    ar->i_ci = ci;
  }
  else status = 0;  /* no such level */
  lua_unlock(L);
  return status;
}
```

在拿到 ci 后，就可以通过 `lua_getinfo` 获取信息，通常 `luaL_traceback` 是不会打印调用链上函数的参数的。自己可以尝试扩展一下，支持打印函数的参数。提示：通过 `lua_getstack` 拿到 ci 后，可以通过 `lua_getinfo` 拿到函数参数的个数，然后调用 `lua_getlocal` 便可以获取到参数的值。