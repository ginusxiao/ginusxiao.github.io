# 提纲
[toc]

# Merge Operator

Rocksdb中Merge Operation用于原子的执行Read-Modify-Write操作。

## 为什么要提供Merge operator
### 从uint64计数器说起
在Rocksdb中会经常遇到更新某个已经存在的记录的情况，用户需要先读取该记录（Get），然后更新该记录（Modify），最后将该记录写入数据库中（Put）。

想象我们要维护一系列uint64的计数器，每一个计数器都有一个不同的名称与之对应。计数器提供如下4个接口：Set，Add，Get和Remove。如下：

```
class Counters {
 public:
  // (re)set the value of a named counter
  virtual void Set(const string& key, uint64_t value);

  // remove the named counter
  virtual void Remove(const string& key);

  // retrieve the current value of the named counter, return false if not found
  virtual bool Get(const string& key, uint64_t *value);

  // increase the named counter by value.
  // if the counter does not exist,  treat it as if the counter was initialized to zero
  virtual void Add(const string& key, uint64_t value);
  };
```

然后我们基于Rocksdb实现该计数器，伪代码如下：

```
class RocksCounters : public Counters {
public:
      static uint64_t kDefaultCount = 0;
      RocksCounters(std::shared_ptr<DB> db);

      // mapped to a RocksDB Put
      virtual void Set(const string& key, uint64_t value) {
        string serialized = Serialize(value);
        db_->Put(put_option_, key,  serialized));
      }

      // mapped to a RocksDB Delete
      virtual void Remove(const string& key) {
        db_->Delete(delete_option_, key);
      }

      // mapped to a RocksDB Get
      virtual bool Get(const string& key, uint64_t *value) {
        string str;
        auto s = db_->Get(get_option_, key,  &str);
        if (s.ok()) {
          *value = Deserialize(str);
          return true;
        } else {
          return false;
        }
      }

      // implemented as get -> modify -> set
      virtual void Add(const string& key, uint64_t value) {
        uint64_t base;
        if (!Get(key, &base)) {
          base = kDefaultValue;
        }
        Set(key, base + value);
      }
    };
```

除了Add操作以外，其它3个操作都可以直接映射到Rocksdb中的某个特定操作。如果我们将该计数器作为一种服务对外提供，在如今多核服务器占主流的情况下，该服务肯定是多线程访问的，如果这些线程没有按照keys空间进行划分，那么很可能存在多个线程并发在某个计数器上执行Add的情况，如果对于一致性有所要求，那么必须在调用Add方法的时候采用外部同步机制（即，不是Add方法内部提供的同步机制），比如锁，来进行同步。那么Add的开销会增加。

如果Rocksdb直接支持Add功能，那么计数器的Add方法将类似于这样：

```
    virtual void Add(const string& key, uint64_t value) {
          string serialized = Serialize(value);
          db->Add(add_option, key, serialized);
    }
```

对于计数器来说，上面的做法很合理，但read-modify-write操作的语义是与用户数据类型相关的，Rocksdb中存放的不都是计数器，这要求Rocksdb提供一个更加通用的库。基于此，Rocksdb提供了一个关于read-modify-write操作的更好的抽象：Merge，它允许用户指定自己的操作语义。

## 如何使用Merge Operator
### 接口概览

Rocksdb中定义了一个新的接口：MergeOperator。它提供了一些方法，用于将原始记录和增量更新（称为merge operands）结合起来。这些方法也可以用于将多个merge operands结合起来形成一个新的merge operands（称为Partial merging 或者Associative merging）。

对于Partial merging，Rocksdb提供了一个单独的接口：AssociativeMergeOperator ，它封装了partial merging的所有细节。对于大多数简单应用（如前文中的uint64的计数器）而言，使用AssociativeMergeOperator 就足够了。

```
// The Merge Operator
//
// Essentially, a MergeOperator specifies the SEMANTICS of a merge, which only
// client knows. It could be numeric addition, list append, string
// concatenation, edit data structure, ... , anything.
// The library, on the other hand, is concerned with the exercise of this
// interface, at the right time (during get, iteration, compaction...)
//
// To use merge, the client needs to provide an object implementing one of
// the following interfaces:
//  a) AssociativeMergeOperator - for most simple semantics (always take
//    two values, and merge them into one value, which is then put back
//    into rocksdb); numeric addition and string concatenation are examples;
//
//  b) MergeOperator - the generic class for all the more abstract / complex
//    operations; one method (FullMergeV2) to merge a Put/Delete value with a
//    merge operand; and another method (PartialMerge) that merges multiple
//    operands together. this is especially useful if your key values have
//    complex structures but you would still like to support client-specific
//    incremental updates.
//
// AssociativeMergeOperator is simpler to implement. MergeOperator is simply
// more powerful.
//
// Refer to rocksdb-merge wiki for more details and example implementations.
//
class MergeOperator {
 public:
  virtual ~MergeOperator() {}

  // Gives the client a way to express the read -> modify -> write semantics
  // key:      (IN)    The key that's associated with this merge operation.
  //                   Client could multiplex the merge operator based on it
  //                   if the key space is partitioned and different subspaces
  //                   refer to different types of data which have different
  //                   merge operation semantics
  // existing: (IN)    null indicates that the key does not exist before this op
  // operand_list:(IN) the sequence of merge operations to apply, front() first.
  // new_value:(OUT)   Client is responsible for filling the merge result here.
  // The string that new_value is pointing to will be empty.
  // logger:   (IN)    Client could use this to log errors during merge.
  //
  // Return true on success.
  // All values passed in will be client-specific values. So if this method
  // returns false, it is because client specified bad data or there was
  // internal corruption. This will be treated as an error by the library.
  //
  // Also make use of the *logger for error messages.
  virtual bool FullMerge(const Slice& key,
                         const Slice* existing_value,
                         const std::deque<std::string>& operand_list,
                         std::string* new_value,
                         Logger* logger) const {
    // deprecated, please use FullMergeV2()
    assert(false);
    return false;
  }

  struct MergeOperationInput {
    explicit MergeOperationInput(const Slice& _key,
                                 const Slice* _existing_value,
                                 const std::vector<Slice>& _operand_list,
                                 Logger* _logger)
        : key(_key),
          existing_value(_existing_value),
          operand_list(_operand_list),
          logger(_logger) {}

    // The key associated with the merge operation.
    const Slice& key;
    // The existing value of the current key, nullptr means that the
    // value dont exist.
    const Slice* existing_value;
    // A list of operands to apply.
    const std::vector<Slice>& operand_list;
    // Logger could be used by client to log any errors that happen during
    // the merge operation.
    Logger* logger;
  };

  struct MergeOperationOutput {
    explicit MergeOperationOutput(std::string& _new_value,
                                  Slice& _existing_operand)
        : new_value(_new_value), existing_operand(_existing_operand) {}

    // Client is responsible for filling the merge result here.
    std::string& new_value;
    // If the merge result is one of the existing operands (or existing_value),
    // client can set this field to the operand (or existing_value) instead of
    // using new_value.
    Slice& existing_operand;
  };

  virtual bool FullMergeV2(const MergeOperationInput& merge_in,
                           MergeOperationOutput* merge_out) const;

  // This function performs merge(left_op, right_op)
  // when both the operands are themselves merge operation types
  // that you would have passed to a DB::Merge() call in the same order
  // (i.e.: DB::Merge(key,left_op), followed by DB::Merge(key,right_op)).
  //
  // PartialMerge should combine them into a single merge operation that is
  // saved into *new_value, and then it should return true.
  // *new_value should be constructed such that a call to
  // DB::Merge(key, *new_value) would yield the same result as a call
  // to DB::Merge(key, left_op) followed by DB::Merge(key, right_op).
  //
  // The string that new_value is pointing to will be empty.
  //
  // The default implementation of PartialMergeMulti will use this function
  // as a helper, for backward compatibility.  Any successor class of
  // MergeOperator should either implement PartialMerge or PartialMergeMulti,
  // although implementing PartialMergeMulti is suggested as it is in general
  // more effective to merge multiple operands at a time instead of two
  // operands at a time.
  //
  // If it is impossible or infeasible to combine the two operations,
  // leave new_value unchanged and return false. The library will
  // internally keep track of the operations, and apply them in the
  // correct order once a base-value (a Put/Delete/End-of-Database) is seen.
  //
  // TODO: Presently there is no way to differentiate between error/corruption
  // and simply "return false". For now, the client should simply return
  // false in any case it cannot perform partial-merge, regardless of reason.
  // If there is corruption in the data, handle it in the FullMergeV2() function
  // and return false there.  The default implementation of PartialMerge will
  // always return false.
  virtual bool PartialMerge(const Slice& key, const Slice& left_operand,
                            const Slice& right_operand, std::string* new_value,
                            Logger* logger) const {
    return false;
  }

  // This function performs merge when all the operands are themselves merge
  // operation types that you would have passed to a DB::Merge() call in the
  // same order (front() first)
  // (i.e. DB::Merge(key, operand_list[0]), followed by
  //  DB::Merge(key, operand_list[1]), ...)
  //
  // PartialMergeMulti should combine them into a single merge operation that is
  // saved into *new_value, and then it should return true.  *new_value should
  // be constructed such that a call to DB::Merge(key, *new_value) would yield
  // the same result as subquential individual calls to DB::Merge(key, operand)
  // for each operand in operand_list from front() to back().
  //
  // The string that new_value is pointing to will be empty.
  //
  // The PartialMergeMulti function will be called when there are at least two
  // operands.
  //
  // In the default implementation, PartialMergeMulti will invoke PartialMerge
  // multiple times, where each time it only merges two operands.  Developers
  // should either implement PartialMergeMulti, or implement PartialMerge which
  // is served as the helper function of the default PartialMergeMulti.
  virtual bool PartialMergeMulti(const Slice& key,
                                 const std::deque<Slice>& operand_list,
                                 std::string* new_value, Logger* logger) const;

  // The name of the MergeOperator. Used to check for MergeOperator
  // mismatches (i.e., a DB created with one MergeOperator is
  // accessed using a different MergeOperator)
  // TODO: the name is currently not stored persistently and thus
  //       no checking is enforced. Client is responsible for providing
  //       consistent MergeOperator between DB opens.
  virtual const char* Name() const = 0;
};

// The simpler, associative merge operator.
class AssociativeMergeOperator : public MergeOperator {
 public:
  virtual ~AssociativeMergeOperator() {}

  // Gives the client a way to express the read -> modify -> write semantics
  // key:           (IN) The key that's associated with this merge operation.
  // existing_value:(IN) null indicates the key does not exist before this op
  // value:         (IN) the value to update/merge the existing_value with
  // new_value:    (OUT) Client is responsible for filling the merge result
  // here. The string that new_value is pointing to will be empty.
  // logger:        (IN) Client could use this to log errors during merge.
  //
  // Return true on success.
  // All values passed in will be client-specific values. So if this method
  // returns false, it is because client specified bad data or there was
  // internal corruption. The client should assume that this will be treated
  // as an error by the library.
  virtual bool Merge(const Slice& key,
                     const Slice* existing_value,
                     const Slice& value,
                     std::string* new_value,
                     Logger* logger) const = 0;


 private:
  // Default implementations of the MergeOperator functions
  virtual bool FullMergeV2(const MergeOperationInput& merge_in,
                           MergeOperationOutput* merge_out) const override;

  virtual bool PartialMerge(const Slice& key,
                            const Slice& left_operand,
                            const Slice& right_operand,
                            std::string* new_value,
                            Logger* logger) const override;
};
```

关于MergeOperator的一些说明：

- MergeOperator::FullMerge()中existing_value可能是nullptr，如果该Merge Operation是关于该key的第一个操作的话，existing_value就会是nullptr，表示该key当前不存在。在existing_value为nullptr的情况下，merge_operation的语义由用户确定（比如在前文的计数器中，Counters::Add可能认为existing_value是0）。
 
- MergeOperator::FullMerge()主要用于existing_value是nullptr，或者是关于Put/Delete操作的记录的情况。
 
- MergeOperator::FullMerge()已经废弃了，使用MergeOperator::FullMergeV2()代替，其实MergeOperator::FullMergeV2()是MergeOperator::FullMerge()接口的另外一种呈现形式而已。
 
- MergeOperator::PartialMerge()主要用于合并left_operand和right_operand都是来自于DB::Merge()的情况。
 
- 在MergeOperator::FullMerge()、MergeOperator::FullMergeV2()和MergeOperator::PartialMerge()中都提供了key这一参数，主要是考虑到用户可能对key空间进行划分，并且在不同的key范围区间具有不同的merge operation语义。比如用户可能会在同一个数据库中存放用户余额（数字，number）和交易历史信息（列表，list），对于余额采用前缀“BAL:”作为key，对于交易历史信息采用前缀“HIS:uid”作为key，那么对于账户余额，以算术加法作为merge operator，对于交易历史信息，以list append作为merge operator。通过传递key作为Merge的回调，允许用户区别对待这两种key类型。比如：

```
     void Merge(...) {
       if (key start with "BAL:") {
         NumericAddition(...)
       } else if (key start with "HIS:") {
         ListAppend(...);
       }
     }
```

关于AssociativeMergeOperator的一些说明:

- AssociativeMergeOperator是MergeOperator的子类。

- AssociativeMergeOperator提供了MergeOperator::FullMergeV2()和MergeOperator::PartialMerge()的默认实现，两者都是通过调用用户提供的继承自AssociativeMergeOperator::Merge()的方法实现。

### 通过Merge Operator来实现uint64计数器

想要使用Merge，用户必须定义一个继承自AssociativeMergeOperator接口（或者是MergeOperator接口）的类，该类中必须实现接口中声明的方法，这些方法将在merge过程中被Rocksdb调用。

定义了该类之后，用户必须通过DB类和Options类中的某些域或者方法来指定Rocksdb使用他们自定义的merge operator。Rocksdb在DB类中提供了新的Merg()方法：

```
    // In addition to Get(), Put(), and Delete(), the DB class now also has an additional method: Merge().
    class DB {
      ...
      // Merge the database entry for "key" with "value". Returns OK on success,
      // and a non-OK status on error. The semantics of this operation is
      // determined by the user provided merge_operator when opening DB.
      // Returns Status::NotSupported if DB does not have a merge_operator.
      virtual Status Merge(
        const WriteOptions& options,
        const Slice& key,
        const Slice& value) = 0;
      ...
    };
```

Rocksdb同样在Options类中提供了新的域：

```
Struct Options {
      ...
      // REQUIRES: The client must provide a merge operator if Merge operation
      // needs to be accessed. Calling Merge on a DB without a merge operator
      // would result in Status::NotSupported. The client must ensure that the
      // merge operator supplied here has the same name and *exactly* the same
      // semantics as the merge operator provided to previous open calls on
      // the same DB. The only exception is reserved for upgrade, where a DB
      // previously without a merge operator is introduced to Merge operation
      // for the first time. It's necessary to specify a merge operator when
      // opening the DB in this case.
      // Default: nullptr
      const std::shared_ptr<MergeOperator> merge_operator;
      ...
    };
```

基于上述接口改变，用户可以通过这些内置的Merge operation来实现第二版本的计数器：

```
    // A 'model' merge operator with uint64 addition semantics
    class UInt64AddOperator : public AssociativeMergeOperator {
     public:
      virtual bool Merge(
        const Slice& key,
        const Slice* existing_value,
        const Slice& value,
        std::string* new_value,
        Logger* logger) const override {

        // assuming 0 if no existing value
        uint64_t existing = 0;
        if (existing_value) {
          if (!Deserialize(*existing_value, &existing)) {
            // if existing_value is corrupted, treat it as 0
            Log(logger, "existing value corruption");
            existing = 0;
          }
        }

        uint64_t oper;
        if (!Deserialize(value, &oper)) {
          // if operand is corrupted, treat it as 0
          Log(logger, "operand value corruption");
          oper = 0;
        }

        auto new = existing + oper;
        *new_value = Serialize(new);
        return true;        // always return true for this, since we treat all errors as "zero".
      }

      virtual const char* Name() const override {
        return "UInt64AddOperator";
       }
    };

    // Implement 'add' directly with the new Merge operation
    class MergeBasedCounters : public RocksCounters {
     public:
      MergeBasedCounters(std::shared_ptr<DB> db);

      // mapped to a leveldb Merge operation
      virtual void Add(const string& key, uint64_t value) override {
        string serialized = Serialize(value);
        db_->Merge(merge_option_, key, serialized);
      }
    };

    // How to use it
    DB* dbp;
    Options options;
    options.merge_operator.reset(new UInt64AddOperator);
    DB::Open(options, "/tmp/db", &dbp);
    std::shared_ptr<DB> db(dbp);
    MergeBasedCounters counters(db);
    counters.Add("a", 1);
    ...
    uint64_t v;
    counters.Get("a", &v);
```

### 更加通用的MergeOperator

对于类似前述计数器的用例，很多情况下都可以使用AssociativeMergeOperator来实现merge operation。但是假如在Rocksdb中存储了一系列json字符串，用户需要去更新该json字符串中的某个域：

```
    ...
    // Put/store the json string into to the database
    db_->Put(put_option_, "json_obj_key",
             "{ employees: [ {first_name: john, last_name: doe}, {first_name: adam, last_name: smith}] }");

    ...

    // Use a pre-defined "merge operator" to incrementally update the value of the json string
    db_->Merge(merge_option_, "json_obj_key", "employees[1].first_name = lucy");
    db_->Merge(merge_option_, "json_obj_key", "employees[0].last_name = dow");
```

这种情况下AssociativeMergeOperator 将无能为力，为什么呢？因为AssociativeMergeOperator对于有以下结合性（associativity）约束：

- 调用Put接口存储到Rocksdb中的记录和Merge接口中的merge operand具有相同的格式；

- 并且采用用户指定的同一个merge operator可以将不同的merge operands合并成单一merge operand；

很明显这种情况下是不满足AssociativeMergeOperator关于结合性的约束，必须区分以json字符串形式存在的基础值（base value）和以json赋值语句形式存在的merge operands，这种情况只能采用更加通用的MergeOperator了，如下：

```
    // A 'model' pseudo-code merge operator with json update semantics
    // We pretend we have some in-memory data-structure (called JsonDataStructure) for
    // parsing and serializing json strings.
    class JsonMergeOperator : public MergeOperator {          // not associative
     public:
      virtual bool FullMerge(const Slice& key,
                             const Slice* existing_value,
                             const std::deque<std::string>& operand_list,
                             std::string* new_value,
                             Logger* logger) const override {
        JsonDataStructure obj;
        if (existing_value) {
          obj.ParseFrom(existing_value->ToString());
        }

        if (obj.IsInvalid()) {
          Log(logger, "Invalid json string after parsing: %s", existing_value->ToString().c_str());
          return false;
        }

        for (const auto& value : operand_list) {
          auto split_vector = Split(value, " = ");      // "xyz[0] = 5" might return ["xyz[0]", 5] as an std::vector, etc.
          obj.SelectFromHierarchy(split_vector[0]) = split_vector[1];
          if (obj.IsInvalid()) {
            Log(logger, "Invalid json after parsing operand: %s", value.c_str());
            return false;
          }
        }

        obj.SerializeTo(new_value);
        return true;
      }


      // Partial-merge two operands if and only if the two operands
      // both update the same value. If so, take the "later" operand.
      virtual bool PartialMerge(const Slice& key,
                                const Slice& left_operand,
                                const Slice& right_operand,
                                std::string* new_value,
                                Logger* logger) const override {
        auto split_vector1 = Split(left_operand, " = ");   // "xyz[0] = 5" might return ["xyz[0]", 5] as an std::vector, etc.
        auto split_vector2 = Split(right_operand, " = ");

        // If the two operations update the same value, just take the later one.
        if (split_vector1[0] == split_vector2[0]) {
          new_value->assign(right_operand.data(), right_operand.size());
          return true;
        } else {
          return false;
        }
      }

      virtual const char* Name() const override {
        return "JsonMergeOperator";
       }
    };

    ...

    // How to use it
    DB* dbp;
    Options options;
    options.merge_operator.reset(new JsonMergeOperator);
    DB::Open(options, "/tmp/db", &dbp);
    std::shared_ptr<DB> db_(dbp);
    ...
    // Put/store the json string into to the database
    db_->Put(put_option_, "json_obj_key",
             "{ employees: [ {first_name: john, last_name: doe}, {first_name: adam, last_name: smith}] }");

    ...

    // Use the "merge operator" to incrementally update the value of the json string
    db_->Merge(merge_option_, "json_obj_key", "employees[1].first_name = lucy");
    db_->Merge(merge_option_, "json_obj_key", "employees[0].last_name = dow");
```

### 使用小窍门

- 在打开或者创建Rocksdb实例的时候，只能传递一个merge operator，如果需要在不同的记录上执行不同的merge operator该怎么办呢？只能在该merge operator上做文章的，每个记录的key或者value都会被传递给merge operator，所以可以通过在不同的key或者不同的value上实现不同的操作，来实现同一个merge operator执行不同功能（类似于前文中描述的银行账户的那样）。

- 由于采用AssociativeMergeOperator需要满足结合性约束（associative constraint），有时候可能不太确定是否满足该约束，于是就不确定是否应该使用AssociativeMergeOperator，在这些情况下，最好的做法就是使用MergeOperator，因为AssociativeMergeOperator是MergeOperator的子类，AssociativeMergeOperator能够实现的功能，MergeOperator也一定能够实现。但是如果确定满足结合性约束的话，使用AssociativeMergeOperator会更加方便。

# Merge Operator实现细节
## Rocksdb数据模型

简单来说，Rocksdb实际上就是一个带版本号的KV存储。对于Rocksdb上的每一个变更都是全局有序的，并且关联到一个单调递增的序列号。对于每一个key来说，Rocksdb保留了关于该key的操作历史。对于任何一个经历了n次变更（OPi）的key（K）来说，逻辑上类似于这样：

    K:   OP1   OP2   OP3   ...   OPn

针对某个key的每一个变更操作都有3个熟悉：类型 - Delete/Put/Merge，序列号和value（Delete可以视为特殊的value）。对于某个key来说序列号是递增的，但不一定是连续的。

当用户执行db->Put或者db->Delete的时候，Rocksdb将该操作添加到相关key的操作历史中，而不会对该key做任何检查（即使Delete一个不存在的key也不会去检查该key是否存在）。

当用户执行db->Get的时候，Rocksdb返回相关key在某个时间点（通过序列号来指定）的状态。对于任何一个key来说，它的初始状态为non-existent，每一个关于该key的变更操作都会将该key转移到另一个新的状态，从这个角度来说，每一个key都是一个状态机，每一个变更操作作为该状态机的状态转移函数。

从状态机的角度来说，Merge也是一个状态转移函数，它将当前的状态和merge operand合并生成一个新的状态。Put则是Merge的变种，它不关注当前状态。Delete则也是Merge的变种，它直接将key的状态转移到初始状态，即non-existent。

### Get

Get操作返回特定时间点的状态：

    K:   OP1    OP2   OP3   ....   OPk  .... OPn
                                        ^
                                        |
                                    Get.seq

假设OPk是Get可见的最近的更新操作，那么k必须满足：

    k = max(i) {seq(OPi) <= Get.seq}

如果OPk是Put操作，则Get直接返回对应的操作中的值，如果OPk是Delete操作，则Get直接返回NotFound。

如果OPk是Merge操作，则需要前向搜索，直到遇到Put或者Delete操作：

    K:   OP1    OP2   OP3   ....    OPk  .... OPn
                Put  Merge  Merge  Merge
                                        ^
                                        |
                                    Get.seq
                     -------------------->
                     
对于这种情况，Get返回类似于：

    Merge(...Merge(Merge(operand(OP2), operand(OP3)), operand(OP4)..., operand(OPk))))

对于OPk是Merge操作的情况下，Get()算法如下：

```
Get(key):
  // in reality, this should be a "deque", but stack is simpler to conceptualize for this pseudocode
  Let stack = [ ];
  for each entry OPi from newest to oldest:
    if OPi.type is "merge_operand":
      push OPi to stack
        while (stack has at least 2 elements and (stack.top() and stack.second_from_top() can be partial-merged)
          OP_left = stack.pop()
          OP_right = stack.pop()
          result_OP = client_merge_operator.PartialMerge(OP_left, OP_right)
          push result_OP to stack
    else if OPi.type is "put":
      return client_merge_operator.FullMerge(v, stack);
    else if v.type is "delete":
      return client_merge_operator.FullMerge(nullptr, stack);
      
  // We've reached the end (OP0) and we have no Put/Delete, just interpret it as empty (like Delete would)
  return client_merge_operator.FullMerge(nullptr, stack);
```

### Compaction

前文中讲到，Rocksdb中会记录所有key的操作历史，但是关于某个key的操作是无限的，必须采取某种机制来减少这些历史，Compaction就是用于在不影响外部可见状态（如snapshot）的情况下减少关于key的操作历史的过程。快照简单的采用序列号来表示：

    K:   OP1     OP2     OP3     OP4     OP5  ... OPn
                  ^               ^                ^
                  |               |                |
               snapshot1       snapshot2       snapshot3
               
对于每一个snapshot，定义它可见的最近的操作为Supporting operation（如OP2是snapshot1的Supporting operation，OP4是snapshot2的Supporting Operation...）。

为了保持snapshot的可见状态，所有的Supporting operations都必须被保留。但是在引入Merge operation之前，可以丢弃那些non-supporting operations（因为Put和Delete操作不依赖于它之前的操作），以上述示例为例，一次full compaction可以将K的操作历史减少为：

    K:   OP2     OP4     OPn
    
但是在引入Merge操作之后，情况就大不一样了。即使某个merge operation不是Supporting operation，但是它后面的merge operations可能依赖与它，同样Put和Delete操作也可能被某个merge operation所依赖。那么该怎么办呢？Rocksdb沿着关于某个key的最新的操作向最旧的操作的方向前进，对于前进过程中遇到的Merge operation，执行入栈（stacking）或者PartialMerge，直到遇到下列情形：

- 遇到Put/Delete操作 - 调用FullMerge(value or nullptr, stack)
 
- 到达key的操作历史的末尾 - 调用FullMerge(nullptr, stack)
 
- 遇到snapshot相关的Supporting operation - 见下面
 
- 如果到达当前compaction的区间范围的结束位置 - 见下面

前两种情形跟Get()中类似，但是Compaction引入了另外两种情况：

- 如果在compaction过程中遇到了snapshot，就必须停止compaction，直接写入当前合并的结果（可能是Merge或者Put），清理栈，继续从遇到的snapshot的Supporting operation开始执行compaction

- 如果到达当前compaction的区间范围的结束位置，则不能简单的执行FullMerge(nullptr, stack)，因为尚未到达key的操作历史的末尾

PartialMerge用于辅助Compaction，它会将多个merge operands合并为一个merge operand。

#### 示例

假设一个计数器K从0开始，经历一系列Add操作（基于DB::Merge()实现），然后被重置为2，接着又经历一系列Add操作：

(Note: In this example we assume associativity, but the idea is the same without PartialMerge as well)

    K:    0    +1    +2    +3    +4     +5      2     +1     +2
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3

Rocksdb沿着该key的最新的操作向最旧的操作的方向前进：

    K:    0    +1    +2    +3    +4     +5      2    (+1     +2)
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3

+2和+1这两个merge operations合并，产生一个新的merge operation：

      (+1  +2) => PartialMerge(1,2) => +3

    K:    0    +1    +2    +3    +4     +5      2            +3
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3
    
    K:    0    +1    +2    +3    +4     +5     (2            +3)
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3

Merge operation和它前面的一个Put操作合并产生一个新的Put操作：

      (2   +3) =>  FullMerge(2, 3) => 5

    K:    0    +1    +2    +3    +4     +5                    5
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3

新产生的Put操作隐藏它之前的non-Supporting operations：

      (+5   5) => 5

    K:    0    +1    +2   (+3    +4)                          5
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3

    (+3  +4) => PartialMerge(3,4) => +7

    K:    0    +1    +2          +7                           5
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3

Merge operation不能和它前面的Supporting operation合并：

       (+2   +7) can not be combined

    K:    0   (+1    +2)         +7                           5
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3

    (+1  +2) => PartialMerge(1,2) => +3

    K:    0          +3          +7                           5
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3

    K:   (0          +3)         +7                           5
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3

    (0   +3) => FullMerge(0,3) => 3

    K:               3           +7                           5
                     ^           ^                            ^
                     |           |                            |
                  snapshot1   snapshot2                   snapshot3
                  
概况起来说：在Compaction过程中，如果某个Supporting operation是Merge operation，它将和它之前的operations结合（入栈，或者执行PartialMerge），直到：

- 遇到另一个Supporting operation（即遇到另一个snapshot边界）

- 遇到Put或者Delete操作，则将该Merge operation和Put/Delete结合，形成一个新的Put

- 到达key操作历史的末尾（end-of-key-history），将Merge operation转换为Put

- 到达当前compaction的区间范围的结束位置（end-of-Compaction-Files），视为遇到snapshot边界

#### Compaction算法针对Merge Operator的改进

Compaction算法如下:

```
Compaction(snaps, files):
  // <snaps> is the set of snapshots (i.e.: a list of sequence numbers)
  // <files> is the set of files undergoing compaction
  Let input = a file composed of the union of all files
  Let output = a file to store the resulting entries

  // in reality, this should be a "deque", but stack is simpler to conceptualize in this pseudo-code
  Let stack = [];
  for each v from newest to oldest in input:
    clear_stack = false
    if v.sequence_number is in snaps:   // encounter a Supporting operation
      clear_stack = true
    else if stack not empty && v.key != stack.top.key:      // encounter a different key
      clear_stack = true

    if clear_stack:
      write out all operands on stack to output (in the same order as encountered)
      clear(stack)

    if v.type is "merge_operand":   // encounter a merge operation
      push v to stack
        while (stack has at least 2 elements and (stack.top and stack.second_from_top can be partial-merged)):
          v1 = stack.pop();
          v2 = stack.pop();
          result_v = client_merge_operator.PartialMerge(v1,v2)
          push result_v to stack
    if v.type is "put": // encounter a put operation
      write client_merge_operator.FullMerge(v, stack) to output
      clear stack
    if v.type is "delete":  // encounter a delete operation
      write client_merge_operator.FullMerge(nullptr, stack) to output
      clear stack

  If stack not empty:   // complete current compaction
    if end-of-key-history for key on stack:     // encounter end-of-key-history
      write client_merge_operator.FullMerge(nullptr, stack) to output
      clear(stack)
    else        // encounter end-of-compaction-file
      write out all operands on stack to output
      clear(stack)

  return output
```

#### 关于参与Compaction的文件挑选算法的改进 

在Compaction过程中，如果关于某个key在Lm层存在大量的分布于多个文件中的merge operations，但是只有部分较新的merge operations参与compaction，那么就会存在这些较新的merge operation被合并写入Lm+1，而Lm层较老旧的merge operation依然在Lm层的情况。为了避免该情况，Rocksdb对compaction进行调整，在Lm层挑选参与合并的文件的时候，扩展挑选的文件集合，以确保包含所有较老的merge operations。


# Merge Operator源码分析
