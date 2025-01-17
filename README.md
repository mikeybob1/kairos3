# 程序加载

## 一、实验介绍

我们在linux中可以从文件系统中加载可执行文件并运行，例如`a.out`：

```shell
$ ./a.out
```

这个实验你将亲自动手解析ELF格式可执行文件，了解用户程序是如何被操作系统加载执行，以及在整个过程中所牵涉到的关键OS功能模块。

## 二、实验目的

通过本次实验，你将会学习到：

- 执行一个可执行文件的基本流程。
- 简单的静态链接ELF文件解析过程。
- 执行程序所牵涉到的OS功能模块的基本概念。

## 三、实验要求

1. 掌握ELF的基本结构，理解ELF文件的两种视图。
2. 理解程序的加载过程和生命周期。
3. 完成实验点，并且成功提交。

## 四、实验步骤

### 1. 查看内核ELF

我们编译出来的内核本身其实就是个ELF文件格式，它被存放在`build/kernel`，我们可以通过`readelf`工具来读取它：

```
$ make kernel # 构建内核
$ readelf -lh build/kernel
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           RISC-V
  Version:                           0x1
  Entry point address:               0x80020000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          920048 (bytes into file)
  Flags:                             0x4, double-float ABI
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         1
  Size of section headers:           64 (bytes)
  Number of section headers:         17
  Section header string table index: 16

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000001000 0x0000000080020000 0x0000000080020000
                 0x0000000000054a40 0x0000000000054a40  RWE    0x1000

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata .data .bss 
```

> `readelf -lh`是什么意思？跳出来的一大堆又是什么？？ 为什么不问问~~神奇海螺~~ChatGPT呢！

> 此外，你还可以自己编写一个HelloWorld.c并使用gcc来编译它，再使用`readelf`来查看相关字段，看看有何异同？如果在编译选项中加入`--static`选项呢？

在这，我们需要关注的是如下几个字段：

- OS/ABI: UNIX - System V
- Machine: RISC-V
- Entry point address: 0x80020000
- Program Headers: ……

第一个字段指定了二进制程序使用的ABI(Application Binary Interface)标准，它规定了该可执行程序的运行环境（初始参数：argc、argv，栈结构，环境变量传递等等），如果你对此感兴趣，那么具体可以参考[System V ABI - OSDev Wiki](https://wiki.osdev.org/System_V_ABI)。

第二个字段指定了该二进制可执行程序的指令集架构。

第三个字段指定了该二进制可执行程序的**入口地址**。

第四个字段则罗列了该程序的所有段（Segment），上面内核的段较为特殊，只有一个，正常情况下一个程序的段可能有好几个，可以分为数据段、代码段等等，分别还可以指定不同的读写执行权限。当程序被加载执行时，程序中所描述的这些段，将被操作系统从文件系统加载进内存中。

### 3. 实现加载

我们已经为你构建了一个加载框架`src/kernel/exec.c::exec`：

```c
(line: 82)
for (i = 0, off = elf.phoff; i < elf.phnum; i++, off += sizeof(ph)) {
	if (reade(ep, 0, (uint64_t)&ph, off, sizeof(ph)) != sizeof(ph)) {
		eunlock(ep);
		goto bad;
	}
	
	if (ph.type != PT_LOAD)
		continue;
		
	if (ph.memsz < ph.filesz) {
		eunlock(ep);
		goto bad;
	}

	// TODO:
	// 1. map a memrory area
	// 2. load ELF segment to the area
}
```

这个循环将会遍历`Program Header`，将所有标记为`Load`的段加载进内存当中。

首先，1. 你需要为目标段创建一片描述这个段的`Memory Area`，接着，2. 你需要将该段所在文件偏移处的内容加载进这片`Memory Area`。

> *一些提示*
> 
> 1) 可以使用`mmap_map`接口来创建`Memory Area`，使用`loadseg`函数来加载程序段，当前，你可以将文件系统的`entry`理解为该文件在内存中的对象实例。
> 2) 你需要充分理解上述`readelf`中`Program Header`中各个字段的含义，他们可以帮助你在文件中定位段的位置以及在虚拟内存中的地址。
> 3) 当上述接口返回错误时，你需要及时处理，处理过程可以参考`exec`的它处实现。

## 五、思考

1. 除了`Load`类型的程序段以外，还有什么类型的段？他们的作用为何？
2. 上述我们实现的是程序的静态加载，那动态加载呢？它又是如何实现的呢？