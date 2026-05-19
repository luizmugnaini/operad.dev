+++
title = "Exploring Emulation With Chirp, a CHIP-8 Interpreter"
date = 2023-02-23
updated = 2025-06-01
description: "A simple CHIP-8 interpreter"

[taxonomies]
tags = ["programming", "side-project", "emulation"]
+++

One of my long-term aspirations is to forge an emulator for the legendary [Game
Boy](https://en.wikipedia.org/wiki/Game_Boy) console. As a side-quest, I decided to write a virtual
machine for the [CHIP-8](https://en.wikipedia.org/wiki/CHIP-8) interpreted language from the
mid-1970s.  Although not a hardware emulation project, it makes you dig a bit into techniques
common to emulation software.

{{ figure(
    src="/img/chirp/maze.png",
    alt="Screenshot of the MAZE game running on chirp",
    position="center",
    caption="MAZE game running on `chirp`",
    caption_style="font-weight: bold; font-style: italic;"
)}}

The heart of {{ repo(name="chirp", text="`chirp`") }} is an interpreter of the CHIP-8 programming
language, with roots in the early days of game development. This project can serve as learning
material for someone interested in dipping their toes in low-level waters. The project is written in
Rust.

{{ figure(
    src="/img/chirp/invaders.png",
    alt="Screenshot of the Space Invaders game running on chirp",
    position="center",
    caption="Space Invaders game running on `chirp`",
    caption_style="font-weight: bold; font-style: italic;"
)}}

I tried to make the external dependencies as minimal as possible:

- [`rand`](https://crates.io/crates/rand).
- [SDL2](https://www.libsdl.org/).

Sure, the number of dependencies is minimal, but that doesn't make the external code smaller. As is
commonly the case in Rust land, these dependencies are definitely not minimal. SDL2 is an absolute
beast and many of its features are not necessary for this project --- it could be completely
replaced by a simple hand-rolled OS-dependent windowing and input handling implementation, however I
didn't bother doing this for this simple side-project.

Throughout the creation of `chirp`, I found [Cowgod's CHIP-8 Technical
 Reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM) to be an awesome resource, serving as
 the main reference for the project.
