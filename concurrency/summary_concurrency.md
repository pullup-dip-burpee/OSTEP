# concurrency 요약
핵심: 어떻게 multiple threads를 잘 돌릴까

## Quiz
1. threads 와 process는 어떻게 다른가요?
2. 

## Threads
- threads vs. process: 한 thread는 같은 process 안에 있는 다른 threads와 memory image 상의 code(and data), heap 부분을 공유. 단 PC(program counter), stack 등은 다름. 

## Locks
- 잘못된 방법: race condition 유발
  - 아래의 lock 함수 안에서 while 문이 끝난 뒤에 interrupt 발생해서 다른 thread로 switch되고, lock이 넘어간다면? 

```C
typedef struct __lock_t {
    int flag;
} lock_t;

void init(lock_t *mutex) {
    lock->flag = 0;
}

void lock(lock_t *mutex) {
    while (mutex->flag == 1) 
        ; //spin-wait (do nothing)
    mutex->flag = 1;
}

void unlock(lock_t *mutex) {
    lock->flag = 0;
}
```

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
    while (TestAndSet(&lock->flag)) 
        ;
}

void unlock(lock_t *lock) {
    lock->flag = 0;
}
```

`TestAndSet` 대신 아래와 같이 `CompareAndSwap`을 사용할 수도 있음. 이 방식은 "lock-free synchronization"등에서 나오지만, 더 강력함. 


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

- Too much spinning? 
  - spinning lock 대신 sleeping mutex 사용 가능. 
  - sleeping(yield) 하면 그만큼 CPU resource 절약되고 performance 향상됨. 다른 이유로는 priority inversion이 있는데, 가령 스레드 T1 T2 T3이 있고, 현재 T2가 lock을 가지고 작업하는 상황. T3 작업 중 T2에 의해 lock이 선점당한다면, T1이 T2보다 우선순위가 높다고 하더라도 T2가 먼저 처리됨 -> correctness에 문제 발생. 해결을 위해 priority inheritance 라는 기법이 생김. 
  - 이런 이유로 대부분의 userspace lock 구현은 sleeping mutex임. 
  - 단, OS의 경우 yield할 대상이 없으므로 kernel에서는 항상 spinning lock 사용함. OS가 spin lock을 얻으면, lock이 있는 동안은 disable interrupts 할 수 있어야 함. 다른 interrupt
  - spinning lock은 되도록 빨리 끝내는 게 좋음

```C
void init() {
    flag = 0;
}

void lock() {
    while (TestAndSet(&flag, 1) == 1)
        yield(); // give up the CPU
}

void unlock() {
    flag = 0;
}
```

- 아래는 queues를 사용하는 방법. 
  - spin만 쓰거나 즉싯 yield CPU 하는 두 방법 모두 리소스 낭비 및 starvation이 있을 수 있어서, queue를 사용해서 해결함. 
  - `park()`는 스레드를 재우고 `unpark(threadID)`는 깨움. 
  - lock_t 구조체가 queue와 guard를 가지고 있음. guard는 spin lock으로 쓰이는데, spinning에 쓰이는 시간을 제한함. 우선 thread가 guard lock을 얻으면, 실제 lock(flag)을 얻을 수 있는지 확인. 얻을 수 있으면 lock을 얻고 guard lock는 0으로 돌림. 그렇지 않으면 queue에 thread id를 넣고, guard lock을 0으로 돌리고, park()로 스레드를 재움. 
  

```C
typedef struct __lock_t {
    int flag;
    int guard;
    queue_t *q;
} lock_t

void lock_init(lock_t *m) {
    m->flag = 0;
    m->guard = 0;
    queue_init(m->q);
}

void lock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1)
        ; //acquire guard lock by spinning
    
    if(m->flag == 0) {
        m->flag = 1;
        m->guard = 0;
    }
    else {
        queue_add(m->q, gettid());
        m->guard = 0;
        park();
    }
}

void unlock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1)
        ; // acquire guard lock by spinning
    if (queue_empty(m->q)) 
        m->flag = 0;
    else
        unpark(queue_remove(m->q)); // hold lock for next thread

    m->guard = 0;
}
```
## Condition Variables(CV)


## Semaphores

