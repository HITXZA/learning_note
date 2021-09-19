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

