﻿---
layout:     post
title:      "Android单元测试全解"
subtitle:   " \"不曾停止\""
date:       2018-08-17 17:00:00
author:     "Weiwq"
header-img: "img/background/post-bg-2015.jpg"
catalog: true
tags:
    - Android
---

> “今天是七夕，写个blog纪念一下 ”

&ensp;&ensp;自动化测试麻烦吗？说实在，麻烦！有一定的学习成本。但是，自动化测试有以下优点：

- **节省时间：可以指定测试某一个activity，不需要一个个自己点**
- **单元测试：既然Java可以进行单元测试，Android为什么就不可以呢？**
- **一键适配：不解释**

Android自动化测试框架主要有：Espresso、UI Automator以及Robolectric。滴滴~~  开车开车！



## 1、Java单元测试

 Android studio（以下简称as）可以跑纯Java代码，这个想必大家都知道。这里就简单介绍一下as如何跑Java代码，作为热身运动吧！
 首先打开测试包，在app->src->test目录下，如下图所示，其中AndroidTest包是针对Android工作的测试，先不管他。
 ![这里写图片描述](https://img-blog.csdn.net/20180619195624409?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RvX3BlcmZlY3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 这里as为我们创建了一个测试类，直接打开，as中采用Junit4的测试包，主要代码如下。

```java
@RunWith(JUnit4.class)
public class ExampleUnitTest {
    @Before
    public void before(){
        //在测试前的工作
    }
    @After
    public void after()
    {
       // 测试完成后的工作
    }
    @Test
    public void addition_isCorrect() {
        //主要工作
    }
}
```

这就是最简单的Java测试，预热完毕，接下来进入本文的主角



## 2、Android单元测试——Espresso 

&ensp;&ensp;AndroidJUnitRunner类是一个JUnit测试运行器，它允许您在Android设备上运行JUnit 3或JUnit 4样式测试类，包括使用Espresso和UI Automator测试框架的测试类。
&ensp;&ensp;测试运行器与您的JUnit 3和JUnit 4（高达JUnit 4.10）测试兼容。 但是，您应该避免将JUnit 3和JUnit 4测试代码混合在同一个包中，因为这可能会导致意外的结果。 如果您正在创建一个测试JUnit 4测试类以在设备或模拟器上运行，那么您的测试类必须以@RunWith（AndroidJUnit4.class）注释为前缀。

先看应用程序的模块级build.gradle依赖：

```java
dependencies {
     androidTestCompile 'com.android.support:support-annotations:25.4.0'
    androidTestCompile 'com.android.support.test:runner:1.0.0' 
    androidTestCompile 'com.android.support.test:rules:1.0.0' 
    androidTestCompile 'com.android.support.test.espresso:espresso-core:3.0.2'
}
android {
    defaultConfig {
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
}
```

如果依赖冲突，请加入以下代码：

```java
androidTestImplementation('com.android.support.test.espresso:espresso-core:3.0.2', {
    exclude group: 'com.android.support', module: 'support-annotations'
})
```



### 2.1获取application

最常见的应用案例就是，在进行网络测试的时候，如果您的项目很大，编译的时间很长，那么单单为看一个请求结果就要花费相当长的时间，
这是不能容忍的，我们可以通过Android的单元测试来模拟请求，如下代码所示。

```java
@RunWith(AndroidJUnit4.class)
public class ExampleInstrumentedTest {
    @Before
    public void init() {
        Context appContext = InstrumentationRegistry.getTargetContext();
        x.Ext.init((Application) appContext.getApplicationContext());
    }

    @Test
    public void useAppContext() {
        // Context of the app under test.
        RequestParams requestParams = new RequestParams("https://www.baidu.com/");
        String str = x.http().getSync(requestParams, String.class);
        System.out.println("\n"+str+"\n");
    }
}
```

通过InstrumentationRegistry，我们就可以获取到context对象，再通过context就可以获取application对象，之后就可以构建一个网络请求，**请注意**，在测试方法中，必须使用**同步**请求，否则测试用例会直接忽略回调方法，直接结束程序，导致无法获取到请求结果。

Android单元测试分为：小型测试、中型测试，大型测试，他们的区别如下。

-  **小型测试（SmallTest）：与系统隔离运行，执行时间较短，最长执行时间为200ms**
- **中型测试（MediumTest）：集成了多个组件，并可以在模拟器或者真机上运行，最长执行时间为1000ms**
- **大型测试（LargeTest）：可以运行UI流程的测试工作，确保APP按照预期在仿真器或实际设备上工作。最长执行时间为1000ms**

在Android单元测试中可以使用断言来判断变量值是否符合预期，常用的有assertThat、assertEquals、assertNotSame等



### 2.2获取对应组件

该框架提供ActivityTestRule来管理被测试的activity，例如MainActivity对应的布局文件如下

```html
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <EditText
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text=""
        android:id="@+id/main_text"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</android.support.constraint.ConstraintLayout>
```

MainActivity的代码这里就不贴出来了，直接看测试代码：

```java
@RunWith(AndroidJUnit4.class)
@LargeTest
public class MainTest {
    @Rule
    public ActivityTestRule<MainActivity> mActivityRule = new ActivityTestRule<>(
            MainActivity.class);

    @Test
    public void Run() {
        onView(withId(R.id.main_text)).perform(
                typeText("Hello MainActivity!"), closeSoftKeyboard());
    }
}
```

这里简单说明一下：

- **withId(R.id.main_text)：通过ID找到对应的组件，并将其封装成一个Matcher**
- **onView()：将窗口焦点给某个组件，并返回ViewInteraction实例**
- **perform()：该组件需要执行的任务，传入ViewAction的实例，可以有多个，意味着用户的多种操作**
- **typeText()：输入字符串任务，还有replaceText方法也可以实现类似的效果，不过没有输入动画**
- **closeSoftKeyboard()：关闭软键盘**

以上就是最基本的自动化测试代码。点击Run方法边上的运行按钮，直接运行在设备上即可，效果如下所示。

<img src="https://img-blog.csdn.net/2018062019422866?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RvX3BlcmZlY3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" width = "40%" height = "40%"  />


类似的还有点击事件：

```java
onView(withId(R.id.main_text)).perform(click());
```

双击事件：

```java
onView(withId(R.id.main_text)).perform(doubleClick());
```

判断是否符合预期

```java
onView(withId(R.id.main_text)).check(matches(withText("Hello MainActivity!")));           
```

更多请看[ViewActions](https://developer.android.google.cn/reference/android/support/test/espresso/action/ViewActions)类提供的API



### 2.3模拟listView的点击事件

以上是针对唯一ID的事件，那么如果有多个组件的ID是一样的呢？例如模拟 listView的item点击事件，是如何区分每一个item呢？先看如何处理多个组件ID相同的情况。
大家知道可以通过ID来查找对应的视图，这里也可以通过显示的文本来查找视图：

```java
onView(withText("Hello MainActivity!"));
```

那么，如果通过ID和显示的文本不就可以定位唯一的视图了吗？如下

```java
onView(allOf(withId(R.id.main_text), withText("Hello MainActivity!")));
```

或者这样来筛选不匹配的视图

```java
onView(allOf(withId(R.id.button_signin), not(withText("Sign-out"))));
```

更多请看[ViewMatchers](https://developer.android.google.cn/reference/android/support/test/espresso/matcher/ViewMatchers)提供的API

接下来看如何模拟listview（GridView和Spinner均适用）的点击事件

我们先创建一个SecondActivity

```java
public class ListActivity extends AppCompatActivity {
    private ListView listView ;
    private List<HashMap<String ,String>> data = new ArrayList<>();
    public static final String KEY =  "key";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        listView  =  findViewById(R.id.list_view);
        initDate();
        listView.setAdapter(new SimpleAdapter(this,data,
                R.layout.item_list,
                new String[]{KEY},
                new int[]{R.id.item_list_text}));
        listView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                Toast.makeText(ListActivity.this,data.get(position).get(KEY),Toast.LENGTH_LONG).show();
            }
        });
    }

    private void initDate() {
        for(int i =0 ;i < 90 ;i++){
            HashMap<String,String> map = new HashMap<>();
            map.put(KEY,"第"+(1+i)+"列");
            data.add(map);
        }
    }
}
```

&ensp;&ensp;对应的布局文件就是一个listView，item对应的布局是一个textView，这里就不贴出来了，主要看测试类：

```java
@RunWith(AndroidJUnit4.class)
@LargeTest
public class ListViewTest {
    private static final String TAG = "ListViewTest ";
    @Rule
    public ActivityTestRule<ListActivity> mActivityRule = new ActivityTestRule<>(
            ListActivity.class);

    @Before
    public void init() {
        mActivityRule.getActivity();
    }

    @Test
    public void Run() {
        onData(allOf(is(instanceOf(Map.class)),
                hasEntry(equalTo(ListActivity.KEY), is("第10列")))).perform(click());
    }
}
```

&ensp;&ensp;这里选择数据为**第10行**的item，并执行点击动作，这里着重讲一下**hasEntry**() 这个方法，该方法需要传两个Matcher，也就是map的键名和对应的值。通过map的键、值来唯一确定一个item，拿到对应的item就可以类似于视图一样去执行动作了，效果如下。

<img src="https://img-blog.csdn.net/20180620211305528?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RvX3BlcmZlY3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" width = "40%" height = "40%"  />

&ensp;&ensp;动画比较快，但是可以看到listview先是滚到第10行，然后才执行点击事件，这是因为Espresso负责滚动目标元素，并将元素放在焦点上。

&ensp;&ensp;有同学马上就提出了，recycleView才是主流，用listview的很少了~~，没事，我们来看如何进行recycleView的自动化测试



### 2.4模拟recycleView点击事件

对recyclerView进行自动化测试需要再添加以下依赖，**注意**，是在之前的依赖基础上添加以下代码。

```java
androidTestCompile 'com.android.support.test.espresso:espresso-contrib:3.0.0'
androidTestCompile 'com.android.support:recyclerview-v7:25.4.0'
```

 我们创建一个RecyclerActivity，内容如下：
 

```java
public class RecyclerActivity extends AppCompatActivity {
    private RecyclerView recyclerView;
    private RecyclerAdapter<String> adapter;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_recycler);
        recyclerView = findViewById(R.id.recycler_view);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        adapter = new RecyclerAdapter<>(this, R.layout.item_list);
        recyclerView.setAdapter(adapter);
        List<String> list = new ArrayList<>();
        for(int i =0 ;i < 50 ;i++){
            list.add("第"+(1+i)+"列");
        }
        adapter.setData(list);
    }
}
```

&ensp;&ensp;对应的布局文件就是一个recyclerview，item的布局只有一个textView，这里也就不贴出来了，adapter也很简单，给textView一个点击事件，如下：

```java
public class RecyclerAdapter<T> extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
    private List<T> data = new ArrayList<>();
    private Context context ;
    private int layout;

    public RecyclerAdapter(Context context, int layout) {
        this.context = context;
        this.layout = layout;
    }

    public void setData(List<T> data) {
        this.data.clear();
        this.data.addAll(data);
        notifyDataSetChanged();
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        return new Holder(LayoutInflater.from(context)
                .inflate(layout,null,false));
    }


    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, final int position) {
        Holder holder1 = (Holder) holder;
        holder1.textView.setText(data.get(position).toString());
        holder1.itemView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(context,data.get(position).toString(),Toast.LENGTH_LONG).show();
            }
        });
    }

    @Override
    public int getItemCount() {
        return data.size();
    }
    private class Holder extends RecyclerView.ViewHolder{
        TextView textView ;
        public Holder(View itemView) {
            super(itemView);
            textView = itemView.findViewById(R.id.item_list_text);
        }
    }
}
```

接下来看测试类:

```java
@RunWith(AndroidJUnit4.class)
@LargeTest
public class RecycleViewTest {
    private static final String TAG = "ExampleInstrumentedTest";
    @Rule
    public ActivityTestRule<RecyclerActivity> mActivityRule = new ActivityTestRule<>(
            RecyclerActivity.class);

    @Test
    public void Run() {
        onView(ViewMatchers.withId(R.id.recycler_view))
                .perform(RecyclerViewActions.actionOnItemAtPosition(10, click()));

    }
}
```

&ensp;&ensp;在run方法中我们可以看到基本与之前的类似，不同的是需要通过RecyclerViewActions类提供的API来执行任务，其中actionOnItemAtPosition的第一个参数是recycleview的item位置，第二个参数是对应的动作，效果与listView的一致，这里就不贴了。
&ensp;&ensp;这里可以看出，recycleview的测试类要优于listView，listView通过item的值来查找对应的item，而recycleview直接通过位置来查找



### 2.5 模拟用户点击actionbar

新建一个MenuActivity，主要代码如下

```java
public class MenuActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_menu);
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.menu_test, menu);
        return super.onCreateOptionsMenu(menu);
    }
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        Toast.makeText(this,item.getTitle(),Toast.LENGTH_SHORT).show();
        return super.onOptionsItemSelected(item);
    }
}
```

menu布局代码如下：

```html
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:android1="http://schemas.android.com/apk/res-auto">
    <item
        android:id="@+id/nav_1"
        android:title="item1"
        android1:showAsAction="never" />
    <item
        android:id="@+id/nav_2"
        android:title="item2"
        android1:showAsAction="never" />
</menu>

```

测试代码如下：

```java
@RunWith(AndroidJUnit4.class)
@LargeTest
public class MenuTest {
    @Rule
    public ActivityTestRule<MenuActivity> mActivityRule = new ActivityTestRule<>(
            MenuActivity.class);
    @Test
    public void test(){
        //打开menu
        openContextualActionModeOverflowMenu();
        //模拟点击item2
        onView(withText("item2"))
                .perform(click());
    }
}
```

效果如下：
![这里写图片描述](https://img-blog.csdn.net/20180715210107970?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RvX3BlcmZlY3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



## 3、Android单元测试——Robolectric

&ensp;&ensp;如果您的应用的测试环境需要单元测试与Android框架进行更广泛的交互，则可以使用Robolectric。 该工具可让您在工作站上或常规JVM中的持续集成环境中运行测试，而无需仿真器，几乎与Android设备运行测试的完全保真度相匹配，但仍比执行设备测试更快，支持Android平台的以下几个方面。

- **Android4.1以及更高**
- **Android Gradle 插件2.4以及更高 **
- **组件生命周期**
- **事件循环**
- **所有资源：SDK, Resources,  Native Method **

grade配置：

```java
testImplementation "org.robolectric:robolectric:3.8"

android {
  testOptions {
    unitTests {
      includeAndroidResources = true
    }
  }
}
```

基本用法如下所示。

```java
@RunWith(RobolectricTestRunner.class)
public class MyActivityTest {

  @Test
  public void clickingButton_shouldChangeResultsViewText() throws Exception {
    MyActivity activity = Robolectric.setupActivity(MyActivity.class);

    Button button = (Button) activity.findViewById(R.id.button);
    TextView results = (TextView) activity.findViewById(R.id.results);

    button.performClick();
    assertThat(results.getText().toString()).isEqualTo("Robolectric Rocks!");
  }
}
```

[Robolectric社区](http://robolectric.org/)已经有详细的说明，这里就不再赘述



## 4、Android测试——UI Automator

先配置依赖

```java
dependencies {
    androidTestCompile 'com.android.support:support-annotations:25.4.0'
    androidTestCompile 'com.android.support.test:runner:1.0.0' 
    androidTestImplementation 'com.android.support.test.uiautomator:uiautomator-v18:2.1.3'
    androidTestCompile 'org.hamcrest:hamcrest-integration:1.3'

}
```

注意，UI Automator最低支持Android 4.3 (API level 18) 

在MainActivity中有四个组件editText、textView和button，布局就不贴出来了，在MainActivity的Java代码中主要是点击方法中，如下：

```java
  @Override
    public void onClick(View view) {
        // Get the text from the EditText view.
        final String text = mEditText.getText().toString();

        final int changeTextBtId = R.id.changeTextBt;
        final int activityChangeTextBtnId = R.id.activityChangeTextBtn;

        if (view.getId() == changeTextBtId) {
            //将edit中的text内容显示到textView中
            mTextView.setText(text);
        } else if (view.getId() == activityChangeTextBtnId) {
            //启动新的activity，并将text传给新的activity显示
            Intent intent = ShowTextActivity.newStartIntent(this, text);
            startActivity(intent);
        }
    }
```

主要看测试代码，这里创建一个ChangeTextBehaviorTest测试类：

```java
@RunWith(AndroidJUnit4.class)
@SdkSuppress(minSdkVersion = 18)
public class ChangeTextBehaviorTest {

    private static final String BASIC_SAMPLE_PACKAGE
            = "com.example.android.testing.uiautomator.BasicSample";

    private static final int LAUNCH_TIMEOUT = 5000;

    private static final String STRING_TO_BE_TYPED = "UiAutomator";

    private UiDevice mDevice;

    @Before
    public void startMainActivityFromHomeScreen() {
        // 获取UiDevice的实例
        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());

        // 模拟用户点击home键
        mDevice.pressHome();
        //获取要加载的包名
        final String launcherPackage = getLauncherPackageName();
        //判断是否为空
        assertThat(launcherPackage, notNullValue());
        //等待目标包 的信息
        mDevice.wait(Until.hasObject(By.pkg(launcherPackage).depth(0)), LAUNCH_TIMEOUT);

        // 启动目标activity,也就是MainActivity
        Context context = InstrumentationRegistry.getContext();
        final Intent intent = context.getPackageManager()
                .getLaunchIntentForPackage(BASIC_SAMPLE_PACKAGE);
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TASK);    // Clear out any previous instances
        context.startActivity(intent);

        // Wait for the app to appear
        mDevice.wait(Until.hasObject(By.pkg(BASIC_SAMPLE_PACKAGE).depth(0)), LAUNCH_TIMEOUT);
    }


    @Test
    public void testChangeText_sameActivity() {
        //将  STRING_TO_BE_TYPED 内容填充到edittext中
        mDevice.findObject(By.res(BASIC_SAMPLE_PACKAGE, "editTextUserInput"))
                .setText(STRING_TO_BE_TYPED);
        //给ID为changeTextBt  的组件模拟用户的点击事件
        mDevice.findObject(By.res(BASIC_SAMPLE_PACKAGE, "changeTextBt"))
                .click();

        // 等待获取MainActivity中ID为textToBeChanged的textView的内容，等待时间为500ms
        UiObject2 changedText = mDevice
                .wait(Until.findObject(By.res(BASIC_SAMPLE_PACKAGE, "textToBeChanged")),
                        500 /* wait 500ms */);
        //判断是否正确
        assertThat(changedText.getText(), is(equalTo(STRING_TO_BE_TYPED)));
    }

    @Test
    public void testChangeText_newActivity() {
        // 同上
        mDevice.findObject(By.res(BASIC_SAMPLE_PACKAGE, "editTextUserInput"))
                .setText(STRING_TO_BE_TYPED);
        mDevice.findObject(By.res(BASIC_SAMPLE_PACKAGE, "activityChangeTextBtn"))
                .click();

        // Verify the test is displayed in the Ui
        UiObject2 changedText = mDevice
                .wait(Until.findObject(By.res(BASIC_SAMPLE_PACKAGE, "show_text_view")),
                        500 /* wait 500ms */);
        assertThat(changedText.getText(), is(equalTo(STRING_TO_BE_TYPED)));
    }

    /**
     * 获取包名
     */
    private String getLauncherPackageName() {
        // Create launcher Intent
        final Intent intent = new Intent(Intent.ACTION_MAIN);
        intent.addCategory(Intent.CATEGORY_HOME);

        // Use PackageManager to get the launcher package name
        PackageManager pm = InstrumentationRegistry.getContext().getPackageManager();
        ResolveInfo resolveInfo = pm.resolveActivity(intent, PackageManager.MATCH_DEFAULT_ONLY);
        return resolveInfo.activityInfo.packageName;
    }
}
```

该框架的逻辑是模拟用户在使用APP的过程，这个测试用例的主要流程是：用户在桌面点击目标APP，进去，输入字符串，用户点击activityChangeTextBtn组件，跳转到ShowTextActivity，并传入内容，让其显示出来。然后点击changeTextBt组件，显示用户输入内容；
效果如下
<img src="https://img-blog.csdn.net/20180715185028699?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RvX3BlcmZlY3Q=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" width = "40%" height = "40%" />

该测试类有三个方法，其中在测试前需要获取 UiDevice的实例，步骤如下：

- **通过调用getInstance（）方法并将Instrumentation对象作为参数传递，获取UiDevice对象以访问要测试的设备。**
- **通过调用UiDevice实例的findObject（）方法，获取UiObject对象以访问设备上显示的UI组件（例如，前景中的当前视图）。**
- **可以通过调用UiObject方法模拟要在该UI组件上执行的特定用户交互;例如，调用performMultiPointerGesture（）来模拟多点触摸手势，调用setText（）来编辑文本字段。**
- **在执行这些用户交互之后，检查UI是否反映了预期的状态或行为。**

显然该框架需要从MainActivity开始，整个的模拟用户使用过程，好处是不会绑定特定的activity，资源具有全局性。源码见[GitHub](https://github.com/googlesamples/android-testing/tree/master/ui/uiautomator/BasicSample)

当然，也可以通过以下的方式拿到对应的组件：

```java
UiObject okButton = mDevice.findObject(new UiSelector()
        .text("OK")
        .className("android.widget.Button"));

// Simulate a user-click on the OK button, if found.
if(okButton.exists() && okButton.isEnabled()) {
    okButton.click();
}
```

如果要访问应用程序中的特定UI组件，请使用UiSelector类。 此类表示当前显示的UI中特定元素的查询。
如果找到多个匹配元素，则布局层次结构中的第一个匹配元素将作为目标UiObject返回。 构建UiSelector时，可以将多个属性链接在一起以优化搜索。 如果未找到匹配的UI元素，则抛出UiAutomatorObjectNotFoundException。
我们可以使用childSelector（）方法嵌套多个UiSelector实例。 例如，以下代码示例显示了测试如何指定搜索以在当前显示的UI中查找第一个ListView，然后在该ListView中搜索以查找具有文本属性Apps的UI元素

```java
UiObject appItem = new UiObject(new UiSelector()
        .className("android.widget.ListView")
        .instance(0)
        .childSelector(new UiSelector()
        .text("Apps")));
```

一旦您的测试获得了UiObject对象，您就可以调用UiObject类中的方法来对该对象所表示的UI组件执行用户交互。您可以指定以下操作：

- **click（）：单击UI元素可见边界的中心。**
- **dragTo（）：将此对象拖动到任意坐标。**
- **setText（）：在清除字段内容后，在可编辑字段中设置文本。相反，clearTextField（）方法清除可编辑字段中的现有文本。**
- **swipeUp（）：对UiObject执行向上滑动操作。类似地，swipeDown（），swipeLeft（）和swipeRight（）方法执行相应的操作。**

如果测试FrameLayout内容，则需要构建UiCollection，例如以下代码：

```java
UiCollection videos = new UiCollection(new UiSelector()
        .className("android.widget.FrameLayout"));

// 检索此集合中的视频数量
int count = videos.getChildCount(new UiSelector()
        .className("android.widget.LinearLayout"));

// 查找特定视频并模拟用户单击它
UiObject video = videos.getChildByText(new UiSelector()
        .className("android.widget.LinearLayout"), "Cute Baby Laughing");
video.click();

// 模拟选择与视频关联的复选框
UiObject checkBox = video.getChild(new UiSelector()
        .className("android.widget.Checkbox"));
if(!checkBox.isSelected()) checkbox.click();
```

对于可滑动视图，可以使用UiScrollable类模拟显示屏上的垂直或水平滚动。 当UI元素位于屏幕外并且您需要滚动以将其置于视图中时，此技术很有用。
以下代码段显示了如何模拟向下滚动“设置”菜单并单击“关于”平板电脑选项

```java
UiScrollable settingsItem = new UiScrollable(new UiSelector()
        .className("android.widget.ListView"));
UiObject about = settingsItem.getChildByText(new UiSelector()
        .className("android.widget.LinearLayout"), "About tablet");
about.click();
```



## 5、后记

在Android上进行单元测试不容易，需要消耗一定的时间，但是如果需要在不同设备上去测试，UI单元优势就十分明显了，这需要在实际项目中灵活运用
附上[代码链接](https://github.com/weiwangqiang/AndroidUnitTest/tree/master)



—— Weiwq 后记 于 2018.08 广州


