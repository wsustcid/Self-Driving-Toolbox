- [2. 运动规划建图](#2-运动规划建图)
  - [2.1 占据栅格地图](#21-占据栅格地图)
  - [2.2 从雷达数据中生成占据栅格地图](#22-从雷达数据中生成占据栅格地图)
    - [2.2.1 利用贝叶斯对数几率进行概率更新](#221-利用贝叶斯对数几率进行概率更新)
    - [2.2.2 逆测量模型(Inverse Measurement Model)](#222-逆测量模型inverse-measurement-model)
    - [2.2.3 Ray Tracing](#223-ray-tracing)
  - [2.3 无人驾驶汽车占据栅格地图更新](#23-无人驾驶汽车占据栅格地图更新)
    - [2.3.1 点云数据预处理](#231-点云数据预处理)
    - [2.3.2 3D点云向2D平面进行投影](#232-3d点云向2d平面进行投影)
  - [2.4 高精地图 (High Definition Road Map)](#24-高精地图-high-definition-road-map)
    - [2.4.1 Lanelet 地图](#241-lanelet-地图)
  - [2.5 补充阅读材料](#25-补充阅读材料)

# 2. 运动规划建图
## 2.1 占据栅格地图
占据格栅是对无人驾驶车辆周围环境进行离散化操作之后得到的结果，这种离散化操作可以应用在二维或三维空间中。
  - 占据格栅中的每个网格代表该位置是否被静态目标占据
  - 这些静态的目标包括树木、建筑物、道路标志以及灯柱等
  - 另外，一些我们不认为是障碍物的目标在无人驾驶问题中也会被分类为被占据空间，主要包括所有汽车不可通行的区域，比如草坪或人行道等
  - 占据格栅中的每个网格用一个二进制的值$m^i \in \{0,1\}$来表示，1代表该位置被静态目标占据，反之0代表没有被占据

**格栅地图假设**
  - 静态环境：在创建地图的过程中，环境中动态的或正在移动的目标都将被从传感器数据中移除
  - 网格之间独立：主要为了简化后续网格更新方程
  - 车辆状态已知：在任意时刻，车辆相对于栅格地图的状态（位置、航向等）都是已知的

**LiDAR数据预处理**  
为了感知车辆之间的距离和周围环境的状态，激光雷达是最常用的传感器。但在将雷达数据用于格栅地图创建之前，雷达数据中的部分数据需要被过滤掉：
  - 首先，雷达数据中所有组成地面的点都需要被移除：因为车辆可以安全的在道路平面上行驶，因此不应该被当做障碍物反射点
  - 其次，出现在车辆最高点之上的点也都需要被移除：因为这部分点云并不会阻碍车辆的正常行驶
  - 最后，点云数据中所有的动态目标需要被移除，包括车辆、行人、自行车，甚至动物等
  
经过上述预处理步骤，3D的点云将会被投影到2D平面上，用于栅格地图的构建。

**概率占据栅格**  
为了处理环境和传感器噪声，在实际使用中，占据栅格通常被修改为概率栅格地图：
  - 每个网格不再存储是否被占据的二进制值，而是存储该网格被占据的概率
  - 因此，该占据格栅地图可以被表示为置信地图
    $$ bel_t (m^i) = p(m^i | (y,x)) $$
  - 其中$m^i$代表某个独立的网格，可以由y,x计算得到；y,x是传感器的测量值（遇到障碍物被反射回来的二维坐标）
  - 最后为了再次转化为二进制地图，可以通过定义一个置信度阈值，高于此阈值为被占据，反之是free状态

**占据栅格贝叶斯更新**  
为了获得精度更高的置信度地图，我们期望能够结合时刻1到t的多组测量值来对地图进行更新，以增加其可信度。具体的，我们可以使用递归的方式来更新置信度：
  - 网格$m^i$在t时刻的置信度被定义为给定时刻1-t所有的测量值及车辆位置，该网格被占据的概率：
    $$ bel_t(m^i) = p(m^i | (y,x)_{1:t}) $$
  - 进一步的，可以使用贝叶斯公式$P(A|B)=P(B|A)*P(A)$来将多个测量值合并到一个置信地图中：
     $$ bel_t(m^i) = \eta p(y_t |m^i) bel_{t-1}(m^i) $$
  - 右侧第一项为测量模型：代表给定一个被占据的网格$m^i$，得到某个特定测量值的概率
  - 右侧第二项是先验概率：代表该网格在之前时刻的置信度，在之前的更新中已经被计算出来并存储于概率地图中
  - $\eta$是一个归一化系数，使最后计算出的结果满足概率分布


## 2.2 从雷达数据中生成占据栅格地图
上一小节中提到了使用贝叶斯公式来实现对概率栅格地图的更新方式：
  $$ bel_t(m^i) = \eta p(y_t |m^i) bel_{t-1}(m^i) $$
但这种方式却存在潜在的问题！比如对于未被占据的网格，当先验概率为0.000638，此时根据测量模型获得概率值为0.000012时，t时刻更新得到的后验概率为0.000000008。当计算处理较小的浮点数时，经常会产生精度误差，且多次进行浮点数运算，计算效率将会十分低下。

为了解决这一问题，我们可以采用对数几率的方式对概率值进行重新表示
  $$logit(p) = log(\frac{p}{1-p})$$
这样就可以将原始的概率值$[0,1]$区间映射到整个实数区间 $[-\infty, \infty]$。当然我们可以如下公式，利用对数几率再重新恢复原始概率值：
  $$p = \frac{e^{logit(p)}}{1+e^{logit(p)}}$$

### 2.2.1 利用贝叶斯对数几率进行概率更新
  1. 首先，我要要处理的对象为: 给定1-t时刻的测量值，求出栅格$m^i$被占据的概率
     $$ p(m^i | y_{1:t}) $$
  2. 接下来我们要做的第一件事就是期望能够把t时刻的测量值从$y_{1:t}$中剥离出来，这样每次在进行概率更新时，只需依靠最新的测量值，而不用每次都存储并是使用历史所有时刻的测量值(实际上这也是不可能的)，我们可以使用贝叶斯公式：
     $$ p(m^i | y_{1:t}) = \frac{p(m^i, y_{1:t})}{p(y_{1:t})} = \frac{p(y_t|m^i, y_{1:t-1}) * p(m^i | y_{1:t-1}) * p(y_{1:t-1})}{p(y_t|y_{1:t-1})*p(y_{1:t-1})} $$
  3. 基于马尔科夫假设：当$m^i$已知时，当前时刻的测量值与之前时刻的测量无关，则可以进一步将上式化简为：
     $$ p(m^i | y_{1:t}) = \frac{p(y_t|m^i) p(m^i | y_{1:t-1})}{p(y_t|y_{1:t-1})} $$
  4. 对分子中的测量模型再次使用贝叶斯公式，得到
     $$ p(m^i | y_{1:t}) = \frac{p(m^i|y_t)p(y_t) * p(m^i | y_{1:t-1})} {p(m^i) * p(y_t|y_{1:t-1})} $$
  5. 接下来我们求1-p,即栅格$m^i$未被占据的概率，记为 $p(-m^i)$,则有上式可直接获得
     $$ p(-m^i | y_{1:t}) = \frac{p(-m^i|y_t)p(y_t) * p(-m^i | y_{1:t-1})}{p(-m^i) * p(y_t|y_{1:t-1})} $$
  6. 联立4,5两式，则有
     $$ \frac{p(m^i | y_{1:t})}{p(-m^i | y_{1:t}) } = \frac{p(m^i|y_t)p(y_t)p(m^i | y_{1:t-1})} {p(m^i)p(y_t|y_{1:t-1})} * \frac{p(-m^i)p(y_t|y_{1:t-1})} {p(-m^i|y_t)p(y_t)p(-m^i | y_{1:t-1})} $$
     $$\frac{p(m^i | y_{1:t})}{p(-m^i | y_{1:t})} = \frac{p(m^i|y_t) * p(m^i | y_{1:t-1}) * p(-m^i)} {p(-m^i|y_t)*p(-m^i | y_{1:t-1})*p(m^i)}$$
  7. 将 $p(-m^i)$ 重新恢复为 $1-p(m^i)$的形式，并对等式左右两边同时取log，由$logit(p)=log(\frac{p}{1-p})$$, 则可得到对数几率的形式：
     $$ logit(p(m^i|y_{1:t})) = logit(p(m^i|y_t)) + logit(p(m^i|y_{1:t-1}) - logit(p(m^i))) $$
  8. 则最终可以得到贝叶斯对数几率更新模型：
     $$ l_{t,i} = logit(p(m^i|y_t)) + l_{t-1, i} + l_{0,i} $$
  9. 上式表明，在t时刻，栅格i被占据的概率 = t时刻逆测量模型结果 + 上一时刻的置信度 - 初始时刻的置信度；其中，初始时刻的置信度通常被设置为50%的均匀分布，即我们没有任何先验信息。

### 2.2.2 逆测量模型(Inverse Measurement Model)
本节我们将定义逆测量模型的概念，并展示如何从雷达传感器数据中构建一个简单的逆测量模型。逆测量模型是指给定测量值$y_t$，位置$m^i$的网格被占据的概率，而传统的测量模型是指给定网格位置，在该位置上获得某个测量值的概率。因此，我们的任务是将测量模型反转，从而构建逆测量模型。

在本课程中，我们将激光雷达数据当做二维的Range data来处理（加上elevation angle就可以上升至三维： 
  - 扫描方位角：其均匀的分布在360度的方位上
      $$ \Phi^s = [-\Phi^s_{max}, ..., \Phi^s_{max}], \Phi^s_j \in \Phi^s $$
  - 扫描距离：从传感器到遇到障碍物反射回来的（大多数雷达都会包含no echo signal）
  $$ r^s = [r^s_1, ... , r^s_j], r^s_j \in [0, r^s_{max}] $$

  1. 首先，我们定义传感器相对于占据栅格地图坐标系的位置为$x_{1,t}, x_{2,t}$, 朝向为$x_{3,t}$
  2. 其次，基于雷达数据，我们将测量地图划分为三种不同的区域
     - No Information: 即激光雷达射线无法到达的区域，因此无法提供任何信息
     - Low Probability: 即射线正常通过没有遇到任何障碍物的区域
     - High Probability: 即射线与障碍物接触后发生反射的区域(非最大反射距离)
   
   <div align=center><img src=./figs/imm_1.png width=400 ></div>

  3. 为了将这些区域映射到测量地图上，我们需要为每个网格定义其关于车辆的相对距离(Relative Range)和相对方位(Relative Bearing), 以及一个Cloest Relative Bearing。基于这两个指标，对于每个网格，我们通过寻找相对距离和相对方位与测量值最接近的网格来确定该测量值所属的网格：
  $$ r^i = \sqrt{(m^i_x - x_{1,t})^2 + (m^i_y - x_{2,t})^2} $$
  $$ \Phi^i = tan^{-1}(\frac{m^i_y - x_{2,t}}{m^i_x - x_{1,t}}) - x_{3,t} $$
  $$ \kappa = argmin(|\Phi^i - \Phi^s_j|) $$
  <div align=center><img src=./figs/imm_2.png width=400 ></div>
  
  4. 接下来我们再定义两个参数$\alpha$和$\beta$来确定每条射线遇到障碍物发生反射的一个扇形区域，即根据某个反射位置，确定具体哪些网格应该被占据。其中 (a). $\alpha$控制发生反射的距离范围，在其范围内的网格被分配High probability (b). $\beta$ 控制发生反射的角度范围，在其范围内的网格分配low或high probability。具体更新规则如下：
  - No Information (0.5):
  $$ if \quad r^i > min(r^s_{max}, r^s_k+\alpha/2) \quad or \quad |\Phi^i-\Phi^s_k|> \beta/2 $$
  - High Probability (>0.5):
  $$ if \quad r^s_k < r^s_{max} \quad and \quad |r^i - r^s_k|<\alpha/2 $$
  - Low Probability(<0.5):
  $$ if \quad r^i < r^s_k $$
  <div align=center><img src=./figs/imm_3.png width=400 ></div>

  5. 以上便是根据测量值确定每个栅格被占据概率的逆测量模型。
   
### 2.2.3 Ray Tracing
但是，上节提到的逆测量模型更新方式计算起来却过于麻烦，每次都要逐个网格进行更新，并且要确认测量值所属的网格包含大量的浮点数运算，这些需要较大的计算开销。因此，为了提升更新效率，我们可以使用**ray tracing algorithm**，比如 Bresenham's line algorithm，仅沿着激光雷达射线射出的方向进行更新，这样可大大减少需要更新的范围，并且仅需要依据整数加减和位移操作。

  <div align=center><img src=./figs/ray_tracing.png width=400 ></div>

## 2.3 无人驾驶汽车占据栅格地图更新
### 2.3.1 点云数据预处理
在将点云数据用于产生占据栅格地图之前，需要采用几种典型的filtering方法对点云数据进行预处理，主要包括：
  1. 对雷达点云数据进行下采样以满足实时性的要求
  2. 移除位于车辆上方的点云（此类目标并不影响驾驶）
  3. 移除可行驶区域的点或地面点（此类数据同样不影响驾驶 - 依赖感知模块处理结果）
  4. 移除正在运动中的动态目标（如车辆、行人等 - 同样依赖感知模块处理结果获取这些目标的位置信息）

**点云下采样**   
点云的下采样是指通过移除或忽略点云中冗余的点来达到减少点云中点的数量。通常自动驾驶中的激光雷达每秒钟可以产生120万个点，其提供了对车辆周围目标丰富的几何描述，但对于生成占据栅格地图，对某个障碍物目标位置的描述其实仅需要极少量的点即可，因此存在大量的冗余点。更重要的是，同时处理全部的点在计算上是无法实现的，因此我们需要移除*冗余点*来提升计算效率：
  - 最简单的方法是使用systemaic filter：沿着雷达的scan ring每隔n个点保留一次
  - 也可以在range image中使用图像的下采样方法：通过在3D网格中进行空间搜索，将一簇点云用一个单独的占据格栅测量代替

上述两种方法在开源的点云库PCL和计算机视觉库OpenCV中都已经有具体实现

**车辆上方与地面点云的移除**  
首先，车辆上方点云的移除非常简单，直接移除固定高度（如2.4m）以上的点即可。但这种操作基于的假设是地面是平的，所以具体使用时要小心，同时还要考虑悬垂的树木、电线、桥梁和标志等信息。但地面点云的移除却由于其复杂的构成较难精准的移除：
  - 首先是不同的道路几何形状带来的难度：包括用于排水的变化的凹度、不同的slop和bank angle、曲率等
  - 其次是路沿在不同的位置有不同的高度，并且道路边界在点云数据中并不总是能够清楚的定义，这些变化有可能导致部分道路边缘或不可驾驶区域作为地面被移除
  - 最后是地面上的小目标：我们并不希望它也被移除 （如足球、乌龟等）

解决方案之一是基于视觉的方法利用语义分割来对地面进行分割，然后映射到点云中将对应的点进行移除，同时保留不可行驶区域中的点。

**动态目标移除**  
通过感知模块对场景中的动态目标进行识别与跟踪，可以获得动态目标在点云中的3D的bounding box，从而可以移除其范围的点，同时对于bounding box的size通常也会增加一个小的阈值，来消除感知模块中对物体位置估计的误差，增加滤波器的鲁棒性。但仍然有很多难点需要考虑：
  - 并不是所有的车辆都是运动的：一些静态的车辆，比如停着的车辆也应该是grid map的一部分，不应该被移除： 可以使用目标的历史位置来区分动态与静态目标
  - 由于感知模块检测到目标之后也需要计算，因此传到下游时会有部分时延，此时动态目标的点云位置已经发生了变化，因此会丢失部分动态目标点云信息
  - 因此我们可以通过对目标跟踪进行轨迹预测，在实际移除时对bounding box沿着预测轨迹进行移动，将更多的可能构成动态目标的点都进行移除

### 2.3.2 3D点云向2D平面进行投影
在将点云进行预处理之后，接下来就可以将3D点云投影到2D平面上：
  - 最简单的方式是直接将3D点云中z轴的值置为0，这样就将三维点云压缩到了二维平面上
  - 然后将此二维扫描平面进行和占据格栅地图同样方式的划分
  - 对于每个网格，统计此网格内点的数量，以此作为该网格occupancy belief的测量; (点越多，代表该网格被静态目标占据的概率越大)


## 2.4 高精地图 (High Definition Road Map)
高精地图与传统的Paper-based maps 或 online maps类似，但高精地图中包含更多细节信息：
  - 传统地图只是存储道路的大概位置，但高精地图存储的确是精确位置，包括所有的车道线，精度可达厘米级
  - 更进一步的，所有可能会影响车辆驾驶的道路标志或者信号也都会保存在高精地图中

因此需要一个高效的方法来存储来将所有这些细节都保存在高精地图中。

### 2.4.1 Lanelet 地图  
Lanelet的概念于2014年被 Philip Bender 在论文"Lanelets: Efficient Map Representation for Autonomous Driving" 中被首次提出。由于这种方法在存储和传播方面的高效性，目前已被广泛使用。并且其在我们的行为规划阶段也将扮演重要的角色。Lanelet地图主要包含两个组成部分：
  - Lanelet Element：其存储了用于连接道路车道中一小段纵向线段的所有信息
  - Intersection Element: 存储了一个单独的交叉路口中所包含的所有Lanelet Elements, 通常也包含这些Lanelet elements间的连接关系，可用于运动规划任务中的简单检索。  
  
**Lanelet Element**  
一个Lanelet Element 保存了一小段道路的所有信息，主要包含以下信息
  - 给定车道的左右边界
  - 交规元素，比如stop line 或static sign line （一般展现在一段lanelet element的末尾）
  - 交规属性，所有可能影响道路段的属性，比如 速度限制；
  - 和其他lanelet element的连接关系 （方便进行遍历遍历搜索）

每个lanelet element都以一个交规元素或交规属性的改变作为结尾。因为有的element可能非常短，只有几米，比如交叉口的一部分；有的却可以很长，有几百米，比如高速公路的一段路线。  

车道边界用一些点的集合进行表示，用来创建一个连续的折线段：
  - 点的位置为gps坐标的x,y,z值，点和点之间可能只有几厘米，也可能有几米，这个由折线的smoothness决定
  - 点的顺序定义了遍历或前进的方向
  - 道路的曲率也可以被提前计算
  - 车道的中心线也可以通过插值来拟合得到，用作车辆在本车道的期望路径
  
这里有两种类型的lanelet regulations: 出现在lanelet末尾的Regulation Elements 和影响整个lanelet的regulation attributes：
  - Regulatory elements: 通常用线表示，由一组共线点定义。在Regulation elements处通常需要车辆采取相应的动作或做出决策，比如stop line, Traffice lights line, Pedestrian crossings
  - Regulatory attributes: 在整个lanelet中持续存在，比如速度限制，或这个lanelet在十字路口或并道处是否穿过其他lanelet.

**Lanelet Connectivity**  
每个lanelet都有四种可能的联通性：Lanelets directly to the left, lanelets directly to the right, lanelet preceding it, and lanelet following it. 整个lanelet结构是连接在一个有向图中的，是高精地图的基础结构。
<div align=center><img src=./figs/lanelet_connectivity.png width=600 ></div>

**Advantages**  
The intersection elements simply holds a pointer to all regulatory elements, which make up the intersection of interest. All lanelet elements which are part of an intersection also point to this intersection element. This structure acts much like a container and simplifies look-ups throughout the lanelet structure when assigning behaviors. 
  - 这种数据结构简化了运动规划的流程，使得计算起来更加高效
  - 在进行复杂多车道路网的路径规划时，为了规划转弯，需要多个连接关系的变化，因为每个lanelet可以当做单独的节点，这种数据结构使得这种规划成为了可能
  - 并且，可以实现在已知地图中定位动态目标，提升预测能力
  - 除此之外，亦可以提升车辆在交叉路口和其他动态目标的交互能力

**地图创建**  
这种地图的创建方式可大致分为三种方式：
  - 离线创建：通过驾驶汽车在路网中行驶几次来收集数据，然后进行融合和定位；好处是可以进行手动纠正
  - 在线创建：在驾驶汽车首次通过某个路网时，同时进行地图创建。（计算量要求较大，并且误差较大，较少使用）
  - 离线创建在线更新：首先通过离线创建，在检测到变化时进行在线更新


## 2.5 补充阅读材料
  - [1]. S. Thrun, W. Burgard, and D. Fox, Probabilistic robotics. Cambridge, MA: MIT Press, 2010. Read Chapter 9 - Occupancy Grid Mapping for an overview of how occupancy grids are generated.
  - [2]. P. Bender, J. Ziegler, and C. Stiller, “Lanelets: Efficient map representation for autonomous driving,” 2014 IEEE Intelligent Vehicles Symposium Proceedings, 2014. Introduces the concepts of lanelets used in mapping.