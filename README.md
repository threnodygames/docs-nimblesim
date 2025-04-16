# NimbleSim

**Nimble is currently in early-access.**

**NimbleSim** is a component-based behaviour orchestration framework for Unity that makes AI feel more like *storytelling* and less like plumbing.

It lets you define your game agents' behaviour using readable, declarative sequences built from atomic actions — just like building Legos. The result is expressive, modular, and infinitely composable behaviour code that stays easy to write, easy to understand, and easy to maintain.

---

## 🎯 Why NimbleSim?

Ever find yourself writing code like this?

```csharp
// Bee foraging behaviour
 if (IsNighttime())
    {
      if (IsAtHive())
      {
        Sleep();
      }
      else
      {
        ReturnToHive();
      }

      return;
    }

    if (AlreadyHasNectar())
    {
      if (IsAtHive())
      {
        StoreNectar(flower)
      }
      else
      {
        FlyToHive();
      }

      return;
    }

    if (!IsNearFlower())
    {
      flower = FlyToNextFlower();
      return;
    }

    if (flower.IsNotOccupied() && flower.HasNectar())
    {
      flower = FlyToNextFlower();
      return;
    }

    Harvest(flower);
```

You try to make it readable by writing meaningful method names, implementing early returns so that you avoid deep if / else nesting and unnecessary checks.

But this is 35 lines of code, not counting the whitespace. And despite your best efforts, it still has some overhead for understanding, it probably takes some work to add additional behaviours, and extracting parts of this behaviour to share with other actors either involves external methods that will need to be created and attached to this object, or some copy / pasting.

*This is the same code in NimbleSim.*

```csharp
 Sequence tryHarvestFlower = Nimble.Sim()
    .If(flower.IsNotOccupied, flower.HasNectar)
        .Then(_ => Harvest(flower))
        .Or(Action.Nothing())
    .Done();

Sequence forage = Nimble.Sim()
    .If(IsNearFlower)
        .Then(tryHarvestFlower)
        .Or(_ => flower = FlyToNextFlower())
    .Done();

beeBehaviour = Nimble.Sim()
    .If(AlreadyHasNectar)
        .Then(ReturnToHive)
        .Or(forage)
    .RepeatUntil(IsNightTime)
    .Then(ReturnToHive)
    .Then(Sleep)
.Done();
```

18 lines that read just like a story,

Check out the [deep dive example](./DeepDiveExample.md) to see how NimbleSim can orchestrate more complex behaviours.

NimbleSim is:

- 🔍 **Readable** – behaviour scripts read like little stories.
- 🧱 **Composable** – simple atomic actions can be glued together to create layered, reactive behaviours.
- 🧠 **Declarative** – you say *what* the behaviour should do, not *how* to do it.
- 🌀 **Interruptible & Reactive** – behaviours can respond to changes in the world mid-flow.
- 🧰 **Plain C#** – no custom editor or node graph required.

---

## 🛠️ Core Concepts

NimbleSim is built on two building blocks:

### ✅ Actions

Actions are classes which represent some behaviour or intent. While it is valid to build sequences using lambdas, actions provide a significantly higher level of control.

For example, a Goto action may look like this:

```csharp
public class Goto : Action
{
  Vector3 destination;
  float speed = 6.0f;

  public Goto(Vector3 destination)
  {
    this.destination = destination;
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

This action encapsulates its behaviour (move towards something) and its termination condition (within 0.5f units) into a contained block. And because actions don't need to know about the actor who will be executing it until after instantiation, it only needs to be written once and then can be passed to any sequence at all.

Actions can also implement methods which provide precise control over their execution. For example:

- IsComplete() - defines a clear termination condtion for the action
- OnStart() - executed when an action becomes active in a sequence, immediately before the first update call
- OnDone() - executed immediately after the action's final update call, right before it becomes inactive in a sequence

But these are optional.

[Read more about actions](./actions.md)


### 🧱 Sequences

Sequences are a series of steps which are run in some order.

```csharp
Nimble.Sim()
    .First(GoToStore())
    .If(StoreHasMilk())
      .Then(BuyMilk())
      .Or(KnockOverTheShelf())
    .Done()
```

Sequences support composing:

- lambas
- actions
- other sequences

This enables you to create complex & recursive behaviours from small, dedidcated & reusable units.

<!-- [Read more about sequences](./sequences.md) -->

## Getting Start & Support

💡 Ideal Use Cases

- AI for cozy sims & emergent gameplay
- NPC behaviour in systemic worlds
- Game jams & prototyping
- Anywhere behaviour readability matters

🧰 Getting Started

- Add NimbleSim/ to your project (Assets/NimbleSim/)
- Check out the sample scene in Assets/Demo/Scenes/
<!-- - Check out the [tutorial](./tutorial.md) -->
- Create your own Sequences using Nimble.Sim() and plug them into your MonoBehaviours

💬 Support & Feedback

Built by Threnody Games.

Pull requests, suggestions, and questions welcome.

threnodygames@gmail.com
