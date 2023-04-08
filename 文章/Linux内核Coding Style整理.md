## **1、缩进了**

缩进用 Tab, 并且Tab的宽度为8个字符

swich 和 case对齐, 不用缩进

```text
switch (suffix) {
case 'G':
case 'g':
        mem <<= 30;
        break;
case 'M':
case 'm':
        mem <<= 20;
        break;
case 'K':
case 'k':
        mem <<= 10;
        /* fall through */
default:
        break;
}
```

一行只有一个表达式

```text
if (condition) do_this;  /* bad example */
```

不要用空格来缩进 (除了注释或文档)

## **2、代码行长度控制在80个字符以内**

长度过长的行截断时, 注意保持易读性

```text
void fun(int a, int b, int c)
{
        if (condition)
                printk(KERN_WARNING "Warning this is a long printk with "
                       "3 parameters a: %u b: %u "
                       "c: %u \n", a, b, c);
        else
                next_statement;
}
```

## **3、括号和空格的位置**

函数的大括号另起一行

```text
int function(int x)
{   /* 这个大括号 { 另起了一行 */
        body of function
}
```

非函数的语句块(if, switch, for, while, do)不用另起一行

```text
if (x is true) {  /* 这个大括号 { 不用另起一行 */
        we do y
}
```

只有一行的语句块不用大括号

```text
if (condition)
        action();
```

如果if用了大括号, 那么else即使只有一行也要用大括号

```text
if (condition) {
        do_this();
        do_that();
} else {
        otherwise();
}
```

下列 keywords 后面追加一个空格

if, switch, case, for, do, while

下列 keywords 后面 *不要* 追加一个空格

sizeof, typeof, alignof, __attribute

```text
if (condition) {
        do_this();
        do_that();
} else {
        otherwise();
}
```

定义指针时, * 紧靠函数名或者变量名

```text
if (condition) {
        do_this();
        do_that();
} else {
        otherwise();
}
```

下面的二元和三元操作符左右要留一个空格

= + - < > * / % | & ^ <= >= == != ? :

下面的一元操作符后面 *不要* 留空格

& * + - ~ ! sizeof typeof alignof __attribute__ defined

后缀操作符(++ --)前面不要留空格

前缀操作符(++ --)后面不要留空格

结构体成员操作符(. ->)前后都不要留空格

每行代码之后不要有多余的空格

## **4、命名**

全局变量或函数(在确实需要时才使用)要有个描述性的名称

```text
count_active_users()  /* good */
cntusr()              /* cnt  */
```

局部变量名称要简洁(这个规则比较抽象, 只能多看看内核代码中其他人的命名方式)

## **5、Typedefs**

尽量不要使用 typedef, 使用typedef主要为了下面的用途:

1. 完全不透明的类型(访问这些类型也需要对应的访问函数)

ex. pid_t, uid_t, pte_t ... 等等

2. 避免整型数据的困扰

比如int, long类型的长度在不同体系结构中不一致等等, 使用 u8/u16/u32 来代替整型定义

3. 当使用kernel的sparse工具做变量类型检查时, 可以typedef一个类型.

4. 定义C99标准中的新类型

5. 为了用户空间的类型安全

内核空间的结构体映射到用户空间时使用typedef, 这样即使内核空间的数据结构有变化, 用户空间也能正常运行

## **6、函数**

函数要简短,一个函数只做一件事情

函数长度一般不超过2屏(1屏的大小是 80x24), 也就是48行

如果函数中的 switch 有很多简单的 case语句, 那么超出2屏也可以

函数中局部变量不能超过 5~10 个

函数与函数之间空一行, 但是和EXPORT* 之间不用空

```text
int one_func(void)
{
        return 0;
}

int system_is_up(void)
{
        return system_state == SYSTEM_RUNNING;
}
EXPORT_SYMBOL(system_is_up);
```

## **7、函数退出**

将函数的退出集中在一起, 特别有需要清理内存的时候.(goto 并不是洪水猛兽, 有时也很有用)

```text
int fun(int a)
{
        int result = 0;
        char *buffer = kmalloc(SIZE);

        if (buffer == NULL)
                return -ENOMEM;

        if (condition1) {
                while (loop1) {
                        ...
                }
                result = 1;
                goto out;
        }
        ...
        out:
                kfree(buffer);
                return result;
}
```

## **8、注释**

注释code做了什么, 而不是如何做的

使用C89的注释风格(/* ... */), 不要用C99的注释风格(// ...)

注释定义的数据, 不管是基本类型还是衍生的类型

## **9、控制缩进的方法**

用emacs来保证缩进, .emacs中的配置如下:

```text
(defun c-lineup-arglist-tabs-only (ignored)
  "Line up argument lists by tabs, not spaces"
  (let* ((anchor (c-langelem-pos c-syntactic-element))
     (column (c-langelem-2nd-pos c-syntactic-element))
     (offset (- (1+ column) anchor))
     (steps (floor offset c-basic-offset)))
    (* (max steps 1)
       c-basic-offset)))

(add-hook 'c-mode-common-hook
          (lambda ()
            ;; Add kernel style
            (c-add-style
             "linux-tabs-only"
             '("linux" (c-offsets-alist
                        (arglist-cont-nonempty
                         c-lineup-gcc-asm-reg
                         c-lineup-arglist-tabs-only))))))

(add-hook 'c-mode-hook
          (lambda ()
            (let ((filename (buffer-file-name)))
              ;; Enable kernel mode for the appropriate files
              (when (and filename
                         (string-match (expand-file-name "~/src/linux-trees")
                                       filename))
                (setq indent-tabs-mode t)
                (c-set-style "linux-tabs-only")))))
```

使用 indent 脚本来保证缩进. (脚本位置: scripts/Lindent)

## **10、Kconfig配置文件**

"config" 下一行要缩进一个Tab, "help" 下则缩进2个空格

```text
config AUDIT
        bool "Auditing support"
        depends on NET
        help
          Enable auditing infrastructure that can be used with another
          kernel subsystem, such as SELinux (which requires this for
          logging of avc messages output).  Does not do system-call
          auditing without CONFIG_AUDITSYSCALL.
```

不稳定的特性要加上 *EXPERIMENTAL*

```text
config SLUB
        depends on EXPERIMENTAL && !ARCH_USES_SLAB_PAGE_STRUCT
        bool "SLUB (Unqueued Allocator)"
        ...
```

危险的特性要加上 *DANGEROUS*

```text
config ADFS_FS_RW
        bool "ADFS write support (DANGEROUS)"
        depends on ADFS_FS
        ...    
```

## **11、数据结构**

结构体要包含一个引用计数字段 (内核中没有自动垃圾收集, 需要手动释放内存)

需要保证结构体数据一致性时要加锁

结构体中有时甚至会需要多层的引用计数

ex. struc mm_struct, struct super_block

## **12、宏, 枚举类型和RTL**

宏定义常量后者标签时使用大写字母

```text
#define CONSTANT 0x12345
```

宏定义多行语句时要放入 do - while 中, 此时宏的名称用小写

```text
#define macrofun(a, b, c)             \
    do {                    \
        if (a == 5)            \
            do_this(b, c);        \
    } while (0)
```

宏定义表达式时要放在 () 中

```text
#define CONSTANT 0x4000
#define CONSTEXP (CONSTANT | 3)
```

枚举类型 用来定义多个有联系的常量

## **13、打印内核消息**

保持打印信息的简明清晰

比如, 不要用 "dont", 而是使用 "do not" 或者 "don't"

内核信息不需要使用 "." 结尾

打印 "(%d)" 之类的没有任何意义, 应该避免

选择合适的打印级别(调试,还是警告,错误等等)

## **14、分配内存**

分配内存时sizeof(指针) 而不是 sizeof(结构体)

```text
p = kmalloc(sizeof(*p), ...);
```

这样的话, 当p指向其他结构体时, 上面的代码仍然可用

分配内存后返回值转为 void指针是多余的, C语言本身会完成这个步骤

## **15、内联的弊端**

如果一个函数有3行以上, 不要使用 inline 来标记它

## **16、函数返回值和名称**

如果函数功能是一个动作或者一个命令时, 返回 int型的 error-code

比如, add_work() 函数执行成功时返回 0, 失败时返回对应的error-code(ex. -EBUSY)

如果函数功能是一个判断时, 返回 "0" 表示失败, "1" 表示成功

所有Exported函数, 公用的函数都要上述2条要求

所有私有(static)函数, 不强制要求, 但最好能满足上面2条要求

如果函数返回真实计算结果, 而不是是否成功时, 通过返回计算结果范围外的值来表示失败

比如一个返回指针的函数, 通过返回 NULL 来表示失败

## **17、不要重复发明内核宏**

内核定义的宏在头文件 *include/linux/kernel.h* 中, 想定义新的宏时, 首先看看其中是否已有类似的宏可用

## **18、编辑器模式行和其他**

不要在代码中加入特定编辑器的内容或者其他工具的配置,

比如 emacs 的配置

```text
-*- mode: c -*-
or
Local Variables:
compile-command: "gcc -DMAGIC_DEBUG_FLAG foo.c"
End:
```

vim 的配置

```text
vim:set sw=8 noet 
```

每个人使用的开发工具可能都不一样, 这样的配置对有些人来说就是垃圾

---

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/548067470