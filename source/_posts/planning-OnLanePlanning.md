---
title: planning_OnLanePlanning（2）
categories: apollo-源码分析
comments: true
description: 我的爱宛如孤岛之花，不为人知的绽放,不为人知的凋零！
cover: https://www.loliapi.com/acg/?id=23
date: 2023-01-11 19:24:12
tags: apollo
keywords:
---

# OnLanePlanning 源码分析

> 因为入口Init 中，默认选择的规划器是 `OnLanePlanning` 所以我们主要介绍`OnLanePlanning` 

## planning_base 

> 这里的planning_base 不是真的planning_base 基类，而是代码中对 规划器的变量名

```c++
bool PlanningComponent::Init() {
  AINFO << "planning component init 函数调用!";
  injector_ = std::make_shared<DependencyInjector>();
  if (FLAGS_use_navigation_mode) {
    planning_base_ = std::make_unique<NaviPlanning>(injector_); //相对地图规划器
  } else {
    planning_base_ = std::make_unique<OnLanePlanning>(injector_); //默认规划器
  }
  AINFO << "本次选择的规划器是 -> " << planning_base_->Name();
		。。。。。 中间省略一些代码 。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。
  AINFO << "正在执行planning_base 的初始化 ";
  planning_base_->Init(config_); // 我们看到init 函数，他直接调用了对应规划器的init 函数
```

在上文中提到，在init 函数中，会设置一个默认规划器初始规划其是 OnLanePlanning

`planning_base_ = std::make_unique<OnLanePlanning>(injector_);`

而在init函数的中间他会率先执行 planning_base_ 的 init函数也就是 OnLanePlanning 的init函数

`planning_base_ -> Init(config_);`

## plannerDispatcher

> 在`OnLanePlanning` 的构造函数中指定了 `OnLanePlannerDispatcher` (第5行)

我们发现他的构造函数只有一条语句 也就是说，一开始OnLanePlanning 选择的plannerDispatcher 就是OnLanePlannerDispatcher (这不废话？不选自己对应的难道选别人的？)

```c++
class OnLanePlanning : public PlanningBase {
 public:
  explicit OnLanePlanning(const std::shared_ptr<DependencyInjector>& injector) : PlanningBase(injector) {
    AINFO << "onlane planning 类构造函数 新建合适的planner";
    planner_dispatcher_ = std::make_unique<OnLanePlannerDispatcher>();
  }
```

所以，后续的 planner_dispathcer 都是指向的 `OnLanePlannerDispatcher` 类

## Init 函数

> 在`OnLanePlanner` 的init 函数中指定了 planner 为 `PUBLIC_ROAD` (默认planner) (第44行)

```c++
Status OnLanePlanning::Init(const PlanningConfig& config) {
  AINFO << "on_lane_planning 初始化 ";
  config_ = config;
  // 检查config文件，这里并不完全，也可以添加其他检查项
  if (!CheckPlanningConfig(config_)) {
    return Status(ErrorCode::PLANNING_ERROR, "配置文件错误捏! " + config_.DebugString());
  }
  
  // TaskFactory  工厂类在 planningBase init 中初始化
  PlanningBase::Init(config_); // 基础规划其初始化

  // planner_dispatcher 在OnLanePlannner 构造函数中创建
  // planner_factory 路径规划工厂类注册，包含rtk publick_road lattice等

  planner_dispatcher_->Init();

// 交通配置文集 modules/planning/conf/traffic_rule_config.pb.txt
  ACHECK(apollo::cyber::common::GetProtoFromFile(
      FLAGS_traffic_rule_config_filename, &traffic_rule_configs_))
      << "加载交通配置文件失败!"
      << FLAGS_traffic_rule_config_filename;

  // clear planning history
  // 清除规划历史
  injector_->history()->Clear();

  // clear planning status
  // 清除规划状态
  injector_->planning_context()->mutable_planning_status()->Clear();

  // load map
  // 加载地图
  hdmap_ = HDMapUtil::BaseMapPtr();
  ACHECK(hdmap_) << "加载地图失败！";

  // instantiate reference line provider
  AINFO << "启动参考线提供器,会另启动一个线程,执行一个定时任务,每隔50ms提供一次参考线";
  reference_line_provider_ = std::make_unique<ReferenceLineProvider>(injector_->vehicle_state(), hdmap_); 
  reference_line_provider_->Start(); // 生成参考线

  // dispatch planner
  // 创建planner object 这里的配置文集那是 PUBLIC_ROAD planner
  // 为Planning分配具体的Planner
  planner_ = planner_dispatcher_->DispatchPlanner(config_, injector_);

  if (!planner_) {
    AINFO << "planner 为nullptr 分配失败!";
    return Status(ErrorCode::PLANNING_ERROR, "planning is not initialized with config : " + config_.DebugString());
  }else {
    AINFO << "分配planner -> "  << planner_->Name(); 
  }

  if (config_.learning_mode() != PlanningConfig::NO_LEARNING) {
    PlanningSemanticMapConfig renderer_config;
    ACHECK(apollo::cyber::common::GetProtoFromFile(
        FLAGS_planning_birdview_img_feature_renderer_config_file,
        &renderer_config))
        << "Failed to load renderer config"
        << FLAGS_planning_birdview_img_feature_renderer_config_file;

    BirdviewImgFeatureRenderer::Instance()->Init(renderer_config);
  }

  start_time_ = Clock::NowInSeconds();

  return planner_->Init(config_); // 返回对planner 进行初始化的结果
}
```



## RunOnce 函数

> 以规划模式OnLanePlanning，执行RunOnce。在RunOnce中**先执行交通规则**，再规划轨迹。规划轨迹的函数是 `Plan`。

## 简单总结：

> 不知道planning大概由哪些组成可以看我之前的文章 [planning 入口](https://yuri2078.github.io/2023/01/11/planning-component/)

1. 我把整个planning 模块的结构分为四层，除了第一层`component` ,其他几个都是在`planning_compoent` 的init 函数中被确认的
   1. `OnLanePlanner` 的构造函数中指定了第二层的 `OnLanePlannerDispatcher  `(planning_base)
   2. `OnLanePlanner` 的`Init`函数中指定了第三层的 `OnLanePlannerDispatcher` (planerDispatcher)
   3. `OnLanePlanner` 的`Init`函数中指定了第四层的 `PUBLIC_ROAD `(planner)
2. 在`init` 函数中他会读取文件，清除历史记录，并且分配planner。
3. 在`Init` 函数中，他会启动另一个线程开始生成参考线每50ms 生成一次，此时需要`routing` 输入
4. `Planner` 初始化 会生成一个管理场景的对象，用来管理场景。
5. `RunOnce` 在planning_compoent 中被不断调用。



