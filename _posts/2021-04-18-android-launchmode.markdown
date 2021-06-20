---
layout:     post
title:      "再谈Android启动模式"
subtitle:   " \"温故知新，重新认识Android的启动模式\""
date:       2021-04-18 17:30:00
author:     "Weiwq"
header-img: "img/background/post-sample-image.jpg"
catalog: true
tags:
    - Android

---

> “在整理完启动模式后，我发现大家对启动模式的理解是有误区的“

笔者会用adb的打印，看每一种启动模式下，任务栈的变化。

## 引言

再谈启动模式，貌似没啥意思。但是你能正确回答下面的问题吗？

- 问题1：singleTask启动模式，在启动新的Activity的时候，真的会重新创建新的任务栈吗，所谓的任务栈存在指的是什么？
- 问题2：设置了Intent.FLAG_ACTIVITY_NEW_TASK，每次都会创建新的任务栈吗？
- 问题3：对于singleInstance，显式指定taskAffinity为应用包名，那在启动的时候，还会创建新的任务栈吗？

以下实验基于Android 9.0。如果大家只是想了解每种启动模式的基本概念，可以只看对应的任务栈图，直接忽略掉每一小点。

## 一、ActivityRecord、TaskRecord、ActivityStack的区别 

在回答上面的问题前，我们先整理下ActivityRecord、TaskRecord、ActivityStack的区别。

<img src="/img/blog_android_launchmode/1.webp" width="60%" height="40%"> 

- `ActivityRecord`：一个ActivityRecord对应一个Activity，保存了Activity的所有信息；但一个Activity可能会有多个ActivityRecord，因为Activity可以被多次启动，这个主要取决于其启动模式。

- `TaskRecord`：一个TaskRecord由一个或者多个ActivityRecord组成，这就是我们常说的任务栈，具有后进先出的特点。

- `ActivityStack`：ActivityStack是用来管理TaskRecord的，包含了多个TaskRecord。

这里简单贴一下代码，让读者对这些概念有一些了解

```java

/**
 * State and management of a single stack of activities.
 */
class ActivityStack extends ConfigurationContainer {

    /**
     * The back history of all previous (and possibly still
     * running) activities.  It contains #TaskRecord objects.
     */
    private final ArrayList<TaskRecord> mTaskHistory = new ArrayList<>();
    //..... 省略代码
}

class TaskRecord extends ConfigurationContainer {

    /** 
      *List of all activities in the task arranged in history order 
      */
    final ArrayList<ActivityRecord> mActivities;
   //..... 省略代码
}


/**
 * An entry in the history stack, representing an activity.
 */
final class ActivityRecord extends ConfigurationContainer {
        final ActivityInfo info; // all about me
        int launchMode;         // the launch mode activity attribute.
        final String launchedFromPackage; // always the package who started the activity.
       //..... 省略代码
}
```

### 启动模式的实际值

如果大家看了上面的代码，就会发现launchMode的值是int类型，与我们认知的不太一样，这里贴一下映射关系。

```java

public class ActivityInfo extends ComponentInfo implements Parcelable {
    /** 
       * Constant corresponding to <code>standard</code> in 
       * the {@link android.R.attr#launchMode} attribute.
       */
    public static final int LAUNCH_MULTIPLE = 0;
    /** 
      * Constant corresponding to <code>singleTop</code> in 
      * the {@link android.R.attr#launchMode} attribute. 
      */
    public static final int LAUNCH_SINGLE_TOP = 1;
    /** 
      * Constant corresponding to <code>singleTask</code> in 
      * the {@link android.R.attr#launchMode} attribute. 
      */
    public static final int LAUNCH_SINGLE_TASK = 2;
    /** 
      * Constant corresponding to <code>singleInstance</code> in 
      * the {@link android.R.attr#launchMode} attribute. 
     */
    public static final int LAUNCH_SINGLE_INSTANCE = 3;
    
    public String taskAffinity;
  }

```

我们注意到ActivityInfo里面有一个taskAffinity字段，想必大家应该知道其作用了——指定任务栈名。不知道也没关系，后面会结合启动模式讲解。

###  使用dumpsys  命令查看任务栈

我们先来看看在launcher下，任务栈的情况

在cmd下运行 `adb shell dumpsys activity a`可以得到如下信息

```cmd
ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)
Display #0 (activities from top to bottom):

  Stack #0: type=home mode=fullscreen
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)
    Task id #1
    mBounds=Rect(0, 0 - 0, 0)
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
    * TaskRecord{a9becdf #1 I=com.miui.home/.launcher.Launcher U=0 StackId=0 sz=1}
      userId=0 effectiveUid=u0a23 mCallingUid=1000 mUserSetupComplete=true mCallingPackage=com.android.systemui
      intent={act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10800100 cmp=com.miui.home/.launcher.Launcher}
      realActivity=com.miui.home/.launcher.Launcher
      autoRemoveRecents=false isPersistable=false numFullscreen=1 activityType=2
      rootWasReset=false mNeverRelinquishIdentity=true mReuseTask=false mLockTaskAuth=LOCK_TASK_AUTH_PINNABLE
      Activities=[ActivityRecord{3705f28 u0 com.miui.home/.launcher.Launcher t1}]
      askedCompatMode=false inRecents=true isAvailable=true
      stackId=0
      * Hist #0: ActivityRecord{3705f28 u0 com.miui.home/.launcher.Launcher t1}
          packageName=com.miui.home processName=com.miui.home
          launchedFromUid=0 launchedFromPackage=null userId=0
          app=ProcessRecord{2a0244b 2458:com.miui.home/u0a23}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10800100 cmp=com.miui.home/.launcher.Launcher }
          frontOfTask=true task=TaskRecord{a9becdf #1 I=com.miui.home/.launcher.Launcher U=0 StackId=0 sz=1}
          taskAffinity=null
          fullscreen=true noDisplay=false immersive=false launchMode=2


```

-  Stack #0：即home下的ActivityStack id为0。
- Task id #1：即ActivityStack中有一个id为1的TaskRecord。
- Activities：用来存放ActivityRecord的，任务栈存放Activity的位置，可以通过这个关键字查看Activity的入栈出栈情况。“com.miui.home/.launcher.Launcher” 就是对应Activity的类路径。
- taskAffinity：细心的读者应该会注意到这个字段，就是我们在manifest文件中配置的，这里为null表明没有添加taskAffinity。
- launchMode：即启动模式，对应值为int。

## 二、Standard模式

>  默认值。系统在启动该 Activity 的任务中创建 Activity 的新实例，并将 intent 传送给该实例。Activity 可以多次实例化，每个实例可以属于不同的任务，一个任务可以拥有多个实例。

比如 A启动B，B启动A的任务栈变化情况如下

<img src="/img/blog_android_launchmode/3.webp" width="100%" height="40%">

新建一个项目，MainActivity启动 SecondActivity，都是标准的启动模式。对应的Activity配置如下

```java
        <activity
            android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity
            android:name=".StandardActivity"/>
```

### 1）intent不设置flag，不配置taskAffinity

新建一个项目，MainActivity启动 SecondActivity，都是标准的启动模式。打印的任务栈如下：

```java

  Stack #9: type=standard mode=fullscreen
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)
   Task id #18359
    mBounds=Rect(0, 0 - 0, 0)
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
    * TaskRecord{b751d5b #18359 A=com.example.launchmode U=0 StackId=31 sz=2}
      affinity=com.example.launchmode
      ....
    Activities=[ActivityRecord{8b23942 u0 com.example.launchmode/.MainActivity t18359}, ActivityRecord{d0a45e5 u0 com.example.launchmode/.StandardActivity t18359}]
      ....

```

可以看到Activities 里面有两个ActivityRecord，分别对应MainActivity，SecondActivity。这里贴一下对于的生命周期（注意，面试必考题）和TaskId，可以看到Task Id是一样的：

```java
04-18 10:53:37.981 15379 15379 D MainActivity: onCreate:  Task id 18359
04-18 10:53:38.029 15379 15379 D MainActivity: onStart:  Task id 18359
04-18 10:53:38.032 15379 15379 D MainActivity: onResume:  Task id 18359
04-18 10:53:39.611 15379 15379 D MainActivity: ------------- start activity -------------
04-18 10:53:39.622 15379 15379 D MainActivity: onPause:  Task id 18359
04-18 10:53:39.635 15379 15379 D StandardActivity: onCreate:  Task id 18359
04-18 10:53:39.649 15379 15379 D StandardActivity: onStart:  Task id 18359
04-18 10:53:39.651 15379 15379 D StandardActivity: onResume:  Task id 18359
04-18 10:53:40.199 15379 15379 D MainActivity: onStop:  Task id 18359
```

Activity里面可以通过 `getTaskId()` 方式获取到TaskId，具体实现就不再讲解，作为一个课后作业。

### 2）只设置intent为FLAG_ACTIVITY_NEW_TASK

在MainActivity启动StandardActivity的时候，设置flag如下

```java
    public void next(View view) {
        Log.d(TAG, "------------- start activity -------------");
        Intent intent = new Intent(this, StandardActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
    }
```

对应的生命周期和TaskId如下，可见并没有启动新的任务栈

```java
04-18 10:57:12.584 15563 15563 D MainActivity: onCreate:  Task id 18361
04-18 10:57:12.631 15563 15563 D MainActivity: onStart:  Task id 18361
04-18 10:57:12.634 15563 15563 D MainActivity: onResume:  Task id 18361
04-18 10:57:14.096 15563 15563 D MainActivity: ------------- start activity -------------
04-18 10:57:14.105 15563 15563 D MainActivity: onPause:  Task id 18361
04-18 10:57:14.117 15563 15563 D StandardActivity: onCreate:  Task id 18361
04-18 10:57:14.132 15563 15563 D StandardActivity: onStart:  Task id 18361
04-18 10:57:14.134 15563 15563 D StandardActivity: onResume:  Task id 18361
04-18 10:57:14.677 15563 15563 D MainActivity: onStop:  Task id 18361
```

### 3）只配置taskAffinity

在manifest中配置StandardActivity的taskAffinity为 `com.demo.launchmode`

```java
        <activity
            android:name=".StandardActivity"
            android:taskAffinity="com.demo.launchmode"/>
```

直接启动standardActivity，对应的周期和TaskId如下，可见这种方式也不会启动新的任务栈。

```java
04-18 11:02:51.802 16954 16954 D MainActivity: onCreate:  Task id 18363
04-18 11:02:51.849 16954 16954 D MainActivity: onStart:  Task id 18363
04-18 11:02:51.852 16954 16954 D MainActivity: onResume:  Task id 18363
04-18 11:02:53.283 16954 16954 D MainActivity: ------------- start activity -------------
04-18 11:02:53.293 16954 16954 D MainActivity: onPause:  Task id 18363
04-18 11:02:53.306 16954 16954 D StandardActivity: onCreate:  Task id 18363
04-18 11:02:53.321 16954 16954 D StandardActivity: onStart:  Task id 18363
04-18 11:02:53.323 16954 16954 D StandardActivity: onResume:  Task id 18363
04-18 11:02:53.873 16954 16954 D MainActivity: onStop:  Task id 18363
```

### 4）intent设置FLAG_ACTIVITY_NEW_TASK 、配置taskAffinity

按照2）、3）方式，同时在intent设置FLAG_ACTIVITY_NEW_TASK 和配置taskAffinity。MainActivity与standardActivity的TaskId就不一样了。对应的栈结构，可以通过上面的dumpsys命令查看，这里就不列出来了。

```java
04-18 11:04:55.107 17185 17185 D MainActivity: onCreate:  Task id 18365
04-18 11:04:55.154 17185 17185 D MainActivity: onStart:  Task id 18365
04-18 11:04:55.156 17185 17185 D MainActivity: onResume:  Task id 18365
04-18 11:04:56.707 17185 17185 D MainActivity: ------------- start activity -------------
04-18 11:04:56.721 17185 17185 D MainActivity: onPause:  Task id 18365
04-18 11:04:56.751 17185 17185 D StandardActivity: onCreate:  Task id 18366
04-18 11:04:56.774 17185 17185 D StandardActivity: onStart:  Task id 18366
04-18 11:04:56.776 17185 17185 D StandardActivity: onResume:  Task id 18366
04-18 11:04:57.560 17185 17185 D MainActivity: onStop:  Task id 18365
```

**画重点：要同时指定taskAffinity和FLAG_ACTIVITY_NEW_TASK才能在新的任务栈启动Standard的Activity**



## 三、SingleTop模式

> 如果当前任务的顶部已存在 Activity 的实例，则系统会通过调用其 `onNewIntent()` 方法来将 intent 转送给该实例，而不是创建 Activity 的新实例。Activity 可以多次实例化，每个实例可以属于不同的任务，一个任务可以拥有多个实例（但前提是返回堆栈顶部的 Activity 不是该 Activity 的现有实例）。

如果A以Standard启动B，B再以SingleTop启动B，对应的图为：

<img src="/img/blog_android_launchmode/4.webp" width="100%" height="40%">

如果A以Standard启动B，B再以SingleTop启动A，对应的图为：

<img src="/img/blog_android_launchmode/5.webp" width="100%" height="40%">

对应的Activity配置如下

```java
   <activity
            android:name=".SingleTopActivity"
            android:launchMode="singleTop" />
```

### 1）intent不设置flag，不配置taskAffinity

​	对应的周期是，这种情况是不会创建新的任务栈

```java

04-18 16:40:43.267 24395 24395 D MainActivity: onCreate:  Task id 18428
04-18 16:40:43.315 24395 24395 D MainActivity: onStart:  Task id 18428
04-18 16:40:43.318 24395 24395 D MainActivity: onResume:  Task id 18428
04-18 16:40:59.145 24395 24395 D MainActivity: ------------- start activity -------------
04-18 16:40:59.168 24395 24395 D MainActivity: onPause:  Task id 18428
04-18 16:40:59.180 24395 24395 D SingleTopActivity: onCreate:  Task id 18428
04-18 16:40:59.196 24395 24395 D SingleTopActivity: onStart:  Task id 18428
04-18 16:40:59.198 24395 24395 D SingleTopActivity: onResume:  Task id 18428
04-18 16:40:59.747 24395 24395 D MainActivity: onStop:  Task id 18428

```

### 2）只设置intent为FLAG_ACTIVITY_NEW_TASK

对应的周期和TaskId如下，可见这种方式不会启动新的任务栈。

```java

04-18 10:28:48.458 14065 14065 D MainActivity: onCreate:  Task id 18343
04-18 10:28:48.505 14065 14065 D MainActivity: onStart:  Task id 18343
04-18 10:28:48.508 14065 14065 D MainActivity: onResume:  Task id 18343
04-18 10:28:50.358 14065 14065 D MainActivity: ------------- start activity -------------
04-18 10:28:50.367 14065 14065 D MainActivity: onPause:  Task id 18343
04-18 10:28:50.380 14065 14065 D SingleTopActivity: onCreate:  Task id 18343
04-18 10:28:50.394 14065 14065 D SingleTopActivity: onStart:  Task id 18343
04-18 10:28:50.396 14065 14065 D SingleTopActivity: onResume:  Task id 18343
04-18 10:28:50.939 14065 14065 D MainActivity: onStop:  Task id 18343

```

### 3）只配置taskAffinity

对应的周期和TaskId如下，可见这种方式也不会启动新的任务栈。

```java

04-18 10:35:29.907 14560 14560 D MainActivity: onCreate:  Task id 18349
04-18 10:35:29.954 14560 14560 D MainActivity: onStart:  Task id 18349
04-18 10:35:29.957 14560 14560 D MainActivity: onResume:  Task id 18349
04-18 10:35:32.096 14560 14560 D MainActivity: ------------- start activity -------------
04-18 10:35:32.107 14560 14560 D MainActivity: onPause:  Task id 18349
04-18 10:35:32.118 14560 14560 D SingleTopActivity: onCreate:  Task id 18349
04-18 10:35:32.132 14560 14560 D SingleTopActivity: onStart:  Task id 18349
04-18 10:35:32.134 14560 14560 D SingleTopActivity: onResume:  Task id 18349
04-18 10:35:32.699 14560 14560 D MainActivity: onStop:  Task id 18349

```

### 4）intent设置FLAG_ACTIVITY_NEW_TASK 、配置taskAffinity

周期、taskId如下，可以看到，singleTop的taskid已经变了。即这种方式可以启动新的任务栈。

```java

04-18 10:31:35.382 14357 14357 D MainActivity: onCreate:  Task id 18345
04-18 10:31:35.430 14357 14357 D MainActivity: onStart:  Task id 18345
04-18 10:31:35.432 14357 14357 D MainActivity: onResume:  Task id 18345
04-18 10:31:37.143 14357 14357 D MainActivity: ------------- start activity -------------
04-18 10:31:37.158 14357 14357 D MainActivity: onPause:  Task id 18345
04-18 10:31:37.184 14357 14357 D SingleTopActivity: onCreate:  Task id 18346
04-18 10:31:37.199 14357 14357 D SingleTopActivity: onStart:  Task id 18346
04-18 10:31:37.201 14357 14357 D SingleTopActivity: onResume:  Task id 18346
04-18 10:31:38.004 14357 14357 D MainActivity: onStop:  Task id 18345

```

### 5）栈顶复用

在栈顶的时候，singletop的周期如下。可以看到会先调用onPause，然后才调用onNewIntent，onResume也会重新调用。

```java
04-18 10:21:18.128 13663 13663 D SingleTopActivity: onCreate:  Task id 18338
04-18 10:21:18.143 13663 13663 D SingleTopActivity: onStart:  Task id 18338
04-18 10:21:18.145 13663 13663 D SingleTopActivity: onResume:  Task id 18338
04-18 10:21:21.629 13663 13663 D SingleTopActivity:  -------------- start singleTop activity --------------
04-18 10:21:21.651 13663 13663 D SingleTopActivity: onPause:  Task id 18338
04-18 10:21:21.653 13663 13663 D SingleTopActivity: onNewIntent:  Task id 18338
04-18 10:21:21.655 13663 13663 D SingleTopActivity: onResume:  Task id 18338

```

**画重点：onNewIntent 方法中传入的intent是新启动Activity的intent，与getIntent() 方法返回的intent不是同一个intent**

### 6） 多个实例的情况

在介绍singleTop的时候，有提到一个任务栈里面可以有多个这种模式的实例，我们来验证一下

配置文件中设置singleTopActivity为主入口，singleTopActivity启动MainActivity，MainActivity再启动singleTopActivity。

```java
       <activity
            android:name=".MainActivity"
            android:launchMode="standard">
        </activity>

        <activity
            android:name=".SingleTopActivity"
            android:launchMode="singleTop">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```

对应的任务栈如下，我们可以看到id为 18351的任务栈中有两个SingleTopActivity的实例（不是同一个实例）。

```java
Task id #18351
* TaskRecord{f846b29 #18351 A=com.example.launchmode U=0 StackId=23 sz=3}
    Activities=[ActivityRecord{7e1765c u0 com.example.launchmode/.SingleTopActivity t18351}, ActivityRecord{95d526f u0 com.example.launchmode/.MainActivity t18351}, ActivityRecord{76569fe u0 com.example.launchmode/.SingleTopActivity t18351}]

```

**画重点：singleTop只进行栈顶的复用，如果不在栈顶，就会创建新的实例**

## 四、singleTask模式

> 系统会创建新任务，并实例化新任务的根 Activity。但是，如果另外的任务中已存在该 Activity 的实例，则系统会通过调用其 `onNewIntent()` 方法将 intent 转送到该现有实例，而不是创建新实例。Activity 一次只能有一个实例存在。

如果A以Standard方式启动B，B再以SingleTask启动A，对应的图为：

<img src="/img/blog_android_launchmode/6.webp" width="100%" height="40%">

这种启动模式会比较复杂，先验证什么时候会启动新的任务栈。对应的Activity配置如下

```java
    <activity
            android:name=".SingleTaskActivity"
            android:launchMode="singleTask" />
```

### 1）intent不设置flag，不配置taskAffinity

周期和任务栈如下。恩？SingleTaskActivity与MainActivity的TaskId一样？是的，由于我们没有给SingleTaskActivity配置taskAffinity，所以SingleTaskActivity会复用默认的任务栈。

```java
04-18 11:18:33.266 17490 17490 D MainActivity: onCreate:  Task id 18368
04-18 11:18:33.313 17490 17490 D MainActivity: onStart:  Task id 18368
04-18 11:18:33.316 17490 17490 D MainActivity: onResume:  Task id 18368
04-18 11:18:35.122 17490 17490 D MainActivity: ------------- start activity -------------
04-18 11:18:35.132 17490 17490 D MainActivity: onPause:  Task id 18368
04-18 11:18:35.146 17490 17490 D SingleTaskActivity: onCreate:  Task id 18368
04-18 11:18:35.160 17490 17490 D SingleTaskActivity: onStart:  Task id 18368
04-18 11:18:35.162 17490 17490 D SingleTaskActivity: onResume:  Task id 18368
04-18 11:18:35.712 17490 17490 D MainActivity: onStop:  Task id 18368
```

### 2）只设置intent为FLAG_ACTIVITY_NEW_TASK

设置NEW_TASK总可以创建新的任务栈吧？然鹅，周期和任务栈如下。啊哈？也没有启动新的任务栈？是不是有点超出认知范围了？

```java
04-18 11:22:33.415 17817 17817 D MainActivity: onCreate:  Task id 18371
04-18 11:22:33.462 17817 17817 D MainActivity: onStart:  Task id 18371
04-18 11:22:33.465 17817 17817 D MainActivity: onResume:  Task id 18371
04-18 11:22:34.887 17817 17817 D MainActivity: ------------- start activity -------------
04-18 11:22:34.897 17817 17817 D MainActivity: onPause:  Task id 18371
04-18 11:22:34.912 17817 17817 D SingleTaskActivity: onCreate:  Task id 18371
04-18 11:22:34.928 17817 17817 D SingleTaskActivity: onStart:  Task id 18371
04-18 11:22:34.930 17817 17817 D SingleTaskActivity: onResume:  Task id 18371
04-18 11:22:35.493 17817 17817 D MainActivity: onStop:  Task id 18371
```

对应的任务栈如下，可以看到，这种方式，并没有创建新的任务栈。不是说好的 `系统会创建新任务`吗？神奇。。。

```java
Task id #18371
   ....
   affinity=com.example.launchmode
   Activities=[ActivityRecord{e70466d u0 com.example.launchmode/.MainActivity t18368}, ActivityRecord{da03210 u0 com.example.launchmode/.SingleTaskActivity t18368}]
```

### 3）只配置taskAffinity

上面两种方式都不能让singleTask这种启动模式创建新的任务栈，那么我们再来看看这种方式，

先配置Activity 的taskAffinity

```java
       <activity
            android:name=".SingleTaskActivity" 
            android:launchMode="singleTop"
            android:taskAffinity="com.demo.launchmode"/>
```

对应的生命周期和TaskId如下：

```java
04-18 11:46:13.367 18454 18454 D MainActivity: onCreate:  Task id 18378
04-18 11:46:13.415 18454 18454 D MainActivity: onStart:  Task id 18378
04-18 11:46:13.417 18454 18454 D MainActivity: onResume:  Task id 18378
04-18 11:46:14.973 18454 18454 D MainActivity: ------------- start activity -------------
04-18 11:46:14.989 18454 18454 D MainActivity: onPause:  Task id 18378
04-18 11:46:15.014 18454 18454 D SingleTaskActivity: onCreate:  Task id 18379
04-18 11:46:15.029 18454 18454 D SingleTaskActivity: onStart:  Task id 18379
04-18 11:46:15.032 18454 18454 D SingleTaskActivity: onResume:  Task id 18379
04-18 11:46:15.834 18454 18454 D MainActivity: onStop:  Task id 18378
```

taskId终于变了。这是为什么呢？

其实，在singleTask模式下，先会检查对应的任务栈是否已经存在，如果不存在就创建对应的任务栈，如果已经存在了就直接复用。由于我们没有指定taskAffinity（默认为包名），所以在1）2）的情况下，就不会创建新的任务栈，直接复用了MainActivity的任务栈。

### 4）intent设置FLAG_ACTIVITY_NEW_TASK 、配置taskAffinity

对应的周期和Taskid如下：

```java
04-18 11:28:45.628 18164 18164 D MainActivity: onCreate:  Task id 18375
04-18 11:28:45.678 18164 18164 D MainActivity: onStart:  Task id 18375
04-18 11:28:45.681 18164 18164 D MainActivity: onResume:  Task id 18375
04-18 11:28:47.804 18164 18164 D MainActivity: ------------- start activity -------------
04-18 11:28:47.819 18164 18164 D MainActivity: onPause:  Task id 18375
04-18 11:28:47.845 18164 18164 D SingleTaskActivity: onCreate:  Task id 18376
04-18 11:28:47.863 18164 18164 D SingleTaskActivity: onStart:  Task id 18376
04-18 11:28:47.865 18164 18164 D SingleTaskActivity: onResume:  Task id 18376
04-18 11:28:48.666 18164 18164 D MainActivity: onStop:  Task id 18375
```

这种方式也能创建新的任务栈。

在了解了任务栈的创建时机，我们再来看看实例的复用问题。

### 5）实例复用（非栈顶）

在对singleTask的作用描述中，有句很重要的话——`Activity 一次只能有一个实例存在`。怎么理解呢？

我们先启动singleTaskActivity，再启动StandardActivity，最后再启动singleTaskActivity，对应的周期如下：

```java
04-18 12:13:36.873 19652 19652 D SingleTaskActivity: onCreate:  Task id 18391
04-18 12:13:36.903 19652 19652 D SingleTaskActivity: onStart:  Task id 18391
04-18 12:13:36.905 19652 19652 D SingleTaskActivity: onResume:  Task id 18391
04-18 12:13:40.613 19652 19652 D SingleTaskActivity: ------------- start activity -------------
04-18 12:13:40.648 19652 19652 D SingleTaskActivity: onPause:  Task id 18391
04-18 12:13:40.669 19652 19652 D StandardActivity: onCreate:  Task id 18391
04-18 12:13:40.687 19652 19652 D StandardActivity: onStart:  Task id 18391
04-18 12:13:40.689 19652 19652 D StandardActivity: onResume:  Task id 18391
04-18 12:13:41.247 19652 19652 D SingleTaskActivity: onStop:  Task id 18391
04-18 12:13:44.651 19652 19652 D StandardActivity: ------------- start activity -------------
04-18 12:13:44.681 19652 19652 D StandardActivity: onPause:  Task id 18391
04-18 12:13:44.690 19652 19652 D SingleTaskActivity: onNewIntent:  Task id 18391
04-18 12:13:44.691 19652 19652 D SingleTaskActivity: onStart:  Task id 18391
04-18 12:13:44.693 19652 19652 D SingleTaskActivity: onResume:  Task id 18391
04-18 12:13:45.245 19652 19652 D StandardActivity: onStop:  Task id 18391
04-18 12:13:45.248 19652 19652 D StandardActivity: onDestroy:  Task id 18391

```

通过上面的周期，可以看出最后在启动singleTaskActivity后，StandardActivity走了onDestiny方法，即被退栈。对应的任务栈如下

```java
  Task id #18391
      ....
       Activities=[ActivityRecord{a81eec8 u0 com.example.launchmode/.SingleTaskActivity t18391}]
```

整体流程如下

<img src="/img/blog_android_launchmode/2.jpg" width="80%" height="40%">

### 6）实例复用（栈顶）

这种情况与singleTop模式的周期是一致的，如下：

```java
04-18 12:27:44.120 19913 19913 D SingleTaskActivity: onCreate:  Task id 18393
04-18 12:27:44.147 19913 19913 D SingleTaskActivity: onStart:  Task id 18393
04-18 12:27:44.150 19913 19913 D SingleTaskActivity: onResume:  Task id 18393
04-18 12:27:46.043 19913 19913 D SingleTaskActivity: ------------- start activity -------------
04-18 12:27:46.048 19913 19913 D SingleTaskActivity: onPause:  Task id 18393
04-18 12:27:46.049 19913 19913 D SingleTaskActivity: onNewIntent:  Task id 18393
04-18 12:27:46.050 19913 19913 D SingleTaskActivity: onResume:  Task id 18393
```

## 五、singleInstance模式

> 与 `"singleTask"` 相似，唯一不同的是系统不会将任何其他 Activity 启动到包含该实例的任务中。该 Activity 始终是其任务唯一的成员；由该 Activity 启动的任何 Activity 都会在其他的任务中打开。

在Android 6.0与Android9.0 上，SingleInstance的实现方式不太一样。

我们先看在Android 6.0上，在A以singleInstance 方式启动B的任务栈，即创建新的Task

<img src="/img/blog_android_launchmode/7.webp" width="100%" height="40%">

Android 9上，在A以singleInstance 方式启动B的任务栈，可见在Android 9 上是直接创建了一个新的Stack和Task。

<img src="/img/blog_android_launchmode/8.jpg" width="90%" height="40%">

对应配置文件

```java
        <activity
            android:name=".SingleInstanceActivity"
            android:launchMode="singleInstance" />
```

### 1）intent不设置flag，不配置taskAffinity

直接启动SingleInstanceActivity，对应的周期与TaskId如下:

```java
04-18 12:47:27.360 20390 20390 D MainActivity: onCreate:  Task id 18397
04-18 12:47:27.407 20390 20390 D MainActivity: onStart:  Task id 18397
04-18 12:47:27.410 20390 20390 D MainActivity: onResume:  Task id 18397
04-18 12:47:29.020 20390 20390 D MainActivity: ------------- start activity -------------
04-18 12:47:29.034 20390 20390 D MainActivity: onPause:  Task id 18397
04-18 12:47:29.062 20390 20390 D SingleInstanceActivity: onCreate:  Task id 18398
04-18 12:47:29.076 20390 20390 D SingleInstanceActivity: onStart:  Task id 18398
04-18 12:47:29.079 20390 20390 D SingleInstanceActivity: onResume:  Task id 18398
04-18 12:47:29.882 20390 20390 D MainActivity: onStop:  Task id 18397
```

神奇，尽管SingleInstanceActivity 没设置taskAffinity，SingleInstanceActivity与MainActivity 的TaskId也会不一样。我们再来看看任务栈情况

这个是SingleInstanceActivity的

```java
  Stack #70: type=standard mode=fullscreen
    Task id #18398
    * TaskRecord{4fbb7bd #18398 A=com.example.launchmode U=0 StackId=70 sz=1}
       affinity=com.example.launchmode
       ....
       Activities=[ActivityRecord{6a70f2c u0 com.example.launchmode/.SingleInstanceActivity t18398}]
```

这个是MainActivity的

```java
  Stack #69: type=standard mode=fullscreen
    Task id #18397
    * TaskRecord{b3c2503 #18397 A=com.example.launchmode U=0 StackId=69 sz=1}
      affinity=com.example.launchmode
      ....
      Activities=[ActivityRecord{83be7c8 u0 com.example.launchmode/.MainActivity t18397}]

```

我们发现，他们都不在同一个ActivityStack里面了！！！！也就是说，即使没有配置affinity ，singleInstance启动模式也会直接创建新的stack

### 2）只设置intent为FLAG_ACTIVITY_NEW_TASK

与1）方式一样，这里就不贴出来了

### 3）只配置taskAffinity

按照如下配置：

```java
   <activity
            android:name=".SingleInstanceActivity"
            android:launchMode="singleInstance"
            android:taskAffinity="com.demo.launchdemo"/>
```

SingleInstanceActivity 的 taskId会变

```java
04-18 14:55:07.879 21977 21977 D MainActivity: onCreate:  Task id 18403
04-18 14:55:07.926 21977 21977 D MainActivity: onStart:  Task id 18403
04-18 14:55:07.929 21977 21977 D MainActivity: onResume:  Task id 18403
04-18 14:55:11.811 21977 21977 D MainActivity: ------------- start activity -------------
04-18 14:55:11.832 21977 21977 D MainActivity: onPause:  Task id 18403
04-18 14:55:11.844 21977 21977 D SingleInstanceActivity: onCreate:  Task id 18404
04-18 14:55:11.863 21977 21977 D SingleInstanceActivity: onStart:  Task id 18404
04-18 14:55:11.866 21977 21977 D SingleInstanceActivity: onResume:  Task id 18404
04-18 14:55:12.700 21977 21977 D MainActivity: onStop:  Task id 18403
```

我们重点看看SingleInstanceActivity的 ActivityStack

```java
  Stack #76: type=standard mode=fullscreen
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)
    Task id #18404
    * TaskRecord{4614180 #18404 A=com.demo.launchdemo U=0 StackId=76 sz=1}
      affinity=com.demo.launchdemo
      ....
      Activities=[ActivityRecord{801ecc6 u0 com.example.launchmode/.SingleInstanceActivity t18404}]
```

下面是MainActivity的 ActivityStack

```java
  Stack #75: type=standard mode=fullscreen
    Task id #18403
    * TaskRecord{426f3fe #18403 A=com.example.launchmode U=0 StackId=75 sz=1}
      affinity=com.example.launchmode
      .....
      Activities=[ActivityRecord{98c0ac9 u0 com.example.launchmode/.MainActivity t18403}]
```

可以发现，SingleInstanceActivity也是放在新的ActivityStack中，并且affinity是我们指定的，其他并无差异。

### 4）intent设置FLAG_ACTIVITY_NEW_TASK 、配置taskAffinity

同样的，这种方式与3）的效果是一致的。

### 5）成员唯一

我们先启动MainActivity，然后启动SingleInstanceActivity，再启动StandardActivity，来看看任务栈的情况

从下面的TaskId，可以看出，由SingleInstanceActivity启动的StandardActivity 实例与MainActivity是同一个TaskId。

```java
04-18 15:08:16.086 22635 22635 D MainActivity: onCreate:  Task id 18411
04-18 15:08:16.121 22635 22635 D MainActivity: onStart:  Task id 18411
04-18 15:08:16.124 22635 22635 D MainActivity: onResume:  Task id 18411
04-18 15:08:17.948 22635 22635 D MainActivity: ------------- start activity -------------
04-18 15:08:17.963 22635 22635 D MainActivity: onPause:  Task id 18411
04-18 15:08:17.989 22635 22635 D SingleInstanceActivity: onCreate:  Task id 18412
04-18 15:08:18.004 22635 22635 D SingleInstanceActivity: onStart:  Task id 18412
04-18 15:08:18.007 22635 22635 D SingleInstanceActivity: onResume:  Task id 18412
04-18 15:08:18.810 22635 22635 D MainActivity: onStop:  Task id 18411
04-18 15:08:19.502 22635 22635 D SingleInstanceActivity: ------------- start activity -------------
04-18 15:08:19.514 22635 22635 D SingleInstanceActivity: onPause:  Task id 18412
04-18 15:08:19.526 22635 22635 D StandardActivity: onCreate:  Task id 18411
04-18 15:08:19.540 22635 22635 D StandardActivity: onStart:  Task id 18411
04-18 15:08:19.542 22635 22635 D StandardActivity: onResume:  Task id 18411
04-18 15:08:20.398 22635 22635 D SingleInstanceActivity: onStop:  Task id 18412
```

对应的MainActivity任务栈

```java
  Stack #83: type=standard mode=fullscreen
  Task id #18411
    * TaskRecord{8c3ba81 #18411 A=com.example.launchmode U=0 StackId=83 sz=2}
      affinity=com.example.launchmode
      .....
      Activities=[ActivityRecord{52ceb58 u0 com.example.launchmode/.MainActivity t18411}, ActivityRecord{3436465 u0 com.example.launchmode/.StandardActivity t18411}]
```

SingleInstanceActivity 所处的任务栈如下：

```java
  Stack #84: type=standard mode=fullscreen
   Task id #18412
    * TaskRecord{1f72f67 #18412 A=com.example.launchmode U=0 StackId=84 sz=1}
      affinity=com.example.launchmode
      .....
      Activities=[ActivityRecord{7b786ce u0 com.example.launchmode/.SingleInstanceActivity t18412}]
```

可见，singleInstance所处的任务栈有且仅有一个实例。

### 6）多SingleInstance情况

这里不禁有一个疑问：ASingleInstanceActivity去启动BSingleInstanceActivity，那BSingleInstanceActivity的实例也是重新创建新的ActivityStack吗？

这里创建一个新的OtherSingleInstanceActivity，启动MainActivity，然后启动SingleInstanceActivity，再启动OtherSingleInstanceActivity

我们再来看看对应的周期和TaskId，可以看出来，三个Activity对应的TaskId都不一样

```java
04-18 15:21:20.029 22908 22908 D MainActivity: onCreate:  Task id 18414
04-18 15:21:20.079 22908 22908 D MainActivity: onStart:  Task id 18414
04-18 15:21:20.082 22908 22908 D MainActivity: onResume:  Task id 18414
04-18 15:21:21.496 22908 22908 D MainActivity: ------------- start activity -------------
04-18 15:21:21.508 22908 22908 D MainActivity: onPause:  Task id 18414
04-18 15:21:21.540 22908 22908 D SingleInstanceActivity: onCreate:  Task id 18415
04-18 15:21:21.563 22908 22908 D SingleInstanceActivity: onStart:  Task id 18415
04-18 15:21:21.565 22908 22908 D SingleInstanceActivity: onResume:  Task id 18415
04-18 15:21:22.350 22908 22908 D MainActivity: onStop:  Task id 18414
04-18 15:21:22.833 22908 22908 D SingleInstanceActivity: ------------- start activity -------------
04-18 15:21:22.845 22908 22908 D SingleInstanceActivity: onPause:  Task id 18415
04-18 15:21:22.875 22908 22908 D OtherSingleInstanceActivity: onCreate:  Task id 18416
04-18 15:21:22.952 22908 22908 D OtherSingleInstanceActivity: onStart:  Task id 18416
04-18 15:21:22.954 22908 22908 D OtherSingleInstanceActivity: onResume:  Task id 18416
04-18 15:21:23.687 22908 22908 D SingleInstanceActivity: onStop:  Task id 18415
```

我们再来看看OtherSingleInstanceActivity的任务栈

```java

  Stack #88: type=standard mode=fullscreen
    Task id #18416
    * TaskRecord{9b24f6 #18416 A=com.example.launchmode U=0 StackId=88 sz=1}
      affinity=com.example.launchmode
      ....
      Activities=[ActivityRecord{5037835 u0 com.example.launchmode/.OtherSingleInstanceActivity t18416}]
```

SingleInstanceActivity的任务栈

```java

 Stack #87: type=standard mode=fullscreen
    Task id #18415
    * TaskRecord{7663b64 #18415 A=com.example.launchmode U=0 StackId=87 sz=1}
      affinity=com.example.launchmode
      ....
      Activities=[ActivityRecord{384be99 u0 com.example.launchmode/.SingleInstanceActivity t18415}]
```

MainActivity 的任务栈

```java
 Stack #86: type=standard mode=fullscreen
    Task id #18414
    * TaskRecord{ff7f1cd #18414 A=com.example.launchmode U=0 StackId=86 sz=1}
      affinity=com.example.launchmode
      Activities=[ActivityRecord{9dba354 u0 com.example.launchmode/.MainActivity t18414}]
```

可以看到affinity 一样，但是 ActivityStack id 不一样，说明每一个SingleInstance的Activity独占一个ActivityStack。不像其他三种（Standard，singleTop，singleTask）那样，新建一个TaskRecord。

可见SingleInstance与SingleTask在实现上还是有很大的区别，他们只是在概念上很像罢了。

## 六、总结

估计大家都迷糊了，这里做一个总结吧。尤其是singleTask这个模式，与预期的还是有点差别的。

|  是否创建新的任务  | intent不设置flag，不配置taskAffinity | 只设置intent为FLAG_ACTIVITY_NEW_TASK | 只配置taskAffinity | intent设置FLAG_ACTIVITY_NEW_TASK 、配置taskAffinity |
| :----------------: | :----------------------------------- | :----------------------------------- | :----------------- | :-------------------------------------------------- |
|    Standard模式    | 否                                   | 否                                   | 否                 | 是                                                  |
|   SingleTop模式    | 否                                   | 否                                   | 否                 | 是                                                  |
|   SingleTask模式   | 否                                   | 否                                   | 是                 | 是                                                  |
| singleInstance模式 | 是                                   | 是                                   | 是                 | 是                                                  |

回到文章开始提的三个问题，想必大家都知道答案了（不知道的就再看一遍！！！），这里贴一下吧。

- 问题1：singleTask启动模式，在启动新的Activity的时候，真的会重新创建新的任务栈吗？

  答：不一定，只有指定了taskAffinity 才会创建对应的任务栈，否则直接复用应用默认的任务栈。

  

- 问题2：设置了Intent.FLAG_ACTIVITY_NEW_TASK，每次都会创建新的任务栈吗？

  答：不一定，看Activity所属的任务栈（即Affinity，默认为包名）是否已创建，如果没有就会新建。

  

- 问题3：对于singleInstance，显式指定taskAffinity为应用包名，那在启动的时候，还会创建新的任务栈吗？

  答：会的，实际上是新建一个ActivityStack（如果已经存在了，就不创建），里面存放一个对应包的TaskRecord。

**画重点**

- onNewIntent 方法中传入的intent是新启动Activity的intent，与getIntent() 方法返回的intent不是同一个intent
- 在Activity中，可以通过getTaskId()方法获取到TaskId
- SingleInstance在Android 6.0与Android9.0的实现方式不一样（应该是从Android7开始就不一样了），但是概念是一样的



——Weiwq  于 2021.04 广州

