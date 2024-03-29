---
layout:     post
title:      "Android右到左布局适配方案"
subtitle:   " \"不曾停止\""
date:       2018-12-10 22:00:00
author:     "Weiwq"
header-img: "img/background/post-bg-js-module.jpg"
catalog: true
tags:
    - Android
---

> “好久没有写blog了，感觉工作了都没有自己的时间了~~”

&emsp;&emsp;呜呼，伊朗的项目终于做完了，大部分都是在整理右到左布局的需求。好在android sdk 从API17（Android4.2）开始支持右到左布局的需求，但是会有很多坑需要去填。

&emsp;&emsp;Android中的大部分组件是支持右到左布局的，只需要在Androidmanifest中配置如下：

```java
    <application
         ....
        android:supportsRtl="true">
    </application>
```

我们先看一个demo，MainActivity对应的布局文件如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.example.test.activity.MainActivity">
    <android.support.v7.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
        android:background="@color/fastlane_background"
        />
    <TextView
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:gravity="center_vertical"
        android:text="hello world "
        android:background="@color/selected_background"/>
</LinearLayout>
```

我们可以动态设置系统的语言来模仿用户环境，如下将app的配置设置为波斯语，当然，最简单的是到手机设置中直接设置当前语言为波斯语（设置有风险，语言需谨慎～～）：

```java
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Locale locale = new Locale("fa_IR");//fa_IR是波斯语（伊朗）的代码
        Locale.setDefault(locale);
        Configuration config = new Configuration();
        config.locale = locale;
        DisplayMetrics dm = getResources().getDisplayMetrics();
        getResources().updateConfiguration(config, dm);
        setContentView(R.layout.activity_main);
    }
```

以下是没有加android:supportsRtl="true"的布局，很正常不过的布局吧？

![在这里插入图片描述](/img/blog_android_rtl/img1.png)

那么加了之后呢？

![在这里插入图片描述](/img/blog_android_rtl/img2.png)

显然toolbar的内容已经反过来了。且慢，textview好像没有任何变化?_?，难道textview不支持右到左布局？

## 1、textview gravity的右到左布局适配

大家知道textview有一个gravity的属性，其中可以通过以下两种方式指定textview的内容在左边还是在右边

&emsp;&emsp;1) left，right：绝对位置，指定内容显示在textview的左边还是右边，这个比较好理解

&emsp;&emsp;2) start，end：这个大家应该见过，该值作用的结果与系统布局方向有关，如下所示:

![在这里插入图片描述](/img/blog_android_rtl/img3.png)

&emsp;&emsp;举一反三，drawableStart，drawableEnd和layout_marginStart，layout_marginEnd以及layout_toStartOf，layout_toEndOf等也是同理，都是相对布局方向而言的。

&emsp;&emsp;有同学马上就会想到，直接指定textview的gravity为start不就解决上面的问题了嘛！修改后的textview如下：

```xml
    <TextView
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:gravity="center_vertical|start"
        android:text="hello world "
        android:background="@color/selected_background"/>
```

效果如下：

![在这里插入图片描述](/img/blog_android_rtl/img4.png)

不对啊，这不按套路出牌呀。内容怎么还是在左边。。。

同学们发现没有，内容是英文的，而伊朗是用波斯语言，会不会与语言有关呢？我们再加一个textView显示波斯语言的：

```xml
    <TextView
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:background="@color/fti_left_color"
        android:gravity="center_vertical|start"
        android:text="فارسی" />
```

结果如下：

![在这里插入图片描述](/img/blog_android_rtl/img5.png)

是不是反过来了？由于波斯语言是右到左显示的，所以其内容也是默认居右边的。这说明，textview 的内容布局位置与文字显示的方向相关的。为了与伊朗用户的用户习惯保持一致，就算textview的内容是英文的，也应该居右边显示，那么如何做到呢？

 Android提供了Java代码动态获取当前布局方向，如下：

```java
    /**
    * 
    * @param context
    * @return 如果是右到左布局，返回true
    */
   public boolean isRtl(Context context) {
       return context.getResources().getConfiguration().getLayoutDirection() == View.LAYOUT_DIRECTION_RTL;
   }
```

 然后我们可以通过该方法设置textView的gravity

```java
  textView.setGravity(isRtl(this) ? Gravity.RIGHT:Gravity.LEFT);
```

结果大家显然都知道了，这里就不列出来了。

但是这样还是觉得很麻烦，有没有更便捷的属性？有！

| name                           | description                                                 | 描述            |
| ------------------------------ | ----------------------------------------------------------- | ------------- |
| android:layoutDirection        | attribute for setting the direction of a component's layout | 设置组件的布局排列方向   |
| android:textDirection          | attribute for setting the direction of a component's text   | 设置组件的文字排列方向   |
| android:textAlignment          | attribute for setting the alignment of a component's text   | 设置文字的对齐方式     |
| getLayoutDirectionFromLocale() | method for getting the Locale-specified direction           | 获取指定地区的惯用布局方式 |

其中，android:layoutDirection和android:textDirection的值有：

| value   | type | description                                                                                                      |
| ------- | ---- | ---------------------------------------------------------------------------------------------------------------- |
| inherit | int  | Horizontal layout direction is inherited <br> 继承水平方向                                                             |
| rtl     | int  | Horizontal layout direction is from Right to Left <br>布局方向是右到左                                                   |
| ltr     | int  | Horizontal layout direction is from Left to Right <br>布局方向是左到右                                                   |
| locale  | int  | Horizontal layout direction is deduced from the default language script for the locale <br>从区域设置的默认语言脚本推导出水平布局方向 |

android:textAlignment的值有：

| Constant  | value | Description                                                                                                                                                                                      |
| --------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| center    | 4     | Center the paragraph, for example: ALIGN_CENTER.<br> 居中                                                                                                                                          |
| gravity   | 1     | Default for the root view. The gravity determines the alignment, ALIGN_NORMAL, ALIGN_CENTER, or ALIGN_OPPOSITE, which are relative to each paragraph’s text direction.<br>由view通过gravity属性定义对齐方式 |
| inherit   | 0     | Default. <br> 默认                                                                                                                                                                                 |
| textEnd   | 3     | Align to the end of the paragraph, for example: ALIGN_OPPOSITE. <br> 与段落的末尾对齐                                                                                                                    |
| textStart | 2     | Align to the start of the paragraph, for example: ALIGN_NORMAL.<br> 与段落的开头对齐                                                                                                                     |
| viewEnd   | 6     | Align to the end of the view, which is ALIGN_RIGHT if the view’s resolved layoutDirection is LTR, and ALIGN_LEFT otherwise.<br> 与view的末尾对齐，                                                      |
| viewStart | 5     | Align to the start of the view, which is ALIGN_LEFT if the view’s resolved layoutDirection is LTR, and ALIGN_RIGHT otherwise.<br> 与view的开头对齐                                                     |

还是原来的布局，这里多加一行

```java
  <TextView
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:background="@color/selected_background"
        android:gravity="center_vertical|start"
        android:textDirection="locale"
        android:text="hello world " />
```

效果如下

![在这里插入图片描述](/img/blog_android_rtl/img6.png)

要给辣么多个textview添加该属性，多麻烦！没事，有全局的设置～～

```java
<resources>

    <style name="AppTheme" parent="@style/Theme.AppCompat.Light.NoActionBar" >
        <item name="android:textViewStyle">@style/TextDirection</item>
    </style>
    <style name="TextDirection" parent="android:Widget.TextView">
        <item name="android:textDirection">locale</item>
    </style>
</resources>
```

然后在Androidmanifest文件中引用该style

```xml
    <application
        ......
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        .......
    </application>
```

类似的还有EditText

```java
    <style name="AppTheme" parent="@style/Theme.AppCompat.Light.NoActionBar">
         .....
        <item name="editTextStyle">@style/EditTextStyle</item>
    </style>

    <style name="EditTextStyle" parent="@android:style/Widget.EditText">
        <item name="android:textAlignment">viewStart</item>
        <item name="android:gravity">start</item>
        <item name="android:textDirection">locale</item>
    </style>
```

## 2、组件之间的相对位置

在处理组件之间的相对位置，一般会用到layout_marginLeft，layout_marginRight或者layout_toLeftOf，layout_toRightOf等。那么如果你的app需要兼顾右到左布局的时候，这些布局约束就不合适了，需要用Start，End来代替Left，Right。天呐，如果有几十上百个布局文件，不是要改死程序员了么？别怕，Android studio已经为我们提供了一键适配右到左布局的需求，如下图所示

![在这里插入图片描述](/img/blog_android_rtl/img7.png)

然后会弹出一个对话框，直接点击“run”

![在这里插入图片描述](/img/blog_android_rtl/img8.png)

接着会提示你哪些需要添加适配属性的，直接点击“Do Refactor”

![在这里插入图片描述](/img/blog_android_rtl/img9.png)

如果有不需要修改的，可以选中不修改的文件，或者某一行代码。选择Remove即可

![在这里插入图片描述](/img/blog_android_rtl/img10.png)

这样我们的布局文件就自定添加start、end属性。
如果布局中使用了drawableStart或者drawableEnd属性就需要注意了，需要考虑图标是否需要跟随布局方向变化而变化，例如作为方向标示的就不应该变化位置，因此需要依旧使用drawableLeft和drawableRight

## 3、start，end？

start,end不是已经讲过了吗？很简单，是相对布局方向的。真的是这样的吗？我们来看一段代码

```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".activity.RelativeActivity">

    <Button
        android:id="@+id/button1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentTop="true"
        android:text="button1" />

    <Button
        android:id="@+id/button2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentTop="true"
        android:layout_marginStart="100dp"
        android:layout_marginEnd="20dp"
        android:layout_toEndOf="@+id/button1"
        android:layoutDirection="ltr"
        android:paddingStart="50dp"
        android:text="button2"/>

    <Button
        android:id="@+id/button3"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentTop="true"
        android:layout_marginStart="20dp"
        android:layout_toEndOf="@+id/button2"
        android:text="button3" />

</RelativeLayout>
```

预览效果如下

![在这里插入图片描述](/img/blog_android_rtl/img17.jpg)

button2同时设置了layout_marginStart和paddingStart，并且指定layoutDirection的方向为ltr。一切是如此和谐～～跑出来的效果如下：

![在这里插入图片描述](/img/blog_android_rtl/img18.jpg)

哎，怎么同是start，作用的位置与预期不一致呢？这里画一张图就明白了：

![在这里插入图片描述](/img/blog_android_rtl/img19.jpg)

当然，如果不手动指定layoutDirection的方向，也就不会出现这个情况了。

从这里可以看出来：start，end是优先考虑view本身的布局方向，其次才是受父viewgroup的布局方向限制

## 4、布局约束的右到左布局适配

在开发中，有时需要动态移动view的位置，一般的做法是修改view的layout，设置x，y轴或者设置marginLayoutParams等，当然还有一般的动画。
还是我们的mainActivity，对应的布局有两个textview1，textview2，点击textview2会修改textview1的位置，对应的代码如下：

```java
    private TextView textview1, textview2;
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
//        Locale locale = new Locale("fa_IR");
//        Locale.setDefault(locale);
//        Configuration config = new Configuration();
//        config.locale = locale;
//        DisplayMetrics dm = getResources().getDisplayMetrics();
//        getResources().updateConfiguration(config, dm);
        setContentView(R.layout.activity_main);
        textview1 = findViewById(R.id.textview1);
        textview2 = findViewById(R.id.textview2);
    }
    @Override
    public void onClick(View v) {
        if (v.getId() == R.id.textview2) {
            ViewGroup.MarginLayoutParams marginLayoutParams = (ViewGroup.MarginLayoutParams) textview1.getLayoutParams();
            marginLayoutParams.leftMargin += 20 ;
            marginLayoutParams.topMargin += 10;
            textview1.setLayoutParams(marginLayoutParams);
        }
    }
```

效果如果，我们成功的移动了textview1

![在这里插入图片描述](/img/blog_android_rtl/img11.gif)

我们再解除被注释掉的代码，再试试看：

![在这里插入图片描述](/img/blog_android_rtl/img12.gif)

怎么没有往右下角移动?_? 不是已经设置了leftMargin了么，怎么只有topMargin有效？

在右到左布局下，Android的会从右到左依次绘制view，虽然我们设置了leftMargin，但是，系统在绘制view的时候会以rightMargin约束条件为准，如下图所示，所以就导致了leftMargin无效的问题。

需要注意的是，在右到左布局下，系统坐标并没有改变，getLeft,getTop,getX,getY等值没有发生变化。

![在这里插入图片描述](/img/blog_android_rtl/img13.png)

所以，我们需要稍作修改 

```java
  @Override
    public void onClick(View v) {
        if (v.getId() == R.id.textview2) {
            ViewGroup.MarginLayoutParams marginLayoutParams = (ViewGroup.MarginLayoutParams) textview1.getLayoutParams();
            marginLayoutParams.rightMargin += 20 ;
            marginLayoutParams.topMargin += 10;
            textview1.setLayoutParams(marginLayoutParams);
        }
    }
```

结果如下：

![在这里插入图片描述](/img/blog_android_rtl/img14.gif)

那如果在右到左布局下，textview的移动动画与左到右布局下一致呢？很简单，同时设置leftMargin和rightMargin即可：

```java
    @Override
    public void onClick(View v) {
        if (v.getId() == R.id.textview2) {
            int screenWidth = getWindowManager().getDefaultDisplay().getWidth() ;
            ViewGroup.MarginLayoutParams marginLayoutParams = (ViewGroup.MarginLayoutParams) textview1.getLayoutParams();
            marginLayoutParams.leftMargin += 20;
            marginLayoutParams.rightMargin = screenWidth
                - textview1.getWidth()-marginLayoutParams.leftMargin ;
            marginLayoutParams.topMargin += 10;
            textview1.setLayoutParams(marginLayoutParams);
        }
    }
```

效果如下

![在这里插入图片描述](/img/blog_android_rtl/img15.gif)

## 5、英文，波斯文，阿拉伯数字混排

在项目中，由于英文是强左到右，波斯文是强右到左的，在混排的时候，就会存在文字方向的冲突，需要unicode来强制指定文字的方向，这里做总结。

| unicode | 说明                                | 翻译：代码                     |
| ------- | --------------------------------- | ------------------------- |
| \u202A  | 将英文，阿拉伯数字等非强右到左文字放在波斯文等强RTL文字的右边  | 早上6点： قبل از ظهر  \u202A6 |
| \u200F  | 也是将英文，阿拉伯数字放在波斯文的右边，区别是与波斯文之间没有空格 | 早上6点：قبل از ظهر\u200F6    |
| \u202E  | 获取文字的镜像文                          | 12: \u202E21              |

在显示文件路径时候，如果包含波斯语，会导致路径显示错误的情况，需要在string的前后加上\u202D 和\u202C（更多参考([UTF-8 常用标点符号](https://www.runoob.com/charsets/ref-utf-punctuation.html))）

```java
        String content = "sdcard\\فارسی\\3D\\فارسی.mp3";
        textview1.setText("\u202D"+content+"\u202C");
```

实际显示如下：

![在这里插入图片描述](/img/blog_android_rtl/img16.png)

或者使用如下接口

```java
    // 获取ltr的字符串
    public static String formatLtrString(String content) {
        return BidiFormatter.getInstance().unicodeWrap(content, TextDirectionHeuristicsCompat.LTR);
    }
```

## 总结

1、如果使用到drawableLeft、drawableRight属性，请确定图标是否需要跟随系统布局方法改变而改变，如果是，请修改为drawableStart、drawableEnd；反之不用

2、在代码中需要动态修改view的布局约束，例如marginLayoutParams，layoutParams.setMargins()等，需要同时考虑left和right，或者采用修改layout的方式。

3、获取view的位置可以通过以下代码：

```java
     int[] location = new int[2];
     view.getLocationOnScreen(location);
     float x = location[0];
     float y = location[1]
```

4、在relativelayout中添加layout rule的时候，用START_OF、END_OF代替LEFT_OF、RIGHT_OF。

5、在右到左布局下，其坐标布局方式不变，getLeft，getRight等不变。变的是布局约束，view的绘制受right（start）约束。

6、一些方向图标，重新做一个相对方向的放到 mipmap-ldrtl-xxxhdpi 包下

7、textview，editText需要对内容做RTL处理，设置style，详情见第一节

8、如果需要修改view的位置，请不要使用setX，setY方式。推荐采用ViewGroup.MarginLayoutParams来设置margin，或者采用layout的方式。同时需要注意该约束是相对父view而言的。

9、判断字符串是否是强RTl，常常用于控制textDirection

```java
   /**
     * 判断是否是强右到左字符
     * 如果包含强右到左字符，返回true，反之返回false
     * @param content 字符
     * @return 是否是右到左字符
     */
    protected boolean isRtlString(String content) {
        if (TextUtils.isEmpty(content)) {
            return false;
        }
        return TextDirectionHeuristics.ANYRTL_LTR.isRtl(content, 0, content.length());
    }
}
```

—— Weiwq 于 2018.12 广州
