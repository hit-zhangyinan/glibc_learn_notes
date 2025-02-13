# `__atomic_` 系列 api

## pause 汇编指令 (Intel SDM Volume 2B)

* Improves the performance of spin-wait loops.

  It is recommended that a `PAUSE` instruction be placed in all spin-wait loops.

* Inserting a pause instruction in a spin-wait loop greatly reduces the processor's power consumption.

Operation: Execute_Next_Instruction(DELAY);

[gcc __atomic Builtins](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)

## `__atomic_exchange_n`

* This built-in function implements an atomic exchange operation.

  It writes `val` into `*ptr`, and returns the previous contents of `*ptr`.

  All memory order variants are valid.

使用 c 表示：

```c
type __atomic_exchange_n (type *ptr, type val, int memorder)
{
    type tmp = *ptr;
    *ptr = val;
    return tmp;
}
```

## `__atomic_store_n`

* This built-in function implements an atomic store operation.

  It writes `val` into `*ptr`.

  The valid memory order variants are `__ATOMIC_RELAXED`, `__ATOMIC_SEQ_CST`, and `__ATOMIC_RELEASE`.

使用 c 表示：

```c
void __atomic_store_n (type *ptr, type val, int memorder)
{
    (void)(memorder);   // don't care it
    *ptr = val;
}
```

## `__atomic_compare_exchange_n`

使用 c 表示：

```c
bool __atomic_compare_exchange_n (type *ptr, type *expected, type desired, bool weak, int success_memorder, int failure_memorder)
{
    if (*ptr == *expected) {
        *ptr = desired;     /* writes desired into *ptr */
        return true;        /* If desired is written into *ptr then true is returned */
    }
    else {                  /*    *ptr != *expected    */
        *expected = *ptr;   /* current contents of *ptr are written into *expected */
        return false;       /* Otherwise, false is returned */
    }
}
```

## `__atomic_load_n`

* This built-in function implements an atomic load operation.

  It returns the contents of `*ptr`.

  The valid memory order variants are `__ATOMIC_RELAXED`, `__ATOMIC_SEQ_CST`,
                                    `__ATOMIC_ACQUIRE`, and `__ATOMIC_CONSUME`.

```c
type __atomic_load_n (type *ptr, int memorder)
{
    return *ptr;
}
```
