---
title: Presentation和键盘那些事
date: 2020-07-21
categories: android
tags:

- Presnetation

---

Presentation，直接翻译过来就是展示的意思，根据官方的描述， presentation 是一种 dialog ，用于在别的屏幕上展示内容， presentation 与 display 关联， presentation 中使用的 context 和 Activity 的 context 不同。

<!---begin--->

项目里面内存压力挺大，线上的 oom 一直是我们一道坎，处于这样的目的，我们项目中的多进程要着手推起来了。

普通模块跨进程问题不是很大，做好组件化，做好进程保活和通讯问题即可，重点是如何将一个大的 Activity 分割开来，因为我们整个项目业务比较重的就那几个 Activity ，不能将整个 Activity 跨进程，如何拆分业务就是个难题。

兜兜转转，我们目光定在了这个叫 presentation 的东西上，通过安卓官方提供的这个技术，我们可以将 Activity 中的一个 texture 作为一个 display 用于 Presentation 的展示，而 Presentation 处于子进程中，这样我们就由不同的进程提供的业务/视图 共同构成一个 Activity。

但是这个方案在推行的时候遇到了一个核心问题， presentation 设计就是单向输出，就是将 presentation 展示都 texture 上，并不处理任何输入事件，也就是说这个跨进程的 view 只能展示不能触控。

这个问题也很简单，触摸事件可以序列化呀！我们在主进程中拦截 texture 的触摸事件，跨进程传给 presentation 让他处理，不就完事了吗， easy 

然而我们后面又遇到了第二个问题， presentation 中无法唤醒键盘，强行唤醒键盘也不行，上网查了一下说的也是 presentation 只能输入不能输出，那怎么办呢，一开始想到的是在主进程放一个 editText 之类的来接收和处理键盘，在 presentation 检测到 view 需要输入等情况，就在主进程让 editText 获取焦点唤醒键盘，然后将输入转到 presentataion 中。这个操作对业务的 presentation 侵入性很强，要处理里面每一个可输入的 view ，而且不能触发系统的复制粘贴功能，最后这个方案被沉底，作为最后的办法。

在摸索的时候 无意中发现，官方的 presentation 除了我们继承 还有别的地方在继承，一看是 flutter ，了解了一下 flutter 中有个技术，将 flutter 和原生的 view 混用，具体是定义了一个叫 androidView 的东西，获取原生 view 中的纹理信息和其他 flutter 的 view 然后通过 flutter 的引擎 最后渲染到 presentation 中，大概操作是包了一个 Context ，拦截了获取 windowManager ，生成了个代理对里面的一些方法进行了拦截篡改，还有构造了一个根 view 覆写了他的 requestSendAccessibilityEvent 方法，看完这个我心中一波窃喜，有救了。然后模仿着写了一个，发现毛用都没，根本就没用到这个 context 的 windowManager 的任何方法，也没有调用到这个 view 的 requestSendAccessibilityEvent 。

没办法，那就只能看看唤醒不了键盘的原因是什么，用一些别的手段来试试了，通过断点 InputMethodManager ，发现唤醒键盘是卡在了 checkFocusNoStartInput 这个方法的这个地方

```java
    private boolean checkFocusNoStartInput(boolean forceNewFocus) {
        // This is called a lot, so short-circuit before locking.
        if (mServedView == mNextServedView && !forceNewFocus) {
            return false;
        }
      	...
        ...
        ...
    }
```

mServedView 和 mNextServedView 都为空，看了一下 mNextServedView 的赋值只有在 focusIn 的时候才有调用，那不就很简单了，我直接反射调用 focusin 这个方法 让他 inputMtehodManager 的 mNextServedView 赋值，然后就能进一步往下走了。说干就干， checkFocusNoStartInput 这步过了，走到了 startInputInner 然而还是不能唤起键盘，原因是调用 mService.startInputOrWindowGainedFocus ，最后返回的结果是 InputBindResult.ResultCode.ERROR_NOT_IME_TARGET_WINDOW ，查了一下这个错误码是因为窗口不可聚焦，那我再修改一下

```java
getWindow().clearFlags(WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE);
```

这样就可以了吗，还是不行。

在万念俱灰之际，突然想到 InputMethodManager 是通过 Context 获取的，那我能不能在 Presentation 里面的 view 初始化的时候使用的 Context 是主进程的，那获取到的 InputMethodManager 就是主进程的的 InputMethodManager ，然后 InputMethodManagerService 的 startInputOrWindowGainedFocus 方法，最后调用到的是 WindowManagerService 的 inputMethodClientHasFocus ，他会检查当前 window 的 inputMethodManager 和传进来的 inputMethodManager 是不是同一个，如果是则可以继续走，因为我这里传进来的 Context 是主进程的 就和当前的 Window 是同一个 imm，就完成了。

这个输入法的适配可以说是误打误撞完成的，感觉这一块还有很多的知识值得我继续去深入学习，希望后面能有更多的机会来研读源码。

