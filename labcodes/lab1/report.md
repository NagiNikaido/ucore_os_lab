# Lab1 report

## 计64 2016011322 陶东来

### 练习1

生成ucore.img的对应makefile代码。可以看到需要先生成bootblock和kernel：

```makefile
    UCOREIMG	:= $(call totarget,ucore.img)

    $(UCOREIMG): $(kernel) $(bootblock)
        $(V)dd if=/dev/zero of=$@ count=10000
        $(V)dd if=$(bootblock) of=$@ conv=notrunc
        $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

    $(call create_target,ucore.img)
```

生成bootblock：

```makefile
    $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
        @echo + ld $@
        $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
        @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
        @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
        @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

可以看到需要首先生成bootasm.o、bootmain.o和sign。

生成bootasm.o和bootmain.o：
```makefile
    bootfiles = $(call listf_cc,boot)
    $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
```
实际上命令应该为：
```makefile
    gcc -Iboot/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs \
        -nostdinc -fno-stack-protector -Ilibs/ -Os \
        -c boot/bootasm.S -o obj/boot/bootasm.o
```
bootmain.o同理。其中的关键参数有：
```
    -fno-builtin 除非使用__builtin__前缀，否则不进行内联函数优化
    -fno-PIC 不生成位置无关代码(Position-Indepedent Code)，在boot阶段代码装载的位置都是固定的，不用考虑动态链接
    -fno-stack-protector 不生成用于检测缓冲区溢出的代码
    -ggdb 生成可供gdb使用的调试信息
    -m32 生成适用于32位环境的代码
    -gstabs 生成stabs格式的调试信息，这样ucore的monitor可以显示出函数调用栈信息
    -nostdinc 不使用C标准库。C标准库是OS提供的服务，现在必须start from scratches
    -Os 为减小代码大小而进行优化，以便塞进主引导扇区中。
    -I<dir> 添加搜索头文件的路径
```

生成sign：
```makefile
    $(call add_files_host,tools/sign.c,sign,sign)
    $(call create_target_host,sign,sign)
```
实际命令为：
```shell
    gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
    gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```

接下来生成bootblock.o：
```shell
    ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
          obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```

其中的关键参数有：
```
    -m <emulation> 模拟为<emulation>上的链接器
    -nostdlib 不使用标准库
    -N 设置代码段和数据段均可读写
    -e <entry> 指定入口
    -Ttext 指定代码段开始位置
```

然后，拷贝二进制代码bootblock.o到bootblock.out
```shell
    objcopy -S -O binary obj/bootblock.o obj/bootblock.out
```
其中的关键参数为
```
    -S 移除所有符号和重定向信息
    -O <bfdname> 指定输出格式
```

最后使用sign工具处理bootblock.out，生成bootblock
```shell
    bin/sign obj/bootblock.out bin/bootblock
```

生成kernel：
```makefile
    $(kernel): tools/kernel.ld
    $(kernel): $(KOBJS)
        @echo + ld $@
        $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
        @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
        @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```

kernel依赖kernel.ld和obj/kern/*/*.o。其中kernel.ld已经存在，而后者生成方式均类似，这里以trap.o为例：
```shell
    gcc -Ikern/trap/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs \
        -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ \
        -Ikern/trap/ -Ikern/mm/ -c kern/trap/trap.c -o obj/kern/trap/trap.o
```

生成kernel时的命令为：
```shell
    ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
             obj/kern/init/init.o obj/kern/libs/stdio.o \
             obj/kern/libs/readline.o obj/kern/debug/panic.o \
             obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o \
             obj/kern/driver/clock.o obj/kern/driver/console.o \
             obj/kern/driver/picirq.o obj/kern/driver/intr.o \
             obj/kern/trap/trap.o obj/kern/trap/vectors.o \
             obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  \
             obj/libs/string.o obj/libs/printfmt.o
```
其中的关键参数为
```shell
    -T <scriptfile> 指定链接器使用的脚本
```

这样bootblock和kernel就生成完毕了，接下来将它们合并到一个文件中。首先先生成一个有10000个块的文件，每个块512字节，以0填充：
```shell
    dd if=/dev/zero of=bin/ucore.img count=10000
```

然后把bootblock的内容写到第一个块：
```shell
    dd if=bin/bootblock of=bin/ucore.img conv=notrunc
```

最后从第二个块开始写入kernel的内容：
```shell
    dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

这样ucore.img就生成完毕了。

一个被系统认为是符合硬盘主引导扇区的特征有：
+ 只有512字节
+ 倒数两个字节依次为0x55和0xAA

### 练习2

1. 从加电开始单步追踪

修改tools/gdbinit，去掉以下内容：
```gdb
    break ker_init
    continue
```

然后在lab1目录下执行
```shell
    make debug
```

这样只需要在gdb调试界面下执行
```
    si
```
即可单步跟踪了。

2.从0x7c00设置断点

修改tools/gdbinit为
```
    b *0x7c00
    c
    x /10i $pc

```
即可。

3.比对是否一致

借助2中的gdbinit即可对比，得知一致。

### 练习3
分析bootloader进入保护模式的过程

这个过程实现在bootasm.S中，注释的描述也已经非常详尽了，因此这里不再赘述，只简单叙述以下过程：
+ 首先将flag和段寄存器置0,清理环境
+ 接着打开A20线，使得全部32位地址线可用
+ 接下来需要初始化GDT表，直接装载静态GDT表即可
+ 然后进入保护模式，只需将cr0寄存器PE位置置1即可
+ 接下来上一些扫尾工作：通过longjmp更新cs的基地址，然后设置段寄存器并建立堆栈
+ 最后call bootmain，完成！

### 练习4
分析bootloader加载ELF格式OS的过程。

为了加载ELF格式的文件，bootloader使用了3个辅助函数：
```C
    static void
    waitdisk(void){
        /* 用一个循环不断扫描0x1F7来确认硬盘是否准备好 */
    }

    static void
    readsect(void *dst, uint32_t secno) {
        /* 将第secno个扇区读到dst位置，这里只读一个扇区 */
        /* 将读取扇区个数（此处为1）、扇区号和0x20命令输入从0x1F2开始的6个字节中*/
        /* 在磁盘准备好之后可以从0x1F0开始将512个字节读到dst的位置*/
    }

    static void
    readseg(unintptr_t va, uint32_t count, uint32_t offset) {
        /* 将kernel中从offset开始的count字节读到虚拟地址va */
        /* 机理就是反复调用readsect函数，不断将扇区读入 */
    }

    void
    bootmain(void) {
        /* boot的main函数 */
        /* 会先读入并校验ELF头，检测ELF_MAGIC */
        /* 通过检测之后，依照ELF的描述表将数据存入内存 */
        /* 最后依照ELF头的入口信息，找到内核入口并进入*/
    }
```



 



