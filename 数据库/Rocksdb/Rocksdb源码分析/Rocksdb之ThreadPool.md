# 提纲
[toc]

# 简介
线程池会启动数个线程来执行计划的任务，Rocksdb定义了线程池的接口类ThreadPool，其中定义了每个线程池的实现类都必须实现的方法。

# 线程池接口类ThreadPool
线程池中线程的个数可以通过SetBackgroundThreads()来动态调整。
```
class ThreadPool {
 public:
  virtual ~ThreadPool() {}

  // Wait for all threads to finish.
  // Discard those threads that did not start
  // executing
  /*等待线程池中所有线程退出*/
  virtual void JoinAllThreads() = 0;

  // Set the number of background threads that will be executing the
  // scheduled jobs.
  /*设置线程池中线程的数目*/
  virtual void SetBackgroundThreads(int num) = 0;

  // Get the number of jobs scheduled in the ThreadPool queue.
  /*返回线程池中任务队列中任务个数*/
  virtual unsigned int GetQueueLen() const = 0;

  // Waits for all jobs to complete those
  // that already started running and those that did not
  // start yet. This ensures that everything that was thrown
  // on the TP runs even though
  // we may not have specified enough threads for the amount
  // of jobs
  /*等待所有已经执行的或者还在任务队列中等待的任务完成，然后终止所有线程*/
  virtual void WaitForJobsAndJoinAllThreads() = 0;

  // Submit a fire and forget jobs
  // This allows to submit the same job multiple times
  /*提交任务*/
  virtual void SubmitJob(const std::function<void()>&) = 0;
  // This moves the function in for efficiency
  virtual void SubmitJob(std::function<void()>&&) = 0;
};
```

# 线程池实现类ThreadPoolImpl
类ThreadPoolImpl实现ThreadPool接口，并提供更多有用的函数：
```
class ThreadPoolImpl : public ThreadPool {
 public:
  ThreadPoolImpl();
  ~ThreadPoolImpl();

  ThreadPoolImpl(ThreadPoolImpl&&) = delete;
  ThreadPoolImpl& operator=(ThreadPoolImpl&&) = delete;

  // Implement ThreadPool interfaces

  // Wait for all threads to finish.
  // Discards all the the jobs that did not
  // start executing and waits for those running
  // to complete
  void JoinAllThreads() override;

  // Set the number of background threads that will be executing the
  // scheduled jobs.
  void SetBackgroundThreads(int num) override;
  // Get the number of jobs scheduled in the ThreadPool queue.
  unsigned int GetQueueLen() const override;

  // Waits for all jobs to complete those
  // that already started running and those that did not
  // start yet
  void WaitForJobsAndJoinAllThreads() override;

  // Make threads to run at a lower kernel priority
  // Currently only has effect on Linux
  /*降低线程池中所有线程的优先级*/
  void LowerIOPriority();

  // Ensure there is at aleast num threads in the pool
  // but do not kill threads if there are more
  /*确保线程池中至少有num个线程*/
  void IncBackgroundThreadsIfNeeded(int num);

  // Submit a fire and forget job
  // These jobs can not be unscheduled

  // This allows to submit the same job multiple times
  void SubmitJob(const std::function<void()>&) override;
  // This moves the function in for efficiency
  void SubmitJob(std::function<void()>&&) override;

  // Schedule a job with an unschedule tag and unschedule function
  // Can be used to filter and unschedule jobs by a tag
  // that are still in the queue and did not start running
  /*计划一个任务*/
  void Schedule(void (*function)(void* arg1), void* arg, void* tag,
                void (*unschedFunction)(void* arg));

  // Filter jobs that are still in a queue and match
  // the given tag. Remove them from a queue if any
  // and for each such job execute an unschedule function
  // if such was given at scheduling time.
  /*撤销一个计划任务*/
  int UnSchedule(void* tag);

  /*设置/获取Env信息*/
  void SetHostEnv(Env* env);
  Env* GetHostEnv() const;

  // Return the thread priority.
  // This would allow its member-thread to know its priority.
  /*返回线程的优先级*/
  Env::Priority GetThreadPriority() const;

  // Set the thread priority.
  /*设置线程优先级*/
  void SetThreadPriority(Env::Priority priority);

  /*执行某个函数，该函数返回值为result，如果函数执行失败，则打印信息，并且终止*/
  static void PthreadCall(const char* label, int result);

  struct Impl;

 private:
   /*真正的线程池相关的接口都在impl_中实现，ThreadPoolImpl作为一个wrapper*/
   std::unique_ptr<Impl>   impl_;
};
```

## ThreadPoolImpl相关方法
ThreadPoolImpl中的方法都是借助于ThreadPoolImpl::Impl类来实现：

终止所有线程，无需等待任务完成
```
void ThreadPoolImpl::JoinAllThreads() {
  impl_->JoinThreads(false);
}
```

设置线程池中线程数目，允许减少线程数目
```
void ThreadPoolImpl::SetBackgroundThreads(int num) {
  impl_->SetBackgroundThreadsInternal(num, true);
}
```

返回任务队列中任务个数
```
unsigned int ThreadPoolImpl::GetQueueLen() const {
  return impl_->GetQueueLen();
}
```

等待任务完成，然后终止所有线程
```
void ThreadPoolImpl::WaitForJobsAndJoinAllThreads() {
  impl_->JoinThreads(true);
}
```

降低线程池中线程的优先级
```
void ThreadPoolImpl::LowerIOPriority() {
  impl_->LowerIOPriority();
}
```

增加线程池中线程池的数目
```
void ThreadPoolImpl::IncBackgroundThreadsIfNeeded(int num) {
  impl_->SetBackgroundThreadsInternal(num, false);
}
```

提交任务，直接提交的是可调用的std::function对象
```
void ThreadPoolImpl::SubmitJob(const std::function<void()>& job) {
  auto copy(job);
  impl_->Submit(std::move(copy), std::function<void()>(), nullptr);
}
```

提交任务，直接提交的是可调用的std::function对象
```
void ThreadPoolImpl::SubmitJob(std::function<void()>&& job) {
  impl_->Submit(std::move(job), std::function<void()>(), nullptr);
}
```

计划任务，提交的是任务执行函数和撤销任务函数（如果有）
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

撤销任务
```
int ThreadPoolImpl::UnSchedule(void* arg) {
  return impl_->UnSchedule(arg);
}
```

设置Env
```
void ThreadPoolImpl::SetHostEnv(Env* env) { impl_->SetHostEnv(env); }
```

获取Env
```
Env* ThreadPoolImpl::GetHostEnv() const { return impl_->GetHostEnv(); }
```

获取线程的优先级
```
Env::Priority ThreadPoolImpl::GetThreadPriority() const {
  return impl_->GetThreadPriority();
}
```

设置线程的优先级
```
void ThreadPoolImpl::SetThreadPriority(Env::Priority priority) {
  impl_->SetThreadPriority(priority);
}
```

## ThreadPoolImpl::Impl类定义
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

  /*保护任务队列和条件变量*/
  std::mutex               mu_;
  std::condition_variable  bgsignal_;
  /*线程数组*/
  std::vector<port::Thread> bgthreads_;
};
```

## ThreadPoolImpl::Impl类方法
ThreadPoolImpl::Impl构造函数：
```
inline
ThreadPoolImpl::Impl::Impl() : 
      low_io_priority_(false),
      priority_(Env::LOW), //默认优先级别是LOW
      env_(nullptr),
      total_threads_limit_(1), //初始的最大线程数目为1
      queue_len_(),
      exit_all_threads_(false),
      wait_for_jobs_to_complete_(false),
      queue_(),
      mu_(),
      bgsignal_(),
      bgthreads_() {
}
```

等待所有线程终止，参数wait_for_jobs_to_complete指示是否等待任务完成：
```
void ThreadPoolImpl::Impl::JoinThreads(bool wait_for_jobs_to_complete) {

  std::unique_lock<std::mutex> lock(mu_);
  assert(!exit_all_threads_);

  /*设置是否需要等待任务完成*/
  wait_for_jobs_to_complete_ = wait_for_jobs_to_complete;
  /*设置终止所有线程*/
  exit_all_threads_ = true;

  lock.unlock();

  /*唤醒所有可能处于等待状态的线程，让它们各自终止（见ThreadPoolImpl::Impl::BGThread）*/
  bgsignal_.notify_all();

  /*等待所有线程*/
  for (auto& th : bgthreads_) {
    th.join();
  }

  /*清除所有线程*/
  bgthreads_.clear();

  /*复位exit_all_threads和wait_for_jobs_to_complete*/
  exit_all_threads_ = false;
  wait_for_jobs_to_complete_ = false;
}
```

设置需要降低线程池中所有线程的优先级，每个线程在运行过程中，会检测low_io_priority_是否被设置为true，且该线程尚未执行过降低优先级的操作，则降低该线程的优先级（见ThreadPoolImpl::Impl::BGThread）：
```
inline
void ThreadPoolImpl::Impl::LowerIOPriority() {
  std::lock_guard<std::mutex> lock(mu_);
  low_io_priority_ = true;
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

通过BGThreadMetadata封装创建线程相关的参数：
```
struct BGThreadMetadata {
  ThreadPoolImpl::Impl* thread_pool_;
  size_t thread_id_;  // Thread count in the thread.
  BGThreadMetadata(ThreadPoolImpl::Impl* thread_pool, size_t thread_id)
      : thread_pool_(thread_pool), thread_id_(thread_id) {}
};
```

ThreadPoolImpl::Impl::BGThreadWrapper封装线程执行主体，最终调用ThreadPoolImpl::Impl::BGThread：
```
void* ThreadPoolImpl::Impl::BGThreadWrapper(void* arg) {
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

设置线程池中线程数目，可以增加也可以减少，如果减少线程数目，则需要检查allow_reduce是否被设置：
```
void ThreadPoolImpl::Impl::SetBackgroundThreadsInternal(int num,
  bool allow_reduce) {
  std::unique_lock<std::mutex> lock(mu_);
  /*如果设置了终止所有线程，则直接返回*/
  if (exit_all_threads_) {
    lock.unlock();
    return;
  }
  
  /*可以直接增加线程数目，但是减少线程数目需要判断allow_reduce是否为true*/
  if (num > total_threads_limit_ ||
      (num < total_threads_limit_ && allow_reduce)) {
      
    /*确保线程池中至少有1个线程*/
    total_threads_limit_ = std::max(1, num);
    
    /*唤醒所有线程（如果是减少线程数目，就需要销毁那些最晚创建的线程）*/
    WakeUpAllThreads();
    /*根据需要启动线程（如果是增加线程数目，则需要创建新的线程）*/
    StartBGThreads();
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
     * 这里跟以往不一样了，以往都是调用pthread_create来创建新的线程，而这里通过
     * std::thread 的构造函数构造新的 std::thread 对象并将它与执行线程关联，最终
     * 等价于调用BGThreadWrapper(new BGThreadMetadata(this, bgthreads_.size()))。
     *
     * 在BGThreadWrapper中会调用ThreadPoolImpl::Impl::BGThread，在其中运行线程主体
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

提交任务，如果有必要会启动新的线程（线程池中当前线程数目少于设定的最大线程数目），然后将任务添加到任务队列中，并唤醒线程来执行该任务，如果线程池中线程数目没有超过当前设定的最大线程数目，则通过notify_one()随便唤醒哪个线程执行都可以，如果线程池中线程数目超过了当前设定的最大线程数目，则需要通过notify_all()唤醒全部线程：
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

撤销任务，从任务队列中删除arg对应的任务，并更新任务队列长度，最后如果被撤销的任务注册了撤销函数，则运行该撤销函数：
```
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

## 线程创建时机
- 通过SetBackgroundThreadsInternal设置最大线程数目，且当前线程池中的线程数目少于设定的最大线程数目，则创建新的线程，直到线程数目等于设定的最大线程数目：

```
void ThreadPoolImpl::Impl::SetBackgroundThreadsInternal(int num,
  bool allow_reduce) {
  std::unique_lock<std::mutex> lock(mu_);
  if (exit_all_threads_) {
    lock.unlock();
    return;
  }
  if (num > total_threads_limit_ ||
      (num < total_threads_limit_ && allow_reduce)) {
    total_threads_limit_ = std::max(1, num);
    WakeUpAllThreads();
    
    /*如果当前线程池中的线程数目少于设定的最大线程数目则创建新的线程*/
    StartBGThreads();
  }
}
```

- 提交任务的时候，会检查当前线程池中的线程数目是否少于设定的最大线程数目，如果少于最大线程数目，则创建新的线程：

```
void ThreadPoolImpl::Impl::Submit(std::function<void()>&& schedule,
  std::function<void()>&& unschedule, void* tag) {

  std::lock_guard<std::mutex> lock(mu_);

  if (exit_all_threads_) {
    return;
  }

  /*如果当前线程池中的线程数目少于设定的最大线程数目则创建新的线程*/
  StartBGThreads();

  // Add to priority queue
  queue_.push_back(BGItem());

  auto& item = queue_.back();
  item.tag = tag;
  item.function = std::move(schedule);
  item.unschedFunction = std::move(unschedule);

  queue_len_.store(static_cast<unsigned int>(queue_.size()),
    std::memory_order_relaxed);

  if (!HasExcessiveThread()) {
    // Wake up at least one waiting thread.
    bgsignal_.notify_one();
  } else {
    // Need to wake up all threads to make sure the one woken
    // up is not the one to terminate.
    WakeUpAllThreads();
  }
}
```

## 线程唤醒时机
- 提交任务的时候，如果线程池中线程数目没有超过当前设定的最大线程数目，则通过notify_one()随便唤醒哪个线程执行都可以，如果线程池中线程数目超过了当前设定的最大线程数目，则通过notify_all()唤醒全部线程：

```
void ThreadPoolImpl::Impl::Submit(std::function<void()>&& schedule,
  std::function<void()>&& unschedule, void* tag) {

  std::lock_guard<std::mutex> lock(mu_);

  if (exit_all_threads_) {
    return;
  }

  StartBGThreads();

  // Add to priority queue
  queue_.push_back(BGItem());

  auto& item = queue_.back();
  item.tag = tag;
  item.function = std::move(schedule);
  item.unschedFunction = std::move(unschedule);

  queue_len_.store(static_cast<unsigned int>(queue_.size()),
    std::memory_order_relaxed);

  if (!HasExcessiveThread()) {
    // Wake up at least one waiting thread.
    /* 如果线程池中线程数目没有超过当前设定的最大线程数目，则通过notify_one()
     * 随便唤醒哪个线程执行都可以
     */
    bgsignal_.notify_one();
  } else {
    // Need to wake up all threads to make sure the one woken
    // up is not the one to terminate.
    /* 如果线程池中线程数目超过了当前设定的最大线程数目，则通过notify_all()
     * 唤醒全部线程
     */
    WakeUpAllThreads();
  }
}
```

- 在需要终止所有线程的时候，会唤醒所有线程：

```
void ThreadPoolImpl::Impl::JoinThreads(bool wait_for_jobs_to_complete) {
  std::unique_lock<std::mutex> lock(mu_);
  assert(!exit_all_threads_);

  wait_for_jobs_to_complete_ = wait_for_jobs_to_complete;
  exit_all_threads_ = true;

  lock.unlock();

  /*唤醒所有线程，线程会检测到exit_all_threads_被设置，所以会终止*/
  bgsignal_.notify_all();

  for (auto& th : bgthreads_) {
    th.join();
  }

  bgthreads_.clear();

  exit_all_threads_ = false;
  wait_for_jobs_to_complete_ = false;
}
```

- 如果当前线程池中线程数目超过了设定的最大线程数，且当前线程是当前线程池中编号最大的线程，则在当前线程终止之前唤醒其它所有线程，以便编号比当前线程小1的线程能够接收到销毁信号（因为线程池中是按照线程创建顺序的反序来销毁线程的，超出最大线程数目的线程中后一个线程被销毁之后需要主动唤醒它前面的那个线程）：

```
void ThreadPoolImpl::Impl::BGThread(size_t thread_id) {
  bool low_io_priority = false;
  while (true) {
    // Wait until there is an item that is ready to run
    std::unique_lock<std::mutex> lock(mu_);
    // Stop waiting if the thread needs to do work or needs to terminate.
    while (!exit_all_threads_ && !IsLastExcessiveThread(thread_id) &&
           (queue_.empty() || IsExcessiveThread(thread_id))) {
      bgsignal_.wait(lock);
    }

    if (exit_all_threads_) {  // mechanism to let BG threads exit safely
      if(!wait_for_jobs_to_complete_ ||
          queue_.empty()) {
        break;
       }
    }

    if (IsLastExcessiveThread(thread_id)) {
      // Current thread is the last generated one and is excessive.
      // We always terminate excessive thread in the reverse order of
      // generation time.
      /* 当前线程是超出设定最大线程数部分的编号最大的线程，即将消耗，在销毁
       * 之前有义务唤醒它之前的那个也需要被销毁的线程
       */
      auto& terminating_thread = bgthreads_.back();
      terminating_thread.detach();
      bgthreads_.pop_back();

      if (HasExcessiveThread()) {
        // There is still at least more excessive thread to terminate.
        /* 线程池中线程数目仍然超过设定的最大线程数目，当前线程有义务唤醒它
         * 之前的那个需要被销毁的线程，然后当前线程才终止
         */
        WakeUpAllThreads();
      }
      
      /*退出while循环，终止*/
      break;
    }

    ......
  }
}
```

- 如果通过SetBackgroundThreadsInternal设置了减少线程池中线程数目，则需要唤醒所有线程，以便那些超过当前设定的最大线程数目部分的线程终止：

```
void ThreadPoolImpl::Impl::SetBackgroundThreadsInternal(int num,
  bool allow_reduce) {
  std::unique_lock<std::mutex> lock(mu_);
  if (exit_all_threads_) {
    lock.unlock();
    return;
  }
  if (num > total_threads_limit_ ||
      (num < total_threads_limit_ && allow_reduce)) {
    total_threads_limit_ = std::max(1, num);
    /*唤醒所有线程（以便在减少线程数目情况下销毁那些最晚创建的线程）*/
    WakeUpAllThreads();
    StartBGThreads();
  }
}
```

## 线程销毁时机
- 在ThreadPoolImpl::Impl::BGThread中如果检测到外部调用了ThreadPoolImpl::Impl::JoinThreads，则销毁所有线程：

```
void ThreadPoolImpl::Impl::BGThread(size_t thread_id) {
  bool low_io_priority = false;
  while (true) {
    ......

    /*销毁所有线程（所有线程在检测到exit_all_threads_被设置为true时，直接break，退出while循环）*/
    if (exit_all_threads_) {  // mechanism to let BG threads exit safely
      if(!wait_for_jobs_to_complete_ ||
          queue_.empty()) {
        break;
       }
    }

    ......
  }
}
```

- 在ThreadPoolImpl::Impl::BGThread中如果检测到当前线程池中线程数目超过当前设定的最大线程数目，且当前线程是线程池中最晚创建的线程，则销毁当前线程：
```
void ThreadPoolImpl::Impl::BGThread(size_t thread_id) {
  bool low_io_priority = false;
  while (true) {
    ......

    /* 当前线程池中线程数目超过当前设定的最大线程数目，且当前编号为thread_id的
     * 线程是线程池中最晚创建的线程，则销毁当前线程
     */
    if (IsLastExcessiveThread(thread_id)) {
      // Current thread is the last generated one and is excessive.
      // We always terminate excessive thread in the reverse order of
      // generation time.
      auto& terminating_thread = bgthreads_.back();
      terminating_thread.detach();
      bgthreads_.pop_back();

      if (HasExcessiveThread()) {
        // There is still at least more excessive thread to terminate.
        WakeUpAllThreads();
      }
      break;
    }
    ......
  }
}