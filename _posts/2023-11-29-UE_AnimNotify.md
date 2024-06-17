---
layout: post
title:  "UE中的AnimNotify"
author: "ZZZZZY"
comments: true
tags: Dev
excerpt_separator: <!--more-->
---

UE中的Montage是驱动游戏逻辑中动画的重要组成部分，而其中AnimNotify和AnimNotifyState是使得动画驱动逻辑层重要的桥梁，本文将简单分析下AnimNotify的工作原理，更多会从UE的源码出发，因此全文的可读性会略差。 <!--more--> 

UE中AnimNotify逻辑主要分为两部分：收集和触发。其中收集发生在动画Evaluate之前，一般来说在MontageUpdate中，而触发则发生在Evaluate之后，即PostAnimEvaluation中，后续我们会就两个主题分开说明。

收集AnimNotifyEvents：

```c++
USkinnedMeshComponent::TickComponent
USkeletalMeshComponent::TickPose
	UAnimInstance::UpdateAnimation
        UAnimInstance::UpdateMontage
			FAnimMontageInstance::Advance
    			Montage::HandleEvent收集这个时间片的AnimNotifies
```

触发AnimNotifyEvents：

```c++
USkeletalMeshComponent::DispatchParallelTickPose
USkeletalMeshComponent::DispatchParallelEvaluationTasks
USkeletalMeshComponent::CompleteParallelAnimationEvaluation
USkeletalMeshComponent::PostAnimEvaluation
	UAnimInstance::DispatchQueuedAnimEvents
	UAnimInstance::TriggerAnimNotifies
```

#### AnimNotify的Trigger

我们先来看AnimNotify的触发逻辑，相比收集逻辑要更简单一些。

在上一段落我们可以看到当触发AnimNotify时代码的调用堆栈。

触发逻辑的核心在UAnimInstance::TriggerAnimNotifies中实现，此时所有触发的Notify已经添加到了NotifyQueue中，具体如何添加的我们后面再看，我们

先看看在这个函数的触发逻辑如何处理，核心复杂的逻辑在于NotifyState的处理，主要的思路是

1. ActiveAnimNotifyState 记录了上一帧Active的AnimNotifyStates
2. NewActiveAnimNotifyState记录这一帧Active的AnimNotifyStates，并将其从ActiveAnimNotifyState中移除
3. 这样ActiveAnimNotifyState就是这一帧需要执行End的AnimNotifyStates
4. 最后再将NewActiveAnimNotifyState覆盖ActiveAnimNotifyState

非常经典的类似于AliveList的做法，简化后的代码如下(不是完整的UE源码)：

```c++
void UAnimInstance::TriggerAnimNotifies(float DeltaSeconds)
{
    // AnimNotifyState freshly added that need their 'NotifyBegin' event called.
    TArray<const FAnimNotifyEvent *> NotifyStateBeginEvent;

    for (int32 Index=0; Index<NotifyQueue.AnimNotifies.Num(); Index++)
    {
        if(const FAnimNotifyEvent* AnimNotifyEvent = NotifyQueue.AnimNotifies[Index].GetNotify())
        {
            // AnimNotifyState
            if (AnimNotifyEvent->NotifyStateClass)
            {
                int32 ExistingItemIndex = INDEX_NONE;
                if (ActiveAnimNotifyState.Find(*AnimNotifyEvent, ExistingItemIndex))
                {
			      // 依然活跃的AnimNotifyState，则从ActiveAnimNotifyState中移除，防止后续的End逻辑
                    ActiveAnimNotifyState.RemoveAtSwap(ExistingItemIndex, 1, false); 
                }
                else
                {
				  // 如果是未触发过的AnimStateNotifyEvent，则添加准备触发Begin
                    NotifyStateBeginEvent.Add(AnimNotifyEvent);
                }
                NewActiveAnimNotifyState.Add(*AnimNotifyEvent);
                continue;
            }

            // Trigger non 'state' AnimNotifies
            TriggerSingleAnimNotify(NotifyQueue.AnimNotifies[Index]);
        }
    }

    // AnimStateNotifyEnd
    // 对上一帧活跃但是这一帧无效的AnimNotifyState执行End逻辑
    // Send end notification to AnimNotifyState not active anymore.
    for (int32 Index = 0; Index < ActiveAnimNotifyState.Num(); ++Index)
    {
        const FAnimNotifyEvent& AnimNotifyEvent = ActiveAnimNotifyState[Index];
        if (AnimNotifyEvent.NotifyStateClass && ShouldTriggerAnimNotifyState(AnimNotifyEvent.NotifyStateClass))
        {
            AnimNotifyEvent.NotifyStateClass->NotifyEnd();
        }
    }

    //  AnimStateNotifyBegin
	//  对新活跃的AnimNotifyState执行Begin逻辑
    for (int32 Index = 0; Index < NotifyStateBeginEvent.Num(); Index++)
    {
        const FAnimNotifyEvent* AnimNotifyEvent = NotifyStateBeginEvent[Index];
        if (ShouldTriggerAnimNotifyState(AnimNotifyEvent->NotifyStateClass))
        {
            AnimNotifyEvent->NotifyStateClass->NotifyBegin();
        }
    }

    //  AnimStateNotifyTick
    ActiveAnimNotifyState = MoveTemp(NewActiveAnimNotifyState);
    for (int32 Index = 0; Index < ActiveAnimNotifyState.Num(); Index++)
    {
        const FAnimNotifyEvent& AnimNotifyEvent = ActiveAnimNotifyState[Index];
        if (ShouldTriggerAnimNotifyState(AnimNotifyEvent.NotifyStateClass))
        {
            AnimNotifyEvent.NotifyStateClass->NotifyTick();
        }
    }
}
```

这里可以看出来，AnimState的驱动取决于这一次TriggerAnimNotifies时传入的AnimStateEvents，同时保证了NotifyEnd始终再NotifyBegin前执行，

但是这样依然可能存在的隐患是，如果在一帧中，需要NotifyEnd的NotifyEvent和需要NotifyStart的NotifyEvent同时存在，那么只用NotifyStart会触发，

而End在下一帧才触发，导致调用顺序的的错误。 PS：这也是需要在需要的时机设置AnimNotify为BranchPoint的原因

### AnimNotifyEvent的收集

AnimNotifyEvent的收集相比之下要复杂些，收集的时机在AnimEvalution之前。我们可以先简单思考下，如何收集AnimNotify，思路其实很简单，就是在每次UpdateAnim时，从上一次更新的时间到这次更新的时候内，获取所有时间重叠的AnimNotifies，AnimNotifyTrigger阶段会处理触发的逻辑，当然BranchPoint会马上特殊处理，我们后续再看。

明白了思路之后，再去看源码其实就是一个复杂化的版本，需要考虑动画的循环等等其他因素。一般来说的最佳实践是在montage中放AnimNotify，所以我们从Montage中的动画通知收集看起。



```c++
void FAnimMontageInstance::Advance(float DeltaTime, struct FRootMotionMovementParams* OutRootMotionParams, bool bBlendRootMotion)
{
	while (bPlaying && MontageSubStepper.HasTimeRemaining() && (++NumIterations < MaxIterations))
	{
		EMontageSubStepResult SubStepResult = MontageSubStepper.Advance(Position, &BranchingPointMarker);
		......
		const bool bHaveMoved = (SubStepResult == EMontageSubStepResult::Moved);
		if(bHaveMoved)
		{
			// Save position before firing events.
			if (!bInterrupted)
			{
				HandleEvents(PreviousSubStepPosition, Position, BranchingPointMarker);
			}
		}
	}
}
```
HandleEvent负责在AnimMontage的更新时收集时间片的AnimNotify，这里其实还包含了AnimNotify在MontageBlendIn和Out时的处理情况，让我们后面再关注，先关注HandleEvent中的实现。
```c++
void FAnimMontageInstance::HandleEvents(float PreviousTrackPos, float CurrentTrackPos, const FBranchingPointMarker* BranchingPointMarker)
{
    FAnimNotifyContext NotifyContext(TickRecord);
    
    // We already break up AnimMontage update to handle looping, so we guarantee that PreviousPos and CurrentPos are contiguous.
    Montage->GetAnimNotifiesFromDeltaPositions(PreviousTrackPos, CurrentTrackPos, NotifyContext);
    
    // Queue active non-'branching point' notifies.
	AnimInstance->NotifyQueue.AddAnimNotifies(NotifyContext.ActiveNotifies, NotifyWeight);
}
```

GetAnimNotifiesFromDeltaPositions是Montage获取AnimNotifies的核心方法，而Montage的我们知道是一种UAnimComposite结构，包含了多个AnimSequences，所以核心肯定是获取当前Montage时间段内的AnimSequence，这里会同时处理Montage的Loop、PlayBackward的情况，让我们跳过

 **UAnimComposite::GetAnimNotifiesFromDeltaPositions**(自行去源码中检索)，最终会调用到AnimSequence::GetAnimNotifiesFromDeltaPositions
```c++
void UAnimSequenceBase::GetAnimNotifiesFromDeltaPositions(const float& PreviousPosition, const float& CurrentPosition,  FAnimNotifyContext& NotifyContext) const
{
	for (int32 NotifyIndex=0; NotifyIndex<Notifies.Num(); NotifyIndex++)
	{
	    const FAnimNotifyEvent& AnimNotifyEvent = Notifies[NotifyIndex];
	    const float NotifyStartTime = AnimNotifyEvent.GetTriggerTime();
	    const float NotifyEndTime = AnimNotifyEvent.GetEndTriggerTime();
	
	    // Note that if you arrive with zero delta time (CurrentPosition == PreviousPosition), only Notify States will be extracted
	    if( (NotifyStartTime <= CurrentPosition) && (NotifyEndTime > PreviousPosition) )
	    {
	        if (NotifyContext.TickRecord)
	        {
	            NotifyContext.ActiveNotifies.Emplace(&AnimNotifyEvent, this, NotifyContext.TickRecord->MirrorDataTable);
	            NotifyContext.ActiveNotifies.Top().GatherTickRecordData(*NotifyContext.TickRecord); 
	        }
	        else
	        {
	            NotifyContext.ActiveNotifies.Emplace(&AnimNotifyEvent, this, nullptr);
	        }
	
	        const bool bHasFinished = CurrentPosition >= NotifyEndTime;
	        if (bHasFinished)
	        {
	            // KeyPoint!!!!!!!!!!!
	            NotifyContext.ActiveNotifies.Top().AddContextData<UE::Anim::FAnimNotifyEndDataContext>(true);
	        }
	    }
	}
}
```

可以看到这里实现逻辑非常简单，只要Notify的时间有重叠就会被捕获，但是对于NotifyEndTime小于当前时间的，也设置一个标记位，这会使得UAnimNotifyLibrary::NotifyStateReachedEnd返回为true。

但是这也会衍生出我们上一个主题最后提到的问题，如果在如果在一帧中，需要NotifyEnd的NotifyEvent和需要NotifyStart的NotifyEvent同时存在，那么只有NotifyStart会触发，而End在下一帧才触发，导致调用顺序的的错误，因为我们并没有考虑在一次采样的时间片里处理更小的时间片。

回到**FAnimMontageInstance::HandleEvents**中，我们现在获取到了时间片的AnimNotifies，全部存储在了NotifyContext.ActiveNotifies中，然后我们执行下列代码来真正添加需要的AnimNotifies

```
AnimInstance->NotifyQueue.AddAnimNotifies(NotifyContext.ActiveNotifies, NotifyWeight);
```

```
void FAnimNotifyQueue::AddAnimNotifiesToDest(bool bSrcIsLeader, const TArray<FAnimNotifyEventReference>& NewNotifies, TArray<FAnimNotifyEventReference>& DestArray, const float InstanceWeight)
{
    for (const FAnimNotifyEventReference& NotifyRef : NewNotifies)
    {
        const FAnimNotifyEvent* Notify = NotifyRef.GetNotify();
        if( Notify)
        {
            // only add if it is over TriggerWeightThreshold
            if (bNotify->TriggerWeightThreshold <= InstanceWeight )
            {
                // Only add unique AnimNotifyState instances just once. We can get multiple triggers if looping over an animation.
                // It is the same state, so just report it once.
                Notify->NotifyStateClass ? DestArray.AddUnique(NotifyRef) : DestArray.Add(NotifyRef);
            }
        }
    }
}
```

这里会传MontageCurrentWeight，Notify基于weight的过滤会在这里处理，Montage触发的情况下bSrcIsLeader会被设置为true