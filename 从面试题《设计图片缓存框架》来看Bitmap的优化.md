# 从面试题《设计图片缓存框架》来看缓存和Bitmap的优化

近日看到安卓群里探讨了两个问题，一个是Leakcanary的实现原理，一个是图片缓存框架设计。看了大佬们的讨论瞬间觉得这两者有藕断丝连的关系，借着这个机会也学习并总结一下。

本文主要从以下几点分别进行总结，先整理一些必要的先行知识，最后贴上自己图片缓存框架的设计类图

- LruCache实现原理
- 内存的回收及引用
- Bitmap的优化
- Leakcanary原理
- 图片缓存框架的设计

<!--more-->

### LruCache实现原理

谈到图片缓存，可能首先大家想到的是内存缓存，关于内存缓存的方式多种多样，这里主要探讨利用LruCache实现缓存的方法及原理。

打开`LruCache`的源码会发现`LruCache`本质上是对`LinkedHashMap`进行了一层封装。

```java
public class LruCache<K, V> {
    private final LinkedHashMap<K, V> map;
    /** Size of this cache in units. Not necessarily the number of elements. */
    private int size;
    private int maxSize;

    private int putCount;
    private int createCount;
    private int evictionCount;
    private int hitCount;
    private int missCount;
    //...
    }
```

那为什么利用`LinkedHashMap`就能巧妙的实现Least recently used（*最近最少使用算法*）？

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
{
   	/**
     * The head of the doubly linked list.
     */
    private transient LinkedHashMapEntry<K,V> header;

    public V get(Object key) {
        LinkedHashMapEntry<K,V> e = (LinkedHashMapEntry<K,V>)getEntry(key);
        if (e == null)
            return null;
        e.recordAccess(this);
        return e.value;
    }
       /**
     * LinkedHashMap entry.
     */
    private static class LinkedHashMapEntry<K,V> extends HashMapEntry<K,V> {
        // These fields comprise the doubly linked list used for iteration.
        LinkedHashMapEntry<K,V> before, after;

        LinkedHashMapEntry(int hash, K key, V value, HashMapEntry<K,V> next) {
            super(hash, key, value, next);
        }

        /**
         * Removes this entry from the linked list.
         */
        private void remove() {
            before.after = after;
            after.before = before;
        }

        /**
         * Inserts this entry before the specified existing entry in the list.
         */
        private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

        /**
         * This method is invoked by the superclass whenever the value
         * of a pre-existing entry is read by Map.get or modified by Map.set.
         * If the enclosing Map is access-ordered, it moves the entry
         * to the end of the list; otherwise, it does nothing.
         */
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }
		//...
    }
    
 }
```

先来看看`LinkedHashMap`如何实现的，其本质就是一个**双向链表**。`HashMap`的优势在于随机访问速度快，而`LinkedHashMap`在其之上进行扩展，使其具有了有序性。

首先可以看到`LinkedHashMap`继承自`HashMap`，并且`LinkedHashMapEntry`继承自`HashMapEntry`，在`LinkedHashMap`内部会存储一个名为`header`的头指针。同时注意`get`方法中调用了`recordAccess`方法，下文会提到该方法。

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/MapEntry.png)

对于一个容器的使用，无外乎增删改查，那我们从`put`方法来看看这个`LinkedHashMap`干了些什么。在`LinkedHashMap`中，并没有`put`方法，该方法继承自父类`HashMap`，看一下该方法。

```java
    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        int hash = sun.misc.Hashing.singleWordWangJenkinsHash(key);
        int i = indexFor(hash, table.length);
        for (HashMapEntry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);//此处会调用到HashMapEntry的recordAccess方法
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
```

可以看到，如果节点不存在则添加节点，如果节点存在在修改值之后会调用到`recordAccess`方法，而这个方法由子类`LinkedHashMapEntry`重写。再次贴上几个关键方法的实现。

```java
        /**
         * Removes this entry from the linked list.
         */
        private void remove() {
            before.after = after;
            after.before = before;
        }

        /**
         * Inserts this entry before the specified existing entry in the list.
         */
        private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

        /**
         * This method is invoked by the superclass whenever the value
         * of a pre-existing entry is read by Map.get or modified by Map.set.
         * If the enclosing Map is access-ordered, it moves the entry
         * to the end of the list; otherwise, it does nothing.
         */
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }
```

可以看到这段逻辑中先后调用了`remove`和`addBefore`方法。`remove`方法用于将节点从链表中删除，并连接删除节点的前后节点。`addBefore`方法则用于将这个节点置于整个双向链表的表头。`get`、`remove`方法也采用类似的方式实现。

就是说**每一次你访问数据的时候，都会将节点置于整个双向链表的表头**。将最近使用的节点置于链表前面，链表末端自然是很久没访问的节点。这样也就实现了LruCache算法。



### 内存中的引用类型

关于引用类型主要列出强、软、弱、虚四种引用类型的特征，并且介绍一下api，这里就不探讨虚拟机回收的问题了。

//集合里面存着虚引用的引用，如果虚引用被回收，引用还在么？

| 引用类型 |                  | 特征                 |
| -------- | ---------------- | -------------------- |
| 强引用   | StrongReference  | 不会回收             |
| 软引用   | SoftReference    | 当内存不足时会回收   |
| 弱引用   | WeakReference    | 当发生gc的时候会回收 |
| 虚引用   | PhantomReference | 任何时候都会被回收   |

`WeakReference`和`SoftReference`都继承自`Reference`，这个类有两个构造方法。以软引用的构造方法为例，` public SoftReference(T referent, ReferenceQueue<? super T> q)`。第二个参数传入引用队列，当软引用或弱引用被回收的时候，会把这个软引用或弱引用加入引用队列。LeakCanary就是利用了这个特点，思想类似于设置一个回收成功的监听。

```java
public abstract class Reference<T> {
    /* -- Constructors -- */

    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = queue;
    }
}

```



### Bitmap的优化

#### 内存大小计算

内存大小计算公式 = 宽 * 高 * 单位像素所占字节数

| 配置      | 单位像素所占字节数 |
| --------- | ------------------ |
| ARGB_8888 | 4                  |
| ARGB_4444 | 2                  |
| RGB_565   | 2                  |

**Bitmap加载到内存中的大小，与图片文件大小无关，与图片格式无关，只与图片的分辨率有关**。而图片加载到内存中的尺寸与像素密度有关，这个像素密度会根据drawable的目录变化而变化。

|                 | inDensity值 |
| --------------- | ----------- |
| drawable-ldpi   | 120         |
| drawable-mdpi   | 160         |
| drawable-hdpi   | 240         |
| drawable-xhdpi  | 320         |
| drawable-xxhdpi | 480         |

PS:以1.5倍递增



#### 图片压缩及内存复用

通常把资源加载到内存，都会进行压缩，当图片本身的尺寸超过显示控件的尺寸时，加载过大的图片也会浪费内存。关于压缩很常见了，以下代码还提供了内存复用。还可以增加一个alpha通道的标志位，如果不需要alpha通道将图片格式设置为` options.inPreferredConfig = Bitmap.Config.RGB_565;`即可。

```java
    public static Bitmap resizeBitmap(Resources resources, int id, int toW, int toH) {
        return resizeBitmap(resources, id, toW, toH, null);
    }

    public static Bitmap resizeBitmap(Resources resources, int id, int toW, int toH, Bitmap reusable) {
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(resources, id, options);
        int fromW = options.outWidth;
        int fromH = options.outHeight;
        options.inSampleSize = calculateInSampleSize(fromW, fromH, toW, toH);
        options.inJustDecodeBounds = false;
        if (reusable != null) {
            options.inMutable = true;//是否复用内存
            options.inBitmap = reusable;//复用内存
        }
        return BitmapFactory.decodeResource(resources, id, options);
    }

    public static int calculateInSampleSize(int fromW, int fromH, int toW, int toH) {
        int inSampleSize = 1;
        while (fromW > toW && fromH > toH) {
            inSampleSize *= 2;
            fromW /= inSampleSize;
            fromH /= inSampleSize;
        }
        return inSampleSize;
    }
```



关于内存复用，存在版本兼容问题，在此做个总结

**Android 4.4之前版本(api < 19)** 

4.4之前的版本图片格式只有jpg和png，必须同等宽高且inSampleSize为1才可以复用bitmap。且被复用的`Bitmap#inPreferredConfig`会覆盖新设置待分配内存的`Bitmap#inPreferredConfig`

**Android 4.4及之后版本(api >= 19)** 

复用的`Bitmap`的内存必须大于等于待分配内存的`Bitmap`



**getByteCount 和 getAllocationByteCount的区别**

如果被复用的`Bitmap`比待分配内存的`Bitmap`要大，那么`getByteCount`表示待分配内存的大小，实际大小可能更大。`getAllocationByteCount`表示被复用的`Bitmap`的大小。



### Leakcanary监控泄漏原理

先看一下监控的工作机制，转载自[LeakCanary 中文使用说明](https://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)。

1. `RefWatcher.watch()` 创建一个 [KeyedWeakReference](https://github.com/square/leakcanary/blob/master/library/leakcanary-watcher/src/main/java/com/squareup/leakcanary/KeyedWeakReference.java) 到要被监控的对象。
2. 然后在后台线程检查引用是否被清除，如果没有，调用GC。

```java
  /**
   * Watches the provided references and checks if it can be GCed. This method is non blocking,
   * the check is done on the {@link Executor} this {@link RefWatcher} has been constructed with.
   *
   * @param referenceName An logical identifier for the watched object.
   */
  public void watch(Object watchedReference, String referenceName) {
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    if (debuggerControl.isDebuggerAttached()) {
      return;
    }
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

    watchExecutor.execute(new Runnable() {
      @Override public void run() {
        ensureGone(reference, watchStartNanoTime);
      }
    });
  }

  void ensureGone(KeyedWeakReference reference, long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();

    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
    removeWeaklyReachableReferences();
    if (gone(reference) || debuggerControl.isDebuggerAttached()) {
      return;
    }
    gcTrigger.runGc();
    removeWeaklyReachableReferences();
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();

      if (heapDumpFile == null) {
        // Could not dump the heap, abort.
        return;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, watchDurationMs, gcDurationMs,
              heapDumpDurationMs));
    }
  }
```

每个`KeyedWeakReference`都和一个唯一的UUID映射，`RefWatcher`内部通过`Set<String> retainedKeys`保存所有虚引用的UUID。

`watchExecutor`对象是一个`AndroidWatchExecutor`实例，其会利用`IdleHandler`在主线程空闲的时候向后台线程发送一个延迟消息，检查该虚引用是否被清除。首先要明白如果该虚引用被清除了，会被加入到引用队列中。所以检测步骤如下

1. 循环该队列并清除映射表中对应的UUID
2. 利用UUID检查该虚引用是否存在，如果存在触发GC。如果不存在表明清除成功。
3. 触发GC后，再次循环队列清除UUID
4. 如果引用依然没清除，表明内存泄漏，进行分析

```java
  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }

  private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
```

所以LeakCanary其实就是**为检测对象生成一个虚引用，利用虚引用发生gc时会被回收，且被回收后会加入引用队列中的特点，来检测是否发生了内存泄漏。**



### 图片缓存框架的设计

上一下整体设计的类图、流程图、时序图，画的比较生疏，可能存在错误。代码已经传到[Github](https://github.com/lhc20040808/ImageCache)了，这里就不贴了，里面的技术点基本就是上面整理的。框架的健壮性和设计还有待提高，功能也并不完善，不足之处希望大佬指出。



**业务流程图**

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/Activity1__缓存业务流程_1.png)



**缓存框架类图**

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/Model__缓存框架类图_0.png)



**缓存时序图**

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/Collaboration1__Interaction1__bitmap缓存时序图_2.png)

