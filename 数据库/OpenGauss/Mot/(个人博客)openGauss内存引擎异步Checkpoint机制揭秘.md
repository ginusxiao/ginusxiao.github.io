# 提纲
[toc]

# 简介
对于内存数据库系统来说，如何保证ACID中的Durability是一个非常关键的问题，许多现代内存数据库都采用Checkpoint机制来保证Durability。openGauss内存引擎参考CALC算法来实现其Checkpoint。

CALC(Checkpoint Asyncronously using Logical Consistency)是一个轻量级的异步的checkpoint算法，它的设计基于以下考虑：
- Checkpoint过程不应该太影响事务吞吐
- Checkpoint过程不应该为事务处理引入不可接受的时延
- Checkpoint过程需要尽量少的额外内存
    - 虽然内存价格在不断下降，但是对于纯内存数据库来说，内存资源是相对宝贵的资源，因此完全的多版本(也就是说数据库的更新不会就地更新，而是追加更新)对许多用户来实是过于奢侈的

基于上述考虑中的前2个，CALC算法采用异步执行机制。但是在Checkpoint的过程中，数据库记录仍然在不断的更新，必须有一种机制确保Checkpoint过程可以看到数据库某一时刻的一个快照。

基于上述考虑中的第3个，CALC算法为每个记录在任意时刻至多维护2个版本的数据，一个版本被称为live版本，用于表示该记录当前最新的数据，另一个版本被称为stable版本，用于表示该记录在Checkpoint过程应当被写入Checkpoint文件的数据。

# CALC算法的推理
1. 在没有发生Checkpoint的时候，所有的数据可以直接更新live版本。
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/BB3803F5DD51428A90CB4C98E9CDBF4C/114368)

    对于操作序列Op(1),Op(2),Op(3),Op(1),Op(2),Op(4)，所有的更新都直接更新live版本，即使Op(1)和Op(2)分别发生了2次，也直接在live版本上执行就地更新。

    此时stable版本虽然不存在，但是可以认为live版本就是stable版本。

    CALC算法称该阶段为rest阶段。

2. Checkpoint被触发的时刻，有些事务性更新操作可能已经完成，而有些事务性更新操作可能没有完成。对于这些没有完成的事务性更新操作，如果把它们排除在本次Checkpoint之外，则本次Checkpoint可能会将错误的记录写入Checkpoint文件中，因为这些未完成的事务性更新操作是在Checkpoint被触发之前就开始执行的，他们会直接更新live版本的数据。
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/7D82A8D1DD0B4CEDA9290F7616A78462/114416)

    在T1时刻Checkpoint被触发，此时对于操作序列Op(1),Op(2),Op(3),Op(1),Op(2),Op(4),Op(4)，第一个Op(4)及其之前的操作都已经完成，但是第二个Op(4)还没有完成，key为4的记录的live版本中保存的是第一个Op(4)更新后的结果，此时如果将第二个Op(4)操作排除在本次Checkpoint之外，则可能会发生如下情况：在Checkpoint将key为4的记录写入Checkpoint文件之前，第二个Op(4)已经完成，当Checkpoint将key为4的记录写入Checkpoint文件的时候，理应将第一个Op(4)更新后的live版本写入Checkpoint文件，但是现在的live版本中保存的是第二个Op(4)更新后的结果，所以本次Checkpoint会产生不一致的状态！

    为了解决上述问题，必须保证所有在Checkpoint被触发之前已经开始的事务性操作被包含在本次Checkpoint中。因此，CALC算法引入了prepare阶段，在Checkpoint被触发之后就进入prepare阶段，并且一直持续到Checkpoint被触发之前已经开始的所有事务性操作都完成为止。

3. 在prepare阶段，如果有新的事务性更新操作产生，则这些操是否能够直接更新live版本呢？这要看情况而定。
    - 情况1：操作所涉及的记录不存在stable版本，则必须在更新live版本之前，将live版本的记录保存到stable版本中，然后再更新live版本。

        ![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/45EF8568EA494885986E470C05D9F25E/114512)

    如图所示，在prepare阶段产生了新的事务性操作Op(3)，则很显然，它不能直接更新live版本，否则本次Checkpoint可能会将错误的记录写入Checkpoint文件中，因为prepare阶段的Op(3)操作何时完成是不确定的，如果它不被包含在本次Checkpoint中，则可能会出现它所产生的更新被写入Checkpoint文件中的情况，从而导致不一
    致的Checkpoint状态。所以在更新live版本之前，会先执行(1)将live版本拷贝到stable版本，然后再执行(2)将更新写入live版本。

    - 情况2：操作所涉及的记录已经存在stable版本，则可以直接更新live版本。

        ![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/5BD7B357B3AD4D9D8EF4EB3DE75A79D7/114537)

    紧接着情况1，操作序列中又来了一个新的操作Op(3)，但是key为3的记录中已经存在stable版本，当前操作Op(3)可以直接更新live版本。prepare阶段的第二个Op(3)操作何时完成是不确定的，如果它不被包含在本
    次Checkpoint中，则key为3的记录stable版本会被写入Checkpoint文件中，而如果它被包含在本次Checkpoint中，则key为3的记录live版本会被写入Checkpoint文件中。

4. 在prepare阶段开始，并且在prepare阶段完成的事务性更新操作是可以被包含在本次Checkpoint中的，同时它对应的记录中如果有stable版本，则该stable版本是可以删除的，因为既然它被包含在本次Checkpoint中，则stable版本对于本次Checkpoint来说已经过时了。

5. 当prepare阶段结束之后，就可以确定本次Checkpoint的consistency point了，所有在consistency point之前已经committed的事务的数据更新都会反映到Checkpoint文件中，而那些在consistency point之后的数据更新则不会。从此，CALC算法进入resolve阶段。
    ![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/AA32533BD8E94059B8C6AC5A8E8EFB5E/114628)

6. 在resolve阶段，可能会有一些在prepare阶段开始的事务陆续完成，对于这些事务来说，对应的记录如果存在stable版本，则stable版本不能被删除，因为这些stable版本是本次Checkpoint所需要的。还可能会有新的事务性更新操作在resolve阶段产生，对于这些事务，如果对应的记录没有stable版本，则在更新live版本之前，必须将live版本拷贝到stable版本，然后再更新live版本。
    ![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/C47C9EF8FAB54BCBA01CA26B5D383A50/114655)

    如图所示，在prepare阶段产生的最后一个事务性操作Op(3)在resolve阶段完成了。同时，在resolve阶段产生了新的事务性操作Op(2)，它在更新key为2的记录之前，因为没有stable版本，所以会先执行(1)将live版本拷贝到stable版本，然后再执行(2)将更新写入live版本。

7. 至此，可以开始执行写Checkpoint文件的过程了，CALC算法将之称为capture阶段。

8. 当所有属于本次Checkpoint的记录都写入Checkpoint文件之后，本次Checkpoint就结束了。

# CALC算法的设计
CALC算法分为5个阶段，分别是：
- rest阶段，表示没有Checkpoint被触发或者被执行
- prepare阶段，Checkpoint被触发之后，Consistency point被确定之前的一个阶段
- resolve阶段，Consistency point被确定之后，开始Checkpoint IO之前的一个阶段
- capture阶段，开始Checkpoint IO，将所有属于本次Checkpoint的记录都写入Checkpoint文件
- complete阶段，capture阶段完成之后，转换为rest阶段之前

CALC算法除了为每个记录维护了live和stable两个版本以外，还维护了一个名为stable_status的字节向量，每个bit位代表一个记录的stable版本是否为空。每个bit位的取值有2种，分别是available和not_available，available的初始值为1，not_available的初始值为0，stable_status中所有bit为都设置为not_available。

为了方便下面的讲解，会假设内存数据库采用基于锁的事务并发控制机制，同时假设内存数据库使用了commit-log，每个事务在提交的时候，会先向该commit-log中追加一个commit标记，然后再释放事务所申请的锁。

## rest阶段
在rest阶段，所有的记录都只更新live版本，stable版本则为空，stable_status中相应bit位被设置为not_available。在rest阶段开始执行的事务，它的读或者写操作都只使用live版本。

## prepare阶段
当Checkpoint被触发之后，进入prepare阶段。在prepare阶段开始的事务，它的读或者写操作都只使用live版本。但是在更新某个记录之前，如果该记录在stable_status中的bit位是not_available，则在更新live版本之前会先将live版本拷贝到stable版本中，因为不确定在prepare阶段开始的事务会在哪个阶段完成。

另外，对于在prepare阶段开始的事务，如果它被提交，那么在它释放事务所申请的锁之前，会检测系统当前是否仍然处于prepare阶段，如果是，则该事务所更新的记录中的stable版本将被移除。

prepare阶段会一直持续到系统中所有活跃的事务都是在prepare阶段开始的事务为止，也就是持续到在rest阶段开始的事务都完成为止。

## resolve阶段
当prepare阶段结束之后，进入resolve阶段。此时会在commit-log中追加一个特殊的标记，确定一个consistency point，所有在该consistency point之前提交成功的事务相关记录更新都会反映在Checkpoint文件中，在该consistency point之后提交的事务则不被反映在Checkpoint文件中。

对于在prepare阶段开始的事务，如果在resolve阶段提交成功，则它所涉及的记录的stable版本不能被删除，同时stable_status中相应的bit位被设置为available。

对于在resolve阶段开始的事务，它们一定是在consistency point之后完成，因此在更新记录的live版本之前，必须先将live版本拷贝到stable版本中，除非该记录已经有一个显式的stable版本。

resolve阶段会一直持续到所有在prepare阶段开始的事务都完成并且释放了事务所申请的锁。

## capture阶段
在resolve阶段结束之后，进入capture阶段。对于在capture阶段开始的事务，或者在capture阶段之前已经开始的事务，在更新记录的live版本之前，必须先将live版本拷贝到stable版本中，除非该记录已经有一个显式的stable版本。

当capture阶段开始之后，会启动一个后台线程，扫描所有的记录，并对每个记录执行：
- 将该记录的stable版本(如果该记录没有显式的stable版本，则live版本就作为它的stable版本)写入Checkpoint文件中
- 删除该记录中显式的stable版本
- 如果该记录在stable_status中的相应的bit位为not_available，则将该记录在stable_status中的相应的bit位设置为available，以防止在capture阶段运行的事务再次设置stable版本

capture阶段直到处理完所有的记录之后结束。

## complete阶段
一旦capture阶段结束之后，就进入complete阶段，complete阶段会一直持续到所有在capture阶段开始的事务都完成之后结束，然后进入rest阶段。

在capture阶段结束之后，stable_status中所有的bit位都是available，而在rest阶段开始之后，我们希望stable_status中所有的bit位都是not_available。因此，必须重置stable_status。但是如果是通过逐一遍历stable_status中每个bit位的方式来重置，则比较耗时，所以会将not_available和available的值进行反转，如之前not_available是0，available是1，则反转之后not_available为1，available变为0。当下一次再次对not_available和available的值进行反转时，反转之后not_available为0，available变为1。

# CALC算法的伪代码
下面列出CALC算法的伪代码及其注释，方便更好的理解CALC算法：

```
# 初始化
Initialized Database status:
    bit not available = 0;
    bit available = 1;
    bit stable status[DB SIZE];
    foreach key in Database
        db[key].live contains actual record value;
        # 所有记录的stable版本都初始化为空，表示live版本作为stable版本
        db[key].stable is empty;
        # stable_status中所有bit位都初始化为not_available，表示相应记录的stable版本为空
        stable_status[key] = not_available;

# 更新记录
function ApplyWrite(txn, key, value)
    if (txn.start-phase is PREPARE)
        # 对于在prepare阶段开始的事务：
        #   如果stable_status中相应的bit位为not_available，则先将live版本拷贝到stable
        #   版本，然后再更新live版本，此时不会将stable_status中相应的bit位设置为
        #   available，而是留给事务完成之后视情况而定：
        #       如果事务是在prepare阶段完成，则写入Checkpoint文件的应当是live版本，
        #       无需将stable_status中相应的bit位设置为available
        #       如果事务是在prepare阶段完成，需要将stable_status中相应的bit位设置为available
        #   
        #   如果stable_status中相应的bit位为available，则一定是如下的情况：
        #       本事务(记为current)在prepare阶段开始之前，存在另一个在prepare阶段开始的
        #       关于相同key的操作的事务(记为preceeding)，并且precedding一定是在resolve阶段
        #       完成，current事务的更新不能去更新stable版本，而只需要更新live版本
        if (stable_status[key] == not_available)
            db[key].stable = db[key].live;
    else if (txn.start-phase is RESOLVE OR CAPTURE)
        # 对于在resolve阶段或者capture阶段开始的事务：
        #   如果相关记录已经存在stable版本，则保持该stable版本不变即可
        #   如果相关记录不存在stable版本，则先将live版本拷贝到stable版本，并设置
        #   stable_status中相应的bit位为available，然后更新live版本
        if (stable_status[key] == not_available)
            db[key].stable = db[key].live;
            stable_status[key] = available;
    else if (txn.start-phase is COMPLETE OR REST)
        # 对于在complete阶段或者rest阶段开始的事务：
        #   因为已经完成了写Checkpoint文件的过程，live版本直接作为stable版本，如果
        #   存在显式的stable版本，则删除该stable版本
        if (db[key].stable is not empty)
            Erase db[key].stable;
    # 更新live版本
    db[key].live = value
  
# 执行事务    
function Execute(txn)
    # 设置事务在current-phase所在的阶段开始
    txn.start-phase = current-phase;
    # 申请锁
    request txn’s locks;
    # 运行事务逻辑，更新记录
    run txn logic, using ApplyWrite for updates;
    # 在commit-log中添加一个事务commit标记
    append txn commit token to commit-log;
    # 如果事务在prepare阶段开始，且在prepare阶段提交成功，则删除事务所涉及的记录中
    # 的stable版本，因为写入Checkpoint文件中的应当是live版本
    # 
    # 如果事务在prepare阶段开始，且在resolve阶段提交成功，则设置stable_status中相应
    # bit位位available，提前为capture阶段做准备，因为在capture阶段之后所有记录在
    # stable_status中的bit位都应该被设置为available
    if (txn.start-phase is PREPARE)
        if (txn committed during PREPARE phase)
            foreach key in txn
                Erase db[key].stable;
        else if (txn committed during RESOLVE phase)
            foreach key in txn
                stable_status[key] = available;
    # 释放锁
    release txn’s locks;

# CALC算法执行总流程    
function RunCheckpointer()
    while (true)
        # 当前处于rest阶段
        SetPhase(REST);
        # 等待Checkpoint被触发
        wait for signal to start checkpointing;
        # 进入prepare阶段
        SetPhase(PREPARE);
        # prepare阶段持续到所有在prepare阶段之前开始的事务都完成
        wait for all active txns to have start-phase == PREPARE;
        # 进入resolve阶段
        SetPhase(RESOLVE);
        # resolve阶段持续到所有在prepare阶段开始的事务都完成
        wait for all active txns to have start-phase == RESOLVE;
        # 进入capture阶段
        SetPhase(CAPTURE);
        # 逐一处理每个记录
        foreach key in db
            # 如果存在stable版本，则将该stable版本写入Checkpoint文件中，然后删除stable版本
            # 如果不存在stable版本，则将live版本写入Checkpoint文件中，否则将stable版本写入
            # Checkpoint文件，同时删除stable版本
            # Capture阶段执行完成后，所有记录在stable_status中相应的bit位都为available，
            # 这是为了防止在ApplyWrite过程中再去更新stable版本
            # Capture阶段执行完成后，所有记录的stable版本都为空
            if (stable_status[key] == available)
                write db[key].stable to Checkpoint;
                Erase db[key].stable;
            else if (stable_status[key] == not_available)
                stable_status[key] = available;
                val = db[key].live;
                if (db[key].stable is not empty)
                    write db[key].stable to Checkpointing;
                    Erase db[key].stable;
                else if (db[key].stable is empty)
                    write val to Checkpointing;
        # 进入complete阶段
        SetPhase(COMPLETE);
        # complete阶段持续到所有在resolve阶段或者在capture阶段开始的事务都完成
        wait for all active txns to have start-phase == COMPLETE;
        # 将not_available和available的值进行反转
        SwapAvailableAndNotAvailable ();
```

# 增量Checkpoint && pCALC
在CALC算法中，每次Checkpoint都是在整个数据库上执行，这样会比较消耗资源，尤其是在自从上次Checkpoint以来只发生了较少的记录变更的情况下，因此又产生了pCALC(Partial CALC)算法。pCALC只对那些自上次Checkpoint以来发生变更的记录上执行Checkpoint，也就是说只执行增量Checkpoint。多个增量Checkpoint合并之后就可以产生完整的Checkpoint。

为了执行增量Checkpoint，pCALC会追踪记录那些自从上次Consistency point以来发生变更的所有记录。可以采用hash table，或者bit vector，或者bloom filter + bit vector来跟踪所有自上次Consistency point以来发生变更的所有记录。

对多个增量Checkpoint的合并可以交给单独的线程执行，在合并过程中，如果多个增量Checkpoint中包含相同的记录，则总是使用最新的增量Checkpoint中的记录。

# 恢复
## 基于CALC算法的恢复
对于那些可以容忍丢失部分已提交的事务的变更的情形，直接从最新的全量Checkpoint中恢复数据库即可。

对于那些不可以容忍丢失任何已提交的事务的变更的情形，则需要借助于Checkpoint文件和commit-log来进行恢复。首先从Checkpoint中恢复数据库，然后对commit-log进行重放。

## 基于pCALC算法的恢复
方式1，将所有的增量Checkpoint进行合并，产生一个新的Checkpoint，然后基于该合并之后的Checkpoint进行恢复，这种方式相对较慢，因为系统中可能存在非常多的增量Checkpoint。

方式2，在系统运行过程中，在增量Checkpoint之外，增加一些周期性的全量Checkpoint，在恢复的时候，只用合并最近一个全量Checkpoint和自该全量Checkpoint以来的所有增量Checkpoint，然后基于合并之后的Checkpoint进行恢复。

方式3，在系统运行过程中，对增量Checkpoint进行合并，生成全量Checkpoint，过一段时间之后，再次对上一次生成的全量Checkpoint和自该全量Checkpoint以来的所有增量Checkpoint进行合并，生成新的全量Checkpoint，以此类推。

# 参考
Low-Overhead Asynchronous Checkpointing in Main-Memory Database Systems


