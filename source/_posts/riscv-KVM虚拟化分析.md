---
title: riscv KVM虚拟化分析
tags:
  - riscv
  - KVM
  - QEMU
  - 虚拟化
description: >-
  本文分析Linux内核里KVM相关的逻辑，体系架构基于riscv。具体调试的时候，我们
  使用了两层的qemu模型，第一层qemu使能了riscv的h扩展，第二层qemu使用kvm启动。
  全文使用从上到下的思路分析，如果需要了解相关的硬件特性可以直接跳到最后。 使用v5.19-rc8内核代码、6.2.0 qemu代码作为分析代码。
abbrlink: 44358
date: 2022-09-01 21:25:16
categories:
---

内核kvm基本框架
----------------

 kvm的入口函数在体系构架相关的代码里，riscv在arch/riscv/kvm/main.c里，riscv_kvm_init
 直接调用到KVM的总入口函数kvm_init，kvm_init创建一个/dev/kvm的字符设备，随后所有
 的kvm相关的操作都依赖这个字符设备。

 kvm_init的大概逻辑：
```
 kvm_init
       /*
        * 以riscv为例, 主要是做一些基本的硬件检测，比较重要的是gstage mode和vmid
        * 的检测。riscv里的两级地址翻译，第一级叫VS stage，第二级叫G stage，这里
        * 检测的gstage mode就是第二级翻译的配置。
        */
   +-> kvm_arch_init
   [...]
       /* 注册/dev/kvm的字符设备 */
   +-> misc_register
   +-> kvm_preempt_ops.sched_in = kvm_sched_in;
   +-> kvm_preempt_ops.sched_out = kvm_sched_out;
```
 /dev/kvm这个字符设备只定义了对应的ioctl，这个ioctl支持的最主要的功能是创建一个虚拟机。
 我们看下KVM_CREATE_VM的逻辑:
```
 kvm_dev_ioctl_create_vm
   +-> kvm_create_vm
         /* 分配gstage的pgd，vmid，guest的timer */
     +-> kvm_arch_init_vm
       /*
        * 这个ioctl会创建一个匿名文件，ioctl返回值是文件的fd, 这个fd就代表新创建的虚拟机，
        * 这个fd只实现了ioctl和release回调，release就是销毁虚拟机，ioctl用来配置虚拟机
        * 的各种资源，比如创建虚拟机的CPU(KVM_CREATE_VCPU)、给虚拟机配置内存(KVM_SET_USER_MEMORY_REGION)
        * 等等。
        */
   +-> file = anon_inode_getfile("kvm-vm", &kvm_vm_fops, kvm, O_RDWR)
```
 创建虚拟机的CPU的基本逻辑：
```
 kvm_vm_ioctl_create_vcpu
       /*
        * riscv的实现在arch/riscv/kvm/vcpu.c
        * 把HSTAUS_SPV, HSTATUS_SPVP, HSTATUS_VTW配置到虚拟机vcpu的软件结构里，
        * 这里SPV比较有意思，虚拟机启动的时候，会先根据如上软件结构里的HSTATUS
        * 更新hstatus寄存器，然后sret跳转到虚拟机启动的第一条指令，sret会根据SPV
        * 寄存器的值配置机器的V状态，这里SPV是1，sret指令会先把V状态配置成1，然后
        * 跳到虚拟机启动的第一条指令。这里描述的是虚拟机最开始启动时候的逻辑。
        */
   +-> kvm_arch_vcpu_create
         /* 配置vcpu的timer，实现为配置一个hrtimer，在定时到时，注入时钟中断 */
     +-> kvm_riscv_vcpu_timer_init
     +-> kvm_riscv_rest_vcpu
           /* 软件之前配置好的信息，在这个函数里写到硬件里 */
       +-> kvm_arch_vcpu_load
         +-> csr_write更新CSR寄存器
         +-> kvm_riscv_gstage_update_hgatp  更新hgatp
         +-> kvm_riscv_vcpu_timer_restore   更新htimedelta
         /*
          * 为每个vcpu创建一个匿名的fd，这个fd实现的回调函数有：release、ioctl和mmap，
          * ioctl提供vcpu的控制接口，比如，运行vcpu(KVM_RUN)等等。
          */
   +-> create_vcpu_fd
```
 给虚拟机配置内存:
```
 /* kvm_userspace_mem是从用户态传进来的虚拟机内存的配置信息 */
 struct kvm_userspace_memory_region kvm_userspace_mem;

 kvm_vm_ioctl_set_memory_region(kvm, &kvm_userspace_mem)
   +-> kvm_set_memory_region
     +-> __kvm_set_memory_region
       +-> kvm_prepare_memory_region
             /* arch/riscv/kvm/mmu.c */
         +-> kvm_arch_prepare_memory_region
               /*
                * 虚拟机的物理地址是host的用户态分配的一段虚拟内存，这里面有三个
                * 地址: 1. 这段虚拟地址的va；2. 这段虚拟地址对应的物理地址；3. 虚拟机
                * 的物理地址(gpa)，这三个地址对应的实际内存是相同的，但是各自的数值
                * 是不同的。实际上，第2级翻译是gpa->pa，但是host上申请到的va在host
                * S mode上的翻译是va->pa(页表基地址是satp)，所以，我们就要把gpa->pa
                * 的映射插到第2级翻译对应的页表里(hgatp)。
                * 
                * 我们自然会联想第2级翻译缺页在哪里处理，这个逻辑单独在下面看。
                */
           +-> gstage_ioremap
                 /* 配置第二级的页表 */
             +-> gstage_set_pte
       +-> kvm_create_memslot
       +-> kvm_commit_memory_region

```
 
 我们先看下vcpu run的逻辑：
```
 kvm_vcpu_ioctl
   +-> case KVM_RUN
     +-> kvm_arch_vcpu_ioctl_run
           /* arch/riscv/kvm/vcpu.c */
       +-> kvm_riscv_vcpu_enter_exit
             /* arch/riscv/kvm/vcpu_switch.S */
         +-> __kvm_riscv_switch_to
```
 __kvm_riscv_switch_to里把S mode的相关寄存器保存起来，换上VS状态的寄存器，然后sret
 跳到vcpu代码入口运行。vcpu的初始状态在如上vcpu create的逻辑中配置到vcpu的软件结构，
 通过这里的__kvm_riscv_switch_to配置到硬件CSR寄存器。

第2级翻译缺页的逻辑可以从vcpu_switch.S里的__kvm_riscv_switch_to入手看，这个函数
是vcpu运行的入口函数，在投入运行前，这个函数里把__kvm_switch_return这个函数的地址
配置给了stvec，当vcpu运行出现异常时，就会跳到__kvm_switch_return继续执行，这样就会
从上面的kvm_riscv_vcpu_enter_exit出来，继续执行kvm_riscv_vcpu_exit, 第2级缺页异常
在这个函数里处理：
```
 kvm_riscv_vcpu_exit
   +-> gstage_page_fault
         /*
          * 这个函数里会用host va(不是gpa)，判断是不是有合法的vma存在，如果有合法
          * 的vma存在，就可以分配内存，并且创建第二级页表，创建第2级map的时候使用
          * gpa->pa
          */
     +-> kvm_riscv_gstage_map
       [...]
```

如上是虚拟机进入以及运行的逻辑，在用户态看，就是进入一个ioctl，停在里面运行代码，
直到运行不下去了，ioctl就返回了，返回值以及ioctl的输出参数携带退出的原因和参数。
从kvm内部看，虚拟机退出是他执行指令的时候遇到了异常或者中断，异常或中断处理后从ioctl
返回到qemu线程的用户态。触发虚拟机退出的源头包括外设的MMIO访问，在构建虚拟机的地址空间
时，没有对外设的MMIO gpa对第二级映射，这样第二级翻译的时候就会触发缺页异常，kvm的
处理缺页的代码处理完缺页后就会退出虚拟机(vcpu run ioctl返回)。发生异常的指令的PC
保存在sepc里，qemu会再次通过vcpu run ioctl进来，然后通过sret从sepc处继续运行。
```
 /* arch/riscv/kvm/vcpu.c */
 kvm_arch_vcpu_ioctl_run
       /* 这里一进来run vcpu就处理MMIO，可能是上次时MMIO原因退出的，这样当然要接着MMIO的上下文继续跑 */
   +-> if (run->exit_reason == KVM_EXIT_MMIO)
               kvm_riscv_vcpu_mmio_return(vcpu, vcpu->run)
       /* 投入运行虚拟机, 异常后也从这里退出来 */
   +-> kvm_riscv_vcpu_enter_exit
       /* 处理异常*/
   +-> kvm_riscv_vcpu_exit
     +-> gstage_page_fault
       +-> emulate_load
             /* 在这里配置退出条件 */
         +-> run->exit_reason = KVM_EXIT_MMIO
```

随后独立考虑中断虚拟化。

qemu kvm的基本逻辑
-------------------

 [qemu tcg翻译执行核心逻辑分析](https://wangzhou.github.io/qemu-tcg翻译执行核心逻辑分析/)已经介绍了虚拟机启动的相关逻辑，从qemu构架上看
 kvm和tcg处于同一个层面上, 都是cpu模拟的一种加速器。

 虚拟机初始化逻辑:
```
 /* accel/kvm/kvm-all.c */
 kvm_init
   +-> qemu_open_old("/dev/kvm", O_RDWR)
       /* 创建虚拟机 */
   +-> kvm_ioctl(s, KVM_CREATE_VM, type);
       /* 虚拟机内存配置入口 */
   +-> kvm_memory_listener_register
     +-> kvm_region_add
       +-> kvm_set_phys_mem
         +-> kvm_set_user_memory_region
           +-> kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION, &mem)
```
 kvm vcpu线程启动的逻辑：
```
 riscv_cpu_realize
   +-> qemu_init_vcpu(cs)
         /* kvm对应的回调函数在：accel/kvm/kvm-accel-ops.c: kvm_vcpu_thread_fn */
     +-> cpus_accel->create_vcpu_thread(cpu)
         (kvm_vcpu_thread_fn)
       +-> kvm_init_vcpu
         +-> kvm_get_vcpu
               /* 创建vcpu */
           +-> kvm_vm_ioctl(s, KVM_CREATE_VCPU, (void *)vcpu_id)
       +-> kvm_cpu_exec(cpu)
             /* 运行vcpu */
         +-> kvm_vcpu_ioctl(cpu, KVM_RUN, 0)
```
 
riscv H扩展spec分析
-------------------

 riscv H扩展的目的是在硬件层面创建出一个虚拟的机器出来，基于此可以支持各种类型的
 虚拟化，比如，可以在linux上支持KVM。先不考虑中断和外设，我们看看要创建一个虚拟机
 我们需要些什么，我们需要GPR寄存器、系统寄存器以及一个“物理”地址空间，在这个虚拟机
 里运行的程序认为这就是他们的全部世界。我们可以把host的GPR和host的系统寄存器给虚拟
 机里的程序用，对于每个虚拟机和host，当他们需要运行的时候，由一个更底层的程序把他们
 的GRP值和系统寄存器值换到物理GPR和系统寄存器上，这样每次虚拟机和虚拟机切换、虚拟机
 和host切换都要切全部寄存器。不同虚拟机不能直接使用host物理地址作为他们的“物理”地址
 空间，如果这样，就要小心划分host物理地址，避免虚拟机物理地址之间相互影响，我们会
 再加一个层翻译，这层翻译把虚拟机物理地址翻异成host物理地址，虚拟机自身看不到这层
 翻译，虚拟机正常做load/store访问(先假设load/store访问的是虚拟机物理地址)，load/store
 执行的时候会查tlb，可能做page walk，还可能报缺页异常，这些在虚拟机的世界里都不感知，
 查tlb和做page talk是硬件自己搞定的，处理缺页是更加底层的程序搞定的(hypvisor)。
 为了支持这层翻译以及相关的异常，就需要在给硬件加相关的寄存器，可以想象，我们要增加
 这层翻译对应的页表的基地址寄存器，还要增加对应的异常上下文寄存器，这些寄存器在虚拟
 机切换的时候都要切换成对应虚拟机的。

 只有host的时候，只要一层翻译就好，但是如果是运行在虚拟机里的系统，就需要两级翻译，
 运行在虚拟机里的系统自己不感知是运行在虚拟机上的，但是，硬件需要知道某个时刻是运行
 的是guest还是host的系统，这样硬件需要有一个状态表示，当前运行的是guest还是host的
 系统。

 riscv的H扩展增加了CPU的状态，增加了一个隐式的V状态，当V=0的时候，CPU的U/M状态还和
 之前是一样的，S状态处在HS状态，当V=1的时候，CPU原来的U/S状态变成了VU/VS状态。
 V状态在中断或异常时由硬件改变，还有一个改变的地方是sret/mret指令。具体的变化逻辑
 是: 1. 当在V状态trap进HS时，硬件会把V配置成0; 2. CPU trap进入M状态，硬件会把V配置成0;
 3. sret返回时, 恢复到之前的V状态；4. mret返回时, 恢复到之前的V状态。这里说的之前
 的V状态，riscv的hstatus寄存器上的SPV(Supervisor Previous Virtualization mode)表示
 "之前的V状态"，上述的sret和mret从这个寄存器中得到之前的V状态。如前所述，kvm在启动
 虚拟机之前会配置hstatus的SPV为1，这样使用sret启动虚拟机后，V状体被置为1。

 增加了hypervisor和guest对应的两组寄存器，其中hypervisor对应的寄存器有: hstatus, hedeleg,
 hideleg, hvip, hip, hie, hgeip, hgeie, henvcfg, henvcfgh, hounteren, htimedelta, htimedeltah,
 htval, htinst, hgatp, guest对应的寄存器有：vsstatus, vsip, vsie, vstvec, vsscratch, vsepc,
 vscause, vstval, vsatp。

 对于这些系统寄存器，我们可以大概分为两类，一类是配置hypvisor的行为的，一类是VS/VU
 的映射寄存器。我们一个一个寄存器看下。VS/VU的映射寄存器就是CPU在运行在V状态时使用
 的寄存器，这些寄存器基本上是S mode寄存器的翻版，riscv spec提到，当系统运行在V状态
 时，硬件的控制逻辑依赖这组vs开头的寄存器，这时对S mode相同寄存器的读写被映射到vs
 开头的这组寄存器上。

 hedeleg/hideleg表示是否要把HS的中断继续委托到VS去处理，在进入V模式前，如果需要，
 就要提前配置好。具体的委托情况可以参考[这里](https://wangzhou.github.io/riscv中断异常委托关系分析/)
 这里需要注意的是，RV协议上提到，当H扩展实现时，VS的几个中断的mideleg对应域段硬件
 直接就配置成1了，也就是说默认被代理到HS处理。如果GEILEN非零，也就是有SGEI，那么
 SGEI也会直接硬件默认代理到HS处理。

 hgatp是第二级页表的基地址寄存器。

 hvip用于给虚拟机VS mode注入中断，写VSEIP/VSTIP/VSSIP域段，会给VS mode注入相关中断，
 riscv spec里没有说，注入的中断在什么状态下会的到响应？如果有多个VM实例，中断注入
 了哪个实例里?

 hip/hie是hypvisor下中断相关的pending和enable控制。hip/hie包含hvip的各个域段，除了
 如上的域段，还有一个SGEIP域段。协议上这里写的比较绕，先是总述了hip/hie里各个域段
 在不同读写属性下对应的逻辑是怎么样的，然后分开bit去描述。细节的逻辑是，hip.SGEIP
 是只读的，只有在hgeip/hgeie表示的vCPU里有中断可以处理时才是1，所以这个域段表示这个
 物理CPU上的vCPU是否有外部中断需要处理; hip.VSEIP也是只读的，在hvip.VSEIP是1或者
 hgeip有pending bit时，hip.VSEIP为1。

 hgeip/hgeie是SGEI的pending和enable控制，如果hgeip/hgeie是一个64bit的寄存器，那么
 它的1-63bit可以表示1-63个vcpu的SGEI pending/enable bit，每个bit描述直通到该vCPU
 上的中断，所以，协议上说要配合中断控制器使用。hgeip是一个只读寄存器。

 所以，VSEIP是一个SEIP的对照中断，而SGEI是一个直通中断的汇集信号。

 vcpu怎么响应这个直通的中断？我们把这个逻辑独立出来在这里描述:
 [https://wangzhou.github.io/riscv-AIA逻辑分析/](https://wangzhou.github.io/riscv-AIA逻辑分析/)

 罗列出riscv上所有的中断类型，S mode/M mode/VS mode的外部中断/时钟中断/软件中断
 这些一共下来就是9种中断类型，再加上supervisor guest external interrupt。
 
 htval/htinst是HS异常时的参数寄存器。htval用来存放guest page fault的IPA，其他情况
 暂时时0，留给以后扩展。两级翻译的具体流程在独立的文档中描述。

 这些寄存器的qemu实现比较有意思，VS在实际运行的时候会把vs开头寄存器中配置的值copy
 到S mode的寄存器上，把HS mode的寄存器中原来的值，存在硬件里。qemu上VS实际运行依赖
 的还是S mode寄存器上的当前状态，qemu的实现如果和协议一样，qemu在VS时对S mode寄存器
 的改写应该同时写到vsxxx这组寄存器上，VS状态的实际控制应该依赖于vsxxx这组寄存器，
 qemu目前的实现应该逻辑上也是对的。要让guest内核可以直接运行到KVM上，原来使用的
 寄存器名字也是不能改变的。

 在退出V状态的时候再把S mode的寄存器上的值保存会VS状态的寄存器上，同时把之前保存
 在硬件里的HS mode的寄存器的值写入S mode寄存器。HS工作的时候使用S mode寄存器，同时
 使用hypervisor寄存器里的配置信息。

 实际硬件的物理实现可能只需要做一个映射就好，并不需要qemu中类似的拷贝，如下是上面
 逻辑的示意图。

 虚拟机开始运行，HS切入VS的示意：
```
1. 配置vsxxx寄存器
                   |
                   |
                   v
 +-- vsstatus vsip ... vsatp ---- status  sip  ...  satp ---------+ <---- 寄存器接口
 |                                ^     \                         |
 |                               /       \                        |
 |    3. copy vsxxx to HS       /         \     2. save HS mode   |
 |       register or map vsxxx /           \       register in    |
 |       to HS register       /             \      hardware or    |
 |                           /               \     stop mapping   |
 |                          /                 v                   | <---- 硬件
 |   vsstatus vsip ... vsatp      status_hs  sip_hs ...  satp_hs  |
 |                                                                |
 +----------------------------------------------------------------+
 sret的硬件逻辑完成步骤2和步骤3。
```

 虚拟机停止运行，VS/VU切入HS：(初始状态是虚拟机跑在S mode寄存器上)
```
 +-- vsstatus vsip ... vsatp ---- status  sip  ...  satp ---------+ <---- 寄存器接口
 |                                /     ^                         |
 |                               /       \                        |
 |    1. copy HS mode register  /         \    2. copy HS register|
 |       to vsxxx or stop      /           \      saved in hw to  |
 |       mapping              /             \     register or do  |
 |                           /               \    mapping         |
 |                          v                 \                   | <---- 硬件
 |   vsstatus vsip ... vsatp      status_hs  sip_hs ...  satp_hs  |
 |                                                                |
 +----------------------------------------------------------------+
 中断或者异常的硬件处理逻辑完成步骤1和步骤2。
```
 所以，从总体上看，不管是在HS还是VS，实际运行的时候使用的都是S mode的寄存器。当HS是处理
 hypervisor的业务时，使用hypervisor相关寄存器里的定义。

 新增加的虚拟化相关的指令大概分两类，一类是和虚拟化相关的TLB指令，一类是虚拟化相关的访存指令。
 虚拟化扩展和TLB相关的指令有：hfence.vvma和hfence.gvma，虚拟化相关的访存指令有：hlv.xxx, hsv.xxx，
 这些指令提供在U/M/HS下的带两级地址翻译的访存功能，也就是虽然V状态没有使能，用这些指令依然可以
 得到gva两级翻译后的pa。

qemu riscv H扩展基本逻辑
------------------------

 qemu支持riscv H扩展的基本逻辑主要集中在中断和异常的处理逻辑，新增寄存器支持，以及
 新增虚拟化相关指令的支持。
```
 riscv_cpu_do_interrupt
       /* 从V状态进入HS，会把sxxx寄存器保存到vsxxx，把xxx_hs推到sxxx里 */
   +-> riscv_cpu_swap_hypervisor_regs(env)
       /* 保存当前状态 */
   +-> env->hstatus = set_field(env->hstatus, HSTATUS_SPVP, env->priv);
       /* 保存当前V状态 */
   +-> env->hstatus = set_field(env->hstatus, HSTATUS_SPV, riscv_cpu_virt_enabled(env));
       /* 保存异常gpa地址 */
   +-> htval = env->guest_phys_fault_addr;
       /* 后面可以看到cause, 异常pc, tval都是靠S mode寄存器报给软件的, 最后把模式切到S mode */
   +-> riscv_cpu_set_mode(env, PRV_S);

```
 在sret/mret指令里会处理V状态以及寄存器的倒换：
```
 helper_sret
     /* 在H扩展打开的分支里会有如下的硬件操作 */
   +-> prev_priv = get_field(mstatus, MSTATUS_SPP);
   +-> prev_virt = get_field(hstatus, HSTATUS_SPV);
   +-> hstatus = set_field(hstatus, HSTATUS_SPV, 0);
   +-> mstatus = set_field(mstatus, MSTATUS_SPP, 0);
   +-> mstatus = set_field(mstatus, SSTATUS_SIE, get_field(mstatus, SSTATUS_SPIE));
   +-> mstatus = set_field(mstatus, SSTATUS_SPIE, 1);
   +-> env->mstatus = mstatus;
   +-> env->hstatus = hstatus;
       /* 如果之前是V状态使能的，这里要做寄存器的倒换: 把S mode寄存器保存到xxx_hs，把vsxxx寄存器存到S mode寄存器里 */
   +-> riscv_cpu_swap_hypervisor_regs(env);
       /* 使能V状态 */
   +-> riscv_cpu_set_virt_enabled(env, prev_virt);
```
 
 新增了vsxxx以及hxxx寄存器的访问代码。新增加的虚拟化相关的指令大概分两类，一类是
 和虚拟化相关的TLB指令，一类是虚拟化相关的访存指令，可以直接查看他们的qemu实现,
 TLB相关的指令依然是刷新全部TLB，访存相关的指令和普通访存指令的实现基本一样，不同
 的是在mem_idx上增加了TB_FLAGS_PRIV_HYP_ACCESS_MASK，表示要做两级地址翻译。

运行情况跟踪
-------------

 1. 在第二层qemu的启动命令里加--trace "kvm_*"跟踪第二层qemu中kvm相关的配置，主要是
    一些kvm相关的ioctl。
 
 2. 在第一层qemu的启动命令里加-d int，观察第一层qemu上虚拟化相关的各种异常，实际
    上我们用第一层qemu模拟host机器，这里就是观察host机器在虚拟化下的各种异常。

 3. 在启动虚拟机的时候，kvm是把代码直接放到host机器上跑，tcg里-d参数的那些调试手段
    都起不上用处了，但是，guest的主要代码是直接运行在host机器上的，我们这里使用两层
    qemu，所以，guest的主要代码会跑在第一层qemu上，在第一层qemu加上-d选项是可以看到
    guest中运行的指令的。

    如下是在第一层qemu上增加-d cpu,in_asm，kvm启动guest内核的一段log，我们hack了下
    cpu的打印，只保留了V状态、pc和hstatus，不然一层qemu的启动就会慢的要命，不过，
    即使这样，两层qemu完全启动的全部log也有5G大小，我们把注释写到log内部。
```
----------------
IN: 
Priv: 1; Virt: 0
0xffffffff800155d8:  02153c23          sd              ra,56(a0)      <---- __kvm_riscv_switch_to对应的汇编
0xffffffff800155dc:  04253023          sd              sp,64(a0)            这个是kvm为guest指令指令准备环境。
0xffffffff800155e0:  04353423          sd              gp,72(a0)
0xffffffff800155e4:  04453823          sd              tp,80(a0)
0xffffffff800155e8:  f920              sd              s0,112(a0)
0xffffffff800155ea:  fd24              sd              s1,120(a0)
0xffffffff800155ec:  e54c              sd              a1,136(a0)
0xffffffff800155ee:  e950              sd              a2,144(a0)
0xffffffff800155f0:  ed54              sd              a3,152(a0)
0xffffffff800155f2:  f158              sd              a4,160(a0)
0xffffffff800155f4:  f55c              sd              a5,168(a0)
0xffffffff800155f6:  0b053823          sd              a6,176(a0)
0xffffffff800155fa:  0b153c23          sd              a7,184(a0)
0xffffffff800155fe:  0d253023          sd              s2,192(a0)
0xffffffff80015602:  0d353423          sd              s3,200(a0)
0xffffffff80015606:  0d453823          sd              s4,208(a0)
0xffffffff8001560a:  0d553c23          sd              s5,216(a0)
0xffffffff8001560e:  0f653023          sd              s6,224(a0)
0xffffffff80015612:  0f753423          sd              s7,232(a0)
0xffffffff80015616:  0f853823          sd              s8,240(a0)
0xffffffff8001561a:  0f953c23          sd              s9,248(a0)
0xffffffff8001561e:  11a53023          sd              s10,256(a0)
0xffffffff80015622:  11b53423          sd              s11,264(a0)
0xffffffff80015626:  46853283          ld              t0,1128(a0)
0xffffffff8001562a:  47053303          ld              t1,1136(a0)
0xffffffff8001562e:  6d853383          ld              t2,1752(a0)
0xffffffff80015632:  00000e97          auipc           t4,0            # 0xffffffff80015632
0xffffffff80015636:  0bae8e93          addi            t4,t4,186
0xffffffff8001563a:  46053f03          ld              t5,1120(a0)
0xffffffff8001563e:  100292f3          csrrw           t0,sstatus,t0

 V      =   0
 pc       ffffffff800155d8
 hstatus  0000000200000000
----------------
IN: 
Priv: 1; Virt: 0
0xffffffff80015642:  60031373          csrrw           t1,0x600,t1    <---- 更新hstatus, SPV配置为1，为sret切入V状态做准备。

 V      =   0
 pc       ffffffff80015642
 hstatus  0000000200000000
----------------
IN: 
Priv: 1; Virt: 0
0xffffffff80015646:  106393f3          csrrw           t2,scounteren,t2

 V      =   0
 pc       ffffffff80015646
 hstatus  00000002002001c0                                            <---- SPV已经为1
----------------
IN: 
Priv: 1; Virt: 0
0xffffffff8001564a:  105e9ef3          csrrw           t4,stvec,t4

 V      =   0
 pc       ffffffff8001564a
 hstatus  00000002002001c0
----------------
IN: 
Priv: 1; Virt: 0
0xffffffff8001564e:  14051e73          csrrw           t3,sscratch,a0

 V      =   0
 pc       ffffffff8001564e
 hstatus  00000002002001c0
----------------
IN: 
Priv: 1; Virt: 0
0xffffffff80015652:  141f1073          csrrw           zero,sepc,t5

 V      =   0
 pc       ffffffff80015652
 hstatus  00000002002001c0
----------------
IN: 
Priv: 1; Virt: 0
0xffffffff80015656:  12553c23          sd              t0,312(a0)
0xffffffff8001565a:  14653023          sd              t1,320(a0)
0xffffffff8001565e:  02753023          sd              t2,32(a0)
0xffffffff80015662:  01c53823          sd              t3,16(a0)
0xffffffff80015666:  01d53c23          sd              t4,24(a0)
0xffffffff8001566a:  36853083          ld              ra,872(a0)
0xffffffff8001566e:  37053103          ld              sp,880(a0)
0xffffffff80015672:  37853183          ld              gp,888(a0)
0xffffffff80015676:  38053203          ld              tp,896(a0)
0xffffffff8001567a:  38853283          ld              t0,904(a0)
0xffffffff8001567e:  39053303          ld              t1,912(a0)
0xffffffff80015682:  39853383          ld              t2,920(a0)
0xffffffff80015686:  3a053403          ld              s0,928(a0)
0xffffffff8001568a:  3a853483          ld              s1,936(a0)
0xffffffff8001568e:  3b853583          ld              a1,952(a0)
0xffffffff80015692:  3c053603          ld              a2,960(a0)
0xffffffff80015696:  3c853683          ld              a3,968(a0)
0xffffffff8001569a:  3d053703          ld              a4,976(a0)
0xffffffff8001569e:  3d853783          ld              a5,984(a0)
0xffffffff800156a2:  3e053803          ld              a6,992(a0)
0xffffffff800156a6:  3e853883          ld              a7,1000(a0)
0xffffffff800156aa:  3f053903          ld              s2,1008(a0)
0xffffffff800156ae:  3f853983          ld              s3,1016(a0)
0xffffffff800156b2:  40053a03          ld              s4,1024(a0)
0xffffffff800156b6:  40853a83          ld              s5,1032(a0)
0xffffffff800156ba:  41053b03          ld              s6,1040(a0)
0xffffffff800156be:  41853b83          ld              s7,1048(a0)
0xffffffff800156c2:  42053c03          ld              s8,1056(a0)
0xffffffff800156c6:  42853c83          ld              s9,1064(a0)
0xffffffff800156ca:  43053d03          ld              s10,1072(a0)
0xffffffff800156ce:  43853d83          ld              s11,1080(a0)
0xffffffff800156d2:  44053e03          ld              t3,1088(a0)
0xffffffff800156d6:  44853e83          ld              t4,1096(a0)
0xffffffff800156da:  45053f03          ld              t5,1104(a0)
0xffffffff800156de:  45853f83          ld              t6,1112(a0)
0xffffffff800156e2:  3b053503          ld              a0,944(a0)
0xffffffff800156e6:  10200073          sret                          <---- 配置V状态，跳到sepc，即guest内核首地址。

 V      =   0
 pc       ffffffff80015656
 hstatus  00000002002001c0
----------------
IN: 
Priv: 1; Virt: 1
0x0000000080000000:  5a4d              addi            s4,zero,-13   <---- guest内核首地址，第二层qemu没有配置bios，直接跑内核。
0x0000000080000002:  0ca0106f          j               4298            # 0x800010cc  <--- 跳转到了第2个4K页面上，触发了第二级地址翻译异常。
                                                                                          后面是退出到kvm里解决地址异常，退出到HS mode，
 V      =   1                                                                             V状态清0。(todo: 需要确定下)
 pc       0000000080000000
 V      =   0
 pc       ffffffff800156ec
 hstatus  00000002002001c0
 V      =   0
 pc       ffffffff800156f0
 hstatus  00000002002001c0
 V      =   0
 pc       ffffffff80015780
 hstatus  00000002002001c0
 V      =   0
[...]
```
