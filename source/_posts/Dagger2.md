---
title: Dagger2使用
date: 2019-07-22
tags: [框架,Dagger2]
categories: Android第三方框架
toc: true
---
[Dagger官网地址](https://dagger.dev/)

Dagger是一个完全静态的、编译时依赖 **注入框架** ，适用于Java和Android。它是Square创建的早期版本的一个改编版本，现在由谷歌维护。
<!--more-->
### 说说什么是依赖注入？

依赖注入对于刚接触到这个概念的同学可能不太清楚什么意思，说起来也比较简单甚至我们每天都在编写这样的代码。

```java
public class A{
    private B b;
    public A(B _b){
        this.b = _b;
    }
}
```
上面的代码是不是很常见，我们发现业务类A依赖了业务类B。
这就是一个典型的依赖注入。
那我们来想想如果A被很多的其他业务类依赖如果A做了修改是不是要修改很多的地方呢？
这样我们的Dagger就应该登场了，Dagger就为了解决此类问题而生的。

#### 简单使用：

##### 导依赖

build.gradle中导入依赖如：
```java
dependencies {
    implementation 'com.google.dagger:dagger-android:2.17'
    annotationProcessor"com.google.dagger:dagger-compiler:2.17"
    implementation 'com.google.dagger:dagger-android-support:2.17' // if you use the support libraries
    annotationProcessor 'com.google.dagger:dagger-android-processor:2.17'
}
```

##### 业务类ClassA

```java
package com.baweigame.daggerdemoapplication;

import javax.inject.Inject;

public class ClassA {
    @Inject
    public ClassA(){

    }

    public int Add(int a,int b){
        return a+b;
    }
}

```

##### 业务类ClassB

```java
package com.baweigame.daggerdemoapplication;

import javax.inject.Inject;

public class ClassB {
    @Inject
    public ClassB(){

    }

    @Inject
    ClassA classA;

    public int Add(int a,int b){
        return classA.Add(a,b);
    }
}

```

##### MainActivityComponent

```java
package com.baweigame.daggerdemoapplication;

import android.app.Activity;

import dagger.Component;

@Component
public interface MainActivityComponent{
    void inject(MainActivity activity);
}
```

##### MainActivity

```java
package com.baweigame.daggerdemoapplication;

import android.app.Activity;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.TextView;
import android.widget.Toast;

import javax.inject.Inject;

import dagger.Component;

public class MainActivity extends AppCompatActivity {
    private TextView tvTest;

    @Inject
    ClassB classB;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();
        initListener();

        DaggerMainActivityComponent.create().inject(this);
    }

    private void initListener() {
        tvTest.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(MainActivity.this, ""+classB.Add(2,3), Toast.LENGTH_SHORT).show();
            }
        });
    }

    private void initView() {
        tvTest = (TextView) findViewById(R.id.tv_test);
    }


}
```

我们看上面的例子，比较简单，首先业务类B依赖了业务类A，MainActivity依赖了业务类B，我们发现用Dagger实现起来是不是很清爽。如果我们不用Dagger我的代码大概应该是下面的样子：
```java
--一段伪代码

ClassA classA=new ClassA();
ClassB classB=new ClassB();
classB.set(classA);

classB.Add(2,3);
```
使用Dagger2
```java
 @Inject
 ClassB classB;

 classB.Add(2,3);

```
感觉是不是很简洁。
上面的例子中出现了一些新的“东西”，我们发现大多出现的是注解。

---
下面是Dagger的常用注解。

**Componet 注解**

```java
@Retention(value=RUNTIME)
@Target(value=TYPE)
@Documented
public @interface Component {
    Class<?>[] modules() default {};
    Class<?>[] dependencies() default {};
}

```

参考链接：https://dagger.dev/api/latest/dagger/Component.html

注释接口或抽象类，为其从一组模块生成注入依赖关系的实现。生成的类的类型名称将带有@Component注释，前缀为Dagger。例如，@Component interface MyComponent{…}将生成一个名为DaggerMyComponent的实现。

**Subcomponent 注解**

```java
@Retention(value=RUNTIME)
@Target(value=TYPE)
@Documented
public @interface Subcomponent {
    Class<?>[] modules() default {};
}
```

参考链接：https://dagger.dev/api/latest/dagger/Subcomponent.html

从父组件或子组件继承绑定的子组件。

**Module 注解**

```java
@Documented
@Retention(value=RUNTIME)
@Target(value=TYPE)
public @interface Module{
    Class<?>[] includes() default {};
    Class<?>[] subcomponents default {};
}
```


参考链接：https://dagger.dev/api/latest/dagger/Module.html

**Provides 注解**

```java
@Documented
@Target(value=METHOD)
@Retention(value=RUNTIME)
public @interface Provides{}
```
参考链接：https://dagger.dev/api/latest/dagger/Provides.html

注释模块的方法，以创建提供程序方法绑定。方法的返回类型绑定到其返回值。组件实现将依赖项作为参数传递给方法。

**MapKey 注解**

```java
@Documented
@Target(value=ANNOTATION_TYPE)
@Retention(value=RUNTIME)
public @interface MapKey{
    boolean unwrapValue() default true;
}
```

参考链接：https://dagger.dev/api/latest/dagger/MapKey.html

标识用于将Key与方法返回的值关联以组成映射的注释类型。
每个使用@ provide和@IntoMap注释的提供程序方法都必须有一个注释来标识映射条目的键。该注释的类型必须使用@MapKey进行注释。
通常，键注释只有一个成员，其值用作映射键。

**Dagger 2中用到的定义在 JSR-330的其他注解**
```java
public @interface Inject {
}
 
public @interface Scope {
}
 
public @interface Qualifier {
}
```

上面的例子中我们接触到了@Inject和@Component两个注解，下面来说说这两个注解是做什么用的？
如果想使用Dagger来实现依赖注入就至少要使用这两个注解：
**@Inject**用于标记需要注入的依赖，或者标记用于提供依赖的方法。
依赖注入中最重要的注解，JSR-330标准中的一部分。

Dagger 2中有3种方式提供依赖：

1、构造函数注入，如我们上面例子中的
```java
ClassA:

@Inject
public ClassA(){

}
```
注：如果存在多个构造函数，我们只能标注一个，不能同时标注多个。

2、属性注入
注：被标注的属性不能使用private修饰。

3、方法注入


**@Component**用来完成注入，在上面的例子中目标MainActivity就是使用Component完成注入。


**名词解释：**

JSR-330
JSR即Java Specification Requests，意思是java规范提要。
而JSR-330则是 Java依赖注入标准

本文参考文章：

[Dagger2 最清晰的使用教程](https://blog.csdn.net/heng615975867/article/details/80355090)

[深入浅出Dagger2 : 从入门到爱不释手](https://www.jianshu.com/p/626b2087e2b1)