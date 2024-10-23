---
layout: post
title:  "Herringbone Wang Tiles"
author: "ZZZZZY"
comments: true
tags: Dev
excerpt_separator: <!--more-->
---
最近在工作之余的午休时间发现了Herringbone Wang Tiles，用于实现随机地牢的生成。一些比较知名的肉鸽游戏都基于此算法来实现随机地牢的生成，例如Noita、Moonring等等。恰好在年中的时候，想给自己的独立游戏实现一套随机地图生成算法，当时调研了许多方案，但是并未放过多注意力到Herringbone Wang Tiles。因此非常感兴趣，想借这个文章简单了解下这个算法。

------

先简单数落数落自己当初实现的方案：把地图生成分为两块，一块是地牢迷宫的生成，另一块则是大世界的地图生成。大世界的生成类似于矮人要塞类似的地图，需要有生态、海拔、地形等基本的要素。实现的思路是随机生成Voronoi Graph，然后再基于高度图、适度图、噪声图修饰，创建出包含了生态的大世界地图。地牢迷宫的生成则使用了截然不同的方案，因为更希望使用预先设计好的关卡，在此基础上再添加点随机性，为此选择了基于网格化的地图生成方案，大致思路就是基于规则的NxN网格，通过构建网格路径，并在构建每个路径网格点时从RoomPool中选择匹配的Room，最终形成了满足设计需求的随机地图，进一步的资料可见参考【5】。不过可惜的是上述工作，有部分完成了，有部分只进展到一半，后续便因为各种原因暂时搁置了，后续很希望有机会能再写一篇文章，讲一讲上述实现的细节。

现在让我们回到主题，上一段只是想说明，不同的需求下，会有着不同的地牢生成方案。在我看来其中最为重要的一个主题，在于对随机过程的干预，我们总希望过程化的结果，能满足我们提前设定好的一些预期，不然最简单的洪水填充就能够生成足够复杂的地牢。另一个主题则在于速度，我们不能在地图生成上无休止的递归下去，特别是如果需要生成非常巨大的地图。Herringbone Wang Tiles就在一定程度上满足了上述两个主题，快速又有效！

##### Wang Tile

先来了解下Wang Tile。首先先思考这个么一个问题，如果我们预设了很多小块的地形（我们规定它们是正方形的Tile)，然后我们将它们随机紧凑的平铺在地图上，那么其中最重要的问题就是考虑每个地形（Tile）之间的约束关系。比如陆地不能直接相邻海洋，而是先相邻海岸线再相邻海洋，当然还有其他预先定义好的约束关系。

这里我们正式引入Wang Tile[王氏砖 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/王氏砖)。WangTile的定义简单来说就是正方形的砖块，且每一条边包含一种颜色，然后将这些砖块以王氏规则平铺整个平面：即二个王氏砖拼合时，其相邻的边需要有相同的颜色，在王氏砖拼合时，不允许旋转王氏砖，王氏砖也不能翻面。从定义上看，Wang Tile要处理的问题和我们上文中提出的问题一致。而且事实上很多纹理例如大理石、树皮等，正是基于Wang Tile的方式来生成复杂的纹路。

基于WangTile的方式，为了得到最好的结果（完美平铺整个平面），我们需要预先定义足够数量的Tile，假设我们在竖直方向的变化可能性为N，水平方向的变化可能性为M，则可能需要准备N^2 * M^2（可以优化为2xNxM）的数量，这一般称之为。考虑到如果想支持9种约束的话，就需要准备6561种Tile，这绝对是令人无法接受的。

------

##### Herringbone Tile





------

参考：

[1] [地图生成 - Noita Wiki](https://noita.wiki.gg/zh/wiki/地图生成)  
[2] [云风的 BLOG: 随机地形生成](https://blog.codingnow.com/2014/09/sandbox_world.html)  
[3] [Herringbone Wang Tiles](https://nothings.org/gamedev/herringbone/)  
[4] [Making maps with noise functions](https://www.redblobgames.com/maps/terrain-from-noise/)  
[5] [Ludomotion - Unexplored 2 Dev Blog - Level Generation](https://www.ludomotion.com/blogs/level-generation/)