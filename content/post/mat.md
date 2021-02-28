---
title: mat
date: 2018-01-04 18:13:54
draft: true
---

### 打开Mat中的Bitmap原图

在使用MAT查看应用程序内存使用情况的时候,我们经常会碰到`Bitmap`对象以及`BitmapDrawable$BitmapState`对象,而且在内存使用上,`Bitmap`所占用的内存占大多数.在这样的情况下, `Bitmap`所造成的内存泄露尤其严重, 需要及时发现并且及时处理.在这样的需求下, 当我们在`MAT`中发现和图片相关的内存泄露的时候, 如果能知道是那一张图片,对分析问题会有很大的帮助.


在`MAT`中打开`Dominator Tree`视图 , 选择一个`Bitmap`对象： 


![Imgur](https://i.imgur.com/H7v1sDQ.png)


查看`Inspector`窗口,内容如下图：

![Imgur](https://i.imgur.com/JINSxPI.png)


`mBuffer`的值保存的是图片的二进制数据。`mHeight`和`mWidth`对应图片宽高。我们将`mBuffer`数据保存成一个文件，**文件名必须以.data为后缀**。


![Imgur](https://i.imgur.com/st2uyqB.png)

![Imgur](https://i.imgur.com/gXSbZ86.png)


下载工具[gimp](https://www.gimp.org/downloads/)用来打开刚才的`image.data`文件。


`Image Type`选择 `RGB Alpha`，宽高值输入上面得到的宽高值，其他值保持不变，就可以看到图片了。

![Imgur](https://i.imgur.com/enBkoxJ.png)




