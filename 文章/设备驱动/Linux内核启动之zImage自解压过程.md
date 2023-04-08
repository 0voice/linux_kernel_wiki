## **1、zImage来历**

顶层vmlinux ---->

arch/arm/boot/Image --->

arch/arm/boot/compressed/piggy.gz --->

arch/arm/boot/compressed/vmlinux --->

arch/arm/boot/**zImage**

如果要分析zImage的反汇编反汇编文件，可将arch/arm/boot/compressed/vmlinux进行反汇编，

arm-linux-xxx-objdump –d vmlinux > vmlinux.dis

对顶层的vmlinux反汇编得到的是未压缩的内核的反汇编文件，这个vmlinux才是真正的Linux内核。

## 2、**piggy.gz压缩文件的特点**

**gzip -f -9 < Image > piggy.gz**

在piggy.gz的结尾四个字节表示的是 Image 镜像的大小，并且是以小端格式存放的。下面我们验证一下：

![img](https://pic1.zhimg.com/80/v2-3668c9d91b4e32ac2f7e76e2325fa340_720w.webp)

可以看到，Image的大小是6806148B，十六进制值就是67DA84，接下来看看piggy.gz的结尾：

![img](https://pic3.zhimg.com/80/v2-4c4d80504e931db15b4c772ea4a8a83a_720w.webp)

![img](https://pic4.zhimg.com/80/v2-195629b07facdc2f50d041e0c082a02b_720w.webp)

可以看到，确实是将0x67DA84以小端的格式存放在了piggy.gz的结尾四字节中了。

**vmlinux.lds**

```text
1:  /*
   2:   *  linux/arch/arm/boot/compressed/vmlinux.lds.in
   3:   *
   4:   *  Copyright (C) 2000 Russell King
   5:   *
   6:   * This program is free software; you can redistribute it and/or modify
   7:   * it under the terms of the GNU General Public License version 2 as
   8:   * published by the Free Software Foundation.
   9:   */
  10:  OUTPUT_ARCH(arm)
  11:  ENTRY(_start)
  12:  SECTIONS
  13:  {
  14:    /DISCARD/ : {
  15:      *(.ARM.exidx*)
  16:      *(.ARM.extab*)
  17:      /*
  18:       * Discard any r/w data - this produces a link error if we have any,
  19:       * which is required for PIC decompression.  Local data generates
  20:       * GOTOFF relocations, which prevents it being relocated independently
  21:       * of the text/got segments.
  22:       */
  23:      *(.data)
  24:    }
  25:   
  26:    . = 0;
  27:    _text = .;
  28:   
  29:    .text : {
  30:      _start = .;
  31:      *(.start)
  32:      *(.text)
  33:      *(.text.*)
  34:      *(.fixup)
  35:      *(.gnu.warning)
  36:      *(.rodata)
  37:      *(.rodata.*)
  38:      *(.glue_7)
  39:      *(.glue_7t)
  40:      *(.piggydata)
  41:      . = ALIGN(4);
  42:    }
  43:   
  44:    _etext = .;
  45:   
  46:    _got_start = .;
  47:    .got            : { *(.got) }
  48:    _got_end = .;
  49:    .got.plt        : { *(.got.plt) }
  50:    _edata = .;
  51:   
  52:    . = ALIGN(8);
  53:    __bss_start = .;
  54:    .bss            : { *(.bss) }
  55:    _end = .;
  56:   
  57:    . = ALIGN(8);        /* the stack must be 64-bit aligned */
  58:    .stack        : { *(.stack) }
  59:   
  60:    .stab 0        : { *(.stab) }
  61:    .stabstr 0        : { *(.stabstr) }
  62:    .stab.excl 0        : { *(.stab.excl) }
  63:    .stab.exclstr 0    : { *(.stab.exclstr) }
  64:    .stab.index 0        : { *(.stab.index) }
  65:    .stab.indexstr 0    : { *(.stab.indexstr) }
  66:    .comment 0        : { *(.comment) }
  67:  }
  68:  
```

**arch/arm/boot/compressed/head.S**

```text
1:  .section ".start", #alloc, #execinstr
   2:  /*
   3:  * 清理不同的调用约定
   4:  */
   5:  .align
   6:  .arm @ 启动总是进入ARM状态
   7:  start:
   8:  .type start,#function
   9:  .rept 7
  10:  mov r0, r0
  11:  .endr
  12:  ARM( mov r0, r0 )
  13:  ARM( b 1f )
  14:  THUMB( adr r12, BSYM(1f) )
  15:  THUMB( bx r12 )
  16:  .word 0x016f2818 @ 用于boot loader的魔数
  17:  .word start @ 加载/运行zImage的绝对地址（编译时确定）, 在vmlinux.lds中可以看到，zImage的链接起始地址是0
  18:  .word _edata @ zImage结束地址,分析vmlinux.lds可以看到，_edata是 .got 段的结束地址，后面紧接的就是.bss段和.stack段
  19:  THUMB( .thumb )
  20:  1: mov r7, r1 @ 保存构架ID到r7（此前由bootloader放入r1）
  21:     mov r8, r2 @ 保存内核启动参数地址到r8（此前由bootloader放入r2）
  22:  #ifndef __ARM_ARCH_2__
  23:  /*
  24:  * 通过Angel调试器启动 - 必须进入 SVC模式且关闭FIQs/IRQs
  25:  * (numeric definitions from angel arm.h source).
  26:  * 如果进入时在user模式下，我们只需要做这些
  27:  */
  28:  mrs r2, cpsr @ 获取当前模式
  29:  tst r2, #3 @ 判断是否是user模式
  30:  bne not_angel
  31:  mov r0, #0x17 @ angel_SWIreason_EnterSVC
  32:  ARM( swi 0x123456 ) @ angel_SWI_ARM  swi会产生软中断，会跳入中断向量表，这个向量表用的是bootloader的，因为在head.S中并没有建立新的向量表
  33:  THUMB( svc 0xab ) @ angel_SWI_THUMB
  34:  not_angel:
  35:  mrs r2, cpsr @ 关闭中断
  36:  orr r2, r2, #0xc0 @ 以保护调试器的运作 关闭IRQ和FIQ
  37:  msr cpsr_c, r2
  38:  #else
  39:  teqp pc, #0x0c000003@ 关闭中断（此外bootloader已设置模式为SVC）
  40:  #endif
 

GOT表是什么？

GOT（Global Offset Table）表中每一项都是本运行模块要引用的一个全局变量或函数的地址。可以用GOT表来间接引用全局变量、函数，也可以把GOT表的首地址作为一个基准，用相对于该基准的偏移量来引用静态变量、静态函数。

更多介绍请参考：

http://blog.sina.com.cn/s/blog_54f82cc201011oqy.html

http://blog.sina.com.cn/s/blog_54f82cc201011oqv.html

接着分析head.S

   1:  /*
   2:  * 注意一些缓存的刷新和其他事务可能需要在这里完成
   3:  * - is there an Angel SWI call for this?
   4:  */
   5:  /*
   6:  * 一些构架的特定代码可以在这里被连接器插入，
   7:  * 但是不应使用 r7（保存构架ID）, r8（保存内核启动参数地址）, and r9.
   8:  */
   9:  .text
  10:  /*
  11:  * 此处确定解压后的内核映像的绝对地址（物理地址），保存于r4
  12:  * 由于配置的不同可能有的结果
  13:  * （1）定义了CONFIG_AUTO_ZRELADDR
  14:  *      ZRELADDR是已解压内核最终存放的物理地址
  15:  *      如果AUTO_ZRELADDR被选择了, 这个地址将会在运行是确定：
  16:  *      将当pc值和0xf8000000做与操作，
  17:  *      并加上TEXT_OFFSET（内核最终存放的物理地址与内存起始的偏移）
  18:  *      这里假定zImage被放在内存开始的128MB内
  19:  * （2）没有定义CONFIG_AUTO_ZRELADDR
  20:  *      直接使用zreladdr（此值位于arch/arm/mach-xxx/Makefile.boot文件确定）
  21:  */
  22:  #ifdef CONFIG_AUTO_ZRELADDR
  23:  @ 确定内核映像地址
  24:  mov r4, pc
  25:  and r4, r4, #0xf8000000
  26:  add r4, r4, #TEXT_OFFSET
  27:  #else
  28:  ldr r4, =zreladdr                             以我的板子为例，它的值是0x80008000,即它的物理内存起始地址是0x80000000
  29:  #endif
  30:  bl cache_on /* 开启缓存（以及MMU） */         关于cache_on这部分，我在这里就不分析了
  31:  restart: adr r0, LC0   获取LC0的运行地址，而不是链接地址
  32:  ldmia r0, {r1, r2, r3, r6, r10, r11, r12}
       /*
            依次将LC0的链接地址放入r1中，bss段的链接起始和结束地址__bss_start和_end分别放入r2和r3中
            _edata的链接地址放入r6中，piggy.gz的倒数第四个字节的地址放入r10中， got表的起始和结束链接地址分别放入r11和r12中
      */
33: ldr sp, [r0, #28]   此时r0中存放的还是LC0的运行地址，所以加28后正好是LC0数组中的.L_user_stack_end的值(栈的结束地址)，他在head.S的结尾定义

reloc_code_end:

        .align
        .section ".stack", "aw", %nobits
.L_user_stack:    .space    4096
.L_user_stack_end:

  34:  /*
  35:  * 我们可能运行在一个与编译时定义的不同地址上，
  36:  * 所以我们必须修正变量指针
  37:  */
  38:  sub r0, r0, r1 @ 计算偏移量                              编译地址与运行地址的差值，以此来修正其他符号的地址
  39:  add r6, r6, r0 @ 重新计算_edata                          获得_edata的实际运行地址
  40:  add r10, r10, r0 @ 重新获得压缩后的内核大小数据位置      获取Image大小存放的实际物理地址    
  41:  /*
  42:  * 内核编译系统将解压后的内核大小数据
  43:  * 以小端格式
  44:  * 附加在压缩数据的后面(其实是“gzip -f -9”命令的结果)
  45:  * 下面代码的作用是将解压后的内核大小数据正确地放入r9中（避免了大小端问题）
  46:  */
  47:  ldrb r9, [r10, #0]      以我们的例子，此时r9为0x84
  48:  ldrb lr, [r10, #1]                    lr为 0xDA
  49:  orr r9, r9, lr, lsl #8                r9为0xDA84
  50:  ldrb lr, [r10, #2]                    lr为0x67
  51:  ldrb r10, [r10, #3]                   r10是0x00
  52:  orr r9, r9, lr, lsl #16               r9为0x67DA84
  53:  orr r9, r9, r10, lsl #24              r9是0x0067DA84
  54:  /*
  55:  * 下面代码的作用是将正确的当前执行映像的结束地址放入r10
  56:  */
  57:  #ifndef CONFIG_ZBOOT_ROM
  58:  /* malloc 获取的内存空间位于重定向的栈指针之上 (64k max) */
  59:  add sp, sp, r0           使用r0修正sp，得到堆栈的实际结束物理地址，为什么是结束地址？因为栈是向下生长的
  60:  add r10, sp, #0x10000    执行这句之前sp中存放的是栈的结束地址，执行后，r10中存放的是堆空间的结束物理地址
  61:  #else
  62:  /*
  63:  * 如果定义了 ZBOOT_ROM， bss/stack 是非可重定位的,
  64:  * 但有些人依然可以将其放在RAM中运行,
  65:  * 这时我们可以参考 _edata.
  66:  */
  67:  mov r10, r6
  68:  #endif
  69:  /*
  70:  * 检测我们是否会发生自我覆盖的问题
  71:  * r4 = 解压后的内核起始地址（最终执行位置）   在我们的例子中r4就是0x80008000
  72:  * r9 = 解压后内核的大小                       即arch/arm/boot/Image的大小，0x67DA84
  73:  * r10 = 当前执行映像的结束地址, 包含了 bss/stack/malloc 空间（假设是非XIP执行的）, 上面已经分析了r10存放的是堆空间的结束地址
  74:  * 我们的基本需求是:
  75:  * （若最终执行位置r4在当前映像之后）r4 - 16k 页目录 >= r10 -> OK
  76:  * （若最终执行位置r4在当前映像之前）r4 + 解压后的内核大小 <= 当前位置 (pc) -> OK
  77:  *  如果上面的条件不满足，就会自我覆盖，必须先搬运当前映像
  78:  */
  79:  add r10, r10, #16384
  80:  cmp r4, r10         @ 假设最终执行位置r4在当前映像之后
  81:  bhs wont_overwrite
  82:  add r10, r4, r9     @ 假设最终执行位置r4在当前映像之前
  83:  ARM( cmp r10, pc )  @ r10 = 解压后的内核结束地址，注：这里的r4+r9计算出的大小并不包含栈和堆空间
  84:  THUMB( mov lr, pc )
  85:  THUMB( cmp r10, lr )
  86:  bls wont_overwrite
```

**下面是LC0区域的数据内容：**

```text
 1:          .align    2
   2:          .type    LC0, #object
   3:  LC0:        .word    LC0            @ r1
   4:          .word    __bss_start        @ r2
   5:          .word    _end     @ r3
   6:          .word    _edata             @ r6
   7:          .word    input_data_end - 4    @ r10 (inflated size location)
   8:          .word    _got_start        @ r11
   9:          .word    _got_end    @ ip
  10:          .word    .L_user_stack_end    @ sp
  11:          .size    LC0, . - LC0
```

还记得piggy.gz的结尾四个字节的含义吗？他表示的是arch/arm/boot/Image的大小，在LC0区域中：

.word input_data_end - 4 @ r10 (inflated size location)

的意思：

input_data_end 在piggy.gzip.S中定义：

```text
   1:      .section .piggydata,#alloc
   2:      .globl    input_data
   3:  input_data:
   4:      .incbin    "arch/arm/boot/compressed/piggy.gzip"
   5:      .globl    input_data_end
   6:  input_data_end:
```

所以，"input_data_end – 4" 表示的是piggy.gz的倒数第四个字节的地址，也是为了便于获得Image的大小。

接着分析head.S:

```text
   1:  /*
   2:  * 将当前的映像重定向到解压后的内核之后（会发生自我覆盖时才执行，否则就被跳过）
   3:  * r6 = _edata（已校正）
   4:  * r10 = 解压后的内核结束地址   对于我的例子: r10就是  0x80008000 + 0x67DA84(arch/arm/boot/Image的大小)
          这样可以保证一次重定位就可以达到预期目标
   5:  * 因为我们要把当前映像向后移动, 所以我们必须由后往前复制代码，
   6:  * 以防原数据和目标数据的重叠
   7:  */
   8:  /*
   9:  * 将解压后的内核结束地址r10扩展（reloc_code_end - restart），
  10:  * 并对齐到下一个256B边界。
  11:  * 这样避免了当搬运的偏移较小时的自我覆盖
  12:  */
  13:      add r10, r10, #((reloc_code_end - restart + 256) & ~255) 
  14:      bic r10, r10, #255
  15:  /* 获取需要搬运的当前映像的起始位置r5，并向下做32B对齐. */
  16:      adr r5, restart
  17:      bic r5, r5, #31
  18:      sub r9, r6, r5  @ _edata - restart（已向下对齐）= 需要搬运的大小
  19:      add r9, r9, #31
  20:      bic r9, r9, #31 @ 做32B对齐 ，r9 = 需要搬运的大小
  21:      add r6, r9, r5  @ r6 = 当前映像需要搬运的结束地址，由于r5和r9都已经对齐，所以r6也对齐了。
          /* 上面对齐以后，有利于后面的代码拷贝，而且对齐的同时又保证了拷贝代码的完整性，至少没有少拷贝代码*/
  22:      add r9, r9, r10 @ r9 = 当前映像搬运的目的地的结束地址
  23:  /* 搬运当前执行映像，不包含 bss/stack/malloc 空间*/
  24:      1: ldmdb r6!, {r0 - r3, r10 - r12, lr}
  25:      cmp r6, r5
  26:      stmdb r9!, {r0 - r3, r10 - r12, lr}
  27:      bhi 1b
  28:  /* 保存偏移量，用来修改sp和实现代码跳转 */
  29:      sub r6, r9, r6            求要搬移的代码的原始地址与目的地址的偏移量
  30:      #ifndef CONFIG_ZBOOT_ROM
  31:  /* cache_clean_flush 可能会使用栈，所以重定向sp指针 */
  32:      add sp, sp, r6            修正sp地址，修正后sp指向重定向后的代码的栈区的结束地址（栈向下生长），栈区后面紧跟的就是堆空间
  33:      #endif
  34:      bl cache_clean_flush @ 刷新缓存
  35:  /* 通过搬运的偏移和当前的实际 restart 地址来实现代码跳转*/
  36:      adr r0, BSYM(restart)    获得restart的运行地址
  37:      add r0, r0, r6           获得重定向后的代码的restart的物理地址
  38:      mov pc, r0               跳到重定向后的代码的restart处开始执行
  39:  /* 在上面的跳转之后，程序又从restart开始。
  40:  * 但这次在检查自我覆盖的时候，新的执行位置必然满足
  41:  * 最终执行位置r4在当前映像之前，r4 + 压缩后的内核大小 <= 当前位置 (pc)
  42:  * 所以必然直接跳到了下面的wont_overwrite执行
  43:  */
  44:  wont_overwrite:
  45:  /*
  46:  * 如果delta（当前映像地址与编译时的地址偏移）为0, 我们运行的地址就是编译时确定的地址.
  47:  * r0 = delta
  48:  * r2 = BSS start（编译值）
  49:  * r3 = BSS end（编译值）
  50:  * r4 = 内核最终运行的物理地址
  51:  * r7 = 构架ID(bootlodaer传递值)
  52:  * r8 = 内核启动参数指针(bootlodaer传递值)
  53:  * r11 = GOT start（编译值）
  54:  * r12 = GOT end（编译值）
  55:  * sp = stack pointer（修正值）
  56:  */
  57:      teq r0, #0 @测试delta值
  58:      beq not_relocated @如果delta为0，无须对GOT表项和BSS进行重定位
  59:      add r11, r11, r0 @重定位GOT start
  60:      add r12, r12, r0 @重定位GOT end
  61:      #ifndef CONFIG_ZBOOT_ROM
  62:  /*
  63:  * 如果内核配置 CONFIG_ZBOOT_ROM = n,
  64:  * 我们必须修正BSS段的指针
  65:  * 注意：sp已经被修正
  66:  */
  67:      add r2, r2, r0 @重定位BSS start
  68:      add r3, r3, r0 @重定位BSS end
  69:  /*    
  70:  * 重定位所有GOT表的入口项
  71:  */
  72:      1: ldr r1, [r11, #0] @ 重定位GOT表的入口项
  73:      add r1, r1, r0 @ 这个修正了 C 引用
  74:      str r1, [r11], #4
  75:      cmp r11, r12
  76:      blo 1b
  77:      #else
  78:  /*
  79:  * 重定位所有GOT表的入口项.
  80:  * 我们只重定向在（已重定向后）BSS段外的入口
  81:  */
  82:      1: ldr r1, [r11, #0] @ 重定位GOT表的入口项
  83:      cmp r1, r2 @ entry < bss_start ||
  84:      cmphs r3, r1 @ _end < entry table
  85:      addlo r1, r1, r0 @ 这个修正了 C 引用
  86:      str r1, [r11], #4
  87:      cmp r11, r12
  88:      blo 1b
  89:      #endif
  90:  /*
  91:  * 至此当前映像的搬运和调整已经完成
  92:  * 可以开始真正的工作的
  93:  */
  94:      not_relocated: mov r0, #0
  95:      1: str r0, [r2], #4 @ 清零 bss（初始化BSS段）
  96:      str r0, [r2], #4
  97:      str r0, [r2], #4
  98:      str r0, [r2], #4
  99:      cmp r2, r3
 100:      blo 1b
 101:  /*
 102:  * C运行时环境已经充分建立.
 103:  * 设置一些指针就可以解压内核了.
 104:  * r4 = 内核最终运行的物理地址
 105:  * r7 = 构架ID
 106:  * r8 = 内核启动参数指针
 107:  *
 108:  * 下面对r0～r3的配置是decompress_kernel函数对应参数
 109:  * r0 = 解压后的输出位置首地址
 110:  * r1 = 可用RAM空间首地址  即堆空间的起始地址
 111:  * r2 = 可用RAM空间结束地址  即堆空间的结束地址
 112:  * r3 = 构架ID
 113:  * 就是这个decompress_kernel（C函数）输出了"Uncompressing Linux..."
 114:  * 以及" done, booting the kernel.\n"
 115:  */
 116:      mov r0, r4
 117:      mov r1, sp @ malloc 获取的内存空间位于栈指针之上
 118:      add r2, sp, #0x10000 @ 64k max
 119:      mov r3, r7
 120:      bl decompress_kernel
 121:  /*
 122:  * decompress_kernel(misc.c)--调用-->
 123:  * do_decompress(decompress.c)--调用-->
 124:  * decompress(../../../../lib/decompress_xxxx.c根据压缩方式的配置而不同)
 125:  */
 126:  /*
 127:  * 以下是为跳入解压后的内核，再次做准备（恢复解压前的状态）
 128:  */
 129:      bl cache_clean_flush
 130:      bl cache_off@ 数据缓存必须关闭（内核的要求）
 131:      mov r0, #0 @ r0必须为0
 132:      mov r1, r7@ 恢复构架ID到r1
 133:      mov r2, r8 @ 恢复内核启动参数指针到r2
 134:      mov pc, r4 @ 跳入解压后的内核映像(Image)入口（arch/arm/kernel/head.S）
```

下面简要看一下解压缩程序的调用过程：

## 3、**arch/arm/boot/compressed/misc.c**

```text
 1:  void
   2:  decompress_kernel(unsigned long output_start, unsigned long free_mem_ptr_p,
   3:          unsigned long free_mem_ptr_end_p,
   4:          int arch_id)
   5:  {
   6:      int ret;
   7:   
   8:      output_data        = (unsigned char *)output_start;    /*解压地址*/
   9:      free_mem_ptr        = free_mem_ptr_p;               /*堆栈起始地址*/
  10:      free_mem_end_ptr    = free_mem_ptr_end_p;           /*堆栈结束地址*/
  11:      __machine_arch_type    = arch_id;                      /*uboot传入的machid*/
  12:   
  13:      arch_decomp_setup();
  14:   
  15:      putstr("Uncompressing Linux...");
  16:      ret = do_decompress(input_data, input_data_end - input_data,
  17:                  output_data, error);
  18:      if (ret)
  19:          error("decompressor returned an error");
  20:      else
  21:          putstr(" done, booting the kernel.\n");
  22:  }
```

**arch/arm/boot/compressed/decompressed.c**

```text
 1:  #  define Tracec(c,x)
   2:  #  define Tracecv(c,x)
   3:  #endif
   4:   
   5:  #ifdef CONFIG_KERNEL_GZIP
   6:  #include "../../../../lib/decompress_inflate.c"
   7:  #endif
   8:   
   9:  #ifdef CONFIG_KERNEL_LZO
  10:  #include "../../../../lib/decompress_unlzo.c"
  11:  #endif
  12:   
  13:  #ifdef CONFIG_KERNEL_LZMA
  14:  #include "../../../../lib/decompress_unlzma.c"
  15:  #endif
  16:   
  17:  int 
do_decompress
(u8 *input, int len, u8 *output, void (*error)(char *x))
  18:  {
  19:      return 
decompress
(input, len, NULL, NULL, output, NULL, error);
  20:  }
```

**lib/decompress_inflate.c**

```text
 1:  #ifdef STATIC
   2:  /* Pre-boot environment: included */
   3:   
   4:  /* prevent inclusion of _LINUX_KERNEL_H in pre-boot environment: lots
   5:   * errors about console_printk etc... on ARM */
   6:  #define _LINUX_KERNEL_H
   7:   
   8:  #include "zlib_inflate/inftrees.c"
   9:  #include "zlib_inflate/inffast.c"
  10:  #include "zlib_inflate/inflate.c"
  11:   
  12:  #else /* STATIC */
  13:  /* initramfs et al: linked */
  14:   
  15:  #include <linux/zutil.h>
  16:   
  17:  #include "zlib_inflate/inftrees.h"
  18:  #include "zlib_inflate/inffast.h"
  19:  #include "zlib_inflate/inflate.h"
  20:   
  21:  #include "zlib_inflate/infutil.h"
  22:   
  23:  #endif /* STATIC */
  24:   
  25:  #include <linux/decompress/mm.h>
  26:   
  27:  #define GZIP_IOBUF_SIZE (16*1024)
  28:   
  29:  static int INIT nofill(void *buffer, unsigned int len)
  30:  {
  31:      return -1;
  32:  }
  33:   
  34:  /* Included from initramfs et al code */
  35:  STATIC int INIT gunzip(unsigned char *buf, int len,
  36:                 int(*fill)(void*, unsigned int),
  37:                 int(*flush)(void*, unsigned int),
  38:                 unsigned char *out_buf,
  39:                 int *pos,
  40:                 void(*error)(char *x)) {
  41:      u8 *zbuf;
  42:      struct z_stream_s *strm;
  43:      int rc;
  44:      size_t out_len;
  45:   
  46:      rc = -1;
  47:      if (flush) {
  48:          out_len = 0x8000; /* 32 K */
  49:          out_buf = malloc(out_len);
  50:      } else {
  51:          out_len = 0x7fffffff; /* no limit */
  52:      }
  53:      if (!out_buf) {
  54:          error("Out of memory while allocating output buffer");
  55:          goto gunzip_nomem1;
  56:      }
  57:   
  58:      if (buf)
  59:          zbuf = buf;
  60:      else {
  61:          zbuf = malloc(GZIP_IOBUF_SIZE);
  62:          len = 0;
  63:      }
  64:      if (!zbuf) {
  65:          error("Out of memory while allocating input buffer");
  66:          goto gunzip_nomem2;
  67:      }
  68:   
  69:      strm = malloc(sizeof(*strm));
  70:      if (strm == NULL) {
  71:          error("Out of memory while allocating z_stream");
  72:          goto gunzip_nomem3;
  73:      }
  74:   
  75:      strm->workspace = malloc(flush ? zlib_inflate_workspacesize() :
  76:                   sizeof(struct inflate_state));
  77:      if (strm->workspace == NULL) {
  78:          error("Out of memory while allocating workspace");
  79:          goto gunzip_nomem4;
  80:      }
  81:   
  82:      if (!fill)
  83:          fill = nofill;
  84:   
  85:      if (len == 0)
  86:          len = fill(zbuf, GZIP_IOBUF_SIZE);
  87:   
  88:      /* verify the gzip header */
  89:      if (len < 10 ||
  90:         zbuf[0] != 0x1f || zbuf[1] != 0x8b || zbuf[2] != 0x08) {
  91:          if (pos)
  92:              *pos = 0;
  93:          error("Not a gzip file");
  94:          goto gunzip_5;
  95:      }
  96:   
  97:      /* skip over gzip header (1f,8b,08... 10 bytes total +
  98:       * possible asciz filename)
  99:       */
 100:      strm->next_in = zbuf + 10;
 101:      strm->avail_in = len - 10;
 102:      /* skip over asciz filename */
 103:      if (zbuf[3] & 0x8) {
 104:          do {
 105:              /*
 106:               * If the filename doesn't fit into the buffer,
 107:               * the file is very probably corrupt. Don't try
 108:               * to read more data.
 109:               */
 110:              if (strm->avail_in == 0) {
 111:                  error("header error");
 112:                  goto gunzip_5;
 113:              }
 114:              --strm->avail_in;
 115:          } while (*strm->next_in++);
 116:      }
 117:   
 118:      strm->next_out = out_buf;
 119:      strm->avail_out = out_len;
 120:   
 121:      rc = zlib_inflateInit2(strm, -MAX_WBITS);
 122:   
 123:      if (!flush) {
 124:          WS(strm)->inflate_state.wsize = 0;
 125:          WS(strm)->inflate_state.window = NULL;
 126:      }
 127:   
 128:      while (rc == Z_OK) {
 129:          if (strm->avail_in == 0) {
 130:              /* TODO: handle case where both pos and fill are set */
 131:              len = fill(zbuf, GZIP_IOBUF_SIZE);
 132:              if (len < 0) {
 133:                  rc = -1;
 134:                  error("read error");
 135:                  break;
 136:              }
 137:              strm->next_in = zbuf;
 138:              strm->avail_in = len;
 139:          }
 140:          rc = zlib_inflate(strm, 0);
 141:   
 142:          /* Write any data generated */
 143:          if (flush && strm->next_out > out_buf) {
 144:              int l = strm->next_out - out_buf;
 145:              if (l != flush(out_buf, l)) {
 146:                  rc = -1;
 147:                  error("write error");
 148:                  break;
 149:              }
 150:              strm->next_out = out_buf;
 151:              strm->avail_out = out_len;
 152:          }
 153:   
 154:          /* after Z_FINISH, only Z_STREAM_END is "we unpacked it all" */
 155:          if (rc == Z_STREAM_END) {
 156:              rc = 0;
 157:              break;
 158:          } else if (rc != Z_OK) {
 159:              error("uncompression error");
 160:              rc = -1;
 161:          }
 162:      }
 163:   
 164:      zlib_inflateEnd(strm);
 165:      if (pos)
 166:          /* add + 8 to skip over trailer */
 167:          *pos = strm->next_in - zbuf+8;
 168:   
 169:  gunzip_5:
 170:      free(strm->workspace);
 171:  gunzip_nomem4:
 172:      free(strm);
 173:  gunzip_nomem3:
 174:      if (!buf)
 175:          free(zbuf);
 176:  gunzip_nomem2:
 177:      if (flush)
 178:          free(out_buf);
 179:  gunzip_nomem1:
 180:      return rc; /* returns Z_OK (0) if successful */
 181:  }
 182:   
 183:  #define decompress gunzip
```

为了便于理解，可以参考下面一张图：

![img](https://pic1.zhimg.com/80/v2-ae8dd36b0679b027dd0cf237541655fc_720w.webp)

----

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/548745216