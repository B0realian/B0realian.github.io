---
layout: default
modal-id: 4
date: 2024-06-07
img: hellracer.png
alt: image-alt
project-date: June 2024
client: School Project
category: Games Development
description: 3D single player kart racing
---
This was our second group project which started immediatly after our solo 3D projects and lasted
8 weeks. This time, we were four programmers and four artists and I somehow managed to trick all
the others into creating this game in C++, even though the others had even less experience than
I did. The artists hadn't even been in Unreal before.
At any rate, I wanted to try something besides logic this time and so opted for particle FX,
obstacles and sound. But I wanted to control them with C++! Having had but a brief encounter
with Niagara in the previous project, I started by just learning how to make more advanced FX.
In the process I created a burning hoop for the player to jump through (someone had suggested
this)

![The Ring of Fire](img/portfolio/Hellracer/ring.gif "It burns, burns, burns")

which is probably the most complicated visual effect I made for this game due to the material
controlling the centre of the ring looking like this:

![Material nodes all over the place](img/portfolio/Hellracer/ringmaterial.png "Not BP nodes. Only material nodes.")

However, besides a collision box that registers when the player drives through the ring, there
was no code necessary. A much simpler effect that required at least some programming was one
of the obstacles in the game: fire geysers!

![Pillars of flame spurting from the ground](img/portfolio/Hellracer/geysers.gif "Bring marshmallows.")

Since you can see where the flames erupt, I wanted to at least add some randomness to their behaviour.

````cpp
if (bRandomised)
{
	TimeActive = FMath::RandRange(1.f, 5.f);
	TimeInactive = FMath::RandRange(3.f, 10.f);
	StartDelay = FMath::RandRange(0.f, 5.f);
}
else
{
	if (TimeActive < 1.f) TimeActive = 1.f;
	if (TimeInactive < 3.f) TimeInactive = 3.f;
}
````

````cpp
void AFlameObstacle::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

	if (StartDelay > 0.f)
	{
		StartDelay -= DeltaTime;
		return;
	}
	
	if (bActive)
	{
		ActiveTimer += DeltaTime;
		if (ActiveTimer >= TimeActive)
		{
			DeactivateSystem();
		}
	}
	else if (!bActive)
	{
		InactiveTimer += DeltaTime;
		if (InactiveTimer >= TimeInactive)
		{
			ActivateSystem();
		}
	}

	if (bCleanup)
	{
		ActiveTimer += DeltaTime;
		if (ActiveTimer >= TimeActive + 2.8f)
		{
			FlameObstacleEffect->DestroyComponent();
			FlameObstacleEffect = nullptr;
			bCleanup = false;
		}
	}
}
````

Another simple obstacle were large swinging axes over a bridge

![Axes swinging over a bridge](img/portfolio/Hellracer/axes.gif "Do not ask for whom the axes swing.")

their movement controlled thus:

````cpp
float AAxeAttack::GetSwingSpeed()
{
	float CurrentRotation = GetActorRotation().Pitch;
	if (FMath::Abs(CurrentRotation) >= SwingDistance && SwingTimer >= 0.2f)
	{
		Swing = -Swing;
		SwingTimer = 0.f;
		AxeAudio->FadeIn(0.3f);
	}
	SwingInertia = FMath::Cos(FMath::DegreesToRadians(GetActorRotation().Pitch));
	return (Swing * SwingSpeed * SwingInertia);
}

void AAxeAttack::Rotate(float DeltaTime)
{
	AddActorLocalRotation(SwingRotation * GetSwingSpeed() * DeltaTime);
}
````

Out of all the systems I created during this project, these axes were the one that kept haunting
me with strange behaviour. Since I swing them with cosine, they stop if they ever get to 90&deg;,
which is why I always made sure to set swingdistance to less than 90. Regardless of this, not
only would they intermittently get stuck, they would frequently start at 180&deg; and so swing
at potential birds above the bridge like hellish windscreen wipers. Trying to set start rotation
in BeginPlay did nothing, nor did setting it in the constructor. I still don't understand this
but I am guessing something happens when they are off camera and not being rendered. In the end,
I had to solve it by checking rotation in Tick and re-initialise them if they went above 90&deg;.

The obstacle that was most complicated to write code for was the trident attack.

![Flaming tridents hitting the track](img/portfolio/Hellracer/tridents.gif "In hell, the marshmallow is you!")

This was because I realised it would be hopeless to place them in the level except for where
they are supposed to strike the ground and I needed to time this with when the player is close
but also to give the player fair warning. So I ended up making a spawner as well as the system
itself where the system is actually both a warning system and the tridentsystem. Initially, I
made the spawner calculate a distance ahead along the players forward vector but even though
we designed the level to be fairly straight there, this became too chaotic and so I simply
added a random distance to a manually placed location for each trident.

````cpp
if (ACharacterInput* Driver = Cast<ACharacterInput>(OtherActor))
{
	if (bSetSpawnManually)
	{
		float RandomX = FMath::RandRange(-ManualRangeX, ManualRangeX);
		float RandomY = FMath::RandRange(-ManualRangeY, ManualRangeY);
		SpawnPosition = FVector(ManualLocation.X + RandomX, ManualLocation.Y + RandomY, ManualLocation.Z);
	}
	else
	{
		FVector DriverLocation = Driver->GetActorLocation();
		FVector DriverVector = Driver->GetActorForwardVector();
		SpawnPosition = DriverLocation + SpawnDistance * DriverVector;
		SpawnPosition.Z += TridentHeightAdjust;
		SpawnPosition.X += TridentXAdjust;
		SpawnPosition.Y += TridentYAdjust;
	}
	SpawnTridentAttack(SpawnPosition);
}
````

The spawned system begins with the warning effect and starts a timer for spawning the
trident. It is a bit crude and a couple of weeks later I learnt how to make timed
events in unreal but never had the time to update this code.

````cpp
void ATridentAttack::BeginPlay()
{
	Super::BeginPlay();
	TargetComponent->OnComponentBeginOverlap.AddDynamic(this, &ATridentAttack::OnCapsuleBeginOverlap);
	SystemLocation = GetActorLocation();
	SystemRotation = GetActorRotation();
	AActor* ActorRef = GetWorld()->GetFirstPlayerController()->GetPawn();
	CarDriver = Cast<ACharacterInput>(ActorRef);
	MovementVector = -FVector(	FMath::Sin(FMath::DegreesToRadians(SystemRotation.Yaw)) * FMath::Sin(FMath::DegreesToRadians(SystemRotation.Roll)),  
							    FMath::Cos(FMath::DegreesToRadians(SystemRotation.Yaw)) * FMath::Sin(FMath::DegreesToRadians(SystemRotation.Roll)),  
								FMath::Cos(FMath::DegreesToRadians(SystemRotation.Roll)));															 
	TridentLocation = SystemLocation - (TridentOffsetDistance * MovementVector);
	ImpactTime = (TridentOffsetDistance / (ProjectileSpeed * NiagaraVelocityScale.Z)) - 0.05f;
	WarningAudio->SetSound(WarningSound);
	TridentAudio->SetSound(FlyingSound);
	FireAudio->SetSound(BurningSound);
	ImpactAudio->SetSound(ImpactSound);
	
	SpawnWarning();
}

void ATridentAttack::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
		
	if (bAttackInitialising)
	{
		WarningTimer += DeltaTime;
		if (WarningTimer >= WarningTime)
		{
			SpawnProjectile();
			WarningAudio->Stop();
			bAttackStarted = true;
			bAttackInitialising = false;
			WarningTimer = 0.f;
		}
	}
	else if (bAttackStarted)
	{
		AttackTimer += DeltaTime;
		if (AttackTimer >= 9.f - WarningTime)
		{
			bAttackActive = false;
			bAttackStarted = false;
			AttackTimer = 0.f;
			DeactivateAndDestroy();
		}
		else if (AttackTimer >= ImpactTime && !bImpact)
		{
			bImpact = true;
			TridentAudio->Stop();
			ImpactAudio->Play();
			bAttackActive = true;
			CarDriver->StartScreenShake(1, 0.5f);
		}
	}
}
````

The trident itself is spawned above. Since the smoketrail needs to change behaviour
post impact, I am setting its variables here and let Niagara lerp between the two
positions:

````cpp
void ATridentAttack::SpawnProjectile()
{
	FRotator EffectRotation = FRotator(SystemRotation.Pitch, -SystemRotation.Yaw, SystemRotation.Roll);
	TridentEffect = UNiagaraFunctionLibrary::SpawnSystemAtLocation(this, Trident, TridentLocation, EffectRotation, FVector(1.f), true, true);
	FVector ProjectileVector = MovementVector * NiagaraVelocityScale;
	FVector SmokeOffsetFlying = MovementVector * -100.f;
	TridentEffect->SetNiagaraVariableVec3(FString("TridentVelocity"), ProjectileVector);
	TridentEffect->SetNiagaraVariableFloat(FString("TridentSpeed"), ProjectileSpeed);
	TridentEffect->SetNiagaraVariableVec3(FString("SmokeOffset"), SmokeOffsetFlying);
	TridentEffect->SetNiagaraVariableVec3(FString("SmokeOffsetStanding"), FVector(0.f, 0.f, 100.f));
	TridentAudio->Play();
	FireAudio->Play();
}
````

Another particle effect I want to mention is the tire particles:

![Kart driving with particles coming off its tires](img/portfolio/Hellraced/tireparticles.gif "Blood and dust.")

