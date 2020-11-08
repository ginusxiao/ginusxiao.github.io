# 提纲
[toc]

## Cassandra Java Driver简介
关于Cassandra Java Driver，可以参考：

[Cassandra Java驱动程序](https://zhuanlan.zhihu.com/p/84791851)

## Cassandra Java Driver Connection pooling
[Cassandra Java Driver Connection pooling](https://docs.datastax.com/en/developer/java-driver/3.1/manual/pooling/)

其中，关于Session和Connection的关系可以通过下图进行说明：
![Session和Connection的关系](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAu4AAABiCAIAAACAksEXAAAPLklEQVR42u2d%0Ae2wV1RaH6YO+W6TlFCIepVrQYE0FbBqKgi0YI7EGQWKsgYKJGv8wxERNNIoG%0Ao0QCiYJRQWOReIM8gmAKiKAkFEoVqbQW4ZbyKFDSllrp00Ntr3eFnTuZe07P%0A7vR95sz3/dGcPXt39uy11l77NzO7PSP+AQAAALAtIzABAAAAIGUAAAAAhlXK%0A/AcAAADAJiBlAAAAACkDAAAAgJQBAAAAQMoAAAAAUgYAAAAAKQMAAACAlAEA%0AAABAygAAAABSBgAAAAApAwAAAICUAQAAAKQMUgYAAACQMgAAAABIGQAAAACk%0ADAAAACBlAAAAAJAyAAAAAEgZAAAAAKQMAAAAIGUAAAAAkDIAAAAASBkAAABA%0AyiBlAAAAACkDAAAAgJQBAAAAQMoMKxUVFW+99dbdd9894gYYBB9hScD7Q9Np%0AaWnpiy++OHHixIiIiLFjx2ZnZxcWFhq1f/3118qVK9PS0hISEuLj4+WDFOWg%0A+QyLFi2aNGmSfCgpKcnPz1enkvYZGRmrVq3q6Oiw0pHvtR0/fjw5OVkOSo9E%0ADlLGBkyePFlmmsw3kjs+wpKA94ey0/T09DVr1pw4caKlpaWysvKhhx6SNp98%0A8omqXbp0qRQXL15cW1tbX1+/ZMkSKcpB49c7OztdLtdLL70kn0W7FBQUnD59%0Aur29/dSpU4888og0fvbZZ6105HVtBw8eFDEUGhq6fv16wgYpYzNI7vgISwLe%0AH8ZOq6urpc2ECRNUMS4uTooiYlRRPkhRDhrti4uL5cj+/ft9T1VTUyNVsbGx%0AVjoyX9vOnTujoqIiIiK+/vprQgUpw/S21N22bdtmzpw5evToxMTE3Nzcs2fP%0A4gh8hCXBmVKmrq5O2oiMUEWJE18pk5SUZLR/44034uPjPR6PPykzZswYKx0Z%0A11ZQUBAeHh4TE7N3717iBCnD9LbaXXp6emlpaVNT04oVK6Q4a9YsHIGPsCQ4%0AU8q8/PLL0iYtLU0VX3nlFSkuWbKk/gbqBdNrr71mtJ8yZcr8+fO9TtLR0XH6%0A9OnHHntMGr/++utWOlLXtnr16pCQkJtuuunw4cMECVKG6d2L7o4ePaqKzc3N%0AUoyMjMQR+AhLggOlzJo1a1SbzZs3qyPt7e1ZWVkjTMyYMcPY9ltTUyPK4/PP%0AP/ftRTF58uSGhgYrHZl/y9hAAzaQMnfdddcIJyHjDczkbjwa7erq8roAfISP%0AsCQEgfetdLpq1SqpFWny4YcfGgdfeOEF9VSmrq7OeCojB1Xthg0bpL0IGq9T%0AXb9+vby8/OGHH5bGCxcutNKRura1a9fKz5iYmG4337CGBqKUkSv7x0nIeOVG%0AsLW1VRS9BHpnZ2eAJHfNEXyEj7AkBIH3e+x05cqVSl589tln5uOjRo3y3Ssj%0AB1Vx3rx506ZN89fdpUuXvPYIazoyrk2pmaioqN27d7OG9jaukDJD4QaJbJH2%0AjY2N4gzxBMkdHwWxlMGSTpYygeZ9fafvvPOOHA8NDf3iiy+8qhISEvxJGY/H%0AIzJl+fLl/rq7fPmy17ZfTUfma1u3bp3aEbxjxw7W0F7FFVJmKNxw8uTJs2fP%0A1tTUiCfa29tJ7vgoiKUMlnSylAk072s6ffvtt+VgWFjYpk2bfH/lueeeM2/7%0AVf9m5vnnn5eqffv2yeeSkhKjcWZmppzkzJkzMuSKigr1gknOb6Ujr2v76KOP%0AQkJCwsPDjc00rKFW4gopMxRuKC4uLisrE0/U1ta2tLT42yk2ZJOc5I6PsCRS%0Axjne77ZTfzszVK3H43n//ffvueee+BukpaVJUe24WrZsmcvlMr/mKCoqWrBg%0AgdvtFgmSlJSUk5Pz1VdfWezI1xRKzYj02bhxI2toj3GFlBk6N+zdu1c8IWr9%0A4sWL165dC/z93vgIH2FJcIL3+0BqaurixYuJWBvFFVJmYNywZcuW77///tix%0AY1VVVX/88QfJHR8FsZTBkk6WMrbzvh0hPyNlhscNmzdv/u67737++eczZ86Q%0A3PFRcEsZLOlkKWM77yNlnBBXwyxlRILl5eW53e6IiIi4uLiUlJTc3FxjbAPi%0AzoE6j9OS+65du2bOnOlyucLCwsQ14qPMzMzBNuPgeSrIfOT1Sl7c9MADD+zc%0AuXMIHBHEUkaTjgJ2AfAa2mCnO6TM8EasedaPHDkyNTX1zTff7OjoCHylog9L%0Ae0uZdevWyTL5+OOPV1RUeDyeqqqqrVu35uTkIGWGPbl/+umn0mDFihUNDQ2t%0Ara1lZWUffPBBeno6UiagpIz6LHPnyJEjaWlp6n+JImX6lpH06Qgpg5QJHCmj%0Alulz5849+uijUhQ1g5QZNilz+PDhkJCQjIyMrq6uQV3YkDJ98JGIfWnQ1NTE%0AA8zAlzIKGZccueOOO5AyfchIPaYjm64QzKNglTIKccFAzXqkTB+ljNz9SLMd%0AO3ZYGbz+5uPy5cv5+fm33HJLZGRkSkpKXl7eoUOHun0Ob/x6UVHRnDlz4uPj%0AExISsrKydu/e7XVmuS1Tf48XGhrqQCkTExMjDXJzc2VQjY2N3bbR2FDjkR6d%0AZZxk3759999/v1xJdHT0jBkzpOjlo+3bt5u/CVnuUZwsZdra2tQzZysGtGJe%0AR0mZHtNRfwKyx3DVTCXht99+e/LJJ8eNGydTZtq0acZrxG6T26BOIqRMoEkZ%0AWafU141ZzMzCpk2bRPpISEydOvXLL7+0vs72Oef7W4WDRMokJydLswsXLliU%0AMnLP5K921qxZ8vmbb75pb28XKxQUFMjdlSYvHzx4MDw8XCZwVVWVrNOLFi2S%0ABuJUc/vVq1f/+uuvnZ2dznwq8+6775qDb/z48U8//bTYzaINNR7RVJk9JTlX%0AROT06dMlQqqrq+WDFI1EbHwTsvioublZ/Z9NObOTpcxPP/1kvj/TG9CKeR0l%0AZXpMR/0JSH2tfiqJkUWFuN3uAwcOtLS0lJaWyiJh8QXTgE8ipEygSZnKykop%0ATpgwwWJm3r9/v/rWzOobyAfr62yfc36QP5URo6iHHxalTFhYmL/a2NhY+Swa%0A0DwejRHFGXKkvLxcFevq6qQ4adIkc/tffvnF4dt+ZTjZ2dlidkPQqO8WsWJD%0AjUc0VWZPqTl25MgR4/m/FOX+0tyypKREFSXF+96aOGqvTHFxsdor8/HHH1sx%0AoBXzOirae0xH/QlIfa1+Ks2ePVuK/jZ066XMgE8ipExA7ZU5f/682iuzfPly%0Ai6vbgw8+qP4ZnSpKbFhfZ/uc84Ncyrhcrl49ldGYWCanKsbExNx7773Lli2r%0AqanRGFG9PfHCOL8q9mpPuE2/jNTK0ERiS7jLVImLizPf9OttqPGIpsp8VXIn%0AKp8lvaqi3DVKUQ6aW16/ft2IWCtTJZh85NVszJgxXn/BpDegFfM6Ktp7TEf9%0ACUh9rX4qqVp/L3n1UmbAJ5F9vR80EevbTMSEebXSh1NiYqJvSFhcZ/uc89kr%0A838mNu9Zkelnrr148eLSpUvdbrdhX+POQyNl6uvrB2onnRP+00ZhYaH5pk1v%0AQ41HNFWaLKxuGaXTPv/hRtC/YNIvvV4G7K152SvTn4DU1+qnkqr9888/+y9l%0A+j+JeCoTOE9lPB7P8ePHMzIypPjtt99aXN28pIwKCYvrbJ9zviP+gikzM7Pb%0A51Feg4+IiJDPbW1t5m0Bvr00NTWJOeR4bGyscVB68WqpHrJt374dKWMdFfQT%0AJ060YkO9R/xVaZ6NqwehmjcgSBn9CxEvA/bWvA75CyZNOupPQOpr9VMpJydH%0Aanft2tVtrW9yG9RJhJQJtL0yV65ckcUxKyvL4uqmdrT4e8GkX2f7k/N9AzV4%0ApIywdu1a0YALFiw4efKkCMDz58/LXdHs2bP9vaV77733ZEEtKyubMmWKuXb6%0A9OlbtmyRX5eT7NmzR47PmTPH6OXmm2+WI7///rs5bYnPUlJSSkpKxG2nTp3a%0AsGHDfffdh5QxuPPOO1999dUffvjh3LlzIv+vXr2qvvp1/fr1Vmyo8YimynfH%0AotqeJnpfPuj3pSJlut2m6s+AvTWvE4S7Ph31JyD1tfqpJAejoqJuu+22H3/8%0AsbW19cSJE0888YQmuQ3qJELKBJqUMR4oHjhwwEo4qW/89rftV7/O9jnndxuo%0AQSVl1P78vLy88ePHjxw5Mjo6+tZbb507d66vz2RBzc3NTUxMHDVqlCjQbdu2%0AmWuPHj361FNPyWyPjIx0u93PPPNMbW2t0UVBQcHYsWO9IuDYsWPz5s1LSkoS%0A36Smpubn58uVIGUM5s+fn56e7nK5xCmS/kaPHp2dnS1mN7fR2FDjEU1Vt39H%0AGn0DcbqY1+JzdaRMjwbsrXkd8gxSk476E5A9hqtmKgmyqCxcuDA5OVmmzNSp%0AU807onyT26BOIqRMAEoZiQc5IirEYjhJzNx+++0SD5LhN27caH2d7XPO97cK%0AB5WUCQ74Vhp85BwfYUkng5QJsogd+v+yiJRhepPc8RGWJNrxPlIGKYOUYXrj%0AI3yEJQEpQ8QiZUgcJHd8hJTBkkgZZAcRi5TBDUwVfISUIdqZR0B+RsowvfER%0APsKSRDveR8ogZZAyJHd8hI+wJCBliFikDG4gueMjpAyWZB4BEYuUYXozVfAR%0AUoZox/tIGeJquKTMuHHjHPUVpgkJCbab3vgIH2FJcIL37Qj5OSCkjHDt2rXq%0A6ury8vKioqLCwsJ/BTsyRhmpjFdGLWO3xWzBR/gIS4ITvG9HyM8BIWWam5uv%0AXLlSWVlZWlp66NChPcGOjFFGKuOVUcvYbTFV8BE+wpLgBO/bEfJzQEiZtra2%0AhoaGS5cuyZWVlZWVBDsyRhmpjFdGLWO3xVTBR/gIS4ITvG9HyM8BIWU8Ho8I%0AK7kmUVgXLlz4d7AjY5SRynhl1DJ2W0wVfISPsCQ4wft2hPwcEFLm77//lqsR%0AbdXU1NTY2Hg12JExykhlvDJqGbstpgo+wkdYEpzgfTtCfg4IKaPo+h+dwY4x%0AUttNGHyEj7AkOMH7doT8HBBSBgAAAGBQQcoAAAAAUgYAAAAAKQMAAACAlAEA%0AAACkDAAAAABSBgAAAAApAwAAAICUAQAAAKQMAAAAAFIGAAAAACkDAAAASBmk%0ADAAAACBlAAAAAJAyAAAAAEgZAAAAQMoAAAAAIGUAAAAAkDIAAAAASBkAAABA%0AygAAAAAgZQAAAACQMgAAAICUQcoAAAAAUgYAAABg2KQMAAAAgO1AygAAAABS%0ABgAAAGA4+C9kR5aojf9bNAAAAABJRU5ErkJggg==%0A)

简单来说，应用是通过Session来访问Cassandra集群的，每个Cassandra集群中可以包含多个Session，假设Cassandra集群中包含n个节点，则一个Session中会包含n个连接池，每个节点对应一个连接池，而一个连接池中则包含m个连接(所以上图中，pool和connection之间是1：n的关系，这里的n并不一定等于节点个数)，一个连接则可以并发的发送一系列请求，每个发送的请求都对应一个stream id，而一个连接中的stream id的数目是有限的，在V2版本的协议中，至多有128个stream id，而V3版本的协议中，至多有32768个stream id。

## 选择连接和发送请求的过程
### 请求到达时选择连接的过程
当请求到达时，会尝试从当前的host的连接池中获取一个连接，步骤如下：
- Cassandra Java Driver会首先对连接池中的所有连接中in-flight状态的请求的数目进行排序，找出具有最小in-flight状态请求数目的连接，对应的连接记为least_busy；
- 获取least_busy中in-flight请求的数目，记为inFlight，并和min{该连接中剩余可用的stream ids数目，一个连接中最大的并发请求数目}进行比较：
    - 如果inFlight不小于后者，则将当前的获取连接的操作加入到等待队列中，等待可用的连接，具体执行如下步骤：
        - 生成一个PendingBorrow对象，该对象中会定义一个超时任务timeoutTask，这个timeoutTask的作用是若当达到一定的超时时间之后，这个PendingBorrow对象仍然存在于等待队列中，且该timeoutTask没有被cancel，则表明经过这个超时时间之后，仍然没有可用的连接来供当前的获取连接的操作使用；
        - 将该PendingBorrow对象添加到等待队列pendingBorrows中；
        - 返回该pendingBorrow对象中的关于connection的Future；
    - 如果inFlight小于后者，则返回该least_busy；
- 通过该获取到的连接least_busy发送请求，如果发送请求的过程中出现异常，则执行以下异常处理：
    - 如果是ConnectionException，则获取下一个host，并尝试从它的连接池中获取一个连接，过程同上；
    - 如果是BusyConnectionException，则获取下一个host，并尝试从它的连接池中获取一个连接，过程同上；
    - 如果是RuntimeException，则获取下一个host，并尝试从它的连接池中获取一个连接，过程同上；

如果某个连接(记为avail_conn)可用时，就会检查等待队列中是否存在等待获取连接的操作，执行如下步骤：
- 获取avail_conn中in-flight请求的数目，记为inFlight，并和min{该连接中剩余可用的stream ids数目，一个连接中最大的并发请求数目}进行比较：
    - 如果inFlight不小于后者，则表明这个连接又处于busy状态了，已经被其它获取连接的操作占用了，直接退出；
    - 如果inFlight小于后者，则更新avail_conn中in-flight请求的数目(加1)，并继续执行如下步骤：
        - 从等待队列pendingBorrows中移除队列头部的PendingBorrow对象，记为pendingBorrow；
        - 设置pendingBorrow对应的connection为avail_conn；
        - cancel pendingBorrow中的timeoutTask；
        
#### 源码片段
```
    public ResultSetFuture SessionManager::executeAsync(final Statement statement) {
        if (isInit) {
            DefaultResultSetFuture future = new DefaultResultSetFuture(this, cluster.manager.protocolVersion(), makeRequestMessage(statement, null));
            # 创建一个RequestHandler，并借助其来发送请求
            new RequestHandler(this, future, statement).sendRequest();
            return future;
        } else {
            ...
        }
    }
    
    void RequestHandler::sendRequest() {
        startNewExecution();
    }
    
    private void RequestHandler::startNewExecution() {
        if (isDone.get())
            return;

        Message.Request request = callback.request();
        int position = executionCount.incrementAndGet();

        # 生成一个新的推测执行
        SpeculativeExecution execution = new SpeculativeExecution(request, position);
        runningExecutions.add(execution);
        
        # 执行
        execution.findNextHostAndQuery();
    }
    
    void SpeculativeExecution::findNextHostAndQuery() {
        try {
            Host host;
            # 获取下一个host，并尝试从该host获取一个连接，如果成功获取到连接，则通过该连接发送请求
            while (!isDone.get() && (host = queryPlan.next()) != null && !queryStateRef.get().isCancelled()) {
                # 从该host获取一个连接，如果成功获取到连接，则通过该连接发送请求
                if (query(host))
                    return;
            }
            
            # 如果所有host都尝试过了，仍然没有将请求成功发送出去，则返回NoHostAvailableException
            reportNoMoreHosts(this);
        } catch (Exception e) {
            ...
        }
    }
    
    private boolean SpeculativeExecution::query(final Host host) {
        # 获取host对应的连接池
        HostConnectionPool pool = manager.pools.get(host);
        if (pool == null || pool.isClosed())
            return false;

        PoolingOptions poolingOptions = manager.configuration().getPoolingOptions();
        # 从连接池中获取一个连接
        ListenableFuture<Connection> connectionFuture = pool.borrowConnection(
                poolingOptions.getPoolTimeoutMillis(), TimeUnit.MILLISECONDS,
                poolingOptions.getMaxQueueSize());
                
        Futures.addCallback(connectionFuture, new FutureCallback<Connection>() {
            # 成功获取到连接的时候，onSuccess会被调用
            @Override
            public void onSuccess(Connection connection) {
                if (current != null) {
                    if (triedHosts == null)
                        triedHosts = new CopyOnWriteArrayList<Host>();
                    triedHosts.add(current);
                }
                
                current = host;
                try {
                    # 发送请求
                    write(connection, SpeculativeExecution.this);
                } catch (ConnectionException e) {
                    # 返回当前的连接给连接池
                    if (connection != null)
                        connection.release();
                    # 尝试下一个host
                    findNextHostAndQuery();
                } catch (BusyConnectionException e) {
                    # 返回当前的连接给连接池
                    connection.release();
                    # 尝试下一个host
                    findNextHostAndQuery();
                } catch (RuntimeException e) {
                    # 返回当前的连接给连接池
                    if (connection != null)
                        connection.release();
                    # 尝试下一个host
                    findNextHostAndQuery();
                }
            }

            # 当在当前的host上没有获取到连接的时候，onFailure会被调用
            @Override
            public void onFailure(Throwable t) {
                if (t instanceof BusyPoolException) {
                    logError(host.getSocketAddress(), t);
                } else {
                    logger.error("Unexpected error while querying " + host.getAddress(), t);
                    logError(host.getSocketAddress(), t);
                }
                
                # 继续尝试下一个host
                findNextHostAndQuery();
            }
        });
        
        return true;
    }
    
    # 从当前连接池获取一个可用的连接
    ListenableFuture<Connection> HostConnectionPool::borrowConnection(long timeout, TimeUnit unit, int maxQueueSize) {
        Phase phase = this.phase.get();
        if (phase != Phase.READY)
            return Futures.immediateFailedFuture(new ConnectionException(host.getSocketAddress(), "Pool is " + phase));

        if (connections.isEmpty()) {
            ...
        }

        int minInFlight = Integer.MAX_VALUE;
        
        # 获取处于infight状态的请求数目最少的连接
        Connection leastBusy = null;
        for (Connection connection : connections) {
            int inFlight = connection.inFlight.get();
            if (inFlight < minInFlight) {
                minInFlight = inFlight;
                leastBusy = connection;
            }
        }

        if (leastBusy == null) {
            ...
        } else {
            while (true) {
                # leastBusy所代表的连接中处于inflight状态的请求的数目
                int inFlight = leastBusy.inFlight.get();

                # min(leastBusy.maxAvailableStreams(), options().getMaxRequestsPerConnection(hostDistance))表示leastBusy
                # 中可以承受的处于inflight状态的请求的最大数目
                #
                # 如果inFlight不小于后者，则表明不能将当前的请求添加到当前的连接中，则需要将当前获取连接的请求添加到等待队列中
                if (inFlight >= Math.min(leastBusy.maxAvailableStreams(), options().getMaxRequestsPerConnection(hostDistance))) {
                    return enqueue(timeout, unit, maxQueueSize);
                }

                # 否则，增加当前连接中处于inflight状态的请求的数目
                if (leastBusy.inFlight.compareAndSet(inFlight, inFlight + 1))
                    break;
            }
        }

        # 更新连接池中所有连接上处于inflight状态的请求的数目
        int totalInFlightCount = totalInFlight.incrementAndGet();
        
        # 更新连接池中目前为止处于inflight状态的请求的总数目的最大值
        // update max atomically:
        while (true) {
            int oldMax = maxTotalInFlight.get();
            if (totalInFlightCount <= oldMax || maxTotalInFlight.compareAndSet(oldMax, totalInFlightCount))
                break;
        }

        # 检查是否需要启动新的连接
        int connectionCount = open.get() + scheduledForCreation.get();
        if (connectionCount < options().getCoreConnectionsPerHost(hostDistance)) {
            maybeSpawnNewConnection();
        } else if (connectionCount < options().getMaxConnectionsPerHost(hostDistance)) {
            // Add a connection if we fill the first n-1 connections and almost fill the last one
            int currentCapacity = (connectionCount - 1) * options().getMaxRequestsPerConnection(hostDistance)
                    + options().getNewConnectionThreshold(hostDistance);
            if (totalInFlightCount > currentCapacity)
                maybeSpawnNewConnection();
        }

        # 返回leastBusy所代表的连接
        return leastBusy.setKeyspaceAsync(manager.poolsState.keyspace);
    }

    # 将当前获取连接的请求添加到等待队列
    private ListenableFuture<Connection> HostConnectionPool::enqueue(long timeout, TimeUnit unit, int maxQueueSize) {
        # 如果timeout为0，或者maxQueueSize为0，则不能添加到等待队列，直接返回Exception
        if (timeout == 0 || maxQueueSize == 0) {
            return Futures.immediateFailedFuture(new BusyPoolException(host.getSocketAddress(), 0));
        }

        while (true) {
            # 如果等待队列中处于等待状态的元素数目操作了整个连接池中所允许的最大的等待队列大小，则直接返回Exception
            int count = pendingBorrowCount.get();
            if (count >= maxQueueSize) {
                return Futures.immediateFailedFuture(new BusyPoolException(host.getSocketAddress(), maxQueueSize));
            }
            
            # 更新等待队列中处于等待状态的元素数目(加1)
            if (pendingBorrowCount.compareAndSet(count, count + 1)) {
                break;
            }
        }

        # 生成一个PendingBorrow对象，代表当前即将添加到等待队列中的用于获取连接的请求，
        # 在PendingBorrow中会包含一个关于Connection的future，以及一个超时任务，若在给定
        # 的超时时间内成功获取到connection，则设置其所对应的Connection，否则超时任务被
        # 触发，返回BusyPoolException
        PendingBorrow pendingBorrow = new PendingBorrow(timeout, unit, timeoutsExecutor);
        # 添加到等待队列
        pendingBorrows.add(pendingBorrow);

        // If we raced with shutdown, make sure the future will be completed. This has no effect if it was properly
        // handled in closeAsync.
        if (phase.get() == Phase.CLOSING) {
            pendingBorrow.setException(new ConnectionException(host.getSocketAddress(), "Pool is shutdown"));
        }

        # 返回PendingBorrow对象中关于Connection的future
        return pendingBorrow.future;
    }
    
    # 当一个连接connection可用时，尝试满足处于等待队列中的一个获取连接的请求
    private void HostConnectionPool::dequeue(final Connection connection) {
        while (!pendingBorrows.isEmpty()) {
            // We can only reuse the connection if it's under its maximum number of inFlight requests.
            // Do this atomically, as we could be competing with other borrowConnection or dequeue calls.
            while (true) {
                # 获取连接中处于inflight状态的请求的数目
                int inFlight = connection.inFlight.get();
                
                # min(leastBusy.maxAvailableStreams(), options().getMaxRequestsPerConnection(hostDistance))表示leastBusy
                # 中可以承受的处于inflight状态的请求的最大数目
                #
                # 如果inFlight不小于后者，则表明不能将当前的请求添加到当前的连接中，则需要将当前获取连接的请求添加到等待队列中
                if (inFlight >= Math.min(connection.maxAvailableStreams(), options().getMaxRequestsPerConnection(hostDistance))) {
                    // Connection is full again, stop dequeuing
                    return;
                }
                
                # 至此，connection中可以容纳一个新的请求，增加处于inflight状态的请求计数
                if (connection.inFlight.compareAndSet(inFlight, inFlight + 1)) {
                    // We acquired the right to reuse the connection for one request, proceed
                    break;
                }
            }

            # 从等待队列头部获取一个元素，是PendingBorrow类型的对象
            final PendingBorrow pendingBorrow = pendingBorrows.poll();
            if (pendingBorrow == null) {
                // Another thread has emptied the queue since our last check, restore the count
                connection.inFlight.decrementAndGet();
            } else {
                # 减少等待队列中元素的数目
                pendingBorrowCount.decrementAndGet();
                
                // Ensure that the keyspace set on the connection is the one set on the pool state, in the general case it will be.
                ListenableFuture<Connection> setKeyspaceFuture = connection.setKeyspaceAsync(manager.poolsState.keyspace);
                // Slight optimization, if the keyspace was already correct the future will be complete, so simply complete it here.
                if (setKeyspaceFuture.isDone()) {
                    try {
                        # 设置pendingBorrow对象对应的connection，同时取消pendingBorrow对象中的定时任务
                        if (pendingBorrow.set(Uninterruptibles.getUninterruptibly(setKeyspaceFuture))) {
                            totalInFlight.incrementAndGet();
                        } else {
                            connection.inFlight.decrementAndGet();
                        }
                    } catch (ExecutionException e) {
                        ...
                    }
                } else {
                    ...
                }
            }
        }
    }
```

### 发送请求的过程
#### 源码片段
```
    private boolean RequestHandler::query(final Host host) {
        ...
        
        # 从连接池中获取一个连接
        ListenableFuture<Connection> connectionFuture = pool.borrowConnection(
                poolingOptions.getPoolTimeoutMillis(), TimeUnit.MILLISECONDS,
                poolingOptions.getMaxQueueSize());
                
        # 当获取连接成功/失败时候的回调
        Futures.addCallback(connectionFuture, new FutureCallback<Connection>() {
            # 当成功获取连接时候，onSuccess会被调用
            @Override
            public void onSuccess(Connection connection) {
                if (current != null) {
                    if (triedHosts == null)
                        triedHosts = new CopyOnWriteArrayList<Host>();
                    triedHosts.add(current);
                }
                current = host;
                try {
                    # 发送请求
                    write(connection, SpeculativeExecution.this);
                } catch (ConnectionException e) {
                    ...
                } catch (BusyConnectionException e) {
                    ...
                } catch (RuntimeException e) {
                    ...
                }
            }

            @Override
            public void onFailure(Throwable t) {
                ...
            }
        });
        
        return true;
    }

    # 发送请求
    private void RequestHandler::write(Connection connection, Connection.ResponseCallback responseCallback) throws ConnectionException, BusyConnectionException {
        ...

        connectionHandler = connection.write(responseCallback, statement.getReadTimeoutMillis(), false);
        
        ...
    }
    
    ResponseHandler Connection::write(ResponseCallback callback, long statementReadTimeoutMillis, boolean startTimeout) throws ConnectionException, BusyConnectionException {

        ResponseHandler handler = new ResponseHandler(this, statementReadTimeoutMillis, callback);
        dispatcher.add(handler);

        # 生成一个Request
        Message.Request request = callback.request().setStreamId(handler.streamId);

        ...

        if (DISABLE_COALESCING) {
            # DISABLE_COALESCING默认为false
            channel.writeAndFlush(request).addListener(writeHandler(request, handler));
        } else {
            flush(new FlushItem(channel, request, writeHandler(request, handler)));
        }
        
        return handler;
    }
    
    private void Connection::flush(FlushItem item) {
        # 每一个Connection都对应一个Channel，其中Channel是Netty中的概念，
        # 这里获取Connection关联的Channel所对应的EventLoop
        EventLoop loop = item.channel.eventLoop();
        
        # 获取EventLoop所对应的Flusher，如果不存在，则创建之
        Flusher flusher = flusherLookup.get(loop);
        if (flusher == null) {
            Flusher alt = flusherLookup.putIfAbsent(loop, flusher = new Flusher(loop));
            if (alt != null)
                flusher = alt;
        }

        # 添加flusher的队列中
        flusher.queued.add(item);
        
        # 在EventLoop中执行Flusher的运行逻辑
        flusher.start();
    }
```

### Flusher类的定义
```
    private static final class Flusher implements Runnable {
        final WeakReference<EventLoop> eventLoopRef;
        final Queue<FlushItem> queued = new ConcurrentLinkedQueue<FlushItem>();
        final AtomicBoolean running = new AtomicBoolean(false);
        final HashSet<Channel> channels = new HashSet<Channel>();
        int runsWithNoWork = 0;

        # 通过构造方法设置Flusher所对应的EventLoop
        private Flusher(EventLoop eventLoop) {
            this.eventLoopRef = new WeakReference<EventLoop>(eventLoop);
        }

        void start() {
            # 如果Flusher不处于running状态，则设置其处于running状态，然后将当前的Flusher
            # 对应的工作添加到EventLoop中执行
            if (!running.get() && running.compareAndSet(false, true)) {
                EventLoop eventLoop = eventLoopRef.get();
                if (eventLoop != null)
                    eventLoop.execute(this);
            }
        }

        # Flusher的运行逻辑
        @Override
        public void run() {
            boolean doneWork = false;
            FlushItem flush;
            
            # 从Flusher的队列中获取请求并逐一处理，直到队列为空
            while (null != (flush = queued.poll())) {
                Channel channel = flush.channel;
                if (channel.isActive()) {
                    # channels表示该Flusher本轮处理中所有设计到的channel
                    channels.add(channel);
                    
                    # 通过Channel发送请求
                    channel.write(flush.request).addListener(flush.listener);
                    doneWork = true;
                }
            }

            # 对Flusher本轮处理中所有涉及到的Channel执行Flush
            // Always flush what we have (don't artificially delay to try to coalesce more messages)
            for (Channel channel : channels)
                channel.flush();
                
            # 清空本轮处理所涉及到的所有的Channel
            channels.clear();

            if (doneWork) {
                runsWithNoWork = 0;
            } else {
                # 如果连续5轮以上运行run()方法都没有执行任何有效的工作，则设置该Flusher的
                # running状态为false
                // either reschedule or cancel
                if (++runsWithNoWork > 5) {
                    running.set(false);
                    if (queued.isEmpty() || !running.compareAndSet(false, true))
                        return;
                }
            }

            EventLoop eventLoop = eventLoopRef.get();
            if (eventLoop != null && !eventLoop.isShuttingDown()) {
                eventLoop.schedule(this, 10000, TimeUnit.NANOSECONDS);
            }
        }
    }    
```

## Netty Channel和EventLoop
### 基本概念
Netty可以说是有Channel、EventLoop、ChannelFuture聚合起来的一个网络抽象代表

Channel——Socket；
EventLoop——控制流、多线程处理、并发
ChannelFuture——异步通知

### Channel接口
基本的I/O操作（bing()、connect()、read()、和write（））依赖于底层网络传输所提供的原始。在基于Java的网络编程中，其基本的构造是class Socket。Netty的Channel接口所提供的API，大大地降低了直接使用Socket类的复杂性。此外，Channel也是拥有许多预定义的、专门化实现的广泛类层次结构的根，下面是一个简短的部分清单：

EmbeddedChannel；
LocalServerChannel；
NioDatagramChannel；
NioSctpChannel；
NioSocketChannel；
EventLoop接口
EventLoop定义了Netty的核心抽象，用于处理连接的生命周期中所发生的事件。

一个EventLoopGroup包含一个或者多个EventLoop；
一个EventLoop在它的生命周期内只和一个Thread绑定；
所有由EventLoop处理得I/O事件都将在它专有的Thread上处理
一个Channel在它的生命周期内只注册一个EventLoop；
一个EventLoop可能会被分配给一个或多个Channel。
注意，一个给定的Channel的I/O操作都是由相同的Thread执行的，实际上消除了对于同步的需要

### ChannelFuture接口
Netty所有的I/O操作都是异步的。因为一个操作可能不会立即返回，所以我们需要一种用于在之后得某个时间点确定其结果的方法。为此，Netty提供了ChannelFuture接口，其addListener()方法注册了一个ChannelFutureListener，以便在某个操作完成时（无论是否成功）得到通知。

