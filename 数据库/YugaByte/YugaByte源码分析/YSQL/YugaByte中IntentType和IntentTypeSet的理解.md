# 提纲
[toc]

## IntentType
```
constexpr int kWeakIntentFlag         = 0b000;
constexpr int kStrongIntentFlag       = 0b010;
constexpr int kReadIntentFlag         = 0b000;
constexpr int kWriteIntentFlag        = 0b001;

YB_DEFINE_ENUM(IntentType,
    // kWeakRead = 0b000 = 0
    ((kWeakRead,      kWeakIntentFlag |  kReadIntentFlag))
    // kWeakWrite = 0b001 = 1
    ((kWeakWrite,     kWeakIntentFlag | kWriteIntentFlag))
    // kStrongRead = 0b010 = 2
    ((kStrongRead,  kStrongIntentFlag |  kReadIntentFlag))
    // kStrongRead = 0b011 = 3
    ((kStrongWrite, kStrongIntentFlag | kWriteIntentFlag))
);
```

## IntentTypeSet
```
# kIntentTypeSetMapSize = 16
# 表示所有IntentType的组合个数，包括：
#   空的IntentTypeSet，共1个
#   4种IntentType中选取1个IntentType形成IntentTypeSet，共4个
#   4种IntentType中选取2个IntentType形成IntentTypeSet，共6个
#   4种IntentType中选取3个IntentType形成IntentTypeSet，共4个
#   4种IntentType中选取4个IntentType形成IntentTypeSet，共1个
constexpr int kIntentTypeSetMapSize = 1 << kIntentTypeMapSize;

# IntentTypeSet标识IntentType的组合，它被定义为一个关于IntentType的EnumBitSet，顾名思义，EnumBitSet就是关于Enum的bitset，在实现上EnumBitSet是对std::bitset的封装
typedef EnumBitSet<IntentType> IntentTypeSet;

# EnumBitSet在实现上是对std::bitset的封装
template <class Enum>
class EnumBitSet {
 public:
  typedef EnumBitSetIterator<Enum> const_iterator;

  EnumBitSet() = default;
  
  # impl_是一个std::bitset
  # 直接采用value来设置impl_这个std::bitset，impl_这个bitset对应的值就等于value
  explicit EnumBitSet(uint64_t value) : impl_(value) {}

  # 传递一个list作为参数
  # 遍历list中的每一个Enum类型的元素，将之转换为整形数，然后将impl_中这个整形数
  # 所在的bit位设置为true
  explicit EnumBitSet(const std::initializer_list<Enum>& inp) {
    for (auto i : inp) {
      impl_.set(to_underlying(i));
    }
  }

  # 获取给定的Enum对应的整形数，并检查impl_中这个整形数所在的bit位是否为true
  bool Test(Enum value) const {
    return impl_.test(to_underlying(value));
  }

  # 获取impl_对应的整型值
  uintptr_t ToUIntPtr() const {
    return impl_.to_ulong();
  }

  # 检查impl_中是否没有任何一个bit为true
  bool None() const {
    return impl_.none();
  }

  # 检查impl_中是否有至少一个bit为true
  bool Any() const {
    return impl_.any();
  }

  # 检查impl_中是否所有bit位都为true
  bool All() const {
    return impl_.all();
  }

  # 获取给定的Enum对应的整形数，并将impl_中这个整形数所在的bit位设置为true
  EnumBitSet& Set(Enum value) {
    impl_.set(to_underlying(value));
    return *this;
  }

  # 获取impl_中第一个为true的bit位
  const_iterator begin() const {
    return const_iterator(List(static_cast<Enum*>(nullptr)).begin(), &impl_);
  }

  # 获取impl_中下一个为true的bit位
  const_iterator end() const {
    return const_iterator(List(static_cast<Enum*>(nullptr)).end(), &impl_);
  }
  
  ...
  
 private:
  std::bitset<MapSize(static_cast<Enum*>(nullptr))> impl_;
}

template <class Enum>
class EnumBitSetIterator {
 public:
  typedef typename decltype(List(static_cast<Enum*>(nullptr)))::const_iterator ImplIterator;
  typedef std::bitset<MapSize(static_cast<Enum*>(nullptr))> BitSet;

  # 构造方法中
  # 初始化之后iter_将指向第一个为true的bit位
  EnumBitSetIterator(ImplIterator iter, const BitSet* set) : iter_(iter), set_(set) {
    # 找到set_中第一个为true的bit位，并用iter_指向它
    FindSetBit();
  }

  # 获取当前iter_锁指向的bit位
  Enum operator*() const {
    return *iter_;
  }

  # 获取下一个为true的bit位
  EnumBitSetIterator& operator++() {
    ++iter_;
    FindSetBit();
    return *this;
  }

  EnumBitSetIterator operator++(int) {
    EnumBitSetIterator result(*this);
    ++(*this);
    return result;
  }

 private:
  # 找到set_中第一个为true的bit位，并用iter_指向它
  void FindSetBit() {
    while (iter_ != List(static_cast<Enum*>(nullptr)).end() && !set_->test(to_underlying(*iter_))) {
      ++iter_;
    }
  }

  friend bool operator!=(const EnumBitSetIterator<Enum>& lhs, const EnumBitSetIterator<Enum>& rhs) {
    return lhs.iter_ != rhs.iter_;
  }

  ImplIterator iter_;
  const BitSet* set_;
}
```

## Strong IntentTypeSet和Weak IntentTypeSet互转
```
inline IntentTypeSet StrongToWeak(IntentTypeSet inp) {
  # 以inp为{kStrongRead, kStrongWrite}为例：
  # inp.ToUIntPtr()所代表的的整形数转换为2进制是：0b1100，kStrongIntentFlag对应的2进制是：0b010，
  # inp.ToUIntPtr() >> kStrongIntentFlag就等价于0b1100 >> 2得到0b0011，然后再转换成IntentTypeSet
  # 就是{kWeakRead, kWeakWrite}
  IntentTypeSet result(inp.ToUIntPtr() >> kStrongIntentFlag);
  DCHECK((inp & result).None());
  return result;
}

inline IntentTypeSet WeakToStrong(IntentTypeSet inp) {
  # 以inp为{kWeakRead, kWeakWrite}为例：
  # inp.ToUIntPtr()所代表的的整形数转换为2进制是：0b0011，kStrongIntentFlag对应的2进制是：0b010，
  # inp.ToUIntPtr() << kStrongIntentFlag就等价于0b0011 << 2得到0b1100，然后再转换成IntentTypeSet
  # 就是{kStrongRead, kStrongWrite}
  IntentTypeSet result(inp.ToUIntPtr() << kStrongIntentFlag);
  DCHECK((inp & result).None());
  return result;
}
```

## kIntentTypeSetAdd
用于在成功对给定IntentTypeSet加锁或者解锁的情况下，该如何修改SharedLockManager中记录的锁状态信息(持有哪种类型的锁，以及持有该锁的数目)。

对于kIntentTypeSetAdd来说，它是被SharedLockManager用来记录给定key上的各种intent lock被加锁的次数时使用的，见LockedBatchEntry::num_holding。比如，某个key上已经被一个使用者加了{kWeakRead, kStrongRead}锁，那么此时LockedBatchEntry::num_holding中会加上kIntentTypeSetAdd[5]，也就是LockedBatchEntry::num_holding中bit[0 - 15]为0000 0000 0000 0001，而LockedBatchEntry::num_holding中bit[32 - 48]为0000 0000 0000 0001，LockedBatchEntry::num_holding反映的是此时有1个使用者对这个key加了kWeakRead锁，有1个使用者对这个key加了kStrongRead锁；此时如果另一个使用者要来加{kWeakRead, kStrongRead}锁，且顺利通过了冲突检测，则LockedBatchEntry::num_holding中会再次加上kIntentTypeSetAdd[5]，那么LockedBatchEntry::num_holding中bit[0 - 15]为0000 0000 0000 0010，而LockedBatchEntry::num_holding中bit[32 - 48]为0000 0000 0000 0010，LockedBatchEntry::num_holding反映的是此时有2个使用者对这个key加了kWeakRead锁，有2个使用者对这个key加了kStrongRead锁。
```
const size_t kIntentTypeBits = 16;

const std::array<LockState, kIntentTypeSetMapSize> kIntentTypeSetAdd = GenerateByMask(1);

std::array<LockState, kIntentTypeSetMapSize> GenerateByMask(LockState single_intent_mask) {
  DCHECK_EQ(single_intent_mask & kSingleIntentMask, single_intent_mask);
  std::array<LockState, kIntentTypeSetMapSize> result;
  # kIntentTypeSetMapSize为16，表示所有IntentType共有16中组合，每个idx代表一种组合
  for (size_t idx = 0; idx != kIntentTypeSetMapSize; ++idx) {
    result[idx] = 0;
    # IntentTypeSet(idx)用于获取idx所对应的IntentTypeSet，idx这个整形数中如果某个bit为是true，
    # 则最终的IntentTypeSet中应当包含这个bit位所对应的IntentType，举个例子：
    #
    # 假设idx为5，对应的2进制是0b0101，它的第0个第2个bit位是true，所以对应的IntentTypeSet为
    # 0所对应的IntentType(kWeakRead)和2所对应的IntentType(kStrongRead)的组合，即
    # {kWeakRead, kStrongRead}
    #
    # 还是上面的例子，假设idx为5，则for循环会分别处理kWeakRead和kStrongRead：
    # 处理kWeakRead：result[5] |= 1 << (0 * 16)，
    # 处理kStrongRead：result[5] |= 1 << (2 * 16)，
    # 最终的result[5] = 0b0001 0000 0000 0000 0000 0000 0000 0000 0001
    #
    # 那么result[idx]计算出的是啥呢？
    #   如果idx对应的IntentTypeSet中包含kWeakRead(0)，则将result[idx]中的第0位设置为1
    #   如果idx对应的IntentTypeSet中包含kWeakWrite(1)，则将result[idx]中的第16位设置为1
    #   如果idx对应的IntentTypeSet中包含kStrongRead(2)，则将result[idx]中的第32位设置为1
    #   如果idx对应的IntentTypeSet中包含kStrongWrite(3)，则将result[idx]中的第48位设置为1
    for (auto intent_type : IntentTypeSet(idx)) {
      result[idx] |= single_intent_mask << (to_underlying(intent_type) * kIntentTypeBits);
    }
  }
  return result;
}
```


## kIntentTypeSetConflicts
用于获取给定idx对应的IntentTypeSet对应的冲突掩码，比如idx为5，则对应的IntentTypeSet为{kWeakRead, kStrongRead}，因为它会与kWeakWrite和kStrongWrite冲突，所以对应的冲突掩码为：0b1111 1111 1111 1111 0000 0000 0000 0000 1111 1111 1111 1111 0000 0000 0000 0000。

```
const size_t kIntentTypeBits = 16;
# kSingleIntentMask = 0b1111 1111 1111 1111
const LockState kSingleIntentMask = (static_cast<LockState>(1) << kIntentTypeBits) - 1;

# 获取给定IntentType的Mask，比如：
#   如果IntentType为kWeakRead，则它对应的IntentTypeMask为0b1111 1111 1111 1111，
#   如果IntentType为kWeakWrite，则它对应的IntentTypeMask为0b1111 1111 1111 1111 0000 0000 0000 0000，
#   如果IntentType为kStrongRead，则它对应的IntentTypeMask为0b1111 1111 1111 1111 0000 0000 0000 0000 0000 0000 0000 0000，
#   如果IntentType为kStrongWrite，则它对应的IntentTypeMask为0b1111 1111 1111 1111 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000，
# 
# 可以简单理解为，IntentType将一个uint64的整数分为4段，每一段对应一种IntentType，
# 第0段对应的就是kWeakRead的Mask，第1段对应的就是kWeakWrite的Mask，
# 第2段对应的就是kStrongRead的Mask，第3段对应的就是kStrongWrite的Mask
LockState IntentTypeMask(IntentType intent_type) {
  return kSingleIntentMask << (to_underlying(intent_type) * kIntentTypeBits);
}

# 用于获取给定idx对应的IntentTypeSet对应的冲突掩码
const std::array<LockState, kIntentTypeSetMapSize> kIntentTypeSetConflicts = GenerateConflicts();

# 判断2种IntentType之间是否冲突：如果2种IntentType中至少有一个是Strong的，
# 且这2种IntentType分别是read的和write的，则就是冲突的
bool IntentTypesConflict(IntentType lhs, IntentType rhs) {
  auto lhs_value = to_underlying(lhs);
  auto rhs_value = to_underlying(rhs);
  // The rules are the following:
  // 1) At least one intent should be strong for conflict.
  // 2) Read and write conflict only with opposite type.
  return ((lhs_value & kStrongIntentFlag) || (rhs_value & kStrongIntentFlag)) &&
         ((lhs_value & kWriteIntentFlag) != (rhs_value & kWriteIntentFlag));
}

# 检查2种IntentTypeSet之间是否冲突
bool IntentTypeSetsConflict(IntentTypeSet lhs, IntentTypeSet rhs) {
  for (auto intent1 : lhs) {
    for (auto intent2 : rhs) {
      if (IntentTypesConflict(intent1, intent2)) {
        return true;
      }
    }
  }
  return false;
}

# 生成冲突掩码
std::array<LockState, kIntentTypeSetMapSize> GenerateConflicts() {
  std::array<LockState, kIntentTypeSetMapSize> result;
  # kIntentTypeSetMapSize = 16
  for (size_t idx = 0; idx != kIntentTypeSetMapSize; ++idx) {
    result[idx] = 0;
    
    # 遍历IntentTypeSet(idx)所对应的IntentTypeSet中的每个IntentType，看它和kIntentTypeList中的每种
    # IntentType之间是否存在冲突，如果存在冲突，则将相应的IntentType的Mask添加到冲突列表中
    #
    # 比如idx为5，则IntentTypeSet(idx)对应的就是{kWeakRead, kStrongRead}，这里会执行如下：
    #   1. 检查kWeakRead和kWeakRead, kWeakWrite, kStrongRead, kStrongWrite之间是否存在冲突，
    #   如果存在冲突，则将与之冲突的IntentType的Mask添加到result[idx]中，检查发现只有kStrongWrite
    #   和kWeakRead存在冲突，所以result[idx]会或上kStrongWrite的Mask(0b1111 1111 1111 1111
    #   0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000)
    #
    #   2. 检查kStrongRead和kWeakRead, kWeakWrite, kStrongRead, kStrongWrite之间是否存在冲突，
    #   如果存在冲突，则将与之冲突的IntentType的Mask添加到result[idx]中，检查发现有kWeakWrite
    #   和kStrongWrite都与kStrongRead存在冲突，所以result[idx]会或上kWeakWrite的Mask(0b1111 1111
    #   1111 1111 0000 0000 0000 0000)和kStrongWrite的Mask(0b1111 1111 1111 1111 0000 0000 0000
    #   0000 0000 0000 0000 0000 0000 0000 0000 0000)
    for (auto intent_type : IntentTypeSet(idx)) {
      # 遍历kIntentTypeList中的每个IntentType(共4种，在IntentType中定义的)
      for (auto other_intent_type : kIntentTypeList) {
        # 检查是否存在冲突，如果存在冲突，则将与之存在冲突的IntentType对应的Mask添加到它的冲突列表中
        if (IntentTypesConflict(intent_type, other_intent_type)) {
          result[idx] |= IntentTypeMask(other_intent_type);
        }
      }
    }
  }
  return result;
}
```

## kIntentTypeSetMask
用于获取给定idx对应的IntentTypeSet对应的Mask，比如idx为5，则对应的IntentTypeSet为{kWeakRead, kStrongRead}，则对应的Mask为：0b1111 1111 1111 1111 0000 0000 0000 0000 1111 1111 1111 1111。
```
const size_t kIntentTypeBits = 16;
# kSingleIntentMask = 0b1111 1111 1111 1111
const LockState kSingleIntentMask = (static_cast<LockState>(1) << kIntentTypeBits) - 1;

const std::array<LockState, kIntentTypeSetMapSize> kIntentTypeSetMask = GenerateByMask(
    kSingleIntentMask);
    
# 生成每种IntentType组合(即IntentTypeSet)对应的Mask    
std::array<LockState, kIntentTypeSetMapSize> GenerateByMask(LockState single_intent_mask) {
  DCHECK_EQ(single_intent_mask & kSingleIntentMask, single_intent_mask);
  std::array<LockState, kIntentTypeSetMapSize> result;
  for (size_t idx = 0; idx != kIntentTypeSetMapSize; ++idx) {
    result[idx] = 0;
    # 遍历idx对应的IntentTypeSet中的每种IntentType，获取该IntentType的Mask，保存到result[idx]中
    for (auto intent_type : IntentTypeSet(idx)) {
      result[idx] |= single_intent_mask << (to_underlying(intent_type) * kIntentTypeBits);
    }
  }
  return result;
}    
```
