# 提纲
[toc]

## 介绍
```
PostgreSQL actually treats every SQL statement as being executed within a transaction. If you do not issue a BEGIN command, then each individual statement has an implicit BEGIN and (if successful) COMMIT wrapped around it. A group of statements surrounded by BEGIN and COMMIT is sometimes called a transaction block.
```

事实上，PostgreSQL将每一条SQL语句都视为在一个事务中执行。如果没有显式的使用BEGIN命令，则没一条语句都隐式的在其前后分别添加一个BEGIN和COMMIT命令(这就是所谓的autocommit)。如果一条或者多条语句的前后分别使用了BEGIN和COMMIT命令，则这些语句被称为transaction block。

```
It's possible to control the statements in a transaction in a more granular fashion through the use of savepoints. Savepoints allow you to selectively discard parts of the transaction, while committing the rest. After defining a savepoint with SAVEPOINT, you can if needed roll back to the savepoint with ROLLBACK TO. All the transaction's database changes between defining the savepoint and rolling back to it are discarded, but changes earlier than the savepoint are kept.

After rolling back to a savepoint, it continues to be defined, so you can roll back to it several times. Conversely, if you are sure you won't need to roll back to a particular savepoint again, it can be released, so the system can free some resources. Keep in mind that either releasing or rolling back to a savepoint will automatically release all savepoints that were defined after it.
```

通过使用savepoints，可以对事务中的语句更加细粒度的控制。savepoints允许用户选择性的丢弃事务的某一部分，但是提交事务的其它部分。通过SAVEPOINT命令来定义savepoint，通过ROLLBACK TO SAVEPOINT来回退到某个检查点，当回退到某个savepoint时，则这个savepoint和ROLLBACK TO之间的更新将会被丢弃。

如果用户不在需要rollback到某个savepoint了，则这个savepoint可以通过RELEASE SAVEPOINT来销毁一个savepoint。

## 事务模型
[PostgreSQL事务模型](http://www.gxlcms.com/mysql-312925.html)

## 子事务
[PostgreSQL子事务及性能分析](https://www.modb.co/db/23355)

在PostgreSQL中savepoint就是通过子事务实现的，在PL/pgSQ中输入带有Exception子句的语句块时，也会开启一个新的子事务：
```
BEGIN
   PERFORM 'Some work is done';
 
   BEGIN  -- a block inside a block
      PERFORM 12  (factorial(0) - 1);
   EXCEPTION
      WHEN division_by_zero THEN
         NULL;  -- ignore the error
   END;
 
   PERFORM 'try to do more work';
END;
```

## 相关命令
BEGIN - 启动一个transaction block
```
BEGIN [ WORK | TRANSACTION ] [ transaction_mode [, ...] ]

where transaction_mode is one of:

    ISOLATION LEVEL { SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED }
    READ WRITE | READ ONLY
    [ NOT ] DEFERRABLE
```

START TRANSACTION - 和BEGIN命令相同
```
START TRANSACTION [ transaction_mode [, ...] ]

where transaction_mode is one of:

    ISOLATION LEVEL { SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED }
    READ WRITE | READ ONLY
    [ NOT ] DEFERRABLE
```

COMMIT - 提交当前的transaction
ROLLBACK - 终止当前的transaction 
SAVEPOINT - 在当前事务中定义一个新的savepoint
```
SAVEPOINT savepoint_name
```

ROLLBACK TO SAVEPOINT - 回滚到某个savepoint
```
ROLLBACK [ WORK | TRANSACTION ] TO [ SAVEPOINT ] savepoint_name
```

RELEASE SAVEPOINT - 销毁某个savepoint
```
RELEASE [ SAVEPOINT ] savepoint_name
```
