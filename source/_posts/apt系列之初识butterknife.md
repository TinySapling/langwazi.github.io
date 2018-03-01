---
title: 浅析butterknife
date: 2018-01-29 21:42:04
tags:
  - apt
  - 源码阅读
---

![butterknife](http://p1wp1oc1z.bkt.clouddn.com/bf.png)

[butterknife](https://github.com/JakeWharton/butterknife)对于做android小伙伴来说都不会感到陌生（完全不知道的话自己面壁去），其中用到了javapoet和apt（已被annotationProcessor取代）这两个技术。

 <!-- more -->

APT
--
android-apt 是一个Gradle插件，他存在的目的主要有两个。

1.  允许配置只在编译时作为注解处理器的依赖，而不添加到最后的APK或library。

2.  设置源路径，使注解处理器生成的代码能被Android Studio正确的引用。

关于gradle插件在这里不做展开，简单说apt就在编译时的一个辅助工具。
[参考资料](https://www.jianshu.com/p/2494825183c5)

javapoet
--
poet，诗人？
![杜甫](http://p1wp1oc1z.bkt.clouddn.com/u=3244538114,929704141&fm=173&s=6BAC3C6241836FE91CF9BCC30100E0A1&w=383&h=397&img.JPEG)
咳咳，javapoet的作用就是生成java代码，github仓库地址：
https://github.com/square/javapoet 

```java
MethodSpec main = MethodSpec.methodBuilder("main")
.addModifiers(Modifier.PUBLIC, Modifier.STATIC)
.returns(void.class)
.addParameter(String[].class, "args")
.addStatement("$T.out.println($S)", System.class, "Hello, JavaPoet!")
.build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
.addModifiers(Modifier.PUBLIC, Modifier.FINAL)
.addMethod(main)
.build();

JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
.build();

javaFile.writeTo(System.out);

```
上面这段javapoet的代码运行后将会生成一个com.example.helloworld.HelloWorld的java文件，文件内容如下：

```java
package com.example.helloworld;

public final class HelloWorld {
  public static void main(String[] args) {
    System.out.println("Hello, JavaPoet!");
  }
}

```

![interesting.png](http://p1wp1oc1z.bkt.clouddn.com/timg.jpg)

通过javapoet可以省去许多模板代码的编写。

butterknife套路初体验
--
本文中用到的butterknife版本为8.8.1，我们假设前面的步骤都是合理正确的。
```java
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.tv)
    TextView tvTest;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        tvTest.setText("hello wrold");
    }
}
```
上面这段代码是如何自动findviewbyid？look！（编译后才能看到）

![位置](http://p1wp1oc1z.bkt.clouddn.com/TIM%E6%88%AA%E5%9B%BE20180129232026.png)
这个MainActivity_ViewBinding文件就是butterknife通过javapoet为我们生成，打开文件内容如下：
```java
public class MainActivity_ViewBinding implements Unbinder {
  private MainActivity target;

  @UiThread
  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public MainActivity_ViewBinding(MainActivity target, View source) {
    this.target = target;

    target.tvTest = Utils.findRequiredViewAsType(source, R.id.tv, "field 'tvTest'", TextView.class);
  }

  @Override
  @CallSuper
  public void unbind() {
    MainActivity target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    target.tvTest = null;
  }
}
```

MainActivity作为参数传递了进来，通过Utils.findRequiredViewAsType方法（实际就是findViewById）赋值给我们MainActivity.tvTest。

假设这个文件生成的过程我们是清楚了解的，那么这个文件时如何和MainActivity发生关系的（好像有点污啊喂）？
是**ButterKnife.bind(this)** 让他们~~发生关系~~，点击去可以看到好多好多方法。
```java
 @NonNull @UiThread
  public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);
  }
  
  //进入createBinding方法可以看到
  Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);
  //省略若干代码
   return constructor.newInstance(target, source);
```
可以看到通过findBindingConstructorForClass这个方法得到了constructor然后在实例化了一个对象，这个对象就是MainActivity_ViewBinding，在MainActivity_ViewBinding的构造方法中完成findViewById。

findBindingConstructorForClass方法的参数为Class<?> cls；

这里的实际值为MainActivity.class,findBindingConstructorForClass关键代码：
``` java
 Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
      //noinspection unchecked
 bindingCtor = 
      (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
return bindingCtor;
```
通过传递进来的class参数获取类名并拼接上 **_ViewBinding 后缀**，最后获取到2个参数的构造方法，然后实例化对象完成绑定。

![套路](http://p1wp1oc1z.bkt.clouddn.com/TIM%E6%88%AA%E5%9B%BE20180130005242.png)

以上就是发生关系的套路。

套路之APT
---
APT代码生成是整个套路中最重要的一个步骤，通过解析所使用的注解生成对应的代码。
clone下源码我们注意这三个module↓
![工程](http://p1wp1oc1z.bkt.clouddn.com/TIM%E6%88%AA%E5%9B%BE20180130220421.png)

- butterknife（类型为android llibrary）在上一节简单的阅读了Activity下bindview的部分代码。

- butterknife-annotations(类型为 java library)中全部是定义相关注解，@BindView，@OnClick等注解。
  以在activity中使用@BindView注解为例。
```java
//BindView.java
@Retention(CLASS) @Target(FIELD)
public @interface BindView {
   /** View ID to which the field will be bound. */
   @IdRes int value();
}
```
  定义了一个值为资源id，它作用在field，该注解保留到class文件。

-  butterknife-compiler（类型为java library）是处理和生成代的地方，其中*ButterKnifeProcessor*类是处理通往新世界的大门。

`ButterKnifeProcessor`

继承自一个抽象的`AbstractProcessor`，而一个Processor类需要注意下面的几个方法。
``` java
public class ExampleProcessor extends AbstractProcessor {
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        //该方法必须重写,通过processingEnv可以获取到一些需要的工具
        super.init(processingEnv);
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        //处理注解的方法
        //annotations：需要处理的注解集合
        //roundEnv:通过该参数可以获取对应注解的一些信息
        return false;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        //该返回值决定需要处理的注解类型
        return super.getSupportedAnnotationTypes();
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        //一般返回该值
        return SourceVersion.latestSupported();
    }
}
```
ButterKnifeProcessor的大体结构也是如此，下面开始查看他的源码。
`getSupportedAnnotationTypes`方法，可以看到它将需要处理的注解类通过`getCanonicalName`获取到名字后add到`Set<String> types = new LinkedHashSet<>()`并返回。

`process`方法简化后代码如下
``` java 
  Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);
  for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
      //通过遍历生成文件，一个BindingSet对应一个文件的生成
      BindingSet binding = entry.getValue();
      JavaFile javaFile = binding.brewJava(sdk, debuggable);
      javaFile.writeTo(filer);
    }
```
BindingSet：是封装了代码生成逻辑的类，一个BindingSet对象就可以生成一个文件；TypeElement： 表示类或者接口的元素。假如在MainActivity中使用注解，那么它表示MainActivity这个元素。

那么是如何生成这个map的？进入`findAndParseTargets`找到下面代码片段
``` java
//只摘要BindView功能，省略N多代码简化后大致如下
Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
for (Element element : roundEnv.getElementsAnnotatedWith(BindView.class)) {
    try {
          //解析每一个BindView注解
          parseBindView(element, builderMap);
        } catch (Exception e) {
          e.printStackTrace();
        }
}

//遍历builderMap并开始build
Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
                new ArrayDeque<>(builderMap.entrySet());
Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
while (!entries.isEmpty()) {
     Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();
     TypeElement type = entry.getKey();
     BindingSet.Builder builder = entry.getValue();
     bindingMap.put(type, builder.build());
}

return bindingMap;
```
`parseBindView`超级简化版代码如下
``` java
//获取封装此元素的最里层元素
//enclosingElement此处可以理解成：用来表示element(某个注解)所在的java类(MainActivity)的元素.
TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
//判断是否存已经存在enclosingElement(例如MainActivity)的构建
BindingSet.Builder builder = builderMap.get(enclosingElement);
if (builder == null) {
   builder = BindingSet.newBuilder(enclosingElement);
}
//获取信息
TypeMirror elementType = element.asType();
//获取控件id
int id = element.getAnnotation(BindView.class).value();
//名字
Name simpleName = element.getSimpleName();
String name = simpleName.toString();
TypeName type = TypeName.get(elementType);
//放入id和field
builder.addField(new Id(id), new FieldViewBinding(name, type));
builderMap.put(enclosingElement, builder);
```

可以看到该方法主职能还是比较清晰的，目的就是采集各种信息并添加到`builderMap`中。
`parseBindView`方法将需要处理的信息获取到后添加到`builderMap`交给`findAndParseTargets`方法遍历并生成一个`Map<TypeElement, BindingSet>`集合,最终在`process`方法中遍历集合调用集合元素的方法生成对应的代码。


动手实践
--

![嘻嘻](http://p1wp1oc1z.bkt.clouddn.com/TIM%E5%9B%BE%E7%89%8720180130005858.jpg)

赶紧，趁热来一波.......   我是说来一波仿写。学习完理论怎么也要来一波demo压压惊啊！demo功能非常简单就是实现butterknife的bindView功能就可以了，demo代码中都有相关的注释。
基本上就是~~复制粘贴~~butterknife相关代码，剔除无用代码。虽然是粘贴复制，但我想说的是如果没有阅读清楚理解逻辑又如何从这么大一个库里面粘贴复制呢？依赖那么多....
![图片](http://p1wp1oc1z.bkt.clouddn.com/3d4d18e8857f910811c429e2cbaf9e27_hd.jpg)

需要注意的地方是processor所在的compiler module的依赖问题，依赖如下：
``` 
 //需要注意这个依赖
 //AutoService 主要的作用是注解 processor 类，并对其生成 META-INF 的配置信息
 implementation 'com.google.auto.service:auto-service:1.0-rc4'
 //注解所在的module
 implementation project(':hello-annotations')
 //javapoet生成代码的库
 implementation 'com.squareup:javapoet:1.10.0'
```
在`Processor`处理类上添加atuo注解`@AutoService(Processor.class)`

传送门→[练习demo地址](https://github.com/langwazi/HelloApt)