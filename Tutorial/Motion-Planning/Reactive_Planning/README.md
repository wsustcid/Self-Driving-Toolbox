<!--
 * @Author: Shuai Wang
 * @Github: https://github.com/wsustcid
 * @Version: 1.0.0
 * @Date: 2021-09-27 17:53:22
 * @LastEditTime: 2021-11-26 11:11:55
-->

- [6. 静态环境下反应式导航](#6-静态环境下反应式导航)
  - [6.1 车辆运动学模型](#61-车辆运动学模型)
  - [6.2 碰撞检查](#62-碰撞检查)
    - [6.2.1 基于条带的碰撞检查](#621-基于条带的碰撞检查)
    - [6.2.2 基于圆的碰撞检查](#622-基于圆的碰撞检查)
  - [6.3  Trajectory Rollout Algorithm](#63--trajectory-rollout-algorithm)
    - [6.3.1 轨迹集合生成](#631-轨迹集合生成)
    - [6.3.2 轨迹传播](#632-轨迹传播)
    - [6.3.3 基于条带的碰撞检查](#633-基于条带的碰撞检查)
    - [6.3.4 目标方程](#634-目标方程)
    - [6.3.5 算法示例](#635-算法示例)
  - [6.4 动态窗口](#64-动态窗口)
  - [6.5 补充阅读材料](#65-补充阅读材料)


# 6. 静态环境下反应式导航
反应式运动规划器是指基于机器人周围的局部信息规划出一条无碰撞的路径，使得机器人向着目标点前进。在本节中，我们主要关注静态环境下自动驾驶汽车的行为和路径规划。

## 6.1 车辆运动学模型
**运动学模型与动力学模型**  
典型的考虑摩擦力的粒子动力学模型为
$$ M\ddot{x} + B\dot{x} = F $$
  - 动力学模型使用力和力矩作为输入
  - 同时考虑了运动体的质量和惯性，但代价是模型较为复杂，计算开销大

而粒子运动学模型
$$ \ddot{x} = a $$
  - 忽略了物体质量和惯性对其运动的影响
  - 使用其线速度和角速度（有时也会使用他们的导数）作为模型输入

对于路径规划和轨迹优化问题，我们通常使用运动学模型，以使运动规划问题在计算上更容易处理，而将因对动力学简化引起的问题留给控制器进行解决。

**自行车模型**  
自动驾驶汽车运动学模型可被建模为自行车模型：
$$ \dot{x} = v \cos{\theta} \\ 
   \dot{y} = v \sin{\theta} \\
   \dot{\theta} = \frac{v\tan{\delta}}{L} \\
   \delta_{min} <= \delta <= \delta_{max} \\
   v_{min} <= v <= v_{max} $$
<div align=center><img src=./figs/bicycle_model.png width=400></div>
  
  - 车辆底盘相对于固定坐标系下的位置为$x, y$, 其航向角为 $\theta$, 这三个量构成了车辆运动学模型在状态空间中任意一点的状态
  - 模型在每一点的输入为速度$v$和转向角$\delta$ ($\theta$的产生是由于$v$和$\delta$的共同作用，可以看做绕后轴中心以$L$为半径的圆周运动，由$v=wl$可得到航向角的变换率)
  - 利用此模型，基于控制输入和车辆当前状态，我们便可以计算出车辆轨迹是如何随着时间推移进行演化的

需要注意的是，我们通常没有对车辆状态的直接控制能力，即我们无法直接指定车辆到达具体的$x,y$点，而是通过产生一系列的控制输入，通过车辆运动学模型驱动车辆到达具体的位置。因此，控制输入序列即代表了车辆在未来时刻会跟随的轨迹。

**动力学模型离散化**  
那么具体我们如何根据输入序列来计算对应的轨迹呢？显然，利用上述连续时间的微分方程计算轨迹较为复杂，而使用离散模型则可以较为方便和高效的基于一系列控制输入实现对轨迹的传播生成。我们使用零阶保持器原理来对模型进行离散化：
$$ x_n = \sum_0^{n-1}v_i \cos(\theta_i)\Delta t = x_{n-1} + v_{n-1}\cos(\theta_{n-1})\Delta t \\
   y_n = \sum_0^{n-1}v_i \sin(\theta_i)\Delta t = y_{n-1} + v_{n-1}\sin(\theta_{n-1})\Delta t \\
   \theta_n = \sum_0^{n-1}\frac{v_i \tan(\delta_i)}{L}\Delta t = \theta_{n-1} + v_{n-1}\frac{v_i \tan(\delta_i)}{L}\Delta t  $$

通过此递归式的离散化模型，通过给定控制序列，我们可以得到车辆轨迹的精确近似

**避障**  
上述离散化的运动学模型不仅可以用于轨迹规划，同样可以用于运动预测。如果我们知道了驾驶场景中其他车辆的运动学模型，通过对其可能采取的控制输入进行合理猜测，则基于运动学模型，我们就可以对其未来的运动轨迹进行预测，从而更好的帮助更好的规划自身运动进而避免碰撞。

为了能够实现避障，需要车辆能够拥有更高的机动能力。我们可以通过设计某种steering function来不断地改变steering input，其实这就便是局部路径规划的核心：通过计算出合理的控制输入，来使得车辆能够安全的到达目标点。
<div align=center><img src=./figs/model_trajectory.png width=400></div>

## 6.2 碰撞检查
运动规划模块中的碰撞检查是确保给定轨迹或规划出的路径在物体经过时不会与沿途任何障碍物发生碰撞的过程。然而，碰撞检查是一个存在于很多领域中的非常具有挑战性的计算密集型问题：
  - 保证安全无碰撞的最优路径不仅需要关于环境的完美信息
  - 也需要大量的计算能力来进行精确的计算，特别是对于复杂的形状和具有详细细节的环境

但对于无人驾驶领域中的实时规划问题，以上两点我们都不具备。正如第二章中提到的，给我们提供相关信息的格栅地图是对环境的不完美估计，这也就意味着我们需要在我们的碰撞检查算法中添加一定的缓冲来增加一定的容错能力。

### 6.2.1 基于条带的碰撞检查
具体来讲，碰撞检查相当于沿着给定路径的每个点旋转和平移车辆的足迹(footprint)。足迹中的每个点都会根据路径点的航向角和位置进行对应的旋转和平移。这种方法用集合$S$来表示：
$$ S = \cup_{p\in P} F(x(p), y(p), \theta(p)) $$
  - $P$代表一条路径：有一组点$p$组成
  - $F$是一个函数：返回一组足迹内部的点，这些点分别根据 $\theta, x, y$ 进行了对应的旋转和平移
   
如左图所示，假设我们在$(1.0, -1.0, -\frac{\pi}{2})$处有一路径点，为了得到在该点的车辆足迹，我们需要将车辆足迹旋转$-\frac{\pi}{2}$，并沿着$x,y$方向分别平移$1.0, -1.0$。如果我们对给定路径上的每一点均执行上述操作，便可以得到车辆沿着该路径的条带，表示为所有旋转和平移过的足迹的并集。
  - 通过检查整个点集中是否包含障碍物，如果有，则代表该路径会发生碰撞
  - 否则，则不会发生碰撞

<div align=center><img src=./figs/swath.png width=600></div>

接下来展示条带的具体计算过程。假设我们具有车辆足迹和规划路径在栅格地图上的离散表示，车辆足迹包含$K$个点，则从算法角度，计算条带需要我们旋转和平移车辆足迹内的所有$K$个点，并且在规划轨迹中的每个点都需要重复此过程。
<div align=center><img src=./figs/swath_computation.png width=600></div>

如图(a)所示，假设我们的占据格栅地图分辨率为1m，车辆初始位置位于原点处，在原点处的车辆足迹占据以下三个网格点：$(0,0), (1,0), (2,0)$，我们期望获得经过旋转和平移后在路径点$(1.0, 2.0, \frac{\pi}{2})$处的车辆足迹：
  - 首先，我们将足迹内的每一点相对于原点旋转$\frac{\pi}{2}$,得到新的足迹点为$(0,0), (0,1), (0,2)$，如图(b)所示
  - 然后对每个足迹点进行对应的平移操作，得到$(1,2), (1,3), (1,4)$,如图(c)所示
  - 注意上述步骤的操作顺序，如果我们先平移然后旋转，则会到达错误的位置；
  - 更进一步，为了得到实际的占据栅格索引，对点$x,y$进行对应的偏移，偏移量为占据栅格对应x,y维度的尺寸，然后再除以网格分辨率$\delta$，即可得到对应的网格索引
    $$ x_i = \frac{x(p)+ \frac{X}{2}}{\delta} \\
       y_i = \frac{y(p)+ \frac{Y}{2}}{\delta} $$
  - 最后将所有的点加入集合中，便得到了最终的条带。通过维护一个集合数据结构，保证了条带中没有重复点
    $$ S = S \cup (x_i, y_i) $$

总结：
  - 显然，从上述计算过程可以看出，随着问题规模的增加，条带法所需的计算量也逐步增大，很难用于实时的路径规划
  - 除此之外，当使用精确的汽车足迹计算条带时时十分危险的，因为我们所获得信息是不精准的，并且这里没有任何关于障碍物位置误差的缓冲
  - 但此方法在lattice 规划器中却十分有用，因为在lattice planner中，我们将每一步的控制动作限制在一个很小的动作集合中，因此所需的条带也是一个很小的受限集合，并且，他们的并集可以预先离线计算
  - 在实际进行在线碰撞检查时，该问题就转化为了简单的多个数组查找问题

### 6.2.2 基于圆的碰撞检查
为了同时减轻不完美信息和较大的计算要求带来的挑战，我们使用保守近似的方式来实现碰撞检查，通过牺牲最优性来提升速度和鲁棒性。
  - 我们近似的目标为既能提升算法速度，而又不影响安全性
  - 同时期望能够最小化路径变成次优的程度

所谓保守近似，是指即使不存在碰撞时，也可能报告会可能发生碰撞，但绝对不会出现实际会发生碰撞时而不报告。
  - 具体来讲，我们使用三个重叠的圆来替代车辆的足迹，显然，车辆足迹是三个圆的子集，因此无论何种轨迹，原始足迹产生的条带必然是使用重叠的圆形成的条带的子集
  - 之所以称其为保守近似，是因为当障碍物位于圆内部但位于车辆足迹外部时，也会被判定为碰撞，虽然实际不会发生；但当障碍物位于圆外时，则肯定不会被检测到碰撞，实际也肯定不会发生。
  - This means that the collision checker may contain some false positive collisions, but will not contain a false negative. 
  - 因此，这种近似方式为不完美信息带来的误差提供了一种很好的缓冲
  - 不仅如此，通过圆形近似，其进行碰撞检查时计算起来十分高效：仅需要检查圆心距离障碍物任意一点的距离是否小于此圆半径
  - 并且，如果我们使用占据栅格地图，其可以直接提供每个网格点距离最近目标的距离，这可以通过简单的查表实现，因此在实际进行碰撞检查时将十分高效！

<div align=center><img src=./figs/circle_checking.png width=600></div>

总结：
  - 但保守近似有可能会消除所有的可行的无碰撞路径，即使该路径存在
  - 或者会导致丧失安全通过狭窄通道的能力，这会使得车辆在不该停的地方卡住,或计算出不必要的更为曲折的路线
  - 另外，碰撞检查的精度和离散化的分辨率息息相关，较高的碰撞检查精度要求格栅地图和路径点有较大的分辨率，当然，也也会消耗更多的计算资源。因此，在具体实践时，需要工程师在精度和计算速度之间做好平衡。


## 6.3  Trajectory Rollout Algorithm
总的来讲，trajectory roll-out planner 主要包含以下几个步骤：
  - 首先基于轨迹传播方程在工作空间中产生一系列的候选轨迹集合，使得机器人从当前位置出发可以沿着这些轨迹运动
  - 然后基于机器人周围的局部障碍物信息，判断哪些轨迹是collision-free的，哪些不是
  - 进而通过定义目标方程（一般包含对朝向目标点运动的奖励项），从所有的无碰撞路径中选出一条最优路径
  - 最终通过不断重复上述过程，我们便可以得到一个 receding horizon planner，该规划器基于环境信息，做出对应的反应，进而稳步的朝着目标前进

### 6.3.1 轨迹集合生成
算法的第一步便是在每一个时间步上都要产生一个轨迹集合。对于trajectory roll-out, 每一条轨迹可以对应于对机器人在多个时间步上施加固定的控制输入，并且持续一个定常的时间窗口。因此，为了产生一系列待选轨迹，我们可以对整个可行的控制输入值区间进行均匀采样：
  - 通过对整个输入区间进行密集采样采样，我们可以获得各种各样的待选轨迹，可以提升轨迹搜索的质量和机器人的移动能力，但同时也会带来更多的计算开销，因为每条轨迹后续都需要进行轨迹传播、碰撞检查、评分等步骤
  - 但如果我们仅仅使用小范围的输入区间，虽然可以提升计算效率，但会丢失一些潜在的轨迹，降低规划器生成的轨迹的质量

### 6.3.2 轨迹传播
一旦我们选定了输入集合，我们便可以使用运动学模型，从当前状态开始生成未来状态，进行产生轨迹：
  - 通常我们设定速度为定常值
  - 对转向角在$[-\frac{\pi}{4}, \frac{\pi}{4}]$ 区间进行采样

最终便可以产生一个弧线轨迹集合
<div align=center><img src=./figs/roll_out_trajectory.png width=300></div>


### 6.3.3 基于条带的碰撞检查 
对于碰撞检查算法，假设我们得到了一个车辆工作空间离散化表示的占用网格，这种离散化表示将以矩阵的形式存储，其中矩阵的每个值将表示工作空间中的相应位置是否被占用，在此基础之上，我们便可以使用基于条带的方法执行碰撞检查：
  - 通过沿着待选路径在每一个时间步上旋转平移车辆足迹，将所有足迹取并集，我们得到轨迹的条带。
  - 因为汽车的足迹对应于占用网格中的一组索引，因此沿路径的每个旋转和平移点也将对应于占用网格中的不同索引，这些索引将存储在一组数据结构中以消除重复项。
  - 然后我们可以检查条带的每个点以查看条带的哪些点与占用网格的占用元素重叠：我们通过遍历条带集中的每个点并检查占用网格中的相关索引是否存在障碍来实现
  - 如果条带中的任何一点被占用，则该轨迹包含碰撞，否则，为无碰撞轨迹
  - 通过对上步中所有的轨迹进行碰撞检查，我们便可以得到一组无碰撞的且满足运动学约束可行轨迹，用于后续操作

### 6.3.4 目标方程
对向着目标点运动的轨迹进行奖励是运动规划问题的最终目标。实现本目标的简单方式是在目标方程中添加候选轨迹的终点与目标点之间的距离。然而，在反应式规划器中，我们也鼓励一些其他的行为，来最大化的提升规划器中未来时间步可用的可行路径的灵活性：
  - 最小化距离车道中心线的距离
  - 惩罚路径的曲率
  - 对最大化最近障碍物之间的距离进行奖励

$$ J = \alpha_1 || x_n - x_{goal} || + \alpha_2 \sum_{i=1}^{n} \kappa_i^2  + \alpha_3 \sum_{i=1}^n || x_i - P_{center}(x_i) || $$

当然，目标方程并不是唯一的，需要根据具体应用进行相应的设计和调整，一旦我们选定了目标方程，便可基于目标方程对所有无碰撞路径进行评分，最终选出最优轨迹。


### 6.3.5 算法示例
如图1所示，障碍物用红色表示，目标区域用黄色表示
  - 假设车辆的初始位置为$ (0,0,0) $
  - 转向角的允许范围为 $[\frac{-\pi}{4}, \frac{\pi}{4}]$,采样步长为$\frac{\pi}{8}$
  - 设定定常速度为 $0.5 m/s$
  - 规划器的采样步长为 0.1s, 每一个规划轨迹持续时长为2s

算法具体执行步骤如下：
1. 首先我们在对转向角进行采样，利用 $\frac{-\pi}{4}, \frac{-\pi}{8}, 0, \frac{\pi}{8}, \frac{\pi}{4}$ 产生的待选轨迹如图2所示
2. 接下来基于栅格地图进行碰撞检查，如果条带中的任一足迹点索引出现在占据空间，则被标记为碰撞路径，用红色表示，否则用绿色表示
3. 接下来基于目标方程对可行轨迹进行评分，最优轨迹用黑色表示
4. 至此，便完成了第一次规划迭代
5. 在此位置，我们便拥有了一条车辆可执行轨迹，但在进入下一次规划循环之前，我们不会执行完整条轨迹，而是执行少数的最初的几个点。具体精确的点的数量取决于我们的规划频率和规划时长，这便是滚动优化的思想。
6. 在这里我们规划两秒，执行1秒，黑色为实际执行的部分，橙色为剩余的部分
7. 在下一个规划循环中我们重复上述步骤，直至到达终点。

<div align=center><img src=./figs/roll_out_demo.png width=600></div>

总结：
  - 此算法并不是一次直接规划出一条到达目标的路径，而是贪心的对子路径进行采样，这样做的缺点是规划器会比较短视，因此可能会卡在死角处
  - 并且找到的路径通常是次优的
  - 但这种方法极大地降低了规划问题的复杂度，计算速度较快，能够用于实时规划


## 6.4 动态窗口
动态窗口法允许我们对车辆轨迹施加线加速度和角加速度约束，以提高生成轨迹的舒适性。

回顾我们之前提到的自行车模型，我们并没有对其施高阶的约束，比如 acceleration 或者 jerk,这些高阶的约束项是正是影响乘客舒适性的关键，接下来我们尝试在运动学模型中添加在这些约束项
<div align=center><img src=./figs/bicycle_model_acc_constraint.png width=600></div>

  - 当然，施加这两项约束之后，将限制车辆中的乘客在车辆执行规划轨迹时所承受的力和扭矩的程度，提升舒适性
  - 但同时也限制了车辆的机动性，使得并不能对所有可能的转向角进行完整采样，因为有些转角角加速度过大，同样，在规划迭代过程中我们也不能过快的加速或减速

以角加速度为例，因 $\dot{\theta}=\frac{v\tan{\delta}}{L}$ 和 $|\ddot{\theta}| = |\frac{\dot{\theta_2} - \dot{\theta_1}}{\Delta t} |$, 将两式合并，则约束的具体体现为
$$ |\tan{\delta_2} - \tan{\delta_1}| \leq \frac{ \ddot{\theta}_{max} L \Delta t}{v} $$

具体例子如下：
<div align=center><img src=./figs/example.png width=600></div>

我们也可以基于同样的逻辑对线加速度施加同样的约束。总的来说，动态窗口法允许我们在规划过程中施加更多的约束在轨迹的生成过程中，使得最终生成的运动能够更好同时满足更多的目标。


## 6.5 补充阅读材料
 - [1]. Fox, D.; Burgard, W.; Thrun, S. (1997). "[The dynamic window approach to collision avoidance](https://ieeexplore.ieee.org/document/580977/)". Robotics & Automation Magazine, IEEE. 4 (1): 23–33. doi:10.1109/100.580977. This gives an overview of dynamic windowing and trajectory rollout.
 - [2]. M. Pivtoraiko, R. A. Knepper, and A. Kelly, “[Differentially constrained mobile robot motion planning in state lattices](https://onlinelibrary.wiley.com/doi/abs/10.1002/rob.20285),” Journal of Field Robotics, vol. 26, no. 3, pp. 308–333, 2009.  This paper is a great resource for generating state lattices under kinematic constraints.