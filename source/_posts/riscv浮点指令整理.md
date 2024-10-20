---
title: riscv浮点指令整理
tags:
  - riscv
description: 本文整理riscv上的浮点指令。整理依赖的spec版本是20191213，以来的qemu版本是v8.1.0-rc3。
abbrlink: 967
date: 2023-08-25 22:48:27
categories:
---

基本逻辑
---------

riscv当前的版本提供了多个浮点指令支持的扩展，它们分别是单精度浮点(F)、双精度浮点(D)、
四倍精度浮点(Q)以及目前只是一个占位符的十进制浮点(L)。

riscv在提供浮点指令的同时也增加了32个64bit的浮点指令寄存器(f0-f31)，以及一个浮点
特性相关的控制寄存器(fcsr)。

riscv浮点指令再从功能上细分，大概可以分为：计算相关，IO相关，转换相关。riscv的浮点
指令汇编命名的规则是: f + 功能 + .精度定义，比如，fadd.s就是单精度浮点加法。

具体指令
---------

列一个表格总结下浮点相关的指令，必要的地方直接给出说明：
```
--------------------------------------------------------------------
 计算：                                                             
                                                                    
     基础计算：                                                     
                                                                    
           fadd.s    fadd.d                                         
           fsub.s    fsub.d                                         
           fmul.s    fmul.d                                         
           fdiv.s    fdiv.d                                         
           fsqr.s    fsqr.d                                         
           fmin.s    fmin.d                                         
           fmax.s    fmax.d                                         
          fmadd.s   fmadd.d    乘加，两数相乘再加上一个数           
         fnmsub.s  fnmsub.d    相乘的结果取反后再减去一个数         
                                                                    
      符号注入：                                                    
                                                                    
          fsgnj.s   fsgnj.d    符号注入都是取rs1绝对值, 取rs2的符号 
         fsgnjn.s  fsgnjn.d    取rs2的符号再取反                    
         fsgnjx.s  fsgnjx.d    取rs2/rs1的符号的异或作为符号        
                                                                    
      比较：                                                        
                                                                    
            feq.s     feq.d                                         
            flt.s     flt.d                                         
            fle.s     fle.d                                         
--------------------------------------------------------------------
 转换：                                                             
                                                                    
     寄存器数据移动：                                               
                                                                    
            fmv.sx              转换指令的两个寄存器都是后一个是源，
            fmv.xs              前一个是目的，比如sx是，s <- x。    
                                                                    
     数据格式转换：                                                 
                                                                    
            fcvt.sw             s: 单精度，d: 双精度，              
            fcvt.dwu            w: word(32bit), wu: unsigned word   
            fcvt.wus                                                
            fcvt.wud                                                
            fcvt.sd                                                 
            fcvt.ds                                                 
--------------------------------------------------------------------
  IO和分类：                                                        
            flw                                                     
            fsd                                                     
            fclass.s fclass.d   返回一个浮点数的分类                
--------------------------------------------------------------------
```

qemu实现
---------

运算相关的浮点指令，qemu的实现都是使用helper函数，在helper函数里先尝试用硬件实现
直接计算，精度不符合要求的话再调用qemu里的软浮点函数完成运算。这里需要注意的是，
浮点运算里可以配置不同的rm值，确定数据的取舍类型，qemu会根据硬件的具体配置，传入
不同的rm值。
