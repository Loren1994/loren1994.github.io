---
layout: post
title: Flutter框架搭建之本地存储
date: 2019-10-25
tags: [flutter,sp,存储]
author: loren1994
category: blog
---

# Flutter框架搭建之本地存储

### SharedPreferences

> 针对android端，利用android的sharedPreference做本地存储。实际还是调用android原生api。

##### 添加依赖

~~~~yaml
dependencies:
    shared_preferences: ^0.5.3+4
~~~~

##### 封装思路

###### 封装一个单例，满足常用数据类型的存储，可再封装一个公共类存储具体内容。

> sp存储为异步，返回类型只能为Future。

##### SpUtil

~~~~dart
class SpUtil {
  static SpUtil _instance;

  static Future<SpUtil> get instance async {
    return await getInstance();
  }

  static SharedPreferences _spf;

  SpUtil._();

  Future _init() async {
    _spf = await SharedPreferences.getInstance();
  }

  static Future<SpUtil> getInstance() async {
    if (_instance == null) {
      _instance = new SpUtil._();
    }
    if (_spf == null) {
      await _instance._init();
    }
    return _instance;
  }

  static bool _beforeCheck() {
    if (_spf == null) {
      return true;
    }
    return false;
  }

  // 判断是否存在数据
  bool hasKey(String key) {
    Set keys = getKeys();
    return keys.contains(key);
  }

  Set<String> getKeys() {
    if (_beforeCheck()) return null;
    return _spf.getKeys();
  }

  get(String key) {
    if (_beforeCheck()) return null;
    return _spf.get(key);
  }

  getString(String key) {
    if (_beforeCheck()) return null;
    return _spf.getString(key);
  }

  Future<bool> putString(String key, String value) {
    if (_beforeCheck()) return null;
    return _spf.setString(key, value);
  }

  bool getBool(String key) {
    if (_beforeCheck()) return null;
    return _spf.getBool(key);
  }

  Future<bool> putBool(String key, bool value) {
    if (_beforeCheck()) return null;
    return _spf.setBool(key, value);
  }

  int getInt(String key) {
    if (_beforeCheck()) return null;
    return _spf.getInt(key);
  }

  Future<bool> putInt(String key, int value) {
    if (_beforeCheck()) return null;
    return _spf.setInt(key, value);
  }

  double getDouble(String key) {
    if (_beforeCheck()) return null;
    return _spf.getDouble(key);
  }

  Future<bool> putDouble(String key, double value) {
    if (_beforeCheck()) return null;
    return _spf.setDouble(key, value);
  }

  List<String> getStringList(String key) {
    return _spf.getStringList(key);
  }

  Future<bool> putStringList(String key, List<String> value) {
    if (_beforeCheck()) return null;
    return _spf.setStringList(key, value);
  }

  dynamic getDynamic(String key) {
    if (_beforeCheck()) return null;
    return _spf.get(key);
  }

  Future<bool> remove(String key) {
    if (_beforeCheck()) return null;
    return _spf.remove(key);
  }

  Future<bool> clear() {
    if (_beforeCheck()) return null;
    return _spf.clear();
  }
}
~~~~

##### UserInfo

~~~~dart
class UserInfo {
  //session
  static setSession(String session) async {
    await SpUtil.getInstance().then((sp) {
      sp.putString("session", session);
    });
  }

  static Future<String> getSession() async {
    return await SpUtil.getInstance().then((sp) {
      return sp.getString("session");
    });
  }
    
  //用户名
  static setUserName(String name) async {
    await SpUtil.getInstance().then((sp) {
      sp.putString("userName", name);
    });
  }

  static Future<String> getUserName() async {
    return await SpUtil.getInstance().then((sp) {
      return sp.getString("userName");
    });
  }

  //清除所有
  static clear() async {
    await SpUtil.getInstance().then((sp) {
      sp.clear();
    });
  }
}
~~~~

##### 使用

~~~~dart
//获取数据
_getUserInfo() async {
	UserInfo.getUserName().then((name) {
      setState(() {
        userName = name ?? "";
      });
    });
	//或者这样写
	var name = await UserInfo.getUserName();
	setState(() {
 	 userName = name ?? "";
	});
}
~~~~

> 存储过程为异步，返回为future，需要在then中拿到返回结果，由于是单例模式，所以第一次调用SpUtil.getInstance()时稍微慢一些，之后存取都会很快，个位毫秒数。

##### 另一个常用库 - fluttertoast

~~~~yaml
fluttertoast: ^3.1.3
~~~~

##### 使用

~~~~dart
Fluttertoast.showToast(msg: "test");
~~~~

> 原理同sharedPreference，都是最终调用原生Api。