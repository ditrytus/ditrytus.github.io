---
layout: post
title: "Stopping time"
description: "How to implement pause button."
date: 2018-03-27
tags: [Unity, C#, gamedev, time, pause]
comments: true
---

Most of video games require an ability to suspend action. There can be number of reasons to do so:
* we want to give player an access to set of menus like: game options, save, load etc.;
* game play itself might require opening full screen menus like maps, journals or inventory while allowing player not to worry on what is going on on the main scene;
* or just to give a player an ability to take a rest without sacrificing victory.

## Simple implementation

In my previous post In my previous post [Manipulating time]({% post_url 2018-03-26-manipulating-time %}) I described how you can use `Time.timeScale` field to manipulate the flow of time. The type of `Time.timeScale` is `float` so by invoking the following instruction in your script:

```
Time.timeScale = 0;
```

Yes, you guessed it, you will stop the clock entirely. It seems that implementing game pause is extremely simple.

All we need is an class that will have two states: paused and unpaused and will alternate `timeScale` between `1.0f` and `0.0f` accordingly:

```
using UnityEngine;

public class PauseManager
{
    public bool IsPaused { get; private set;}

    public PauseManager()
    {
        Resume();
    }
    
    public void Pause()    
    {
        Time.timeScale = 0.0f;
        IsPaused = true;
    }

    public void Resume()
    {
        Time.timeScale = 1.0f;
        IsPaused = false;
    }
}
```

Having this simple implementation, we need to add some UI element to for example a button which on click will either invoke `Pause()` or `Resume()` methods. For that purpose we can create a script that can be directly attached to GameObject in the editor:

```
using UnityEngine;
using UnityEngine.UI;

public class PauseButtonHandler : MonoBehaviour {

    private PauseManager pauseManager;
    private Text uiText;
    private Button button;

    void Start()
    {
        pauseManager = new PauseManager();
        uiText = GetComponentInChildren<Text>();
        button = GetComponent<Button>();
        button.onClick.AddListener(OnClick);
    }

    void OnClick()
    {
        if(pauseManager.IsPaused)
        {
            pauseManager.Resume();
            uiText.text = "Pause";
        }
        else
        {
            pauseManager.Pause();
            uiText.text = "Resume";
        }
    }
}
```

Additionally our script will also alter a text label on a button so that the player knows what the button will do.

*NOTE: `PauseButtonHandler` directly instantiates `PauseManager` which is not a good design. `PauseManager` manipulates a `Time.timeScale` which is global to our game so it would be natural to implement `PauseManager` as a singleton or somehow inject a shared instance to a component but I am omitting this aspect for the sake of focusing on pausing itself.*

Button with the attached script will effectively pause and resume the entire game. You can implement it differently for example to pause a game on a pressing a keyboard key like `P`. The key here is a `PauseManager` class that is responsible for managing pause state and manipulating the clock.

You can find a working example of the above code in `Assets/Scenes/pause_simple.unity` scene.

## Stopping time only for some scripts

The above example is simple and works but it has some limitations. Pausing a game that way will pause all the elements that are governed by `Time.time` which is basically *EVERYTHING* in your game except your custom scripts that explicitly use `Time.unscaledTime`. In some situations that is not what we want especially if there are some elements in our game that we want to keep animating even if the main gameplay is paused. For example:
* animated game menus or other effects that are visible only when the game is paused
* a tactical view of a strategy game where all the units are paused but you can still move camera or issue commands

Since we can't use `Time.timeScale` directly we need to wrap the game clock into our own class that we will use selectively in the parts of code that we want to stop.

```
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PausableTime
{
    public static PausableTime Instance { get; private set; }

    static PausableTime()
    {
        Instance = new PausableTime();
    }

    public bool IsPaused { get; private set; }

    private float pauseTime = 0.0f;

    public float gapTime;

    public float Time
    {
        get
        {
            float result;
            if (IsPaused)
            {
                result = pauseTime;
            }
            else
            {
                result =  UnityEngine.Time.time - gapTime;
            }
            
            return result;
        }
    }

    public float DeltaTime
    {
        get
        {
            float result;
            if (IsPaused)
            {
                result = 0.0f;
            }
            else
            {
                result = UnityEngine.Time.deltaTime;
            }
            return result;
        }
    }

    public PausableTime()
    {
        IsPaused = false;
        gapTime = 0.0f;
    }

    public void Pause()
    {
        if (IsPaused)
        {
            return;
        }

        pauseTime = Time;

        IsPaused = true;
    }

    public void Resume()
    {
        if (!IsPaused)
        {
            return;
        }

        gapTime = UnityEngine.Time.time - pauseTime;

        IsPaused = false;
    }
}
```

A lot of code but the idea is pretty simple here. The class is similar to the `PauseManager` in a sense that it exposes `Pause()` and `Resume()` methods and also keeps pause state in `IsPaused` property. Additionally it exposes `Time` and `UnscaledTime` which mimic `Time.time` and `Time.unscaledTime` field, however `PausableTime` does not depend on `Time.timeScale` but implements the logic of stopping the clock itself. The goal here is to be able to stop the clock without changing `Time.timeScale`. Algorithm used is very simple:

1. On `Pause()` the class stores the current value of `Time` to a private field `pauseTime`.
2. On `Resume()` the class stores the amount of time for how long the clock was stopped in  private field `gapTime`.
3. When we want to read the value of `Time` we will do it differently depending on whether the time is paused or not:
    1. If the clock `IsPaused` then we only need to return `pauseTime` since we conveniently saved the value of clock when pause occurred.
    2. If the clock is not paused we can't just return the value of `Time.time`, we have to subtract the length of the last pause (`gapTime`) from the system clock to avoid the value of `Time` skipping forward when we resume.
4. When we want to read the value of `DeltaTime` the algorithm is much simpler than time, we only need to return `0.0f` if the clock is paused because no time should have elapsed since the last frame, or `Time.deltaTime` is the game is not paused.

Additionally the class is already implemented as singleton because it probably will be accessed by many scripts in our code.

Having our new `PausableTime` class we have to connect it to our Pause button. For that we need to reimplement `PauseButtonHandler`:

```
using UnityEngine;
using UnityEngine.UI;

public class PausableTimeButtonHandler : MonoBehaviour {

    private Text uiText;

    void Start()
    {
        uiText = GetComponentInChildren<Text>();
        GetComponent<Button>().onClick.AddListener(OnClick);
    }

    void OnClick()
    {
        if(PausableTime.Instance.IsPaused)
        {
            PausableTime.Instance.Resume();
            uiText.text = "Pause";
        }
        else
        {
            PausableTime.Instance.Pause();
            uiText.text = "Resume";
        }
    }
}
```

No big change here, just instead of using a local variable `pauseManager` we are accessing `PausableTime.Instance` singleton.

Now we need to modify our scripts so that they know about our newly created `PausableTime` class and use it instead of the standard `Time`.

Let's look at a script that rotates an object on all 3 axes using `Time.deltaTime`:

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Spinner : MonoBehaviour {
    
    public float speed = 180.0f;
    
    void Update () {
        var delta = Time.deltaTime;
        transform.Rotate(delta * speed, delta * speed, delta * speed);    
    }
}
```

It's pausable counterpart will look like this:

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PausableSpinner : MonoBehaviour {
    
    public float speed = 180.0f;

    void Update () {
        var delta = PausableTime.Instance.DeltaTime;
        transform.Rotate(delta * speed, delta * speed, delta * speed);    
    }
}
```

The only change that was needed was to replace `Time.deltaTime` to `PausableTime.Instance.DeltaTime`.

Let's look at another script that swings an object smoothly up and down:

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PausableSwing : MonoBehaviour {

    private float startTime;
    private float startY;

    public float deviation = 1.0f;
    public float interval = 1.0f;
    
    void Start () {
        startTime = PausableTime.Instance.Time;
        startY = transform.position.y;
    }
    
    void Update ()
    {
        transform.position = new Vector3(
            transform.position.x,
            startY + Mathf.Sin(2 * Mathf.PI * (PausableTime.Instance.Time / interval)) * deviation,
            transform.position.z);
    }
}
```

The above script uses a `Mathf.Sin()` function to oscillate a value between `1.0f` and `-1.0f`. By injecting `PausableTime.Instance.Time` to it's parameter we are able to stop that motion.

## Summary

You can use the above method of mixing `UnityEngine.Time` and `PausableTime` to create time dependent scripts that can be paused without having to stop the entire game. You could try to implement your scripts in a way that depending on some public boolean field they will either be pausable or not.

## Further challenges

Class `PausableTime` only allows us to control a time flow in scripts but we can't use it to selectively stop physics simulation or pause an execution of [Destroy(gameObject, timeout)](https://docs.unity3d.com/ScriptReference/Object.Destroy.html) function. I will address these issues in the further posts.