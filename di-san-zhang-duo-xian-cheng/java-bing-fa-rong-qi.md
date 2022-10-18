# JAVA并发容器

JAVA提供了很多种容器，从并发角度可分为线程安全和线程不安全两种。无论是列表、map、set、queue都有两种不同的实现方式。

我们知道诸如ArrayList,LinkedList，hashmap，hashset,ArrayDeque都是线程不安全的。如果涉及到并发场景，就应该使用对应的并发容器。JAVA中存在的并发容器：

* CopyOnWriteArrayList
* ConcurrentHashMap
* HashTable
* 多个阻塞队列
* ConcurrentLinkedQueue
* ConcurrentLinkedDeque
* ConcurrentSkipListMap

**1、CopyOnWriteArrayList**

写时复制，简单思想是在有写入或者删除操作时会拷贝新的一份数组数据，执行完后，设置新数组，在整个写入过程会加入可重入锁。

它的优点是在进行写入的过程，读请求不会阻塞，写入时候会加入ReentrantLock，因此写和写肯定是阻塞的。其缺点也比较明显，即不能保证数据的一致性，此外因为要进行复制，会额外造成内存浪费。所以，该容器更多得适用于读多写少，且对数据一致性要求不高的场景。否则，请使用读写锁ReentrantReadWriteLock。读写锁是分别有一个读锁，一个写锁，读锁是共享锁，写锁是独占锁。

CopyOnWriteArrayList这点和操作系统的RCU非常类似。



**2、ConcurrentHashMap**

说该容器之前，要先看看HashMap。

HashMap

先看看问题：

1、为什么是线程不安全的；

2、什么时侯红黑树退化成链表；

3、添加元素、扩容步骤

添加的过程：

如果索引数组是null，要扩容进行初始化。

1、计算出key的hash值；

2、如果对应索引位置是空，直接写入新节点，否则走第3步；

3、如果目标位置旧数据等于待插入数据，则更新该索引位置，否则走第4步；

4、如果是红很黑数，则走红黑数逻辑；如果是链表，则遍历，如果是某一个，则更新，如果不是，则插入到链表尾部；

5、size+1,如果此时数量大于threadshold，就要进行扩容。

扩容步骤：

当数组为空，初始化一个容量16，负载因子0.75的数组；

如果不是，就进行两倍的扩容；

如果当前位置的next是null，直接计算hash取模得到新位置坐标赋值（当然，也是原数组位置或者原数组位置+oldCap（原数组长度）），如果是有冲突数据，原数据或者移动到对应的索引位置，或者移动到对应索引位置+原数组的大小。

扩容触发条件：

1、数组为空；

2、冲突链表长度达到阈值 TREEFY\_THREADHOLD 8，但整个hashMap的size不到64；

3、hashMap的size达到阈值  16 \*0.75=12；

Hashmap的容量为什么是2的次幂？

最重要的原因就是为了保证hash的均匀分布。

其计算在数组的位置时，使用了 hash &(n-1)，其中n就是数组的长度，当n是2的幂次时，正好n-1是全1，与运算可以尽量保留hash值得本身数据，如果不是2的幂次，n-1之后的二进制有些位置就是0，与运算可能会导致不同hash值最后计算的位置是相同，增大了hash冲突的可能性。

其次，采用2的次幂的另外一个原因是增加计算速度。我们知道，通常情况下，我们在计算key在数组中的bucket时，会进行取模运算，即拿hash % 数组长度。这种计算方式相当于位运算还是慢的。使用2的次幂，可以直接 n-1进行位运算。



**ConcurrentHashMap**

ConcurrentHashMap是并发操作比较优秀的容器，目前也是得到了广泛的应用。

在JDK1.8之前，其采用数组+链表的结构，并将整个数组分段（Segment），并定义一个内部类Segment，其继承自ReentrantLock，有了段，加锁时不必将整个Map加锁，只需要对某个段加锁，这样不会将整个Map锁死。它的思想就是将整个锁分离，从而提高并发性能。

自JDK1.8开始，其数据结构和HashMap一样都是采用了数组+链表+红黑数的数据结构。它不再使用之前的分段锁，而是使用CAS+synchorized锁实现，关于CAS的介绍，我之前写过文章： [Java多线程共享变量同步机制 ](http://hbnnforever.cn/article/javaconrulock.html)。但是1.8仍然保留了原有的Segment的概念，这是为了兼容之前版本的序列化。

```
  public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

它调用了一个原子操作：

```
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;

}
```

看上面代码可看见其调用了Unsafe的getObjectVolatile这个native方法。通过使用volative定义，保持了数据可见性。

其他原子操作：

```
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

```
casTabAt即采用CAS的方式更新数据。
```

ConcurrentMap在执行put操作的时侯，主要有三个重要的并发控制。

1、但table是空时，会初始化数据。此时会通过volatile变量进行CAS操作来初始化数组；

2、如果对应数组元素（链表表头或者红黑树的根）为null，就采用CAS添加，即调用上面的casTabAt；

3、如果不为null，就使用synchorized加锁。代码：

```
 final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
             //null,cas添加
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                 不为null，加锁，向链表或者红黑树增加元素
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

综上所述，Concurrent实现线程安全的三大法宝：volatile变量+CAS+synchorized锁。



**3、HashTable**

&#x20;      HashTable是线程安全的，但是性能却不怎么好，因为该类的几乎每个方法几乎都用到了synchorized内置锁。所以一般情况下，我们也不用它。

```
 public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }

  public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }
```

**4、阻塞队列**

阻塞队列是线程安全的，通常用于线程池中的等待队列。

ArrayBlockingQueue：是一个数组结构的有界阻塞队列，此队列按照先进先出原则对元素进行排序

LinkedBlockingQueue：是一个单向链表实现的阻塞队列，此队列按照先进先出排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool使用了这个队列。

SynchronousQueue ：一个不存储元素的阻塞队列，每个插入操作必须等到另外一个线程调用移除操作，否则插入操作一直处理阻塞状态，吞吐量通常要高于LinkedBlockingQueue，Executors.newCachedThreadPool使用这个队列。这个有点像golang中的channel。

PriorityBlockingQueue：优先级队列，基于数组的堆结构(默认是小顶堆），进入队列的元素按照优先级会进行排序。

DelayQueue：一个使用优先级队列实现的无界阻塞队列。



**5、ConcurrentLinkedQueue**

ConcurrentLinkedQueue是一种适用于并发操作的无阻塞的队列，FIFO。

ConcurrentLinkeQueue的线程安全且无阻塞主要取决于其节点是volatile定义的Node，且入队和出队采用CAS操作。



**6、ConcurrentLinkedDeque**

ConcurrentLinkedDequeu是ConcurrentLinkedQueue双端队列版本，支持FIFO，FILO。
