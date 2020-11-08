# 提纲
[toc]

## io_service
boost::asio提供了一个跨平台的异步编程IO模型库，io_service类在多线程编程模型中提供了任务队列和任务分发功能。

它的总体使用流程总结如下，参考[这里](https://blog.csdn.net/guotianqing/article/details/100730340)：


```
#include <boost/asio/io_service.hpp>
#include <boost/bind.hpp>
#include <boost/thread/thread.hpp>

/*
 * Create an asio::io_service and a thread_group (through pool in essence)
 */
boost::asio::io_service ioService;
boost::thread_group threadpool;


/*
 * This will start the ioService processing loop. All tasks 
 * assigned with ioService.post() will start executing. 
 */
boost::asio::io_service::work work(ioService);

/*
 * This will add 2 threads to the thread pool. (You could just put it in a for loop)
 */
threadpool.create_thread(
    boost::bind(&boost::asio::io_service::run, &ioService)
);
threadpool.create_thread(
    boost::bind(&boost::asio::io_service::run, &ioService)
);

/*
 * This will assign tasks to the thread pool. 
 * More about boost::bind: "http://www.boost.org/doc/libs/1_54_0/libs/bind/bind.html#with_functions"
 * You can use strand when necessary, if so, remember add "strand.h"
 */
ioService.post(boost::bind(myTask, "Hello World!"));
ioService.post(boost::bind(clearCache, "./cache"));
ioService.post(boost::bind(getSocialUpdates, "twitter,gmail,facebook,tumblr,reddit"));

/*
 * This will stop the ioService processing loop. Any tasks
 * you add behind this point will not execute.
*/
ioService.stop();

/*
 * Will wait till all the threads in the thread pool are finished with 
 * their assigned tasks and 'join' them. Just assume the threads inside
 * the threadpool will be destroyed by this method.
 */
threadpool.join_all();
```

