---
title: qemu tcg中helper函数的使用和实现分析
tags:
  - QEMU
description: >-
  之前已经在各种qemu的分析中提到了helper函数，本文把这些信息总结整理到一起，
  方便随后查看。helper函数中使用的各种技术也总结在这篇文档之中。分析基于qemu v7.1.50版本。
abbrlink: 35098
date: 2023-09-14 19:24:57
categories:
---

基本逻辑
---------

qemu tcg中的helper函数想要达到的目的是在tb里插入一个host上的函数来模拟guest CPU
的行为，helper函数可以改变guest CPU里的数据。

一个helper函数要完整运行起来的基本逻辑大概可以分为: 1. 静态定义；2. 前端翻译；3.后端翻译。
静态定义是用qemu提供的API来定义helper函数，并实现helper函数的逻辑，静态定义的helper
函数可以被qemu提前识别，为后面helper函数的前端翻译做好准备。qemu的前端翻译在运行
helper函数(gen_helper_xxx)时，其实是把helper函数的生成信息进行转化，打包成CALL中间码。
后端翻译是根据CALL IR生成具体的host侧函数调用代码。

如上一系列工作完成后，在tb运行时就可以直接调用到定义的helper函数。

helper函数实现分析
-------------------

本节具体分析helper函数的实现逻辑，如果只是看helper函数的写法，可以跳过本节。

静态定义是helper函数使用者唯一需要做的事，我们这里看看qemu内部是怎么支持的，下面
一节说明了怎么加一个helper函数。可以看到和一个helper函数有关的对象有三个: help.h
里定义的宏在不同的地方展开成两个不同的对象，op_helper.c里定义的helper函数逻辑本身。

分别看下这三个对象，helper函数逻辑本身比较直白，就是最后要调用的那个函数的逻辑，
所以它的名字一般是helper_xxx。

helper.h里定义的宏在tcg/tcg.c里展开成struct TCGHelperInfo，并被加入到all_helpers
这个helper函数定义的全局表里，然后依次加入helper函数全局hash table，这个过程中会
更新函数参数的配置参数：
```
/* tcg/tcg.c */
tcg_context_init
      /* 在这个函数中调用的layout_arg_xxx函数中，这里重点关注kind这个参数 */
  +-> init_call_layout(&all_helpers[i]);
  +-> g_hash_table_insert
```
前端翻译会把helper函数的参数的配置整合到CALL IR的参数里。

helper.h里定义的宏在translate.c里展开成gen_helper_xxx这样的前端翻译函数，可以看到
translate.c里展开了一堆函数，这里translate.c里include了helper-gen.h，helper-gen.h
里include了helper.h。

展开的这个gen_helper_xxx函数是"生成helper函数"的函数，核心逻辑就是使用tcg_gen_callN
生成CALL中间码。qemu在前端翻译的时候执行gen_helper_xxx插入CALL中间码。

[这里](https://wangzhou.github.io/qemu模拟系统指令/)以riscv的ecall指令模拟为例展示了helper函数的静态定义。

前端翻译的逻辑比较简单，其实就是调用tcg_gen_callN生成CALL IR。

todo: 具体看下后端翻译。

helper函数写法
---------------

以riscv为例，增加一个helper函数的一般套路是: 1. 在target/riscv/op_helper.c里增加
函数的定义；2. 在target/riscv/helper.h增加对应的宏，宏的参数分别是：helper函数名字、
函数的返回值、函数的入参；3. 在中间码里用gen_helper_xxx直接调用helper函数，返回值
保存在gen_helper_xxx的第一个参数里，常数入参需要用tcg_const_i32/i64生成下常数TCGv，
实际上是为这个常数分配TCG寄存器存储空间。

在具体定义helper函数的时候，函数的输入、输出和flag可以有多种类型。flags描述helper
函数的一些属性，qemu后端可以根据这些属性做一定的优化。这里重点看下输入输出的各种
标记，总体上看helper函数的输入输出标记可以有i32/i64/int/s32/ptr/f64/f32/f16/s64/i128/cptr/tl/void/noreturn，
其中void/noreturn只用于返回值，根据qemu的内部定义，f16/f32和i32一样，f64和i64一样，
cptr和ptr一样，int和s32一样，tl(target long)会根据guest系统定义成i32或i64，所以
不重复的输入输出标记有i32/i64/s32/s64/i128/ptr/void/noreturn。

todo: 为什么这里必须使用对应的输入标记？

需要注意的是，如果要在helper函数里改系统寄存器的值，应该把系统寄存器的索引作为常量
传入helper函数。也可以把想要修改的值通过helper函数的返回值返回，再在helper函数外部
修改系统寄存器的值。

相关文章链接
-------------

关于call中间码和helper函数的内容，我们之前有过分析，call IR的分析可以参考[这里](https://wangzhou.github.io/qemu-tcg中间码优化和后端翻译/)。
有关怎么加helper函数的简介可以参考[这里](https://wangzhou.github.io/qemu-tcg翻译执行核心逻辑分析/)。
关于helper函数静态定义的一些分析可以参考[这里](https://wangzhou.github.io/qemu模拟系统指令/)。
