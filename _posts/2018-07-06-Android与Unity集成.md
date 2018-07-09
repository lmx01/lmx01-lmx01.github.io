---
layout:     post
title:    Android与Unity的简单集成 
subtitle:   Android制作原生页面,Unity渲染3D界面
date:       2018-07-06
author:    Lmx 
header-img: 
catalog: true
tags: 
    - Android
    - Uniy
---
# 参考：
1. [Medium:Android+Unity Integration](https://medium.com/@ashoni/android-unity-integration-47756b9d53bd)
# 一、介绍
目前，在市场上，Android SDK一般用于制作平面效果的app，而Unity一般用于制作一些具有3D 物体的app，如游戏领域里面；但是，有一种需求就是希望在Android制作
的平面app里面，渲染一些高级特效，如微博/支付宝里面制作的AR特效，个人觉得不适合使用openGL执行操作渲染，因为openGL代码过于底层，不适合制作太多的高级
特效，推测这些使用的是unity。所以，本文参考网上一些资料，总结Android SDK直接调用Unity的方法。
#  二、Unity导出Android Studio工程
（待定）
# 三、Android Studio使用Unity导出的工程
## 1. 在AndroidMainifest.xml中，给Activity添加如下属性：
```
<meta-data
    android:name="unityplayer.UnityActivity"
    android:value="true" />
<!--设置该属性后，unity将不会弹出权限访问窗口，用户需要自行在别的地方添加所需权限-->
<meta-data
    android:name="unityplayer.SkipPermissionsDialog"
    android:value="true" />
```
## 2. UnityPlayer  mUnityPlayer
在Android里面，把Unity当做一个视图View，在使用的时候，需要在当前Activity类覆写如下方法：
```
public UnityPlayer mUnityPlayer = null; //变量名字不能变

@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mUnityPlayer = new UnityPlayer(this);
}

@Override
protected void onResume() {
    super.onResume();
    mUnityPlayer.resume();
}

@Override
protected void onPause() {
    super.onPause();
    mUnityPlayer.pause();
}

@Override
protected void onDestroy() {
    mUnityPlayer.quit();
    super.onDestroy();
}

// This ensures the layout will be correct.
@Override
public void onConfigurationChanged(Configuration newConfig) {
    super.onConfigurationChanged(newConfig);
    mUnityPlayer.configurationChanged(newConfig);
}

// Notify Unity of the focus change.
@Override
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    mUnityPlayer.windowFocusChanged(hasFocus);
}

// For some reason the multiple keyevent type is not supported by the ndk.
// Force event injection by overriding dispatchKeyEvent().
@Override
public boolean dispatchKeyEvent(KeyEvent event) {
    if (event.getAction() == KeyEvent.ACTION_MULTIPLE)
        return mUnityPlayer.injectEvent(event);
    return super.dispatchKeyEvent(event);
}

@Override
public boolean onKeyDown(int keyCode, KeyEvent event) {
    return mUnityPlayer.injectEvent(event);
 }

 @Override
 public boolean onTouchEvent(MotionEvent event) {
    return mUnityPlayer.injectEvent(event);
 }

 /*API12*/
 public boolean onGenericMotionEvent(MotionEvent event) {
    return mUnityPlayer.injectEvent(event);
 }
```

## 3. Android与Unity间方法调用
- Android调用Unity方法：
```
    /*
    * 第一个参数是Unity中的对象名字
    * 第二个参数是函数名
    * 第三个参数是传给函数的参数，目前参数只有这个String接口
    */
    UnityPlayer.UnitySendMessage(String,String,String);
```

- Android实现Unity所需的方法：
    只需在当前类中，将方法定义为public即可。
    Unity中使用C#脚本调用：
```
using (AndroidJavaClass cls_UnityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer"))
{
    using (AndroidJavaObject obj_Activity = cls_UnityPlayer.GetStatic<AndroidJavaObject>("currentActivity"))
    {
        obj_Activity.Call("function name", "arg");
        obj_Activity.CallStatic("static function name");
    }
}
```
# 四、使用陷阱
## 1. Unity退出时，杀死整个app
- 原因：
    查看Unity的导出包中，发现mUnityPlayer.quit()函数里面调用了this.kill()，直接杀死了整个进程。之所以这么做，是因为在Unity中，整个app其实只有一个Activity，Unity中的场景变化都只是进行View切换，所以当用户关闭Activity时，就认为app结束，杀死整个进程；
```
public void quit() {
    ...
    this.kill();
    ...
}

protected void kill() {
    Process.killProcess(Process.myPid());
}
```
- 解决方法有如下两种：
    - 重载kill方法：
```
public class MyUnityPlayer extends UnityPlayer {
    @Override
    protected void kill() {
    }
}
```
**理论上，重载该方法后，quit方法将不会调用killProcess，但实际使用时，程序比较奇怪，偶尔还是会杀死进程,目前没有解决。如果你已经解决了，还麻烦告诉我下，为啥，谢谢！**

    - 将Unity所在的Activity当作一个新的进程
给当前Activity添加如下属性：
```
android:process="自己取的进程名，一般格式是包路径+类名"
```

