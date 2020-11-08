# 提纲
[toc]

# LevelFilesBrief
结构体LevelFilesBrief用于保存某一level中所有文件信息，这一level中有多少文件，files数组中就具有多少个元素，每一个元素都是FdWithKeyRange类型，包括文件描述符，文件中最小key和文件中最大key。
```
// Data structure to store an array of FdWithKeyRange in one level
// Actual data is guaranteed to be stored closely
struct LevelFilesBrief {
  size_t num_files;
  FdWithKeyRange* files;
  LevelFilesBrief() {
    num_files = 0;
    files = nullptr;
  }
};

struct FdWithKeyRange {
  FileDescriptor fd;
  Slice smallest_key;    // slice that contain smallest key
  Slice largest_key;     // slice that contain largest key

  FdWithKeyRange()
      : fd(),
        smallest_key(),
        largest_key() {
  }

  FdWithKeyRange(FileDescriptor _fd, Slice _smallest_key, Slice _largest_key)
      : fd(_fd), smallest_key(_smallest_key), largest_key(_largest_key) {}
};
```
