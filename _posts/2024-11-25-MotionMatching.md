---
layout: post
title:  "UE中的MotionMatching"
author: "ZZZZZY"
comments: true
tags: Dev
excerpt_separator: <!--more-->
---
MotionMatching应该算是Locomotion开发中始终绕不开的话题，从育碧最开始的分享让人大呼惊讶，到现在几乎成了3A游戏的标配。从一个已有的实现出发，来分析MM的相关实现是一个很好的思路。UE的MotionMaching实现从5.0的实验版本到5.5的Beta版，代码逐步趋于稳定，因此我们接下来以UE作为例子，来逐步聊聊MM。

本身MM的相关的内容很多，本文会从MM的理论实现出发，到UE的具体实现分析，分成几部分逐步完成。

------

先老生常谈的说一下为什么需要MotionMatching，一般来说以往的Locomotion的实现都是为角色设定固定的几种移动速度，例如走路速度，跑步速度，同时动画状态机中根据移动状态播放对应的移动动画，这些动画会从表现上尽量去匹配实际的移动速度。这对于大部分情况下，表现的非常不错，特别是处于某种恒定的运动状态时（匀速跑），动画的表现能够完美匹配。但是游戏往往会频繁出现角色运动状态的改变，列如运动方向的突变，运动速度或者加速度的改变。在处理这些问题上，动画状态机就会变得比较困难。虽然最后都能够用动画状态机的方式实现，但往往需要依赖大量的动画曲线，同时动画状态机也会变得错综复杂，甚至依然难以达到期望的表现。

举一些例子，



------

参考：

[1] [Motion Matching 中的代码驱动移动和动画驱动移动 - 知乎](https://zhuanlan.zhihu.com/p/432663486)  
[2] [Unreal Engine 5.4 : Motion Matching Explained | 1-Hour Deep Dive Tutorial & Guide | First look - YouTube](https://www.youtube.com/watch?v=GwHyhPouBb0)

