吞吐量 存储量 单机存不下了就得扩展了 扩展方式是分库分表  或者引入分布式存储

# mysql





page是内外村交换的基本单位，也是存储的基本单位。

页内记录是逻辑有序

不是物理有序

![image-20210909202314078](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909202314078.png)

页和页之间是双向链表

![image-20210909202342109](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909202342109.png)





![image-20210909202502698](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909202502698.png)



通过看自由空间 可以验证为逻辑有序，你要是物理有序那早就挤到前面了 就不存在自由空间了



插入策略：

1、先使用自由空间链表，也就是蓝色的那些

2、再使用未使用空间



#### 页内查询： 

遍历：效率低

二分查找：用二分快很多  slot区很关键

![image-20210909203233001](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909203233001.png)

slot区把链表分成很多sublist

我们先对slot进行二分查找 找到sublist 然后在对sublist进行查找就行了

嗯。。。总的来说就是  跳表 二级索引





# 02 mysql innodb的内存管理：

明天补一下。。。



# 03 mysql事务实现原理拆解以及设计深度剖析

![image-20210909210138061](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909210138061.png)

你做你的，我做我的，我没提交你看不见 你没提交我看不见



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



MVCC就是解决幻读  用的是快照读 你改的当前数据 我读的是历史记录 （快照

历史版本有很多 我读哪个呢 ？ 看事务id  事务id是递增的

![image-20210909211148049](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909211148049.png)

B怎么提交都无所谓 A都是读某一个历史版本 B只能修改当前版本  不影响A读的那个版本

![image-20210909211234704](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909211234704.png)





![image-20210909211522169](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909211522169.png)

21-32之间的都提交了哈 现在还没提交的事务这是

最小的10 最大的53

![image-20210909211549310](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909211549310.png)

如果小的话 已经提交了  所以是可见

如果不是 如果事务id大于最大id  这个事务是创建快照之后创建的 所以就回退指针到上一个版本

如果是在活跃列表里 证明还没提交



这样就实现了RR级别的快照读。



![image-20210909212011265](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909212011265.png)



![image-20210909212136076](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210909212136076.png)

为啥不做一个计数器统计呢？

答：