# Chapter 4 - 进程调度

Created by : Mr Dk.

2019 / 10 / 14 14:24

Nanjing, Jiangsu, China

---

## 4.1 多任务

多任务 OS 能够同时并发地交互执行多个进程的 OS

* 在单 CPU 上，会产生多个进程同时运行的幻觉
* 在多 CPU 上，会有多个进程真正同时、并行地运行

多任务系统可以被分为两类：

* 非抢占式多任务 OS (cooperative multitasking)
* 抢占式多任务 OS (preemptive multitasking)

Linux 提供了抢占式的多任务模式

* 由调度程序决定什么时候停止一个进程的运行
* 这个强制挂起的操作就叫做 __抢占__
* 进程被抢占前能够运行的时间是设置好的 - 时间片 (timeslice)

在非抢占式的多任务模式下，除非进程自己主动停止运行，否则将会一直执行

进程主动挂起自己的操作称为 __让步 (yielding)__

---

## 4.2 Linux 的进程调度

Linux 的第一版到 2.4，调度程序都想当简陋，设计近乎原始

在 Linux 2.5 中，引入了 __O(1)__ 调度程序

* 该调度程序可以在恒定时间内完成工作，不管输入有多大
* 对于调度响应时间敏感的程序有先天不足 (交互进程)
* 对于大服务器的工作负载很理想
* 在桌面系统上表现不佳

在 Linux 2.6 中，为了提高对交互程序的调度性能

引入了 _Rotating Staircase Deadline scheduler (RSDL)_

将 __公平调度__ 的概念引入了 Linux 程序

在 2.6.23 内核中替代了 O(1)

被称为 __完全公平调度算法__ (CFS)

---

## 4.3 策略

策略决定调度程序在何时让什么进程运行

### 4.3.1 I/O 消耗型和处理器消耗型的进程

I/O 消耗型 - 大部分时间都用于提交 I/O 请求，或等待 I/O 请求

* 进程经常处于可运行状态
* 多数 GUI 程序都属于 I/O 密集型

CPU 消耗型 - 时间大部分用于执行代码

* 除非被抢占，否则一直不停地运行
* 策略应当是尽量降低调度频率，延长其运行时间

调度策略需要在两个矛盾的目标上寻找平衡

* 进程响应迅速
* 最大系统利用率

Linux 更倾向于优先调度 I/O 消耗型进程

* 保证交互式应用和桌面系统的性能

但调度程序也未忽略 CPU 消耗型的进程

### 4.3.2 进程优先级

根据进程的 __价值__ 和 __对 CPU 时间的需求__ 来对进程分级

* 优先级高的先运行，低的后运行
* 相同优先级按轮转方式进行调度

Linux 使用了两种不同的优先级范围：

* nice 值
  * `-20` 到 `+19`
  * 越大的 nice 值意味着越低的优先级
* 实时优先级
  * 越高的实时优先级意味着进程优先级越高
  * 任何实时进程的优先级都高于普通的进程

### 4.3.3 时间片

时间片过长 - 系统对交互的响应表现欠佳

时间片过短 - 增大进程切换带来的 CPU 耗时

矛盾：

* I/O 消耗型进程不需要较长的时间片
* CPU 消耗型进程希望时间片越长越好

Linux 的 CFS 调度器并没有直接给进程分配时间片

而是将 CPU 的 __使用比__ 划分给了进程

* 进程所获的 CPU 时间与系统负载相关
* 比例还会进一步受 nice 值影响
* nice 值作为权重，调整进程使用的 CPU 时间比

使用 CFS 调度器，进程的抢占时机取决于新的可运行程序消耗了多少 CPU 使用比

* 如果消耗的使用比比当前进程小，则新进程投入运行

### 4.3.4 调度策略的活动

---

## 4.4 Linux 调度算法

### 4.4.1 调度器类

Linux 调度器以模块方式提供

允许不同进程可以有针对性地选择调度算法

允许多种不同的可动态添加的调度算法并存

每个调度算法调度属于自己范畴的进程

每个调度器有一个优先级

按照优先级顺序遍历调度器类

拥有一个可执行进程的最高优先级调度器类胜出

完全公平调度 (CFS) 是针对 __普通进程__ 的调度类 - `SCHED_NORMAL`

### 4.4.2 Unix 系统中的进程调度

问题：

1. nice 的单位对应到 CPU 的绝对时间有问题
2. 把 nice 的值减小 1 所带来的效果极大地取决于 nice 的初始值
3. ...
4. ...

实质：分配绝对的时间片引发的固定的切换频率，给公平性造成了很大变数

CFS 对时间片的分配方式进行了根本性的重新设计

确保了进程调度能够有恒定的公平性

### 4.4.3 公平调度

CFS 的出发点：

* n 个进程，则每个进程能获得 1/n 的 CPU 时间

在任何可测量周期内，给予 n 个进程中每个进程同样多的时间

CFS 允许每个进程运行一段时间、循环轮转、选择运行最少的进程作为下一个运行进程

不再给每个进程分配时间片

CFS 在所有可运行进程总数的基础上计算出一个进程应该运行多久

不依靠 nice 值来计算时间片

* nice 值被作为进程获得 CPU 运行比的权重

CFS 为调度周期的近似值设立了一个目标 - __目标延迟__

* 调度周期越小，越接近完美的多任务
* 但必须承受更高的切换代价和更差的系统吞吐能力

若任务数量趋近于无限，所获得的 CPU 使用比将趋近于 0

* CFS 为每个进程获得的时间片设置了一个底线 - 最小粒度
* 确保进程的最少运行时间

另外，绝对的 nice 值不再影响调度决策 - 相对值才会影响分配比

进程的 CPU 时间由它自己和所有其它可运行进程的 nice 差值决定

nice 值对时间片的作用不再是算术加权，而是几何加权

CFS 确保给每个进程公平的 CPU 使用比

---

## 4.5 Linux 调度的实现

### 4.5.1 时间记账

所有调度器都需要对进程的运行时间记账

```c
struct sched_entity {
    struct load_weight load;
    struct rb_node run_node;
    struct list_head group_node;
    unsigned int on_rq;
    u64 exec_start;
    u64 sum_exec_runtime;
    u64 vruntime;
    u64 prev_sum_exec_runtime;
    u64 last_wakeup;
    u64 avg_overlap;
    u64 nr_migrations;
    u64 start_runtime;
    u64 avg_wakeup;
    // ...
}
```

这个结构作为一个名为 `se` 的成员变量，嵌入在 PCB 内

* `vruntime` 存放进程的虚拟运行时间
  * ns 为单位
  * 记录一个程序到底运行了多长时间，以及它还应该再运行多久

记账功能实现：

* 由系统定时器周期性调用

```c
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_of(cfs_rq)->clock;
    unsigned long delta_exec;
    
    if (unlikely(!curr))
        return;
    
    delta_exec = (unsigned long)(now - curr->exec_start); // 计算当前进程的执行时间
    if (!delta_exec)
        return;
    __update_curr(cfs_rq, curr, delta_exec); // 根据可运行进程总数对运行时间进程加权计算
    curr->exec_start = now;
    
    if (entity_is_task(curr)) {
        struct task_struct *curtask = task_of(curr);
        trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
        cpuacct_charge(curtask, delta_exec);
        account_group_exec_runtime(curtask, delta_exec);
    }
}
```

```c
static inline void __update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr, unsigned long delta_exec)
{
    unsigned long delta_exec_weighted;
    
    schedstat_set(curr->exec_max, max((u64)delta_exec, curr->exec_max));
    
    curr->sum_exec_runtime += delta_exec;
    schedstat_add(cfs_rq, exec_clock, delta_exec);
    delta_exec_weighted = calc_delta_fair(delta_exec, curr);
    
    curr->vruntime += delta_exec_weighted;
    update_min_vruntime(cfs_rq);
}
```

### 4.5.2 进程选择

均衡进程的虚拟运行时间：

* 挑选一个具有最小 `vruntime` 的进程

如何实现选择具有最小 `vruntime` 的进程？

CFS 使用 __红黑树__ 来组织可运行进程队列

* 迅速找到 `vruntime` 最小的进程
* 自平衡 BST
* 只需找到树的最左下子节点即可

```c
static struct sched_entity *__pick_next_entity(struct cfs_rq *cfs_rq)
{
    struct rb_node *left = cfs_rq->rb_leftmost;
    if (!left)
        return NULL;
    return rb_entry(left, struct sched_entity, run_node);
}
```

> 并不遍历获得最左子节点
>
> 该值已经缓存在 `rb_leftmost` 中
>
> 如果返回值为 NULL，则说明没有最左子节点，也就是树中没有任何结点
>
> 即，没有可运行进程 - CFS 便选择 idle 任务运行

当进程变为可运行状态 (wake up) 或第一次创建进程时

CFS 将进程加入红黑树中：

```c
static void enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    if (!(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATE))
        se->vruntime += cfs_rq->min_vruntime;
    
    update_curr(cfs_rq);
    account_entity_enqueue(cfs_rq, se);
    
    if (flags & ENQUEUE_WAKEUP) {
        place_entity(cfs_rq, se, 0);
        enqueue_sleeper(cfs_rq, se);
    }
    
    update_stats_enqueue(cfs_rq, se);
    check_spread(cfs_rq, se);
    if (se != cfs_rq->curr)
        __enqueue_entity(cfs_rq, se);
}
```

将一些统计信息记录到 `se` 中，然后调用繁重的插入操作：

```c
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    struct rb_node **link = &cfs_rq->tasks_timeline.rb_node;
    struct rb_node *parent = NULL;
    struct sched_entity *entry;
    s64 key = entity_key(cfs_rq, se);
    int leftmost = 1;
    
    while (*link) {
        parent = *link;
        entry = rb_entry(parent, struct sched_entity, run_node);
        
        if (key < entity_key(cfs_rq, entry)) {
            link = &parent->rb_left;
        } else {
            link = &parent->rb_right;
            leftmost = 0; // 新插入结点不是最左结点
        }
    }
    
    if (leftmost) {
        // 新插入结点是最左结点，更新缓存
        cfs_rq->rb_leftmost = &se->run_node;
    }
    
    rb_link_node(&se->run_node, parent, link); // 插入子节点
    rb_insert_color(&se->run_node, &cfs_rq->tasks_timeline); // 更新自平衡属性
}
```

进程从树中的删除发生在进程阻塞 (不可运行状态) 或终止时：

```c
static void dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int sleep)
{
    update_curr(cfs_rq);
    
    update_stats_dequeue(cfs_rq, se);
    clear_buddies(cfs_rq, se);
    
    if (se != cfs_rq->curr)
        __dequeue_entity(cfs_rq, se);
    account_entity_dequeue(cfs_rq, se);
    update_min_vruntime(cfs_rq);
    
    if (!sleep)
        se->vruntime -= cfs_rq->min_vruntime;
}
```

```c
static void __dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    if (cfs_rq->rb_leftmost == &se->run_node) {
        // 要删除的结点是最左结点
        struct rb_node *next_node;
        
        next_node = rb_next(&se->run_node);
        cfs_rq->rb_leftmost = next_node; // 更新缓存
    }
    
    rb_erase(&se->run_node, &cfs_rq->tasks_timeline);
}
```

### 4.5.3 调度器入口

进程调度的入口点是函数 `schedule()`

* 是内核其它部分调用进程调度器的入口
* 在其中调用 `pick_next_task()`，以优先级从高到低依次检查调度类
* 选择最高优先级的调度类中，最高优先级的进程

```c
static inline struct task_struct *pick_next_task(struct rq *rq)
{
    const struct sched_class *class;
    struct task_struct *p;
    
    // 优化：如果所有任务都在公平类中
    // 所有可运行进程数 == cfs 的可运行进程数
    if (likely(rq->nr_running == rq->cfs.nr_running)) {
        p = fair_sched_class.pick_next_task(rq);
        if (likely(p))
            return p;
    }
    
    class = sched_class_highest;
    for ( ; ; ) {
        p = class->pick_next_task(rq);
        if (p) // 永远不会返回 NULL，因为 idle 总是返回非 NULL 的 p
            return p;
        class = class->next;
    }
}
```

由于绝大多数进程运行在 CFS 类中，因为上面函数中做了一定的优化

循环以优先级为序，遍历每个调度类

每个调度类都实现了 `pick_next_task()` 函数

返回下一个可运行进程的指针

### 4.5.4 睡眠和唤醒

睡眠：

* 进程把自己标记为休眠状态
* 从可执行红黑树中移出，放入等待队列
* 然后调用 `schedule()` 选择和执行一个进程

唤醒：

* 进程被设置为可执行状态
* 从等待队列移到可执行红黑树中

休眠通过 __等待队列__ 进行处理

* 等待队列是由等待某些事件发生的进程组成的简单链表
* 内核用 `wake_queue_head_t` 来代表等待队列
* 进程将自己放入等待队列，并设置为不可执行状态

休眠和唤醒的实现不能有纰漏，因为可能存在竞争条件：

```c
DEFINE_WAIT(wait); // 创建等待队列的等待项
add_wait_queue(q, &wait); // 将等待项加入到队列中

while (!condition) {
    // condition 是等待的事件
    prepare_to_wait(&q, &wait, TASK_INTERRUPTIBLE); // 进程状态变更
    if (signal_pending(current)) // 唤醒不是因为事件的发生，而是信号
        // 检查并处理信号
    schedule(); // 调度，继续等待
}
// 等待的事件到来
finish_wait(&q, &wait); // 将自己设置为 TASK_RUNNING，移除等待队列
```

唤醒通过 `wake_up()` 函数进行

* 唤醒等待队列上的所有进程
* 设置进程的 `TASK_RUNNING` 状态
* 加入可执行红黑树
* 如果被唤醒进程的优先级比当前进程高，则设置 `need_resched` 标志

* 进程被唤醒并不是因为等待的条件达成
* 可能是因为收到了信号

---

## 4.6 抢占和上下文切换

上下文切换，即从一个可执行进程切换到另一个可执行进程

* 由 `context_switch()` 函数负责处理
* 该函数被 `schedule()` 调用

完成两项基本工作：

* 调用 `switch_mm()`，把虚拟内存映射切换到新进程中

* 调用 `switch_to()`，从上一个进程的 CPU 状态切换到新进程的 CPU 状态

  > 这两个好像都是和体系结构有关的宏

内核提供一个 `need_resched` 标志，表明是否需要重新执行一次调度

何时设置这个标志呢？

* 某个进程应该被抢占时
* 当一个优先级高的进程进入可执行状态时

该标志对于内核来说是一个信息

表示有其它进程应当被运行了，要尽快执行调度程序

内核检查该标志，并调用 `schedule()` 切换到新进程

检查该标志的时机：

* 返回用户空间前
* 从中断返回时

每个进程都有一个 `need_resched` 标志

* 之前位于 `task_struct` 中，2.6 位于 `thread_info` 中
* 因为访问 PCB 中的数据比访问全局变量快
  * `current` 宏速度很快
  * 而且通常都在 cache 中

### 4.6.1 用户抢占

如果 `need_resched` 标志被设置

就会导致调度函数被调用

时机：

* 从系统调用返回用户空间时
* 从中断处理程序返回用户空间时

### 4.6.2 内核抢占

在不支持内核抢占的内核中，内核代码可以一直执行，到完成为止

在 2.6 的内核中，引入了抢占能力

* 只要重新调度是安全的，内核就可以在任何时间抢占正在执行的任务

什么时候重新调度才是安全的？

* 只要没有持有锁，内核就可以被抢占
* 如果没有持有锁，正在执行的代码就是可重入的

因此，为每个 `thread_info` 引入 `preempt_count` 计数器

* 初始值为 0
* 使用锁的时候 +1，释放锁的时候 -1
* 当数值为 0 时，内核就可以被抢占

当中断返回内核空间时

* 若 `need_reschd` 被设置，`preempt_count` 为 0
* 在说明有更重要的任务要被执行，且可以安全抢占
* 调度程序会被调用

如果内核代码被阻塞了，或者显式调用了 `schedule()`

* 内核抢占会显式地发生
* 这种形式的内核抢占从来都是受支持的
* 显式调用意味着它应该清楚自己是可以被安全地抢占的

时机：

* 中断处理程序返回内核空间之前
* 内核代码再一次具有可抢占性 (所有锁被释放)
* 内核任务显式调用调度函数
* 内核中的任务阻塞

---

## 4.7 实时调度策略

两种实时调度策略：

* `SCHED_FIFO`
  * 简单的先入先出调度算法
  * 不使用时间片，执行到自身受阻塞或显式释放 CPU 为止
  * 比普通的 `SCHED_NORMAL` 进程更先得到调度
* `SCHED_RR`
  * 与 `SCHED_FIFO` 大致相同
  * 耗尽预先分配的时间片以后就不再继续执行
  * 实时轮转调度算法

两种算法都使用了静态优先级

Linux 的实时调度算法提供了 __软实时__ 工作方式

* 尽力保证在限定时间到来前运行
* Linux 对于实时任务的调度不做任何保证

---

## 4.8 与调度相关的系统调用

Linux 提供了一些系统调用

用于管理与调度程序相关的参数

### 4.8.1 与调度策略和优先级相关的系统调用

* 设置和获取进程的调度策略和实时优先级
  * `sched_setscheduler()`
  * `sched_getscheduler()`
* 设置和获取进程的实时优先级
  * `sched_setparam()`
  * `sched_getparam()`
* 对于普通进程，可以调用 `nice()`

### 4.8.2 与处理器绑定有关的系统调用

使进程 __尽量__ 在同一个 CPU 上运行

但允许用户 __强制指定__ 这个进程必须在某个 CPU 上执行

### 4.8.3 放弃处理器时间

通过 `sched_yield()` 系统调用

让进程显式地将 CPU 时间让给其它进程

* 对于普通进程，将其从活动队列移到过期队列
* 对于实时进程，将其移动到活动队列的最后

---

## Summary

不看代码就完全无法理解所谓的 CFS 调度算法

哎 有点无奈...... 😥

---

