---
layout:     post
title:      "java注解详解"
subtitle:   " \"使用java注解的正确姿势\""
date:       2021-09-27 15:30:00
author:     "Weiwq"
header-img: "img/background/about-bg-walle.jpg"
catalog:  true
top: false
tags:
    - Java

---

> “揭开java注解的神秘面纱“

# 介绍

想必大家在接触java，甚至部分工作几年的，对于类、方法、字段上的 `@xxx ` 都有一种迷茫：这是啥玩意，它是怎么运行起来的？

别慌，这就是java的注解，一个很常见但又神秘的特性。

我们从最熟悉的Override注解开始，Override对应的声明如下，可以看到，注解与接口的声明很相似，只不过多了一个`@`。

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

同时他也依赖了其他两个注解Target和Retention。target的声明如下，用于声明注解的作用域，比如 Override是作用于方法的，如果在其他域使用该注解，编译器将会报错。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    // 注解值
    ElementType[] value();
}
// 注解类型
public enum ElementType {
   // 用于接口、类、枚举
    TYPE,
    // 字段和枚举常量
    FIELD,
    // 方法
    METHOD,
    // 参数
    PARAMETER,
    // 构造函数
    CONSTRUCTOR,
    // 局部变量
    LOCAL_VARIABLE,
    // 注解
    ANNOTATION_TYPE,
    // 包
    PACKAGE
}
```

而Retention的声明如下，其中CLASS、RUNTIME就是大名鼎鼎的编译时注解、运行时注解。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
    // 返回注解策略
    RetentionPolicy value();
}
// 注解策略
public enum RetentionPolicy {
    // 注释将由编译器丢弃
    SOURCE,
    // 注释将由编译器记录在类文件中，但无需在运行时由 VM 保留，这是默认值行为。
    CLASS,
    // 注释将由编译器和运行时由 VM 保留，因此可以反射地读取它们
    RUNTIME
}
```

简单的来说，注解的声明有两个重要的注解：作用域（target）和保留策略（Retention）。其中保留策略很重要，它决定了注解的生命长度。

道理都懂，问题是注解怎么用，只是好（装）看（B）么？来，教你真功夫！

#  1、SOURCE注解

**作用**：source注解又称源码注解，给编译器读的，在编译成class文件的时候会被去掉，用于协助开发者编写正确的代码。

有如下代码，其中Override是源码注解，Test注解是编译时注解。

```java
public class Main {
    private static class Parent {
        void read() {
            System.out.println("read");
        }
    }
    private static class Child extends Parent {
        @Override
        void read() {
            super.read();
        }
        @Test(id = 29)
        public void Test() {
        }
    }
}
```

对应的编译class文件如下，可以看到Override注解已经被移除，但是Test注解还在。

<img src="/img/blog_java_annotation/1.png" width="50%" height="40%">

那SOUIRCE注解是怎么帮助编写正确的代码呢？

且看下面的例子：setLeve 方法需要限制传入的参数，只能传LEVE_1或者LEVE_2。我们可以通过定义Level 注解来实现。

```java
import androidx.annotation.IntDef;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

public class Main {

    public static final int LEVE_1 = 1;
    public static final int LEVE_2 = 2;

    @Retention(RetentionPolicy.SOURCE) // 源码注解
    @Target(ElementType.PARAMETER) // 作用于参数
    @IntDef({LEVE_1, LEVE_2}) // 限制值的范围
    public @interface Level {
    }

    public static void main(String[] args) { 
        Main main = new Main();
        main.setLeve(0); // 报错
        main.setLeve(1); // 报错
        main.setLeve(LEVE_1); // 正确
    }
    // 限制合法参数为LEVEL_1 和 LEVEL_2
    public void setLeve(@Level int level) {
        System.out.println("level " + level); 
    }
}
```

一般如果要实现上述需求，需要定义对应的枚举来实现，这里通过Android 提供的IntDef 注解，定义对应参数的值范围，达到枚举的效果，并且性能比枚举好。


# 2、运行时注解

**作用**：保留到运行阶段。主要在代码执行的时候会获取该注解 ，做一些反射的操作。

我们用运行时注解实现butterKnife的功能，核心思路：通过遍历指定的注解，拿到值后，用activity的方法获取view，再反射绑定到对应的属性上。

## 接口

首先定义两个注解接口

```java
// 用于绑定view
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface FindView {
    int value();
}

// 用于绑定方法
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface OnClick {
    int value();
}

```

## 处理器

定义ViewProcessor，用于在运行时解析注解。

```java
public class ViewProcessor {
    private static final String TAG = "ViewProcessor";

    public void inject(Activity activity) {
        try {
            injectId(activity);
            injectOnClick(activity);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void injectOnClick(Activity activity) {
        Class<?> cls = activity.getClass();
        // 获取全部声明的方法
        for (Method method : cls.getDeclaredMethods()) {
            Log.d(TAG, "injectOnClick method is : " + method.getName());
            // 获取该方法上的注解
            Annotation[] annotations = method.getAnnotations();
            for (Annotation annotation : annotations) {
                if (!(annotation instanceof OnClick)) {
                    continue;
                }
                // 找到OnClick注解
                OnClick findView = (OnClick) annotation;
                // 获取OnClick的值
                int id = findView.value();
                // 找到对应的view
                View view = activity.findViewById(id);
                if (view == null) {
                    continue;
                }
                view.setOnClickListener((view1) -> {
                    Log.d(TAG, "injectOnClick: callback");
                    try {
                        // 反射调用该方法
                        method.setAccessible(true);
                        method.invoke(activity);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                });
            }
        }
    }

    private void injectId(Activity activity) throws IllegalAccessException {
        Class<?> cls = activity.getClass();
        for (Field field : cls.getDeclaredFields()) {
            Log.d(TAG, "injectOnClick filed is : " + field);
            Annotation[] annotations = field.getAnnotations();
            for (Annotation annotation : annotations) {
                if (!(annotation instanceof FindView)) {
                    continue;
                }
                // 找到FindView注解
                FindView findView = (FindView) annotation;
                int id = findView.value();
                View view = activity.findViewById(id);
                if (view == null) {
                    continue;
                }
                field.setAccessible(true);
                // 给该域负值
                field.set(activity, view);
            }
        }
    }
}
```

## 使用

在onCreate的时候，初始化注解处理器，实现注解的解析。

```java
public class RuntimeActivity extends AppCompatActivity {

    @FindView(R.id.runtime_button1)
    private Button mButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_runtime);
        // 初始化注解处理器
        ViewProcessor bindViewHelper = new ViewProcessor();
        bindViewHelper.inject(this);
        mButton.setOnClickListener((view) -> {
            Toast.makeText(this, "运行时注解 FindView", Toast.LENGTH_SHORT).show();
        });
    }

    @OnClick(R.id.runtime_button2)
    private void onClick2() {
        Toast.makeText(this, "运行时注解 OnClick", Toast.LENGTH_SHORT).show();
    }
}
```

## 小结

- 优点：通过反射方式，实现赋值和方法调用，对于域或方法的访问范围不做要求，框架实现较为简单。
- 缺点：使用大量反射，运行时性能较差。

# 3、编译时注解——概念

**作用**：在编译期间生效的，常用于在编译期间插入模板代码。

## 什么是APT

这里不得不提一下APT，APT(Annotation Processing Tool)是 javac 提供的一种可以处理注解的工具，用来在编译时扫描和处理注解的，简单来说就是可以通过 APT 获取到注解及其注解所在位置的信息，可以使用这些信息在编译器生成代码。编译时注解就是通过 APT 来通过注解信息生成代码来完成某些功能，典型代表有 ButterKnife、Dagger等。

## AbstractProcessor

AbstractProcessor 是实现编译注解的关键入口，自定义的注解处理器都是需要继承于它，其中以下方法比较重要：

- init：主要做一些初始化的动作，比如Elements、Filer 和 Message等。
- getSupportedAnnotationTypes：用来设置支持的注解类型
- getSupportedSourceVersion：获取java版本。
- process：解析注解，生成代码模板的实现回调。

## Element

Element 用于表示程序元素，例如模块、包、类或方法。每个元素代表一个静态的、语言级别的构造。而Elements 是处理 Element 的工具类，只提供接口。

# 4、编译时注解——实现

我们用编译时注解重写一下上面的butterKnife。

项目中，有如下module

- app：用于demo演示
- api：用于定义注解，比如BindView
- butterKnife：用于处理注解，生成代码的逻辑

## 接口

该模块定义了两个注解BindView和Onclick

```java

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.CLASS)
public @interface BindView {
    int value();
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
public @interface Onclick {
    int[] value();
}

```

## 处理器

butterKnife 模块需要依赖第三方库

```java
dependencies {
    // 用来生成META-INF/services/javax.annotation.processing.Processor文件
    implementation 'com.google.auto.service:auto-service:1.0-rc2'
    // 用于创建Java文件
    implementation 'com.squareup:javapoet:1.12.1'
    // 导入javaX包
    targetCompatibility = '1.8'
    sourceCompatibility = '1.8'
}
```

对应的注解处理器是 ButterKnifeProcessor

```java
// 用于声明该类为注解处理器
@AutoService(Processor.class)
public class ButterKnifeProcessor extends AbstractProcessor {
    // 用于打印日志信息
    private Messager mMessager;
    // 用于解析 Element
    private Elements mElements;
    // 存储某个类下面对应的BindModel
    private Map<TypeElement, List<BindModel>> mTypeElementMap = new HashMap<>();
    // 存储id绑定的方法，即OnClick
    private Map<Integer, Element> mOnclickElementMap = new HashMap<>();
    // 用于将创建的java程序输出到相关路径下。
    private Filer mFiler;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        mMessager = processingEnv.getMessager();
        mElements = processingEnv.getElementUtils();
        mFiler = processingEnv.getFiler();
    }
    /**
     * 此方法用来设置支持的注解类型，没有设置的无效（获取不到）
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        HashSet<String> supportTypes = new LinkedHashSet<>();
        // 把支持的类型添加进去
        supportTypes.add(BindView.class.getCanonicalName());
        supportTypes.add(Onclick.class.getCanonicalName());
        return supportTypes;
    }
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

   @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {
        mMessager.printMessage(Diagnostic.Kind.NOTE, "===============process start =============");
        mTypeElementMap.clear();
        // Process each @BindView element.
        for (Element element : roundEnv.getElementsAnnotatedWith(BindView.class)) {
            verifyAnnotation(element, BindView.class, ElementKind.FIELD);
            TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
            Name qualifiedName = enclosingElement.getQualifiedName();
            Name simpleName = element.getSimpleName();

            // Assemble information on the field.
            int id = element.getAnnotation(BindView.class).value();
            String content = String.format("====> qualifiedName: %s simpleName: %s id: %d"
                    , qualifiedName, simpleName, id);
            mMessager.printMessage(Diagnostic.Kind.NOTE, content);
            List<BindModel> modelList = mTypeElementMap.get(enclosingElement);
            if (modelList == null) {
                modelList = new ArrayList<>();
            }
            modelList.add(new BindModel(element, id));
            mTypeElementMap.put(enclosingElement, modelList);
        }

        for (Element element : roundEnv.getElementsAnnotatedWith(Onclick.class)) {
            verifyAnnotation(element, Onclick.class, ElementKind.METHOD);
            int[] ids = element.getAnnotation(Onclick.class).value();
            for (int id : ids) {
                mOnclickElementMap.put(id, element);
            }
        }

        mTypeElementMap.forEach((typeElement, bindModels) -> {
            String packageName = mElements.getPackageOf(typeElement)
                    .getQualifiedName().toString();
            String className = typeElement.getSimpleName().toString();
            String bindClass = className + "_ViewBind";
             // 生成构造函数
            MethodSpec.Builder builder = MethodSpec.constructorBuilder()
                    .addModifiers(Modifier.PUBLIC) // 声明为public
                    .addParameter(ClassName.bestGuess(className), "target"); // 添加构造参数
            bindModels.forEach(model -> {
                // 构造函数内添加代码
                builder.addStatement("target.$L = ($L)target.findViewById($L)",
                        model.getViewFieldName(), model.getViewFieldType(), model.getResId());
            });
            String viewPath = "android.view.View";
            mOnclickElementMap.forEach((id, element) -> {
                // 构造函数内添加代码
                builder.addStatement("(($L) target.findViewById($L)).setOnClickListener((view) -> {\n" +
                                "            target.$L();\n" +
                                "        })",
                        viewPath, id, element.getSimpleName().toString());
            });
            // 构建类
            TypeSpec typeSpec = TypeSpec.classBuilder(bindClass)
                    .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                    .addMethod(builder.build())
                    .build();
            // 生成java file
            JavaFile javaFile = JavaFile.builder(packageName, typeSpec)
                    .addFileComment("auto create by ButterKnife ")
                    .build();
            try {
                // javaFile 写到指定路径下
                javaFile.writeTo(mFiler);
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
        mMessager.printMessage(Diagnostic.Kind.NOTE, "===============process end=============");
        return true;
    }
    
    // 做校验动作
    private boolean verifyAnnotation(Element element, Class<?> annotationClass, ElementKind targetKind) {
        if (element.getKind() != targetKind) {
            error(element, "%s must be declared on field.", annotationClass.getSimpleName());
            return false;
        }
        Set<Modifier> modifiers = element.getModifiers();
        if (modifiers.contains(PRIVATE) || modifiers.contains(STATIC)) {
            error(element, " %s %s must not be private or static.",
                    annotationClass.getSimpleName(),
                    element.getSimpleName());
            return false;
        }
        return true;
    }

    /**
     * 打印错误日志方法
     */
    private void error(Element element, String message, Object... args) {
        if (args.length > 0) {
            message = String.format(message, args);
        }
        mMessager.printMessage(Diagnostic.Kind.NOTE, message, element);
    }
}
```

上面的代码大家可能会不知所措，其通过JavaPoet 来声明生成的java文件结构。具体的用法可以参考[JavaPoet使用详解](https://blog.csdn.net/IO_Field/article/details/89355941)

## 使用

app模块主要依赖

```java
android {
    defaultConfig {
        // 声明注解器
        javaCompileOptions {
            annotationProcessorOptions {
                includeCompileClasspath true
                classNames = ["com.example.compiler.ButterKnifeProcessor"]
            }
        }
    }
}
dependencies {
    // 注意，是通过 annotationProcessor方式依赖butterKnife 而不是implementation。
    annotationProcessor project(path: ':butterKnife')
    implementation project(path: ':api')
}
```

然后就可以使用注解替换对应的代码了

```java
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.button1)
    public Button button1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ViewInjector.injectView(this);
        button1.setOnClickListener((view) -> {
            Toast.makeText(this, "编译时:点我1", Toast.LENGTH_SHORT).show();
        });
    }

    @Onclick(R.id.button2)
    public void press1() {
        Toast.makeText(this, "编译时: 点我2", Toast.LENGTH_SHORT).show();
    }
}
```

点击build后，在build/generated/ap_generated_sources/debug/out/com/example/annotation/路径下，生成对应的代码

```java
public final class MainActivity_ViewBind {
    public MainActivity_ViewBind(MainActivity target) {
        target.button1 = (android.widget.Button) target.findViewById(2131230812);
        ((android.view.View) target.findViewById(2131230813)).setOnClickListener((view) -> {
            target.press1();
        });
    }
}
```

## 小结 

- 优点：通过插入编译期间，生成模板代码，即java文件，避免了运行时注解中使用反射的实现方式，提高运行时效率。
- 缺点：域或方法的访问范围必须是public，否则就会失败。

# 后记

也许我们不一定要自己造轮子，但应该需要知道对应的基础实现原理，这样我们才能透过现象看本质。

附上源码[github链接](https://github.com/weiwangqiang/csdnDemo/tree/annotation)

——Weiwq  于 2021.09 广州

