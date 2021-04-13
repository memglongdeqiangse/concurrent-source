# Lock接口研读

### 1.Lock出现的目的

lock接口位于 java.util.concurrent.locks 包下，它的方法有六个，其中lock()和unlock()方法很好理解，即加锁和解锁，那么其他的呢？

```java
public interface Lock {	
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

关于Jdk中的Lock与synchronized关键字貌似有相同的作用，是否重复造轮子了？

答案是：NO！

#### 1.1死锁的知识（来自王宝令老师的[Java并发实战](<https://time.geekbang.org/column/article/85001>)）

**死锁（deadlock）：集合中的每一个进程都在等待只能由本集合中的其他进程才能引发的事件，那么该组进程是死锁的。**

**也就是说互相竞争某一资源的线程因为互相等待，从而永久阻塞，无法解开。**

举个例子：线程t1请求资源X，线程t2请求资源Y，而恰恰X被t2持有，Y被t1持有，t1和t2谁都拿不到，从而永久地阻塞。。。

**死锁发生的四个条件**：

1. 互斥，共享资源 X 和 Y 只能被一个线程占用；
2. 占有且等待，线程 T1 已经取得共享资源 X，在等待共享资源 Y 的时候，不释放共享资源 X；
3. 不可抢占，其他线程不能强行抢占线程 T1 占有的资源；
4. 循环等待，线程 T1 等待线程 T2 占有的资源，线程 T2 等待线程 T1 占有的资源，就是循环等待。

**这四个条件同时发生才发生死锁，同时，我们只要打破其中的一个就可以破坏死锁。**

1. 其中，互斥这个条件我们没有办法破坏，因为我们用锁为的就是互斥。

2. 对于“占有且等待”这个条件，我们可以一次性申请所有的资源，这样就不存在等待了。（如添加一个allocator角色分配所有资源给申请者，释放时则释放所有资源）

3. 对于“不可抢占”这个条件，占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源，这样不可抢占这个条件就破坏掉了。（**核心是要能够主动释放它占有的资源，这一点 synchronized 是做不到的。原因是 synchronized 申请资源的时候，如果申请不到，线程直接进入阻塞状态了，而线程进入阻塞状态，啥都干不了，也释放不了线程已经占有的资源。**）

   ```java
   
   class Allocator {
     private List<Object> als =
       new ArrayList<>();
     // 一次性申请所有资源
     synchronized boolean apply(
       Object from, Object to){
       if(als.contains(from) ||
            als.contains(to)){
         return false;  
       } else {
         als.add(from);
         als.add(to);  
       }
       return true;
     }
     // 归还资源
     synchronized void free(
       Object from, Object to){
       als.remove(from);
       als.remove(to);
     }
   }
   
   class Account {
     // actr应该为单例
     private Allocator actr;
     private int balance;
     // 转账
     void transfer(Account target, int amt){
       // 一次性申请转出账户和转入账户，直到成功
       while(!actr.apply(this, target));
       try{
         // 锁定转出账户
         synchronized(this){              
           // 锁定转入账户
           synchronized(target){           
             if (this.balance > amt){
               this.balance -= amt;
               target.balance += amt;
             }
           }
         }
       } finally {
         actr.free(this, target)
       }
     } 
   }
   ```

4. 对于“循环等待”这个条件，可以靠按序申请资源来预防。所谓按序申请，是指资源是有线性顺序的，申请的时候可以先申请资源序号小的，再申请资源序号大的，这样线性化后自然就不存在循环了。

   ```java
   
   class Account {
     private int id;
     private int balance;
     // 转账
     void transfer(Account target, int amt){
       Account left = this        ①
       Account right = target;    ②
       if (this.id > target.id) { ③
         left = target;           ④
         right = this;            ⑤
       }                          ⑥
       // 锁定序号小的账户
       synchronized(left){
         // 锁定序号大的账户
         synchronized(right){ 
           if (this.balance > amt){
             this.balance -= amt;
             target.balance += amt;
           }
         }
       }
     } 
   }
   ```


#### 1.2 synchronized 的缺点

synchronized 申请资源的时候，如果申请不到，线程直接进入阻塞状态了。

要解决的三个方面：

1. 能够响应中断。synchronized 的问题是，持有锁 A 后，如果尝试获取锁 B 失败，那么线程就进入阻塞状态，一旦发生死锁，就没有任何机会来唤醒阻塞的线程。但如果阻塞状态的线程能够响应中断信号，也就是说当我们给阻塞的线程发送中断信号的时候，能够唤醒它，那它就有机会释放曾经持有的锁 A。这样就破坏了不可抢占条件了。
2. 支持超时。如果线程在一段时间之内没有获取到锁，不是进入阻塞状态，而是返回一个错误，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件。
3. 非阻塞地获取锁。如果尝试获取锁失败，并不进入阻塞状态，而是直接返回，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件。

这也就是Lock出现的原因，这三个方法：

```java
// 支持中断的API, 线程在请求lock并被阻塞时，可以响应中断
void lockInterruptibly() 
  throws InterruptedException;
// 支持超时的API
boolean tryLock(long time, TimeUnit unit) 
  throws InterruptedException;
// 支持非阻塞获取锁的API,尝试是否能获得锁，不管能不能获得锁都会立刻返回，返回结果为是否获得锁
boolean tryLock();
```



### 2.Java中断

先贴个[链接](https://www.zhihu.com/question/41048032/answer/89431513)，建议详细读一读！！！

举个例子，一个线程t1调用 Thread.sleep() 的时候会抛出一个 InterruptedException，要求你处理，如果其他线程t2 interrupt了t1可以在t1的catch中用 isInterrupted() 来查看是否被中断了。

也就是说，我们的锁支持线程在阻塞时可以响应中断的话，那么释放它的资源就可以破坏不可抢占条件，避免死锁了。

### 3. newCondition()

剩下的 newCondition() 方法是来解决同步问题，它的返回值是它同包下的Condition接口，接下来[分析Condition接口](./Condition.md)。

### 参考链接

<https://time.geekbang.org/column/intro/159>

<https://www.zhihu.com/question/36771163/answer/68974735>

<https://www.zhihu.com/question/41048032/answer/89431513>

