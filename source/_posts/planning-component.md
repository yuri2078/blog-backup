---
title: planning_component（1）
categories: apollo-源码分析
comments: true
description: 我的爱宛如孤岛之花，不为人知的绽放,不为人知的凋零！
cover: https://www.loliapi.com/acg/?id=21
date: 2023-01-11 13:35:05
tags: apollo
keywords:
feature: true

---

# apollo planning 模块分析

> 只做简单分析，教程来自 知乎 -> [自动驾驶Player](https://www.zhihu.com/people/ge-ge-yao-xue-xi)
>
> 更加详细的内容请去看 [原文](https://zhuanlan.zhihu.com/p/494813220?utm_id=0)

## 前言

> 这里是我的个人理解不是官方理解，也不是原文的理解

### planning 结构

1. Planning_component 入口 (作为cyber 组件的入口，我把它当作第一层)

   > 这个作为planning 模块的入口主要的作用就是两点
   >
   > 1、对一些基础的东西进行初始化 (init 函数)
   >
   > 2、作为cyber 组件 不断执行规划逻辑 (Proc 函数)

2. 三个规划器 （默认的是 `OnLanePlanning`）

   > 我认为这是`planning` 的第二层。 在 `component` 中选择规划器进行下一步的规划

   1. `NavinPlanning` 处理普通道路  
   2. `OnLanePlanning` 处理情况复杂的人行横道  主要的应用场景是开放道路的自动驾驶
   3. `OpenSpasePlanning` 处理泊车或者断路掉头情况   主要的应用场景是自主泊车和狭窄路段的掉头

3. 两个路径规划器

   > 每个规划器会调用自己对应的路径规划器，然后来进行路径规划。
   >
   > 他在`OnLanePlanning` 的构造函数中就被构造，然后初始化
   >
   > 他的作用就是选择创建对应的 planner 

   1. `on_lane_planner_dispatcher` (OpenSpasePlanning 、 OnLanePlanning 都是用的这个)
   2. `navi_planner_dispatcher` (NavinPlanning 用的这个)

4. 几个 planner 

   > 每个planner 承担了具体的规划逻辑(Plan 函数)

   主要介绍 `on_lane_planner_dispatcher` 下默认的 `public_road_planner` 

### planning 规划流程

1. 在入口`planning_component` 中进行初始化(Init 函数只执行一次)
   1. 选择对应的规划器，并进行初始化
   2. 加载配置，检查配置文件
   3. 读取一系列的配置数据
2. 在入口`planning_component` 中执行具体的规划逻辑(Proc 函数，不断执行)
   1. 检查路由
   2. 执行具体规划逻辑
   3. 发布新路线

## cyber 组件

**每个组件的入口函数是 Proc 函数**

1. apollo 是基于 cyber 通信框架的

2. 具体可以去看我的另外一篇文章 [cyberRT开发教程](https://yuri2078.github.io/2022/09/17/cyber/)

3. 下面是一个最基础的cyber 组件框架 （.h 文件） planning 的差不多也是如此

   ```c++
   #ifndef MY_COMPONENTS
   #define MY_COMPONENTS
   
   #include "cyber/cyber.h" // cyber 基础头文件
   #include "cyber/component/component.h" // component 基础头文件
   #include "cyber/demo_cpp/components/Student.pb.h" // 通信载体protobuf 文件编译成的c++ 文件
   
   using apollo::cyber::Component;
   using apollo::cyber::demo_cpp::Student;
   
   //继承需要添加数据模板
   class my_components : public Component<Student> 
   {
   private:
       /* data */
   public:
   
       bool Init() override; //表示重写函数，初始化函数
       bool Proc(const std::shared_ptr<Student> &stu) override; //数据处理函数
       
   };
   
   CYBER_REGISTER_COMPONENT(my_components) //注册组件
   #endif
   ```

### 简单cyber 组件的组成

#### Conponent 类

**知识点**

1. 他继承自`Component` 类，他是所有通信模块的基类

2. 他的原型  他是继承自`ConponentBase` 这个我就不管了，这是隔壁cyber的东西

   ```c++
   template <typename M0, typename M1, typename M2,  typename M3> // 接受四个模板类型 也就是消息载体 - protobuf
   class Component<M0, M1, M2, M3> : public ComponentBase { 
   ```

3. `Component<M0, M1, M2, M3>` 由此看出他最多接收4个数据类型。

   > 他需要的都是通信载体，也就是protobuf 类信息，如果不知道这是啥，可以百度捏。简单来说就是自定义数据类

#### Init 函数

>  继承自`Component_base ` 纯虚类，所以需要重写

函数原型 `virtual bool Init() = 0;`

**知识点**

1. Init 函数来自于Conponent_base 类 
2. 他不接受任何参数，并且会在执行Proc 函数之前被调用
3. 用途是 初始化

#### Proc 函数

> 继承自 `Conponent` 纯虚类，所以需要重写

函数原型  - planning 模块对应的函数原型

```c++
virtual bool Proc(const std::shared_ptr<M0>& msg,
                  const std::shared_ptr<M1>& msg1,
                  const std::shared_ptr<M2>& msg2, 
                  const std::shared_ptr<M2>& msg3) = 0;
```

知识点 

1. 他最多只能接受四个msg
2. 我们需要在Proc 函数中完成对传入的消息载体的处理
3. 这里是处理消息的主要地方

#### 注册组件

`CYBER_REGISTER_COMPONENT(my_components)` //对组件进行注册

### 别的模块

> 通过上面对简单组件的了解我门可以知道

1. apollo 的模块也是一个cyber组件
2. 每个模块通过接受消息，然后处理消息，最后将消息发送出去（通过cyber 通信框架）
3. cyber 采用的通信协议是谷歌的protobuf 所以`Component` 类需要接受的四个消息都是由protobuf编译来的
4. 一个组件的入口函数是proc函数，但之前他会执行init 函数就行初始化
5. Proc 函数才是组件的主体处理逻辑



## 常用类

| 名称                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| `EgoInfo`类           | 包含了自车信息，例如：当前位置点，车辆状态，外围Box等。      |
| `Frame`类             | 包含了一次Planning计算循环中的所有信息。                     |
| `FrameManager`类      | Frame的管理器，每个Frame会有一个整数型id。                   |
| `LocalView`类         | Planning计算需要的输入，下文将看到其定义。                   |
| `Obstacle`类          | 描述一个特定的障碍物。障碍物会有一个唯一的id来区分。         |
| `PlanningContext`类   | Planning全局相关的信息，例如：是否正在变道。这是一个[单例](https://en.wikipedia.org/wiki/Singleton_pattern)。 |
| `ReferenceLineInfo`类 | 车辆行驶的参考线，下文会专门讲解。                           |
| `path`文件夹          | 描述车辆路线信息。包含：PathData，DiscretizedPath，FrenetFramePath三个类。 |
| `speed`文件夹         | 描述车辆速度信息。包含SpeedData，STPoint，StBoundary三个类。 |
| `trajectory`文件夹    | 描述车辆轨迹信息。包含DiscretizedTrajectory，PublishableTrajectory，TrajectoryStitcher三个类。 |
| `planning_gflags.h`   | 定义了模块需要的许多常量，例如各个配置文件的路径。           |

## planning 模块入口

> 有了上面的基础相信你已经可以很好的阅读planning的基础源码了，让我们一起来看看planning的cyber组件吧
>
> 本文的源码基于 apollo-8.0 edu 版本
>
> 两个源码重要的地方都有注释

### planning_component.h

```c++
namespace apollo {
namespace planning {

class PlanningComponent final // final
    : public cyber::Component<prediction::PredictionObstacles, canbus::Chassis,
                              localization::LocalizationEstimate> {
 public:
  PlanningComponent() = default; //默认构造函数

  ~PlanningComponent() = default; //默认析构函数

 public:
  bool Init() override;

  bool Proc(const std::shared_ptr<prediction::PredictionObstacles>& rediction_obstacles, // 预测的障碍物信息(prediction_obstacles)
            const std::shared_ptr<canbus::Chassis>& chassis, // 车辆底盘(chassis)信息(车辆的速度，加速度，航向角等信息)
            const std::shared_ptr<localization::LocalizationEstimate>& localization_estimate)  // 车辆当前位置(localization_estimate)
            override; // 表示重写函数

 private:
  void CheckRerouting();
  bool CheckInput();

 private:
  std::shared_ptr<cyber::Reader<perception::TrafficLightDetection>> traffic_light_reader_; // 读取红绿灯信息
  std::shared_ptr<cyber::Reader<routing::RoutingResponse>> routing_reader_; // 读取路由模块信息
  std::shared_ptr<cyber::Reader<planning::PadMessage>> pad_msg_reader_;
  std::shared_ptr<cyber::Reader<relative_map::MapMsg>> relative_map_reader_; // 读取相对地图信息
  std::shared_ptr<cyber::Reader<storytelling::Stories>> story_telling_reader_;

  std::shared_ptr<cyber::Writer<ADCTrajectory>> planning_writer_; // 将规划好的线路，发布到Control模块订阅的Topic中
  std::shared_ptr<cyber::Writer<routing::RoutingRequest>> rerouting_writer_; // 是否需要重新规划路线
  std::shared_ptr<cyber::Writer<PlanningLearningData>> planning_learning_data_writer_;

  std::mutex mutex_;
  perception::TrafficLightDetection traffic_light_;
  routing::RoutingResponse routing_;
  planning::PadMessage pad_msg_;
  relative_map::MapMsg relative_map_;
  storytelling::Stories stories_;

  LocalView local_view_;   // 用于存放模块运行需要的数据

  std::unique_ptr<PlanningBase> planning_base_; // 规划器基类
  std::shared_ptr<DependencyInjector> injector_;

  PlanningConfig config_;
  MessageProcess message_process_;
};

// 在cyber中注册组件
CYBER_REGISTER_COMPONENT(PlanningComponent)

}  // namespace planning
}  // namespace apollo
```

### planning_component.cc

```c++
namespace apollo {
namespace planning {

using apollo::cyber::ComponentBase;
using apollo::hdmap::HDMapUtil;
using apollo::perception::TrafficLightDetection;
using apollo::relative_map::MapMsg;
using apollo::routing::RoutingRequest;
using apollo::routing::RoutingResponse;
using apollo::storytelling::Stories;

//初始化函数，用于初始化planning模块
bool PlanningComponent::Init() {
  AINFO << "planning component init 函数调用!";
  injector_ = std::make_shared<DependencyInjector>();

  //NavinPlanning 处理普通道路  
  //OnLanePlanning 处理情况复杂的人行横道  主要的应用场景是开放道路的自动驾驶。
  //OpenSpasePlanning 处理泊车或者断路掉头情况   主要的应用场景是自主泊车和狭窄路段的掉头


//选择相应的规划器 默认是OnLanePlanning 模块
  AINFO << "选择相应的规划器 默认是OnLanePlanning 模块 ";
  if (FLAGS_use_navigation_mode) {
    planning_base_ = std::make_unique<NaviPlanning>(injector_); //相对地图规划器
    AINFO << "默认选择的是 : naviplanning";
  } else {
    planning_base_ = std::make_unique<OnLanePlanning>(injector_); //默认规划器
    AINFO << "默认选择的是 : onlaneplanning";
  }

//加载config 文件
  AINFO << "正在加载配置文件";
  ACHECK(ComponentBase::GetProtoConfig(&config_))
      << "加载配置文件失败! 失败文件路径 ： "
      << ComponentBase::ConfigFilePath();

// 这一句不知道干啥的
  if (FLAGS_planning_offline_learning ||
      config_.learning_mode() != PlanningConfig::NO_LEARNING) {
    if (!message_process_.Init(config_, injector_)) {
      AERROR << "failed to init MessageProcess";
      return false;
    }
  }

//执行OnLanePlanning 的初始化
  AINFO << "正在执行planning_base 的初始化 ";
  planning_base_->Init(config_);

  //获取路由（routig）模块信息
  AINFO << "获取路由 routing信息";
  routing_reader_ = node_->CreateReader<RoutingResponse>(  //新建reader
      config_.topic_config().routing_response_topic(),
      [this](const std::shared_ptr<RoutingResponse>& routing) {
        AINFO << " 收到路由数据：运行路由回调。"
              << routing->header().DebugString();
        std::lock_guard<std::mutex> lock(mutex_);
        routing_.CopyFrom(*routing);
      });

  //读取红绿灯
  AINFO << "读取红绿灯信息";
  traffic_light_reader_ = node_->CreateReader<TrafficLightDetection>(
      config_.topic_config().traffic_light_detection_topic(),
      [this](const std::shared_ptr<TrafficLightDetection>& traffic_light) {
        ADEBUG << "Received traffic light data: run traffic light callback.";
        std::lock_guard<std::mutex> lock(mutex_);
        traffic_light_.CopyFrom(*traffic_light);
      });

  pad_msg_reader_ = node_->CreateReader<PadMessage>(
      config_.topic_config().planning_pad_topic(),
      [this](const std::shared_ptr<PadMessage>& pad_msg) {
        ADEBUG << "Received pad data: run pad callback.";
        std::lock_guard<std::mutex> lock(mutex_);
        pad_msg_.CopyFrom(*pad_msg);
      });

  story_telling_reader_ = node_->CreateReader<Stories>(
      config_.topic_config().story_telling_topic(),
      [this](const std::shared_ptr<Stories>& stories) {
        ADEBUG << "Received story_telling data: run story_telling callback.";
        std::lock_guard<std::mutex> lock(mutex_);
        stories_.CopyFrom(*stories);
      });

//是否使用导航模式
  AINFO << "判断是否使用导航模式";
  if (FLAGS_use_navigation_mode) {
    //读取相对地图
    AINFO << "使用导航模式，正在读取相对地图";
    relative_map_reader_ = node_->CreateReader<MapMsg>(
        config_.topic_config().relative_map_topic(),
        [this](const std::shared_ptr<MapMsg>& map_message) {
          ADEBUG << "Received relative map data: run relative map callback.";
          std::lock_guard<std::mutex> lock(mutex_);
          relative_map_.CopyFrom(*map_message);
        });
  }
  //发布规划好的线路
  planning_writer_ = node_->CreateWriter<ADCTrajectory>(
      config_.topic_config().planning_trajectory_topic());

  //发布重新规划请求
  rerouting_writer_ = node_->CreateWriter<RoutingRequest>(
      config_.topic_config().routing_request_topic());

  planning_learning_data_writer_ = node_->CreateWriter<PlanningLearningData>(
      config_.topic_config().planning_learning_data_topic());

  return true;
}

bool PlanningComponent::Proc(
    const std::shared_ptr<prediction::PredictionObstacles>& prediction_obstacles, // 预测的障碍物信息(prediction_obstacles)
    const std::shared_ptr<canbus::Chassis>& chassis, // 车辆底盘(chassis)信息(车辆的速度，加速度，航向角等信息)
    const std::shared_ptr<localization::LocalizationEstimate>& localization_estimate) // 车辆当前位置(localization_estimate) 
{
  AINFO << "planning—component Proc 函数调用!";
  ACHECK(prediction_obstacles != nullptr);  // 检查障碍物信息是不是为空

  // check and process possible rerouting request
  // 检查是否需要重新规划路线
  CheckRerouting();

  // process fused input data
  // 数据放入local_view 中，并且检查输入的数据
  local_view_.prediction_obstacles = prediction_obstacles;
  local_view_.chassis = chassis;
  local_view_.localization_estimate = localization_estimate;
  {
    std::lock_guard<std::mutex> lock(mutex_);
    if (!local_view_.routing ||
        hdmap::PncMap::IsNewRouting(*local_view_.routing, routing_)) {
      local_view_.routing =
          std::make_shared<routing::RoutingResponse>(routing_);
    }
  }
  {
    std::lock_guard<std::mutex> lock(mutex_);
    local_view_.traffic_light =
        std::make_shared<TrafficLightDetection>(traffic_light_);
    local_view_.relative_map = std::make_shared<MapMsg>(relative_map_);
  }
  {
    std::lock_guard<std::mutex> lock(mutex_);
    local_view_.pad_msg = std::make_shared<PadMessage>(pad_msg_);
  }
  {
    std::lock_guard<std::mutex> lock(mutex_);
    local_view_.stories = std::make_shared<Stories>(stories_);
  }

  if (!CheckInput()) {
    AERROR << "Input check failed";
    return false;
  }

  if (config_.learning_mode() != PlanningConfig::NO_LEARNING) {
    // data process for online training
    message_process_.OnChassis(*local_view_.chassis);
    message_process_.OnPrediction(*local_view_.prediction_obstacles);
    message_process_.OnRoutingResponse(*local_view_.routing);
    message_process_.OnStoryTelling(*local_view_.stories);
    message_process_.OnTrafficLightDetection(*local_view_.traffic_light);
    message_process_.OnLocalization(*local_view_.localization_estimate);
  }

  // publish learning data frame for RL test
  if (config_.learning_mode() == PlanningConfig::RL_TEST) {
    PlanningLearningData planning_learning_data;
    LearningDataFrame* learning_data_frame =
        injector_->learning_based_data()->GetLatestLearningDataFrame();
    if (learning_data_frame) {
      planning_learning_data.mutable_learning_data_frame()
                            ->CopyFrom(*learning_data_frame);
      common::util::FillHeader(node_->Name(), &planning_learning_data);
      planning_learning_data_writer_->Write(planning_learning_data);
    } else {
      AERROR << "fail to generate learning data frame";
      return false;
    }
    return true;
  }

  ADCTrajectory adc_trajectory_pb;

  // 执行注册好的planning信息，并且生成线路
  planning_base_->RunOnce(local_view_, &adc_trajectory_pb); 
  auto start_time = adc_trajectory_pb.header().timestamp_sec();
  common::util::FillHeader(node_->Name(), &adc_trajectory_pb);

  // modify trajectory relative time due to the timestamp change in header
  const double dt = start_time - adc_trajectory_pb.header().timestamp_sec();
  for (auto& p : *adc_trajectory_pb.mutable_trajectory_point()) {
    p.set_relative_time(p.relative_time() + dt);
  }
  // 发布信息
  planning_writer_->Write(adc_trajectory_pb);

  // record in history
  auto* history = injector_->history();
  history->Add(adc_trajectory_pb);

  return true;
}

void PlanningComponent::CheckRerouting() {
  AINFO << "检查路由函数调用!";
  auto* rerouting = injector_->planning_context()
                        ->mutable_planning_status()
                        ->mutable_rerouting();
  if (!rerouting->need_rerouting()) {
    return;
  }
  common::util::FillHeader(node_->Name(), rerouting->mutable_routing_request());
  rerouting->set_need_rerouting(false);
  rerouting_writer_->Write(rerouting->routing_request());
}

bool PlanningComponent::CheckInput() {
  AINFO << "检查输入函数调用!";
  ADCTrajectory trajectory_pb;
  auto* not_ready = trajectory_pb.mutable_decision()
                        ->mutable_main_decision()
                        ->mutable_not_ready();

  if (local_view_.localization_estimate == nullptr) {
    not_ready->set_reason("localization not ready");
  } else if (local_view_.chassis == nullptr) {
    not_ready->set_reason("chassis not ready");
  } else if (HDMapUtil::BaseMapPtr() == nullptr) {
    not_ready->set_reason("地图没有准备好!");
  } else {
    // nothing
  }

  if (FLAGS_use_navigation_mode) {
    if (!local_view_.relative_map->has_header()) {
      not_ready->set_reason("relative map not ready");
    }
  } else {
    if (!local_view_.routing->has_header()) {
      not_ready->set_reason("路由没有准备好!");
    }
  }

  if (not_ready->has_reason()) {
    AERROR << not_ready->reason() << "; skip the planning cycle.";
    common::util::FillHeader(node_->Name(), &trajectory_pb);
    planning_writer_->Write(trajectory_pb);
    return false;
  }
  return true;
}

}  // namespace planning
}  // namespace apoll
```

### Init 函数

他主要做三件事

1. 选择规划器
2. 读取并检查配置文件
3. 读取并检查输入的数据

### Proc 函数

planning 的 Proc 函数需要接收三个消息分别是,他们也是plannning启动的关键输入

1. `const std::shared_ptr<prediction::PredictionObstacles>` // 预测的障碍物信(prediction_obstacles)
2. `const std::shared_ptr<canbus::Chassis>`// 车辆底盘(chassis)信息(车辆的速度，加速度，航向角等信息)
3. `const std::shared_ptr<localization::LocalizationEstimate> ` // 车辆当前位置(localization_estimate)

他的任务就是不断进行规划

1. 检查数据
2. 生成路线，并将新的路线发布

```bash
Log file created at: 2023/01/11 18:53:45
Running on machine: in-dev-docker
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
I0111 18:53:45.310637 20642 class_loader_utility.h:79] registerclass:PlanningComponent,apollo::cyber::ComponentBase,/opt/apollo/neo/packages/planning-dev/latest/lib/libplanning_component.so
I0111 18:53:45.315501 20642 planning_component.cc:43] planning component init 函数调用! ///////
I0111 18:53:45.315992 20642 planning_component.cc:52] 选择相应的规划器 默认是OnLanePlanning 模块 
I0111 18:53:45.316018 20642 planning_component.cc:58] 默认选择的是 : onlaneplanning
I0111 18:53:45.316021 20642 planning_component.cc:62] 正在加载配置文件
I0111 18:53:45.318209 20642 planning_component.cc:77] 正在执行planning_base 的初始化 
I0111 18:53:45.828123 20642 planning_component.cc:81] 获取路由 routing信息
I0111 18:53:45.828958 20642 planning_component.cc:92] 读取红绿灯信息
I0111 18:53:45.829712 20642 planning_component.cc:118] 判断是否使用导航模式
E0111 18:53:45.880179 20647 reference_line_provider.cc:195] Routing is not ready.
E0111 18:53:45.940057 20648 reference_line_provider.cc:195] Routing is not ready.

I0111 18:53:50.352406 20652 planning_component.cc:149] planning—component Proc 函数调用! ///////////
I0111 18:53:50.352429 20652 planning_component.cc:239] 检查路由函数调用!
E0111 18:53:50.352438 20652 pnc_map.cc:267] Route is empty.
I0111 18:53:50.352522 20652 planning_component.cc:252] 检查输入函数调用!
E0111 18:53:50.352545 20652 planning_component.cc:279] 路由没有准备好!; skip the planning cycle.
E0111 18:53:50.352632 20652 planning_component.cc:185] Input check failed
E0111 18:53:50.400401 20649 reference_line_provider.cc:195] Routing is not ready.
E0111 18:53:50.450472 20657 reference_line_provider.cc:195] Routing is not ready.
I0111 18:53:50.451936 20650 planning_component.cc:149] planning—component Proc 函数调用! ///////////
I0111 18:53:50.451952 20650 planning_component.cc:239] 检查路由函数调用!
E0111 18:53:50.451958 20650 pnc_map.cc:267] Route is empty.
I0111 18:53:50.452006 20650 planning_component.cc:252] 检查输入函数调用!
E0111 18:53:50.452024 20650 planning_component.cc:279] 路由没有准备好!; skip the planning cycle.
E0111 18:53:50.452095 20650 planning_component.cc:185] Input check failed
E0111 18:53:50.510488 20644 reference_line_provider.cc:195] Routing is not ready.
I0111 18:53:50.552870 20645 planning_component.cc:149] planning—component Proc 函数调用! ///////////
```

总结 ：

1. 通过日志文件我们发现Init 函数只会执行一次，而Proc函数则是在规划期间一直执行的
2. init函数只是起到了初始化的作用
3. 真正进行规划的Proc函数

## c++ 知识点

1. `std::unique_ptr` 智能指针 -> 独占指针，该指针指向的空间不能共享，只能move，并且生存周期结束会释放对应的内存
2. `std::shared_ptr` 智能指针 -> 共享指针，该指针指向的空间可以共享，能够计数，并且生存周期结束会释放对应的内存
3. `final` 表示该类禁止继承
4. `override` 表示重写函数
5. `default` 表示自动生成的默认构造/析构函数
6. `namespace` 命名空间
7. `using ` 使用命名空间/别名

## 总结

1. planning 模块继承自 `Component` 类，他需要接收三个模板类型分别是

   - `const std::shared_ptr<prediction::PredictionObstacles>` // 预测的障碍物信(prediction_obstacles)
   - `const std::shared_ptr<canbus::Chassis>`// 车辆底盘(chassis)信息(车辆的速度，加速度，航向角等信息)
   - `const std::shared_ptr<localization::LocalizationEstimate> ` // 车辆当前位置(localization_estimate)

2. 以上三个数据类型对应了三个protobuf文件里面存储者各种数据 需要的可以去看看

   > vscode 安装protobuf 插件既可以舒适的浏览portobuf文件，如果还需要深入了解各种含义可以自行百度

   1. `/apollo_workspace/modules/common_msgs/prediction_msgs/prediction_obstacle.proto` 
   2. `/apollo_workspace/modules/common_msgs/chassis_msgs/chassis.proto`
   3. `/apollo_workspace/modules/common_msgs/localization_msgs/localization.proto`

3. planning 默认选择的是OnLanePlanning 规划器，所以后续会以这个为主讲解

4. planning 执行所检查的配置文件大多在`/apollo_workspace/modules/planning/conf` 中可以找到

5. 因为默认选择onlaneplanning 所以以后的planning_base 默认都是指向OnLanePlanning类

   ```c++
   planning_base_ = std::make_unique<OnLanePlanning>(injector_);
   ```

6. vscode 中按抓ctrl + 函数名可以直接跳转

   

