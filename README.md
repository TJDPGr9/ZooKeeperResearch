# ZooKeeperSearch
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
     - 另2人给出ZooKeeper论文中提到的API是如何使用这个机制的（需要阅读[ZooKeeper论文](https://pdos.csail.mit.edu/6.824/papers/zookeeper.pdf)相关部分，API有3个：
       - exists
       - getData
       - getChildren)
    - 剩余1人制作简易PPT、讲稿并上台答辩。
P.S. 该分工为初步分工，如果发现工作量倾斜较大会进行相应减少或调整，比如API可以先只分析1个。

