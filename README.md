# NimbleSim

**Nimble is currently in early-access.**

**NimbleSim** is a component-based behavior orchestration framework for Unity that makes AI feel more like *storytelling* and less like plumbing.

It lets you define your game agents' behavior using readable, declarative sequences built from atomic actions — just like building Legos. The result is expressive, modular, and infinitely composable behavior code that stays easy to write, easy to understand, and easy to maintain.

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

- 🔍 **Readable** – behavior scripts read like little stories.
- 🧱 **Composable** – simple atomic actions can be glued together to create layered, reactive behaviors.
- 🧠 **Declarative** – you say *what* the behavior should do, not *how* to do it.
- 🌀 **Interruptible & Reactive** – behaviors can respond to changes in the world mid-flow.
- 🧰 **Plain C#** – no custom editor or node graph required.

---

## 🛠️ Core Concepts

NimbleSim is built on two building blocks:

### ✅ Atomic Actions

In the demo above, I used 90% callbacks, and this is a perfectly valid way to use NimbleSim.

```csharp
Nimble.Sim().Do(() => Debug.Log("Hello"));
```

But NimbleSim provides an additional feature with `Actions`.

Actions are atomic behaviour units that can be composed into sequences. They provide optional lifecycle hooks like:

`OnStart` - Runs when the action first becomes active in a sequence
`OnEnd` - Runs when the action completes & just before the sequence moves to the next step
`Update` - Runs every frame
`IsComplete` - Assessed after every update & tells the sequence when to move to the next step

`Get`

Get is the only method that you are required to implement. This is called whenever an action is reloaded (like in a looping sequence).

This method is important because it gives you the ability to decide how you want an action to work on reload. If you return a new instance, then all of the state you stored in the action will be reset. If you return the current reference, then state will be retained.

Here is an example of a simple action that sends an actor to a location:

```csharp
public class Goto : Action
{
  Vector3 destination;
  float speed = 4.0f;

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

Each lifecycle hook receives a reference to the game object that was passed into the sequence, e.g.:

```csharp
beebehaviour.update(this.gameObject)
```

This is what enables sequences to be decoupled from actors right up until execution.

🧱 Sequences

Sequences are chains of actions, conditions, and modifiers:

```csharp
Nimble.Sim()
    .Do(new Goto(store))
    .Until(() => store.HasStock())
    .Then(() => store.Take(1))
    .AndRepeatForever();
```

This reads like a sentence:

go to the store → wait until there's stock → take one → repeat forever

You can define complex behavior flows using just .Do(), .Then(), .Until(), .Repeat(), .If(), .RandomlyDoOneOf(), etc.

🔁 Reuse & Composition

Since behaviors are just C# methods returning Sequence, you can also return this sequence to any actor.

```csharp
public Sequence TakeWhenAvailable()
{
    return Nimble.Sim()
        .Do(new Goto(gameObject))
        .Until(() => HasStock())
        .Then(_ => Take(1))
        .AndThenStop();
}
```

Now the actor can just call store.TakeWhenAvailable() and get a plug-and-play sequence tailored to that object. It’s like writing gameplay as modular scripts.

🧪 Demo Included

- A sample scene is included in Assets/Demo/ with:
- An agent running around randomly and dying
- A camera following system
- Repetition
- Timing

All handled by Nimble. You just write the story.

💡 Ideal Use Cases

- AI for cozy sims & emergent gameplay
- NPC behavior in systemic worlds
- Game jams & prototyping
- Anywhere behavior readability matters

🧰 Getting Started

- Add NimbleSim/ to your project (Assets/NimbleSim/)
- Check out the sample scene in Assets/Demo/Scenes/
- Create your own Sequences using Nimble.Sim() and plug them into your MonoBehaviours

💬 Support & Feedback

Built by Threnody Games.

Pull requests, suggestions, and questions welcome.

threnodygames@gmail.com
