# Simple locks w/o herd effect

## 从测试类到简单锁

结合```org.apache.zookeeper.lock```下的测试类```WriteLockTest```解释```ZooKeeper```如何利用Watch实现锁机制。

测试类```WriteLockTest```需要继承```ClientBase```类，这样测试类就可以作为```ZooKeeper```服务的客户端了。

测试方法```runTest```中，新建了3个写锁，分别位于对应的节点。遍历3个节点，创建一个客户端用于建立与```ZooKeeper```服务的会话。并且将第一个节点选举为领导者，该领导者位于根目录下，名字是当前类```WriteLockTest```的全类名。然后设置锁的监听器，注册两个回调函数```lockReleased```与```lockAcquired```，用于上锁与释放锁。



接着节点开始调用lock方法，在确认路径存在后，就需要用```retryOperation()```尝试10次lock操作。下面代码的```zop```是实现类```LockZooKeeperOperation```的一个实例。```retryOperation```会调用execute方法。

```java
public synchronized boolean lock() throws KeeperException, InterruptedException {
        if (isClosed()) {
            return false;
        }
        ensurePathExists(dir);

        return (Boolean) retryOperation(zop);
    }
```

## 避免羊群效应的简单锁伪代码

在```LockZooKeeperOperation```中，```execute```方法可以用如下伪代码表示：



```
Lock
1 n = create(l + "/lock-", EPHEMERAL|SEQUENTIAL)
2 C = getChildren(l, false)
3 if n is lowest znode in C, exit
4 p = znode in C ordered just before n
5 if exists(p, true) wait for watch event
6 goto 2

Unlock
1 delete(n)
```

 

 这段伪代码的含义是：

### Lock API

 1. 每一个想要获取锁的客户端都需要在I下创建一个带有“/lock-”前缀的```znode```节点n，这些节点按申请顺序排序给予一个由小到大的后缀。注意，这里n表示任意一次节点创建，也就是任意一次某客户端希望获取锁的请求，因此后缀大小依赖于有没有比它更早试图获取锁的申请。<em>EPHEMERAL</em>表示创建的节点是临时节点，当创建节点的会话结束时，该节点将被自动删除。<em>SEQUENTIAL</em>表示在节点名后面添加一个单调递增的序号。这是为了确保在并发创建节点的情况下，节点名称的唯一性和有序性；

 2. 创建节点后，获取节点I下所有的子节点，放到列表C中，此时不设置监听；

 3. 如果刚创建的n节点是列表C中后缀最小的一个节点，则表示该节点应该获得锁，直接退出；

 4. 否则，获取后缀比n小的节点列表，并找到该列表中后缀最大的那一个节点p，也就是刚刚比n节点小的那个节点；

 5. 用exists函数判断节点p是否存在，如果存在则对这个节点设置监听，以便在节点p发生变化（即前一节点锁释放节点被删除）能够触发```LockWatcher```调用```lock```方法后一个节点即可获得该互斥锁，```lock```的伪代码已给出；这里用到了观察者模式，即在被观察者（前一节点）中注册观察者列表，当被观察者状态发生变化时（即节点被删除），发送通知给观察者（这里为后一个节点）；

    ```java
    Stat stat = zookeeper.exists(lastChildId, new LockWatcher());
    ```

    #### 观察者机制运用

    那么上述代码中exists是如何让Watcher起作用的呢？
    
    ```java
    public Stat exists(final String path, Watcher watcher) throws KeeperException, InterruptedException {
            ...
            if (watcher != null) {
                wcb = new ExistsWatchRegistration(watcher, clientPath);
            }
        	...
    }
    ```

    - ```exists()```中的```ExistsWatchRegistration```在构造时就会在相应客户端路径上注册watcher。
    
    ```java
    public void register(int rc) {
                if (shouldAddWatch(rc)) {
                    Map<String, Set<Watcher>> watches = getWatches(rc);
                    synchronized (watches) {
                        Set<Watcher> watchers = watches.get(serverPath);
                        if (watchers == null) {
                            watchers = new HashSet<>();
                            watches.put(serverPath, watchers);
                        }
                        watchers.add(watcher);
                    }
                }
            }
    ```
    
    - 从上述代码可以看出，首先判断是否已经添加过该```Watcher```，然后通过调用```ExistsWatchRegistration```中的```getWatches```方法获取所有Exists方法注册的Watcher，之后的操作需要保证原子性，防止重复注册Watcher。然后获取客户端路径上注册过的Watcher，将相应的watcher添加到客户端路径中即注册成功。
    
    
    
    - ```processTxn```方法的事务头```TxnHeader```的Opcode判断事务类型。
    
    ```java
    	case OpCode.delete:
        case OpCode.deleteContainer:
             ...
             deleteNode(deleteTxn.getPath(), header.getZxid());
              break;
    	...
        case OpCode.closeSession:
    		...
             killSession(sessionId, header.getZxid());
    		...
    		break;
    ```
    
    
    
    - 如果是关闭会话事务，那么会调用```killSession```方法，里面调用```deleteNodes```，删除所有相关的节点，从而循环调用```deleteNode```删除节点；但如果是删除或删除容器事件，那么可以直接触发```deleteNode```。
    
    
    
    - 最后在```deleteNode```中会用```triggerWatch```方法通知已注册的Watcher删除事件
    
    ```java
    	WatcherOrBitSet processed = dataWatches.triggerWatch(path, EventType.NodeDeleted, zxid);
        childWatches.triggerWatch(path, EventType.NodeDeleted, zxid, processed);
        childWatches.triggerWatch("".equals(parentName) ? "/" : parentName, EventType.NodeChildrenChanged, zxid);
    ```
    
    ，该函数的机制已经在之前在介绍观察者机制的核心源码时介绍过。


6. 如果本轮节点n没有获取到锁，则转到步骤2继续尝试获取锁

### Unlock API

再详细解释一下unlock:

```java
public synchronized void unlock() throws RuntimeException {

    if (!isClosed() && id != null) {
        try {

            ZooKeeperOperation zopdel = () -> {
                zookeeper.delete(id, -1);
                return Boolean.TRUE;
            };
            zopdel.execute();
        }catch(...){...} 
        finally {
            LockListener lockListener = getLockListener();
            if (lockListener != null) {
                lockListener.lockReleased();
            }
            id = null;
        }
    }
}
```

 解锁时需要判断会话保持开启并且对应锁存在，然后利用函数式的Lambda表达式创建一个匿名类实现```ZooKeeperOperation```接口，delete实现了execute方法，如果执行成功就返回true，下面调用该实例的execute方法才是真正的执行删除，删除完成后需要释放原先的互斥锁，以便紧跟在后面的节点上锁。



在解释完没有羊群效应的简单锁的远离之后，就可以用它来实现领导人选举了，***注意，每次释放锁需要30s的时间来让之前的Leader挂掉***，因此释放后需要等待30s。前一个领导人牺牲后实施顺位继承，而且不用引发竞争，通过```dumpNodes```方法可以进行调试判断该机制是否能正常工作。
