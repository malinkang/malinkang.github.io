---
title: "Container详解"
date: 2019-02-19T14:21:37+08:00
draft: false
toc: true
---



`Container`是一个拥有绘制、定位、调整大小的`widget`。



### padding和margin

padding和margin分别设置`Container`的内边距和外边距。可取值包括下面四个：

* EdgeInsets.all(50)：设置所有的padding为同一个值50。
* EdgeInsets.only(left: 50,right: 50)：只设置左边和右边。
* EdgeInsets.fromLTRB(50,10,50,10)：分别设置左上右下的值为50、10。
* EdgeInsets.symmetric(vertical: 10,horizontal: 50)：如果上下或者左右的padding值一样可以指定vertical的值为上下的padding值。horizontal指定左右的padding值。

```dart
Scaffold(
    appBar: AppBar(title: Text('Container')),
    body: Center(
    child: Column(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: <Widget>[
        Container(
            padding: EdgeInsets.all(50),
            decoration: BoxDecoration(
                border: Border.all(color: Colors.red, width: 1),
                borderRadius: BorderRadius.all(Radius.circular(20))),
            child: Text(
            "确定",
            style: TextStyle(color: Colors.red),
            ),
        ),
        Container(
            padding: EdgeInsets.only(left: 50,right: 50),
            decoration: BoxDecoration(
                border: Border.all(color: Colors.red, width: 1),
                borderRadius: BorderRadius.all(Radius.circular(20))),
            child: Text(
            "确定",
            style: TextStyle(color: Colors.red),
            ),
        ),
        Container(
            padding: EdgeInsets.fromLTRB(50, 10, 50, 10),
            decoration: BoxDecoration(
                border: Border.all(color: Colors.red, width: 1),
                borderRadius: BorderRadius.all(Radius.circular(20))),
            child: Text(
            "确定",
            style: TextStyle(color: Colors.red),
            ),
        ),
        Container(
            padding: EdgeInsets.symmetric(vertical: 10, horizontal: 50),
            decoration: BoxDecoration(
                border: Border.all(color: Colors.red, width: 1),
                borderRadius: BorderRadius.all(Radius.circular(20))),
            child: Text(
            "确定",
            style: TextStyle(color: Colors.red),
            ),
        ),
        ],
    ),
)),
```


![](/images/flutter-container-padding.png)


### width和height

`width`和`height`指定宽高，如果不指定则为子`widget`的宽高。如果想要完全撑满父容器，可以将`width`和`height`设置为`double.infinity`。

### decoration

`decoration`经常被用来改变一个`Container`的展示效果。其概念类似与android中的shape。一般实际场景中会使用他的子类BoxDecoration。BoxDecoration提供了对背景色，边框，圆角，阴影和渐变等功能的定制能力。

* `image: DecorationImage`设置一张图片作为背景。
* `border: Border`设置边界。
* `borderRadius: BorderRadius`设置边界圆角。当`shape`是`BoxShape.circle`设置`borderRadius`将不起作用
* `shape: BoxShape`设置形状。
* `gradient`设置渐变。可选值包括三种类型的渐变`LinearGradient`、`RadialGradient`、`SweepGradient`。

```dart
Scaffold(
    appBar: AppBar(title: Text('BorderRadius')),
    body: Center(
        child: Column(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: <Widget>[
            Container(
            height: 200,
            width: 200,
            decoration: BoxDecoration(
                color: Colors.yellow,
                //设置图片
                image: DecorationImage(
                    fit: BoxFit.fitWidth,
                    image: NetworkImage(
                    'https://flutter.io/images/catalog-widget-placeholder.png',
                    ),
                ),
                //设置边界
                border: Border.all(color: Colors.deepOrange, width: 3),
                //设置阴影
                boxShadow: const [
                    BoxShadow(blurRadius: 10),
                ],
                //设置边界圆角
                borderRadius: BorderRadius.all(Radius.circular(18))),
            ),
            Container(
            height: 200,
            width: 200,
            decoration: BoxDecoration(
                gradient: RadialGradient(
                //渐变
                colors: const [
                    Colors.green,
                    Colors.deepOrange,
                    Colors.pinkAccent,
                    Colors.deepPurple
                ],
                ),
                //设置边界圆角
                shape: BoxShape.circle,
            ),
            )
        ],
        ),
    ),
),
```

![](/images/flutter-box-decoration-image.png)

## 2.Column和Row

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
