---
layout: article
title: Another One Bites the Crust
description: A pizza delivery driving video game.
cover: /assets/images/aobtc/fly.png

article_header:
  background_image:
    src: /assets/images/aobtc/shoot.png
---
<!--more-->

## Overview
Another One Bites the Crust (AOBTC) is a pizza delivery video game where players compete to fulfil deliveries as quickly as possible, earning tips to eventually win the game.
The game was made for _CPSC 585: Games Programming_, which I took in the third year of my Bachelor's degree.
In the course, we worked in teams of four to develop a driving video game over the course of the semester.
My colaborators on the game were [Gorman Law](https://www.linkedin.com/in/gorman-law/), [Daniel Lewis](https://www.linkedin.com/in/scraniel/), and [Alexei Pepers](https://www.linkedin.com/in/apepers/).

The game was developed in C++ and OpenGL without the use of any game engine.
The PhysX physics engine was used for rigid body dynamics and the driving model.
The source code for the project can be viewed [here](https://github.com/benjaminulmer/bite-the-crust).

### Video Demo
<div>{%- include extensions/youtube.html id='v8WvPdpNaj8' -%}</div>

### Contributions
While each team member inevitably contributed to many parts of the code, each member's primary responsibilities were as follows:
- Gorman Law: graphics code, shaders, menu logic, asset creation/sourcing
- Daniel Lewis: navigation graph, AI, sound
- Alexei Pepers: game logic, data specification and loading, communication between game engines
- Benjamin Ulmer: physics engine, driving model, player input, split-screen

## Feature Highlights
This project is quite large, and I worked on the code a long time ago; going into details on every aspect of the game is not feasible.
Instead, I will briefly highlight some of my favourite features, specifically ones that I contributed to in some fashion.

### Driving Model
Creating a satisfying and responsive driving model is a difficult task.
Add on to that maintaining the feel of driving a delivery van, creating the driving model for AOBTC proved to be a significant challenge.

There are two components of a game's driving model:
1. Physical model---this includes every physical aspect of the vehicle (e.g. weight, dimensions, and engine properties) and how player inputs are mapped to controls of the physics vehicle.
2. Gameplay model---this is every other aspect of the game that can be modified to change how the vehicle "feels" to drive. A simple example is changing the camera's field of view, affecting the player's perception of their speed.

For tuning the physical model, this was mostly a process of trial and error.
My goal was for the physical vehicle to be stable and maneuverable, but have lower acceleration and top speed.
I applied a few well-known tricks to work toward this---such as lowering the vehicle's centre of mass---but overall, it was an iterative process of modifying different properties until the desired result was achieved.
While I am mostly satisfied with the final physical model, it is far from perfect.

With the physical driving model in place, the vehicles felt more like tanks than tipsy, rickety pizza vans.
I aimed for stability in the physical model to prevent the vans from tipping over during gameplay; however, I wanted the player to _feel_ like they were about to tip over at any point.
I achieved this by decoupling the physical vehicle model from the rendered one.
Rotating the rendered vehicle as it corners creates the illusion of instability while keeping the physical vehicle perfectly steady.
You can see this tilting clearly in the video clip below.

<div>{%- include extensions/video.html path='/assets/videos/aobtc/tilt.mp4' width='100%' -%}</div>

Creating this tilt effect is quite straightforward, accomplished with a simple [exponential smoothing](https://en.wikipedia.org/wiki/Exponential_smoothing).
A tilt angle for the frame is calculated, which is proportional to the vehicle's forward speed and the player's steering input.
A weighted average between this value and the previous tilt angle gives the final value used to transform the vehicle's model matrix before rendering.
In code,
```c++
float alpha = 0.01f;
float beta = 0.008f;
tipAngle = (1.f - alpha) * tipAngle + (alpha * input.steer * physicsVehicle->computeForwardSpeed() * beta);
```
where `input.steer` is between -1 and 1.
The values of `alpha` and `beta` were obtained through trial and error.

### Split-Screen
Split-screen functionality was a stretch goal for the project; because of this, work on the feature was not started until the last few weeks of the course.
While implementing split-screen functionality is simple in theory, a poorly designed codebase can make any simple task difficult.
Thankfully, the codebase for AOBTC was designed well enough that implementing split-screen only took a few days.
A few parts of the code needed to be updated, such as properly sizing minimap elements, but overall the process was straightforward.
Additionally, the game's graphics demands are low enough that no optimizations were needed to get four-player split-screen running at 60 fps.

![Four player split-screen in Another One Bites the Crust](/assets/images/aobtc/split-screen.png)

While the AI players for AOBTC are mostly competent, split-screen is definitely the most fun way to play the game.

### Space Mode
Like any good video game, AOBTC needed a silly easter egg.
Entering the [Konami Code](https://en.wikipedia.org/wiki/Konami_Code) activates _space mode_, where the vans are replaced with spaceships and the skybox changes to a starry night.

<div>{%- include extensions/video.html path='/assets/videos/aobtc/space-1.mp4' width='100%' -%}</div>

Besides the obvious changes to the vehicle models and skybox, space mode also modifies the game in the following ways:
- Gravity is reduced
- The tilt visual effect on vehicles is reversed---vehicles tilt _into_ corners as opposed to out of them
- Players can jump mid-air with no limits

The reduced gravity makes the game quite difficult to play, making the vehicles much less stable and increasing the air time and sliding distance of fired pizza boxes.
For these reasons, the AI also struggles more than usual in space mode.
Still, it does allow for some interesting delivery strategies...

<div>{%- include extensions/video.html path='/assets/videos/aobtc/space-2.mp4' width='100%' -%}</div>

## Reflection
AOBTC was the first large project I worked on in C++.
My knowledge of the language was extremely limited at the time; however, I learned a lot throughout the project.
The project was my first exposure to using third-party libraries, using git to manage a non-trivial code base with multiple contributors, and working in an agile-like development environment.

Looking back at the code now, it is not nearly as poorly written as I would have expected.
Many aspects would be improved by safer and more modern constructs (e.g. const references and smart pointers over raw pointers). Still, overall, the code has decent bones.
The game is also surprisingly fun for what it is (I may or may not be biased).
Overall, I'm still proud of AOBTC to this day.
