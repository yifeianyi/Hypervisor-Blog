# 虚拟化

# Daily

## 2023.11.8 - 2023.11.12

- 补训练营第二阶段的坑。

- 学习了下 ArceOS 的框架和组件化OS理论

- 学习了虚拟化的一些简单基础知识

- 看了下RISC-V的H扩展资料

  

## 2023.11.13

在 ArceOS 上跑通了Linux，在看源码ing

### 简要记录

硬件虚拟化目前好像还不是很成熟。具体到RISC-V而言，貌似只有qemu完全支持了 RISC-V的H扩展，虽然目前已经有了开源的 H扩展的RTL实现，但没有商业化已流片的处理器支持H扩展（起码我目前查到的资料好像是没有)。x86 和 ARM 有了一些基本的解决方案，但好像还有不少问题没有解决？是块有潜力的半荒地。

RISC-V的引入了虚拟化的 H扩展后，通过添加新的CSR寄存器和指令，构建出了新的一套系统。或许也不能说是添加了新的CSR寄存器和指令？严谨说法应该是改造并添加了。

在引入了H扩展后的特权级模式下，HS 模式取代了原有的 S 模式，或者说 HS 模式其实也是S模式（说白的就是S模式在 H扩展下 功能加强了），然后往上加了一层VS模式从而实现虚拟化的设置。

![image-20231114034016757](img/image-20231114034016757.png)

下图是 <code>mstatus</code> 寄存器 不支持H扩展和 支持H扩展的位域图。可以看到RV32是不存在H扩展的可能的，在RISC-V 架构中，H扩展至少位宽得是64位。扩展后 <code>mstatus</code> 的 37 和 38 位从原来的写保留位，变成了 GVA 和 MPV。

![image-20231114043302766](img/image-20231114043302766.png)

设置 MPV 的值 其实就是设置 Virtualization Mode (V)。可以看出，即使处理器支持了 H扩展，使用权还是在程序员手里，不想用硬件虚拟化的话可以直接设置 $$V = 0$$ ，那就跟使用传统特权级模式一样了。

![image-20231114044005127](img/image-20231114044005127.png)

而 Hypercaft 是 hypervisor 的一个实现实例，也是我们这次的研究和扩展的对象。明天继续研究源码瞅瞅。。。

## 2023.11.14

不务正业的去刨了下8086成为经典的原因，看[处理器指令系统漫谈](https://blog.sciencenet.cn/blog-102148-1190924.html) 这个系列的博客看得有点上头。

学了下x86的汇编和指令集资料。。。果断弃疗，今天摸大鱼了。

## 2023.11.15

- 学习多核架构 和 多核系统启动的过程
- 梳理 Hypercraft的启动过程 的源码

恩，明天不摸鱼了。



## 2023.11.16

- 完成练习2-4

疑问：二阶段虚拟化的方式，只能加速一重虚拟机的运行速度。那如果进行虚拟化嵌套，第三层是否会性能骤降？



## 2023.11.17

看了不少资料，但感觉又没看啥东西，没有形成一个有效的思路。脑子有点混乱。

开完组会后发现可能有以下原因：

- 只完成通过二阶标准的3个lab，没有做文件系统，导致提及设备树部分理解上出现了点问题
- 思考多核支持实现的时候没有仔细看task的要求，也没有先把问题进行拆分最简化，上来思考复杂问题的实现，导致解决问题的思路不够清晰，效率低下。
- 没有看过构建 ArceOS 和 hypervisor 工程的相关Makefile文件 ，加之对 qemu 编译加载程序的过程不够熟悉，对工程结构和构建把握较差。

### 明日计划

- 大概把rCore 文件系统的文档和源码过一遍
- 参考arm 和 x86 的 练习文档理解下框架代码（rv的没有相关文档介绍，看视频容易犯困不知道为什么



## 2023.11.18 - 19

补文件系统相关的知识，但貌似还是不能满足多核支持的修改代码需求。



## 2023.11.20

学习设备树相关知识

安装dtc时发现，阿里云也暂停了对Ubuntu20.04的支持，换到 22.04 重新配环境



## 2023.11.21

- 修改 arceos/makefile 中ARCH、A、HV 的值，Hypercaft 默认启动。
- 修改 qemu.mk ，添加 build_dt 目标，用于使用qemu生成 guest 相关的 dt 文件。

  - 对比了下工程目录原有的 Linux.dts ，发现 qemu导出的 isa 在指令集模块 和 mmu 类型上有点区别，暂时不知道是否会造成影响。

  