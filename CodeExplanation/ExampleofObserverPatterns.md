在org.apache.zookeeper.server.DataTree类含有两个成员变量 
```java
	private IWatchManager dataWatches;
	private IWatchManager childWatches;
```
，表示两种WatchManager类型，用于监视数据与子节点。

这个接口的实现类是```WatchManager```

该实现类有字典watchTable，实现watch名字到对象实例集合的映射。

```java
	private final Map<String,Set<Watch>> watchTable=new HashMap<>();
```

这个里面装的就是各个观察者。

当中的 `key` 为 `String` 类型，这个表示的是节点的 `path` ，通过这个映射关系将 `path` 映射到各个节点所含有的监听器集合。

---



首先分析监听器是如何注册的，在 `org.apache.zookeeper.server.DataTree` 中含有 `addWatch`  函数，该函数用于将监听器注册到数据树上，首先获取到监听器的模式，然后根据需求，分别注册到到两个监听器上。

在这个函数中会调用到 `WatchManager` 的注册函数，通过调用该函数直接将监听器注册到管理者中进行管理。

---



然后分析监听器的删除，在 `org.apache.zookeeper.server.DataTree` 中含有 `removeCnxn`  函数，该函数接收一个参数，监听器对象，直接调用 `WatchMananger` 中的方法对某个监听器进行注销；同时还有多个函数重载，依据不同的参数对监听器进行注销；还有一个函数可以注销全部的监听器。

在这个函数中会调用到 `WatchManager` 的注销函数，通过调用该函数直接将监听器注册到管理者中进行管理。

调用 
```java
	removeWatcher(org.apache.zookeeper.Watcher)
```
，该函数直接删除注销监听器，同时对一些垃圾进行回收处理，确保数据树的干净整洁，避免对内存空间造成较大的损伤和无效的存储。

---



## 触发广播

`org.apache.zookeeper.server.DataTree#deleteNode` 删除节点时的触发

`org.apache.zookeeper.server.DataTree#createNode(java.lang.String, byte[], java.util.List<ACL>, long, int, long, long, Stat)` 增加节点时的触发

下面只对状态改变时进行说明，增加删除同理

`org.apache.zookeeper.server.DataTree#setData`

在调用到该函数时，首先会对相应的状态进行设置，改变需要调整的内容

```java
public Stat setData(String path, byte[] data, int version, long zxid, long time) throws NoNodeException {
    	......
        dataWatches.triggerWatch(path, EventType.NodeDataChanged, zxid);
        return s;
    }
```

在调整完相应的状态之后触发 `WatchManager` 的 `trigger` 方法，扳机扣动触发监听器，传入各种状态。

## 调用

上述过程中调用到扳机方法，下面具体对 `org.apache.zookeeper.server.watch.WatchManager` 中的函数 `triggerWatch` 进行说明。即对具体的监听器触发进行说明。该函数接收多个参数，重点参数为 `path` 节点的路径， `type` 触发的事件的类型等，这些变量用于找到需要触发的触发器集合，同时说明触发的事件类型和内容。

在函数中首先定义 

```java
	WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path, zxid);
````
用于创建一个`WatchedEvent`对象，表示发生在给定路径上的特定类型的事件。然后使用

```java
	Set<Watcher> watchers = new HashSet<>();
```

创建一个HashSet用于存放触发的Watcher。然后使用`synchronized (this) {`：加锁，确保线程安全，再执行下面的代码。
```java
	PathParentIterator pathParentIterator = getPathParentIterator(path);
```
通过路径获取一个迭代器，用于遍历父路径。在接下来的循环中 
```java
	for (String localPath : pathParentIterator.asIterable()) {
```
遍历路径的父路径，使用 
```java
	Set<Watcher> thisWatchers = watchTable.get(localPath);
```
来根据路径获取对应的监听器集合。这样遍历当前路径下的监听器，检查它们的监听模式并添加到`watchers`集合中。对于不同的监听模式，可能会修改监听器的状态。

当然依据 
```java
if (thisWatchers.isEmpty()) 
{ 
	watchTable.remove(localPath); 
}
```
如果当前路径下的监听器集合为空，需要从`watchTable`中移除。这样就获取到了所有需要触发的监听器集合。

遍历`watchers`集合，对每个Watcher执行相应的处理。如果存在需要抑制的Watcher（在`suppress`中），则跳过。其中 
```java
	w.process(e);
```
：调用Watcher的`process`方法，触发Watcher的处理逻辑。根据事件类型更新服务器的监控指标。最后需要返回一个`WatcherOrBitSet`对象，其中包含触发的Watcher集合。这样就完成了被监听者对所有需要通知到的监听器的通知。

总体来说，这段代码的主要目的是遍历给定路径及其父路径上的Watcher，根据事件类型触发相应的Watcher，并更新监控指标。  

