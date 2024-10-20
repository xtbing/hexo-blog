---
title: Linux内核cleanup特性
tags:
  - Linux内核
description: 本文梳理Linux内核中的cleanup特性的基本逻辑。梳理基于的内核版本是v6.10-rc4。
abbrlink: 16617
date: 2024-07-23 23:07:56
categories:
---

基本介绍
---------

Linux内核补丁“[v3,00/57] Scope-based Resource Management”向内核增加了资源自动释放
的机制，“小范围”内成对使用的资源可以直接使用这种机制，比如成对使用的内存申请、成对
使用的加锁和释放锁操作。

这个特性的原理其实比较直白，就是使用了GCC里的cleanup属性，在定义变量的时候cleanup
属性容许为这个变量添加一个函数，GCC编译的时候在这个变量生命周期结束的地方插入对应
的函数调用。

一个简单的C代码的示例如下：
```
#include <stdio.h>

void print(int *t)
{
	printf("cleanup: %d\n", *t);
}

void do_something(void)
{
	printf("do something\n");
}

int main()
{
	printf("main function start\n");
	int a __attribute__((__cleanup__(print))) = 10;

	for (int i = 0; i < 3;  i++) {
		int b __attribute__((__cleanup__(print))) = i;
		do_something();
	}

	return 0;
}
```
编译运行的输出是：
```
sherlock@m1:~/tests/cleanup$ ./a.out 
main function start
do something
cleanup: 0
do something
cleanup: 1
do something
cleanup: 2
cleanup: 10
```
可以看出，a的生命周期是整个main函数，所以在最后调用print，b的生命周期是一次for循
环，所以for循环里，每次在do_something之后都会调用print。

Linux内核用宏封装了如上特性，include cleanup.h后就可以直接使用。比如，我们可以把
kernel/sched/core.c的sched_getaffinity中原来保护cpumask_and()的锁的写法改成：
```
	guard(raw_spinlock_irqsave)(&p->pi_lock);
	cpumask_and(mask, &p->cpus_mask, cpu_active_mask);
```

原理分析
---------

内核在include/linux/cleanup.h定义各种宏来封装cleanup属性，在各个头文件中，比如锁
的头文件中，使用cleanup.h中的宏定义class类型。所以，具体使用的时候，只要在文件里
include对应资源的头文件和cleanup.h就好。在需要使用cleanup的地方就可以直接用宏静态
定义对应class类型的对象变量。

直接看cleanup.h中的宏会有点不直观。我们可以在编译内核或者驱动的时候加上V=1，这样
会打印出详细的编译命令，然后我们把对应的编译命令改成只做预编译，再次单独运行对应
的编译命令，得到的预编译后的文件看上去会比较直观。

我们以一个简单的驱动编译为例看下，这个驱动的具体内容如下：
```
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/cleanup.h>

MODULE_LICENSE("Dual BSD/GPL");

static int __init busyloop_init(void)
{
	spinlock_t lock;
	int tmp;

	spin_lock_init(&lock);

	guard(spinlock_irq)(&lock);
	while (1) {
		tmp++;
	}

	return 0;
}

static void __exit busyloop_exit(void) {}

module_init(busyloop_init);
module_exit(busyloop_exit);

MODULE_AUTHOR("Sherlock");
MODULE_DESCRIPTION("The driver is for testing soft lockup");
```

加V=1编译的输出如下：
```
sherlock@m1:~/tests/softlockup$ make V=1 -C ~/repos/linux M=`pwd` modules
make: Entering directory '/home/sherlock/repos/linux'
make --no-print-directory -C /home/sherlock/repos/linux \
-f /home/sherlock/repos/linux/Makefile modules
make -f ./scripts/Makefile.build obj=/home/sherlock/tests/softlockup need-builtin=1 need-modorder=1 
# CC [M]  /home/sherlock/tests/softlockup/busyloop.o
  gcc -Wp,-MMD,/home/sherlock/tests/softlockup/.busyloop.o.d  -nostdinc -I./arch/arm64/include -I./arch/arm64/include/generated  -I./include -I./arch/arm64/include/uapi -I./arch/arm64/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -mlittle-endian -DKASAN_SHADOW_SCALE_SHIFT= -fmacro-prefix-map=./= -std=gnu11 -fshort-wchar -funsigned-char -fno-common -fno-PIE -fno-strict-aliasing-mgeneral-regs-only -DCONFIG_CC_HAS_K_CONSTRAINT=1 -Wno-psabi -mabi=lp64 -fno-asynchronous-unwind-tables -fno-unwind-tables -mbranch-protection=pac-ret -Wa,-march=armv8.5-a -DARM64_ASM_ARCH='"armv8.5-a"' -DKASAN_SHADOW_SCALE_SHIFT= -fno-delete-null-pointer-checks -O2 --param=allow-store-data-races=0 -fstack-protector-strong -fno-omit-frame-pointer -fno-optimize-sibling-calls -fno-stack-clash-protection -falign-functions=4 -fno-strict-overflow -fno-stack-check -fconserve-stack -Wall -Wundef -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Werror=strict-prototypes -Wno-format-security -Wno-trigraphs -Wno-frame-address -Wno-address-of-packed-member -Wmissing-declarations -Wmissing-prototypes -Wframe-larger-than=2048 -Wno-main -Wvla -Wno-pointer-sign -Wcast-function-type -Wno-stringop-overflow -Wno-array-bounds -Wno-alloc-size-larger-than -Wimplicit-fallthrough=5 -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -Wextra -Wunused -Wno-unused-but-set-variable -Wno-unused-const-variable -Wno-packed-not-aligned -Wno-format-overflow -Wno-format-truncation -Wno-stringop-truncation -Wno-override-init -Wno-missing-field-initializers -Wno-type-limits -Wno-shift-negative-value -Wno-maybe-uninitialized -Wno-sign-compare -Wno-unused-parameter -g -fno-var-tracking -femit-struct-debug-baseonly -mstack-protector-guard=sysreg -mstack-protector-guard-reg=sp_el0 -mstack-protector-guard-offset=1224  -DMODULE  -DKBUILD_BASENAME='"busyloop"' -DKBUILD_MODNAME='"busyloop"' -D__KBUILD_MODNAME=kmod_busyloop -c -o /home/sherlock/tests/softlockup/busyloop.o /home/sherlock/tests/softlockup/busyloop.c  
# cmd_gen_order /home/sherlock/tests/softlockup/modules.order
  {   echo /home/sherlock/tests/softlockup/busyloop.o; :; } > /home/sherlock/tests/softlockup/modules.order
sh ./scripts/modules-check.sh /home/sherlock/tests/softlockup/modules.order
make -f ./scripts/Makefile.modpost
# MODPOST /home/sherlock/tests/softlockup/Module.symvers
   scripts/mod/modpost -M        -o /home/sherlock/tests/softlockup/Module.symvers -T /home/sherlock/tests/softlockup/modules.order -i Module.symvers -e 
make -f ./scripts/Makefile.modfinal
# CC [M]  /home/sherlock/tests/softlockup/busyloop.mod.o gcc -Wp,-MMD,/home/sherlock/tests/softlockup/.busyloop.mod.o.d -nostdinc -I./arch/arm64/include -I./arch/arm64/include/generated -I./include -I./arch/arm64/include/uapi -I./arch/arm64/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -mlittle-endian -DKASAN_SHADOW_SCALE_SHIFT= -fmacro-prefix-map=./= -std=gnu11 -fshort-wchar -funsigned-char -fno-common -fno-PIE -fno-strict-aliasing -mgeneral-regs-only -DCONFIG_CC_HAS_K_CONSTRAINT=1 -Wno-psabi -mabi=lp64 -fno-asynchronous-unwind-tables -fno-unwind-tables -mbranch-protection=pac-ret -Wa,-march=armv8.5-a -DARM64_ASM_ARCH='"armv8.5-a"' -DKASAN_SHADOW_SCALE_SHIFT= -fno-delete-null-pointer-checks -O2 --param=allow-store-data-races=0 -fstack-protector-strong -fno-omit-frame-pointer -fno-optimize-sibling-calls -fno-stack-clash-protection -falign-functions=4 -fno-strict-overflow -fno-stack-check -fconserve-stack -Wall -Wundef -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Werror=strict-prototypes -Wno-format-security -Wno-trigraphs -Wno-frame-address -Wno-address-of-packed-member -Wmissing-declarations -Wmissing-prototypes -Wframe-larger-than=2048 -Wno-main -Wvla -Wno-pointer-sign -Wcast-function-type -Wno-stringop-overflow -Wno-array-bounds -Wno-alloc-size-larger-than -Wimplicit-fallthrough=5 -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -Wextra -Wunused -Wno-unused-but-set-variable -Wno-unused-const-variable -Wno-packed-not-aligned -Wno-format-overflow -Wno-format-truncation -Wno-stringop-truncation -Wno-override-init -Wno-missing-field-initializers -Wno-type-limits -Wno-shift-negative-value -Wno-maybe-uninitialized -Wno-sign-compare -Wno-unused-parameter -g -fno-var-tracking -femit-struct-debug-baseonly -mstack-protector-guard=sysreg -mstack-protector-guard-reg=sp_el0 -mstack-protector-guard-offset=1224 -DMODULE -DKBUILD_BASENAME='"busyloop.mod"' -DKBUILD_MODNAME='"busyloop"' -D__KBUILD_MODNAME=kmod_busyloop -c -o /home/sherlock/tests/softlockup/busyloop.mod.o /home/sherlock/tests/softlockup/busyloop.mod.c
# LD [M]  /home/sherlock/tests/softlockup/busyloop.ko
  ld -r  -EL  -maarch64elf -z noexecstack   --build-id=sha1  -T scripts/module.lds -o /home/sherlock/tests/softlockup/busyloop.ko /home/sherlock/tests/softlockup/busyloop.o /home
/sherlock/tests/softlockup/busyloop.mod.o
make: Leaving directory '/home/sherlock/repos/linux'
```

我们进入内核源码路径，GCC的参数加上-E, 重新指定输出文件:
```
  gcc -E -Wp,-MMD,/home/sherlock/tests/softlockup/.busyloop.o.d  -nostdinc -I./arch/arm64/include -I./arch/arm64/include/generated  -I./include -I./arch/arm64/include/uapi -I./arch/arm64/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ -mlittle-endian -DKASAN_SHADOW_SCALE_SHIFT= -fmacro-prefix-map=./= -std=gnu11 -fshort-wchar -funsigned-char -fno-common -fno-PIE -fno-strict-aliasing-mgeneral-regs-only -DCONFIG_CC_HAS_K_CONSTRAINT=1 -Wno-psabi -mabi=lp64 -fno-asynchronous-unwind-tables -fno-unwind-tables -mbranch-protection=pac-ret -Wa,-march=armv8.5-a -DARM64_ASM_ARCH='"armv8.5-a"' -DKASAN_SHADOW_SCALE_SHIFT= -fno-delete-null-pointer-checks -O2 --param=allow-store-data-races=0 -fstack-protector-strong -fno-omit-frame-pointer -fno-optimize-sibling-calls -fno-stack-clash-protection -falign-functions=4 -fno-strict-overflow -fno-stack-check -fconserve-stack -Wall -Wundef -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Werror=strict-prototypes -Wno-format-security -Wno-trigraphs -Wno-frame-address -Wno-address-of-packed-member -Wmissing-declarations -Wmissing-prototypes -Wframe-larger-than=2048 -Wno-main -Wvla -Wno-pointer-sign -Wcast-function-type -Wno-stringop-overflow -Wno-array-bounds -Wno-alloc-size-larger-than -Wimplicit-fallthrough=5 -Werror=date-time -Werror=incompatible-pointer-types -Werror=designated-init -Wextra -Wunused -Wno-unused-but-set-variable -Wno-unused-const-variable -Wno-packed-not-aligned -Wno-format-overflow -Wno-format-truncation -Wno-stringop-truncation -Wno-override-init -Wno-missing-field-initializers -Wno-type-limits -Wno-shift-negative-value -Wno-maybe-uninitialized -Wno-sign-compare -Wno-unused-parameter -g -fno-var-tracking -femit-struct-debug-baseonly -mstack-protector-guard=sysreg -mstack-protector-guard-reg=sp_el0 -mstack-protector-guard-offset=1224  -DMODULE  -DKBUILD_BASENAME='"busyloop"' -DKBUILD_MODNAME='"busyloop"' -D__KBUILD_MODNAME=kmod_busyloop -c -o /home/sherlock/tests/softlockup/busyloop.i /home/sherlock/tests/softlockup/busyloop.c  
```
可以看到在busyloop.i中展开guard()的结果:
```
[...]

typedef struct { spinlock_t *lock; ; } class_spinlock_irq_t; static inline __attribute__((__gnu_inline__)) __attribute__((__unused__)) __attribute__((__no_instrument_function__)) void class_spinlock_irq_destructor(class_spinlock_irq_t *_T) { if (_T->lock) { spin_unlock_irq(_T->lock); } } static inline __attribute__((__gnu_inline__)) __attribute__((__unused__)) __attribute__((__no_instrument_function__)) void *class_spinlock_irq_lock_ptr(class_spinlock_irq_t *_T) { return _T->lock; } static inline __attribute__((__gnu_inline__)) __attribute__((__unused__)) __attribute__((__no_instrument_function__)) class_spinlock_irq_t class_spinlock_irq_constructor(spinlock_t *l) { class_spinlock_irq_t _t = { .lock = l }, *_T = &_t; spin_lock_irq(_T->lock); return _t; }

[...]

static int __attribute__((__section__(".init.text"))) busyloop_init(void)
{
 spinlock_t lock;
 int tmp;

 do { spinlock_check(&lock); *(&lock) = (spinlock_t) { { .rlock = { .raw_lock = { { .val = { (0) } } }, } } }; } while (0);

 class_spinlock_irq_t __UNIQUE_ID_guard339 __attribute__((__cleanup__(class_spinlock_irq_destructor))) = class_spinlock_irq_constructor(&lock);
 while (1) {
  tmp++;
 }

	return 0;
}

[...]
```
从上面的宏展开可以看到，guard静态定义了一个类型为class_spinlock_irq_t的匿名对象，
在初始化函数里把锁指针传入对象中，并调用下加锁函数。在cleanup函数中获得锁指针，并
在cleanup函数中调用释放锁的函数。因为匿名对象的生命周期是busyloop_init这个函数，
所以在这个函数结束的时候会调用cleanup函数。

如果busyloop_init变成了如下，只想对tmp和tmp1上锁，那么就需要使用scoped_guard宏，
所示如下：
```
static int __init busyloop_init(void)
{
	spinlock_t lock;
	spinlock_t s_lock;
	int tmp, tmp1, tmp2;

	spin_lock_init(&lock);

	guard(spinlock_irq)(&lock);
	while (1) {
		scoped_guard(spinlock_irq, &s_lock) {
			tmp++;
			tmp1++;
		}
		tmp2++;
	}

	return 0;
}
```
宏展开是这样的：
```
static int __attribute__((__section__(".init.text"))) busyloop_init(void)
{
 spinlock_t lock;
 spinlock_t s_lock;
 int tmp, tmp1, tmp2;

 do { spinlock_check(&lock); *(&lock) = (spinlock_t) { { .rlock = { .raw_lock = { { .val = { (0) } } }, } } }; } while (0);

 class_spinlock_irq_t __UNIQUE_ID_guard339 __attribute__((__cleanup__(class_spinlock_irq_destructor))) = class_spinlock_irq_constructor(&lock);
 while (1) {
  for (class_spinlock_irq_t scope __attribute__((__cleanup__(class_spinlock_irq_destructor))) = class_spinlock_irq_constructor(&s_lock), *done = ((void *)0); class_spinlock_irq_lock_ptr(&scope) && !done; done = (void *)1) {
   tmp++;
   tmp1++;
  }
  tmp2++;
 }

	return 0;
}
```
如果scoped_guard这里换成guard，会变成在每次while循环中对tmp/tmp1/tmp2加锁和释放锁。
