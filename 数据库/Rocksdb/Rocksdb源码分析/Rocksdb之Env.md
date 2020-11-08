# 提纲
[toc]

# Env简介
作为一个跨平台的存储引擎，Rocksdb会面对各种平台，如linux/mac/win等，为了屏蔽各平台之间的差异，Rocksdb抽象出了Env类，并内部实现了env_posix/env_hdfs/memenv等Env。所有Env实现都支持可以多线程并发访问（而无需外部同步机制），用户可以根据实际需要自定义Env接口。

# Env中提供的接口
include/rocksdb/env.h中主要提供了以下类或者抽象：

- Env相关的配置选项
```
// Options while opening a file to read/write
struct EnvOptions { ... };
```

- 顺序读的文件抽象
```
// A file abstraction for reading sequentially through a file
class SequentialFile { ... };
```

- 随机读的文件抽象
```
// A file abstraction for randomly reading the contents of a file.
class RandomAccessFile { ... };
```

- 顺序写的文件抽象
```
// A file abstraction for sequential writing.  The implementation
// must provide buffering since callers may append small fragments
// at a time to the file.
class WritableFile { ... };
```

- 随机读写的文件抽象
```
// A file abstraction for random reading and writing.
class RandomRWFile { ... };
```

- 目录抽象
```
// Directory object represents collection of files and implements
// filesystem operations that can be executed on directories.
class Directory { ... };
```

- 日志接口
```
// An interface for writing log messages.
class Logger { ... };
```

- 锁住某个文件（用于阻止多个进程并发访问同一个数据库实例）
```
// Identifies a locked file.
class FileLock { ... };
```

- Env类
在Env类中提供了以下接口：
    文件操作；
    目录操作；
    锁住/解锁文件（防止多进程并发访问数据库）；
    计划任务；
    线程调度；
    创建线程；
    线程状态更新/获取；
    时间操作；
    为WAL写/Manifest文件写/SST File写/SST File读等优化相关的EnvOptions选项更新；
    
```
class Env {
 public:
  struct FileAttributes {
    // File name
    std::string name;

    // Size of file in bytes
    uint64_t size_bytes;
  };

  Env() : thread_status_updater_(nullptr) {}

  virtual ~Env();

  // Return a default environment suitable for the current operating
  // system.  Sophisticated users may wish to provide their own Env
  // implementation instead of relying on this default environment.
  //
  // The result of Default() belongs to rocksdb and must never be deleted.
  // 提供操作系统相关的默认Env
  static Env* Default();

  // Create a brand new sequentially-readable file with the specified name.
  // On success, stores a pointer to the new file in *result and returns OK.
  // On failure stores nullptr in *result and returns non-OK.  If the file does
  // not exist, returns a non-OK status.
  //
  // The returned file will only be accessed by one thread at a time.
  // 创建一个具有指定名称的全新的顺序读的文件
  virtual Status NewSequentialFile(const std::string& fname,
                                   unique_ptr<SequentialFile>* result,
                                   const EnvOptions& options)
                                   = 0;

  // Create a brand new random access read-only file with the
  // specified name.  On success, stores a pointer to the new file in
  // *result and returns OK.  On failure stores nullptr in *result and
  // returns non-OK.  If the file does not exist, returns a non-OK
  // status.
  //
  // The returned file may be concurrently accessed by multiple threads.
  // 创建一个具有指定名称的全新的随机的只读文件
  virtual Status NewRandomAccessFile(const std::string& fname,
                                     unique_ptr<RandomAccessFile>* result,
                                     const EnvOptions& options)
                                     = 0;

  // Create an object that writes to a new file with the specified
  // name.  Deletes any existing file with the same name and creates a
  // new file.  On success, stores a pointer to the new file in
  // *result and returns OK.  On failure stores nullptr in *result and
  // returns non-OK.
  //
  // The returned file will only be accessed by one thread at a time.
  // 创建一个具有指定名称的全新的顺序写的文件（事先删除已经存在的同名文件）
  virtual Status NewWritableFile(const std::string& fname,
                                 unique_ptr<WritableFile>* result,
                                 const EnvOptions& options) = 0;

  // Create an object that writes to a new file with the specified
  // name.  Deletes any existing file with the same name and creates a
  // new file.  On success, stores a pointer to the new file in
  // *result and returns OK.  On failure stores nullptr in *result and
  // returns non-OK.
  //
  // The returned file will only be accessed by one thread at a time.
  // 创建一个具有指定名称的全新的顺序写的文件（事先删除已经存在的同名文件）
  virtual Status ReopenWritableFile(const std::string& fname,
                                    unique_ptr<WritableFile>* result,
                                    const EnvOptions& options) {
    Status s;
    return s;
  }

  // Reuse an existing file by renaming it and opening it as writable.
  // 重用某个业已存在的文件（先重命名，然后打开之）
  virtual Status ReuseWritableFile(const std::string& fname,
                                   const std::string& old_fname,
                                   unique_ptr<WritableFile>* result,
                                   const EnvOptions& options);

  // Open `fname` for random read and write, if file doesn't exist the file
  // will be created.  On success, stores a pointer to the new file in
  // *result and returns OK.  On failure returns non-OK.
  //
  // The returned file will only be accessed by one thread at a time.
  // 打开一个具有指定名称的随机读写的文件（如果文件不存在则创建之）
  virtual Status NewRandomRWFile(const std::string& fname,
                                 unique_ptr<RandomRWFile>* result,
                                 const EnvOptions& options) {
    return Status::NotSupported("RandomRWFile is not implemented in this Env");
  }

  // Create an object that represents a directory. Will fail if directory
  // doesn't exist. If the directory exists, it will open the directory
  // and create a new Directory object.
  //
  // On success, stores a pointer to the new Directory in
  // *result and returns OK. On failure stores nullptr in *result and
  // returns non-OK.
  // 创建一个代表某个目录的Directory对象（如果目录不存在则失败，如果目录存在则打开该目录并创建Directory对象）
  virtual Status NewDirectory(const std::string& name,
                              unique_ptr<Directory>* result) = 0;

  // Returns OK if the named file exists.
  //         NotFound if the named file does not exist,
  //                  the calling process does not have permission to determine
  //                  whether this file exists, or if the path is invalid.
  //         IOError if an IO Error was encountered
  // 查看指定名称的文件是否存在
  virtual Status FileExists(const std::string& fname) = 0;

  // Store in *result the names of the children of the specified directory.
  // The names are relative to "dir".
  // Original contents of *results are dropped.
  // Returns OK if "dir" exists and "*result" contains its children.
  //         NotFound if "dir" does not exist, the calling process does not have
  //                  permission to access "dir", or if "dir" is invalid.
  //         IOError if an IO Error was encountered
  // 返回某个目录下所有的子文件
  virtual Status GetChildren(const std::string& dir,
                             std::vector<std::string>* result) = 0;

  // Store in *result the attributes of the children of the specified directory.
  // In case the implementation lists the directory prior to iterating the files
  // and files are concurrently deleted, the deleted files will be omitted from
  // result.
  // The name attributes are relative to "dir".
  // Original contents of *results are dropped.
  // Returns OK if "dir" exists and "*result" contains its children.
  //         NotFound if "dir" does not exist, the calling process does not have
  //                  permission to access "dir", or if "dir" is invalid.
  //         IOError if an IO Error was encountered
  // 返回某个目录下所有的子文件的属性
  virtual Status GetChildrenFileAttributes(const std::string& dir,
                                           std::vector<FileAttributes>* result);

  // Delete the named file.
  // 删除指定的文件
  virtual Status DeleteFile(const std::string& fname) = 0;

  // Create the specified directory. Returns error if directory exists.
  // 创建指定名称的目录（如果目录已经存在则返回错误）
  virtual Status CreateDir(const std::string& dirname) = 0;

  // Creates directory if missing. Return Ok if it exists, or successful in
  // Creating.
  // 创建指定名称的目录（如果已经存在或者创建成功则返回OK）
  virtual Status CreateDirIfMissing(const std::string& dirname) = 0;

  // Delete the specified directory.
  // 删除指定的目录
  virtual Status DeleteDir(const std::string& dirname) = 0;

  // Store the size of fname in *file_size.
  // 返回指定文件的大小
  virtual Status GetFileSize(const std::string& fname, uint64_t* file_size) = 0;

  // Store the last modification time of fname in *file_mtime.
  // 删除指定的文件的修改时间
  virtual Status GetFileModificationTime(const std::string& fname,
                                         uint64_t* file_mtime) = 0;
  // Rename file src to target.
  // 重命名某个文件
  virtual Status RenameFile(const std::string& src,
                            const std::string& target) = 0;

  // Hard Link file src to target.
  // 创建硬链接
  virtual Status LinkFile(const std::string& src, const std::string& target) {
    return Status::NotSupported("LinkFile is not supported for this Env");
  }

  // Lock the specified file.  Used to prevent concurrent access to
  // the same db by multiple processes.  On failure, stores nullptr in
  // *lock and returns non-OK.
  //
  // On success, stores a pointer to the object that represents the
  // acquired lock in *lock and returns OK.  The caller should call
  // UnlockFile(*lock) to release the lock.  If the process exits,
  // the lock will be automatically released.
  //
  // If somebody else already holds the lock, finishes immediately
  // with a failure.  I.e., this call does not wait for existing locks
  // to go away.
  //
  // May create the named file if it does not already exist.
  // 锁定指定的文件。用于阻止多个进程并发访问同一个数据库。
  virtual Status LockFile(const std::string& fname, FileLock** lock) = 0;

  // Release the lock acquired by a previous successful call to LockFile.
  // REQUIRES: lock was returned by a successful LockFile() call
  // REQUIRES: lock has not already been unlocked.
  // 解除锁定某个文件，和LockFile一一对应。
  virtual Status UnlockFile(FileLock* lock) = 0;

  // Priority for scheduling job in thread pool
  // 线程池中任务调度优先级
  enum Priority { LOW, HIGH, TOTAL };

  // Priority for requesting bytes in rate limiter scheduler
  // 限流调度优先级
  enum IOPriority {
    IO_LOW = 0,
    IO_HIGH = 1,
    IO_TOTAL = 2
  };

  // Arrange to run "(*function)(arg)" once in a background thread, in
  // the thread pool specified by pri. By default, jobs go to the 'LOW'
  // priority thread pool.

  // "function" may run in an unspecified thread.  Multiple functions
  // added to the same Env may run concurrently in different threads.
  // I.e., the caller may not assume that background work items are
  // serialized.
  // When the UnSchedule function is called, the unschedFunction
  // registered at the time of Schedule is invoked with arg as a parameter.
  // 在指定优先级的线程池中调度一个线程运行"(*function)(arg)"。默认地，任务
  // 进入优先级为"LOW"的线程池中。任务在哪个线程中执行是不确定的。
  // 如果注册了unschedFunction，则在调用UnSchedule的时候该函数会被调用
  virtual void Schedule(void (*function)(void* arg), void* arg,
                        Priority pri = LOW, void* tag = nullptr,
                        void (*unschedFunction)(void* arg) = 0) = 0;

  // Arrange to remove jobs for given arg from the queue_ if they are not
  // already scheduled. Caller is expected to have exclusive lock on arg.
  // 如果指定arg的任务尚未被调度执行，则从任务队列中删除之
  virtual int UnSchedule(void* arg, Priority pri) { return 0; }

  // Start a new thread, invoking "function(arg)" within the new thread.
  // When "function(arg)" returns, the thread will be destroyed.
  // 启动一个新的线程，执行"(*function)(arg)"，当"(*function)(arg)"返回的时候，
  // 线程将会被销毁
  virtual void StartThread(void (*function)(void* arg), void* arg) = 0;

  // Wait for all threads started by StartThread to terminate.
  // 等待所有通过StartThread启动的线程终止
  virtual void WaitForJoin() {}

  // Get thread pool queue length for specific thread pool.
  // 返回指定优先级的线程池中队列长度
  virtual unsigned int GetThreadPoolQueueLen(Priority pri = LOW) const {
    return 0;
  }

  // *path is set to a temporary directory that can be used for testing. It may
  // or many not have just been created. The directory may or may not differ
  // between runs of the same process, but subsequent calls will return the
  // same directory.
  // 返回测试用的临时目录
  virtual Status GetTestDirectory(std::string* path) = 0;

  // Create and return a log file for storing informational messages.
  // 创建日志文件
  virtual Status NewLogger(const std::string& fname,
                           shared_ptr<Logger>* result) = 0;

  // Returns the number of micro-seconds since some fixed point in time.
  // It is often used as system time such as in GenericRateLimiter
  // and other places so a port needs to return system time in order to work.
  // 返回从某个时间点以来的微秒数
  virtual uint64_t NowMicros() = 0;

  // Returns the number of nano-seconds since some fixed point in time. Only
  // useful for computing deltas of time in one run.
  // Default implementation simply relies on NowMicros.
  // In platform-specific implementations, NowNanos() should return time points
  // that are MONOTONIC.
  // 返回自从某个时间点以来的纳秒数
  virtual uint64_t NowNanos() {
    return NowMicros() * 1000;
  }

  // Sleep/delay the thread for the perscribed number of micro-seconds.
  // 休眠指定时间
  virtual void SleepForMicroseconds(int micros) = 0;

  // Get the current host name.
  // 返回当前Host Name
  virtual Status GetHostName(char* name, uint64_t len) = 0;

  // Get the number of seconds since the Epoch, 1970-01-01 00:00:00 (UTC).
  // 返回自从1970-01-01 00:00:00 (UTC)以来的秒数
  virtual Status GetCurrentTime(int64_t* unix_time) = 0;

  // Get full directory name for this db.
  // 返回当前数据库实例的绝对路径
  virtual Status GetAbsolutePath(const std::string& db_path,
      std::string* output_path) = 0;

  // The number of background worker threads of a specific thread pool
  // for this environment. 'LOW' is the default pool.
  // default number: 1
  // 设置指定优先级的线程池中后台工作线程的数目，默认线程个数是1
  virtual void SetBackgroundThreads(int number, Priority pri = LOW) = 0;

  // Enlarge number of background worker threads of a specific thread pool
  // for this environment if it is smaller than specified. 'LOW' is the default
  // pool.
  // 如果指定优先级的线程池中线程数目少于通过SetBackgroundThreads设定的数目，则
  // 增加指定优先级的线程池中线程数目
  virtual void IncBackgroundThreadsIfNeeded(int number, Priority pri) = 0;

  // Lower IO priority for threads from the specified pool.
  // 降低指定优先级的线程池中IO优先级
  virtual void LowerThreadPoolIOPriority(Priority pool = LOW) {}

  // Converts seconds-since-Jan-01-1970 to a printable string
  // 以字符串的形式返回自从1970-01-01 00:00:00 (UTC)以来的秒数
  virtual std::string TimeToString(uint64_t time) = 0;

  // Generates a unique id that can be used to identify a db
  // 为该数据库创建一个唯一的ID，用于标识该数据库
  virtual std::string GenerateUniqueId();

  // OptimizeForLogWrite will create a new EnvOptions object that is a copy of
  // the EnvOptions in the parameters, but is optimized for writing log files.
  // Default implementation returns the copy of the same object.
  // 基于给定的EnvOptions参数，创建一个新的EnvOptions对象，以为写WAL文件做优化
  virtual EnvOptions OptimizeForLogWrite(const EnvOptions& env_options,
                                         const DBOptions& db_options) const;
  // OptimizeForManifestWrite will create a new EnvOptions object that is a copy
  // of the EnvOptions in the parameters, but is optimized for writing manifest
  // files. Default implementation returns the copy of the same object.
  // 基于给定的EnvOptions参数，创建一个新的EnvOptions对象，以为写manifest文件做优化
  virtual EnvOptions OptimizeForManifestWrite(
      const EnvOptions& env_options) const;

  // OptimizeForCompactionTableWrite will create a new EnvOptions object that is a copy
  // of the EnvOptions in the parameters, but is optimized for writing table
  // files. Default implementation returns the copy of the same object.
  // 基于给定的EnvOptions参数，创建一个新的EnvOptions对象，以为写SST文件做优化
  virtual EnvOptions OptimizeForCompactionTableWrite(
      const EnvOptions& env_options,
      const ImmutableDBOptions& db_options) const;

  // OptimizeForCompactionTableWrite will create a new EnvOptions object that is a copy
  // of the EnvOptions in the parameters, but is optimized for reading table
  // files. Default implementation returns the copy of the same object.
  // 基于给定的EnvOptions参数，创建一个新的EnvOptions对象，以为读SST文件做优化
  virtual EnvOptions OptimizeForCompactionTableRead(
      const EnvOptions& env_options,
      const ImmutableDBOptions& db_options) const;

  // Returns the status of all threads that belong to the current Env.
  // 返回所有率属于当前Env的线程的状态
  virtual Status GetThreadList(std::vector<ThreadStatus>* thread_list) {
    return Status::NotSupported("Not supported.");
  }

  // Returns the pointer to ThreadStatusUpdater.  This function will be
  // used in RocksDB internally to update thread status and supports
  // GetThreadList().
  // 返回ThreadStatusUpdater指针，用于更新线程状态
  virtual ThreadStatusUpdater* GetThreadStatusUpdater() const {
    return thread_status_updater_;
  }

  // Returns the ID of the current thread.
  // 返回当前线程的ID
  virtual uint64_t GetThreadID() const;

 protected:
  // The pointer to an internal structure that will update the
  // status of each thread.
  // 用于更新每一个线程的状态
  ThreadStatusUpdater* thread_status_updater_;

 private:
  // No copying allowed
  Env(const Env&);
  void operator=(const Env&);
};
```

- 对Env的封装，将所有调用都转发给另一个Env对象
```
// An implementation of Env that forwards all calls to another Env.
// May be useful to clients who wish to override just part of the
// functionality of another Env.
class EnvWrapper : public Env { ... };
```

- 对WritableFile的封装，将所有调用都转发给另一个WritableFile对象
```
// An implementation of WritableFile that forwards all calls to another
// WritableFile. May be useful to clients who wish to override just part of the
// functionality of another WritableFile.
// It's declared as friend of WritableFile to allow forwarding calls to
// protected virtual methods.
class WritableFileWrapper : public WritableFile { ... };
```

- 全局函数 

```
// a set of log functions with different log levels.
// 不同日志级别的日志函数
extern void Header(Logger* info_log, const char* format, ...);
extern void Debug(Logger* info_log, const char* format, ...);
extern void Info(Logger* info_log, const char* format, ...);
extern void Warn(Logger* info_log, const char* format, ...);
extern void Error(Logger* info_log, const char* format, ...);
extern void Fatal(Logger* info_log, const char* format, ...);

// A utility routine: write "data" to the named file.
// 写字符串到指定名称的文件中
extern Status WriteStringToFile(Env* env, const Slice& data,
                                const std::string& fname,
                                bool should_sync = false);

// A utility routine: read contents of named file into *data
// 从指定名称的文件中读取字符串
extern Status ReadFileToString(Env* env, const std::string& fname,
                               std::string* data);
```

# Posix Env的实现
## 文件相关的接口
文件相关的接口实现上都大同小异，这里主要以顺序读文件为例讲解Posix Env中是如何实现顺序读文件相关接口的：

```
class PosixEnv : public Env {
  ......
    
  virtual Status NewSequentialFile(const std::string& fname,
                                   unique_ptr<SequentialFile>* result,
                                   const EnvOptions& options) override {
    result->reset();
    int fd = -1;
    /*flags初始化为O_RDONLY*/
    int flags = O_RDONLY;
    FILE* file = nullptr;

    /*根据options来设置flags*/
    if (options.use_direct_reads && !options.use_mmap_reads) {
#ifdef ROCKSDB_LITE
      return Status::IOError(fname, "Direct I/O not supported in RocksDB lite");
#endif  // !ROCKSDB_LITE
#if !defined(OS_MACOSX) && !defined(OS_OPENBSD) && !defined(OS_SOLARIS)
      flags |= O_DIRECT;
#endif
    }

    /*打开文件*/
    do {
      IOSTATS_TIMER_GUARD(open_nanos);
      fd = open(fname.c_str(), flags, 0644);
    } while (fd < 0 && errno == EINTR);
    if (fd < 0) {
      return IOError(fname, errno);
    }

    /* 如果options中设置了set_fd_cloexec，则通过fcntl设置该描述符在使用execl执行
     * 的程序里被关闭，而在使用fork调用的子进程中不关闭
     */
    SetFD_CLOEXEC(fd, &options);

    if (options.use_direct_reads && !options.use_mmap_reads) {
#ifdef OS_MACOSX
      if (fcntl(fd, F_NOCACHE, 1) == -1) {
        close(fd);
        return IOError(fname, errno);
      }
#endif
    } else {
      do {
        IOSTATS_TIMER_GUARD(open_nanos);
        /*只读模式打开fd对应的文件并关联一个标准的文件IO流file*/
        file = fdopen(fd, "r");
      } while (file == nullptr && errno == EINTR);
      if (file == nullptr) {
        close(fd);
        return IOError(fname, errno);
      }
    }
    
    /*通过fname，file，fd和options创建PosixSequentialFile对象并赋值给result*/
    result->reset(new PosixSequentialFile(fname, file, fd, options));
    return Status::OK();
  }
  
  ...
};
```

PosixSequentialFile继承自SequentialFile，并实现SequentialFile定义的接口：
```
class PosixSequentialFile : public SequentialFile {
 private:
  /*文件名*/
  std::string filename_;
  /*文件对象指针*/
  FILE* file_;
  /*文件句柄*/
  int fd_;
  /*是否使用DirectIO*/
  bool use_direct_io_;
  /*逻辑块大小（这里为什么命名为logical sector size???）*/
  size_t logical_sector_size_;

 public:
  PosixSequentialFile(const std::string& fname, FILE* file, int fd,
                      const EnvOptions& options);
  virtual ~PosixSequentialFile();

  virtual Status Read(size_t n, Slice* result, char* scratch) override;
  virtual Status PositionedRead(uint64_t offset, size_t n, Slice* result,
                                char* scratch) override;
  virtual Status Skip(uint64_t n) override;
  virtual Status InvalidateCache(size_t offset, size_t length) override;
  virtual bool use_direct_io() const override { return use_direct_io_; }
  virtual size_t GetRequiredBufferAlignment() const override {
    return logical_sector_size_;
  }
};
```

在PosixSequentialFile的构造函数中会根据传递的参数设置各个域，除logical_sector_size_以外都是直接赋值，通过GetLogicalBufferSize(fd_)来获取逻辑块大小并赋值给logical_sector_size_：
```
PosixSequentialFile::PosixSequentialFile(const std::string& fname, FILE* file,
                                         int fd, const EnvOptions& options)
    : filename_(fname),
      file_(file),
      fd_(fd),
      use_direct_io_(options.use_direct_reads),
      logical_sector_size_(GetLogicalBufferSize(fd_)) {
  assert(!options.use_direct_reads || !options.use_mmap_reads);
}
```

调用GetLogicalBufferSize(fd_)来获取logical sector size：
```
size_t GetLogicalBufferSize(int __attribute__((__unused__)) fd) {
#ifdef OS_LINUX
  struct stat buf;
  /*如果fstat失败，或者fstat返回信息对应的主设备号为0，则采用默认的kDefaultPageSize*/
  int result = fstat(fd, &buf);
  if (result == -1) {
    return kDefaultPageSize;
  }
  if (major(buf.st_dev) == 0) {
    // Unnamed devices (e.g. non-device mounts), reserved as null device number.
    // These don't have an entry in /sys/dev/block/. Return a sensible default.
    return kDefaultPageSize;
  }

  /*从/sys/dev/block/${major}:${minor}/queue/logical_block_size中读取逻辑块大小*/
  // Reading queue/logical_block_size does not require special permissions.
  const int kBufferSize = 100;
  char path[kBufferSize];
  char real_path[PATH_MAX + 1];
  snprintf(path, kBufferSize, "/sys/dev/block/%u:%u", major(buf.st_dev),
           minor(buf.st_dev));
  if (realpath(path, real_path) == nullptr) {
    return kDefaultPageSize;
  }
  std::string device_dir(real_path);
  if (!device_dir.empty() && device_dir.back() == '/') {
    device_dir.pop_back();
  }
  
  /*处理分区设备路径名问题*/
  // NOTE: sda3 does not have a `queue/` subdir, only the parent sda has it.
  // $ ls -al '/sys/dev/block/8:3'
  // lrwxrwxrwx. 1 root root 0 Jun 26 01:38 /sys/dev/block/8:3 ->
  // ../../block/sda/sda3
  size_t parent_end = device_dir.rfind('/', device_dir.length() - 1);
  if (parent_end == std::string::npos) {
    return kDefaultPageSize;
  }
  size_t parent_begin = device_dir.rfind('/', parent_end - 1);
  if (parent_begin == std::string::npos) {
    return kDefaultPageSize;
  }
  if (device_dir.substr(parent_begin + 1, parent_end - parent_begin - 1) !=
      "block") {
    device_dir = device_dir.substr(0, parent_end);
  }
  
  /*从该设备的queue/logical_block_size中读取逻辑块大小*/
  std::string fname = device_dir + "/queue/logical_block_size";
  FILE* fp;
  size_t size = 0;
  fp = fopen(fname.c_str(), "r");
  if (fp != nullptr) {
    char* line = nullptr;
    size_t len = 0;
    if (getline(&line, &len, fp) != -1) {
      sscanf(line, "%zu", &size);
    }
    free(line);
    fclose(fp);
  }
  if (size != 0 && (size & (size - 1)) == 0) {
    return size;
  }
#endif
  return kDefaultPageSize;
}
```

从PosixSequentialFile中读取：
```
Status PosixSequentialFile::Read(size_t n, Slice* result, char* scratch) {
  assert(result != nullptr && !use_direct_io());
  Status s;
  size_t r = 0;
  do {
    /* fread_unlocked是fread的非线程安全版本，因为它自己不加锁，也不会检查是否被其
     * 它调用锁住，从文件流file_中读取n个1字节的数据项，存放在scratch中
     */
    r = fread_unlocked(scratch, 1, n, file_);
  } while (r == 0 && ferror(file_) && errno == EINTR);
  /*创建一个引用scratch[0, r-1]的slice，并赋值给result*/
  *result = Slice(scratch, r);
  
  /*如果读取的字节数少于要求的字节数*/
  if (r < n) {
    if (feof(file_)) {
      // We leave status as ok if we hit the end of the file
      // We also clear the error so that the reads can continue
      // if a new data is written to the file
      /*到达文件末尾，则清除errno，以便在新的数据写入之后可以继续读取*/
      clearerr(file_);
    } else {
      // A partial read with an error: return a non-ok status
      /*发生了IO Error*/
      s = IOError(filename_, errno);
    }
  }
  // we need to fadvise away the entire range of pages because
  // we do not want readahead pages to be cached under buffered io
  /*
   * posix_fadvise是linux上对文件进行预取的系统调用，其中第四个参数int advice为预取
   * 的方式，主要有以下几种：
   * POSIX_FADV_NORMAL       无特别建议                 重置预读大小为默认值
   * POSIX_FADV_SEQUENTIAL   将要进行顺序操作           设预读大小为默认值的2 倍
   * POSIX_FADV_RANDOM       将要进行随机操作           将预读大小清零（禁止预读）
   * POSIX_FADV_NOREUSE      指定的数据将只访问一次     暂无动作
   * POSIX_FADV_WILLNEED     指定的数据即将被访问       立即预读数据到page cache
   * POSIX_FADV_DONTNEED     指定的数据近期不会被访问   立即从page cache 中丢弃数据
   *
   * 其中，POSIX_FADV_NORMAL、POSIX_FADV_SEQUENTIAL、POSIX_FADV_RANDOM是用来调整预取
   * 窗口的大小，POSIX_FADV_WILLNEED则可以将指定范围的磁盘文件读入到pagecache中，
   * POSIX_FADV_DONTNEED则将指定的磁盘文件中数据从page cache中换出。
   */
  Fadvise(fd_, 0, 0, POSIX_FADV_DONTNEED);  // free OS pages
  return s;
}
```

## 锁住/解锁文件

锁住指定名称的文件并返回关于该文件的FileLock。
```
  virtual Status LockFile(const std::string& fname, FileLock** lock) override {
    *lock = nullptr;
    Status result;
    int fd;
    {
      IOSTATS_TIMER_GUARD(open_nanos);
      /*以读写模式打开文件，如果不存在则创建之*/
      fd = open(fname.c_str(), O_RDWR | O_CREAT, 0644);
    }
    
    /*通过调用LockOrUnlock来实现文件加锁*/
    if (fd < 0) {
      result = IOError(fname, errno);
    } else if (LockOrUnlock(fname, fd, true) == -1) {
      /*加锁失败*/
      result = IOError("lock " + fname, errno);
      close(fd);
    } else {
      /* 加锁成功，通过fcntl设置该描述符在使用execl执行的程序里被关闭，而在使用fork
       * 调用的子进程中不关闭
       */
      SetFD_CLOEXEC(fd, nullptr);
      /*创建一个PosixFileLock管理该文件句柄和文件流*/
      PosixFileLock* my_lock = new PosixFileLock;
      my_lock->fd_ = fd;
      my_lock->filename = fname;
      *lock = my_lock;
    }
    return result;
  }
```

解锁指定文件（通过LockFile中返回的FileLock指定）。
```
  virtual Status UnlockFile(FileLock* lock) override {
    PosixFileLock* my_lock = reinterpret_cast<PosixFileLock*>(lock);
    Status result;
    /*通过调用LockOrUnlock来实现文件解锁*/
    if (LockOrUnlock(my_lock->filename, my_lock->fd_, false) == -1) {
      result = IOError("unlock", errno);
    }
    
    /*关闭文件*/
    close(my_lock->fd_);
    delete my_lock;
    return result;
  }
```

加锁/解锁都是通过fcntl来实现，成功加锁的文件会被存放在lockedFiles中，成功解锁的文件也会从lockedFiles中移除。加锁/解锁过程都是在全局的mutex_lockedFiles锁的保护下进行的。
```
  static int LockOrUnlock(const std::string& fname, int fd, bool lock) {
      /*mutex_lockedFiles是用于管理所有锁住的文件的全局锁*/
      mutex_lockedFiles.Lock();
      if (lock) {
        // If it already exists in the lockedFiles set, then it is already locked,
        // and fail this lock attempt. Otherwise, insert it into lockedFiles.
        // This check is needed because fcntl() does not detect lock conflict
        // if the fcntl is issued by the same thread that earlier acquired
        // this lock.
        /*加锁，尝试将文件插入到lockedFiles，lockedFiles表示所有已经加锁的文件*/
        if (lockedFiles.insert(fname).second == false) {
          /*该文件已经加锁，则直接返回*/
          mutex_lockedFiles.Unlock();
          errno = ENOLCK;
          return -1;
        }
      } else {
        // If we are unlocking, then verify that we had locked it earlier,
        // it should already exist in lockedFiles. Remove it from lockedFiles.
        /*解锁，尝试从lockedFiles中删除文件*/
        if (lockedFiles.erase(fname) != 1) {
          /*该文件尚未加锁，直接返回*/
          mutex_lockedFiles.Unlock();
          errno = ENOLCK;
          return -1;
        }
      }
      
      /*至此，文件需要加锁或者解锁*/
      errno = 0;
      struct flock f;
      memset(&f, 0, sizeof(f));
      /*设置flock的类型是加锁还是解锁*/
      f.l_type = (lock ? F_WRLCK : F_UNLCK);
      f.l_whence = SEEK_SET;
      f.l_start = 0;
      f.l_len = 0;        // Lock/unlock entire file
      /*通过fcntl在整个文件上加锁或者解锁*/
      int value = fcntl(fd, F_SETLK, &f);
      if (value == -1 && lock) {
        // if there is an error in locking, then remove the pathname from lockedfiles
        /*如果是加锁且加锁失败，则从lockedFiles中移除该文件*/
        lockedFiles.erase(fname);
      }
      mutex_lockedFiles.Unlock();
      return value;
}
```

PosixFileLock继承自FileLock，是对被加锁的文件句柄及其名称的封装。
```
class PosixFileLock : public FileLock {
 public:
  int fd_;
  std::string filename;
};
```

## 线程创建/销毁
### 创建线程
PosixEnv中线程相关（不包括线程池中的线程，线程池单独管理）的成员如下：
```
class PosixEnv : public Env {
    ......
    
    /*保护threads_to_join_*/
    pthread_mutex_t mu_;
    std::vector<pthread_t> threads_to_join_;
    
    ......
};
```

创建线程（**这里创建的线程不会加入到线程池中**），并添加到threads_to_join_数组中，在加入到threads_to_join_数组的过程中需要加锁保护，threads_to_join_数组中存放那些已经创建的，等待回收的线程：
```
void PosixEnv::StartThread(void (*function)(void* arg), void* arg) {
  pthread_t t;
  /*StartThreadState是对线程执行函数和参数的封装*/
  StartThreadState* state = new StartThreadState;
  /*分别设置线程执行函数和参数*/
  state->user_function = function;
  state->arg = arg;
  /*调用pthread_create，如果创建线程失败，则打印错误信息并终止进程*/
  ThreadPoolImpl::PthreadCall(
      "start thread", pthread_create(&t, nullptr, &StartThreadWrapper, state));
  /*调用pthread_mutex_lock加锁，如果加锁失败，则打印错误信息并终止进程*/
  ThreadPoolImpl::PthreadCall("lock", pthread_mutex_lock(&mu_));
  /*将创建的线程添加到threads_to_join_数组中*/
  threads_to_join_.push_back(t);
  /*调用pthread_mutex_unlock解锁，如果解锁失败，则打印错误信息并终止进程*/
  ThreadPoolImpl::PthreadCall("unlock", pthread_mutex_unlock(&mu_));
}
```

在PosixEnv::StartThread中通过调用pthread_create来创建线程，线程执行函数为StartThreadWrapper，StartThreadWrapper只是对用户指定的运行函数及其参数的封装，最终还是调用用户指定的运行函数：
```
static void* StartThreadWrapper(void* arg) {
  StartThreadState* state = reinterpret_cast<StartThreadState*>(arg);
  state->user_function(state->arg);
  delete state;
  return nullptr;
}
```

### 销毁所有线程
依次在threads_to_join_数组中的线程上调用pthread_join，并在最后清空threads_to_join_数组：

```
void PosixEnv::WaitForJoin() {
  for (const auto tid : threads_to_join_) {
    pthread_join(tid, nullptr);
  }
  threads_to_join_.clear();
}
```

### 使用示例
EnvWrapper继承自Env类，用户调用EnvWrapper类的方法的时候它直接调用target_ Env的方法来实现：
```
class EnvWrapper : public Env {
  ......

  void StartThread(void (*f)(void*), void* a) override {
    /*调用target_的方法来实现*/
    return target_->StartThread(f, a);
  }
  
  ......
  
private:
  /*target Env*/
  Env* target_;
};
```

在db/db_test.cc中会调用StartThread启动多个线程来测试多线程读取数据库：
```
TEST_P(MultiThreadedDBTest, MultiThreaded) {
  ......
  
  /*启动多个线程*/
  MTThread thread[kNumThreads];
  for (int id = 0; id < kNumThreads; id++) {
    thread[id].state = &mt;
    thread[id].id = id;
    env_->StartThread(MTThreadBody, &thread[id]);
  }
  
  ......
}
```

## 线程池管理

PosixEnv中线程池用于后台任务处理，它包含一个线程池数组，其中包含两个元素，分别对应低优先级的线程池和高优先级的线程池。线程池中线程的创建和销毁是通过计划任务/撤销任务，或者设置线程数目来实现的。
```
class PosixEnv : public Env {
  ......
  
  /*线程池（通过PosixEnv::StartThread启动的线程不会在这里管理）*/
  std::vector<ThreadPoolImpl> thread_pools_;
  
  ......
};
```

在PosixEnv的构造函数中初始化线程池：

```
PosixEnv::PosixEnv()
    : checkedDiskForMmap_(false),
      forceMmapOff_(false),
      page_size_(getpagesize()),
      thread_pools_(Priority::TOTAL) {
  ThreadPoolImpl::PthreadCall("mutex_init", pthread_mutex_init(&mu_, nullptr));
  /*分别初始化低优先级的线程池和高优先级的线程池*/
  for (int pool_id = 0; pool_id < Env::Priority::TOTAL; ++pool_id) {
    thread_pools_[pool_id].SetThreadPriority(
        static_cast<Env::Priority>(pool_id));
    // This allows later initializing the thread-local-env of each thread.
    thread_pools_[pool_id].SetHostEnv(this);
  }
  
  thread_status_updater_ = CreateThreadStatusUpdater();
}
```


设置线程池的最大线程数目：
```
  virtual void SetBackgroundThreads(int num, Priority pri) override {
    assert(pri >= Priority::LOW && pri <= Priority::HIGH);
    thread_pools_[pri].SetBackgroundThreads(num);
  }
```

增加线程池中线程数目：
```
  virtual void IncBackgroundThreadsIfNeeded(int num, Priority pri) override {
    assert(pri >= Priority::LOW && pri <= Priority::HIGH);
    thread_pools_[pri].IncBackgroundThreadsIfNeeded(num);
  }
```

降低线程池中各线程的调度优先级：
```
  virtual void LowerThreadPoolIOPriority(Priority pool = LOW) override {
    assert(pool >= Priority::LOW && pool <= Priority::HIGH);
#ifdef OS_LINUX
    thread_pools_[pool].LowerIOPriority();
#endif
  }
```

## 线程状态更新/获取
### 线程状态维护
在Env类中有一个称为thread_status_updater_的成员，专门用于管理线程状态：
```
class Env {
  ......
  
  protected:
    // The pointer to an internal structure that will update the
    // status of each thread.
    /*用于线程状态管理*/
    ThreadStatusUpdater* thread_status_updater_;
    
  ......
};
```

在PosixEnv中会初始化Env::thread_status_updater_:
```
PosixEnv::PosixEnv()
    : checkedDiskForMmap_(false),
      forceMmapOff_(false),
      page_size_(getpagesize()),
      thread_pools_(Priority::TOTAL) {
  ThreadPoolImpl::PthreadCall("mutex_init", pthread_mutex_init(&mu_, nullptr));
  for (int pool_id = 0; pool_id < Env::Priority::TOTAL; ++pool_id) {
    thread_pools_[pool_id].SetThreadPriority(
        static_cast<Env::Priority>(pool_id));
    // This allows later initializing the thread-local-env of each thread.
    thread_pools_[pool_id].SetHostEnv(this);
  }
  
  /*创建Thread status updater*/
  thread_status_updater_ = CreateThreadStatusUpdater();
}

ThreadStatusUpdater* CreateThreadStatusUpdater() {
  return new ThreadStatusUpdater();
}
```

ThreadStatusUpdater中存储了各自后台线程的状态和所有后台线程状态的指针，每个线程的状态在ThreadStatusData中记录。

ThreadStatusUpdater定义如下（结合ThreadStatusData的定义可以看到只有在打开了ROCKSDB_USING_THREAD_STATUS宏开关的情况下才会维护关于线程的状态信息）：
```
class ThreadStatusUpdater {
 protected:
#ifdef ROCKSDB_USING_THREAD_STATUS
  // The thread-local variable for storing thread status.
  static __thread ThreadStatusData* thread_status_data_;

  // Returns the pointer to the thread status data only when the
  // thread status data is non-null and has enable_tracking == true.
  ThreadStatusData* GetLocalThreadStatus();

  // Directly returns the pointer to thread_status_data_ without
  // checking whether enabling_tracking is true of not.
  ThreadStatusData* Get() {
    return thread_status_data_;
  }

  // The mutex that protects cf_info_map and db_key_map.
  std::mutex thread_list_mutex_;

  // The current status data of all active threads.
  /*所有注册的线程的状态集合*/
  std::unordered_set<ThreadStatusData*> thread_data_set_;

  // A global map that keeps the column family information.  It is stored
  // globally instead of inside DB is to avoid the situation where DB is
  // closing while GetThreadList function already get the pointer to its
  // CopnstantColumnFamilyInfo.
  /* 所有线程共享该map，用于保存每个线程的column family信息，key是ColumnFamilyInfoKey，
   * value是ColumFamilyInfo，在访问之前，必须加锁（thread_list_mutex_）
   */
  std::unordered_map<
      const void*, std::unique_ptr<ConstantColumnFamilyInfo>> cf_info_map_;

  // A db_key to cf_key map that allows erasing elements in cf_info_map
  // associated to the same db_key faster.
  /* db key到ColumnFamilyInfokey集合的映射（一个DB实例可能会保存多个ColumnFamilyInfo
   * 信息，所以会为每个DB实例维护一个ColumnFamilyInfoKey集合，即这里的unordered_set）
   */
  std::unordered_map<
      const void*, std::unordered_set<const void*>> db_key_map_;

#else
  static ThreadStatusData* thread_status_data_;
#endif  // ROCKSDB_USING_THREAD_STATUS
};
```

ThreadStatusData定义如下（可以看到只有在打开了ROCKSDB_USING_THREAD_STATUS宏开关的情况下才会有关于线程的状态信息）：
```
struct ThreadStatusData {
#ifdef ROCKSDB_USING_THREAD_STATUS
  explicit ThreadStatusData() : enable_tracking(false) {
    thread_id.store(0);
    thread_type.store(ThreadStatus::USER);
    cf_key.store(nullptr);
    operation_type.store(ThreadStatus::OP_UNKNOWN);
    op_start_time.store(0);
    state_type.store(ThreadStatus::STATE_UNKNOWN);
  }

  // A flag to indicate whether the thread tracking is enabled
  // in the current thread.  This value will be updated based on whether
  // the associated Options::enable_thread_tracking is set to true
  // in ThreadStatusUtil::SetColumnFamily().
  //
  // If set to false, then SetThreadOperation and SetThreadState
  // will be no-op.
  bool enable_tracking;

  /*线程ID，类型*/
  std::atomic<uint64_t> thread_id;
  std::atomic<ThreadStatus::ThreadType> thread_type;
  std::atomic<void*> cf_key;
  std::atomic<ThreadStatus::OperationType> operation_type;
  std::atomic<uint64_t> op_start_time;
  std::atomic<ThreadStatus::OperationStage> operation_stage;
  std::atomic<uint64_t> op_properties[ThreadStatus::kNumOperationProperties];
  std::atomic<ThreadStatus::StateType> state_type;
#endif  // ROCKSDB_USING_THREAD_STATUS
};
```

### 线程状态维护相关的方法
#### 注册某个线程以跟踪其状态
```
/*线程局部变量*/
__thread ThreadStatusData* ThreadStatusUpdater::thread_status_data_ = nullptr;

void ThreadStatusUpdater::RegisterThread(
    ThreadStatus::ThreadType ttype, uint64_t thread_id) {
  if (UNLIKELY(thread_status_data_ == nullptr)) {
    /*为当前线程创建ThreadStatusData*/
    thread_status_data_ = new ThreadStatusData();
    thread_status_data_->thread_type = ttype;
    thread_status_data_->thread_id = thread_id;
    /*在锁的保护之下添加到thread_data_set_集合中*/
    std::lock_guard<std::mutex> lck(thread_list_mutex_);
    thread_data_set_.insert(thread_status_data_);
  }

  ClearThreadOperationProperties();
}
```

#### 不再跟踪当前线程的状态
```
void ThreadStatusUpdater::UnregisterThread() {
  /*从thread_data_set_集合中移除当前线程的thread_status_data_*/
  if (thread_status_data_ != nullptr) {
    std::lock_guard<std::mutex> lck(thread_list_mutex_);
    thread_data_set_.erase(thread_status_data_);
    delete thread_status_data_;
    thread_status_data_ = nullptr;
  }
}
```

#### 重置当前线程状态

```
void ThreadStatusUpdater::ResetThreadStatus() {
  /*重置ColumnFamilyInfoKey、ThreadState和ThreadOperation*/
  ClearThreadState();
  ClearThreadOperation();
  SetColumnFamilyInfoKey(nullptr);
}
```

#### 设置当前线程的Column Family Info key
```
void ThreadStatusUpdater::SetColumnFamilyInfoKey(
    const void* cf_key) {
  /*获取线程状态thread_status_data_*/
  auto* data = Get();
  if (data == nullptr) {
    return;
  }
  // set the tracking flag based on whether cf_key is non-null or not.
  // If enable_thread_tracking is set to false, the input cf_key
  // would be nullptr.
  data->enable_tracking = (cf_key != nullptr);
  data->cf_key.store(const_cast<void*>(cf_key), std::memory_order_relaxed);
}
```

#### 在全局的ColumnFamilyInfo表中为特定的ColumnFamily创建一个entry
```
void ThreadStatusUpdater::NewColumnFamilyInfo(
    const void* db_key, const std::string& db_name,
    const void* cf_key, const std::string& cf_name) {
  // Acquiring same lock as GetThreadList() to guarantee
  // a consistent view of global column family table (cf_info_map).
  std::lock_guard<std::mutex> lck(thread_list_mutex_);

  /*添加cf_key -> ColumnFamilyInfo的映射*/
  cf_info_map_[cf_key].reset(
      new ConstantColumnFamilyInfo(db_key, db_name, cf_name));
  /*添加db_key -> ColumnFamilyKey set的映射*/
  db_key_map_[db_key].insert(cf_key);
}
```

#### 删除特定ColumnFamilyKey对应的ColumnFamilyInfo
```
void ThreadStatusUpdater::EraseColumnFamilyInfo(const void* cf_key) {
  // Acquiring same lock as GetThreadList() to guarantee
  // a consistent view of global column family table (cf_info_map).
  std::lock_guard<std::mutex> lck(thread_list_mutex_);
  auto cf_pair = cf_info_map_.find(cf_key);
  if (cf_pair == cf_info_map_.end()) {
    return;
  }

  auto* cf_info = cf_pair->second.get();
  assert(cf_info);

  // Remove its entry from db_key_map_ by the following steps:
  // 1. Obtain the entry in db_key_map_ whose set contains cf_key
  // 2. Remove it from the set.
  auto db_pair = db_key_map_.find(cf_info->db_key);
  assert(db_pair != db_key_map_.end());
  size_t result __attribute__((unused)) = db_pair->second.erase(cf_key);
  assert(result);

  cf_pair->second.reset();
  result = cf_info_map_.erase(cf_key);
  assert(result);
}
```

#### 删除某个DB实例对应的全部ColumnFamilyInfo
```
void ThreadStatusUpdater::EraseDatabaseInfo(const void* db_key) {
  // Acquiring same lock as GetThreadList() to guarantee
  // a consistent view of global column family table (cf_info_map).
  std::lock_guard<std::mutex> lck(thread_list_mutex_);
  /*找到DB实例对应的全部ColumnFamilyInfoKey*/
  auto db_pair = db_key_map_.find(db_key);
  if (UNLIKELY(db_pair == db_key_map_.end())) {
    // In some occasional cases such as DB::Open fails, we won't
    // register ColumnFamilyInfo for a db.
    return;
  }

  size_t result __attribute__((unused)) = 0;
  /*遍历该DB实例对应的ColumnFamilyInfoKey集合中的每一个ColumnFamilyInfoKey*/
  for (auto cf_key : db_pair->second) {
    /*找到该ColumnFamilyInfoKey对应的ColumnFamilyInfo*/
    auto cf_pair = cf_info_map_.find(cf_key);
    if (cf_pair == cf_info_map_.end()) {
      continue;
    }
    
    /*删除对应的ColumnFamilyInfo*/
    cf_pair->second.reset();
    result = cf_info_map_.erase(cf_key);
    assert(result);
  }
  
  /*删除该DB实例对应的ColumnFamilyInfoKey集合*/
  db_key_map_.erase(db_key);
}
```

#### 设置当前线程操作的起始时间
```
void ThreadStatusUpdater::SetOperationStartTime(const uint64_t start_time) {
  auto* data = GetLocalThreadStatus();
  if (data == nullptr) {
    return;
  }
  data->op_start_time.store(start_time, std::memory_order_relaxed);
}
```

#### 设置线程当前操作的第i个属性
```
void ThreadStatusUpdater::SetThreadOperationProperty(
    int i, uint64_t value) {
  auto* data = GetLocalThreadStatus();
  if (data == nullptr) {
    return;
  }
  data->op_properties[i].store(value, std::memory_order_relaxed);
}
```

#### 增加线程当前操作的第i个属性
```
void ThreadStatusUpdater::IncreaseThreadOperationProperty(
    int i, uint64_t delta) {
  auto* data = GetLocalThreadStatus();
  if (data == nullptr) {
    return;
  }
  data->op_properties[i].fetch_add(delta, std::memory_order_relaxed);
}
```

#### 清除线程当前操作的属性
```
void ThreadStatusUpdater::ClearThreadOperationProperties() {
  auto* data = GetLocalThreadStatus();
  if (data == nullptr) {
    return;
  }
  for (int i = 0; i < ThreadStatus::kNumOperationProperties; ++i) {
    data->op_properties[i].store(0, std::memory_order_relaxed);
  }
}
```

#### 清除线程当前操作
```
void ThreadStatusUpdater::ClearThreadOperation() {
  auto* data = GetLocalThreadStatus();
  if (data == nullptr) {
    return;
  }
  data->operation_stage.store(ThreadStatus::STAGE_UNKNOWN,
      std::memory_order_relaxed);
  data->operation_type.store(
      ThreadStatus::OP_UNKNOWN, std::memory_order_relaxed);
  ClearThreadOperationProperties();
}
```

#### 设置线程当前操作所处的阶段
```
ThreadStatus::OperationStage ThreadStatusUpdater::SetThreadOperationStage(
    ThreadStatus::OperationStage stage) {
  auto* data = GetLocalThreadStatus();
  if (data == nullptr) {
    return ThreadStatus::STAGE_UNKNOWN;
  }
  return data->operation_stage.exchange(
      stage, std::memory_order_relaxed);
}
```

#### 设置线程状态
```
void ThreadStatusUpdater::SetThreadState(
    const ThreadStatus::StateType type) {
  auto* data = GetLocalThreadStatus();
  if (data == nullptr) {
    return;
  }
  data->state_type.store(type, std::memory_order_relaxed);
}
```

#### 清除线程状态
```
void ThreadStatusUpdater::ClearThreadState() {
  auto* data = GetLocalThreadStatus();
  if (data == nullptr) {
    return;
  }
  data->state_type.store(
      ThreadStatus::STATE_UNKNOWN, std::memory_order_relaxed);
}
```

#### 获取所有注册的线程状态
```
Status ThreadStatusUpdater::GetThreadList(
    std::vector<ThreadStatus>* thread_list) {
  thread_list->clear();
  std::vector<std::shared_ptr<ThreadStatusData>> valid_list;
  uint64_t now_micros = Env::Default()->NowMicros();

  /*在锁的保护之下遍历thread_data_set_中的每一个线程状态数据thread_data*/
  std::lock_guard<std::mutex> lck(thread_list_mutex_);
  for (auto* thread_data : thread_data_set_) {
    assert(thread_data);
    auto thread_id = thread_data->thread_id.load(
        std::memory_order_relaxed);
    auto thread_type = thread_data->thread_type.load(
        std::memory_order_relaxed);
    // Since any change to cf_info_map requires thread_list_mutex,
    // which is currently held by GetThreadList(), here we can safely
    // use "memory_order_relaxed" to load the cf_key.
    /*加载线程当前的ColumnFamilyInfoKey（通过SetColumnFamilyInfoKey来设置）*/
    auto cf_key = thread_data->cf_key.load(
        std::memory_order_relaxed);
    /*获取ColumnFamilyInfoKey对应的ColumnFamilyInfo*/
    auto iter = cf_info_map_.find(cf_key);
    auto* cf_info = iter != cf_info_map_.end() ?
        iter->second.get() : nullptr;
    const std::string* db_name = nullptr;
    const std::string* cf_name = nullptr;
    ThreadStatus::OperationType op_type = ThreadStatus::OP_UNKNOWN;
    ThreadStatus::OperationStage op_stage = ThreadStatus::STAGE_UNKNOWN;
    ThreadStatus::StateType state_type = ThreadStatus::STATE_UNKNOWN;
    uint64_t op_elapsed_micros = 0;
    uint64_t op_props[ThreadStatus::kNumOperationProperties] = {0};
    
    if (cf_info != nullptr) {
      /*从ColumnFamilyInfo中db_name，cf_name等信息*/
      db_name = &cf_info->db_name;
      cf_name = &cf_info->cf_name;
      /*从thread_data中解析出operation_type，operation_stage和state_type等信息*/
      op_type = thread_data->operation_type.load(
          std::memory_order_acquire);
      // display lower-level info only when higher-level info is available.
      if (op_type != ThreadStatus::OP_UNKNOWN) {
        op_elapsed_micros = now_micros - thread_data->op_start_time.load(
            std::memory_order_relaxed);
        op_stage = thread_data->operation_stage.load(
            std::memory_order_relaxed);
        state_type = thread_data->state_type.load(
            std::memory_order_relaxed);
        for (int i = 0; i < ThreadStatus::kNumOperationProperties; ++i) {
          op_props[i] = thread_data->op_properties[i].load(
              std::memory_order_relaxed);
        }
      }
    }
    
    /*emplace_back在vector的末尾添加一个新的元素（通过给定参数生成一个临时变量）*/
    thread_list->emplace_back(
        thread_id, thread_type,
        db_name ? *db_name : "",
        cf_name ? *cf_name : "",
        op_type, op_elapsed_micros, op_stage, op_props,
        state_type);
  }

  return Status::OK();
}
```

### 使用示例
在PosixEnv构造函数中，创建了thread_status_updater_，那么又是如何使用的呢？可以通过如下两种方式：

- 通过Env::GetThreadStatusUpdater()或者EnvWrapper::GetThreadStatusUpdater()来获取ThreadStatusUpdater，然后调用ThreadStatusUpdater的相关方法

在Env类和EnvWrapper类中都提供了GetThreadStatusUpdater()方法用于获取ThreadStatusUpdater实例：
```
class Env {
  ......
  
  virtual ThreadStatusUpdater* GetThreadStatusUpdater() const {
    return thread_status_updater_;
  }
  
  ......
};
  
class EnvWrapper : public Env {
  ......
  
  ThreadStatusUpdater* GetThreadStatusUpdater() const override {
    return target_->GetThreadStatusUpdater();
  }
  
  ......
};
```

然后通过调用Env::GetThreadStatusUpdater()或者EnvWrapper::GetThreadStatusUpdater()来获取ThreadStatusUpdater实例并调用其方法，如rocksdb::SimulatedBackgroundTask::Run：

```
  void Run() {
    std::unique_lock<std::mutex> l(mutex_);
    running_count_++;
    /* 对于Linux来说，Env::Default()返回的就是PosixEnv的实例，调用
     * GetThreadStatusUpdater()获取ThreadStatusUpdater实例，并调用其相关方法
     */
    Env::Default()->GetThreadStatusUpdater()->SetColumnFamilyInfoKey(cf_key_);
    Env::Default()->GetThreadStatusUpdater()->SetThreadOperation(
        operation_type_);
    Env::Default()->GetThreadStatusUpdater()->SetThreadState(state_type_);
    while (should_run_) {
      bg_cv_.wait(l);
    }
    Env::Default()->GetThreadStatusUpdater()->ClearThreadState();
    Env::Default()->GetThreadStatusUpdater()->ClearThreadOperation();
    Env::Default()->GetThreadStatusUpdater()->SetColumnFamilyInfoKey(0);
    running_count_--;
    bg_cv_.notify_all();
  }
```

- 通过ThreadStatusUtil::thread_updater_local_cache_来调用ThreadStatusUpdater的相关方法：

ThreadStatusUtil提供了一系列用于维护线程状态的静态函数，其中维护了一个线程本地存储的ThreadStatusUpdater类型的指针thread_updater_local_cache_以及一个指示thread_updater_local_cache是否被初始化的标识：
```
class ThreadStatusUtil {
 protected:
  // Initialize the thread-local ThreadStatusUpdater when it finds
  // the cached value is nullptr.  Returns true if it has cached
  // a non-null pointer.
  static bool MaybeInitThreadLocalUpdater(const Env* env);

#ifdef ROCKSDB_USING_THREAD_STATUS
  // A boolean flag indicating whether thread_updater_local_cache_
  // is initialized.  It is set to true when an Env uses any
  // ThreadStatusUtil functions using the current thread other
  // than UnregisterThread().  It will be set to false when
  // UnregisterThread() is called.
  //
  // When this variable is set to true, thread_updater_local_cache_
  // will not be updated until this variable is again set to false
  // in UnregisterThread().
  /* 指示thread_updater_local_cache是否被初始化，当调用除UnregisterThread
   * 以外的所有方法的时候，都会设置该标识为true，调用UnregisterThread的时
   * 候，则会设置它为false，当它为true时，thread_updater_local_cache_不会
   * 被更新，直到它被设置为false
   */
  static  __thread bool thread_updater_initialized_;

  // The thread-local cached ThreadStatusUpdater that caches the
  // thread_status_updater_ of the first Env that uses any ThreadStatusUtil
  // function other than UnregisterThread().  This variable will
  // be cleared when UnregisterThread() is called.
  //
  // When this variable is set to a non-null pointer, then the status
  // of the current thread will be updated when a function of
  // ThreadStatusUtil is called.  Otherwise, all functions of
  // ThreadStatusUtil will be no-op.
  //
  // When thread_updater_initialized_ is set to true, this variable
  // will not be updated until this thread_updater_initialized_ is
  // again set to false in UnregisterThread().
  /* 线程本地关于Env::thread_status_updater_的缓存，在该指针不为空的情况下，
   * 调用ThreadStatusUtil的任何方法都会更新本线程的状态
   */
  static __thread ThreadStatusUpdater* thread_updater_local_cache_;
#else
  static bool thread_updater_initialized_;
  static ThreadStatusUpdater* thread_updater_local_cache_;
#endif
};
```

每当调用ThreadStatusUtil的方法去更新/获取线程状态的时候都会检查thread_updater_local_cache_是否初始化了（通过thread_updater_initialized_决定），如果没有初始化之，则会尝试通过Env::GetThreadStatusUpdater()初始化之：
```
bool ThreadStatusUtil::MaybeInitThreadLocalUpdater(const Env* env) {
  if (!thread_updater_initialized_ && env != nullptr) {
    thread_updater_initialized_ = true;
    thread_updater_local_cache_ = env->GetThreadStatusUpdater();
  }
  return (thread_updater_local_cache_ != nullptr);
}
```

在ThreadStatusUtil调用线程状态维护相关的函数的时候，就会通过通过thread_updater_local_cache_调用ThreadStatusUpdater的方法来实现，比如ThreadStatusUtil::SetThreadState就是通过thread_updater_local_cache_->SetThreadState来实现：
```
void ThreadStatusUtil::SetThreadState(ThreadStatus::StateType state) {
  if (thread_updater_local_cache_ == nullptr) {
    // thread_updater_local_cache_ must be set in SetColumnFamily
    // or other ThreadStatusUtil functions.
    return;
  }

  /*实际上调用ThreadStatusUpdater::SetThreadState(state)*/
  thread_updater_local_cache_->SetThreadState(state);
}
```

## 计划任务/撤销任务
### 计划任务
PosixEnv中线程池相关的成员如下：
```
class PosixEnv : public Env {
    ......
    
    /*具有有2个元素的线程池数组，数组元素分别为优先级为LOW的线程池和优先级为HIGH的线程池*/
    std::vector<ThreadPoolImpl> thread_pools_;
};
```

ThreadPoolImpl继承ThreadPool接口，同时封装了ThreadPoolImpl::Impl：
```
class ThreadPoolImpl : public ThreadPool {
   std::unique_ptr<Impl>   impl_;
};
```

ThreadPoolImpl::Impl中封装了任务队列和线程数组，定义如下：
```
struct ThreadPoolImpl::Impl {
  ......
  
private:

  static void* BGThreadWrapper(void* arg);

  /* 是否需要降低线程池中线程的优先级（可以通过ThreadPoolImpl::Impl::
   * LowerIOPriority()来设置之）
   */
  bool low_io_priority_;
  /*该线程池的优先级，可能为LOW或者HIGH*/
  Env::Priority priority_;
  Env*         env_;

  /*线程池中线程最大数目*/
  int total_threads_limit_;
  std::atomic_uint queue_len_;  // Queue length. Used for stats reporting
  /*终止线程池中的所有线程*/
  bool exit_all_threads_;
  /*等待任务结束*/
  bool wait_for_jobs_to_complete_;

  // Entry per Schedule()/Submit() call
  /*以std::function定义的后台任务*/
  struct BGItem {
    void* tag = nullptr;
    std::function<void()> function;
    std::function<void()> unschedFunction;
  };

  /*任务队列*/
  using BGQueue = std::deque<BGItem>;
  BGQueue       queue_;

  std::mutex               mu_;
  std::condition_variable  bgsignal_;
  /*线程数组*/
  std::vector<port::Thread> bgthreads_;
};
```

ThreadPoolImpl借助ThreadPoolImpl::Impl来实现ThreadPool中的接口，如：
```
void ThreadPoolImpl::SubmitJob(std::function<void()>&& job) {
  impl_->Submit(std::move(job), std::function<void()>(), nullptr);
}
```

在PosixEnv的构造函数中会向线程池数组中分别添加优先级为LOW的线程池和优先级为HIGH的线程池：
```
PosixEnv::PosixEnv()
    : checkedDiskForMmap_(false),
      forceMmapOff_(false),
      page_size_(getpagesize()),
      thread_pools_(Priority::TOTAL) {
  ThreadPoolImpl::PthreadCall("mutex_init", pthread_mutex_init(&mu_, nullptr));
  for (int pool_id = 0; pool_id < Env::Priority::TOTAL; ++pool_id) {
    /*设置线程池的优先级*/
    thread_pools_[pool_id].SetThreadPriority(
        static_cast<Env::Priority>(pool_id));
    // This allows later initializing the thread-local-env of each thread.
    /*设置线程池的Env*/
    thread_pools_[pool_id].SetHostEnv(this);
  }
  
  thread_status_updater_ = CreateThreadStatusUpdater();
}
```

在指定优先级的线程池中调度一个线程执行(*function)(void* arg1)，如果注册了unschedFunction，则该函数在调用UnSchedule的时候会被调用。
```
void PosixEnv::Schedule(void (*function)(void* arg1), void* arg, Priority pri,
                        void* tag, void (*unschedFunction)(void* arg)) {
  assert(pri >= Priority::LOW && pri <= Priority::HIGH);
  thread_pools_[pri].Schedule(function, arg, tag, unschedFunction);
}
```

生成可调用对象，并将任务以可调用对象的形式添加到线程池的任务队列中：
```
void ThreadPoolImpl::Schedule(void(*function)(void* arg1), void* arg,
  void* tag, void(*unschedFunction)(void* arg)) {
  /* [arg, function] { function(arg); } 整体是一个lambda表达式，此lambda表达式返回
   * 一个匿名函数，作为可调用对象赋值给std::function对象fn，std::function<void()> fn
   * 表示可调用对象fn无参数且返回值为void，[arg, function]表示arg以传值的形式捕获，
   * function也以传值的形式捕获
   * 
   * 在调用fn对象的时候，就相当于调用function(arg)，但是function和arg都是在调用
   * ThreadPoolImpl::Schedule时确定了的，所以调用fn对象的时候，无需传入任何参数
   * std::function的作用是实现延时调用
   */
  std::function<void()> fn = [arg, function] { function(arg); };

  std::function<void()> unfn;
  if (unschedFunction != nullptr) {
    /*同样，生成一个可调用对象uf*/
    auto uf = [arg, unschedFunction] { unschedFunction(arg); };
    /* std::move是将对象的状态或者所有权从一个对象转移到另一个对象，只是转移，
     * 没有内存的搬迁或者内存拷贝
     */
    unfn = std::move(uf);
  }

  /*调用ThreadPoolImpl::Impl::Submit将该任务添加到线程池的任务队列中*/
  impl_->Submit(std::move(fn), std::move(unfn), tag);
}
```

以std::function的形式提交任务：
```
void ThreadPoolImpl::Impl::Submit(std::function<void()>&& schedule,
  std::function<void()>&& unschedule, void* tag) {

  std::lock_guard<std::mutex> lock(mu_);

  if (exit_all_threads_) {
    return;
  }

  /*如果有必要，则启动新的线程*/
  StartBGThreads();

  // Add to priority queue
  /*添加任务到任务队列*/
  queue_.push_back(BGItem());

  /*设置任务的schedule function和unschedule function*/
  auto& item = queue_.back();
  item.tag = tag;
  item.function = std::move(schedule);
  item.unschedFunction = std::move(unschedule);

  /*更新任务队列长度*/
  queue_len_.store(static_cast<unsigned int>(queue_.size()),
    std::memory_order_relaxed);

  if (!HasExcessiveThread()) {
    // Wake up at least one waiting thread.
    /* 如果线程池中线程数目没有超过当前设定的最大线程数目，则没有线程需要被销毁，
     * 随便唤醒哪个线程执行都可以
     */
    bgsignal_.notify_one();
  } else {
    // Need to wake up all threads to make sure the one woken
    // up is not the one to terminate.
    /* 如果线程池中线程数目超过了当前设定的最大线程数目，则需要销毁那些最晚创建
     * 的线程，所以不能直接调用notify_one()来唤醒线程了，这样可能那个被唤醒的
     * 线程就是那个要被立即消耗的线程，则任务就没有被执行，通过notify_all()可以
     * 确保那些尚未超出当前设定的最大线程数目的线程中第一个被唤醒的线程执行该任务
     */
    WakeUpAllThreads();
  }
}
```

如果当前线程池中线程数目少于设定的最大线程数目，则创建新的线程，并添加到线程池中，对于Linux来说，会调用std::thread的构造函数来创建线程：
```
void ThreadPoolImpl::Impl::StartBGThreads() {
  // Start background thread if necessary
  while ((int)bgthreads_.size() < total_threads_limit_) {

    /* 对于Linux来说，port::Thread对应的是std::thread, 这里调用std::thread的构造函数：
     * template< class Function, class... Args >
     * explicit thread( Function&& f, Args&&... args );
     *
     * 这里跟以往不一样了，以往都是调用pthread_create来创建新的线程，而这里通过std::thread
     * 的构造函数构造新的 std::thread 对象并将它与执行线程关联，最终等价于调用BGThreadWrapper
     * (new BGThreadMetadata(this, bgthreads_.size()))。
     */
    port::Thread p_t(&BGThreadWrapper,
      new BGThreadMetadata(this, bgthreads_.size()));

    // Set the thread name to aid debugging
    /*设置线程名称，方便调试使用*/
#if defined(_GNU_SOURCE) && defined(__GLIBC_PREREQ)
#if __GLIBC_PREREQ(2, 12)
    auto th_handle = p_t.native_handle();
    char name_buf[16];
    snprintf(name_buf, sizeof name_buf, "rocksdb:bg%" ROCKSDB_PRIszt,
             bgthreads_.size());
    name_buf[sizeof name_buf - 1] = '\0';
    pthread_setname_np(th_handle, name_buf);
#endif
#endif

    /*添加到线程数组bgthreads中*/
    bgthreads_.push_back(std::move(p_t));
  }
}
```

BGThreadMetadata封装了创建线程相关的参数（线程所在的线程池，线程在线程组中编号）：
```
struct BGThreadMetadata {
  /*所在的线程池*/
  ThreadPoolImpl::Impl* thread_pool_;
  /*线程编号*/
  size_t thread_id_;  // Thread count in the thread.
  
  /*构造函数*/
  BGThreadMetadata(ThreadPoolImpl::Impl* thread_pool, size_t thread_id)
      : thread_pool_(thread_pool), thread_id_(thread_id) {}
};
```

在ThreadPoolImpl::Impl::StartBGThreads中实际调用BGThreadWrapper来创建后台线程的：

```
void* ThreadPoolImpl::Impl::BGThreadWrapper(void* arg) {
  /*线程创建相关的参数*/
  BGThreadMetadata* meta = reinterpret_cast<BGThreadMetadata*>(arg);
  size_t thread_id = meta->thread_id_;
  ThreadPoolImpl::Impl* tp = meta->thread_pool_;
#ifdef ROCKSDB_USING_THREAD_STATUS
  // for thread-status
  ThreadStatusUtil::RegisterThread(
      tp->GetHostEnv(), (tp->GetThreadPriority() == Env::Priority::HIGH
                             ? ThreadStatus::HIGH_PRIORITY
                             : ThreadStatus::LOW_PRIORITY));
#endif
  delete meta;
  tp->BGThread(thread_id);
#ifdef ROCKSDB_USING_THREAD_STATUS
  ThreadStatusUtil::UnregisterThread();
#endif
  return nullptr;
}
```

ThreadPoolImpl::Impl::BGThread是线程池中线程的执行主体，每个线程可能处于以下状态：

- 等待。对于编号为thread_id的线程，如果同时满足下面两个条件：（1）编号为thread_id的线程无需终止（如果编号为thread_id的线程所在的线程池设置了exit_all_threads，或者，线程池中线程数目超过当前设定的最大线程数目且编号为thread_id的线程是线程池中创建时间最晚的那个线程，那么编号为thread_id线程就必须被销毁，在线程池中线程数目超过当前设定的最大线程数目的情况下，线程池按照创建事件的反序来销毁线程，即最先销毁那些最晚创建的线程）；（2）没有任务可做（任务队列为空），或者，虽然有任务要做但是线程池中线程数目超过当前设定的最大线程数目且编号thread_id是那些超出部分的线程中的一员（超出部分的线程应当被销毁，但是结合（1）和（2）可知编号为thread_id的线程当前还不能销毁，因为它不是这批线程中最晚创建的线程，只有当比它更晚创建的线程都销毁了之后，才能销毁它，任务队列中的任务在那些编号未超出当前设定的最大线程数目的线程中执行）；则编号为thread_id的线程将等待。如果不满足这两个条件中的任何一个，则该线程将被销毁，或者执行任务。

- 终止。处于等待状态的线程被唤醒之后，会检查是否继续等待，如果不满足继续等待的条件，且满足下面两个条件中的任意一个：（1）线程池设置了“终止所有线程”的标识；（2）线程池中线程数目超过了当前设定的最大线程数目且编号为thread_id的线程是这批线程中最后一个创建的；则终止之。

- 执行任务。如果线程不处于“等待”或者“终止”状态，则它一定有任务需要执行，从任务队列中取第一个任务由它执行。在执行任务之前，还可以根据需要，降低线程的优先级。
```
void ThreadPoolImpl::Impl::BGThread(size_t thread_id) {
  bool low_io_priority = false;
  while (true) {
    // Wait until there is an item that is ready to run
    std::unique_lock<std::mutex> lock(mu_);
    // Stop waiting if the thread needs to do work or needs to terminate.
    /* exit_all_threads_：表示终止所有线程
     * IsLastExcessiveThread(thread_id)：判断线程数目是否超过设定的最大线程数且编号
     * 为thread_id的线程是这些线程中最后一个加入该线程池的，如果返回true那么编号为
     * thread_id的线程就必须被终止
     * queue_.empty()：表示任务队列为空，没有需要执行的任务
     * IsExcessiveThread(thread_id)：表示线程编号thread_id已经超过了最大线程数，但
     * 编号为thread_id的线程还不能被终止，因为Rocksdb按照线程创建时间的顺序的反序
     * 来终止超出线程个数的线程
     *
     * 如果无需终止编号为thread_id的线程且线程池中任务队列为空，或者虽然有任务要执行
     * 但是编号为thread_id的线程是超过当前设定的最大线程数目的那一部分线程（应当按
     * 创建时间的反序销毁）且不是当前编号最大的线程（则需要等待更大编号的线程被销毁），
     * 则等待，直到编号为thread_id的线程有任务要执行或者需要被终止，才退出等待
     */
    while (!exit_all_threads_ && !IsLastExcessiveThread(thread_id) &&
           (queue_.empty() || IsExcessiveThread(thread_id))) {
      bgsignal_.wait(lock);
    }

    /*如果需要终止所有线程*/
    if (exit_all_threads_) {  // mechanism to let BG threads exit safely
      /*如果无需等待任务结束，或者当前任务队列为空，则终止编号为thread_id的线程*/
      if(!wait_for_jobs_to_complete_ ||
          queue_.empty()) {
        break;
       }
    }

    /* 如果线程池中线程数目超过了预定的最大线程数目且编号为thread_id的线程是
     * 这批线程中最后一个创建的，则终止之
     */
    if (IsLastExcessiveThread(thread_id)) {
      // Current thread is the last generated one and is excessive.
      // We always terminate excessive thread in the reverse order of
      // generation time.
      /*终止线程池中最后一个创建的线程，它的线程编号一定是thread_id*/
      auto& terminating_thread = bgthreads_.back();
      terminating_thread.detach();
      bgthreads_.pop_back();

      /* 线程数目仍然超过预定的最大线程数目，则唤醒所有线程，以让它们自己检查是
       * 否是线程池中剩余线程中最后创建的线程，如果是就需要终止自己
       */
      if (HasExcessiveThread()) {
        // There is still at least more excessive thread to terminate.
        WakeUpAllThreads();
      }
      break;
    }

    /* 至此，编号为thread_id的线程无需终止，那一定是因为任务队列不为空，
     * 从任务队列中取出第一个任务
     */
    auto func = std::move(queue_.front().function);
    queue_.pop_front();

    /*更新任务队列长度（即任务队列中剩余任务个数）*/
    queue_len_.store(static_cast<unsigned int>(queue_.size()),
                     std::memory_order_relaxed);

    /* 如果通过ThreadPoolImpl::Impl::LowerIOPriority()设置了降低线程池中线程
     * 的优先级，但是未曾降低过该线程的优先级，则降低之
     */
    bool decrease_io_priority = (low_io_priority != low_io_priority_);
    lock.unlock();

#ifdef OS_LINUX
    if (decrease_io_priority) {
#define IOPRIO_CLASS_SHIFT (13)
#define IOPRIO_PRIO_VALUE(class, data) (((class) << IOPRIO_CLASS_SHIFT) | data)
      // Put schedule into IOPRIO_CLASS_IDLE class (lowest)
      // These system calls only have an effect when used in conjunction
      // with an I/O scheduler that supports I/O priorities. As at
      // kernel 2.6.17 the only such scheduler is the Completely
      // Fair Queuing (CFQ) I/O scheduler.
      // To change scheduler:
      //  echo cfq > /sys/block/<device_name>/queue/schedule
      // Tunables to consider:
      //  /sys/block/<device_name>/queue/slice_idle
      //  /sys/block/<device_name>/queue/slice_sync
      /* 设置线程的scheduling class和priority，IOPRIO_PRIO_VALUE(3, 0)表示将线程
       * 设置到IOPRIO_CLASS_IDLE级别（最低级别）
       */
      syscall(SYS_ioprio_set, 1,  // IOPRIO_WHO_PROCESS
              0,                  // current thread
              IOPRIO_PRIO_VALUE(3, 0));
      /*标记“已经降低了当前线程的优先级”*/
      low_io_priority = true;
    }
#else
    (void)decrease_io_priority;  // avoid 'unused variable' error
#endif

    /*调用func()，执行任务（func()中封装了任务）*/
    func();
  }
}
```

### 撤销任务
如果指定arg相关的某个任务尚未调度，则从指定优先级的任务队列中删除之，从调用链来看，arg对应于Schedule的时候的tag参数。
```
int PosixEnv::UnSchedule(void* arg, Priority pri) {
  return thread_pools_[pri].UnSchedule(arg);
}

int ThreadPoolImpl::UnSchedule(void* arg) {
  return impl_->UnSchedule(arg);
}

int ThreadPoolImpl::Impl::UnSchedule(void* arg) {
  int count = 0;

  std::vector<std::function<void()>> candidates;
  {
    /*计划任务或者撤销任务之前都必须加锁*/
    std::lock_guard<std::mutex> lock(mu_);

    // Remove from priority queue
    /*从任务队列中删除具有指定tag的任务*/
    BGQueue::iterator it = queue_.begin();
    while (it != queue_.end()) {
      if (arg == (*it).tag) {
        if (it->unschedFunction) {
          candidates.push_back(std::move(it->unschedFunction));
        }
        it = queue_.erase(it);
        count++;
      } else {
        ++it;
      }
    }
    /*更新任务队列长度*/
    queue_len_.store(static_cast<unsigned int>(queue_.size()),
      std::memory_order_relaxed);
  }


  // Run unschedule functions outside the mutex
  /*运行撤销任务函数*/
  for (auto& f : candidates) {
    f();
  }

  return count;
}
```

### 使用示例
在db/db_impl_compaction_flush.cc中，在合适的时机会尝试计划执行高优先级的Flush任务和低优先级的Compaction任务：
```
void DBImpl::MaybeScheduleFlushOrCompaction() {
  ......
  
  while (unscheduled_flushes_ > 0 &&
         bg_flush_scheduled_ < immutable_db_options_.max_background_flushes) {
    unscheduled_flushes_--;
    bg_flush_scheduled_++;
    /* 计划执行高优先级的Flush任务（从ImmutableMemTable到SST File，因为如果flush
     * 不及时运行，则写入操作会受影响，所以优先级比较高）
     */
    env_->Schedule(&DBImpl::BGWorkFlush, this, Env::Priority::HIGH, this);
  }

  ......

  while (bg_compaction_scheduled_ < bg_compactions_allowed &&
         unscheduled_compactions_ > 0) {
    CompactionArg* ca = new CompactionArg;
    ca->db = this;
    ca->m = nullptr;
    bg_compaction_scheduled_++;
    unscheduled_compactions_--;
    /* 计划执行低优先级的Compaction任务（合并SST File，该任务优先级相比较Flush任务
     * 而言要低）
     */
    env_->Schedule(&DBImpl::BGWorkCompaction, ca, Env::Priority::LOW, this,
                   &DBImpl::UnscheduleCallback);
  }
  
  ......
}
```

在数据库实例被销毁的时候，撤销低优先级的Compaction任务和高优先级的Flush任务：
```
DBImpl::~DBImpl() {
  ......
  
  /*撤销低优先级的Compaction任务，撤销高优先级的Flush任务*/
  int compactions_unscheduled = env_->UnSchedule(this, Env::Priority::LOW);
  int flushes_unscheduled = env_->UnSchedule(this, Env::Priority::HIGH);
  
  ......
}
```

## 为WAL写/Manifest文件写/SST File写/SST File读等优化相关的EnvOptions选项更新
### 为WAL写优化EnvOptions选项
```
EnvOptions OptimizeForLogWrite(const EnvOptions& env_options,
                                 const DBOptions& db_options) const override {
    EnvOptions optimized = env_options;
    optimized.use_mmap_writes = false;
    optimized.use_direct_writes = false;
    optimized.bytes_per_sync = db_options.wal_bytes_per_sync;
    // TODO(icanadi) it's faster if fallocate_with_keep_size is false, but it
    // breaks TransactionLogIteratorStallAtLastRecord unit test. Fix the unit
    // test and make this false
    optimized.fallocate_with_keep_size = true;
    return optimized;
  }
```

### 为Manifest文件写优化EnvOptions选项
```
EnvOptions OptimizeForManifestWrite(
      const EnvOptions& env_options) const override {
    EnvOptions optimized = env_options;
    optimized.use_mmap_writes = false;
    optimized.use_direct_writes = false;
    optimized.fallocate_with_keep_size = true;
    return optimized;
  }
```
