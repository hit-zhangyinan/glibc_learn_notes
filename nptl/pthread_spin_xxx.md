# pthread_spin_xxx

## `pthread_spin_lock`

```c
// File: https://elixir.bootlin.com/glibc/glibc-2.41.9000/source/nptl/pthread_spin_lock.c
versioned_symbol (libc, __pthread_spin_lock, pthread_spin_lock, GLIBC_2_34);

typedef volatile int pthread_spinlock_t;

int __pthread_spin_lock (pthread_spinlock_t *lock)
{
    ...
}

# define atomic_exchange_acquire(mem, desired) \
  ({ __atomic_check_size((mem));                          \
  __atomic_exchange_n ((mem), (desired), __ATOMIC_ACQUIRE); })

#define atomic_spin_nop() __asm ("pause")
```

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
    /* param check operations */                             \
    __atomic_store_n ((mem), (val), memorder);         \
  } while (0)
```

可以得到 `pthread_spin_unlock` 的等价形式：

```c
// atomic
int pthread_spin_unlock (pthread_spinlock_t *lock)
{
    *lock = 0;
    return 0;
}
```
