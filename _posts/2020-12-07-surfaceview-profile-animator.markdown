---
layout:     post
title:      "教你用SurfaceView画高性能的动画"
subtitle:   " \"妈妈再也不用担心动画卡顿了\""
date:       2020-08-20 22:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - Android

---

> “下面会用github上的一个开源drawable动画框架做性能上的对比“

# 引言

在Android上处理简单的动画，估计大家首先想到的是视图动画，补间动画，属性动画这些。再复杂一些就是重写view的onDraw方法，自己用canvas画视图，然后再调用invalid方法。这样动画是出来了，可性能呢？高级一些的框架，比如 [Android-SpinKit](https://github.com/ybq/Android-SpinKit) 就用到了继承drawable的方式画动画，这是一个不错的思路。笔者也用模拟器跑了一下里面的动画，感觉效果还不错。但是经常出现过一会儿，动画就卡住不动了，这就很头疼。

上面说的动画，都是在主线程中实现的，如果我们的主线程卡了，那么动画自然也会卡，会让人很难受。有没有更好的方案呢？答案是有的，我们还有surfaceView神器！

# 1、性能对比

话不多说，我们先直接上对比结果，对比方案就是上面提到的Android-SpinKit 框架。这里笔者画了一个类似的动画。

<img src="/img/blog_surface_animator_height_profile/surface_1.gif" width="50%" height="40%">  ![在这里插入图片描述](/img/blog_surface_animator_height_profile/DoubleBounce.gif)

## CPU对比

执行top命令，查看对应进程的cpu占用率。

其中com.demo.surfaceanimator 进程为笔者写的demo，可以看出，稳定在21%（单核）左右

![在这里插入图片描述](/img/blog_surface_animator_height_profile/surface_2.png)

而进程 com.github.ybq.android.spinkit （为Android-SpinKit 项目），其稳定在43%左右

![在这里插入图片描述](/img/blog_surface_animator_height_profile/compare_2.png)



## GPU 渲染对比

由于GPU的渲染柱状图不支持surfaceview，这里只截取了Android-SpinKit 项目的渲染图，可以看出SpinKit还是很优秀的。

<img src="/img/blog_surface_animator_height_profile/compare_3.png" width="30%" height="30%">

## profile对比

为了保证严谨性，这里用Android studio中的profile工具对比，需要注意的是，这里的CPU是整个系统百分比，会与上面的不太一样。

下图是笔者的demo性能

<img src="/img/blog_surface_animator_height_profile/surface_4.png" width="100%" height="30%">

这个是SpinKit 项目的结果

<img src="/img/blog_surface_animator_height_profile/compare_4.png" width="100%" height="30%">

从上面可以看出，内存占用差不多（空apk的内存占用也很高）cpu占比基本差一倍。

# 2、surfaceView实现动画

从上一小节可以看出，用surfaceView 画动画还是有优势的，由于surface可以在子线程绘制，即使在主线程卡顿的时候，也能实现流畅的动画。上code～～

我们先看SurfaceAnimatorView实现

```java


public class SurfaceAnimatorView extends SurfaceView implements SurfaceHolder.Callback {

    private IAnimator iAnimator;
    private Thread mRenderThread;
    private SurfaceHolder surfaceHolder;
    private Paint mPaintClear;
    // 控制帧率
    private int fps = 1000 / 35; 
    private volatile boolean isStop = false;
    // 负责不断绘制的runnable，在子线程执行
    private Runnable mRenderRunnable = new Runnable() {
        @Override
        public void run() {
            while (!isStop) {
                long start = System.currentTimeMillis();
                onDrawAnimator();
                long spendTime = System.currentTimeMillis() - start;
                try {
                    if ((fps - spendTime) > 0) {
                        Thread.sleep(fps - spendTime);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    };

    private void onDrawAnimator() {

        // 从surfaceHolder 获取离屏的canvas 
        Canvas canvas = surfaceHolder.lockCanvas();
        if (canvas == null) {
            return;
        }
        // 清屏。这步很重要，不然画布会有上次绘制的内容
        canvas.drawPaint(mPaintClear);
        // 将画布给animator，实现对应的动画
        iAnimator.onDraw(canvas);
        // 释放canvas
        surfaceHolder.unlockCanvasAndPost(canvas);
    }

    public SurfaceAnimatorView(Context context) {
        super(context);
        init();
    }

    public SurfaceAnimatorView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public void init() {
        setFocusable(true);
        if (surfaceHolder == null) {
            surfaceHolder = getHolder();
            surfaceHolder.addCallback(this);
        }
        this.setZOrderOnTop(true);
        this.getHolder().setFormat(PixelFormat.TRANSLUCENT);
        mPaintClear = new Paint();
        mPaintClear.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.CLEAR));
        setBackgroundColor(getContext().getResources().getColor(R.color.teal_200));
    }


    @Override
    public void surfaceCreated(SurfaceHolder surfaceHolder) {
        // 初始化animator 
        iAnimator = new CircleLoadingAnimator(getContext());
        iAnimator.onLayout(getWidth(), getHeight());
        isStop = false;
        // 启动绘制的线程
        mRenderThread = new Thread(mRenderRunnable);
        mRenderThread.start();
    }

    @Override
    public void surfaceChanged(SurfaceHolder surfaceHolder, int i, int i1, int i2) {
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder surfaceHolder) {
        isStop = true;
        try {
            mRenderThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

这里抽象出IAnimator 接口，其中CircleLoadingAnimator 是IAnimator 的实现类。为了方便，这里就写一块啦～

````java

public interface IAnimator {
    void onLayout(int width, int height);

    void onDraw(Canvas canvas);
}

// 具体实现动画的类
public class CircleLoadingAnimator implements IAnimator {
    private DecelerateInterpolator fastToSlow = new DecelerateInterpolator();

    // 每帧变化的幅度，越小越慢，帧区别也越小
    private int INCREASE_VALUE = 4;
    private int MAX_CIRCLE_RADIUS = 200;
    private int CIRCLE_RADIUS = MAX_CIRCLE_RADIUS >> 1;
    private int mSmallCircleRadius = 0;
    private int mBigCircleRadius = MAX_CIRCLE_RADIUS;
    private int increaseValue = -1;
    private int recordValue = MAX_CIRCLE_RADIUS;
    private Paint mSmallPaint = new Paint();
    private Paint mBigPaint = new Paint();
    private int mX;
    private int mY;
    private static final String TAG = "CircleLoadingAnimator";

    public CircleLoadingAnimator(Context context) {
        mBigPaint.setStyle(Paint.Style.FILL);
        mBigPaint.setStrokeCap(Paint.Cap.ROUND);
        mBigPaint.setAntiAlias(true);
        mBigPaint.setColor(context.getResources().getColor(R.color.white_90));

        mSmallPaint.setStyle(Paint.Style.FILL);
        mSmallPaint.setStrokeCap(Paint.Cap.ROUND);
        mSmallPaint.setAntiAlias(true);
        mSmallPaint.setColor(context.getResources().getColor(R.color.white));
    }

    @Override
    public void onLayout(int width, int height) {
        mX = width >> 1;
        mY = height >> 1;
    }

    @Override
    public void onDraw(Canvas canvas) {
        updateInCreaseValue();
        recordValue += increaseValue;
        // 模拟属性动画 0 - > 1 的过程
        float value = (float) ((MAX_CIRCLE_RADIUS - recordValue) * 1.0 / (CIRCLE_RADIUS));
        // 更新圆半径
        mBigCircleRadius = (int) (fastToSlow.getInterpolation(1 - value) * CIRCLE_RADIUS + CIRCLE_RADIUS);
        mSmallCircleRadius = (int) (CIRCLE_RADIUS - fastToSlow.getInterpolation(1 - value) * CIRCLE_RADIUS);
        // 画圆
        canvas.drawCircle(mX, mY, mSmallCircleRadius, mSmallPaint);
        canvas.drawCircle(mX, mY, mBigCircleRadius, mBigPaint);
    }

    // 更新边界值
    private void updateInCreaseValue() {
        if (mBigCircleRadius >= MAX_CIRCLE_RADIUS) {
            increaseValue = -1 * INCREASE_VALUE;
        } else if (mBigCircleRadius <= (CIRCLE_RADIUS)) {
            increaseValue = INCREASE_VALUE;
        }
    }
}

````

# 3、小结

​         在SurfaceView里面，可以通过控制帧率的方式控制cpu的消耗，通过测试，帧率控制在30到40之间，就能提高近50%的性能，又能尽可能保证用户体验，在性能比较差的平台上还是很有优势的。关于surfaceView的原理，其他博客已经有比较详细的介绍，这里就不做展开了。



——Weiwq  于 2020.12 广州