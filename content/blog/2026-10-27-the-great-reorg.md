+++
title = "The Great Reorg"
draft = false
date = 2026-10-27
description = "Big strides with the DSL, breaking bootstrap cycles, and building an ecosystem for everyone"

[taxonomies]
authors = ["ikey"]
+++

You didn't watch [the video](https://youtu.be/6I6zBvLgW3M?si=49mHwF3Lrils_uud), did you? OK, it's fine, I'll give you an update here then...

Recently, I've been getting the world around me a little more organised as I get the last parts of the `DSL` sorted.

<!--more-->

## Firstly, a mini progress update

OK, as you STILL haven't watched the video, I'll let you know where we're at codewise. The DSL is the biggest priority right now because ensuring the data format is sound for all operations is key to the entire game.

While the format hasn't **fully stabilised** yet, here's a few interesting updates.

### File format

All `barney` recipe files (modules, specs, etc.) use the `.bny` suffix and require an explicit schema version as the first node:

```kdl,name=demo.bny
/// stick your spdx header here

(barney)version 1

/// Also supported for brevity:
(bny)version 1

package "blahblah" {
...
```

TL;DR: `bny` is the shorthand name for `barney`, and will also be the CLI name.

### Static recipe properties

Recipes, modules and the distro configuration support static, typed properties. These are baked into the build and support binding, string building and, of course, distro-level overrides.

This gives us a mechanism for expressing configuration explicitly, allowing the same recipes to be reused across different compositions without baking distro-specific decisions into every individual recipe.

### Recipe variants

A recipe may have **variants**, which are legal for use only in build dependencies. These non-default variants are never published to the final builds or end users. They're used specifically for supporting bootstrap problems.

A variant is simply a named version of a recipe that overrides its static recipe properties.

This means we can build intermediate versions of packages with different configurations without exposing those versions as part of the final system.

### Cyclical bootstrap dependencies

Combining static recipe properties with recipe variants allows a very powerful mechanism to emerge: explicit handling of cyclic build dependencies.

Consider the interdependency of `systemd` and `util-linux`, without relying on the `dlopen()` approach used today:

```kdl,name=util-linux.bny
package "util-linux" {
    properties {
        (bool) with_systemd #true
    }

    variant "bootstrap" {
        with_systemd #false
    }

    depends {
        build {
            (binary)make
            util-linux variant="bootstrap" conflicts=(prop)with_systemd
            systemd if=(prop)with_systemd
        }
    }
    ...
}
```

```kdl,name=systemd.bny
package "systemd" {
    properties {
        (bool) with_utillinux #true
    }

    variant "bootstrap" {
        with_utillinux #false
    }

    depends {
        build {
            (binary)make
            systemd variant="bootstrap" conflicts=(prop)with_utillinux
            util-linux if=(prop)with_utillinux
        }
    }
    ...
}
```

Build dependencies support conditions bound to static properties, as well as explicit variant selection.

The magic happens when the default variant expresses a build dependency on its own bootstrap variant, with conditional dependencies controlling which other packages are required. Our build graph will allow a single recipe, once compiled, to generate multiple nodes in the graph, each representing a distinct build configuration.

While this is only a poor man's example, there's a lot more you can do with this mechanism. Variants can express different build configurations, with non-default variants restricted to build dependencies and never published as part of the final system. The possibilities are pretty broad.

Using these expressive, declarative dependencies gives us a way to explicitly break bootstrap cycles. Combined with our phase-based bootstrap approach, this means we can describe the build strategy in recipes instead of manually mangling a rootfs or spinning up an interim container just to get a complete build.

No more build nannying. <3

### GitHub organisation

The main projects now live in the [barney-build](https://github.com/barney-build) organisation:

- [barney](https://github.com/barney-build/barney) — Linux composition pipeline.
- [bny-base](https://github.com/barney-build/bny-base) — Shared base recipes.

This gives us a clear separation between the machinery that builds operating systems and the recipes used to produce them.

## Project scope

As I pointed out in my latest video, building a "better distro" is by itself simply a pissing contest. Instead, we want to **build distros better**.

Barney is a Linux composition pipeline: a shared mechanism for building, composing and delivering operating systems. Rather than letting one opinionated distro dictate the recipes, tooling and engineering decisions, we're building a reusable factory that others can extend, derive from or use to produce something entirely their own.

Think of it as a modern take on Yocto, or "bring your own distro". The aim is to make the process of building and maintaining distributions substantially less painful.

Building distros shouldn't be as hard as it is, so let's fix it, k?

The base layers, tooling and recipes will be shared through a public organisation for everyone to build upon, extend or use as they see fit. The goal isn't to enforce a particular distribution's choices on everyone else, but to give people the tools to make their own.

## Pavee OS

I'm building, in what can only be described as **massive irony**, a "wallpaper respin".

[Pavee OS](https://github.com/barney-build/pavee-os) will be **my** opinionated daily-driver desktop, built **on top** of `barney`, `bny-base` and any other layers we produce.

It'll be branded and configured using the distro configuration file and overrides of the static recipe properties.

Pretty cool, right?

I've been building distros the wrong way round for so long now. It's time to invert the relationship: build a reusable, modern ecosystem that others can share without forcing a particular distro's design decisions on every project.

Pavee OS gives me a way to demonstrate the factory in practice, while keeping my own preferences separate from the shared infrastructure.

## Amazon Wishlist

I was asked in a comment on my most recent video to set up a wishlist for things that would make my development journey easier, both for **recording the journey** and **building it**.

Honestly, I thought, y'know what? That's genuinely not a bad idea.

The stuff on there is mostly about making life easier and improving ergonomics (I have multiple nerve entrapments), alongside getting a more capable setup for recording videos and eventually livestreaming.

There's absolutely no expectation to buy anything. These are things I'll work towards getting anyway, but if anyone wants to treat me to an early Christmas present, I'm certainly not going to say no! xD

[Travelling Engineer Wishlist](https://www.amazon.co.uk/hz/wishlist/ls/1L8ZMBIEXT8PC?ref_=wl_share)

## Sponsors

And, of course, sponsoring is an option for those who like what I'm doing.

I've decided, as mentioned in the video, that exclusive videos aren't really a value-add. Instead, the focus will be on early-access builds and being "in the loop" a bit quicker than everyone else.

The aim is to let supporters follow development more closely and get access to early builds, while keeping the normal releases available to everyone.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/R6Q326CJIX)

## And finally

This has actually been a really fun journey so far.

Yes, my current setup is a bit, er, outdated, and with some improvements I'll be able to do livestreams and proper coding videos. It's a bit hard to record at the computer with a house full of kids with additional needs and no decent microphone, for example. xD

I've got the last improvements for this stage of the DSL prototype in progress locally. This is essentially replacing `TaggedValue` with new `NodeTag` and `EntryTag` systems to differentiate between schema and type-restriction rules, and the usual property and variable-binding rules for entries.

I'm also reworking things so that the `kdl` crate isn't exposed outside of `bny_syntax` or `bny_recipe`, allowing for a cleaner separation of concerns.

The realisation that I've needed to focus on tooling and a shared mechanism for building collaboratively, while still having my own daily driver on top without conflicting interests, has been a big one.

And y'know what? I'm happy about it, too. The support I've seen from others who also want an easier, more consistent way of delivering operating systems and appliances has been genuinely encouraging.

The plan is straightforward:

- Share `barney`, `bny-base` and other layers so everyone can build something of their own.
- Document the engineering, decisions and processes involved along the way.
- Use `Pavee OS` to demonstrate the factory in my own peculiar way, without forcing my design decisions on everyone else.

Basically, I'm condensing decades of engineering experience into tooling, documentation and strategy for the next generation of builders.

Until the next update: cheers!
