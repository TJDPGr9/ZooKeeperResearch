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

1. 每一个想要获取锁的客户端都需要在I下创建一个带有“/lock-”前缀的znode节点n，这些节点按申请顺序排序给予一个由小到大的后缀。注意，这里n表示任意一次节点创建，也就是任意一次某客户端希望获取锁的请求，因此后缀大小依赖于有没有比它更早试图获取锁的申请。EPHEMERAL表示创建的节点是临时节点，当创建节点的会话结束时，该节点将被自动删除。SEQUENTIAL表示在节点名后面添加一个单调递增的序号。这是为了确保在并发创建节点的情况下，节点名称的唯一性和有序性；

2. 创建节点后，获取节点I下所有的子节点，放到列表C中，此时不设置监听；

3. 如果刚创建的n节点是列表C中后缀最小的一个节点，则表示该节点应该获得锁，直接退出；

4. 否则，获取后缀比n小的节点列表，并找到该列表中后缀最大的那一个节点p，也就是刚刚比n节点小的那个节点；

5.  用exists函数判断节点p是否存在，如果存在则对这个节点设置监听，以便在节点p发生变化（即前一节点使用完锁之后）能够唤醒它后面的节点n；这里用到了观察者模式，即在被观察者（前一节点）中注册观察者列表，当被观察者状态发生变化时（即节点被删除），发送通知给观察者（这里为后一个节点）；

6. 如果本轮节点n没有获取到锁，则转到步骤2继续尝试获取锁


 

解锁时只需要把节点n删除即可。

 

Simple Locks without Herd Effect的好处是每个客户端只监听排在自己前面的一个子节点而非所有排在自己前面的节点，避免了羊群效应，也就是避免了每一次小于自己的节点状态变化时自己都需要被唤醒一次。因为实际上如果不是后缀紧挨着自己的前面一个节点使用完所被删除，再靠前的节点被删除后自己还是因为不是后缀最小的那个而获取不到锁，唤醒起不到实质效果，为减少无效唤醒最好省去。

这样，任何一个子节点删除的通知只会发给下面一个客户端。通过设置监听，节点也避免了使用轮询来获取锁，而只需要等待前一节点使用完锁之后节点删除所触发的watcher发送通知的动作，后面节点被通知到之后就可以开始自己的获取锁的尝试。