---
title: vscode-apollo-debug
categories: apollo
comments: true
description: 我的爱宛如孤岛之花，不为人知的绽放,不为人知的凋零！
cover: https://www.loliapi.com/acg/?id=30
date: 2023-01-13 19:33:33
tags: 
  - apollo
  - vscode
keywords:
---

# 更高效的阅读源码

## 使用vscode 单步调试 

### 重新编译planning

1. `aem enter` 进入工作空间
2. `buildtool build --dbg --packages planning_customization` 以dbg模式重新编译planning

### 开始单步调试

1. 先使用vscode ssh 连接docker 容器![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230113_194349.png)

2. 新建 launch 文件

   1. 打开apollo 的 `/apollo_workspace` 工作空间

   2. 在vscode 中新建一个文件夹名为`.vscode`

   3. 在`.vscode` 中新建一个`launch.json` 文件![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230113_194733.png)

   4. 文件内容如下

      > 基于apollo aem 安装版本，源码编译的请自己修改相应配置

      ```json
      {
          "version": "0.2.0",
          "configurations": [
              {
                  "name": "apollo-dbg",
                  "type": "cppdbg",
                  "request": "launch",
                  "program": "/opt/apollo/neo/bin/mainboard", // 源码编译的自行更改为对应mainboard 位置
                	// 如果需要调试别的模块，请改成对应的dag 文件就行
                  "args": ["-d","/apollo_workspace/modules/planning/dag/planning.dag"], // planning dag 位置
                  "stopAtEntry": false,
                  "cwd": "/apollo_workspace", // 源码编译的请改为 /apollo
                  "environment": [],
                  "externalConsole": false,
                  "MIMode": "gdb",
                  "setupCommands": [
                      {
                          "description": "为 gdb 启用整齐打印",
                          "text": "-enable-pretty-printing",
                          "ignoreFailures": true
                      }
                  ],
                  "miDebuggerPath": "/usr/bin/gdb"
              }
          ]
      }
      ```

3. 在需要调试的位置打上断点

   > 在没有开启Routing 模块之前，请不要把断电打在 `planning_base_->Init(config_);` 这条语句之前
   >
   > 不然会每隔 50ms AERROR 一次 Routing is not ready.
   >
   > 因为在这个Init 函数中，**Start** 函数 参考线提供器,会另启动一个线程,执行一个定时任务,每隔50ms提供一次参考线

   ![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230113_195519.png)

4. 点击这里开始调试![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230113_200000.png)

5. 等一会，然后等他停在你打的断点这里就成功了

   > 到这里你就可以开始调试了，你可以使用调试菜单就行单步调试了

   ![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230113_200307.png)

## 灵活使用调试菜单

![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230113_200555.png)

### 变量区域

在图上 1 号区域，可以看到此时程序运行临时变量 全局变量的值，需要看直接去看就行

### 调试按钮

> 每个序号对应2 号区域每个按钮

1. 继续 --- 他会直接继续运行程序，知道遇到下一个断点，或程序结束
2. 单步跳过 --- 单步跳过代码，遇到函数会直接跳过，并不会进入函数
3. 单步进入 --- 单步进入代码，如果是普通语句会直接跳过，如果是函数之类的会直接跳到函数内部
4. 单步退出 --- 单步退出代码，效果就是单步退出单步进入的代码
5. 重新开始 --- 重新开始调试
6. 结束调试 --- 结束本次调试



## 使用AINFO 打印日志

> 因为 apollo 使用的是google 的glog 所以，如果有别的需求的也可以去看看glog 的使用方法 

### 使用方法

> 我直介绍最简单的方法，知道这个也够了。如果需要别的方法，也可以去找glog 使用方法，然后自己进行修改

#### AINFO 打印日志

```c++
if (FLAGS_use_navigation_mode) {
    planning_base_ = std::make_unique<NaviPlanning>(injector_); //相对地图规划器
  } else {
    planning_base_ = std::make_unique<OnLanePlanning>(injector_); //默认规划器
  }
```

举例子，当我想知道开始这个`planning_base_` 究竟是哪个的时候，我们直接在这段代码后面打印 他的name 就行,apollo 大部分的类都提供了name 接口，直接打印就行

```c++
AINFO << "本次选择的规划器是 -> " << planning_base_->Name();
```

所以直接 `AINFO << “你想要而内容"` 就行。他会默认换行，但是如果很多 ` <<` 他不会每个都换行，而是同一个AINFO 的内容在一行



### 日志

apollo 的 日志 都在`/opt/apollo/neo/data/log/` 下面。如果是源码版本则是在`/apollo/data/log/` 下

每个 模块名.INFO (例如 : planning.INFO ) 都是对应模块最新一次的日志，而过去的日志回以日期命名存放在目录下面



## 快速阅览日志

> 在工作目录建立硬连接，这样就可以在工作目录直接看到日志了
>
> 如果需要别的日志的直接吧planning 改成对应模块的名字就行了

```bash
ln -s /opt/apollo/neo/data/log/planning.INFO /apollo_workspace/ 
```

