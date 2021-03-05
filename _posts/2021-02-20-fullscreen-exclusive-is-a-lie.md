---
layout: post
title: "Fullscreen Exclusive Is A Lie (...sort of)"
date: 2021-02-20
tags: way-of-rhea tech
author: mason
excerpt_separator: <!--more-->
image: /assets/monsters-and-sprites/fullscreen-exclusive-is-a-lie/settings.jpg
description: Fullscreen exclusive is real, but I haven't been able to find a single game where the fullscreen exclusive vs borderless window settings consistently do what you'd expect.
twitter: https://twitter.com/masonremaley/status/1363279139000770566
reddit: https://www.reddit.com/r/rust_gamedev/comments/lokeml/fullscreen_exclusive_is_a_lie_sort_of/
---
<!-- TODO: ... -->
<!--
seo:
  links:
    - https://www.gamasutra.com/blogs/MasonRemaley/20210111/376012/Setting_An_Executable_Icon_From_The_Command_Line_in_Windows.php
-->

<figure><img src="/assets/monsters-and-sprites/fullscreen-exclusive-is-a-lie/settings.jpg" alt="way of rhea video settings"/></figure>

Fullscreen exclusive is a real thing your computer can decide to grant a window, but, as of yet I haven't been able to find a single game where the fullscreen exclusive vs borderless window settings consistently do what you'd expect them to.

If you're an expert in this topic and believe this post is incorrect in any way, definitely [email me](mailto:mason.remaley+pub@gmail.com) or [DM me on Twitter](https://twitter.com/masonremaley) and I'll issue a correction!

If you're interested in jumping to the conclusion, see [Testing Fullscreen Exclusivity in Other Games]({{ site.baseurl }}{% link _posts/2021-02-20-fullscreen-exclusive-is-a-lie.md %}#testing-fullscreen-exclusivity-in-other-games) and [Takeaways]({{ site.baseurl }}{% link _posts/2021-02-20-fullscreen-exclusive-is-a-lie.md %}#takeaways). If you're interested in my methodology, read on.


<!--more-->

# Table of Contents
* TOC
{:toc}

# Defining Our Terms

Today I'm specifically discussing Windows 10. I'm using OpenGL and calling into win32 directly, but these results will likely be of interest regardless of your graphics API and windowing library. I should probably define some terms before we start:

* When I say fullscreen exclusive, I'm referring to a fullscreen window that bypasses the compositor.
* I'm aware that there's technically a distinction between [full screen optimizations mode](https://devblogs.microsoft.com/directx/demystifying-full-screen-optimizations/) and fullscreen exclusive, but I'm not particularly concerned about that difference. I ran a number of these tests with FSO disabled and saw no discernible difference.

It's also worth noting, most games that have a fullscreen exclusive option will mess with your display's resolution when it's enabled. Whether or not changing your display settings is *necessary* for exclusivity, it's certainly not *sufficient*: again, our goal is to bypass the compositor.

# Resolutions & Scaling

A few days ago, I decided that it was time to finish setting up the video options for [Way of Rhea](/way-of-rhea). I previously supported the following window modes (or, at least, I thought I did!):

* Borderless window
* Windowed

<!-- todo: mention that scaling multiples thing is incorrect? -->
I didn't properly lock the aspect ratio in either of these which could lead to UI and/or game elements being off screen on some setups, so my first goal was to fix that. In doing so, I realized that [it's kind of weird that most games only let you choose from a fixed set of resolutions](https://twitter.com/masonremaley/status/1362331833237708806).

Traditionally this made sense--if you're going to mess with the display settings to get the resolution you want, then you can only support resolutions the display supports! However, if you're in borderless window mode which is becoming more and more common, you can set the resolution to anything you want since you're just resizing a framebuffer.

I figured, if fullscreen exclusive support is gonna dictate which resolutions I can support, maybe I should knock that out first. How hard could that be?

Well, you've presumably already seen the title of this post and have a guess at what the answer to that question is. :)

# Methods of Enabling Fullscreen Exclusive Mode in OpenGL

Presumably DirectX has an API for this, but I'm targeting OpenGL.

The internet is full of hearsay about what you need to do to get fullscreen exclusive mode in OpenGL. Some say that you need to apply the `WS_POPUP` window style, others say you need `WS_POPUP | WS_CLIPCHILDREN | WS_CLIPSIBLINGS`. Others claim you need to call `ChangeDisplaySettings` with `CDS_FULLSCREEN`.

There's a big issue with all of these claims: they're not documented, and there's no obvious way to test if they actually work!

If we're serious about doing things right, we need to find a way to test for fullscreen exclusivity.

# Testing If We're Fullscreen Exclusive

I'd love to be wrong about this, but as far as I can tell, Microsoft/NVIDIA/etc do not provide a way to directly check if a window is fullscreen exclusive. If you search around online for ways to test for exclusivity, you'll find more hearsay:

## Tests That Don't Work

### Volume Meter

People often check for exclusivity by changing the computer volume via a hot key and observing whether or not the volume overlay shows up on top of the game.

We can use this to rule out exclusivity (the volume meter won't show up if we're exclusive since we're bypassing the compositor), but we can't use it to prove exclusivity--I've seen games that are clearly not exclusive manage to hide the volume meter anyway.

(I'm actually rather curious how they accomplished this, if you know how to do this [I'd be interested in hearing from you](mason.remaley+pub@gmail.com).)

### Windows Key

Another common suggestion is to check whether the Windows key works. I can see why people suggest this as a test--on some of my test machines, fullscreen exclusive windows *do* obscure the windows menu. However, this is not true on all of them, and if you're testing a game for which you don't control the source it's very likely that [the windows key was disabled manually by the game.](https://docs.microsoft.com/en-us/windows/win32/dxtecharts/disabling-shortcut-keys-in-games)

If you try this test yourself, you may notice that even on setups for which the windows menu is not obscured, it causes the window to flash black as it loses exclusivity...

### Flashing

The last common suggestion I've seen is to check whether the window/monitor flashes black when it loses focus. In my tests so far, fullscreen exclusive windows *will* flash when they lose focus, however, there are other potential sources of flashes--`ChangeDisplaySettings` for example can cause the monitor to flash even if no settings are changed, and many games call it when gaining/losing focus.

I've managed to convince myself that can tell the difference between a flash due to losing exclusivity and a flash with another cause, but it's very subtle and not something I would want to rely on to check whether or not my game is working correctly!

## A Test That Works

<figure>
	<img src="/assets/monsters-and-sprites/fullscreen-exclusive-is-a-lie/windows-menu.jpg" alt="way of rhea with the windows menu open"/>
	<figcaption>It's not the classic windows key test–there's a little more to it.</figcaption>
</figure>

Alright, we've listed a number of methods that *don't* work. So what do we do?

If we're in fullscreen exclusive mode, we're bypassing the compositor. This means that things like print screen will not behave correctly.

In particular, if you have a fullscreen exclusive window, and you press the print screen shortcut, instead of getting a current screenshot you'll get a screenshot of *the last non-exclusive frame*. Normally this is a nuisance, but today we can take advantage of it!

The test is as follows:

* Launch the possibly-fullscreen-exclusive game
* Move the camera to somewhere memorable
* Temporarily prevent the app from being exclusive, e.g. by pressing the Windows key twice to toggle the Windows menu on and off or obscuring part of the game with another window
* Once the game is back to its normal state, move the camera to a new location
* Press your print screen shortcut
	* It's important that you use print screen--don't take a screenshot with Steam for example, Steam can screenshot exclusive windows correctly
* Check the resulting screenshot:
	* If the screen shot is up to date, you're not exclusive.
	* If the screenshot is of the area of the game you were in while the windows menu/such was present, you're in exclusive mode! (the menu/window itself will not appear in the screenshot)

*Note: I can't promise Microsoft won't fix this someday, you'll have to test for yourself whether these results are reproducible on your version of Windows. If you're never able to get an out of date screenshot, it's either been fixed or you haven't tested a truely fullscreen exclusive app yet.*

# Testing Methods of Gaining Fullscreen Exclusivity in OpenGL

I tested out the various methods of gaining fullscreen exclusivity in OpenGL suggested online on 5 different setups. Here's what I found:

* I've never successfully gained exclusivity without the `WS_POPUP` window style
* I've never observed the `WS_CLIPCHILDREN` or `WS_CLIPSIBLINGS` window styles making a difference
* I've never observed calling `ChangeDisplaySettings` with `CDS_FULLSCREEN` making a difference
	* This shouldn't be too surprising--despite the name, according to [the docs](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-changedisplaysettingsa#CDS_FULLSCREEN) all it does is make the other settings changed by `ChangeDisplaySettings` temporary. [See also Raymond Chen](https://devblogs.microsoft.com/oldnewthing/20080104-00/?p=23923).
* On all 5 setups, I can gain exclusivity at any resolution without without calling `ChangeDisplaySettings`
	* I just render to a framebuffer and upscale to the native resolution, just like in borderless window mode. Nothing about this requires the compositor.
* On two of the five setups, I was unable to gain exclusivity without being [DPI Aware](https://docs.microsoft.com/en-us/windows/win32/hidpi/setting-the-default-dpi-awareness-for-a-process)
	* Despite the overly confusing DPI Awareness API, this does make sense--if you render at the wrong DPI the compositor will need to scale your final result before it's displayed. On the setups where I didn't need to be DPI Aware, I had a "normal" DPI that required no scaling.

Okay, so things are looking good for `WS_POPUP` + DPI Awareness right? Well...

`WS_POPUP` and DPI Awareness do appear to be *necessary* for exclusivity, but they're unfortunately not *sufficient*! On the setup that I expect is most similar to most player's setups (single monitor w/ an NVIDIA GPU) I wasn't able to figure out how to acquire fullscreen exclusivity at all.

Alright, this is kind of a mess! How the heck are other games doing it?

# Testing Fullscreen Exclusivity in Other Games

> Alright, this is kind of a mess! How the heck are other games doing it?

...well apparently, they're not!

I've tested 6 games so far ranging from well received graphically intensive indie games to larger well received AAA games that you would expect to care about this sort of thing.

I won't name the games as I don't want to criticize them in particular, the issue appears to be universal. Here are the "anonymized" results--I tested on the setup that I found to be the most forgiving in terms of allowing exclusivity:


<table style="display: table; font-size: 80%">
  <tr>
    <th>Scale</th>
    <th>Release Year</th>
    <th>Graphics API</th>
    <th>Engine</th>
    <th>Fullscreen is exclusive</th>
    <th>Borderless Window is exclusive</th>
  </tr>
  <tr>
    <td>Indie FPS</td>
    <td>2020</td>
    <td>DirectX</td>
    <td>Unity</td>
    <td>No</td>
    <td>No</td>
  </tr>
  <tr>
    <td>AAA TPS</td>
    <td>2020</td>
    <td>DirectX</td>
    <td>In House</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
  <tr>
    <td>AA TP</td>
    <td>2014, but kept up to date</td>
    <td>DirectX/Vulkan</td>
    <td>In House</td>
    <td>No</td>
    <td>No</td>
  </tr>
  <tr>
    <td>AA FPS</td>
    <td>2020</td>
    <td>DirectX</td>
    <td>Unreal</td>
    <td>No</td>
    <td>No</td>
  </tr>
  <tr>
    <td>Indie FP</td>
    <td>2020</td>
    <td>DirectX</td>
    <td>In House</td>
    <td>Yes</td>
    <td>Yes</td>
  </tr>
  <tr>
    <td>AAA TPS</td>
    <td>2020</td>
    <td>DirectX</td>
    <td>In House</td>
    <td>No</td>
    <td>No</td>
  </tr>
</table>

Of the 6 games tested, not a single one could reliably toggle fullscreen exclusivity. Four of the six never achieved exclusivity on my setup, and the remaining two were always exclusive when fullscreened--even when in borderless window mode.

Of note, this list contains a Unity game, an Unreal game, and four different proprietary engines.

The only thing the fullscreen exclusive option reliably does in the games listed is change your display resolution to match the game--and one of the six (AAA TPS 2020) didn't even do that.

# Takeaways

If nobody else has found a way to offer a reliable fullscreen exclusivity toggle, I see little reason to pretend that I offer that mode either!

In addition, since upscaling from a framebuffer seems to have no effect on exclusivity (and no measurable perf effect as compared to changing the monitor resolution), I have no reason to limit the resolutions at which my game can render.

Taking these observations into account, my plan is as follows:

* Offer a borderless window mode
* Offer a windowed mode
* Offer a render scale option as a percent of the native resolution
* Offer the ability to resize the windowed mode window arbitrarily within the supported aspect ratios

This will give technically adept players options without over promising, and should be less confusing than a large list of resolutions for everyone else.

I am conflicted as to whether or not I should use the `WS_POPUP` style in borderless window mode--on one hand, with it at least *some* players will get fullscreen exclusivity, OTOH those players then won't be able to use print screen. I'll likely end up doing what I do with vsync--making a checkbox for it, but labeling it as a "hint" to acknowledge that I as the developer don't actually have the final say as to whether or not the setting has any effect.