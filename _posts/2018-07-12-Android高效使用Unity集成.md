---
layout:     post
title:    Android高效使用Unity插件
subtitle:   Android制作原生页面,Unity渲染3D界面
date:       2018-07-12
author:    Lmx 
header-img: 
catalog: true
tags: 
    - Android
    - Unity
---
# 前沿
接[上一篇文章](https://lmx01.github.io/2018/07/06/Android%E4%B8%8EUnity%E9%9B%86%E6%88%90/)描述,我们已经完成Android与Unity间的相互调用，但存在一个问题，就是每次启动一个Unity的等待时间会非常久，这样的使用体验肯定不好，本篇将讲述一个优化的方法。
# 思路
在Unity整个工程中，其实只有一个Activity，里面的场景切换，都只是对该Activity进行View的渲染切换。在Android使用Unity插件时，整个Unity的生命周期跟当前Activity（即UnityPlayer对象所在的Activity）的生命周期相关联。所以，结合这种机制，在我们的Android工程中，也应该使用单Activity架构。接着在Activity启动时，伴随着启动Unity，然后，在需要用到Unity时，将它唤醒，不需要使用时，让它休眠，而不是杀掉。这样，Unity就只需启动一次，便可更高效的使用。
# 实现
## 1. Android单Activity架构
我目前使用的是一个开源的架构-[Fragmentation](https://github.com/YoKeyword/Fragmentation),是一个基于Fragment切换更新视图的架构。本文将结合这个架构，实现Unity的高效使用。
## 2. Activity实现Unity接口
结合上一篇文章，在Activity中实现Unity需要重载的函数与接口即可。然后，所有的视图公用一个UnityPlayer对象。
这里要注意的是，实现重载一些回调事件的时候，应该自己加个标志位，判断当前视图是否处于Unity中，如果不处于Unity中，就不应该把这些事件交由Unity处理。
## 3. 启动
- 编写一个专门用于启动Unity的Fragment。在该fragment中，实现Unity从resume到pause的过程。在iOS中，unity启动结束后，会有一个回调事件，用户可以在该回调事件中，启动其他视图。Android中，目前本人没有找到类似的回调方法，目前使用的方法，是Unity在初始化结束后，手动调用一个Android方法。在该方法中，去启动别的视图。
- 另外，Unity启动，会有较常时间，处于白屏状态。我的做法，是用一个FrameLayout容器，包含一个用于启动的ImageView与Unity View，然后，将ImageView放在最上端，这样就可以把它当作app的使用。
## Unity视图的显示与隐藏
- 启动：再编写一个专门用于启动Unity场景的Fragment。同样在该fragment中，实现Unity从resume到pause的过程。在resume后，使用UnityPlayer.UnitySendMessage方法唤醒具体的场景，再退出时，同样可使用UnityPlayer.UnitySendMessage方法,通知场景结束,然后关闭当前Fragment即可
# 异常解决
- 切换Unity时，横屏与竖屏异常
在Unity视图中，一般是使用横屏，而Android原生的界面一般是使用纵屏。在实际使用中，因为Unity在切换横屏时，偶尔会出现进入时，Unity视图被切掉一部分，而不能全屏的问题，我的解决方案时在启动Unity所在的Fragment时，提前先将Fragment切换为横屏，退出时，切换为纵屏解决。
- 全屏与非全屏异常
解决方案与上面一样，都是在进入Unity前，先更改所在的父容器Fragment的状态即可解决。