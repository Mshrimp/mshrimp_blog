## linux中ELF格式二进制程序

### 目录

[TOC]

### 0. 简介



在Linux系统的可执行文件（ELF文件）中，开头时一个文件头，用来描述程序的布局，整个文件的属性等信息，包括文件是否可执行、静态还是动态链接及入口地址等信息；如下图所示：

￼

程序文件中包含了程序头，程序的入口地址等信息不需要写死，调用代码可以通用，根据实际情况加载；此时的文件不是纯碎的二进制可执行文件了，因为包含的程序头不是可执行代码；将这种包含程序头的文件读入到内存后，从程序头中读取入口地址，跳转到入口地址执行；



#### 0.1 文件格式



linux系统中用C语言编写完代码，使用编译器gcc编译生成的是ELF格式的二进制文件；

ELF指：Executable and Linkable Format，可执行链接格式；本文中的目标文件指各种类型符合ELF规范的我呢见，如：二进制可执行文件、Linux下的.o目标文件和.so动态库文件；

可执行文件（Executable file）：经过编译链接后，可以直接运行的程序文件；

共享目标文件（Shared object file）：动态链接库，在可执行文件被加载的过程中动态链接，成为程序代码的一部分；

可重定位文件（Relocatable file）：可重定位文件即目标文件，是源文件编译后但未完成链接的半成品，被用于与其他目标文件合并链接，以构建出二进制可执行文件或动态链接库；

核心转储文件（Core dump file）：当进程意外终止时，系统可以将该进程的地址空间的内容及终止时的一些信息转储到核心转储文件；



#### 0.2 段和节



程序中的段（Segment）和节（Secton）是真正的程序体；段包括代码段和数据段等，段是由节组成的，多个节经过链接后被合并为一个段；

段和节的信息用header描述，程序头是program header，节头是section header；段和节的大小和数量都是不固定的，需要用专门的数据结构来描述，即程序头表（program header table）和节头表（section header table），这是两个数组，元素分别是程序头（program header）和节头（section header）；程序头表（program header table）中的元素全是程序头（program header），节头表（section header table）中的元素全是节头（section header）；程序头表是用来描述段（Segment）的，成为段头表；段是程序本身的组成部分；

由于段和节的大小和数量都是不固定的，程序头表和节头表的大小也不固定，两个表在程序文件中的位置也不固定；需要在一个固定的位置，用一个固定大小的数据结构来描述程序头表和节头表的大小和位置信息，即位于文件最开始部分的ELF header；



### 1. ELF header



ELF文件分为文件头和文件体两部分；先用ELF header从文件全局概要出程序中程序头表、节头表的位置和大小等信息；然后从程序头表和节头表中分别解析出各个段和节的位置和大小等信息；

可执行文件和待重定位文件，文件最开头的部分是ELF header；程序头表对于可执行文件是必须的，而对于待重定位文件是可选的；

ELF文件头用Elf32_Ehdr和Elf64_Ehdr结构体表示；

```c
// include/uapi/linux/elf.h
#define EI_NIDENT   16

typedef struct elf32_hdr{
  unsigned char e_ident[EI_NIDENT];
  Elf32_Half    e_type;
  Elf32_Half    e_machine;
  Elf32_Word    e_version;
  Elf32_Addr    e_entry;  /* Entry point */
  Elf32_Off e_phoff;
  Elf32_Off e_shoff;
  Elf32_Word    e_flags;
  Elf32_Half    e_ehsize;
  Elf32_Half    e_phentsize;
  Elf32_Half    e_phnum;
  Elf32_Half    e_shentsize;
  Elf32_Half    e_shnum;
  Elf32_Half    e_shstrndx;
} Elf32_Ehdr;

typedef struct elf64_hdr {
  unsigned char e_ident[EI_NIDENT]; /* ELF "magic number" */
  Elf64_Half e_type;
  Elf64_Half e_machine;
  Elf64_Word e_version;
  Elf64_Addr e_entry;       /* Entry point virtual address */
  Elf64_Off e_phoff;        /* Program header table file offset */
  Elf64_Off e_shoff;        /* Section header table file offset */
  Elf64_Word e_flags;
  Elf64_Half e_ehsize;
  Elf64_Half e_phentsize;
  Elf64_Half e_phnum;
  Elf64_Half e_shentsize;
  Elf64_Half e_shnum;
  Elf64_Half e_shstrndx;
} Elf64_Ehdr;
```



| 成员        | 大小（32位） | 大小（64位） | 说明                                                         |
| ----------- | ------------ | ------------ | ------------------------------------------------------------ |
| e_ident     | 16           | 16           | 表示ELF字符等信息，开头四个字节是固定不变的elf文件魔数，0x7f 0x45 0x4c 0x46；可用于确认文件类型是否正确； |
| e_type      | 2            | 2            | ELF目标文件的类型；                                          |
| e_machine   | 2            | 2            | ELF目标文件的体系结构类型，即要在哪种硬件平台运行；          |
| e_version   | 4            | 4            | 版本信息；                                                   |
| e_entry     | 4            | 8            | 操作系统运行该程序时，将控制权转交到的虚拟地址；             |
| e_phoff     | 4            | 8            | 程序头表（program header table）在文件内的字节偏移量；如果没有程序头表，该值为0； |
| e_shoff     | 4            | 8            | 节头表（section header table）在文件内的字节偏移量；如果没有节头表，该值为0 |
| e_flags     | 4            | 4            | 与处理器相关的标志；                                         |
| e_ehsize    | 2            | 2            | ELF header的大小；                                           |
| e_phentsize | 2            | 2            | 程序头表（program header table）中每个条目（entry）的大小，即每个用来描述段信息的数据结构的大小；struct Elf32_Phdr； |
| e_phnum     | 2            | 2            | 程序头表中条目的数量，即段的个数；                           |
| e_shentsize | 2            | 2            | 节头表（section header table）中每个条目（entry）的大小，即每个用来描述节信息的数据结构的大小； |
| e_shnum     | 2            | 2            | 节头表中条目的数量，即节的个数；                             |
| e_shstrndx  | 2            | 2            | string name table在节头表中的索引index；                     |



e_ident[EI_NIDENT]是16字节大小的数组，用来表示ELF字符等信息，开头四个字节是固定不变的elf文件魔数，0x7f 0x45 0x4c 0x46；

| e_ident数组               | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| e_ident[0~3] = 0x7f E L F | 固定的ELF文件魔数，用于识别ELF文件                           |
| e_ident[4]                | 表示ELF文件的类型，0：不可识别类型，1：32位elf格式文件，2：64位elf格式文件 |
| e_ident[5]                | 指定编码格式，字节序是大端还是小端，0：非法编码格式，1：小端LSB，2：大端MSB |
| e_ident[6]                | ELF头的版本信息，默认为1，0：非法版本，1：当前版本           |
| e_ident[7~15]             | 保留，置为0                                                  |



e_type：2字节，用来指定ELF目标文件的类型；

```c
// include/uapi/linux/elf.h
/* These constants define the different elf file types */
#define ET_NONE   0		// 未知目标文件格式
#define ET_REL    1		// 可重定位文件
#define ET_EXEC   2		// 可执行文件
#define ET_DYN    3		// 动态共享目标文件
#define ET_CORE   4		// core文件，程序崩溃时内存映像的转储格式
#define ET_LOPROC 0xff00	// 特定处理器文件的扩展下界
#define ET_HIPROC 0xffff	// 特定处理器文件的扩展上界
```



e_machine：2字节，用来描述ELF目标文件的体系结构类型，即要在哪种硬件平台运行；

```c
// include/uapi/linux/elf-em.h
/* These constants define the various ELF target machines */
#define EM_NONE     0
#define EM_M32      1
#define EM_SPARC    2
#define EM_386      3
#define EM_68K      4
#define EM_88K      5
#define EM_486      6   /* Perhaps disused */
#define EM_860      7
#define EM_MIPS     8   /* MIPS R3000 (officially, big-endian only) */
                /* Next two are historical and binaries and
                   modules of these types will be rejected by
                   Linux.  */
#define EM_MIPS_RS3_LE  10  /* MIPS R3000 little-endian */
#define EM_MIPS_RS4_BE  10  /* MIPS R4000 big-endian */

#define EM_PARISC   15  /* HPPA */
#define EM_SPARC32PLUS  18  /* Sun's "v8plus" */
#define EM_PPC      20  /* PowerPC */
......
```



### 2. 程序头表



程序头表（也称为段表）是一个描述文件中各个段的数组，程序头表描述了文件中各个段在文件中的偏移位置及段的属性等信息；从程序头表里可以得到每个段的所有信息，包括代码段和数据段等；各个段的内容紧跟ELF文件头保存；程序头表中各个段用Elf32_Phdr或Elf64_Phdr结构体表示；

```c
// include/uapi/linux/elf.h
typedef struct elf32_phdr{
  Elf32_Word    p_type;
  Elf32_Off p_offset;
  Elf32_Addr    p_vaddr;
  Elf32_Addr    p_paddr;
  Elf32_Word    p_filesz;
  Elf32_Word    p_memsz;
  Elf32_Word    p_flags;
  Elf32_Word    p_align;
} Elf32_Phdr;

typedef struct elf64_phdr {
  Elf64_Word p_type;
  Elf64_Word p_flags;
  Elf64_Off p_offset;       /* Segment file offset */
  Elf64_Addr p_vaddr;       /* Segment virtual address */
  Elf64_Addr p_paddr;       /* Segment physical address */
  Elf64_Xword p_filesz;     /* Segment size in file */
  Elf64_Xword p_memsz;      /* Segment size in memory */
  Elf64_Xword p_align;      /* Segment alignment, file & memory */
} Elf64_Phdr;
```




Elf32_Phdr或Elf64_Phdr结构体用来描述位于磁盘上的程序中的一个段；

| 成员     | 大小（32位） | 大小（64位） | 说明                                                         |
| -------- | ------------ | ------------ | ------------------------------------------------------------ |
| p_type   | 4            | 4            | 程序中该段的类型；                                           |
| p_offset | 4            | 8            | 本段在文件内的起始偏移；                                     |
| p_vaddr  | 4            | 8            | 本段在内存中的起始虚拟地址；                                 |
| p_paddr  | 4            | 8            | 用于与物理地址相关的系统中；                                 |
| p_filesz | 4            | 8            | 本段在文件中的大小；                                         |
| p_memsz  | 4            | 8            | 本段在内存中的大小；                                         |
| p_flags  | 4            | 4            | 与本段相关的标志；                                           |
| p_align  | 4            | 8            | 本段在文件和内存中的对齐方式；如果为0或1，表示不对齐，否则应该为2的幂次数； |



p_type：4字节，用来指明程序中该段的类型；

```c
// include/uapi/linux/elf.h
/* These constants are for the segment types stored in the image headers */
#define PT_NULL    0 // 忽略 
#define PT_LOAD    1 // 可加载程序段
#define PT_DYNAMIC 2 // 动态链接信息
#define PT_INTERP  3 // 动态加载器名称
#define PT_NOTE    4 // 辅助的附加信息 
#define PT_SHLIB   5 // 保留
#define PT_PHDR    6 // 程序头表
#define PT_TLS     7               /* Thread local storage segment */
#define PT_LOOS    0x60000000      /* OS-specific */
#define PT_HIOS    0x6fffffff      /* OS-specific */
#define PT_LOPROC  0x70000000
#define PT_HIPROC  0x7fffffff
```




p_flags：4字节，用来指明与本段相关的标志；

```c
// include/uapi/linux/elf.h
/* These constants define the permissions on sections in the program    header, p_flags. */
#define PF_R        0x4  // 可读
#define PF_W        0x2  // 可写
#define PF_X        0x1  // 可执行
```



### 3. 节头表



节头表中各个节用Elf32_Shdr或Elf64_Shdr结构体表示；

```c
// include/uapi/linux/elf.h
typedef struct elf32_shdr {
  Elf32_Word    sh_name;
  Elf32_Word    sh_type;
  Elf32_Word    sh_flags;
  Elf32_Addr    sh_addr;
  Elf32_Off sh_offset;
  Elf32_Word    sh_size;
  Elf32_Word    sh_link;
  Elf32_Word    sh_info;
  Elf32_Word    sh_addralign;
  Elf32_Word    sh_entsize;
} Elf32_Shdr;

typedef struct elf64_shdr {
  Elf64_Word sh_name;       /* Section name, index in string tbl */
  Elf64_Word sh_type;       /* Type of section */
  Elf64_Xword sh_flags;     /* Miscellaneous section attributes */
  Elf64_Addr sh_addr;       /* Section virtual addr at execution */
  Elf64_Off sh_offset;      /* Section file offset */
  Elf64_Xword sh_size;      /* Size of section in bytes */
  Elf64_Word sh_link;       /* Index of another section */
  Elf64_Word sh_info;       /* Additional section information */
  Elf64_Xword sh_addralign; /* Section alignment */
  Elf64_Xword sh_entsize;   /* Entry size if section holds table */
} Elf64_Shdr;
```




Elf32_Shdr或Elf64_Shdr结构体用来描述位于磁盘上的程序中的一个节；

成员32位64位说明sh_name44节名称，值是字符串的一个索引；节名称字符串以'\0'结尾，统一存储在字符串表中，使用该字段存储节名称字符串在字符串表中的索引位置；sh_type44节的类型；sh_flags48节的标志；sh_addr48节在内存中的起始地址，指定节映射到虚拟地址空间中的位置；sh_offset48节在文件中的起始位置；sh_size48节的大小；sh_link44引用另一个节头表项，根据节类型有不同的解释；sh_info44节的附加信息，与sh_link联合使用；sh_addralign48节数据在内存中的对齐方式；sh_entsize48指定节中各数据项的长度，各数据项长度要相同；

| 成员         | 大小（32位） | 大小（64位） | 说明                                                         |
| ------------ | ------------ | ------------ | ------------------------------------------------------------ |
| sh_name      | 4            | 4            | 节名称，值是字符串的一个索引；节名称字符串以'\0'结尾，统一存储在字符串表中，使用该字段存储节名称字符串在字符串表中的索引位置； |
| sh_type      | 4            | 4            | 节的类型；                                                   |
| sh_flags     | 4            | 8            | 节的标志；                                                   |
| sh_addr      | 4            | 8            | 节在内存中的起始地址，指定节映射到虚拟地址空间中的位置；     |
| sh_offset    | 4            | 8            | 节在文件中的起始位置；                                       |
| sh_size      | 4            | 8            | 节的大小；                                                   |
| sh_link      | 4            | 4            | 引用另一个节头表项，根据节类型有不同的解释；                 |
| sh_info      | 4            | 4            | 节的附加信息，与sh_link联合使用；                            |
| sh_addralign | 4            | 8            | 节数据在内存中的对齐方式；sh_entsize48指定节中各数据项的长度，各数据项长度要相同； |



sh_name

节名称sh_name的值是字符串的一个索引，节名称字符串以'\0'结尾，字符串统一存放在字符串表中，使用sh_name的值作为字符串表的索引，找到对应的字符串即为节名称；

字符串表中包含多个以'\0'结尾的字符串；在目标文件中，这些字符串通常是符号的名字或节的名字，需要引用某些字符串时，只需要提供该字符串在字符串表中的序号即可；

字符串表中的第一个字符串（序号为0）是空串，即'\0'，可以用于表示没有名字或一个空的名字；如果字符串表为空，节头中的sh_size值为0；


sh_type：节的类型；

```c
// include/uapi/linux/elf.h
/* sh_type */
#define SHT_NULL    0 // 表示该节是无效节头，没有对应的节
#define SHT_PROGBITS    1
#define SHT_SYMTAB  2
#define SHT_STRTAB  3
#define SHT_RELA    4
#define SHT_HASH    5
#define SHT_DYNAMIC 6
#define SHT_NOTE    7
#define SHT_NOBITS  8
#define SHT_REL     9
#define SHT_SHLIB   10
#define SHT_DYNSYM  11
#define SHT_NUM     12
#define SHT_LOPROC  0x70000000
#define SHT_HIPROC  0x7fffffff
#define SHT_LOUSER  0x80000000
#define SHT_HIUSER  0xffffffff
```




sh_flags：节的标志；

```c
// include/uapi/linux/elf.h
/* sh_flags */
#define SHF_WRITE   0x1  // 可写
#define SHF_ALLOC   0x2  // 该节包含内存在进程运行时需要占用内存单元
#define SHF_EXECINSTR   0x4 // 该节内容是指令代码
#define SHF_MASKPROC    0xf0000000
```



### 4. readelf命令



readelf命令用于查看ELF格式的文件信息，

```c
$ readelf -H
Usage: readelf <option(s)> elf-file(s)
 Display information about the contents of ELF format files
 Options are:
  -a --all               Equivalent to: -h -l -S -s -r -d -V -A -I
  -h --file-header       Display the ELF file header
  -l --program-headers   Display the program headers
     --segments          An alias for --program-headers
  -S --section-headers   Display the sections' header
     --sections          An alias for --section-headers
  -e --headers           Equivalent to: -h -l -S
  ......
```




命令：

```c
$ readelf -h file // 显示ELF文件头
$ readelf -l file // 显示所有的程序头
$ readelf -S file // 显示所有的节头
$ readelf -e file // 显示ELF文件头、程序头、节头全部的头信息，相当于同时运行参数-h、-l、-S
```



### 5. ELF文件实例



针对32位系统，通过一个极为简单的C语言的编译和分析，进行ELF文件中各结构体成员功能分析；

代码示例：

```c
int main(void)
{
    while (1);
    return 0;
}
```



#### 5.0 生成可执行文件




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



#### 5.1 ELF header



```c
$ xxd -u -a -g 1 -s 0 -l 0x100 kernel.bin
0000000: 7F 45 4C 46 01 02 01 00 00 00 00 00 00 00 00 00  .ELF............
0000010: 00 02 00 14 00 00 00 01 C0 00 15 00 00 00 00 34  ...............4
0000020: 00 00 15 74 00 00 00 00 00 34 00 20 00 02 00 28  ...t.....4. ...(
0000030: 00 06 00 03 00 00 00 01 00 00 00 00 C0 00 00 00  ................
```

00~0x0F是e_ident数组，e_ident[0]~e_ident[3]的四个字节是固定的ELF魔数，0x7f, 0x45, 0x4c, 0x46；e_ident[4] = 1表示文件是32位的ELF文件，e_ident[5] = 2表示文件时大端字节序，e_ident[6] = 1表示默认版本，e_ident[7]~e_ident[15]的9个字节置为0；

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



#### 5.2 程序头表



```c
$ xxd -u -a -g 1 -s 0 -l 0x100 kernel.bin ......
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



#### 5.3 节头表




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



#### 5.4 readelf查看结果



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
  Type           Offset     VirtAddr   PhysAddr
                 FileSiz    MemSiz     Flg   Align
  LOAD           0x00000000 0xc0000000 0xc0000000
                 0x00001510 0x00001510 R E   0x10000
  GNU_STACK      0x00000000 0x00000000 0x00000000
                 0x00000000 0x00000000 RW    0x4

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
  [Nr] Name              Type            Addr     Off
       Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000
       000000 00      0   0  0
  [ 1] .text             PROGBITS        c0001500 001500
       000010 00  AX  0   0  4
  [ 2] .comment          PROGBITS        00000000 001510
       000038 00      0   0  1
  [ 3] .shstrtab         STRTAB          00000000 001548
       00002a 00      0   0  1
  [ 4] .symtab           SYMTAB          00000000 001664
       000080 10      5   4  4
  [ 5] .strtab           STRTAB          00000000 0016e4
       000025 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)   I (info),
  L (link order), G (group), T (TLS), E (exclude), x (unknown) 
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

There are 6 section headers, starting at offset 0x1574:

Section Headers:
  [Nr] Name              Type            Addr     Off
       Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000
       000000 00      0   0  0
  [ 1] .text             PROGBITS        c0001500 001500
       000010 00  AX  0   0  4
  [ 2] .comment          PROGBITS        00000000 001510
       000038 00      0   0  1
  [ 3] .shstrtab         STRTAB          00000000 001548
       00002a 00      0   0  1
  [ 4] .symtab           SYMTAB          00000000 001664
       000080 10      5   4  4
  [ 5] .strtab           STRTAB          00000000 0016e4
       000025 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)   I (info),
  L (link order), G (group), T (TLS), E (exclude), x (unknown) 
  O (extra OS processing required) o (OS specific), p (processor specific)

Elf file type is EXEC (Executable file)
Entry point 0xc0001500
There are 2 program headers, starting at offset 52
Program Headers:
  Type           Offset     VirtAddr   PhysAddr
                 FileSiz    MemSiz     Flg   Align
  LOAD           0x00000000 0xc0000000 0xc0000000
                 0x00001510 0x00001510 R E   0x10000
  GNU_STACK      0x00000000 0x00000000 0x00000000
                 0x00000000 0x00000000 RW    0x4

Section to Segment mapping:
  Segment Sections...
    00     .text
    01
```




通过readelf命令查看的结果，和按照ELF文件分析的结果，对比结果一致；



### 6. 总结







### 参考资料




《操作系统真象还原》

程序编译-汇编-链接的理解！—03-ELF头和节头表

ELF文件-节和节头



[回到目录](#目录)


