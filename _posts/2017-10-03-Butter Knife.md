---
title: Butter Knife
date: 2017-10-03
categories: android
tags:
- Butter Knife
---


填坑了填坑了！
Butter Knife 这个框架好像很简单的样子，下面讲下具体用法，有时间去看看源码。

<!-- more -->

build.gradle(Module:app)中添加下面的代码。
```gradle
dependencies{
    compile 'com.jakewharton:butterknife:8.6.0'
    annotationProcessor 'com.jakewharton:butterknife-compiler:8.6.0'
}
```
最新版本号可以到 [这里](http://jakewharton.github.io/butterknife/) 看下。

通过注解快速实例化和绑定监听器。
```java

@BindView(R.id.text) TextView textView;

@OnClick(R.id.button)
    public void bt1(Button button){
        textView.setText("check");
    }
```
<br>
我之前一直以为是Button Knife.......