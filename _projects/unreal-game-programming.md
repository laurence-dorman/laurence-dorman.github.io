---
layout: post
title: Unreal Gameplay Programming 🎮🏎
description: Gameplay programming in Unreal Engine 4 | Blueprints / C++
date: 2022-05-25
---

## Table of contents
{:.no_toc}
* TOC
{:toc}

&nbsp;

***

&nbsp;

## [Itch.io link](https://matthewirving.itch.io/delivery-turbo-s)

## Video demonstration
<iframe width="560" height="315" src="https://www.youtube.com/embed/nljhB-kutGY" title="YouTube video player" frameborder="1" allowfullscreen></iframe>

## [1. Programming work](#1-programming-work)

### Feature scripting

#### Boost mechanic

The boost mechanic in the game functions as a risk vs reward system. The player can boost their speed, resulting in a faster time, but at the cost of a higher chance of crashing. Initially the boost functioned by increasing the player's max speed and using 50% of the player's fuel or less if the player had < 50 fuel. 

It was decided by the team that the boost should feel more explosive, so I added an initial impulse to the player's velocity when boosting (1.5x).

Gameplay wise it felt off being able to spam press boost when the player had very little fuel, so the team decided that the player must have at least the fuel cost to be able to boost, now 25% of total fuel.

~~~cpp
// boost pseudocode
if (InputAction.Boost){
	if (fuel > fuel_cost){
		StartBoost();
		FuelManager.BurnFuel();
		Delay(0.5f);
		StopBoosting();
	}
}
~~~

***

#### Package system

It’s a delivery game, so to help push the narrative to the player, it makes sense that they can physically see the packages on the ship. This was done by placing the package assets in the level, then they are attached to the player component inside their own `BeginPlay` scripts using the “`AttachActorToComponent`” function. The reason I did it this way, was so that the packages can have their own scripts in their own class, found in “`BP_Package`”, for flying off, showing the player that they lost a package.

Initially the game only featured a HUD representation of the remaining packages that the player possessed, but it was decided by the team that there needs to be more emphasis on the packages themselves, as the game is about delivering them successfully.

Packages fly off when their member variable “`Falling`” is set to true, then, until `DeathTimer` is depleted, we add a relative location and rotation to the package, the scale of which is randomized for each package. Then once `DeathTimer` is depleted, we will spawn a particle emitter “explosion” and play an explosion sound effect, then call `DestroyActor`. This works well to show the player that they have lost a package.

***

#### Turning system

`BP_Turn` is a placeable actor, it uses a provided enum to set the desired position and rotation of the player. The enum is used in the `BeginPlay` event of the `BP_Turn` asset. It sets the member variables `DesiredPos` and `DesiredRot` based on the position of the `BP_Turn` asset in the level, and the value of the enum `Turn`. The idea basically being that the `DesiredPos` will be the center of the desired face of the cube we want to turn to.

[![Turn system](/assets/images/delivery-turbo-s/turn-zone.png)](/assets/images/delivery-turbo-s/turn-zone.png)

The turn manager turns the player based on values retrieved from the turn zone. It manages the turning of the player, lerping it's rotation and position to the desired values. This is done using a manager to ensure that the player is not turning while already turning, doing the turning logic inside of the individual Turn actor scripts would result in the player turning multiple times if they were to enter multiple turn zones at once.

The turn assets are designed in such a way that the level designer can place them easily and adjust the parameters to get the desired effect. The turn manager is designed in such a way that it can be used for any number of turn zones, and the turn zones can be placed anywhere in the level.

The speed at which the player will turn/move inside the turn zone is determined by the movement speed of the player on entry and how far the player is to the center of the “movement circle”, so if we are closer to the corner we will turn faster.

***

#### Turn Spline System

Previous turn system had some issues with geometry clipping, so a turning system using unreal’s built in “spline system” and a secondary camera inside the turn zone would solve this issue. 

The spline system allows the designer to create a predetermined path, positions and rotations, we can make use of this to add more fluidity to the turn system. There were issues with the previous turn system where sometimes the camera would clip through geometry, this can be solved by making use of a secondary camera inside the turn, which we can smoothly interpolate to in a cinematic style.

> Image shows the spline turn zone actor placed in the level. In white is the path that will be followed by the player. The secondary camera is shown at the top right of the zone outlined in orange. Collision zone is shown in orange outline.

[![Turn spline system](/assets/images/delivery-turbo-s/turn-spline.png)](/assets/images/delivery-turbo-s/turn-spline.png)

Use a timeline node to control the player’s position on the spline with respect to time.

Adjustable editor parameters: play rate, rate at which the timeline will play, 1.0 = default speed.

Basic functionality of the turn spline system:

1. While Turning
2. Update `Timeline` object, retrieve alpha value.
3. Use alpha value in `LerpAlongSpline` function, which retrieves the position and rotation of the spline at that alpha value, and then sets the position and rotation of the player.
4. Rotate the TurnSpline’s camera component toward the player, so that it follows the player.
5. We switch between the main camera and the `TurnSpline`’s camera component by using Unreal’s function “`SetViewTargetWithBlend`”, which takes in an `Actor` reference: we pass in “Self” inside the `TurnSpline` script which the function will use to find the `BP_TurnCameraActor` component, and then “lerp” to that camera using `BlendTime` variable.
6. To switch back to the main camera, we pass the `SetViewTarget` with blend function the player controller actor reference, which will retrieve the main camera and blend back to it.

***

#### Camera system

Initially, the camera’s FOV was static. However, the team decided that the element of speed should be further emphasised, I did this by setting the camera’s FOV based on the player’s current speed, we lerp from a default FOV to the max FOV with the alpha value set by speed divided by boost speed: so that when we are at the boost speed, we will have the max fov.

To add more “juice” to the game, I decided to add a shaking when the player collides with an obstacle. This is done by setting the camera to a random rotation each frame while we are “shaking”, the amplitude of which is determined by a timer: so that as the timer decreases, the shaking amplitude also decreases. This works well to emphasise to the player that they have taken damage, or lost a package.

Although didn’t make it into the final game, a blurring feature was added to work with the steam system, which “blurs” the camera by setting the focal distance of the camera based on a timer.

***

#### Steam system

- Placeable actor that makes use of Unreal’s “Niagara” particle system, and the `Blur` function from the `BP_PlayerCamera` script.

- On player collision with the BoxCollider component of the `BP_SteamSystem` script, we set the camera script’s `Blurred` variable to `true`, and set the camera script’s `TotalBlurTime` variable to the `BlurDuration` variable from the `SteamSystem` script: this means we can set a different blur duration for different steam actors in the level.

[![Steam system](/assets/images/delivery-turbo-s/steam-system.png)](/assets/images/delivery-turbo-s/steam-system.png)

***

### Other project scripting

#### WWise stuff

The game makes use of an external software, called [WWise](https://www.audiokinetic.com/en/products/wwise/), which is used for managing audio. This software required a large amount of binaries otherwise the project wouldn’t compile. Since we were using Git, this would greatly increase the size of the repository, and pull times. Thus I created a script `get_wwise_libs.bat` which would retrieve the required lib files, by checking the user’s C: drive for the WWise files, or if they don’t exist locally, would download from a GitHub repository holding the files. This solution worked great for keeping the repository at a decent size and also being convenient for other team members to use to get the files. 

This meant that instead of storing the large WWise files in the repository, we could just store the script, and the files would be retrieved on demand.

~~~batch
@echo off
cd /d %~dp0
:: Check for Wwise install, so we can move the files instead of downloading.
if exist "C:\Program Files (x86)\Audiokinetic\Wwise 2021.1.6.7774\" (
	robocopy "C:\Program Files (x86)\Audiokinetic\Wwise 2021.1.6.7774\SDK" Plugins\Wwise\ThirdParty /e /xd "C:\Program Files (x86)\Audiokinetic\Wwise 2021.1.6.7774\SDK\source" "C:\Program Files (x86)\Audiokinetic\Wwise 2021.1.6.7774\SDK\samples" "C:\Program Files (x86)\Audiokinetic\Wwise 2021.1.6.7774\SDK\Help" 
	pause
	goto :eof
)
:: If no Wwise install found, download the lib files from my GitHub repo.
echo No Wwise install found at "C:\Program Files (x86)\Audiokinetic\Wwise 2021.1.6.7774", downloading lib files...
powershell -Command "Invoke-WebRequest https://github.com/laurence-dorman/WWiseLibs/archive/refs/heads/main.zip -OutFile WWiseLibs.zip"
powershell Expand-Archive WWiseLibs.zip 
robocopy WWiseLibs\WWiseLibs-main Plugins\Wwise\ThirdParty /e /move
del WWiseLibs.zip
rmdir WWiseLibs
pause
~~~

***

#### Unreal BP diff scripts

Since the project is mainly using Unreal’s blueprints for scripting, I made a set of batch scripts which make use of Unreal’s launch flag “-diff”, which opens a diff window that compares the given blueprint scripts. This was very useful as I was mainly the one who was taking care of merges/conflicts, and the script made finding differences in scripts much easier.

~~~batch
@echo BP Diff Script - Enter paths to each blueprint
@cd C:\Program Files\Epic Games\UE_4.27\Engine\Binaries\Win64

@set /p file1="Enter path to first file: "
@set /p file2="Enter path to second file: "

UE4Editor.exe -diff %file1% %file2%
~~~

&nbsp;

***

&nbsp;

## [2. Additional work](#2-additional-work)

### GitHub management

- Our development process mainly throughout development was working through the week in our own repositories, and then at the start of the next week, I would take care of merging the branches into main and resolving merge conflicts, often making use of the BP Diff scripts previously mentioned.

&nbsp;

***

&nbsp;

## [3. Post-mortem](#3-post-mortem)

I’m happy with the amount of work contributed to the project and felt I helped steer the game’s functionality toward the proposed initial idea.