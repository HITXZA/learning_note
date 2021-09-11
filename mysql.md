吞吐量 存储量 单机存不下了就得扩展了 扩展方式是分库分表  或者引入分布式存储

# mysql





page是内外村交换的基本单位，也是存储的基本单位。

页内记录是逻辑有序

不是物理有序

![image-20210909202314078](.\image\image-20210909202314078.png)

页和页之间是双向链表

![image-20210909202342109](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909202342109.png)





![image-20210909202502698](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909202502698.png)



通过看自由空间 可以验证为逻辑有序，你要是物理有序那早就挤到前面了 就不存在自由空间了



页面中数据的插入策略：

1、先使用自由空间链表，也就是蓝色的那些。用的时候用自由空间链表的头的部分

2、再使用未使用空间



放不下了才用未分配空间，什么情况下放不下呢，记录是变长的情况，放不下放不满都是存在的，变长的情况下，数组特性二分是用不了的



58某老哥：很多年前的mango 写数据都是顺着往下写，不用自由链表空间，用的多了之后可能想获得整个表要加载十几G但是里面存储的很稀疏，可能有效数据只有几百M，所以这个时候他们就做了个操作，就是把数据库的表重新copy到别的地方，再copy回来，把稀疏的弄成密集的（挨着存就行）。

主从机各有一个数据库，然后从机的删了，主机往从写，紧挨着写，从机写完了然后把主机的也删了，把从机写好的密集的数据写回主机，就好了。（。。。）



#### 页内查询： 

遍历：效率低

二分查找：用二分快很多  slot区很关键

![image-20210909203233001](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909203233001.png)

slot区把链表分成很多sublist

我们先对slot进行二分查找 找到sublist 然后在对sublist进行查找就行了

嗯。。。总的来说就是  跳表 二级索引





# 02 mysql innodb的内存管理：

数据库预热：

执行完全表扫描以后，把热数据存在了内存里了

实现：LRU分为LRU_new 和LRU_old

热数据和冷数据

![image-20210911153655075](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911153655075.png)



策略叫冷热分离

用midpoint指针指向5/8 new 是5 old占3

存数据的时候只往冷表里写   不满足条件进不了热表 热表就不会被冲掉

操作：从空闲链表里取出空闲页，

当没有空闲页的时候，要做页面淘汰，

先从Freelist里找>LRU中淘汰>LRU Flush（从尾巴往前找，找到第一个脏的页然后刷了他，然后拿过来用）

![image-20210911154247648](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911154247648.png)



new到old：old到new的时候old区长度会减一，new区加一，那这个时候要移动midpoint，这个热数据到冷数据就解决了。

LRU_new的操作：链表的效率很高，有访问就移动到表头？（最简单想到的思路）

不行！！！ 是因为锁Lock！！！

因为LRU是全局的，很多线程都要去访问表，你一个线程去移动LRU的时候，其他线程只能等啊，这就很难顶了。链表操作是没问题的，问题发生在并发上。

那么innodb是怎么做的呢？  减少锁的竞争，聚合！几次移动用一次移动进行聚合，减少锁的次数





# 03 mysql事务实现原理拆解以及设计深度剖析

![image-20210909210138061](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909210138061.png)

隔离：你做你的，我做我的，我没提交你看不见 你没提交我看不见



![image-20210909210237532](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909210237532.png)

脏读：读取未提交的数据

![image-20210909210343697](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909210343697.png)

A不应该能读到200  B没提交呢 你不应该能读到好吧



不可重复读：

![image-20210909210450081](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909210450081.png)

隔离开 就是说你一个事务中同一个变量的值不能变  我不能读者读着被别人改了



幻读：是表的改变  事务A读取的表 读两次发现两次

![image-20210909210636950](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909210636950.png)

A读不到4d 但是也写不进去  A读的是快照 应该是copy on write  此时B已经把4d写了 所以是有主键冲突

![image-20210911162813925](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911162813925.png)

RR可以解决幻读 但是需要有前提 前提是两次当前读



### MVCC：多版本并发控制 解决读写冲突

![image-20210911163025228](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911163025228.png)

MVCC用的是读的是快照 你写你的 你改的当前数据 我读的是历史记录 （快照）

通过隐藏列来实现的

历史版本有很多 我读哪个呢 ？ 看事务id  事务id是递增的

![image-20210909211148049](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909211148049.png)

B怎么提交都无所谓 A都是读某一个历史版本 B只能修改当前版本  不影响A读的那个历史版本

![image-20210911163358309](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911163358309.png)





事务A 创建快照的这一刻 你没提交的事务我都看不到 我就以提交的为准

当然 你在创建快照之后创建的事务我更看不见了 所以这个我也不需要

下面举例事务ID：

![image-20210909211234704](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909211234704.png)





![image-20210909211522169](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909211522169.png)



21-32之间的都提交了哈 现在还没提交的事务这是

最小的10 最大的是49 而50 51 52可能都提交了 现在事务是53

![image-20210909211549310](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909211549310.png)

如果小的话 说明快照创建的这一刻 这个事务已经提交了 是可见的

如果不是 如果事务id大于最大id  这个事务是创建快照之后创建的（不管你提交没提交 我不看了） 所以就回退指针到上一个版本

如果在中间  就是要看活跃列表 如果是在活跃列表里 证明还没提交 得找上一个版本

如果不在 就提交了 所以是可见的 可以用

这样就实现了RR级别的快照读。

![image-20210911164343000](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911164343000.png)



最小活跃事务id之前的版本全局谁都能看，因为都提交了，所以最小活跃事务id之前的undolog就可以删了。

![image-20210911165046389](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911165046389.png)







![image-20210909212011265](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909212011265.png)



![image-20210909212136076](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909212136076.png)

为啥不做一个计数器统计呢？

答：做不了 每个人看到的计数都不一样啊  快照那么多 你我看到的版本都不一样  我可能看到100条 你可能看到500条 这没法计数 每个人都相当于是定制的 啥时候想获取数量了去数数吧就







![image-20210911170344650](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911170344650.png)



行锁：

是锁一行 但是作用在索引上

PK主键 是索引到一堆数据

二级索引 :索引的是主键 

比如

![image-20210911170502916](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911170502916.png)





![image-20210911170543272](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911170543272.png)



有人问你怎么加锁 先问唯一索引还是非唯一索引 再问隔离级别

这种是4种情况

![image-20210911170751296](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911170751296.png)



![image-20210911170808830](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911170808830.png)

锁住134之后  你再往里插134 是能插入的  非唯一索引 可以插入  你没提交的话  这个时候就出问题了 你删除两条你等会发现还有一条。。。这就是幻读了

幻读：其他事务插入了新的满足条件的事务

这个 行锁不行了 要用间隙锁了

![image-20210911171019611](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911171019611.png)

两次当前读之间 不会有其他事务插入满足条件的数据

间隙锁锁的是区间

![image-20210911171145250](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911171145250.png)

当然这个区间可能会被锁的很大。。比如索引1 10000 那从1到1w都得被锁住 这个就比较难顶



表级锁：

![image-20210911171231561](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911171231561.png) 

mysql server会判断 如果不满足条件 直接释放  这个操作尽量不做哈



死锁情况：

T1先锁了120 T2先锁了130

然后T1想锁130 发现锁不了

T2想锁120 也锁不了 

他俩就卡住了 死锁了 进行不下去了 



![image-20210911171540092](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911171540092.png)



# 05、千亿级海量数据高并发场景分库分表实践落地方案

![image-20210911171943082](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911171943082.png)

比如uid为pk（主键）的时候

把一个表分到128个表里

直接对128取模  x  = uid%128

插入的时候按x的值插入到 INSERT TABLE_UID_x

![image-20210911172106310](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210911172106310.png)



如果是用手机号查询的话 没法用uid寻找表，只能遍历128个表 这个有点不好

这个情况看uid查询和手机号查询的比例

谁比例高 就拿谁去分