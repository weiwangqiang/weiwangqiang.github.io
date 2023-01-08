---
layout:     post
title:      "可以控制移动方向的跑马灯"
subtitle:   " \"基于textview设计\""
date:       2019-02-3 10:10:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - Android
---

> “跑马灯？so easy～～”


## 引言

最近客户提了一个很小很小但是很恶心的需求：在波斯语言环境（右到左布局）下，英文需要从右到左移动，波斯文需要从左到右移动。

第一反应是，这是什么玩意啊，textview的跑马灯方向是Android控制的。不过转念一想也是，不同文字，读取的方向是不一样的，需要做适配。在研究了一波textview源码之后，产出了这篇blog

## 1、 textview的源码解析

这里就直接看onDraw方法，其中有一段代码是这样的：

```java
        final int layoutDirection = getLayoutDirection();
        final int absoluteGravity = Gravity.getAbsoluteGravity(mGravity, layoutDirection);
        if (isMarqueeFadeEnabled()) {
            if (!mSingleLine && getLineCount() == 1 && canMarquee()
                    && (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) != Gravity.LEFT) {
                final int width = mRight - mLeft;
                final int padding = getCompoundPaddingLeft() + getCompoundPaddingRight();
                final float dx = mLayout.getLineRight(0) - (width - padding);
                canvas.translate(layout.getParagraphDirection(0) * dx, 0.0f);
            }

            if (mMarquee != null && mMarquee.isRunning()) {
                final float dx = -mMarquee.getScroll();
                canvas.translate(layout.getParagraphDirection(0) * dx, 0.0f);
            }
        }
```

这里先获取到当前布局方向，然后设置偏移量，其中getLayoutDirection方法是可以重写的，于是尝试一下，创建ControllableMarquee类

```java
public class ControllableMarquee extends TextView {
    private int LAYOUT_DIRECTION = LAYOUT_DIRECTION_RTL;

    public ControllableMarquee(Context context) {
        super(context);
    }

    public ControllableMarquee(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public ControllableMarquee(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    public boolean isFocused() {
        return true;
    }

    @Override
    public int getLayoutDirection() {
        return LAYOUT_DIRECTION;
    }

    public void setText(String content, boolean moveToLeft) {
        LAYOUT_DIRECTION = moveToLeft ? LAYOUT_DIRECTION_LTR : LAYOUT_DIRECTION_RTL;
        setText(content);
    }
}

```

创建一个MarqueeActivity

```java
public class MarqueeActivity extends BaseActivity {

    private ControllableMarquee marqueeDirectionTextView1;
    private ControllableMarquee marqueeDirectionTextView2;
    private static final String TAG = "MarqueeActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_marquee);
        marqueeDirectionTextView1 = findViewById(R.id.marqueeTextView1);
        marqueeDirectionTextView2 = findViewById(R.id.marqueeTextView2);
        marqueeDirectionTextView1.setText("کتابخانه صوتی خالی استکتابخانه صوتی خالی استکتابخانه صوتی خالی است ", true);
        marqueeDirectionTextView2.setText("afewfeohtebqofqodhqbfevwbfoqbwdoqbdowq", false);
    }

    @Override
    public boolean isRtl() {
        return true;
    }
}
```
对应的xml布局如下：

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="20dp"
    tools:context=".activity.MarqueeActivity">

    <TextView
        android:layout_width="100dp"
        android:layout_height="30dp"
        android:background="@color/colorAccent"
        android:ellipsize="marquee"
        android:focusable="true"
        android:marqueeRepeatLimit="marquee_forever"
        android:singleLine="true"
        android:text="کتابخانه  صوتی خالی استکتابخانه صوتی خالی است " />

    <TextView
        android:layout_width="100dp"
        android:layout_height="30dp"
        android:background="@color/colorPrimary"
        android:ellipsize="marquee"
        android:focusable="true"
        android:marqueeRepeatLimit="marquee_forever"
        android:singleLine="true"
        android:text="fewfwefwefwefwecwefdwdwq "/>


    <demo.com.rtldemo.view.ControllableMarquee
        android:id="@+id/marqueeTextView1"
        android:layout_width="100dp"
        android:layout_height="30dp"
        android:background="@color/colorAccent"
        android:ellipsize="marquee"
        android:focusable="true"
        android:marqueeRepeatLimit="marquee_forever"
        android:singleLine="true"
        android:text=" تنظیمتنظیم تنظیم ضبط:"/>

    <demo.com.rtldemo.view.ControllableMarquee
        android:id="@+id/marqueeTextView2"
        android:layout_width="100dp"
        android:layout_height="30dp"
        android:background="@color/colorPrimary"
        android:ellipsize="marquee"
        android:focusable="true"
        android:marqueeRepeatLimit="marquee_forever"
        android:singleLine="true"
        android:text="adbcdffeqeqqweqdwqd " />

</LinearLayout>
```

其中BaseActivity如下，主要设置当前应用的语言环境

```java
public abstract class BaseActivity extends AppCompatActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Locale locale = new Locale("fa_IR");
        Locale.setDefault(locale);
        Configuration config = new Configuration();
        config.locale = locale;
        DisplayMetrics dm = getResources().getDisplayMetrics();
        if (isRtl()) {
            getResources().updateConfiguration(config, dm);
        }
    }

    public abstract boolean isRtl();
}
```

效果如下：

![在这里插入图片描述](/img/blog_controllable_marquee/img1.gif)

这是在模拟器上跑的，并没有任何效果，但是我在tv上测试是没有问题的，有适配的需要。

## 2、TextDirection

上面直接设置layoutDirection不一定有效果，那么得接着看源码
在textview画string的时候，注意到这几行代码：
```java
        if (mEditor != null) {
            mEditor.onDraw(canvas, layout, highlight, mHighlightPaint, cursorOffsetVertical);
        } else {
            layout.draw(canvas, highlight, mHighlightPaint, cursorOffsetVertical);
        }
```
这里的layout是画string的主要实现，那么layout是如何创建的？很容易就找到makeSingleLayout方法：

```java
    /**
     * @hide
     */
    protected Layout makeSingleLayout(int wantWidth, BoringLayout.Metrics boring, int ellipsisWidth,
            Layout.Alignment alignment, boolean shouldEllipsize, TruncateAt effectiveEllipsize,
            boolean useSaved) {
        Layout result = null;
        if (useDynamicLayout()) {
            final DynamicLayout.Builder builder = DynamicLayout.Builder.obtain(mText, mTextPaint,
                    wantWidth)
                    .setDisplayText(mTransformed)
                    .setAlignment(alignment)
                    .setTextDirection(mTextDir)
                    .setLineSpacing(mSpacingAdd, mSpacingMult)
                    .setIncludePad(mIncludePad)
                    .setUseLineSpacingFromFallbacks(mUseFallbackLineSpacing)
                    .setBreakStrategy(mBreakStrategy)
                    .setHyphenationFrequency(mHyphenationFrequency)
                    .setJustificationMode(mJustificationMode)
                    .setEllipsize(getKeyListener() == null ? effectiveEllipsize : null)
                    .setEllipsizedWidth(ellipsisWidth);
            result = builder.build();
        }
        .......
     }
```

这里有一个setTextDirection方法，来设置layout的text布局方向，对于direction会下意识的保持高度警惕，那么如何设置mTextDir呢？接着看：

有这么一段代码：

```java
        mTextDir = getTextDirectionHeuristic();
```

那就接着看getTextDirectionHeuristic方法：

```java
    protected TextDirectionHeuristic getTextDirectionHeuristic() {
			......    
        // Now, we can select the heuristic
        switch (getTextDirection()) {
            default:
            case TEXT_DIRECTION_FIRST_STRONG:
                return (defaultIsRtl ? TextDirectionHeuristics.FIRSTSTRONG_RTL :
                        TextDirectionHeuristics.FIRSTSTRONG_LTR);
            case TEXT_DIRECTION_ANY_RTL:
                return TextDirectionHeuristics.ANYRTL_LTR;
            case TEXT_DIRECTION_LTR:
                return TextDirectionHeuristics.LTR;
            case TEXT_DIRECTION_RTL:
                return TextDirectionHeuristics.RTL;
            case TEXT_DIRECTION_LOCALE:
                return TextDirectionHeuristics.LOCALE;
            case TEXT_DIRECTION_FIRST_STRONG_LTR:
                return TextDirectionHeuristics.FIRSTSTRONG_LTR;
            case TEXT_DIRECTION_FIRST_STRONG_RTL:
                return TextDirectionHeuristics.FIRSTSTRONG_RTL;
        }
    }
```

好，重写getTextDirection方法咯

## 3、ControllableMarquee实现

```java
public class ControllableMarquee extends TextView {
    private int LAYOUT_DIRECTION = LAYOUT_DIRECTION_RTL;
    private int TEXT_DIRECTION = TEXT_DIRECTION_RTL;

    public ControllableMarquee(Context context) {
        super(context);
    }

    public ControllableMarquee(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public ControllableMarquee(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    public boolean isFocused() {
        return true;
    }

    @Override
    public int getLayoutDirection() {
        return LAYOUT_DIRECTION;
    }

    public int getTextDirection() {
        return TEXT_DIRECTION;
    }


    public void setText(String content, boolean moveToLeft) {
        LAYOUT_DIRECTION = moveToLeft ? LAYOUT_DIRECTION_LTR : LAYOUT_DIRECTION_RTL;
        TEXT_DIRECTION = moveToLeft ? TEXT_DIRECTION_LTR : TEXT_DIRECTION_RTL;
        setText(content);
    }
}

```

效果如下：

![在这里插入图片描述](/img/blog_controllable_marquee/img2.gif)

## 后记

本篇主要通过重写textview的getLayoutDirection和getTextDirection方法，就能实现控制跑马灯的方向，避免了textview无法控制方向的问题

明天就要过年了，致仍然在公司奋斗的我和你们～～～新年快乐！

—— Weiwq 后记于 2019.02 广州


