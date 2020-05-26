---
layout: post
title: Flutter框架搭建之网络请求封装
date: 2019-10-25
tags: [flutter,网络,Dio]
author: loren1994
category: blog
---

# Flutter框架搭建之网络请求封装

### 网络请求库

##### 常见网络请求库如下:

* [$\color{red}{Dio}$](https://github.com/flutterchina/dio)

* 自带HttpClient
* 三方库Http

###### 封装网络请求起码包含RESTful请求和统一拦截器，这也是最基本的需求。Dio功能很强大，支持Restful API、FormData、拦截器、请求取消、Cookie管理、文件上传/下载、超时、自定义适配器等，具体可参考其Github。本文章将采用Dio，从最基本的需求出发进行封装网络请求。

> Dio会对返回的json数据自动进行解析，这是特别要注意的地方。

##### 添加依赖

~~~~yaml
dependencies:
  dio: ^3.0.3
~~~~



##### 接口定义

首先需要一个dart文件进行接口url的定义。

~~~~ dart
class Api {
  static const host = "http://192.168.3.168:9999";

  //登录
  static const LOGIN = "$host/users/login";

  //测试
  static const MODULE_TREE = "$host/auth/acl_module/tree";
}
~~~~

##### 接口返回数据

~~~~json
{
    "code":0,
    "subCode":0,
    "msg":"OK",
    "success":true,
    "data":""
}
~~~~

##### DioUtil

> 注释较为详细了。

~~~~dart
class DioUtil {
  factory DioUtil() => _getInstance();

  //一个单例
  static DioUtil get instance => _getInstance();
  static DioUtil _instance;

  static Dio _dio;

  DioUtil._internal() {
    //初始化Dio
    BaseOptions _options = BaseOptions(
      baseUrl: Api.host,
      connectTimeout: 5000,
      sendTimeout: 5000,
      receiveTimeout: 5000,
    );
    _dio = new Dio(_options);
    //dio全局拦截器
    _dio.interceptors
        .add(InterceptorsWrapper(
         //请求拦截
         onRequest: (RequestOptions options) async {
      //携带本地存储的session id
      UserInfo.getSession().then((session) {
        options.headers["Cookie"] = session;
      });
      return options;
       //错误拦截 错误类型为DioError
    }, onError: (dioError) async {
      //不是每个错误都有response
      if (null == dioError.response) return dioError;
      //http请求的状态判断，若接口返回json则response为解析过得json结构
      //也就拿不到http请求的状态码了
      if (dioError.response.statusCode == 401) {
        //session过期
        eventBus.fire(ReLoginEvent());//eventbus发送重新登录消息
        return 0;//自定义一个返回，在_error()中做判断
      } else if (dioError.response.statusCode == 403) {
        //无权限
        eventBus.fire(UnAuthorityEvent());//eventbus发送无权限的消息
        return 0;//自定义一个返回，在_error()中做判断
      } else {
        return dioError;
      }
    }));
    //开启请求日志
    _dio.interceptors.add(LogInterceptor(responseBody: false));
  }

  static DioUtil _getInstance() {
    if (_instance == null) {
      _instance = new DioUtil._internal();
    }
    return _instance;
  }

  //GET请求
  get(url,
      {data,
      Function onSuccess,
      Function onFail,
      bool saveSession = false}) async {
    await _dio.get(url, queryParameters: data).then((Response res) {
      _success(res, onSuccess, onFail, saveSession);
    }).catchError((e) {
      _error(e, onFail);
    });
  }

  //POST请求
  post(url, {data, Function onSuccess, Function onFail}) async {
    await _dio.post(url, queryParameters: data).then((Response res) {
      _success(res, onSuccess, onFail, false);
    }).catchError((e) {
      _error(e, onFail);
    });
  }

  //成功处理
  _success(Response res, Function onSuccess, Function onFail, saveSession) {
    debugPrint("_success : ${res.data}");
    var result = BaseResult.fromJsonExceptData(res.data);
    if (result.success) {
      //本地存储session id
      if (saveSession) {
        var session = res.headers["set-cookie"]
            .toString()
            .split(";")[0]
            .replaceFirst("[", "")
            .toString();
        UserInfo.setSession(session);
      }
      onSuccess(res.data);
    } else {
      onFail(result.msg);
    }
  }

  //失败处理
  _error(dynamic e, Function onFail) {
    debugPrint("catchError : $e");
    if (e.toString().contains("Connection timed out")) {
      onFail("连接超时");
    } else if (e.toString().contains("Network is unreachable")) {
      onFail("网络不可达");
    } else if (e.toString().contains("Connection refused")) {
      onFail("拒绝连接");
    } else if (e.toString().contains("No route to host")) {
      onFail("没有到主机的路由");
    } else if (e.message.toString() == "0") { //错误拦截里的判断，防止多次回调
      debugPrint("已拦截");
    } else {
      onFail("未知错误");
    }
  }
}
~~~~

##### 使用

~~~~dart
DioUtil.instance.get(Api.MODULE_TREE, onSuccess: (resp) {
     ///TODO
    }, onFail: (error) {
      ///TODO
    });
//POST同理
~~~~

##### 一些问题

* dio错误拦截里必须return出，所以无法真正拦截，项目里return 0，之后在_error中判断e.message即可。
* 本文用的dio^3.0.3，api和2.x和1.x相比差别甚大，需注意区分。

