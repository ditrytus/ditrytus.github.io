---
layout: post
title: "My attempt on Unity with UniRx"
description: "How I redid the Unity's basic Space Shooter tutorial's scripts with UniRx."
date: 2017-02-14
tags: [Unity, ReactiveX, UniRx, C#]
comments: true
---

## Unity

Recently I started learning Unity. [Unity](https://unity3d.com) is game engine that can be used free of charge, is very simple to learn and comes with it's own IDE. To me a software is as good as it's documentation and Unity comes with a very good and comprehensive reference. You can find [good tutorials](https://unity3d.com/learn/tutorials) on the official website that will walk you through all the important parts.

I was doing a [Space Shooter](https://unity3d.com/learn/tutorials/projects/space-shooter-tutorial) tutorial from that official website when I quickly noticed, that scripting part of the lessons are aimed for the people who don't have experience with programming and/or C#. In my opinion scripts presented in the tutorial combine too many responsibilities and are not clear to read and understand. Of course when you are following the tutorial and are guided by the tutor through the creation of every single line they seem obvious but I felt that they can be done better.

## Reactive Extensions for Unity

Another thing I was recently exploring is [reactive programming](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754) through a library that supports that programming model [Reactive Extensions](http://reactivex.io). In essence, Reactive Extensions allows you to
model your program's behavior as a combination of time dependent streams of values. Every input to your program, IO, events and passage of time, can be modeled as streams of values. Reactive Extensions allow you to transform and subscribe to these streams in a declarative way which result in a much cleaner and readable code especially when you are dealing with parallel or time dependent logic. Games logic tend to be very time dependent, you constantly have to handle all sorts of events like collisions, game state changes and user input.

Reactive Extensions supports many languages and comes in a lot of flavours and yes, there is a version for Unity too which is called UniRx. You can find [UniRx on github](https://github.com/neuecc/UniRx) or in a [Unity's Asset Store](https://www.assetstore.unity3d.com/en/#!/content/17276).

In this post I will compare the original tutorial scripts with my versions written in UniRx. Two things to note though:
1. I am a beginner when it comes to Reactive Extensions, and I stumbled upon a few problems which I resolved in my own way. If any of you have a better idea how to approach certain problems, or if you'll see that I am completely butchering the idea, please let me know.
2. I decided to rewrite every script with UniRx and in a few of them there is a little or no gain in readability or length of the script, I am aware of that.
3. I assume you did the Space Shooter tutorial yourself or that you are experienced enough to understand simple Unity C# scripts and know the concept of game objects.

You can play the game in the browser:

[http://spaceshooter.gruszecki.software](http://spaceshooter.gruszecki.software)

![Space shooter screenshot]({{ site.baseurl }}/assets/images/spaceshooter_screenshot.png)

## PlayerController

First lets look at tutorial's original `PlayerController` class. This script is attached to a player GameObject and contains *all* the behaviors of player's ship.

{% highlight csharp linenos %}
public class PlayerController : MonoBehaviour
{
  public float speed;
  public float tilt;
  public Boundary boundary;

  public GameObject shot;
  public Transform shotSpawn;
  public float fireRate;
     
  private float nextFire;
    
  void Update ()
  {
    if (Input.GetButton("Fire1") && Time.time > nextFire) 
    {
      nextFire = Time.time + fireRate;
      Instantiate(shot, shotSpawn.position, shotSpawn.rotation);
      GetComponent<AudioSource>().Play ();
    }
  }

  void FixedUpdate ()
  {
    float moveHorizontal = Input.GetAxis ("Horizontal");
    float moveVertical = Input.GetAxis ("Vertical");

    Vector3 movement =
      new Vector3 (moveHorizontal, 0.0f, moveVertical);
    GetComponent<Rigidbody>().velocity = movement * speed;
    
    GetComponent<Rigidbody>().position = new Vector3
    (
      Mathf.Clamp(
        GetComponent<Rigidbody>().position.x,
        boundary.xMin,
        boundary.xMax), 
      0.0f, 
      Mathf.Clamp(
        GetComponent<Rigidbody>().position.z,
        boundary.zMin,
        boundary.zMax)
    );
    
    GetComponent<Rigidbody>().rotation =
      Quaternion.Euler (
        0.0f,
        0.0f,
        GetComponent<Rigidbody>().velocity.x * -tilt);
  }
}
{% endhighlight %}

`PlayerController` does the following things:

1. When `Fire1` button is pressed then player's ship shoots, with a rate not faster then `fireRate` shots per second, playing a sound on every shot.
2. When direction buttons are pressed player's ship moves in that direction.
3. Player's position is clamped to a certain boundary.
4. Player's ship tilts left and right proportionally to the velocity in a given direction.

One thing that I like about Unity is that it's scripting API allows to have so many different behaviors with a few lines of code. What I don't is that because of that simplicity it encourages to pack a lot of unrelated things in one script. My first improvement was to split all distinctive behaviors into separate scripts.

### TiltOnMove

First I extracted tilting behaviour to `TiltOnMove.cs`:

{% highlight csharp linenos %}
public class TiltOnMove : RxBehaviour
{
  public float tilt;

  void Start ()
  {
    var sub1 = Observable.EveryFixedUpdate()
      .Subscribe(_ =>
      {
        GetComponent<Rigidbody>().rotation =
          Quaternion.Euler(
            0.0f,
            0.0f,
            GetComponent<Rigidbody>().velocity.x * -tilt);
      });

    AddSubscriptions(sub1);
  }
}
{% endhighlight %}

The new script has only one parameter `tilt` and does only one thing: tilts object on z axis based on x axis velocity. Advantages are that if any other object in the game should have been tilted according to the same logic, you only need to attach this script and voila!

As you can see instead of using `void FixedUpdate ()` method, I am subscribing to `Observable.EveryFixedUpdate()` observable. The resulting code seems to have more UniRx clutter then necessary and here is where I would consider not using UniRx at all costs.

New base class `RxBehaviour` and `AddSubscriptions(sub1);` method I will explain later on.

### MoveOnInput

Next extracted class is `MoveOnInput.cs`:

{% highlight csharp linenos %}
public class MoveOnInput : RxBehaviour
{
  public float speed;
  
  void Start ()
  {
    var sub1 = Observable.EveryFixedUpdate()
      .Subscribe(_ =>
      {
        GetComponent<Rigidbody>().velocity =
          new Vector3(
            Input.GetAxis("Horizontal"),
            0.0f,
            Input.GetAxis("Vertical"))
          * speed;
      });

    AddSubscriptions(sub1);
  }
}
{% endhighlight %}

The sole responsibility of this script is to set objects velocity proportionally to user input and some `speed` factor. By inlining a few local variables: `float moveHorizontal`, `float moveVertical` and `Vector3 movement`, the whole thing could have been compacted into one, still readable instruction.

Like the previous one, this script does not yet benefit from the usage of UniRx but it's single responsibility would allow to attach it to any object that's movement needs to be controlled by user input.

### BoundMovement

Now that we can move objects, we should also be able to limit that movement.

{% highlight csharp linenos %}
public class BoundMovement : RxBehaviour
{
  public Boundary boundary;

  void Start ()
  {
    var sub1 = Observable.EveryFixedUpdate()
      .Subscribe(_ =>
      {
        transform.position = new Vector3(
          Mathf.Clamp(
            transform.position.x,
            boundary.xRange.min,
            boundary.xRange.max),
          Mathf.Clamp(
            transform.position.y,
            boundary.yRange.min,
            boundary.yRange.max),
          Mathf.Clamp(
            transform.position.z,
            boundary.zRange.min,
            boundary.zRange.max)
        );
      });
    
    AddSubscriptions(sub1);
  }
}
{% endhighlight %}

This class limits the position of any object to a certain boundary. Same thing can be achieved with a properly set colliders but for simple game like this doing it without the aid of the physics engine is good enough. Again, with that script extracted we can limit the movement of any game object.

#### Boundary

You probably noticed that I am clamping all 3 axes instead of just `x` and `z`; and that `boundary.xMin` changed into `boundary.xRange.min`. That's because I also refactored `Boundary` class. The original was:

{% highlight csharp linenos %}
public class Boundary 
{
  public float xMin, xMax, zMin, zMax;
}
{% endhighlight %}

It has two sets of similar variables `*Min` and `*Max`. If you would want to have a support for y axis you would get third pair of those. Such repetition screams for introducing a separate structure for that. Since `Min` and `Max` defines a range of `float` values so we can introduce:

{% highlight csharp linenos %}
public class FloatRange {
  public float min;
  public float max;
}
{% endhighlight %}

and we can refactor our `Boundary` class to:

{% highlight csharp linenos %}
public class Boundary
{
  public FloatRange xRange;
  public FloatRange yRange;
  public FloatRange zRange;
}
{% endhighlight %}

In addition such structure looks nicer when serialized in the editor:

![Boundary class in editor]({{ site.baseurl }}/assets/images/boundary_editor.png)

### SpawnOnFire

So far we haven't seen any advantages of using UniRx alone, but the next script shows in what situations it shines. `SpawnOnFire` script on button press instantiates a shot game object in a certain position limiting it to a certain rate.

{% highlight javascript linenos %}
public class SpawnOnFire : RxBehaviour
{
  public GameObject spawnedObject;

  public Transform spawnPoint;

  public float throttle;

  public string buttonName;

  void Start () {
    var sub1 = Observable.EveryUpdate()
      .Where(_ => Input.GetButton(buttonName))
      .ThrottleFirst(TimeSpan.FromSeconds(throttle))
      .Subscribe(_ => {
        Instantiate(
          spawnedObject,
          spawnPoint.position,
          spawnPoint.rotation);
        GetComponent<AudioSource>().Play();
      });
    
    AddSubscriptions(sub1);
  }
}
{% endhighlight %}

Let's look closer into what's going on in a separate lines:

12. `Observable.EveryUpdate()` - we are taking a stream of all game logic updates;
13. `.Where(_ => Input.GetButton(buttonName))` - we are filtering above stream to only those updates when a certain button is pressed;
14. `.ThrottleFirst(TimeSpan.FromSeconds(throttle))` - `Throttle` filters a stream further in such a way that subsequent signals have to occur at least certain `TimeSpan` apart. More frequent values are just discarded. Method `ThrottleFirst` lets through the first element in the stream without the timeout.
15. `.Subscribe(...)` - we are doing our shot instantiation and playing sound. 

To me this code is much more expressive than the original implementation where private field `float nextFire` was used to hold calculated time of when the next shot is allowed.

## GameController

Let's look first at an original code of `GameController` class:

{% highlight csharp linenos %}
public class GameController : MonoBehaviour
{
  public GameObject[] hazards;
  public Vector3 spawnValues;
  public int hazardCount;
  public float spawnWait;
  public float startWait;
  public float waveWait;
  
  public GUIText scoreText;
  public GUIText restartText;
  public GUIText gameOverText;
  
  private bool gameOver;
  private bool restart;
  private int score;
  
  void Start ()
  {
    gameOver = false;
    restart = false;
    restartText.text = "";
    gameOverText.text = "";
    score = 0;
    UpdateScore ();
    StartCoroutine (SpawnWaves ());
  }
  
  void Update ()
  {
    if (restart)
    {
      if (Input.GetKeyDown (KeyCode.R))
      {
        SceneManager.LoadScene(
          SceneManager.GetActiveScene().buildIndex);
      }
    }
  }
  
  IEnumerator SpawnWaves ()
  {
    yield return new WaitForSeconds (startWait);
    while (true)
    {
      for (int i = 0; i < hazardCount; i++)
      {
        GameObject hazard =
          hazards [Random.Range (0, hazards.Length)];
        Vector3 spawnPosition = new Vector3 (
          Random.Range (-spawnValues.x, spawnValues.x),
          spawnValues.y,
          spawnValues.z);
        Quaternion spawnRotation = Quaternion.identity;
        Instantiate (hazard, spawnPosition, spawnRotation);
        yield return new WaitForSeconds (spawnWait);
      }
      yield return new WaitForSeconds (waveWait);
      
      if (gameOver)
      {
        restartText.text = "Press 'R' for Restart";
        restart = true;
        break;
      }
    }
  }
  
  public void AddScore (int newScoreValue)
  {
    score += newScoreValue;
    UpdateScore ();
  }
  
  void UpdateScore ()
  {
    scoreText.text = "Score: " + score;
  }
  
  public void GameOver ()
  {
    gameOverText.text = "Game Over!";
    gameOver = true;
  }
}
{% endhighlight %}

Now boy oh boy we have a lot of stuff going on here. If you can right away, just by looking at this code, tell what it does then you should be getting your PhD now. In the tutorial the code is constructed line by line with a very detailed explanation, but someone who hasn't heard that story or is not familiar with the game will have a hard time to read it. Basically what it does is:

1. Spawns waves of asteroids in fixed intervals.
2. Allows to increase a player's score.
3. Ends the game and allows to restart it.

Note that numbers 2 and 3 are public methods, so it's not GameController's responsibility to know when the game ends, or when to give points, but it knows how to do both these things. So you can't tell from that code what are actual rules of the game.

Another thing I don't like here is that Game Over state and resetting behaviour are smeared over several methods, not only `public void GameOver()`, but you can find a bits of Game Over logic in `Start()`, `Update()` and `SpawnWaves()`. So not only the class itself has mixed responsibilities, but also methods do other stuff than the name suggests.

My approach to deal with it was to actually split the above code into separate classes: `GameOverController`, `HazardWavesController` and `ScoreController`.

### ScoreController

First let's look at the code:

{% highlight csharp linenos %}
public class ScoreController : RxBehaviour
{
  public Text scoreText;
  
  public int score;

  public Subject<int> newScore;

  void Start () {
    newScore = new Subject<int>();

    var sub1 = newScore
      .Select(n => score = score + n)
      .Select(s => $"Score: {s}")
      .SubscribeToText(scoreText);

    newScore.OnNext(score);

    AddSubscriptions(sub1);
  }
}
{% endhighlight %}

Responsibility of that class is simple: to manage player's score. I am exposing public subject `newScore` which is a key here. You can read more on Subjects (here)[http://reactivex.io/documentation/subject.html], but basically they are objects through which you can both send signals, and subscribe to them. On one side I am exposing subject so that other parts of the game could submit new points (I'll tell later what that other part is) and internally subscribing to it to calculate a sum of all points, and to update an appropriate label. UniRx comes with a convenient method `SubscribeToText(...)` which allows to update a `Text` UI object with the values coming in the stream. Basically it works like a data simple binding framework for Unity UI.

### GameOverController

{% highlight csharp linenos %}
public class GameOverController : RxBehaviour
{
  public ISubject<Unit> gameOver;
  public float restartDelay;

  public Text gameOverText;
  public Text restartText;

  void Start () {
    gameOver = new Subject<Unit>();

    // if game is over, we are displaying an appropriate text
    var sub1 = gameOver.Subscribe(_ => {
      gameOverText.gameObject.SetActive(true);
    });

    var sub2 = gameOver

      // after a few seconds 
      .Delay(TimeSpan.FromSeconds(restartDelay))

      // we are showing a text saying
      // that a player can restart the game
      .Subscribe(_ => {
        restartText.gameObject.SetActive(true);

        Observable
          .EveryUpdate()

          // if player presses `R` button
          .Where(__ => Input.GetKeyDown(KeyCode.R))
          .First()

          // the game will restart
          .Subscribe(___ => {
            SceneManager.LoadScene(SceneManager.GetActiveScene().name);
          });
      });
    
    AddSubscriptions(sub1, sub2);
  }
}
{% endhighlight %}

Above class exposes a `gameOver` subject to subscribe to end of game event. Look at the comments in the source code, that's what I like about UniRx, manipulating observables makes much more expressive code than the original juggling with `bool` flags. You can practically read your logic like regular text.

### HazardWavesController

{% highlight csharp linenos %}
public class HazardWavesController : RxBehaviour {
  public GameObject hazard;
  public Vector3 spawnPositions;
  public FloatRange spawnPositionXRange;
  public int initialHazardCount;
  public float spawnInterval;
  public float waveDelay;
  public float waveInterval;
  private int hazardCount;

  public ISubject<Unit> asteroidDestroyed;
  public ISubject<Unit> playerDestroyed;

  public HazardWavesController()
  {
    asteroidDestroyed = new Subject<Unit>();
    playerDestroyed = new Subject<Unit>();
    
    hazardCount = initialHazardCount;
  }

  void Start () {
    var waves = new Subject<Unit>();

    var sub1 = waves
      .Subscribe(_ => {
        Observable
          .Interval(TimeSpan.FromSeconds(spawnInterval))
          .Take(hazardCount)
          .Subscribe(
            __ =>
              SpawnAsteroid(),
            () =>
              Observable
                .Timer(TimeSpan.FromSeconds(waveInterval))
                .Subscribe(___ => {
                  waves.OnNext(Unit.Default);
                }));
        hazardCount++;
      });

    waves.OnNext(Unit.Default);

    AddSubscriptions(sub1);
  }

  void SpawnAsteroid()
  {
    var asteroid = Instantiate(
      hazard,
      new Vector3(
        UnityEngine.Random.Range(
          spawnPositionXRange.min, 
          pawnPositionXRange.max),
        spawnPositions.y,
        spawnPositions.z),
      Quaternion.identity);
    
    var destroyOnCollision =
      asteroid.GetComponent<DestroyOnCollision>();

    destroyOnCollision.asteroidDestroyed = asteroidDestroyed;
    destroyOnCollision.playerDestroyed = playerDestroyed;
  }
}
{% endhighlight %}

This one is a bit longer but only because instantiating an asteroid code is here. Again there is only one responsibility of that class: to create waves of asteroids. I did change a logic here a bit in comparison to the original tutorial code. In original, each wave contains exactly the same number of asteroids, in my code however, each wave contains an increasing number of asteroids.

How it works? There is a `waves` subject which, when signaled, will create a new observable (this is an actual name of those streams of signals I kept talking about) that will create a stream of a `hazardCount` number of signals `spawnInterval` seconds apart:

{% highlight csharp %}
Observable
  .Interval(TimeSpan.FromSeconds(spawnInterval))
  .Take(hazardCount)
  .Subscribe(
    __ =>
      SpawnAsteroid(),
    () =>
      ...
  );
{% endhighlight %}

The `Subscribe` method that follows, has two parameters: first one is a function that will be invoked on every signal in our wave, where we instantiate our asteroid using `void SpawnAsteroid()`; second is invoked when the stream of signal ends. `Take(hazardCount)` transforms an original stream into one that has only certain amount of items and then ends, so the second function will be invoked after the last asteroid in the wave

When our wave ends, I am setting a timer to `waveInterval` amount of seconds to trigger another wave and then the entire thing loops back to the beginning:

{% highlight csharp %}
Observable
  .Timer(TimeSpan.FromSeconds(waveInterval))
  .Subscribe(___ => {
    waves.OnNext(Unit.Default);
  }));
{% endhighlight %}

I am not super proud of this part because couldn't find a clever operators to produce a sequence of signals that are certain amount of time apart and come in a groups of increasing sizes like this:

```
O----O-O----O-O-O----O-O-O-O----O-O-O-O-O----O-O-O-O-O-O----
```
so if there happened to be some UniRx expert who knows how to do it, I am all ears.

Other thing to note. An introduced `FloatRange` class popped out. Script spawns an asteroid on a random x coordinate in some range above the screen. Since I already had a `FloatRange` class ready it felt obvious to use it to encapsulate that x coordinate range where the asteroids are spawned. And that is usually the case with granular classes. You never know when it might come in handy, and since you have a small, specialized piece it is more probable that you will reuse it without repeating.

There are also a few additional lines:

{% highlight csharp %}
public ISubject<Unit> asteroidDestroyed;
public ISubject<Unit> playerDestroyed;

...

var destroyOnCollision =
  asteroid.GetComponent<DestroyOnCollision>();

destroyOnCollision.asteroidDestroyed = asteroidDestroyed;
destroyOnCollision.playerDestroyed = playerDestroyed;
{% endhighlight %}

To properly explain them I first need to explain a `DestroyOnCollision` component that is attached to an asteroid prefab.

### DestroyOnCollision

The original code that comes with the tutorial:

{% highlight csharp linenos %}
public class DestroyByContact : MonoBehaviour
{
  public GameObject explosion;
  public GameObject playerExplosion;
  public int scoreValue;
  private GameController gameController;

  void Start ()
  {
    GameObject gameControllerObject =
      GameObject.FindGameObjectWithTag ("GameController");
    if (gameControllerObject != null)
    {
      gameController =
        gameControllerObject.GetComponent <GameController>();
    }
    if (gameController == null)
    {
      Debug.Log ("Cannot find 'GameController' script");
    }
  }

  void OnTriggerEnter (Collider other)
  {
    if (other.tag == "Boundary" || other.tag == "Enemy")
    {
      return;
    }

    if (explosion != null)
    {
      Instantiate(explosion, transform.position, transform.rotation);
    }

    if (other.tag == "Player")
    {
      Instantiate(
        playerExplosion,
        other.transform.position,
        other.transform.rotation);
      gameController.GameOver();
    }
    
    gameController.AddScore(scoreValue);
    Destroy (other.gameObject);
    Destroy (gameObject);
  }
}
{% endhighlight %}

So the original `DestroyByContact` script is attached to every instance of an asteroid object. It dynamically searches for an instance of the original `GameController` script. It needs it to invoke a `GameOver()` or `AddScore(scoreValue)`. This is the part that I particularly don't like. It's the single asteroid object that decides on when the game should be over or when and how many points to give player for destroying it and it's all wrapped in a script whose name suggests nothing of it. This is a part I really wanted to improve. Take a look at my implementation:

{% highlight csharp linenos %}
public class DestroyOnCollision : RxBehaviour {
  public GameObject asteroidExplosion;

  public GameObject playerExplosion;

  public IObserver<Unit> asteroidDestroyed;

  public IObserver<Unit> playerDestroyed;

  void Start () {
    var collisions = this.OnTriggerEnterAsObservable()
      .Where(c => c.gameObject.tag != "boundary");
    
    var sub1 = collisions.Subscribe(c => {
      Destroy(c.gameObject);
      Destroy(gameObject);
      Instantiate(
        asteroidExplosion,
        transform.position,
        transform.rotation);
      asteroidDestroyed.OnNext(Unit.Default);
    });

    var sub2 = collisions
      .Where(c => c.gameObject.tag == "Player")
      .Subscribe(c => {
        Instantiate(
          playerExplosion,
          c.transform.position,
          c.transform.rotation);
        playerDestroyed.OnNext(Unit.Default);
      });
    
    AddSubscriptions(sub1, sub2);
  }
}
{% endhighlight %}

It exposes two observers (part of a stream that something can be published to) `asteroidDestroyed` and `playerDestroyed`. Both will be signal on specific game logic events, and their names are pretty self explanatory. First in a `collisions` observable we are filtering out the collisions with boundary. We don't want a player to score when an asteroid flies out of the map and gets destroy to free up resources. For every "meaningful" collision we want to destroy both objects: asteroid and the object that collided with it (it's either the player or a laser bolt). If it's the player we are also signaling that the player got destroyed.

Now back to our mysterious lines in `HazardWavesController`. Since it instantiates an asteroids it also explicitly wires up it's instances of `asteroidDestroyed` and `playerDestroyed`.  In other words, all instances of `DestroyOnCollision` will share `HazardWavesController`'s observers and `HazardWavesController` will get all signals from all instantiated asteroids. It further exposes these observers as subjects to some other object that will be able to subscribe to them. 

Good parts:
1. `DestroyOnCollision` is solely interested in destroying things and collisions while `HazardWavesController` is responsible for creating and managing spawned asteroids.
2. Again, the script version in UniRx is more expressive to me.

Bad parts:
1. `DestroyOnCollision` script and thus the asteroid prefab needs to get references for `asteroidDestroyed` and `playerDestroyed` from the object that instantiates it. This is implicit contract and without these references it will throw a `NullReferenceException`.
2. There seems to be a lot of code overhead on inter script communication trying to wire up all references manually. It will work for such a simple game like this, but with hundreds or thousands of such events and game objects it might be a mess.

### A real GameController

I was complaining for `GameController` script that it's methods are triggered from outside, and you can't actually know what causes these important events like scoring and when the game ends. What I basically did so far was just to split that code into more focused classes but something still has to signal `newScore` to `ScoreController` and `gameOver` to `GameOverController`.

All of this happens in the brand new `GameController` that looks like this:

{% highlight csharp linenos %}
public class GameController : RxBehaviour {
  public int pointsPerHazard;

  void Start () {
    var hazardWavesController =
      GetComponent<HazardWavesController>();
    var scoreController = GetComponent<ScoreController>();
    var gameOverController = GetComponent<GameOverController>();

    var sub1 = hazardWavesController
      .asteroidDestroyed
      .Subscribe(_ => {
        scoreController.newScore.OnNext(pointsPerHazard);
      });
    
    var sub2 = hazardWavesController
      .playerDestroyed
      .Subscribe(_ => {
        gameOverController.gameOver.OnNext(Unit.Default);
      });

    AddSubscriptions(sub1, sub2);
  }
}
{% endhighlight %}

`GameController` script along with others is attached to a single `GameObject` in the editor like this:

![Boundary class in editor]({{ site.baseurl }}/assets/images/boundary_editor.png)

So there is a bunch of specialized controllers and a master controller that wires up all game events. It assigns a certain amount of points for destroying an asteroid and ends the game when the player got destroyed but without a details of these two operations. In this arrangement we have all main game logic events wired up on a high level in one place while details on how to handle these events are in a more specialized classes.

## Unsubscribing observables

Every time `Subscribe` method is invoked it creates a subscription. If you subscribe to an observable that you don't necessarily own, like `Observable.Interval(...)`, your callback function will get invoked even when a `GameObject` to which your script was attached was destroyed. Unless an observable stream ends (like the one that had fixed amount of elements), you have to manually unsubscribe to stop receiving signals. When you reload a Unity scene all game objects it contained are destroyed. You would expect the behaviour contained in scripts attached to destroyed objects to cease as well. Unfortunately you have to do it manually. `Subscribe` method returns `IDisposable`, invoking `Dispose()` on that returned object is in fact unsubscribing from observable. This is very clever because it allows subscribing and unsubscribing using `using` (hehe) statement. Here however I can't use it because I want to unsubscribe when the instance of a script is destroyed with it's parent game object, not in a `Start` methods where I am creating all the subscriptions.

I created a base class `RxBehaviour` from which all scripts that contain UniRx code subscribe:

{% highlight csharp linenos %}
public class RxBehaviour : MonoBehaviour {
  private List<IDisposable> subscriptions =
    new List<IDisposable>();

  protected void AddSubscriptions(params IDisposable[] items)
  {
    subscriptions.AddRange(items);
  }

  protected virtual void OnDestroy()
  {
    subscriptions.ForEach(s => s.Dispose());
  }
}
{% endhighlight %}

It allows it's subclasses to add any number of subscriptions, and it will dispose them all when the object is destroyed. Examples on how it's used are in all my above scripts:

{% highlight csharp linenos %}
public class Foo : RxBehaviour {
  void Start () {
    
    // Subscribing to an observable
    var sub1 = ...
    
    // Subscribing to an observable
    var sub2 = ...

    // Registering subscriptions
    AddSubscriptions(sub1, sub2);
  }
}
{% endhighlight %}

In a `Start` method or in a constructor where we put most of our UniRx code, we just need to gather all subscriptions and pass them to `AddSubscriptions` method. `RxBehaviour` will take care of unsubscribing when game object is destroyed.

## Inter-component communication

As I mentioned before, there is a problem of sending signals across scripts that may be attached to a different game objects, to which you might not even have a reference in the design time because they are spawned dynamically.

You can either search for an objects dynamically like `GameObject.FindGameObjectWithTag` or `GetComponent<>` or wire in references manually on instantiation like I did, but all of these approaches have flaws to me. `FindGameObjectWithTag` assumes existence of a certain object which needs to be tested. `GetComponent<>` is even worse because it assumes that a given component is in the same parent game object. Doing things manually on the other hand adds a lot of overhead and is error prone.

I am considering trying to use a [MediatR](https://github.com/jbogard/MediatR) library which allows for:

> In-process messaging with no dependencies.

which sounds exactly what is needed here. I will certainly write another blog post to tell how it performs.

## Renaming scripts

In general you should name a class using a noun expression. Classes define objects which should be thought of as "things", so noun seems natural here. However many Unity scripts are components attached to game objects that define a certain behaviors of that object and how it interacts with the environment. In fact, all scripts inherit from `MonoBehaviour` class. My preference is to use verb expressions to name behaviors rather than nouns. The original tutorial scripts names are inconsistent in that matter.

A certain behaviour is usually triggered by certain events. Common naming convention for event handlers is `On(EventName)` and that's how I decided to rename certain scripts.

I also prefer to use an actual event names as they appear in the scripting API, for example `Collision` instead of `Contact`, or `TriggerExit` instead of `Boundary`.

Here are the scripts renamed according to the above rules:

| *Original name* | *New name* |
| `DestroyByBoundary` | `DestroyOnTriggerExit` |
| `DestroyByContact` | `DestroyOnCollision` |
| `DestroyByTime` | `DestroyOnTimeout` |
| `Mover` | `MoveForward` |
| `RandomRotator` | `RotateRandomly`|

## Summary

Out of my short venture with Unity and UniRx I conclude:

1. Unity is good but Unity + UniRx is better:
- makes code more expressive
- makes writing time dependent code easy
2. You can and should apply good practices from other programming fields in game development too:
- each class should have one responsibility
- don't create mental maps when naming things, ex. By "Contact" I mean "Collision event".
- method should do what it name says it does
- avoid repetitions through refactoring (extract method in this case), you can only benefit from having small focused classes
3. UniRx shouldn't be applied everywhere at any cost.
4. More complex games will require more scalable architecture than what I presented.