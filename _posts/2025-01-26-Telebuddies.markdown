---
layout: default
modal-id: 8
date: 2025-01-26
img: telebuddies.png
alt: telebuddies_altsm.png
project-date: January 2025
client: Game Jam
category: Games Development
description: Portal-like floor-is-lava puzzle-game
---
A month and a half into my internship I had a craving for games creation and signed up for GGJ-25. I teamed up with a couple of class mates and we were joined at the jam by a professional dev. Unfortunately for him (and/or us) we had decided to make the game in UE whereas he only had experience with Unity but he turned out to be a great level designer and this way it was less stressful for him I suppose.

Something that turned out to later cause half an hour of panic was that my class mates wanted to use blueprints while I felt much more comfortable with C++. Naturally, I told them it was no problem since everything is eventually turned into a BP-class anyway. That promise was what made me stressed, I suppose, since naturally we discovered it *could* be a problem.

This year's theme was "Bubbles" and while brainstorming I mentioned that a bubble can refer to a self-contained universe and what if we could base the game on a mechanic that lets the player travel between realms? This idea was soon mutated into something much more akin to Portal, but where you shoot bubbles to which you can teleport.

Since I was more experienced with Niagara, I ended up doing most of the FX (not that there was much of it), such as the air flow effect from the fans.

![Bubbles need air](img/portfolio/TeleBuddies/TB_fan.gif "I guess this is fan art.")
![Captured air](img/portfolio/TeleBuddies/TB_airflow.png "The air is still. Get it?")

I started off trying to use sprites, which of course did not work at all. It is probably obvious to anyone who has ever wanted to create something similar in Niagara, but Ribbons is what gets the job done. So, in essence, a bunch of ribbons with transparency, a lot of curl noise and refraction and there it is. Since it was a little hard to make out what direction the ribbons travel in, I also added some sprites as dust particles.

Clearly, we also needed bubbles, which material became a joint creation between me and one of my class mates. He did the hard part, the shifting colour, while I was busy with something else, and I merely added fresnelled refraction. My team mates wanted me to up the index to ludicrous level, so I did.

![Bubble!](img/portfolio/TeleBuddies/TB_bubble.gif "Now all we need is Bobble.")

And then, of course, we needed a burst effect. I spent a little too much time trying to get the colours to match the feeling of the bubble and in the end the photo realism of the bubble didn't match the low-poly pastel burst anyway. That's game jam art I guess.

![Pop!](img/portfolio/TeleBuddies/TB_burst.gif "Don't poppa me or I'll poppa you!")

Next up, we needed a button to interact with the various obstacles in the game, so I quickly made a messy mesh in Blender and slapped some code to it. We wanted a do-anything button to make level editing easier, so I accomodated all our objects and let the level designer check whatever target he wanted in the editor.

```cpp
void AFloorButton::BeginPlay()
{
	Super::BeginPlay();
	if (bOverlappingNotHitting)
	{
		Collider->OnComponentBeginOverlap.AddDynamic(this, &AFloorButton::OnOverlap);
		Collider->OnComponentEndOverlap.AddDynamic(this, &AFloorButton::OnOverlapEnd);
	}
	else
		MainMesh->OnComponentHit.AddDynamic(this, &AFloorButton::OnHit);
	Collider->SetGenerateOverlapEvents(bOverlappingNotHitting);

	StartingPosition = FVector(GetActorLocation());
	FullyPressedPosition = StartingPosition;
	FullyPressedPosition.Z -= 10.f;
}

void AFloorButton::FindTarget(bool bOverlapping)
{
	if (AudioComponent && !AudioComponent->IsPlaying())
	{
		AudioComponent->SetSound(PressButtonSound);
		AudioComponent->Play(0.f);
	}
	
	switch (ButtonTarget)
	{
		case ETarget::S_Portal:
			SpawnPortal();
			break;
		case ETarget::S_Lasers:
			ToggleLasers();
			if (!bOverlapping)
				GetWorld()->GetTimerManager().SetTimer(UnpressTime, this, &AFloorButton::Unpress, 1.f, false);
			break;
		case ETarget::S_Fans:
			ToggleFans();
			if (!bOverlapping)
				GetWorld()->GetTimerManager().SetTimer(UnpressTime, this, &AFloorButton::Unpress, 1.f, false);
			break;
		default:
			break;
	}
}
``` 

This was my only programming contribution and it very nearly didn't survive when we realised that none of us had a clue how to connect a cpp with pure BP-classes. After some frantic searching through the documentation (all while my team mates figured it would be better if they scripted the class in blueprints instead) I found the magic bullet:

```cpp
UFUNCTION(BlueprintImplementableEvent) void ToggleLasers();
UFUNCTION(BlueprintImplementableEvent) void ToggleFans();
```

Marking the functions as BP implementable meant we could add them to the Event Graph of the BP, thus running both script and code at the same time.

The game is currently available for download on  
https://globalgamejam.org/games/2025/telebuddies-0-1