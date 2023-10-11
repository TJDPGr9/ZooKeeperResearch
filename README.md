# ZooKeeperResearch
本仓库用于分析ZooKeeper的观察者模式
## Schedule
1. 全员学习观察者模式
2. 提供内容：
   - 由4人对ZooKeeper的[源码](https://github.com/apache/zookeeper)进行分析：
     4人研究包括但不限于以下类的关系
     1. org.apache.zookeeper.Watcher 接口
     2. org.apache.zookeeper.Watcher.Event.EventType 枚举
     3. org.apache.zookeeper.WatchedEvent 类
     4. org.apache.zookeeper.ZooKeeper 类
     5. org.apache.zookeeper.ZooKeeper.States 枚举
     6. org.apache.zookeeper.server.ZooKeeperServer 类

     其中
     - 1人给出以上类、接口的监听机制解释，
     - 1人绘制UML图并审阅相关解释。
     - 另2人给出ZooKeeper论文中提到的API是如何使用这个机制的（需要阅读[ZooKeeper论文](https://pdos.csail.mit.edu/6.824/papers/zookeeper.pdf)的摘要与Introduction,ZooKeeper service的2.1与2.2部分，

       2.4部分只需要由1人阅读一个案例并写出伪代码进行解释即可，毕竟ZooKeeper允许用户自己设计一些原语，推荐Simple Locks w/o herd effect，因为有现成的源代码并且是监听相关。
       
       与watch相关的API有3个：
       - exists
       - getData
       - getChildren。
       
       相关代码在源码的测试类中，有
       - RemoveWatchesTest
       - RemoveWatchesCmdTest
       - ConfigWatcherPathTest
       
       三个测试类有选择地阅读（推荐1、2选择其中一个即可），只要覆盖了主要的watch相关API的基本使用情景即可。（**这2个人不需要阅读main目录下代码，1人负责论文伪代码解析，1人负责测试代码解析**）
    - 剩余1人制作简易PPT、讲稿并上台答辩。

P.S. 该分工为初步分工，如果发现工作量倾斜较大会进行相应减少或调整，比如API可以先只分析1个。

