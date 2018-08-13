# os

《Orange's 一个操作系统的实现（第二版）》学习笔记

# 在 ubuntu 上构建实验环境

本人用的操作系统是 ubuntu，所以所有的实验都是在 ubuntu 系统进行的，下面介绍
一下如何在 ubuntu 系统上构建实验环境。

操作系统的版本是 16.04.4，需要用到的软件有：

1. nasm。nasm 是一个汇编器, 用来编译我们的汇编程序，选择 nasm 是因为 nasm 的语法友好。
2. bochs。bochs 是虚拟计算机，用来调试我们的程序。
3. freedos。方便调试 COM 程序。

## 安装 nasm

```bash
sudo apt install nasm
```

安装好后会提供 `nasm` 命令和 `ndisasm` 命令，用来编译和反编译。

## 安装 bochs

```
sudo apt install bochs bochs-x
```

## 安装 freedos

在 bochs 官网下载，解压后在解压目录执行 `bochs` 即可。
下载地址：[http://bochs.sourceforge.net/guestos/freedos-img.tar.gz](http://bochs.sourceforge.net/guestos/freedos-img.tar.gz)

## 第一个 os （仅仅是引导程序）

首先新建一个名字为 os（可以换成任何你喜欢的名字）的目录。目录结构如下：

```bash
├── bochsrc  # bochs 的配置文件
├── boot.asm  # 引导程序汇编代码
├── boot.bin  # 引导程序二进制代码
└── os.img  # bochs 使用的映像文件
```

#### 创建一个软盘映像（os.img）。

```bash
pan@ubu:~/os$ bximage
========================================================================
                                bximage
                  Disk Image Creation Tool for Bochs
          $Id: bximage.c 11315 2012-08-05 18:13:38Z vruppert $
========================================================================

Do you want to create a floppy disk image or a hard disk image?
Please type hd or fd. [hd] fd

Choose the size of floppy disk image to create, in megabytes.
Please type 0.16, 0.18, 0.32, 0.36, 0.72, 1.2, 1.44, 1.68, 1.72, or 2.88.
 [1.44] 
I will create a floppy image with
  cyl=80
  heads=2
  sectors per track=18
  total sectors=2880
  total bytes=1474560

What should I name the image?
[a.img] os.img

Writing: [] Done.

I wrote 1474560 bytes to os.img.

The following line should appear in your bochsrc:
  floppya: image="os.img", status=inserted
```

#### 配置 bochs（bochsrc）。

新建一个名为 bochsrc 的文件，其内容如下：

```
###############################################################
# Configuration file for Bochs
###############################################################

# how much memory the emulated machine will have
megs: 32

 # filename of ROM images
romimage: file=/usr/share/bochs/BIOS-bochs-latest
vgaromimage: file=/usr/share/vgabios/vgabios.bin

# what disk images will be used
floppya: 1_44=os.img, status=inserted

# choose the boot disk.
boot: floppy

# where do we send log messages?
log: bochsout.txt
```

#### 编写引导程序

boot.asm

```ASM
org 7c00h  ; 告诉编译器程序加载到 7c00 处
mov ax, cs
mov ds, ax
mov es, ax
call DispStr  ; 调用显示字符串函数
jmp $  ; 无限循环

DispStr:
    mov ax, bootMessage
    mov bp, ax  ; ES:BP = 串地址
    mov cx, msgLen  ; CX = 串长度
    mov ax, 1301h  ; AH = 13, AL = 01
    mov bx, 000ch  ; 页号为 0 (BH = 0) 黑底红字（BL = 0ch, 高亮）
    mov dl, 0
    int 10h  ; 10h 号中断
    ret

bootMessage: db "Hello, OS world!"
msgLen: equ $ - bootMessage

times 510 - ($ - $$) db 0  ; 填充剩下的空间，使生成的二进制代码恰好为 512 字节
dw 0xaa55  ; 结束标志
```

一些说明：

在 NASM 中，任何不被方括号括起来的标签或变量名都被认为是地址，访问标签中的
内容必须使用 []。所以

```
mov ax, bootMessage
```

将会把 "Hello, OS world!" 这个字符串的首地址传给寄存器 ax。又比如，如果有：

```
foo dw 1
```

则 `mov ax, foo` 将把 foo 的地址传给 ax，而 `mov bx, [foo]` 将把 bx 的值赋值为 1。
并且，在 NASM 中，变量和标签是一样的，也就是说：

```
foo dw 1
```

等价于

```
foo: dw 1
```

$ 表示当前行被汇编后的地址。$$ 表示一个节的开始处被汇编后的地址。在这里，我们
的程序只有一个节，所以 $$ 实际上就表示程序被编译后的开始地址，也就是 0x7c00。

编译

```
nasm boot.asm -o boot.bin
```

将二进制代码写入软盘映像

```
dd if=boot.bin of=os.img bs=512 count=1 conv=notrunc
```

在 os 目录执行

```
bochs
```

这时会停在那里，方便使用者进行调试。在这里我们就不调试了，直接输入
continue 或 c 继续运行即可，如下所示：

```
<bochs:1> c
```

如果没什么问题的话应该能看到红色的 **Hello, OS world!** 字样。

## 使用 freedos

#### 为什么要用 freedos

《Orange's 一个操作系统的实现》对此的解释大意如下：

> 引导扇区空间有限，只有 512 字节，如果我们的程序超过了 512 字节，就不行了。要
解决这个问题有两个方法：一是写一个引导扇区，但是对目前的我们来说难度太大，虽然
可以把别人写好的直接拿来用，但是这方面教程比较匮乏；二是我们把程序编译成 COM
文件，让 dos 来运行它，优点是很多保护模式的教程是基于 dos 讲的，大家有不明白
的可以参考其他教程。

#### 将文件拷贝到 freedos

freedos 下载解压后的目录结构类似于下面的样子：

```
├── bochsrc
├── a.img
├── b.img
└── c.img
```

我们可以将源程序拷贝到 c.img。之所以拷到 c.img 是因为 c.img 的空间比较大。可以
用 `fdisk -l c.img` 查看一下，输出如下所示：

```
Disk c.img: 10.2 MiB, 10653696 bytes, 20808 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe3657373

设备       启动 Start 末尾 扇区 Size Id 类型
c.img1     *       17 20484 20468  10M  1 FAT12
```

第一行显示的有大小信息，可以看到为 10.2 MB。

拷贝前需要先挂载分区，挂载方法如下所示：

```
sudo mkdir /mnt/dos
# loop 用来把文件当成硬盘分区
# offset = Start * Sector size
# Start 和 Sector size 信息可以用 fdisk 命令查看，参考上面的 fdisk -l c.img。
sudo mount c.img /mnt/dos -o loop,offset=8704
```

挂载后执行 `sudo cp -R your_program /mnt/dos` 将你的程序拷贝过去即可。然后执行
`sudo umount /mnt/dos` 卸载分区。
