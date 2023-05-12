---
layout:     post
title:      "操作系统文件共享"
description: "file share"
date:     2022-04-05
author: sfeng
tags: ["linux"]
URL: "/2022/04/05/file-share/"
---
# 文件共享

## 简介

linux 操作系统通过硬链接(Hard Link)与软连接(Soft Link)两种方式实现文件共享

注意：

> 多个用户共享同一个文件，意味着系统中只有`一份`文件数据。并且只要某个用户修改了该文件的数据，其他用户也可以看到文件数据的变化。                                                                            如果是多个用户都`复制`了同一个文件，那么系统中会有`好几份`文件数据。其中一个用户修改了自己的那份文件数据，对其他用户的文件数据并没有影响。
> 

## 硬链接

介绍硬链接之前，需要了解一个知识点，索引节点即inode。在操作系统中，每个文件都存在目录项，目录项记录该文件的文件名与其其他元数据，比如大小，属性，在磁盘哪个块等。操作系统为了查询文件节省时间，将除了文件名的其他元数据信息都放在索引节点里，那么目录项就只记录文件名和索引节点指针两个信息，加快文件查询速度。

硬链接的实现方式为多个文件的目录项的索引节点指针指向同一个索引节点，所以通过硬链接创建的文件其属性除了文件名不同，其余属性和源文件相同。

![Untitled](/img/file-share-hard.png)

索引结点中设置一个链接计数变量count,用于表示链接到本索引结点上的用户目录项数。

若count= 2，说明此时有两个用户目录项链接到该索引结点上，或者说是有两个用户在共享此文件。

若某个用户决定删除该文件，则只是要把用户目录中与该文件对应的目录项删除，且索引结点的count值减1。若count>0，说明还有别的用户要使用该文件，暂时不能把文件数据删除，否则会导致指针悬空。

## 软连接

软连接类似于windows中的快捷方式，当User3访问`ccc` 时，操作系统判断文件`ccc`属于Link类型文件，于是会根据其中记录的路径层层查找目录，最终找到User1的目录表中的`aaa` 表项，于是就找到了文件1的索引结点。

文件1删除，但是文件2依然存在，只是通过`C:/User1/aaa`这个路径已经找不到文件1了

![Untitled](/img/file-share-soft.png)

通过软连接的文件其属性与源文件不同，因为他们指向不同的索引节点

## 比较

### 软链接：

- 软链接，以路径的形式存在，类似于Windows操作系统中的快捷方式
- 软链接可以跨文件系统
- 软链接可以对一个不存在的文件名进行链接
- 软链接可以对目录进行链接
- 软链接原文件/链接文件拥有不同的inode号，表明他们是两个不同的文件
- 当源文件目录改变后，软连接访问不到

### 硬链接

- 硬链接，以文件副本的形式存在，但不占用实际空间
- 不允许给目录创建硬链接（可以通过参数添加但仅限root用户）
- 硬链接只有在同一个文件系统中才能创建
- 不能对不存在的文件创建硬链接
- 硬链接原文件/链接文件公用一个inode号，说明他们是同一个文件
- 当源文件目录改变后，硬连接可以访问

## 实验

### 硬链接

通过 `ln` 命令来创建硬链接

```bash
# 创建硬链接
$ echo "file soruce test" > source-file
$ ln source-file source-file-hard

# 发现硬链接文件与源文件属性相同，其inode 也一样，第一行为inode
$ ll -i 
总用量 16
5638317 drwxr-xr-x  2 root root 4096 4月   5 16:55 ./
5636097 drwxrwxrwt 23 root root 4096 4月   5 16:58 ../
5638333 -rw-r--r--  2 root root   45 4月   5 16:56 source-file
5638333 -rw-r--r--  2 root root   45 4月   5 16:56 source-file-hard

# 更新硬链接文件，发现共享
$ echo "hard file" >> source-file-hard 
$ cat source-file
file soruce test
hard file
$ cat source-file-hard 
file soruce test
hard file

# 更新源文件，发现共享
$ echo "source-file-test2" >> source-file
$ cat source-file-hard 
file soruce test
hard file
source-file-test2
$ cat source-file
file soruce test
hard file
source-file-test2

# 删除源文件，发现不影响硬链接
$ rm -rf source-file
$ cat source-file-hard 
file soruce test
hard file
source-file-test2
```

### 软链接

通过 `ln -s` 创建软链接

```bash
# 为 source-dir 目录创建软链接
$ ls source-dir/
source-file
$ ln -s source-dir source-dir-soft

# 发现两目录的属性不一样，inode也不同,软连接文件很小，因为它只是一个符号文件，往该文件写入类容，都会存入源文件。
$ ll -i
总用量 12
5638317 drwxr-xr-x  3 root root 4096 4月   5 17:05 ./
5636097 drwxrwxrwt 23 root root 4096 4月   5 17:06 ../
5638333 drwxr-xr-x  2 root root 4096 4月   5 17:05 source-dir/
5638336 lrwxrwxrwx  1 root root   10 4月   5 17:05 source-dir-soft -> source-dir/

# 更新软链接文件，发现文件共享
$ echo "soft-file" > source-dir-soft/source-file 
$ cat source-dir/source-file 
soft-file
```

## 使用场景

软链接一般被用来设置可执行文件的快捷方式的路径；

日志共享

例如：

有一台服务器根目录大小只有50g，但是其服务日志默认打印在根目录下，时间久了，必然导致根目录暴涨，影响服务器运行。那么可以使用软连接的方式，首先将该日志目录移动到别的磁盘目录下，然后删除根目录下的日志目录，最后使用软连接将根目录的日志目录连接到磁盘较大的目录下。

一般比较重要的文件我们担心文件被误删除且传统复制备份方式占用double数量的空间会造成浪费，可以使用硬链做备份来解决