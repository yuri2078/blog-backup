---
title: vim语法
categories: 小记
comments: true
description: '我的爱宛如孤岛之花，不为人知的绽放,不为人知的凋零！'
cover: https://www.loliapi.com/acg/?id=29
date: 2022-08-04 19:17:31
tags: 
    - vim
    - 常用命令
keywords:
---
# vim语法

## 常规用法

1. vim 文件 用vim打开文件

2. sudo vim 文件 使用su权限打开文件



## 命令行

1. wq 保存并且退出

2. q 推出不保存

3. i 切换到插入模式

4. esc 推出编辑模式，切换到命令行模式

5. h-左 j-上 k-下 l-右

6. 0 到这一行的开头

7. yy 拷贝 dd 剪切 p粘贴

8. set fileencoding 查看编码

9 :1,$s/word1/word2/g 或 :%s/word1/word2/g 用world2 替换world1