# futex 系列 api

## `futex_wait`

[futex_wait](https://elixir.bootlin.com/glibc/glibc-2.41.9000/source/sysdeps/nptl/futex-internal.h#L144)

```c
static __always_inline int
futex_wait (unsigned int *futex_word, unsigned int expected, int private)
{
  int err = lll_futex_timed_wait (futex_word, expected, NULL, private);
  switch (err)
    {
    case 0:
    case -EAGAIN:
    case -EINTR:
      return -err;

    case -ETIMEDOUT: /* Cannot have happened as we provided no timeout.  */
    case -EFAULT: /* Must have been caused by a glibc or application bug.  */
    case -EINVAL: /* Either due to wrong alignment or due to the timeout not
             being normalized.  Must have been caused by a glibc or
             application bug.  */
    case -ENOSYS: /* Must have been caused by a glibc bug.  */
    /* No other errors are documented at this time.  */
    default:
      futex_fatal_error ();
    }
}
```

[lll_futex_timed_wait](https://elixir.bootlin.com/glibc/glibc-2.41.9000/source/sysdeps/nptl/lowlevellock-futex.h#L75)

```c
# define lll_futex_timed_wait(futexp, val, timeout, private)     \
  lll_futex_syscall (4, futexp,                                 \
             __lll_private_flag (FUTEX_WAIT, private),  \
             val, timeout)

# define __lll_private_flag(fl, private) \
  (((fl) | FUTEX_PRIVATE_FLAG) ^ (private))

# define lll_futex_syscall(nargs, futexp, op, ...)                      \
  ({                                                                    \
    long int __ret = INTERNAL_SYSCALL (futex, nargs, futexp, op, __VA_ARGS__);    \
    (__glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (__ret))            \
     ? -INTERNAL_SYSCALL_ERRNO (__ret) : 0);                        \
  })
```

```c
#define INTERNAL_SYSCALL(name, nr, args...)     \
    internal_syscall##nr (SYS_ify (name), args)

/* For Linux we can use the system call table in the header file
    /usr/include/asm/unistd.h
    of the kernel.  But these symbols do not follow the SYS_* syntax
    so we have to redefine the `SYS_ify' macro here.  */
#define SYS_ify(syscall_name)   __NR_##syscall_name

#define internal_syscall4(number, arg1, arg2, arg3, arg4)        \
({                                    \
    unsigned long int resultvar;                    \
    TYPEFY (arg4, __arg4) = ARGIFY (arg4);                 \
    TYPEFY (arg3, __arg3) = ARGIFY (arg3);                 \
    TYPEFY (arg2, __arg2) = ARGIFY (arg2);                 \
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);                 \
    register TYPEFY (arg4, _a4) asm ("r10") = __arg4;            \
    register TYPEFY (arg3, _a3) asm ("rdx") = __arg3;            \
    register TYPEFY (arg2, _a2) asm ("rsi") = __arg2;            \
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;            \
    asm volatile (                            \
    "syscall\n\t"                            \
    : "=a" (resultvar)                            \
    : "0" (number), "r" (_a1), "r" (_a2), "r" (_a3), "r" (_a4)        \
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);            \
    (long int) resultvar;                        \
})

// File: /usr/include/x86_64-linux-gnu/asm/unistd_64.h
#define __NR_futex 202

```

```c
__SC_3264(__NR_futex, sys_futex_time32, sys_futex)

#if __BITS_PER_LONG == 32 || defined(__SYSCALL_COMPAT)
#define __SC_3264(_nr, _32, _64) __SYSCALL(_nr, _32)
#else
#define __SC_3264(_nr, _32, _64) __SYSCALL(_nr, _64)
#endif
```

```c
SYSCALL_DEFINE6(futex, u32 __user *, uaddr, int, op, u32, val,
        const struct __kernel_timespec __user *, utime,
        u32 __user *, uaddr2, u32, val3)
{
    return do_futex(...);
}

int futex_wait(u32 __user *uaddr, unsigned int flags, u32 val, ktime_t *abs_time, u32 bitset)
{
    __futex_wait(...);
}

int __futex_wait(u32 __user *uaddr, unsigned int flags, u32 val,
            struct hrtimer_sleeper *to, u32 bitset)
{
    /* futex_queue and wait for wakeup, timeout, or a signal. */
    futex_wait_queue(hb, &q, to);
}

void futex_wait_queue(struct futex_hash_bucket *hb, struct futex_q *q,
            struct hrtimer_sleeper *timeout)
{
    futex_queue(q, hb);
}

void __futex_queue(struct futex_q *q, struct futex_hash_bucket *hb)
{
    int prio;

    /*
     * The priority used to register this element is
     * - either the real thread-priority for the real-time threads
     * (i.e. threads with a priority lower than MAX_RT_PRIO)
     * - or MAX_RT_PRIO for non-RT threads.
     * Thus, all RT-threads are woken first in priority order, and
     * the others are woken last, in FIFO order.
     */
    prio = min(current->normal_prio, MAX_RT_PRIO);

    plist_node_init(&q->list, prio);
    plist_add(&q->list, &hb->chain);
    q->task = current;
}
```

## `futex_wake`

```c
static __always_inline void
futex_wake (unsigned int* futex_word, int processes_to_wake, int private)
{
  int res = lll_futex_wake (futex_word, processes_to_wake, private);
  /* No error.  Ignore the number of woken processes.  */
  if (res >= 0)
    return;
  switch (res)
    {
    case -EFAULT: /* Could have happened due to memory reuse.  */
    case -EINVAL: /* Could be either due to incorrect alignment (a bug in
             glibc or in the application) or due to memory being
             reused for a PI futex.  We cannot distinguish between the
             two causes, and one of them is correct use, so we do not
             act in this case.  */
      return;
    case -ENOSYS: /* Must have been caused by a glibc bug.  */
    /* No other errors are documented at this time.  */
    default:
      futex_fatal_error ();
    }
}
```

宏的匹配：

### glibc 里的宏

```c
// File: https://elixir.bootlin.com/glibc/glibc-2.41.9000/source/sysdeps/nptl/lowlevellock-futex.h#L28
#define FUTEX_WAIT            0
#define FUTEX_WAKE            1
#define FUTEX_REQUEUE         3
#define FUTEX_CMP_REQUEUE     4
#define FUTEX_WAKE_OP         5
#define FUTEX_OP_CLEAR_WAKE_IF_GT_ONE    ((4 << 24) | 1)
#define FUTEX_LOCK_PI          6
#define FUTEX_UNLOCK_PI        7
#define FUTEX_TRYLOCK_PI       8
#define FUTEX_WAIT_BITSET      9
#define FUTEX_WAKE_BITSET     10
#define FUTEX_WAIT_REQUEUE_PI   11
#define FUTEX_CMP_REQUEUE_PI    12
#define FUTEX_LOCK_PI2          13
#define FUTEX_PRIVATE_FLAG     128
#define FUTEX_CLOCK_REALTIME   256
```

### linux 内核里的宏

```c
// File: https://elixir.bootlin.com/linux/v6.11.5/source/include/uapi/linux/futex.h#L11
#define FUTEX_WAIT           0
#define FUTEX_WAKE           1
#define FUTEX_FD             2
#define FUTEX_REQUEUE        3
#define FUTEX_CMP_REQUEUE    4
#define FUTEX_WAKE_OP        5
#define FUTEX_LOCK_PI        6
#define FUTEX_UNLOCK_PI      7
#define FUTEX_TRYLOCK_PI     8
#define FUTEX_WAIT_BITSET    9
#define FUTEX_WAKE_BITSET    10
#define FUTEX_WAIT_REQUEUE_PI   11
#define FUTEX_CMP_REQUEUE_PI    12
#define FUTEX_LOCK_PI2          13

#define FUTEX_PRIVATE_FLAG      128
#define FUTEX_CLOCK_REALTIME    256
```

```c
int futex_wake(u32 __user *uaddr, unsigned int flags, int nr_wake, u32 bitset)
{
    wake_up_q(&wake_q);
}
```
