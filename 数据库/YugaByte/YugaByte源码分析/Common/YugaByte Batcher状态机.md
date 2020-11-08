# 提纲
[toc]

## Batcher有哪些状态
类Batcher中有一个成员BatcherState，这是一个枚举类型，包括的状态值有：
```
kGatheringOps，
kResolvingTablets，
kTransactionPrepare，
kTransactionReady，
kComplete，
kAborted，
```

通常，BatcherState的状态变迁为kGatheringOps -> kResolvingTablets -> kTransactionPrepare -> kTransactionReady -> kComplete。BatcherState的初始状态为kGatheringOps，
kResolvingTablets，任何一个状态都可以切换到kAborted状态。


