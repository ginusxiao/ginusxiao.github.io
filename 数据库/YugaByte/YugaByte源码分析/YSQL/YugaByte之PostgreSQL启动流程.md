# 提纲
[toc]

## PostgreSQL启动流程
```
TServer main
    - tserver::TabletServerMain(如果设置了启动ysql的话，则执行如下过程来启动PostgreSQL)
        - PgProcessConf::CreateValidateAndRunInitDb
            - 生成PgProcessConf
            - 根据PgProcessConf生成PgWrapper
            # 检查bin/postgres和bin/initdb都是合法的可执行程序
            - PgWrapper::PreflightCheck()
            # 初始化相关的目录
            - PgWrapper::InitDbLocalOnlyIfNeeded()
        # PgSupervisor用于监视PostgreSQL，并在它crash的时候重启之
        - std::unique_ptr<PgSupervisor> pg_supervisor = std::make_unique<PgSupervisor>(pg_process_conf)
        - pg_supervisor->Start()
             - PgSupervisor::CleanupOldServerUnlocked
             - PgSupervisor::StartServerUnlocked
                - PgWrapper::Start
                    - 首先准备启动所需要的参数，存放在@argv中
                    - PgWrapper中封装了一个boost Subprocess(对子进程的封装) @pg_proc_
                    - 对@pg_proc_进行配置，包括设置启动PostgreSQL所需要的参数
                    # 启动PostgreSQL进程
                    - pg_proc_->Start()
                        - 最终会到达src/backend/main/main.c：main()
                            - PostgresServerProcessMain
                                # 启动postmaster
                                - PostmasterMain
                                    # postmaster主循环
                                    - ServerLoop()
                                        - 无线循环，执行select，检查是否有请求到达，对于每一个接收到的请求，执行如下：
                                            - BackendStartup
                                                # fork一个子进程
                                                - fork_process()
                                                - 在子进程中执行以下步骤：
                                                    - BackendInitialize
                                                    - BackendRun
                                                        # postgres进程的运行主体
                                                        - PostgresMain
```
postmaster主循环是一个无限循环，它等待客户端的连接请求并为之提供服务。在无限循环里，postmaster进程通过调用select接口检查是否有客户端连接请求，如果没有，继续循环，如果有，就创建一个postgres子进程，并且将这个连接交给新创建的postgres子进程进行处理，然后继续循环。

在PostgresMain中又有一个无限循环，它不断的从客户端(也就是postgres中所谓的FrontEnd)读取命令，并处理命令。关于PostgreSQL的命令格式，请参考[这里](https://blog.csdn.net/weixin_34302798/article/details/88687718)。
