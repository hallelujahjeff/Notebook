# Lyra 初始化流程
***

## 关卡初始化流程

#### Experience的初始化
首先进入一个关卡，以L_ShooterGym为例
Lyra使用了GameExperience的概念用来配置一场游戏的各种玩法，在概念上与GameMode类似，但更偏向于策划进行灵活的资产配置：
![](./Image/2025-07-26-15-01-28.png)
进入关卡后默认的Pawn对象类在这里可以设置。

Experience类的选择是在GameMode初始化InitGame时决定的，GameMode会按照优先级从多个配置处读取ExperienceId：
```cpp
void ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
	FPrimaryAssetId ExperienceId;
	FString ExperienceIdSource;

	// Precedence order (highest wins)
	//  - Matchmaking assignment (if present)
	//  - URL Options override
	//  - Developer Settings (PIE only)
	//  - Command Line override
	//  - World Settings
	//  - Dedicated server
	//  - Default experience

	UWorld* World = GetWorld();

	if (!ExperienceId.IsValid() && UGameplayStatics::HasOption(OptionsString, TEXT("Experience")))
    // ...
```


## 角色初始化流程
Lyra的角色（小蓝人）的继承链为:B_Hero_ShooterMannequin -> B_HeroDefault -> CharacterDefault -> LyraCharacter (C++)
其中从B_HeroDefault开始，加入了LyraHero组件，这个控件会处理到角色输入这一层。 而B_Hero_ShooterMannequin则是实际的游戏对象，包括了输入、动画、能力等。


### 角色输入
Lyra使用EnhancedInputSystem来处理玩家的输入逻辑，可以认为是UE4InputSystem的强化版。简单来说EnhancedInputSystem将输入配置拆为三个粒度的模块:
- InputAction: 和UE4的Input系统一致，将角色的输入抽象为具体的事件，和输入解耦。 比如移动、跳跃、射击都属于InputAction
- InputMappingContext(IMC)：用于将InputAction和控制器的键位映射起来，还能够配置每个键位具体的映射方式，比如轴映射、按下/释放、长按等，功能强大
![](./Image/2025-07-26-22-57-20.png)
- PlayerMappingInputConfig(PMI)：PMI是IMC的更进一层封装，也是实际注册输入时，使用的对象。它可以包含多个IMC，通过调用`UEnhancedInputLocalPlayerSubsystem->AddPlayerMappableConfig()`动态为操控角色注册一套键位

简单来说，让角色能够处理玩家输入，需要做两件事：
1. 为角色绑定一套PMI，通过`UEnhancedInputLocalPlayerSubsystem->AddPlayerMappableConfig()`
```cpp
	if (const ULyraInputConfig* InputConfig = PawnData->InputConfig)
	{
		// Register any default input configs with the settings so that they will be applied to the player during AddInputMappings

		// DefaultInputConfigs是记录在HeroComponent的默认键位
		for (const FMappableConfigPair& Pair : DefaultInputConfigs)
		{
			if (Pair.bShouldActivateAutomatically && Pair.CanBeActivated())
			{
				FModifyContextOptions Options = {};
				Options.bIgnoreAllPressedKeysUntilRelease = false;
				// Actually add the config to the local player							
				Subsystem->AddPlayerMappableConfig(Pair.Config.LoadSynchronous(), Options);	
			}
		}
```

2. 通过`BindAction`，将输入产生的InputAction与实际逻辑绑定起来
```cpp
	LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Move, ETriggerEvent::Triggered, this, &ThisClass::Input_Move, /*bLogIfNotFound=*/ false);
	LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Look_Mouse, ETriggerEvent::Triggered, this, &ThisClass::Input_LookMouse, /*bLogIfNotFound=*/ false);
	LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Look_Stick, ETriggerEvent::Triggered, this, &ThisClass::Input_LookStick, /*bLogIfNotFound=*/ false);
	LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Crouch, ETriggerEvent::Triggered, this, &ThisClass::Input_Crouch, /*bLogIfNotFound=*/ false);
	LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_AutoRun, ETriggerEvent::Triggered, this, &ThisClass::Input_AutoRun, /*bLogIfNotFound=*/ false);
```