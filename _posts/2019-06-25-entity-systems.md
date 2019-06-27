---
layout: post
title: "Way of Rhea's Entity System"
date: 2019-06-05
categories: monsters-and-sprites way-of-rhea engine
author: mason
excerpt_separator: <!--more-->
twitter: https://twitter.com/masonremaley/status/1143655586740838400
reddit: https://www.reddit.com/r/rust_gamedev/comments/c5lwi5/way_of_rheas_entity_system/
---

I posted [this poll](https://twitter.com/masonremaley/status/1135083247598047232) on Twitter few weeks ago:

<figure>
  <a href="https://twitter.com/masonremaley/status/1135083247598047232"><img src="/assets/monsters-and-sprites/entity-systems/poll.png"/></a>
</figure>

Entity systems won by a long shot, so that's what I'm going to be writing about today.

In particular, I'm going to outline the process that lead me to [Way of Rhea](https://www.anthropicstudios.com/way-of-rhea)'s current entity system. _Way of Rhea_ is being built in a [custom engine](https://www.masonremaley.com/projects/game-engine/) and [scripting language](https://www.masonremaley.com/projects/scripting-language/) written in [Rust](https://www.rust-lang.org/), but the ideas described should still be applicable elsewhere. Hopefully this writeup will be found helpful, or at least interesting. :)

## The Ad Hoc Approach

_Way of Rhea_'s [initial prototype](https://twitter.com/masonremaley/status/988634973245669377) didn't have an explicit entity system&mdash;I wanted to get something playable on the screen ASAP to validate that the game idea was worth spending time on. Each time I wanted to introduce a new entity type, I just made a new struct, and an ad hoc decision on where to store it.

This approach is severely undervalued. Letting yourself be inconsistent during the early stages of a project has two big advantages:
- It lets you prototype just the thing you're actually trying to build.
- It generates a lot of data on what a generic system would _actually need to accomplish_. It's hard to build a good cart before you know anything about the horse. :)

<figure>
  <a href="/assets/monsters-and-sprites/entity-systems/old-editor.jpg"><img src="/assets/monsters-and-sprites/entity-systems/old-editor.jpg"></a>
  <figcaption>Some entities like the player were hard coded into the all-encompassing "world" struct, while others were stored in a tile map and exposed to the editor via the pictured GUI.</figcaption>
</figure>

As most entities in the game were fairly independent of each other, this approach served me well for almost a year. As time wore on, though, I had more and more ideas that couldn't be expressed well in the system I'd built up...

<figure>
  <a href="/assets/monsters-and-sprites/entity-systems/physics-small.jpg"><img src="/assets/monsters-and-sprites/entity-systems/physics-small.jpg"></a>
  <figcaption>Many of the puzzles I wanted to build involved entities being run through some sort of physics simulation, which would have been difficult to add to the existing codebase.</figcaption>
</figure>

It was going to be difficult to add things like physics puzzles to the game if there wasn't a good way to share data and behavior between entities. This problem seemed chronic enough that it was worth solving the general case.

<!--more-->

## Common Approaches

In my mind, most entity systems fall into one of the following categories:
- Decked out versions of the ad hoc approach, well tuned to the needs of a specific game or game type. For lack of a better term I'll refer to this as the static approach.
- Very dynamic and generic approaches that let you mix and match "components" at will. I'll refer to this as the dynamic approach.

### The Static Approach
I've seen the first approach in a lot of engines that were initially designed for a specific game. If you take the time to understand the somewhat arbitrary metaphysics of one of these systems, they're generally pretty easy to work with&mdash;things you're likely to want to do are usually directly baked in and have a nice workflow.

On the other hand, if you have a gameplay idea not anticipated by the system, you may have hard time implementing it without some [funny business](https://www.geek.com/games/a-train-you-ride-in-fallout-3-is-actually-an-npc-wearing-a-train-hat-1628532/){:target="_blank"}.

<figure>
  <a href="http://www.insidemacgames.com/features/view.php?ID=312#"><img src="/assets/monsters-and-sprites/entity-systems/dim3-inside-mac-games.jpg"></a>
  <figcaption>dim3 was the first 3d engine I ever used, described by its creator Brian Barnes as a "game without content". The game had a number of pre-baked object types (lights, scenery items, etc), and the option to introduce new objects via scripts. Screenshot taken from <a href="http://www.insidemacgames.com/features/view.php?ID=312#">Inside Mac Games</a> as I couldn't find a copy of the editor.</figcaption>
</figure>


### The Dynamic Approach
On the other end of the spectrum is the dynamic approach. When people talk about "entity component systems", they're usually talking about something in this category.

This approach has some bold promises: you can mix and match existing pieces like legos, possibly even creating completely new entity types at runtime.

<figure style="width:75%">
  <a href="/assets/monsters-and-sprites/entity-systems/unity.jpg"><img src="/assets/monsters-and-sprites/entity-systems/unity.jpg"></a>
  <figcaption>In Unity's approach, entities are built up of a large number of components that may or may not depend on each other.</figcaption>
</figure>

This sounds great, but in practice, I've seen it have two major problems:
- When designing a component, you either anticipate every possible use, or you don't. If you do you're likely wasting your time on pairings that will never occur, but if you don't, you can't truely mix and match things freely.
- You've essentially built a dynamic type system for your game, and taken on all the problems that come with one. For example, in Unity, one component can fail to get data from another at runtime if the component in question isn't present on the entity or was deleted at runtime&mdash;and there's not always a good way to respond to that failure.

This isn't to say that the dynamic approach is never the way to go&mdash;but in practice it does have costs that I don't often see discussed.

## The Best Of Both Worlds

Neither of these routes felt quite right for _Way of Rhea_. Between the gameplay not being very dynamic, and [the engine](https://www.masonremaley.com/projects/game-engine/)'s support for arbitrary hot swapping, there's not much need for mixing and matching components at runtime. At the same time, the problem of sharing behavior between entities felt like it was worth solving.

To find a better compromise, we'll need to break down what we actually want out of the _Way of Rhea_ entity system.

Want:
- The ability to have data and/or behavior specific to an entity type
- The ability to have data and/or behavior shared by a set of entity types

Would like:
- Static knowledge of the fields that will be present on a given entity

Don't need:
- We don't need to create new entity or component types at runtime

The single bullet under the "don't need" category changes things. **Most of these systems get built specifically to allow dynamic changes.** If you're working on a large team, you don't want the designers to have to call in a programmer every time they want to try something new, but that isn't an issue on a small indie team where the designer is the programmer.

Taking all this into account, I came up with a fairly straightforward&mdash;but still flexible&mdash;approach that satisfies the requirements at hand. I'm going to copy and paste bits and pieces from the _Way of Rhea_ source to explain it. As mentioned previously, the actual game logic is written in [my scripting language](https://www.masonremaley.com/projects/scripting-language/), but the syntax is very similar to [Rust](https://www.rust-lang.org/) which my engine, compiler, and VM are written in.

Here's the current declaration of `Entity`:
```rs
pub enum Entity {
    Gate(@mut Gate),
    Elevator(@mut Elevator),
    Orb(@mut Orb),
    Stand(@mut Stand),
    Teleporter(@mut Teleporter),
    Blocker(@mut Blocker),
    SpriteSpawnPoint(@mut SpawnPoint),
    Sprite(@mut Sprite),
}
```

*Similarly to in Rust the `enum` keyword in my scripting language represents a [sum type](https://en.wikipedia.org/wiki/Sum_type){:target="_blank"}, equivalent to a tagged union in C.*

Each entity must be one of these 8 variants. The variants are baked into the system, but it's trivial to add another and hot swap it in.

Given an arbitrary entity, you can use a match statement&mdash;similar to a switch statement in C&mdash;to determine its type. For example:

```rs
match entity {
    Gate(gate) => ::log::info(f"It's a gate: {gate}"),
    Elevator(elevator) => ::log::info(f"It's an elevator: {elevator}"),
    _ => ::log::info("It's something else!"),
}
```

Once you've matched on an entity and gotten its inner type, you just have an instance of that entity's struct, and can operate on it directly like you would any struct. For example, you could pass the `gate` struct above to this update function:
```rs
pub fn update(self: @mut Gate, entities: @Entities) {
    // Check if we should be open
    let mut should_be_open = false;
    let intersecting = {
      // ... (truncated for brevity) ...
    };

    // Apply any necessary state changes
    if should_be_open {
        if !self.open {
            self.open = true;
            mixer::play_sound("door open clean.ogg", MAX_VOLUME);
        }
    } else {
        if self.open {
            self.open = false;
            mixer::play_sound("door close clean.ogg", MAX_VOLUME);
        }
    }
}
```

Alright, so nothing particularly novel so far. The trick lies in how we share data and behavior between entity types.

_Way of Rhea_ has a lot of color based puzzles in it, and as such most interactive entities in the game have a `color` field. If you know the type of an entity you can read that field directly, but what if you want to get the color of an arbitrary entity?

You can probably guess&mdash;we just define a function that does the match for us!
```rs
pub fn color(self: Entity) -> Option::<ColorKind> {
    match self {
        Gate(gate) => Option::Some::<ColorKind>(gate.color),
        Elevator(elevator) => Option::Some::<ColorKind>(elevator.color),
        Orb(orb) => Option::Some::<ColorKind>(orb.color),
        Sprite(sprite) => Option::Some::<ColorKind>(sprite.drawable.color),
        SpriteSpawnPoint(sprite_spawn_point) => Option::Some::<ColorKind>(sprite_spawn_point.drawable.color),
        Stand(_)
        | Teleporter(_)
        | Blocker(_) => Option::None::<ColorKind>,
    }
}
```

If the entity has a color, this function returns it. If not, it returns `None`. When you don't know the type of an entity and are okay with the possibility of absence, you read components through one of these functions. When you need a component to be present, you just get a direct reference to the concrete type. If you wanted to hold a reference to any arbitrary entity that must have a color, you could go as far as defining a new enum that can only hold those variants.

The physics simulation, once implemented, will use this pattern as well: when it's time to update the physics objects, it will be trivial to loop over all the entities and call something like `physics::update(rigid_body)` on each entity that has a rigid body.

<figure>
  <a href="/assets/monsters-and-sprites/entity-systems/new-editor.gif"><img src="/assets/monsters-and-sprites/entity-systems/new-editor.gif"></a>
  <figcaption>The level editor has been updated to take advantage of the entity system. The component getters are used to draw controls for all component types that exist on at least one of the selected entities.</figcaption>
</figure>

There are a few other subtitles around spatial partitioning and serialization, but this is the general idea behind the system. I haven't lived with it for very long yet, so if I discover anything interesting about it down the line I'll try to write up another post.

Thanks for reading this far!
