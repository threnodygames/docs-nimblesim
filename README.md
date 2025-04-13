# NimbleSim

**Nimble is currently in early-access.**

**NimbleSim** is a component-based behavior orchestration framework for Unity that makes AI feel more like *storytelling* and less like plumbing.

It lets you define your game agents' behavior using readable, declarative sequences built from atomic actions — just like building Legos. The result is expressive, modular, and infinitely composable behavior code that stays easy to write, easy to understand, and easy to maintain.

---

## 🎯 Why NimbleSim?

Most AI frameworks fall into one of two camps:
- overly abstract black boxes (like behavior trees or state machines with visual editors)
- pure imperative spaghetti (custom logic per agent with booleans flying everywhere)

**NimbleSim breaks the mold.**

It’s:
- 🔍 **Readable** – behavior scripts read like little stories.
- 🧱 **Composable** – simple atomic actions can be glued together to create layered, reactive behaviors.
- 🧠 **Declarative** – you say *what* the behavior should do, not *how* to do it.
- 🌀 **Interruptible & Reactive** – behaviors can respond to changes in the world mid-flow.
- 🧰 **Plain C#** – no custom editor or node graph required.

---

## 🛠️ Core Concepts

NimbleSim is built on two building blocks:

### ✅ Atomic Actions
These are the smallest behavior units. You can pass in functions, or use custom `Action` classes.

```csharp
Nimble.Sim().Do(() => Debug.Log("Hello"));
```

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

✨ Comparison Example
Let's say you want an agent to move between 4 points randomly, and lose 1 health each time, until it dies.

💀 In plain Unity:
```csharp
{
    if (!isDead)
    {
        if (ReachedTarget())
        {
            health -= 1;
            SetRandomTarget();
        }

        if (health <= 0)
        {
            Die();
        }
        else
        {
            MoveTowardsTarget();
        }
    }
}
```

Messy, stateful, error-prone. 🤮

😌 In NimbleSim:
```csharp
Sequence move(Transform target)
{
    return Nimble.Sim()
        .Do(new MoveTo(target))
        .ThenWaitForThisManySeconds(2)
        .Then(_ => health -= 1)
        .AndThenStop();
}

Sequence wander = Nimble.Sim()
    .RandomlyDoOneOf(
        move(pointA),
        move(pointB),
        move(pointC),
        move(pointD)
    )
    .AndRepeatForever();

Sequence fullBehavior = Nimble.Sim()
    .Do(wander)
    .Until(() => health <= 0)
    .Then(_ => Destroy(this.gameObject))
    .AndThenStop();
```

That’s it. No Update() logic. No flags. No chaos. Just clean behavior flow.

🔁 Reuse & Composition
Since behaviors are just C# methods returning Sequence, you can define reusable logic like this:

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

Now any agent can just call store.TakeWhenAvailable() and get a plug-and-play sequence tailored to that object. It’s like writing gameplay as modular scripts.

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
