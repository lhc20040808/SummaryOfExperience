# Android App 启动过程分析与优化

### 应用的启动方式

- `冷启动`

  当应用启动时，后台没有该应用的进程。系统会重新创建一个新的进程分配给该应用，先创建和初始化Application类，再创建和初始化Launch Activity。

- `热启动`

  热启动会从已有的进程中来启动，所以热启动只需要创建和初始化Launch Activity。

### 应用的启动过程

App启动过程可分解为以下三个过程

1. 用户点击Launcher上的app icon
2. 系统为App创建进程，显示启动窗口
3. App在进程中创建自己的组件

系统的进程分配以及一些窗口切换的动画效果，与ROM有关，我们无法处理。我们所能够优化的，也就是第三个步骤。



安利一下这篇[Android 应用进程启动流程](https://www.jianshu.com/p/594e338e5fd3)

### 启动时间的计算

启动时间的定义

- Display Time（API19+）

  通过Displayed关键过滤Log查看信息

  > 12-30 07:27:13.531 566-587/system_process I/ActivityManager: Displayed com.example/.activity.SplashActivity: +1s485ms

  这个信息在Activity窗口完成所有的启动事件之后，第一次绘制的时候输出。启动launch activity时，这个时间包括了从启动进程时到第一次布局与绘制的时间，不包含用户点击app图标然后系统开始准备启动activity的时间。

- am命令启动

  > adb shell
  >
  > am start -W 包名/启动Activity的全限定名
  冷启动

  > vbox86p:/ # am start -W com.example/com.example.activity.SplashActivity                                                                            
  >
  > Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.example/.activity.SplashActivity }
  >
  > Status: ok
  >
  > Activity: com.example/.activity.SplashActivity
  >
  > ThisTime: 870
  >
  > TotalTime: 870
  >
  > WaitTime: 878
  >
  > Complete

  热启动

  > vbox86p:/ # am start -W com.example/com.example.activity.SplashActivity                                                                            
  >
  > Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.example/.activity.SplashActivity }
  >
  > Warning: Activity not started, its current task has been brought to the front
  >
  > Status: ok
  >
  > Activity: com.example/.activity.MainActivity
  >
  > ThisTime: 0
  >
  > TotalTime: 0
  >
  > WaitTime: 2
  >
  > Complete

  - ThisTime：最后一个启动的Activity的启动耗时
  - TotalTime：所有Activity的启动耗时
  - WaitTime：ActivityManagerService启动App的Activity时的总时间



### 启动时间的优化

- Theme

  - 取消预加载画面（取消加载时的黑屏/白屏），加载时展示laucher界面

    > ```Xml
    > <style name="StartStyle">
    >     <item name="android:windowDisablePreview">true</item>
    > </style>
    > ```

    ​

  - 设置自己的预加载画面，可以设置图片

    > ```xml
    > <style name="StartStyle">
    >     <item name="android:windowBackground">@color/blue</item>
    > </style>
    > ```

    PS:利用Theme的方式并没有优化启动速度，仅仅从感性让用户觉得变快了

- 异步初始化

  将第三方框架和部分模块的初始化放入异步线程去执行

- 延迟初始化/懒加载

  可以通过mDecorview.post方法在contentview初始化完毕后，再去执行耗时初始化操作。或者当需要用的时候再进行懒加载。

- IntentService

  将初始化任务放到IntentService中去做。

  IntentService是继承于Service并处理异步请求的一个类，在IntentService内部，有一个工作线程来处理耗时操作。启动IntentService的方式和启动传统Service方式一样。同时，当任务执行完成后，IntentService会自动停止。

  ​

### 参考文档

[Android App优化之提升你的App启动速度之实例挑战](https://www.jianshu.com/p/4f10c9a10ac9)

[一触即发 App启动优化最佳实践](https://segmentfault.com/a/1190000007406875)

[Android 应用进程启动流程](https://www.jianshu.com/p/594e338e5fd3)