# java 并发相关

以下部分总结自  `深入理解java虚拟机` 

## 硬件效率与一致性

硬件层面为了提升 `cpu` 与 `主存` 通信效率，于是增加了一层 `高速缓存(cache)` ,计算前将即将要用到的数据写入 cache，计算后将数据从缓存写回主存。

但是随着多核时代的到来，cpu 不再只有 `一个`，而每个cpu 都有各自的 `cache`， 那么如果某一个变量同时存在于2个 `cache` 中，应该以哪一个为准？不过这个cpu 已经帮我们考虑了，读写主存都要根据一定的协议。但是某些情况下还是需要程序员自己在代码中指明。。

## java 内存模型

java 将规定所有变量都存 `主存` 中, 下图是 线程，主存，工作内存关系。

![java-memory-model.png](java-concurrent/java-memory-model.png)

内存间的操作分为

1. `lock` 锁定
2. `unlock` 
3. `read`   主存 -> 工作内存
4. `load`   在工作内存 建变量的副本  
5. `use`    使用 `4` 的副本，执行业务逻辑
6. `assign` 将 `5` 的结果覆盖 `4` 的副本
7. `store`  存主存
8. `write`  将 `7` 的数据, 覆盖主存原值

## volatile

1. 保证每一次的读到必定是新的。 (`read`, `load`, `use` 和 `assign`, `store`, `write` 操作必须原子，中间不和被穿插)
2. 禁止指令重排序优化。 (字节码会增加 Memory Barrier, 保证，Barrier 前后不可穿越)

## java thread 实现

### 线程种类

1. 内核线程(轻量级进程)  系统帮你调度， 但是线程创建，销毁是系统调用代价高。
2. 用户线程  没有 `1` 的创建，销毁代价， 但是需要自己做cpu 时间片切片
3. 用户线程加轻量级进程

java 使用的是上面的 `1`

### 线程调度

1. 抢占式调度  系统分配cpu时间，线程会被随时暂停  可以配置 `线程优先级` 
2. 协同式调度  线程执行时间又线程本身控制（可能会出现自己因为bug 不切换，于是一直阻塞）

### 线程状态

1. 创建
2. 运行
3. 无限等待  
    
    1. Object.wait() 针对对象
    2. Thread.join() 
    3. LockSupport.park() 禁止线程接受调度

4. 期限等待  等待一段时间，或者唤醒动作的发生
    
    1. Thread.sleep()
    2. Object.wait(timeout)
    3. Thread.join(timeout)
    4. LockSupport.parkNanos()
    5. LockSupport.parkUntil()
 
5. 阻塞  表示等待获取一个排他锁
6. 结束    

### wait park 区别？

`wait` 针对对象，和 `notify`, `notifyAll` 一起用在 `synchronized` 代码块中。

[使用wait 的理由](http://blog.yemou.net/article/query/info/tytfjhfascvhzxcytp16) ：

    Synchronized这个关键字用于保护共享数据，阻止其他线程对共享数据的存取。
    但是这样程序的流程就很不灵活了，如何才能在当前线程还没退出Synchronized数据块时让其他线程也有机会访问共享数据呢？此时就用以上三个方法来灵活控制。 
    wait()方法使当前线程暂停执行并释放对象锁标志，`让其他线程可以进入Synchronized数据块`，当前线程被放入对象等待池中。
    当调用 notify()方法后，将从对象的等待池中移走一个任意的线程并放到锁标志等待池中，只有锁标志等待池中的线程能够获取锁标志, 如果锁标志等待池中没有线程，则notify()不起作用
    notifyAll()则从对象等待池中移走所有等待那个对象的线程并放到锁标志等待池中。

`park` 针对Thread，使Thread 进行阻塞处理。可以指定阻塞队列的目标对象，每次可以指定具体的线程唤醒


 
## 线程安全

1. 互斥同步（阻塞同步）（悲观并发策略）
    
    1. synchronized
        通过native的 monitorEnter 和 moniterExit 实现，如果资源对象已经被占有，则线程需要阻塞等待知道被释放.
    
    2. ReentrantLock
        比起 synchronized 增加了以下功能
            
            1. 等待可中断， 当持有锁的线程长期不释放锁，正在等待的线程可以选择放弃等待。
            2. 公平锁，    按照申请锁的顺序依次获得锁
            3. 绑定多个Condition  ， 多个地方wait，
    

2. 非阻塞同步 （基于冲突检测的乐观并发策略）

    原理：
    
    经验分析，大多数的情况下资源并没有被占有。
    
    直接进行操作，如果没有其他进程争用，则成功，如果冲突了，则不断重试(自旋), 免去将 thread wait(调用系统，慢)。


    依靠硬件级别的 CAS ，但无法完美处理 ABA 问题。
    

3. 无同步方案
    
    1. 函数式的代码，结果只被入参影响
    2. ThreadLocal

## 锁优化

1. 适应性自旋
    
    原理： 
    
    因为互斥同步对性能最大的影响是 阻塞的实现，`wait`, `notify` 都需要转入内核态完成，慢，成本高。
    
    由于大多数情况，不会长期持有锁，所以在用户态，稍稍死循环n次，check 锁是否已经释放。
    
    当然，如果还是失败，还是会 `wait`
    
    `适应性` 1.6 开始自旋时间由前一次在同一锁上的自旋时间及锁的拥有者的状态决定。
    
2. 锁消除

    原理：
    
    编译时优化，去掉必定在一个线程内 lock unlock 的行为
    
3. 轻量级锁
    
    原理：
    
    实践认为，大多数情况下没有多线程竞争，减少 `直接` 使用传统的重量级锁而造成的损失。
    
    实现方式就是 先用 上面的 自旋锁 和 CAS ，试图在用户态等一会儿，获得锁

4. 偏向锁
    
    原理：
    
    针对 synchronize ，在无竞争的情况下连 `CAS` 都不做, 仅仅比较对象上面的 MarkWork 是否和自己是同一线程


## 参考

http://www.infoq.com/cn/articles/java-se-16-synchronized
http://ifeve.com/java_lock_see/ 
http://my.oschina.net/u/140462/blog/490897
http://www.cnblogs.com/javaminer/p/3892288.html


## CLH 

http://blog.csdn.net/aesop_wubo/article/details/7533186

http://www.programering.com/a/MjM5gTNwATE.html

http://coderbee.net/index.php/concurrent/20131115/577


http://agapple.iteye.com/blog/970055


# 线程安全？

首先什么是线程安全。

线程安全 有3个要求

1. 可见性：一个线程对主内存的修改可以及时的被其他线程观察到。
2. 有序性：一个线程观察其他线程中的指令执行顺序，由于指令 重排序的存在，该观察结果一般杂乱无序。
3. 原子性：提供了互斥访问。

以上的解释：

1. 常见于 xx.size(), xx.contains() 要保证当前被写入的也要被发现。
2. 可以由volatile（禁止指令重排序）/synchronized（一个变量最多只能有一个线程对其lock）实现
3. 串行执行.

    所以，所谓的要让数据结构线程安全，其实要做的事情就是要保证以上3个要求，而实现以上3个要求的方法就是将原本要求并行的操作，想办法串行化。而串行化的方法就是在关键地方加各种加锁。(其实锁的实现(除了偏向锁和自旋锁)的实现底层是AQS，而AQS 的实现就是一个 FIFO 的Queue，so...)


## 总结下，平时常用的数据结构什么情况下会发生线程不安全？

对于一个变量来说，如果多线程多该变量操作，而每个线程（java的线程为轻量级进程）自己都会有一份该变量的拷贝，于是 就发生了不安全。

对于一个数组，多线程写？ 貌似多线程写是依赖 一个变量(比如 i)，那么又回到上面的问题了。

对于一个List，什么情况下会发生不安全？这可以参考 Java Collections 里面的synchronizedList, 限制了些什么来倒推。

具体看这个类，`java.util.Collections.SynchronizedCollection`, 然后会发现，他很暴力的吧所有的操作都加了锁。

之所以这么做是因为，list 内部实现就是一个数组，
数组关键就是要保证他的index 线程安全，但是index 的安全是依赖使用者的，但是多个线程没法保证，自己将使用的index都是不同，
比如 threadA 和 threadB 各自都认为当前index为0，同时add 一个 value，理想情况，list的结果应该是2，但是结果确实1.

所以解决这个问题的最简单的方法就是，将并发的问题专为同步的问题，全部加锁就ok了。。

对于一个Map，先看HashTable 他和SynchronizedCollection的实现类似，简单的将所有的操作都用synchronized 包起来。

而HashMap 则使用分段锁，也就是将数据拆分，用多个锁，细节看[这里](http://blog.csdn.net/guangcigeyun/article/details/8278346)

## 练习

```java
/**
 * (R, C, V)组成的表格, 要求：
 * <ul>
 * <li>使用Java标准库实现
 * <li>每个R可以有多个不重复的C，每个(R, C)最多有一个V
 * <li>线程安全，满足其他要求的前提下尽可能支持高并发
 * <li>实现一个以高并发为目标的通用版本
 * <li>实现一个C是enum，V是Boolean，以减少内存使用和读多于写为优化目标的特殊版本
 * </ul>
 *
 * @param <R> 行类型
 * @param <C> 列类型
 * @param <V> 值类型
 */
public interface ConcurrentTable<R, C, V> {

  /**
   * 判断是否存在元组
   *
   * @param row
   * @param column
   * @param value
   * @return 如果存在元组，返回true，否则返回false
   * @throws NullPointerException 参数有null
   */
  boolean contains(R row, C column, V value);

  /**
   * 插入元组
   *
   * 什么
   *
   * @param row
   * @param column
   * @param value
   * @return 如果(R, C, V)不存在则返回null，否则返回原有V值
   * @throws NullPointerException 参数有null
   */
  V put(R row, C column, V value);

  /**
   * 如果(R, C, V)不存在，插入元组并返回null，否则直接返回原有V值 该操作需要保证原子性
   *
   * @param row
   * @param column
   * @param value
   * @return 如果(R, C, V)不存在则返回null，否则返回原有V值
   * @throws NullPointerException 参数有null
   */
  V putIfAbsent(R row, C column, V value);

  /**
   * 通过(R, C)获取V
   *
   * @param row
   * @param column
   * @return (R, C)对应的V值
   * @throws NullPointerException 参数有null
   */
  V get(R row, C column);

  /**
   * 获得row对应的[startColumnInclude, endColumnExclude)区间的(R, V)元组，以NavigableMap的形式返回，
   * 修改返回的map不改变table
   *
   * @param row
   * @param startColumnInclude C下限（包含），不能为null
   * @param endColumnExclude   C上限（不包含），不能为null
   * @return (R, V)元组map
   * @throws NullPointerException 参数有null
   */
  NavigableMap<C, V> getColumnRange(R row, C startColumnInclude, C endColumnExclude);

}
```

对于以上的一个结构，类似于 guava 的 Table 结构，

我们参考下guava，看下他的实现。

1. ArrayTable， 内部直接就是个二维数组。简单粗暴。
2. HashBasedTable,  数据以 `Map<R, Map<C, V>>` 形式存储

1. 优点，直接index，快。 缺点，sizefixed，二维数组，定义好不可更改，
2. 优点，不用定义row，col size，利用map可扩展。缺点，性能差，比如定位一个value，我要2次hash(这还算好).如果要根据 column 遍历，我得先遍历所有的row，然后再遍历所有column compare column 的key， 这是O^2 的，这操作用1 只需要 O n


再根据 减少内存使用 ＋ 读多于写 很明显的指出 在array的实现上 别使用 `CopyOnWriteArrayList`, 和是用 ReadWriteLock

以下是实现

```java
import java.util.List;
import java.util.NavigableMap;
import java.util.TreeMap;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

import com.google.common.collect.ImmutableList;
import com.google.common.collect.ImmutableMap;

import static com.google.common.base.Preconditions.checkArgument;
import static com.google.common.base.Preconditions.checkElementIndex;

/**
 * <li>每个R可以有多个不重复的C，每个(R, C)最多有一个V
 * <p/>
 * <li>实现一个C是enum，V是Boolean，以减少内存使用和读多于写为优化目标的特殊版本
 * <p/>
 * <p/>
 * 20085
 *
 * TODO 优化 由于C 是enum， 可以省去columnList, 直接用 enum.ordinal
 *
 * @author caorong
 */
public class MyConcurrentTable<R, C extends Enum, V> implements ConcurrentTable<R, C, V> {

  private final ImmutableList<R> rowList;
  private final ImmutableList<C> columnList;
  private final ImmutableMap<R, Integer> rowKeyToIndex;
  private final ImmutableMap<C, Integer> columnKeyToIndex;
  private final V[][] array;
  final ReadWriteLock rwl = new ReentrantReadWriteLock();
  private final Lock readLock = rwl.readLock();
  private final Lock writeLock = rwl.writeLock();

  public MyConcurrentTable(Iterable<? extends R> rowKeys, Iterable<? extends C> columnKeys) {
    this.rowList = ImmutableList.copyOf(rowKeys);
    this.columnList = ImmutableList.copyOf(columnKeys);
    checkArgument(!rowList.isEmpty());
    checkArgument(!columnList.isEmpty());

    // 提供通过 rowkey和column key 反查, 用不着遍历了
    rowKeyToIndex = index(rowList);
    columnKeyToIndex = index(columnList);

    V[][] tmpArray = (V[][]) new Object[rowList.size()][columnList.size()];
    array = tmpArray;
  }

  private static <E> ImmutableMap<E, Integer> index(List<E> list) {
    ImmutableMap.Builder<E, Integer> columnBuilder = ImmutableMap.builder();
    for (int i = 0; i < list.size(); i++) {
      columnBuilder.put(list.get(i), i);
    }
    return columnBuilder.build();
  }

  @Override
  public boolean contains(R row, C column, V value) {
    // read lock
    readLock.lock();
    try {
      return containsRow(row) && containsColumn(column);
    } finally {
      readLock.unlock();
    }
  }

  public boolean containsRow(Object rowKey) {
    return rowKeyToIndex.containsKey(rowKey);
  }

  public boolean containsColumn(Object columnKey) {
    return columnKeyToIndex.containsKey(columnKey);
  }

  public V set(int rowIndex, int columnIndex, V value) {
    // In GWT array access never throws IndexOutOfBoundsException.
    checkElementIndex(rowIndex, rowList.size());
    checkElementIndex(columnIndex, columnList.size());
    V oldValue = array[rowIndex][columnIndex];
    array[rowIndex][columnIndex] = value;
    return oldValue;
  }

  @Override
  public V put(R row, C column, V value) {
    // write lock
    Integer rowIndex = rowKeyToIndex.get(row);
    checkArgument(rowIndex != null, "Row %s not in %s", row, rowList);
    Integer columnIndex = columnKeyToIndex.get(column);
    checkArgument(columnIndex != null, "Column %s not in %s", column, columnList);
    writeLock.lock();
    try {
      return set(rowIndex, columnIndex, value);
    } finally {
      writeLock.unlock();
    }
  }

  @Override
  public V putIfAbsent(R row, C column, V value) {
    return put(row, column, value);
  }

  public V at(int rowIndex, int columnIndex) {
    // In GWT array access never throws IndexOutOfBoundsException.
    checkElementIndex(rowIndex, rowList.size());
    checkElementIndex(columnIndex, columnList.size());
    readLock.lock();
    try {
      return array[rowIndex][columnIndex];
    } finally {
      readLock.unlock();
    }
  }

  @Override
  public V get(R row, C column) {
    Integer rowIndex = rowKeyToIndex.get(row);
    Integer columnIndex = columnKeyToIndex.get(column);
    return (rowIndex == null || columnIndex == null) ? null : at(rowIndex, columnIndex);
  }

  @Override
  public NavigableMap<C, V> getColumnRange(R row, C startColumnInclude, C endColumnExclude) {
    readLock.lock();
    try {
      int colStart = columnKeyToIndex.get(startColumnInclude);
      int colEnd = columnKeyToIndex.get(endColumnExclude);
      TreeMap treeMap = new TreeMap<C, V>();
      Integer rowIndex = rowKeyToIndex.get(row);
      for (int i = colStart; i < colEnd; i++) {
        treeMap.put(columnList.get(i), array[rowIndex][i]);
      }
      return treeMap;
    } finally {
      readLock.unlock();
    }
  }

  public static void main(String[] args) {

  }
}
```

