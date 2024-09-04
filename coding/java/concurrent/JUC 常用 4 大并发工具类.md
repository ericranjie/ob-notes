# 

Linux爱好者

 _2021年09月24日 11:50_

↓推荐关注↓

![](http://mmbiz.qpic.cn/mmbiz_png/IgylNib7ZE2L6Kv5EcGWb5mYVbzlgzP2R6qdGee7kZ03T7nia7rtkSnKvEvEOwnGxX4Zf2VVrlQNwWVrLlEhn4OQ/300?wx_fmt=png&wxfrom=19)

**Go开发大全**

点击获取6万star的Go开源库。[ Go开发大全 ] 日常分享 Go, 云原生、k8s、Docker和微服务方面的技术文章和行业动态。

21篇原创内容

公众号

  **什么是JUC?**

JUC就是java.util.concurrent包,这个包俗称JUC,里面都是解决并发问题的一些东西

该包的位置位于java下面的rt.jar包下面

## **4大常用并发工具类:**

CountDownLatch

CyclicBarrier

Semaphore

ExChanger

## **CountDownLatch:**

CountDownLatch,俗称闭锁,作用是类似加强版的Join,是让一组线程等待其他的线程完成工作以后才执行

就比如在启动框架服务的时候,我们主线程需要在环境线程初始化完成之后才能启动,这时候我们就可以实现使用CountDownLatch来完成

`/**        * Constructs a {@code CountDownLatch} initialized with the given count.        *        * @param count the number of times {@link #countDown} must be invoked        *        before threads can pass through {@link #await}        * @throws IllegalArgumentException if {@code count} is negative        */       public CountDownLatch(int count) {           if (count < 0) throw new IllegalArgumentException("count < 0");           this.sync = new Sync(count);       }   `

在源码中可以看到,创建CountDownLatch时,需要传入一个int类型的参数,将决定在执行次扣减之后,等待的线程被唤醒

![Image](https://mmbiz.qpic.cn/mmbiz_png/1QxwhpDy7ia04oPHkc1mY76tdSW4uyt2ichGkSjHPCiaekgUNiaibncfzOGFgUjyJdb6907mQgnhZHj6kibF3OIqpFHQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

通过这个类图就可以知道其实CountDownLatch并没有多少东西

方法介绍:

CountDownLatch:初始化方法

await:等待方法,同时带参数的是超时重载方法

countDown:每执行一次,计数器减一,就是初始化传入的数字,也代表着一个线程完成了任务

getCount:获取当前值

toString:这个就不用说了

里面的Sync是一个内部类,外面的方法其实都是操作这个内部类的,这个内部类继承了AQS,实现的标准方法,AQS将在后面的章节写

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主线程中创建CountDownLatch(3),然后主线程await阻塞,然后线程A,B,C各自完成了任务,调用了countDown,之后,每个线程调用一次计数器就会减一,初始是3,然后A线程调用后变成2,B线程调用后变成1,C线程调用后,变成0,这时就会唤醒正在await的主线程,然后主线程继续执行

说一千道一万,不如代码写几行,上代码:

休眠工具类,之后的代码都会用到

`package org.dance.tools;      import java.util.concurrent.TimeUnit;      /**    * 类说明：线程休眠辅助工具类    */   public class SleepTools {          /**        * 按秒休眠        * @param seconds 秒数        */       public static final void second(int seconds) {           try {               TimeUnit.SECONDS.sleep(seconds);           } catch (InterruptedException e) {           }       }          /**        * 按毫秒数休眠        * @param seconds 毫秒数        */       public static final void ms(int seconds) {           try {               TimeUnit.MILLISECONDS.sleep(seconds);           } catch (InterruptedException e) {           }       }   }   `

`package org.dance.day2.util;      import org.dance.tools.SleepTools;      import java.util.concurrent.CountDownLatch;      /**    * CountDownLatch的使用,有五个线程,6个扣除点    * 扣除完成后主线程和业务线程,才能执行工作    *  扣除点一般都是大于等于需要初始化的线程的    * @author ZYGisComputer    */   public class UseCountDownLatch {          /**        * 设置为6个扣除点        */       static CountDownLatch countDownLatch = new CountDownLatch(6);          /**        * 初始化线程        */       private static class InitThread implements Runnable {              @Override           public void run() {                  System.out.println("thread_" + Thread.currentThread().getId() + " ready init work .....");                  // 执行扣减 扣减不代表结束               countDownLatch.countDown();                  for (int i = 0; i < 2; i++) {                   System.out.println("thread_" + Thread.currentThread().getId() + ".....continue do its work");               }              }       }          /**        * 业务线程        */       private static class BusiThread implements Runnable {              @Override           public void run() {                  // 业务线程需要在等初始化完毕后才能执行               try {                   countDownLatch.await();                   for (int i = 0; i < 3; i++) {                       System.out.println("BusiThread " + Thread.currentThread().getId() + " do business-----");                   }               } catch (InterruptedException e) {                   e.printStackTrace();               }           }       }          public static void main(String[] args) {              // 创建单独的初始化线程           new Thread(){               @Override               public void run() {                   SleepTools.ms(1);                   System.out.println("thread_" + Thread.currentThread().getId() + " ready init work step 1st.....");                   // 扣减一次                   countDownLatch.countDown();                   System.out.println("begin stop 2nd.....");                   SleepTools.ms(1);                   System.out.println("thread_" + Thread.currentThread().getId() + " ready init work step 2nd.....");                   // 扣减一次                   countDownLatch.countDown();                  }           }.start();           // 启动业务线程           new Thread(new BusiThread()).start();           // 启动初始化线程           for (int i = 0; i <= 3; i++) {               new Thread(new InitThread()).start();           }           // 主线程进入等待           try {               countDownLatch.await();               System.out.println("Main do ites work.....");           } catch (InterruptedException e) {               e.printStackTrace();           }          }      }   `

返回结果:

`thread_13 ready init work .....   thread_13.....continue do its work   thread_13.....continue do its work   thread_14 ready init work .....   thread_14.....continue do its work   thread_14.....continue do its work   thread_15 ready init work .....   thread_15.....continue do its work   thread_11 ready init work step 1st.....   begin stop 2nd.....   thread_16 ready init work .....   thread_16.....continue do its work   thread_16.....continue do its work   thread_15.....continue do its work   thread_11 ready init work step 2nd.....   Main do ites work.....   BusiThread 12 do business-----   BusiThread 12 do business-----   BusiThread 12 do business-----   `

通过返回结果就可以很直接的看到业务线程是在初始化线程完全跑完之后,才开始执行的

## **CyclicBarrier:**

CyclicBarrier,俗称栅栏锁,作用是让一组线程到达某个屏障,被阻塞,一直到组内的最后一个线程到达,然后屏障开放,接着,所有的线程继续运行

这个感觉和CountDownLatch有点相似,但是其实是不一样的,所谓的差别,将在下面详解

CyclicBarrier的构造参数有两个

`/**        * Creates a new {@code CyclicBarrier} that will trip when the        * given number of parties (threads) are waiting upon it, and        * does not perform a predefined action when the barrier is tripped.        *        * @param parties the number of threads that must invoke {@link #await}        *        before the barrier is tripped        * @throws IllegalArgumentException if {@code parties} is less than 1        */       public CyclicBarrier(int parties) {           this(parties, null);       }   `

`/**        * Creates a new {@code CyclicBarrier} that will trip when the        * given number of parties (threads) are waiting upon it, and which        * will execute the given barrier action when the barrier is tripped,        * performed by the last thread entering the barrier.        *        * @param parties the number of threads that must invoke {@link #await}        *        before the barrier is tripped        * @param barrierAction the command to execute when the barrier is        *        tripped, or {@code null} if there is no action        * @throws IllegalArgumentException if {@code parties} is less than 1        */       public CyclicBarrier(int parties, Runnable barrierAction) {           if (parties <= 0) throw new IllegalArgumentException();           this.parties = parties;           this.count = parties;           this.barrierCommand = barrierAction;       }   `

很明显能感觉出来,上面的构造参数调用了下面的构造参数,是一个构造方法重载

首先这个第一个参数也树Int类型的,传入的是执行线程的个数,这个数量和CountDownLatch不一样,这个数量是需要和线程数量吻合的,CountDownLatch则不一样,CountDownLatch可以大于等于,而CyclicBarrier只能等于,然后是第二个参数,第二个参数是barrierAction,这个参数是当屏障开放后,执行的任务线程,如果当屏障开放后需要执行什么任务,可以写在这个线程中

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主线程创建CyclicBarrier(3,barrierAction),然后由线程开始执行,线程A,B执行完成后都调用了await,然后他们都在一个屏障前阻塞者,需要等待线程C也,执行完成,调用await之后,然后三个线程都达到屏障后,屏障开放,然后线程继续执行,并且barrierAction在屏障开放的一瞬间也开始执行

上代码:

`package org.dance.day2.util;      import org.dance.tools.SleepTools;      import java.util.Map;   import java.util.Random;   import java.util.concurrent.BrokenBarrierException;   import java.util.concurrent.ConcurrentHashMap;   import java.util.concurrent.CyclicBarrier;      /**    * CyclicBarrier的使用    *    * @author ZYGisComputer    */   public class UseCyclicBarrier {          /**        * 存放子线程工作结果的安全容器        */       private static ConcurrentHashMap<String, Long> resultMap = new ConcurrentHashMap<>();          private static CyclicBarrier cyclicBarrier = new CyclicBarrier(5,new CollectThread());          /**        * 结果打印线程        * 用来演示CyclicBarrier的第二个参数,barrierAction        */       private static class CollectThread implements Runnable {              @Override           public void run() {                  StringBuffer result = new StringBuffer();                  for (Map.Entry<String, Long> workResult : resultMap.entrySet()) {                   result.append("[" + workResult.getValue() + "]");               }                  System.out.println("the result = " + result);               System.out.println("do other business.....");              }       }          /**        * 工作子线程        * 用于CyclicBarrier的一组线程        */       private static class SubThread implements Runnable {              @Override           public void run() {                  // 获取当前线程的ID               long id = Thread.currentThread().getId();                  // 放入统计容器中               resultMap.put(String.valueOf(id), id);                  Random random = new Random();                  try {                   if (random.nextBoolean()) {                       Thread.sleep(1000 + id);                       System.out.println("Thread_"+id+"..... do something");                   }                   System.out.println(id+" is await");                   cyclicBarrier.await();                   Thread.sleep(1000+id);                   System.out.println("Thread_"+id+".....do its business");               } catch (InterruptedException e) {                   e.printStackTrace();               } catch (BrokenBarrierException e) {                   e.printStackTrace();               }              }       }          public static void main(String[] args) {              for (int i = 0; i <= 4; i++) {               Thread thread = new Thread(new SubThread());               thread.start();           }          }      }   `

返回结果:

`11 is await   14 is await   15 is await   Thread_12..... do something   12 is await   Thread_13..... do something   13 is await   the result = [11][12][13][14][15]   do other business.....   Thread_11.....do its business   Thread_12.....do its business   Thread_13.....do its business   Thread_14.....do its business   Thread_15.....do its business   `

通过返回结果可以看出前面的11 14 15三个线程没有进入if语句块,在执行到await的时候进入了等待,而另外12 13两个线程进入到了if语句块当中,多休眠了1秒多,然后当5个线程同时到达await的时候,屏障开放,执行了barrierAction线程,然后线程组继续执行

解释一下CountDownLatch和CyclicBarrier的却别吧!

首先就是CountDownLatch的构造参数传入的数量一般都是大于等于线程,数量的,因为他是有第三方控制的,可以扣减多次,然后就是CyclicBarrier的构造参数第一个参数传入的数量一定是等于线程的个数的,因为他是由一组线程自身控制的

区别

CountDownLatch　　CyclicBarrier

控制　　    第三方控制　　　　  自身控制

传入数量　　大于等于线程数量    等于线程数量

## **Semaphore:**

Semaphore,俗称信号量,作用于控制同时访问某个特定资源的线程数量,用在流量控制

一说特定资源控制,那么第一时间就想到了数据库连接..

之前用等待超时模式写了一个数据库连接池,打算用这个Semaphone也写一个

`/**        * Creates a {@code Semaphore} with the given number of        * permits and nonfair fairness setting.        *        * @param permits the initial number of permits available.        *        This value may be negative, in which case releases        *        must occur before any acquires will be granted.        */       public Semaphore(int permits) {           sync = new NonfairSync(permits);       }   `

在源码中可以看到在构建Semaphore信号量的时候,需要传入许可证的数量,这个数量就是资源的最大允许的访问的线程数

接下里用信号量实现一个数据库连接池

连接对象

`package org.dance.day2.util.pool;      import org.dance.tools.SleepTools;      import java.sql.*;   import java.util.Map;   import java.util.Properties;   import java.util.concurrent.Executor;      /**    * 数据库连接    * @author ZYGisComputer    */   public class SqlConnection implements Connection {          /**        * 获取数据库连接        * @return        */       public static final Connection fetchConnection(){           return new SqlConnection();       }          @Override       public void commit() throws SQLException {           SleepTools.ms(70);       }          @Override       public Statement createStatement() throws SQLException {           SleepTools.ms(1);           return null;       }          @Override       public PreparedStatement prepareStatement(String sql) throws SQLException {           return null;       }          @Override       public CallableStatement prepareCall(String sql) throws SQLException {           return null;       }          @Override       public String nativeSQL(String sql) throws SQLException {           return null;       }          @Override       public void setAutoCommit(boolean autoCommit) throws SQLException {          }          @Override       public boolean getAutoCommit() throws SQLException {           return false;       }          @Override       public void rollback() throws SQLException {          }          @Override       public void close() throws SQLException {          }          @Override       public boolean isClosed() throws SQLException {           return false;       }          @Override       public DatabaseMetaData getMetaData() throws SQLException {           return null;       }          @Override       public void setReadOnly(boolean readOnly) throws SQLException {          }          @Override       public boolean isReadOnly() throws SQLException {           return false;       }          @Override       public void setCatalog(String catalog) throws SQLException {          }          @Override       public String getCatalog() throws SQLException {           return null;       }          @Override       public void setTransactionIsolation(int level) throws SQLException {          }          @Override       public int getTransactionIsolation() throws SQLException {           return 0;       }          @Override       public SQLWarning getWarnings() throws SQLException {           return null;       }          @Override       public void clearWarnings() throws SQLException {          }          @Override       public Statement createStatement(int resultSetType, int resultSetConcurrency) throws SQLException {           return null;       }          @Override       public PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency) throws SQLException {           return null;       }          @Override       public CallableStatement prepareCall(String sql, int resultSetType, int resultSetConcurrency) throws SQLException {           return null;       }          @Override       public Map<String, Class<?>> getTypeMap() throws SQLException {           return null;       }          @Override       public void setTypeMap(Map<String, Class<?>> map) throws SQLException {          }          @Override       public void setHoldability(int holdability) throws SQLException {          }          @Override       public int getHoldability() throws SQLException {           return 0;       }          @Override       public Savepoint setSavepoint() throws SQLException {           return null;       }          @Override       public Savepoint setSavepoint(String name) throws SQLException {           return null;       }          @Override       public void rollback(Savepoint savepoint) throws SQLException {          }          @Override       public void releaseSavepoint(Savepoint savepoint) throws SQLException {          }          @Override       public Statement createStatement(int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException {           return null;       }          @Override       public PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException {           return null;       }          @Override       public CallableStatement prepareCall(String sql, int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException {           return null;       }          @Override       public PreparedStatement prepareStatement(String sql, int autoGeneratedKeys) throws SQLException {           return null;       }          @Override       public PreparedStatement prepareStatement(String sql, int[] columnIndexes) throws SQLException {           return null;       }          @Override       public PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException {           return null;       }          @Override       public Clob createClob() throws SQLException {           return null;       }          @Override       public Blob createBlob() throws SQLException {           return null;       }          @Override       public NClob createNClob() throws SQLException {           return null;       }          @Override       public SQLXML createSQLXML() throws SQLException {           return null;       }          @Override       public boolean isValid(int timeout) throws SQLException {           return false;       }          @Override       public void setClientInfo(String name, String value) throws SQLClientInfoException {          }          @Override       public void setClientInfo(Properties properties) throws SQLClientInfoException {          }          @Override       public String getClientInfo(String name) throws SQLException {           return null;       }          @Override       public Properties getClientInfo() throws SQLException {           return null;       }          @Override       public Array createArrayOf(String typeName, Object[] elements) throws SQLException {           return null;       }          @Override       public Struct createStruct(String typeName, Object[] attributes) throws SQLException {           return null;       }          @Override       public void setSchema(String schema) throws SQLException {          }          @Override       public String getSchema() throws SQLException {           return null;       }          @Override       public void abort(Executor executor) throws SQLException {          }          @Override       public void setNetworkTimeout(Executor executor, int milliseconds) throws SQLException {          }          @Override       public int getNetworkTimeout() throws SQLException {           return 0;       }          @Override       public <T> T unwrap(Class<T> iface) throws SQLException {           return null;       }          @Override       public boolean isWrapperFor(Class<?> iface) throws SQLException {           return false;       }   }   `

连接池对象

`package org.dance.day2.util.pool;      import java.sql.Connection;   import java.util.ArrayList;   import java.util.HashSet;   import java.util.Iterator;   import java.util.LinkedList;   import java.util.concurrent.Semaphore;      /**    * 使用信号量控制数据库的链接和释放    *    * @author ZYGisComputer    */   public class DBPoolSemaphore {          /**        * 池容量        */       private final static int POOL_SIZE = 10;          /**        * useful 代表可用连接        * useless 代表已用连接        *  为什么要使用两个Semaphore呢?是因为,在连接池中不只有连接本身是资源,空位也是资源,也需要记录        */       private final Semaphore useful, useless;          /**        * 连接池        */       private final static LinkedList<Connection> POOL = new LinkedList<>();          /**        * 使用静态块初始化池        */       static {           for (int i = 0; i < POOL_SIZE; i++) {               POOL.addLast(SqlConnection.fetchConnection());           }       }          public DBPoolSemaphore() {           // 初始可用的许可证等于池容量           useful = new Semaphore(POOL_SIZE);           // 初始不可用的许可证容量为0           useless = new Semaphore(0);       }          /**        * 获取数据库连接        *        * @return 连接对象        */       public Connection takeConnection() throws InterruptedException {           // 可用许可证减一           useful.acquire();           Connection connection;           synchronized (POOL) {               connection = POOL.removeFirst();           }           // 不可用许可证数量加一           useless.release();           return connection;       }          /**        * 释放链接        *        * @param connection 连接对象        */       public void returnConnection(Connection connection) throws InterruptedException {           if(null!=connection){               // 打印日志               System.out.println("当前有"+useful.getQueueLength()+"个线程等待获取连接,,"                       +"可用连接有"+useful.availablePermits()+"个");               // 不可用许可证减一               useless.acquire();               synchronized (POOL){                   POOL.addLast(connection);               }               // 可用许可证加一               useful.release();           }       }      }   `

测试类:

`package org.dance.day2.util.pool;      import org.dance.tools.SleepTools;      import java.sql.Connection;   import java.util.Random;      /**    * 测试Semaphore    * @author ZYGisComputer    */   public class UseSemaphore {          /**        * 连接池        */       public static final DBPoolSemaphore pool = new DBPoolSemaphore();          private static class BusiThread extends Thread{           @Override           public void run() {               // 随机数工具类 为了让每个线程持有连接的时间不一样               Random random = new Random();               long start = System.currentTimeMillis();               try {                   Connection connection = pool.takeConnection();                   System.out.println("Thread_"+Thread.currentThread().getId()+                           "_获取数据库连接耗时["+(System.currentTimeMillis()-start)+"]ms.");                   // 模拟使用连接查询数据                   SleepTools.ms(100+random.nextInt(100));                   System.out.println("查询数据完成归还连接");                   pool.returnConnection(connection);               } catch (InterruptedException e) {                   e.printStackTrace();               }           }       }          public static void main(String[] args) {           for (int i = 0; i < 50; i++) {               BusiThread busiThread = new BusiThread();               busiThread.start();           }       }      }   `

测试返回结果:

`Thread_11_获取数据库连接耗时[0]ms.   Thread_12_获取数据库连接耗时[0]ms.   Thread_13_获取数据库连接耗时[0]ms.   Thread_14_获取数据库连接耗时[0]ms.   Thread_15_获取数据库连接耗时[0]ms.   Thread_16_获取数据库连接耗时[0]ms.   Thread_17_获取数据库连接耗时[0]ms.   Thread_18_获取数据库连接耗时[0]ms.   Thread_19_获取数据库连接耗时[0]ms.   Thread_20_获取数据库连接耗时[0]ms.   查询数据完成归还连接   当前有40个线程等待获取连接,,可用连接有0个   Thread_21_获取数据库连接耗时[112]ms.   查询数据完成归还连接   ...................   查询数据完成归还连接   当前有2个线程等待获取连接,,可用连接有0个   Thread_59_获取数据库连接耗时[637]ms.   查询数据完成归还连接   当前有1个线程等待获取连接,,可用连接有0个   Thread_60_获取数据库连接耗时[660]ms.   查询数据完成归还连接   当前有0个线程等待获取连接,,可用连接有0个   查询数据完成归还连接   ...................   当前有0个线程等待获取连接,,可用连接有8个   查询数据完成归还连接   当前有0个线程等待获取连接,,可用连接有9个   `

通过执行结果可以很明确的看到,一上来就有10个线程获取到了连接,,然后后面的40个线程进入阻塞,然后只有释放链接之后,等待的线程就会有一个拿到,然后越后面的线程等待的时间就越长,然后一直到所有的线程执行完毕

最后打印的可用连接有九个不是因为少了一个是因为在释放之前打印的,不是错误

从结果中可以看到,我们对连接池中的资源的到了控制,这就是信号量的流量控制

## **Exchanger:**

Exchanger,俗称交换器,用于在线程之间交换数据,但是比较受限,因为只能两个线程之间交换数据

`/**        * Creates a new Exchanger.        */       public Exchanger() {           participant = new Participant();       }   `

这个构造函数没有什么好说的,也没有入参,只有在创建的时候指定一下需要交换的数据的泛型即可,下面看代码

`package org.dance.day2.util;      import java.util.HashSet;   import java.util.Set;   import java.util.concurrent.Exchanger;      /**    * 线程之间交换数据    * @author ZYGisComputer    */   public class UseExchange {          private static final Exchanger<Set<String>> exchanger = new Exchanger<>();          public static void main(String[] args) {              new Thread(){               @Override               public void run() {                   Set<String> aSet = new HashSet<>();                   aSet.add("A");                   aSet.add("B");                   aSet.add("C");                   try {                       Set<String> exchange = exchanger.exchange(aSet);                       for (String s : exchange) {                           System.out.println("aSet"+s);                       }                   } catch (InterruptedException e) {                       e.printStackTrace();                   }               }           }.start();              new Thread(){               @Override               public void run() {                   Set<String> bSet = new HashSet<>();                   bSet.add("1");                   bSet.add("2");                   bSet.add("3");                   try {                       Set<String> exchange = exchanger.exchange(bSet);                       for (String s : exchange) {                           System.out.println("bSet"+s);                       }                   } catch (InterruptedException e) {                       e.printStackTrace();                   }               }           }.start();          }      }   `

执行结果:

`bSetA   bSetB   bSetC   aSet1   aSet2   aSet3   `

通过执行结果可以清晰的看到,两个线程中的数据发生了交换,这就是Exchanger的线程数据交换了

以上就是JUC的4大常用并发工具类了

  

> 转自：彼岸舞
> 
> 链接：www.cnblogs.com/flower-dance/p/13714006.html

  

- EOF -

推荐阅读  点击标题可跳转

1、[Linux 中断，本质到底是什么？](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666557696&idx=3&sn=894de15bf8d7c41735637c14f0d8948a&chksm=80dcb5abb7ab3cbd9e0d89a52ea924dfe4a9f9a6923fed751ef39668324661df59947bcf50c2&scene=21#wechat_redirect)

2、[Linux 虚拟网络设备之 bridge](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666557662&idx=3&sn=e44279ed1e0367eef0161a468df212cd&chksm=80dcb475b7ab3d63a6f840e39a4a2e6bc736fa4d68750ae75b5a11f971f464e122188b40a9a6&scene=21#wechat_redirect)

3、[利用 Linux 查找重复文件](http://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666557648&idx=3&sn=345d2f5d13c52018ccc431d31f1b6e0d&chksm=80dcb47bb7ab3d6da5c015d07c12047d45de1891acd1f855a101551d3f155990cffeab246b58&scene=21#wechat_redirect)

  

看完本文有收获？请分享给更多人  

推荐关注「Linux 爱好者」，提升Linux技能

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=19)

**Linux爱好者**

点击获取《每天一个Linux命令》系列和精选Linux技术资源。「Linux爱好者」日常分享 Linux/Unix 相关内容，包括：工具资源、使用技巧、课程书籍等。

75篇原创内容

公众号

点赞和在看就是最大的支持❤️

Reads 527

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3sjicd8JxDra10FRIqT54Zke2sfhibTDdtdnVhv5Qh3wLHZmKPjiaD7piahMAzIH6Cnltd1Nco17Ihjw/300?wx_fmt=png&wxfrom=18)

Linux爱好者

1ShareWow

Comment

Comment

**Comment**

暂无留言