---
title: 障碍物绕行仿真调试
categories: apollo
comments: true
description: 我的爱宛如孤岛之花，不为人知的绽放,不为人知的凋零！
cover: https://www.loliapi.com/acg/?id=20
date: 2023-01-07 18:38:58
tags: apollo
keywords:
---

# 障碍物绕行仿真调试

> 通过本次实验了解仿真练习

## 添加仿真场景

1. 打开[控制台](https://apollo.baidu.com/workspace/) 进行仿真实验免费包申请
2. 选择 左侧仿真 ->  场景编辑 -> 添加需要的场景
3. 如果不会 请参考[官方教程](https://apollo.baidu.com/community/article/104)
4. 之后在**个人场景集** 中添加你需要的场景



## Dreamview 仿真

> 云端实验室启动直接 ./scripts/bootstrap_neo.sh 就行

1. ` sudo systemctl start docker` 启动docker服务
2. `docker start apollo_dev_用户名` 启动apollo容器
3. `bash docker/scripts/dev_into.sh` 在当前终端中进入apollo docker容器
4. `bash scripts/bootstrap.sh` 启动dreamview 
5. 访问 `http://localhost:8888` 查看dreamview界面

## 进行障碍物绕行实验

1. 打开 `SIm Control ` 选项

   ![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230107_185637.png)

2.  左侧 Profile 模块中下载自己的场景![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230107_191607.png)

3. 在模块功能中 打开 预测 规划 路径 这三个模块![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230107_191752.png)

4. 在路由中 选择路径 ，之后点击发送路径就行![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230107_191829.png)

5. 查看效果

6. 更改conf文件，修改横向超车距离

7. 文件位置`/apollo/modules/planning/conf/planning.conf `

8. 修改这一行数据为1.5 就可以吧横向超车距离改为1.5 米![](https://cdn.jsdelivr.net/gh/yuri2078/images/apollo/Screenshot_20230107_194041.png)



