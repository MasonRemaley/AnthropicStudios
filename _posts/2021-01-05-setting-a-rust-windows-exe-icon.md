---
layout: post
title: "Setting a Rust Executable's Icon in Windows"
date: 2021-01-05
tags: tech
author: mason
excerpt_separator: <!--more-->
---
<!-- TODO -->
<!-- twitter: https://twitter.com/masonremaley/status/1312803634833510400 -->
<!-- reddit: https://www.reddit.com/r/gamedev/comments/fst8i3/symmetric_matrices_triangle_numbers/ -->

<!--
  intro to the problem, make sure googleable
  steps the solution
  further thoughts
-->

<!-- TODO: add image -->

This morning, I decided it was long overdue that [Way of Rhea](/way-of-rhea) get its own icon.

<figure><img src="/assets/monsters-and-sprites/setting-a-rust-windows-exe-icon/icon.png"/></figure>

I believe that if you're building a project in Visual Studio there's a UI through which you can change your exe's icon--but I'm not a Visual Studio user. It took me quite a while to figure out how to set an exe's icon from the command line, so I figured I'd document what I learned here in the hopes of saving someone else some time.

<!--more-->

It's possible to dynamically load and set a *window* icon via code, but for the icon to show up in the file explorer it actually needs to be baked into the executable. This makes sense--`explorer.exe` shouldn't have to have to run an executable to determine its icon!

The rest of this post will walk you through how to do this. The only Rust specific bit is the syntax by which I pass arguments to the linker.

# Table of contents
* TOC
{:toc}

# Instructions

## 1. Create an icon

First, you'll need to create [a **square** image ideally at least 256x256px](https://docs.microsoft.com/en-us/windows/win32/uxguide/vis-icons), and save it as a `.ico`. If you're not sure how to create a `.ico`, you can use [GIMP](https://www.gimp.org/).


## 2. Create a resources file

Next, you'll need to create a [`.rc` file](https://docs.microsoft.com/en-us/windows/win32/menurc/about-resource-files) that [provides the icon path](https://docs.microsoft.com/en-us/windows/win32/menurc/icon-resource). Here's what it should look like assuming you aren't including any other resources:

**resources.rc**
```rc
arbitrary_name_here ICON "path\to\your\icon.ico"
```

## 3. Compile the resources file

Next, you'll need to compile your `.rc` file. The official way to do this is via [`rc.exe`](https://docs.microsoft.com/en-us/windows/win32/menurc/resource-compiler).

Unfortunately, `rc.exe` is not in the path by default, so you'll need to find it. Mine is located at `C:\Program Files\ (x86)\Windows Kits\10\bin\10.0.18362.0\x86\rc.exe`. It was likely placed there when I installed Visual Studio.

Once you've located `rc.exe`, you can use it to compile your `.rc` file into a `.res` file:

```sh
rc resources.rc
```

(*Side note--programmatically determining the path to `rc.exe` is, unfortunately, not easy. If you can't work around this, here are some options:*

- *Require the user to provide the path*
- *Call it from within a [Developer Command Prompt](https://docs.microsoft.com/en-us/dotnet/framework/tools/developer-command-prompt-for-vs) where it's already in the path*
- *Use [LLVM](https://llvm.org/)'s reimplementation [`llvm-rc`](https://github.com/llvm/llvm-project/tree/62ec4ac90738a5f2d209ed28c822223e58aaaeb7/llvm/tools/llvm-rc)*
- *Check out [how the Zig compiler finds similar files](https://github.com/ziglang/zig/blob/master/src/windows_sdk.cpp), and write up something similar for `rc.exe`*

*If you've found a better way to do this, or know if it's possible to use [vswhere](https://github.com/Microsoft/vswhere) for this purpose, [let me know](mailto:mason.remaley+pub@gmail.com) and I'll update this post!*)

## 4. Link with the compiled resources

Lastly, you need to link with the `.res` file when building your executable. How exactly you do this depends on your compiler and linker.

Here's how I do it in Rust, with unrelated options removed for clarity:

```sh
cargo rustc -- -C link-args="resources.res"
```


---

That's it! If everything has gone well, your executable should display your icon in the file explorer and the task bar.

<figure><img src="/assets/monsters-and-sprites/setting-a-rust-windows-exe-icon/task-bar.png"/></figure>

If things *aren't* working correctly, there are third party tools like *Resource Hacker* that you can use to compare your final executable's resources to that of an executable with a working icon.
