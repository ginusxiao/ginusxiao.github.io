# 准备知识
## ucontext function family
### 参考
[wiki](https://en.wikipedia.org/wiki/Setcontext)

## setjmp/logjmp
### 参考
[wiki](https://en.wikipedia.org/wiki/Setjmp.h)

## qemu coroutine
### coroutine
[coroutine核心机制](http://www.cnblogs.com/VincentXu/p/3350389.html)

### qemu lock
CoMutex: act as the functionality of pthread_mutex_t, but to synchronize coroutines

CoQueue: act as the functionality of pthread_cond_t, used to queue coroutines and execute them later

CoRwlock: act as the functionality of pthread_rwlock_t, it support upgrade/downgrade from readlock/writelock to writelock/readlock








