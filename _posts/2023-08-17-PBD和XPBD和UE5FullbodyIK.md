---
layout: post
title:  "PBD和XPBD和UE5FullbodyIK"
author: "ZZZZZY"
comments: true
tags: Dev
excerpt_separator: <!--more-->
---
*当前版本比较潦草，仅仅为了记录，后续会陆续修改*

项目中对攀爬等一些全身行为使用了UE的FullbodyIK，而从UE5开始FullbodyIK使用类似于PBD的方式实现，所以文章将会从基础的PBD开始，到XPBD，最后到UE5的FullbodyIK。

原则上来说会尽量省去数学步骤，作为普通的开发+数学渣，就把注意力专注于实现的思想上

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
```
forall vertices i
	initialize Xi = Xi0, Vi = Vi0, Wi = 1/Mi
endfor

loop:
	forall vertices i do
		Vi = Vi + DeltaTime * FuncExt(Xi) * Wi
	endfor
	
	DampVelocities(V1..Vn)
	
	forall vertices i do
		Pi = Xi + DeltaTime * Vi
	endfor

	forall vertices i do
		GenerateCollsionConstraints(Xi -> Pi)
	endfor

	loop SolverIterations times
		ProjectConstraints(C1, C2... CN, P1, P2... Pn)
	endloop

	forall vertices i do
		Vi = (Pi - Xi) / DeltaTime
		Xi = Pi
	endfor

	VelocityUpdate(V1, V2,.. Vn)
endLoop
```



<!--more-->
