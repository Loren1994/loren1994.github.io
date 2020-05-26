---
layout: post
title: Flutter单例模式
date: 2020-05-26
tags: [flutter,dart,单例]
author: loren1994
category: blog
---

# Flutter单例模式

#### 分析

不搞骚操作的话，Flutter代码都是单线程运行的，至于线程安全，我找到了这样一段文章，似乎在说明在Flutter层上不用过于在意线程安全：

Platform Task Runner

Flutter Engine的主Task Runner，类似于Android Main Thread或者iOS的Main Thread。但是需要注意他们还是有区别的。

一般来说，一个Flutter应用启动的时候会创建一个Engine实例，Engine创建的时候会创建一个线程供Platform Runner使用。跟Flutter Engine的所有交互（接口调用）必须在Platform Thread进行，否则可能导致无法预期的异常。这跟iOS UI相关的操作都必须在主线程进行相类似。需要注意的是在Flutter Engine中有很多模块都是非线程安全的。

规则很简单，对于Flutter Engine的接口调用都需保证在Platform Thread进行。阻塞Platform Thread不会直接导致Flutter应用的卡顿（跟iOS android主线程不同）。尽管如此，也不建议在这个Runner执行繁重的操作，长时间卡住Platform Thread应用有可能会被系统Watchdog强杀。

#### 实现

* 懒汉式

~~~~dart
class SingleClazz {
  static SingleClazz _instance;

  factory SingleClazz() => _getInstance();

  static SingleClazz _getInstance() {
    if (_instance == null) {
      _instance = SingleClazz._internal();
    }
    return _instance;
  }

	//命名构造函数
  SingleClazz._internal(); //可在此初始化一些引用
}
~~~~

* 饿汉式

~~~~dart
class SingleClazz {
  static SingleClazz _instance = SingleClazz._internal();
  
  factory SingleClazz() => _getInstance();
  
  static SingleClazz _getInstance() {
    return _instance;
  }
  
  SingleClazz._internal();
}
~~~~

#### 使用场景

* 需要频繁实例化然后销毁的对象
* 创建对象时耗时过长、资源消耗过多，但又经常用到
* 有状态的工具类
* 频繁访问数据库或文件的对象