---
title: 常用Linux命令
date: 2017-11-07 11:03:08
draft: true
---

### mv命令

mv命令是move的缩写，可以用来移动文件或者将文件改名。

-b ：若需覆盖文件，则覆盖前先行备份。 
-f ：force 强制的意思，如果目标文件已经存在，不会询问而直接覆盖；
-i ：若目标文件 (destination) 已经存在时，就会询问是否覆盖！
-u ：若目标文件已经存在，且 source 比较新，才会更新(update)。

```sh
#将Document中的test目录移动到Downloads目录下
mv -f test  ../Downloads
```


### 参考

* [每天一个linux命令（7）：mv命令](http://www.cnblogs.com/peida/archive/2012/10/27/2743022.html)
* [shell script 读取properties 文件](http://daohao123.iteye.com/blog/1964372)
http://blog.csdn.net/bitcarmanlee/article/details/50973454



