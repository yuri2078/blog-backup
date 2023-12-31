---
title: apollo安装教程
categories: apollo
comments: true
description: 我的爱宛如孤岛之花，不为人知的绽放，不为人知的凋零！
cover: https://www.loliapi.com/acg/?id=7
date: 2022-9-15 18:22:15.201
tags:
  - apollo
  - 教程

keywords:
---

# 配置 apollo 教程

> 前言 ： 这是一整套的搭建教程，如果仅仅是需要看代码，直接本地克隆代码就行
>
> 如果需要实机使用apollo 工具进行开发，获取调试之类的才需要完整构建
>
> 这个教程是基于虚拟机，如果需要wsl 请看官方教程，以及如果你仅仅想轻量化使用pnc 可以考虑edu 版本

## apollo 官方教程

[edu 版本安装](https://apollo.baidu.com/community/article/82)

[WSL2 win子系统安装](https://apollo.baidu.com/community/article/83)

[官方文档](https://apollo.baidu.com/community/Apollo-Homepage-Document/Apollo_Doc_CN_8_0?doc=%2F%25E5%25AE%2589%25E8%25A3%2585%25E8%25AF%25B4%25E6%2598%258E%2F%25E6%25BA%2590%25E7%25A0%2581%25E5%25AE%2589%25E8%25A3%2585)

## 安装虚拟机

1. 下载虚拟机

   - ` https://www.aliyundrive.com/s/eN7uiuj2HtQ` (阿里云盘)

   - `https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html` (官网链接)

2. 激活虚拟机 `https://zhuanlan.zhihu.com/p/386892372`

## 安装 ubuntu 22.04 

> 官方推荐使用 ubuntu 18.04 这是以 22.04举列子，一样的

1. 下载镜像 `https://releases.ubuntu.com/22.04/ubuntu-22.04.1-desktop-amd64.iso`官网链接

2. 打开 VMware 安装 ubuntu

   1. 新建虚拟机 选择以典型安装 然后下一步
   2. 选择第二个 （以镜像文件安装） 选择刚下载的 ubantu 22.04 镜像 然后下一步
   3. 用户名 随意设置 密码 需要记住 （开机后的 root 密码）然后下一步
   4. 选择一个空的文件夹存放虚拟机
   5. 分配空间至少分配 80G （亲测 50G 加 8G 内存 gcc 编译内存不足）(选择合并成一个文件，多个文件都行)
   6. 内存分配最好 4G 以上，不然容易卡 （我是 8G）核心数和线程数不想卡，就多分点。我分了 3 核 4 线程
   7. 然后下一步等他打开虚拟机
   8. 重新启动等他弹出设置语言界面（中文 Chinese 英文都行）![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_162532.png)
   9. 然后 continue![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_162546.png)
   10. 然后点击 Install Now(后面问你是否确定，直接确定 continue 就行)![20220915_162649](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_162649.png)
   11. 然后让你输入名字 密码（这是登录系统的密码，也就是锁屏密码）输入之后下一步等安装就行了![20220915_162733](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_162733.png)
   12. 等他安装好了之后会问你重启吗？ 立即重启就行了，然后输入你刚刚设置的登录密码登录就行了

3. 开机之后一路右上角 skip down 就行，不同具体设置![20220915_163732](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_163732.png)

## 安装英伟达驱动

> 需要使用感知模块并且有显卡的需要安装一下英伟达显卡驱动  ----- 以下教程来自官方文档

1. 安装必要软件包

   ```bash
   sudo apt-get update
   sudo apt-add-repository multiverse
   sudo apt-get update
   sudo apt-get install nvidia-driver-455
   ```

   

2. 输入 `nvidia-smi`来校验 NVIDIA GPU 驱动是否在正常运行（可能需要在安装后重启系统以使驱动生效）

   ```
   Prompt> nvidia-smi
   Mon Jan 25 15:51:08 2021
   +-----------------------------------------------------------------------------+
   | NVIDIA-SMI 460.27.04    Driver Version: 460.27.04    CUDA Version: 11.2     |
   |-------------------------------+----------------------+----------------------+
   | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
   | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
   |                               |                      |               MIG M. |
   |===============================+======================+======================|
   |   0  GeForce RTX 3090    On   | 00000000:65:00.0  On |                  N/A |
   | 32%   29C    P8    18W / 350W |    682MiB / 24234MiB |      7%      Default |
   |                               |                      |                  N/A |
   +-------------------------------+----------------------+----------------------+
   
   +-----------------------------------------------------------------------------+
   | Processes:                                                                  |
   |  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
   |        ID   ID                                                   Usage      |
   |=============================================================================|
   |    0   N/A  N/A      1286      G   /usr/lib/xorg/Xorg                 40MiB |
   |    0   N/A  N/A      1517      G   /usr/bin/gnome-shell              120MiB |
   |    0   N/A  N/A      1899      G   /usr/lib/xorg/Xorg                342MiB |
   |    0   N/A  N/A      2037      G   /usr/bin/gnome-shell               69MiB |
   |    0   N/A  N/A      4148      G   ...gAAAAAAAAA --shared-files      105MiB |
   +-----------------------------------------------------------------------------+
   ```

   

3. 为了在容器内获得 GPU 支持，在安装完 docker 后需要安装 NVIDIA Container Toolkit。 运行以下命令安装 NVIDIA Container Toolkit：

   

   ```bash
   distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
   curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
   curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
   sudo apt-get -y update
   sudo apt-get install -y nvidia-docker2
   ```

   安装完成后，重启 Docker 以使改动生效。

   ```bash
   sudo systemctl restart docker
   ```

   安装完毕后，可以在**APOLLO容器内**输入`nvidia-smi`来校验 NVIDIA GPU 在容器内是否能正常运行（详见1）。



## 开始配置 apollo

### 准备秘钥

> 如果能够使用github 版本就不用搞这个直接 git clone https://github.com/ApolloAuto/apollo.git 就行，如果提示没有git 就执行 sudo apt install git就行
>
> 如果看不懂上面在说什么就直接无视这句话就行

1. 注册 gitte `https://gitee.com/`
2. 点击右上角加号 -> 新建仓库
3. 仓库的名字和路径随便填 然后点击创建![20220915_193426](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_193426.png)
4. 进入仓库，点击顶部一排选项右边的 ‘’管理‘’
5. 点击部署公钥管理 -> 添加公钥 名字随便，内容填下面输出的内容
6. 打开终端输入 `ssh-keygen -t ed25519 -C "xxxxx@xxxxx.com" ` 然后直接回车 3 次就行
7. 输入`cat ~/.ssh/id_ed25519.pub` 然后复制输出的 ssh -ed ...... 添加到公钥
8. 填写完成之后终端 输入`ssh -T git@gitee.com`按提示回复 yes 然后看到他输出 Hi **Anonymous**就行了

### 更换国内源

1. 打开终端输入 `sudo vim /etc/docker/daemon.json`

2. 然后按 `i` 进入插入模式

3. 复制下面的语句`ctrl shift + v` 复制进去

   ```json
   {
       "registry-mirrors": [
           "http://hub-mirror.c.163.com",
           "https://docker.mirrors.ustc.edu.cn",
           "https://registry.docker-cn.com"
       ]
   }
   ```

4. 按`ESC`键后输入 ` :wq!`保存并退出

5. 输入`service docker restart`重新加载docker

6. 输入 `docker info` 末尾出现类似的就是成功了

   ```bash
    Insecure Registries:
     127.0.0.0/8
    Registry Mirrors:
     http://hub-mirror.c.163.com/
     https://docker.mirrors.ustc.edu.cn/
     https://registry.docker-cn.com/
    Live Restore Enabled: false
   
   ```

### 开始跑命令

1. 浏览器打开`https://wwho.lanzoue.com/ioHPm0jsee5g`下载 docker.txt

2. 下载完成后 ctrl + alt + t 打开终端

3. cd Download 目录

4. 输入 ls 查看是否有一个 docker.txt

5. 输入 bash docker.txt 开始跑命令

6. 然后他会提示你输入 root 密码， 注意这里输入密码不会有提示，直接输入之后回车就行，看不到输入内容

7. 然后安心等待他跑完就行。大概需要好几个小时，因为需要下载很多东西

   ```shell
   λ ~/ master* cd Downloads #进入下载目录
   λ ~/Downloads/ master* ls #查看当前目录下的所有文件
   docker.sh
   λ ~/Downloads/ master* bash docker.sh
   [sudo] password for yuri: #这里输入看不见的

   ```

8. 看到这样的输出就是成功![20220915_192648](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_192648.png)

### 开始 build apollo

1. 扩展虚拟内存 （防止 gcc 编译报错）

2. 终端照着输入就行 (需要输入 root 密码)

   ```bash
   sudo swapoff /swapfile
   sudo rm /swapfile
   sudo dd if=/dev/zero of=/swapfile bs=1M count=16240
   sudo mkswap /swapfile
   sudo swapon /swapfile
   free -m #查看虚拟内存是不是挂载上了,看第二行 第一列 是不是16000多就行了
   ```

3. 打开终端输入

   ```bash
   cd apollo
   bash docker/scripts/dev_into.sh
   sudo bash apollo.sh build
   
   # 之用输入上面三个命令就行
   
   # 明确自己需要的东西，如果只要cyber 最后一句就是 
   sudo bash apollo.sh build cyber # 只要cyber
   sudo bash apollo.sh build dreamview # 只要dreamview
   sudo bash apollo.sh build planning # 只要预测模块
   sudo bash apollo.sh build_dbg # 以dbug 模式构建
   #开始build 好的配置大概build 半小时， 差的四五个小时
   #不要让屏幕熄灭，ubantu默认5分钟锁屏，可以去设置- 电源 里面改成永不熄灭，还有自己的电脑最好也搞一下
   ```

4. 等出现绿色的 OK 和 Enjoy 则表示成功 如图![20220915_211913](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_211913.png)

5. 测试回放 demo

   `bash scripts/bootstrap.sh start` 然后会提示你在 http://localhost:8888 中访问他

6. 打开浏览器输入刚刚的网址出现这个界面就是成功![20220915_212020](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_212020.png)

7. 下载测试 demo 下载链接：`https://link.zhihu.com/?target=https%3A//blog.shipengx.com/download/demo_3.5.record`

8. 下载完成执行 `mv ~/Downloads/demo_3.5.record ~/apollo/docs/demo_guide `命令把他复制进测试文件夹

9. 进入 demo_guide 文件夹 `cd ~/apollo/docs/demo_guide`

10. 执行 `cyber_recorder play -f demo_3.5.record -l`然后打开浏览器就能看到该 demo![20220915_213125](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo//20220915_213125.png)
