<!--
 * @Author: Shuai Wang
 * @Github: https://github.com/wsustcid
 * @Version: 1.0.0
 * @Date: 2021-09-27 17:51:02
 * @LastEditTime: 2021-10-13 20:37:43
-->
- [4. 动态目标交互](#4-动态目标交互)
  - [4.1 运动预测](#41-运动预测)
    - [4.1.1 运动预测模型的要求](#411-运动预测模型的要求)
    - [4.1.2 运动预测问题简化](#412-运动预测问题简化)
    - [4.1.3 定常速度运动预测模型](#413-定常速度运动预测模型)
  - [4.2 Map-Aware 运动预测](#42-map-aware-运动预测)
    - [4.2.1 提升位置预测](#421-提升位置预测)
    - [4.2.2 提升速度预测](#422-提升速度预测)
    - [4.2.3 存在问题](#423-存在问题)
  - [4.3 碰撞时间计算](#43-碰撞时间计算)
    - [4.3.1 基于仿真的方法](#431-基于仿真的方法)
    - [4.3.2 基于估计的方法](#432-基于估计的方法)
    - [4.3.3 方案比较](#433-方案比较)
  - [4.4 补充阅读材料](#44-补充阅读材料)


# 4. 动态目标交互
本章节我们将解决自动驾驶车辆与动态目标的交互问题。

## 4.1 运动预测
动态目标的运动估计试图估计环境中所有动态目标在未来一段时间内的位置、朝向以及速度，这一信息对于我们的运动规划任务十分重要：
  - 基于其他目标可能的运动行为，我们可以为自动驾驶车辆规划未来驾驶动作或决策行为
  - 进一步的，基于预测出的目标轨迹，也可以进一步确认自动驾驶车辆将要执行的行车路线不会在未来时刻与任意目标发生碰撞


### 4.1.1 运动预测模型的要求
为了能够预测运动目标的运动，我们必须知道一些必要的信息才能进行预测，我们称之为强制性要求，如果没有这些信息，将无法进行预测。当然，还有一些可选的要求，用于进一步提升预测的效果。

**强制性要求**  
  - 首先是动态目标的类别；因为大多数的预测模型，针对不同的目标类型都有不同的预测算法
  - 其次是动态目标当前的状态：如位置、朝向、速度等

**可选要求**
  - 首先是动态目标的历史状态(位置、朝向、速度)或在环境中的历史轨迹，我们基于此信息可以得知该目标在环境中是如何进行机动的；（一般需要目标跟踪以及重识别技术）
  - 其次是高精地图信息：为了确定动态目标的未来行为；因为车辆一般都会沿着其所在车道继续行驶，因此高精地图可以为预测提供较强的线索信息
- 动态目标在当前时刻的图像信息也十分重要：对于车辆，图像可以提供指示灯或刹车灯的状态；对于行人：可以提供其朝向信息，为预测其未来要行进的方向提供重要帮助，即使该目标是静止的。
<div align=center><img src=./figs/prediction_requirements.png width=400 ></div>


### 4.1.2 运动预测问题简化
考虑到运动预测任务的复杂性，我们可以通过几个具体的假设来降低问题的复杂度。这里主要关注两类对象，车辆和行人，因为骑自行车的人、动物等目标都可以用类似的方法类比。

**关于车辆的假设**
  - Physics-based Assumptions: 车辆在运动时必须遵循一系列的物理学约束，比如之前提到的运动学和动力学约束；
  - maneuver-based Assumptions: 车辆在路上的运动，是在受限的环境空间内执行有限的机动指令的结果。具体来说，车辆总是会待在道路上并且遵守驾驶规则，比如在停车线前停车，他们不太可能开车越过人行道、草坪或障碍物
  - Interactions-aware Assumptions: 相比于单独的预测每辆车的运动，我们也需要考虑动态目标之间也会进行交互并做出反应这一假设，比如车辆的并线行为：在目标车道的车辆一般都会减速为并入车辆留出足够的安全距离。
<div align=center><img src=./figs/assumptions_car.png width=600 ></div>
 
**关于行人的假设**
  - physics-based assumptions: 行人一般都有一个较低的最大速度，但他们可以非常快速的改变他们的运动方向和速度，这使得仅仅基于物理的方式预测其运动面临较大的挑战，但好在较短时间内行人运动位置的改变其范围有限。
  - maneuver-based assumptions: 行人一般不会与车辆进行直接交互，他们首选都是在人行道或其他行人专用空间内运动。当行人进入可行驶区域内时，他们可能会与其他车辆进行交互，当然首要的还是使用人行横道。即使基于这些假设，但我们仍然要保留他们在未来采用其他动作的多种可能。
  - 最后，行人一般都享有最高级别的路权，必要的时候一般都需要车辆进行停车操作。但粗心的行人可能会在没有警告的情况下徘徊在道路上，但在受到迎面而来的车辆威胁时通常会停下来。这些类型的交互式假设也可以纳入行人的运动预测中。  
<div align=center><img src=./figs/assumptions_pedestrian.png width=600 ></div>


### 4.1.3 定常速度运动预测模型
这里首先介绍一个计算非常高效的、可以同时应用于车辆和行人的运动预测模型。这个模型仅基于一个假设：所有的运动目标都会保持其当前的速度，不管是大小还是方向。
  - 其输入包括预测时间区间 $T$
  - 采样间隔 $dt$
  - 以及当前动态目标的状态：$x_{obj}$
  - 输出为一系列车辆未来时刻的状态 $x_{1:T}$
<div align=center><img src=./figs/constant_velocity_prediction_model.png width=600 ></div>

很显然，这种模型在直线路段预测的很好，但在他其他情况下却经常失败：
  - 此模型只是简单的属于基于物理学假设的类别，但却没有较为全面的考虑车辆的动力学模型；甚至车辆的加减速、以及转向指令都没有考虑
  - 此模型的另一大缺陷是他没有考虑车辆运动会跟随道路形状变化而变化的趋势，比如在弯道中，此模型仍然预测车辆会继续直线前进，这种预测和行为规划明显是不符合的
  - 还有就是这种模型没有考虑道路标志会使车辆速度做出调整。比如停止标志前车辆会减速，离开停止线后车辆会加速
  - 总的原因就是此模型基于的假设太强，并不适用与大多数的动态目标运动


## 4.2 Map-Aware 运动预测
Map-Aware 运动预测通过施加两类比较宽泛的假设来进一步提升（特别是车辆的）运动预测能力。一类是基于位置的假设来提升车辆状态中的位置信息，另一类是基于速度的假设，来提升对速度的预测。
  - 关于位置的假设：
    - 首先是车辆一般都会沿着指定的路线行驶，比如当车辆行驶在弯道上时，其很大概率会沿着道路转弯
    - 其次是关于车道变换和方向变换，可以基于车辆转向灯的状态进行预测，如果感知模块做出了对应的检测，就可以根据此项额外的信息切换预测状态
  - 关于速度的假设：
    - 所有车辆的速度都会被道路几何的影响：当车辆接近弯道时，车辆很大可能会减速来避免超出车辆的横向加速限制
    - 进一步考虑交规元素，对于速度预测能够被进一步加强：比如动态目标近前方有stop sign，便可以假设车辆会执行减速操作。

当然，以上这些假设考虑的还不够全面，但他们充分证明了我们可以用提升预测能力的信息是十分多样的。但同时需要注意的是，在预测模型中加入越多的约束和假设，模型的泛化能力将会变差。

### 4.2.1 提升位置预测
在之前的方法上，我们基于几大假设，具体做出如下改进来进一步对车辆位置的预测：
  - 首先，遇到弯道时，假设车辆会沿着道路通过弯道：可以通过使用地图中的车道中心线来作为车辆的预测轨迹，而不是之前基于定常速度模型做出的直线预测
  - 其次，驾驶员并不总是沿着道路中心线行驶，经常会变换车道：我们可以基于车辆转向灯的状态进行预测（当然，人类驾驶员并不总是在变道时开启转向灯）
  - 最后就是通常并不仅有一条中心线，比如在交叉路口处：T字形路口车辆就面临两种可能：左转或右转，这就需要我们基于当前场景对智能体行为做出多种假设，主要有两种解决思路：
    - 方式1：根据目标车辆的运动状态、外在表现以及跟踪信息做出其最大可能的行为预测，比如给出一个最大可能的行为及其发生概率；
    - 方式2：将方式1扩充为 multi-hypothesis prediction：对车辆的多种可能行为均做出预测，如T路口包含：左转、右转、或保持静止，可以参考的信息有：转向信号、距离左向或右向中心线的位置以及车辆状态等。当然，每种行为的概率很难做出预测，因此可以基于数据进行学习或基于真实世界测试进行工程化调参。
    - 方式2为行为规划器提供了多种可能，但对于下游模块，也需要同时考虑多种场景，好处是如果能够处理的好，这种方式将更为安全，使得执行防御性的驾驶策略成为可能，当后续基于新的信息发生概率转移时，可以快速的切换并进行重规划。
    - 并且方式2这种方法能够很好的容许人类驾驶员的错误，比如忘记打转向灯
  <div align=center><img src=./figs/multi-hypothesis.png width=300 ></div>

 
### 4.2.2 提升速度预测
接下来关注如何基于道路地图提升速度预测：
  - 第一项提升是可以根据道路几何或曲率提前预判车辆会采取的动作：比如车辆在进入急转弯或执行转向时，都会进行减速，因此我们可以使用一个期望的最大横向加速度，通常为 0.5 ~ 1 $m/s^2$, 来提升车辆沿弯道的速度估计
  - 第二项提升是通过结合交规元素提升预测：比如stop signs, yield signs, speed limit changes or traffic lights都可以对速度预测做出帮助。比如基于停车线，车辆的停车位置便可以预测出一个平滑减速过程。

实际上基于高精地图，我们可以对地图上沿着车道线的常规轨迹进行预处理，并基于正常的驾驶行为，每条车道都可以定义多种假设行为。这些预处理得到的信息不仅可以用于自动驾驶车辆指导自己行为和路径规划，也可以为其他智能体的运动预测提供帮助。

### 4.2.3 存在问题
以上我们依靠对动态目标行为做出的两类假设提升了对于车辆位置和速度的预测，但这些假设其实存在很多问题：
  - 首先动态目标并不总是按照我们期望的常规行为方式运动，比如可能并不会精确地沿着中心线行驶、不会按照限速行驶、加速和减速 速率也可能多变
  - 其次，他们可能对我们当前预测系统中还不存在的信息做出反应，比如前方的坑洞或弹起的球；或者有的车辆没有看到交通灯，进而闯红灯；

这些变化都应该在做出多种假设的预测时被纳入考虑。The best approach is therefore to track the evolution of beliefs over the set of hypotheses, and to update based on evidence from the perception stack at every time step. 
 
 
## 4.3 碰撞时间计算
碰撞时间为自动驾驶车辆行为安全提供了一个很好的评估，在基于当前环境，评定可能的机动性时被广泛使用。如果能够提前知道可能发生的碰撞，自动驾驶系统就能够更好的规划出安全的决策行为或者做出紧急避险操作。
  - 为了评估两个动态目标的碰撞时间，我们使用他们的预测轨迹来确定可能发生的碰撞点，如果碰撞点存在，则碰撞时间就是距离碰撞发生所经历的总时间
  - 计算碰撞时间可以分两步完成
    - 首先沿着两动态目标的预测轨迹，确定碰撞点的位置；
    - 然后如果碰撞点存在，计算动态车辆到达该碰撞点的时间
  - 计算碰撞时间的主要挑战
    - 首先在于其严重依赖车辆在未来时刻行驶轨迹的精确预测：包括位置、朝向、速度等
    - 其次同样也需要精准的车辆几何信息：为了计算发生碰撞时精确的碰撞点
  - 显然，上述信息永远无法精确获取，因此我们要把碰撞时间的计算当做一个估计问题，要定时的更新，并且在决策时预留出一些安全缓冲
  - 目前主要有两种方式计算碰撞时间：
    - The simulation-based approach
    - and the estimation-based approach. 
  
### 4.3.1 基于仿真的方法
基于仿真的方法通过仿真驾驶场景中每辆车在未来一段时间内每一时刻的运动状态，包括位置、朝向、占据位置等，然后利用碰撞检测来判断两辆车是否会发生碰撞。注意这里的碰撞检测和静态场景下的碰撞检测不同
  - 静态场景下的碰撞检测需要构建整条路径的线束然后与静态目标的位置进行比较
  - 而这里动态的碰撞检测只需要检测两个目标在同一时刻的几何位置是否会发生碰撞，因为在下一时刻，他们又都会运动到新的位置。

**算法输入**  
  - $D$ - 动态目标列表 （包含他们预测轨迹以及自身车辆的预测轨迹）
  - $dt$ - 仿真的采样间隔
  - $N_c$ - 用于碰撞检测的圆的数量 (除此之外，还要很多用于碰撞检测的其他参数，比如圆之间的间隔距离等)
  - $T$ - 碰撞检测的仿真时间，注意不一定和预测时间相同，但不能超过它
**算法输出**
  - $P_{coll}$ - 碰撞点列表
  - $TTC$ - 与碰撞点相对应的碰撞时间列表

**算法描述** 
<div align=center><img src=./figs/ttc_algorithm.png width=600 ></div>
  
  - 对于车辆状态的预测，我们必须考虑已获取的预测状态的时刻可能和期望计算的仿真时刻并不相同的情况，为了获取在仿真时刻对车辆位置的最佳估计：
    - 我们首先寻找在仿真时刻之前最接近的车辆状态，可以通过遍历整个预测轨迹或通过跟踪在仿真循环中计算的当前预测路径的参考状态获得；
    - 然后以此作为输入，利用剩余的时间间隔来获取当前仿真时刻的具体位置；
  -  关于碰撞点的计算，如下图所示
  <div align=center><img src=./figs/collision_detection.png width=600 ></div>
  
  - 显然在碰撞检测精度和检测效率上二者存在trade-off: 减少圆的数量能够大大降低计算开销，但精度也会降低
     
### 4.3.2 基于估计的方法
基于估计的方法通过计算车辆几何位置沿着预测路径的演进来判断是否会发生碰撞。每辆车都可以估计出一个未来时刻的线束，然后通过计算两条线束的交点，来判断是否有可能发生碰撞：
  - 如果任意两辆或多辆汽车在同一时刻占据同一空间，则可能的碰撞将会被标记
  - 然后利用车辆到达该碰撞点的时间作为碰撞时间的估计

这种方法为了加速计算过程，做了很多简化的假设：
  - 基于目标路径交叉点确定碰撞位置
  - 基于简单的几何图形如bounding box来估计目标的占据空间
  - 基于定常速度来估计到达碰撞点的时间  

### 4.3.3 方案比较
两种方案各有优劣：
  - 基于估计的方法因为进行了很多简化，因此不管是在计算时间还是内存开销上都比较小；反之基于仿真的方法因为每一步都需要进行评估，进行动态目标的碰撞检测，因此计算开销很大；
  - 但因为基于估计的方法简化了问题，使得对于碰撞位置和碰撞时间的估计都不够准确；而基于仿真的方法因为没有基于简化假设，因此只要采样间隔足够小，可以得到更为精确地结果；

上述两种区别使得他们有不同的应用场景：
  - 对于对实时性要求较高的场景，比如在线进行碰撞计算，基于估计的方法更为适用；
  - 但对于离线场景，对计算精度要求较高的场景，比如数据集的创建或基于仿真的测试，基于仿真的方法则更为适用。


## 4.4 补充阅读材料
  - [1] C. Urmson, C. Baker, J. Dolan, P. Rybski, B. Salesky, W. Whittaker, D. Ferguson, and M. Darms, “Autonomous Driving in Traffic: Boss and the Urban Challenge,” AI Magazine, vol. 30, no. 2, p. 17, 2009. This gives an overview of some of the methods used to handle dynamic obstacles in the DARPA Urban Challenge.