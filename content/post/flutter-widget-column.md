---
title: "Column和Row详解"
date: 2019-03-19T14:21:37+08:00
draft: false
toc: true
---

### MainAxisAlignment

```dart
Scaffold(
    appBar: AppBar(title: Text('Flutter')),
    body: Column(
        children: <Widget>[
        Text("MainAxisAlignment.start",style:TextStyle(
            color: Colors.blueAccent,
            fontSize: 18
        )),
        Row(
            mainAxisAlignment: MainAxisAlignment.start,
            children: <Widget>[
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            ],
        ),
        Text("MainAxisAlignment.center",style:TextStyle(
            color: Colors.blueAccent,
            fontSize: 18
        )),
        Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: <Widget>[
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            ],
        ),
        Text("MainAxisAlignment.spaceAround",style:TextStyle(
            color: Colors.blueAccent,
            fontSize: 18
        )),
        Row(
            mainAxisAlignment: MainAxisAlignment.spaceAround,
            children: <Widget>[
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            ],
        ),
        Text("MainAxisAlignment.spaceBetween",style:TextStyle(
            color: Colors.blueAccent,
            fontSize: 18
        )),
        Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: <Widget>[
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            ],
        ),
        Text("MainAxisAlignment.spaceEvenly",style:TextStyle(
            color: Colors.blueAccent,
            fontSize: 18
        )),
        Row(
            mainAxisAlignment: MainAxisAlignment.spaceEvenly,
            children: <Widget>[
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            ],
        ),
        Text("MainAxisAlignment.end",style:TextStyle(
            color: Colors.blueAccent,
            fontSize: 18
        )),
        Row(
            mainAxisAlignment: MainAxisAlignment.end,
            children: <Widget>[
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            Icon(Icons.star,color: Colors.yellow, size: 50),
            ],
        )
        ],
    ),
),
```

![](/images/flutter-main-axis-aligment.png)


### crossAxisAlignment

```dart
Scaffold(
    appBar: AppBar(title: Text('Flutter')),
    body: Column(
        children: <Widget>[
        Text("CrossAxisAlignment.start",style:TextStyle(
            color: Colors.blueAccent,
            fontSize: 18
        )),
        Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: <Widget>[
            Icon(Icons.star,color: Colors.yellow, size: 30),
            Icon(Icons.star,color: Colors.yellow, size: 60),
            Icon(Icons.star,color: Colors.yellow, size: 30),
            ],
        ),
        Text("CrossAxisAlignment.center",style:TextStyle(
            color: Colors.blueAccent,
            fontSize: 18
        )),
        Row(
            crossAxisAlignment: CrossAxisAlignment.center,
            children: <Widget>[
            Icon(Icons.star,color: Colors.yellow, size: 30),
            Icon(Icons.star,color: Colors.yellow, size: 60),
            Icon(Icons.star,color: Colors.yellow, size: 30),
            ],
        ),


        Text(" CrossAxisAlignment.end",style:TextStyle(
            color: Colors.blueAccent,
            fontSize: 18
        )),
        Row(
            crossAxisAlignment: CrossAxisAlignment.end,
            children: <Widget>[
            Icon(Icons.star,color: Colors.yellow, size: 30),
            Icon(Icons.star,color: Colors.yellow, size: 60),
            Icon(Icons.star,color: Colors.yellow, size: 30),
            ],
        )
        ],
    ),
),
```

![](/images/flutter-cross-axis-alignment.png)
## 参考

* [Flutter Layout Cheat Sheet](https://medium.com/flutter-community/flutter-layout-cheat-sheet-5363348d037e)
* [Flutter快速上车之Widget](https://mp.weixin.qq.com/s/kAWPj97w5NfiqAYEpi3zVA)
