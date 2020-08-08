---
layout: article
title: Another One Bites the Crust
description: TODO
cover: /assets/images/aobtc/fly.png

article_header:
  background_image:
    src: /assets/images/aobtc/shoot.png
---
<!--more-->

This article is currently under construction; more details coming soon.
{:.warning}

## Overview
Another One Bites the Crust (AOBTC) is a pizza delivery video game where players compete to fulfil deliveries as quickly as possible, earning tips to eventually win the game.
The game was made for _CPSC 585: Games Programming_, which I took in the third year of my Bachelor's degree.
In the course, we worked in teams of four to develop a driving video game over the course of the semester.
My colaborators on the game were [Gorman Law](https://www.linkedin.com/in/gorman-law/), [Daniel Lewis](https://www.linkedin.com/in/scraniel/), and [Alexei Pepers](https://www.linkedin.com/in/apepers/).

The game was developed in C++ and OpenGL without the use of any game engine.
The PhysX physics engine was used for rigid body dynamics and the driving model.
Source code for the project can be viewed [here](https://github.com/benjaminulmer/bite-the-crust).

### Video Demo
<div>{%- include extensions/youtube.html id='v8WvPdpNaj8' -%}</div>

### Contributions
While each team member inevitably contributed to many parts of the code, the main responsibilities of each member were as follows:
- Gorman Law: graphics code, shaders, menu logic, asset creation/sourcing
- Daniel Lewis: navigation graph, AI, sound
- Alexei Pepers: game logic, data specification and loading, communication between game engines
- Benjamin Ulmer: physics engine, driving model, player input, splitscreen

## Feature Highlights
This project is too large and I wrote the code too long ago to go into details on everything.
Instead, I will briefly highlight some of my favourite features, specifically ones that I contributed to in some fashion.

### Driving Model
Creating a satisfying and responsive driving model is a difficult task.
Add on to that maintaining the feel of driving a delivery van, creating the driving model for AOBTC proved to be significant challenge.

There are two main components of a game's driving model:
1. Physical model. This includes every physical aspect of the vehicle (e.g. weight, dimensions, and engine properties) as well as how player inputs are mapped to controls of the physics vehicle.
2. Gameplay model. This is every other aspect of the game the can be modified to affect how the vehicle "feels" to drive. A simple example is modifying the camera's field of view, affecting the player's perception of their speed.

For tuning the physical model, this was mostly a process of trial and error.
My goal was for the physical vehicle to be stable and manueverable, but have lower acceleration and top speed.
I applied a few well known tricks to work toward this---such as lowering the vehicle's centre of mass---but overall it was an interative process of modifying different properties until the desired result was achieved.
While I am mostly satisfied with the final physical model, it is far from perfect.

With the physical driving model in place, the vehicles felt more like tanks than tipsy, rickety pizza vans.
I aimed for stability in the physical model to prevent the vans from tipping over during gameplay, but I wanted the player to _feel_ like they were about to tip over at any point.
I achieved this by decoupling the physical vehicle model from the rendered one.
Rotating the rendered vehicle as it corners creates the illusion of instability while keeping the physical vehicle perfectly steady.
You can see this tilting clearly in the video clip below.

<div>{%- include extensions/video.html path='/assets/videos/aobtc/tilt.mp4' -%}</div>

Creating this tilt effect is actually quite straighforward---accomplished with a simple [exponential smoothing](https://en.wikipedia.org/wiki/Exponential_smoothing).
In short, a tilt angle for the frame is calculated which is proportional to the vehicle's forward speed and the player's steering input.
A weighted average between this value and the previous tilt angle gives the final value used to transform the model matrix of the vehicle before rendering.
In code,
```c++
float alpha = 0.01f;
float beta = 0.008f;
tipAngle = (1.f - alpha) * tipAngle + (alpha * input.steer * physicsVehicle->computeForwardSpeed() * beta);
```
where `input.steer` is between -1 and 1.
The values of `alpha` and `beta` were obtained through trial and error.

### Space Mode
Easter egg---mostly for fun.
Exploits tilt effect, but reversed.

### Split Screen
Not difficult in theory, but required rewriting and restructuring certain parts of code.
Basic version was easy to get working, but I remember many subtle bugs that needed to be fixed.
Satisfying pay off for mostly well designed code base (as far as undergrad projects go).

## Reflection
First major C++ project.
Looking back at code, not as bad as I would have thought.
Still one of most fun projects I worked on.
Game is surprisingly fun for what it is (I may or may not be biased).
