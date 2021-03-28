#练习3 分析bootloader进入保护模式的过程
***
##1.为何开启A20,如何开启A20

因为8086机器只有16位，只能寻址1MB的空间，而为了访问到1MB以上的空间而且又与以前的机器“回卷”操作兼容，可以通过A20开关来选择访问内存的方式。从实模式到保护模式。

在bootasm文件中搜索a20找到代码如下

	seta20.1:
    inb $0x64, %al          # Wait for not busy(8042 input buffer empty).          
    testb $0x2, %al			#这里是测试al的第2位，看是否跳转
    jnz seta20.1

    movb $0xd1, %al         # 0xd1 -> port 0x64
    outb %al, $0x64         # 0xd1 means: write data to 8042's P2 port，写入信息

	seta20.2:
    inb $0x64, %al          # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2
    movb $0xdf, %al         # 0xdf -> port 0x60
    outb %al,$0x60          # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1，到此成功设置A20gate
刚刚出现A20门的时候，工程师使用8042键盘控制器来控制A20gate，因此在第一行代码中要等到8042可以使用的时候才可以去设置A20gate。设置A20gate分成两步，seta20.1和seta20.2，分别将0x64和0xd1的数据输入al，然后第一块判断a1的第二位，如果是1就跳转回来重做，第二块的逻辑也类似。最后将0x60放入al中，然后在0xdf端口输出0x60的数据，设置A20gate的开启。

##2.如何初始化gdt表
 	lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0

    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.

	# Bootstrap GDT
	.p2align 2                                          # force 4 byte alignment
	gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

	gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
##3.进入保护模式
	    ljmp $PROT_MODE_CSEG, $protcseg
在执行完这句代码之后就进入保护模式了。