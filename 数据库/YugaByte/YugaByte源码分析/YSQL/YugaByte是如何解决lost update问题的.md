# 提纲
[toc]

## lost update问题描述
lost update问题应该有一个形式化描述，分析YugaByte是否解决了lost update问题，也应该是面向该形式化描述进行分析，而不应该是采用特例进行。但是鉴于目前对lost update问题的形式化描述不太确定，所以找了网上关于lost update问题的例子来进行分析。

### 回滚丢失
![image](https://img2018.cnblogs.com/blog/1415873/201906/1415873-20190605112809343-999138139.png)

### 覆盖丢失
![image](https://img2018.cnblogs.com/blog/1415873/201906/1415873-20190605112833419-1781156622.png)

## YugaByte是如何解决该问题的
下面的分析以snapshot isolation为例进行分析。

### 回滚丢失
- T3: 不加锁
- T4：不加锁
- T5-1: 事务B在账户上成功加strong read + strong write锁；
- T5-2: 事务B成功写入intent DB；
- T5-3：事务B释放在账户上所加的锁；
- T6: 事务B更新status tablet上事务状态为committed，并异步的将intent record转换为regular record，但是到底什么时候转换成功是未知的；
T7：根据T6是否成功将intent record转换为regular record，处理会有所不同：
- 若尚未开始转换
    - T7-1：事务A在账户上成功加strong read + strong write锁
    - T7-2：事务A检查到和事务B的intent record之间存在锁冲突，事务B被添加到冲突列表中
    - T7-3：从status tablet获取事务B的状态为committed，但是因为事务A的read_time小于事务B的commit_time，所以鉴定为事务A和事务B之间存在不可解决的冲突
    - T7-4：提示关于事务A的错误并返回响应
    - T8：rollback，但事务A也没啥数据需要rollback的，因为事务A并未写任何记录到intent DB或者regular DB
- 若正在转换过程中
    - 因为此时intent record还没有被删除，所以跟“尚未开始转换”情况类似
- 若已经成功转换
    - T7-1：事务A在账户上成功加strong read + strong write锁
    - T7-2：事务A检查到和事务B的regular record之间存在冲突，冲突的原因是事务A的read_time小于事务B的commit_time，此冲突无法解决
    - T7-3：提示关于事务A的错误并返回响应
    - T8：rollback，但事务A也没啥数据需要rollback的，因为事务A并未写任何记录到intent DB或者regular DB

### 覆盖丢失
- T3: 不加锁
- T4：不加锁
- T5-1：事务B在账户上成功加strong read + strong write锁；
- T5-2：事务B成功写入intent DB；
- T5-3：事务B释放在账户上所加的锁；
- T6：事务B更新status tablet上事务状态为committed，并异步的将intent record转换为regular record，但是到底什么时候转换成功是未知的；
- T7：根据T6是否成功将intent record转换为regular record，处理会有所不同：
- 若尚未成功转换
    - T7-1：事务A在账户上成功加strong read + strong write锁；
    - T7-2：事务A检查到和事务B的intent record之间存在锁冲突，事务B被添加到冲突列表中
    - T7-3：从status tablet获取事务B的状态为committed，但是因为事务A的read_time小于事务B的commit_time，所以鉴定为事务A和事务B之间存在不可解决的冲突
    - T7-4：提示关于事务A的错误并返回响应
    - T8：用户执行commit，yugabyte内部执行的其实是rollback，但事务A也没啥数据需要rollback的，因为事务A并未写任何记录到intent DB或者regular DB
- 若正在转换过程中
    - 因为此时intent record还没有被删除，所以跟“尚未开始转换”情况类似
- 若已经成功转换
    - T7-1：事务A在账户上成功加strong read + strong write锁；
    - T7-2：事务A检查到和事务B的regular record之间存在冲突，冲突的原因是事务A的read_time小于事务B的commit_time，此冲突无法解决
    - T7-3：提示关于事务A的错误并返回响应
    - T8：用户执行commit，yugabyte内部执行的其实是rollback，但事务A也没啥数据需要rollback的，因为事务A并未写任何记录到intent DB或者regular DB
