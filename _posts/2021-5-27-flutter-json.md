---
title: "Flutter Json解析"
date: 2019-04-23T16:15:19+08:00
draft: true
---

```dart
import 'dart:convert';
void main() {
  var jsonString = '{"name": "John Smith","email": "john@example.com"}';
  Map<String, dynamic> user = jsonDecode(jsonString);
  print('Howdy, ${user['name']}!');
  print('We sent the verification link to ${user['email']}.');
}
```

```dart
import 'dart:convert';
class User {
  final String name;
  final String email;
  User(this.name, this.email);
  User.fromJson(Map<String, dynamic> json)
      : name = json['name'],
        email = json['email'];
  Map<String, dynamic> toJson() =>
    {
      'name': name,
      'email': email,
    };
}
void main() {
    var jsonString = '{"name": "John Smith","email": "john@example.com"}';
    Map userMap = jsonDecode(jsonString);
	  var user = new User.fromJson(userMap);
	  print('Howdy, ${user.name}!');
	  print('We sent the verification link to ${user.email}.');
}
```

## 参考

* [JSON和序列化](https://flutterchina.club/json/)