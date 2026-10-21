+++
title = "Barney: A Progress Update"
draft = false
date = 2026-10-21

[taxonomies]
authors = ["ikey"]
+++

Hello all and welcome to the new blog! I'm Ikey Doherty and a little while back, I declared [war on the industry](https://lnkd.in/p/egEgg6nP).
Unfortunately during that time many have cherry-picked singular data points and have failed to grasp the entire picture so
let's have a little rehash.

![screenshot](/images/first-post/barney_progress.png)

<!--more-->

{% <devlab.callout type="info" title="Active development"> %}
The `barney` project is under active development and I expect many parts to
change in time. While the DSL is well considered, the exact syntax and nature
may improve with iteration.
{% </devlab.callout> %}

## Triggered out of retirement: Context

As a **software engineer** of course it triggers me somewhat when I see techbros going all-in AI-slop on a *respin* and selling it as
the elixer of life. And of course, insult to injury, at the time of my announcement Omarchy was being backed left right and center
by the industry to the tune of ~ $13MN.

That in itself wasn't entirely interesting. Instead, the absolute racist drivel being shoved into the limelight by DHH regarding nomadic
people, his comments on how to "deal" with gypsies, pissed me off. I'm an Irish Traveller, and I find any such rhetoric about nomadic
people utterly disgusting. No, I'm not talking about "digital nomads" I'm talking about people much like me that have been oppressed
and persecuted for hundreds of years, indigenous nomadic people.

See the big picture here a minute. When someone like DHH with his vile views, becomes the de-facto posterboy of the software industry...
we're in big shit. The industry at large is happily backing his ventures and normalising his views, which to me is an existential threat.

Hence Barney. Not so much as "no AI slop" but "here's something built by someone you clearly despise due to their background outdoing you
without requiring a right-wing techbro circle of investers". Simply put: defiance in the face of a woefully corrupted industy.
Or just pure old school "fuck you" vibes. Take your pick.

## The DSL

OK, enough with the war cries, let's have a look at the foundation of Barney: A proper DSL. Not **completely** `NIH` - don't worry. It's
based on [KDL](https://kdl.dev/). Essentially I'm building a compiler approach that's building an `AST` atop the core `KDL` pass.

### The distro file

Core to the whole design is the central `distro` file. Essentially this contains the main phases of the bootstrap/build, setting specific variables
for each phase run. Incidentally the currently building phase will be exported as `phase` for binding/reuse in other modules and recipes.

```kdl,name=distro.kdl
phase crosstools {
    variables {
      target "x86_64-linux-gnu"
      prefix "/usr"
    }
}
```

### The module files

Instead of allowing arbitrary shell scripts or string substitutions I've insisted on a hashable, verifiable chain of custody. Effectively we have
a mini scripting language in the DSL in `execute` sections of modules. Quite simply a script bakes to a list of enums, which can be of the following
types:

 - `ForEach`
 - `WithCondition`
 - `Execute`

 Internally we will bind an absolute minimum number of functions, ie `emit`. The entire purpose of a script is to reliably generate a shell-compatible
 script in a reproducible fashion. Essentially it's a string builder with a functional approach, without requiring complex semantics. For some ease of
 use we will adopt Rust-style namespace resolution, i.e. `autotools::configure`.
 
```kdl,name=autotools.kdl
module autotools {
  // autotools::configure
  action configure {
    properties {
      (string) command "./configure"
      (string) prefix "/usr"
      (list) enable "shared"
      (list) disable "static"
      (list) with
      (list) without
      (string) target (var)env::target
    }

    // Begin writing a shell script
    execute {
      emit (prop)command "--prefix=" (prop)prefix
      with enable {
        emit " "
        foreach prop enable {
          emit "--enable-" (var)prop
        }
      }
      with disable {
        emit " "
        foreach prop disable {
          emit "--disable-" (var)prop
        }
      }
      with target {
        emit " --target=" (var)env::target
      }
      emit_newline
    }
  }
}
```

### The recipe files

Right now I haven't implemented the recipe format itself as it's genuinely the least interesting element right now.
The semantics come from the modules, the logic or "magic" is from the design and there will be no special features in
the DSL that need to be implemented for the recipe itself.

It's worth pointing out that flavours / profiles / etc are to be baked into the hashes throughout the entire chain.
You may be wondering, hold on, how do I do complex scripting in the recipes? By and large we'll avoid it because those
strings introduce deviance we cannot support for reproducible builds. In the odd case where some script is absolutely
unavoidable, it will be checked in as some script or patch **file** to ensure it is hashed and doesn't accidentally blow
up reverse dependency build chains.

Now, you're likely looking at that module above thinking "fucksake Ikey, why did you make packaging harder?" - don't worry
I haven't. I've made *deviation* difficult. While the recipe semantics aren't set in (ahem) stone just yet, here's a trivial
example of how the modules might be **used**:

```kdl,name=recipe.kdl
// No more YAML string insanity or indentation woes.
// Everything is node based. AST compiled after valid
// KDL document is read first.
build {
    autotools::configure prefix="/usr"
    autotools::make

    // Conditionals
    with flavor1 {
        autotools::make someTarget
    }
    with default {
        autotools::make someOtherTarget
    }

    // And finally install..
    autotools::make_install
}
```

## Compiler...?

You may be thinking, surely it's going to be too expensive to support the parsing and evaluation of all of these recipe
files given the system complexity? You're right, it is. Barney is being designed effectively as a **compiler** and will
cache the binary form of the recipes for indexing.

This will allow rapid evaluation and change detection for larger projects. While I won't reveal all details just yet, the
entire point of this design is to ensure we can support fully automated bootstraps and rebuilds with heavy caching, and still
support the production of a **binary distro** that can be used entirely client-side or a strict source only (default) mode with
heavy reliance on caching via binary artefacts.

It's worth pointing out the entire point of supporting conditional statements in the DSL at this stage. We will have default recipe
builds that are fully published through their phase, as well as **non default** variants that are explicitly enabled through a combination
of build dependencies and phase options. This will allow us to cycle-break the bootstrap and completely automate it without requiring
any hand-mangled halfway rootfs.

![miette](/images/first-post/miette.png)

Personally I'm sick to death of tooling that shits the bed 4 hours into a build, and would rather prefer to know up front that my recipe
has the *potential* to successfully run. Hence focusing heavily on the developer experience so early on with proper diagnostics <3

## Website

This website has only been live at the time of writing for one day. Much of it is still placeholder but I plan on having full documentation
and guides present here as well as linking downloads, and the odd blog post or two.

## Videos

Much of the journey will be documented on the [YouTube channel](https://www.youtube.com/@ikey.doherty). Lately I've been unable to produce
videos as much as I'd like primarily because of some irritating health issues I'm navigating (nobody wants to see my skin flareups on video!)

## Reference Platform

Two things need to be justified here. You cannot actually produce a meta build tool without having some kind of **reference platform** / distro
with which to test it and improve it. Secondly, you cannot produce a meta build tool that is robust without actually maintaining it. I openly
stated that I am currently not at all interested in maintaining **yet another GPOS**. This remains true.

I am however committed to `barney` and the associated tooling / ecosystem. A reference will be produced in a baseline fashion rather than a
"support mah deskop" "fix mah CVEs" kinda way. Fork it, base on it, its a starting point. It is *not* a user facing environment - that is
a problem for distributors.


## Supporting my efforts

I'm not a millionare nor will I ever be one. I am however grateful for a morning red bull and better computing equipment! You can support
my works either via [Ko-fi](https://ko-fi.com/TravellingEngineer) or [GitHub sponsors](https://github.com/sponsors/TravellingEngineer).
The split exists deliberately so that folks more inclined to support the content can focus on Ko-fi whereas those looking to explicitly support
`barney` and associated works can use GitHub sponsors. Of course, you're free to pick either!

I'm not demanding support in return for anything of course. I'm simply mindful that I can risk-free rent a Hetzner for hosting/building, or
get a dedicated webcam that *isn't* a broken iPhone 13 xD Ideally I'm looking to upgrade to a small form factor desktop unit (AMD plz) and get
off the Dell XPS 13, but again these are "nice to have" and not essential. Red Bull is non-negotiable ofc.
