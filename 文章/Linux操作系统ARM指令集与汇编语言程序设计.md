## 一、实验目的

1.了解并掌握ARM汇编指令集

2.应用ARM指令集编写一个程序操控开发板上的LED灯

## 二、实验要求

应用ARM汇编指令集编写程序，实现正常状态下开发板上的LED灯不亮，按下一个按键之后开发板上的LED灯进入流水灯模式。

## 三、实验原理

**四个LED灯的电路如下图所示：**

![img](https://pic1.zhimg.com/80/v2-614cfeecea624d60bcfd150387828214_720w.webp)

![img](https://pic1.zhimg.com/80/v2-614cfeecea624d60bcfd150387828214_720w.webp)

**四个按键电路图如下所示：**

![img](https://pic3.zhimg.com/80/v2-f42013cca1e1d4acbdf4a01b6cf2fc42_720w.webp)

![img](https://pic3.zhimg.com/80/v2-82d3485cc2fb992c64a639852770d812_720w.webp)

![img](https://pic4.zhimg.com/80/v2-09e73b394ae37b3209c7aa62b448c90f_720w.webp)

**将LED灯的控制地址放入一个寄存器中，并将其设置为输出模式:**

![img](https://pic4.zhimg.com/80/v2-d5010fee1006340b9416d2b9038ab67b_720w.webp)

![img](https://pic2.zhimg.com/80/v2-6b960d906551a8fdf50be58d586440d1_720w.webp)

把按键控制地址的内容全置零，为输入模式（复位值为0，此步骤可省略）。

## 四、实验结果

将生成的二进制代码用烧写脚本烧写到SD卡中，插入开发板的SD卡槽，从SD卡启动，按下按键即开启LED流水灯模式。

## 五、结果分析

通过掩码取出按键数据地址中的值的状态来判断按键是否按下，若按下则跳转到LED流水灯的程序当中。

流水灯程序即顺序的程序结构，依次点亮LED灯，延时并熄灭，达到流水效果。

## 六、附录：实验源代码

```cpp
.text
.globl _start
_start:
	// Set GPM4CON[0:3]as output
	ldr r1, =0x110002E0 @GPM4CON address 					
	ldr r0, =0x00001111
	str r0, [r1]
	//Set GPX3CON[2:5] as input
	ldr r1, =0x11000C60 @GPX3CON address
	mov r0, #0
	str r0,[r1]			@default is input.So,this section canbe omitted
Key_led:
	//Read the state of keys
	ldr r1, =0x11000C64 @GPX3DAT address->keys
	ldr r3, =0xFFFFFFC3 @Key mask
	ldr r0, [r1] 		@ Read the state of keys
	orr r0, r0,r3		@ get the bits of keys
	mov r0,r0,ROR #2	@ align to the led bits
	cmp r0, #0xFFFFFFFE
	beq led_blink
	bl Key_led
led_blink:
	ldr r1, =0x110002E4
	mov r0, #0xE
	str r0, [r1]
	bl delay
	ldr r1, =0x110002E4
	mov r0, #0xD
	str r0, [r1]
	bl delay
	ldr r1, =0x110002E4
	mov r0, #0xB
	str r0, [r1]
	bl delay
	ldr r1, =0x110002E4
	mov r0, #0x7
	str r0, [r1]
	bl delay
	bl Key_led
delay:
	mov r0, #0x200000
delay_loop:
	cmp r0, #0
	sub r0, r0, #1
	bne delay_loop
	mov pc, lr
```

-----

版权声明：本文为知乎博主「玩转Linux内核」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://zhuanlan.zhihu.com/p/449816076