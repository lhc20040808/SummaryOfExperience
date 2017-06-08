# Activity的启动流程

当对页面进行操作的时候，系统底层都会发送一条消息给ActivityThread中的mH(Handler)，通过它来处理消息。建Activity的时候也不例外，系统会发一条msg.what为LAUNCH_ACTIVITY的消息给ActivityThread中的handler。

ActivityThread#handleMessage代码

```java
 public void handleMessage(Message msg) { 
	switch (msg.what) {  
		case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
        //省略其他代码
  		}
 }
```

可以看到handler收到消息后，调用了handleLaunchActivity方法。在handleLaunchActivity方法中，调用performLaunchActivity对Activity进行了初始化，之后调用handleResumeActivity方法，该方法最终会回调Activity#onResume方法。

ActivityThread#handleLaunchActivity代码

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
       //省略部分代码

        // Initialize before creating the activity
        WindowManagerGlobal.initialize();

        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            Bundle oldState = r.state;
          	//这个方法最终会回调onResume方法
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
	//省略部分代码
    }
```

在handleLaunchActivity中，通过performLaunchActivity方法返回了一个Activity的实例。而performLaunchActivity的主要工作则是创建Activity及其上下文环境，初始化phoneWindow，设置Theme，调用activity#onCreate方法。

ActivityThread#performLaunchActivity代码

```java
 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
     
       //省略部分代码

        Activity activity = null;
        try {
        //创建Activity实例
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
           //省略部分代码
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
              //创建上下文环境
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
         
                //省略部分代码
              
              	//将各种信息绑定到activity中
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
               ... //省略部分代码
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                  //设置Theme
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
              	//callActivityOnCreate最终会回调activity#onCreate方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
            	... //省略部分代码
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```

接下来看看Activity#attach方法，在其中初始化了phoneWindow，该类为window抽象类的实现类

```java
 final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
		//初始化一个phoneWindow
        mWindow = new PhoneWindow(this, window);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
```

##### Activity启动流程

ActivityThread#scheduleLaunchActivity 

→ ActivityThread#handleMessage 

→ActivityThread#handleLaunchActivity 

→ ActivityThread#performLaunchActivity  

→ ActivityThread#performResumeActivity



##### 分析完启动Activity流程，从上文源码中大概可知Activity创建页面的流程如下

ActivityThread#performLaunchActivity  

→ Activity#attach(初始化PhoneWindow)  

→ Instrumentation #callActivityOnCreate(这里会回调Activity的onCreate)

→ActivityThread#handleResumeActivity（这里会回调Activity的onResume）



每次创建一个Activity的时候，都需要调用setContentView设置xml布局。那么这个布局到底是如何加载到页面当中呢。

Activity#setContentView

```java
  public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

PhoneWindow#setContentView

```java
    @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    
```

Activity#setContentView方法最后会调用window#setContentView方法。而window的实现类是phoneWindow。在PhoneWindow#setContentView中，会调用installDecor初始化decorView，这个decorView是我们根布局的容器，它的父类是FrameLayout。紧接着在generateLayout加载内容布局并添加进decorView，内容布局的id为com.android.internal.R.id.content。

```java
 private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

			//省略部分代码
            }
        }
    }
```



```java
   /**
     * The ID that the main layout in the XML layout file should have.
     */
    public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;

```



Activity视图层级

![层级结构](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/activity_hierachy.png)



为什么说Activity在onResume执行后才会把界面显示出来？

上文提到当Activity启动后，ActivityThread会创建activity并依次调用其生命周期，并在handleResumeActivity方法中，通过windowManger添加decorView。WindowManager是个接口，其实现类为WindowManagerImpl，而在其方法中又是通过WindowManagerGlobal#addView添加decorView。

ActivityThread#handleResumeActivity

```java

    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
   		...//省略部分代码
        // TODO Push resumeArgs into the activity for consideration
        r = performResumeActivity(token, clearHide, reason);

        if (r != null) {
          ...//省略部分代码
            
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                //获取windowManager
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    //通过windowManger添加dercor
                    wm.addView(decor, l);
                }
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }

           ...//省略部分代码
        }
    
```

WindowManagerImpl#addView，调用的地方在handleResumeActivity中wm.addView(decor, l);

可以看到第一个参数为decorView。接着在addView中，会初始化ViewRootImpl，并调用ViewRootImpl#setView方法，在这个方法当中又会调用requestLayout

```java
  @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
```

ViewRootImpl#requestLayout

```java
  @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
          //post一个runnable
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }

```

接着来看看这个runnable，其中又调用了doTraversal().

```
   final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
```

```java
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }
			//开始进入View的绘制流程
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```

Activity添加View的流程

![View的启动绘制的流程](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/view_draw.png)