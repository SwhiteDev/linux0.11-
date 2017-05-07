# ***内核编程语言和环境***
----------------------------------------------------
## ***as86与GNU as汇编***
----------------------------------------------------
对于汇编这种语言，相信所有同胞们都是望而却步。然而由于操作系统许多关键代码要求很高的执行速度和效率，因此在系统源码中通常会有10%左右的起关键作用的汇编语言。linux的32位初始化代码、所有中断和异常处理、许多宏定义都是用汇编或嵌入汇编。
系统内汇编代码分为两种，一种是as86汇编器用于编译内核中的boot/bootsect.s引导山区程序和boot/setup.s设置程序；其余汇编包括C语言产生的汇编程序均使用gas编译，这里我们只简单介绍一下gas的用法。以后遇到汇编的代码部分会详细说明一下语法
`命令格式：as [选项] [-o objfile] [srcfile]`
## ***链接器、区与重定位***
----------------------------------------------------
在C语言——关于链接的思考和程序运行时的数据结构那两篇文章中已经稍微介绍了一下目标文件结构和链接器的一些知识。这里再次补充。

在一个目标文件中(上面汇编出来的)会存在至少三个区：text、data、bss区，链接器ld会将这些区按照一定规律组合起来变成一个可执行文件，当存在不止一个目标文件时，例如 a.out 和 b.out，ld会将 a 和 b 的text段放在一起、data段放在一起、bss段放一起。

当然ld也会把一些**重定位信息**放进去以便得知程序的哪些数据会变化或者哪些数据该修改如何修改，所以为了执行重定位操作，每次在目标文件中涉及地址时，ld必须知道：

 - 目标文件中对一个地址的引用是从何算起
 - 地址的字节长度是多少
 - 该地址引用的是哪个区，区开始地址多少
 - 是否与PC(程序计数器)有关
 
--------------------------------------------------
### ***链接器涉及的区***
--------------------------------------------------
text、data、bss段就不必多写了，老生常谈。
需要提一下的是**absolute段 **：该区的0地址总是重定位到运行时刻的0地址处，如果你不想ld在重定位操作时改变你所引用的地址，那请使用这个段(我们叫他不可重定位段)。
### 符号
**符号(Symbol)**在编译链接过程中是个很重要的概念，程序员使用符号来命名对象，链接器使用符号进行链接操作，调试器利用符号进行调试(符号类型属性详见include/a.out.h)

 - 符号名以一个一字母或“._”开始
 -  除了名字以外符号还有值和属性，若不定义就使用，其属性为0，指示该符号是一个外部定义的符号。
 -  符号值一般32位，其值是从区开始到该标号处的地址值，对于text、data、bss段，一个符号的值通常会在链接过程中由于ld改变段记得基地址而改变。

**标号(Lable)**是后面紧随一个冒号的符号，此时标号代表活动位置计数器的当前值，也可作为指令的操作数使用，例如使用“＝”给其赋值。

--------------------------------------------------------------
## ***C语言程序***
---------------------------------------------
C程序的编译与链接过程就不在这里赘述了，参照C语言的系列文章，下面来说一下C里面嵌入汇编。

--------------------------------------------------
### ***嵌入汇编***
---------------------------------------------------
具有输入和输出参数的嵌入汇编语句基本格式：
`asm("汇编语句"`
`: 输出寄存器`
`: 输入寄存器`
`: 会被修该的寄存器);`
除了第一行外，后面带冒号的行若不使用都可以省略。其中“输出寄存器”表示当这段嵌入汇编执行结束后，哪些寄存器用于存放输出数据。这些寄存器会分别对应C表达式值或一个内存地址；“输入寄存器”表示在开始执行汇编代码时，这里指定一些寄存器中应存放的输入值，对应C变量或常数值。“会被修改寄存器”表示你已对其中列出的寄存器中的值进行了改动，gcc编译器不能再依赖于它原先对这些寄存器加载的值，必要的话gcc需重新加载这些寄存器。
下面是kernel/traps.c中的一段代码：

```
01 #define get_seg_byte(seg, addr) \
02 ({ \
03 register char _res; \		//定义一个寄存器变量_res
04 _asm_("push %%fs; \	        //先保存fs寄存器原值(段选择符)
05        mov  %%ax,%%fs; \     //然后用seg设置fs
06        movb %%fs:%2,%%al; \  //取seg:addr处1字节到al寄存器
07        pop  %%fs" \          //恢复fs寄存器原内容
08        :"=a" (_res) \        //输出寄存器列表
09        :"0" (seg),"m" (*(addr)); \   //输入寄存器列表
10 _res;})
```
这里需要解释的就第6行和8、9行：gcc将输入输出寄存器编号从0开始，也就是说8行为0，9行前半部分为1，后半部分为2。所以说第6行的"%2"就是(*(addr))偏移地址。第8、9行中的a、m是加载码(常用加载码在最下面附上)，而"0"就代表输出寄存器中的eax。
**寄存器变量**：如果想在汇编中把汇编指令的输出直接写到指定寄存器，那此时使用局部寄存器变量就很方便：
`register int res _asm_("ax")`
这里ax就是变量res所希望使用的寄存器。

--------------------------------------------------
##***C与汇编相互调用***
--------------------------------------------------
**栈**是用来传递函数参数、存储返回信息、临时保存寄存器原值以及存储局部数据。而单个函数所使用的栈称为**栈帧**。栈指针一般用esp，帧指针一般用ebp。
大家都知道main()是C程序中的第一个函数。没错，起始main()也是个被调用的函数，调用者是**crt0.s**(C run-time)是一个桩程序，该程序目标文件被链接在每个程序开始部分。
> 大家可能在自己系统库中怎么也找不到crt0.s，因为现在gcc已将其分为：crt1.o、crti.o、crtbegin.o、crtend.o和crtn.o。链接顺序为crt1、i、begin.o、所有程序模块、crtend、n.o、库模块文件。

--------------------------------------------------
### ***汇编中调用C***
---------------------------------------------
 首先按照逆向顺序将函数参数入栈(最右参数先入栈)，然后执行CALL指令执行被调函数。调用函数返回后，程序需把先前入栈的参数清除。
 

```
//  kernel/system_call.s汇编程序_sys_fork部分
212 	push  %gs
213 	pushl %esi
214 	pushl %edi
215 	pushl %ebp
216 	pushl %eax
217 	call  _copy_process   #调用C函数copy_process()(kernel/fork.c,68)
218 	addl  $20,%esp 	   #丢弃压栈内容
219 1:  ret
```

```
// kernel/fork.c
68 int copy_processs(int nr,long ebp,long esi,long gs,long none, 
					 long ebx,long ecx,long edx,
					 long fs,long es,long ds,
					 long eip,long cs,long eflags,long esp,long ss)
```

--------------------------------------------------
### ***C中调用汇编***
--------------------------------------------------
```
/**
 利用系统调用sys_write()实现显示函数int mywrite(int fd, char *buf, int count)。
 */
SYSWRITE = 4   		#sys_write()系统调用号
.global _mywrite, _myadd
.text
_mywrite:
	pushl %ebp
	movl  %esp, %ebp
	pushl %ebx
	movl 8(%ebp), %ebx   #取三个参数
	movl 12(%ebp),%ecx
	movl 16(%ebp),%edx
	movl $SYSWRITE, %eax #%eax放入系统调用号4
	int  $0x80           #执行系统调用
	popl  %ebx
	movl  %ebp, %esp
	popl  %ebp
	ret
_myadd:
	...
	ret
```

```
int main()
{
	...
	mywrite(1, mystr, strlen(mystr));
	...
}
```

--------------------------------------------------
## ***Linux 0.11目标文件格式***
-----------------------------------------------
![这里写图片描述](http://img.blog.csdn.net/20170507195637179?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzAxMDI4ODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 - **执行头部**：含有目标文件的整体结构信息，内核使用这些参数将可执行文件加载到内存执行，链接程序ld用这些参数将模块组合成可执行文件。
 

```
struct exec {
	unsigned long a_magic    //执行文件魔数，使用N_MAGIC等宏访问。
	unsigned a_text          //代码长度，字节数
	unsigned a_data          //数据长度，字节数
	unsigned a_bss           //未初始化数据段长度，字节数
	unsigned a_syms          //符号表长度，字节数
	unsigned a_entry         //执行开始地址
	unsigned a_trsize        //代码重定位信息长度，字节数
	unsigned a_drsize        //数据重定位信息长度，字节
}//详见include/a.out.h
```

 - **代码区**：程序二进制指令代码和数据信息(可以以只读方式加载)。
 - **数据区**：程序二进制指令代码和数据信息(初始化过的数据，总被加载到可读写内存)。
 - **代码/数据重定位部分**：供ld使用的记录数据；组合目标文件时用于定位代码/数据段中的指针或地址。当ld需要改变目标代码地址时就需要维护修正这些地方。
 

```
struct relocation_info{
	int r_address;               //段内需要重定位地址
	unsigned int r_symbolnum:24; //与r_extern有关。
	                             //指定符号表中一个符号或一个段
	unsigned int r_pcrel:1;      //1比特。PC相关标志
	unsigned int r_length:2      //2比特。指定被重定位字段长度
	unsigned int r_extern:1;     //外部标志位。
	                             //1:以符号值重定位
	                             //0:以段地址重定位
	unsigned int r_pad:4;        //未使用(最好复位掉)
}
```

 - **符号表部分**：也是供ld使用的记录数据；记录模块文件中定义的*全局符号以及需从其他文件中输入的符号，或由ld定义的符号用于模块之间对命名的变量和函数进行交叉引用*。
 - **字符串表部分**：包含一些调试信息。
 

```
//符号表和相应字符串表
struct nlist{
	union {
		char *n_name;         //字符串指针
		struct nlist *n_next; //或指向另一个符号项结构的指针
		long n_strx;          //或符号名在字符串表中的字节偏移值
	} n_un;
	unsigned char n_type;     //该字节分3个字段(详见a.out.h)
	char n_other;             //通常不用
	char n_desc;
	unsigned long n_value;    //符号的值
};
```

```
//n_type
#ifndef N_EXT        
#define N_EXT 1
#endif
#ifndef N_TYPE
#define N_TYPE 036
#endif
#ifndef N_STAB
#define N_STAB 0340
#endif
```
![这里写图片描述](http://img.blog.csdn.net/20170507205922179?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzAxMDI4ODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)	
a.out执行文件映射到进程逻辑地址空间

--------------------------------------------------
### ***链接程序输出***
--------------------------------------------------
链接程序对输入模块文件及相关库进行处理，生成大模块(库)或可执行文件过程中：

 1. 首要任务给输出文件进行存储空间分配
 
 ![这里写图片描述](http://img.blog.csdn.net/20170507210309013?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzAxMDI4ODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 2. 一旦存储位置确定，ld就可继续执行符号绑定和代码修正。
 > 编译建立内核过程中make利用too.s/build.c程序将bootsect、setup、system文件中的执行头去掉，顺序组合成Image内核映像
 
 
 ### ***链接程序预定义变量***
 --------------------------------------------------
 

```
#include <stdio.h>                                                                                                                                 
 
extern int end, etext, edata;
extern int _etext, _edata, _end;

int main()
{
     printf("&etext=%p, &edata=%p, &end=%p\n", \
		     &etext, &edata, &end);
     printf("&_etext=%p, &_edata=%p, &_end=%p\n", \
		     &etext, &edata, &end);
     return 0;
}

```
![这里写图片描述](http://img.blog.csdn.net/20170507212823033?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzAxMDI4ODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我这里为了突出ld预定义变量，将程序改写(去掉extern声明，即直接定义)：

![这里写图片描述](http://img.blog.csdn.net/20170507212925910?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzAxMDI4ODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

-------------------------------------------------------------------
### ***System.map文件***
---------------------------------------------
当运行GNU/gld(ld)使用'-M'选项时，或是使用nm命令，则会打印出链接映像(link map)信息，即由ld产生的目标程序neiucn地址映像信息。其中列出了程序段装入内存中的位置信息。

 - 目标文件及符号信息映射到内存的位置
 - 公共符号如何放置
 - 链接中包含的所有文件成员及其引用的符号
 
那好我们find一下这个System.map文件：

![这里写图片描述](http://img.blog.csdn.net/20170507213814359?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzAxMDI4ODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

没有搜到，估计是要重新编译内核才有的。还是看书上的：

```
c03441a0 B dmi_broken
c03441a4 B is_sony_vaio_laptop
c03441c0 b dmi_ident
```

 - 第1栏为符号值(地址)
 - 第2栏为符号类型。如果是小写的说明符号是局部的；大写说明是全局的(外部的)，参见上文n_type定义。
![这里写图片描述](http://img.blog.csdn.net/20170507222242501?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzAxMDI4ODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20170507222257673?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzAxMDI4ODk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 - 第3栏为对应符号名称

--------------------------------------
## ***make程序和Makefile文件***
------------------------------------------------
 - 相信大家或多或少了解一点make程序，make就是根据Makefile文件里面你写的一些规则、依赖、命令来编译整个工程，用过IDE(集成开发环境)的都熟悉一个快捷键，即一键编译，其实那个的背后就是这个make程序的变更版。
 - make好处有很多，可以节省编译时间，直接在命令行下***`make`***就行了(大工程需要先***`make config`***配置工程文件)，当然如果不嫌麻烦的话可以每次编译都写那么一大堆规则命令在阅读性上也可通过makefile可以知道整个工程的文件结构和依赖关系。
 - 我这里简单介绍一下使用规则，详细的可以看GNU make手册，或者陈皓写的《跟我一起写makefile》。
 

```
目标(target) : 依赖(prerequisites)
	命令(command)
#注意：命令左边一定是以一个'TAB'键开始的！
```

```
/**
 *超级简单的makefile模板，对于文件不多的小工程足够了
 *这个模板我没测试过，大概就这个样子
 */
 # 下面是一些定义的变量，使用方法和环境变量差不多
 SRC    = part_0.c part_1.c part_2.c
 OBJS   = *.o
 CC     = /home/swhite/arm-linux-gcc/gcc
 CFLAGS = -Wall -g -o
 LIBS   = -lsqlite3
 
 #如果想在过程中生成个test程序
 all  : prog test
 
 #下面的这4行也可以直接用两行，即不生成.o文件
 prog : $(OBJS)
	 $(CC) $(CFLAGS) prog $(OBJS) $(LIBS)
 $(OBJS) : $(SRC)
	 $(CC) -c $(SRC)

 test : test.c
	 $(CC) $(CFLAGS) test test.c
	 
 #伪目标clean，命令行下make clean可以清除相关中间文件和可执行文件
 clean :
	 rm $(OBJS) prog test
```
