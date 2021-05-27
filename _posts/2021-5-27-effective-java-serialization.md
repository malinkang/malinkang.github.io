---
title: 《Effective Java》第12章序列化
date: 2016-03-12 09:02:08
tags: ["Effective Java"]
---

对象序列化（object serialization）API，提供了一个框架，用来将对象编码成字节流，并从字节流编码中重新构建对象。“将一个对象编码成一个字节流”，称作将该对象序列化（serializing）；相反的处理过程被称作反序列化（deserializing）。一旦对象被序列化后，它的编码就可以从一台正在运行的虚拟机被传递到另一台虚拟机上，或者被存储到磁盘上，供以后反序列化使用。

## 第74条：谨慎地实现Serializable接口

要想使一个类的实例可被序列化，非常简单，只要在它的声明中加入“implements Serializable”字样即可。正因为太容易了，所以普遍存在这样一种误解，认为程序员毫不费力就可以实现序列化。实际的情形要复杂得多。虽然使一个类可被序列化的直接开销非常低，甚至可以忽略不计，但是为了序列化而付出的长期开销往往是实实在在的。

实现Serializable接口而付出的最大代价是，一旦一个类被发布，就大大降低了“改变这个类的实现”的灵活性。

实现Serializable的第二个代价是，它增加了出现Bug和安全漏洞的可能性。

实现Serializable的第三个代价是，随着类发行新的版本，相关的测试负担也增加了。
