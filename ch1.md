# 第1章 概述、启动与硬件基础

本章一半是导论（OS 是什么、为什么学、历史与结构），一半是硬核铺垫（bootloader 启动链、C 函数调用的栈帧、内联汇编、x86 中断处理）。后者看着像实验课，其实是后面所有章节的地基——不懂中断怎么进入、栈怎么切换，第 2 章之后会寸步难行。

### 1.1 bootloader启动ucore os（19.8 分钟）

**主旨：** 串起 x86 从上电第一条指令到跳入 ucore 内核的完整启动链：BIOS → MBR → bootloader → ELF格式 kernel。

**关键点：** 

1. 加电后 CPU 处于实模式，CS = F000H、EIP = 0000FFF0H，实际地址 = Base + EIP =FFFFFFF0H，即 BIOS 所在的 EPROM（Erasable Programmable Read Only Memory）；此处第一条指令是一条长跳转指令，执行后 CS 与 EIP 都被更新，地址转换规则随即开始生效。

2. 实模式下由 16 位段选择子（Segment Selector，如 CS、DS、SS）和 16 位偏移量（Offset）合成20 位线性地址，即 Base + Offset。

3. BIOS 把启动设备（软盘、硬盘、光盘、USB 盘）的第一个扇区——主引导扇区 MBR（Master BootRecord）的 512 字节——加载到内存 0x7c00，然后跳转到 0x7c00 处的第一条指令执行。

4. bootasm.S 中 bootloader 先开 A20、置 CR0 的 PE 位使能保护模式，用 SEG_NULLASM /SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff) 等构造 GDT（null、code、data 三个描述符），长跳转刷新 CS 进入 32 位保护模式，再装入 PROT_MODE_DSEG 到 ds/es/fs/gs/ss、设 esp = start，最后call bootmain 。

5. bootmain（C 语言）先从磁盘读入 ELF 头并校验 ELF_MAGIC，再按 program header 逐段把p_offset/p_filesz 的内容读到 p_paddr，最后以 ((void (*)(void))ELFHDR->e_entry)() 跳转到内核入口。

6. 保护模式下段选择子不再直接给基址，而是作为 GDT/LDT 的索引取出段基址，加偏移得线性地址，再经分页机制（页目录/页表）转为物理地址。

**易错/考点：** 实模式段基址 = 选择子 << 4，保护模式选择子只是描述符索引；MBR 固定装入 0x7c00 是最常被问的数字。

### 1.1 课程概述（6.2 分钟）

**主旨：** 交代本课程的内容板块、实验体系与预备知识要求。

**关键点：** 

1. 课程内容分三块：基本概念及原理、操作系统介绍、中断及系统调用。

2. 每讲的组织形式为"课程简介 — 练习 — 延伸"，练习环节在 uCore 操作系统上做实验，代码地址https://github.com/chyyuu/mooc_os_lab，延伸环节讨论相关话题或故事。

3. 实验共 9 个：实验 0 准备、实验 1 系统启动及中断、实验 2 物理内存管理、实验 3 虚拟内存管理、实验 4 内核线程管理、实验 5 用户进程管理、实验 6 CPU 调度、实验 7 同步与互斥、实验 8 文件系统。

4. 预备知识要求：计算机组成原理（Intel 80386+）、数据结构、C 和汇编语言。

**易错/考点：** —

### 1.2 c函数调用的实现（7.1 分钟）

**主旨：** 用汇编看清 C 函数调用时栈帧如何建立、现场如何保存与恢复。

**关键点：** 

1. 调用者先用 pushal 以及 pushl %eax / %ebx / %ecx 保存通用寄存器现场，再 call foo ；返回后按 popl %ecx / %ebx / %eax 、popal 逆序恢复。

2. call 自动把返回地址压栈；进入被调函数后 pushl %ebp 保存调用者帧基址、movl%esp,%ebp 建立新栈帧，EBP 指向"Caller's EBP"。

3. 栈帧从高地址到低地址依次是：参数（Argument 01、Argument 02…）、返回地址、Caller's EBP，其下才是局部变量；EBP 作帧基址不动，ESP 随压栈向低地址移动。

4. 函数返回时 popl %ebp 恢复调用者的帧基址，ret 弹出返回地址回到调用点，栈帧随之销毁。

**易错/考点：** 栈向低地址方向增长，所以图中"Low"在上、"High"在下；EBP 恒定指向帧底、ESP 动态变化。

### 1.2 什么是操作系统（16.1 分钟）

**主旨：** 从"资源管理者"与"硬件抽象提供者"两个角度定义 OS，并给出 Kernel 的组成与四大特征。

**关键点：** 

1. OS 首先是资源管理器：负责管理外设、分配资源（画面 2:00 处要点为"资源管理 / 管理外设、分…"）。

2. OS 位于应用软件之下、硬件之上，为应用软件提供服务支撑；办公软件、视频播放软件等都运行在OS 之上（对应 Software—Application—System—Utility/Operating system—Shell/Kernel 的分层图）。

3. OS 把底层硬件抽象成上层可见的对象：CPU 抽象为进程、磁盘抽象为文件、内存抽象为地址空间。

4. Kernel 内部组件包括 CPU 调度器、物理内存管理、虚拟内存管理、文件系统管理、中断处理与设备驱动，uCore 按硬件层—内核（含系统调用接口、进程/线程管理、内存管理、文件系统）—库—应用的层次组织。

5. Kernel 特征之一是并发：计算机系统中同时存在多个运行的程序，需要 OS 管理和调度。

6. 另外三个特征是共享（"同时"访问与互斥共享）、虚拟（利用多道程序设计技术，让每个用户都觉得有一台计算机专门为他服务）、异步（程序执行不是一贯到底而是走走停停、推进速度不可预知，但只要运行环境相同，OS 需保证运行结果也相同）。

**易错/考点：** 并发（concurrency）不等于并行（parallelism）；异步的关键落点是"环境相同则结果必须相同"。

### 1.3 GCC内联汇编（12.9 分钟）

**主旨：** 掌握 GCC 内联汇编（inline assembly）的模板语法与约束符，从而能在 C 代码里直接插入特殊指令、发起系统调用。

**关键点：** 

1. 内联汇编是 GCC 对 C 语言的扩展，可直接在 C 语句中插入汇编指令；用途是调用 C 语言不支持的指令（如特权指令）或在 C 语言中手工优化。

2. 工作机制：用给定的模板和约束来生成汇编指令，在 C 函数内形成汇编源码。

3. 例 3 用 __asm__ volatile ("int $0x80" : "=a"(_res) : "a"(11),"b"(arg1),"c"(arg2),"d"(arg3),"S"(arg4)) 发起系统调用：11 作为系统调用号放入 eax，四个参数分别经ebx、ecx、edx、esi 传入。

4. 约束符含义：a = %eax、b = %ebx、c = %ecx、d = %edx、S = %esi、D = %edi，0 表示"与第一个操作数使用同一寄存器"。

5. 对应的编译结果为 movl $11,%eax；movl -28(%ebp),%ebx；movl -24(%ebp),%ecx；movl

-20(%ebp),%edx；movl -16(%ebp),%esi；int $0x80；movl %eax,-12(%ebp)——返回值经 eax写回 _res。

**易错/考点：** 画面以代码/终端为主；注意 "=a" 是输出约束、"0" 是复用第 0 个操作数寄存器的匹配约束。

### 1.3 为什么学习操作系统（16.4 分钟）

**主旨：** 说明 OS 课在整个计算机课程体系中的位置，以及学习它的工程价值与现实需求。

**关键点：** 

1. 这是一门综合课程，结合程序设计语言、数据结构、算法、计算机体系结构等多门不同的课程。

2. 尽管已经有 Windows、Linux 等很多成熟的操作系统，但操作系统仍在不停发展，在工控等各种领域有强烈需求。

3. 课程立论是操作系统很酷、很有用、有挑战。

4. 操作系统代码直接管理原始硬件，因而必须应对时间依赖行为、非法行为和硬件故障这三类问题。

5. 硬件层面要求良好的硬件管理与合理的资源分配，例如磁盘速度跟 CPU、内存相差很多，操作系统需要协调处理这种速度差异。

**易错/考点：** —

### 1.4 X86中的中断处理（14.6 分钟）

**主旨：** 讲清 x86 中断/异常如何经 IDT 定位到中断服务例程（ISR），以及特权级切换时的栈变化、从中断返回与系统调用的实现路径。

**关键点：** 

1. 每个中断或异常都与一个中断服务例程（ISR，Interrupt Service Routine）关联，这种关联关系存放在中断描述符表（IDT，Interrupt Descriptor Table）中。

2. IDT 的起始地址和大小保存在中断描述符表寄存器 IDTR 中；IDT 表项为 8 字节的门描述符，中断门（Interrupt Gate）与陷阱门（Trap Gate）均含 Segment Selector、Offset 31..16、Offset 15..0，以及 DPL、P、Type 字段。

3. 中断发生时 CPU 用中断号索引 IDT，取出段选择子与偏移，经 GDT/LDT 得到段基址，与 Offset 拼成 ISR 入口的线性地址，再查页表得物理地址，随后压栈并跳转执行（Figure 6-8 InterruptProcedure Call）。

4. 不涉及特权级变化时，硬件只压入 EFLAGS、CS、EIP；一旦发生特权级改变（Privilege-LevelChange），还要切换到内核栈并额外压入 SS、ESP（若有错误码 Error Code 也一并压入），即从Stack 1 换到 Stack 2。

5. 返回用 iret ：它弹出 EFLAGS 和 SS/ESP（是否弹 SS/ESP 取决于是否改变特权级），而 ret 只弹出 EIP、retf 弹出 CS 和 EIP。

6. 用户程序通过系统调用访问 OS 内核服务，实现手段包括使用 Trap、用软件产生中断，或使用专用指令（SYSENTER/SYSEXIT）；kern_init 中遍历 vectors 表，以 SETGATE(idt[i], 0, KERNEL_CS,__vectors[i], DPL_KERNEL) 初始化 IDT，并用 lidt 装入 IDTR。

**易错/考点：** iret 与 ret/retf 的弹栈差异、以及特权级变化时多压 SS/ESP 是高频考点；后半段画面以代码/终端为主（vectors.S、trap.c、Makefile）。

### 1.4 如何学习操作系统（1.9 分钟）

**主旨：** 给出本课程的学习方法立场——操作系统必须靠亲手实践来掌握，而非靠听讲和看书。

**关键点：** 

1. 画面板书写出学习效果的三级递进：「我听到的我会忘记」「我看到的我能记住」「只有我做过的我才能理解」。

2. 三者按感官参与度递增排列，把「做」置于「听」「看」之上，说明被动接收的留存率最低。

3. 由此推出本节的主张：理解操作系统原理要靠自己动手完成实现与调试，而不是记住结论。

**易错/考点：** 本节为学习方法引导，画面只有三行方法论文字，无技术考点。

### 1.5 完成过程1（29.1 分钟）

**主旨：** 通过实验一 bootloader 的编译、链接与调试，走通「BIOS 加电 → 加载主引导扇区 → 切换保护模式 → 跳入 C 代码」的完整启动链条。

**关键点：** 

1. 实验目标：另写一个 bootloader，完成切换到 x86 保护模式并显示信息，为加载 ucore OS 做准备；整个 bootloader 执行代码必须小于 512 字节，才能放进硬盘主引导扇区。

2. 构建链路为 gcc 编译 bootasm.S 与 bootmain.c 得到 bootblock.o，ld 以 -Ttext 0x7c00 链接，objcopy 转成二进制，终端打印「build 512 bytes boot sector: bin/bootblock 'success!'」。

3. 关键编译选项：-m32 生成 x86 代码、-gstabs 生成 stabs 调试信息、-ggdb 生成 gdb 调试信息、-nostdinc 不搜索标准头文件路径、-fno-stack-protector 不生成缓冲区溢出检测代码、-Os 减小代码体积。

4. bootasm.S 先 cli 关中断并把各段寄存器清零、建立栈，再通过 8042 键盘控制器端口0x64/0x60 打开 A20（先等控制器不忙，再发 0xD1 写输出端口指令，再发 0xDF）。

5. 保护模式切换顺序：装载 GDT（三个描述符：空、代码段 STA_X|STA_R 基址 0 限长 0xffffffff、数据段 STA_W ）→ 置 CR0.PE → ljmp $PROT_MODE_CSEG, $protcseg 进入 32 位代码段；GDT用「基址 0、限长 4GB」的平坦模型，使虚拟地址等于物理地址，切换过程中内存映射不变。

6. bootmain.c 先从磁盘读入 ELF 头，校验 e_magic == ELF_MAGIC ，再按 program header 逐个readseg 把段加载到 p_pa ，最后跳转到 e_entry ；读盘走 PIO，经 0x1F2–0x1F7 端口设置扇区号与读命令。

7. 调试方法：qemu 以 -S -s 启动后在 gdb 中 target remote localhost:1234 、break*0x7c00 单步跟踪 BIOS 之后的引导代码；用 -d in_asm -D bin/q.log 导出执行的汇编指令，与 bootasm.S / bootblock.asm 对照。

**易错/考点：** bootloader 的 512 字节上限、链接地址 0x7c00、CR0.PE 置位与 ljmp 远跳转的先后顺序是常考点；画面以代码/终端为主。

### 1.5 操作系统实例（5.5 分钟）

**主旨：** 用 UNIX、Linux、Windows 三大家族的谱系图，说明现实中的操作系统如何分化与衍生。

**关键点：** 

1. UNIX 家族以贝尔实验室的 UNIX 为根，向上分出 UNIX BSD（伯克利软件发行版）与 System V 两大分支。

2. 两大分支继续衍生出 Solaris、AIX/HP-UX、macOS 与 iOS 等商业系统，图中以厂商 logo 标注各支系归属。

3. Linux 家族以 Linux 内核（企鹅标志）为根，各发行版 Red Hat、Fedora、Ubuntu、SUSE、Debian、Gentoo 在其上组合不同软件栈。

4. Android 挂在 Linux 家族的分支上，说明移动操作系统复用了 Linux 内核。

5. Windows 家族以 PC 操作系统为主并逐渐向其他终端拓展（画面为微软各代产品与设备矩阵）。

**易错/考点：** BSD 是 UNIX 的一个分支而非独立体系，Android 属 Linux 家族而非 UNIX 家族，谱系归属易混。

### 1.6 完成过程2（26.9 分钟）

**主旨：** 以 lab1 的调试函数与 lab2 的中断框架为例，讲清「栈回溯」与「中断/异常从入口到分派再返回」的实现路径。

**关键点：** 

1. print_stackframe 先用 read_ebp() 、read_eip() 取当前帧，再循环打印 ebp、eip 以及(uint32_t*)ebp + 2 起的 4 个参数，然后以 eip = ((uint32_t*)ebp)[1] 、ebp =((uint32_t*)ebp)[0] 逐层上溯。

2. 回溯终止条件为 ebp == 0 或层数达到 STACKFRAME_DEPTH ，每层还调用print_debuginfo(eip-1) 打印函数名与行号。

3. IDT 表项由 struct gatedesc 的位域描述（gd_off_15_0、gd_ss、gd_type、gd_dpl、gd_p、gd_off_31_16），由 SETGATE(gate, istrap, sel, off, dpl) 宏按偏移高低位、段选择子、门类型与特权级填充。

4. 中断入口 __alltraps 依次压入 ds/es/fs/gs 与 pushal 保存通用寄存器，构造出 trapframe，再以 %esp 作为参数 call trap ，返回后 trapret 恢复寄存器、addl $0x8, %esp 丢弃中断号与错误码、iret 返回。

5. trap_dispatch 按 tf->tf_trapno 分派：T_PGFLT 交给 pgfault_handler 、T_SYSCALL交给 syscall ，其余走默认分支打印 trapframe。

6. 扩展练习用 T_SWITCH_TOU （120）与 T_SWITCH_TOK （121）两个中断配合switch_to_user() / switch_to_kernel() ，在 kern_init 调用的 switch_test() 中来回切换内核态与用户态。

7. kern_init 建立 GDT 与 TSS：ts.ts_esp0 指向 bootstack + KSTACKSIZE 作为内核栈顶，ltr(GD_TSS) 装载任务寄存器，从而在用户态陷入内核时自动换栈。

**易错/考点：** ebp 链的取值顺序（[ebp] 是新 ebp、[ebp+4] 是新 eip）、trapframe 压栈顺序与 iret 前清 8 字节错误码最易写错；画面以代码/终端为主。

### 1.6 操作系统的历史（11.4 分钟）

**主旨：** 从「纸带加载」到「多道并发」再到「普适计算」，解释操作系统为何演化成今天的样子。

**关键点：** 

1. 早期计算机用纸带传输程序和数据，操作系统只起到加载程序的作用。

2. CPU 等硬件速度快速提升后，单道执行使性能得不到充分利用，且内存容量越来越大，可以同时容纳多个程序。

3. 单道方式下程序发出 I/O 请求后走 startIO → waitIO → endIO ，CPU 在等待 I/O 期间空转（画面时序图中 read 与 I/O 设备严格串行）。

4. 多道程序方式下，一个程序 waitIO 时操作系统 schedule 切换到另一个程序，I/O 完成中断到来后再切回，使 CPU 计算与 I/O 重叠。

5. 引入 timer interrupt 后由时钟中断周期性触发 schedule ，形成抢占式分时，避免单个程序长期霸占 CPU。

6. 个人电脑操作系统的目标从「提高利用率」转为「用户体验与 API」：单用户、很多大型机服务和功能不再需要，但多应用协调与安全（电子商务、医疗记录）问题突出。

7. 演变路径为：主机计算（Mainframe computing）→ 个人计算（Personal computing）→ 网络计算（Internet computing）→ 普适计算（Pervasive computing，含大数据收集、云计算与个性化服务）→ Future OS。

**易错/考点：** 多道程序靠 I/O 中断切换、分时靠 timer interrupt 抢占，两种切换触发源易混。

### 1.7 操作系统结构（10.3 分钟）

**主旨：** 比较单体/分层内核、微内核、外核与虚拟机监视器四种组织方式，说明「哪些功能该留在内核」的不同答案。

**关键点：** 

1. 简单操作系统以 MS-DOS（1981 年起）为例，结构简单、缺乏模块划分与保护（画面右侧文字被截断，仅见「MS-DOS(1981~19…」）。

2. Unix 操作系统于 1972 年由 Kenneth Thompson 和 Dennis Ritchie 在贝尔实验室设计。

3. 分层视图：用户之上是 shells/命令、编译器和解释器、系统库，经 system-call interface 进入内核；内核内依次为信号与终端处理、字符 I/O 与终端驱动、文件系统与块 I/O、CPU 调度与页替换/请求调页/虚拟内存，最底层经内核接口对接终端控制器、设备控制器与内存控制器。

4. 该分层图被标注为「用于实验的 uCore 操作系统」的结构参照。

5. 微内核的设计思想是尽可能把内核功能移到用户空间，内核只保留 IPC 等基本机制，内存管理、文件系统、I/O、设备驱动、网络栈都以用户态服务进程形式挂在外围。

6. 外核（exokernel）是更极端的架构：内核只做硬件资源的安全复用与保护，POSIX、TCP、VM、DFS、IPC 等抽象交给用户态的 library operating systems。

7. VMM（虚拟机监视器）把单独的机器接口转换成很多幻象，每个虚拟机是一个原始计算机系统的有效副本并能完成所有处理器指令，从而让多个 Guest OS 共享同一套物理硬件。

**易错/考点：** 微内核（功能移到用户态）、外核（只复用硬件、抽象交给库 OS）、VMM（虚拟出多台机器）三者职责边界易混；3:51 处画面全黑（画面不清晰）。

### 1.8 小结（1.5 分钟）

**主旨：** 回顾本章的知识地图，把「是什么、为什么学、怎么学」与「实例、演变、结构」串成一条主线。

**关键点：** 

1. 小结逐条列出本章目录：课程概述、什么是操作系统？、为什么学习操作系统？、如何学习操作系统？。

2. 随后追加三条：操作系统实例、操作系统的演变、操作系统结构。

3. 七条目录即本章的组织顺序，前四条回答概念与动机，后三条给出实例、历史与结构三个视角。

**易错/考点：** 本节仅回顾章节目录，未展开新知识点。
