+++
title = 'VR Solar System'
date = 2024-04-01T11:26:57+02:00
draft = false
+++

## Introduction
***

This semester I was looking around for an interesting class. My goal was to pick something where I would be able to learn as much as possible, meaning I wanted to start from zero and choose something I had not engaged with before: Game Dev and VR.  
It was always in the back of my mind that this whole topic kind of exists and is exciting, but I never got around to actually downloading Unity and tinkering with it myself. This course was perfect because we could more or less do whatever we wanted and make it interesting for ourselves, which was my goal.

The objective of the course was to implement our own locomotion and interaction technique on a toy racetrack, which would replace the classic written exam. The track consisted of moving to checkpoints using the locomotion, and then only being able to progress further by solving an interaction challenge - all in VR.  
Oh yeah, and we were given the Quest 3!


## Idea
***
#### First thoughts
My first idea that I instantly got once it was mentioned that we had to make our own locomotion was to make a fast car. I thought it would be extremely fun to drive around a toy racetrack very fast, and at the same time was a bit confused as to why I hadn't heard of something like Need For Speed VR before, given how cool I thought it to be. I had already imagined how I would maybe even use hand tracking and a virtual steering wheel so you could play just by using your hands and nothing else. Sadly it seemed like this would cause a lot of motion sickness, especially at the speed I wanted to allow for to make it exciting. I probably should have tested this before moving on to my next idea, just to be able to have fun driving in VR. Maybe in the future...
#### Second thoughts
The main reason I didn't pursue said idea was that I thought that the implementation would be quite straightforward, and remember, I took this course to learn something new, not be done in a few hours.

I remembered this fun little iPad game called OsmosHD, in which you control a small circular organism by tapping in the opposite direction of where you want to go, thereby expelling bubbles of mass that propel you forward, while your little organism is flying in orbits around bigger bodies, which will absorb you if they are larger and vice versa, with the goal of each level to become the biggest organism.

![Osmos](./osmos.jpg)

I decided to somewhat adapt this to 3D and make a toy solar system where you fly around by boosting yourself from your arms (like Iron Man) while interacting with the gravity of all the planets while trying to complete an arbitrary goal (in this case, the interaction technique required by the course).

Luckily, a project like this already existed, as a YouTuber called [Sebastian Lague](https://www.youtube.com/@SebastianLague) made a [video series](https://www.youtube.com/watch?v=7axImc1sxa0&list=PLFt_AvWsXl0cSy8fQu3cFycsOzNjF31M1) (I highly recommend watching) on a toy solar system inspired by [Outer Wilds](https://store.steampowered.com/app/753640/Outer_Wilds/). This was essentially what I wanted, now I **just** needed to port this to VR. Easy, right? _Ha!_

![SebLague Project 1](./Orbits.PNG)
![SebLague Project 2](./ShapingPlanetsVid2.PNG)

## Interaction and Locomotion ideas
The locomotion and interaction techniques ideas are quite simple. You fly around using boosters shooting from your hands (as described above). For the interaction, you would need to bring an object, shaped like a T, as close as possible to a target of the same shape in terms of rotation and translation, which both spawn during a challenge.

My idea was initially to lock the T shape in place and only have it rotate freely. You would shoot it with the booster giving it rotation from the particles pushing against it. However, after learning that translation must also be part of the challenge, I added a surrounding box, in which the T shape would freely bounce.
Once the target shape is as close as possible, you just press a button on your controller to lock it in place and finalize the challenge. 

The challenge location would be on the different planets to which you had to fly to complete it.

![Interaction Challenge](./interaction-challenge-screenshot.jpeg)

## Porting to VR
I could elaborate on a long boring list of things that were necessary to do to get to a stable point, but what it boils down to is this:

  * Understanding all of the code and how it worked together, what was important and dependent on each other
  * Take out everything I don't need and make sure nothing breaks (This took like 5 hard resets O_O)
  * Add a VR Rig and write a new player controller that ties into the project correctly
  * First make all the custom shaders actually appear (I couldn't get them to be visible for some time), then add the correct math for both eyes
  * Fix any preexisting mistakes (see Floating Point Origin and LoD)

## Here is me completing the "parkour"

{{<youtube ce34-mluCjw>}}

## Some parts of the Implementation explained
***
### Floating Point Origin
This was one of the first things that bugged me a lot. In the initial project, it was necessary to have an origin reset to zero whenever you got too far from it. This is due to floating point precision which results in objects far away from the origin starting to jitter. To do this you just need to teleport everything by the same amount towards the origin. Sounds *so simple*, right?

The initial problem was that at the moment of the shift, there would be a flicker. Terrible for VR. Later I also noticed a small mismatch in player and planet position, maybe half a meter. I was trying everything and nothing would work, either the flicker remained, the mismatch remained, or the player inexplainably shot off at light speed away from everything.

Something that seemed to work was changing the script execution order, but that wasn't a great solution and I tried to move off of it. What finally seemed to be the case was that the flicker came from a mismatch in update order between Transforms and Rigidbodies, while the slight mismatch likely came from Rigidbody interpolation. This meant I moved everything that required an origin update to Rididbodies, while also keeping their interpolation by using 
```C#
Rigidbody.MovePosition();
```
Oh yeah, and the particle system also had issues, which I had to solve by manually teleporting the particles already emitted.
### LOD
While figuring out the problems mentioned above I stumbled upon the LOD calculation code, which was rotating the camera's transform, which I thought could lead to these issues as I had already tried so many things.

What it was doing was "looking" from the player's position at each of the planets using
```C#
Transform.LookAt();
```
and calculating how much screen height they were taking up. I decided to do this manually to stop modifying the camera's transform.

This led me on a tangent about the model, view, and projection matrices which later on helped me with the shaders. What I essentially did was take the current MVP and multiply a rotation matrix between the view and projection matrix, essentially making the camera "look" somewhere else.

This calculation similarly led me onto a tangent about Quaternions which also proved quite useful in general.
### Planet rotation and its consequences
I spent quite a bit of time thinking about rotations and Quaternions during this project. Everything was fairly fine before adding them, but once the planets rotate, there are many considerations to be made, like:
  * T Shape position and rotation
  * Player rotation and velocity when standing on planets
  * Player rotation and velocity when close to planets
A general problem for VR is when the player needs to be rotated or moved, but he is not moving or rotating in the real world. This causes motion sickness.

I had to make sure to solve these problems, while also not making the player, which mostly was myself, very sick (sadly I did not always succeed at that when testing).

The first problem comes from trying to land on planets. You want your feet to be downwards so that you can "walk", in this case, more like "stand". This means whichever rotation you previously had needs to naturally transition into an "upright" position. But this also means that when you are moving relative to the planet while being close to it, or standing on it, you have to be continuously rotated to remain upright. Now once the planet starts rotating, the player also needs to have an appropriate velocity applied to him so that he doesn't "slide off".

The second part is taking off. You will probably be looking towards the sky when choosing your next target - up. If you were to take off like this and fly there, you would have to keep looking up while doing so, which would be quite uncomfortable. So at some point when you are leaving the planet, you need to be rotated in a direction that makes sense.

This part took quite a few iterations to get right, meaning to not make you sick, while also making sure you get reoriented towards something that makes sense, and most importantly that you can look forward/straight in the real world while doing so.
### Shaders
Getting these to work sucked because I had no clue what I was doing, and because this project was heavily dependent on them I had to first learn a lot online to even slightly understand what I was doing.
Finally, I mostly figured out what I had to change in the ocean and atmosphere calculations to be rendered correctly for both eyes, which was essentially manually calculating the view vector (for both eyes) by passing the appropriate matrices to the shader and doing the final calculation there because the view vector differs for each vertex.
### Making my own planets
This was a lot of fun. The original Unity project had a custom script to customize planets, so all you had to figure out was which type of planet you wanted, create all the necessary parts, and then play around by customizing them. Because the planet's atmospheres, oceans, and surfaces are all either post-processing or compute shaders, you just had to change as many noise values as you could in the custom script to keep it distinct from the other planets, while also keeping it still recognizable and interesting. Using this script I made three more planets: An ocean planet with many tiny islands, a giant planet, and a dark mysterious shadow planet.

![waterplanet](./water-planet.jpg)

### Solar System orbits
Wanting to keep the spirit of my idea alive, the worst thing to do would be to make the planets follow hard-coded orbits. The initial project was already doing the gravity calculation which was perfect for my needs, all I did was scale the G constant a bit to make gravity way stronger.
## Interaction challenge
I hate to admit it, but this was almost an afterthought. After spending so much time going on tangents and learning about many different things, I kind of forgot that I still hadn't even started doing the Parkour Challenge.

Compared to the other things this was easy and difficult at the same time. Parts like stats tracking, and general challenge setup were quite boilerplate and easy to do, with some caveats:
  * Figuring out how to guide the player toward the challenge: I decided to write a simple shader that acts like a beacon, that spawns "on" the challenge and is visible from anywhere
  * Some simple math to make the coins spawn equidistantly around a planet
  * Make a new scene with a "menu room" that shows stats and a short text tutorial

A fun part was figuring out how to make any number of challenges possible. I decided to randomly generate them based on a seed by taking all the vertices of a planet mesh, removing all below the ocean, and picking a random one to spawn a challenge "above" it.

This got a bit more complicated after adding planet rotation, because the vertices would no longer match up to their actual position, meaning I now had to also rotate the chosen vertex in just the same way as the planet, which is easy to do by taking the initial planet rotation and getting the delta between the current one, then applying it to the vertex. The same thing goes for the beacon obviously.

The final problem was the T shape. Because the goal was to push it around using my boosters, it had to be a Rigidbody to interact with the particles. The problem here is that you can't just attach it to the planet at a certain point like a transform. What needed to be done was to manually calculate the correct velocity you need to apply for it to stay in place relative to the planet it's on. And finally, you also need to rotate it with the planet, otherwise to an observer standing on the planet (which rotates) it would look like the shape is rotating, while it is actually not. This image explains it better:

![Interaction Challenge expl](./rotationexplanation.PNG)

### Path visualization
I decided early on that I would need to show the player approximately where they are going because otherwise, this is difficult to estimate. I made a Linerenderer show the simulated next X steps where the player would go in the future by simulating the planets X steps into the future and then applying the gravity equation for the simulated player at every step.

It was also important to not have a global reference frame, which would just confuse, but rather make the line relative to the closest planet. Planet rotation also complicated this part, so I made it relative to the rotation as well.

A cool optimization I did was when I noticed that this script was taking about 7% in the profiler, not including graphics. Instead of computing all the steps for the player every single frame, I only added the newest prediction, meaning if you predict 500 steps forward this is a 500x reduction.

This however only works if the player did not input any movement commands. Otherwise, the trajectory completely changes and you have to recompute either way. The trajectory is also recomputed if the inaccuracy between the prediction and your current position is too large, presumably from floating point error accumulation, which happens about every 50 physics timesteps. This reduced this script's profiler share to essentially 0%.
### Performance
Having constantly used AirLink to test the project, I was for some reason not expecting the performance issues once I actually deployed on the Quest 3. Even having the more performant Quest 3, the game still struggled to run, getting about 30 fps, which is especially terrible in VR. Having learned a lot about shaders I understand that fixing this would likely require an immense effort. 

Unfortunately, my current solution is not great. Either turning down all the shader iterations and model poly count which looks significantly worse, just turning the shaders off which also sucks, or playing by streaming from PC, which is the best option but obviously not portable.
### Frustum Culling
Because all the atmospheres and oceans are post-processing effects, they had a custom post-processing "pipeline" script. They were just sorted by distance and added to the effects depending if a planet had them or not. What I did was make sure that they were only added if they were inside the camera frustum (atmospheres and oceans are essentially spheres with x * radius of the planet).

As I later found out, Unity automatically does Frustum Culling for objects, which disappointed me a bit that I had done all this for nothing. However, I am still unsure if it is in this case true because being shaders they might still run. Either way (unity or me) this works, and looking away from all the planets stops rendering their oceans and atmospheres!
## Anecdotes
***
### Horrifically fast-spinning planets
When implementing the planet rotation, I wanted to make sure the rotational speed felt right. To see how ridiculous it would be if I set it stupidly high, I did just that and put on my HMD. **Big** mistake.

I spawned, as usual, just above the "toy earth" called the Humble Abode. Having the rotational values of the moon set to 3 revolutions per second, I looked up. I immediately got a weird feeling in my stomach, something like being on a roller coaster and a haunted house at the same time, while also looking down from a high glass bridge. For some reason having this extremely large object spin incredibly fast gave me an immense feeling of terror, like a huge asteroid flying towards you. I couldn't resist seeing what would happen if I tried to land on it, so I proceeded to fly towards the moon, which just made the horrible feeling stronger. Weirdly it was extremely exciting and horrifying at the same time. The anticipation of hitting it became worse and worse as I got closer until I finally crashed into it and was flung out far away. This whole experience was very unique and insane, which made me do this a few times because of how funny it felt.
### The sun kills you
Gravity on the sun is so strong, that when you stand on the closest planets to it, it actually starts to lift you off when you are on the planet side looking towards the sun. It became really annoying to do the interaction challenge on these planets because you just couldn't stay still while doing it, and had to constantly focus on just staying in place.

I decided to fix this by adding a gravity modifier to planets that only affects the player and turned it way down for the sun and slightly up for every planet. This went well with the general spirit of the game because it allows you to orbit planets easier while also not being sucked into the sun so much.

A fun feature still remains in that when you are flying in orbit around the inner planets, your predicted path will look ridiculous if you were to not think about the sun. You would expect a smooth downward curve, but it's actually heavily curved up.

Just for fun I also made it so that after completing the course, the sun's gravity is increased by 10, thereby instantly launching the player towards it and resetting him to the menu.

### Instability
Having the planets interact purely through gravity made the solar system very sensitive to its starting configuration. I especially felt this when adding three new planets to the outer solar system. With each new planet, the orbits of all the other planets started to differ a lot, and it took some time to get them just right so the solar system didn't shoot out into chaos. Here is an example of just a single planet having its starting velocity differ by 1 m/s:

![orbits good](./OrbitDebugGood.PNG)
![orbits bad](./OrbitDebugBad.PNG)

#### Weird Orbits
Slightly related to the instability mentioned above are the weird orbits of moons, which result from increasing the planet's orbital speed by about 10x - 20x, and about 5x bigger than what they were before. This was necessary to make the system feel less static and actually *look* fast and **big**.

![weird orbits](./weird_orbits.PNG)

As a side effect, where you would expect circular orbits of moons, there are these weirdly shaped ones, which from the point of view of the planet look like the moon being stationary for some time, relative to the planet, then moving quickly to the other side.

## Evaluation
***
#### Methodology
To produce some insights and collect some general experience we were tasked with conducting a simple user study with two additional participants and ourselves.

During the evaluation, I gave both test subjects a quick introduction about how the mechanics work, and what they were supposed to do (follow the beacon, get the T shape in place, and collect the coins).

The locomotion proved much more difficult than I had anticipated. Because I was quite familiar with it through testing many aspects of the game countless times, it was not immediately obvious how much of a challenge it would be to someone who was not familiar with it.

I allowed for a practice session of about 5 minutes where I gave the participants time to fly around and just try to land on a planet. Next, I instructed them to complete the course as fast as possible, while also having adjusted the difficulty by reducing the number of interaction challenges, and therefore planets to fly to, to two, and also reducing the number of coins per planet to two.

Unfortunately on the first attempt, the planet resulting from the random seed was the water planet, which was very difficult to land on, meaning I also changed the seed so that this planet would not come up.
#### Problems and Feedback
Critiques were plentiful and deserved.  
As a result of the study, it was clear that the locomotion was too difficult, however, I noticed that the longer the participants played, the more skilled they became at aiming themselves. It was difficult to explain how the projected path worked, it mostly seemed to confuse the participants who were expecting to end up in a different place than the line was showing and also did not understand how their actions affected how the line moved.

It was also interesting to see how both participants were constantly overshooting planets, gathering too much speed at the start, and then not understanding why they weren't slowing down when close to the planet. Meaning they now had to accelerate in the opposite direction to get back to the planet, just to overshoot it again. If one looked from far away it would maybe look something like a visual binary search.

For the interaction feedback was also mixed, a problem especially was that the jets would move you while you tried bouncing the T shape, which made me later add a secondary fire which still has the exhaust, but does not move you around. However, compared to the locomotion, bouncing the T shape around was rather intuitive and did not require much explanation.
#### Metrics
The error of 1 should be about the length of a T shape (which can be arbitrarily scaled). My time was for the full course (3 planets, 10 coins each), I'd assume my time would be around 2 mins for the smaller one.

| Participant   | Coins | Time | Challenges | Avg Error            | Motion Sickness | Presence | Enjoyment |
| ------------- |:-----:|:----:|:----------:|:--------------------:|:---------------:|:--------:|:---------:|
| me            | 30/30 | 6:21 | 3          | (0.52, -0.50, -0.05) | 4               | 7        | 5         |
| participant 1 | 4/4   | 7:13 | 2          | (0.42, -0.08, 0.82)  | 3               | 5        | 0         |
| participant 2 | 3/4   | 5:46 | 2          | (0.68, -0.12, -0.34) | 2               | 7        | 3         |

#### Takeaways
The general gist was that the tasks were a bit tedious and the controls were "too sensitive", one participant said that they liked to win, and it was too hard to "win", which made them annoyed and frustrated at the game.
Another participant said they would have preferred joystick controls over pointing in the opposite direction of where to go, as the hand position was a bit uncomfortable.

The general atmosphere of the game was complimented, however.
My conclusion from the evaluation is that it's likely that the locomotion suffers from the fact that the interaction is quite finicky, and you need to position yourself too precisely to reach it. If I had chosen another interaction challenge that would be less about the T shape and more exploration-centered, maybe like taking photographs of specific planets or exploring certain spots on foot, the game would have been more fun.

A good solution for the current "parkour" would be a few custom tutorials that intuitively explain how the flying works and let the player experiment a bit for themselves, maybe on a flat plane first, and show them why the projected path is useful.

## Further Ideas
All of the below-stated mechanics are just things that I didn't get around to implementing but seem promising and interesting to implement in the future if I ever return to this project.  

#### Black Hole
Replace the sun with an awesome-looking black hole shader. I even found one, but because I have no clue (yet) about game engine lighting (which I would have to redo after removing the sun) the black hole shader would likely have significant performance costs, especially when running on the Quest.

#### Carousel
I really wanted to implement a grabbing mechanic, because when I was testing out different VR games it was fun to hang on to a wall like a monkey with one hand, while also doing something else. My plan was to be able to grab onto a larger radius than the planets themselves, so you could essentially ride through their orbit like on a carousel. I did implement climbing in my test world and found it fun, but putting it into the main game would require something else you could do while grabbing on. Something that I thought could be interesting would be instead of having to fly through the coins, and thereby collect them, you would instead grab onto the planet, and use a gun that would be anchored to your hip to shoot the coins. Some testing would be necessary to see how difficult this is, from which further ideas could emerge.

#### Sandbox Planets
Spawning in new planets seems like a fun idea, but with the described unstableness of the system, this would cause everything to descend into chaos. A funny mechanic that could use this would function like a last-ditch effort to complete the goal, get a planet closer, or gravity boost off of the spawned planet before everything flings out in all directions. On the other hand, this spawned planet could ignore the solar system's gravity and just have a rigid orbit, only affecting the player's gravity.
Deleting planets could function in the same way.  

#### HUD shader
Self-explanatory, but debatable if the visual clutter is worth it. Something like a watch or arm display would be much better but would be non-trivial to implement. It could display things like speed, the closest planet, jetpack fuel, current score and time.

#### Vignette
I actually found an ["anime" vignette shader](https://github.com/MirzaBeig/Anime-Speed-Lines) which is supposed to give you a sensation of going fast, and figured out how to port it to VR (which went similarly well as the oceans and atmospheres...). However I decided not to add it to the final project, mainly out of performance reasons, but it might be nice to have as something you can turn on/off.



## Conclusion
***
I had a lot of fun doing this project, and learned more than I anticipated which is great. I would recommend this class if you like doing interesting projects where you can tinker tons, and therefore constantly encounter new problems and solutions.

Being curious and trying things out was definitely the most important part of this journey, as it allowed me to explore and learn where I didn't know the right answer.

As a byproduct, I also learned a lot about Unity and general Game Dev, while also scratching the surface of some interesting shader math, with many further topics to explore (knowing what I don't know).

Here are some more nice pictures

![solar system pic 1](./solar-system-view.jpeg)
![solar system pic 2](./solar-system-view2.jpeg)