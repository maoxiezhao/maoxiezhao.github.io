---
layout: post
title:  "手撸自嗨引擎NeptuneEngine(一)BuildSystem"
author: "ZZZZZY"
comments: true
tags: Dev
excerpt_separator: <!--more-->
---
*当前版本比较潦草，仅仅为了记录，后续会陆续修改*

## 前言

终于来到了我最想开的坑系列，毕竟自嗨的最高境界就是基于自嗨而撸的引擎上，再写关于它的自嗨文章。
首先这将是自嗨系列的第一个文章，也将是这个系列的入口，我将会简单介绍下NeptuenEngine，同时会规划（画饼）后续关于这个系列的文章  
NeptuenEngine是一个完全是因为想去做而去做的一个开源引擎，<!--more--> 当然要说的好听点，这是一个用来学习各种功能模块的沙盒，Neptune引擎的实现参考了（抄了）许多的开源引擎和大名鼎鼎的UE，而且目前这个NeptuenEngine并没有开源，但是可以参考NeptuenEngine的初代版本：https://github.com/maoxiezhao/NeptuneEngine.git   *PS:等开源了这段会被替换掉* 

回到正文，NeptuenEngine是一个基于C++17和Vulkan的引擎，基于模块化设计，实现（尝试实现）引擎一些常见的模块，包括但不限于基础的Core模块、RHI、Renderer、Resource、Level、Animation、Script、Editor等等。整体目标是朝着现代化，高性能的方向前进，当然主要是为了自嗨。预计开坑的文章如下：

  1. BuildSystem
  2. ECS实现
  3. MultiThreading
  4. RenderGraph
  5. NeptuenEngine的Foward+管线
  6. Animation

其中BuildSystem也就是本篇内容，同时除了上述以外其实还有许多可以分享的主题比如Vulkan，反射等等，但是饼不能画的太大，主要是迭代后的NeptuenEngine进度感人。

## BuildSystem
BuildSystem.你一定很好奇为什么第一篇文章是关于BuildSystem的，我们提出以下疑问？
* 什么是BuildSystem?
* 为什么不用现用的BuildSytem?
* 你的BuildSystem实现了什么？

BuildSystem作为引擎来说是非常重要的不亚于引擎的本身的重要话题，特别是像UE这种庞然大物的引擎，构建系统更是其中必不可少的一部分。那么BuildSystem到底是啥？简单理解就是基于各种平台构建编译项目代码，比如著名的就有各种Make工具，例如CMake等等，这些工具支持在各种平台构建项目，编译项目同时支持复杂的编译配置项，并能够将项目以模块进行管理配置，处理依赖，总之哪怕从最简单角度来讲这都是一个复杂的系统。

对于开源引擎来说，使用现在的工具是最好不过的选择，例如NeptuneEngineV1使用的就是Premake,Premake支持使用Lua脚本作为配置工具，非常灵活和方便，利用Premake生成各个平台下的项目工程，再使用各个平台的编译工具编译。例如Win下会生成VS工程，后续就走MSBuild,这对于大部分的个人项目来说绰绰有余。但是对我来说，这依然遇到了一些痛点，这也将解释‘为什么不用现用的BuildSytem’这个疑问。