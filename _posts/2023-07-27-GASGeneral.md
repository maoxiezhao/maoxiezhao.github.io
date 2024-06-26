---
layout: post
title:  "GAS系统简易分析"
author: "ZZZZZY"
comments: true
tags: Dev GAS
excerpt_separator: <!--more-->
---

GAS是一个基于UE成熟的Gameplay框架，并且经历了许多上线项目的检验和迭代。学习GAS系统不仅仅是为了在UE中使用，也能够更好的了解和设计一个通用的Gameplay框架。<!--more-->

*PS:所以工作中项目在使用UE5，但是很遗憾并没有使用GAS*

[TOC]

## GAS的核心概念
GAS的核心概念在于将一些GamePlay系统中常见的概念抽象封装。 简单来说对于一个常见游戏的GamePlay，首先会存在一系列的属性合集，这些属性标识了游戏世界的状态，然后我们需要与游戏世界产生的交互，这些交互就是一系列定义好的行为，同时行为会触发一些效果，而效果会修改游戏世界状态。

举个法师施放火球术的例子，法师拥有SP，当消耗一点SP后，法师执行了释放火球术的咒语，火球术的咒语生效，产生了一个火球。火球碰撞到目标后会给目标施加一个新的火焰效果，从而被击中的目标会因为火焰效果而持续扣血并且表现出燃烧特效。

上面这个例子只是一个很简单的例子，任何一款游戏的Gameplay逻辑（战斗或者日常）都比上述描述要复杂的多，但是我们根据OOP的思想可以把上述的中的一些概念抽象出来，而抽象出来的这些概念对于所有游戏的Gameplay都是相通的。我们把游戏世界的属性抽象成GamePlayAttribute,属性集则是AttributeSet。我们把游戏世界中的行为抽象成GamePlayAbility，而因为行为所衍生的效果则抽象为GamePlayEffect,最终为了区分逻辑上的Effect和表现上Effect，我们再对表现上的效果另外定义为GamePlayCue。

Attribute,Abillity,Effect,Cue这些也同时是GAS这个系统核心组成。下面对这几个定义做进一步的介绍，内容基本取自GASDocument [1]

### GamePlayAttribute

Attribute是由FGameplayAttributeData结构体定义的浮点值, 其可以表示从角色生命值到角色等级再到一瓶药水的剂量的任何事物, 如果某项数值是属于某个Actor且游戏相关的, 你就应该考虑使用Attribute. Attribute一般应该只能由GameplayEffect修改, 这样ASC才能预测(Predict)其改变。

一个Attribute是由两个值 —— 一个BaseValue和一个CurrentValue组成的, BaseValue是Attribute的永久值而CurrentValue是BaseValue加上GameplayEffect给的临时修改值后得到的. 例如, 你的Character可能有一个BaseValue为600u/s的移动速度Attribute, 因为还没有GameplayEffect修改移动速度, 所以CurrentValue也是600u/s, 如果Character获得了一个临时50u/s的移动速度加成, 那么BaseValue仍然是600u/s而CurrentValue是600+50=650u/s, 当该移动速度加成消失后, CurrentValue就会变回BaseValue的600u/s。

### GamePlayAbility
GameplayAbility(GA)是Actor在游戏中可以触发的一切行为和技能. 多个GameplayAbility可以在同一时刻激活, 例如奔跑和射击. 其可由蓝图或C++完成. 
GameplayAbility示例:

* 跳跃
* 奔跑
* 射击
* 每X秒被动地阻挡一次攻击
* 使用药剂
* 使用火球术

当然技能本身除了仅仅执行一件事情以外，也可以设计的非常复杂，Aliblity可以是一个一段时间内的行为，也可以触发新的Ability。

### GamePlayEffect
GameplayEffect(GE)是Ability修改其自身和其他Attribute和GameplayTag的容器, 其可以立即修改Attribute(像伤害或治疗)或应用长期的状态buff/debuff(像移动速度加速或眩晕). UGameplayEffect只是一个定义单一游戏效果的数据类, 不应该在其中添加额外的逻辑.。

* GameplayEffect通过Modifier和Execution(GameplayEffectExecutionCalculation)修改Attribute.
* GameplayEffect有三种持续类型: 即刻(Instant), 持续(Duration)和无限(Infinite).

GameplayEffect一般是不实例化的, 当Ability或ASC想要应用GameplayEffect时, 其会从GameplayEffect的ClassDefaultObject创建一个GameplayEffectSpec, 之后成功应用的GameplayEffectSpec会被添加到一个名为FActiveGameplayEffect的新结构体, 其是ASC在名为ActiveGameplayEffect的特殊结构体容器中追踪的内容。

个人觉得GAS的核心其实还是在于Effect，一般游戏中Gameplay中最复杂的部分其实也就是Effect了，例如各种Buff/Debuff，伤害计算，元素反应等等。各个项目也对这块内容做了许多定制化功能，方面策划通过配置得到各种效果。相比之下，GE也做出了他们认为的最佳实践。实际上个人一直认为创建蓝图和在Effect的配置面板中配置数据，要比配置繁琐的表高效和直观许多。

### GamePlayCue

GameplayCue(GC)执行非游戏逻辑相关的功能, 像音效, 粒子效果, 镜头抖动等等. GameplayCue一般是可同步(除非在客户端明确执行(Executed), 添加(Added)和移除(Removed))和可预测的。

GamePlayCue一般会绑定安东GamePlayEffect上，基本上一笔带过。

## **GAS的执行流程分析**

接下来从一个技能触发开始，简单分析下在GAS框架下的执行流程，这里会跳过许多细节，只关注大致的执行流，以及几个模块之间的跳转，基本上会使用只有我自己才能看懂的伪代码表述。
这里定义一个FireAbility,这是一个火焰之手的法术，会直接点燃最近的目标。

#### GamePlayAbility的执行流程

#### GamePlayEffect的执行流程
在GamePlayAbility阶段触发了一个FireEffect，所以接下来到了GamePlayEffect的处理流程。

在上文就说过，GamePlayEffect实际上更像是定义的结构，真正执行Effect时还是会创建一个GameplayEffectSpec实例来执行,然后再根据GameplayEffectSpec构建AppliedActiveGE。我们从ApplyGameplayEffectToTarget开始看起：

```cpp
UAbilitySystemComponent::ApplyGameplayEffectToTarget(FireEffect)
	// 构建了Spec执行实例
	FGameplayEffectSpec	Spec(FireEffect, Context, Level);
	return ApplyGameplayEffectSpecToTarget(Spec, Target, PredictionKey);
		ApplyGameplayEffectSpecToSelf
			// ActiveGameplayEffects is FActiveGameplayEffectsContainer
			ActiveGameplayEffects.ApplyGameplayEffectSpec

FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec(in_effect_spec)
	// 创建了ActiveGE
	var AppliedActiveGE = new FActiveGameplayEffect(NewHandle, Spec, GetWorldTime(), GetServerWorldTime(), InPredictionKey);

    // 处理周期性的Effect，会创建定时器
	if (bSetPeriod && Owner)
		FTimerManager& TimerManager = Owner->GetWorld()->GetTimerManager();
		FTimerDelegate Delegate = FTimerDelegate::CreateUObject(Owner, &UAbilitySystemComponent::ExecutePeriodicEffect, AppliedActiveGE->Handle);
	else
		// 如果时瞬时的Effect则直接执行，当然这个执行逻辑实际代码在ApplyGameplayEffectSpecToSelf中
		// 这里为了简化流程同时区分周期性Effect,放在这个位置
		if (Spec.Def->DurationPolicy == EGameplayEffectDurationType::Instant)
			ExecuteActiveEffectsFrom

// 正常处理一个Effect
UAbilitySystemComponent::ExecuteGameplayEffect
	ExecuteActiveEffectsFrom(ActiveEffect.Spec);

// 周期性Effect执行的回调
UAbilitySystemComponent::ExecutePeriodicEffect(FActiveGameplayEffectHandle	Handle)
	ActiveGameplayEffects.ExecutePeriodicGameplayEffect(Handle);
		FActiveGameplayEffectsContainer::InternalExecutePeriodicGameplayEffect
			// Execute
			ExecuteActiveEffectsFrom(ActiveEffect.Spec);
```



```cpp
ExecuteActiveEffectsFrom
	// Capture our own tags.
	// TODO: We should only capture them if we need to. We may have snapshotted target tags (?) (in the case of dots with exotic setups?)
	SpecToUse.CalculateModifierMagnitudes();

	// ------------------------------------------------------
	//	Modifiers
	//		These will modify the base value of attributes
	// ------------------------------------------------------
	foreach Modifier in Modifiers
		InternalExecuteMod(Modifier)

	// ------------------------------------------------------
	//	Executions
	//		This will run custom code to 'do stuff'
	// ------------------------------------------------------
	CustomExecutionCalculation::Execute()

	// ------------------------------------------------------
	//	Invoke GameplayCue events
	// ------------------------------------------------------
	foreach GamePlayCue in GameplayCues
		InvokeGameplayCueExecuted_FromSpec
```

## 总结分析
总的来说GAS可以说是Gameplay模块的一种非常优雅的实现了，更不要说这本身就是一个开箱即用的，支持同步的框架，对于新的UE小型开发团队来说，就是最优选。

回到正题，从GAS系统设计中，我们可以总结出一些Gameplay系统框架的设计的经验。后续可能会再写一些关于GAS的内容，主要像是GAS的同步之类的，但是应该完全看心情把。还是可能会将以前写的一些内容补发上来，用于记录。

******************
#### 可参考的引用：  
[1] GASDocumentation https://github.com/tranek/GASDocumentation#concepts-ge
官方文档是最好最详细的参考资料

[2] GASDocument-CN https://github.com/BillEliot/GASDocumentation_Chinese#concepts-ge-ec
国人翻译版的官方文档