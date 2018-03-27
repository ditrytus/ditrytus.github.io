---
layout: post
title: "Manipulating time"
description: "How to influence the flow of time in Unity."
date: 2018-03-26
tags: [Unity, C#, gamedev, time]
comments: true
---

## `Time` class

Unity engine offers a static `Time` class that servers as an engines clock and allows to manipulate a flow of time. Here are some of fields that are exposed by `Time`:

| File | Description |
| --- | --- |
| `Time.time` | Returns number of seconds since the game started expressed as `float`. |
| `Time.timeScale` | Gets or sets a `float` value that describes how fast the time passes in your game. |
| `Time.unscaledTime` | Like `Time.time` it returns a number of seconds since the game started however `Time.timeScale` has no effect on it. It will always show real clock seconds. |

Check an [official documentation](https://docs.unity3d.com/ScriptReference/Time.html) for more fields.

## Scaling time

So how `Time.timeScale` affects the speed of game clock? Let's look at a few examples:
* Value `1.0f` means that a clock in the game clock will tick with the same speed as the real time.
* Value `2.0f` means that for each second of real time, two seconds will pass in the game.
* Value `0.5f` means that for every second of real time only half of second will pass in the game.

`Time.timeScale` can have any non-negative value, so in general for every real second `Time.time` will advance for `Time.unscaledTime * Time.timeScale` seconds. Setting values bigger than `1.0f` can be used for effects like "fast forward" while with values smaller than `1.0f` you can use make "bullet time" or other "slow motion" effect.

## Changing `Time.timeScale`

You can set the value of `Time.timeScale` either globally in the editor:

1. Open *TimeManager* menu: **Edit -> Project Settings -> Time**
2. In *Inspector* window set *Time Scale*:
![TimeManger Inspector window with Time Scale highlighted]({{ site.baseurl }}/assets/images/TimeManager_Time_Scale.png)

Or you can set it in a script:

```
Time.timeScale = 3.0f;
```

## In depth time scaling

Something a documentation does not mention is that value of `Time.time` does not always equal `Time.unscaledTime * Time.timeScale`. `Time.timeScale` only describes how fast `Time.time` advances while `Time.time` keeps it's continuity even when `Time.timeScale` is changed. Here is an example:

Let's assume the following situation. We start a game with `Time.timeScale == 1.0f`. After 5 seconds we set `Time.timeScale = 0.5f`. After another 5 more (real) seconds we set `Time.timeScale = 2.0f` and after another 5 more real seconds we set it back to `Time.timeScale == 1.0f`. If `Time.time` would simply equal `Time.unscaledTime * Time.timeScale` we would get the following values:

| `Time.unscaledTime `| `Time.timeScale` | `Time.time` | Note |
| ---- | --- | ---- | --- |
| 0.0 | 1.0 | 0.0 | |
| 5.0 | 0.5 | 2.5 | Clock would move backward! |
| 10.0 | 2.0 | 20.0 | Clock would skip forward! |
| 15.0 | 1.0 | 15.0 | Clock would move backward again! |
| 20.0 | 1.0 | 20.0 |  |

Note that every time we would set `Time.timeScale` the value of time would either instantly move backward or skip some range forward. That would break the continuos flow of time that we intuitively expect. Luckily that is not the case. The real values are:

| `Time.unscaledTime `| `Time.timeScale` | `Time.time` | Note |
| ---- | --- | ---- | --- |
| 0.0 | 1.0 | 0.0 | |
| 5.0 | 0.5 | 5.0 | That is how many seconds `Time.time` counted with normal speed to this point. |
| 10.0 | 2.0 | 7.5 | 5.0 original seconds + 5.0 more seconds * 0.5 previous `timeScale` |
| 15.0 | 1.0 | 17.5 | Previous 7.5 counted with various speeds + 5.0 * 2.0 |
| 20.0 | 1.0 | 22.5 | Previous 17.5 seconds with 5.0 more passed with normal speed. |

The calculation of `Time.time` might seem obscure but you don't need to worry about it. Unity keeps track of it for you.

## Summary

Takeaways from this for you is to remember that:

* `Time.time` never goes backward.
* `Time.time` never skips a value.
* `Time.timeScale` controls only how fast `Time.time` increments.