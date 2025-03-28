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
(the Amiga version), and by the gameplay in Knights of the Sky.

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


Link to forum post:  https://forums.unrealengine.com/t/raw-input-suddenly-stops/2119230

More Stuff coming!

