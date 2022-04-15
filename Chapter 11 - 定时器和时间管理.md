# Chapter 11 - 定时器和时间管理

Created by : Mr Dk.

2019 / 10 / 10 23:54

Nanjing, Jiangsu, China

---

内核中大量函数都是基于时间驱动的，时间管理在内核中占有非常重要的地位。

## 11.1 内核中的时间概念

内核必须在硬件的帮助下才能管理和计算时间。

- 系统定时器：以设定好的频率自行触发
- 内核知道连续两次时钟中断的间隔：节拍 (tick)

内核通过控制时钟中断，维护实际时间。在时钟中断处理程序中，要进行的工作有：

- 更新系统运行时间
- 更新实际时间
- 均衡调度程序中各 CPU 的运行队列
- 检查当前进程是否用尽了自己的时间片 - 如用尽，则重新调度
- 运行超时的定时器
- 更新资源消耗和 CPU 时间的统计值

在每次的时钟中断处理程序中都要被处理。

## 11.2 节拍率：HZ

在系统启动时，按照 HZ 的值对硬件进行设置。大多数体系结构的节拍率都是可调的。

### 11.2.1 理想的 HZ 值

提高节拍率 → 时钟中断产生得更加频繁 → 中断处理程序会更频繁地执行

更高的时钟中断解析度提高了时间驱动事件的准确度。若某个时刻随机触发定时器，可能在任何时刻超时。只有在时钟中断到来时才可以执行它。

### 11.2.2 高 HZ 的优势

- 内核定时器能以更高的频率和准确率运行
- 依赖定时值执行的系统调用，能以更高的精度运行 - `poll()`、`select()`
- 减少等待时钟中断到来的时间，提升系统性能
- 对资源消耗的测量会有更精细的解析度
- 提高进程抢占的准确度

### 11.2.3 高 HZ 的劣势

CPU 必须花时间来执行时钟中断处理程序 → 系统负担加重。频繁打乱 CPU 的 cache，并增加耗电。但随着现代硬件能力的提升，增加的负担不会对系统的性能有较大影响。

> 无节拍的 OS？动态调度时钟中断，不以固定的频率触发时钟中断，而是按需动态调度和重新设置，省电。

## 11.3 jiffies

全局变量 `jiffies` 用于记录自系统启动以来产生的节拍总数。启动时，内核将该变量初始化为 0。每次时钟中断处理程序都会增加该变量的值。

### 11.3.1 jiffies 的内部表示

```c
extern unsigned long volatile jiffies;
```

32-bit 的 jiffies 变量，在 100HZ 的时钟频率下，497 天后会 overflow；64-bit 的变量，任何人都别指望会看到它溢出。由于历史的原因，又要考虑与已有内核代码的兼容。所以定义了：

```c
extern u64 jiffies_64;
```

在内核映像的链接程序中，用 `jiffies_64` 覆盖 `jiffies`。原有代码直接访问 `jiffies` 的低 32-bit，而时间管理代码使用整个 64-bit，避免 overflow。

### 11.3.2 jiffies 的回绕

如果 jiffies 变量超出最大存放范围，则会回绕到 0：

```c
unsigned long timeout = jiffies + HZ/2;

if (timeout > jiffies) {
    // not timeout
} else {
    // timeout
}
```

上面的程序就会存在问题。好在内核提供了四个宏，用于比较节拍计数。能够正确处理节拍计数回绕的情况 (安全版本)：

```c
#define time_after(unknown,known) ((long)(known) - (long)(unknown) < 0)
#define time_before(unknown,known) ((long)(unknown) - (long)(known) < 0)
#define time_after_eq(unknown,known) ((long)(unknown) - (long)(known) >= 0)
#define time_before_eq(unknown,known) ((long)(known) - (long)(unknown) >= 0)
```

`unknwon` 通常是 `jiffies`，`known` 是需要对比的值。

### 11.3.3 用户空间和 HZ

## 11.4 硬时钟和定时器

### 11.4.1 实时时钟

实时时钟 (RTC) 用于持久存放系统时间。系统关闭后，也可以靠主板上的微型电池保持计时。通常，RTC 和 CMOS 集成在一起。系统启动时，内核读取 RTC 来初始化墙上时间。

### 11.4.2 系统定时器

根本思想：提供一种周期性触发中断机制。在 x86 中，采用可编程中断时钟 (PIT)。内核在启动时对 PIT 进行编程初始化，使其能够产生时钟中断。

> 8259 chip???

## 11.5 时钟中断处理程序

分为两个部分：

- 与体系结构相关
- 与体系结构无关

与体系结构相关的部分，作为 **系统定时器** 的 **中断处理程序**，注册到内核中：

- 获得 `xtime_lock` 锁，对系统时间进行维护，更新 RTC
- 调用体系结构无关的时钟例程 `tick_periodic()`
- 释放 `xtime_lock` 锁
- 退出

体系结构无关的例程：

- 累加 `jiffies_64` 变量
- 更新资源消耗的统计值 (当前进程消耗的系统时间和用户时间)
- 执行已经到期的动态定时器
- 执行进程调度
- 更新墙上时间
- 计算平均负载值

```c
static void tick_periodic(int cpu)
{
    if (tick_do_timer_cpu == cpu) {
        write_seqlock(&xtime_lock);
        tick_next_period = ktime_add(tick_next_period, tick_period);
        do_time(1);
        write_sequnlock(&xtime_lock);
    }

    update_process_times(user_mode(get_irq_regs()));
    profile_tick(CPU_PROFILING);
}
```

对于 `do_timer()` 函数来说，承担了对 `jiffies_64` 的实际增加操作：

```c
void do_timer(unsigned long ticks)
{
    jiffies_64 += ticks;
    update_wall_time(); // 更新墙上时钟
    calc_global_load(); // 更新系统的平均负载统计值
}
```

`update_process_times()` 更新耗费的各种节拍：

```c
void update_process_times(int user_tick)
{
    struct task_struct *p = current;
    int cpu = smp_processor_id();
    account_process_tick(p, user_tick); // 更新进程运行时间
    run_local_timers();
    rcu_check_callbacks(cpu, user_tick);
    printk_tick();
    scheduler_tick(); // 减少进程时间片
    run_posix_cpu_timers(p);
}
```

`account_process_tick()` 对进程的时间进行实质性更新。`user_tick` 的值是通过查看系统寄存器来设置的。

```c
void account_process_tick(struct task_struct *p, int user_tick)
{
    cputime_t one_jiffy_scaled = cputime_to_scaled(cputime_one_jiffy);
    struct rq *rq = this_rq();

    if (user_tick)
        account_user_time(p, cputime_one_jiffy, one_jiffy_scaled);
    else if ((p != rq->idle) || (irq_count() != HARDIRQ_OFFSET))
        account_system_time(p, HARDIRQ_OFFSET, cputime_one_jiffy, one_jiffy_scaled);
    else
        account_idle_time(cputime_one_jiffy);
}
```

内核对进程进行时间计数时，是根据中断发生时 CPU 所处的模式进行分类统计的。把这一个节拍全部算给中断发生时的 CPU 模式了 - 实际上，进程在一个节拍期间，可能多次进出内核态，但没有更精密的统计算法了。

## 11.6 实际时间

当前实际时间，即墙上时间。

```c
struct timespec xtime;

struct timespec {
    _kernel_time_t tv_sec; // s
    long tv_nsec;          // ns
}
```

`tv_sec` 存放着 1970.1.1 依赖经过的时间。

读写 `xtime` 变量需要申请 `xtime_lock` 锁。从用户空间取得墙上时间的接口 - `gettimeofday()`。它对应内核中的系统调用 `sys_gettimeofday()`，几乎完全取代了 `time()` 系统调用。

```c
asmlinkage long sys_ettimeofday(struct timeval *tv, struct timezone *tz)
{
    if (likely(tv)) {
        struct timeval ktv;
        do_gettimeofday(&ktv); // 与体系结构相关
        if (copy_to_user(tv, &ktv, sizeof(ktv)))
            return -EFAULT;
    }
    if (unlikely(tz)) {
        if (copy_to_user(tz, &sys_tz, sizeof(sys_tz)))
            return -EFAULT;
    }
    return 0;
}
```

内核主要会在文件系统中，修改各种时间戳时，使用 `xtime`。

## 11.7 定时器

也叫 **动态定时器** 或 **内核定时器**。使用简单：

- 初始化
- 设置一个超时时间
- 指定超时后执行的函数
- 激活

定时器 **不周期执行**，超时后自动撤销（动态定时器）。

### 11.7.1 使用定时器

```c
struct timer_list {
    struct list_head entry; // 定时器链表入口
    unsigned long expires; // 定时值 (jiffies 为单位)
    void (*function)(unsigned long); // 定时器处理函数
    unsigned long data; // 处理函数的参数
    struct tvec_t_base_s *base; // 定时器内部值
};
```

```c
struct timer_list my_timer;

init_timer(&my_timer);
my_timer.expires = jiffies + delay;
my_timer.data = 0;
my_timer.function = my_function;

add_timer(&my_timer);
```

显然，处理函数需要符合以下原型：

```c
void my_timer_function(unsigned long data);
```

内核可以保证不会在超时时间到期前运行处理函数，但有可能延误定时器处理程序的执行。所以不能用定时器来实现任何 **硬实时任务**。

更改定时器（顺带会激活）：

```c
mod_timer(&my_timer, jiffies + new_delay);
```

在定时器超时前停止寄存器（已超时的定时器会被自动删除）：

```c
del_timer(&my_timer);
```

该函数返回后，保证了定时器将来不会再被激活。但在多 CPU 的机器上，定时器处理程序可能已经在其它 CPU 上运行了。删除定时器时，应当等待其它 CPU 上运行的定时器处理程序都退出：

```c
del_timer_sync(&my_timer);
```

### 11.7.2 定时器竞争条件

不能通过删除 + 创建定时器的方法代替 `mod_timer()` 函数，因为在多 CPU 的机器上是不安全的。内核异步执行中断处理程序，应当重点保护定时器中断处理程序中的共享数据。

### 11.7.3 实现定时器

内核在 **时钟中断** 发生后执行定时器。时钟中断处理程序调用 `run_local_timers()` 函数：

```c
void run_local_timers(void)
{
    hrtimer_run_queues();
    raise_softirq(TIMER_SOFTIRQ); // 定时器软中断
    softlockup_tick();
}
```

触发的软中断由 `run_timer_softirq()` 函数处理，运行当前 CPU 上所有超时的定时器。内核中，所有的定时器都以链表的形式存放在一起，但寻找超时定时器而遍历整个链表是不明智的。内核按定时器的超时时间划分为 5 组，定时器超时时间接近时，定时器 **随组一起下移**。

确保了内核尽可能减少搜索超时定时器所带来的负担。

## 11.8 延迟执行

短暂地推迟执行任务，比如等待硬件完成某些工作。

### 11.8.1 忙等待

延迟的时间是节拍的整数倍，精确率要求不高时使用。

```c
unsigned long timeout = jiffies + 10;
while (time_before(jiffies, timeout))
    ;
```

效率低下，更好的方法是在代码等待时，允许内核重新调度执行其它任务：

```c
unsigned long timeout = jiffies + 5 * HZ;
while (time_before(jiffies, timeout))
    cond_resched();
```

由于 `jiffies` 变量被标记为 `volatile`

- 指示编译器在每次访问变量时，都要重新从主存中获得
- 而不是通过寄存器中的变量别名访问

### 11.8.2 短延迟

- 需要比时钟节拍还短的延时
- 要求延迟的时间精确

发生在和硬件同步时。内核提供了三个 μs、ns 和 ms 级别的延迟函数：

```c
void udelay(unsigned long usecs);
void ndelay(unsigned long nsecs);
void mdelay(unsigned long msecs);
```

这些函数依靠 **执行数次循环** 达到延迟效果

- 内核可以知道 CPU 在 1s 内能执行多少次循环
- 该值存放在 `loops_per_jiffy` 变量中，由内核启动时的 `calibrate_delay()` 计算
- 可以通过 `/proc/cpuinfo` 读到

### 11.8.3 schedule_timeout()

让需要延迟执行的任务睡眠，直到指定的延迟时间耗尽后再重新运行。不能保证睡眠时间正好等于指定的延迟时间，只能保证尽量接近。指定时间到期后，内核唤醒任务，并放回运行队列：

```c
set_current_state(TASK_INTERRUPTIBLE); // 若不想接受信号唤醒，也可以设为 TASK_UNINTERRUPTIBLE
schedule_timeout(s*HZ);
```

`schedule_timeout` 实际上是内核定时器的一个简单应用：

```c
signed long schedule_timeout(signed long timeout)
{
    timer_t timer;
    unsigned long expire;

    switch (timeout) {
        case MAX_SCHEDULE_TIMEOUT:
            // 无限期睡眠
            schedule();
            goto out;
        default:
            if (timeout < 0) {
                // printk(KERN_ERR "schedule_timeout: wrong timeout")
                current->state = TASK_RUNNING;
                goto out;
            }
    }

    expire = timeout + jiffies;

    init_timer(&timer);
    timer.expires = expire;
    timer.data = (unsigned long) current; // 当前进程的地址作为超时处理函数的参数 (用于唤醒)
    timer.function = process_timeout; // 超时处理函数

    add_timer(&timer);
    schedule(); // 当前任务已经睡眠，不会被调度到

    del_timer_sync(&timer); // 任务被提前唤醒 (收到信号)，则撤销定时器

    timeout = expire - jiffies;
out:
    return timeout < 0 ? 0 : timeout;
}
```

超时处理函数：

```c
void process_timeout(unsigned long data)
{
    wake_up_process((task_t *) data); // 唤醒设置定时器的进程
}
```

## Summary

比 0.12 的时钟管理复杂了很多啊，主要是还要考虑多 CPU 中存在的竞争条件问题。
