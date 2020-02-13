# 练习1
## 练习1.1 
> 操作系统镜像文件ucore.img是如何一步一步生成的？（需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果。

### ucore.img的生成过程
1. 编译`libs`和`kern`目录下的.c和.S文件，生成.o文件，并连接得到`bin/kernel`文件
2. 编译`boot`目录下的.c和.S文件，生成.o文件，并连接得到`bin/block.out`文件
3. 编译`tools/sign.c`文件，得到`bin/sign`文件
4. 利用`bin/sign`工具将`bin/bootblock.out`文件转换为512字节的`bin/bootblock`文件，并将`bin/bootblock`的最后两个字节设置为0x55AA
5. 为`bin/ucore.img`分配5000KB的内存空间，并将`bin/bootblock`复制到`bin/ucore.img`的第一个块里，将`bin/kernel`复制到`bin/ucore.img`的第二个块开始的位置。

### kernel的生成过程
**代码：**
```makefile
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

**代码解释：**
1. 第一行指出`kernel`的目标路径，`bin/kernel`
2. 第三行指出`kernel`目标文件依赖`tools/kernel.ld`文件
3. 第五行指出`kernel`目标文件依赖的obj文件，`KOBJS=obj/libs/*.o obj/kern/**/*.o`
4. 第七行指出使用`tools/kernel.ld`脚本连接。实际执行代码为：
```
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
```
5. 第八行使用objdump将bin/kernel反汇编为obj/kernel.asm，以便后续调试。实际执行代码为：
```
objdump -S bin/kern > obj/kernel.asm
```
6. 第九行使用objdump解析bin/kernel，得到符号表文件obj/kernel.sym。实际执行代码为：
```
objdump -t bin/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$/d' > obj/kernel.sym
```

### bootblock的生成过程
**代码：**
```makefile
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```

**代码解释：**
1. 第一行，获取`boot`目录下的源文件（.c .S），`bootfiles=boot/*.c boot/*.S`
2. 第二行，将`boot/*.c boot/*.S`编译成`obj/boot/*.o`
3. 第四行，指定`bootblock`的目标路径，`bootblock=bin/bootblock`
4. 第六行，声明`bin/bootblock`依赖于`obj/boot/*.o bin/sign`
5. 第八行，连接所有`obj/boot/*.o`生成`obj/bootblock.o`
6. 第九行，使用`objdump`反汇编`obj/bootblock.o`为`obj/bootblock.asm`
7. 第十行，使用`objcopy`将`obj/bootblock.o`转换为`obj/bootblock.out`并去掉重定位和符号信息。
8. 第十一行，使用`bin/sign`工具将`obj/bootblock.out`转换为`obj/bootblock`（使得最终的文件大小为512字节，并且以0x55AA结尾，即ELF格式）

### sign工具的生成
**sign的生成代码：**
```makefile
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

**代码解释：**
1. 第一行，设置`__objs_sign=obj/sign/tools/sign.o`
2. 第二行，生成`bin/sign`


### Makefile代码解释
```makefile
# 定义了五个变量，PROJ、EMPTY、SPACE、SLASH、V。
PROJ	:= challenge
EMPTY	:=
SPACE	:= $(EMPTY) $(EMPTY)
SLASH	:= /

V       := @
#need llvm/cang-3.5+
#USELLVM := 1
# try to infer the correct GCCPREFX

# 如果未定义GCCPREFIX，则运行该shell脚本
# 执行命令i386-elf-objdump -i，2>&1将标准错误重定向到标准输出
# 通过grep查找输出的结果中是否包含开头为elf32-i386的字符串
# 如果包含，则输出i386-elf-i赋值给GCCPREFIX
# 如果不包含，则执行objdump -i命令（显示所有支持的架构和目标格式），并查找输出结果中是否包含elf32-i386
# 如果包含，则输出空赋值给GCCPREFIX
# 如果以上两种情况均不满足，则输出错误信息，并退出
ifndef GCCPREFIX
GCCPREFIX := $(shell if i386-elf-objdump -i 2>&1 | grep '^elf32-i386$$' >/dev/null 2>&1; \
	then echo 'i386-elf-'; \
	elif objdump -i 2>&1 | grep 'elf32-i386' >/dev/null 2>&1; \
	then echo ''; \
	else echo "***" 1>&2; \
	echo "*** Error: Couldn't find an i386-elf version of GCC/binutils." 1>&2; \
	echo "*** Is the directory with i386-elf-gcc in your PATH?" 1>&2; \
	echo "*** If your i386-elf toolchain is installed with a command" 1>&2; \
	echo "*** prefix other than 'i386-elf-', set your GCCPREFIX" 1>&2; \
	echo "*** environment variable to that prefix and run 'make' again." 1>&2; \
	echo "*** To turn off this error, run 'gmake GCCPREFIX= ...'." 1>&2; \
	echo "***" 1>&2; exit 1; fi)
endif

# try to infer the correct QEMU
# 如果未定义QEMU，则执行该shell命令
# 查询qemu-system-i386的路径
# 如果存在，则QEMU被赋值为qemu-system-i386
# 如果不存在，则查询i386-elf-qemu
# 如果存在，则QEMU被赋值为i386-elf-qemu
# 否则，输出错误信息
ifndef QEMU
QEMU := $(shell if which qemu-system-i386 > /dev/null; \
	then echo 'qemu-system-i386'; exit; \
	elif which i386-elf-qemu > /dev/null; \
	then echo 'i386-elf-qemu'; exit; \
	elif which qemu > /dev/null; \
	then echo 'qemu'; exit; \
	else \
	echo "***" 1>&2; \
	echo "*** Error: Couldn't find a working QEMU executable." 1>&2; \
	echo "*** Is the directory containing the qemu binary in your PATH" 1>&2; \
	echo "***" 1>&2; exit 1; fi)
endif

# eliminate default suffix rules
.SUFFIXES: .c .S .h

# delete target files if there is an error (or make is interrupted)
.DELETE_ON_ERROR:

# define compiler and flags
ifndef  USELLVM

# HOSTCC是主机所用的编译器
HOSTCC		:= gcc

# -g是为了生成符号表，以便gdb能进行调试
# -Wall是生成警告信息
# -O2是优化处理
HOSTCFLAGS	:= -g -Wall -O2

# CC是i386，elf32格式的编译器
CC		:= $(GCCPREFIX)gcc

# -fno-builtin: 不接收不以__builtin_开头的内置函数
# -Wall: 警告
# -ggdb: 生成GDB用的调试信息
# -m32: 编译32位程序
# -gstabs: 此选项以stabs格式生成调试信息,但是不包括gdb扩展调试信息
# -nostdinc: 不搜索标准头文件目录，仅搜索-I指定的目录
CFLAGS	:= -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)

# 如果 -fno-stack-protector 存在，则开启 -fno-stack-protector
# -fstack-protector: 生成额外的代码检查缓冲区溢出
# -E: 仅进行预处理
# -x c: 指定编程语言为c语言
CFLAGS	+= $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
else
HOSTCC		:= clang
HOSTCFLAGS	:= -g -Wall -O2
CC		:= clang
CFLAGS	:= -march=i686 -fno-builtin -fno-PIC -Wall -g -m32 -nostdinc $(DEFS)
CFLAGS	+= $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
endif

# 源文件类型为c或者S
CTYPE	:= c S

# ld: 连接器
LD      := $(GCCPREFIX)ld

# -m: 使用模拟器
# -V: 查看所有连接器支持的模拟器
# head -n 1: 取第一行
# -nostdlib: 不连接标准库
LDFLAGS	:= -m $(shell $(LD) -V | grep elf_i386 2>/dev/null | head -n 1)
LDFLAGS	+= -nostdlib

OBJCOPY := $(GCCPREFIX)objcopy
OBJDUMP := $(GCCPREFIX)objdump

COPY	:= cp
MKDIR   := mkdir -p
MV		:= mv
RM		:= rm -f
AWK		:= awk
SED		:= sed
SH		:= sh
TR		:= tr
TOUCH	:= touch -c

OBJDIR	:= obj
BINDIR	:= bin

ALLOBJS	:=
ALLDEPS	:=
TARGETS	:=

include tools/function.mk

# 列出$(1)中所有的.c和.S文件
listf_cc = $(call listf,$(1),$(CTYPE))

# for cc
add_files_cc = $(call add_files,$(1),$(CC),$(CFLAGS) $(3),$(2),$(4))
create_target_cc = $(call create_target,$(1),$(2),$(3),$(CC),$(CFLAGS))

# for hostcc
add_files_host = $(call add_files,$(1),$(HOSTCC),$(HOSTCFLAGS),$(2),$(3))
create_target_host = $(call create_target,$(1),$(2),$(3),$(HOSTCC),$(HOSTCFLAGS))

cgtype = $(patsubst %.$(2),%.$(3),$(1))
objfile = $(call toobj,$(1))
asmfile = $(call cgtype,$(call toobj,$(1)),o,asm)
outfile = $(call cgtype,$(call toobj,$(1)),o,out)
symfile = $(call cgtype,$(call toobj,$(1)),o,sym)

# for match pattern
match = $(shell echo $(2) | $(AWK) '{for(i=1;i<=NF;i++){if(match("$(1)","^"$$(i)"$$")){exit 1;}}}'; echo $$?)

# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
# include kernel/user

INCLUDE	+= libs/

CFLAGS	+= $(addprefix -I,$(INCLUDE))

LIBDIR	+= libs

$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)

# -------------------------------------------------------------------
# kernel

KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

# 给kernel的include路径加上-I
KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

KOBJS	= $(call read_packet,kernel libs)

# create kernel target
# 给kernel添加上目标路径 => bin/kernel
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)

# -------------------------------------------------------------------

# create bootblock
# 列出boot中的.c和.S文件 => bootmain.c、bootasm.S
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

# 给bootblock添加目标路径 => bin/bootblock
bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	# 使用ld连接bootasm.o和bootmain.o，生成bootblock.o
	# -N使data和text节可读可写，-e start指出入口符号为start，-Ttext 0x7C00，将代码重定位到0x7C00
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	# 使用objdump反汇编bootblock.o，生成bootblock.asm文件
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	# 使用objcopy将bootblock.o转换为bootblock.out，-S表示去掉重定位和符号信息，-O binary表示文件格式为二进制
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	# 使用sign工具将bootblock.out转换为bootblock文件
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)

# -------------------------------------------------------------------

# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)

# -------------------------------------------------------------------

# create ucore.img
# 为ucore.img添加目标路径 => bin/ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

# dd: 转换、复制文件
# if: 输入文件
# of: 输出文件
# count: 要复制的块数。（默认每个块为512字节）
# seek: 输出到目标文件时需要跳过的块数，即从第seek块之后开始写入
# conv=notrunc: 不截断输出文件
# 以下命令的意思是：
# 1.为ucore.img分配10000*512块的空间，全部置0
# 2.将bootblock复制到ucore.img的开头处
# 3.将kernel复制到ucore.img的第二块开始处
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)

# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

$(call finish_all)

IGNORE_ALLDEPS	= clean \
				  dist-clean \
				  grade \
				  touch \
				  print-.+ \
				  handin

ifeq ($(call match,$(MAKECMDGOALS),$(IGNORE_ALLDEPS)),0)
-include $(ALLDEPS)
endif

# files for grade script

TARGETS: $(TARGETS)

.DEFAULT_GOAL := TARGETS

.PHONY: qemu qemu-nox debug debug-nox
qemu-mon: $(UCOREIMG)
	$(V)$(QEMU)  -no-reboot -monitor stdio -hda $< -serial null
qemu: $(UCOREIMG)
	$(V)$(QEMU) -no-reboot -parallel stdio -hda $< -serial null
log: $(UCOREIMG)
	$(V)$(QEMU) -no-reboot -d int,cpu_reset  -D q.log -parallel stdio -hda $< -serial null
qemu-nox: $(UCOREIMG)
	$(V)$(QEMU)   -no-reboot -serial mon:stdio -hda $< -nographic
TERMINAL        :=gnome-terminal
debug: $(UCOREIMG)
	$(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
	
debug-nox: $(UCOREIMG)
	$(V)$(QEMU) -S -s -serial mon:stdio -hda $< -nographic &
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"

.PHONY: grade touch

GRADE_GDB_IN	:= .gdb.in
GRADE_QEMU_OUT	:= .qemu.out
HANDIN			:= proj$(PROJ)-handin.tar.gz

TOUCH_FILES		:= kern/trap/trap.c

MAKEOPTS		:= --quiet --no-print-directory

grade:
	$(V)$(MAKE) $(MAKEOPTS) clean
	$(V)$(SH) tools/grade.sh

touch:
	$(V)$(foreach f,$(TOUCH_FILES),$(TOUCH) $(f))

print-%:
	@echo $($(shell echo $(patsubst print-%,%,$@) | $(TR) [a-z] [A-Z]))

.PHONY: clean dist-clean handin packall tags
clean:
	$(V)$(RM) $(GRADE_GDB_IN) $(GRADE_QEMU_OUT) cscope* tags
	-$(RM) -r $(OBJDIR) $(BINDIR)

dist-clean: clean
	-$(RM) $(HANDIN)

handin: packall
	@echo Please visit http://learn.tsinghua.edu.cn and upload $(HANDIN). Thanks!

packall: clean
	@$(RM) -f $(HANDIN)
	@tar -czf $(HANDIN) `find . -type f -o -type d | grep -v '^\.*$$' | grep -vF '$(HANDIN)'`

tags:
	@echo TAGS ALL
	$(V)rm -f cscope.files cscope.in.out cscope.out cscope.po.out tags
	$(V)find . -type f -name "*.[chS]" >cscope.files
	$(V)cscope -bq 
	$(V)ctags -L cscope.files
```




## 练习1.2 
> 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

> 这里我们可以打开sign.c进行查看，sign工具对bootblock进行了规范化，使得其大小为512个字节，且最后两个字节为0x55,0xAA。



# 练习2
> 使用qemu执行并调试lab1中的软件：
> 1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。
> 2. 在初始化位置0x7c00设置实地址断点，测试断点正常。
> 3. 从0x7c00开始跟踪代码运行，将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较。
> 4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

**启动qemu：**
```
qemu -S -s -hda bin/ucore.img -monitor stdio
```

**启动gdb：**
```
gdb
set architecture i8086
target remote :1234
```

**常用的gdb命令：**

[链接](https://www.linuxidc.com/Linux/2017-01/139028.htm)

> 1. 运行程序
> - run(r): 运行程序
> 2. 查看源代码
> - list(l): 查看最近十行代码 
> 3. 设置断点
> - break(b): 打断点，后面接行号/函数名/*地址
> 4. 单步调试
> - continue(c): 运行至下一个断点
> - next(n): 单步跟踪，不进入函数
> - step(s): 单步跟踪，可进入函数
> - nexti(ni): 单步跟踪一条机器指令，不进入函数
> - stepi(si): 单步跟踪一条机器指令
> 5. 查看运行时数据
> - print(p): 打印数据，后面接变量名/*地址/$寄存器
> 6. 查看内存
> - examine(x)/nfu address
> 	- n: 表示显示内存长度
>   - f: 表示输出格式
> 		- x: 十六进制
>		- d: 十进制
> 		- u: 十六进制无符号整型
> 		- o: 八进制
> 		- t: 二进制
> 		- a: 十六进制
> 		- c: 字符
> 		- f: 浮点数
> 		- i: 汇编指令
>   - u: 表示内存单位长度
> 		- b: 单字节
> 		- h: 双字节
> 		- w: 四字节
> 		- g: 八字节
> 7. 终止程序
> - kill(k): 终止正在调试的程序
> 8. 退出调试
> - quit(q): 退出gdb


# 练习3
> 分析bootloader进入保护模式的过程

**[为何开启A20，以及如何开启A20？](http://hengch.blog.163.com/blog/static/107800672009013104623747/)**
> Intel早期的8086 CPU提供了20根地址线，能寻址的范围是0~2^20(00000h~fffffh)共1MB的内存空间，
> 但是8086的数据处理位宽为16位，无法直接寻址1MB的内存空间，所以8086提供了段地址加偏移地址的地址转换机制。
> 计算机的寻址结构为segment:offset，segment和offset都是16位的寄存器，最大值位0xffffh，换算成物理地址的计算方法是
> 把segment左移四位，再加上offset，所以理论上segment:offset的寻址能力为0xffff0+0xffff=0x10ffefh，大概为1088KB，
> 也就是说，segment:offset这种表示方法的寻址能力超过了实际的20位地址线所能表示的物理地址大小，因此当寻址超过1MB的
> 内存时，会发生“回卷”（不会产生异常）。但下一代的基于Inter 80286 CPU的计算机系统提供了24根地址线，这样CPU的寻址
> 能力就成了2^24=16M，同时也提供了保护模式，可以访问到1MB以上的内存了。此时如果访问1MB以上的内存，系统就不会再
> “回卷”了，这就造成了向下不兼容。为了向下兼容（即提供“回卷”功能），于是出现了A20 Gate。
> 
> 这个A20地址线默认是被屏蔽的（总为0），直到系统软件去打开它，这个打开它的开关就叫做A20 Gate。
> 这样一来，在实模式中，寻址超过1MB时会被“回卷”。而在保护模式中寻址1MB以上的内存空间时，就需要将A20 Gate打开。
> 
> 那么A20 Gate又是怎么实现的呢？
> 早期的PC机，控制键盘有一个单独的单片机8042，现如今这个芯片已经给集成到了其它大片子中，但其功能和使用方法还是一样，
> PC机刚刚出现A20 Gate的时候，估计实在找不到控制它的地方了，同时也不值得为这点小事增加芯片，于是工程师使用这个8042
> 键盘控制器来控制A20 Gate，但是A20 Gate真的和键盘无关。
>
> 如何开启A20 Gate呢？
> ![键盘控制器8042的逻辑结构图](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1_figs/image012.png)
> 8042键盘控制器的IO端口是0x60～0x6f，实际上IBM PC/AT使用的只有0x60和0x64两个端口（0x61、0x62和0x63用于与XT兼容
> 目的）。8042通过这些端口给键盘控制器或键盘发送命令或读取状态。输出端口P2用于特定目的。位0（P20引脚）用于实现CPU复
> 位操作，位1（P21引脚）用户控制A20信号线的开启与否。系统向输入缓冲（端口0x64）写入一个字节，即发送一个键盘控制器命
> 令。可以带一个参数。参数是通过0x60端口发送的。 命令的返回值也从端口 0x60去读。
> 
> 8042有4个寄存器：
> - 1个8-bit长的Input buffer；Write-Only；
> - 1个8-bit长的Output buffer； Read-Only；
> - 1个8-bit长的Status Register；Read-Only；
> - 1个8-bit长的Control Register；Read/Write。
>
>有两个端口地址：60h和64h，有关对它们的读写操作描述如下：
> - 读60h端口，读output buffer
> - 写60h端口，写input buffer
> - 读64h端口，读Status Register
> 操作Control Register，首先要向64h端口写一个命令（20h为读命令，60h为写命令），然后根据命令从60h端口读出Control
> Register的数据或者向60h端口写入Control Register的数据（64h端口还可以接受许多其它的命令）。
>
> 端口的操作都是通过向64h发送命令，然后在60h进行读写的方式完成，其中本文要操作的A20 Gate被定义在Output Port的bit
> 1上，所以有必要对Outport Port的操作及端口定义做一个说明。
> - 读Output Port：向64h发送0d0h命令，然后从60h读取Output Port的内容
> - 写Output Port：向64h发送0d1h命令，然后向60h写入Output Port的数据
>
> 打开A20 Gate的具体步骤大致如下（参考bootasm.S）：
> - 等待8042 Input buffer为空；
> - 发送Write 8042 Output Port （P2）命令到8042 Input buffer；
> - 等待8042 Input buffer为空；
> - 将8042 Output Port（P2）得到字节的第2位置1，然后写入8042 Input buffer；

**如何初始化GDT表？**
> [段描述符](https://blog.csdn.net/qq_36916179/article/details/91621947)
> ![段描述符结构图](https://img-blog.csdnimg.cn/20190612163123572.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2OTE2MTc5,size_16,color_FFFFFF,t_70)
>
> 首先，声明三个段描述符:
> 1. 第一个为空段描述符，全部清零
> 2. 第二个为代码段描述符，可执行可读，基地址为0，段界限为0xffffffffh，粒度为4B
> 3. 第三个为数据段描述符，可写，基地址为0，段界限为0xfffffffffh，粒度为4B
>
> 其次，声明段描述符表的界限和基址，并加载到GDTR中
> - 低16位为段界限
> - 高16位为基地址

**如何使能和进入保护模式：**
> 1. 屏蔽中断，设置串地址增长方向为正向，将ds、es、ss清零
> - cli: 中断允许标志位(IF)，0表示不响应可屏蔽中断
> - cld: 方向标志位(DF)，0表示递增，1表示递减
> 2. 开启A20
> - cpu waiting for 8042 are not busy
> - cpu --write 8042 output port--> 8042's input buffer
> - cpu waiting for 8042 are not busy
> - cpu --set A20 bit--> 8042's input buffer
> 3. 加载全局描述符表
> - lgdt: load global descriptor table，将数值加载到gdtr(全局描述符表寄存器)中，高32位为基地址，低16位为段界限
> 4. 从实模式切换到保护模式
> - cr0: 含有控制处理器操作模式和状态的系统控制标志
> 	- PE(Protected-Mode Enable Bit): 第0位。PE=0，CPU处于实模式；PE=1，CPU处于保护模式，并使用分段机制
> 	- PG(Paging Enable Bit): 第31位。PG=0，不启用分页机制；PG=1，启用分页机制


# 练习4
> 分析bootloader加载ELF格式的OS的过程
>
> 通过阅读bootmain.c，了解bootloader如何加载ELF文件。通过分析源代码和通过qemu来运行并调试bootloader&OS。
> - bootloader如何读取硬盘扇区的？
> - bootloader是如何加载ELF格式的OS？

**bootloader如何读取硬盘扇区的？**
> CPU使用[LBA模式](https://blog.csdn.net/cosmoslife/article/details/9029657)的PIO(Program IO)方式来访问硬盘
> 的（即所有的IO操作是通过CPU访问硬盘的IO地址寄存器完成）。
>
> 读取的流程：
> 1. 等待磁盘准备好
> 2. 发出读取扇区的命令
> 3. 等待磁盘准备好
> 4. 把磁盘扇区数据读到指定内存

| IO地址 | 功能 |
| --- | --- |
| 0x1f0 | 读数据，当0x1f7不为忙状态时，可以读。|
| 0x1f2 | 要读写的扇区数 |
| 0x1f3 | 若是LBA模式，则为LBA的0-7位 |
| 0x1f4 | 若是LBA模式，则为LBA的8-15位 |
| 0x1f5 | 若是LBA模式，则为LBA的16-23位 |
| 0x1f6 | 第0-3位，若是LBA模式，则为LBA的24-27位；第4位，主盘为0，从盘为1 |
| 0x1f7 | 状态和命令寄存器。操作时先给命令，再读取，如果不是忙状态就从0x1f0端口读数据 |

**bootloader是如何加载ELF格式的OS？**
> 1. 从硬盘读取第一页(4KB)到内存0x10000处
> 2. 检查是否为ELF格式，即检查开头四个字节的magic number是否为0x7f 45 4c 4d
> 3. 根据ELF header找到program header，然后逐个加载各个段
> 4. 根据ELF header中的入口信息，找到内核入口，并执行

# 练习5
> 实现函数调用跟踪堆栈
格式：
```
ebp:0x00007b28 eip:0x00100992 args:0x00010094 0x00010094 0x00007b58 0x00100096
    kern/debug/kdebug.c:305: print_stackframe+22
```

**代码：**
```c
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
      uint32_t ebp;
      uint32_t eip;
      uint32_t* p_args;

      ebp = read_ebp();
      eip = read_eip();
      
      int i;
      int j;
      for(i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i ++){
          cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
          p_args = (uint32_t*)ebp + 2;
          for(j = 0; j < 4; j ++){
            cprintf("0x%08x ", p_args[j]);
          }
          cprintf("\n");
          print_debuginfo(eip - 1);
          eip = *((uint32_t *)(ebp + 4));
          ebp = *((uint32_t *)ebp);
      }
}
```


# 练习6
> 完善中断初始化和处理

**1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？**

> 中断描述符一个表项占8个字节，第三四个字节为段选择子，最低的两个字节和最高的两个字节构成偏移值

**2. 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。**
```c
void
idt_init(void) {
     /* LAB1 YOUR CODE : STEP 2 */
     /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
      *     All ISR's entry a
      extern uintptr_t __vectors[];ddrs are stored in __vectors. where is uintptr_t __vectors[] ?
      *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
      *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
      *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which will be used later.
      * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
      *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup each item of IDT
      * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'lidt' instruction.
      *     You don't know the meaning of this instruction? just google it! and check the libs/x86.h to know more.
      *     Notice: the argument of lidt is idt_pd. try to find it!
      */

      // uintptr_t __vectors[] is defined in vectors.S, so we refer to it using the keyword 'extern'.
      extern uintptr_t __vectors[];
      // the length of IDT.
      const uint32_t length = sizeof(idt) / sizeof(struct gatedesc);
      // Setup the entries of ISR in IDT.
      uint32_t i;
      for(i = 0; i < length; i ++){
          SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
      }
      SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_KERNEL);
      // Tell the CPU where and how long is the IDT,
      // i.e., load IDT's base address and limit into IDTR.
      lidt(&idt_pd);
}
```

**3. 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字“100 ticks”。**
```c
static void
trap_dispatch(struct trapframe *tf) {
    char c;

    switch (tf->tf_trapno) {
    case IRQ_OFFSET + IRQ_TIMER:
        /* LAB1 YOUR CODE : STEP 3 */
        /* handle the timer interrupt */
        /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
         * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
         * (3) Too Simple? Yes, I think so!
         */
        ticks ++;
        if (ticks % TICK_NUM == 0) {
            print_ticks();
        }
        break;
    case IRQ_OFFSET + IRQ_COM1:
        c = cons_getc();
        cprintf("serial [%03d] %c\n", c, c);
        break;
    case IRQ_OFFSET + IRQ_KBD:
        c = cons_getc();
        cprintf("kbd [%03d] %c\n", c, c);
        break;
    //LAB1 CHALLENGE 1 : YOUR CODE you should modify below codes.
    case T_SWITCH_TOU:
    case T_SWITCH_TOK:
        panic("T_SWITCH_** ??\n");
        break;
    case IRQ_OFFSET + IRQ_IDE1:
    case IRQ_OFFSET + IRQ_IDE2:
        /* do nothing */
        break;
    default:
        // in kernel, it must be a mistake
        if ((tf->tf_cs & 3) == 0) {
            print_trapframe(tf);
            panic("unexpected trap in kernel.\n");
        }
    }
}
```

# 扩展练习 1
> 增加syscall，即增加一用户态函数，当内核态初始化完毕后，可以从内核态返回到用户态的函数，而用户态的函数又通过系统
> 调用得到内核态的服务。

```
当trap发生时，会在栈上保存相应的寄存器里的信息，以便处理完trap后恢复。

现在我们来分析下当特权级变化中断发生时栈的变化情况，先分析特权态到用户态的转变：

int中断使得eflags、cs、eip被压栈（注意，这里ss、esp并没有被压栈，因为CPL并没有发生变化，但是之后要用上，所以这里要空出两个位置以备用）
|            |
|            |
|   eflags   |
|   cs       |
|   eip      | <----- esp

然后操作系统根据中断向量号和IDTR，查找中断向量表，找到中断例程，并跳到相应的中断例程执行。
中断例程中又压入了错误信息和中断向量号，此时的栈看起来是这样的：
|            |
|            |
|   eflags   |
|   cs       |
|   eip      |
|   err      |
|   num      | <----- esp

之后，跳到中断处理的通用方法(__alltraps)中，又继续压入了其它寄存器的值：
|            |
|            |
|   eflags   |
|   cs       |
|   eip      |
|   err      |
|   num      | 
|   ds       |
|   es       |
|   fs       |
|   gs       |
|   eax      |
|   ecx      |
|   edx      |
|   ebx      |
|   esp      | # 此esp没用
|   ebp      |
|   esi      |
|   edi      | <----- esp

接着，又压入了esp的值作为参数传递给trap函数，此时的esp指向的栈正好对应上trapframe结构体：
|            |
|            |
|   eflags   |
|   cs       |
|   eip      |
|   err      |
|   num      | 
|   ds       |
|   es       |
|   fs       |
|   gs       |
|   eax      |
|   ecx      |
|   edx      |
|   ebx      |
|   esp      | # 此esp没用
|   ebp      |
|   esi      |
|   edi      | <--| # tf 指向这里
|   esp      |  --|

紧接着，调用了trap函数，后面的栈的变化就不用细看了，因为我们已经得到这个trapframe结构体了。
接下来的任务就是修改I/O特权级和段寄存器了。
    stack                                           switchk2u
|            |                                  |   ss       | --> USER_DS
|            | <------------------------------- |   esp      |                                  
|   eflags   |                                  |   eflags   | --> IOPL=3
|   cs       |                                  |   cs       | --> USER_CS
|   eip      |                                  |   eip      |
|   err      |                                  |   err      |
|   num      |                                  |   num      |
|   ds       |                                  |   ds       | --> USER_DS
|   es       |                                  |   es       | --> USER_DS
|   fs       |                                  |   fs       |
|   gs       |                                  |   gs       |
|   eax      |                                  |   eax      |
|   ecx      |                                  |   ecx      |
|   edx      |                                  |   edx      |
|   ebx      |                                  |   ebx      |
|   esp      | # 此esp没用                       |   esp      |
|   ebp      |                                  |   ebp      |
|   esi      |                                  |   esi      |
|   edi      | # tf 指向这里                |--> |   edi      |
|   esp      | ----------------------------|  
|   ...      |                        

然后就是一路出栈，将保存在switchk2u里的内容弹出到相应的寄存器中，最后iret的时候需要注意，此时CPL=0，DPL=3，发生了切换，所以会继续弹出ss和esp。

最后，movl %ebp, %esp，是将esp指向lab1_switch_to_user函数栈帧开始处，使得函数能正常的退回到上一个栈帧。

int之前寄存器的状态：
eax            0x1e	30
ecx            0x0	0
edx            0x103627	1062439
ebx            0x10094	65684
esp            0x7b90	0x7b90
ebp            0x7b98	0x7b98
esi            0x10094	65684
edi            0x0	0
eip            0x1001d1	0x1001d1 <lab1_switch_to_user+6>
eflags         0x206	[ PF IF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x23	35
gs             0x23	35

int之后寄存器的状态：
eax            0x1e	30
ecx            0x0	0
edx            0x103627	1062439
ebx            0x10094	65684
esp            0x7b80	0x7b80
ebp            0x7b98	0x7b98
esi            0x10094	65684
edi            0x0	0
eip            0x1022c5	0x1022c5 <vector120+2>
eflags         0x6	[ PF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x23	35
gs             0x23	35

iret之前寄存器的状态：
eax            0x1e	30
ecx            0x0	0
edx            0x103627	1062439
ebx            0x10094	65684
esp            0x10f958	0x10f958 <switchk2u+56>
ebp            0x7b98	0x7b98
esi            0x10094	65684
edi            0x0	0
eip            0x101e9a	0x101e9a <__trapret+10>
eflags         0x2	[ ]
cs             0x8	8
ss             0x10	16
ds             0x23	35
es             0x23	35
fs             0x23	35
gs             0x23	35

iret之后寄存器的状态：
eax            0x1e	30
ecx            0x0	0
edx            0x103627	1062439
ebx            0x10094	65684
esp            0x7b90	0x7b90
ebp            0x7b98	0x7b98
esi            0x10094	65684
edi            0x0	0
eip            0x1001d3	0x1001d3 <lab1_switch_to_user+8>
eflags         0x3206	[ PF IF #12 #13 ]
cs             0x1b	27
ss             0x23	35
ds             0x23	35
es             0x23	35
fs             0x23	35
gs             0x23	35
```

```
刚才已经分析了从内核态到用户态的转换，现在来分析从用户态到内核态的转换，思路是差不多的。

int发生时，CPL=3,DPL=0，会发生特权级的转换，所以会压入esp、ss、eflags、cs和eip。
|   ss       |
|   esp      |
|   eflags   |
|   cs       |
|   eip      |
|   err      |
|   num      | <----- esp

之后的步骤和前面一样
|   ss       |
|   esp      |
|   eflags   |
|   cs       |
|   eip      |
|   err      |
|   num      | 
|   ds       |
|   es       |
|   fs       |
|   gs       |
|   eax      |
|   ecx      |
|   edx      |
|   ebx      |
|   esp      | # 此esp没用
|   ebp      |
|   esi      |
|   edi      | <--| # tf 指向这里
|   esp      |  --|

紧接着，调用了trap函数，后面的栈的变化就不用细看了，因为我们已经得到这个trapframe结构体了。
接下来的任务就是修改I/O特权级和段寄存器了。
首先，esp中保存着原来的栈的位置，然后往下移动trapframe去掉ss和esp大小的内存，iret的时候没有发生特权级的切换，所以
用不着这两个，所以不用拷贝。
    temp                                             stack
|   ss       |                                  |            |
|   esp      | -------------------------------> |            |                                  
|   eflags   | --> IOPL=0                       |            | 
|   cs       | --> KERNEL_CS                    |            | 
|   eip      |                                  |            |
|   err      |                                  |            |
|   num      |                                  |            |
|   ds       | --> KERNEL_DS                    |            | 
|   es       | --> KERNEL_DS                    |            | 
|   fs       |                                  |            |
|   gs       |                                  |            |
|   eax      |                                  |            |
|   ecx      |                                  |            |
|   edx      |                                  |            |
|   ebx      |                                  |            |
|   esp      | # 此esp没用                       |            |
|   ebp      |                                  |            |
|   esi      |                                  |            |
|   edi      | # tf 指向这里                |--> |            | <-- switchu2k
|   esp      | ----------------------------|
|   ...      |   

然后把数据拷贝回原来的栈上，即切换之前的栈上：
    temp                                             stack
|   ss       |                                  |            |
|   esp      | -------------------------------> |            |                                      
|   eflags   | --> IOPL=0                       |   eflags   | 
|   cs       | --> KERNEL_CS                    |   cs       | 
|   eip      |                                  |   eip      |
|   err      |                                  |   err      |
|   num      |                                  |   num      |
|   ds       | --> KERNEL_DS                    |   ds       | 
|   es       | --> KERNEL_DS                    |   es       | 
|   fs       |                                  |   fs       |
|   gs       |                                  |   gs       |
|   eax      |                                  |   eax      |
|   ecx      |                                  |   ecx      |
|   edx      |                                  |   edx      |
|   ebx      |                                  |   ebx      |
|   esp      | # 此esp没用                       |   esp      |
|   ebp      |                                  |   ebp      |
|   esi      |                                  |   esi      |
|   edi      | # tf 指向这里                |--> |   edi      | <-- switchu2k
|   esp      | ----------------------------|
|   ...      |      

最后，恢复堆栈。

int之前寄存器的状态：
eax            0x1e	30
ecx            0x0	0
edx            0x103647	1062471
ebx            0x10094	65684
esp            0x7b98	0x7b98
ebp            0x7b98	0x7b98
esi            0x10094	65684
edi            0x0	0
eip            0x1001da	0x1001da <lab1_switch_to_kernel+3>
eflags         0x3206	[ PF IF #12 #13 ]
cs             0x1b	27
ss             0x23	35
ds             0x23	35
es             0x23	35
fs             0x23	35
gs             0x23	35

int之后寄存器的状态：
eax            0x1e	30
ecx            0x0	0
edx            0x103647	1062471
ebx            0x10094	65684
esp            0x10fd68	0x10fd68 <stack0+1000>
ebp            0x7b98	0x7b98
esi            0x10094	65684
edi            0x0	0
eip            0x1022ce	0x1022ce <vector121+2>
eflags         0x3006	[ PF #12 #13 ]
cs             0x8	8
ss             0x10	16
ds             0x23	35
es             0x23	35
fs             0x23	35
gs             0x23	35

iret之前寄存器的状态：
eax            0x1e	30
ecx            0x0	0
edx            0x103647	1062471
ebx            0x10094	65684
esp            0x7b8c	0x7b8c
ebp            0x7b98	0x7b98
esi            0x10094	65684
edi            0x0	0
eip            0x101e9a	0x101e9a <__trapret+10>
eflags         0x3002	[ #12 #13 ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x23	35
gs             0x23	35

iret之后寄存器的状态：
eax            0x1e	30
ecx            0x0	0
edx            0x103647	1062471
ebx            0x10094	65684
esp            0x7b98	0x7b98
ebp            0x7b98	0x7b98
esi            0x10094	65684
edi            0x0	0
eip            0x1001dc	0x1001dc <lab1_switch_to_kernel+5>
eflags         0x206	[ PF IF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x23	35
gs             0x23	35
```

```c
// init.c
static void
lab1_switch_to_user(void) {
    //LAB1 CHALLENGE 1 : TODO
    asm volatile (
        "sub 0x8, %%esp \n"
        "int %0 \n"
        "movl %%ebp, %%esp \n"
        :
        : "i" (T_SWITCH_TOK)
    );
}

static void
lab1_switch_to_kernel(void) {
    //LAB1 CHALLENGE 1 :  TODO
    asm volatile (
        "int %0 \n"
        "movl %%ebp, %%esp \n"
        :
        : "i" (T_SWITCH_TOU))
    );
}
```

```c
// trap.c
static void
trap_dispatch(struct trapframe *tf) {
    char c;

    switch (tf->tf_trapno) {
    case IRQ_OFFSET + IRQ_TIMER:
        /* LAB1 YOUR CODE : STEP 3 */
        /* handle the timer interrupt */
        /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
         * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
         * (3) Too Simple? Yes, I think so!
         */
        ticks ++;
        if (ticks % TICK_NUM == 0) {
            print_ticks();
        }
        break;
    case IRQ_OFFSET + IRQ_COM1:
        c = cons_getc();
        cprintf("serial [%03d] %c\n", c, c);
        break;
    case IRQ_OFFSET + IRQ_KBD:
        c = cons_getc();
        cprintf("kbd [%03d] %c\n", c, c);
        break;
    //LAB1 CHALLENGE 1 : YOUR CODE you should modify below codes.
    case T_SWITCH_TOU:
        if (tf->tf_cs != USER_CS) {
            switchk2u = *tf;
            switchk2u.tf_cs = USER_CS;
            switchk2u.tf_ds = switchk2u.tf_es = switchk2u.tf_ss = USER_DS;
            switchk2u.tf_esp = (uint32_t)tf + sizeof(struct trapframe) - 8;
		
            // set eflags, make sure ucore can use io under user mode.
            // if CPL > IOPL, then cpu will generate a general protection.
            switchk2u.tf_eflags |= FL_IOPL_MASK;
		
            // set temporary stack
            // then iret will jump to the right stack
            *((uint32_t *)tf - 1) = (uint32_t)&switchk2u;
        }
        break;
    case T_SWITCH_TOK:
        if (tf->tf_cs != KERNEL_CS) {
            tf->tf_cs = KERNEL_CS;
            tf->tf_ds = tf->tf_es = KERNEL_DS;
            tf->tf_eflags &= ~FL_IOPL_MASK;
            switchu2k = (struct trapframe *)(tf->tf_esp - (sizeof(struct trapframe) - 8));
            memmove(switchu2k, tf, sizeof(struct trapframe) - 8);
            *((uint32_t *)tf - 1) = (uint32_t)switchu2k;
        }
        break;
    case IRQ_OFFSET + IRQ_IDE1:
    case IRQ_OFFSET + IRQ_IDE2:
        /* do nothing */
        break;
    default:
        // in kernel, it must be a mistake
        if ((tf->tf_cs & 3) == 0) {
            print_trapframe(tf);
            panic("unexpected trap in kernel.\n");
        }
    }
}
```