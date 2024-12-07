# pthread_xxx 系列 api 的源码学习

## `pthread_create`

```c
// file: nptl/pthread_create.c

int
__pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
		      void *(*start_routine) (void *), void *arg)
{
    ...
    const struct pthread_attr *iattr = (struct pthread_attr *) attr;
    ...
    struct pthread *pd = NULL;
    int err = allocate_stack (iattr, &pd, &stackaddr, &stacksize);
    ...
    pd->start_routine = start_routine;
    pd->arg = arg;
    pd->joinid = iattr->flags & ATTR_FLAG_DETACHSTATE ? pd : NULL;
    pd->schedpolicy = self->schedpolicy;
    pd->schedparam = self->schedparam;
    ...
    /* Setup tcbhead.  */
    tls_setup_tcbhead (pd);
    ...
    /* Pass the descriptor to the caller.  */
    *newthread = (pthread_t) pd;
    LIBC_PROBE (pthread_create, 4, newthread, attr, start_routine, arg);
    ...
    /* Start the thread.  */
    retval = create_thread (pd, iattr, &stopped_start, stackaddr,
			      stacksize, &thread_ran);

}
```

```c
static int create_thread (struct pthread *pd, const struct pthread_attr *attr,
			  bool *stopped_start, void *stackaddr,
			  size_t stacksize, bool *thread_ran)
{
    const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
			   | CLONE_SIGHAND | CLONE_THREAD
			   | CLONE_SETTLS | CLONE_PARENT_SETTID
			   | CLONE_CHILD_CLEARTID
			   | 0);
    ...
    struct clone_args args =
    {
        .flags = clone_flags,
        .pidfd = (uintptr_t) &pd->tid,
        .parent_tid = (uintptr_t) &pd->tid,
        .child_tid = (uintptr_t) &pd->tid,
        .stack = (uintptr_t) stackaddr,
        .stack_size = stacksize,
        .tls = (uintptr_t) tp,
    };
    int ret = __clone_internal (&args, &start_thread, pd);
    ...
    int res = INTERNAL_SYSCALL_CALL (sched_setaffinity, pd->tid,
					   attr->extension->cpusetsize,
					   attr->extension->cpuset);
    ...
    int res = INTERNAL_SYSCALL_CALL (sched_setscheduler, pd->tid,
					   pd->schedpolicy, &pd->schedparam);
}
```

```c
int
__clone_internal (struct clone_args *cl_args,
		  int (*func) (void *arg), void *arg) { }
```

```c
static int _Noreturn start_thread (void *arg)
{
    struct pthread *pd = arg;
    ...
    LIBC_PROBE (pthread_start, 3, (pthread_t) pd, pd->start_routine, pd->arg);
    ...
}
```

## `pthread_join`

```c
int
___pthread_join (pthread_t threadid, void **thread_return)
{
  return __pthread_clockjoin_ex (threadid, thread_return, 0 /* Ignored */,
				 NULL, true);
}
```

```c
int
__pthread_clockjoin_ex (pthread_t threadid, void **thread_return,
                        clockid_t clockid,
                        const struct __timespec64 *abstime, bool block)
{
    struct pthread *pd = (struct pthread *) threadid;
    ...
    /* Is the thread joinable?.  */
    if (IS_DETACHED (pd))
    /* We cannot wait for the thread.  */
        return EINVAL;
    ...
    LIBC_PROBE (pthread_join, 1, threadid);
    ...
    LIBC_PROBE (pthread_join_ret, 3, threadid, result, pd_result);
}
```

## `pthread_exit`

```c
void __pthread_exit (void *value)
{
    __do_cancel ();
}
```

## `pthread_setcancelstate`

```c
int
__pthread_setcancelstate (int state, int *oldstate)
{
    volatile struct pthread *self;

    if (state < PTHREAD_CANCEL_ENABLE || state > PTHREAD_CANCEL_DISABLE)
        return EINVAL;

    self = THREAD_SELF;

    int oldval = atomic_load_relaxed (&self->cancelhandling);
    ...
    atomic_compare_exchange_weak_acquire (&self->cancelhandling,
						&oldval, newval);
    ...
}
```

## `pthread_setcanceltype`

```c
int
__pthread_setcanceltype (int type, int *oldtype)
{
    if (type < PTHREAD_CANCEL_DEFERRED || type > PTHREAD_CANCEL_ASYNCHRONOUS)
        return EINVAL;

    volatile struct pthread *self = THREAD_SELF;

    int oldval = atomic_load_relaxed (&self->cancelhandling);
    ...
    atomic_compare_exchange_weak_acquire (&self->cancelhandling,
						&oldval, newval);
    ...
}
```

## `pthread_cancel`

```c
int
__pthread_cancel (pthread_t th)
{
    volatile struct pthread *pd = (volatile struct pthread *) th;
    if (pd->tid == 0)
    /* The thread has already exited on the kernel side.  Its outcome
       (regular exit, other cancelation) has already been
       determined.  */
        return 0;

    if (pd == THREAD_SELF)
        __do_cancel ();
    else
        result = __pthread_kill_internal (th, SIGCANCEL);
}
```

## `pthread_kill`

```c
int
__pthread_kill (pthread_t threadid, int signo)
{
  /* Disallow sending the signal we use for cancellation, timers,
     for the setxid implementation.  */
  if (is_internal_signal (signo))
    return EINVAL;

  return __pthread_kill_internal (threadid, signo);
}

int
__pthread_kill_internal (pthread_t threadid, int signo)
{
    return __pthread_kill_implementation (threadid, signo, 0);
}
```

```c
static inline bool
is_internal_signal (int sig)
{
  return (sig == SIGCANCEL) || (sig == SIGSETXID);
}
```

```c
static int
__pthread_kill_implementation (pthread_t threadid, int signo, int no_tid)
{
    struct pthread *pd = (struct pthread *) threadid;
    ...
    ret = INTERNAL_SYSCALL_CALL (tgkill, __getpid (), pd->tid, signo);
}
```

## `pthread_mutex_init`

```c
static const struct pthread_mutexattr default_mutexattr =
{
    /* Default is a normal mutex, not shared between processes.  */
    .mutexkind = PTHREAD_MUTEX_NORMAL
};

int
___pthread_mutex_init (pthread_mutex_t *mutex,
		      const pthread_mutexattr_t *mutexattr)
{
    const struct pthread_mutexattr *imutexattr;
    imutexattr = ((const struct pthread_mutexattr *) mutexattr
		?: &default_mutexattr);
    ...
    memset (mutex, '\0', __SIZEOF_PTHREAD_MUTEX_T);
    /* Default values: mutex not used yet.  */
    // mutex->__count = 0;	already done by memset
    // mutex->__owner = 0;	already done by memset
    // mutex->__nusers = 0;	already done by memset
    // mutex->__spins = 0;	already done by memset
    // mutex->__next = NULL;	already done by memset
}
```

## `pthread_cond_init`

```c
int
__pthread_cond_init (pthread_cond_t *cond, const pthread_condattr_t *cond_attr)
{
    struct pthread_condattr *icond_attr = (struct pthread_condattr *) cond_attr;

    memset (cond, 0, sizeof (pthread_cond_t));
    cond->__data.__wrefs |= xxx;
}
```

## `pthread_cond_wait`

```c
int
___pthread_cond_wait (pthread_cond_t *cond, pthread_mutex_t *mutex)
{
    /* clockid is unused when abstime is NULL. */
    return __pthread_cond_wait_common (cond, mutex, 0, NULL);
}
```

## `pthread_self`

```c
pthread_t
__pthread_self (void)
{
  return (pthread_t) THREAD_SELF;
}
```
