---
layout: post
title:  "PBD和XPBD和UE5FullbodyIK"
author: "ZZZZZY"
comments: true
tags: Dev
excerpt_separator: <!--more-->
---
*当前版本比较潦草，仅仅为了记录，后续会陆续修改*

项目中对攀爬等一些全身行为使用了UE的FullbodyIK，而从UE5开始FullbodyIK使用类似于PBD的方式实现，所以文章将会从基础的PBD开始，到XPBD，最后到UE5的FullbodyIK。原则上来说会尽量省去数学步骤，作为普通的开发+数学渣，就把注意力专注于实现的思想上

一个非常好的简短的关于Physis的系列教程：https://matthias-research.github.io/pages/tenMinutePhysics/index.html

<!--more-->

### 1.PBD(Position Base Dynamic)
个人理解物理模拟的核心就是求解物理约束，一般物理引擎在求解约束时，会通过施加力（冲量）来修改速度，然后依赖速度修改系统状态以达到约束要求，本质上就是一个牛顿形式的物理运动学计算（Emmmm这对于力学的朋友来说应该很简单，但对于我这种学渣来说，光是实现一个带摩擦力的球体碰撞模拟就已经100%CPU运转，这里有个坑点，以后会开个坑讲讲篮球游戏的故事）

对于PBD来说，它往往不考虑力的作用，而是直接处理位移X，基于质点位移来定义约束，求解约束后得到新的位置，然后基于新的位置再计算更新速度信息。听起来好像很简单，而且抛开了大量的物理概念，甚至相比传统的基于力的物理模拟还有某些方面的优势，让我们继续前进，先了解总体的算法步骤。

首先做出如下定义：
* Xi 位置信息
* Vi 速度信息
* Mi 质量

算法的流程伪码如下(不想添加图片，所以这里可能会和经典实现步骤有出入)：
```c
// (1) 初始化状态
forall vertices i
	initialize Xi = Xi0, Vi = Vi0, Wi = 1/Mi
endfor
loop:
	// (2) 计算当前速度
	forall vertices i do
		Vi = Vi + DeltaTime * FuncExt(Xi) * Wi
	endfor
	DampVelocities(V1..Vn)	// 处理阻尼
	// (3) 更新位移
	forall vertices i do
		Pi = Xi + DeltaTime * Vi
	endfor
	// (4) 生成碰撞约束
	forall vertices i do
		GenerateCollsionConstraints(Xi -> Pi)
	endfor
	// (5) 迭代求解约束
	loop SolverIterations times
		ProjectConstraints(C1, C2... CN, P1, P2... Pn)
	endloop
    // (6) 基于求解约束后的位置更新速度和实际位置
	forall vertices i do
		Vi = (Pi - Xi) / DeltaTime
		Xi = Pi
	endfor
	// 更新速度，这里可以基于速度来处理摩擦
	VelocityUpdate(V1, V2,.. Vn)
endLoop
```

其中(1)(2)(3)流程上就比较明了我们直接略过，重点关注(4)(5)，先关注约束的迭代求解。这里又回到前文所说的求解约束问题，对于PBD来说，所有对象被视为了质点，我们希望所有的质点移动后，能够基于质点的位置保持稳定。因此我们定义这个质点约束并希望这个约束为0，进行一阶泰勒展开近似有

$$
C(p + \Delta p) \approx C(p) + \nabla p C(p)*\Delta p = 0 \tag{1}
$$

那么DeltaP应该是什么呢？PBD假定DeltaP就是沿着约束C(p)的梯度向量，也就是约束变化的最大方向，在添加一个待定系数标量来限定梯度方向移动的步长，然后这种方式最重要的是保持动量守恒。

$$
\Delta p = \lambda \nabla p C(p) \tag{2}
$$

将公式（2）带入（1）中，可以得到lambda的值：

$$
\lambda = -\frac {C(p)} {\nabla p C(p) *  \nabla p C(p)} = -\frac {C(p)} {|\nabla p C(p)|^2}
$$

再带回（2）得到deltaP

$$
\Delta p =-\frac {C(p)} {|\nabla p C(p)|^2} \nabla p C(p)
$$

然后这只是一个质点对于另一个质点的约束，对于该质点包含的所有约束，基于Newton-Raphson方法得到该质点最终的DealtaP为

$$
\Delta p_i = -s\nabla p_i C(p_1,....p_n)	\tag{3}
$$

其中s为

$$
s = \frac {C(p_1,....p_n)} {\sum_j {|\nabla p_j C(p_1,....p_n)|^2}}
$$

考虑到不同的质量再乘上质量倒数做修正wi = 1 / m，先带入（2），再重现推导（3）最终得到

$$
\Delta p_i = -s w_i\nabla p_i C(p_1,....p_n)	\tag{4}
$$

$$
s = \frac {C(p_1,....p_n)} {\sum_j w_j {|\nabla p_j C(p_1,....p_n)|^2}}
$$

至此就得到在约束条件下实际应该得到的位移，此时再考虑上约束的刚度stiffness，简单来说就是对DeltaP做一个修正直接相乘DeltaP  = DeltaP * K，此时多次迭代n后会得到误差DeltaP*(1 - K)^n，这是一个基于K非线性的结果，为了得到线性关系PBD让DeltaP乘上K' = 1 - (1 - k)^(1/n)，这样能保证线性关系（就不求证了）。



至于约束的构建，则需要考虑许多情况，最基础的约束是一个距离约束，这个约束是基于迭代过程中全局固定的

$$
C(p1, p2) = |p1 - p2| - d = 0
$$

但是还需要考虑各个质点的碰撞约束，这在算法步骤的（4）会进行生成GenerateCollsionConstraints(Xi -> Pi)。碰撞约束是每次外部迭代实时产生的，对Xi前往Pi点打射线，来获取碰撞点qc和碰撞表面的法线Nc，则可以得到约束函数

$$
C(p) = (p - q_c) - n_c
$$

当然这只是简单静态碰撞的情况，但是总是能够构建出合适的一系列约束函数（这是个复杂问题），例如对于布料模拟来说，我们可以构建基于两个点距离的拉伸约束，再构建一个基于4个点（两个平面的弯曲）的弯曲约束等等。

了解PBD思想后，让我们快速从算法细节中跳出，尝试实践一下，下面我们简单构建用PBD构建一个小球碰撞的模拟系统：



总的来说PBD的思想是很简洁精妙的，我们不考虑外部条件（对于这个系统来说），只考虑内部约束，我们从内部通过迭代来让整个系统自我调整，逐渐趋于稳定。总的来说这就很像一个全身的IK系统所需要的状态，全身的IK系统往往也只需要考虑内部的约束，不过在讨论UE的FullbodyIK之前，让我们迅速结束掉PBD，来了解下XPBD

### 2.XPBD

