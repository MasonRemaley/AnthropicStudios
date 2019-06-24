---
layout: post
title: "Scripting Language Cleanup"
date: 2018-06-30
categories: monsters-and-sprites way-of-rhea language engine
author: mason
excerpt_separator: <!--more-->
twitter: https://twitter.com/AnthropicSt/status/1013202971998539776
---

When we built the original [Monsters and Sprites](/monsters-and-sprites) demo, we only had 9 days to get it working before the [Playcrafting](https://playcrafting.com/){:target="_blank"} expo we had signed up for, so we had to cut a lot of corners. Since then I've been doing bug fixes and working on a lot of miscellaneous [engine](https://masonremaley.com/projects/game-engine/)/[language](https://masonremaley.com/projects/scripting-language/) features that I either couldn't get done in time for the demo, or didn't realize were important until I started building it.

We've made a few game updates since then ([we now have sound!](https://twitter.com/AnthropicSt/status/1010568311690743808)), but this post is specifically going to explore some language updates I've made.

I made a bunch of smaller changes that I'm going to just gloss over here:

- Adding new compiler warnings
- Fixing a parser error in compound assignments for structs (e.g. `x.y += 2`)
- Fixing bugs in the type checker
- Supporting a similar struct initializer shorthand to what Rust allows (e.g. `@Vec2 { x: x, y: y }` can now be written as `@Vec2 { x, y }`)

And some slightly bigger ones I'll write about more in depth.

<!--more-->

<br>

---

<br>

# Numeric Types

My initial pass at numeric types for the language involved using big integers for all integer types, and ratios of big integers for decimals. Under the hood these were just types exported by [rust-num](https://github.com/rust-num/num){:target="_blank"}. In practice, this resulted in a lot more trouble than it was worth:

- The engine almost always had to convert them back to normal number types before passing them to (for example) OpenGL anyway
- Ratios don't work well for physics as the numerators/denominators get very large making it unclear how to ideally convert them back to floats when you inevitably need to
- The semantics are arguably more confusing and subtle: I can set up fixed size integers to crash when they get too large, big ints just get slower.

So this change was pretty straightforward, I actually made parts of it during the week I was working on the demo: `int` and `ratio` are now replaced with `i8`, `i16`, `i32`, `i64`, `isize`, `u8`, `u16`, `u32`, `u64`, `usize`, `f32`, and `f64`. If I ever feel like I really want the other types I can add them as additional options.

The biggest open question here is how to decide what type a literal is. Previously it was easy: If your number had a decimal, it was a `ratio`, otherwise it was an `int`. Now it's not so clear. To start with I've just set `i32`/`f32` as the defaults and required a postfix (e.g. `10u8`) for all other types, but I'm considering some kind of type inference here to make this less verbose.

<br>

---

<br>

# Format Strings

Up until this point, the language had no good way to handle `println` because I wanted to wait until I came up with a decent solution to make a decision on this, so I was stuck doing a lot of this:

```rust
println("High Score: " + (score * 100) as string + "!");
```

This is inefficient since currently there's no optimization preventing it from reallocating after every addition, but more importantly, it's difficult to read and write. Many languages have some sort of special function type that takes a variable number of parameters to solve this:

```c
printf("High Score: %i\n", score * 100);
```

I'm hesitant to add support for that to this language so early though as the only thing I see myself using it for right now is string formatting. If I'm going to add something like this I want more use cases to evaluate it against. At the same time I found myself writing a lot of print statements when building the Monsters and Sprites demo. So instead of creating a whole new function type, I decided just to add a format string syntax:

```rust
println(f"High Score: {score * 100}!");
```

Breaking this down, `println` is just a normal function of type `fn(string)` like before, but is being called here with a "format string"--that's where the magic is. Whenever you see a string prefixed by `f"`, the contents are interpolated. So for example:

```
$ let name = "world"
$ f"Hello, {name!}"
"Hello, world!"
```

This syntax was (roughly) taken from Python3.

The type checker/codegen implementation was pretty easy here, I just check that every format arg can be casted to a string, and then in codegen do the cast if necessary and emit a `Concat` bytecode instruction which I've implemented to allocate a string with the capacity equal to that of the sum of the lengths of each piece of the string, and then slot in the pieces. Surprisingly the [lexing](https://en.wikipedia.org/wiki/Lexical_analysis){:target="_blank"}/[parsing](https://en.wikipedia.org/wiki/Parsing){:target="_blank"} for once turned out to be the hard part.

For those unfamiliar, compilers normally break down source code into "tokens" like `Plus`, `Minus`, or `Identifier(score)` in a phase called lexing before the parser converts the tokens into an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree), usually using [recursive descent](https://en.wikipedia.org/wiki/Recursive_descent_parser){:target="_blank"}. Normally lexing and parsing are completely separably.

The problem I ran into was that format strings can contain arbitrary expressions inside of them that should be evaluated as normal code if they're between braces, which means the lexer needs to know when to stop parsing strings and start parsing expressions, and also when the expression is finished. However, the lexer by definition doesn't know when the expression is finished--only the parser does.

To solve this problem, I decided to change the pattern slightly--the parser now handles strings instead of the lexer. When the parser hits a quote, it switches the lexer into a mode where it just reads the characters directly without converting them into tokens. When parsing format strings, the parser just switches modes whenever it enters/exists a format arg. This sounds a little odd at first when you're used to the normal pattern, but it ends up being really elegant.

<br>

---

<br>

# Imports

The language doesn't yet support methods. I'm not really a fan of the way classes/methods are implemented in most languages, so I'm holding off on making any decisions here until I have a better idea of how I want things to work. It turns out that so far in practice...not having methods has been fine. I've just put structs that I'd like to treat as classes in their own files and put free functions with parameters named
`self` in the same namespace:

```rust
// player.welp

import math;

pub struct Ball {
    position: @math::Vec2,
    velocity: @math::Vec2,
    size: @math::Vec2,
}

pub fn new(position: @math::Vec2) -> @mut Ball {
    @mut Ball {
        position,
        velocity: @mut Math::Vec2 { x: 0.0, y: 0.0 },
        size: @mut math::Vec2 { x: 1.0, y: 1.0 },
    }
}

pub fn update(self: @Ball) {
    // ...
}

// ...
```

The biggest annoyance here is that because each type gets its own namespace, you end up having to say things like `ball::Ball` a lot which is silly. This lead me to decid it was time to update how I handle imports a bit: you can now import specific types/statics instead of whole modules, and do so under aliased names to prevent clashes and such.

```rust
use ball::Ball;
use dbg::println as log;
use ball as b;
```

<br>

---

<br>

With all of these changes, I was able to easily make sure I don't miss any existing code when updating it to use new features/idioms by just adding new compiler warnings. So, for example, if I had previously written code that looked like this:

```rust
@Vec2 { x: x, y: y }
```

I now get some helpful warnings pointing out that there's now a nicer way to do this:
```
$ @Vec2 { x: x, y: y }
warning: redundant field name in initializer, when assigning a field to a
variable of the same name you can simply name the field and give it no explicit
value (use #[allow(redundant_field_name)] to ignore)
  --> input:1:12
  | 
1 | @Vec2 { x: x, y: y }
  |            ^        

warning: redundant field name in initializer, when assigning a field to a
variable of the same name you can simply name the field and give it no explicit
value (use #[allow(redundant_field_name)] to ignore)
  --> input:1:18
  | 
1 | @Vec2 { x: x, y: y }
  |                  ^  
```

For bigger changes, I just have old code error out at compile time if it doesn't match up with the newer version of the language. I'm my only user at the moment so while versioning is still important, backwards compat isn't. This is also one of the reasons I've held off on making any of these tools public: I want to maintain the ability to make drastic changes without bothering anyone besides myself.
