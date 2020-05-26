---
layout: post
title: Flutter之BLoC模式
date: 2020-05-26
tags: [flutter,单例,bloc]
author: loren1994
category: blog
---

# Flutter之BLoC模式

#### BLoC

其全称为 Business Logic Component，表示为业务逻辑组件，简称 BLoC。

要使用BLoC模式，就离不开stream，通过stream可以实现UI与业务逻辑的分离，分离后有以下好处：

* 业务改动不影响UI页面，或UI可以尽可能少的改动
* 改动UI时不会对业务逻辑造成影响

使用stream，可以不必setState，大大减少了build次数，使得状态管理更加简洁、统一。

#### BLoC封装

* BaseBloc

~~~~dart
abstract class BaseBloc {
  void dispose();
}
~~~~

> 定义一些公共接口

* BlocProvider

~~~~dart
import 'package:flutter/material.dart';
import 'package:gridleader/blocs/base_bloc.dart';

Type _typeOf<T>() => T;

class BlocProvider<T extends BaseBloc> extends StatefulWidget {
  final T bloc;
  final Widget child;

  const BlocProvider({
    Key key,
    @required this.child,
    @required this.bloc,
  }) : super(key: key);

  @override
  _BlocProviderState createState() => _BlocProviderState<T>();

  static T of<T extends BaseBloc>(BuildContext context) {
    _BlocProviderInherited<T> provider =
        context.getElementForInheritedWidgetOfExactType<_BlocProviderInherited<T>>()?.widget;
    return provider?.bloc;
  }
}

class _BlocProviderState<T extends BaseBloc> extends State<BlocProvider<T>> {
  @override
  Widget build(BuildContext context) {
    return _BlocProviderInherited<T>(
      bloc: widget.bloc,
      child: widget.child,
    );
  }

  @override
  void dispose() {
    widget.bloc?.dispose();
    super.dispose();
  }
}

class _BlocProviderInherited<T> extends InheritedWidget {
  _BlocProviderInherited({
    Key key,
    @required Widget child,
    @required this.bloc,
  }) : super(key: key, child: child);

  final T bloc;

  @override
  bool updateShouldNotify(InheritedWidget oldWidget) {
    return false;
  }
}
~~~~

> bloc管理类，使用一个StatefulWidget将bloc和page对应起来，使用时一目了然，dispose时会释放bloc，方便了管理。
>
> InheritedWidget 是在树中高效向下传递信息的基类部件。使用InheritedWidget的好处是当在页面中获取对应的bloc时更加快速、性能更好。[InheritedWidget 的运用与源码解析](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU0OTk2MjMwNA%3D%3D%26mid%3D2247483946%26idx%3D1%26sn%3De7f4c267f3c115a6984b4f3ba04deee8%26chksm%3Dfba69505ccd11c1336e6ebb591447976c69665a7c0aea0926df8e5890e72198b4e9e07a2525c%26token%3D1384658154%26lang%3Dzh_CN%23rd)

* BusinessBloc

~~~~dart
class BusinessBloc implements BaseBloc {
  static BusinessBloc _instance;

  factory BusinessBloc() => _getInstance();

  static BusinessBloc _getInstance() {
    if (_instance == null) {
      _instance = BusinessBloc._internal();
    }
    return _instance;
  }

  BusinessBloc._internal();

  StreamController<TestBean> _streamController = StreamController.broadcast();
  Stream<TestBean> get stream => _streamController.stream;

  getData(String doorplateId) {
    DioUtil.instance.get(Api.TEST_URL, onSuccess: (resp) {
      TestBean result = BaseResult.fromJson(resp).getObject();
      _streamController.sink.add(result);
    }, onFail: (error) {
      Fluttertoast.showToast(msg: error);
    });
  }

  @override
  void dispose() {
    _streamController.close();
    _instance = null;
  }
}
~~~~

> 单例模式的bloc。需根据使用场景绝定是否需要用单例模式。

* 页面使用

~~~~dart
@override
void initState() {
    super.initState();
    SVResidentBloc _bloc = BlocProvider.of(context);
    _bloc.getData(widget.doorplateId);
}

//build中return的Widget
StreamBuilder<TestBean>(
                      stream: _bloc.orgTreeStream,
                      builder: (context, snapdata) => snapdata.data == null
                          ? CircularProgressIndicatorContainer()
                          : Text(snapdata.data.testName))
~~~~

> 为防止数据加载过程中snapdata.data为空导致的报错，要判断为空时显示加载进度组件。

* 页面引用时

~~~~dart
BlocProvider(child: TestPage(), bloc: BusinessBloc())
~~~~

