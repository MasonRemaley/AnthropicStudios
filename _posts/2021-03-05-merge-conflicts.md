---
layout: post
title: "Merge Conflicts: Avoiding Them, Making Them Less Painful"
date: 2021-03-05
tags: tech
author: mason
excerpt_separator: <!--more-->
description: Can we avoid merge conflicts? Unfortunately, no, but we can decrease how often we get them and take steps to make them less painful when they do happen.
image: /assets/posts/2021-03-05-merge-conflicts/conflict.jpg
twitter: https://twitter.com/masonremaley/status/1367889259223515137
---

<figure><img src="/assets/posts/2021-03-05-merge-conflicts/conflict.jpg" alt="a merge conflict"/></figure>

> Hey Mason could you explain to me a safe way to use github with multiple users for a Unity project? I know a little about github, and I’ve set up the project with the Brackeys’ tutorial. That all went great. I even have the other people pulling the project onto their machines successfully. And I can successfully do a push from mine. My concern is what if they make changes, and want to push. Do I need to do a push first before they do or do they need to do a pull before they do a push? I’m a bit confused on the sequence, and in 120 the git got all screwed up and that has me concerned!

Normally, I reserve this blog for updates on [Way of Rhea](/way-of-rhea), or [low level engine work](/2021/02/20/fullscreen-exclusive-is-a-lie/). This post is a little different.

Way of Rhea is entirely self funded. Some of that funding comes from savings from when I worked full time as a programmer, some of it comes from freelance work[^1], and the rest comes from teaching grad and undergrad students game dev.

Regardless of whether I'm currently teaching grad or undergrad, Unity or engine dev, general C++ or graphics programming, there's one key programming skill that nobody seems to ever teach my students before they get to my class: version control.

<!--more-->

I'm not sure why this is. I suppose everyone hopes someone else will talk to them about it. Regardless, I always make a point to spend a little class time on version control, because I can't in good conscience watch my students struggle through multi-week assignments without a VCS.

Today, I got the really great question above from a student. I spent a few minutes writing up an answer, and then realized, I should probably *save* this answer somewhere so I can point future students to it.

So, that's what this post is! A public note to myself for later. My answer follows.

---

Hey everyone! @Rob mentioned having issues with merge conflicts in git, and I realized I never talked about these. If you're not using git, this may not be relevant to you, but if you are it's probably worth reading on!

# Table of Contents
* TOC
{:toc}


# Merge Conflicts

A merge conflict occurs when two people edit the same file, and then try to push (upload) their changes to the git repo (e.g. GitHub.) So, for example, imagine this scenario:
- You pull (download) the latest project files from GitHub
- You make some changes, and commit them
- You push (upload) your changes

<br>
When you get to that last step, three things could occur:
1. Nobody else has edited the file you were editing, you're good! It gets uploaded and you go about your day.
2. Someone else edited it, but they edited different parts of the file than you, and git manages to automatically merge their changes with yours. Awesome, you go about your day.
3. Someone else edited it, and git isn't able to merge their changes with yours. Maybe you were working on the same lines. **You get a merge conflict.**



# Resolving Merge Conflicts

Merge conflicts are a fact of life working with git. How do we deal with them?

How you resolve a merge conflict is going to depend on what tools you're using. I don't recall off the top of my head, but GitHub Desktop (for example) may provide a GUI that lets you choose which changes to which files to keep/how to merge them together. At the end of the day though, when git gets a merge conflict, it inserts stuff like this into your code/data files:

```cs
<<<<<<< HEAD
void FooBar() {
    DoStuff();
}
=======
void FooBarRenamed() {
    DoStuff();
}
>>>>>>> branch_name
```

What this is trying to tell you is "you both edited this function, and I don't know which version you want to keep". When this happens, you have to manually edit the file (or use your GUI tool) to pick the version of the snippet of code you want, or to merge the two together into one.


# Can We Avoid Merge Conflicts?

Okay, this all seems pretty annoying. Can we avoid merge conflicts? Unfortunately, no--this is a fundamental issue with allowing multiple people to edit the same files at the same time. The only way to 100% avoid them would be to never edit files simultaneously.

In some version control systems, you can "lock" a file to do just that--prevent anyone else from editing it while you're working on it. Git does not have this feature, and it isn't a perfect solution. (e.g. what if you forget to unlock a file before you leave for vacation? Your coworkers are going to need an escape hatch to override your lock, and you're going to come back to a merge conflict.)

We might not be able to avoid 100% of merge conflicts, but we can minimize how often they occur, and we can make them less painful when they do occur!

# Minimizing How Often Merge Conflicts Occur

Here are some ways you can minimize how often merge conflicts occur:

- Communicate what you're working on with your team. If you know someone is working on `ZombieAttack.cs`, then you know if you edit that file you might get a merge conflict. (Or an automatic merge that doesn't do what you would've wanted, it can't always guess right!)
- If you're messing with a scene file to test out your code, instead of messing with the main scene file, make your own scene file for testing purposes. e.g. I might create "Mason's AI Island.unity" and use that when testing an AI so as not to cause whoever's building the real map merge conflicts.
- If you're making a sweeping change to the entire project, let everyone know--and give them a chance to push (upload) their changes before you do it. If they push their changes first, and then you make the change, they can pull (download) your sweeping change before they continue working and nobody will get a conflict.



# Making Merge Conflicts Less Painful

Sometimes, you really do need to edit code someone else is also editing. Here are some things you can do to make the resulting merge conflict less painful:

- Pull often. A smaller merge conflict is easier to resolve than a large one. The more often you pull, the more up to date the code you're working on is. Same goes for pushing often, pushing often will help keep everyone else in sync--but don't push before you're ready for everyone else to be affected by what you've changed!
- Write good commit messages. If you understand what the commit that caused the conflict was trying to do, it'll be easier to merge it!
- Enable text mode for scenes (Unity specific.) IIRC this is `Edit > Project Settings > Editor > Force Text`. This will output text instead of binary for scene files, which git will understand better.
- Make smaller commits--the smaller your commits, the less complicated your merges will be. This will also let you push and pull more often.
- [Use the diff3 conflictstyle](#the-diff3-conflict-style)
- It bears repeating, pull often!

## The Diff3 Conflict Style

Thanks to [@andy_kelley](https://twitter.com/andy_kelley/status/1367916737711067138?s=20) for this tip!

The default "conflict style" is a little confusing. If you're working with the command line, you can enable a slightly friendlier one like this:

```bash
git config --global merge.conflictstyle diff3
```

With this mode enabled, your merge conflicts will now look like this:

```cpp
  int main() {
++<<<<<<< HEAD
 +    // This is a comment
 +    do_stuff(1.0, 2.0);
 +    do_other_stuff(3.0, 4.0);
 +    return 0;
++||||||| merged common ancestors
++      // This is a comment
++      do_stuff();
++      do_other_stuff();
++      return 0;
++=======
+     // This is a comment
+     do_stuff_ex();
+     do_other_stuff_ex();
+     return 0;
++>>>>>>> b2
  }
```

The middle section is the most recent commit both branches share before the conflict. Being able to see the "original" code often makes it much easier to figure out how to merge the changes together!

*Note: A lot of my students use GitHub Desktop, and there doesn't appear to be an option in the GitHub Desktop UI for this. At some point I'll hunt down the location of GitHub Desktop's `.gitconfig` and update this post--if you happen to know where it is, feel free to let me know!*

# Lastly: Don't Panic

I've been using git as my primary VCS for a long time, but I definitely remember that early on, my first merge conflict scared me. There were all these `>>>>`s in my code! And nothing compiled! I must have done something terribly wrong!

Don't worry: this is normal and expected. Git has the full history of everything you've done to your project, you haven't lost any of it. Yes, this current version, isn't working, but it likely will with a few very small changes, and you also have all previous commits saved both on your computer and on the server you're pushing to which you can always revert to.

Merge conflicts are a normal part of working with git, if you're super stuck on resolving yours shoot me a message and I can help walk you through it.[^2]

[^1]: Need someone to overhaul your shader compiler, build a reflection system for your engine, or improve your enemy AI? I do a lot of gamedev freelance--[feel free to get in touch!](mailto:mason.remaley+pub@gmail.com)
[^2]: This offer was intended for my students. I'm always happy to help anyone with general gamedev questions, but if we're strangers and you bring a very specific merge conflict problem to me don't be surprised if I don't have time to look at it in a lot of detail. :)
