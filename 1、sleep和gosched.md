ps：加了go关键字的就是子go程  不加的就是主go程

##### 0、计算golang程序运行的时间

start := time.Now()
//some func or operation
cost := time.Since(start)
fmt.Printf("cost=[%s]",cost)

##### 1、sleep和gosched:

1、sleep是睡眠 然后让出cpu gosched是主动让出
2、sleep睡眠之前有设置一个定时器，睡眠期间不可以尝试获取cpu，睡眠醒来之后才可以争取cpu。而gosched可以出现刚把cpu让出去就又争夺回来的情况

##### 2、return和goExit（）

return : 后续的指令都不会执行了



![image-20210919173659339](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210919173659339.png)



看好了哈 这个runtime.Goexit() 退出后 你之前执行过的defer是可以生效的 但是return不行

defer类似于一个注册的机制 程序结束前为了方便资源回收 

![image-20210919173719948](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210919173719948.png)

##### 3、GOMAXPROCS 设置并行计算时cpu核数的最大值

设置当前进程使用的最大cpu核数，返回上一次调用成功的设置值，首次调用返回默认值。

##### 下图分析：

应该是起了一堆go程
然后都攒一块没执行
因为cpu在主go程里
然后等主go成让出时间片以后
其他的go程执行打印一堆0
从换行能看出来



![image-20210919195627905](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210919195627905.png)





不借助锁 也不允许go程直接读数据

我们借助中间的一个部件，channel

channel类似于一个缓冲区   是通过通信来共享内存  而不是通过共享内存来通信

![image-20210920093527600](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210920093527600.png)

channel的定义： 

make(chan 空格 在channel中传递的数据类型，容量n)

如果容量n是大于0的 说明是有缓冲channel   能存n个前面声明的数据类型

make(chan int )  make (chan int, 2 )

len(ch) 得到的是channel中剩余未读取的数据个数，cap(ch)是通道的容量。

channel有两个端：

写端： 传入端 chan<-

读端：传出端 <-chan

如果想完成通信，要求读端和写端必须同时满足条件，才能在chan上进行数据流动，否则阻塞。

一个基于无缓存Channels的发送操作将导致发送者goroutine阻塞，直到另一个goroutine在相同的Channels上执行接收操作，当发送的值通过Channels成功传输之后，两个goroutine可以继续执行后面的语句。反之，如果接收操作先发生，那么接收者goroutine也将阻塞，直到有另一个goroutine在相同的Channels上执行发送操作。

总结： 

无缓冲的channel，不管读写，谁先操作谁被阻塞，直到另一个goroutine把数据写进入或者读出来，这个被阻塞的goroutine才能继续执行。

读写要发生在不同的goroutine里 要不然会deadlock(死锁)

例子： 打印机是共享资源

如果不加锁，两个打印交替执行 打出来就是错误的 很难顶熬

![image-20210920094726189](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210920094726189.png)



从管道里读数据的时候，如果无缓冲区的管道是空的，他会阻塞，直到有别的goroutine往管道写了数据才能往下走哈

![image-20210920100841254](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210920100841254.png)





模拟图：

![image-20210920161501945](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210920161501945.png)

![image-20210920161454168](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210920161454168.png)





案例分析：

![image-20210920162701846](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210920162701846.png)

这个地方按理分析应该是

子go程写0

主go程读0

子go程写1

主go程读1

...

这么进行的

但是实际上 当主go程被唤醒，他想执行fmt.Println(),但是这个语句是一个IO操作，是很耗时的，这个时候子go程已经不被阻塞了，这些时候主go程一直没有得到硬件资源，所以我们看到是子go程连着打了两句

子go程写0

子go程写1

然后子go程阻塞了 硬件资源到了主go程手上





有缓冲的channel：

通道容量非0

channel应用于两个go程中 一个读 一个写

缓冲区可以进行数据存储，存储至容量上限，然后会阻塞。不需要同时操作缓冲区。

为啥会出现下面的情况呢？ 子go程还没写完  主go程就读到了？

![image-20210920182629959](C:\Users\86188\AppData\Roaming\Typora\typora-user-images\image-20210920182629959.png)



还是因为io操作耗时，会导致io的语句和其他语句有延迟，正常来讲应该是读完了，然后马上打印，但是实际上你要打印你必须得获取硬件资源，这个不一定能获取到，然后这个时候别的go程占用着硬件资源，就先让其他go程运行了。所以就是说3被传入了通道但是没在子go程中打印，然后切到主go程 主go程读了4个出来。





