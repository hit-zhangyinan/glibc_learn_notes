# pthread_spin_xxx

## `pthread_spin_lock`

```c
// File: https://elixir.bootlin.com/glibc/glibc-2.41.9000/source/nptl/pthread_spin_lock.c
versioned_symbol (libc, __pthread_spin_lock, pthread_spin_lock, GLIBC_2_34);

int
__pthread_spin_lock (pthread_spinlock_t *lock)
{
    ...
}

typedef volatile int pthread_spinlock_t;

#define atomic_spin_nop() __asm ("pause")
# define atomic_spin_nop() do { /* nothing */ } while (0)
```

pause 汇编指令的作用 (Intel SDM Volume 2B)

* Improves the performance of spin-wait loops.

  It is recommended that a `PAUSE` instruction be placed in all spin-wait loops.

* Inserting a pause instruction in a spin-wait loop greatly reduces the processor's power consumption.

Operation: Execute_Next_Instruction(DELAY);

## `pthread_spin_unlock`

```c
versioned_symbol (libc, __pthread_spin_unlock, pthread_spin_unlock,
                  GLIBC_2_34);

int
__pthread_spin_unlock (pthread_spinlock_t *lock)
{
  /* The atomic_store_release synchronizes-with the atomic_exchange_acquire
     or atomic_compare_exchange_weak_acquire in pthread_spin_lock /
     pthread_spin_trylock.  */
  atomic_store_release (lock, 0);
  return 0;
}

# define atomic_store_release(mem, val) \
  do {                                                         \
    __atomic_check_size_ls((mem));                             \
    __atomic_store_n ((mem), (val), __ATOMIC_RELEASE);         \
  } while (0)

#  define atomic_store_release(mem, val) \
   do {                                                       \
     atomic_thread_fence_release ();                          \
     atomic_store_relaxed ((mem), (val));                     \
   } while (0)

# define atomic_store_relaxed(mem, val) \
  do {                                                         \
    __atomic_check_size_ls((mem));                             \
    __atomic_store_n ((mem), (val), __ATOMIC_RELAXED);         \
  } while (0)
```

[gcc __atomic Builtins](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)

```c
void __atomic_store_n (type *ptr, type val, int memorder)
{
    (void)(memorder);   // don't care it
    *ptr = val;
}
```

This built-in function implements an atomic store operation. It writes `val` into `*ptr`.

The valid memory order variants are `__ATOMIC_RELAXED`, `__ATOMIC_SEQ_CST`, and `__ATOMIC_RELEASE`.

经过以上分析，可以得到 `pthread_spin_unlock` 的等价形式：

```c
// atomic
int pthread_spin_unlock (pthread_spinlock_t *lock)
{
    *lock = 0;
    return 0;
}
```
