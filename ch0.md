# 第0章 实验课：环境准备与 Lab3

这批 6 个视频不是理论课，而是实验课（Lab0 环境准备 + Lab3 虚拟内存）。它们回答一个很实际的问题：这门课的知识怎么落到代码上。Lab3 的 do_pgfault 与 FIFO 换出，正好是第 5、6 章理论的一次完整实现，建议学完第 6 章后再回看。

### 0.0 清华 操作系统原理 Lab3.1（23.1 分钟）

**主旨：** 交代实验三虚拟内存管理的目标——在实验二的页表机制上实现 Page Fault（缺页异常）处理与页面置换，并给出两个必做练习、关键数据结构与页访问异常的处理路径。

**关键点：** 

1. 实验三在实验二"物理内存连续分配 + 二级页表"的基础上，借助页表机制和实验一的中断异常处理机制完成 Page Fault（缺页异常）处理与 FIFO 页面置换，原理上的最大区别是"在磁盘上缓存内存页"，从而提供一个比实际物理内存空间"更大"的虚拟内存空间。

2. 练习1 需在 LAB2 EXERCISE 1 处补全 do_pgfault(mm/vmm.c)，为未被映射的地址建立物理页映射；设置访问权限时要考虑所在 VMA 的权限，且必须操作内存控制结构所指定的页表，而不是内核页表，通过 check_pgfault() 测试会输出 "check_pgfault() succeeded!"。

3. 练习2 需补全 vmm.c 的 do_pgfault 以及 swap_fifo.c 中的 map_swappable 与 swap_out_victim，实现基于 FIFO 的页面替换，在 LAB2 EXERCISE 2 处填写代码，通过 check_swap() 测试即基本正确；扩展练习是实现识别 dirty bit 的 extended clock 页替换算法（非必做，酌情加分）。

4. 初始化顺序为 pmm_init()、pic_init()、idt_init()，随后由 kern_init 调用 vmm_init()（kern/mm/vmm.c），vmm_init 内部再调用 check_vmm()，由 check_vma_struct() 与check_pgfault() 分别自检 VMA 结构与缺页处理。

5. vma_struct 表示"使用同一页目录表的一段虚拟地址区间"，含 vm_mm、vm_start、vm_end、vm_flags 和按起始地址排序的 list_link；mm_struct 用 mmap_list 串起所有 VMA，用mmap_cache 缓存最近访问的 VMA 以加速查找，pgdir 指向页目录表，sm_priv 存放 swapmanager 的私有数据。

6. 页访问异常路径为 vector14 → alltraps → trap/trap_dispatch → pgfault_handler → do_pgfault；Page-Fault Error Code 中 P 区分"页不存在"与"页级保护违规"，W/R 区分读/写，US 区分用户态/超级用户态，RSVD 表示保留位被置 1，ID 表示是否由取指引起。

**易错/考点：** Error Code 各位含义（P=0 是缺页、P=1 是保护违规）以及 do_pgfault 必须使用 mm->pgdir 而非内核页表，是最易失分处。

### 0.0 清华 操作系统原理 Lab3.2（14.1 分钟）

**主旨：** 围绕页换入换出机制提出并回答五个设计问题：换出哪一页、虚拟页与磁盘扇区如何对应、何时换入换出、用什么数据结构支撑、以及具体怎么实现。

**关键点：** 

1. 换出页的选择由页替换算法决定（kern/mm/swap.c），课程列出 FIFO（First In First Out，Lab3-2必做）、Clock 与 Enhanced Clock 三种，实验用 check_swap() 验证。

2. 虚拟页与磁盘扇区的对应关系复用 PTE：页不在内存时该表项被解释为 swap_entry_t，24 位 offset记录磁盘上的换出位置，7 位 reserved 保留，最低位为 0 表明它指向的是 swap 项而非物理页。

3. 页表项（Figure 4-5）中 P=1 表示页存在、R/W=1 可写、U/S=1 允许 ring 3 访问、A=1 表示已被访问、D=1 表示已被写过，这些标志正是替换算法挑选牺牲页的依据。

4. 换入由缺页驱动：check_swap() 触发 page fault 后进入 do_pgfault() 把页从磁盘读回；换出分主动策略与被动策略，ucore 采用被动策略，即内存不足时才换出。

5. struct Page 增加 pra_page_link 与 pra_vaddr 两个域，前者把可换出的物理页串成链表、后者记录该页对应的虚拟地址；swap_manager 抽象出 map_swappable（把页加入可换出集合）、swap_out_victim（挑出牺牲页并换出）、check_swap（自检）三个函数指针。

6. swap_check() 的测试场景是：mm_create() + vma_create() 建立 4KB~24KB 的访问区间，free_page 只保留 4 个物理页却连续访问 5 个虚拟页，人为制造缺页，再由check_content_set/check_content_access 与 _fifo_check_swap() 校验置换结果。

**易错/考点：** 页不在内存时 PTE 被复用为 swap_entry_t（最低位为 0），不要与存在位 P 混淆；FIFO 只按进入内存的先后淘汰，完全不看访问频度。

### 0.0 清华 操作系统原理 Lab3.3（31.4 分钟）

**主旨：** 以源码走查的方式串起从 kern_init 到 do_pgfault、再到 FIFO 换出的完整执行路径，指明练习代码应填在哪些函数里。

**关键点：** 

1. kern_init() 依次调用 pmm_init()、pic_init()、idt_init()、vmm_init()、ide_init()、swap_init()、clock_init()，虚拟内存与换页子系统在时钟中断使能之前完成初始化。

2. check_vma_struct() 先用 mm_create() 建 mm、用 vma_create(0, PTSIZE, VM_WRITE) 建 VMA插入 mmap_list，再按 step 批量创建与查找 VMA，校验链表按 vm_start 有序且 find_vma 命中正确区间。

3. check_pgfault() 建好 mm 与 VMA 后循环写 0~4 页触发缺页；do_pgfault 中先用 get_pte(mm->pgdir, addr, 1) 取页表项，若该表项为 0 则用 pgdir_alloc_page 分配物理页并建立映射。

4. do_pgfault 的另一分支处理换入：PTE 存在但最低位为 0 时是 swap 项，调用 swap_in(mm, addr,&page) 从磁盘读回，再用 page_insert 更新页表，并用 swap_map_swappable 把该页设为可换出。

5. swap_fifo.c 中 fifo_init 初始化 free_list，fifo_map_swappable 用 list_add 把页的 pra_page_link挂到队尾，fifo_swap_out_victim 从队首取第一个页作为牺牲页并 list_del，体现"先进先出"。

6. 换出时先判断页是否被写过，脏页需用 swapfs_write 写回磁盘，之后把 PTE 改写为 swap_entry_t并调用 tlb_invalidate 刷新 TLB（快表）。

**易错/考点：** 画面以代码/终端为主；核心是 do_pgfault 的"分配新页"与"从 swap 换入"两条分支，以及FIFO 用 list_add 入队尾、list_del 取队首的配对关系。

### 0.1 实验准备1（11.7 分钟）

**主旨：** Lab0 实验准备开篇，说明课程实验的环境构成、工具链与 x86-32 硬件基础，并演示如何下载、导入 VirtualBox 实验镜像。

**关键点：** 

1. 实验在预装好的 Ubuntu 虚拟机上完成，工具分为 shell 命令（ls、cd、rm、pwd 等）、系统维护工具（apt、git）、源码阅读与编辑工具（understand、gedit、vim）、源码比较工具（diff、meld）、开发编译调试工具（gcc、gdb、make）以及硬件模拟器 qemu。

2. x86-32 硬件基础包括 Intel 80386 的运行模式概述、内存架构概述、寄存器概述三部分，此外还需了解 ucore 的面向对象编程方法与通用数据结构。

3. 虚拟硬盘压缩包放在网盘（pan.baidu.com/s/1gd6PM6J），解压后得到 mooc-os.vbox、mooc-os.vbox-prev、mooc-os.vdi 三个文件，合计约 4GB 多，用 VirtualBox 打开 .vbox 即可加载。

4. 虚拟机登录用户名是 moocos，口令是"空格键"。

5. 在 VirtualBox"新建虚拟机"向导中为 mooc-os 分配内存，推荐 512MB，界面可调范围是 4MB 到2048MB。

**易错/考点：** 口令是空格键而不是空字符串，且导入时应通过 .vbox 配置文件加载，不要直接打开 .vdi。

### 0.2 实验准备2（23.7 分钟）

**主旨：** 在虚拟机里配好共享文件夹，并用真实命令演示 shell 操作、依赖安装、源码检索、git 提交、make 编译与 gdb 调试的完整实验工作流。

**关键点：** 

1. 在 VirtualBox 的"设置→共享文件夹"中把宿主机 /tmp 挂载进虚拟机，名称 /tmp、自动挂载"是"、访问权限"固定分配"，用于宿主机与 guest 之间传文件。

2. shell 命令 ls、cd、rm、pwd、mkdir、find 基于 bash（Bourne-Again SHell），完成对文件与目录的基本操作。

3. prepare.sh 用 apt-get 一次性装齐实验依赖：build-essential、git、qemu-system-x86、vim-gnome、gdb、make、diffutils、exuberant-ctags、tmux、openssh-server、cscope、meld，执行前先 sudo apt-get update。

4. 源码定位靠 find ./labcodes/ -name "*.c" 一类组合（如再接 | less）快速找到 kern/init/init.c、kern/mm/vmm.c 等目标文件。

5. git 流程为：修改 README 后 git status 显示 "Changes not staged for commit"，git add README进入暂存区，git commit -m "update README" 提交后工作区变 clean。

6. gcc 是 C 语言编译器、gdb 是执行程序调试器、make 是软件工程管理工具且必须依赖 makefile 才知道如何编译链接；执行 make qemu 后 ucore 打印 "(THU.CST) os is loading ..." 与 "Kernelexecutable memory footprint: 64KB"。

**易错/考点：** make 离开 makefile 无法工作，qemu 只是硬件模拟器，需与 gdb 配合才能做内核源码级调试。

### 0.3 实验准备（31.6 分钟）

**主旨：** 补齐 x86-32 的运行模式、地址空间与寄存器基础，说明 ucore 如何用 C 语言模拟面向对象与通用双向链表，最后演示 Eclipse 配合 gdb 的源码级调试。

**关键点：** 

1. 80386 有实模式、保护模式、SMM 模式和虚拟 8086 模式四种运行模式；加电后处于实模式，可访问的物理内存不能超过 1MB，无法发挥 32 位 CPU 的 4GB 内存管理能力。

2. 保护模式支持内存分页机制从而良好支持虚拟内存，并支持多任务与 0~3 共 4 个优先级：操作系统运行在最高优先级 0，应用程序运行在较低级别，配合检查机制既能在任务间安全共享数据也能很好地隔离任务。

3. 地址是访问内存空间的索引；80386 是 32 位处理器，可寻址物理内存为 2^32=4GB，物理地址是处理器提交到总线上访问内存和外设的最终地址、一个计算机系统只有一个物理地址空间，而线性地址空间是各应用程序在虚存管理下看到的、相互隔离的地址空间。

4. 80386 寄存器分 8 组：通用寄存器、段寄存器、指令指针寄存器、标志寄存器、控制寄存器、系统地址寄存器、调试寄存器、测试寄存器。

5. EIP 的低 16 位就是 8086 的 IP，存放下一条要执行指令的地址，在分段地址转换中表示段内偏移；EFLAGS 的 IF（中断允许标志）由 CLI/STI 控制，只对可屏蔽外部中断有效，对不可屏蔽中断和故障中断没有作用。

6. ucore 用 C 语言模拟面向对象：pmm_manager 把 init、init_memmap、alloc_pages、free_pages、nr_free_pages、check 组织成函数指针表；通用双向链表 list_entry 只有 prev/next两个指针，再靠 le2page 与 to_struct（(char *)(ptr) - offsetof(type, member)）从链表节点反推出宿主 Page 结构的首地址；Eclipse 中在 kern_init 下断点后可单步进入 cons_init 观察执行。

**易错/考点：** to_struct 是用节点地址减去成员偏移反推宿主地址、le2page 是它的特化用法；IF 位对 NMI（不可屏蔽中断）无效是常考细节。
