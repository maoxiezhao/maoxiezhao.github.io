---
layout: post
title:  "类镜之边缘的运动系统浅析"
subtitle: 基于代码去分析一个类镜之边缘的运动系统.
author: "ZZZZZY"
comments: true
categories: GameDev
tags: Dev
sidebar: []
excerpt_size: 100
post_banner:
  image: https://s21.ax1x.com/2025/05/26/pVSDTeS.jpg
  opacity: 0.818
  height: "320px"
hidden:
  - related_posts
---
镜之边缘是一款以跑酷作为核心玩法的游戏，也是我心目中最优秀的跑酷游戏。尽管该游戏至今已经有十多年的历史，但依然难有后辈游戏能够在跑酷系统上超越它。作为一款跑酷为核心的游戏，镜之边缘的TraversalSystem构建的不可谓不优秀，优秀的表现，流程的手感，不愧是老Dice的技术力。Locomotion也是本人日常工作的核心内容，所以非常好奇镜之边缘是如何构建这样优秀的TraversalSystem。<!--more-->

StillAlive是镜之边缘这款游戏的民间导出版本，基于Unity和C#实现，关于TraversalSystem的实现，基于参照了原始游戏的版本。简单分析StillAlive的代码，能够很好的了解到类似镜之边缘这样的游戏，一套优秀的TraversalSystem是如何构建的。

一个TraversalSystem，可以从某种程度上分成两部分：一部分是逻辑层，负责各个运动行为的表现和切换；另一部分则是物理层，在不同的运动状态下，所执行不同的物理运动逻辑。后文也会基于上述的思路从两个部分拆解StillAlive中的运动系统，相信基本能囊括到TraversalSystem的方方面面。

>本文会分为几部分，多次写完，整体结构可能会多次调整

TdMove概述
------------

TdMove用于实现在不同运动状态下，执行不同的游戏逻辑，类似于TurnInPlace，Slide，Climb等等。所有的运动状态被细分拆解成一系列的TdMove，从而能够以OOP的方式组织代码。TdMove以类型数组的方式存储在角色对象上（Pawn），数组的索引和枚举值一一对应，从而能够取出对应的TdMove类，执行后续实例化逻辑。大致代码如下：
~~~csharp
public enum EMovement
{
    MOVE_None,
    MOVE_Walking,
    MOVE_Falling,
    MOVE_Grabbing,
    // ..... 
    MOVE_BotGetDistance,
    MOVE_Cutscene,
    MOVE_MAX
};
class Pawn
{
    // 当前运动状态的枚举值
    public TdPawn.EMovement MovementState;
    [repnotify] public TdPawn.EMovement ReplicatedMovementState;

    // TdMove的类
    public array<TdMove> MoveClasses = new array<TdMove>() {
        default,
        ClassT<TdMove_Walking>(),
        ClassT<TdMove_Falling>(),
        ClassT<TdMove_Grab>(),
        // .......
        ClassT<TdMove_Cutscene>(),
    }
    
    // TdMove的实例对象，由MoveClasses构建
    public array<TdMove> Moves;
};
~~~

可以从代码中看到，Pawn中还存储了当前MovementState（MovementState同时是网络复制对象），可以想到TdMove以类似于FSM的形式去触发，在StillAlive中TdPawn_SetMove会设置当前MovementState，并执行对应TdMove的进入和退出逻辑，同时标记网络复制对象，从而可以同步到各个模拟端。

~~~csharp
class Pawn
{
    public bool TdPawn_SetMove(EMovement NewMove, bool _bViaReplication)
    {
        if(_bViaReplicat1ion)
        {
            // ...
        }
        else
        {
            // 执行上一个TdMove的退出逻辑
            if(((int)MovementState) != ((int)TdPawn.EMovement.MOVE_None/*0*/))
            {
                Moves[((int)MovementState)].StopMove();
            }

            OldMovementState = ((TdPawn.EMovement)MovementState);
            MovementState = ((TdPawn.EMovement)NewMove);

            // 执行上一个TdMove的PostStop
            if(((int)OldMovementState) != ((int)TdPawn.EMovement.MOVE_None/*0*/))
            {
                Moves[((int)OldMovementState)].PostStopMove();
            }

            // 标记网络复制
            ReplicatedMovementState = ((TdPawn.EMovement)NewMove);
            SetNetUpdateTime(WorldInfo.TimeSeconds - ((float)(1)));

            // 执行新的Move逻辑
            Moves[((int)MovementState)].StartMove();

            // 分发NewMove的事件
            NotifyNewMove();
        }
    }
};
~~~
现在看看TdMove本身的一些实现，结构上大致和传统FSM的State类似，其中核心的两个方法是StartMove和StopMove，用于执行进入和退出时的响应逻辑。先看看StillAlive中TdMove的实现：

~~~csharp
// 仅列出核心的几个函数
class TdMove
{
    // 当前Move能否执行
    public virtual bool CanDoMove()
    {
        // 检测行为冷却
        if((LastStopMoveTime > 0.0f) && RedoMoveTime > (CurrentWorldTime - LastStopMoveTime))
        {
            return false;
        }
        // 检测玩家状态(死亡)
        if (!CheckPawnStatus())
        {
            return false;
        }
        
        return true
    }

    public virtual void StartMove()
    {
        // 基类的StartMove提供了一些通用的配置，类似于禁止移动，关闭碰撞等等
        PawnOwner.SetIgnoreMoveInput(DisableMovementTime);
        PawnOwner.SetIgnoreLookInput(DisableLookTime);
        PawnOwner.SetCollision(true, bDisableCollision, default(bool?));
        PawnOwner.LookAtRelease()
    }
    
    public virtual void StopMove()
    {
        // 通用清除行为，恢复StartMove时的一些设置，恢复碰撞，取消禁止移动
        PawnOwner.StopIgnoreLookInput();
        PawnOwner.StopIgnoreMoveInput();
        // ...
    }
    
    // 特殊的运动逻辑，发生在PerformPhyscis（实际运动）之前
    public virtual unsafe void PrePerformPhysics(float DeltaTime)
};
~~~
TdMove作为基类，实现了一些通用的配置逻辑，用于在Move开始和结束时，设置角色的一些通用状态（碰撞、动画）。同时也封装了一些可公用的工具方法，例如MovementLineCheck、FindLedge等等，供各个派生类使用（虽然更合适的做法是封装一个Utility的静态工具类）。

TdMove中还有另外一个非常重要的方法是PrePerformPhysics，这是一个在每次玩家执行物理运动前都会调用到的方法。换句话说，在单机模式下这可以被视为一种Tick逻辑。TdMove能够在这个函数中，执行自定义的逻辑影响到后续的PerformPhysics，比如设置速度、加速度、运动模式。

其实从这里也可以看出，TdMove相比于后文会讲到的PhysBehavior，更倾向于为表现层逻辑。即执行这个Move时，角色需要处于什么状态，播什么动画，执行特定的表演相机，可以理解为一种MovementAbility。如果放在现在的UE5中去实现类似的系统，GameAbilitySystem会是一套合适的框架，这个后续可以延申一下。

现在我们需要知道StillAlive在何时去调用TdPawn_SetMove，显而易见的思路就是检测当前条件，触发对应行为。更具体的说法就是，收集角色当前状态信息和通用的环境信息，根据优先级依次去检测各个Move的条件。从这里引申出来的两个重要的问题，一是如何更方便的配置不同优先级的检测序列，另外一个则是如何优化检测的效率，这里每个问题都可以展开述说，但是先搁置不谈。

回到StillAlive中，触发SetMove的情况大致分为四部分：
- TdMove本身逻辑去设置下一个Move的类型。例如爬梯子过程中，解除到地面，手动切换到MOVE_Walking
- PhysicsVolume，如LadderVolume、ZiplineVolume，当Pawn接触到PhysicsVolume会尝试设置对应的Move
- PlayerMoveManager，一个巨大的FSM，处理在不同的MovementState时对特定MovementAction（后续探讨这个概念）的响应，在响应的过程中会设置特定的TdMove。
- CheckAutoMoves，最核心的部分，CheckAutoMoves发生在TdPawn的performPhysics中，即实际物理运动之前。CheckAutoMoves会基于角色当前位置和运动趋势，向前方检测障碍物的信息，判断障碍物是否满足特定行为的需求，列如VaultOver，WallClimbi等等。这类行为往往也会影响到后续的PhysBehavior。

基于上述四个部分，就可以囊括StillAlive中所有Move相关的逻辑。从这四个部分去进一步分析代码，就能够更清晰地了解到StillAive地TraversalSystem地实现。

镜之边缘的Move行为非常丰富，在进一步分析更多的TdMove之前，先简单看一下StillAlive中TdMove的继承链：
@startuml

class TdMove {
    + virtual bool CanDoMove()
    + virtual void StartMove()
    + virtual void StopMove()
}

class TdPhysicsMove
TdMove <|-- TdPhysicsMove

class TdMove_Walking
TdPhysicsMove <|-- TdMove_Walking

class TdMove_ZipLine
TdPhysicsMove <|-- TdMove_ZipLine
class TdMove_WallClimb
TdPhysicsMove <|-- TdMove_WallClimb

class TdCustomMove
TdMove <|-- TdCustomMove

class TdMove_Slide
TdCustomMove <|-- TdMove_Slide

@enduml

TdMoveExamples
------------
接下来让我们从具体的实现入手，进一步分析相关的实现，先从Zipline这个相对简单的例子入手。
#### TdMove_Zipline
StillAlive定义了TdZiplineVolume，TdZiplineVolume继承自TdMovementVolume，本质就是一种空间碰撞体（对于UE熟悉的碰撞应该会很熟悉这个概念），TdMovementVolume提供了PawnUpdate的虚函数，用于实现各个物理体积的特殊逻辑。现在看看TdZiplineVolume的PawnUpdate实现：

~~~csharp
public class TdZiplineVolume : TdMovementVolume
{
    public override void PawnUpdate(TdPawn CurrentPawn)
    {
        EMovement CurrentMovementState = CurrentPawn.MovementState;
        if(CurrentMovementState == EMovement.MOVE_ZipLine || 
            CurrentMovementState == EMovement.MOVE_IntoZipLine)
        {
            return;
        }
        
        TdMove_IntoZipLine tdMove = CurrentPawn.Moves[(int)EMovement.MOVE_IntoZipLine];
        tdMove.ZipLine = this;
        if(tdMove.CanDoMove())
        {
            CurrentPawn.SetMove(TdPawn.EMovement.MOVE_IntoZipLine);    
        }
    }
}
~~~

实现的非常直白，如果玩家没有处于Zipline状态，则获取TdMove_IntoZipline实例，如果CanDoMove满足，则设置为当前的Move。TdMove_intoZipline的CanDoMove我们不列出具体代码，基本思路就是继续检查玩家当前的状态上能否执行Zipline，玩家朝向和Zipline的夹角是否满足要求（Zipline在玩家角色正前方）。StartMove值得展开一下，期望的效果是角色播放一个抓取动作，同时位置变换到最近的Zipline位置，朝向朝向Zipline的方向，大致的代码实现如下：

~~~csharp
public partial class TdMove_IntoZipLine : TdPhysicsMove
{
    public override void StartMove()
    {
        // 基于Zipline的Spline，根据玩家位置找到Spline上最近的点，这将是目标对齐点
        Vector DestToReach;
        ZipLine.FindClosestPointOnDSpline(PawnOwner.Location, ref DestToReach);
        
        // 计算角色预期旋转，朝向Zipline方向
        Rotator wantedrotation = ZipLine.GetSlopeOnSpline(0);
        
        // 基于当前速度计算对齐间隔（速度越快对其越快）
        float TimeToDestination = VSize(DestToReach - PawnOwner.Location) / IntoClimbSpeed;
        
        // 设置对齐（精确）位置
        SetPreciseLocation(DestToReach, TdMove.EPreciseLocationMode.PLM_Fly, IntoClimbSpeed);        
        // 设置对齐（精确）旋转
        SetPreciseRotation(WantedRotation, TimeToDestination);
        // 播放对应动画
        PlayMoveAnim(CNT_FullBody, "ZiplineStart");
    }
}
~~~

上述代码同时展示了对于大部分Move行为的范式逻辑：计算目标位置和旋转，播放对应动画表现，配合动画表现并在预期时间内将角色设置到目标位置和旋转。**个人认为这是TraversalSystem中最为核心的部分，可以延申出的几个关键问题是：如何正确计算目标位置（或者多个目标位置），如何选择合适的动画，如何处理位移和选择。**

我们不过多追究具体的代码实现，但是让我们简单看看PreciseLocation和PreciseRotation的处理。这两个方法记录了目标变换，以及插值的速度和时间。因为角色位置和旋转实际上属于物理属性，所以具体的处理逻辑发生在基类的TdMove::PrePerformPhysics中，基于DeltaTime会逐渐插值到我们设置的目标变换上。这里说一句题外话，给动画的RootMotion加上MotionWarping，将固定的动画位移缩放以匹配到目标点，也是非常常见的方案。

当角色到达PreciseLocation后，会触发ReachedPreciseLocation回调，这使得我们能够推进后续行为。对于TdMove_IntoZipline来说，则相当于完成进入（对齐）Zipline的行为，可以正式推进Zipline逻辑。

~~~csharp
public partial class TdMove_IntoZipLine : TdPhysicsMove
{
    public override void ReachedPreciseLocation()
    {
        if(PawnOwner.Moves[(int)EMovement.MOVE_ZipLine].CanDoMove())
        {
            PawnOwner.SetMove(EMovement.MOVE_ZipLine);        
        }
        else
        {
            PawnOwner.SetMove(TdPawn.EMovement.MOVE_Falling);
        }
    }
}
~~~

如果说TdMove_IntoZipline是一个典型的行为式Move，那么TdMove_Zipline则代表着另一种典型的状态式Move。先抛开StillAlive的实现，去思考Zipline状态时需要做的事，一是施加基于Zipline方向的加速度，且需要将速度方向限制在Zipline方向上，并且在快到末尾的时候能够减速，到达末尾时能正确退出状态；二是需要处理Zipline过程中的表现，正确播放动画，播放声音。

在StillAlive中，定义了ZiplineStatus枚举，用于标注Zipline当前的执行状态。然后我们需要在每次执行运动逻辑前，检测当前的ZiplineStatus，从而执行不同的分支逻辑。这段逻辑实现在TdMove_Zipline::PrePerformPhysics中：

~~~csharp
public partial class TdMove_ZipLine : TdPhysicsMove
{
    public enum EZipLineStatus 
    {
        ZLS_Moving,        // zipline默认运动中
        ZLS_CloseToEnd, // 接近Zipline末端
        ZLS_Impact,     // 接触Zipline末端
        ZLS_MAX
    };
    
    public override unsafe void PrePerformPhysics(float DeltaTime)
    {       
        // 仅主控端和服务端调用后续逻辑
        TdPawn PawnOwner = this.PawnOwner;
        if (PawnOwner.Role < ROLE_AutonomousProxy)
        {
            return;
        }
        
        // 计算角色当前的速度
        // 大致思路就是计算角色所处的ZiplineSegment，基于Segment的两个端点和玩家的位置关系，计算当前的速度向量
        // 主要是要让速度匹配上Zipline的方向，具体代码略过
        FVector NewVelocity;
        // ...
        PawnOwne.Velocity = NewVelocity;
        
        // 处理ZiplineStatus
        // 先检测运动路径上是否存在障碍
        EZipLineStatus Status = this.ZiplineStatus;
        FVector Start = PawnLocation;
        FVector End = NextPoint; // 基于当前运动状态预测的目标点
        if (this.MovementTraceForBlocking(Start, End, a2))
        {
            if (Status == ZLS_Impact)
            {
                // 如果已经Impact则停止当前移动
                PawnOwner.Velocity = FVector::ZeroVector;
                PawnOwner.Acceleration = FVector::ZeroVector;
            }
            else if (Status == ZLS_CloseToEnd)
            {
                this.PlayForwardImpact();
            }
            else // Status == ZLS_Moving
            {
                // 先切换CloseToEnd状态
                this.PrepareForForwardImpact();
                  this.ZipLineStatus = ZLS_CloseToEnd;
            }
        }
    }
}
~~~

PrePerformPhysics实现了和运动（速度）有关的所有逻辑，基于当前所处Zipline的Segment计算速度的向量，根据DeltaTime预测计算运动后的位置，并调用MovementTraceForBlocking检测运动路径上是否存在障碍，来驱动ZiplineStatus的切换。

例如当ZiplineStatus等于ZLS_CloseToEnd时，会触发PlayForwardImpact，这意味着角色已经到达了Zipline的另一端末尾（或者撞上障碍物）。我们需要结束Zipline状态。除此以外类似于这种状态式Move，还需要处理额外的一些打断行为，通过在一些回调中设置新的Move来退出ZiplineMove。

~~~csharp
public partial class TdMove_ZipLine : TdPhysicsMove
{
    // 到达目标点，0.8s后切换到Falling，结束Zipline
    public virtual void PlayForwardImpact()
    {
        ZipLineStatus = TdMove_ZipLine.EZipLineStatus.ZLS_Impact;
        PawnOwner.SetIgnoreLookInput(0.80f);
        PawnOwner.SetIgnoreMoveInput(0.80f);
        SetTimer(0.80f);
    }
    public override void OnTimer()
    {
        PawnOwner.SetMove(TdPawn.EMovement.MOVE_Falling);
    }
    
    // 死亡则退出Zipline
    public override int HandleDeath(int Damage)
    {
        PawnOwner.SetMove(TdPawn.EMovement.MOVE_Falling);
        return base.HandleDeath(Damage);
    }
}
~~~

至此我们已经基本梳理整个Zipline的逻辑。在StillAlive中，楼梯、游泳等可以明确定义边界的区域，都被实现为PhysicsVolume的方式，相关的Move驱动逻辑也都和上文中的Zipline类似，便不再赘述。

#### TdMove_WallClimb

StillAlive中还有很多运动行为，更多和玩家物理环境有关。例如角色面前的障碍物的形状，它的高度和宽度，决定了角色面对这个障碍物所执行的行为，如果障碍物足够矮且宽，类似于栅栏，则执行跨越行为。如果是一面足够高的墙，则尝试攀爬墙面。当然这依然能够使用基于PhysicVolume的方式实现，只要在这个PhysicsVolume上标注这个障碍物的具体信息，甚至还能够以更省性能的方式实现相关功能。但是和梯子等物体不同，对于一款能够自由穿梭跑酷的游戏来说，能够翻越攀爬的障碍物到处都是，你很难管理遍布全图的PhysicVolume，所以更通用的一种方式是，在玩家移动的时候，通过一定的射线检测评估面前障碍物的形状，如果满足条件则尝试执行对应行为，这也正好是上文说的SetMove的第四种方式CheckAutoMoves，现在让我们进一步看看它的相关实现。


Physics物理运动
------------

~~~csharp
public virtual unsafe void startNewPhysics(FLOAT deltaTime, INT Iterations)
{
    if ( (deltaTime < 0.0003f) || (Iterations > 7) )
        return;

    switch (Physics)
    {
        case PHYS_None         : return;
        case PHYS_Walking      : physWalking(deltaTime, Iterations); break;
        case PHYS_Falling      : physFalling(deltaTime, Iterations); break;
        case PHYS_Flying       : physFlying(deltaTime, Iterations); break;
        case PHYS_Swimming     : physSwimming(deltaTime, Iterations); break;
        case PHYS_Spider       : physSpider(deltaTime, Iterations); break;
        case PHYS_Ladder       : physLadder(deltaTime, Iterations); break;
        case PHYS_RigidBody    : physRigidBody(deltaTime); break;
        case PHYS_SoftBody     : NativeMarkers.MarkUnimplemented(); break;
        case PHYS_Interpolating: physInterpolating(deltaTime); break;
        default:
            setPhysics(PHYS_None);
            break;
    }
}
~~~


> 未完待续。。。

------

参考：

[1] [Mirror's Edge Decompiled](https://www.eideren.com/posts/mirrors-edge-decompiled.html)