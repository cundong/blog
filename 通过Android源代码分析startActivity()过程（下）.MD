#通过Android源代码分析startActivity()过程（下）

继续分析startActivity()。

[上一篇blog][1]讲到通过zygote进程fork出了一个新的应用进程，接下来我们需要分析这个进程启动之后的事情。

zygote进程fork出一个新的app进程后，就会调用他ActivityThread类的的main方法来启动它，所以，一个新进程的真正入口并不是我们通常所说的Application#onCreate方法，而是ActivityThread#main方法。

我们就从这个main方法开始分析：

```java
public static void main(String[] args) {

	    //1. 为本进程准备好UI线程的消息队列和Looper
        Looper.prepareMainLooper();

	    //2.把ApplicationThread跟远程AMS关联起来
        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

	    //3. 开始执行消息循环
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }   
```
第1、3两个步骤是Android的消息处理机制，主要用于为本进程开启一个消息循环，我们主要来看第2步，即ActivityThread#attach：

```java
private void attach(boolean system) {
        if (!system) {

	        //1.拿到ams的代理对象ActivityManagerProxy，调用它的attachApplication方法
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }

	        //2.增加垃圾回收机制的观察器，通过这里我看可以看到，当使用的内存大于总内存的3/4，就会触发GC
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                        }
                    }
                }
            });
        } else {
        }
    }
```

ActivityThread#attach方法中，通过ams留在通应用进程中的代理对象ActivityManagerProxy，来实际调用了ActivityManagerService#attachApplication方法，并且把本地的mAppThread（ApplicationThread对象）传给了它，这样ams就能通过这个传递过来的代理对象来调用应用进程的回调接口了。

我们进到ActivityManagerService#attachApplication看一看：

```java

 @Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }

```
可以看到 attachApplication()方法接收了一个IApplicationThread类型的参数thread，ApplicationThread是什么？IApplicationThread是一个接口，他就是应用进程留在ams中的一个代理，关于这一点，可以通过ApplicationThread 的UML图看看：

![此处输入图片的描述][2]

ActivityManagerService#attachApplication中调用了ActivityManagerService#attachApplicationLocked：

```java
    /**
    *  @param IApplicationThread thread 应用进程留在ams中的代理对象
    *  @param pid 调用ams的进程id
    */
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

	try {

	    //1.
            ProfilerInfo profilerInfo = profileFile == null ? null
                    : new ProfilerInfo(profileFile, profileFd, samplingInterval, profileAutoStop);
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    enableTrackAllocation, isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());

	     //当新的进程启动起来了，会有Activity、Service出现，因此我们要更新当前存在的Lru算法
            updateLruProcessLocked(app, false, null);
        } catch (Exception e) {

            return false;
        }

	//2.初始化，并且调度当前进程中所有的Activity
        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }

	//3.初始化，并且调度当前进程中所有的Service
        // Find any services that should be running in this process...
        if (!badApp) {
            try {
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
                badApp = true;
            }
        }

	//4.初始化，并且调度当前进程中所有的Broadcast
        // Check if a next-broadcast receiver is in this process...
        if (!badApp && isPendingBroadcastProcessLocked(pid)) {
            try {
                didSomething |= sendPendingBroadcastsLocked(app);
            } catch (Exception e) {
                // If the app died trying to launch the receiver we declare it 'bad'
                Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
                badApp = true;
            }
        }

        return true;
    }

```

核心的步骤一共有4步，由于3、4两步跟Activity无关，我们就只看1、2两步。

## 1 IApplicationThread#bindApplication

调用IApplicationThread#bindApplication接口，真正起作用的是ApplicationThread#bindApplication接口：

```java
 public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableOpenGlTrace, boolean trackAllocation, boolean isRestrictedBackupMode,
                boolean persistent, Configuration config, CompatibilityInfo compatInfo,
                Map<String, IBinder> services, Bundle coreSettings) {

            if (services != null) {
                // Setup the service cache in the ServiceManager
                ServiceManager.initServiceCache(services);
            }

            setCoreSettings(coreSettings);

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableOpenGlTrace = enableOpenGlTrace;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            sendMessage(H.BIND_APPLICATION, data);
        }
```
这里就是接收一些参数，并且往消息队列中发送了what=BIND_APPLICATION的消息。

再去看Looper是如何处理这个消息的：

```java
case BIND_APPLICATION:
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
        AppBindData data = (AppBindData)msg.obj;
        handleBindApplication(data);
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        break;
```

处理的地方在ActivityThread#handleBindApplication中：

```java

    private void handleBindApplication(AppBindData data) {

	// 得到一个Instrumentation对象
        if (data.instrumentationName != null) {
            InstrumentationInfo ii = null;
            try {
                ii = appContext.getPackageManager().
                    getInstrumentationInfo(data.instrumentationName, 0);
            } catch (PackageManager.NameNotFoundException e) {
            }
            if (ii == null) {
                throw new RuntimeException(
                    "Unable to find instrumentation info for: "
                    + data.instrumentationName);
            }

            mInstrumentationPackageName = ii.packageName;
            mInstrumentationAppDir = ii.sourceDir;
            mInstrumentationSplitAppDirs = ii.splitSourceDirs;
            mInstrumentationLibDir = ii.nativeLibraryDir;
            mInstrumentedAppDir = data.info.getAppDir();
            mInstrumentedSplitAppDirs = data.info.getSplitAppDirs();
            mInstrumentedLibDir = data.info.getLibDir();

            ApplicationInfo instrApp = new ApplicationInfo();
            instrApp.packageName = ii.packageName;
            instrApp.sourceDir = ii.sourceDir;
            instrApp.publicSourceDir = ii.publicSourceDir;
            instrApp.splitSourceDirs = ii.splitSourceDirs;
            instrApp.splitPublicSourceDirs = ii.splitPublicSourceDirs;
            instrApp.dataDir = ii.dataDir;
            instrApp.nativeLibraryDir = ii.nativeLibraryDir;
            LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                    appContext.getClassLoader(), false, true, false);
            ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

            try {
                java.lang.ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            } catch (Exception e) {
                throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            mInstrumentation.init(this, instrContext, appContext,
                   new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
                   data.instrumentationUiAutomationConnection);

            if (mProfiler.profileFile != null && !ii.handleProfiling
                    && mProfiler.profileFd == null) {
                mProfiler.handlingProfiling = true;
                File file = new File(mProfiler.profileFile);
                file.getParentFile().mkdirs();
                Debug.startMethodTracing(file.toString(), 8 * 1024 * 1024);
            }

        } else {
            mInstrumentation = new Instrumentation();
        }

        try {

            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;
            // Do this after providers, since instrumentation tests generally start their
            // test thread at this point, and we don't want that racing.

	    //调用Instrumentation的onCreate回调，这里面其实什么都没有做
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

	   //调用Instrumentation的callApplicationOnCreate回调
            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }

```
handleBindApplication主要就是做了两件事：

### 1.1 初始化Instrumentation类  

这个Instrumentation类就是Activity的一个工具类，Activity的所有生命周期回调接口的调用都是通过调用Instrumentation类的相关回调接口开始调用的，只要Activity出现，它就会跟着出现。

### 1.2 mInstrumentation.callApplicationOnCreate()

mInstrumentation.onCreate()是一个空函数，里面什么都没有做，callApplicationOnCreate()调用了Application的onCreate()接口：

```java
public void callApplicationOnCreate(Application app) {
      app.onCreate();
}
```
看到这里，我们看到当前进程的所有Activity都还没被启动起来，Application的onCreate()回调接口却被调用到了。那么，Activity什么时候被启动呢？不要着急，它马上就上场了，被ActivityStackSupervisor带上场。

## 2 ActivityStackSupervisor#attachApplicationLocked

如果当前进程中有Activity需要展现在用户面前，它就需要被放到Activity的回退栈中去管理，在Android中，关于回退栈的管理主要有两个类完成：ActivityStack和ActivityStackSupervisor。

ActivityStack代表回退栈中的一个单独的Activity；ActivityStackSupervisor通过管理ActivityStack来实现对回退站的管理。

来看ActivityStackSupervisor#attachApplicationLocked源码：

```java
 boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFrontStack(stack)) {
                    continue;
                }
                ActivityRecord hr = stack.topRunningActivityLocked(null);
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Exception in new application when starting activity "
                                  + hr.intent.getComponent().flattenToShortString(), e);
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisibleLocked(null, 0);
        }
        return didSomething;
    }
```
这个方法中，迭代了当前应用进程回退栈中所有的Activity，并且调用了realStartActivityLocked()方法来真正的启动Activity。

那就来看看ActivityStackSupervisor#realStartActivityLocked吧：

```java

    final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {

        try {

            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);

        } catch (RemoteException e) {

        }
        return true;
    }

```
这里面远程调用了ActivityThread#scheduleLaunchActivity接口（app.thread是一个代理对象）：

```java

        @Override
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);

            sendMessage(H.LAUNCH_ACTIVITY, r);
        }

```
scheduleLaunchActivity中接收数据，并且发送一个waht=H.LAUNCH_ACTIVITY的消息到消息队列中。

我们再去看消息队列中是如何处理这个消息的：

```java
case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
```
看来处理这个消息的真正地方是 ActivityThread#handleLaunchActivity ：

```java
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {

	//在创建Activity对象之前，初始化WindowsManagerService
	WindowManagerGlobal.initialize();

	    //1.调用performLaunchActivity，得到一个Activity对象
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;

	    //2.调用handleResumeActivity接口
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);
        } else {

        }
    }
```
## 2.1  ActivityThread#handleLaunchActivity

来看源码：

```java
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");

	//1.为new一个Activity对象准备各种辅助信息
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

	//2.通过Instrumentation类new出一个新的Activity对象来
        Activity activity = null;
        try {
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
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);

		//3.调用Activity的attach方法， 把Application、ActivityThread、intent等参数传递给Activity，并且为把WindowsManagerService引用传似给Activity
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;

		//4.通过Instrumentation类，调用它的callActivityOnCreate()接口，实际上会调用到Activity#onCreate
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;

		//5.调用Activity#performStart接口，实际上会调用Activity#onStart
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }

		//6.通过Instrumentation类，调用它的callActivityOnRestoreInstanceState()接口，实际上会调用到Activity#onRestoreInstanceState
                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }

		//7.通过Instrumentation类，调用它的callActivityOnPostCreate()接口，实际上会调用到Activity#onPostCreate
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
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

performLaunchActivity主要做了以下几件事：

### 1) new 一个Activity对象出来

```java
activity = mInstrumentation.newActivity( cl, component.getClassName(), r.intent);
```

### 2) 调用Activity的attach方法，传递必要的参数给Activity

```java
 activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);
```
 Activity类接收并保存了这些必要参数。

### 3) Activity#onCreate

通过Instrumentation类，调用它的callActivityOnCreate()接口，实际上会调用到Activity#onCreate：

```java
if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
}
```

*4) Activity#onStart

通过Activity#performStart，实际上会调用到Activity#onStart：

```java
	r.stopped = true;
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
```
performStart()：

```java
final void performStart() {

        mInstrumentation.callActivityOnStart(this);

    }
```
这里注意一个细节，当调用完activity.performStart()后，立刻给r.stopped 赋值为 false了，后面会用到这个stoped变量。

###  5) Activity#onRestoreInstanceState

通过Instrumentation类，调用它的callActivityOnRestoreInstanceState()接口，实际上会调用到Activity#onRestoreInstanceState：

```java
if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
 }
```

###  6) Activity#onPostCreate
通过Instrumentation类，调用它的callActivityOnPostCreate()接口，实际上会调用到Activity#onPostCreate：

```java
if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
}
```

看到这里，我们看到，一个Activity启动时的生命周期方法已经被调用了4个，分别是：onCreate()——>onStart()——>onRestoreInstanceState()——>onPostCreate()，接着往下看吧。

## 2.2 ActivityThread#handleResumeActivity

再看ActivityThread#handleResumeActivity：

```java

    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {

	//1.调用performResumeActivity()

        ActivityClientRecord r = performResumeActivity(token, clearHide);

        if (r != null) {
            final Activity a = r.activity;

            if (localLOGV) Slog.v(
                TAG, "Resume " + r + " started activity: " +
                a.mStartedActivity + ", hideForNow: " + r.hideForNow
                + ", finished: " + a.mFinished);

            final int forwardBit = isForward ?  WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

	    //2.询问ams，当前Activity是否应该显示出来

            // If the window hasn't yet been added to the window manager,
            // and this guy didn't finish itself or start another activity,
            // then go ahead and add the window.
            boolean willBeVisible = !a.mStartedActivity;
            if (!willBeVisible) {
                try {
                    willBeVisible = ActivityManagerNative.getDefault().willActivityBeVisible(
                            a.getActivityToken());
                } catch (RemoteException e) {
                }
            }

	    //3.如果需要被显示出现，那么就要通过WindowManager的add方法，把当前Activity的跟布局（dector）add到来
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();

		//注意，到这里把decor设置为不可见了
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }

            // Get rid of anything left hanging around.
            cleanUpPendingRemoveWindows(r);

            // The window is now visible if it has been added, we are not
            // simply finishing, and we are not starting another activity.
            if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                if (r.newConfig != null) {
                    r.tmpConfig.setTo(r.newConfig);
                    if (r.overrideConfig != null) {
                        r.tmpConfig.updateFrom(r.overrideConfig);
                    }
                    if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity "
                            + r.activityInfo.name + " with newConfig " + r.tmpConfig);
                    performConfigurationChanged(r.activity, r.tmpConfig);
                    freeTextLayoutCachesIfNeeded(r.activity.mCurrentConfig.diff(r.tmpConfig));
                    r.newConfig = null;
                }
                if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward="
                        + isForward);
                WindowManager.LayoutParams l = r.window.getAttributes();
                if ((l.softInputMode
                        & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                        != forwardBit) {
                    l.softInputMode = (l.softInputMode
                            & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                            | forwardBit;
                    if (r.activity.mVisibleFromClient) {
                        ViewManager wm = a.getWindowManager();
                        View decor = r.window.getDecorView();
                        wm.updateViewLayout(decor, l);
                    }
                }
                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;

		//这个地方，又通过Activity#makeVisible方法，把decor设置为可见了
                if (r.activity.mVisibleFromClient) {
                    r.activity.makeVisible();
                }
            }

            if (!r.onlyLocalRequest) {
                r.nextIdle = mNewActivities;
                mNewActivities = r;
                if (localLOGV) Slog.v(
                    TAG, "Scheduling idle handler for " + r);
                Looper.myQueue().addIdleHandler(new Idler());
            }
            r.onlyLocalRequest = false;

            // Tell the activity manager we have resumed.
            if (reallyResume) {
                try {
                    ActivityManagerNative.getDefault().activityResumed(token);
                } catch (RemoteException ex) {
                }
            }
        } else {
            // If an exception was thrown when trying to resume, then
            // just end this activity.
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {
            }
        }
    }
```
handleResumeActivity主要做了以下几件事：

### 1) 调用performResumeActivity()

```java
    public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide) {
        ActivityClientRecord r = mActivities.get(token);
        if (localLOGV) Slog.v(TAG, "Performing resume of " + r
                + " finished=" + r.activity.mFinished);
        if (r != null && !r.activity.mFinished) {
            if (clearHide) {
                r.hideForNow = false;
                r.activity.mStartedActivity = false;
            }
            try {
                r.activity.onStateNotSaved();
                r.activity.mFragments.noteStateNotSaved();
                if (r.pendingIntents != null) {
                    deliverNewIntents(r, r.pendingIntents);
                    r.pendingIntents = null;
                }
                if (r.pendingResults != null) {
                    deliverResults(r, r.pendingResults);
                    r.pendingResults = null;
                }
                r.activity.performResume();

                EventLog.writeEvent(LOG_AM_ON_RESUME_CALLED,
                        UserHandle.myUserId(), r.activity.getComponentName().getClassName());

                r.paused = false;
                r.stopped = false;
                r.state = null;
                r.persistentState = null;
            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException(
                        "Unable to resume activity "
                        + r.intent.getComponent().toShortString()
                        + ": " + e.toString(), e);
                }
            }
        }
        return r;
    }
```
这里面调用了Activity#performResume接口：

```java
final void performResume() {

	//1.
        performRestart();

        mLastNonConfigurationInstances = null;

        mCalled = false;

	//2.
	mInstrumentation.callActivityOnResume(this);

	//3.
        onPostResume();
    }
```

### 2) performRestart

```java
final void performRestart() {
        if (mStopped) {
            mInstrumentation.callActivityOnRestart(this);
            performStart();
        }
    }
```

如果当前Activity的状态是stopped，这个方法才有用，否则没什么用，才会调用mInstrumentation.callActivityOnRestart方法，然后再调用performStart()方法，否则这两个方法都不会被调用。

还记得上面提到的细节么：

> 这里注意一个细节，当调用完activity.performStart()后，立刻给r.stopped 赋值为 false了;

也就是说，当一个Activity是初次启动，performRestart里面的逻辑永远都不会被执行到，只有Activity是恢复启动的时候才会执行里面的逻辑，这与我们之前了解的Activity生命周期相关知识也是一致的。

mInstrumentation.callActivityOnRestart中会调用Activity#onRestart，performStart()会调用Activity#onStart

### 3) mInstrumentation.callActivityOnResume(this);

mInstrumentation.callActivityOnResume会真正的调用到Activity#onResume()方法。

### 4) onPostResume();

onPostResume()就是Activity#onPostResume。

看到这里，Activity初次启动所用到的生命周期回调接口就都登场了：onCreate()——>onStart()——>onRestoreInstanceState()——>onPostCreate——>onResume()——>onPostResume()。

我们常用的只有：onCreate()——>onStart()——>onRestoreInstanceState()——>onResume()这几个。

### 5)  把当前Activity真正的展示到用户面前

获取到当前Activity#window的DecorView：

```java
View decor = r.window.getDecorView();
```

通过WindowManager的add()方法，把他加载View树上，这样用户就能看到当前Activity了。

好了，到此为止，一个应用进程启动后，Activity就展示到用户面前了，好累。

  [1]: http://my.oschina.net/liucundong/blog/653128
  [2]: http://7xr1sh.dl1.z0.glb.clouddn.com/369936149295136074.jpg