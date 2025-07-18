---
layout: post
title:  "UE中的MotionWarping"
author: "ZZZZZY"
comments: true
tags: Dev
excerpt_separator: <!--more-->
---

在实现项目中相关的自定义移动行为的时候，时常需要能够自适应处理各种场景，例如实现诸如Mantle或者Vault行为时，提供的动作资源往往是基于固定高度的动画，但是希望能够适应不同高度的障碍物。很自然的想法是希望能够在动画的不同阶段，将动作提供的位移进行缩放，以能够适应不同的地形。幸运的是早在早些年的游戏中类似这样的技术就已经提出，Doom和地平线等游戏中就在他们的GDC分享中说明相关的实现。同时UE5中也提供了MotionWarping，能够在Montage中配置对应的动画通知，提供Target信息后，能够将这段位移进行变形以达到目标点。本文就基于UE的MotionWarping简单地进行分析。

同以往的文章一样，本文依旧是基于个人的记录，并不会对文章的可读性和正确性进行保证。
<!--more-->

本文将分成两块内容，一是基于简单的数学原理分析如何实现MotionWarping，二是就UE5的源码进行分析，其他的细节例如UE5中的一些概念会一笔带过，同时本文会仅仅只分析Location，Rotation可以自行去源码中检索。

### 1. 数学原理

数学原理的分析中，我们只是简单讨论下如何实现我们需要的效果，毕竟正儿八经的数学分析本人也不擅长。

假设我们在一段位移中，我们从X0点移动到了X1点，然后此时我们期望修改目标点，从X0移动到X2点。直观地做法是直接计算Scale=(X2-X0) / (X1-X0)，然后直接缩放速度，Vnew = Vold*Scale？在讨论这个简单的做法是否正确之前，我们需要知道原始的RootMotion是如何驱动速度的。引擎会在每次Tick时，采样当前位移减去上一次采样的位移，得到位移差后除以此次DeltaTime得到速度。所以上述缩放速度的做法等价为缩放这一次Tick中采样的位移差，即Delta(X)new = Delta(X)old * Scale。那么只要我们把总的位移平分为数次位移后，然后每次都乘上一个相同的缩放，那么最终总的位移就是Total(X) =Sum(Scale* Delta(X))，提取Scale常量后，就是总的位移的缩放比，看起来虽然绕了一圈，但是乘以缩放这个思路是完全可行的！

但是实际情况总是复杂一些，实际游戏中每次Tick的DeltaTime完全不一样，甚至极端情况某一帧时角色因为外界系统发生了一个非常大的偏移，或者说我们在一次位移的中途忽然希望执行Warping，那么该怎么办呢？

直接给出解决方案，既然没办法一开始就计算出Scale，那么每次都去计算Scale，我们已知当前的位移差DeltaX，以及剩余位移TotalX。我们计算当前实际位置到期望位置的差值ExpectTotalX,那么我们调整这一帧的位移 = (ExpectTotalX / TotalX) * DeltaX。emm，验证过程就省去了，直觉上这也是符合的hhhhh。

### 2.源码分析

上文中我们非常简单地分析了下数学上的原理，但是实际上还是有些细节我们没有考虑，实际上UE中的实现也从5.0开始到5.3发生了些变换，变得更为完善，我们分析代码的同时，也能够知道到底还需要考虑哪些情况。

UE中MotionWarping实现基于RootMotionModifier，RootMotionModifier会绑定CharacterMovement的委托ProcessRootMotionPreConvertToWorld，使得RootMotionModifier能够在提取出RootMotion之后且作用之前，能够对其进行修改。我们关注ProcessRootMotion函数，这是MotionWarping处理RootMotion的核心函数。

*注意下列代码我会做一些修改，以尽可能达到只展示核心逻辑的目的，同时为了减少展示的代码量，我会只考虑WarpTranslation同时IgnoreZAxis的情况*

```c++
FTransform URootMotionModifier_SkewWarp::ProcessRootMotion(const FTransform& InRootMotion, float DeltaSeconds)
{
    // 这里提取上一次采样时间到结束时间的总变换，以及当前时间片内的变换
    FTransform RootMotionTotal = ExtractRootMotionFromAnimation(Animation.Get(), PreviousPosition, EndTime);
	FTransform RootMotionDelta = ExtractRootMotionFromAnimation(Animation.Get(), PreviousPosition, FMath::Min(CurrentPosition, EndTime));

    // 处理Locaiton的Warp
    if (bWarpTranslation)
	{
        // 获取当前的Root位置
        const FVector CurrentLocation = (CharacterOwner->GetActorLocation() - UpVector() * CapsuleHalfHeight);
        const FVector DeltaTranslation = RootMotionDelta.GetLocation(); // 动画当前位移
		const FVector TotalTranslation = RootMotionTotal.GetLocation(); // 动画剩余总位移	
        const FTransform MeshTransform = ... // 获取Mesh变换矩阵
        // 得到Mesh空间下的TargetLocation
		TargetLocation = MeshTransform.InverseTransformPositionNoScale(TargetLocation);

        // 执行位置的Warp
		FVector WarpedTranslation = WarpTranslation(DeltaTranslation, TotalTranslation, TargetLocation);
		FinalRootMotion.SetTranslation(WarpedTranslation);
    }
}
```

ProcessRootMotion收集了必要的信息，并执行到Mesh空间下的转换，得到了当前位置，目标位置，以及动画当前位移和剩余的总位移，让我们继续看WarpTranslation中的实现。

```c++
FVector URootMotionModifier_SkewWarp::WarpTranslation(const FVector& DeltaTranslation, const FVector& TotalTranslation, const FVector& TargetLocation)
{
    FVector CurrentLocation = FVector::Zero; // CurrentTransform.GetLocation();
    FVector FutureLocation = CurrentLocation + TotalTranslation;
    
    // 计算期望目标点的位移差，和动画目标位移差
    FVector CurrentToWorldOffset = TargetLocation - CurrentLocation;
    FVector CurrentToRootOffset = FutureLocation - CurrentLocation; 
    
    // 核心：这里UE构建了一个ToRootSyncSpace矩阵，用于将所有的位置变换到“当前位置LookAt动画目标点的”的空间，相当于先动画位移方向作为正方向构建了坐标系
    
    // Put everything into RootSyncSpace.
    FVector RootMotionInSyncSpace = ToRootSyncSpace.InverseTransformVector(DeltaTranslation);
    FVector CurrentToWorldSync = ToRootSyncSpace.InverseTransformVector(CurrentToWorldOffset);
    FVector CurrentToRootMotionSync = ToRootSyncSpace.InverseTransformVector(CurrentToRootOffset);

    // 这里将期望的位移投影到了动画位移的方向，计算了缩放比
    float ProjectedScale = FVector::DotProduct(CurrentToWorldSync, CurrentToRootMotionSyncNorm) / CurrentToRootMotionSync.Size();
    if (ProjectedScale != 0.0f)
    {
        // 计算X轴正方向的缩放，因为X正方向是LookAtRootDir，所以上面计算的ProjectedScale就是
        // X轴的缩放
        FMatrix ScaleMatrix;
        ScaleMatrix.SetIdentity();
        ScaleMatrix.SetAxis(0, FVector(ProjectedScale, 0.0f, 0.0f));

        // 计算Y轴的缩放
        // 计算实际位移方向和X轴（CurrentToRootDir）的夹角，然后乘上Tan(angle)
		FVector FlatToWorldDir = FlatVectorNorm(CurrentToWorldSyncNorm);
		FVector FlatToRootDir = FlatVectorNorm(CurrentToRootMotionSyncNorm);
		float AngleAboutZ = FMath::Acos(
            FVector::DotProduct(FlatToWorldDir, FlatToRootDir));
		float AngleAboutZNorm = FMath::DegreesToRadians(
            FRotator::NormalizeAxis(FMath::RadiansToDegrees(AngleAboutZ)));	
        if (FlatToWorld.Y < 0.0f)
			AngleAboutZNorm *= -1.0f;
    
        FMatrix ShearXAlongY;
        ShearXAlongY.SetIdentity();
        ShearXAlongY.SetAxis(0, FVector(1.0f, FMath::Tan(AngleAboutZNorm), 0.0f));
		
        // Skew and scale the Root motion.
        FMatrix ScaledSkewMatrix = ScaleMatrix * ShearXAlongY;
        TargetRootMotion = ScaledSkewMatrix.TransformVector(RootMotionInSyncSpace);
    }
    else
    {
        // 如果期望的位移完全和动画位移方向正交,是最简单的情况，计算实际位移和动画位移的比例，
        // 移动方向直接使用当前位置到目标位置的朝向
        // Figure out ratio between remaining Root and remaining World. Then project scaled length of current Root onto World.
        float Scale = CurrentToWorldSync.Size() / CurrentToRootMotionSync.Size();
        float StepTowardTarget = RootMotionInSyncSpace.ProjectOnTo(RootMotionInSyncSpace).Size();
 		TargetRootMotion = CurrentToWorldSyncNorm * (Scale * StepTowardTarget)
    }
    
    return TargetRootMotion;
}
```

我们看到UE中的实现最为不同的一点在于，相比于直接缩放这次Delta的Rootmotion，还考虑到了期望位移方向和动画位移方向正交的情况：

* 对于正交的情况，则直接朝向正交方向位移缩放后的比例。
* 而对于常规情况而言，则基于AnimCurrentToRoot构建坐标系，将实际位移和动画位移的缩放比分解到坐标系的X轴和Y轴上，同时对于Y轴上的移动也能够处理移动的正负

总的来说UE的MotionWarp更为完善了，考虑了更多的情况。所以实际使用上也非常方便，实际生产中也大量使用了MotionWarping。

接着吐糟一句，MotionWarping的使用本身也就意味着程序化的理念在动画的使用过程中越来越普及，常规的AnimStateMacine方案真的可能难以应对越来越复杂的情况，和MotionWarping所希望解决的问题类似，角色也希望能在各种情况下表现出平滑也合理的动画表现。如何能够程序化的适应各种情况呢，简单的有DistanceMatching，复杂的有MotionMatching都必定会是越来越普及的解决方案。在UE5.3中PoseSearch作为MotionMaching的实现插件也越来越完善，后续会写一篇文章基于UE的实现，好好来说说MotionMatching。
