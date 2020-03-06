---
layout: post
title: "Hot Swapping in Way of Rhea (Part 1: Overview)"
date: 2019-06-05
categories: monsters-and-sprites way-of-rhea engine
author: mason
excerpt_separator: <!--more-->
---
<!-- TODO: Move up -->
<!-- TODO: twitter: https://twitter.com/masonremaley/status/1143655586740838400 -->
<!-- TODO: reddit: https://www.reddit.com/r/rust_gamedev/comments/c5lwi5/way_of_rheas_entity_system/ -->

<!-- TODO:
NOTE: Needs screenshots or pictures or diagrams or code or something cool? Demo video??
NOTE: Talk more about how the scripts are used, or hint at it being in another post later?
-->

The engine [Way of Rhea](/way-of-rhea) is built on top of supports hot swapping of all assets, including scripts. That is to say, if you edit a script on disk, 80-100ms later you'll find yourself in the same spot in the game but with the new code running.

[VIDEO DEMO]
<!-- TODO: video or gif demo here? can embed if we want -->

I took what I consider to be a very pragmatic approach to hot swapping. In this write up, I'll discuss the process that lead me to this approach, and some implementation details. Part 2 of this post, when written, will dive into details of the serialization system powering this approach.

*As always, I'm interested in sharing my thought process, not attempting to talk others into adopting my approach--every project has a different set of tradeoffs so YMMV.*

<!--more-->

## Goals

My primary goal in implementing hot swapping was to reduce iteration time. It turns out, introducing hot swapping had a lot of unanticipated benefits as well, such as reducing the need to build as many GUIs.

That being said, seeing as my original goal was to reduce itreation time, it's worth pointing out that there are other (non mutually-exclusive) ways I could have achieved this. I've outlined a few below from least to most involved, ending with hot swapping.

### Cheat Codes

[Cheat codes](https://en.wikipedia.org/wiki/Cheat_code#Cheat_codes){:target="_blank"} are a common way to cut down on iteration time.

For example, in an action game, you might add a secret key combo that turns off damage, lets you run extra fast, and turns off player collision detection. If you're tweaking a final boss, you can use this cheat code to jump over to where the boss is to test your changes without having to play through the entire level every time you make a change.

Cheat codes are great, but it's worth noting that they only help once you're back in the game. You still have to wait for the game to recompile, and the level to load.

### Tweak Files

One step above cheat codes are tweak files. A tweak file is just a text file on disk with some predefined variables in it that the engine can refresh at runtime. It's essentially Hot Swapping Lite&trade;.

For example, in the action game mentiond previously, you might have a `boss_run_speed` variable in your tweak file. You can then edit this variable while the game is running without having to restart at all!

*(If you'd like, you can use your platform's file watch API to pick up changes to the tweak file and automatically reload it. Alternatively you can just refresh when a special hot key is pressed. I don't recommend trying to use timestamps or such for this, that tends to be flakey unless you're very careful, I'll discuss this more in a later section but IMO a flaky solution to this problem is worse than not solving it at all.)*

Overall tweak files are super powerful. Unlike cheat codes, they let you change the game without restarting it, though you do have to decide up front which variables to expose.

### GUIs

One step above a tweak file is an in game GUI.

A basic version of this might just expose the tweak file variables to the engine's GUI system, maybe giving you things like color pickers when it sees a color or such. A more involved version of this might be something like an in-game level editor.

An in game GUI can be incredibly powerful, but it can also be a lot of work to build, and you do have to anticipate the types of changes you'll want to use it to make.

### Hot Swapping

Finally, there's hot swapping. I'm labeling hot swapping as most involved as it's going to have deeper effects on the design of the engine, but it's not necessarily more work.

Hot swapping lets you edit code on disk, and then see the changes in game shortly afterwards without restarting the whole game. It's not mutually exclusive with the above approaches, but it does aid them in various ways:
* You can add/adjust cheat codes while playing!
* Every file is automatically a tweak file!
* It makes building GUIs much easier--if you're taking an [immediate mode](https://caseymuratori.com/blog_0001){:target="_blank"} approach to GUIs you can just edit numbers in code until things look right, no need to build a WYSIWYG editor or such.
  * It also cuts down on the number of GUIs that you need to create.

This is super powerful, but it's worth noting, it's only directly useful to programmers! I'm the sole programmer and game designer on [Way of Rhea](/way-of-rhea), so this works great for me. If I had game designers that didn't program much on my team I might consider investing more heavily in GUIs.

## Hot Swapping Approaches

Alright, say you've looked at the above options, maybe considered some others I haven't thought of, and decied that hot swapping is right for your project. How do you actually implement it?

I considered a few different approaches...

### "Edit and Continue"

One approach is to try to piggy back off of something like Visual Studio's "Edit and Continue".

Systems like this have never worked reliably for me, and it's important to me that my hot swapping is reliable. The last thing I want is to make a last minute change before an expo thinking I've improved things, and then find out upon launching the game at the booth that edit and continue got me into some kind of weird state where the game worked at the time, but now can't run from a full reboot.

<!-- TODO: mention problem of field names? -->

### Dynamic Libraries

A common approach to hot swapping is to break the engine up into DLLs that can be reloaded at runtime. For example, if the renderer is its own DLL, you can recompile and reload the renderer while keeping the rest of the game running. You'll need to have some way of transferring any state the renderer built up to the new DLL.

This is super neat! I was more interested in swapping out "game code" than "engine code", and was already planning on writing the game code in a scripting language, so I didn't end up doing things this way.

*(I did briefly play around with this approach, and found that at the time Rust could not reliably reload the same DLL repeatedly on macOS and Windows without crashing. I'm told that this has been fixed on macOS, I'm not sure about Windows, I haven't tried again since my initial test.)*

### Language Support

The first approach that I invested serious time was baking in support for hot swapping into the scripting language.

Way of Rhea is built in custom scripting language, so this seemed like the obvious solution. I built in the ability to add new code to the running program at any time. If nobody is referencing the old code, it gets deleted.

This was super cool (and is still a little nifty as it allows for an in-game REPL), but didn't end up being all that useful for hot swapping. It was too hard to reason about--you end up with a bunch of old and new code running at the same time.

### Serialize, Recompile, Deserialize

The solution I ended up going with is very straightforward, it's essentially the DLL approach but with a scripting language, and all gameplay code is considered part of the same library.

The process is as follows. Wait for a file to change, and then:
* Serialize the game state
* Recompile all scripts
* Deserialize the game state

It's worth breaking this down a bit.

#### Serialization

We need to serialize the game state before we do the swap, otherwise, we'll just end up back at the beginning of the game!

I achieved this by providing a callback for the user to implement that specifies how to serialize the game. In my initial proof of concept I just implemented the serialization manually, nowadays my language has built in support for serialization so the callback implementation is very short. I'll write more about this in part 2.

#### Recompile All Scripts

I considered doing some sort of incremental compilation, but my compiler is currently fast enough that it'd be hard to justify adding that kind of complexity, so right now I just recompile everything.

#### Deserialize The Game State

Lastly, we need to deserialize the serialized state into the recompiled program. When doing so we'll hold onto existing assets (textures, sounds, etc) so as to not waste time reloading them, and then only after we've finished the hot swap drop any assets that are no longer needed.

Just like with serialization, I provide a callback for the user to implement how deserialization should be handled. In my proof of concept I implemented it manually, now I take advantage of the serialization system I've added to my language.

There's one problem though: the user could have made any arbitrary change to the code; the serialized data structures may not match up with the data structures in the new code at all!

I'll dive into the details of my solution for this in part 2, but the short version is:
- There's a simple set of (user-visible) rules that the deserializer follows to determine whether the data type it's deserializing into is backwards compatible with the serialized data
- Not all data is serialized, so not all data needs to be backwards compatible (e.g. I don't serialize the state of something like a particle system)
- If serialization fails, I simply cancel the hot swap and inform the user

After living with this system, I realized that some engines I've worked with in the past have implemented hot swapping very similarly, but ended up with a poor user experience because they got this last stage wrong. It's not worth trying to do magic here IMO, if it's unclear how to deserialize the data, you're far better off just letting the user know!

## In Practice

### Other Assets

I hot swap all other assets in the game as well. Most are pretty straightforward--you just free the existing asset, then load the new one and give it the old handle. For maps, however, it's a little trickier. Unlike sounds or textures which have no state in Way of Rhea, maps contain dynamic entities that the player can pick up and move. I've gone through a few different approaches to this since starting work on Way of Rhea, I'm not 100% sold on my current approach but may discuss it more in the future.

### When to hot swap?

Technically, I leave it up to the user--the engine provides an API you can call to ask for a hot swap. In practice the engine also exposes a file watch API, and I always just hot swap whenever a relevant file changes.

I'm currently using [notify](https://crates.io/crates/notify) to watch the filesystem. It's a nifty layer on top of the file watch APIs for the platforms I care about supporting hot swapping on. If you're not working in Rust you can look for a similar library in your language, or just talk directly to the OS APIs on your target platforms.

### Results

Having hot swapping has been great. I've found this approach to be robust, and it's handled everything I need it to. The `scripts` folder in Way of Rhea currently contains ~15k lines of code, including all the gameplay code, an in game level editor, a GUI library, math libraries, an entity system, debugging tools, misc container types, an asset pipeline, etc. I'm able to comfortably edit all of these subsystems while the game continues running. I don't think I'd want to work on another game without *some* solution to this problem in place, and I don't currently have any reason to move away from my current one.

<!-- TODO: Add mailing list link, mentionf ollow on twitter? -->
If you appreciated this post and want to support me, feel free [wishlist Way of Rhea on Steam!](https://store.steampowered.com/app/1110620/Way_of_Rhea/) If you want to hear when part two of this post comes out, consider subscribing to our mailing list.
