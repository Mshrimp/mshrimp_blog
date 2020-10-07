
## linux应用程序加载过程分析-load_elf_binary




### 目录


[TOC]


### 简介




Linux内核支持多种可执行程序格式，内核对所支持的每种格式都被注册为一个linux_binfmt结构；struct linux_binfmt结构体保存处理相应格式的可执行文件的函数指针；struct linux_binfmt结构体保存在内核管理的链表中，链表头是formats；其中linux_binfmt对象的load_binary函数指针指向的就是一个可执行程序的处理函数；




### 进程空间模型




Linux系统中一个进程的虚拟空间包含以下几个基本部分：


代码区（text）：存放可执行的代码；


数据区（data）：存放经过初始化的数据；


数据区（bss）：存放未经初始化的数据，并且是全局的；


堆（heap）：进程运行时动态分配内存，向上扩展；


栈（stack）：进程的栈空间，向下扩展；


如下图所示：


![Linux进程空间模型](linux程序加载过程分析-load_elf_binary/Linux进程空间模型.PNG)


ELF文件装载的最终目的有两个：


确定各个区域的边界；text、data、bss区的起始和终止位置，heap、stack区的起始位置，其中heap和stack区的终止位置是动态变化的；


对text、data区的内容进行mmap映射；ELF文件的内容不会直接被真正复制到内存，而是在需要的时候，内核通过page fault，将ELF文件内存复制到内存中；




### 静态链接和动态链接




从编译、链接、运行的角度看，应用程序和库程序的链接有两种方式：静态链接和动态链接；


静态链接：是固定的、静态的链接，是把应用程序需要用到的库函数的目标代码从程序库中抽取出来，链接进应用软件的目标映像中；


动态链接：是指库函数的代码不加入到应用程序的目标映像，应用软件在编译、链接阶段并不完成和库函数的链接，而是把库函数的映像也交给用户，在启动应用软件目标映像运行时，才把程序库的映像装载到用户空间并定位，再完成应用软件和库函数的链接；


由于链接方式的不同，就有两种不同的ELF格式映像，一种是静态链接的，在装载、启动运行时无需装载函数库映像、无需进行动态链接；另一种是动态链接，需要在装载、启动运行时同事装载函数库映像并进行动态链接；


Linux内核支持两种ELF映像；并且装载、启动ELF映像必须由内核完成，动态链接的实现可以在内核或用户空间中完成；GNU把对于动态链接ELF映像的支持进行分工：把ELF映像的装载、启动加载到内核中，而把动态链接的实现放在用户空间中（如：glibc），并提供一个软件工具--解释器（ld-linux.so.2），而解释器的装载、启动也由内核负责；




#### linux_binfmt


Linux内核支持多种可执行程序格式，内核对所支持的每种格式都被注册为一个linux_binfmt结构；struct linux_binfmt结构体保存处理相应格式的可执行文件的函数指针；


```c
// include/linux/binfmts.h
struct linux_binfmt {
    struct list_head lh;
    struct module *module;
    int (*load_binary)(struct linux_binprm *);	// 加载新的进程
    int (*load_shlib)(struct file *);	// 加载动态共享库
    int (*core_dump)(struct coredump_params *cprm);	// 将当前进程上下文保存到文件
    unsigned long min_coredump; /* minimal dump size */
};
```


linux_binfmt结构体中提供了3种方法来加载、执行可执行程序：


	a) load_binary：读取存放在可执行文件中的信息为当前进程建立一个新的执行环境；


	b) load_shlib：用于动态地把一个共享库捆绑到一个已经运行的进程，是由uselib()系统调用激活的；


	c) core_dump：在core文件中存放当前进程的执行上下文，该文件通常在进程接收到一个”dump”信号时创建，格式取决于可执行程序类型；




Linux支持的可执行程序格式举例：


| 格式        | linux_binfmt     | load_binary           |

| ----------- | ---------------- | --------------------- |

| a.out       | aout_format      | load_aout_binary      |

| flat        | flat_format      | load_flat_binary      |

| script脚本  | script_format    | load_script           |

| misc_format | misc_format      | load_misc_binary      |

| em86        | em86_format      | load_format           |

| elf_fdpic   | elf_fdpic_format | load_elf_fdpic_binary |

| elf         | elf_format       | load_elf_binary       |




struct linux_binfmt结构体保存在内核管理的链表中，链表头是formats，可以通过register_binfmt()函数将struct linux_binfmt结构体添加到该链表中，通过unregister_binfmt()函数将struct linux_binfmt结构体从该链表中删除；


```c
// include/linux/binfmts.h
/* Registration of default binfmt handlers */
static inline void register_binfmt(struct linux_binfmt *fmt)
{
    __register_binfmt(fmt, 0);
}

// fs/exec.c
void __register_binfmt(struct linux_binfmt * fmt, int insert)
{
    BUG_ON(!fmt);
    if (WARN_ON(!fmt->load_binary))
        return;
    write_lock(&binfmt_lock);
    insert ? list_add(&fmt->lh, &formats) :
         list_add_tail(&fmt->lh, &formats);
    write_unlock(&binfmt_lock);
}
EXPORT_SYMBOL(__register_binfmt);

void unregister_binfmt(struct linux_binfmt * fmt)
{
    write_lock(&binfmt_lock);
    list_del(&fmt->lh);
    write_unlock(&binfmt_lock);
}
EXPORT_SYMBOL(unregister_binfmt);
```




在系统启动时，内核为编译进内核的每个可执行格式都执行register_binfmt()函数；在实现一个新可执行格式的模块在被装载时，也执行register_binfmt()函数，当该模块被卸载时，执行unregister_binfmt()函数；


在执行可执行程序时，内核在search_binary_handler()函数中，通过list_for_each_entry()宏遍历所有注册的linux_binfmt对象，用已注册的load_binrary方法来加载和解析ELF格式的可执行文件，用注册的load_shlib方法来加载和解析ELF格式的动态链接库；




其中linux_binfmt对象的load_binary和load_shlib函数指针指向的就是一个可执行程序和动态链接库的处理函数；针对elf格式的linux_binfmt对象elf_format，如下：


```c
// fs/binfmt_elf.c
static struct linux_binfmt elf_format = {
    .module     = THIS_MODULE,
    .load_binary    = load_elf_binary,
    .load_shlib = load_elf_library,
    .core_dump  = elf_core_dump,
    .min_coredump   = ELF_EXEC_PAGESIZE,
};
```


elf_format对象用来支持ELF文件的执行操作，必须像内核注册elf_format结构，加入到内核支持的可执行程序的队列；


```c
// fs/binfmt_elf.c
static int __init init_elf_binfmt(void)
{
    register_binfmt(&elf_format);
    return 0;
}

static void __exit exit_elf_binfmt(void)
{
    /* Remove the COFF and ELF loaders. */
    unregister_binfmt(&elf_format);
}

core_initcall(init_elf_binfmt);
module_exit(exit_elf_binfmt);
```






### load_binary




在运行程序时，扫描注册linux_binfmt对象的formats链表，依次调用各个数据结构所提供的load处理程序来加载；针对elf格式的linux_binfmt对象elf_format，ELF文件的加载程序为load_elf_binary()函数；


```c
// fs/binfmt_elf.c
static struct linux_binfmt elf_format = {
    ......
    .load_binary    = load_elf_binary,
    ......
}
```




```c
static int load_elf_binary(struct linux_binprm *bprm)
```


load_elf_binary()函数的参数struct linux_binprm *bprm包含装载二进制文件按所需的信息；






### load_elf_binary










load_elf_binary()函数的实现代码比较长，这里将代码分段分析；


```c
// fs/binfmt_elf.c
static int load_elf_binary(struct linux_binprm *bprm)
{
    struct file *interpreter = NULL; /* to shut gcc up */
    unsigned long load_addr = 0, load_bias = 0;
    int load_addr_set = 0;
    char * elf_interpreter = NULL;
    unsigned long error;
    struct elf_phdr *elf_ppnt, *elf_phdata, *interp_elf_phdata = NULL;
    unsigned long elf_bss, elf_brk;
    int retval, i;
    unsigned long elf_entry;
    unsigned long interp_load_addr = 0;
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long reloc_func_desc __maybe_unused = 0;
    int executable_stack = EXSTACK_DEFAULT;
```








```c
    struct pt_regs *regs = current_pt_regs();
    struct {
        struct elfhdr elf_ex;
        struct elfhdr interp_elf_ex;
    } *loc;
    struct arch_elf_state arch_state = INIT_ARCH_ELF_STATE;

    loc = kmalloc(sizeof(*loc), GFP_KERNEL);
    if (!loc) {
        retval = -ENOMEM;
        goto out_ret;
    }

    /* Get the exec-header */
    loc->elf_ex = *((struct elfhdr *)bprm->buf);

    retval = -ENOEXEC;
    /* First of all, some simple consistency checks */
    if (memcmp(loc->elf_ex.e_ident, ELFMAG, SELFMAG) != 0)
        goto out;

    if (loc->elf_ex.e_type != ET_EXEC && loc->elf_ex.e_type != ET_DYN)
        goto out;
    if (!elf_check_arch(&loc->elf_ex))
        goto out;
    if (!bprm->file->f_op->mmap)
        goto out;
```


在执行load_elf_binary()函数之前，内核已经将bprm->buf赋值为二进制文件的前128字节；loc->elf_ex = *((struct elfhdr *)bprm->buf);是使用该部分的128字节头文件；之后检查文件头的前四个字节；


```c
// include/uapi/linux/elf.h
#define ELFMAG      "\177ELF"
#define SELFMAG     4
```


并检查文件类型是否为ET_EXEC或ET_DYN；ET_EXEC表示可执行文件；ET_DYN表示共享库；




```c
    elf_phdata = load_elf_phdrs(&loc->elf_ex, bprm->file);
    if (!elf_phdata)
        goto out;
```


通过load_elf_phdrs()函数加载由elf_ex指向的目标文件的程序头表；


```c
static struct elf_phdr *load_elf_phdrs(struct elfhdr *elf_ex,
                       struct file *elf_file)
{
    struct elf_phdr *elf_phdata = NULL;
    int retval, size, err = -1;

    if (elf_ex->e_phentsize != sizeof(struct elf_phdr))
        goto out;

    if (elf_ex->e_phnum < 1 ||
        elf_ex->e_phnum > 65536U / sizeof(struct elf_phdr))
        goto out;

    size = sizeof(struct elf_phdr) * elf_ex->e_phnum;
    if (size > ELF_MIN_ALIGN)
        goto out;

    elf_phdata = kmalloc(size, GFP_KERNEL);
    if (!elf_phdata)
        goto out;

    /* Read in the program headers */
    retval = kernel_read(elf_file, elf_ex->e_phoff,
                 (char *)elf_phdata, size);
    if (retval != size) {
        err = (retval < 0) ? retval : -EIO;
        goto out;
    }

    /* Success! */
    err = 0;
out:
    if (err) {
        kfree(elf_phdata);
        elf_phdata = NULL;
    }
    return elf_phdata;
}
```


load_elf_phdrs()函数是通过kernel_read()函数从ELF文件中读取程序头表，保存在elf_phdata指针指向的内存中；ELF文件头elf32_hdr中的e_phoff即程序头表在文件中的偏移量，e_phnum是程序头表中表项的个数，即文件中段的数量；一个可执行程序至少要有一个段（Segment），所有段的大小不能超过65536U（64K）；


```c
    elf_ppnt = elf_phdata;
    elf_bss = 0;
    elf_brk = 0;

    start_code = ~0UL;
    end_code = 0;
    start_data = 0;
    end_data = 0;
```


在解析程序头表中的各个表项之前，先初始化各个段的起始和终止位置；


```c
    for (i = 0; i < loc->elf_ex.e_phnum; i++) {
        if (elf_ppnt->p_type == PT_INTERP) {
            /* This is the program interpreter used for
             * shared libraries - for now assume that this
             * is an a.out format binary
             */
            retval = -ENOEXEC;
            if (elf_ppnt->p_filesz > PATH_MAX ||
                elf_ppnt->p_filesz < 2)
                goto out_free_ph;

            retval = -ENOMEM;
            elf_interpreter = kmalloc(elf_ppnt->p_filesz,
                          GFP_KERNEL);
            if (!elf_interpreter)
                goto out_free_ph;

            retval = kernel_read(bprm->file, elf_ppnt->p_offset,
                         elf_interpreter,
                         elf_ppnt->p_filesz);
            if (retval != elf_ppnt->p_filesz) {
                if (retval >= 0)
                    retval = -EIO;
                goto out_free_interp;
            }
            /* make sure path is NULL terminated */
            retval = -ENOEXEC;
            if (elf_interpreter[elf_ppnt->p_filesz - 1] != '\0')
                goto out_free_interp;

            interpreter = open_exec(elf_interpreter);
            retval = PTR_ERR(interpreter);
            if (IS_ERR(interpreter))
                goto out_free_interp;

            would_dump(bprm, interpreter);

            /* Get the exec headers */
            retval = kernel_read(interpreter, 0,
                         (void *)&loc->interp_elf_ex,
                         sizeof(loc->interp_elf_ex));
            if (retval != sizeof(loc->interp_elf_ex)) {
                if (retval >= 0)
                    retval = -EIO;
                goto out_free_dentry;
            }

            break;
        }
        elf_ppnt++;
    }
```


在解析操作之前，需要确定ELF中程序头表中的各个表项是否指定了解释器，如果指定了解释器，需要装载这个解释器程序，并用该解释器来解析ELF文件中对应的程序头表；


通过elf_ppnt->p_type判断是否有需要加载的解释器；如果有，则根据偏移地址elf_ppnt->p_offset和大小elf_ppnt->p_filesz，将解释器段的内容从程序文件bprm->file读取到申请的空间elf_interpreter中；通过open_exec(elf_interpreter)打开解释器文件；通过kernel_read()读取解释器的头部，即文件的前128字节；


```c
    /* Some simple consistency checks for the interpreter */
    if (elf_interpreter) {
        retval = -ELIBBAD;
        /* Not an ELF interpreter */
        if (memcmp(loc->interp_elf_ex.e_ident, ELFMAG, SELFMAG) != 0)
            goto out_free_dentry;
        /* Verify the interpreter has a valid arch */
        if (!elf_check_arch(&loc->interp_elf_ex))
            goto out_free_dentry;

        /* Load the interpreter program headers */
        interp_elf_phdata = load_elf_phdrs(&loc->interp_elf_ex,
                           interpreter);
        if (!interp_elf_phdata)
            goto out_free_dentry;

        /* Pass PT_LOPROC..PT_HIPROC headers to arch code */
        elf_ppnt = interp_elf_phdata;
        for (i = 0; i < loc->interp_elf_ex.e_phnum; i++, elf_ppnt++)
            switch (elf_ppnt->p_type) {
            case PT_LOPROC ... PT_HIPROC:
                retval = arch_elf_pt_proc(&loc->interp_elf_ex,
                              elf_ppnt, interpreter,
                              true, &arch_state);
                if (retval)
                    goto out_free_dentry;
                break;
            }
    }

    /*
     * Allow arch code to reject the ELF at this point, whilst it's
     * still possible to return an error to the code that invoked
     * the exec syscall.
     */
    retval = arch_check_elf(&loc->elf_ex, !!interpreter, &arch_state);
    if (retval)
        goto out_free_dentry;
```


检查是否有动态链接器，检查并读取解释器的程序表头，通过load_elf_phdrs()函数读取解释器的程序表头；




```c
    /* Now we do a little grungy work by mmapping the ELF image into
       the correct location in memory. */
    for(i = 0, elf_ppnt = elf_phdata;
        i < loc->elf_ex.e_phnum; i++, elf_ppnt++) {
        int elf_prot = 0, elf_flags;
        unsigned long k, vaddr;
        unsigned long total_size = 0;

        if (elf_ppnt->p_type != PT_LOAD)
            continue;

        if (unlikely (elf_brk > elf_bss)) {
            unsigned long nbyte;

            /* There was a PT_LOAD segment with p_memsz > p_filesz
               before this one. Map anonymous pages, if needed,
               and clear the area.  */
            retval = set_brk(elf_bss + load_bias,
                     elf_brk + load_bias);
            if (retval)
                goto out_free_dentry;
            nbyte = ELF_PAGEOFFSET(elf_bss);
            if (nbyte) {
                nbyte = ELF_MIN_ALIGN - nbyte;
                if (nbyte > elf_brk - elf_bss)
                    nbyte = elf_brk - elf_bss;
                if (clear_user((void __user *)elf_bss +
                            load_bias, nbyte)) {
                    /*
                     * This bss-zeroing can fail if the ELF
                     * file specifies odd protections. So
                     * we don't check the return value
                     */
                }
            }
        }
```


从目标映像的所有程序头中搜索类型为PT_LOAD的段（Segment），二进制映像中，只有PT_LOAD类型的段才是需要装载的；在装载之前，需要确定装载的地址，涉及到页面对齐；






```c
    if (elf_interpreter) {
        unsigned long interp_map_addr = 0;

        elf_entry = load_elf_interp(&loc->interp_elf_ex,
                        interpreter,
                        &interp_map_addr,
                        load_bias, interp_elf_phdata);
        if (!IS_ERR((void *)elf_entry)) {
            /*
             * load_elf_interp() returns relocation
             * adjustment
             */
            interp_load_addr = elf_entry;
            elf_entry += loc->interp_elf_ex.e_entry;
        }
        if (BAD_ADDR(elf_entry)) {
            retval = IS_ERR((void *)elf_entry) ?
                    (int)elf_entry : -EINVAL;
            goto out_free_dentry;
        }
        reloc_func_desc = interp_load_addr;

        allow_write_access(interpreter);
        fput(interpreter);
        kfree(elf_interpreter);
    } else {
        elf_entry = loc->elf_ex.e_entry;
        if (BAD_ADDR(elf_entry)) {
            retval = -EINVAL;
            goto out_free_dentry;
        }
    }

    kfree(interp_elf_phdata);
    kfree(elf_phdata);

    set_binfmt(&elf_format);

#ifdef ARCH_HAS_SETUP_ADDITIONAL_PAGES
    retval = arch_setup_additional_pages(bprm, !!elf_interpreter);
    if (retval < 0)
        goto out;
#endif /* ARCH_HAS_SETUP_ADDITIONAL_PAGES */
```


填写程序的入口地址；如果需要装载解释器，通过load_elf_interp()函数装载解释器映像，load_elf_interp()函数返回值为解释器映像的入口地址，将返回值赋值为进入用户空间的入口地址elf_entry；如果不需要装载解释器，则入口地址就是目标映像的入口地址；


在需要加载解释器时，内核把控制权交给动态链接器，动态链接器检查程序对共享库的依赖，并在需要时对其进行加载，通过load_elf_interp()函数实现；


```c
    install_exec_creds(bprm);
    retval = create_elf_tables(bprm, &loc->elf_ex,
              load_addr, interp_load_addr);
    if (retval < 0)
        goto out;
    /* N.B. passed_fileno might not be initialized? */
    current->mm->end_code = end_code;
    current->mm->start_code = start_code;
    current->mm->start_data = start_data;
    current->mm->end_data = end_data;
    current->mm->start_stack = bprm->p;

    if ((current->flags & PF_RANDOMIZE) && (randomize_va_space > 1)) {
        current->mm->brk = current->mm->start_brk =
            arch_randomize_brk(current->mm);
#ifdef compat_brk_randomized
        current->brk_randomized = 1;
#endif
    }

    if (current->personality & MMAP_PAGE_ZERO) {
        /* Why this, you ask???  Well SVr4 maps page 0 as read-only,
           and some applications "depend" upon this behavior.
           Since we do not have the power to recompile these, we
           emulate the SVr4 behavior. Sigh. */
        error = vm_mmap(NULL, 0, PAGE_SIZE, PROT_READ | PROT_EXEC,
                MAP_FIXED | MAP_PRIVATE, 0);
    }

#ifdef ELF_PLAT_INIT
    /*
     * The ABI may specify that certain registers be set up in special
     * ways (on i386 %edx is the address of a DT_FINI function, for
     * example.  In addition, it may also specify (eg, PowerPC64 ELF)
     * that the e_entry field is the address of the function descriptor
     * for the startup routine, rather than the address of the startup
     * routine itself.  This macro performs whatever initialization to
     * the regs structure is required as well as any relocations to the
     * function descriptor entries when executing dynamically links apps.
     */
    ELF_PLAT_INIT(regs, reloc_func_desc);
#endif
```


通过create_elf_tables()函数在内存中生成elf映射表，填写目标文件的参数环境变量等必要信息；在完成装载，启动用户空间的映像运行之前，还需要为目标映像和解释器准备好有关信息，如：argc、envc等；这些信息需要复制到用户空间，使这些信息在CPU进入解释器或目标映像的程序入口时出现在用户空间堆栈上；


```c
    start_thread(regs, elf_entry, bprm->p);
    retval = 0;
out:
    kfree(loc);
out_ret:
    return retval;

    /* error cleanup */
out_free_dentry:
    kfree(interp_elf_phdata);
    allow_write_access(interpreter);
    if (interpreter)
        fput(interpreter);
out_free_interp:
    kfree(elf_interpreter);
out_free_ph:
    kfree(elf_phdata);
    goto out;
}
```


start_thread()函数操作，将eip和esp改成新的地址，使得CPU在返回用户空间时就进入新的程序入口；如果存在解释器映像，这个入口就是解释器映像的程序入口，否则就是目标映像的程序入口；




解释器映像存在与否；如果目标映像与各种库的链接是静态链接，无需依靠共享库（即动态链接库），那就不需要解释器映像；否则，就一定要有解释器映像存在；




对于一个目标程序，在编译时，除非显式地使用static标签，否则所有程序都是动态链接的，即需要解释器；程序在被内核加载到内存，内核跳转到用户空间后并不是执行程序的，而是把控制权交给用户空间的解释器，由解释器加载运行用户程序所需要的动态库（如：libc等），然后将控制权转交给用户程序；






```c
    elf_ppnt = elf_phdata;
    for (i = 0; i < loc->elf_ex.e_phnum; i++, elf_ppnt++)
        switch (elf_ppnt->p_type) {
        case PT_GNU_STACK:
            if (elf_ppnt->p_flags & PF_X)
                executable_stack = EXSTACK_ENABLE_X;
            else
                executable_stack = EXSTACK_DISABLE_X;
            break;

        case PT_LOPROC ... PT_HIPROC:
            retval = arch_elf_pt_proc(&loc->elf_ex, elf_ppnt,
                          bprm->file, false,
                          &arch_state);
            if (retval)
                goto out_free_dentry;
            break;
        }





    /* Flush all traces of the currently running executable */
    retval = flush_old_exec(bprm);
    if (retval)
        goto out_free_dentry;

    /* Do this immediately, since STACK_TOP as used in setup_arg_pages
       may depend on the personality.  */
    SET_PERSONALITY2(loc->elf_ex, &arch_state);
    if (elf_read_implies_exec(loc->elf_ex, executable_stack))
        current->personality |= READ_IMPLIES_EXEC;

    if (!(current->personality & ADDR_NO_RANDOMIZE) && randomize_va_space)
        current->flags |= PF_RANDOMIZE;

    setup_new_exec(bprm);

    /* Do this so that we can load the interpreter, if need be.  We will
       change some of these later */
    retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP),
                 executable_stack);
    if (retval < 0)
        goto out_free_dentry;

    current->mm->start_stack = bprm->p;
















        if (elf_ppnt->p_flags & PF_R)
            elf_prot |= PROT_READ;
        if (elf_ppnt->p_flags & PF_W)
            elf_prot |= PROT_WRITE;
        if (elf_ppnt->p_flags & PF_X)
            elf_prot |= PROT_EXEC;

        elf_flags = MAP_PRIVATE | MAP_DENYWRITE | MAP_EXECUTABLE;

        vaddr = elf_ppnt->p_vaddr;
        if (loc->elf_ex.e_type == ET_EXEC || load_addr_set) {
            elf_flags |= MAP_FIXED;
        } else if (loc->elf_ex.e_type == ET_DYN) {
            /* Try and get dynamic programs out of the way of the
             * default mmap base, as well as whatever program they
             * might try to exec.  This is because the brk will
             * follow the loader, and is not movable.  */
            load_bias = ELF_ET_DYN_BASE - vaddr;
            if (current->flags & PF_RANDOMIZE)
                load_bias += arch_mmap_rnd();
            load_bias = ELF_PAGESTART(load_bias);
            total_size = total_mapping_size(elf_phdata,
                            loc->elf_ex.e_phnum);
            if (!total_size) {
                retval = -EINVAL;
                goto out_free_dentry;
            }
        }

        error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt,
                elf_prot, elf_flags, total_size);
        if (BAD_ADDR(error)) {
            retval = IS_ERR((void *)error) ?
                PTR_ERR((void*)error) : -EINVAL;
            goto out_free_dentry;
        }

        if (!load_addr_set) {
            load_addr_set = 1;
            load_addr = (elf_ppnt->p_vaddr - elf_ppnt->p_offset);
            if (loc->elf_ex.e_type == ET_DYN) {
                load_bias += error -
                             ELF_PAGESTART(load_bias + vaddr);
                load_addr += load_bias;
                reloc_func_desc = load_bias;
            }
        }
        k = elf_ppnt->p_vaddr;
        if (k < start_code)
            start_code = k;
        if (start_data < k)
            start_data = k;

        /*
         * Check to see if the section's size will overflow the
         * allowed task size. Note that p_filesz must always be
         * <= p_memsz so it is only necessary to check p_memsz.
         */
        if (BAD_ADDR(k) || elf_ppnt->p_filesz > elf_ppnt->p_memsz ||
            elf_ppnt->p_memsz > TASK_SIZE ||
            TASK_SIZE - elf_ppnt->p_memsz < k) {
            /* set_brk can never work. Avoid overflows. */
            retval = -EINVAL;
            goto out_free_dentry;
        }

        k = elf_ppnt->p_vaddr + elf_ppnt->p_filesz;

        if (k > elf_bss)
            elf_bss = k;
        if ((elf_ppnt->p_flags & PF_X) && end_code < k)
            end_code = k;
        if (end_data < k)
            end_data = k;
        k = elf_ppnt->p_vaddr + elf_ppnt->p_memsz;
        if (k > elf_brk)
            elf_brk = k;
    }

    loc->elf_ex.e_entry += load_bias;
    elf_bss += load_bias;
    elf_brk += load_bias;
    start_code += load_bias;
    end_code += load_bias;
    start_data += load_bias;
    end_data += load_bias;

    /* Calling set_brk effectively mmaps the pages that we need
     * for the bss and break sections.  We must do this before
     * mapping in the interpreter, to make sure it doesn't wind
     * up getting placed where the bss needs to go.
     */
    retval = set_brk(elf_bss, elf_brk);
    if (retval)
        goto out_free_dentry;
    if (likely(elf_bss != elf_brk) && unlikely(padzero(elf_bss))) {
        retval = -EFAULT; /* Nobody gets to see this, but.. */
        goto out_free_dentry;
    }

```






### 总结




加载流程如下：


填充并且检查目标程序ELF头部；


load_elf_phdrs()加载目标程序的程序头表；


如果需要动态链接，则寻找和处理解释器段；


检查并读取解释器的程序表头；


装入目标程序的段segment；


填写程序的入口地址；


create_elf_tables()填写目标文件的参数环境变量等必要信息；


start_thread()准备进入新的程序入口；






读取并检查目标可执行程序的头信息，检查完成后加载目标程序的程序头表；


如果需要解释器则读取并检查解释器的头信息，检查完成后加载解释器的程序头表；


装入目标程序的段Segment，这些是目标程序二进制代码中真正的可执行映像；


填写程序的入口地址（如果有解释器需要填入解释器的入口地址，否则填入可执行程序的入口地址）


create_elf_tables()填写目标文件的参数环境变量等必要信息；


start_thread()准备进入新的程序入口；






load_elf_binary()函数的加载过程；




检查ELF文件格式的有效性，如模数、段的数量等；


寻找动态链接的.interp段，设置动态链接路径；


根据ELF可执行文件程序表头的描述，对ELF文件进行映射，如代码、数据、只读数据等；


初始化ELF进程环境，如进程启动时的edx寄存器地址，应该是DT_FINI的地址；


将系统调用的返回地址修改为ELF可执行文件的入口点，对于ELF可执行文件，入口点就是e_entry指向的地址，动态链接的ELF文件的入口点是动态链接器；




load_elf_binary()函数执行完之后，从内核态返回用户态时，eip寄存器就直接跳转到ELF程序的入口地址了；新的程序开始执行，ELF文件装载完毕；




### 参考资料




[Linux内核中ELF可执行文件的装载/load_elf_binary()函数解析](http://blog.sina.com.cn/s/blog_d9889c5b0101e66i.html)


[ELF文件的加载过程(load_elf_binary函数详解)--Linux进程的管理与调度（十三）](https://blog.csdn.net/gatieme/article/details/51628257)


[linux 可执行文件创建 学习笔记](https://blog.csdn.net/titer1/article/details/45008793)




[回到目录](#目录)