---
layout: post
title: Spawn Library Activity 和 Fragment 改造方案
categories: [Android 实践]
description: 如何让代码结构更加灵活
keywords: Android, 代码重构
---

## Spawn Library 的优势

在公司 Android 业务中，为了适应快速开发，Spawn Library 起到了不可磨灭的作用。

在 Spawn Library 设计之初，使用 `Loader` 作为加载器，并提供了`RequestLoader` ，可以通过继承 `Request` 访问不同的接口解析数据并返回，使用 PullToRefresh 作为下拉刷新组件，提供加载状态的占屏动画。业务方通过继承基类就能快速实现列表页面、详情页面。

可以说，这些都是非常优秀的设计。Spawn Library 充分满足了在业务快速发展阶段对开发效率的要求。

![Spawn Library](https://github.com/Alonexx/alonexx.github.io/raw/master/images/android_practice/2016-07-30-spawn-library-class-uml.png)

## Spawn Library 存在的不足

但是随着技术的发展和业务的逐渐稳定，Spawn Library 的一些不足开始展现出来：

1. Spawn Library 鼓励继承（inheritance）而不是组合（combination）。这导致了类的继承层级过深，虚方法混乱，非常不便于扩展。

2. Spawn Library 基类绑定了布局，当业务需要变化（比如在布局中添加一个 Header 、添加一个浮层、通过多个接口访问数据）时就不能很好的适应了。业务方如果仍想使用 Spawn Library 基类的功能，必须拷贝一份代码修改或使用特殊的方式修改基类的布局。这让业务方的代码变得令人困惑、难以理解。

3. Spawn Library 基类绑定了数据加载方式必须使用 `Loader` 。如今，平台推广使用 Retrofit ，其表达能力更强，代码更少，不再需要 `Loader` 作为加载方式。但是基类又提供了 `LoaderCallbacks` 的虚方法，子类必须写一个空实现。

## 需要解决的几个问题

1. 最重要的就是解除和 Loader 的绑定。

2. 看看能否优化布局层级。

3. Activity 和 Fragment 的形式能否更灵活。使用 RecyclerView 替换 ListView 之后，如何解决 HeaderView 的问题？能不能使用 CoordinatorLayout ？能不能使用 Toolbar ？让 Activity 的形式表现为几个轮子拼起来，而不是在一个大轮子上各种继承，靠查看基类源代码来理解基类到底干了啥。

## 解决方案

初步构想是将类的层级降低，使用组合代替继承。将在 Activity 和 Fragment 的生命周期中添加的逻辑使用 delegate 来实现，多个 delegate 组合起来形成链条，在生命周期中调用。对于有关视图的逻辑，比如下拉刷新、加载状态使用 View 组件进行封装，而不以基类的形式扩展。

![New Base Library](https://github.com/Alonexx/alonexx.github.io/raw/master/images/android_practice/2016-07-30-new-base-library-class-uml.png)

这样做的好处有：

1. Fragment 和 Activity 的视图可以更自由地扩展，比如支持 Toolbar、RecyclerView，使用 CoordinatorLayout、ConstraintLayout。
2. 视图是否提供加载状态展示、下拉刷新完全由上层决定，而不捆绑在基类中。
3. 不再使用 Loader 作为加载器。
4. 资源清理。
因为各个业务方的代码要集成到平台 APP 中，这就产生了一个问题：有一些类的静态成员比如 Gson 的实例、供 Retrofit 使用的一些 Executors 会永远存在在内存中，这会增加平台内存的负担。如何能实现在业务方所有 Activity 都退出时清理这些静态成员呢？ Application 提供了 registerActivityLifecycleCallbacks() 方法，这个代码要写进平台的 Application 中，并且会影响所有的 Activity 创建和销毁，这显然不合适。因此想到了引用计数器方法。在 Activity 启动和销毁的时候回调计数器，这样就可以在 Activity 启动的时候加载静态成员、在 Activity 被销毁时检查资源是否还有引用，如果没有就清理这些资源。
