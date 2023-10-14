org.apache.zookeeper.server.DataTree类有俩成员变量```dataWatches```与```childWatches```，表示两种Watch类型，用于监视数据与子节点。

这个接口的实现类是```WatchManager```

该实现类有字典watchTable，实现watch名字到对象实例集合的映射。

```java
	private final Map<String,Set<Watch>> watchTable=new HashMap<>();
```

这个里面装的就是各个观察者

---



**注册**

`org.apache.zookeeper.server.DataTree#addWatch` 函数

```java
public void addWatch(String basePath, Watcher watcher, int mode) {
        WatcherMode watcherMode = WatcherMode.fromZooDef(mode);
        dataWatches.addWatch(basePath, watcher, watcherMode);
        if (watcherMode != WatcherMode.PERSISTENT_RECURSIVE) {
            childWatches.addWatch(basePath, watcher, watcherMode);
        }
    }
```

调用到org.apache.zookeeper.server.watch.WatchManager下的
```java
	addWatch(java.lang.String, org.apache.zookeeper.Watcher, org.apache.zookeeper.server.watch.WatcherMode)
```

该函数讲监听器注册加入



**删除**

`org.apache.zookeeper.server.DataTree#removeCnxn`

```java
public void removeCnxn(Watcher watcher) {
        dataWatches.removeWatcher(watcher);
        childWatches.removeWatcher(watcher);
    }
```

调用

`org.apache.zookeeper.server.watch.WatchManager#removeWatcher(org.apache.zookeeper.Watcher)`

```java
@Override
    public synchronized void removeWatcher(Watcher watcher) {
        Map<String, WatchStats> paths = watch2Paths.remove(watcher);
        if (paths == null) {
            return;
        }
        for (String p : paths.keySet()) {
            Set<Watcher> list = watchTable.get(p);
            if (list != null) {
                list.remove(watcher);
                if (list.isEmpty()) {
                    watchTable.remove(p);
                }
            }
        }
        for (WatchStats stats : paths.values()) {
            if (stats.hasMode(WatcherMode.PERSISTENT_RECURSIVE)) {
                --recursiveWatchQty;
            }
        }
    }
```

删除注销监听器

---



## 触发广播

`org.apache.zookeeper.server.DataTree#deleteNode` 删除节点时的触发

`org.apache.zookeeper.server.DataTree#createNode(java.lang.String, byte[], java.util.List<ACL>, long, int, long, long, Stat)` 增加节点时的触发

下面只对状态改变时进行说明，增加删除同理

`org.apache.zookeeper.server.DataTree#setData`
```java
public Stat setData(String path, byte[] data, int version, long zxid, long time) throws NoNodeException {
        Stat s = new Stat();
        DataNode n = nodes.get(path);
        if (n == null) {
            throw new NoNodeException();
        }
        byte[] lastData;
        synchronized (n) {
            lastData = n.data;
            nodes.preChange(path, n);
            n.data = data;
            n.stat.setMtime(time);
            n.stat.setMzxid(zxid);
            n.stat.setVersion(version);
            n.copyStat(s);
            nodes.postChange(path, n);
        }

        // first do a quota check if the path is in a quota subtree.
        String lastPrefix = getMaxPrefixWithQuota(path);
        long bytesDiff = (data == null ? 0 : data.length) - (lastData == null ? 0 : lastData.length);
        // now update if the path is in a quota subtree.
        long dataBytes = data == null ? 0 : data.length;
        if (lastPrefix != null) {
            updateQuotaStat(lastPrefix, bytesDiff, 0);
        }
        nodeDataSize.addAndGet(getNodeSize(path, data) - getNodeSize(path, lastData));

        updateWriteStat(path, dataBytes);
        dataWatches.triggerWatch(path, EventType.NodeDataChanged, zxid);
        return s;
    }
```


## 调用

`org.apache.zookeeper.server.watch.WatchManager#triggerWatch(java.lang.String, org.apache.zookeeper.Watcher.Event.EventType, long, org.apache.zookeeper.server.watch.WatcherOrBitSet)`

```java

    @Override
    public WatcherOrBitSet triggerWatch(String path, EventType type, long zxid, WatcherOrBitSet supress) {
        // 这里就是监听到的事件
        WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path, zxid);
        // 这玩意拿来存放监听器
        Set<Watcher> watchers = new HashSet<>();
        // 加个锁保证线程安全
        synchronized (this) {
            PathParentIterator pathParentIterator = getPathParentIterator(path);
            for (String localPath : pathParentIterator.asIterable()) {
                // 这里根据 path 节点路径来获取到当前节点的监听器
                Set<Watcher> thisWatchers = watchTable.get(localPath);
                if (thisWatchers == null || thisWatchers.isEmpty()) {
                    continue;
                }
                // 下面开始遍历取监听器
                Iterator<Watcher> iterator = thisWatchers.iterator();
                while (iterator.hasNext()) {
                    Watcher watcher = iterator.next();
                    Map<String, WatchStats> paths = watch2Paths.getOrDefault(watcher, Collections.emptyMap());
                    WatchStats stats = paths.get(localPath);
                    if (stats == null) {
                        LOG.warn("inconsistent watch table for watcher {}, {} not in path list", watcher, localPath);
                        continue;
                    }
                    // 这里把取到的监听器放到 watchers 里面，后面拿来遍历 
                    if (!pathParentIterator.atParentPath()) {
                        watchers.add(watcher); // 增加
                        WatchStats newStats = stats.removeMode(WatcherMode.STANDARD);
                        if (newStats == WatchStats.NONE) {
                            iterator.remove();
                            paths.remove(localPath);
                        } else if (newStats != stats) {
                            paths.put(localPath, newStats);
                        }
                    } else if (stats.hasMode(WatcherMode.PERSISTENT_RECURSIVE)) {
                        watchers.add(watcher); // 增加
                    }
                }
                if (thisWatchers.isEmpty()) {
                    watchTable.remove(localPath); // 监听器都没，说明死了，删掉
                }
            }
        }
        if (watchers.isEmpty()) {
            if (LOG.isTraceEnabled()) {
                ZooTrace.logTraceMessage(LOG, ZooTrace.EVENT_DELIVERY_TRACE_MASK, "No watchers for " + path);
            }
            return null;
        }
		
        // 这里遍历这个集合
        for (Watcher w : watchers) {
            if (supress != null && supress.contains(w)) {
                continue;
            }
            // 该函数即为触发调用监听器自己的方法执行 即为 通知各个监听器
            w.process(e);
        }

        switch (type) {
            case NodeCreated:
                ServerMetrics.getMetrics().NODE_CREATED_WATCHER.add(watchers.size());
                break;

            case NodeDeleted:
                ServerMetrics.getMetrics().NODE_DELETED_WATCHER.add(watchers.size());
                break;

            case NodeDataChanged:
                ServerMetrics.getMetrics().NODE_CHANGED_WATCHER.add(watchers.size());
                break;

            case NodeChildrenChanged:
                ServerMetrics.getMetrics().NODE_CHILDREN_WATCHER.add(watchers.size());
                break;
            default:
                // Other types not logged.
                break;
        }

        return new WatcherOrBitSet(watchers);
    }
  
```

