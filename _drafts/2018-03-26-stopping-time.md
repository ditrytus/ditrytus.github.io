---
layout: post
title: "Stopping time"
description: "How to implement pause."
date: 2018-03-26
tags: [Unity, C#, gamedev, time, pause]
comments: true
---

Most of video games require an ability to suspend action. There can be number of reasons to do so:
* we want to give player an access to set of menus like: game options, save, load etc.;
* game play itself might require opening full screen menus like maps, journals or inventory while allowing player not to worry on what is going on on the main scene;
* or just to give a player an ability to take a rest without sacrificing victory.

## What Unity offers

Unity engine offers a static `Time` class that servers as an engines clock and allows to manipulate a flow of time. Here are some of fields that are exposed by `Time`:

`Time.time` - returns number of seconds since the game started expressed as `float`.

`Time.timeScale` - gets or sets a `float` value that describes how fast the time passes in your game. 

`Time.unscaledTime` - as `Time.time` it returns a number of seconds since the game started however `Time.timeScale` has no effect on it. It will always show real clock seconds.

Check an [official documentation](https://docs.unity3d.com/ScriptReference/Time.html) for more fields.

### Scaling time

So how `Time.timeScale` affects the speed of game clock? Let's look at a few examples:
* Value `1.0f` means that a clock in the game clock will tick with the same speed as the real time.
* Value `2.0f` means that for each second of real time, two seconds will pass in the game.
* Value `0.5f` means that for every second of real time only half of second will pass in the game.

`Time.timeScale` can have any non-negative value, so in general for every real second `Time.time` will advance for `Time.unscaledTime * Time.timeScale` seconds. Setting values bigger than `1.0f` can be used for effects like "fast forward" while with values smaller than `1.0f` you can use make "bullet time" or other "slow motion" effect.

You can set the value of `Time.timeScale` either globally in the editor:

1. Open *TimeManager* menu: **Edit -> Project Settings -> Time**
2. In *Inspector* window set *Time Scale*:
![TimeManger Inspector window with Time Scale highlighted]({{ site.baseurl }}/assets/images/TimeManager_Time_Scale.png)

Or you can set it in a script:

```
Time.timeScale = 3.0f;
```

Something a documentation does not mention is that value of `Time.time` does not always equal `Time.unscaledTime * Time.timeScale`. `Time.timeScale` only describes how fast `Time.time` advances while `Time.time` keeps it's continuity even when `Time.timeScale` is changed. Here is an example:

Let's assume the following situation. We have a script that after 5 seconds