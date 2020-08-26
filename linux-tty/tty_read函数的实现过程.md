## tty_read函数的实现过程

#### 目录

[0. 简述](#0-简述)

[1. tty_read](#1-tty_read)

[2. 控制台读取](#2-控制台读取)

[3. 单次读取限制](#3-单次读取限制)

[4. 读取缓冲区](#4-读取缓冲区)

[5. 缓冲区阀门](#5-缓冲区阀门)

[6. 数据读取](#6-数据读取)

[7. 总结](#7-总结)

[8. 参考资料](#8-参考资料)


[TOC]


#### 0. 简述


tty核心中的读写函数tty_read和tty_write，内部调用的是线路规程的read和write操作；


tty_write()函数通过ld->ops->write()从tty核心进入到下一层的线路规程，调用n_tty_write()函数，通过tty->ops->write()函数继续进入到下一层uart驱动，使用uart_write()函数将数据从用户空间写入到终端；


tty_read()函数通过ld->ops->read()从tty核心进入到下一层的线路规程，调用n_tty_read()函数；在n_tty_read()函数中不需要通过下一层的uart_read()函数读取，而是从线路规程的数据环形缓冲区中读取数据到用户空间；


本文先忽略tty的写入操作，将重点解析tty线路规程中的n_tty_read()函数的操作；




#### 1. tty_read


```c
// tty_io.c
static ssize_t tty_read(struct file *file, char __user *buf, size_t count,
            loff_t *ppos)
{
    struct tty_struct *tty = file_tty(file);
    struct tty_ldisc *ld;

    /* We want to wait for the line discipline to sort out in this
       situation */
    ld = tty_ldisc_ref_wait(tty);
    if (ld->ops->read)
        i = ld->ops->read(tty, file, buf, count);
    else
        i = -EIO;
    tty_ldisc_deref(ld);
	......
}
```


在tty核心层的tty_read()函数中，通过tty_ldisc_ref_wait()函数从tty_struct结构体中获取到tty_ldisc结构指针，直接使用tty_ldisc_ops结构中的ld->ops->read函数完成tty的read操作，从而进入到驱动的下一层，tty线路规程；


```c
ld->ops->read(tty, file, buf, count)
```




```CQL
// n_tty.c
struct tty_ldisc_ops tty_ldisc_N_TTY = {
    .name            = "n_tty",
    .read            = n_tty_read,
    .write           = n_tty_write,
    ......
};
```




#### 2. 控制台读取


在控制台中，ld->ops->read对应的是tty_ldisc_N_TTY结构中的n_tty_read()函数；n_tty_read()函数与n_tty_write()不同，不是通过下一层的read函数实现（如：uart_write），n_tty_read()函数是从缓冲区中读取数据的，函数实现比较长，为了尽可能多地详细解析该函数，下面将该函数截取成多段进行解析；


```c
static ssize_t n_tty_read(struct tty_struct *tty, struct file *file,
             unsigned char __user *buf, size_t nr)
{
    struct n_tty_data *ldata = tty->disc_data;
    unsigned char __user *b = buf;
    DEFINE_WAIT_FUNC(wait, woken_wake_function);
    int c;
    int minimum, time;
    ssize_t retval = 0;
    long timeout;
    int packet;
    size_t tail;

    c = job_control(tty, file);
    if (c < 0)
        return c;

    /*
     *  Internal serialization of reads.
     */
    if (file->f_flags & O_NONBLOCK) {
        if (!mutex_trylock(&ldata->atomic_read_lock))
            return -EAGAIN;
    } else {
        if (mutex_lock_interruptible(&ldata->atomic_read_lock))
            return -ERESTARTSYS;
    }

    down_read(&tty->termios_rwsem);
```


n_tty_read()函数的开始部分，用于一些数据的初始化和校验；


```c
    DEFINE_WAIT_FUNC(wait, woken_wake_function);
    down_read(&tty->termios_rwsem);

    minimum = time = 0;
    timeout = MAX_SCHEDULE_TIMEOUT;
    if (!ldata->icanon) {
        minimum = MIN_CHAR(tty);
        if (minimum) {
            time = (HZ / 10) * TIME_CHAR(tty);
            if (time)
                ldata->minimum_to_wake = 1;
            else if (!waitqueue_active(&tty->read_wait) ||
                 (ldata->minimum_to_wake > minimum))
                ldata->minimum_to_wake = minimum;
        } else {
            timeout = (HZ / 10) * TIME_CHAR(tty);
            ldata->minimum_to_wake = minimum = 1;
        }
    }

    packet = tty->packet;
    tail = ldata->read_tail;

    add_wait_queue(&tty->read_wait, &wait);
```


当输入缓冲区中的数据超过了最低限度数据量minimum_to_wake时，要唤醒正在等待从该设备读取数据的进程；minimum_to_wake的值一般都是1，即缓冲区中的数据量超过1个，就要唤醒读取进程；


#### 3. 单次读取限制


minimum = MIN_CHAR(tty)操作，获取termios.c_cc[VMIN]数组的值，作为本次读取操作能够读取到的最大数据量；termios.c_cc[VMIN]数组的值，可以在打开控制台之后通过设置termios参数进行设置；


```c
// linux/tty.h
#define MIN_CHAR(tty) ((tty)->termios.c_cc[VMIN])
```


从同一个终端设备读取的操作应该是互斥的，所以要放在临界区中；还要在当前进程的系统堆栈中准备一个wait_queue_t数据结构wait，并挂入到目标终端的读取等待队列read_wait中，使终端设备的驱动程序在有数据可以读取时可以唤醒这个进程；如果终端设备的输入缓冲区中已经有数据，不需要进入睡眠，可以在读取到了数据之后再把它从队列里去掉即可；


```c
    while (nr) {
        /* First test for status change. */
        if (packet && tty->link->ctrl_status) {
            unsigned char cs;
            if (b != buf)
                break;
            spin_lock_irq(&tty->link->ctrl_lock);
            cs = tty->link->ctrl_status;
            tty->link->ctrl_status = 0;
            spin_unlock_irq(&tty->link->ctrl_lock);
            if (tty_put_user(tty, cs, b++)) {
                retval = -EFAULT;
                b--;
                break;
            }
            nr--;
            break;
        }
```


伪终端设备可以通过ioctl()系统调用将主从的通信方式设置为“packet”模式，此时packet值为1；此种情况和控制台没有什么关系，我也不懂，所以这部分跳过；


```c
    unsigned char __user *b = buf;
    ......
        if (((minimum - (b - buf)) < ldata->minimum_to_wake) &&
            ((minimum - (b - buf)) >= 1))
            ldata->minimum_to_wake = (minimum - (b - buf));
```


指针b定义时指向的是用户空间的buf缓存，用来保存读取到的数据，随着字符的读出而向后递增；（b-buf）是已经读出的字符数；ldata->minimum_to_wake的值在读取过程中会趋近于1；


```c
        if (!input_available_p(tty, 0)) {
            up_read(&tty->termios_rwsem);
            tty_buffer_flush_work(tty->port);
            down_read(&tty->termios_rwsem);
            if (!input_available_p(tty, 0)) {
                if (test_bit(TTY_OTHER_CLOSED, &tty->flags)) {
                    retval = -EIO;
                    break;
                }
                if (tty_hung_up_p(file))
                    break;
                if (!timeout)
                    break;
                if (file->f_flags & O_NONBLOCK) {
                    retval = -EAGAIN;
                    break;
                }
                if (signal_pending(current)) {
                    retval = -ERESTARTSYS;
                    break;
                }
                up_read(&tty->termios_rwsem);

                timeout = wait_woken(&wait, TASK_INTERRUPTIBLE,
                        timeout);

                down_read(&tty->termios_rwsem);
                continue;
            }
        }
```


在input_available_p()函数中会检查输入缓冲区中是否有数据，在“规范模式”下，检查的是经过加工后的数据数量，在原始模式下则是检查原始字符的数量；


```c
static inline int input_available_p(struct tty_struct *tty, int poll)
{
    struct n_tty_data *ldata = tty->disc_data;
    int amt = poll && !TIME_CHAR(tty) && MIN_CHAR(tty) ? MIN_CHAR(tty) : 1;

    if (ldata->icanon && !L_EXTPROC(tty))
        return ldata->canon_head != ldata->read_tail;
    else
        return ldata->commit_head - ldata->read_tail >= amt;
}
```


如果缓冲区中没有数据可以读取，当前进程要休眠等待，直到缓冲区有数据可以读取时才会被唤醒；


为了能够讲述这部分环境，假定此时缓冲区中没有数据，当前进程进入休眠；之后缓冲区有数据时，当前进程被唤醒并调度运行；


```c
        if (ldata->icanon && !L_EXTPROC(tty)) {
            retval = canon_copy_from_read_buf(tty, &b, &nr);
            if (retval)
                break;
```


当前进程被唤醒时，此时缓冲区中应该有可以读取的数据；在规范模式下，缓冲区中的字符是经过加工了的，要累积到一个缓冲行才会唤醒等待读出的进程（缓冲行，即碰到'\n'字符）；此时的读取操作在canon_copy_from_read_buf()函数中完成；canon_copy_from_read_buf()函数的实现在下文讲述；


```c
        } else {
            int uncopied;

            uncopied = copy_from_read_buf(tty, &b, &nr);
            uncopied += copy_from_read_buf(tty, &b, &nr);
            if (uncopied) {
                retval = -EFAULT;
                break;
            }
        }
```


在非规范模式下，缓冲区中的字符是未经加工的，不存在缓冲行的概念，在原始模式可以把字符'\0'复制到用户空间，这里使用copy_from_read_buf()函数进行成片的拷贝；由于缓冲区是环形的，缓冲的字符可能跨越环形缓冲区的结尾，被分割成两部分，所以要使用copy_from_read_buf()函数两次；copy_from_read_buf()函数的实现在后面讲述；



#### 4. 读取缓冲区

在控制台的线路规程中，使用struct n_tty_data结构体表示该设备的数据；其中包含的read_buf成员作为读取的缓冲区使用；

```c
// n_tty.c
struct n_tty_data {
    /* producer-published */
    size_t read_head;
    
    /* shared by producer and consumer */
    char read_buf[N_TTY_BUF_SIZE];
    
    /* consumer-published */
    size_t read_tail;
}
```

read_buf是一个N_TTY_BUF_SIZE字节的数组；

```c
// linux/tty.h
#define N_TTY_BUF_SIZE 4096
```

定义的read_buf缓冲区是线性数组，但是却是作为环形缓冲区使用的；read_head成员是环形缓冲区空闲位置的开始，产生数据的进程从read_head位置开始往缓冲区写入数据；read_tail成员是环形缓冲区保存数据位置的开始，读取数据的进程从read_tail位置开始从缓冲区读取数据；

以下针对具体的读取操作进行说明；

```c
tty->read_buf[]	// 环形缓冲区；
tty->read_tail	// 指向缓冲区当前可以读取的第一个字符；
tty->read_head	// 指向缓冲区当前可以写入的第一个地址；
```

read_cnt是通过缓冲区的read_head-read_tail计算得到，表示缓冲行中的已保存字符个数；

```c
// n_tty.c
static inline size_t read_cnt(struct n_tty_data *ldata)
{
    return ldata->read_head - ldata->read_tail;
}
```

n_tty_data结构体在线路规程被打开时申请结构体空间，并进行初始化；

```c
// n_tty.c
static int n_tty_open(struct tty_struct *tty)
{
    struct n_tty_data *ldata;

    /* Currently a malloc failure here can panic */
    ldata = vzalloc(sizeof(*ldata));
    if (!ldata)
        return -ENOMEM;

    ldata->overrun_time = jiffies;
    mutex_init(&ldata->atomic_read_lock);
    mutex_init(&ldata->output_lock);

    tty->disc_data = ldata;
    tty->closing = 0; 
    /* indicate buffer work may resume */
    clear_bit(TTY_LDISC_HALTED, &tty->flags);
    n_tty_set_termios(tty, NULL);
    tty_unthrottle(tty);
    return 0;
}
```




#### 5. 缓冲区阀门


```c
        n_tty_check_unthrottle(tty);
```


缓冲区是环形的，空间也是有限的；如果缓冲区数据来的太快，应用程序来不及从缓冲区读取数据；为了防止环形缓冲区中的数据被覆盖，底层的驱动程序可能因为缓冲区已满而暂时关闭了“阀门”，禁止数据继续进入缓冲区；


```c
static void n_tty_check_unthrottle(struct tty_struct *tty)
{
    if (tty->driver->type == TTY_DRIVER_TYPE_PTY) {
        if (chars_in_buffer(tty) > TTY_THRESHOLD_UNTHROTTLE)
            return;
        if (!tty->count)
            return;
        n_tty_kick_worker(tty);
        tty_wakeup(tty->link);
        return;
    }

    while (1) {
        int unthrottled;
        tty_set_flow_change(tty, TTY_UNTHROTTLE_SAFE);
        if (chars_in_buffer(tty) > TTY_THRESHOLD_UNTHROTTLE)
            break;
        if (!tty->count)
            break;
        n_tty_kick_worker(tty);
        unthrottled = tty_unthrottle_safe(tty);
        if (!unthrottled)
            break;
    }
    __tty_set_flow_change(tty, 0);
}
```


在读取过程中，通过chars_in_buffer()检查缓冲区，如果缓冲区中剩余的字符数量减少到了关闭阀门的要求以下（数量小于TTY_THRESHOLD_UNTHROTTLE），则在n_tty_check_unthrottle()函数中通过调用tty_unthrottle_safe()函数重新打开“阀门”，数据就可以重新进入缓冲区；


```c
        if (b - buf >= minimum)
            break;
        if (time)
            timeout = time;
    }
    if (tail != ldata->read_tail)
        n_tty_kick_worker(tty);
    up_read(&tty->termios_rwsem);

    remove_wait_queue(&tty->read_wait, &wait);
    if (!waitqueue_active(&tty->read_wait))
        ldata->minimum_to_wake = minimum;

    mutex_unlock(&ldata->atomic_read_lock);
```


当前进程已经读取到了所要求的输入，需要放在临界区的操作已完成，读取操作已经完成，将当前进程从等待read_wait中移除；


```c
    if (b - buf)
        retval = b - buf;

    return retval;
}
```


指针buf指向用户空间的缓冲区，指针b指向该缓冲区中的下一个空闲位置，（b-buf）是已经读入buf缓冲区中的字符数量；如果(b - buf >= minimum)，则本次读取结束；


n_tty_read()函数的参数nr是表示用户空间缓冲区的大小，是读取字符数量的上限；n_tty_read()函数以读取到的字符数量为返回值；



#### 6. 数据读取




以下针对具体的读取操作进行说明；




##### 1) 规范模式下的读取


canon_copy_from_read_buf()函数只有在规范模式下会被调用，该函数按缓冲行将数据从tty缓冲区中读取到用户空间；


```c
static int canon_copy_from_read_buf(struct tty_struct *tty,
                    unsigned char __user **b,
                    size_t *nr)
{
    struct n_tty_data *ldata = tty->disc_data;
    size_t n, size, more, c;
    size_t eol;
    size_t tail;
    int ret, found = 0;

    /* N.B. avoid overrun if nr == 0 */
    if (!*nr)
        return 0;

    n = min(*nr + 1, smp_load_acquire(&ldata->canon_head) - ldata->read_tail);

    tail = ldata->read_tail & (N_TTY_BUF_SIZE - 1);
    size = min_t(size_t, tail + n, N_TTY_BUF_SIZE);

    n_tty_trace("%s: nr:%zu tail:%zu n:%zu size:%zu\n",
            __func__, *nr, tail, n, size);

    eol = find_next_bit(ldata->read_flags, size, tail);
    more = n - (size - tail);
    if (eol == N_TTY_BUF_SIZE && more) {
        /* scan wrapped without finding set bit */
        eol = find_next_bit(ldata->read_flags, more, 0);
        if (eol != more)
            found = 1;
    } else if (eol != size)
        found = 1;

    size = N_TTY_BUF_SIZE - tail;
    n = eol - tail;
    if (n > N_TTY_BUF_SIZE)
        n += N_TTY_BUF_SIZE;
    c = n + found;

    if (!found || read_buf(ldata, eol) != __DISABLED_CHAR) {
        c = min(*nr, c);
        n = c;
    }

    n_tty_trace("%s: eol:%zu found:%d n:%zu c:%zu size:%zu more:%zu\n",
            __func__, eol, found, n, c, size, more);

    if (n > size) {
        ret = tty_copy_to_user(tty, *b, read_buf_addr(ldata, tail), size);
        if (ret)
            return -EFAULT;
        ret = tty_copy_to_user(tty, *b + size, ldata->read_buf, n - size);
    } else
        ret = tty_copy_to_user(tty, *b, read_buf_addr(ldata, tail), n);

    if (ret)
        return -EFAULT;
    *b += n;
    *nr -= n;

    if (found)
        clear_bit(eol, ldata->read_flags);
    smp_store_release(&ldata->read_tail, ldata->read_tail + c);

    if (found) {
        if (!ldata->push)
            ldata->line_start = ldata->read_tail;
        else
            ldata->push = 0;
        tty_audit_push(tty);
    }
    return 0;
}
```


最终通过tty_copy_to_user()函数中的copy_to_user()函数完成数据的拷贝；


```c
static inline int tty_copy_to_user(struct tty_struct *tty,
                    void __user *to,
                    const void *from,
                    unsigned long n)
{
    struct n_tty_data *ldata = tty->disc_data;

    tty_audit_add_data(tty, from, n, ldata->icanon);
    return copy_to_user(to, from, n);
}
```




##### 2) 非规范模式下的读取


copy_from_read_buf()函数在非规范模式下，将数据从tty缓冲区中直接读取到用户空间，该函数会被调用两次，第一次是从tty->disc_data->read_tail指针指向的位置到缓冲区结尾，第二次是从缓冲区开头，到tty->disc_data->read_head指针指向的位置；该函数的读取操作需要在ldata->atomic_read_lock信号锁的保护下进行；


```c
static int copy_from_read_buf(struct tty_struct *tty,
                      unsigned char __user **b,
                      size_t *nr)
{
    struct n_tty_data *ldata = tty->disc_data;
    int retval;
    size_t n;
    bool is_eof;
    size_t head = smp_load_acquire(&ldata->commit_head);
    size_t tail = ldata->read_tail & (N_TTY_BUF_SIZE - 1);

    retval = 0;
    n = min(head - ldata->read_tail, N_TTY_BUF_SIZE - tail);
    n = min(*nr, n);
    if (n) {
        retval = copy_to_user(*b, read_buf_addr(ldata, tail), n);
        n -= retval;
        is_eof = n == 1 && read_buf(ldata, tail) == EOF_CHAR(tty);
        tty_audit_add_data(tty, read_buf_addr(ldata, tail), n,
                ldata->icanon);
        smp_store_release(&ldata->read_tail, ldata->read_tail + n);
        /* Turn single EOF into zero-length read */
        if (L_EXTPROC(tty) && ldata->icanon && is_eof &&
            (head == ldata->read_tail))
            n = 0;
        *b += n;
        *nr -= n;
    }
    return retval;
}
```




#### 7. 总结


一般情况下，典型的读取终端过程可以分为以下三部分：


1. 当前进程准备从终端缓冲区读取数据，但是缓冲区还没有足够字符可以读取，进入睡眠；

2. 如果有输入字符，底层驱动将足够的字符写入缓冲区之后，把睡眠的进程唤醒；

3. 睡眠的读取进程被唤醒后，开始完成读取操作；




#### 8. 参考资料


《Linux内核情景分析》----控制台驱动





[回到目录](目录)

