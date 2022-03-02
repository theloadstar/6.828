# Lab1

## Part 1

首先，简要阅读一下[Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)，熟悉AT&T汇编语法。（<font color = "red">**Ex1.**</font>）

![lab_1_pc_layout](graph/lab_1_pc_layout.png)

​	一个典型的32位PC内存分布图如上所示。在最早的时候，PC普遍只有1MB的内存。其中初始的640KB被标记为“Low Memory”，是早期PC唯一能够使用的内存。余下的384KB为保留地址，有其特殊作用：例如BIOS、video显示的缓冲区等。BIOS负责执行基本的系统初始化，如激活显卡和检查已安装的内存数量。在执行此初始化之后，BIOS从某些适当的位置(如 floppy disk(软盘), hard disk, CD-ROM, or the network)加载操作系统，并将机器的控制权传递给操作系统。尽管现代的PC内存中其BIOS已经从Low Memory中提取出来，但之后的PC内存设计均保存了Low Memory以达到向后兼容的目的。

---

​	打开两个终端，均进入到`lab`目录。其中一个执行`make qemu-gdb`，另一个执行`make gdb`。![lab1_gdb](graph/lab1_gdb.png)

出现上面这张图的信息，说明存在并行的gdb服务（[ref](https://stackoverflow.com/questions/22373239/gdb-remote-debugging-fails-strangely))，关掉终端重开后👇：

![lab1_gdb_2](graph/lab1_gdb_2.png)

​	注意这里有一条关键的语句`[f000:fff0] 0xffff0: ljmp $0xf000,$0xe05b`。在解析这行代码之前，我们先简要讲一下两个概念。

1. 段寄存器：在AT&T中，存在不同的段，包括代码段CS、数据段DS、堆栈段SS、扩展段ES等，相应寄存器（寄存器名小写，同名）保留着对应段的初始地址。
2. 在实模式寻址中，指令的寻址有如下格式`physical address = 16 * segment + offset.`使用这样的方式是因为我们有1MB即20位的寻址空间，而寄存器只有16位。

回到上面那行指令`[f000:fff0] 0xffff0: ljmp $0xf000,$0xe05b`，以第二个分号为分界，前半部分表示这条指令的地址，后半部分为这条指令的内容；对于前半部分，括号内的即为`segment`和`offset`，计算出的结果即为括号外的内容`0xffff0`，该地址为BIOS结束前的16字节（这也解释了为什么第一条指令会往回跳：16byte的空间基本实现不了什么功能（那为什么不直接从0xfffff开始，空着16byte所为何事？））；对于后半部分，`ljmp`表示段间跳转，其中第一个寄存器为段基地址，第二个为段内偏移。按照实模式寻址的方法，计算出跳转的地址为`0xfe05b`，该地址即为下一条执行指令的地址。

<font color = "red">**Ex2**</font>：这里个人感觉就是体会一下`si`单步调试指令的用法，毕竟BIOS的代码太多太长了，一条条分析下去不现实。这里有几篇博客部分分了其中的代码，拿来熟悉AT&T汇编代码也很不错[博客1](https://www.cnblogs.com/fatsheep9146/p/5078179.html)，[博客2](https://blog.csdn.net/old_memory/article/details/79572498)。

此外，自己在`si`的时候发现一个细节：

![lab1_gdb_si](graph/lab1_gdb_si.png)

注意指令的地址，非跳转的相邻指令之间，并不是严格的相差4个字节，存在相差2字节、6字节的情况，这点与MIPS不同。推测是因为AT&T每条指令的长度并不是像MIPS一样固定的。

参考：

* [AT&T汇编指令总结](https://blog.csdn.net/zmcomputer/article/details/5908206)
* [AT&T汇编指令](https://www.cnblogs.com/jokerjason/p/9578646.html)
* [深入理解计算机系统(4) AT&T汇编基本指令](https://www.daimajiaoliu.com/daima/4ed9bac6910040c)

## Part 2

​		一般的硬盘会以512字节为一个扇区，当BIOS找到可启动的扇区（boot loader）时，会将装载到物理地址为`0x7c00`至`0x7dff`的内存中，并使用`jmp`命令跳转到`[0x0000:0x7c00]`的位置，以将控制权转移给该boot loader。目前的CD-ROM多使用2048字节为一个扇区，这里我们还是使用前者。

---

​	<font color = "red">boot.s源码阅读</font>

```
#include <inc/mmu.h>

# Start the CPU: switch to 32-bit protected mode, jump into C.
# The BIOS loads this code from the first sector of the hard disk into
# memory at physical address 0x7c00 and starts executing in real mode
# with %cs=0 %ip=7c00.

.set PROT_MODE_CSEG, 0x8         # kernel code segment selector
.set PROT_MODE_DSEG, 0x10        # kernel data segment selector
.set CR0_PE_ON,      0x1         # protected mode enable flag

.globl start
start:
  .code16                     # Assemble for 16-bit mode
  cli                         # Disable interrupts
  cld                         # String operations increment

  # Set up the important data segment registers (DS, ES, SS).
  xorw    %ax,%ax             # Segment number zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment

  # Enable A20:
  #   For backwards compatibility with the earliest PCs, physical
  #   address line 20 is tied low, so that addresses higher than
  #   1MB wrap around to zero by default.  This code undoes this.
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60

  # Switch from real to protected mode, using a bootstrap GDT
  # and segment translation that makes virtual addresses 
  # identical to their physical addresses, so that the 
  # effective memory map does not change during the switch.
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
  
  # Jump to next instruction, but in 32-bit code segment.
  # Switches processor into 32-bit mode.
  ljmp    $PROT_MODE_CSEG, $protcseg

  .code32                     # Assemble for 32-bit mode
protcseg:
  # Set up the protected-mode data segment registers
  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS
  movw    %ax, %ss                # -> SS: Stack Segment
  
  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call bootmain

  # If bootmain returns (it shouldn't), loop.
spin:
  jmp spin

# Bootstrap GDT
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULL				# null seg
  SEG(STA_X|STA_R, 0x0, 0xffffffff)	# code seg
  SEG(STA_W, 0x0, 0xffffffff)	        # data seg

gdtdesc:
  .word   0x17                            # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```

<font color = "red">main.c源码阅读</font>

```c
#include <inc/x86.h>
#include <inc/elf.h>

/**********************************************************************
 * This a dirt simple boot loader, whose sole job is to boot
 * an ELF kernel image from the first IDE hard disk.
 *
 * DISK LAYOUT
 *  * This program(boot.S and main.c) is the bootloader.  It should
 *    be stored in the first sector of the disk.
 *
 *  * The 2nd sector onward holds the kernel image.
 *
 *  * The kernel image must be in ELF format.
 *
 * BOOT UP STEPS
 *  * when the CPU boots it loads the BIOS into memory and executes it
 *
 *  * the BIOS intializes devices, sets of the interrupt routines, and
 *    reads the first sector of the boot device(e.g., hard-drive)
 *    into memory and jumps to it.
 *
 *  * Assuming this boot loader is stored in the first sector of the
 *    hard-drive, this code takes over...
 *
 *  * control starts in boot.S -- which sets up protected mode,
 *    and a stack so C code then run, then calls bootmain()
 *
 *  * bootmain() in this file takes over, reads in the kernel and jumps to it.
 **********************************************************************/

#define SECTSIZE	512
#define ELFHDR		((struct Elf *) 0x10000) // scratch space

void readsect(void*, uint32_t);
void readseg(uint32_t, uint32_t, uint32_t);

void
bootmain(void)
{
	struct Proghdr *ph, *eph;

	// read 1st page off disk
	readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);

	// is this a valid ELF?
	if (ELFHDR->e_magic != ELF_MAGIC)
		goto bad;

	// load each program segment (ignores ph flags)
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
	eph = ph + ELFHDR->e_phnum;
	for (; ph < eph; ph++)
		// p_pa is the load address of this segment (as well
		// as the physical address)
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);

	// call the entry point from the ELF header
	// note: does not return!
	((void (*)(void)) (ELFHDR->e_entry))();

bad:
	outw(0x8A00, 0x8A00);
	outw(0x8A00, 0x8E00);
	while (1)
		/* do nothing */;
}

// Read 'count' bytes at 'offset' from kernel into physical address 'pa'.
// Might copy more than asked
void
readseg(uint32_t pa, uint32_t count, uint32_t offset)
{
	uint32_t end_pa;

	end_pa = pa + count;

	// round down to sector boundary
	pa &= ~(SECTSIZE - 1);

	// translate from bytes to sectors, and kernel starts at sector 1
	offset = (offset / SECTSIZE) + 1;

	// If this is too slow, we could read lots of sectors at a time.
	// We'd write more to memory than asked, but it doesn't matter --
	// we load in increasing order.
	while (pa < end_pa) {
		// Since we haven't enabled paging yet and we're using
		// an identity segment mapping (see boot.S), we can
		// use physical addresses directly.  This won't be the
		// case once JOS enables the MMU.
		readsect((uint8_t*) pa, offset);
		pa += SECTSIZE;
		offset++;
	}
}

void
waitdisk(void)
{
	// wait for disk reaady
	while ((inb(0x1F7) & 0xC0) != 0x40)
		/* do nothing */;
}

void
readsect(void *dst, uint32_t offset)
{
	// wait for disk to be ready
	waitdisk();

	outb(0x1F2, 1);		// count = 1
	outb(0x1F3, offset);
	outb(0x1F4, offset >> 8);
	outb(0x1F5, offset >> 16);
	outb(0x1F6, (offset >> 24) | 0xE0);
	outb(0x1F7, 0x20);	// cmd 0x20 - read sectors

	// wait for disk to be ready
	waitdisk();

	// read a sector
	insl(0x1F0, dst, SECTSIZE/4);
}
```

