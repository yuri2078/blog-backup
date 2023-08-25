---
title: vscode-apollo
avatar: https://cdn.jsdelivr.net/gh/yuri2078/images/img/custom/avatar.jpg
categories: apollo
cover: https://www.loliapi.com/acg/?id=31
date: 2023-01-10 20:04:19
tags: 
   - apollo
   - vscode
excerpt: 这是文章的摘要内容。
---

# 使用vscode 查看apollo源码


## 下载vscode 

> 如果使用的wsl请自己百度vscode 链接wsl

1. 官方[下载连接](https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-x64)

2. 下载下载应该是 `code_1.74.2-1671533413_amd64.deb`

3. 打开终端输入

   ```
   cd Downloads 
   sudo dpkg -i code_1.74.2-1671533413_amd64.deb
   ```

4. 等安装完成之后直接终端输入`code` 或者 在桌面打开就行



## 安装插件

1. 进入vscode后点击这里搜索插件进行安装![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230110_211429.png)
2. 一共需要安装5个插件
   1. **c/c++**  搜索之后他会自动安装三个插件，等待就行
   2. **C/C++ Extension Pack** 
   3. C/C++ Themes //c/c++ 自带的主题插件，可以不要
   4. **Chinese** //中文插件，英语好的也可以不要
   5. **docker** 
   6. **Dev Containers**



## 进入容器

1. 安装完成之后进入容器

   > 如果这里的箭头不是绿色的就先去终端启动一下容器

   ![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230110_212250.png)

2. 然后他会问你是否进去点击got it 就行![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230110_212436.png)

3. 然后他会进入一个新得vscode 等待这里的进度条跑完![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230110_212512.png)

4. 进度结束之后，点击这里，然后选择打开一个文件夹，选择/apollo_workspace 就行了

   > 这里目录选择/apollo_workspace 就行，选择Planning 会有头文件报错

   ![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230110_213010.png)

5. 点击插件功能 把刚刚在本地安装的模块再在容器里面安装一遍

   > 容器内别的可以不要，c/c++ 的一定要，不然一堆报错

   ![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230110_213217.png)

6. 然后就可以阅读源码了，不要不知道的东西，可以直接按住ctrl + 函数名 就行跳转. 并且也有了语法提示

7. ![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230110_214107.png)
