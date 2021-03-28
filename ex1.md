#练习一
---
##1. 操作系统镜像文件ucore.img如何生成
在输入make V=之后，可以获得详细信息如下

	+ cc kern/init/init.c           
      gcc -c kern/init/init.c -o obj/kern/init/init.o

	+ cc kern/libs/readline.c       
      gcc -c kern/libs/readline.c -o 
      obj/kern/libs/readline.o

	+ cc kern/libs/stdio.c          
      gcc -c kern/libs/stdio.c -o obj/kern/libs/stdio.o

	+ cc kern/debug/kdebug.c        
      gcc -c kern/debug/kdebug.c -o obj/kern/debug/kdebug.o

	+ cc kern/debug/kmonitor.c      
      gcc  -c kern/debug/kmonitor.c -o         
      obj/kern/debug/kmonitor.o

	+ cc kern/debug/panic.c         
      gcc  -c kern/debug/panic.c -o obj/kern/debug/panic.o

	+ cc kern/driver/clock.c       
      gcc  -c kern/driver/clock.c -o obj/kern/driver/clock.o

	+ cc kern/driver/console.c    
      gcc -c kern/driver/console.c -o 
      obj/kern/driver/console.o

	+ cc kern/driver/intr.c       
      gcc -c kern/driver/intr.c -o obj/kern/driver/intr.o

	+ cc kern/driver/picirq.c     
      gcc -c kern/driver/picirq.c -o 
      obj/kern/driver/picirq.o

	+ cc kern/trap/trap.c          
      gcc -c kern/trap/trap.c -o obj/kern/trap/trap.o

	+ cc kern/trap/trapentry.S      
      gcc -c kern/trap/trapentry.S -o 
      obj/kern/trap/trapentry.o

	+ cc kern/trap/vectors.S        
      gcc -c kern/trap/vectors.S -o obj/kern/trap/vectors.o

	+ cc kern/mm/pmm.c              
      gcc -c kern/mm/pmm.c -o obj/kern/mm/pmm.o

	+ cc libs/printfmt.c            
      gcc -c libs/printfmt.c -o obj/libs/printfmt.o

	+ cc libs/string.c             
      gcc -c libs/string.c -o obj/libs/string.o

	+ ld bin/kernel                 
      ld -o bin/kernel  
      obj/kern/init/init.o      obj/kern/libs/readline.o 
      obj/kern/libs/stdio.o     obj/kern/debug/kdebug.o 
      obj/kern/debug/kmonitor.o obj/kern/debug/panic.o 
      obj/kern/driver/clock.o   obj/kern/driver/console.o 
      obj/kern/driver/intr.o    obj/kern/driver/picirq.o
      obj/kern/trap/trap.o      obj/kern/trap/trapentry.o 
      obj/kern/trap/vectors.o   obj/kern/mm/pmm.o  
      obj/libs/printfmt.o       obj/libs/string.o

	+ cc boot/bootasm.S             //编译bootasm.S
     gcc  -c boot/bootasm.S -o obj/boot/bootasm.o

	+ cc boot/bootmain.c            //编译bootmain.c
     gcc -c boot/bootmain.c -o obj/boot/bootmain.o

	+ cc tools/sign.c               //编译sign.c
    gcc -c tools/sign.c -o obj/sign/tools/sign.o
    gcc -O2 obj/sign/tools/sign.o -o bin/sign

	+ ld bin/bootblock              //根据sign规范生成bootblock
    ld -m  elf_i386 -nostdlib -N -e start -Ttext 0x7C00 
    obj/boot/bootasm.o  obj/boot/bootmain.o
    -o obj/bootblock.o
在makefile文件中搜索ucore.img可以找到

	#create ucore.img
	UCOREIMG	:=$(call totarget,ucore.img)
	$(UCOREIMG):$(kernel)$(bootblock)
		
		$
		$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
	$(call create_target,ucore.img)

可以看出ucore.img镜像文件是由kernel和bootblock文件生成的。

	$(V)dd if=/dev/zero of=$@
	count=100000
这句语句可以看到UCOREIMG分配了一定空间。

	(V)dd if=$(bootblock) of=$@ conv=notrunc
这句语句将bootblock复制到上面分配的空间当中。

	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
该语句将kernel复制到分配的空间当中。

1.kernel生成
	
	kernel = $(call totarget,kernel)
	$(kernel): tools/kernel.ld
通过链接来生成kernel目标文件

	$(kernel): $(KOBJS)               
kernel的生成还依赖KOBJS

	@echo + ld $@                
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)   
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)  
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
	$(call symfile,kernel)
	kernel = $(call totarget,kernel)
2.bootblock生成

	bootfiles = $(call listf_cc,boot)      
用boot替换listf\_cc里面的变量，将listf\_cc的返回值赋给bootfiles,也就是滤出.c,.s文件

	$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
编译bookfiles

	bootblock = $(call totarget,bootblock) 
生成bootblock
	$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)  
生成目标文件bootblock需要依赖于sign和bootfiles

	@echo + ld $@        
将以下文件与bootblock连接起来

	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)    
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

	$(call create_target,bootblock)
3.生成sign工具

	$(call add_files_host,tools/sign.c,sign,sign)
	$(call create_target_host,sign,sign)

* 由sign工具、bootfile生成bootblock
* 由KOBJS生成kernel
* 由kernel和bootblock生成最终的ucore.img
* **

##2.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?
通过查阅资料我们可以知道bootblock区域包含用于引导的最小指令集，而在上一个问题中我们发现bootblock的生成需要依赖于sign.c文件和bootfiles文件，其中bootfiles提供开机启动所需要的文件，而sign.c则代表生成bootblock的规范。
因此，我们去在文件夹中查看sign.c文件。
文件代码如下：

	char buf[512];  //定义buf数组
    memset(buf, 0, sizeof(buf));
      // 把buf数组的最后两位置为 0x55, 0xAA
    buf[510] = 0x55;  
    buf[511] = 0xAA;
    FILE *ofp = fopen(argv[2], "wb+");
    size = fwrite(buf, 1, 512, ofp);
    if (size != 512) {       //大小为512字节
        fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);            
        return -1;
    }
通过查看这里的代码，我们可以发现sign规范中给buf提供了512个字节的空间，而且bootblock的格式是最后两个字节分别是0x55和0xAA，而这也是操作系统课上提到的两个神奇的数。