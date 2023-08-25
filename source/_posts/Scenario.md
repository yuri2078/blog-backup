---
title: Scenario(3)
categories: apollo-源码分析
comments: true
description: 我的爱宛如孤岛之花，不为人知的绽放,不为人知的凋零！
cover: https://www.loliapi.com/acg/?id=27
date: 2023-01-17 18:49:27
tags: apollo
keywords:
---

# Scenario 

> 本文参考自 ： [原文](https://zhuanlan.zhihu.com/p/494816954)

## planner选择

```javascript
I0202 19:15:10.702800 25408 on_lane_planning.cc:165] 分配planner -> PUBLIC_ROAD
I0202 19:15:10.702884 25408 scenario_manager.cc:59] ScenarioManager 初始化
I0202 19:15:10.725889 25408 scenario_manager.cc:72] 创建Scenario -> LANE_FOLLOW
```

1. 根据日志我们可以看出他会在`on_lane_planning`中分配planner -> public_road
2. 所以如果是on_lane_planning的话他默认使用public——road 进行具体的规划

## public_road_plannner

```c++
Status PublicRoadPlanner::Init(const PlanningConfig& config) {
  config_ = config;
  // ScenarioManager会实例化一个全局的scenario_manager_对象来进行场景管理，
  // PublicRoadPlanner初始化时会调用配置文件里的参数来建立这个对象
  scenario_manager_.Init(config); // 实列化一个scenario—manager对象用来管理场景
  return Status::OK();
}

// 具体的场景规划
Status PublicRoadPlanner::Plan(const TrajectoryPoint& planning_start_point,
                               Frame* frame,
                               ADCTrajectory* ptr_computed_trajectory) {
                            
  scenario_manager_.Update(planning_start_point, *frame); // 决策使用哪个场景
  scenario_ = scenario_manager_.mutable_scenario(); // 获取当前场景
  auto result = scenario_->Process(planning_start_point, frame); // 处理当前场景

  // 打印debug信息
  if (FLAGS_enable_record_debug) {
    auto scenario_debug = ptr_computed_trajectory->mutable_debug()
                              ->mutable_planning_data()
                              ->mutable_scenario();
    scenario_debug->set_scenario_type(scenario_->scenario_type());
    scenario_debug->set_stage_type(scenario_->GetStage());
    scenario_debug->set_msg(scenario_->GetMsg());
  }

  // 场景处理成功
  if (result == scenario::Scenario::STATUS_DONE) {
    // 只有在场景处理完成的时候才会进行场景Update
    // STATUS_DONE
    scenario_manager_.Update(planning_start_point, *frame);
    AINFO << "场景处理完成 ";
  }
  else if (result == scenario::Scenario::STATUS_UNKNOWN)
  {
    AERROR << "场景处理失败!";
    return Status(common::PLANNING_ERROR, "不知道你返回了什么东西捏!");
  }
  return Status::OK();
}
```

我们看public_road 的源码可以看出，他只有两个函数

### Init 阶段

> 调用配置文里的参数，实列化一个scenario—manager对象用来管理场景.所以本阶段只做一件事就是调用 ScenarioManager的init 函数进行初始化

```c++
bool ScenarioManager::Init(const PlanningConfig& planning_config) {
  AINFO << "ScenarioManager (Init) -- 初始化";
  planning_config_.CopyFrom(planning_config); // 复制配置文件
  RegisterScenarios(); // 检查配置文件，注册场景
  // 创建场景，默认为lane_follow
  default_scenario_type_ = ScenarioType::LANE_FOLLOW; // 设置默认场景type
  current_scenario_ = CreateScenario(default_scenario_type_); // 创建场景
  AINFO << "初始化(Scenario manager init) 创建场景 -> " << current_scenario_->Name();
  return true;
}
```

1. 通过传入的配置文件，进行配置 
2. 注册所有场景 RegisterScenarios()函数
3. 设置默认的场景 默认 -> LANE_FOLLOW
4. 创建场景 CreateScenario() 通过传入的场景type来创建场景

### Plan 阶段

> 这个阶段就是执行具体的规划逻辑的地方

#### Update

>  ScenarioManager类的Update()函数，用来决策当前处在什么场景。如果进入了新的场景，会创建一个新的对象来进行之后的规划逻辑

```c++
void ScenarioManager::Update(const common::TrajectoryPoint& ego_point, const Frame& frame) {
  ACHECK(!frame.reference_line_info().empty());
  Observe(frame);
  ScenarioDispatch(frame); // 会根据配置选择基于规则还是基于学习的决策方法
}
```

##### ScenarioDispatch 

函数中他会根据你的配置文件选择基于规划还是基于学习的决策方法。如果遇到新得场景则会更新场景

```c++
void ScenarioManager::ScenarioDispatch(const Frame& frame) {
  ACHECK(!frame.reference_line_info().empty()); // 检查参考线是否为空
  ScenarioType scenario_type; 
  // 获取历史点的个数
  int history_points_len = 0; 
  if (injector_->learning_based_data() &&
      injector_->learning_based_data()->GetLatestLearningDataFrame()) {
    history_points_len = injector_->learning_based_data()
                             ->GetLatestLearningDataFrame()
                             ->adc_trajectory_point_size();
  }
  // 默认为 0 也就是 PlanningConfig::NO_LEARNING 这是默认模式，别的就是基于学习的模式
  if ((planning_config_.learning_mode() == PlanningConfig::E2E ||
       planning_config_.learning_mode() == PlanningConfig::E2E_TEST) &&
      history_points_len >= FLAGS_min_past_history_points_len)
  {
    scenario_type = ScenarioDispatchLearning();
  }
  else
  {
    // 选择不基于学习的决策方式
    scenario_type = ScenarioDispatchNonLearning(frame);
  }

  ADEBUG << "select scenario: " << ScenarioType_Name(scenario_type);
  AINFO << "选择的场景 -> " << ScenarioType_Name(scenario_type);

  // update PlanningContext
  UpdatePlanningContext(frame, scenario_type);

  // 如果新场景不是当前场景就创建场景，并且更新
  if (current_scenario_->scenario_type() != scenario_type) {
    current_scenario_ = CreateScenario(scenario_type); // 如果需要就更新场景
    AINFO << "更新场景 -> " << current_scenario_->Name();
  }
  else
  {
    AINFO << "仍然是 -> " << current_scenario_->Name() << " 不更新场景";
  }
}
```

##### ScenarioDispatchNonLearning

因为配置文件默认是 `learning_mode() = 0` 所以他默认会调用`ScenarioDispatchNonLearning`方法来更新场景

`ScenarioDispatchNonLearning()`函数默认从`lanefollow`场景开始判断，首先根据驾驶员的意图来安排场景，如果不是默认的lanefollow场景，直接输出当前场景；如果是lanefollow场景，会依次判断是否属于别的场景；即剩余场景之间的跳转必须经过lanefollow这个场景

我们看他的代码

```c++
ScenarioType ScenarioManager::ScenarioDispatchNonLearning(const Frame& frame) {
  ////////////////////////////////////////
  // 默认场景: LANE_FOLLOW
  ScenarioType scenario_type = default_scenario_type_; 

  ////////////////////////////////////////
  // Pad Msg scenario
  scenario_type = SelectPadMsgScenario(frame); // 在场景判断时，首先调用函数SelectPadMsgScenario()，根据驾驶员意图来安排场景

  // 判断当前场景是不是默认场景
  if (scenario_type == default_scenario_type_) {
    // check current_scenario (not switchable)
    switch (current_scenario_->scenario_type()) {
      case ScenarioType::LANE_FOLLOW:
      case ScenarioType::PULL_OVER:
        break;
      case ScenarioType::BARE_INTERSECTION_UNPROTECTED:
      case ScenarioType::EMERGENCY_PULL_OVER:
      case ScenarioType::PARK_AND_GO:
      case ScenarioType::STOP_SIGN_PROTECTED:
      case ScenarioType::STOP_SIGN_UNPROTECTED:
      case ScenarioType::TRAFFIC_LIGHT_PROTECTED:
      case ScenarioType::TRAFFIC_LIGHT_UNPROTECTED_LEFT_TURN:
      case ScenarioType::TRAFFIC_LIGHT_UNPROTECTED_RIGHT_TURN:
      case ScenarioType::VALET_PARKING:
      case ScenarioType::YIELD_SIGN:
        // must continue until finish
        // 如果当前场景没有处理完毕，那么继续处理。
        if (current_scenario_->GetStatus() !=
            Scenario::ScenarioStatus::STATUS_DONE) {
          scenario_type = current_scenario_->scenario_type(); // 改变scenario_type 继续处理当前场景
        }
        break;
      default:
        break;
    }
  }

  /*如果是default_scenario_type_ 就开始判断是不是属于别的场景*/
  ////////////////////////////////////////
  // ParkAndGo / starting scenario
  if (scenario_type == default_scenario_type_) {
    if (FLAGS_enable_scenario_park_and_go) {
      scenario_type = SelectParkAndGoScenario(frame);
    }
  }

  ////////////////////////////////////////
  // intersection scenarios
  if (scenario_type == default_scenario_type_) {
    scenario_type = SelectInterceptionScenario(frame);
  }

  ////////////////////////////////////////
  // pull-over scenario
  if (scenario_type == default_scenario_type_) {
    if (FLAGS_enable_scenario_pull_over) {
      scenario_type = SelectPLANE_FOLLOW:
      case ScenarioType::PULL_OVERullOverScenario(frame);
    }
  }

  ////////////////////////////////////////
  // VALET_PARKING scenario
  if (scenario_type == default_scenario_type_) {
    scenario_type = SelectValetParkingScenario(frame);
  }

  return scenario_type;
}
```

`ScenarioDispatchNonLearning`函数一开始设置了一个默认的场景类型，并且调用`SelectPadMsgScenario` 函数来检查驾驶员是否有意更改场景。如果驾驶员不需要更改场景，那么进入接下的对比，如果需要直接返回了。那么这里`scenario_type` 就不会是`default_scenario_type_` 那么直接返回场景类型，然后通过`**ScenarioDispatch**` 函数新建新的场景了。如果不是

1. `SelectPadMsgScenario` 函数表示驾驶员需要更改场景`scenario_type` 就不是默认类型 `ScenarioDispatchNonLearning` 里的内容直接跳过，不需要再判断属于哪个类型。chang jing
2. `SelectPadMsgScenario` 函数表示驾驶员不需要更改场景，`scenario_type` 就是默认类型可以进行接下来的判断程序。
3. 从代码中我们可以看到只有当前场景是`LANE_FOLLOW 或者 PULL_OVER` 或者当前场景已经处理完毕的时候，我们才能更改为别的场景
4. 如果当前场景不是默认场景并且并没有处理好那么他将会继续处理场景。

##### SelectPadMsgScenario

> 检查驾驶员是否有意更改场景

##### CreateScenario

> 该函数在`ScenarioDispatch`函数中被调用，他会根据传入的场景类型来创建场景，并调用该场景的Init 函数进行初始化，每个场景的初始化他会创建第一个默认的Stage

```c++
// 调用场景初始化函数 ScenarioManager::CreateScenario
  if (ptr != nullptr) {
    AINFO << "CreateScenario函数调用 创建Scenario -> " << ScenarioType_Name(scenario_type) << "成功，并进行初始化";
    ptr->Init();
  }

// 场景初始化
void Scenario::Init() {
  ACHECK(!config_.stage_type().empty());
  AINFO << this->Name() << " 场景(Init) 函数调用进行初始化,并开始遍历 Stage";
  // set scenario_type in PlanningContext
  auto* scenario = injector_->planning_context()
                       ->mutable_planning_status()
                       ->mutable_scenario();
  scenario->Clear();
  scenario->set_scenario_type(scenario_type());

  for (const auto& stage_config : config_.stage_config()) {
    stage_config_map_[stage_config.stage_type()] = &stage_config;
  }
  for (int i = 0; i < config_.stage_type_size(); ++i)
  {
    auto stage_type = config_.stage_type(i);
    AINFO << "Stage " << i << "  ->  " << StageType_Name(stage_type);
    ACHECK(common::util::ContainsKey(stage_config_map_, stage_type))
        << "stage type : " << StageType_Name(stage_type)
        << " has no config";
  }
  ADEBUG << "init stage "
         << StageType_Name(config_.stage_type(0));
  // // 初始化后分配该场景下的默认stage
  current_stage_ = CreateStage(*stage_config_map_[config_.stage_type(0)], injector_);
  
}
```



#### **mutable_scenario**

> 获取当前场景(Scenario) 这个函数就直接将智能指针`(std::unique_ptr<Scenario> current_scenario_;)`中的地址返回了。
>
> `return current_scenario_.get();`

#### Process

> 调用当前Scenario对象的Process()函数，来执行这个场景对应的逻辑.根据日志来看他默认选择的场景是LANE_FOLLOW ，所以这里的Process函数是LANE_FOLLOW下的
>
> **I0202** 19:15:18.855336 25410 scenario_manager.cc:826] 选择的场景 -> LANE_FOLLOW

在实例化Scenario后会调用它的LoadConfig()与void Scenario::Init() 两个函数，读取配置文件，并且初始化当前对象，注册配置文件中定义的该场景拥有的stage。其中会在`ScenarioManager -> Init -> RegisterScenarios 中调用LoadConfig` 进行注册场景stage。在`ScenarioManager -> Init -> CreateScenario`中创建好场景之后会调用场景对应的`Init`函数

场景的执行在"[scenario.cc](https://link.zhihu.com/?target=http%3A//scenario.cc/)"和对应的场景目录中，在apollo中每个场景由一个或者多个阶段(stage)注册而来，每个阶段stage又由不同的任务(task)组成。执行一个场景，就是顺序执行不同阶段的不同任务。在每一个规划周期，所谓决策的其中一个作用就是定位到当前所在的scenario以及stage，并且按顺序执行完当前stage中注册的所有task

```c++
// 场景的执行在"scenario.cc"和对应的场景目录中，在apollo中每个场景由一个或者多个阶段(stage)注册而来，
// 每个阶段stage又由不同的任务(task)组成。执行一个场景，就是顺序执行不同阶段的不同任务。
// 在每一个规划周期，所谓决策的其中一个作用就是定位到当前所在的scenario以及stage，并且按顺序执行完当前stage中注册的所有task
Scenario::ScenarioStatus Scenario:: Process(
    const common::TrajectoryPoint& planning_init_point, Frame* frame) {
  AINFO << "当前 Stage 是 -> " << current_stage_->Name();
  if (current_stage_ == nullptr) {
    AWARN << "当前场景为nullptr!";
    return STATUS_UNKNOWN;
  }
  if (current_stage_->stage_type() == StageType::NO_STAGE) {
    scenario_status_ = STATUS_DONE;
    return scenario_status_;
  }
  auto ret = current_stage_->Process(planning_init_point, frame); // 处理当前stage 并且返回处理状态
  switch (ret) {
    // 异常状态
    case Stage::ERROR: {
      AERROR << "Stage '" << current_stage_->Name() << "' 返回异常";
      scenario_status_ = STATUS_UNKNOWN;
      break;
    }
    // 正在处理状态
    case Stage::RUNNING: {
      scenario_status_ = STATUS_PROCESSING;
      break;
    }
    // 完成状态
    case Stage::FINISHED: {
      auto next_stage = current_stage_->NextStage();
      if (next_stage != current_stage_->stage_type()) {
        AINFO << "由 Stage -> " << current_stage_->Name() << " 转到 Stage -> " << StageType_Name(next_stage);
        // 如果stage 全部处理完毕，就返回完成状态
        if (next_stage == StageType::NO_STAGE) {
          scenario_status_ = STATUS_DONE;
          return scenario_status_;
        }
        // 检查配置文件是否存在
        if (stage_config_map_.find(next_stage) == stage_config_map_.end()) {
          AERROR << "stage 配置文件查找失败" << next_stage;
          scenario_status_ = STATUS_UNKNOWN;
          return scenario_status_;
        }
        // 如果当前stage处理完毕，并且配置文件正常，且接下来有未处理的stage 就新建一个stage
        current_stage_ = CreateStage(*stage_config_map_[next_stage], injector_);
        if (current_stage_ == nullptr) {
          AWARN << "当前stage 是一个空指针";
          return STATUS_UNKNOWN;
        }
      }
      // 如果当前场景不为空，且未处于全部状态就返回正在处理，否则返回处理完毕`
      if (current_stage_ != nullptr &&
          current_stage_->stage_type() != StageType::NO_STAGE) {
        scenario_status_ = STATUS_PROCESSING;
      } else {
        scenario_status_ = STATUS_DONE;
      }
      break;
    }
    default: {
      AWARN << "返回状态异常 -> " << ret;
      scenario_status_ = STATUS_UNKNOWN;
    }
  }
  return scenario_status_;
}
```

1. 因为一开始`Scenario`会调用对应`Init`函数，所以第一个`Process`函数执行的Stage 是第一个Stage
2. 开始处理之后，如果当前stage 是没有stage状态，或者当前没有场景就直接返回对应状态
3. 如果场景异常 就返回异常状态
4. 如果场景正在处理，就返回正在处理状态
5. 如果场景处理完毕

   1. 如果下一个Stage类型不等于当前Stage，且下一个stage类型是no——stage 就直接返回处理完毕
   2. 如果下一个Stage类型不等于当前Stage，但是配置文件中找不到该类型就返回类型为止
   3. 如果下一个Stage类型不等于当前Stage，且不输入以上两种情况就更新Stage


# Task

> 通过前面的知识点 我们可以知道当执行具体的规划任务的时候，我们会调用`Scenario`的`process`函数，依次执行`Stage`，所以我们需要了解各个场景的stage执行。而apollo 又将每个Stage 又分为不同的task.

## lane_follow

> 这是默认的跟车场景(Scenario) 我们看他的配置文件只有一个stage。所以我们直接看这个stage
>
> scenario_type: LANE_FOLLOW
> stage_type: LANE_FOLLOW_DEFAULT_STAGE

## **CreateStage**

> 看`Scenario.h` 我们可以看出，**CreateStage** 函数是纯虚函数，所以我们需要重写该函数。各个不同的场景有着自己不同的创建场景的需求。我们看`lane_follow` 他就一个Stage，直接判断是不是该场景，然后创建返回就行了

```c++
std::unique_ptr<Stage> LaneFollowScenario::CreateStage(
    const ScenarioConfig::StageConfig& stage_config,
    const std::shared_ptr<DependencyInjector>& injector) {
  if (stage_config.stage_type() != StageType::LANE_FOLLOW_DEFAULT_STAGE) {
    AERROR << "Follow lane 不支持的类型 ->  "
           << StageType_Name(stage_config.stage_type());
    return nullptr;
  }
  return std::unique_ptr<Stage>(new LaneFollowStage(stage_config, injector));
}
```

## Process

> 前面看到`plannear` 的process 函数会不断调用stage进行规划，而stage的规划函数也是`Process` 所以对于场景和stage 具体规划流程都在Process函数当中

```c++
// LaneFollowStage 的Process() 函数执行主要的规划逻辑，在函数内部会对变道的效率来进行判断从而选择是否按照变道进行规划，或者保持本车道运行
Stage::StageStatus LaneFollowStage::Process(
    const TrajectoryPoint& planning_start_point, Frame* frame) {
  bool has_drivable_reference_line = false;

  // 参考线的条数
  ADEBUG << "Number of reference lines:\t"
         << frame->mutable_reference_line_info()->size();
  AINFO << "Number of reference lines:\t" << frame->mutable_reference_line_info()->size();
  unsigned int count = 0;
  // 遍历所有参考线，直到找到可以用来规划的参考线之后退出
  for (auto& reference_line_info : *frame->mutable_reference_line_info()) {
    // TODO(SHU): need refactor
    if (count++ == frame->mutable_reference_line_info()->size()) {
      break;
    }
    ADEBUG << "No: [" << count << "] Reference Line.";
    ADEBUG << "IsChangeLanePath: " << reference_line_info.IsChangeLanePath();

    // 找到可用来规划的参考线，退出循环
    if (has_drivable_reference_line) {
      reference_line_info.SetDrivable(false);
      break;
    }
    // 执行具体规任务
    auto cur_status = PlanOnReferenceLine(planning_start_point, frame, &reference_line_info);
    // 如果规划完成
    if (cur_status.ok()) {
      // 如果发生lanechange，判断reference_line的cost
      AINFO << "规划完成";
      if (reference_line_info.IsChangeLanePath())
      {
        AINFO << "发生变道!!";
        ADEBUG << "reference line is lane change ref.";
        ADEBUG << "FLAGS_enable_smarter_lane_change: "
               << FLAGS_enable_smarter_lane_change;

        // 如果规划成功后，还需要判断目标车道的变道cost，如果cost太高，那么就会舍弃掉这条目标车道的reference_line, 
        // 此时放弃变道的规划，继续循环使用原车道的reference_line进行规划
        if (reference_line_info.Cost() < kStraightForwardLineCost &&
            (LaneChangeDecider::IsClearToChangeLane(&reference_line_info) ||
             FLAGS_enable_smarter_lane_change)) {
          // If the path and speed optimization succeed on target lane while
          // under smart lane-change or IsClearToChangeLane under older version
          has_drivable_reference_line = true;
          reference_line_info.SetDrivable(true);
          LaneChangeDecider::UpdatePreparationDistance(
              true, frame, &reference_line_info, injector_->planning_context());
          ADEBUG << "\tclear for lane change";
          AINFO << "变道成功!";
        }
        else
        {
          LaneChangeDecider::UpdatePreparationDistance(
              false, frame, &reference_line_info,
              injector_->planning_context());
          reference_line_info.SetDrivable(false);
          ADEBUG << "\tlane change failed";
          AINFO << "变道失败！继续使用原车道规划";
        }
      }
      else
      {
        ADEBUG << "reference line is NOT lane change ref.";
        // 如果没有lanechange，stage执行结果为OK，则has_drivable_reference_line置位true
        has_drivable_reference_line = true;
      }
    } else {
      reference_line_info.SetDrivable(false);
    }
  }
  // 根据has_drivable_reference_line这个标志位的结果，返回stage执行的结果
  return has_drivable_reference_line ? StageStatus::RUNNING
                                     : StageStatus::ERROR;
}
```

根据函数我们可以看出`LaneFollowStage`的`Process`函数的作用就是不断遍历参考线，直到找到可以用来规划的参考线。并且如果规划的结果需要变道他会计算变道的`cost`然后再判断具体是不是需要变道

## PlanOnReferenceLine

> auto cur_status = PlanOnReferenceLine(planning_start_point, frame, &reference_line_info);
>
> 通过代码我们可以看到，stage 的具体规划逻辑实在`PlanOnReferenceLine`中的

```c++
// 具体规划逻辑
Status LaneFollowStage::PlanOnReferenceLine(
    const TrajectoryPoint& planning_start_point, Frame* frame,
    ReferenceLineInfo* reference_line_info) {
  // 判断是否有lanechange意图，如果有计算cost
  if (!reference_line_info->IsChangeLanePath()) {
    reference_line_info->AddCost(kStraightForwardLineCost);
  }
  ADEBUG << "planning start point:" << planning_start_point.DebugString();
  ADEBUG << "Current reference_line_info is IsChangeLanePath: "
         << reference_line_info->IsChangeLanePath();

  // 先把状态设置ok
  auto ret = Status::OK();
  // 遍历task
  for (auto* task : task_list_) {
    // 记录开始时间
    const double start_timestamp = Clock::NowInSeconds();
    // 调用task的Execute 进行具体的规划
    ret = task->Execute(frame, reference_line_info);
    // 记录结束时间
    const double end_timestamp = Clock::NowInSeconds();
    // 统计耗时，单位ms
    const double time_diff_ms = (end_timestamp - start_timestamp) * 1000;
    ADEBUG << "after task[" << task->Name()
           << "]:" << reference_line_info->PathSpeedDebugString();
    ADEBUG << task->Name() << " time spend: " << time_diff_ms << " ms.";
    AINFO  << "task ->  " << task->Name() << " 执行耗时 : " << time_diff_ms << " ms.";
    RecordDebugInfo(reference_line_info, task->Name(), time_diff_ms);

    if (!ret.ok()) {
      AERROR << "Failed to run tasks[" << task->Name()
             << "], Error message: " << ret.error_message();
      AINFO << "执行失败捏!";
      break;
    }
    。.......
```

通过代码我们可以看出他的规划逻辑就是遍历stage对应的`task`，然后调用task对应的`Execute`函数

## 分割线

## **LANE_CHANGE_DECIDER**

> 根据日志我们看出`LANE_FOLLOW_DEFAULT_STAGE`的第一个task就是`LANE_CHANGE_DECIDER`

```javascript
I0206 19:24:20.367132 18917 scenario.cc:75] 目前正在执行的Stage是 -> LANE_FOLLOW_DEFAULT_STAGE
I0206 19:24:20.367136 18917 lane_follow_stage.cc:101] Number of reference lines:	1
I0206 19:24:20.367144 18917 lane_follow_stage.cc:196] task ->  LANE_CHANGE_DECIDER 执行耗时 : 0.00405312 ms.
I0206 19:24:20.367161 18917 lane_follow_stage.cc:196] task ->  PATH_REUSE_DECIDER 执行耗时 : 0.00119209 ms.
I0206 19:24:20.367166 18917 lane_follow_stage.cc:196] task ->  PATH_LANE_BORROW_DECIDER 执行耗时 : 0.00238419 ms.
I0206 19:24:20.368664 18917 lane_follow_stage.cc:196] task ->  PATH_BOUNDS_DECIDER 执行耗时 : 1.48964 ms.
I0206 19:24:20.377274 18917 lane_follow_stage.cc:196] task ->  PIECEWISE_JERK_PATH_OPTIMIZER 执行耗时 : 8.58712 ms.
I0206 19:24:20.379036 18917 lane_follow_stage.cc:196] task ->  PATH_ASSESSMENT_DECIDER 执行耗时 : 1.73497 ms.
I0206 19:24:20.379052 18917 lane_follow_stage.cc:196] task ->  PATH_DECIDER 执行耗时 : 0.00619888 ms.
I0206 19:24:20.379058 18917 lane_follow_stage.cc:196] task ->  RULE_BASED_STOP_DECIDER 执行耗时 : 0.00238419 ms.
I0206 19:24:20.379397 18917 lane_follow_stage.cc:196] task ->  SPEED_BOUNDS_PRIORI_DECIDER 执行耗时 : 0.333786 ms.
I0206 19:24:20.380827 18917 lane_follow_stage.cc:196] task ->  SPEED_HEURISTIC_OPTIMIZER 执行耗时 : 1.42074 ms.
I0206 19:24:20.380861 18917 lane_follow_stage.cc:196] task ->  SPEED_DECIDER 执行耗时 : 0.0245571 ms.
I0206 19:24:20.381150 18917 lane_follow_stage.cc:196] task ->  SPEED_BOUNDS_FINAL_DECIDER 执行耗时 : 0.281811 ms.
I0206 19:24:20.383031 18917 lane_follow_stage.cc:196] task ->  PIECEWISE_JERK_SPEED_OPTIMIZER 执行耗时 : 1.87016 ms.
I0206 19:24:20.383045 18917 lane_follow_stage.cc:196] task ->  RSS_DECIDER 执行耗时 : 0.00429153 ms.
```

他的作用：

- **判断当前是否进行变道，以及变道的状态，并将结果存在变量lane_change_status中；**
- **变道过程中将目标车道的reference line放置到首位，变道结束后将当前新车道的reference line放置到首位**

默认配置

```json
default_task_config: {
  task_type: LANE_CHANGE_DECIDER
  lane_change_decider_config {
    enable_lane_change_urgency_check: false
    enable_prioritize_change_lane: false
    enable_remove_change_lane: false
    reckless_change_lane: false
    change_lane_success_freeze_time: 1.5
    change_lane_fail_freeze_time: 1.0
  }
}
```

在`LaneChangeDecider`中我们并没有看到`Execute`函数，但是前面说task执行就会调用task的`Execute`函数这是为什么呢？

我们看task 类的`Execute`的函数

我们看`LaneChangeDecider`继自`decider`类的`Execute`函数

```c++
apollo::common::Status Decider::Execute(
    Frame* frame, ReferenceLineInfo* reference_line_info) {
  Task::Execute(frame, reference_line_info);
  return Process(frame, reference_line_info);
}
```

他是重写类Execute函数，并且他的作用就是调用基类task的`Execute`函数(他的作用就是复制配置文件)后，重新调用Process类。所以一个task的规划逻辑还是在`decider`的子类的`Process`函数中



