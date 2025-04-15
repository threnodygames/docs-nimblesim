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

This is 18 lines of code and they read just like a story. When you look at the Bee class, it tells you exactly what it intends to do in simple language.

Additionally, Nimble sequences don't actually belong to anything until you use them, e.g.:

```csharp
beeBehaviour.Update(gameObject);
```

This means that the entire sequence and any of its sub-sequences can be passed around to whatever wants to use it.

A butterfly may also want to forage for and harvest a flower, but doesn't care about returning to the hive or the time of day. With NimbleSim, all you have to do to achieve this is return the sequence from a function.

This means you can return behaviour sequences from the object that the sequence relates to.

This method lives on the 'Flower' class, and now the agent doesn't have to know anything about interacting with a flower, it just asks it what to do when it reaches it.

```csharp
Sequence Forage(Forager forager) {
  Sequence tryHarvestFlower = Nimble.Sim()
    .If(IsNotOccupied, HasNectar)
        .Then(_ => forager.Harvest(this))
        .Or(Action.Nothing())
    .Done();

Sequence forage = Nimble.Sim()
    .If(IsNearFlower)
        .Then(tryHarvestFlower)
        .Or(_ => forager.FlyToNextFlower())
    .Done();
}
```

The bee sequence then becomes:

```csharp
beeBehaviour = Nimble.Sim()
    .If(AlreadyHasNectar)
        .Then(ReturnToHive)
        .Or(currentFlower.Forage(this))
    .RepeatUntil(IsNightTime)
    .Then(ReturnToHive)
    .Then(Sleep)
.Done();
```

And the butterly:

```csharp
butterflyBehaviour = Nimble.Sim()
    .First(this.FlyToNextFlower())
    .Then(currentFlower.Forage(this))
    .Then(FloatAroundAndLookCute)
    .RepeatForever()
.Done()
```

This approach enables you to create agents that don't have to know what they're going to do until they reach the object and ask it.

And then every object in your world can contain simple-language scripts that tell you exactly what it does.

However, NimbleSim allows you to take this even further.

Let's say we have a 'Field' actor running a sequence that randomly decides to spawn a flower every 3 seconds. Here's that sequence:

```csharp
spawnFlowers = Nimble.Sim()
    .Maybe(SpawnFlower)
    .Wait(3)
.RepeatAllForever();
```

For context, the field is storing a reference to each of these flowers when they are created:

```csharp
void SpawnFlower()
{
    GameObject spawned = Spawn.RandomlyOnSurface(flower, this.gameObject);

    availableFlowers.Push(spawned);
}
```

Now lets say we create a parallel sequence inside Field that
reports to a beehive whenever a flower is available.

```csharp
 void CallForager()
{
    Flower flower = availableFlowers.Pop().GetComponent<Flower>();

    hive.CallForager(flower);
}

callForager = Nimble.Sim()
    .If(HasAvailableFlower)
        .Then(CallForager)
        .Or(Action.Nothing())
    .RepeatAllForever();
```

When the hive receives this instruction, it's loads the flower into a stack and runs a sequence that waits until a worker bee is available, at which point it feeds the sequence to the Bee.

```csharp
 void SendBee()
  {
    Bee worker = availableWorkerBees.Pop();
    Flower freeFlower = openFlowers.Pop();

    beesOutWorking.Add(worker);

    worker.GiveHiveInstruction(Forage(freeFlower));
  }

  return Nimble.Sim()
      .WaitUntil(HasOpenFlower)
      .WaitUntil(HasWorkerBee)
      .Then(SendBee)
      .RepeatAllForever();

```

The Forage method lives on the Hive, but it is returning a sequence that gets passed down to the Bee, who then executes it.

```csharp
 Sequence Forage(Flower flower)
  {
    return Nimble.Sim()
      .First(new Goto(flower.gameObject.transform.position))
      .Then(HarvestForHive(flower))
      .Then(ComeHome())
    .Done();
  }
```

And here's 'HarvestForHive':

```csharp
Sequence HarvestForHive(Flower flower)
  {
    return Nimble.Sim()
      .First(flower.Harvest())
      .Then(ComeHome())
      .Then(actor =>
      {
        Bee bee = actor.GetComponent<Bee>();

        beesOutWorking.Remove(bee);
        availableWorkerBees.Push(bee);

        nectar += 5;
      })
    .Done();
  }
```

In addition to the behaviour above, the hive is running parallel sequences:

```csharp
Sequence HiveGrowth()
  {
    return Nimble.Sim()
      .WaitUntil(() => nectar >= 10)
      .Then(SpawnNewWorker)
      .Then(() => nectar -= 10)
    .RepeatAllForever();
  }
```
It's even running a sequence to update some stat text on the screen whenever it needs to.

```csharp
  Sequence UpdateStats()
  {
    return Nimble.Sim()
      .WaitUntil(() => stats.text != GetCurrentStats())
      .Then(() => stats.text = GetCurrentStats())
    .RepeatAllForever();
  }
```

The Bee class looks like this.

```csharp
public class Bee : MonoBehaviour
{
    Sequence activeInstruction;

    public void GiveHiveInstruction(Sequence instruction)
    {
        activeInstruction = instruction;
    }

    void Update()
    {
        activeInstruction?.Update(gameObject);
    }
}
```

9 lines of code for an agent that is able to react to instructions, gather resources, wait, and return. It doesn't even have any code related to harvesting the flower, because that lives on the flower itself:

```csharp
 public Sequence Harvest()
  {
    return Nimble.Sim()
      .Wait(3)
      .Then(() => Destroy(this.gameObject))
    .Done();
  }
```

And this sequence is 100% reusable, it doesn't care whether a Bee is executing it or a Butterfly, and neither does the flower. The agent asks "How do I harvest you?" and the flower responds.

## Recap

The system just described looks like this:

- A field that:
    - has a random chance to spawn a flower every 2 seconds
    - communicates with a nearby Hive when a flower is ready to be harvested
- A hive that:
    - instantly reacts to comms from the Field and instructs bees to go harvest
    - receives nectar from worker bees and grows its colony accordingly
    - writes its statistics out to the screen
- Flowers that:
    - communicates with bees and tells them to wait before completing the harvest
    - manages its own death when harvesting is complete, without ever knowing who or when it will be harvested
- Bees that:
    - wait for instructions from the hive
    - react to comms from the hive when flowers are ready and travel to them
    - harvest the flowers
    - carry the harvest back to the hive and then resume waiting for instructions

All of this in around 270 lines of code (including using statements, whitespace & curly braces) and only 8 NimbleSim sequences.

```csharp
// Field - spawn flowers
 Nimble.Sim()
    .Maybe(SpawnFlower)
    .Wait(2)
.RepeatAllForever();

// Field - call forager
Nimble.Sim()
    .If(HasAvailableFlower)
        .Then(CallForager)
        .Or(Action.Nothing())
    .RepeatAllForever();

// Flower - Harvest me
Nimble.Sim()
    .Wait(3)
    .Then(() => Destroy(this.gameObject))
.Done();

// Hive - Ask flower for harvest instructions, store nectar and come home
Nimble.Sim()
    .First(flower.Harvest())
    .Then(ComeHome())
    .Then(actor =>
        {
        Bee bee = actor.GetComponent<Bee>();

        beesOutWorking.Remove(bee);
        availableWorkerBees.Push(bee);

        nectar += 5;
        })
.Done();

//  Hive - Go to flower & then harvest
Nimble.Sim()
    .First(new Goto(flower.gameObject.transform.position))
    .Then(HarvestForHive(flower))
.Done();

//  Hive - Send bee only when flower & bee are available
Nimble.Sim()
    .WaitUntil(HasOpenFlower)
    .WaitUntil(HasWorkerBee)
    .Then(SendBee)
    .RepeatAllForever();

//  Hive - Spawn new worker when nectar is available
Nimble.Sim()
    .WaitUntil(() => nectar >= 10)
    .Then(SpawnNewWorker)
    .Then(() => nectar -= 10)
.RepeatAllForever();

//  Hive - Update stat text
Nimble.Sim()
    .WaitUntil(() => stats.text != GetCurrentStats())
    .Then(() => stats.text = GetCurrentStats())
.RepeatAllForever();
```

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
