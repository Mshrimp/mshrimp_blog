## linux中ELF二进制程序解析----ELF文件实例解析


### 目录


[TOC]




针对32位系统，通过一个极为简单的C语言程序的编译和分析，进行ELF文件中各结构体成员功能分析；


代码示例：


```c
// cat main.c
int main(void)
{
    while (1);
    return 0;
}
```




### 0. 生成可执行文件



编译为中间文件，并链接为目标文件；


```c
$ gcc -c -o main.o main.c
$ ld main.o -Ttext 0xc0001500 -e main -o kernel.bin
```


通过file命令查看main.o文件属性；main.o文件是32位大端可重定向的ELF文件，机器平台为PowerPC；


```c
$ file main.o main.o: ELF 32-bit MSB relocatable, PowerPC or cisco 4500, version 1 (SYSV), not stripped
```


通过file命令查看kernel.bin文件属性；kernel.bin文件是32位大端可执行的ELF文件，机器平台为PowerPC；


```c
$ file kernel.bin kernel.bin: ELF 32-bit MSB executable, PowerPC or cisco 4500, version 1 (SYSV), statically linked, not stripped
```


通过nm命令查看main.o文件的符号表，符号表中仅有一个main函数符号；


```c
$ nm main.o 00000000 T main
```


通过nm查看kernel.bin文件的符号表，


```c
$ nm kernel.bin c0011510 A __bss_start c0011510 A _edata c0011510 A _end c0001500 T main
```




### 1. ELF header




ELF为二进制文件，不能通过一般的方式来打开，在Linux环境下可以使用xxd命令，xxd命令主要用来查看文件的十六进制格式，也可以查看二进制文件；在Windows环境下，使用Notepad++等文本编辑工具就可以打开查看；


```c
$ xxd -u -a -g 1 -s 0 -l 0x100 kernel.bin
0000000: 7F 45 4C 46 01 02 01 00 00 00 00 00 00 00 00 00  .ELF............
0000010: 00 02 00 14 00 00 00 01 C0 00 15 00 00 00 00 34  ...............4
0000020: 00 00 15 74 00 00 00 00 00 34 00 20 00 02 00 28  ...t.....4. ...(
0000030: 00 06 00 03 00 00 00 01 00 00 00 00 C0 00 00 00  ................
```


0x00~0x0F是e_ident数组，e_ident[0]~e_ident[3]的四个字节是固定的ELF魔数，0x7f, 0x45, 0x4c, 0x46；e_ident[4] = 1表示文件是32位的ELF文件，e_ident[5] = 2表示文件时大端字节序，e_ident[6] = 1表示默认版本，e_ident[7]~e_ident[15]的9个字节置为0；


0x10~0x11是e_type，2个字节，e_type = 0x00 02，表示类型是ET_EXEC，即可执行文件；


0x12~0x13是e_machine，2个字节，e_machine = 0x00 14，表示机器是EM_PPC，即power pc硬件平台；


0x14~0x17是e_version，4个字节，e_version = 0x00 00 00 01，表示版本信息；


0x18~0x1B是e_entry，4个字节，e_entry = 0xC0 00 15 00，表示程序的虚拟入口地址；


0x1C~0x1F是e_phoff ，4个字节，e_phoff = 0x00 00 00 34，表示程序头表在文件中的偏移量；


0x20~0x23是e_shoff，4个字节，e_shoff = 0x00 00 15 74，表示节头表在文件内的偏移量；


0x24~0x27是e_flags，4个字节，e_flags = 0x00 00 00 00，表示属性；


0x28~0x29是e_ehsize，2个字节，e_ehsize = 0x00 34，表示ELF header大小是0x34字节；程序头表紧跟ELF header之后；


0x2A~0x2B是e_phentsize，2个字节，e_phentsize = 0x00 20，表示program header结构ELF32_Phdr的大小，此处为0x20；


0x2C~0x2D是e_phnum，2个字节，e_phnum = 0x00 02，表示程序头表中段的个数，此处为2个段；


0x2E~0x2F是e_shentsize，2个字节，e_shentsize = 0x00 28，表示节头表中节的大小，此处为0x28；


0x30~0x31是e_shnum，2个字节，e_shnum = 0x00 06，表示节头表中节的个数，此处为6个节；


0x32~0x33是e_shstrndx，2个字节，e_shstrndx = 0x00 03，表示string name table在节头表中的索引为3；




### 2. 程序头表




```c
$ xxd -u -a -g 1 -s 0 -l 0x100 kernel.bin
......
0000030: 00 06 00 03 00 00 00 01 00 00 00 00 C0 00 00 00  ................
0000040: C0 00 00 00 00 00 15 10 00 00 15 10 00 00 00 05  ................
0000050: 00 01 00 00 64 74 E5 51 00 00 00 00 00 00 00 00  ....dt.Q........
0000060: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 06  ................
0000070: 00 00 00 04 00 00 00 00 00 00 00 00 00 00 00 00  ................
```


在ELF header中，程序头表偏移e_phoff = 0x00 00 00 34，所以程序头表的偏移位置为0x34，程序头表中段的大小e_phentsize = 0x00 20，段的个数e_phnum = 0x00 02，表示程序头表中有两个段，每个段大小0x20字节；ELF header大小e_ehsize = 0x00 34，程序头表紧跟ELF header之后；0x34~0x73之间共0x40个字节，是两个程序头的内容，从0x34地址开始，按照Elf32_Phdr结构分析；




第一个段头在0x34~0x53，共0x20个字节；


0x34~0x37是p_type，4个字节，p_type = 0x00 00 00 01，表示类型为PT_LOAD，为可加载程序段；


0x38~0x3B是p_offset，4个字节，p_offset = 0x00 00 00 00，表示本段在文件内的偏移量；


0x3C~0x3F是p_vaddr，4个字节，p_vaddr = 0xC0 00 00 00，表示本段被加载到内存后的起始虚拟地址；


0x40~0x43是p_paddr，4个字节，p_paddr = 0xC0 00 00 00，保留，和p_vaddr一致；


0x44~0x47是p_filesz，4个字节，p_filesz = 0x00 00 15 10，表示本段在文件中的大小；


0x48~0x4B是p_memsz，4个字节，p_memsz = 0x00 00 15 10，表示本段在内存中的大小；


0x4C~0x4F是p_flags，4个字节，p_flags = 0x00 00 00 05，表示本段的属性，5 = 4 + 1 = PF_R + PF_X，此处为可读和可执行权限；


0x50~0x53是p_align，4个字节，p_align = 0x00 01 00 00，表示本段的对齐方式，此处为64K对齐；




第二个段头在0x54~0x73，共0x20个字节；


0x54~0x57是p_type，4个字节，p_type = 0x64 74 E5 51，表示类型为PT_GNU_STACK，为GNU的栈段；


```c
// include/uapi/linux/elf.h
#define PT_LOOS    0x60000000
#define PT_GNU_STACK    (PT_LOOS + 0x474e551)
```


之后的分析省略；




### 3. 节头表



在ELF header中，节头表偏移e_shoff = 0x00 00 15 74，所以节头表的偏移位置为0x1574，节头表中段的大小e_shentsize = 0x00 28，段的个数e_shnum = 0x00 06，表示节头表中有六个节，每个节0x28字节；0x1574~0x1663共0xF0个字节，是六个节头的内容，从0x1574地址开始，按照Elf32_Shdr结构分析；


```c
$ xxd -u -g 1 -s 0x1570 -l 0x100 kernel.bin
0001570: 74 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  t...............
0001580: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
0001590: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 1B  ................
00015a0: 00 00 00 01 00 00 00 06 C0 00 15 00 00 00 15 00  ................
00015b0: 00 00 00 10 00 00 00 00 00 00 00 00 00 00 00 04  ................
00015c0: 00 00 00 00 00 00 00 21 00 00 00 01 00 00 00 00  .......!........
00015d0: 00 00 00 00 00 00 15 10 00 00 00 38 00 00 00 00  ...........8....
00015e0: 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 11  ................
00015f0: 00 00 00 03 00 00 00 00 00 00 00 00 00 00 15 48  ...............H
0001600: 00 00 00 2A 00 00 00 00 00 00 00 00 00 00 00 01  ...*............
0001610: 00 00 00 00 00 00 00 01 00 00 00 02 00 00 00 00  ................
0001620: 00 00 00 00 00 00 16 64 00 00 00 80 00 00 00 05  .......d........
0001630: 00 00 00 04 00 00 00 04 00 00 00 10 00 00 00 09  ................
0001640: 00 00 00 03 00 00 00 00 00 00 00 00 00 00 16 E4  ................
0001650: 00 00 00 25 00 00 00 00 00 00 00 00 00 00 00 01  ...%............
0001660: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```



第一个节头在0x1574~0x159B，共0x28个字节；


0x1574~0x1577是sh_name，4个字节，sh_name = 0x00 00 00 00，表示节名称是在字符串表索引值为0x00的字符串，该字符串为空；


0x1578~0x157B是sh_type，4个字节，sh_type = 0x00 00 00 00，表示该节是一个无效的节头，没有对应的节；该节中其他成员也无意义；



第二个节头在0x159C~0x15C3，共0x28个字节；


0x159C~0x159F是sh_name，4个字节，sh_name = 0x00 00 00 1B，表示节名称是在字符串表索引值为0x1B的字符串；该字符串为".text"；


0x15A0~0x15A3是sh_type，4个字节，sh_type = 0x00 00 00 01，表示


0x15A4~0x15A7是sh_flags，4个字节，sh_flags = 0x00 00 00 06，0x06 = 0x04 + 0x02 = SHF_EXECINSTR + SHF_ALLOC，表示该节内容是指令代码，并且包含内存在进程运行时需要占用内存单元；


0x15A8~0x15AB是sh_addr，4个字节，sh_addr = 0xC0 00 15 00，表示该节在内存中的起始地址，节映射到虚拟地址空间中的位置；


0x15AC~0x15AF是sh_offset，4个字节，sh_offset = 0x00 00 15 00，表示该节在文件中的起始位置；


0x15B0~0x15B3是sh_size，4个字节，sh_size = 0x00 00 00 10，表示该节的大小；


0x15B4~0x15B7是sh_link，4个字节，sh_link = 00 00 00 00


0x15B8~0x15BB是sh_info，4个字节，sh_info = 0x00 00 00 00


0x15BC~0x15BF是sh_addralign，4个字节，sh_addralign = 0x00 00 00 04，表示该节的数据在内存中以16字节对齐；


0x15C0~0x15C3是sh_entsize，4个字节，sh_entsize = 0x00 00 00 00


以后各节的解析省略；




### 4. readelf查看结果



1) 显示程序的ELF文件头


```c
$ readelf -h kernel.bin
ELF Header:
  Magic:   7f 45 4c 46 01 02 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, big endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           PowerPC
  Version:                           0x1
  Entry point address:               0xc0001500
  Start of program headers:          52 (bytes into file)
  Start of section headers:          5492 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         6
  Section header string table index: 3
```



2) 显示程序所有的程序头


```c
$ readelf -l kernel.bin

Elf file type is EXEC (Executable file)
Entry point 0xc0001500
There are 2 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0xc0000000 0xc0000000 0x01510 0x01510 R E 0x10000
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4

 Section to Segment mapping:
  Segment Sections...
   00     .text
   01
```



3) 显示程序所有的节头


```c
$ readelf -S kernel.bin
There are 6 section headers, starting at offset 0x1574:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        c0001500 001500 000010 00  AX  0   0  4
  [ 2] .comment          PROGBITS        00000000 001510 000038 00      0   0  1
  [ 3] .shstrtab         STRTAB          00000000 001548 00002a 00      0   0  1
  [ 4] .symtab           SYMTAB          00000000 001664 000080 10      5   4  4
  [ 5] .strtab           STRTAB          00000000 0016e4 000025 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```



4) 综合显示程序所有的头信息，包含ELF文件头、程序头、节头信息；


```c
$ readelf -e kernel.bin
ELF Header:
  Magic:   7f 45 4c 46 01 02 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, big endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           PowerPC
  Version:                           0x1
  Entry point address:               0xc0001500
  Start of program headers:          52 (bytes into file)
  Start of section headers:          5492 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         6
  Section header string table index: 3

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        c0001500 001500 000010 00  AX  0   0  4
  [ 2] .comment          PROGBITS        00000000 001510 000038 00      0   0  1
  [ 3] .shstrtab         STRTAB          00000000 001548 00002a 00      0   0  1
  [ 4] .symtab           SYMTAB          00000000 001664 000080 10      5   4  4
  [ 5] .strtab           STRTAB          00000000 0016e4 000025 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0xc0000000 0xc0000000 0x01510 0x01510 R E 0x10000
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4

 Section to Segment mapping:
  Segment Sections...
   00     .text
   01
```



通过readelf命令查看的结果，和按照ELF文件分析的结果，对比结果一致；




### 5. 总结




通过xxd命令打开ELF二进制文件，对比ELF文件头、程序头表、节头表的结构体进行解析，可以得出程序中程序头和节头在文件中的位置和其他信息；在内核中通过这种固定的结构体进行解析，将各个程序头和节头等信息按照按照一定的布局加载到内存中，之后就可以执行了；




### 参考资料




《操作系统真象还原》