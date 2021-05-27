---
title: "Flutter Http请求"
date: 2019-04-23T16:34:46+08:00
draft: true
---

Dart提供了http package来进行网络请求。

### 1.添加http package

```
dependencies:
  http: <latest_version>
```
### 2.发起网络请求

Get请求

```dart
Future<Post> fetchPost() async {
  final response = await http.get('https://jsonplaceholder.typicode.com/posts/1'); //请求
  final json = JSON.decode(response.body);  //解析
  return new Post.fromJson(json); //将map转化为Post对象
}
```

Post请求

```dart
Future<Post> fetchPost() async {
  final response =
      await http.post('https://github.com/login/oauth/access_token', headers: {
    'Accept': 'application/json',
  }, body: {
    'client_id': '6b598bb603fb043814ef',
    'client_secret': 'd8c0d51d07402360e0e381dc9b94dfcdb71c8630',
    'code': 'c70104cbe626195072af'
  });
  print(response.body);
  final responseJson = json.decode(response.body);
  return new Post.fromJson(responseJson);
}
```

### 3.显示数据


为了获取数据并将其显示在屏幕上，我们可以使用FutureBuilder widget！FutureBuilder Widget可以很容易与异步数据源一起工作。

我们需要提供两个参数：

1. future参数是一个异步的网络请求，在这个例子中，我们传递调用fetchPost()函数的返回值(一个future)。

2. 一个 builder 函数，这告诉Flutter在future执行的不同阶段应该如何渲染界面。

```dart
new FutureBuilder<Post>(
  future: fetchPost(),
  builder: (context, snapshot) {
    if (snapshot.hasData) {
      return new Text(snapshot.data.title);
    } else if (snapshot.hasError) {
      return new Text("${snapshot.error}");
    }
   
    // By default, show a loading spinner
    return new CircularProgressIndicator();
  },
);
```

dio是另外一个强大的Dart Http请求库，[文档](https://github.com/flutterchina/dio/blob/master/README-ZH.md)写的非常详细，这里不再赘述。

## 参考

* [从互联网上获取数据](https://flutterchina.club/cookbook/networking/fetch-data/)

