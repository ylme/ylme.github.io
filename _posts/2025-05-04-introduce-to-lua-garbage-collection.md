- [Lua GC 简介](#lua-gc-简介)
  - [基本概念](#基本概念)
    - [标记清除算法](#标记清除算法)
    - [Lua 中的可回收对象类型](#lua-中的可回收对象类型)
    - [Lua GC 中的根集 root set](#lua-gc-中的根集-root-set)
    - [Lua GC 中的标记](#lua-gc-中的标记)
    - [Lua GC 中的清除](#lua-gc-中的清除)
    - [Lua GC API](#lua-gc-api)
  - [增量 GC](#增量-gc)
    - [增量 GC's pace](#增量-gcs-pace)
    - [增量 GC's tri-color marking](#增量-gcs-tri-color-marking)
    - [增量 GC's barrier](#增量-gcs-barrier)
  - [分代 GC](#分代-gc)
    - [分代 GC's age](#分代-gcs-age)
    - [分代 GC's minor collection](#分代-gcs-minor-collection)
    - [分代 GC's barrier](#分代-gcs-barrier)
    - [分代 GC's pace](#分代-gcs-pace)
  - [GC 优化原则](#gc-优化原则)
  - [GC OOM?](#gc-oom)

# Lua GC 简介

截止到 Lua 5.4 ，其 GC 模块演变如下：

* until Version 5.0: a stop-the-world collector.
* Version 5.1: an incremental collector.
* Version 5.2: an experimental generational collector.
* Version 5.4: the experimental generational collector is deleted.
* Version 5.4: a generational collector comes again.

**本文基于 Lua 5.4.7** 主要介绍 Lua GC 的基本流程。希望读完后，能更好的理解 [Garbage Collection in Lua](https://lua.org/wshop18/Ierusalimschy.pdf) 和 [Lua GC 的工作原理](https://blog.codingnow.com/2018/10/lua_gc.html) 。  
如果自己想动手实现一个垃圾收集器可以参考 [Baby's First Garbage Collector](https://journal.stuffwithstuff.com/2013/12/08/babys-first-garbage-collector/) 。

## 基本概念

[Lua 5.4 手册](https://lua.org/manual/5.4/)中垃圾回收的基本概念如下：

> Lua performs automatic memory management.
> * This means that you do not have to worry about allocating memory for new objects or freeing it when the objects are no longer needed.
> * Lua manages memory automatically by running a garbage collector to collect all dead objects.
> * All memory used by Lua is subject to automatic management: strings, tables, userdata, functions, threads, internal structures, etc.
> * An object is considered dead as soon as the collector can be sure the object will not be accessed again in the normal execution of the program.
> * Note that the time when the collector can be sure that an object is dead may not coincide with the programmer's expectations.
> * The only guarantees are that Lua will not collect an object that may still be accessed in the normal execution of the program, and it will eventually collect an object that is inaccessible from Lua (Here, *inaccessible from Lua* means that neither a variable nor another live object refer to the object).

### 标记清除算法

Lua GC 模块基于标记清除 mark and sweep 算法进行垃圾回收：

* 先进行标记，通常从根集 root set 开始遍历并标记所有对象，被标记的对象表示该对象可达。
* 标记完成后开始清理，此时遍历所有对象（包括未被标记的对象）：
  - 若对象已被标记，则清理其标记。
  - 若对象未被标记，表示其不可达，已不再被使用，则清除此对象。
* 清除完成后，所有对象又都变回未标记状态，等待下一次垃圾回收。

### Lua 中的可回收对象类型

Lua 中的可回收对象类型是 `GCObject` ，并且统一通过 `luaC_newobj/luaC_newobjdt` 函数创建。

```c
/* Common type for all collectable objects */
typedef struct GCObject {
  CommonHeader;
} GCObject;
```

Lua 很细节的遵循 C 标准通过 `GCUnion` 进行对象类型的转换，而没有直接使用强制类型转换，如 `(Table*)o` 或 `(GCObject*)t` 。

* 比如调用 `obj2gco` 将具体的对象类型转换成 `GCObject*` 。
* 比如调用 `gco2t` 将 `GCObject*` 转换成具体的 Table 类型 `Table*` 。

```c
/*
** Union of all collectable objects (only for conversions)
** ISO C99, 6.5.2.3 p.5:
** "if a union contains several structures that share a common initial
** sequence [...], and if the union object currently contains one
** of these structures, it is permitted to inspect the common initial
** part of any of them anywhere that a declaration of the complete type
** of the union is visible."
*/
union GCUnion {
  GCObject gc;  /* common header */
  struct TString ts;
  struct Udata u;
  union Closure cl;
  struct Table h;
  struct Proto p;
  struct lua_State th;  /* thread */
  struct UpVal upv;
};
```

通过 `union GCUnion` 可查看到具体的可回收对象类型。对象是否能被垃圾收集器回收通过 `iscollectable` 进行判断：

* 对于不可被垃圾收集器回收的对象类型如 number ，该 macro 返回值一定等于 `0` 。
* 对于可被回收类型，该 macro 可能返回 `0` 。

```c
#define iscollectable(o) (rawtt(o) & BIT_ISCOLLECTABLE)
```

### Lua GC 中的根集 root set

在 `restartcollection` 函数中展示了根集包含哪些对象，可知：

* 主线程 `g->mainthread` 。
* 注册表 `g->l_registry` 即通过 `LUA_REGISTRYINDEX` 获取的 table 。
* 全局元表是 `lua_setmetatable` 函数为非 `"table"` 和 `"userdata"` 类型设置的元表，因为由虚拟机全局持有，所以也是根集，具体见 `markmt` 函数。
* `g->tobefnz` 链表也是根集的一部分，主要和 `__gc` 元方法相关，见 http://lua-users.org/lists/lua-l/2014-11/msg00294.html 。

```c
/*
** mark root set and reset all gray lists, to start a new collection
*/
static void restartcollection (global_State *g) {
  cleargraylists(g);
  markobject(g, g->mainthread);
  markvalue(g, &g->l_registry);
  markmt(g);
  markbeingfnz(g);  /* mark any finalizing object left from previous cycle */
}
```

### Lua GC 中的标记

通常，通过如下 macro 判断并标记对象。若对象是可回收的，并且是白色，则调用 `reallymarkobject` 函数标记对象。知识点：若对象是黑色或灰色，则 macro 中的 `if` 条件不满足，则不会调用 `reallymarkobject` 去标记。

```c
#define markvalue(g,o) { checkliveness(g->mainthread,o); \
  if (valiswhite(o)) reallymarkobject(g,gcvalue(o)); }

#define markkey(g, n) { if keyiswhite(n) reallymarkobject(g,gckey(n)); }

#define markobject(g,t) { if (iswhite(t)) reallymarkobject(g, obj2gco(t)); }

#define markobjectN(g,t)  { if (t) markobject(g,t); }
```

在 `reallymarkobject` 函数中，对于字符串类型，则直接设置为黑色，标记完成。对于类型为 `LUA_VLCL, LUA_VCCL, LUA_VTABLE, LUA_VTHREAD, LUA_VPROTO` 则先设置对象为灰色，然后链接到 `g->gray` 链表中，等待后续在 `propagatemark` 函数中具体处理。
想想这样设计也是一种巧妙的避免标记过程中产生死循环的办法，假设有两个 table `t1` 和 `t2` 相互引用，并且他们都被根集对象引用，如 `t1.key = t2; t2.key = t1; _G.t1 = t1; _G.t2 = t2;` 你会怎么实现标记呢？ 

```c
case LUA_VLCL: case LUA_VCCL: case LUA_VTABLE:
case LUA_VTHREAD: case LUA_VPROTO: {
  linkobjgclist(o, g->gray);  /* to be visited later */
  break;
}
```

在 `propagatemark` 函数中，先设置对象本身为黑色，然后根据对象类型，遍历对象引用的其它对象，调用上述 macro 进行标记。比如对于 table 则遍历其包含的 key 和 value 并标记。

### Lua GC 中的清除

在 lstate.h 注释中提到了一些链表，如下：

```c
/*
** Some notes about garbage-collected objects: All objects in Lua must
** be kept somehow accessible until being freed, so all objects always
** belong to one (and only one) of these lists, using field 'next' of
** the 'CommonHeader' for the link:
**
** 'allgc': all objects not marked for finalization;
** 'finobj': all objects marked for finalization;
** 'tobefnz': all objects ready to be finalized;
** 'fixedgc': all objects that are not to be collected (currently
** only small strings, such as reserved words).
*/
```

在清除阶段，遍历 `g->allgc` 链表，调用 `sweeplist` 函数进行清除处理。
对于设置 `__gc` 元方法的对象，包含如下处理：

* 在被设置 `__gc` 时，将其从 `g->allgc` 链表中移动到 `g->finobj` 链表中。
* 在其不可达时，将其从 `g->finobj` 链表移动到 `g->tobefnz` 链表中，等待调用 `__gc` 元方法。
* 在调用 `__gc` 元方法前，会将该对象从 `g->tobefnz` 链表移动到 `g->allgc` 链表中。
* 若其不再被使用，则等遍历 `g->allgc` 链表进行清除操作时被回收。

在 `sweeplist` 函数中，若对象未被标记，则 `if isdeadm(ow, marked)` 判断成立，执行清除操作，回收其内存。

```c
if (isdeadm(ow, marked)) {  /* is 'curr' dead? */
  *p = curr->next;  /* remove 'curr' from list */
  freeobj(L, curr);  /* erase 'curr' */
}
```

若对象已被标记，则设置其颜色为白色，等待下一次垃圾回收处理。

```c
curr->marked = cast_byte((marked & ~maskgcbits) | white);
```

### Lua GC API

无论在 Lua 代码中调用 `collectgarbage` 函数，还是 C 代码中调用 `lua_gc` 函数，都是由 gc 模块 lgc.c 提供支持。比如：

* `luaC_fullgc` 进行一次全量 GC 。
* `luaC_step` 进行一次 GC step 。
* `luaC_changemode` 修改 GC 模式。
* `luaC_fix` 设置对象不会被垃圾回收。

Lua 虚拟机内部是如何触发 GC 处理的呢？虚拟机运行时在特定的时机调用 macro `luaC_condGC` 尝试调用 `luaC_step` 函数触发一次 GC step 。如下所示仅 `g->GCdebt > 0` 时才会调用 `luaC_step` 函数。

```c
/*
** Does one step of collection when debt becomes positive.
*/
#define luaC_condGC(L,pre,pos) \
  { if (G(L)->GCdebt > 0) { pre; luaC_step(L); pos;}; \
    condchangemem(L,pre,pos); }
```

`global_State` 中如下字段与 GC 息息相关：

```c
typedef struct global_State {
  l_mem totalbytes;  /* number of bytes currently allocated - GCdebt */
  l_mem GCdebt;  /* bytes allocated not yet compensated by the collector */
  lu_mem GCestimate;  /* an estimate of the non-garbage memory in use */
} global_State;
```

* `totalbytes + GCdebt` 表示虚拟机内存分配总的字节数，见 `gettotalbytes` 定义。
* `GCdebt` 可以理解为 GC 欠虚拟机的债务。
  * 每次分配内存时 `GCdebt` 增加，见 `luaM_realloc_/luaM_malloc_` 函数。
  * 每次回收内存时 `GCdebt` 减少，见 `luaM_free_/luaM_realloc_` 函数。
  * 当 `GCdebt > 0` 时，可以理解为额度已经用完，需要进行垃圾处理还债。
  * 当 `GCdebt < 0` 时，可以理解为还有额度，所以不用进行垃圾回收处理。
* `GCestimate` 表示非垃圾对象占用的字节数。

当 `GCdebt` 为负数时，值越小则下一次触发 GC 则需要更长时间。虚拟机目前仅根据内存分配去评估 GC 触发（比如没有时间机制去触发） ，而 `GCdebt` 在此过程中发挥着重要作用。后续会再详细介绍 `GCdebt` 。  
这里先思考一个问题：**为什么 Lua 不用 `totalbytes` 表示虚拟机内存分配总的字节数呢，而是要通过 `totalbytes + GCdebt` 计算得到**？  
我个人理解是，如果用 `totalbytes` 表示总的字节数，那么每次内存变更时，除了要更新 `GCdebt` 还要更新 `totalbytes` 。但是现在的实现方式，每次内存变更时仅需更新 `GCdebt` 即可，可以节约 1 次算术运算。

## 增量 GC

[Lua 5.4 手册](https://lua.org/manual/5.4/)中关于增量 GC 的描述如下：

> In incremental mode, each GC cycle performs a mark-and-sweep collection in small steps interleaved with the program's execution. 

相比 stop-the-world GC 实现，增量 GC 的特点就是减缓了 GC 停顿，将 GC 操作分成一个个 GC step ，如下所示，与程序的执行交织在一起：

* 一次完整的 GC cycle 被分割成了一个个的 GC step 。
* 增量 GC 并未减少 GC 开销，只是减缓了 GC 停顿。

```
(pause)--(step)--(step)--(step)(pause)--(step)-- ...
```

增量 GC 被分成一个个 GC step 后，由 `g->gcstate` 控制 GC 状态，包含的状态如下所示：

```c
#define GCSpropagate  0
#define GCSenteratomic  1
#define GCSatomic 2
#define GCSswpallgc 3
#define GCSswpfinobj  4
#define GCSswptobefnz 5
#define GCSswpend 6
#define GCScallfin  7
#define GCSpause  8
```

各状态与 GC 阶段对应关系如下：

* `GCSpropagate` 对应**标记阶段**。在此过程中，所有可达的对象被标记为黑色或灰色。
* `GCSenteratomic` 和 `GCSatomic` 字面意义表示**原子阶段**，由 `atomic` 函数实现：
  - 遍历并标记所有灰色对象。
  - 处理带有 `__gc` 元方法的对象。
  - 处理弱表。
  - **`atomic` 函数执行完毕后，所有灰色对象都被标记**。
* `GCSswpallgc` 对应**清除阶段**。在此过程中，遍历 `g->allgc` 链表进行清除处理。
* `GCSswpfinobj` 和 `GCSswptobefnz` 清除带有 `__gc` 的对象，实际仅会将对象颜色设置为白色（想想为什么）。
* `GCSswpend` 表示清除阶段结束：**所有可回收对象都变成白色，之前已创建的对象被设置为白色，新创建的对象本身就是白色**。
* `GCScallfin` 调用 `__gc` 元方法。
* `GCSpause` 表示本次 GC cycle 完成，等待下一次 GC 。

`keepinvariant` 表示此 GC 状态期间，需要保持 GC 不变性原则，即黑色对象不能指向白色对象。黑色对象是该对象及其引用的对象都已被标记，倘若其又指向白色对象，显然与黑色对象的定义相矛盾。
标记阶段需要遵循该原则。因为标记阶段被标记为黑色的对象，可能再次被修改，从而可能打破该原则。因此 Lua 引入了 barrier 机制，监听对象修改并及时做处理，来维持该不变性原则。

```c
#define keepinvariant(g)  ((g)->gcstate <= GCSatomic)
```

总结如下：

* `keepinvariant(g)` 期间需要维持不变性原则。
* `atomic` 函数执行完时，所有的标记处理才完成，所有可回收对象都变成黑色。但是也要注意 `atomic` 函数被调用时有可能会带来比较长的 GC 停顿。这也是增量 GC 中不可控制 GC 执行的 pace 的地方。
* 在清理阶段过程中，还未被清理的对象有可能从黑色变成灰色，比如 table 被修改，如 `t.x = {}` 。新创建的对象是白色（并不是未被标记的那种白色，后面会说明）。
* 等状态切换到 `GCSswpend` 时，清理阶段完成，所以可回收对象都变成白色。

### 增量 GC's pace

pace 就是指 GC 执行的步调，具体是指：

* 当 GC pause 时，何时开始新一轮的 GC cycle 。
* 已经处于 GC cycle 中时，何时开始一次的 GC step ，以及在一次 GC step 中应该做多少工作。

根据 Lua 手册可知，有三个参数可以用来控制 pace 。无论是何时开始新一轮 GC cycle 还是何时开始一次 GC step ，本质都是设置 `g->GCdebt` 值。

> In this mode, the collector uses three numbers to control its garbage-collection cycles: the garbage-collector pause, the garbage-collector step multiplier, and the garbage-collector step size.

`luaE_setdebt` 函数用于设置 debt ，但是每次设置都会保证 `totalbytes + GCdebt` 是不变的，因为这是当前虚拟机内存分配总的字节数。
Lua 仅通过内存分配控制 GC 的执行。当暂时不想执行 GC 时（一轮 GC cycle 结束或一次 GC step 结束），就将 debt 设置为负数，随着程序执行过程中内存分配导致 debt 增加从而变成正数时，此时便会触发 GC 。

```c
/*
** set GCdebt to a new value keeping the value (totalbytes + GCdebt)
** invariant (and avoiding underflows in 'totalbytes')
*/
void luaE_setdebt (global_State *g, l_mem debt) {
  l_mem tb = gettotalbytes(g);
  lua_assert(tb > 0);
  if (debt < tb - MAX_LMEM)
    debt = tb - MAX_LMEM;  /* will make 'totalbytes == MAX_LMEM' */
  g->totalbytes = tb - debt;
  g->GCdebt = debt;
}
```

> * The garbage-collector pause controls how long the collector waits before starting a new cycle.
> * Values equal to or less than 100 mean the collector will not wait to start a new cycle.
> * A value of 200 means that the collector waits for the total memory in use to double before starting a new cycle. 

当一轮 GC cycle 完成时，调用 `setpause` 函数，确认何时开始新的一轮 GC cycle 。

* `g->GCestimate` 在 GC 切换到 `GCSswpallgc` 状态即进入清除阶段时，先赋值为当前虚拟机内存分配总的字节数。
  * 在清除阶段过程中，函数 `sweepstep` 不断的更新 `g->GCestimate` 减去回收的字节数。
  * 当 GC cycle 完成时 `g->GCestimate` 便是除去垃圾对象后分配的字节数。
  * 注意这时并不是指虚拟机当前没有垃圾了。只有等下一轮 GC cycle 开始时，才能去找出垃圾。
* 为什么 pause 为 `200` 表示双倍呢？因为 `PAUSEADJ` 是 `100` 。最终公式是 `g->GCestimate / 100 * 200 = g->GCestimate * 2` 。
* 为什么 pause 小于等于 `100` 时会立刻开始新一轮的 GC cycle 呢？因为此时一定是  `g->GCestimate <= gettotalbytes(g)` ，计算 `gettotalbytes(g) - threshold` 得到的 debt 肯定是大于等于 0 。后续内存分配很快就会让 `g->GCdebt > 0` ，也就会触发 GC 的执行。

```c
/*
** Set the "time" to wait before starting a new GC cycle; cycle will
** start when memory use hits the threshold of ('estimate' * pause /
** PAUSEADJ). (Division by 'estimate' should be OK: it cannot be zero,
** because Lua cannot even start with less than PAUSEADJ bytes).
*/
static void setpause (global_State *g) {
  l_mem threshold, debt;
  int pause = getgcparam(g->gcpause);
  l_mem estimate = g->GCestimate / PAUSEADJ;  /* adjust 'estimate' */
  lua_assert(estimate > 0);
  threshold = (pause < MAX_LMEM / estimate)  /* overflow? */
            ? estimate * pause  /* no overflow */
            : MAX_LMEM;  /* overflow; truncate to maximum */
  debt = gettotalbytes(g) - threshold;
  if (debt > 0) debt = 0;
  luaE_setdebt(g, debt);
}
```

下面再来看 step multiplier 和 step size ，见 Lua 手册：

> * The garbage-collector step multiplier controls the speed of the collector relative to memory allocation, that is, how many elements it marks or sweeps for each kilobyte of memory allocated.
> * Larger values make the collector more aggressive but also increase the size of each incremental step.
> * The garbage-collector step size controls the size of each incremental step, specifically how many bytes the interpreter allocates before performing a step.

`incstep` 函数控制每次 GC step 调用。这里会将字节数转换成 units of work 即对象的个数，来衡量 step size 。
注意：当 `gcstate` 是 `GCSpropagate` 状态时，即处于标记阶段期间，一定要保证每次 step 处理标记足够多的对象，也就是保证标记速度大于对象创建的速度，这样才能最终标记完所有对象。否则就会一直处于标记阶段，永远无法推进到清除阶段。
手册中提到若 step multiplier 小于 `100` ，有可能会永远无法完成一个 GC cycle 。我认为可能就是指标记速度太慢导致的。

手册中描述 step multiplier 表示每分配 1 KB 字节，需要标记或清除的对象个数。低于 `100` 时可能用于无法完成一个 GC cycle 。那么 `100` 又是如何得来的呢？我自己理解是 `1024 / WORK2MEM` 估算得来的，即 `1024 / sizeof(TValue)` 的上限肯定不会超过 100 。
再来看 `incstep` 函数对 stepsize 的计算是 `stepsize = stepbytes / WORK2MEM * stepmul` ，因此若 step multiplier 不低于 `100` ，`stepsize` 肯定满足标记速度大于创建速度。
手册中提到 step size 是以 2 为底的指数计算得到的字节数，因此 `incstep` 进行计算 `(cast(l_mem, 1) << g->gcstepsize) / WORK2MEM` 转换成需要处理的对象数量。

```c
/*
** Performs a basic incremental step. The debt and step size are
** converted from bytes to "units of work"; then the function loops
** running single steps until adding that many units of work or
** finishing a cycle (pause state). Finally, it sets the debt that
** controls when next step will be performed.
*/
static void incstep (lua_State *L, global_State *g) {
  int stepmul = (getgcparam(g->gcstepmul) | 1);  /* avoid division by 0 */
  l_mem debt = (g->GCdebt / WORK2MEM) * stepmul;
  l_mem stepsize = (g->gcstepsize <= log2maxs(l_mem))
                 ? ((cast(l_mem, 1) << g->gcstepsize) / WORK2MEM) * stepmul
                 : MAX_LMEM;  /* overflow; keep maximum value */
  do {  /* repeat until pause or enough "credit" (negative debt) */
    lu_mem work = singlestep(L);  /* perform one single step */
    debt -= work;
  } while (debt > -stepsize && g->gcstate != GCSpause);
  if (g->gcstate == GCSpause)
    setpause(g);  /* pause until next cycle */
  else {
    debt = (debt / stepmul) * WORK2MEM;  /* convert 'work units' to bytes */
    luaE_setdebt(g, debt);
  }
}
```

此时再看 `setpause` 和 `incstep` 中调用 `luaE_setdebt` 设置 debt 。应该就明白了增量 GC's pace 的运作机制吧。

### 增量 GC's tri-color marking

lgc.h 中关于三色标记注释：

> Collectable objects may have one of three colors:
> * white, which means the object is not marked;
> * gray, which means the object is marked, but its references may be not marked;
> * black, which means that the object and all its references are marked.
> The main invariant of the garbage collector, while marking objects, is that a black object can never point to a white one.

* 代码中定义了两种白色，分别对应未被标记的对象的白色（会被回收），以及新创建对象的白色和清除阶段完成后的对象的白色（不会被回收）。
* 当一个 table 被完全标记后，变成黑色，但在标记阶段的过程中，该 table 被修改，如 `t.x = {}` 则该 table 此时变成灰色，并被链到 `g->grayagain` 链表，等待后续再被标记，因为黑色对象不能指向白色对象。
* `struct GCObject` 的 `marked` 字段存放 GC 相关的标记，其中仅 3 位用于记录颜色标记，并且对象是否死亡也是使用颜色位进行判断，所以这 3 位表示了 4 种形态：白色，黑色，灰色，是否死亡。
* `struct global_State` 的 `currentwhite` 字段表示当前全局的白色标记，通过 `luaC_white(g)` 获取，而 `otherwhite(g)` 表示上一次的白色标记。每次赋值时采用 flip current white 的方式去设置 `g->currentwhite` 。

如下代码是颜色的 macro 定义，以及用于判断对象颜色以及对象是否死亡的判断：

```c
#define WHITE0BIT 3  /* object is white (type 0) */
#define WHITE1BIT 4  /* object is white (type 1) */
#define BLACKBIT  5  /* object is black */

#define iswhite(x)      testbits((x)->marked, WHITEBITS)
#define isblack(x)      testbit((x)->marked, BLACKBIT)
#define isgray(x)  /* neither white nor black */  \
  (!testbits((x)->marked, WHITEBITS | bitmask(BLACKBIT)))
#define isdeadm(ow,m) ((m) & (ow))
#define isdead(g,v) isdeadm(otherwhite(g), (v)->marked)
```

下面举例说明 `o->marked` 字段中颜色位的设置以及 `g->currentwhite` 如何会影响对象是否是白色的判断。当理解 `isdead(g, v)` 实现后，就能感受到这里的巧妙。下面的 x 或者 y 位表示对应 `o->marked` 字段的其它位，和颜色位无关，忽略即可。

```
WHITE0 xx001yyy
WHITE1 xx010yyy
BLACK  xx100yyy
GRAY   xx000yyy
```

假设初始时 `g->currentwhite` 是 WHITE0 。这期间创建的可回收对象的颜色也都是 WHITE0 。等所有对象都被标记完后，则反转赋值 `g->currentwhite` 为 WHITE1 。注意 `g->currentwhite` 的值只可能是 `WHITE0 00001000` 或 `WHITE1 00010000` 。

* 对于 `WHITE0 xx001yyy` 颜色对象（未被标记的垃圾对象），`isdeadm` 判断是 `otherwhite(g) & xx001yyy` 简化成 `001 & 001` ，结果不为 `0` ，表示此对象已死亡。
* 对于 `WHITE1 xx010yyy` 颜色对象（未被标记的新创建的对象），`isdeadm` 判断是 `otherwhite(g) & xx010yyy` 简化成 `001 & 010` ，结果为 `0` ，表示对象仍存活。
* 对于 `BLACK xx100yyy` 对象，`isdeadm` 简化后的判断是 `001 & 100` ，结果为 `0` ，表示对象仍存活。
* 对于 `GRAY xx000yyy` 对象，`isdeam` 简化后的判断是 `001 & 000` ，结果为 `0` ，表示对象仍存活。

### 增量 GC's barrier

前面提到，根据 `keepinvariant` macro 定义，增量 GC 标记阶段，需要保证黑色对象不能指向白色对象。而 barrier 就是实现这种保证的机制，监控对象的修改，并采取相应的处理。

* `luaC_barrier_` 函数表示 move collector forward ，因为将新对象进行标记，相当于推进垃圾收集器向前工作。
  - 需要对象 p 是黑色和其指向的对象 o 是白色才会被调用。
  - 若 `keepinvariant` 返回 true ，那么会对 o 进行标记，确保 o 不是白色。
* `luaC_barrierback_` 函数表示 move collector backward ，因为将已标记完成的黑色对象变成灰色。
  - 需要对象 p 是黑色和其指向的对象 o 是白色才会被调用。
  - 然后将对象 p 设置成灰色链到 `g->grayagain` 链表中等待后续处理。
  - move collector backward 主要针对 table 。想想如果对 table 采用 forward 方式，每次 table 被修改时都会触发 barrier 。而采用 backward 方式，只会对一个标记完成的 table 触发一次 barrier 。

注意，虽然增量 GC 的不变性原则仅需在标记阶段保证，但 `luaC_barrier_` 函数和 `luaC_barrierback_` 函数并不是仅在标记阶段才会被调用。

## 分代 GC

手册中关于分代 GC 的描述如下：

> In generational mode, the collector does frequent minor collections, which traverses only objects recently created. If after a minor collection the use of memory is still above a limit, the collector does a stop-the-world major collection, which traverses all objects.

* minor collection 仅会遍历最近创建的对象并清理其中的垃圾对象。
* major collection 遍历所有对象并清除其中的垃圾，由于过程中 stop-the-world ，因此可能会产生比较长的 GC 停顿。
* 当经过足够长的一段时间后，有些对象不再被使用，该对象仅会在 major collection 中被清理。
* 就 minor collection 而言，由于只遍历最近创建的对象做清理，相比增量 GC 其实减少了 GC 的开销。

### 分代 GC's age

分代 GC 中定义年龄的定义如下：

```c
/* object age in generational mode */
#define G_NEW   0 /* created in current cycle */
#define G_SURVIVAL  1 /* created in previous cycle */
#define G_OLD0    2 /* marked old by frw. barrier in this cycle */
#define G_OLD1    3 /* first full cycle as old */
#define G_OLD   4 /* really old object (not to be visited) */
#define G_TOUCHED1  5 /* old object touched this cycle */
#define G_TOUCHED2  6 /* old object touched in previous cycle */
```

下面引用 lgc.h 中的一段注释说明，下面其中一部分，解释了分代 GC 的基础 age 概念。（备注：截止到 Lua 5.4.7 发布时 lgc.h 中还未包含该注释）。

```c
/*
** In generational mode, objects are created 'new'. After surviving one
** cycle, they become 'survival'. Both 'new' and 'survival' can point
** to any other object, as they are traversed at the end of the cycle.
** We call them both 'young' objects.
** If a survival object survives another cycle, it becomes 'old1'.
** 'old1' objects can still point to survival objects (but not to
** new objects), so they still must be traversed. After another cycle
** (that, being old, 'old1' objects will "survive" no matter what)
** finally the 'old1' object becomes really 'old', and then they
** are no more traversed.
*/
```

* 初始创建时对象的 age 是 `G_NEW` 。经过一轮 minor collection 后，还存活的对象的 age 变成 `G_SURVIVAL` 。
* 前面手册提到的 objects recently created 就是注释中提到的 young objects ，即 `G_NEW` 和 `G_SURVIVAL` 年龄的对象。它们在每次进行 minor collection 时都会被遍历。
* 一般情况下，对象年龄变更顺序是 `G_NEW -> G_SURVIVAL -> G_OLD1 -> G_OLD` 。
  - 当对象变成 `G_SURVIVAL` 年龄时，其可能指向任意年龄的对象。而 `G_SURVIVAL` 年龄对象本身就是会被遍历的 young objects ，所以其引用的 `G_NEW` 或 `G_SURVIVAL` 年龄对象也会被遍历。
  - 当对象变成 `G_OLD1` 年龄时，其还有可能指向 `G_SURVIVAL` 年龄的对象，所以还要遍历 `G_OLD1` 对象目的是为了标记其引用的 `G_SURVIVAL` 对象。
  - 当对象变成 `G_OLD` 时，便不会再被遍历。

因此，通常来说，对象从创建开始，经历 2 次 minor collection 便变成 old 年龄的对象，此时是 `G_OLD1` 年龄。但此时还需再被遍历 1 次，即再经历 1 次 minor collection 。为了标记 `G_OLD1` 对象指向 `G_SURVIVAL` 对象。
然后就变成真正的 old 年龄对象，即 `G_OLD` 年龄。此时 `G_OLD` 对象不会指向 young objects ，所以也就无需再被遍历。

### 分代 GC's minor collection

Lua 中分代 GC 是在原有的三色标记基础之上实现的。也就是对于分代 GC 而言，除了要关注对象的颜色，还需要关注对象的年龄。分代 GC 中标记清除最大的不同是：**没有了清除阶段将存活的对象的颜色设置为白色的操作**。而 Lua 中的标记调用 `markvalue/markkey/markobject/markobjectN` 需要对象是白色才会调用 `reallymarkobject` 函数进行标记。所以，分代 GC 需要额外管理标记操作。

见 lgc.h 中关于分代 GC 中颜色的注释说明（备注：截止到 Lua 5.4.7 还未包含此注释）：

```c
/*
** The generational mode must also control the colors of objects,
** because of the barriers.  While the mutator is running, young objects
** are kept white. 'old', 'old1', and 'touched2' objects are kept black,
** as they cannot point to new objects; exceptions are threads and open
** upvalues, which age to 'old1' and 'old' but are kept gray. 'old0'
** objects may be gray or black, as in the incremental mode. 'touched1'
** objects are kept gray, as they must be visited again at the end of
** the cycle.
*/
```

minor collection 的具体实现是 `youngcollection` 函数。

* `G_NEW` 对象本身就是白色，所以一定会被 GC 标记。对于 `G_SURVIVAL` 对象，是在 `sweepgen` 函数中，将其颜色设置为白色。

```c
if (getage(curr) == G_NEW) {  /* new objects go back to white */
  int marked = curr->marked & ~maskgcbits;  /* erase GC bits */
  curr->marked = cast_byte(marked | G_SURVIVAL | white);
}
```

* `markold` 函数将对象的年龄从 `G_OLD1` 改为 `G_OLD` 。若对象是黑色，则调用 `reallymarkobject` 标记对象。若对象是灰色，则会在后续的 `atomic` 函数中被遍历进行标记。确保其指向的 `G_SURVIVAL` 对象能被遍历。

```c
if (getage(p) == G_OLD1) {
  lua_assert(!iswhite(p));
  changeage(p, G_OLD1, G_OLD);  /* now they are old */
  if (isblack(p))
    reallymarkobject(g, p);
}
```

### 分代 GC's barrier

Barrier 是为了保证不变性原则。分代 GC 的不变性原则是 old objects 不能指向 young objects ，准确描述是 `G_OLD` 对象不能指向 `G_NEW` 或 `G_SURVIVAL` 对象。因为设计就是 `G_OLD` 对象不再被遍历，所以如果指向了 young objects ，会导致 young objects 不能被标记，从而产生问题。同增量 GC 类似，分代 GC 也有 forward 和 backward barrier 去维持不变性原则。

见 lgc.h 中关于分代 GC 中 barrier 的说明：（备注：截止到 Lua 5.4.7 还未包含此注释）

```c
/*
** To keep its invariants, the generational mode uses the same barriers
** also used by the incremental mode. If a young object is caught in a
** forward barrier, it cannot become old immediately, because it can
** still point to other young objects. Instead, it becomes 'old0',
** which in the next cycle becomes 'old1'. So, 'old0' objects is
** old but can point to new and survival objects; 'old1' is old
** but cannot point to new objects; and 'old' cannot point to any
** young object.
**
** If any old object ('old0', 'old1', 'old') is caught in a back
** barrier, it becomes 'touched1' and goes into a gray list, to be
** visited at the end of the cycle.  There it evolves to 'touched2',
** which can point to survivals but not to new objects. In yet another
** cycle then it becomes 'old' again.
*/
```

`G_OLD0` 是处理新创建的对象，既要避免新对象在 `sweepgen` 函数中被设置为白色（因为新对象是被老对象指向的），也要遍历新对象本身，因为它的指向 young objects 需要被遍历。

* `luaC_barrier_` 函数中会标记新对象，并将其年龄设置为 `G_OLD0` 。
* 接着在 minor collection 中的 `sweepgen` 函数，其年龄从 `G_OLD0` 变成 `G_OLD1` 。本次 GC 完成时，它指向的 young objects 仅是 `G_SURVIVAL` 对象。
* 在下一次的 minor collection 时，其变成 `G_OLD1` ，同上文描述的处理一致。

`G_TOUCHED1, G_TOUCHED2` 针对 `G_OLD` 年龄的 table 指向了新创建对象。

* `luaC_barrierback_` 函数中将对象年龄从 `G_OLD` 改为 `G_TOUCHED1` 。并且将其颜色改为灰色，并将对象链到 `g->grayagain` 链表中。该对象指向的新创建对象则是白色和 `G_NEW` 年龄。
* 接着在 minor collection 中的 `atomic` 函数，遍历 `g->grayagain` 链表进行遍历并标记。并且在标记过程中，调用 `genlink` 函数，再次将该对象链到 `g->grayagain` 链表中，确保下一次 GC 依旧能遍历此对象。因为本次 GC 完成时，该对象还有可能指向 `G_SURVIVAL` 对象。本次 GC 完成时，在 `correctgraylist` 函数中，将对象年龄从 `G_TOUCHED1` 改为 `G_TOUCHED2` 。
* 在下一次的 minor collection 中，对象年龄从 `G_TOUCHED2` 变回了 `G_OLD` 。此时，该对象便不再指向 young objects 。

### 分代 GC's pace

Lua 手册中提到有两个参数控制分代 GC 的 pace ，如下所示：

> * The minor multiplier controls the frequency of minor collections. For a minor multiplier x, a new minor collection will be done when memory grows x% larger than the memory in use after the previous major collection.
> * The major multiplier controls the frequency of major collections. For a major multiplier x, a new major collection will be done when memory grows x% larger than the memory in use after the previous major collection.

增量 GC 则不存在像增量 GC 那样复杂的内部状态，在每个 step 中调用 `genstep` 函数，要么进行 minor collection ，要么进行 major collection 。当 step 执行结束，本次的分代 GC 也就执行结束。但是 major collection 又被成为 bag collection ，因为会进行全量 GC 操作，可能触发比较长的 GC 停顿。
一次 step 执行结束，也是调用 `luaE_setdebt` 设置 debt ，等待 debt 增长触发下一次 step 。

```
--(step)--(step)--(step)-- ...
```

当对象的生命期主要在 2 个 minor collection 内时，这时才利好分代 GC 。而 old 对象，只能依赖在内存增长到一定程序时，通过 major collection 回收。
实际游戏服务器实际的业务需求，似乎并不太符合这种模式：

* 登录时会创建大量的对象，登出时销毁。这期间的对象大概率都会变成 old 对象。
* LRU 缓存也通常是在一定时间后过期，销毁内存中的数据。这期间的对象大概率也是会变成 old 对象。

## GC 优化原则

Roberto Ierusalimschy 在 PIL 中提及，在 [Garbage Collection in Lua](https://lua.org/wshop18/Ierusalimschy.pdf) 和 [Lua GC 的工作原理](https://blog.codingnow.com/2018/10/lua_gc.html) 也提到过类似的观点：

> Any garbage collector trades memory for CPU time.
> * At one extreme, the collector might not run at all. It would spend zero CPU time, at the price of a huge memory consumption.
> * At the other extreme, the collector might run a complete cycle after every single assignment. The program would use the minimum memory necessary, at the price of a huge CPU consumption. 

GC 优化的本质就是空间换时间。假如想减缓 GC 停顿或是减少 GC 的 CPU 开销，而这时进程内存处于健康线以下时，可以控制 GC 的 pace 减少作业量，以进程占用的内存增加为代价，来换取 GC 的 CPU 开销降低。比如新开服时，前 2 个小时，进行此优化，之后再将 GC 的 pace 改回正常步调。

PIL 也有提及具体的手段：

* 调整 GC 参数，理解了前文关于 GC's pace 的说明，应该就能根据实际需要去做调整。
* 关闭 Lua GC ，业务自身通过 `collectgarbage("step", n)` 自行触发 GC 。 

## GC OOM?

顺便提一下，我司有个线上的游戏服务器发生过 OOM 。用的增量 GC 并且 pause 是默认值 `200` 即内存占用 double 时开始下一次垃圾回收，然后当时触发了某个操作导致短时间内进程的内存急剧增大，然后间接的导致下一次垃圾回收需要内存占用很大时才会执行，但此时已经超过机器的内存上限，从而发生了 OOM 。其实，游戏服务器好几个服务的内存占用都在 20GB 往上，所以 pause 为 `200` 确实有可能发生 OOM 的风险。  
之后，项目中将大部分服务的 pause 设置成 `100` 。这样当一轮增量 GC 执行完毕后，立刻开始下一次的垃圾回收。

Lua 5.4 分代 GC 的 major collection 的 major multiplier 默认值是 `100` ，也是当下一次内存占用 double 时触发，可能也需要注意是否可能发生 OOM 。  
其实，总结就是当进程内存处于高位时，比如占用内存在总内存的 70% 以上时，采用内存占用 double 后才触发 GC 的策略，有可能会导致还没有来得及 GC 就已经达到了机器的内存上限，从而发生 OOM 。

此外，对于大部分业务都使用 Lua 开发的进程，还可以活用 Lua emergency GC ，参考 [skynet lalloc](https://github.com/cloudwu/skynet/blob/master/service-src/service_snlua.c#L489) 。

* 定制 lua_State 内存分配函数，并在设置 `mem_limit` 。
* 当内存分配字节数超过 `mem_limit` 时，返回 `NULL` 从而触发 emergency GC 。

```c
static void *
lalloc(void * ud, void *ptr, size_t osize, size_t nsize) {
  struct snlua *l = ud;
  size_t mem = l->mem;
  l->mem += nsize;
  if (ptr)
    l->mem -= osize;
  if (l->mem_limit != 0 && l->mem > l->mem_limit) {
    if (ptr == NULL || nsize > osize) {
      l->mem = mem;
      return NULL;
    }
  }
  if (l->mem > l->mem_report) {
    l->mem_report *= 2;
    skynet_error(l->ctx, "Memory warning %.2f M", (float)l->mem / (1024 * 1024));
  }
  return skynet_lalloc(ptr, osize, nsize);
}
```

这里 `mem_limit` 要选择合适的值，预估一个业务使用的内存上限，并且低于机器的内存上限。保证向操作系统申请内存分配时不会失败，当 Lua 虚拟机内存不断上升达到 `mem_limit` 时触发 emergency GC ，整个进程还能正常运行下去。
