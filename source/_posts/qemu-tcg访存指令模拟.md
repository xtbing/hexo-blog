---
title: qemu tcg访存指令模拟
abbrlink: 19272
date: 2022-07-25 15:45:10
tags: [QEMU, 内存管理]
description: "本文分析qemu tcg里关于load/store的流程，以riscv平台为分析对象。qemu的版本是5.1.50。
      CPU访存主要有使用load/store的显示访存也有隐式访存，比如CPU取指令就属于一种
      隐式访存，qemu对CPU取指令的模拟逻辑可以参考: htts://wangzhou.github.io/qemu-tcg取指令逻辑分析/"
categories:
---

qemu load/store基本流程模拟
----------------------------

qemu里load/store分两种，一种是load/store guest地址，对应的中间码是qemu_ld/st_xxx，
另一种是load/store的是host地址，对应的中间码是ld/st_xxx，这一节中我们讨论的是前者。
qemu里load/store在system mode和user mode下的模拟又是不一样的，到了具体地方，我们
会具体说明下。

我们从一个普通的load指令的模拟代码入手分析。
/* qemu/target/riscv/insn_trans/trans_rvi.c.inc */
```
  trans_lw
    -> gen_load
      -> tcg_gen_qemu_ld_tl
        -> gen_ldst_i64 INDEX_op_qemu_ld_i64
```
如上，load操作的中间码是用gen_ldst_i64生成的，中间码的op code是INDEX_op_qemu_ld_i64。

正常的流程，翻译成中间码后，还要有中间码翻译成host指令的过程，qemu把load/store里
处理tlb访问、页表page walk、发起缺页异常的这些操作插到了把中间码翻译成host指令的过程中。
接着上面INDEX_op_qemu_ld_i64往下分析。

中间码翻译成host指令在tcg_gen_code函数里，tcg_gen_code一路调用下去，会到
tcg/riscv/tcg-target.c.inc里的tcg_out_qemu_ld_slow_path。
```
  tcg_gen_code
       /* tcg/riscv/tcg-target.c.inc, 这里我们假设host平台也是riscv */
    -> tcg_out_op
         /* 可以看到这里分了有MMU和没有MMU的情况，一般user mode是没有MMU的 */
      -> tcg_out_qemu_ld
           /*
            * 有MMU的情况，tlb_load这个函数检测是否tlb可以直接命中，如果命中，直接
            * 就load数据了。没有MMU的情况, 计算地址后直接用tcg_out_qemu_ld_direct
            * 生成的host指令load数据，需要注意的是没有MMU的时候，必然也没有tlb了,
            * 另一个需要注意的点是，没有MMU的时候，qemu会直接load/store guest PA，
            * 但是guest PA和host VA逻辑上并不相等，两者之差保存在全局变量guest_base
            * 中，并在tcg_target_qemu_prologue里传递给TCG_GUEST_BASE_REG寄存器，
            * 所以可以看到这里会结合TCG_GUEST_BASE_REG计算出对应的host VA，然后
            * 针对host VA做load。
            *
            * tlb查找直接用tb里生成的host汇编指令做了，得到pa放到TMP0寄存器里,
            * 如下生成的host汇编里插入了跳转指令，如果tlb有命中就继续执行
            * tcg_out_qemu_ld_direct里生成的汇编指令，如果tlb没有命中就跳转到慢速
            * 路径的代码上执行，后端翻译到这里的时候还不能确定慢速路径的跳转目的
            * 地址是什么，所以可以看到如下函数里生成的跳转指令的目标先配置成了0。
            */
        -> tcg_out_tlb_load(s, addr_regl, addr_regh, oi, label_ptr, 1)
           /*
            * tlb hit时，直接做load, 上一步得到的guest物理地址，计算得到host虚拟
            * 地址，通过寄存器传递给下面生成的指令。guest物理地址和host虚拟地址
            * 只相差一个addend。
            */
        -> tcg_out_qemu_ld_direct(s, data_regl, data_regh, base, opc, is_64)
           /*
            * 创建一个lable的描述信息，放到后端解码的上下文里，这里没有生成后端指令。
            *
            * 慢速路径的返回地址: label->raddr = tcg_splitwx_to_rx(code_ptr)
            * 跳转指令的地址记录在: label->label_ptr[0]
            */
        -> add_qemu_ldst_label(s, 1, oi, (is_64 ? TCG_TYPE_I64 : TCG_TYPE_I32),
                               data_regl, data_regh, addr_regl, addr_regh,
                               s->code_ptr, label_ptr)
           /*                                                      
            *    注意这里code_ptr已经是上面ld_direct生成指令的后面 
            *    一个地址了，而label_ptr里的内容还是bne指令对应的  
            *    地址。                                            
            *                tb code buff                          
            *                  ---+---                             
            *    find tlb --->    |                                
            *                     |                                
            *                     |                                
            *                     |                                
            *                    bne <-------- label_ptr[0]     --------+
            *                     |                                     |
            *    direct ld --->   |                                     |
            *                     |                                     |
            *                    -+-                                    |
            *    code_ptr ------> |  <------ other code  <---------+    |
            *                     |                                |    |
            *                    -+-                               |    |
            *                     |  <------ ld slow path code  <--+----+
            *                     |                                |
            *                    goto  ----------------------------+
            *
            * 所以，bne需要跳到慢速路径上，慢速路径的执行结果有两种:
            * 1. 找见了pa，填充了tlb，这种情况应该回到快速路径上执行，
            *    可以看到慢速路径后面是返回到了code_ptr这里继续执行。
            * 2. 有异常产生，软件处理异常后，重新从ld指令开始翻译执行。
            */
       /*
        * tcg/tcg-ldst.c.inc, 解析ldst label, 生成慢速路径的代码，补上上面bne的
        * 跳转目的地址。
        */
    -> tcg_out_ldst_finalize
         /* tcg/riscv/tcg-target.c.inc */
      -> tcg_out_qemu_ld_slow_path
           /*
            * label_ptr[0]保存的是bne的地址, 但是注意这里的code_ptr是慢速路径指令
            * 的起始地址，如下函数找见上面bne指令的地址，然后把慢速路径的跳转地址
            * 补起来。
            */                                         
        -> reloc_sbimm12(l->label_ptr[0], tcg_splitwx_to_rx(s->code_ptr)))

           /* 各种不同数据宽度的load函数被定义到qemu_ld_helpers这个表里 */
        -> tcg_out_call(s, qemu_ld_helpers[opc & MO_SSIZE])
              /* 取ldul为例子，这是一个公共函数，定义在accel/tcg/cputlb */
           -> helper_le_ldul_mmu
                /* accel/tcg/cputlb.c */
             -> load_helper

           /* 如上的函数要把数据放到a0里，所以慢速路径是要把整个load执行完 */
        -> tcg_out_mov(s, (opc & MO_SIZE) == MO_64, l->datalo_reg, a0)
           /* 跳到direct ld结束的地址 */
        -> tcg_out_goto(s, l->raddr)
```

在load_helper里做如上提到的load的各种硬件模拟。load_helper里对于没有找到tlb的情况
会调用体系构架相关的回调函数做tlb fill：
```
  tlb_fill
       /* target/riscv/cpu.c */
    -> cc->tcg_ops_tlb_fill risv下是riscv_cpu_tlb_fill
         /*
          * 先做page walk，翻译成功自然可以，翻译失败报缺页异常，我们把page walk
          * 的逻辑分析独立放到下面。
          */
      -> riscv_raise_exception
```

riscv page walk分析
--------------------

 qemu riscv page walk基本逻辑可以参考[这里](https://wangzhou.github.io/qemu-tcg中riscv-page-table-walk分析/)。

load/store host地址
--------------------

 我们分析riscv上的ld_i64的实现，它对应的中间码是INDEX_op_ld_i64。可以看见qemu后端
 会直接把中间码翻译成host load/store指令，当中间码里的offset参数和host load/store
 指令里的offset立即数的位宽不匹配的时候，qemu后端会做必要的调整。
```
 /* tcg/riscv/tcg-target.c.inc */
 tcg_out_ldst
   /*
    * 根据load/store的基地址寄存器的分配情况决定host load/store offset立即数位宽
    * 不够的时候该怎么办: 基地址寄存器分配了0寄存器时要加入auipc计算地址，基地址
    * 寄存器可用时，只要重新调整下基地址寄存器里的数值就好。
    */
   [...]
   /* 直接生成host load/store指令 */
   tcg_out_opc_store()/tcg_out_opc_imm()
```

注意在qemu代码(v8.0.0)的docs/devel/loads-stores.rst有qemu内部各种内存访问API的说明。
