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

简单使用：

build.gradle中导入依赖如：
```java
dependencies {
    implementation 'com.google.dagger:dagger-android:2.17'
    annotationProcessor"com.google.dagger:dagger-compiler:2.17"
    implementation 'com.google.dagger:dagger-android-support:2.17' // if you use the support libraries
    annotationProcessor 'com.google.dagger:dagger-android-processor:2.17'
}
```

业务类ClassA

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

业务类ClassB

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

MainActivityComponent

```java
package com.baweigame.daggerdemoapplication;

import android.app.Activity;

import dagger.Component;

@Component
public interface MainActivityComponent{
    void inject(MainActivity activity);
}
```

MainActivity

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
上面的例子中出现了一些新的“东西”，下面我们来了解一下Dagger。

---
