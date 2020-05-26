---
layout: post
title: Dart语法特性整理
date: 2020-05-26
tags: [flutter,dart,语法]
author: loren1994
category: blog
---

# Dart语法特性整理

#### 函数重载

dart不支持函数重载，但是可以使用参数默认值的方式间接支持，这也是年轻一代的语言中的共性。

~~~~dart
//1.报错 test已经定义
_test(){

}
_test(int a){

}
//2.正常
_test({int a=0}){
  
}
_test();
_test(a:2);
~~~~

#### 抽象类

抽象类使用abstract关键字定义，是不能被实例化的，通常用来定义接口以及部分实现。
但与其他语言不太一样的地方是，抽象方法也可以定义在非抽象类中。

#### Interface

每个类的内部都隐式的定义了一个接口，这个接口包含类的成员的所有实例，以及类实现的所有接口。
如果你想让A类支持B类所有的API，并且不通过继承B类来实现，那么A类应该实现B接口。
一个类可以实现一个或多个接口，通过implements关键字。

#### 多继承

Dart语言集合了现代编程语言的众多优点，Mixin继承机制也是其一。具体的读法应该叫做 mix in，翻译下就是混入。`mixins`是一种实现多重继承的方式，通过它可以给现有的类添加特性。在`with`关键字后面可以跟随一个或多个`mixin`的名称。

想要实现一个mixin，你可以创建一个继承自Object的、没有构造器的类，如果不想让mixin类被当做普通类使用的话，就用`mixin`关键字替换`class`关键字。换句话说mixin类可以使用class定义，也可以是抽象类，也可以当做普通类使用。

~~~~dart
mixin Test {
 int a=0;
 
 void foo(){
 	print("test foo");
 }
}
~~~~

> 使用`on`关键字可以指定mixin类的父类。

#### class、interface、mixin比较

Dart是没有interface这种东西的，但并不意味着这门语言没有接口，Dart任何一个类都是接口，你可以实现任何一个类，只需要重写那个类里面的所有具体方法。所以说，一个普通类class，即是普通类，也是接口，也可以当做mixin来使用。

#### 访问修饰符

Data中没有 public  private protected这些访问修饰符合但是我们可以使用 `_` 把一个属性或者方法定义成私有。

#### 构造函数

- `ClassName(...) //普通构造函数`
- `Classname.identifier(...) //命名构造函数`
- `const ClassName(...) //常量构造函数`
- `factroy ClassName(...) //工厂构造函数`

如果你定义了一个类，而没有定义构造函数，那么它将有一个默认的构造函数，这个构造函数没有参数。

#### 生成器函数

在 dart 中有生成器函数的语法，在很多其他的语言中也有，比如 js、c#，这个语法看上去和 `async` `await` 语法很像，使用的关键字是 `async*` `sync*` `yield` `yield*`官方对于这个语法的说明可以参考这个连接[generators](https://www.dartlang.org/guides/language/language-tour#generators)。其实`async` `await`也是一种生成器语法，生成器语法就是你返回的类型通常情况下和 return 的类型可能不一致，比如你`return 1`，但是返回值上却需要写Widget。

`sync*` 返回迭代器

~~~~dart
var list1 = [1,2,3];
var list2 = <Text>[];
Iterable<Widget> _buildItem(List list) sync* {
   for (int i = 0; i < list.length; i++) {
     yield Text(list[i]);
   }
}
//使用
list2 = _buildItem(list1).toList()
~~~~

`async*` 返回值是一个Stream

~~~~dart
main(List<String> arguments) {
  genStream().listen((data) {
    print("stream1 : $data");
  });
  genStream2().listen((data) {
    print("stream2 : $data");
  });
}

//生成器写法
Stream<int> genStream({int max = 10}) async* {
  int i = 0;
  while (i < max) {
    yield i;
    await Future.delayed(Duration(milliseconds: 300));
    i++;
  }
}

//普通写法
Stream<int> genStream2({int max = 10}) {
  StreamController<int> controller = StreamController();

  Future<void>.delayed(Duration.zero).then((_) async {
    int i = 0;
    while (i < max) {
      controller.add(i);
      await Future.delayed(Duration(milliseconds: 300));
      i++;
    }
    controller.close();
  });

  return controller.stream;
}
~~~~

> 两种写法达到了一样的效果，但是生成器函数代码会更加简洁一些。

`yield*` 关键字是结合递归使用的,可以配合`sync*` 也可以配合`async*`

~~~~dart
Iterable<int> naturalsDownFrom(int n) sync* {
  if (n > 0) {
    yield n;
    yield* naturalsDownFrom(n - 1);
  }
}

Stream<int> naturalsStreamDownFrom(int n) async* {
  if (n > 0) {
    yield n;
    yield* naturalsStreamDownFrom(n - 1);
  }
}
~~~~

> 官方的说法是：使用yield*会有性能优化，所以还是建议使用生成器函数。