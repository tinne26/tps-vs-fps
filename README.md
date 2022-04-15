# Ebiten TPS vs FPS

This document aims to help users of the [Ebiten](https://github.com/hajimehoshi/ebiten) game engine and shed some light on the topic of ticks per second (TPS) vs frames per second (FPS), which has proven to be a common point of confusion both for newcomers and experienced devs alike.

## Time in game engines
Let's say we want to create an impressive game where a shrimp appears from the left of the screen and starts moving to the right. Then, at any time, we want to allow the player to press space to make the shrimp change direction!

Ok. ¯\\\_(ツ)\_/¯

How do you update the shrimp's position? At which point? How much? Do you use a physical simulation that considers speed and elapsed time, or do you just move it a fixed amount of pixels on each update?

Well, *it depends*. Most modern game engines offer two ways to work with time:
- **Delta times**: the time elapsed between frames. This can vary depending on the frames per second (FPS) the game is running at. With delta times, you could figure out how much to move the shrimp at each frame by multiplying the elapsed time and the shrimp's speed. Classic physics.
- **Fixed timestep loops**: fixed timestep loops allow a function to be called periodically, at a fixed interval, so you can pretend that time passes uniformly between updates. With a fixed timestep loop, you could move the shrimp a fixed distance on each update. If you wanted your shrimp to move 100 pixels per second and your fixed timestep was 1/50 seconds, on each update you would want to move the shrimp `100*(1/50) = 2` pixels.

For example, in [Unity](https://docs.unity3d.com/Manual/TimeFrameManagement.html) you have `Time.deltaTime` and `Time.fixedDeltaTime`. In [Godot](https://docs.godotengine.org/en/stable/tutorials/scripting/idle_and_physics_processing.html), instead, you have separate methods for *idle processing* (frame by frame) and *physics processing* (fixed timestep).

In Ebiten, the `Update` function is part of a *fixed timestep loop*, and it's called based on the ticks per second (TPS) of your game. By default, Ebiten's TPS is 60, which means that the `Update` method will be called 60 times per second. In other words, unless you modify the TPS with [`SetMaxTPS`](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#SetMaxTPS), the fixed timestep will be `1000/60 = 16.666` milliseconds. To use delta times, instead, you would use the standard Golang `time` package, but we won't talk about this now.

So... if **both approaches can be used, what should you do?** Well, in Ebiten, you almost always want to stick to `Update` and the fixed timestep loop... but to understand why, we first need to talk about the advantages and disadvantages of each method.

## Fixed timestep vs variable deltas
Let's start with *fixed timestep loops*. Using a fixed timestep and letting the game engine call your game's update method seems like the easiest way to make games:
- You can assume time passes uniformly. You don't need tricky delta time calculations (or even any time calculations at all).
- You can make your game deterministic, storing inputs for replays or faking them for in-game scenes or automated testing.

On the other hand, there are also some downsides:
- Fixed timesteps can slow down your whole game if the game starts lagging.
- Interpolations based on fixed timesteps may not be as smooth as those based on finer variable time deltas.

Let's look at *variable time deltas* now:
- Simulation can be smoother under a variable frame rate.

- Time computations are more complex.

- Time deltas tied to the game logic will make the game non-deterministic.

- Too big time deltas can break your game. The classic example are bullets going through walls due to missing the collision window (too much lag causing the time delta to grow too big), but there are many other ways in which things can go wrong. 

  Notice how the Godot game engine specifically named its fixed timestep loop *physics processing*, highlighting how this is an important issue and why you generally don't want your physics and some other elements of your game to be processed with variable time deltas.

Most modern games are actually using both methods for different parts of the game. This makes sense because big games nowadays end up running in wildly different devices, with many configurable quality levels, with gamers pushing their hardware to run at 140FPS and all that stuff. Despite all this, we still want first-person shooters to be sharply responsive, open world games to keep up the pace no matter how crowded the area gets, high-quality animations to play as smoothly as possible... and this means that game developers and modern engines need to use every trick in the book to try to push for the best results.

If you are working with Ebiten, though, **those are almost never your main concerns**. Ebiten is mostly used for 2D games, which will often feature low resolution assets, pixel art, shrimps and choppy animations. Here, keeping it simple and preserving determinism are typically more important than smoothness under astringent performance constraints. You should have enough headroom to make your games work properly even on lower end devices while using only the fixed timestep `Update` method.

## Back to Ebiten: Update vs Draw
Ebiten has two main methods that you have to implement to make your game work: `Update` and `Draw`. As we have seen, `Update` is called regularly based on the ticks per second (TPS) configured for your game. On the other hand, `Draw` will be called based on the refresh rate of your screen. If your screen has a refresh rate of 60FPS, by default Ebiten will try to call `Draw` 60 times each second.

Your main logic should be processed on `Update`, using the fixed timestep. If you need smoother visual effects (maybe related to shaders), you may compute delta times by yourself in the `Draw` method, but understand that this is rarely needed; don't complicate your own life unless you have a good reason for it.

In very rare cases (*stares fixedly at “very rare”*), one may decide to use only time deltas and process all the logic inside `Draw` itself.

If the game lags, Ebiten will prioritize TPS over FPS in order to avoid the game slowing down. Some people get really concerned about this, but if your game lags what you should be doing is profiling and optimizing your code, *not worrying about time deltas*.

## Other common concerns
**But TPS are not fixed? `CurrentTPS` can return different values? How is that reliable?**

If your game is not lagging, `Update` may be called a few microseconds early or late, but TPS will be stable and compensated in the long run. You won't be losing time or advancing in time unless the game starts *lagging a lot* (and in that case, you should start profiling and optimizing instead).

`CurrentTPS` is mostly a debug method that you can use while developing to keep track of the game performance. That said, the best method to keep track of your game's performance is to set `FPSModeVsyncOffMaximum` (only for development, not releases!) and displaying the `CurrentFPS` value in the screen. FPS will start fluctuating earlier than TPS if something is lagging.

**But I learned that using time deltas is the way to do things right!**

It's the main method to manage time in most game engines and the main topic of most "game loop implementation" tutorials. That explains why a lot of people is confused when working with Ebiten, but the "Fixed timestep vs variable deltas" section already discussed the advantages and disadvantages of each method; for a library like Ebiten, the fixed timestep loop makes perfect sense.

**But if `Update` and `Draw` can be called at different rates, that's... weird?**

You may have `Update` be called multiple times consecutively before `Draw`, or the other way around. It's good to keep this in mind in order to avoid developing an incorrect mental model of how the main Ebiten loop works, but once you understand it it's not that surprising.

**But is it still reasonable to compute time deltas if I really need them?**

In some cases —for example when working with shaders—, if you want some visual effect to be as smooth as possible and have good reason to believe that `Update` will be called fairly less often than `Draw` (so, TPS are lower than FPS), computing time deltas may make sense. In most cases, though, worrying about time deltas in Ebiten causes more harm than good. If you have read this document and understand the differences clearly, do whatever you want. Otherwise, keep your hands out of `time.Now()` and continue trying to understand.

What you should **never** do is computing time deltas in the `Update` method: if computing elapsed times in a method that's part of a fixed timestep loop doesn't trigger any alarms, you probably still don't fully grasp the difference between TPS and FPS, between fixed timesteps and variable time deltas, between `Update` and `Draw`.

**Can I change TPS during the game?**

The API allows it, and Hajime Hoshi mentioned using it to implement a turbo mode for a game. It's really hard to come up with reasonable use-cases for it outside a few tricks like these, though.

**But you are wrong about...** 

Feel free to drop by [Ebiten's discord server](https://discord.gg/3tVdM5H8cC) and duel ;)

## Quick summary
- `Update` is called on a **fixed timestep loop** controlled by the TPS (ticks per second).
- `Draw` is called based on the refresh rate of the display in use, which determines the maximum frames per second (FPS) your game may run at.
- You rarely need to compute time deltas on your games. Use fixed values and rely on `Update` being called at fixed intervals instead.
- If you are computing time deltas in your game anyway, it should only be in the `Draw` method.
- For the kind of games developed in Ebiten, the *ease of use and determinism* provided by a fixed timestep loop are typically more important than *smoothness and responsiveness under high system pressure*. Ebiten games rarely lead to high system pressure, so they should be able to perform stably.
