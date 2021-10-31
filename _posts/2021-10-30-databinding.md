---
layout:     post
title:      "DataBinding的双向绑定实现原理"
subtitle:   " \"从源码带你剖析DataBinding\""
date:       2021-10-31 16:38:00
author:     "Weiwq"
header-img: "img/background/about-bg-walle.jpg"
catalog:  true
top: false
tags:
    - Android

---

> “ 悄悄咪咪告诉你，DataBinding是怎么实现双向绑定的“

在讲DataBinding之前，有必要讲讲ViewBinding

# ViewBinding

## 配置

要使用ViewBinding，只需要在gradle 添加如下配置即可

```cmd
android {
        ...
        viewBinding {
            enabled = true
        }
 }
```

如果需要在生成绑定类时忽略某个布局文件，请将 tools:viewBindingIgnore="true" 属性添加到相应布局文件的根视图中：

```cmd
<LinearLayout
        ...
            tools:viewBindingIgnore="true" >
        ...
</LinearLayout>
```
## 用法

在创建xml文件后，Android Studio会自动创建对应的类，类名格式为：XML 文件的名称转换为驼峰式大小写，并在末尾添加“Binding”一词。

比如，创建了一个activity_view.xml

```cmd
<LinearLayout 
           ...>
    <TextView
        android:id="@+id/viewBindingView"
        android:layout_width="match_parent"
        android:layout_height="20dp" />
</LinearLayout>
```

生成的绑定类就叫做ActivityViewBinding，然后在对应的activity中使用布局绑定

```java

public class ViewBindingActivity extends AppCompatActivity {
    private ActivityViewBinding activityViewBinding;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 初始化布局和view
        activityViewBinding = ActivityViewBinding.inflate(getLayoutInflater());
        // 设置contentView
        setContentView(activityViewBinding.getRoot());
        // 获取到id为viewBindingView的textView，并设置文本
        activityViewBinding.viewBindingView.setText("你好，我是viewBinding");
    }
}

```

## 原理

在build之后，会在如下路径生成对应的类

![](/img/blog_databinding/2.png)

生成如下代码

```java
public final class ActivityViewBinding implements ViewBinding {
    @NonNull
    private final LinearLayout rootView;
    // 通过遍历xml，找到声明id了的view，id作为变量名
    @NonNull
    public final TextView viewBindingView;

    private ActivityViewBinding(@NonNull LinearLayout rootView, @NonNull TextView viewBindingView) {
        this.rootView = rootView;
        this.viewBindingView = viewBindingView;
    }

    @Override
    @NonNull
    public LinearLayout getRoot() {
        return rootView;
    }

    // 传入LayoutInflater，用于获取对应布局的view
    @NonNull
    public static ActivityViewBinding inflate(@NonNull LayoutInflater inflater) {
        return inflate(inflater, null, false);
    }

    @NonNull
    public static ActivityViewBinding inflate(@NonNull LayoutInflater inflater,
                                              @Nullable ViewGroup parent, boolean attachToParent) {
        // 通过指定的布局获取到rootView
        View root = inflater.inflate(R.layout.activity_view, parent, false);
        if (attachToParent) {
            parent.addView(root);
        }
        return bind(root);
    }
    // 核心代码，通过代码模板，找到对应的view
    @NonNull
    public static ActivityViewBinding bind(@NonNull View rootView) {
        int id;
        missingId:
        {
            id = R.id.viewBindingView;
            TextView viewBindingView = rootView.findViewById(id);
            if (viewBindingView == null) {
                break missingId;
            }
            return new ActivityViewBinding((LinearLayout) rootView, viewBindingView);
        }
        String missingId = rootView.getResources().getResourceName(id);
        throw new NullPointerException("Missing required view with ID: ".concat(missingId));
    }
}
```

生成的类通过LayoutInflater获取布局，然后依次初始化view。
# DataBinding

## 配置

DataBinding 跟ViewBinding一样，只需要在gradle中添加如下配置即可

```java
android {
        ...
        dataBinding {
            enabled = true
        }
    }
```

## 用法

我们先看activity_main.xml布局：

```java
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable
            name="video"
            type="com.example.databinding.VideoBean" />
    </data>
    <LinearLayout xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        tools:context=".MainActivity">
        <TextView
            android:id="@+id/bind_text1"
            android:layout_width="match_parent"
            android:layout_height="30dp"
            android:gravity="center"
            android:text="@{video.title}" />

        <TextView
            android:id="@+id/bind_text2"
            android:layout_width="match_parent"
            android:layout_height="30dp"
            android:gravity="center"
            android:text="@{video.url}" />
    </LinearLayout>
</layout>
```

与正常的xml布局不一样的是，DataBinding 以layout作为第一层。第二层分别是 data和view的根布局。

其中data下的name作为对象，type为类路径名

```java
package com.example.databinding;
class VideoBean {
    public String title;
    public String url;
    public VideoBean(String title, String url) {
        this.title = title;
        this.url = url;
    }
}
```

然后在布局中就可以把对象的属性值显示到TextView上

```java
<TextView
     ...
     android:text="@{video.title}" />
```

在对应的activity中，使用的方式与ViewBinding类似

```java
public class MainActivity extends AppCompatActivity {
    private ActivityMainBinding mainBinding;
    private VideoBean video;

    @Override
    protected void onCreate(@Nullable  Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 获取对应的binding类
        mainBinding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(mainBinding.getRoot());
        video = new VideoBean("小黄人", "http://xiaohuangren.com");
        // 设置对应的数据
        mainBinding.setVideo(video);
        // 设置周期监听，可选
        mainBinding.setLifecycleOwner(this);
    }
}
```

全程我们没有初始化对应的view，只是获取bean，然后更新一下即可

![](/img/blog_databinding/1.png)

当然，我们也可以在布局中添加对应的事件响应

```java
        <Button
            android:id="@+id/bind_text2"
            android:layout_width="match_parent"
            android:layout_height="50dp"
            android:onClick="@{video::increaseScore}"
            android:text="@{String.valueOf(video.score + 1)}" />
```

在类中添加 increaseScore方法

```java
public class VideoBean {
    private ActivityMainBinding activityMainBinding;
    public int score;

    public void increaseScore(View view) {
        score++;
        activityMainBinding.setVideo(this);
    }
    ...
}
```
更多见[布局和绑定表达式](https://developer.android.google.cn/topic/libraries/data-binding/expressions?hl=zh-cn)

## 原理

### view的初始化

生成的类路径与ViewBinding一样，直接看看对应的类

```java
// 抽象类
public abstract class ActivityMainBinding extends ViewDataBinding {
  @NonNull
  public final Button bindText2;
  @Bindable
  protected VideoBean mVideo;
  protected ActivityMainBinding(Object _bindingComponent, View _root, int _localFieldCount,
                                TextView bindText1, Button bindText2) {
    super(_bindingComponent, _root, _localFieldCount);
    this.bindText1 = bindText1;
    this.bindText2 = bindText2;
  }

  // 把data下的参数都声明了get/set方法
  public abstract void setVideo(@Nullable VideoBean video);

  @Nullable
  public VideoBean getVideo() {
    return mVideo;
  }

  @NonNull
  public static ActivityMainBinding inflate(@NonNull LayoutInflater inflater) {
    return inflate(inflater, DataBindingUtil.getDefaultComponent());
  }

  @NonNull
  @Deprecated
  public static ActivityMainBinding inflate(@NonNull LayoutInflater inflater,
                                            @Nullable Object component) {
    return ViewDataBinding.<ActivityMainBinding>inflateInternal(inflater, R.layout.activity_main, null, false, component);
  }
  ...
}
```

这里的ActivityMainBinding 继承了ViewDataBinding，但是内部却没有类似ViewBinding那样，直接通过findViewById方式初始化我们需要的view，那这些view是怎么来的？

这个问题我们先放一放，上面的inflate方法调用了ViewDataBinding的inflateInternal 方法

```java
public abstract class ViewDataBinding extends BaseObservable implements ViewBinding {
    ...
    protected static <T extends ViewDataBinding> T inflateInternal(
            @NonNull LayoutInflater inflater, int layoutId, @Nullable ViewGroup parent,
            boolean attachToParent, @Nullable Object bindingComponent) {
        return DataBindingUtil.inflate(
                inflater,
                layoutId,
                parent,
                attachToParent,
                checkAndCastToBindingComponent(bindingComponent)
        );
    }
}
```

来到DataBindingUtil 这个工具类

```java
public class DataBindingUtil {
   private static DataBinderMapper sMapper = new DataBinderMapperImpl();
    
   public static <T extends ViewDataBinding> T inflate(
            @NonNull LayoutInflater inflater, int layoutId, @Nullable ViewGroup parent,
            boolean attachToParent, @Nullable DataBindingComponent bindingComponent) {
        final boolean useChildren = parent != null && attachToParent;
        final int startChildren = useChildren ? parent.getChildCount() : 0;
        final View view = inflater.inflate(layoutId, parent, attachToParent);
        if (useChildren) {
            return bindToAddedViews(bindingComponent, parent, startChildren, layoutId);
        } else {
            // activity里面传的parent为null ，所以会走这里
            return bind(bindingComponent, view, layoutId);
        }
    }
    static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View root,
            int layoutId) {
        return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
    }

}
```
DataBinderMapperImpl 的实现如下

```java
package androidx.databinding;

public class DataBinderMapperImpl extends MergedDataBinderMapper {
  DataBinderMapperImpl() {
    // 将impl添加到mMappers中
    addMapper(new com.example.databinding.DataBinderMapperImpl());
  }
}
// 对应的基类
public class MergedDataBinderMapper extends DataBinderMapper {
    .... 
    @Override
    public ViewDataBinding getDataBinder(DataBindingComponent bindingComponent, View view,
            int layoutId) {
        // 遍历 mMappers
        for(DataBinderMapper mapper : mMappers) {
            ViewDataBinding result = mapper.getDataBinder(bindingComponent, view, layoutId);
            if (result != null) {
                return result;
            }
        }
        if (loadFeatures()) {
            return getDataBinder(bindingComponent, view, layoutId);
        }
        return null;
    }
}

```

那就看看com.example.databinding.DataBinderMapperImpl 的实现

```java
public class DataBinderMapperImpl extends DataBinderMapper {
  private static final int LAYOUT_ACTIVITYMAIN = 1;

  private static final SparseIntArray INTERNAL_LAYOUT_ID_LOOKUP = new SparseIntArray(1);

  static {
    // 保存我们创建的layout与 index的映射
    INTERNAL_LAYOUT_ID_LOOKUP.put(com.example.databinding.R.layout.activity_main, LAYOUT_ACTIVITYMAIN);
  }

  @Override
  public ViewDataBinding getDataBinder(DataBindingComponent component, View view, int layoutId) {
    // 通过layoutId，获取到index，上面传的是R.layout.activity_main，
    // 所以这里localizedLayoutId为LAYOUT_ACTIVITYMAIN
    int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
    if(localizedLayoutId > 0) {
       // 获取到TAG，上面的代码没有设置tag的地方，但是通过debug得知对应的tag为"layout/activity_main_0"
      final Object tag = view.getTag();
      if(tag == null) {
        throw new RuntimeException("view must have a tag");
      }
      switch(localizedLayoutId) {
        case  LAYOUT_ACTIVITYMAIN: {
          if ("layout/activity_main_0".equals(tag)) {
            return new ActivityMainBindingImpl(component, view);
          }
          throw new IllegalArgumentException("The tag for activity_main is invalid. Received: " + tag);
        }
      }
    }
    return null;
  }
  ...
}
```

至于view的tag是怎么来的，笔者猜测是Android Studio自动添加的

![](/img/blog_databinding/3.png)

最终我们获取到了ActivityMainBindingImpl的实例，对应实现如下

```java
public class ActivityMainBindingImpl extends ActivityMainBinding  {
    @Nullable
    private static final androidx.databinding.ViewDataBinding.IncludedLayouts sIncludes;
    @Nullable
    private static final android.util.SparseIntArray sViewsWithIds;
    static {
        sIncludes = null;
        sViewsWithIds = null;
    }
    // views
    @NonNull
    private final android.widget.LinearLayout mboundView0;
    // listeners
    private OnClickListenerImpl mVideoIncreaseScoreAndroidViewViewOnClickListener;
    // Inverse Binding Event Handlers

    public ActivityMainBindingImpl(@Nullable androidx.databinding.DataBindingComponent bindingComponent, @NonNull View root) {
        // 通过mapBindings方法获取view的数组
        this(bindingComponent, root, mapBindings(bindingComponent, root, 3, sIncludes, sViewsWithIds));
    }
    private ActivityMainBindingImpl(androidx.databinding.DataBindingComponent bindingComponent, View root, Object[] bindings) {
        // 调用super即ActivityMainBinding的构造函数初始化view
        super(bindingComponent, root, 0
            , (android.widget.TextView) bindings[1]
            , (android.widget.Button) bindings[2]
            );
        this.bindText1.setTag(null);
        this.bindText2.setTag(null);
        this.mboundView0 = (android.widget.LinearLayout) bindings[0];
        this.mboundView0.setTag(null);
        setRootTag(root);
        invalidateAll();
    }
    ...
    public void setVideo(@Nullable com.example.databinding.VideoBean Video) {
        this.mVideo = Video;
        synchronized(this) {
            mDirtyFlags |= 0x1L;
        }
        notifyPropertyChanged(BR.video);
        // 请求重新刷新
        super.requestRebind();
    }
    
    // 主要处理UI的刷新和事件的设置
    @Override
    protected void executeBindings() {
        long dirtyFlags = 0;
        synchronized(this) {
            dirtyFlags = mDirtyFlags;
            mDirtyFlags = 0;
        }
        java.lang.String stringValueOfVideoScoreInt1 = null;
        int videoScoreInt1 = 0;
        android.view.View.OnClickListener videoIncreaseScoreAndroidViewViewOnClickListener = null;
        java.lang.String videoTitle = null;
        int videoScore = 0;
        com.example.databinding.VideoBean video = mVideo;

        if ((dirtyFlags & 0x3L) != 0) {
                if (video != null) {
                    // 初始化点击事件
                    videoIncreaseScoreAndroidViewViewOnClickListener = (((mVideoIncreaseScoreAndroidViewViewOnClickListener == null) ? (mVideoIncreaseScoreAndroidViewViewOnClickListener = new OnClickListenerImpl()) : mVideoIncreaseScoreAndroidViewViewOnClickListener).setValue(video));
                    // 给变量赋值
                    videoTitle = video.title;
                    // 给变量赋值
                    videoScore = video.score;
                }
                ....
        }
        if ((dirtyFlags & 0x3L) != 0) {
            // 给TextView设置text，其实就是调用TextView的setText方法
            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.bindText1, videoTitle);
            // 设置监听
            this.bindText2.setOnClickListener(videoIncreaseScoreAndroidViewViewOnClickListener);
            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.bindText2, stringValueOfVideoScoreInt1);
        }
    }
    // 点击事件的回调
    public static class OnClickListenerImpl implements android.view.View.OnClickListener{
        private com.example.databinding.VideoBean value;
        public OnClickListenerImpl setValue(com.example.databinding.VideoBean value) {
            this.value = value;
            return value == null ? null : this;
        }
        @Override
        public void onClick(android.view.View arg0) {
            // 调用我们设置的方法
            this.value.increaseScore(arg0); 
        }
    }
    ...
}
```

ActivityMainBindingImpl 在构造函数中，通过mapBindings函数初始化我们需要的view

```java
public abstract class ViewDataBinding extends BaseObservable implements ViewBinding {
      ....
      private static void mapBindings(DataBindingComponent bindingComponent, View view,
            Object[] bindings, IncludedLayouts includes, SparseIntArray viewsWithIds,
            boolean isRoot) {
        final int indexInIncludes;
        final ViewDataBinding existingBinding = getBinding(view);
        if (existingBinding != null) {
            return;
        }
        Object objTag = view.getTag();
        final String tag = (objTag instanceof String) ? (String) objTag : null;
        boolean isBound = false;
        if (isRoot && tag != null && tag.startsWith("layout")) {
            final int underscoreIndex = tag.lastIndexOf('_');
            if (underscoreIndex > 0 && isNumeric(tag, underscoreIndex + 1)) {
                final int index = parseTagInt(tag, underscoreIndex + 1);
                // 这里的index为0，即给0下标赋值为rootView
                if (bindings[index] == null) {
                    bindings[index] = view;
                }
                indexInIncludes = includes == null ? -1 : index;
                isBound = true;
            } else {
                indexInIncludes = -1;
            }
        } else if (tag != null && tag.startsWith(BINDING_TAG_PREFIX)) {
            // child view的tag为"binding_1"、"binding_2"....
            // 解析该view所属的下标
            int tagIndex = parseTagInt(tag, BINDING_NUMBER_START);
            if (bindings[tagIndex] == null) {
                bindings[tagIndex] = view;
            }
            isBound = true;
            indexInIncludes = includes == null ? -1 : tagIndex;
        } ....
        if (view instanceof  ViewGroup) {
            // 如果当前是viewGroup，那就遍历所有的child。
            final ViewGroup viewGroup = (ViewGroup) view;
            final int count = viewGroup.getChildCount();
            int minInclude = 0;
            for (int i = 0; i < count; i++) {
                final View child = viewGroup.getChildAt(i);
                boolean isInclude = false;
                ....
                if (!isInclude) {
                    // 递归调用
                    mapBindings(bindingComponent, child, bindings, includes, viewsWithIds, false);
                }
            }
        }
    }    
}
```

可以看到，dataBinding在初始化view的时候，相对来说比viewBinding要费事一些，它需要不断的递归遍历所有的view。

### 刷新UI

回到ActivityMainBindingImpl，executeBindings 这个刷新的方法是在哪里调用的呢？

还记得在MainActivity里面，我们需要调用ActivityMainBinding的setVideo方法设置VideoBean吗，该方法实现如下

```java
    public void setVideo(@Nullable com.example.databinding.VideoBean Video) {
        this.mVideo = Video;
        synchronized(this) {
            mDirtyFlags |= 0x1L;
        }
        notifyPropertyChanged(BR.video);
        // 请求重新刷新
        super.requestRebind();
    }
```

我们看看` super.requestRebind()` 做了啥？

```java
public abstract class ViewDataBinding extends BaseObservable implements ViewBinding {
   private static final boolean USE_CHOREOGRAPHER = SDK_INT >= 16;
   ....
   protected ViewDataBinding(DataBindingComponent bindingComponent, View root, int localFieldCount) {
        ....
        if (USE_CHOREOGRAPHER) {
            mChoreographer = Choreographer.getInstance();
            // 屏幕刷新的回调
            mFrameCallback = new Choreographer.FrameCallback() {
                @Override
                public void doFrame(long frameTimeNanos) {
                    // 直接调用了run方法
                    mRebindRunnable.run();
                }
            };
        }
        ....
    }
    
   protected void requestRebind() {
        // mContainingBinding 没有做初始化
        if (mContainingBinding != null) {
            mContainingBinding.requestRebind();
        } else {
            final LifecycleOwner owner = this.mLifecycleOwner;
            if (owner != null) {
                Lifecycle.State state = owner.getLifecycle().getCurrentState();
                // 获取activity的生命周期，如果不是处于活跃状态（STATE>=STARTED）就忽略掉
                // 可以防止内存泄露
                if (!state.isAtLeast(Lifecycle.State.STARTED)) {
                    return; // wait until lifecycle owner is started
                }
            }
            ....
            if (USE_CHOREOGRAPHER) {
                // 监听下一帧的回调
                mChoreographer.postFrameCallback(mFrameCallback);
            } else {
                mUIThreadHandler.post(mRebindRunnable);
            }
        }
    }
}
```
mRebindRunnable 的实现如下

```java
public abstract class ViewDataBinding extends BaseObservable implements ViewBinding {
    ...
    private final Runnable mRebindRunnable = new Runnable() {
        @Override
        public void run() {
            ....
            executePendingBindings();
        }
    };
    
    public void executePendingBindings() {
        if (mContainingBinding == null) {
            executeBindingsInternal();
        } else {
            mContainingBinding.executePendingBindings();
        }
    }
    
    private void executeBindingsInternal() {
        ....
        mRebindHalted = false;
        if (mRebindCallbacks != null) {
            mRebindCallbacks.notifyCallbacks(this, REBIND, null);
            // The onRebindListeners will change mPendingHalted
            if (mRebindHalted) {
                mRebindCallbacks.notifyCallbacks(this, HALTED, null);
            }
        }
        if (!mRebindHalted) {
            // 这里触发UI的刷新
            executeBindings();
            if (mRebindCallbacks != null) {
                // 发送notify
                mRebindCallbacks.notifyCallbacks(this, REBOUND, null);
            }
        }
        mIsExecutingPendingBindings = false;
    }
}
```

其内部通过Choreographer 监听新一帧的刷新，触发UI的刷新（调用setText方法），这样有一个好处是，我们可以在子现场调用set方法更新bean数据。

# 后记

viewBinding，DatatBinding与编译时注解有一点点类似，都是在编译的时候生成对应的模板代码，通过这种方式让我们免去写过多findViewById，和setText这种代码，同时也能帮我们解决异步更新ui的问题。但DatatBinding这种实现要依赖于Android Studio去实现模板代码。

用DataBinding很容易的实现数据的双向绑定，也就是MVVM模式。但是里面还是有一定的学习成本，而且不方便debug，各取所需吧。

——Weiwq  于 2021.10 广州

