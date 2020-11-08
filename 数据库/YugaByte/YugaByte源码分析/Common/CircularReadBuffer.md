# 提纲
[toc]

## CirtularReadBuffer的构建
在CircularReadBuffer的构造方法中会初始胡它对应的MemTracker，以及Buffer。
```
CircularReadBuffer::CircularReadBuffer(size_t capacity, const MemTrackerPtr& parent_tracker)
    : consumption_(MemTracker::FindOrCreateTracker("Receive", parent_tracker, AddToParent::kFalse),
                   capacity),
      buffer_(static_cast<char*>(malloc(capacity))), capacity_(capacity) {
}
```

## 判断CircularReadBuffer的buffer_中是否有可用空间
```
bool CircularReadBuffer::Empty() {
  return size_ == 0;
}
```

## 判断CircularReadBuffer是否被读取到的数据填充满了(buffer_满了，则CircularReadBuffer一定满了，即使有prepend_)
```
bool CircularReadBuffer::Full() {
  # buffer_中数据的大小等于buffer_的总容量
  return size_ == capacity_;
}
```

## 读取数据到CircularReadBuffer之前先从CircularReadBuffer中获取可以存放数据的IoVecs
```
Result<IoVecs> CircularReadBuffer::PrepareAppend() {
  IoVecs result;

  if (!prepend_.empty()) {
    # prepend_中还有可用空间，则读取的数据先写入prepend_中，prepend_.size()
    # 返回prepend_中可以用来存放数据的空间大小
    result.push_back(iovec{prepend_.mutable_data(), prepend_.size()});
  }

  # 接着，获取buffer_中可以存放数据的iovec
  size_t end = pos_ + size_;
  if (end < capacity_) {
    result.push_back(iovec{buffer_.get() + end, capacity_ - end});
  }
  size_t start = end <= capacity_ ? 0 : end - capacity_;
  if (pos_ > start) {
    result.push_back(iovec{buffer_.get() + start, pos_ - start});
  }

  if (result.empty()) {
    # 没有可用的IoVecs了
    static Status busy_status = STATUS(Busy, "Circular read buffer is full");
    return busy_status;
  }

  # 返回可以用于存放数据的IoVecs
  return result;
}
```

## 当读取数据到PrepareAppend()返回的IoVecs之后，更新相关元信息
当读取数据到PrepareAppend()返回的IoVecs的时候，如果prepend_不为空，则优先将数据添加到prepend_的空闲空间中，然后再将数据添加到buffer_中。DataAppended()则在读取了长度为len的数据之后调用，因为如果prepend_有可用空间，则会先将读取到的数据添加到prepend_中，所以先看读取到的数据是否全部填充在prepend_中，如果prepend_中填充满了，则会将数据添加到buffer_中。这里用于更新prepend_中的可用空间的起始位置，以及buffer_中的数据大小。
```
void CircularReadBuffer::DataAppended(size_t len) {
  if (!prepend_.empty()) {
    # prepend_有空闲空间，则读取到的数据会优先填充在prepend_中，在prepend_
    # 中填充的数据量至多为std::min(len, prepend_.size())
    size_t prepend_len = std::min(len, prepend_.size());
    
    # 更新prepend_中可用空间的第一个地址
    prepend_.remove_prefix(prepend_len);
    
    # 计算填充在buffer_中的数据的大小
    len -= prepend_len;
  }
  
  # 更新buffer_中存放的数据的大小
  size_ += len;
}
```

## 当读取数据到PrepareAppend()返回的IoVecs之后(并不一定IoVecs中的所有的IoVec都被填充了)，返回数据存放在哪些IoVecs中
```
IoVecs CircularReadBuffer::AppendedVecs() {
  IoVecs result;

  size_t end = pos_ + size_;
  if (end <= capacity_) {
    result.push_back(iovec{buffer_.get() + pos_, size_});
  } else {
    result.push_back(iovec{buffer_.get() + pos_, capacity_ - pos_});
    result.push_back(iovec{buffer_.get(), end - capacity_});
  }

  return result;
}
```

## 当从CircularReadBuffer中消费数据之后，更新相关元信息，并设置prepend_
```
void CircularReadBuffer::Consume(size_t count, const Slice& prepend) {
  # 更新剩余的可读取的数据的起始位置
  pos_ += count;
  if (pos_ >= capacity_) {
    pos_ -= capacity_;
  }
  
  # 更新剩余的可读取的数据的大小
  size_ -= count;
  if (size_ == 0) {
    pos_ = 0;
  }
  
  # 用给定的prepend来设置prepend_，后续从Socket中读取的时候都会优先将数据读取到prepend_中
  DCHECK(prepend_.empty());
  prepend_ = prepend;
  
  # 如果传递进来的prepend中有可用空间，则设置had_prepend为true
  had_prepend_ = !prepend.empty();
}
```

## 检查CircularReadBuffer中是否有可读取的数据
```
bool CircularReadBuffer::ReadyToRead() {
  # 如果当前prepend_中没有可用的空间了，且(之前prepend_中有可用的空间，
  # 或者当前的buffer_中有可读取的数据)
  #
  # 注意：这里prepend_.empty()是prepend_当前的状态，表示现在prepend_
  # 中没有可用空间了，而had_prepend_是在CircularReadBuffer::Consume
  # 中设置的，表示prepend_中之前有可用的空间
  return prepend_.empty() && (had_prepend_ || !Empty());
}
```





