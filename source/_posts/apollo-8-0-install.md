---
title: apollo-8.0安装教程
categories: apollo
comments: true
description: 我的爱宛如孤岛之花，不为人知的绽放,不为人知的凋零！
cover: https://www.loliapi.com/acg/?id=5
date: 2023-01-08 19:29:44
tags: 
   - 教程
   - apollo
keywords:
---

# apollo 8.0 安装教程

> 本文基于官方软件包方式[安装教程](https://apollo.baidu.com/community/Apollo-Homepage-Document/Apollo_Doc_CN_8_0?doc=%2F%25E5%25AE%2589%25E8%25A3%2585%25E8%25AF%25B4%25E6%2598%258E%2F%25E8%25BD%25AF%25E4%25BB%25B6%25E5%258C%2585%25E5%25AE%2589%25E8%25A3%2585%2F%25E8%25BD%25AF%25E4%25BB%25B6%25E5%258C%2585%25E5%25AE%2589%25E8%25A3%2585)

## 更新系统

```bash
sudo apt-get update
sudo apt-get upgrade
```

## 安装docker

> 如果提示curl 未知 直接终端输入 `sudo apt-get install curl` 就行

```bash
wget http://apollo-pkg-beta.bj.bcebos.com/docker_install.sh
bash docker_install.sh
```

## dokcer 更换国内源

> 需要的更换某些情况下国内源会快一点

1. 终端输入`sudo vim /etc/docker/daemon.json`

2. 按 `i` 进入插入编辑模式

3. 复制下面的内容粘贴进终端去

   > 终端粘贴命令是 ctrl + shift + v

   ```json
   {
     "registry-mirrors": [
       "https://hub-mirror.c.163.com",
       "https://ustc-edu-cn.mirror.aliyuncs.com",
       "https://ghcr.io",
       "https://mirror.baidubce.com"
     ]
   }
   ```

4. 按`ESC` 退出编辑模式 输入 `:wq!` 保存并且退出

5. 终端输入` sudo systemctl restart docker` 重启docker服务

6. 输入`docker info` 查看是否换源成功

7. 最后一段输出以下内容就是成功

   ```bash
    Insecure Registries:
     127.0.0.0/8
    Registry Mirrors:
     https://hub-mirror.c.163.com/
     https://ustc-edu-cn.mirror.aliyuncs.com/
     https://ghcr.io/
     https://mirror.baidubce.com/
    Live Restore Enabled: false
   
   ```

   

## 安装Apollo 包管理工具

1. 添加apt 源

   ```bash
   sudo bash -c "echo 'deb https://apollo-pkg-beta.cdn.bcebos.com/neo/beta bionic main' >> /etc/apt/sources.list"
   wget -O - https://apollo-pkg-beta.cdn.bcebos.com/neo/beta/key/deb.gpg.key | sudo apt-key add -
   sudo apt update
   ```

2. 安装aem 工具

   ```bash
   sudo apt install apollo-neo-env-manager-dev
   ```

   

3. 输入`aem -h` 查看是否安装成功

4. 看到以下输出就是成功

   ```bash
   Usage:
       aem [OPTION]
   
   Options:
       start : start a docker container with apollo development image.
       start_gpu : start a docker container with apollo gpu support development image.
       enter : enter into the apollo development container.
       stop : stop all apollo development container.
       install_core : install the core module of apollo.
       bootstrap : run dreamview and monitor module.
       build : build package in workspace.
       install : install package in workspace.
       init: init single workspace. 
       update: update core modules of apollo.
   ```



## 安装英伟达驱动

> 如果仅仅学习规划和控制就没必要安装英伟达驱动捏,当然你要想看看有无gpu是否有区别，也可以装

1. 安装英伟达驱动

   ```bash
   sudo apt-get update 
   sudo apt-add-repository multiverse 
   sudo apt-get update 
   sudo apt-get install nvidia-driver-455
   ```

2. 安装完了输入`nvidia-smi ` 查看是否安装成功

   ```bash
   Sun Jan  8 19:54:14 2023       
   +-----------------------------------------------------------------------------+
   | NVIDIA-SMI 525.60.11    Driver Version: 525.60.11    CUDA Version: 12.0     |
   |-------------------------------+----------------------+----------------------+
   | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
   | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
   |                               |                      |               MIG M. |
   |===============================+======================+======================|
   |   0  NVIDIA GeForce ...  Off  | 00000000:29:00.0  On |                  N/A |
   | 33%   36C    P0     2W /  38W |    942MiB /  2048MiB |      6%      Default |
   |                               |                      |                  N/A |
   +-------------------------------+----------------------+----------------------+
                                                                                  
   +-----------------------------------------------------------------------------+
   | Processes:                                                                  |
   |  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
   |        ID   ID                                                   Usage      |
   |=============================================================================|
   |    0   N/A  N/A      1057      G   /usr/lib/xorg/Xorg                369MiB |
   |    0   N/A  N/A      1485      G   /usr/bin/kwin_x11                 202MiB |
   |    0   N/A  N/A      1554      G   /usr/bin/plasmashell               43MiB |
   |    0   N/A  N/A      1636      G   /usr/bin/latte-dock                19MiB |
   |    0   N/A  N/A      2618      G   /usr/bin/krunner                    6MiB |
   |    0   N/A  N/A      2987      G   ...tePerProcess --no-sandbox       75MiB |
   |    0   N/A  N/A      3189      G   ...127400283826853805,131072      123MiB |
   |    0   N/A  N/A     41342      G   ...RendererForSitePerProcess       46MiB |
   |    0   N/A  N/A     49537      G   ...AAAAAAAAA= --shared-files       39MiB |
   +-----------------------------------------------------------------------------+
   
   ```

3. 安装docker容器 英伟达支持

   ```bash
   distribution=$(. /etc/os-release;echo $ID$VERSION_ID) 
   curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - 
   curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list 
   sudo apt-get -y update 
   sudo apt-get install -y nvidia-docker2
   ```

4. 终端输入 `sudo systemctl restart docker  ` 重启docker

## 创建 并进去Apollo 容器

1. 克隆官方项目

   > 如果提示git 找不到 就先执行`sudo apt-get install git`

   ```bash
   git clone https://github.com/ApolloAuto/application-demo.git
   cd application-demo
   ```

2. 下载docker镜像

   > 前面没有换源，此处可能会非常慢捏。

   ```bash
   aem start
   ```

3. 最后终端出现如下就OK

   ```bash
   Copying files from `/etc/skel' ...
   [ OK ] Congratulations! You have successfully finished setting up Apollo Dev Environment.
   [ OK ] To login into the newly created apollo_neo_dev_yuri container, please run the following command:
   [ OK ]   aem enter
   [ OK ] Enjoy!
   
   ```

   

4. 输入 `aem enter` 进入工作空间 安装dreamView 和 planning

> 在安装前需要先进入Apollo 工作空间 

1. 如果需要profile 模块，可以看[官方教程](https://apollo.baidu.com/community/article/104) 

   > 看第二个，将场景在本地仿真

2. 安装`dreamview` 

   ```bash
   sudo apt install apollo-neo-dreamview-dev apollo-neo-monitor-dev
   ```

   

3. 启动`dreamview` 然后打开8888端口就可以看到了

   ```bash
   aem bootstrap start
   ```

4. 安装planning 模块 和 预测模块

   ```bash
   buildtool build --packages planning_customization
   buildtool install --legacy predition-
   ```

5. 如果这里报ERROR 就是之前文件拷贝不全，执行下面的语句就行

   ```
   git clone https://github.com/ApolloAuto/application-demo.git
   sudo cp -r application-demo/* /apollo
   ```

   

5. 之后如果修改了源码，直接重新编译一遍planning模块就行





