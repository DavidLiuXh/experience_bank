* 先放一个用C++11实现的FSM的代码: [kuafu](https://github.com/DavidLiuXh/kuafu)
* 咱们先来看一下什么是有限状态机(Finite-state machine, FSM), 先给一个 [百度百科的解释](https://baike.baidu.com/item/%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E6%9C%BA/2081914?fr=aladdin)
* 简单说就是作一件事可能会经过多个不同状态的转换, 转换依赖于在不同时间发生的不同事件来触发, 举个例子,比如 TCP的状态转换图, 在实现上就可以用FSM.
![tcp.jpeg](https://upload-images.jianshu.io/upload_images/2020390-913795540557367a.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
---
##### 传统的实现方案
* if...else : 搞一大堆if else, 一个函数写很长很长......
* swich...case : 也搞一大堆一个函数写很长很长......
##### FSM的实现方案
* 根据具体的业务需要, 将业务的处理流程定义为一个状态机, 此状态机中存在以下必要元素
  1. 根据业务需要, 拆解抽象出若干个不同状态 `State`, 并确定此状态机的初始状态;
  2. 根据实现需要, 抽象出用于触发状态转换的事件 `Event`;
  3. 为了处理一个`Event`, 需要定义状态的转换过程`Transition`;
  4. 状态机要先判断当前所处的状态是否与当前发生的`Event`匹配(注意: 相同的状态可能同时匹配多个Event);
* 用张简图来说明一下
![fsm.jpg](https://upload-images.jianshu.io/upload_images/2020390-9c253b04e8f598b4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  1. MachineSet可以同时管理多个Machine;
  2. 外部触发的Event进入到MachineSet的事件队列;
  3. 事件队列里的Event被顺序处理, 被Dispatch到match的Machine;
  4. Machine根据当前的所处的state和Event类型来判断当前Event是否有效;
  5. 如果上面(4)中的Event有效, 则进行状态转换;
  6. 状态转换具体来说涉及到三个回调函数:
      6.1 当前state离开, 是第一个回调,需要使用者根据实际需要处理;
      6.2 trasition这个转换过程, 是第二个回调;
      6.3 新state的进入, 是第三个回调;
* 一个简单的状态机,差不多就是上面这些内容, 剩下的就是用程序语言把它实现出来了;
##### FSM的C++ 实现
* 先放一个用C++11实现的FSM的代码: [kuafu](https://github.com/DavidLiuXh/kuafu)
* 实现简介:
  主要就是按上面的思路, 封装了
   [MachineSet](https://github.com/DavidLiuXh/kuafu/blob/master/fsm/machine_set.h),
   [Machine](https://github.com/DavidLiuXh/kuafu/blob/master/fsm/machine.h), 
   [Event](https://github.com/DavidLiuXh/kuafu/blob/master/fsm/event.h), 
   [Transition](https://github.com/DavidLiuXh/kuafu/blob/master/fsm/transition.h),
   [Predicate](https://github.com/DavidLiuXh/kuafu/blob/master/fsm/transition_predicate.h)
* 对于Event的处理, 提供两种方案:
   1. 直接使用MachineSet提供的`StartBackground`, 开启一个work thread, 在这个work thread中不断从存储event的fifo队列中获取event后dispatch到各个machine;
   2. 不使用MachineSet提供的event fifo, 实现自己的`MachineSetHandler`, 将其实例注册到MachineSet, 从event的派发;
##### 一个具体的实际
* 我们来使用上面的FSM的实现来模拟一个用户登陆的场景;
* 定义用到的Event和几种不同的事件类型
```
enum class FoodEventType {
    FET_LOGIN, // 开始登陆 
    FET_LOGIN_FAILED, // 登陆失败
    FET_LOGIN_OK,  // 登陆成功
    FET_LOGOUT, // 登出
};

class FoodEvent : public kuafu::EventTemplate<FoodEventType> {
    public:
        using underlying_type = std::underlying_type<FoodEventType>::type;

        FoodEvent(FoodEventType type, const kuafu::MachineBaseSharedPtr& machine)
            :kuafu::EventTemplate<FoodEventType>(type, machine) {
            }
       ...
    }
};
```
* 定义用到的状态机, 从 [kuafu::StateMachine](https://github.com/DavidLiuXh/kuafu/blob/master/fsm/machine.h) 继承, 其中包括用过的几种state和transition
```
class FoodMachine : public kuafu::StateMachine {
    public:
        FoodMachine(const std::string& name);

    public:
        virtual void Birth();

    public:
       // 需要用到的三种状态: 启动, 登陆中, 成功
        kuafu::StateSharedPtr startup_;
        kuafu::StateSharedPtr loging_;
        kuafu::StateSharedPtr welcome_;

        // 需要用到的几种转换
        kuafu::TransitionSharedPtr startup_loging_;
        kuafu::TransitionSharedPtr loging_welcome_;
        kuafu::TransitionSharedPtr loging_startup_;
        kuafu::TransitionSharedPtr welcome_startup_;
        kuafu::TransitionSharedPtr welcome_timeout_;
};
```
* 在`Birth()`函数中构造 state和 transition, `Birth()`是`StateMachine`的一个虚函数, 每个用户实现的Machine都需要实现它:
```
void FoodMachine::Birth() {
    startup_ = kuafu::State::MakeState(*this, "startup");
    loging_ = kuafu::State::MakeState(*this, "loging");
    welcome_ = kuafu::State::MakeState(*this, "welcom", 5000);

    startup_loging_ = kuafu::Transition::MakeTransition("startup_loging",
                startup_,
                loging_,
                std::make_shared<kuafu::SimplePredicate<FoodEvent>>(FoodEventType::FET_LOGIN));
    loging_welcome_ = kuafu::Transition::MakeTransition("loging_welcome",
                loging_,
                welcome_,
                std::make_shared<kuafu::SimplePredicate<FoodEvent>>(FoodEventType::FET_LOGIN_OK));
    loging_startup_ = kuafu::Transition::MakeTransition("loging_startup",
                loging_,
                startup_,
                std::make_shared<kuafu::SimplePredicate<FoodEvent>>(FoodEventType::FET_LOGIN_FAILED));
    welcome_startup_ = kuafu::Transition::MakeTransition("welcome_startup",
                welcome_,
                startup_,
                std::make_shared<kuafu::SimplePredicate<FoodEvent>>(FoodEventType::FET_LOGOUT));
    welcome_timeout_ = kuafu::Transition::MakeTransition("welcome_timeout",
                welcome_,
                welcome_,
                std::make_shared<kuafu::TimeoutPredicate>(type_));
}
```
* 创建`MachineSet`, 并开始event处理线程
```
kuafu::MachineSetSharedPtr machine_set = kuafu::MachineSet::MakeMachineSet();
machine_set->StartBackground(500);
```
* 创建用户定义的Machine, 设置初始状态
```
std::shared_ptr<FoodMachine> food_machine = kuafu::MakeMachine<FoodMachine>("food_machine");
food_machine->SetStartState(food_machine->startup_);
```
* 设置state和transition相应的回调
```
food_machine->startup_->OnEnter = [&](kuafu::MachineBase& machine,
                        const kuafu::StateSharedPtr& state) {
                INFO_LOG("Enter " << state->GetName());
            };
...
food_machine->startup_loging_->OnTransition = [&](kuafu::MachineBase& machine,
                        const kuafu::StateSharedPtr& from_state,
                        kuafu::ITransitionSharedPtr transition,
                        kuafu::EventSharedPtr event,
                        const kuafu::StateSharedPtr& to_state) {
                INFO_LOG(transition->GetName()
                            << " | "
                            << from_state->GetName()
                            << " -> "
                            << to_state->GetName());
            };
...
```
* 模拟event发生:
```
machine_set->Enqueue(std::make_shared<kuafu::MachineOperationEvent>(
                            kuafu::MachineOperator::MO_ADD,
                            food_machine));

            machine_set->Enqueue(std::make_shared<FoodEvent>(
                            FoodEventType::FET_LOGIN,
                            food_machine));
            machine_set->Enqueue(std::make_shared<FoodEvent>(
                            FoodEventType::FET_LOGIN_OK,
                            food_machine));
            machine_set->Enqueue(std::make_shared<FoodEvent>(
                            FoodEventType::FET_LOGOUT,
                            food_machine));
```