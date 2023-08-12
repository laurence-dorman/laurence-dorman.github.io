---
layout: post
title: PSVita 2.5D Minigame 🎮🚀
description: Developing a game for the PSVita | C++
date: 2021-05-20
---

>**Disclaimer: This is university coursework.**

## Table of contents
{:.no_toc}
* TOC
{:toc}

&nbsp;

***

&nbsp;

## Video demonstration
<iframe width="560" height="315" src="https://www.youtube.com/embed/7B4qlt_PCJc" title="YouTube video player" frameborder="1" allowfullscreen></iframe>

&nbsp;

***

&nbsp;

## [Introduction](#introduction)

The purpose of the application was to develop a game for the PS-Vita and to develop it in such a way that the application can be improved upon easily, by strong use of object-oriented design patterns. The main features of the application are the:
-	[Menu builder system](#menu-manager).
-	[State system](#state-machine).
-	[Particle system](#particle-system).
-	[Game physics powered by Box2D](#game-physics-with-box2d).

The application functions mainly by control of the state manager class, which controls nicely what is being updated and rendered at any one time.

It was only necessary to complete one level of the prototype as a demonstration. The game had to be developed using C++.

&nbsp;

***

&nbsp;

## [1. Initial concept](#1-the-concept)

Jetpack infinite runner game. The player has to fly as high as possible and collect fuel to keep the jetpack going. You may also need to avoid obstacles, at a low altitude this could be birds, kites, etc. At a higher altitude airplanes, helicopters, then asteroids, spaceships, space stations etc. 

Controls: Left analog stick to control jetpack direction (left = 270 degrees, right = 90 degrees etc.) - maybe less so you aren't completely horizontal.

Outside of the prototype, a round system would be implemented, where the player has coins that can be used for upgrades, and the player can buy new jetpacks, fuel, etc. The player can also buy new levels, which would be different planets, and the player can buy new characters.

&nbsp;

***

&nbsp;

## [2. Application design and techniques used](#2-application-design-and-techniques-used)

### State Machine

#### Virtual functions

Since the application relies so heavily on the state system, the program was created in such a way that it would be simple to add new states and to build upon/improve the state system itself, therefore the program features the object-oriented style state machine, with virtualised functions for each state:

~~~cpp
#state_manager.h
virtual void Update(float frame_time, const gef::SonyController* controller) = 0;
virtual void Render() = 0;
virtual void onEnter() = 0;
virtual void onExit() = 0;
virtual void Reset() = 0;
~~~

#### State machine in practice

As an example, this is how we can use the state machine to render the current state:

~~~cpp
void SceneApp::Render()
{
  state_manager_->Render();
}
~~~

The state manager will hold a pointer `current_state_`, which will point to the current state. The state manager will then call the virtualised functions of the current state:

~~~cpp
void StateManager::Render()
{
  if (current_state_) {
    current_state_->Render();
  }
}
~~~

#### Changing states

The `setState()` function takes an enum `STATE`. The enum is defined in the state manager header file, the function handles calling the `onExit()` function of the current state, and the `onEnter()` function of the new state.

~~~cpp
void StateManager::setState(STATE s)
{
    if (current_state_) { // if current_state_ isnt null
        current_state_->onExit(); // do onExit for that state
    }

    current_state_ = states_[s];

    current_state_->onEnter(); // do onEnter for new state
}
~~~

I like this `setState()` function because it will call the `onExit()`/`onEnter()` functions automatically when changing states, so you do not need to call them yourself.

***

### Menu Manager

The application was designed so that menus would be easy to add and adjust, for example, this is the code for adding the elements to the main menu state:

~~~cpp
float b_offset = 75.f;
float b_scale = 0.8f;

menu_manager_->addElement("START GAME", b_scale, b_offset, StateManager::STATE::INGAMESTATE, MenuElement::NORMAL);
menu_manager_->addElement("HOW TO PLAY", b_scale, b_offset, StateManager::STATE::HOWTOPLAYSTATE, MenuElement::NORMAL);
menu_manager_->addElement("SETTINGS", b_scale, b_offset, StateManager::STATE::SETTINGSSTATE, MenuElement::NORMAL);
menu_manager_->addElement("QUIT GAME", b_scale, b_offset, StateManager::STATE::QUIT, MenuElement::NORMAL);
~~~

The `b_offset` value adjusts the distance between each of the elements, and `b_scale` adjusts the overall scale of each button and it’s elements (text and sprite). The `addElement()` function has overloads depending on what “type” of element you want to add, such as a normal button that switches the game state or a button that changes a game setting.

The menu manager also has functionality for settings buttons which can change the game settings, such as the volume, difficulty etc by overloading the `addElement()` function. 

~~~cpp
menu_manager_->addElement("MASTER VOLUME", b_scale, b_offset, MenuElement::SLIDER, settings_->master_volume_, 0, 10);
menu_manager_->addElement("DIFFICULTY", b_scale, b_offset, MenuElement::SLIDER, settings_->difficulty_, 1, 5);
menu_manager_->addElement("MUSIC", b_scale, b_offset, MenuElement::TOGGLE, settings_->b_music_);
menu_manager_->addElement("SND FX", b_scale, b_offset, MenuElement::TOGGLE, settings_->b_sfx_);
menu_manager_->addElement("BACK", b_scale, b_offset, StateManager::STATE::MENUSTATE, MenuElement::NORMAL);
~~~

The settings are initialised in the `Init()` function of the scene and used to initialise a new `Settings` object, which stores the pointers.

~~~cpp
master_volume_ = 5;
b_sfx_ = true;
b_music_ = true;
difficulty_ = 4;

settings_ = new Settings(&master_volume_, &b_sfx_, &b_music_, &difficulty_);
~~~

This settings object is then passed as a parameter to the StateManager constructor:

~~~cpp
state_manager_ = new StateManager(&platform_, sprite_renderer_, renderer_3d_, font_, camera_, audio_manager_, settings_);
~~~

The logic inside the menu manager that handles a settings slider being pressed is as follows:

~~~cpp
if (controller->buttons_pressed() & gef_SONY_CTRL_RIGHT) {
if (elements_[position_]->getType() == MenuElement::SLIDER) {
		audio_manager_->PlaySample(2, 0);
		elements_[position_]->addValue(1);
	}
~~~

***

### Particle System

The particle system’s only use as of now in the application is for the jetpack, to create the “smoke” coming out of the jetpack when the player is boosting (holding X), this is inside of the `ParticleManager` `Update()` function:

~~~cpp
if (player_->isThrusting()) {
		timer_ += frame_time;
		if (timer_ >= 0.02f) { // spawn rate =~ 0.02s
			timer_ = 0.f;
			addParticle(&gef::Vector4(-0.55f, 0.f, 0.f)); // left booster
			addParticle(&gef::Vector4(0.55f, 0.f, 0.f));  // right booster
		}
	}
~~~

The `AddParticle()` function inside ParticleManager looks like this:

~~~cpp
gef::Material* particle_material = new gef::Material(); // create new material (each particle has own material)
particle_material->set_colour(gef::Colour(1.0f, 1.0f, 1.0f).GetABGR()); // initial colour is white
gef::Mesh* particle_mesh = primitive_builder_->CreateSphereMesh(0.6f, 5, 5, gef::Vector4(0.f, 0.f, 0.f), particle_material); // create sphere mesh with material

gef::Matrix44* transform = new gef::Matrix44(); // create transform matrix that translates to pos
transform->SetIdentity();
transform->SetTranslation(*pos);

*transform = *transform * player_->transform(); // translates from player by pos

particles_.push_back(new Particle(particle_material, particle_mesh, transform));
~~~

Which adds a new particle to the `particles_` vector at the passed in position, the particle has it’s own mesh and material so that it can be adjusted individually, which is important for changing the colour and alpha values of the material, for example:

This is inside of the `Particle` `Update()` function:
~~~cpp
material_->set_colour(gef::Colour(time_left_, time_left_, time_left_, time_left_).GetABGR());
~~~

When a particle is created, it is passed the matrix transform of the player, which is used to place the particle in the right position relative to the player position, and so that the particles “move away” from the player relative to the player rotation.
~~~cpp
particles_.push_back(new Particle(particle_material, particle_mesh, transform));
~~~

And then, inside the `Particle` constructor, we set it's transform to the passed in transform:
~~~cpp
this->set_transform(*transform);
~~~

Each particle generates it’s own `rotation_` float, which will determine which direction it is moving in, from 260-280 degrees.
~~~cpp
rotation_ = gef::DegToRad(float(rand() % (280 - 260 + 1) + 260));
~~~

Move along `rotation_` in the `Particle` `Update()` function:
~~~cpp
transform.SetTranslation(gef::Vector4(cosf(rotation_) / 5.f, sinf(rotation_)/ 5.f, 0)); // move particle along rotation_ angle
transform = transform * this->transform();
~~~

***

### Game physics with Box2D

Box2D is a physics engine that is used to simulate 2D physics in games, it is used in this application to simulate the physics of the player and to detect collisions between the player and the fuel objects.

* Initialise the physics world
~~~cpp
// initialise the physics world
b2Vec2 gravity(0.0f, -9.81f);
world_ = new b2World(gravity);
~~~

* We can then use the `b2World` `Step()` function to update the physics world, this is done inside the `UpdateSimulation()` function of the `InGameState` class:
~~~cpp
world_->Step(timeStep, velocityIterations, positionIterations); // update box2d world
~~~

For collision detection, we can retrieve the contact list using the `GetContactList()` function, and then loop through the contact points to check if the player has collided with a fuel object:

~~~cpp
// get the head of the contact list
b2Contact* contact = world_->GetContactList();
// get contact count
int contact_count = world_->GetContactCount();
for (int contact_num = 0; contact_num < contact_count; ++contact_num)
	{
		if (contact->IsTouching())
		{
			// get the colliding bodies
			b2Body* bodyA = contact->GetFixtureA()->GetBody();
			b2Body* bodyB = contact->GetFixtureB()->GetBody();

			// DO COLLISION RESPONSE HERE

			GameObject* gameObjectA = NULL;
			GameObject* gameObjectB = NULL;

			gameObjectA = reinterpret_cast<GameObject*>(bodyA->GetUserData().pointer);
			gameObjectB = reinterpret_cast<GameObject*>(bodyB->GetUserData().pointer);

			if (gameObjectA) { // check that gameobjects arent null pointers
				if (gameObjectB)
				{
					if (gameObjectA->type() == PLAYER && gameObjectB->type() == FUEL) // if player type collides with fuel type
					{
						audio_manager_->PlaySample(4, 0); // play sound
						fuel_manager_->spawnFuel(world_, 1, player_->getPosition()); // spawn fuel function (removes old fuel, adds new fuel)
						player_->addFuel(50.f / *state_manager_->settings_->difficulty_); // add fuel, amount based on difficulty
					}
				}
			}
		}

		// Get next contact point
		contact = contact->GetNext();
	}
~~~

&nbsp;

***

&nbsp;

## [User guide](#user-guide)

Once the application is running, you will first see the splash screen, which can be skipped by pressing X or Start.

Then you will be on the main menu screen, which can be navigated using the D-Pad, as with all the other menus. Normal and toggle buttons will be activated using the X button, and slider elements use the Right/Left  D-Pad arrows to increment/decrement the setting respectively.

Once inside the game, you will use the X button to activate the jetpack thrusters, and the left analog stick to control the direction of the jetpack.

The aim is to fly as high as possible without running out of fuel, fuel can be collected by colliding with the fuel objects placed above you in the level.
Difficulty can be adjusted in the settings menu from (1-5), 5 being the very hard and 1 being far too easy.

It’s possible that the controls don’t match up with you, since I was using an Xbox 360 controller throughout development, but I have done my best to change the controls to hopefully work with a Playstation controller. (Some of the buttons were switched around such as Cross and Square).

&nbsp;

***

&nbsp;

## [Conclusion](#conclusion)

On reflection of the game as a whole, I am happy with what I have built so far, things like the MenuManager I think work quite well, as well as the StateManager. However I wished to add a lot more to the game such as more collectables other than fuel like powerups, speed boosts, coins, coins which could be spent in an “Upgrade shop”, to upgrade the jetpack by increasing fuel or speed. I also wished to add a “Level 2” above the clouds, which is in the “Space area”, with the black background. This level would feature things such as enemies or UFOs which the player would have to avoid or risk dying. I believe the “Level 1” which is in the game right now is quite good as a start, since the only challenge is locating and getting to the fuel in time, which sometimes are obscured by clouds which I think is a good gameplay element.

One problem I have with the game in it’s current state are the lack of assets as a whole, textured assets, and animations. I had quite a lot of issues with the model loader in particular loading fbx’s which have animations and textures and then loading that into the application. So in future I would hope to create a better model import pipeline that makes this easier. I actually rigged my player model in blender with a skeleton with the hopes of programmatically animating it inside of the game using Box2D’s joints feature, however this proved quite challenging so I didn’t manage to implement it.

I believe what I have built so far to be a good foundation for further building upon to create a good game application.

> The full source code for the application can be found [here](https://github.com/laurence-dorman/Jetpack-Game-PSVITA).

## [References](#references)

- player model – [“Low Poly Character”](https://sketchfab.com/3d-models/standard-low-poly-character-1a655774022a4dfbb7a60b417d06f4e4) (Accessed: 16 April 2021).

- jetpack thruster sfx – [“Blow torch light up”](https://www.soundsnap.com/node/84677) (Accessed 19 May 2021).

- menu sfx –  [“Gta 4 - Menu Sound Effects”](https://www.youtube.com/watch?v=bWxUthnVhuo) (Accessed 19 May 2021).

- music – [“TRASHY ALIENS”](https://soundimage.org/looping-music) (Accessed 19 May 2021).

- cloud model – [“Cloud”](https://sketchfab.com/3d-models/cloud-852ae741a60b44ec886fbf34bbd058f8 ) (Accessed 19 May 2021).

- fuel model – [“Jerry Can”](https://sketchfab.com/3d-models/jerry-can-972c63e9bc4f4ff4a760f9b23042c512) (Accessed 20 May 2021).

- fuel collected sfx - [“Game Modern Damage or Hit Sound 4”](https://www.storyblocks.com/audio/stock/game-modern-damage-or-hit-sound-4-rt7xobzlaw8k8umgkqi.html) (Accessed 20 May 2021).

