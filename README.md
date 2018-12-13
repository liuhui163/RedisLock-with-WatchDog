## redis分布式锁+WatchDog任务自动延时解决方案

- 分布式锁一般有三种实现方式：1. 数据库乐观锁；2. 基于Redis的分布式锁；3. 基于ZooKeeper的分布式锁
- 本次实现是以redis来实现分布式锁的，redis速度非常之快，官方测试的数据是读写性能可以达到10万每秒，如此优越的性能让大多数人选择用它作为分布式锁的实现

- 首先，为了确保分布式锁可用，我们至少要确保锁的实现同时满足以下四个条件：

- 1.互斥性。在任意时刻，只有一个客户端能持有锁。
- 2.不会发生死锁。即使有一个客户端在持有锁的期间崩溃而没有主动解锁，也能保证后续其他客户端能加锁。
- 3.具有容错性。只要大部分的Redis节点正常运行，客户端就可以加锁和解锁。
- 4.解铃还须系铃人。加锁和解锁必须是同一个客户端，客户端自己不能把别人加的锁给解了。

第一点和第二点是利用redis命令 setNX、get和getSet配合完成的 （锁key对应的value存的是redis里过期时间的时间戳）<br/>
	1.如果是初次加锁、锁过期或者线程执行完加锁代码解锁后，自然setNX成功，并设置过期时间。 <br/>
	2.如果setNX失败，通过redis命令get获取到锁key对应的value，通过判断这个value跟redis时间对比， <br/>
	  如果已经过期还未释放锁，那么value会小于redis时间，然后再执行getSet命令，获取返回值， <br/>
	  通过对比返回值跟上面的value是否一致，能判断是不是众多线程中第一个getSet成功的， <br/>
	  如果一致，那么让这个线程获取锁，并设置过期时间，否则返回失败，满足先到先得原则。 <br/>
第三点可以用redis 主从、sentinel、cluster 三种集群方式来提高容错性  <br/>
第四点在解锁的时候，1.先拿到加锁成功返回的value 2.通过get命令拿到存储在redis里面锁key对应的value  <br/>
      然后对比这两个值来判定是不是同一个线程执行解锁，如果是，那么将key删除，解锁成功，否则解锁失败  <br/>
	  
- 通过上面的做法，锁已经可用了，但是我们还要思考一个问题： <br/>
  如果业务的运行时间因为某些原因超出了我们的估算，那么我们应该怎么办？难道直接让锁过期？<br/>
  这样显然不行，因为如果锁过期了的话，那么就会有其他线程获取到锁，这样会让业务并行执行， <br/>
  从而失去了我们使用分布式锁的初衷：在同一时间只有一个节点内的一个线程能执行业务。 <br/>
  
- 因此本次实现的第二个重点来了，WatchDog监控狗的实现： <br/>
  首先我们做一个切面，将所有含有注解@Lock的方法都进行切面处理，具体步骤如下: <br/>
  1.执行lock方法加锁，如果获取锁失败直接抛出异常，如果成功进行第二部 <br/>
  2.将真正的业务执行逻辑放入线程池中，返回一个Future对象 <br/>
  3.这时候我们将用到netty实现的时间轮算法定时器HashedWheelTimer， <br/>
	首先根据注解设置的延长次数将延时任务注册到时间轮中,注册规则:次数*超时时间*9/10(默认9/10),次数依次递减 <br/>
	我们可以在@Lock注解里面自己定义延时权重，默认是9/10 <br/>
	TimerTask具体的任务如下: <br/>
	(1).判断Timeout对象（netty的具体实现是HashedWheelTimeout）是否已经取消或者任务是否已经完成 （Future.isDone()） <br/>
	(2).如果已取消或业务任务已完成不做任何操作，否则将锁的key延长时间，具体的实现稍后再说 <br/>
	(3).将锁延长时间的方法会返回成功或者失败，如果失败，那么我们将任务取消，出现这种情况时，有以下原因： <br/>
	  因为业务超时被其他锁占用或者值被改变,或者值为null;解决方案：1.确认是否有对锁的key做过set 2.调整@Lock里面delayWight的值以确保在锁超时之前完成延时的动作 <br/>
  4.注册到时间轮之后会返回一个Timeout对象，将这个对象放到队列中，稍后有用 <br/>
  5.进行一个空循环，条件是(!Future.isDone())，当任务完成后跳出循环，这时候遍历延长任务队列，将能取消的定时任务取消，这个过程放在其他线程执行 <br/>
    最后返回任务执行的结果(Future.get())。
	
- 最后，再说一下延长锁时间的实现： <br/>
  首先通过get命令拿到锁key对应的value，然后判断是否与加锁时的value相等，如果不相等，锁被其他线程占有或key被设置了其他值，返回失败。 <br/>
  如果相等，我们将key对应的value设置为延长锁时间之后的value，然后通过redis的pttl命令获取锁的key剩余过期时间，
  然后重新设置过期时间为剩余时间+初始设置的过期时间。



