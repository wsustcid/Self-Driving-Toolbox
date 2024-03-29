<!--
 * @Author: Shuai Wang
 * @Github: https://github.com/wsustcid
 * @Version: 1.0.0
 * @Date: 2021-09-27 09:10:13
 * @LastEditTime: 2021-11-26 21:01:50
-->
- [Tutorials of Autonomus Driving](#tutorials-of-autonomus-driving)
  - [1. Introduction of Self-Driving Planning Module](#1-introduction-of-self-driving-planning-module)
  - [2. Motion Planning for Self-Driving Cars](#2-motion-planning-for-self-driving-cars)
  - [3. Dig into Apollo](#3-dig-into-apollo)
  - [4. Game Themory](#4-game-themory)


# Tutorials of Autonomus Driving
自动驾驶是一个还在不断发展的领域，因此不仅需要学习无人驾驶相关的知识，机器人学、人工智能等领域的知识也需要持续跟进，只有这样才能不断保持自己的技术先进性。

本部分主要涉及一些理论知识的课程学习，包含课程笔记、课后作业和项目实践等。期望通过理论与实践相结合的方式拓宽理论的宽度和深度，使得最终对整个理论框架有较为全面和系统的了解。


## 1.[ Introduction of Self-Driving Planning Module](Intro-AD/README.md)
本课程主要参考百度无人驾驶书籍和一些技术资料，对整个无人驾驶规划模块(包括路由寻径、行为决策、运动规划)有一个整体上的认识和了解
  - 了解各模块的功能及模块之间的划分逻辑与组织关系
  - 了解各模块主流算法与发展路线

通过本模块，可以对整个无人驾驶规划模块的构成和算法原理有一个初步的了解，但因为涉及具体的算法实现较少，因此一些具体代码细节没有包含，后续还需要参考实际无人驾驶系统代码实现如Apollo等，真正了解各种算法是如何实现、组织与应用的。

## 2. [Motion Planning for Self-Driving Cars](Motion-Planning/README.md)
本课程是多伦多大学自动驾驶汽车系列课程的第四门，主要介绍自动驾驶中主要的规划任务，包括任务规划、行为规划和局部规划。其核心内容包括
  - 构建环境中静态元素的占用网格图，并使用它们进行碰撞检查
  - 使用 Dijkstra 算法和 A* 算法找到图形或道路网络上的最短路径
  - 使用有限状态机选择要执行的安全决策行为
  - 设计最优的、平滑的路径和速度分布使得车辆在遵守交通法规的同时安全地绕过障碍物
  - 最终将实现一个简单的分层运动规划器，并在 CARLA 模拟器一系列场景中实现自主导航，包括避开停在车道上的车辆、跟随领先车辆和安全地穿过十字路口。

通过本课程，将对无人驾驶规划任务的整体流程、理论基础和相关实现细节有一个较好的了解，可以实际动手构建一个简单的完整的规划模块，相较于直接上手实际的无人驾驶系统，如Apollo等，更容易上手和了解整个规划任务的核心流程。完成本教程后，下一步便可以根据自己具体负责的子模块甚至子问题，基于实际的无人驾驶系统进行具体的研究。


## 3. [Dig into Apollo](Dig-into-Apollo/README.md)


## 4. [Game Themory](Game-Theory/README.md)
