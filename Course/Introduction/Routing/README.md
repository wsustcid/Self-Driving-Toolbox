- [2. 路由寻径](#2-路由寻径)
  - [2.1 概述](#21-概述)
  - [2.2 基于图的路径规划算法](#22-基于图的路径规划算法)
    - [2.2.1 DijKstra](#221-dijkstra)
    - [2.2.2 Floyd 算法](#222-floyd-算法)
    - [2.2.3 A*](#223-a)
    - [2.2.4 D*](#224-d)
  - [2.3. 基于采样的路径规划算法](#23-基于采样的路径规划算法)
    - [2.3.1 PRM (Probabilistic Roadmap)](#231-prm-probabilistic-roadmap)
    - [2.3.2 RRT (Rapidly-exploring Random Trees)](#232-rrt-rapidly-exploring-random-trees)

# 2. 路由寻径
## 2.1 概述
全局路径规划可分为静态路径规划和动态路径规划: 
  - 主要以静态道路交通信息为基础的路径规划是静态路径规划
  - 主要以动态交通信息来确定路权大小，以起始点和终止点之间的*交通阻抗力*最小为准则实现最小代价的路径规划称为动态路径规划
    - 交通阻抗根据具体应用的不同，采用不同的标准，如：最少行车时间、最短行车距离、最低通行收费等
    - 距离、时间、收费等信息都可以存储在数字道路地图图层的道路属性中

最终我们把自动驾驶汽车在高精地图(HD-Map)绘制的路网(Road Map)上进行道路(Road)级别的寻径问题，抽象成一个求解在带权有向图上的最短路径问题。
  - 首先，Routing模块会根据Road级别的高精地图，在一定范围内汽车所有可能经过的Road上进行分散“撒点”，这些点被称为 Road Point, 这些点代表了对汽车可能经过的Road上的位置的抽样
  - 然后，这些点与点之间，用有向的加权边进行连接，权值代表了汽车从一个点行驶到另一个点的潜在代价(Cost, 时间、路程、能耗等)
  - 最终，通过常用的A*, DijKstra等路径搜索算法得到从起点到终点的最佳道路形式序列


基于上述定义，全局路径规划算法又可分为**基于图的路径规划算法**和**基于采样的路径规划算法**。路径规划算法的主要发展路线如下图所示：
<div align=center><img src=./figs/planning_timeline.png width=600></div>


## 2.2 基于图的路径规划算法 
### 2.2.1 DijKstra
DijKstra算法在1956年被提出，通过使用类似于广度优先搜索的方法结合贪心思想（同一层节点的访问顺序贪心的选择权重最小的）解决带权图的单源最短路径问题：即寻找从某个起始节点出发，距离图中其他所有节点的最短路径，从而产生一个最短路径树
  - 该算法设置了一个顶点集合$S$ (Closed Set)，放入该集合S中的节点与源节点$s$之间的最短路径均已确定；
  - 该算法不断的从 $D = V-S$ (Open Set) 中选择距离$s$最小的节点$u$加入$S$;
  - 由于$u$的加入，可进一步更新$D$集合中所有相关节点距离$s$的最短距离: $d[v] = min(d[v], d[u]+w(u,v)))$，该步骤称为松弛操作

**算法描述**
<div align=center><img src=./figs/dijkstra.png width=600></div>

**算法实现**  
  - 定义一个Cost/Dist 数组，保存并更新每个节点具体源节点s的最短距离
  - 对于节点集合的划分，可用一个visited标记数组实现，每确定一个节点，便标记为已访问
  - 最后一个关键之处在于每次如何从未访问节点中找出最短距离的节点？
    - 通过遍历的方式寻找同时满足未访问和拥有最短距离的节点 
    - 通过使用二叉堆或斐波那契堆用作优先队列来查找最小顶点来优化寻找速度
  - 确定节点后仅更新与该节点存在边的节点即可
  - 对于边数小于顶点数的稀疏图，可以使用邻接表进一步提升算法的运行效率
  - [代码实现](./src/dijkstra.cpp)

**算法正确性证明**  
  - 为什么每次贪心的把距离源节点最近的节点u加入S之后，他就不用再更新了，就是最终的最优结果？
  - 假设存在一个更优的节点u',使得s->u'->u的距离小于 s->u; 因为权值非负，此时必然有 s->u' 距离小于 s->u的距离，如果这个条件满足，我们当时就不会选择u加入s,而会选择u',因此u'不会存在。

**总结**
  - 该算法不能处理带有负权边的图。
  - 该算法的执行时间和占用空间与图/网中节点数目有关，当节点数目较大时，其算法的时间和空间复杂度急剧增加。
  - 因此无法在大的城市交通网络图中直接应用，很难满足路径规划实时性的要求。
  - BFS和Dijkstra算法的区别：
    - 广度优先是先将未访问的邻居压入队列，再将未访问邻居的未访问过的邻居压入队列再依次访问；依次按步长遍历所有邻居
    - 而Dijkstra是在剩余的所有未访问过的顶点中找出最小的并访问，循环做这个是直到所有点都被访问完
  

### 2.2.2 Floyd 算法
Floyd算法求解所有任意两点间的最短路径(All Pairs Shortest Paths)，可以正确处理有向图或负权(但不可存在负权回路)的最短路径问题。该算法的核心是Dijkstra中的松弛操作结合动态规划的思想：
  - 我们的目标是求解任意两个节点i,j之间的最短路径，其中间可能没有中继节点，也可能最多包含所有节点
  - 因此我们可以定义一个三维的dp数组 $dp[k][i][j]$ 表示从节点$i$到节点$j$，最多使用$1$~$k$中的任意节点作为中继节点形成一条最短路径
  - 初始状态$k=0$, 代表没有中继节点，$dp[0][i][j] = wij$，也就是邻接矩阵：对角线元素为0，有边便是权重，无边无穷大
  - 当有中继节点时，考虑$dp[k][i][j]$如何从$dp[k-1][i][j]$进行状态转移而来，相比于之前的状态，我们新加入一个节点$k$可以新用作中继节点(具体这个中继节点的位置在哪里我们不做考虑)，我们此时面临两种选择：
    - 不使用节点$k$作为中继节点，则 $dp[k][i][j] = dp[k-1][i][j]$;
    - 使用节点$k$作为某个中继节点，则 $dp[k][i][j] = dp[k-1][i][k] + dp[k-1][k][j]$ 
  - 显然上述求解过程包含了子问题，符合动态规划的求解过程，我们只需要从$k=0$开始不断迭代增加$k$的个数即可
  - 最终我们要求的结果就是节点$1$~$N$都可以作为中继节点时的最短路径，即迭代$k=1... N$;

比较容易产生疑惑的是我新加入节点为何就直接和i相连，没有考虑节点顺序具体是如何实现找到具体某条最短路径的？这个需要具体手动执行一遍才能想清楚，一定要注意dp[2][3]并不代表路径2->3 而是代表从2到3的最短路径的距离，他中间可能有中继节点1，因此如果我在2->4之间加入中继3可是实现松弛，但其实最终路径是2->1->3->4;

注意：
  - 由于时间复杂度依然较高，因此同样不适用计算大量数据

**算法描述**
<div align=center><img src=./figs/floyd.png width=600></div>

  - 由于dp的迭代仅和上一时刻k有关，因此可以不存储维度k，直接在原始数组上进行迭代更新即可
  - 路径恢复（如2->1->3->4）可以通过定义一个next[i][j]数组，存储从节点i到j的最短路径最后加入的中继节点
  - 比如 next[2][4] = next[2][3], 然后next[2][3] = next[2][1] = 1; 其实保存的就是第一个中继节点
  - 这样我们就可以不断溯源，下一次在查找next[1][4] = next[1][3]=3;
  - next[3][4]=4
  - 直到找到next[4][4]返回最终路径

**算法实现**
  - [代码实现](src/floyd.cpp)


### 2.2.3 A*
传统的遍历搜索过程如BFS，Dijkstra等具有盲目性，效率较低，当节点数量过多时，可能无法在有限时间内搜索到目标点，这时就需要用到启发式搜索。启发式搜索就是在状态空间的搜索过程中加入与目标节点有关的启发式信息，引导搜索朝着最优的方向前进。

A*算法就是一种 informed search algorithm，是在DijKstra算法基础上建立的启发式搜索算法，多应用于实现道路网的最佳优先搜索。其主要思想为：
  - 在选择下一个访问节点时，不再单纯的选择距离起始节点最近的节点，而是通过引入多种有用的路网信息，计算当前候选节点与目标节点之间的某种目标函数，如最短行车距离、最短行车时间、最少行车费用等，以此目标函数值为标准来评价候选节点是否为最优路径应该选择的节点；
  - 因此该算法的核心是确定如下形式的启发式估价函数： $f'(n) = g(n) + h'(n)$， 其中，$g(n)$是从起点$s$到候选节点$n$的实际代价； $h'(n)$是候选节点$n$到目标点$D$的估计代价。
 
**算法描述**
```c++
 function A_Star(start, goal)
     closedSet := the empty set //已经被访问过的节点集合：已确定了最小cost
     openSet := set containing the initial node //待访问节点集合：通常用优先队列实现对f值排序
     cameFrom := empty map // 路径记录集合：cameFrom[n]记录从start到n的最优路径中n的上一个节点
     // 初始化: 从起点开始
     g_score[start] := 0 // g(n)
     h_score[start] := heuristicFun(start, goal) // h(n)
     f_score[start] := h_score[start] // f(n)=h(n)+g(n)

     while openSet is not empty
         // 从优先队列中取出具有最优f值的节点进行访问
         x := the node in openSet having the lowest f_score[] value
         remove x from openSet
         add x to closedSet  // 将 x加入已访问节点
         // 到达终点，返回路径
         if x = goal  
             return reconstruct_path(cameFrom, goal)
         // 遍历所有x的邻接节点，尝试进行松弛操作
         for each y in neighbor_nodes(x)
             // 若已确定，则跳过，选取下一个邻居节点
             if y in closedSet
                 continue
             
             if y not in openSet 
                 is_better := true // 若不在待访问集合中，权重为无穷大，必然可以松弛 
             else if g_score[x] + dist_between(x,y) < g_score[y] 
                 is_better := true //可以进一步松弛
             else
                 tentative_is_better := false // 无法松弛
             if is_better = true    
                 cameFrom[y] := x // 来源节点，但后续也许会被进一步更新
                 g_score[y] := g_score[x] + dist_between(x,y)
                 h_score[y] := heuristicFun(y, goal)
                 f_score[y] := g_score[y] + h_score[y]
                 add y to openSet // 仅将未访问的可以松弛的点加入待访问节点

     return failure
 
 function reconstruct_path(cameFrom,current_node)
     // 从后向前回溯，恢复路径
     if cameFrom[current_node] is set
         p = reconstruct_path(cameFrom, cameFrom[current_node])
         return (p + current_node)
     else
         return current_node
```

### 2.2.4 D*
D* (Dynamic A*) 适用于对周围环境未知或周围环境存在动态变化的场景，是一种增量式的启发式搜索算法：
  - 同A* 算法类似，D* 也是通过维护一个优先队列来确定节点的访问扩展顺序
  - 但D* 是从终点开始向起始节点进行遍历，直到起始节点出队
  - 另外，该算法给每个节点定义了不同的状态，当中间某节点状态发生改变时，会在该点处重新进行寻路，因此可以处理环境的动态变化，且由于D*存储了空间中每个点到终点的最短路径信息，因此在重新规划时可以大大加快搜索速度

该算法有多种变种，包括 
  - original D*
  - Focused D* (A* + D*) 
  - D* Lite (A* + Dynamic SWSF-FP)

**参考资料**
  - [D*](http://web.mit.edu/16.412j/www/html/papers/original_dstar_icra94.pdf)
  - [D* Lite](http://idm-lab.org/bib/abstracts/papers/aaai02b.pdf)
  - [Robotic Motion Planning: A* and D* Search](https://www.cs.cmu.edu/~motionplanning/lecture/AppH-astar-dstar_howie.pdf)


## 2.3. 基于采样的路径规划算法
基于图搜索的方法搜索效率严重依赖于节点的数量，当维度较高或节点数量增多时，其算法复杂度也会急剧增长。基于采样的运动规划通过在地图空间进行概率采样，减少点的搜索数量，可大大提升搜索效率。

典型的基于采样的路径规划方法分为综合查询方法和单一查询方法两类：
  - 综合查询：首先通过构建采样和碰撞检测建立完整无向图，得到构型空间的完整连接；然后通过图搜索得到可行路径；
  - 单一查询：从起始位置出发，建立局部路线图，然后逐步向目标区域延伸构建路径树，直到到达目标点

### 2.3.1 PRM (Probabilistic Roadmap)
PRM就是一种综合查询的方法，分为构图阶段和查询阶段。
  - 在构图阶段，通过在自由空间随机采样路标点，然后以特定规则连接各路标点，从而形成一个随机的网络，这个网络被称为概率地图；
  - 在查询阶段：可以通过图搜索算法求出连接起点到终点的最优轨迹；

**算法描述**
  1. 初始化：定义$G(V,E)$为无向图，其中顶点集合V代表无碰撞构型，边集$E$代表无碰撞路径，初始状态为空；
  2. 构型采样：从构型空间中采样生成一定数量的无碰撞的点 $\alpha(i)$并加入到顶点集V中；
  3. 邻域计算：定义距离$rad$, 对于已经存在于顶点集$V$中的点，如果他与$\alpha(i)$的距离小于$rad$,则将其称为点$\alpha(i)$的邻域点；
  4. 边线连接：将点$\alpha(i)$与其邻域点相连，生成连线e;
  5. 碰撞检测：检查连线e是否与障碍物发生碰撞，如果无碰撞，则将其加入到连线集E中
  6. 上述步骤只是生成了无碰撞的构型图，实际找到最短路径还需要结合A*等搜索算法进行路径搜索；
<div align=center><img src=./figs/prm_demo.png width=600></div>

注意：
  - 采样点的数量设置越多，找到更优路径的概率也就越大，当然也会增加搜索时间；
  - 邻域设置过小，边线较少可能找不到可行解；反之设置过大会生成较多距离较远点之间的连线，增加搜索时间

### 2.3.2 RRT (Rapidly-exploring Random Trees)
RRT是一种单一查询方法，从起点开始向外扩展树状结构，其扩展方向是通过在规划空间内随机采样决定的。

**算法描述**
```c++
Qgoal // 定义代表搜索成功的目标区域
Counter = 0 // 迭代次数
lim = n // 迭代终止条件
G(V,E) // 待生成的包含节点的边的图，初始为空

While counter < lim:
    Xnew  = RandomPosition() // 随机生成一个节点
    if IsInObstacle(Xnew) == True: // 检测节点有效性，避开障碍
        continue
    Xnearest = Nearest(G(V,E),Xnew) // 在图中寻找其最近节点
    Link = Chain(Xnew,Xnearest) // 建立连接
    G.append(Link)
    // 到达终点区域时建图完成
    if Xnew in Qgoal:
        Return G
Return G
// 因为生成的路径含有较多分支，因此最后也需要搜索算法在搜索树中寻找一条连接起点和终点的最短路径
```
<div align=center><img src=./figs/rrt_demo.png width=600></div>

注意：
  - 为了让该算法更为实用，实际实现时通常会定义一个growth factor 来限制新产生的节点和树之间连接的长度，如果Xnew与Xnearest的距离超过了此限制，那么就在他们之间的连线上最大距离的位置重新产生一个点建立连接。相当于随机点控制路径树生长的方向，growth factor控制生长的速率。
  - 并且在产生随机点的过程中，还可以增加其向特定区域如终点区域生成点的概率，贪心的引导树向着终点进行生长。

