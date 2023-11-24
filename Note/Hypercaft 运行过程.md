# Hypercaft 运行分析

目前拿到手的原生 Hypercaft 默认配置上，好像是支持一个 pcpu 跑 一个 vcpu，运行一个 guest OS。

下面以这个目标，梳理下这个过程。

## 概览

- _start()
  - PC = 0x8020_0000: opensbi处理结束后的，pc进入_start时的地址
  - a0：hart_id 值，即cpu编号。
  - a1：dtb的值。 
- rust_entry(cpu_id, dtb)
  - class_bss
  - 初始化 root cpu
    - 初始化过程暂时不管
  - 设置 trap 地址
- rust_main(cpu_id, dtb)   
  - //ArceOS入口，muti-core 中其它的唤醒时刻。
  - //初始化 各种外设和检测设置
  - 这里的



## rust_main(cpu_id, dtb)   

### 简介

> It is called from the bootstrapping code in [axhal]. `cpu_id` is the ID of
> the current CPU, and `dtb` is the address of the device tree blob. It
>
> finally calls the application's `main` function after all initialization
> work is done.
> In multi-core environment, this function is called on the primary CPU,
> and the secondary CPUs call [`rust_main_secondary`].

1. 初始化堆区、
2. platform_init();  //对risc-v 来说，这里就是初始化时钟和 中断的设置
3. 初始化调度器
4. 初始化各种IO
5. **如果使用了** SMP多核，这里启动cpu_id。
   1. 目前这里没有用到dtb信息
   2. 从传参信息看，start_secondary_cpus 传入的事 root cpu的id 号
   3. 从运行效果看，non-root cpu 的id号不同于root cpu 。（里面应该有一些操作暂时不深挖。
6. 进入 main(cpu_id) //开启了hv的情况下。



---

## apps/hv/src/main.rs  main中

1. PerCPU 初始化boot cpu
   1. 
2. 从 Hal库中获取当前的 pcpu
3. 创建一个vcpu
   1. 传入设备树地址，以此来返回一个CPU的页表root
   2. 使用 pcpu 中的方法创建一个 vcpu，即把vcpu绑定在 指定的pcpu中。
4. 创建一个VmCpus实例vcpus，用于指定哪些vcpu是供给同一个VM的
5. 创建一个VM，并把打包好的vcpus 给vm使用。
   1. vm创建后初始化集合里的vcpu
6. 启动

内存布局参考图：

![image-20231122113033034](E:\笔记\Hypervisor-Blog\Note\img\image-20231122113033034.png)

---



### boot cpu 初始化

`PerCpu::<HyperCraftHalImpl>::init(0, 0x4000);` ：设置boot_hart_id 和 栈大小

![image-20231122021302643](E:\笔记\Hypervisor-Blog\Note\img\image-20231122021302643.png)

这里的代码和注释显示，pcpu的个数应该是通过 dtb 的相关接口获得。但这里没有实现。

但有点很迷。。。看注释和之前老师们讲解的意思，PerCpu这个结构体，应该是 每个pcpu人手一个，但这里出现了这么段代码：

```rust
let pcpu_size = core::mem::size_of::<PerCpu<H>>() * cpu_nums;
```

cpu_nums 是一个 局部变量？？？so，这里先放一放。（支持多核的话，这里目测要自己改。

---

在 smp.rs 中实现的 `impl<H: HyperCraftHal> PerCpu<H>`  ，对于初始化主要有两个接口：

- init();  boot cpu 的初始化
- setup_this_cpu() : TP cpu 的初始化
  - 使用 tp 寄存器 进程安全。

---

在 pcpu 初始化的过程中，对于每个pcpu 都创建了一个vcpu_queue，用于管理每个 pcpu 上的 vcpu 状态。

---

### guest page table 设置

> `let gpt = setup_gpm(0x9000_0000).unwrap();` 这里的0x9000_0000 是 DTB 的地址。

这里有点迷，可能考虑到不同架构间的兼容性，源码里的设备树地址是直接写死的，在 rust_main 中的地址 dtb 数据根本没传过来。

#### 概览

主要分 3 part：

- guest 页表创建
- 机器硬件信息传递
- 外设地址 MMIO 的设置

![image-20231123091840105](E:\笔记\Hypervisor-Blog\Note\img\image-20231123091840105.png)

这里我重点关注前两个部分。

#### gpt的获取



![image-20231123095140523](E:\笔记\Hypervisor-Blog\Note\img\image-20231123095140523.png)

创建一个`NestedPageTable` ，并封装成 `guestPageTable` 返回。目前 guest 好像只支持sv39

下面是它的创建过程。

![image-20231123095257617](E:\笔记\Hypervisor-Blog\Note\img\image-20231123095257617.png)

设置 guest4个页的地址。暂时没看懂为什么这么设置。

![image-20231123101707742](E:\笔记\Hypervisor-Blog\Note\img\image-20231123101707742.png)

申请 4个 4k 物理页的地址。

#### 机器元数据设置



---

### vcpu创建

- create_vcpu（）
  - 在 hypercraft/src/arch/riscv/smp.rs 中。


![image-20231122024132110](E:\笔记\Hypervisor-Blog\Note\img\image-20231122024132110.png)

直接给当前的 pcpu的vcpu队列上锁后，push进去。，然后返回它 的入口指针。

即这个函数用于获取 vcpu 的入口地址。

- vcpus = VmCpu::new();

![image-20231123005722243](E:\笔记\Hypervisor-Blog\Note\img\image-20231123005722243.png)

创建了一个Vcpu 集合 inner，inner 最多只能存 8 个虚拟核。

其中 marker 为 `PhantomData` 类型，该通常用于在类型系统中引入一些信息，而不引入实际的数据。这里疑似用于 VM 的核间通信？ 或者说 vcpus 的共享变量区域？

---

- vcpus.add_vcpu(vcpu0).unwrap();

![image-20231123010815275](E:\笔记\Hypervisor-Blog\Note\img\image-20231123010815275.png)

使用 `get` 方法从 `self.inner` 数组中获取指定索引 `vcpu_id` 处的 `Once<VCpu<H>>` 实例。

然后 `call_once` 方法，把vcpu 加到 VmCpus 中。

注意：在该方法的源码（hypercraft/src/vcpus.rs）底部，有两个预留的接口保证了vcpu 共享内存和线程同步的安全：

![image-20231123012214636](E:\笔记\Hypervisor-Blog\Note\img\image-20231123012214636.png)

目测可以用来实现 vcpus 的核间通信。

> 这里目前还有个小疑问，创建出来的 vcpus 的指令集架构由谁决定？怎么设置？
>
> ![image-20231123014318446](E:\笔记\Hypervisor-Blog\Note\img\image-20231123014318446.png)
>
> 目前原始工程上 guest 跑的事 rv64ima，而 host 环境是：
>
> ![image-20231123014425616](E:\笔记\Hypervisor-Blog\Note\img\image-20231123014425616.png)
>
> 所以这里应该不存在指令集的翻译问题。
>
> 如果 vcpu 的 ISA 与 pcpu 不一致，可能还需要进行二进制翻译相关的工作。



---

### vcpu初始化与 vm启动

![image-20231123010815275](E:\笔记\Hypervisor-Blog\Note\img\image-20231123010815275.png)

#### VM创建

![image-20231123013255961](E:\笔记\Hypervisor-Blog\Note\img\image-20231123013255961.png)

##### 对物理机的抽象

RISC-V 的环境上，对物理机的抽象有四点：

- 处理器个数
- 页表
- 页的格式
- PLIC：虚拟机的外围中断控制器，可用于协调外设，也可用于核间通信。

##### 初始化

VM的页设置和 plic 的值不用改，直接用默认值。

把 vcpus 和 gpt 丢进去即可。

#### vcpu初始化

![image-20231123021746500](E:\笔记\Hypervisor-Blog\Note\img\image-20231123021746500.png)

设置每个vcpu的mmu映射关系，默认使用的事sv39，但可以配置。



## VM 运行分析

![image-20231123121346868](E:\笔记\Hypervisor-Blog\Note\img\image-20231123121346868.png)

### vm.run分析

代码有点长，主要分成两part：

- 单个vcpu的运行

![image-20231124035427843](E:\笔记\Hypervisor-Blog\Note\img\image-20231124035427843.png)

- 该trap后的处理

![image-20231124035508923](E:\笔记\Hypervisor-Blog\Note\img\image-20231124035508923.png)



![image-20231124041059068](E:\笔记\Hypervisor-Blog\Note\img\image-20231124041059068.png)

![image-20231124052824151](E:\笔记\Hypervisor-Blog\Note\img\image-20231124052824151.png)

pc走的是host的pc，无需保存。
