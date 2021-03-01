# concurrency 요약
핵심: 어떻게 multiple threads를 잘 돌릴까

## Threads
- threads vs. process: 한 thread는 같은 process 안에 있는 다른 threads와 memory image 상의 code(and data), heap 부분을 공유. 단 PC(program counter), stack 등은 다름. 

## Locks
- SW만으로는 온전히 atomicity 지원하기 어려워서 HW의 도움 필요. 아래와 같음. 

```C
int TestAndSet (int *old_ptr, int new) {
    int old = *old_ptr;
    *old_ptr = new;
    return old;
}
```

위 코드를 이용해서 아래와 같은 lock 사용 가능

```C
typedef struct __lock_t {
    int flag;
} lock_t;

void init(lock_t *lock) {
    lock->flag = 0;
}

void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag)) {

    }
}
```

`TestAndSet` 대신 아래와 같이 `CompareAndSwap`을 사용할 수도 있음. 


```C
int CompareAndSwap(int *ptr, int expected, int new) {
    int original = *ptr;
    if (original == expected)
        *ptr = new;
    return original;
}
```

```C
void lock(lock_t *lock) {
    while (CompareAndSwap(&lock->flag, 0, 1) == 1)
        ; //spin
}
```

## Condition Variables(CV)


## Semaphores

