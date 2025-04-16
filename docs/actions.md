# Actions

Actions represent some behaviour or intent. While it is completely valid to build sequences out of simple lambdas, Actions provide significantly more control.

## How to create an action

Actions are classes that inherit from `NimbleSim.Action`. There are two types:

- Basic actions 
- Conditional actions

---

### Basic actions

Basic actions are classes that inherit `NimbleSim.Action`. 

```csharp
class GoTo : Action {}
```

Actions receive a number of methods from the base class:

#### 🔄 OnStart

_Implementing this method is optional._

```csharp
protected override void OnStart(GameObject actor) {}
```

##### 🎯 Purpose

This method is run once when an action becomes active in a sequence, right before the first `Update` call. 

##### 🛠️ Usage

Use this method to do any one-time setup for your action.

##### 🎭Arguments

`actor` is the GameObject passed to the sequence's `Update` method. Read more in the [sequence docs](./sequences.md).

---

#### Update

_Implementing this method is optional._

```csharp
protected override void Update(GameObject actor) {}
```

##### 🎯 Purpose

This method is run once per frame, same as MonoBehaviour updates.

##### 🛠️ Usage

Use this method to do any logic that needs to be syncrhonised with the game loop. 

##### 🎭Arguments

`actor` is the GameObject passed to the sequence's `Update` method. Read more in the [sequence docs](./sequences.md).

#### IsComplete

_Implementing this method is optional._

```csharp
protected override bool IsComplete(GameObject actor) {}
```

##### 🎯 Purpose

This method is run once per frame **before** Update().

It provides a clear termination point for your action and informs the sequence when to move to the next step.

##### 🛠️ Usage

Use this to define the termination condition for your action.

It's possible to create one-off actions by returning `true` from this method & doing some behaviour in `OnStart`, but in that case it may be better to just use a lambda.

⚠️ This method is optional, but leaving it out will default to `false`, meaning your action will run forever. 

The sequence builder provides circuit breaker methods which override this, e.g. `Until`, `RepeatUntil`, and usually this is why you may not implement `IsComplete`, but if you don't have guards like this, your sequence will stall indefinitely.

##### 🎭Arguments

`actor` is the GameObject passed to the sequence's `Update` method. Read more in the [sequence docs](./sequences.md).

#### OnDone

_Implementing this method is optional._

```csharp
protected override void OnDone(GameObject actor) {}
```

##### 🎯 Purpose

This method is run once when an action becomes inactive in a sequence, immediately after `IsComplete` returns true.

##### 🛠️ Usage

Use this method to do any cleanup for your action.

##### 🎭Arguments

`actor` is the GameObject passed to the sequence's `Update` method. Read more in the [sequence docs](./sequences.md).

#### Get

_This method is required._

```csharp
public override Action Get()
```

This is a required method and is used by the sequencer to get a handle on the action before it becomes active.

For example, in a looping sequence, the sequencer will call this method each time it revisits this action. Without this method, the sequencer would be required to use the original reference passed to it, and that means all state would be retained.

Returning a new instance from this method allows you to avoid that.

Unless you have a specific reason for doing otherwise, it's recommended that you return a new instance from this method.

### Conditional actions

Conditional actions are actions that inherit from `NimbleSim.ConditionalAction`

```csharp
class HasStock : ConditionalAction {}
```

These are used only with `If` steps and share all of the same methods as `Action` plus an additional 2:

#### IsTrue

```csharp
  public override bool IsTrue(GameObject actor);
```

This method is used to tell the `If` statement that it should take the success branch, e.g. the `Then` step.

#### IsFalse

```csharp
  public override bool IsFalse(GameObject actor);
```

This method is used to tell the `If` statement that it should take the failure branch, e.g. the `Or` step.

##### Why both?

Conditional actions inherit the `Update` method and can run behaviour. For example, you might have a sequence like this:

```csharp
Nimble.Sim()
  .If(CheckStoreHasMilk())
    .Then(BuyMilk())
    .Or(ThrowATantrum())
  .Done()
```

The `CheckStoreHasMilk()` conditional action may run some behaviour that navigates the full store before making any decision, and until a decision is made, it's impossible to know if the condition is `true` or `false`.

Having both methods available means that NimbleSim can support this kind of conditional.

## Value of Actions - Example

Often a game will have a number of foundational behaviours that should be shared across many different entities.

For example, your game may feature a bee, a butterfly, a human and a dog, all of which should be capable of moving from one location to another. Their movement look different, but utimately they all go from a -> b.

In NimbleSim, an action to do that may look like this:

```csharp
public class Goto : Action
{
  Vector3 destination;
  float speed;

  public Goto(Vector3 destination, float speed)
  {
    this.destination = destination;
    this.speed = speed;
  }

  public override void Update(GameObject actor)
  {
    actor.transform.position = Vector3.MoveTowards(
      actor.transform.position,
      destination,
      speed * Time.deltaTime
    );
  }

  protected override bool IsComplete(GameObject actor)
  {
    return Vector3.Distance(actor.transform.position, destination) < 0.5f;
  }

  public override Action Get()
  {
    return new Goto(destination);
  }
}
```

Each of these actors could receive this action as part of their behaviour sequence, but with different speeds according to their actor type.

You may also decide that you want all of these actors to look at the object they're moving to. You may write an action like this:

```csharp
public class LookAt : Action
{
  Vector3 destination;
  Quaternion targetRotation;
  float rotationSpeed;

  public LookAt(Vector3 destination, float rotationSpeed)
  {
    this.destination = destination;
    this.rotationSpeed = rotationSpeed;
  }

  public override void Update(GameObject actor)
  {
    Vector3 direction = (destination - actor.transform.position).normalized;

    if (direction != Vector3.zero)
    {
      targetRotation = Quaternion.LookRotation(direction);
      actor.transform.rotation = Quaternion.RotateTowards(
        actor.transform.rotation,
        targetRotation,
        rotationSpeed * Time.deltaTime
      );
    }
  }

  protected override bool IsComplete(GameObject actor) => Quaternion.Angle(
    actor.transform.rotation,
    targetRotation
  ) < 0.1f;

  public override Action Get()
  {
    return new LookAt(destination, rotationSpeed);
  }
}
```

With only these two actions, you can the NimbleSim sequence builder to create a composite behaviour, e.g. 'goToAndLookAt', and then return it from a method.

```csharp
Sequence GoToAndLookAt(float destination, float moveSpeed, float rotationSpeed) {
  return Nimble.Sim()
      .DoTheseAtSameTime(
        new Goto(destination, moveSpeed),
        new LookAt(destination, rotationSpeed)
    )
    .Done();
}
```

This sequence can be used by any of the actors in your game who should exhibit this kind of behaviour. All they would do is retrieve and store the sequence in their Start method, and then execute it during update.

```csharp
goToAndLookAt.Update(gameObject)
```

Now let’s say you want to play a walking animation while the actor moves. Each actor's walking animation may differ, but with NimbleSim, these can be composed easily. You can create a third action

```csharp
public class Animate : Action
{
  AnimationClip clip;

  public Animate(AnimationClip clip)
  {
    this.clip = clip;
  }

  protected override void OnStart(GameObject actor)
  {
    actor.GetComponent<Animation>().Play(clip.name);
  }

  protected override void OnDone(GameObject actor)
  {
    actor.GetComponent<Animation>().Stop(clip.name);
  }

  public override Action Get()
  {
    return new Animate(clip);
  }
}
```

And then your actors simply do:

```csharp
var humanWalk = Nimble.Sim()
    .While(goToAndLookAt)
      .Do(new Animate(humanWalkClip))
  .Done();

var dogWalk = Nimble.Sim()
  .While(goToAndLookAt)
    .Do(new Animate(dogWalkClip))
  .Done();
```

You could also wrap this composition in a closure to make it more reusable:

```csharp
Func<Vector3, Sequence> GotoWithAnimation(float moveSpeed, float turnSpeed, AnimationClip clip)
{
  return (Vector3 destination) => {
    var moveTo = Nimble.Sim()
      .DoTheseAtSameTime(
        new Goto(destination, moveSpeed),
        new LookAt(destination, turnSpeed)
      )
      .Done();

    return Nimble.Sim()
      .While(moveTo)
        .Do(new Animate(clip))
      .Done();
  };
}
```

Now you can define different movement styles like this:

```csharp
var run = GotoWithAnimation(runSpeed, runTurnSpeed, runAnimation);
var walk = GotoWithAnimation(walkSpeed, walkTurnSpeed, walkAnimation);
```

Then compose a higher-level behavior for your actor:

```csharp
Sequence goTo =
  Nimble.Sim()
    .If(IsRunning)
      .Then(run(destination))
      .Or(walk(destination))
    .Done();
```

With only 3 actions and a few sequences, you can support the movement of pretty much any actor in your game.