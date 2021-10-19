# 一、概述

CMS垃圾收集器是一款用于**老年代**的，使用清除-整理算法的垃圾收集器。

对于 CMS，GC ROOTS 包含：statck、register、gloabls 和 **年轻代**。

> 要注意：CMS 是一个老年代的收集器，不是整个堆的垃圾收集器，既然只收集老年代，它必须把当前处于非收集区域的年轻代算作是 GC ROOTS。这跟年轻代 GC 时要把老年代的脏卡算作 GC ROOTS 的道理一样，只不过 HotSpot 没有用 card table 来记录 young -> old 引用，所以就干脆扫描整个年轻代作为 GC ROOTS。

名词解释：

mutator：把GC之外的代码（主要是应用程序的逻辑）叫做mutator

collector：把GC的代码叫做collector

# 二、认识 ParNew

要知道 CMS 垃圾处理器是用来回收老年代的垃圾对象的，而年轻代中的垃圾对象怎么回收呢？需要搭配使用 ParNew 收集器进行年轻代中垃圾对象的回收，ParNew GC 就是 Serial GC 的多线程版本，垃圾收集的时候需要 STW。

这里有一个问题：YGC 的时候，如何解决跨代引用的问题？（这个问题应该是最简单的，网上一搜一大堆）

所谓跨代引用实际上就是一个对象在年轻代中没有被引用，但是却在老年代中被引用了，如果只是扫描年轻代的话，这个对象就会被标记为垃圾对象，就会被回收，导致系统错误。

如何解决这个问题呢？使用的技术就是**卡表（card table）**，当然还有另一种方法就是扫描整个老年代，那这样 YGC 就是扫描整个堆内存，代价太高了，所以使用了一种空间换时间的思想。

卡表：把堆内存分为一个个大小为 512KB 的区域，称之为**卡（card）**，卡表的结构通常使用 byte 数组表示，一个 byte 对应一个卡，如果 byte 的值为 0 表示这个卡是干净的（没有引用的变更），1 表示有引用变更了。

每次 YGC 结束的时候，会重置卡表，此时卡表对应的卡全是干净的，程序运行时，如果某个卡对应的内存区域有引用的变更，则修改卡表，将对应下标的值修改为 1，此时这块卡是脏卡（dirty card）。

当 YGC 开始时，先 STW，然后根据 GC ROOTS（stack、register、globals）扫描年轻代，扫描完后，就到了问题的核心了，跨代引用的对象怎么办呢？扫描卡表中老年代区域的卡，如果是脏卡，就去扫描这个脏卡表示的区域，这样避免了扫描整个老年代了。

YGC 结束后，将卡表重置，再次进行标记，再次重置，以此往复。

# 三、三色标记

实际上三色标记中的色并不只是颜色，而是我们对它的一种叫法，但是实际上他不会用 red white 这种东西来定义，而是用例如 0,1,2 这样的值来定义不同的对象。

其实三色标记就是我们 CMS 在扫描过程中对对象的一种定义。那么具体的定义如下：

- 黑色：1、对象本身及其引用都被扫描过；2、黑色不能直接指向白色；3、不能被回收
- 灰色：1、对象本身被扫描；2、还存在至少一个引用没有被扫描
- 白色：1、没有被扫描；2、扫描结束后被回收

**标记过程**：

1. 初始时，所有对象都是白色
2. 将 GC ROOTS 直接关联的对象标记为灰色
3. 遍历灰色对象的引用，遍历完后，灰色对象本身置为黑色，引用置为灰色
4. 重复步骤3，直到所有的灰色对象被遍历完
5. 结束后，只有黑色和白色对象，黑色存活，白色被回收

这个过程的正确执行的前提是**没有其它线程改变对象的引用**，但是在 CMS 并发标记的过程中，用户线程仍在运行，因为就会出现漏标或错标。

##### **错标**

如图：假设正在遍历对象B，而此时用户线程执行了 `A.B = null`，切断了 A 到 B 的引用。

![image-20210709093617533](http://snail-resources.oss-cn-beijing.aliyuncs.com/1625794577.98471m58F43HwCa.png)

本来执行完`A.B = null`后，B、D、E 都应该被回收的，但是由于 B 已经变为灰色了，它仍会被当做存活对象，继续遍历下去。
最终的结果就是本轮 GC 不会回收 B、D、E，它们最后都会变为黑色，留到下次 GC 时回收，这就是是浮动垃圾。

**漏标**

如图：假设正在遍历对象B，而此时用户线程执行了 `B.D = null`，切断了 B 到 D 的引用，然后又建立了 A 到 D 的引用。

![image-20210709093809222](http://snail-resources.oss-cn-beijing.aliyuncs.com/1625794689.934198IxW7v6yAC0.png)

由于 A 已经被遍历过了，而 B 又与 D 断开引用了，所以本次标记结束后，D 对象一直是白色，最终是要被回收掉的，如果被回收了，就发生系统错误。

可以看到漏标的结果比错标严重的多，浮动垃圾可以下次GC清理，而把不该回收的对象回收掉，将会造成程序运行错误。

**漏标只有同时满足以下两个条件才会发生**：

1. 黑色对象重新引用了该白色对象；即黑色对象成员变量增加了新的引用。
2. 灰色对象断开了白色对象的引用（直接或间接的引用）；即灰色对象原来成员变量的引用发生了变化。

只要打破其中一个条件就可以解决漏标的问题，CMS 选择打破第一个条件，G1 选择打破第二个条件。

CMS：写屏障 + 增量更新

G1：写屏障 + 原始快照（Snapshot At The Beginning，STAB）

**写屏障**

给某个对象的成员变量赋值时，其底层代码大概长这样：

```c
void oop_field_store(oop* field, oop new_value) { 
    *field = new_value; // 赋值操作
} 
```

所谓的写屏障，其实就是指在赋值操作前后，加入一些处理（可以参考AOP的概念）：

```c
void oop_field_store(oop* field, oop new_value) {  
    pre_write_barrier(field); // 写屏障-写前操作
    *field = new_value; 
    post_write_barrier(field, new_value);  // 写屏障-写后操作
}
```

**增量更新**：当有新的引用插入时，记录下新的引用对象。

```c
void post_write_barrier(oop* field, oop new_value) {  
  if($gc_phase == GC_CONCURRENT_MARK && !isMarkd(field)) {
      remark_set.add(new_value); // 记录新引用的对象
  }
}
```

**原始快照**：当原来成员变量的引用发生变化之前，记录下原来的引用对象。

```c
void pre_write_barrier(oop* field) {
    oop old_value = *field; // 获取旧值
    remark_set.add(old_value); // 记录 原来的引用对象
}

// 优化
void pre_write_barrier(oop* field) {
  // 处于GC并发标记阶段 且 该对象没有被标记（访问）过
  if($gc_phase == GC_CONCURRENT_MARK && !isMarkd(field)) { 
      oop old_value = *field; // 获取旧值
      remark_set.add(old_value); // 记录  原来的引用对象
  }
}
```

# 二、CMS 运行阶段

## 1、初始化标记（STW）

STW 后，扫描 GC ROOTS 直接可达的对象，并压入标记栈，扫描完后恢复应用程序线程。

这里的 GC ROOTS 只有 stack、register、globals，并没有包含年轻代，但是应该包含年轻代，就像年轻代进行垃圾收集时会把老年代作为 GC ROOTS 一样。

那这里为什么没有包含呢？

1. 年轻代对象更新频繁
2. 年轻代对象很多，可能有 80% 需要回收的对象，扫描这些没意义

可能在并发标记期间，年轻代的对象发生了一次回收，此时标记年轻代的对象没有太大的意义。

## 2、并发标记

该阶段与用户线程一起运行，运行过程也很简单，从标记栈中弹出对象，然后遍历对象，将对象直接引用的子对象标记为灰色并压入标记栈，遍历完后将对象置为黑色对象，表示遍历完了，重复出栈、入栈，直至标记栈为空，并发标记阶段结束。

存在的问题：因为是并发运行，所以用户线程会在并发标记过程中可以插入一些新的引用，这些引用可不能被清理掉，如何解决呢？

CMS 就是使用了增量更新技术，上面讲过，将并发标记期间，老年代中引用的变化记录下来（老年代指向老年代 和 老年代指向年轻代的引用），然后在并发标记阶段，以这些记录下来的引用为 GC ROOTS，进行扫描。

CMS在整个并发收集过程中让所有新创建的对象都是黑色的，也就是说在CMS GC进行中创建的对象在这轮收集都会保证存活。这样虽然会有浮动垃圾的问题，但实现起来比较简单，收集速度受影响较小。

浮动垃圾的由来：

1. 并发标记阶段产生的对象
2. 对象的引用变了

## 3、预清理阶段

这个阶段会处理在并发标记过程中eden区发生变化的引用（特指 eden 指向 old gen 的引用变化），此外还会处理 dirty card 中的引用。

## 4、可中断预清理

这个阶段主要处理 from 和 to 区域对象引用 old gen 的变化，同样也会继续处理 dirty card 的对象引用。这个阶段默认设置的时间是 5s，如果执行逻辑超过 5s，会自动终止这个阶段，或者当 eden 区使用内存值小于 CMSScheduleRemarkEdenPenetration，默认 50% 时，也会退出这个阶段。

如果这个阶段能处理掉一大半的对象引用的话，会大大降低下个阶段 remark 得停顿时间。有一种对降低 remark 时延非常有效的方式，就是在可中断预清理阶段碰上 young gc。经过 young gc 后 remark 就能大大降低时延，为什么？ 原因就是因为 remark 需要对整个 young gen 进行一次扫描，如果之前发生过一次 young gc，那对 remark 阶段来说，就省了扫描大量对象引用的时间。

## 5、重新标记（STW）

这里先学习一个概念：mod union table。引用R大的描述：

> card table 只有一份，既要用来支持 young GC 又要用来支持 CMS。每次 young GC 过程中都涉及重置和重新扫描 card table，这样是满足了 young GC 的需求，但却破坏了 CMS 的需求—— CMS需要的信息可能被 young GC 给重置掉了。
>
> 为了避免丢失信息，就在card table之外另外加了一个bitmap叫做mod-union table。在CMS concurrent marking正在运行的过程中，每当发生一次young GC，当young
> GC要重置card table里的某个记录时，就会更新mod-union table对应的bit。
>
> 这样，最后到CMS remark的时候，当时的card table外加mod-union table就足以记录在并发标记过程中old gen发生的所有引用变化了。

card table 和 mod union table 中存放的都是 dirty card，重新标记之前会有两个步骤：1、预清理；2、可中断的预清理。这两个阶段都是处理 dirty card，帮助 remark 尽可能的提前处理掉大部分的 dirty card，但是预清理和可中断的预清理都是并行的，不会处理完的 dirty card 的，最后还是得依赖于 remark 阶段进行处理。

重新标记阶段主要是解决并发标记过程中对象引用的变更，修正这些变更，以保证不会发生漏标的情况，这里还会使用年轻代作为 GC ROOTS 扫描，如果年轻代中的对象很多，那么重新标记阶段的时间就会变长，对象少，重新标记阶段的时间就会变短。

这个阶段依赖于 card table、mod union table 以及增量更新时记录的引用进行修正。

这里有个疑问：增量更新 和 card table 及 mod union table 的作用是不是冲突了，增量更新已经记录下来所有引用的变更了，那么还需要 card table 做什么？大佬解释解释。

## 6、并发清除

重新标记阶段会把待清理

## 7、并发重置

TODO

# 四、缺点

1. **内存碎片**（原因是采用了标记-清除算法）
2. **对 CPU 资源敏感**（原因是并发时和用户线程一起抢占 CPU）
3. **浮动垃圾**：在并发标记阶段产生了新垃圾不会被及时回收，而是只能等到下一次GC

# 五、日志解读

```
# 年轻代 GC，使用 ParNew
0.210 [GC (Allocation Failure) [ParNew: 279616K->34942K(314560K), 0.0246537 secs] 279616K->79707K(1013632K), 0.0246999 secs] [Times: user=0.06 sys=0.09, real=0.03 secs]

# 老年代 GC

# 初始计标记，速度很快只用了3.1毫秒
# 372071K：老年代的使用量；699072K：老年代的总容量；413038K：堆的使用量；1013632K：整个堆的容量；0.0003105 secs：耗时
0.526: [GC (CMS Initial Mark) [1 CMS-initial-mark: 372071K(699072K)] 413038K(1013632K), 0.0003105 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

# 启动并发标记步骤
0.526: [CMS-concurrent-mark-start]
0.529: [CMS-concurrent-mark: 0.003/0.003 secs] [Times: user=0.00 sys=0.01, real=0.01 secs]

# 启动预清理步骤，这个阶段会尽可能在重新标记前，处理掉一些在并发标记阶段发生变化的引用关系，从而降低重新标记阶段的停顿时间
# 清理 Eden 区中发生变化的引用（dirty card）
0.529: [CMS-concurrent-preclean-start]
0.530: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

# 启动可中断的预清理，这个阶段主要处理 from 和 to 区域对象引用 old gen 的变化，同样也会继续处理 dirty card 的对象引用。这个阶段默认设置的时间是5s，如果执行逻辑超过5s，会自动终止这个阶段，或者当eden区使用内存值小于 CMSScheduleRemarkEdenPenetration，默认 50% 时，也会退出这个阶段。
0.530: [CMS-concurrent-abortable-preclean-start]
0.844: [CMS-concurrent-abortable-preclean: 0.006/0.315 secs] [Times: user=1.34 sys=0.12, real=0.31 secs]

# 重新标记
# 35015K：年轻代当前的使用量；314560K：年轻代的总容量；
0.845: [GC (CMS Final Remark) [YG occupancy: 35015 K (314560 K)]
# Rescan：进行重新标记，耗时 0.0004597 secs
0.845: [Rescan (parallel) , 0.0004597 secs]
# 处理弱引用
0.845: [weak refs processing, 0.0000086 secs]
# 卸载 class
0.845: [class unloading, 0.0004208 secs]
# 清理类级元数据和内部化字符串的符号和字符串表
0.845: [scrub symbol table, 0.0004006 secs]
0.846: [scrub string table, 0.0001479 secs]
# 老年代的使用情况及堆的使用情况
[1 CMS-remark: 677072K(699072K)] 712088K(1013632K), 0.0014893 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

# 启动并发清除，任务是清除那些没有标记的无用对象并回收内存
0.846: [CMS-concurrent-sweep-start]
0.847: [CMS-concurrent-sweep: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]

# 启动并发重置，作用是重新设置CMS算法内部的数据结构
0.847: [CMS-concurrent-reset-start]
0.849: [CMS-concurrent-reset: 0.001/0.001 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
```


