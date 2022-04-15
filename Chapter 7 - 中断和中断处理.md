# Chapter 7 - 中断和中断处理

Created by : Mr Dk.

2019 / 10 / 19 10:02

Nanjing, Jiangsu, China

---

硬件响应很慢，内核应该在等待硬件响应期间处理其它任务，等到硬件真正完成请求后，再进行处理。**轮询 (polling)** 可能是一种解决方法，但更好的方法是，让硬件在需要的时候向内核发出信号。

## 7.1 中断

中断的本质是一种电信号，由硬件设备发送到 CPU。CPU 接收到中断后，马上向 OS 反应此信号的到来，然后由 OS 负责处理这些新到来的数据。生成中断时，设备并不考虑与 CPU 的时钟同步——中断可以随时产生。

中断控制器是一个简单的电子芯片，将多路中断管线采用复用技术，只通过一个和 CPU 相连接的管线通信。CPU 接收到中断信号后，转为处理中断。每个中断都有一个唯一的数字标识，使 OS 能对其进行区分。

> 异常：异常产生时必须考虑与 CPU 时钟同步，因此也被称为同步中断。异常与中断的处理方式类似。

## 7.2 中断处理程序

与特定的中断相关联，一个设备的中断处理程序是它驱动设备程序的一部分，由内核调用，运行于 **中断上下文** 中。中断上下文与进程上下文不同，不可阻塞。中断随时可能发生，因此中断处理程序也就随时可能执行。所以必须保证中断处理程序能够快速执行，也要让中断处理程序在尽可能短的时间内完成。

## 7.3 上半部与下半部的对比

中断处理程序 **运行得快** / **工作量多** 是两个矛盾的目标。因此把中断处理切分为两部分：

- 上半部（top half）：接收到中断后立刻开始执行，只做有严格时限的工作（应答或复位硬件），在 **关中断** 的条件下被执行
- 下半部（bottom half）：在 **开中断** 的条件下被执行

> 以网卡为例，上半部负责将最新的报文从设备拷贝到内核（时序要求严格），下半部负责对报文进行具体处理。

## 7.4 注册中断处理程序

如果设备使用中断，那么驱动程序就注册一个中断处理程序。驱动程序通过 `request_irp()` 注册一个中断处理程序。

```c
int request_irq(unsigned int irq,
               irq_handler_t handler,
               unsigned long flags,
               const char *name,
               void *dev);
```

- `irq` 为要分配的中断号
- `handler` 是一个函数指针，函数原型需要满足：

  ```c
  typedef irqreturn_t (*irq_handler_t)(int, void *);
  ```

### 7.4.1 中断处理程序标志

- `IRQF_DISABLED` - 内核在处理中断处理程序期间，要禁止所有的其它中断
- `IRQF_SAMPLE_RANDOM` - 中断对内核熵池有贡献 - 产生随机数
- `IRQF_TIMER` - 为时钟中断准备
- `IRQF_SHARED` - 在多个共享处理程序之间共享中断线，否则，在每条中断线上只能有一个处理程序

`name` 参数是设备的 ASCII 文本表示`dev` 参数用于共享中断线。当中断处理程序需要被释放时，该参数提供唯一的标识信息。内核调用中断处理程序时，会将这个指针传递。

`request_irp()` 成功执行后返回 0，如果返回非 0，则中断处理程序不会被注册。通常是中断线已被占用，而没有指定共享中断线。

此外，这个函数可能会睡眠，所以不能在中断上下文或其它不允许阻塞的代码中使用。

### 7.4.2 一个中断例子

```c
if (request_irq(irqn, my_interrupt, IRQF_SHARED, "my_device", my_dev))
{
    printk(KERN_ERR "my_device: cannot register IRQ %d\n", irqn);
    return -EIO;
}
```

### 7.4.3 释放中断处理程序

卸载驱动程序时：

- 注销中断处理程序
- 释放中断线

```c
void free_irq(unsigned int irq, void *dev);
```

如果中断线不是共享的，那么同时将禁用这条中断线；如果是共享的，那么仅删除对应处理程序，在所有处理程序被卸载后才禁用中断线。

## 7.5 编写中断处理程序

```c
static irqreturn_t intr_handler(int irq, void *dev);
```

在 Linux 2.0 之前没有 `dev` 参数，只能通过中断号区分驱动程序。`dev` 参数可以用于区分共享同一中断处理程序的多个设备。

中断处理程序的两个可能的返回值：`IRQ_NONE` 和 `IRQ_HANDLED`：

- `IRQ_NONE` - 检测到中断，但中断对应的设备并不是处理函数所指定的来源设备
- `IRQ_HANDLED` - 确实是中断处理程序所对应的设备产生的中断

利用这两个值，内核可以知道设备是否发出了虚假请求

> 重入
>
> Linux 的中断处理程序是无须重入的，因为在给定的中断处理程序正在执行时，相应中断线在所有 CPU 上都会被屏蔽。同一个中断处理程序绝对不会被同时调用以处理嵌套的中断。

### 7.5.1 共享的中断处理程序

`IRQF_SHARED` 标志必须被设置。对于每个注册的中断处理程序来说，`dev` 参数必须唯一，中断处理程序必须能够区分它的设备是否发生了中断。内核接收到中断后，将依次调用在该中断线上注册的每一个处理程序。因此，处理程序必须知道它是否改为这个中断负责。如果不是相关设备产生的中断，则处理程序应当立即退出。这需要硬件提供类似状态寄存器的机制，以便中断处理程序检查。

## 7.6 中断上下文

Interrupt context。

Re-cap: 进程上下文。通过 `current` 关联当前进程，进程以进程上下文的形式连接在内核中，进程上下文可以睡眠，也可以调用调度程序。而中断上下文没有后备进程，因此不可以睡眠，也不能从中断上下文中调用可以睡眠的函数！中断上下文有严格的时间限制，应当迅速、简洁，尽量不要处理繁重的工作，尽量把工作分离到下半部中。

曾经，中断处理程序共享所中断进程的内核栈（两页）。在 Linux 2.6 的早期版本中，内核栈减为 1 页，减轻内存的压力。每个 CPU 专门为中断处理程序分配了 1 页中断栈。虽然栈小了，但平均可用栈空间大得多。

## 7.7 中断处理机制的实现

对于每条中断线，在内核接收到中断信号后，对跳转到对应的唯一位置，在栈中保存中断号和当前寄存器的值（作为 C 函数的参数）。然后调用：

```c
unsigned int do_IRQ(struct pt_regs regs);
```

该函数提取出中断号，禁止这条中断线上的所有中断。如果中断线上有一个有效的处理程序，就调用如下程序，运行这条中断线上安装的所有中断处理程序（如果是共享的话）：

```c
irqreturn_t handle_IRQ_event(unsigned int irq, struct irqaction *action)
{
    irqreturn_t ret, retval = IRQ_NONE;
    unsigned int status = 0;

    if (!(action->flags & IRQF_DISABLED))
        // 此时已经关中断
        // 若处理程序不需要关中断，则开中断
        local_irq_enable_in_hardirq();

    do {
        trace_irq_handler_entry(irq, action);
        ret = action->handler(irq, action->dev_id);
        trace_irq_handler_exit(irq, action, ret);

        switch (ret) {
            case IRQ_WAKE_THREAD:
                ret = IRQ_HANDLED;

                if (unlikely(!action->thread_fn)) {
                    warn_no_thread(irq, action);
                    break;
                }
                if (likely(!test_bit(IRQTF_DIED, &action->thread_flags))) {
                    set_bit(IRQTF_RUNTHREAD, &action->thread_flags);
                    wake_up_process(action->thread);
                }

            case IRQ_HANDLED:
                status |= action->flags;
                break;
            default:
                break;
        }

        retval |= ret;
        action = action->next;
    } while (action);

    if (status & IRQF_SAMPLE_RANDOM)
        // 将中间时间间隔作为随机数生成器
        add_interrupt_randomness(irq);

    local_irq_disable(); // 重新关中断

    return retval;
}
```

## 7.8 /proc/interrupts

## 7.9 中断控制

Linux 内核提供了一组接口，操作机器上的中断状态

- 禁止中断系统
- 屏蔽整个机器上的一条中断线

> 不再使用全局的 `cli()`。这种方法虽然适用于所有体系结构，但完全以 x86 为中心。以前代码仅需通过全局禁止中断达到互斥，现在需要使用本地中断控制和自旋锁达到互斥。
>
> 取消 `cli()` 的优点：
>
> - 强制驱动程序的编写者实现真正的加锁
> - 具有特定目的的细粒度锁比全局锁要快许多
