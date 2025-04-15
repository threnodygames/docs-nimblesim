# 🐝 NimbleSim Deep Dive

**NimbleSim** is a modular, declarative AI system where agents are able to execute simple, plain-language sequences of behavour.

I'm going to demonstrate how with NimbleSim you can simulate complex & reactive behaviour in only a few lines of code; lines which read like a story.

NimbleSim can be used in many ways, but in this specific example, I want to introduce what I think is a cool approach to AI that NimbleSim does well; that is, a simulation where the actors don't have any behaviour on them whatsoever, and are instead driven by the environment.

---

## 🌻 The Scenario

We want to simulate:

- A **field** that grows flowers randomly
- A **hive** that sends out bees to collect nectar
- **Bees** that forage and return
- **Flowers** that handle being harvested

All of this will be built using only 8 **NimbleSim sequences** & ~270 lines of code, with no coroutines, no hand-crafted if / elses, no state machines and barely any state tracking.

You can watch a short video of the finished simulation [here](https://youtu.be/oGIw8FmthE0?si=5QitdVDW8_PPZrNW)

---

## 🌱 Field Behaviour - Random Spawning

Starting with the field, we want to make it *maybe* grow a flower every 3 seconds. For this, we'll need timers, looping and random chance.

Here's how to do that in NimbleSim:

```csharp
spawnFlowers = Nimble.Sim()
    .Maybe(SpawnFlower)
    .Wait(3)
    .RepeatAllForever();
```

The spawn method simply instantiates the flower at a random location & pushes it to a stack of `availableFlowers`, which is used in the next step.

## 🌱 Field Behaviour - Call Forager

If the field has an available flower, then it should call the hive and ask for a forager to come harvest the flower, then repeat this over and over:

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

That completes the field behaviour.

---

## 🏠 Hive Behavior - Send Bee

When the Field calls for a forager, the hive pushes the flower to a stack.

```csharp
public void CallForager(Flower flower)
{
  openFlowers.Push(flower);
}
```

The Hive is also running a looping sequence that repeatedly checks if:

- there is a flower available to be foraged
- there is an available bee to handle it

This is what that sequence looks like, along with the `SendBee` method.

```csharp
Nimble.Sim()
    .WaitUntil(HasOpenFlower)
    .WaitUntil(HasWorkerBee)
    .Then(SendBee)
    .RepeatAllForever();

void SendBee()
{
    Bee worker = availableWorkerBees.Pop();
    Flower freeFlower = openFlowers.Pop();

    beesOutWorking.Add(worker);
    worker.GiveHiveInstruction(Forage(freeFlower));
}
```

### 🏠 Hive Behavior - Forage

The result of the `Forage` method is passed to the bee as an instruction.

This method tells the bee where to go. It also tells the bee how to get home & how to add nectar to the stores. The second `Then` call here can take a callback which receives the actor executing the sequence (i.e. the specific bee object). 

This enables the hive to tell the bee how to remove itself from the worker bees upon return, and then re-add itself to the available workers stack.

All of this is baked into the sequence and passed to the bee, who just executes it.

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

  Sequence Forage(Flower flower)
  {
    return Nimble.Sim()
      .First(new Goto(flower.gameObject.transform.position))
      .Then(HarvestForHive(flower))
      .Then(ComeHome())
    .Done();
  }
```

### 🐣 Hive Growth & Stat Tracking

The hive runs multiple background sequences too:

```csharp
// Grow new bee when there's enough nectar
Nimble.Sim()
    .WaitUntil(() => nectar >= 10)
    .Then(SpawnNewWorker)
    .Then(() => nectar -= 10)
    .RepeatAllForever();

// Update stat UI
Nimble.Sim()
    .WaitUntil(() => stats.text != GetCurrentStats())
    .Then(() => stats.text = GetCurrentStats())
    .RepeatAllForever();
```

## 🌸 Flower Behavior

Flowers return a sequence which describes how they should be harvested.

```csharp
public Sequence Harvest()
{
    return Nimble.Sim()
        .Wait(3)
        .Then(() => Destroy(this.gameObject))
        .Done();
}
```

So a bee (or butterfly, or ant, or ghost??) can just say, "Hey flower, how do I harvest you?" and the flower responds with instructions.

---

## 🐝 Bee Behavior

Finally, the actual bee:

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

That’s **all** the logic it needs. It doesn't know how to harvest a flower, doesn't know what the Field is doing or how much nectar is in the hive — it just follows instructions.

---

## 🧠 Recap: The System in 8 Sequences

### Field
```csharp
// Spawn flowers
Nimble.Sim()
    .Maybe(SpawnFlower)
    .Wait(2)
    .RepeatAllForever();

// Call hive when flower is ready
Nimble.Sim()
    .If(HasAvailableFlower)
        .Then(CallForager)
        .Or(Action.Nothing())
    .RepeatAllForever();
```

### Flower
```csharp
// Provide a harvest sequence
Nimble.Sim()
    .Wait(3)
    .Then(() => Destroy(this.gameObject))
    .Done();
```

### Hive
```csharp
// Handle harvesting
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

// Send bee to flower
Nimble.Sim()
    .First(new Goto(flower.gameObject.transform.position))
    .Then(HarvestForHive(flower))
    .Done();

// Wait for bee + flower
Nimble.Sim()
    .WaitUntil(HasOpenFlower)
    .WaitUntil(HasWorkerBee)
    .Then(SendBee)
    .RepeatAllForever();

// Grow hive
Nimble.Sim()
    .WaitUntil(() => nectar >= 10)
    .Then(SpawnNewWorker)
    .Then(() => nectar -= 10)
    .RepeatAllForever();

// Update UI
Nimble.Sim()
    .WaitUntil(() => stats.text != GetCurrentStats())
    .Then(() => stats.text = GetCurrentStats())
    .RepeatAllForever();
```

---

## 🎉 Conclusion

In around **270 lines of code**, we’ve created a living, breathing AI simulation that:

- grows flowers randomly
- assigns bees to harvest them
- reacts dynamically to world state
- supports reusable, actor-agnostic sequences
