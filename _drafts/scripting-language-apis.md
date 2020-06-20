---
layout: post
title: "Scripting Language APIs"
tags: way-of-rhea tech
author: mason
excerpt_separator: <!--more-->
---
<!-- date: 2019-06-05 --> <!-- TODO: ... -->
<!-- twitter: https://twitter.com/masonremaley/status/1143655586740838400
reddit: https://www.reddit.com/r/rust_gamedev/comments/c5lwi5/way_of_rheas_entity_system/ TODO: ... -->

<!-- TODO: intro -->

# Stack Based API

The first API I came across for talking to a scripting language from engine code was Lua's stack based C API. Here's hello world executed in Lua, from a C program:

```c
lua_getglobal(ctx, "print");
lua_pushstring(ctx, "Hello, World!");
lua_call(ctx, 1, 0);
```

Let's break that down. First, we push the function we want to call onto the stack:

```c
lua_getglobal(ctx, "print"); // [print]
```

Next, we push any arguments that we want to pass to the function:
```c
lua_pushstring(ctx, "Hello, World!"); // [print, "Hello, World!"]
```

Lastly, we call the function, and specify the number of argumetns we have and return results we expect:
```c
lua_call(ctx, 1, 0); // []
```

On one hand, this is a lot of boiler plate, and you can get it wrong for example by pushing the wrong number of arguments.

On the other hand, this is very flexible. You're directly talking to the interpreter, so you don't need some common subset to exist between the two languages to make things work, and you're not limited by what template magic was able to accomplish.
<!--more-->
# Higher Level APIs

When I started designing my language, I figured I could do better than this. I'd heard about people using template metaprogramming in C++ to reduce the boiler plate it takes to call Lua, and figured I'd just build my language to support that kind of API in Rust to begin with.

I ended up with this:

```rs
let print: Fun<(ScriptString,), Result<(), Panic>> = vm.lookup("std", "print").unwrap();
let message = ScriptString::new(&mut vm, "Hello, world!");
print.call(&mut vm, message);
```

Alright well it turns out it's not actually less code, but it *is* statically typed. Through lots of crazy generics and some codegen I'm able to pass (opaque) struct pointers from the scripting language to Rust as well.

The problem with this approach is simply that the implementation is super complicated, and it assumes that features in one language necessarily translate to features in another. I think if you're gonna go this route, you should define a common subset of each language that are easy to expose to the other from the start--I didn't, I started with the goal over *everything* being available to this interface, and as time went on and I added more features to the language, found that some features either realistically didn't translate well or weren't worth the time it would take to implement.

# extern "C"

So, where does that leave us?

Well, the higher level approach suffices for now. It's not ideal, but it's not worth changing for the time being either because it's getting the job done. I think if I was tasked with improving the situation without making any other changes to the language, I'd probably look for a compromise. Maybe the latter API style is built on top of the former, or maybe I go with the higher level API but just set a very small subset of the language that gets expose that way from the beginning.

I'm not planning on doing either of those things, because I'm planning on making a bigger change to the language that allows for another approach--

After I ship [Way of Rhea](/way-of-rhea), I'm going to try my hand at writing an x64 backend for my game. I originally targeted a [bytecode](http://craftinginterpreters.com/a-bytecode-virtual-machine.html) because the conventional wisdom says that that's what you do to be platform independent. Maybe that was true in the past, but...

Most of my players are on Windows, and everyone on Windows is on x64 nowadays. I don't even have to care about executable format for non-Windows users who are still on x64--the game will just compile all the code at runtime, and then jump into the compiled code.

What does this look like from an API standpoint? Well, we just need both languages to support the C ABI and we're ready to go. I haven't written this yet, but I imagine it'd look something like this:

```rs
extern "C" {
	fn print(_: *const c_char);
}

let message = CString::new("Hello, world!").unwrap();
unsafe {
	hello(message.as_ptr();
}
```

It's as if we implemented option 2 for a subset of the language, but:
- It requires very little implementation effort (assuming we're already targeting x64 and using a superset of the C ABI)
- We can now talk to *any* language that supports the C ABI
