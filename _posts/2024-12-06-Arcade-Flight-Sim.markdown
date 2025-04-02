---
layout: default
modal-id: 6
date: 2024-12-06
img: arcadeflight.png
alt: image-alt
project-date: December 2024
client: School Project
category: Games Development
description: Arcade Flight Sim
---
Hard on the heels of the VR project came our last project before starting our internships. We had
8 weeks to showcase what we had learnt after a year of studies. How and what to do was left to
ourselves to decide. So I decided to keep learning new things in Unreal Engine!

My goals for this project was to create my own character movement (i.e. to not use the Character
Movement Component supplied by the engine), to utilize raw input controls, to write a decent AI,
and to get the hang of creating UI widgets in code. In the process, I also learnt how to
leverage dynamic materials, something I ended up spending more time on than several of my other
goals. I was also forced to spend quite a lot of time in finding an internship, a process that
I, in all honesty, found harrowing and depressing. If you ever need to fill your ears with the
bitter moans of an otherwise enthusiastic programmer, you know how to contact me.

So, a natural fit for my combined goals was a flight simulator! I've loved flight sims since
playing so many of them on my Amiga 500. I marvelled at the performance of F29 Retaliator which
was written entirely in Assembler, captivated by the story and progression in Wing Commander
(the Amiga version), and by the gameplay in Knights of the Sky. To temper your expectations
straight off the bat, I should add that this ended up being more of a proof of concept than a
proper game.

![The proof is in the pudding.](img/portfolio/ArcadeFlight/AFS_kill.gif "Who needs Bruckheimer?")

First off, I wrote player movement more or less how I pictured I wanted it. I was going to add
raw input at a later stage and figured I'd have to modify the feel then. The code snippet below
is the final result, although, as I'd come to realise early in the project, nothing would truly
reach a stage where I was wholly satisfied due to the time constraint.

````cpp
void APilot::StickControls(const FInputActionValue& Value)
{
	const FVector2D InputVector = Value.Get<FVector2D>();
	const float RollValue = InputVector.X * RollRate * RollInputLag;
	CurrentRoll = GetActorRotation().Roll;
	SineRoll = SineD(CurrentRoll);
	if (InputVector.Y < 0.f)
		PitchDownLag = 1.f - FMath::Abs(((1.f - PitchDownCoeff) * SineRoll));
	else
		PitchDownLag = 1.f;
	const float PitchValue = InputVector.Y * PitchRate * PitchDownLag;
	AddActorLocalRotation(FRotator(PitchValue, 0.f, RollValue));

	RollInputLag += 0.01f;
	if (FMath::Abs(InputVector.X) < 0.1f)
		RollInputLag = 0.1f;
	else if (RollInputLag >= 1.f)
		RollInputLag = 1.f;
	else if (RollInputLag >= 0.7f)
		RollInputLag += 0.09f;
	else if (RollInputLag >= 0.4f)
		RollInputLag += 0.04f;
	else if (RollInputLag >= 0.2f)
		RollInputLag += 0.01f;
}

void APilot::StickInputEnd()
{
	RollInputLag = 0.1f;
}
````

While the RollInputLag code isn't terrible performance-wise it *does* still make my eyes water and
it was one of those things I was going to fix. The end result provides some wonderful feel to
rolling however, so that code was always going to be rewritten after everything else was taken care
of.

The thrust was even more simplistic:
````cpp
void APilot::ThrottleUp(const FInputActionValue& Value)
{
	if (AfterBurning) return;
	TargetThrustRatio += 0.05f;
	if (TargetThrustRatio > 1.f) TargetThrustRatio = 1.f;
	TargetThrust = TargetThrustRatio * MaxThrust;
	CrosshairPtr->UCrosshairWidget::SetThrustValue(FText::AsNumber(TargetThrustRatio * 100));
}
````

Next, it was time to hook up HOTAS! It all seems fairly straight forward in UE: enable the Raw
Input Module and add mapping with modifiers, much like with normal input. And after some initial
confusion about how the arrays map to the module (I first thought there was a separate input array
for each controller) it just worked! I spent a couple of days fine tuning and then it stopped
working. A whole day passed going over every possible setting and trying every conceivable value
in every conceivable combination. Another day, frantically searching forums, disabling and
re-enabling Raw Input, resetting the usual suspects (Intermediary, Saved, and Binaries). I took
another day starting a new project and recreating all my classes and finally, everything worked
again!

Until the next day when suddenly nothing worked again. Having been ultra careful not to interfere
with anything input related, I was beginning to suspect the problem lay with the engine and so
began to look for where input is cached. Fortunately, the first and most obvious place to look
proved to be sufficient! Defaultinput.ini contains all settings in plain text, including raw input
if you have made any changes from default. Simply deleting the lines referring to raw input and
then reimporting the settings in the editor got everything to work again. I made a forum post
detailing the steps although have not heard anything since.

https://forums.unrealengine.com/t/raw-input-suddenly-stops/2119230

Now that I had a fix (however the raw input worked at time of Build persists in the Build) I was
back on track! The downside was that with spending nearly two weeks finding an internship (I had
to write a program as a work sample) and another week debugging raw input, I was now about halfway
through the project and had *very* little to show for it. So I decided to quickly add some
destructible buildings before creating enemies. All I needed was a couple of cubes and a decent
particle explosion, the latter of which would be repurposed for the enemies anyway!

That's when I stumbled over dynamic materials. I was immediately struck by how incredibly versatile
the system is and decided to shift focus towards learning how to leverage it.

````cpp
void ADestructibleObjectSimple::RandomizeSettings()
{
	MaterialDissolveAlpha = FMath::RandRange(MinAlpha, MaxAlpha);
	int RotationRate = FMath::RandRange(0, 3);
	DissolveTextureRotation = 0.25f * RotationRate;
}

void ADestructibleObjectSimple::SetDynamicMaterial()
{
	DynamicMaterial = DestructibleMesh->CreateDynamicMaterialInstance(0);
	DynamicMaterial->SetScalarParameterValue(FName("Specular"), Specular);
	DynamicMaterial->SetScalarParameterValue(FName("Roughness"), Roughness);
	DynamicMaterial->SetVectorParameterValue(FName("BaseColour"), BaseColour);
	DynamicMaterial->SetScalarParameterValue(FName("Rotation"), DissolveTextureRotation);
	DynamicMaterial->SetScalarParameterValue(FName("Alpha"), 0.f);
	DestructibleMesh->SetMaterial(0, DynamicMaterial);
}
````

I started with a simple version that starts by randomizing its "destroyed" state. Basically, I am
just dissolving the texture by discarding pixels according to an alpha-mask. When the mesh is hit,
I spawn a particle system and adjust visibility.

````cpp
void ADestructibleObjectSimple::OnHit(Blah blah)
{
	if (bDestroyed) return;

	DynamicMaterial->SetScalarParameterValue(FName("Alpha"), MaterialDissolveAlpha);
	if (ParticleSystem)
		ParticleEffect = UNiagaraFunctionLibrary::SpawnSystemAtLocation(this, ParticleSystem,
			GetActorLocation());
	if (AudioComponent)
		AudioComponent->Play(0.f);
	this->SetActorEnableCollision(false);
	bDestroyed = true;
}
````

Depending on the target alpha-value, more or less of the texture will be visible. Since I am also
randomizing the texture's rotation, I am able to instantiate quite a few of these without it
looking too repetitive, even though I just used the first noise texture I found as alpha mask.

![For all the window haters](img/portfolio/ArcadeFlight/AFS_destruction.gif "Saving on window cleaning.")

I decided to make a slightly more ambitious variant while at it. No, I mean it, there was no time
to get truly fancy with this, so it still *looks* pretty basic.

````cpp
void ADestructibleObject::SetDynamicMaterial(int i)
{
	DynamicMaterials[i] = DestructibleWalls[i]->CreateDynamicMaterialInstance(0);
	DynamicMaterials[i]->SetScalarParameterValue(FName("Specular"), Specular);
	DynamicMaterials[i]->SetScalarParameterValue(FName("Roughness"), Roughness);
	DynamicMaterials[i]->SetVectorParameterValue(FName("BaseColour"), BaseColour);
	DynamicMaterials[i]->SetScalarParameterValue(FName("EdgeSize"), EdgeSize);
	DynamicMaterials[i]->SetScalarParameterValue(FName("EdgeIntensity"), EdgeIntensity);
	DynamicMaterials[i]->SetVectorParameterValue(FName("EdgeColour"), EdgeColour);
	if (bDestroyed)
		DynamicMaterials[i]->SetScalarParameterValue(FName("Alpha"), MaterialDissolveAlpha);
	else
	{
		DynamicMaterials[i]->SetTextureParameterValue(FName("DissolveTexture"), DissolveTextures[i]);
		DynamicMaterials[i]->SetScalarParameterValue(FName("Alpha"), 0.f);
	}
	DestructibleWalls[i]->SetMaterial(0, DynamicMaterials[i]);
}
````

I was going to create an event handler but the prototype tick-version is still there:

````cpp
void ADestructibleObject::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

	if (bCooling && DissolveDelay > 0.f)
	{
		DissolveDelay -= DeltaTime;
	}
	else if (bCooling)
	{
		EdgeIntensity -= DeltaTime * EdgeCoolingRate;
		for (int i = 0; i < 5; i++)
			SetDynamicMaterial(i);
		if (EdgeIntensity < 1)
			bCooling = false;
	}
}

void ADestructibleObject::OnHit(A lot of parameters)
{
	if (bDestroyed) return;

	HitPoints--;
	if (HitPoints <= 0)
	{
		if (ParticleSystem)
		{
			ParticleEffect = UNiagaraFunctionLibrary::SpawnSystemAtLocation(this, ParticleSystem,
				GetActorLocation() + ParticleSpawnOffset);
			ParticleEffect->SetColorParameter(FName("DebrisColour"), BaseColour);
		}
		if (AudioComponent)
			AudioComponent->Play(0.f);
		this->SetActorEnableCollision(false);
		bCooling = true;
		bDestroyed = true;
	}
}

````

So, the main principle is the same: use particle effect (an explosion with flying debris in this
case) to hide the change of alpha. This time I also added edge colour to produce a glowing ember
effect that wanes with time.

![First: explosion!](img/portfolio/ArcadeFlight/AFS_explode1.png "KA-BOOM!")
![Second: Fire & smoke.](img/portfolio/ArcadeFlight/AFS_explode2.png "Burning down the house!")
![Third: Charred remains.](img/portfolio/ArcadeFlight/AFS_explode3.png "Looks well done to me.")

Writing code is one thing. Implementing it, I find, is more laborious. It took way longer than
anticipated to populate the scene with these objects and when it was finally done I had precious
little time to start and finish two remaining and crucial issues: enemies and UI. I decided the
most important part was enemies and so started writing an AI. I discovered even a simple AI requires
a lot of testing and progress was slow. When I finally had something that worked more or less as
intended (but still required a LOT of tweaking) I shelved it until later. Of course, meaning I
never had the time to finish it.

````cpp
void AEnemyFlyboy::EvaluateState()
{
	if (CurrentState == EState::S_AvoidCollision) return;
	if (Health <= 5)
		CurrentState = EState::S_Retreating;
	
	if (TailedByPlayer())
	{
		switch (CurrentState)
		{
			case EState::S_Patrolling:
				if (DistanceToTargetSq(PlayerLocation) < Sq(InterceptDistance))
				{
					CurrentState = EState::S_GainAltitude;
					PingInterval = PingIntervalIntercept;
					GainAltBehaviour();
				}
				else
					PatrolBehaviour();
				break;
			case EState::S_Intercepting:
				if (DistanceToTargetSq(PlayerLocation) > Sq(InterceptDistance))
				{
					CurrentState = EState::S_Patrolling;
					PingInterval = PingIntervalPatrol;
					PatrolBehaviour();
				}
				else if (DistanceToTargetSq(PlayerLocation) < Sq(AttackDistance))
				{
					CurrentState = EState::S_Evading;
					PingInterval = PingIntervalAttack;
					EvasiveBehaviour();
				}
				else
				{
					CurrentState = EState::S_GainAltitude;
					GainAltBehaviour();
				}
				break;
			case EState::S_Attacking:
				if (DistanceToTargetSq(PlayerLocation) > Sq(AttackDistance))
				{
					CurrentState = EState::S_GainAltitude;
					PingInterval = PingIntervalIntercept;
					GainAltBehaviour();
				}
				else
				{
					CurrentState = EState::S_Evading;
					EvasiveBehaviour();
				}
				break;
			case EState::S_GainAltitude:
				if (DistanceToTargetSq(PlayerLocation) > Sq(InterceptDistance))
				{
					CurrentState = EState::S_Patrolling;
					PingInterval = PingIntervalPatrol;
					PatrolBehaviour();
				}
				else if (DistanceToTargetSq(PlayerLocation) < Sq(AttackDistance))
				{
					CurrentState = EState::S_Evading;
					PingInterval = PingIntervalAttack;
					EvasiveBehaviour();
				}
				else
					GainAltBehaviour();
				break;
			case EState::S_Evading:
				if (DistanceToTargetSq(PlayerLocation) > Sq(AttackDistance))
				{
					CurrentState = EState::S_GainAltitude;
					PingInterval = PingIntervalIntercept;
					GainAltBehaviour();
				}
				break;
			case EState::S_Retreating:
				if (DistanceToTargetSq(PlayerLocation) < Sq(AttackDistance))
				{
					CurrentState = EState::S_Evading;
					PingInterval = PingIntervalAttack;
					EvasiveBehaviour();
				}
				else
					RetreatBehaviour();
				break;
			default:
				break;
		}

		// You get the idea...
````

In UE, there are a lot of finished functions for AI that are obviously far superior to this. The
point was to write my own code, however short on time I was. For what it's worth, I consider this
alright if taking into account that it was my first enemy AI and that I only had a week to cobble
the whole enemy class together. As it is, the enemy will detect the player when within a certain
distance, aim for you and fire, until you get too close at which point the enemy will try to veer
away. If damaged to the point that it leaves a smoke trail, it will try to avoid combat altogether.

![Stupid AI!](img/portfolio/ArcadeFlight/AFS_AI.gif "Only way to win is to make stupid AI, right?")

Last, it was time to add a little UI. There are countless tutorials on creating widgets with BP,
of course, but I knew my Blueprint-skills were barely extant by now and so I spent a day learning
Slate. That was roughly enough for me to realise I wasn't going to make it and so I pivoted to
creating it in C++ instead. I had two days, frantically learning how to, and then, of course, it
took about half an hour to write my two UI-classes.

````cpp
#pragma once

#include "CoreMinimal.h"
#include "Components/TextBlock.h"
#include "Components/Image.h"
#include "Blueprint/UserWidget.h"
#include "CrosshairWidget.generated.h"


UCLASS(ABSTRACT)
class ARCADEFLIGHTSIM_API UCrosshairWidget : public UUserWidget
{
	GENERATED_BODY()

protected:
	UPROPERTY(BlueprintReadWrite, meta = (BindWidget)) UImage* CrosshairImage;
	UPROPERTY(BlueprintReadWrite, meta = (BindWidget)) UTextBlock* ThrustText;
	UPROPERTY(BlueprintReadWrite, meta = (BindWidget)) UTextBlock* SpeedText;
	UPROPERTY(BlueprintReadWrite, meta = (BindWidget)) UTextBlock* HealthText;
	UPROPERTY(BlueprintReadWrite, meta = (BindWidget)) UTextBlock* AltitudeText;

public:
	void SetThrustValue(FText NewValue);
	void SetSpeedValue(FText NewValue);
	void SetHealthValue(FText NewValue);
	void SetAltitudeValue(FText NewValue);
};

#include "CrosshairWidget.h"
#include <Kismet/GameplayStatics.h>
#include "Engine/World.h"


void UCrosshairWidget::SetThrustValue(FText NewValue)
{
	ThrustText->SetText(NewValue);
}

void UCrosshairWidget::SetSpeedValue(FText NewValue)
{
	SpeedText->SetText(NewValue);
}

void UCrosshairWidget::SetHealthValue(FText NewValue)
{
	HealthText->SetText(NewValue);
}

void UCrosshairWidget::SetAltitudeValue(FText NewValue)
{
	AltitudeText->SetText(NewValue);
}
````

Of course, the code for a widget is perhaps a tenth of the work required to make it, at least
straightforward ones like mine.

No game project plan survives contact with reality. I made far too grandiose plans for this
project and came out of it feeling like I had just fallen on my face. It does still look like I
got too little done for 8 weeks. Considering I only managed to spend 5 weeks, however, it looks
considerably better. At least that's what I tell myself!

![Is it a bird? Is it a plane? Oh.](img/portfolio/ArcadeFlight/AFS_flying.gif "This is me going from workplace to workplace applying for a job.")