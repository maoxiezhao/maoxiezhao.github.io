---
layout: post
title:  "Herringbone Wang Tiles"
subtitle: 过程化生成中一种常用的模式.
author: "ZZZZZY"
comments: true
tags: Dev
sidebar: []
excerpt_size: 100
post_banner:
  image: https://s21.ax1x.com/2025/07/19/pV3IBGQ.png
  opacity: 0.818
  height: "320px"
hidden:
  - related_posts
---
最近在工作之余的午休时间发现了Herringbone Wang Tiles，用于实现随机地牢的生成。一些比较知名的肉鸽游戏都基于此算法来实现随机地牢的生成，例如Noita、Moonring等等。恰好在年中的时候，想给自己的独立游戏实现一套随机地图生成算法，当时调研了许多方案，但是并未放过多注意力到Herringbone Wang Tiles。因此非常感兴趣，想借这个文章简单了解下这个算法。

先简单数落数落自己当初实现的方案：把地图生成分为两块，一块是地牢迷宫的生成，另一块则是大世界的地图生成。大世界的生成类似于矮人要塞类似的地图，需要有生态、海拔、地形等基本的要素。实现的思路是随机生成Voronoi Graph，然后再基于高度图、适度图、噪声图修饰，创建出包含了生态的大世界地图。地牢迷宫的生成则使用了截然不同的方案，因为更希望使用预先设计好的关卡，在此基础上再添加点随机性，为此选择了基于网格化的地图生成方案，大致思路就是基于规则的NxN网格，通过构建网格路径，并在构建每个路径网格点时从RoomPool中选择匹配的Room，最终形成了满足设计需求的随机地图，进一步的资料可见参考【5】。不过可惜的是上述工作，有部分完成了，有部分只进展到一半，后续便因为各种原因暂时搁置了，后续很希望有机会能再写一篇文章，讲一讲上述实现的细节。

现在让我们回到主题，上一段只是想说明，不同的需求下，会有着不同的地牢生成方案。在我看来其中最为重要的一个主题，在于对随机过程的干预，我们总希望过程化的结果，能满足我们提前设定好的一些预期，不然最简单的洪水填充就能够生成足够复杂的地牢。另一个主题则在于速度，我们不能在地图生成上无休止的递归下去，特别是如果需要生成非常巨大的地图。Herringbone Wang Tiles就在一定程度上满足了上述两个主题，快速又有效！

#### Wang Tile

先来了解下Wang Tile。首先先思考这个么一个问题，如果我们预设了很多小块的地形（我们规定它们是正方形的Tile)，然后我们将它们随机紧凑的平铺在地图上，那么其中最重要的问题就是考虑每个地形（Tile）之间的约束关系。比如陆地不能直接相邻海洋，而是先相邻海岸线再相邻海洋，当然还有其他预先定义好的约束关系。

这里我们正式引入Wang Tile[王氏砖 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/王氏砖)。WangTile的定义简单来说就是正方形的砖块，且每一条边包含一种颜色，然后将这些砖块以王氏规则平铺整个平面：即二个王氏砖拼合时，其相邻的边需要有相同的颜色，在王氏砖拼合时，不允许旋转王氏砖，王氏砖也不能翻面。从定义上看，Wang Tile要处理的问题和我们上文中提出的问题一致，且从理论上支持了平铺的可行性。而且事实上很多纹理例如大理石、树皮等，正是基于Wang Tile的方式来生成复杂的纹路。

基于WangTile的方式，为了得到最好的结果（完美平铺整个平面），我们需要预先定义足够数量的Tile，假设我们在竖直方向的变化可能性为N，水平方向的变化可能性为M，则可能需要准备N^2 * M^2的数量，这一般称之为完全随机集。考虑到如果想支持9种约束的话，就需要准备6561种Tile，这绝对是令人无法接受的。基于[Wang Tiles for Image and Texture Generation](http://research.microsoft.com/~cohen/WangFinal.pdf)的范式，数量可以缩减到2xNxM，并保证能满足非周期性的随机平铺，但想要更好的表现，依然需要提供大量的Tile资源。

考虑另外一个问题，我们目前假定了所有的Tile都是规则的正方形，但是很多时候我们希望支持更多的形状，例如1x2或者2x2，我们称为LargeTile。WantTile无法直接支持这种情况，但是我们可以将1x2或者2x2的LargeTile分解成多个1x1的Tile，然后对这些Tile之间以独一无二的颜色来构建新的约束。这能够使得WangTile支持LargeTile，但是考虑到如果采用简单的扫描线随机算法来生成WangTile的话，那么为了满足平铺的可行性，拆分后的Tile也必须要支持所有的颜色（约束）匹配。那么当我们想添加一块1x3的LargeTile时，假设目前已经有了3种颜色约束，我们就至少需要多准备2x3x3+3+3=24种Tile，这显然是疯狂的。

WangTile还有一些其他问题，比如无法很好的支持Tile的旋转，或者在地图生成时，存在一些边缘性问题，具体可见[3]。

#### Herringbone Tile

我们在上文中提出我们想要实现的需求：预先定义的Tile，任意大小的四边形，适量的资源量，足够快的速度，满足非周期性平铺。WangTile给我们提供了方法，但是没法完全支持我们的需求。[3]的作者因此提出了一种新的设计，称之为Herringbone Tile。Herringbone Tile将Tile定义为1:2或者2：1的矩形，以一种交错的方式平铺，效果如下图：

<figure>
<img src="https://s21.ax1x.com/2024/10/24/pAdvuW9.png" alt="shell">
</figure>

这种平铺方式，简单说就是满足下列规则：

- 垂直矩形的右边缘的顶部始终邻接水平矩形的左边缘
- 垂直矩形右边缘的底部始终邻接垂直矩形的左上边缘

基于这种规则，能够满足铺满一个矩形区域。同时对于一个Tile而言，存在6种表现一致的边缘匹配模式（类似于WangTile的水平和垂直匹配）。事实上，可以把这种匹配方式视为一个特化的六边形匹配。在Cohen-et-al-style的平铺模式下，对于一个HerringboneTile来说需要处理3个边缘的约束匹配，假设我们存在8种匹配情况，那么就需要准备2 * 8 = 16种水平Tile和垂直Tile。如果需要满足完全随机集，则需要准备64种水平Tile和垂直Tile。最终可以形成一个拥有更多匹配模式，更少资源量的地图生成机制。


------

参考：

[1] [地图生成 - Noita Wiki](https://noita.wiki.gg/zh/wiki/地图生成)  
[2] [云风的 BLOG: 随机地形生成](https://blog.codingnow.com/2014/09/sandbox_world.html)  
[3] [Herringbone Wang Tiles](https://nothings.org/gamedev/herringbone/)  
[4] [Making maps with noise functions](https://www.redblobgames.com/maps/terrain-from-noise/)  
[5] [Ludomotion - Unexplored 2 Dev Blog - Level Generation](https://www.ludomotion.com/blogs/level-generation/)