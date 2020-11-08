# 提纲
[toc]

## Overview of PostgreSQL Internals
### 一个查询的路径
1. 在查询之前，必须确保应用程序和PostgreSQL之间已经建立连接。
2. 应用程序发送查询请求到PostgreSQL server，并等待响应。
3. *Parser*进行语法检查，创建一个*query tree*。
4. *rewrite system*查找可以应用到该*query tree*的任何*rule*(存储在系统目录中)，对找到的*rule*，它会执行*rule bodies*中给定的转换。
    *rewrite system*的一个应用是实现*视图*。
5. *planner/optimizer*接手(重写过的)*query tree*并创建一个将被作为*executor*的输入的*query plan*。
    它会先创建所有可能导向相同结果的路径。例如，如果在一个被扫描的关系上有一个索引，则有两条可供扫描的路径，其中之一是一个简单的顺序扫描，而另一个则是使用索引。接下来执行每条路径所需的代价被估算出来并且代价最低的路径将被选中。代价最低的路径将被扩展成一个完整的计划可供*executor*使用。
6. *executor*递归地逐步遍历*plan tree*并按照计划表述的方式获取行。

### 连接如何建立
PostgreSQL以一种简单的“一用户一进程”的客户端/服务器模型实现。在该模型中，一个客户端进程仅连接到一个服务器进程。由于我们无法预先知道会有多少连接被建立，我们必须使用一个主进程在每次连接请求时生产一个新的服务器进程。该主进程被称为postmaster，它在一个特定的TCP/IP端口监听进入的连接。当一个连接请求被监测到时，postmaster会产生一个新的服务器进程postgres。服务器作业之间通过信号和共享内存通信，以保证并发数据访问时的数据完整性。

## 前端/后端协议
在PostgreSQL中，Frontend就是client，Backend就是server。

协议分为2个阶段，startup阶段和normal operation阶段。

在startup阶段，frontend打开一个到backend的连接并且认证自身以满足backend的要求

在正常操作中，frontend发送查询和其它命令到backend，然后backend返回查询结果和其它响应。在少数几种情况（比如NOTIFY）中，backend会发送未被请求的消息，但这个会话中的绝大多部分都是由frontend请求驱动的。

PostgreSQL包含2个子协议：simple query和extended query。在正常操作中，SQL命令可以通过两个子协议中的任何一个执行。 

在“simple query”协议中，frontend只是发送一个文本查询串，然后backend马上分析并执行它。
![image](https://image-static.segmentfault.com/244/311/2443110757-5be96f8278389_articlex)
> 如果在查询中包含多条SQL语句，则上述过程会重复执行多次，但是sync只在所有命令都处理完之后运行一次。


而在“extended query”协议中，查询的处理被分割为多个步骤：分析、参数值绑定和执行。不同步骤之间的状态被保留在2种对象中：*prepared statements*和*portals*。一个prepared statements代表一个文本查询字符串经过分析、语意解析以及规划之后的结果。一个portal则代表一个已经可以执行的或者已经被部分执行过的语句，所有缺失的参数值都已经填充到位了。
![image](https://image-static.segmentfault.com/111/841/1118410008-5bf623dca716e_articlex)
> 跟simple query的区别在于：每一个步骤都是单独发送消息。事实上，frontend无需等待接收到backend的响应之后才发送下一个消息，真实情况下，网络序列可能是这样：
![image](https://image-static.segmentfault.com/420/604/4206045842-5be98082beb0c_articlex)


frontend和backend之间所有的通讯都是通过一个消息流进行的。消息的第一个字节标识消息类型， 然后后面跟着的四个字节声明消息剩下部分的长度（这个长度包括长度域自身，但是不包括消息类型字节）。 剩下的消息内容由消息类型决定。


### Message Flow
![image](https://www.postgresql.org/media/img/developer/backend/flow.gif)

参考：https://www.postgresql.org/developer/backend/


