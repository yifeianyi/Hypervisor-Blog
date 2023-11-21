# Hypercaft 启动过程

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



## 0、modules/axhal/src/

## 1、apps/hv/src/main.rs  main中

1. PerCPU 初始化boot cpu
2. 从 Hal库中获取当前的 pcpu
3. 创建一个vcpu
   1. 传入设备树地址，以此来返回一个CPU的页表root
   2. 使用 pcpu 中的方法创建一个 vcpu，即把vcpu绑定在 指定的pcpu中。
4. 创建一个VmCpus实例vcpus，用于指定哪些vcpu是供给同一个VM的
5. 创建一个VM，并把打包好的vcpus 给vm使用。
   1. vm创建后初始化集合里的vcpu
6. 启动