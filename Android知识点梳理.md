# Android知识点梳理

[TOC]

## 前言

本文整理了从网上搜罗的一些面试题目和自己觉得有必要掌握的知识点，也算对自己所学做一个查漏补缺。有些题目没附总结，随着学习深入未来会不断完善，也会加入新的章节。

有些问题会附自己总结的资料或一些个人读过感觉讲的挺不错的参考资料，但可能为了搞清这个知识点，我读了很多文章帮助理解，甚至读了几遍源码。最后再回过头读大牛的文章，才有种柳暗花明又一村的感觉。

有些问题资料繁多，也不特附参考资料了。比如事件分发机制，网上从Activity开始讲的文章一大堆。

有些总结属于个人归纳，可能不全面，也可能有理解错误的地方，欢迎指出。



## Java

1. ##### JAVA的三大特性

   面向对象的三大特性：封装、继承、多态

   多态允许不同的对象对同一消息做出不同的响应，能消除类型之间的耦合关系。

   实现方式：实现接口，继承父类重写方法，方法重载

2. ##### 类的加载过程，Person p = new Person();为例进行说明

   类的加载有五个过程，分别是`加载→验证→准备→解析→初始化`五个阶段。

   首先通过类的全限定名找到Person.class文件，将这个文件所代表的静态存储结构转换为**方法区**的运行时数据结构，创建一个表示该类型的`java.lang.Class`实例。

   验证该文件流中所包含的信息是否符合虚拟机要求。

   准备阶段为类变量分配内存（在方法区中进行分配）并设置初始化值，给常量赋值。

   最后初始化类，类变量初始化语句和静态代码块会被编译器收集到一起，放到`<clinit>`（类/接口初始化方法）中去执行。

   接着在**堆内存**中开辟空间，分配内存地址，对实例属性设置初始化值。然后显示地给实例属性赋值并执行构造代码块。再执行与之对应的构造方法，最后把内存地址赋给**栈内存**中的p变量。

   *PS：因为Java虚拟机必须确保初始化过程被正确同步，所以这个特性可以被用来写单例模式*

3. ##### GC机制

   引用计数法：每当有个地方引用这个对象，计数器就加1。反之减1。

   可达性分析法：以根集对象为起始点进行搜索，查看对象是否可达

   Java 垃圾回收机制最基本的做法是分代回收。内存中的区域被划分成不同的世代，对象根据其存活的时间被保存在对应世代的区域中。一般的实现是划分成3个世代：年轻、年老和永久。内存的分配是发生在年轻世代中的。当一个对象存活时间足够长的时候，它就会被复制到年老世代中。对于不同的世代采用不同的垃圾回收算法。

   年轻代回收：Minor GC

   老年代回收：Major GC

   整堆回收：Full GC

   Android ART默认采用CMS收集器

4. ##### 类的加载机制，谈谈ClassLoader

5. ##### Java分派机制

6. ##### 静态方法能否被重写

   静态方法是类的方法，不能重写

7. ##### 简单介绍一下java中的泛型，泛型擦除以及相关的概念

   Java字节码中是不含泛型的信息的，会被编译器在编译的时候去掉，这个过程就是类型擦除。泛型的好处是消除强制类型转换，使编译器能有效的保证类型安全。

8. ##### JAVA对象池机制( [Java常量池理解与总结](https://www.jianshu.com/p/c7f47de2ee80))


    Float,Double并没有实现常量池技术

9. ##### float、double与0比较

   因为double类型和float类型都是有精度的，取的都是近似值。

   比较方法

   - 使用BigDecimal
   - 转换成字符串
   - Math.fbs(x) <= EPSINON

10. ##### 内部类访问局部变量为什么要加上final

  因为生命周期不同，局部变量在方法结束之后就销毁。所以编译器会在内部类中生成一个局部变量的拷贝。为了解决原值修改导致与拷贝值不同的问题，加上final修饰，以保证两个值相同。

11. ##### long s = 499999999 * 499999999 能得到正确的值么？

    不能，右侧值的计算默认是int类型





## Android

1. ##### 生命周期及启动模式，其各自有特点

   整个生命周期含六个步：onCreat()→onStart()→onResume()→onPause()→onStop()→onDestroy()

   启动模式分四种：standard、singleTop、singleTask、singleInstance

   在onPause()不能太耗时，因为onPause()必须执行完成，新Activity才会执行

2. ##### activity的启动流程([Activity启动过程](https://www.jianshu.com/p/d4cc363813a7))

3. ##### IntentFilter

   action匹配：只要Intent中的action和过滤规则中的一个action匹配即可。没有设置过滤规则的过滤器只能与没有设置action的隐式匹配。

   category匹配:隐式Intent中的category必须全部能与过滤器的category匹配才算匹配成功

4. ##### 下拉状态栏是否影响Activity的生命周期?锁屏的生命周期是什么样的？

   不影响，下拉状态栏，生命周期不变。但点击通知栏进入别的应用会影响。

   锁屏会执行onPause()→onStop()

5. ##### View的绘制流程

   [探究 Android View 绘制流程，Xml 文件到 View 对象的转换过程](https://xiaozhuanlan.com/topic/4071968532)

   LayoutInflater是一个服务，实现类是PhoneLayoutInflater。内部有一套解析xml的流程并生成view。可以通过设置策略把生成view的过程委托给别人。AppCompatActivity把普通的view换成兼容view就是通过这种方式。

6. ##### 动态注册和静态注册的区别，有序广播和标准广播的区别（[静态/动态注册广播的区别](http://blog.csdn.net/pengkv/article/details/38798709)）

   动态广播不是常驻型广播，跟随activity生命周期，activity结束之前要移除广播接收器。

   静态广播是常驻型广播，应用程序关闭之后也能接收信息。

   有序广播会根据优先级高的先接收，同优先级下动态先于静态，同优先级同类型下，先注册的动态广播先于后注册的动态广播，先扫描的静态广播先于后扫描的静态广播。

7. #####  BroadcastReceiver，LocalBroadcastReceiver 区别

   [LocalBroadcastManager原理分析](http://gityuan.com/2017/04/23/local_broadcast_manager/)

   [Broadcast广播机制分析](http://gityuan.com/2016/06/04/broadcast-receiver/)

   BroadcastReceiver通信走的是Binder机制。通过Binder向AMS(Activity Manager Service)进行注册，广播发送者通过Binder机制向AMS发送广播。AMS查找符合相应条件的BroadcastReceiver，将广播送到相应的消息队里中。因此可以跨进程通信。

   LocalBroadcastReceiver核心是通过Handler实现。因此效率更高，也不受其他app影响。

8. ##### Service的两种启动模式及其生命周期（参考资料：[ Android中的Service：Binder，Messenger，AIDL](http://blog.csdn.net/luoyanglizi/article/details/51594016)）

   ![img](https://upload-images.jianshu.io/upload_images/1981935-bd709d5989105a12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/675)

   分为两种启动模式，一种是startService，一种bindService。

   startService会执行onCreate()→onStartCommand()，多次执行startService()只会多次调用onStartCommand，调用stopService()或stopSelf()后→onDestroy()。

   bindService会执行onCreate()→onbind()，所有service解绑后执行onUnbind()→onDestroy()

9. ##### Fragment生命周期及其特点

   - Fragment可以作为Activity界面的一部分组成出现；
   - 可以在一个Activity中同时出现多个Fragment，并且一个Fragment也可以在多个Activity中使用；
   - 在Activity运行过程中，可以添加、移除或者替换Fragment；
   - Fragment可以响应自己的输入事件，并且有自己的生命周期，它们的生命周期会受宿主Activity的生命周期影响

   ![Fragment生命周期](https://raw.githubusercontent.com/JackyAndroid/AndroidInterview-Q-A/master/picture/fragment-life.png)

10. ##### Fragment之间传递数据的方式

  bundle、利用回调、EventBus

11. ##### Gradle中compileSdkVersion、buildToolsVersion和TargetSdkVersion的区别

    compileSdkVersion是编译版本，代表运行这个项目所需的sdk版本

    buildToolsVersion是构建工具的版本，可以用高版本的构建工具去构建一个低版本sdk的工程

    targetSdkVersion 是 Android 系统提供前向兼容的主要手段。只要 APK 的 targetSdkVersion 不变，即使这个 APK 安装在新 Android 系统上，其行为还是保持老的系统上的行为，这样就保证了系统对老应用的前向兼容性。

12. ##### 什么是ANR？如何定位ANR？

    ANR（Application Not Responding）触发的必要条件是主线程阻塞。分为以下三类

    - 主线程5s内无响应

    - service阻塞20s
    - 前台广播阻塞10s或后台广播阻塞60s

    ​

    定位ANR方法

    - 查看 /data/anr/traces.txt文件

    - 在子线程中每隔5s向主线程发送消息来判断主线程是否阻塞

      ```java
      private void exception(){

              new Thread(new Runnable() {
                  @Override
                  public void run() {
                      while(flag){
                          lasttick = mTick;
                          mHandler.post(tickerRunnable);//向主线程发送消息 计数器值+1
                          try {
                              Thread.sleep(5000);
                          } catch (InterruptedException e) {
                              e.printStackTrace();
                          }
                          if(mTick == lasttick){
                              flag = false;
                              handleAnrError();
                          }
                      }
                  }
              }).start();
          }

          //发生anr的时候，在此处写逻辑
          private void handleAnrError(){

          }
          private final Runnable tickerRunnable = new Runnable() {
              @Override public void run() {
                  mTick = (mTick + 1) % 10;
              }
          };
      ```

      ​

13. ##### Handler机制及其图解([Android消息机制](http://gityuan.com/2015/12/26/handler-message-framework/))

    ![handler图解](http://gityuan.com/images/handler/handler_java.jpg)

    Handler通过sendMessage()发送msg到MessageQueue队列。Looper通过loop，不断提取出达到触发条件的msg，并交由msg的target来处理。经过dispatchMessage()后，交回给Handler的handleMessagel()来进行相应的处理。

14. ##### Looper 死循环为什么不会导致应用卡死?

    Looper其实会导致线程阻塞，这个阻塞发生在native层。这也是子线程在完成所有事情后需要调用`Thread#quit()`的原因，不然这个子线程会一直处于等待状态无法退出。当有消息发送过来的时候，native层会被唤醒，继而导致java层继续执行。

    不会导致应用卡，因为系统不断给主线程的消息队列发送消息。比如生命周期，垂直同步信号，手势事件等。

15. ##### HandlerThread的实现

    继承自Thread类，封装了looper的初始化操作，在初始化`run()`和`getLooper()`上加了锁保证线程同步。

16. ##### 事件分发机制及其图解（[Activity是如何接收到touch事件的（窗口与用户输入系统）](https://yq.aliyun.com/articles/41488)）

    ​

    用`Thread.dumpStack()`打印堆栈信息，对查看调用流程有帮助*

    ​

    ![事件分发机制](http://upload-images.jianshu.io/upload_images/1344733-635359e0278f911c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    ​

17. Binder机制

18. Serializable和Parcelable的区别

19. ##### IntentService作用是什么

    用子线程来处理所有传送至onStartComman()的intent。并且生成一个工作队列来传送Intent对象给onHandleIntent()方法，同一时刻只传送一个Intent对象，这样一来，就不必担心多线程的问题。在所有的请求(Intent)都被执行完以后会自动停止服务。

20. Android是如何进行资源管理的

21. ##### AsyncTask缺陷

    3.0之后AsyncTask都串行调用。同时可能引起内存泄漏，Activity进行销毁了AsyncTask还在执行。即使调用了cancel()方法，doInBackGround()仍然会执行到方法结束。

22. 描述Android Studio点击Build后发生了什么

23. ##### 画出Android大体架构图

    ![Android架构图](https://images2015.cnblogs.com/blog/316892/201512/316892-20151230165437151-489072197.png)

    ​

    ![Android系统架构图](https://images2015.cnblogs.com/blog/316892/201512/316892-20151231114234745-1475433360.png)

    ​

24. ##### 对 Dalvik、ART 虚拟机有基本的了解([Android开发——JVM、Dalvik以及ART的区别](http://blog.csdn.net/seu_calvin/article/details/52354964))

    在Dalvik下，应用每次运行都需要通过即时编译器(JIT)将字节码转换为机器码，即每次都要编译加运行。因此安装的过程很快，但启动应用的效率变低。而在ART环境中，应用安装的时候，字节码就会预编译成机器码。因此首次安装的时候速度会变慢，但之后每次启动效率会提高。

25. ##### APP如何沙箱，为何这么做([理解Android安全机制](http://www.cnblogs.com/lao-liang/p/5089336.html))

    进程沙箱隔离机制是一种安全机制。在Android系统中，基于linux的UID/GID机制进行了修改，用一个用户ID识别一个应用程序。应用程序及其运行的Dalvik虚拟机运行在独立的Linux进程空间，与其它应用程序完全隔离。这样做可以防止应用程序互相干扰，降低对系统和其他应用程序的风险。

26. ##### 大体说清一个应用程序安装到手机上时发生了什么([深度探究apk安装过程](http://www.androidchina.net/6667.html))

    拷贝APK到指定目录，解压APK，创建应用的数据目录，解析AndroidManifinest.xml文件

27. ##### APP保活

    [APP保活系列](http://blog.csdn.net/andrexpert/article/details/53485360)

28. recycleview listview 的区别,性能

29. 大图加载



## NDK

1. ##### JNI如何调用Java层代码

2. ##### 插件化原理

3. ##### 热修复原理

4. ##### 增量更新原理




## 设计模式

1. ##### 单例模式有哪些？写一个线程安全的单例并分析其线程安全的原因

   Double Check

   ```java
   public class Singleton{
   private volatile static Singleton mSingleton;
   private Singleton(){
   }
   public static Singleton getInstance(){
     if(mSingleton == null){\\A
       synchronized(Singleton.class){\\C
        if(mSingleton == null)
         mSingleton = new Singleton();\\B
         }
       }
       return mSingleton;
     }
   }
   ```

   由于操作系统可以对指令进行重排序，多线程环境下就可能将一个未初始化的对象引用暴露出来，因此加上volatile防止重排序

   ​

   静态内部类

   ```java
   public class Singleton {
           private Singleton() {
           }
           private static class SingletonHolder {
                   private static final Singleton INSTANCE = new Singleton();
            }
           public static Singleton getInstance() {
                   return SingletonHolder.INSTANCE;
           }
   }
   ```

   类加载并被初始化的时候会初始化一次类变量（静态域）并给其赋值，因此有虚拟机保证线程安全。

2. ##### 图解MVP架构，一句话解释MVP和MVC的区别

   MVC

   ##### ![MVC](https://upload-images.jianshu.io/upload_images/1534031-9ae5ba8a11651f08.jpg?imageMogr2/auto-orient/)

   MVP

   ![MVP](https://upload-images.jianshu.io/upload_images/1534031-8f4b6c89c33a6002.jpg?imageMogr2/auto-orient/)

   在MVP中View并不直接使用Modle，而是通过presenter进行两者之间的通信，降低了耦合性。

3. ##### 项目中设计了哪些设计模式

   网络框架等getInstance，单例，其中对外暴露的构造者模式进行调用

   TableView中用过view回收池，享元模式。Android源码中，handler所发送的message采用了享元模式，message自身维护了一个单链表。

   Animator中不同状态绘制不同图形，状态模式。属性动画中对插值器，估值器采用了策略模式。

4. ##### 简绘`观察者模式`UML图，Android中哪里使用了观察者模式

   广播，ListView适配器更新数据都使用了观察者模式

5. ##### 简绘`策略模式`UML图，Android中哪里使用了策略模式

   属性动画的插值器，估值器使用了策略模式

6. ##### 简绘`装饰者模式`UML图，Android中哪里使用了装饰者模式

7. ##### 适配器模式，装饰者模式，外观模式的异同？

   外观模式是对系统统一的封装（一般SDK都用外观模式对外提供一个统一的接口），屏蔽内部实现细节（对客户端不透明）。

   装饰者模式是以透明的方式动态地扩展对象功能（对客户端透明）。

   适配器模式是为了融合两个不兼容的类。




## 数据结构及算法

<u>**除了完成基本的功能，要考虑边界条件、特出输入及错误处理。**</u>
<u>**解决复杂的问题： 画图能使抽象问题形象化，举例使抽象问题具体化，分解使复杂问题简单化。**</u>

1. ##### list，map，set都有哪些具体的实现类，区别都是什么

2. ##### SpareArray原理

3. ##### HashMap实现原理

   HashMap是一个链表散列的数据结构，是线程不安全的。

   ##### 如何解决Hash冲突？

   - 探测法
   - 拉链法(haspmap采用的拉链法)
   - 再散列
   - 建立一个公共的溢出区

4. ##### 二插查找树的特性

5. ##### 平衡二叉树的特性

6. ##### 快速排序

   ```java
   public static void quickSort(int[] array, int start, int end) {
           if (start >= end) {
               return;
           }

           int left = start;
           int right = end - 1;
           int tmp = array[start];
           int t;

           while (left < right) {
   ```


               while (array[right] >= tmp && left < right) {
                   right--;
               }
    
               while (array[left] <= tmp && left < right) {
                   left++;
               }
    
               if (left < right) {
                   t = array[left];
                   array[left] = array[right];
                   array[right] = t;
               }


           }
           array[start] = array[left];
           array[left] = tmp;
           quickSort(array, start, left);
           quickSort(array, left + 1, end);
       }
   ```

7. ##### 二分查找

   ```java
   public static int binarySort(int[] array, int key, int start, int end) {
           int left = start;
           int right = end - 1;

           while (left <= right) {
               int mid = (left + right) >>> 1;
               int midVal = array[mid];
               if (midVal < key) {
                   left = mid + 1;
               } else if (midVal > key) {
                   right = mid - 1;
               } else {
                   return mid;
               }
           }
           return -1;
       }
   ```

8. ##### 堆有哪些数据结构

   堆是完全二叉树，底层用线性结构来实现二叉树。

9. ##### LRUCache算法


   在Java中通过LinkedHashMap来实现LRU算法，该容器扩展自HashMap。增加了before和after两个引用。添加、修改、访问节点时都会将该节点移值链表的顶端。当容器中节点的数量超过自身容量，就会移除链表末端的节点。

10. ##### 将一个字符串转换成int型数字

11. ##### 给出一个搜索二叉树，输出一个排序好的双向链表

12. ##### 给出二叉树和一个值，找出所有和为这个值的路径

13. ##### 给定一个int型 n，输出1~n的字符串例如 n = 4 输出“1 2 3 4”

    ```java
    public static String printNum(int x) {
            if (x == 0) {
                return "";
            } else {
                return printNum(x - 1) + " " + x;
            }
        }
    ```

14. ##### 字符串反转

15. ##### 输出所有的笛卡尔积组合

16. ##### 回型打印二维数组

17. ##### 算法n/m，怎么判断得数是无限循环小数

    step1、判断分数可化为有限小数的方法，如果分母不含除2，5外的任何质因数，那么这个分数必可化为有限小数，并且小数部分的位数等于分母中质因数2与5中个数较多的那个数的个数

    step2、化简分数，即求出最大公约数

    step3、分解质因数

18. ##### 算法判断单链表成环与否？

    定义两个指针，一个指针一次走一步，一个指针一次走两步。如果走的快的追上了走的慢的，那就是环形链表。如果走到了末尾都没有追上，那就不是环形链表。

19. ##### 合并多个有序链表（假设都是递增的）

20. ##### 链表中倒数第 k 个结点

    定义两个指针，第一个指针往前走k-1步，接着两个指针一起遍历。当第一个指针所指的结点下一个结点为null时，第二指针指向了倒数第k个结点。




## 网络

1. ##### 三次握手与四次挥手的原因，为什么需要这样做?

   (SYN表示建立连接，FIN表示关闭连接，ACK表示响应)

   三次握手主要为了防止已经失效的连接请求突然传送到服务端产生错误。比如有个连接请求在某个网络结点长时间滞留了，以致于连接释放了才到达服务端。这时候服务端就会向客户端发送确认报文，同意建立连接。如果不采用三次握手，此时连接就会建立，浪费服务端的资源。

   四次挥手是因为tcp链接是全双工通道，需要双向关闭。接收FIN意味着没有要接收的数据，但仍然可以继续发送数据。

2. ##### 网络五层协议（[OSI七层与TCP/IP五层网络架构详解](https://www.jianshu.com/p/85af586fba54)）

   ##### ![网络分层模型](https://upload-images.jianshu.io/upload_images/5959612-d2df4f482c0898a2.png?imageMogr2/auto-orient/)

3. ##### 描述一次完成的HTTP请求过程

   ##### ![Http请求过程](https://raw.githubusercontent.com/JackyAndroid/AndroidInterview-Q-A/master/picture/http.png)

   先进行域名解析（从浏览器dns缓存找，找不到去操作系统自身dns缓存找，找不到去读hosts文件，再找不到就去DNS服务器查）。然后通过ip请求服务器，请求过程从TCP三次握手建立成功的连接请求开始，客户端按照指定的格式开始向服务端发送HTTP请求，服务端接收请求后，解析HTTP请求，处理完业务逻辑，最后返回一个HTTP的响应给客户端。

4. ##### HTTP 协议中与缓存相关的 HTTP Header 有哪些？

   Cache-Control、Expires、Last-Modified、ETag。前两个控制失效的日期，后两个用来验证网页的有效性。

5. ##### 列举出你所知道的 HTTP 状态码，并描述它们的含义与发生的场景？([HTTP状态码（HTTP Status Code）及常用场景](http://www.focuznet.com/siteroad/t2949/))

   | 状态码  | 解释                                       |
   | ---- | :--------------------------------------- |
   | 1xx  | 表示临时响应并需要请求者继续执行操作的状态代码。                 |
   | 2xx  | 表示成功处理了请求的状态代码。                          |
   | 3xx  | 表示要完成请求，需要进一步操作。 通常，这些状态代码用来重定向。         |
   | 4xx  | 表示请求可能出错，妨碍了服务器的处理。                      |
   | 5xx  | 表示服务器在尝试处理请求时发生内部错误。 这些错误可能是服务器本身的错误，而不是请求出错。 |

6. ##### http 的session&cookie的区别

7. ##### 如何验证证书的合法性（[Android安全开始之安全使用HTTPS](https://www.sslchina.com/news201610-how-android-developers-securely-use-https-and-tls/)）

   HTTPS通信所用到的证书由CA提供，需要在服务器中进行相应的设置才能生效。另外在我们的客户端设备中，**只要访问的HTTPS的网站所用的证书是可信CA根证书签发的，如果这些CA又在浏览器或者操作系统的根信任列表中，就可以直接访问**。自建CA需要在客户端中预埋证书文件，或者将证书硬编码写在代码中进行校验。

8. ##### https中哪里用了对称加密，哪里用了非对称加密，对加密算法（如RSA）等是否有了解

9. ##### 多线程断点续传原理（[Android多线程断点续传下载](https://www.jianshu.com/p/2b82db0a5181)）

   在请求头中设置range，写的时候也从下载的位置开始写。

   ```java
   conn.setRequestProperty("Range", "bytes="+startPos+"-"+endPos);  

   byte[] buffer = new byte[1024];  
   int offset = 0;  
   print("Thread "+this.threadId+" starts to download from position "+startPos);  
   RandomAccessFile threadFile = new RandomAccessFile(this.saveFile,"rwd");  
   threadFile.seek(startPos);  
   ...  
   threadFile.write(buffer,0,offset); 
   ```

10. ##### WebSocket相关以及与socket的区别

11. ##### TCP与UDP区别与应用

    | TCP                 | UDP               |
    | ------------------- | ----------------- |
    | 面向连接，通信之前必须要与对方建立连接 | 面向非连接，不管对方状态就直接发送 |
    | 传输可靠                | 传输不可靠             |
    | 速度慢，建立连接需要开销        | 速度快               |

    ​

12. ##### 网络请求优化（[APP优化之网络优化](https://www.jianshu.com/p/d4c2c62ffc35)）

    API更合理的设计来减少网络请求的频次

    压缩request和response，减少传输数据量

    适当的缓存数据

    证书位数太长也会影响SSL握手的耗时（[SSL延迟有多大？](http://www.ruanyifeng.com/blog/2014/09/ssl-latency.html)）（[SSL/TLS协议运行机制的概述](SSL/TLS协议运行机制的概述)）




## 并发编程

1. ##### ConcurrentHashmap原理

   ConcurrentHashmap内部维护一个segment数组，segment内部又维护一个哈希表。segment继承于ReentrantLock类，因此通过分段锁提高容器并发访问的效率。当容量不够进行扩容的时候，也仅是对需要扩容的段进行重哈希。

2. ##### CopyOnWriteArrayList原理

   CopyOnWrite主要用于读多写少的并发场景。往一个容器添加元素的时候，先将当前容器中的内容copy进一个新容器，然后往新容器里添加元素。添加完了以后再将原容器指向新容器。

3. ##### Volatile原理（[Java 并发编程：volatile的使用及其原理](http://www.cnblogs.com/paddix/p/5428507.html)）

   可见性实现：

   修改变量时会强制将修改后的值刷新到主内存中，并且其他线程再读该变量的值时，由于其工作内存中对应的变量值失效，要重新读取主内存的值

   有序性实现：

   对volative变量限制编译器重排序和处理器重排序

   ​

4. ​ 四种线程池区别，以及常见应用场景，线程池的深入了解

   常用的四种线程池有

   - CachedThreadPool
   - FixedThreadPool
   - ScheduledThreadPool
   - SingleThreadExecutor

5. ##### Integer类是不是线程安全的

   Integer类自身内部通过`不变模式`(用final修饰属性和类)保证了线程安全。但自增自减都不是线程安全的，在使用的时候不可避免的要加锁。也可以使用AtomicInteger，一个提供原子操作的Integer类。通过CAS来实现线程安全，并且不会阻塞线程。CAS是一种原子操作，包含三个值，内存位置、预期原值、新值。仅当内存中的值和预期原值相同，在会将新值存入内存。

6. ##### ThreadLocal原理

   通过Thread内部自己维护的ThreadLocalMap成员来保存数据，key为TreadLocal当前对象。但如果让每一个线程保存了同一个对象实例，也无法实现线程安全，这点需要在应用层得到保证。为了防止内存泄漏，ThreadLocalMap中的每个结点都持有ThreadLocal的弱引用，并且线程退出的时候会进行资源清理。但当我们使用线程池的时候，可能导致会导致线程不会退出，造成内存泄漏。

7. ##### ReetrantLock和Synchronized区别

   ReetrantLock采用的是乐观锁，默认是非公平的，乐观锁的机制就是CAS操作。重入锁的灵活性更高。

   简单举例：

   - 时间锁等候 `tryLock(long time,TimeUnit unit)`
   - 可中断锁等候`lockInterruptibly()`

   Synchroinized采用的是悲观锁，其他线程只能依靠阻塞来等待线程释放锁，但JVM已经做了优化。



## 性能优化

1. ##### 内存泄漏的例子及检查方式

   **内存泄漏的原因：长生命周期的对象拥有短生命周期对象的引用**

   **检查方式：Android Profile、MAT**

   可能产生泄漏的常见原因：

   单例模式持有上下文环境(如果必须是Activity的话用弱引用，否则可以用ApplicationContext)

   静态成员变量。`项目中遇到过的自定义Toast由于View持有上下文环境导致内存泄漏`

   非静态内部类

   集合类持有对象

   未关闭或者释放的资源

2. ##### Leakcanary源码分析

3. ##### 如何计算一张图片的大小

   一个BitMap位图占用的内存=图片长度\*图片宽度\*单位像素占用的字节数。

   | 图片类型      | 单位像素占用的字节数 |
   | --------- | ---------- |
   | ALPHA_8   | 1          |
   | ARGB_4444 | 2          |
   | ARGB_8888 | 4          |
   | RGB_565   | 2          |

4. Bitmap的优化

   图片压缩→内存缓存→内存复用→磁盘缓存

5. 统计启动时间长度





## 开源框架源码

1. Glide源码

2. OkHttp源码
3. EventBus实现原理
4. Retrofit与之前的网络库有什么优势



## 架构

1. ##### 模块化\OSGI实现（好处，原因）

2. ##### 动态布局

3. ##### 怎么去除重复代码？

   开放性问题

   - 公共的代码放到基类
   - 用include标签，ViewStub标签减少布局的重复
   - 将不变的重复代码提炼出来。