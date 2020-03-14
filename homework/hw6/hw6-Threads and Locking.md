# Homework 6: Threads and Locking
这一部分只是简单地体会一下race condition和锁。我们需要做的就是跑一下代码，修一下bug，也就是加一下锁，代码运行结果如下：
两线程结果：
```
1: put time = 0.014111
0: put time = 0.014961
0: get time = 22.406305
0: 14824 keys missing
1: get time = 22.508685
1: 14824 keys missing
completion time = 22.524100
```
单线程结果：
```
0: put time = 0.007870
0: get time = 24.662313
0: 0 keys missing
completion time = 24.674224
```
我的因为是wsl2，跑得比较慢，但是它们之间的相对速度还是可以体现的。在指引中有着一句话需要说明一下：
> The completion time for the 1 thread case (~7.0s) is slightly less than for the 2 thread case (~7.7s), but the two-thread case did twice as much total work during the get phase. Thus the two-thread case achieved nearly 2x parallel speedup for the get phase on two cores, which is very good.

双线程时get阶段的工作量是单线程时的两倍。这主要是因为put阶段，双线程每个线程put NKEYS/2个entry，单线程一共put NKEYS个entry，两者相同；get阶段每个线程都遍历一次数组中的链表，所以get阶段双线程工作量大致是单线程的两倍。这里也可以看到，O(n)的查找速度使得get阶段的时间远远大于O(1)的put阶段。

为什么missing？
---
当两个线程同时对同一个bucket进行put时，会导致先put的线程的entry丢失

解决missing
---
很简单，在写之前拿一下锁就行了

全局声明一个锁：
```
pthread_mutex_t lock;
```
main中初始化：
```
pthread_mutex_init(&lock,NULL);
```
put之前锁一下，put之后解锁：
```
static 
void put(int key, int value)
{
  int i = key % NBUCKET;
  pthread_mutex_lock(&lock);
  insert(key, value, &table[i], table[i]);
  pthread_mutex_unlock(&lock);
}
```
这里注意一定要在调用insert前锁住，因为这时访问了table[i]，而这个量恰恰是要锁住的
得到结果：
双线程：
```
1: put time = 0.020500
0: put time = 0.021094
1: get time = 18.934134
1: 0 keys missing
0: get time = 18.998906
0: 0 keys missing
completion time = 19.020445
```
单线程：
```
0: put time = 0.008679
0: get time = 21.121396
0: 0 keys missing
completion time = 21.130402
```
