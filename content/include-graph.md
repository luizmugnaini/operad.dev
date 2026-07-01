+++
title = "Include Graph Tool, a Proof of Concept"
date = 2026-06-30
description = "A tool that maps the #include graph of a C & C++ codebase"

[taxonomies]
tags = ["programming", "c", "cpp", "rust", "visualisation"]
+++

As part of [MAC5856 Open Source Development class](/mac5856) we were asked to create an analysis of
a chosen dataset. Since I'm not a data scientist by any means and I'm interested in development
tooling for long-running large codebases, I've decided to tackle C and C++ codebases as my dataset
of choice, and build a proof of concept tool to help in the management of huge codebases.

Given the simple nature of the dependency resolution of C and C++ --- where a `#include` simply pastes
the code of the source into the current translation unit --- changes in one of the included headers
can potentially incur a full rebuild of a large part of your codebase. Many teams spend time
optimising build time by splitting headers, avoiding includes, forward-declaring, you name it... I
wanted to build something that helped us view this web of includes at a high level.

A disclaimer up front: this is a **proof of concept**, not a production analyser --- it has bugs,
and doesn't cover all edge cases by any means. I'll be precise later about where it cuts corners and
what a serious version would need to do. The goal here was to get from "a directory full of C & C++"
to "a picture I can poke at" as directly as possible.

## What the tool does

The [tool](https://github.com/luizmugnaini/poc-include-graph) is split into two independent
pieces connected by a database:

- An extractor that walks a source tree, parses every `#include` it finds, and records the
  inclusion relationships into a SQLite database.
- A visualiser that loads that database and renders the include graph as an interactive 2D node-link
  diagram.

{{ figure(
    src="/img/include-graph/overview.png",
    alt="The visualiser showing the include graph of a codebase",
    position="center",
    caption="The visualiser with the full Blender codebase loaded",
    caption_style="font-weight: bold; font-style: italic;"
)}}

## The Extractor

The extractor is intentionally tiny. It uses `nftw` to recurse through the target directory (not
cross-platform at all), and for every file with a C or C++ extension it scans line by line for
`#include` directives. Each file becomes a row in a `Files` table, and each include becomes a row in
an `Edges` table:

```sql
CREATE TABLE Files (id INTEGER PRIMARY KEY AUTOINCREMENT, path TEXT UNIQUE);
CREATE TABLE Edges (source_id INTEGER, target_id INTEGER, UNIQUE(source_id, target_id));
```

An edge points from the includer to the included file: `source_id` includes `target_id`. This
directionality is very important for the actual analysis of the generated include graph.

Resolving an include path to an actual file on disk is unfortunately difficult due to the way the
compiler resolves includes. Since you can pass include directories to the compiler driver, for an
accurate search we would either need to integrate with the build system or make the user explicitly
give us the include directories and order of resolution --- which is not a good UX. Not to mention
includes with file paths defined via macros... looking at you FreeType!

Since this is a proof of concept, we simply do a best-effort resolution with full and relative path
analysis, which is not enough for solving the problem --- but can be a good start.

> For this exact reason, take the analysis in this blog post with a grain of salt. We're aware that
> this is only a crude approximation, which can actually be quite good or bad depending on how
> you manage your includes.

## The Visualiser

For the visualisation of the generated data, the UI code queries the database in order to build an
in-memory graph and lays it out with a force-directed simulation: nodes repel each other, edges act
as springs, and after a fixed number of steps the layout settles and the view fits itself to the
result. From there, you have an interactive exploration tool in your hands.

The features that matter for analysis are:

- **A global stats panel.** Rankings for the whole codebase by metrics such as: most
  included, most includes, highest rebuild impact, heaviest to compile, and caught cycles.
  {{ figure(
      src="/img/include-graph/graph_stats.png",
      alt="A window showing the global statistics of the include graph.",
      position="center",
      caption="The globally computed statistics over the entire Blender codebase",
      caption_style="font-weight: bold; font-style: italic;"
  )}}
- **Selection with dependency highlighting.** Click a file and the tool runs a breadth-first search
  in both directions --- over the files it includes (its descendants) and the files that include it
  (its ancestors) --- fading the colour with distance so you can see how far the ripple reaches.
  {{ figure(
      src="/img/include-graph/selection.png",
      alt="A selected node with its dependency ancestors and descendants highlighted",
      position="center",
      caption="Selecting a node highlights all of its related files. In yellow we have files that
        the selected node includes, while in green we have the files that include the node.",
      caption_style="font-weight: bold; font-style: italic;"
  )}}
- **Rebuild impact.** For a selected header, how many files end up depending on it transitively?
  This is the count of its transitive includers: the set of translation units that would potentially
  be recompiled if that header changed.

  This is the most invisible problem in C and C++ codebases since copy-paste imports result in
  transitive inclusions which will affect your compilation times even if you don't know what you're
  implicitly including. Since standard headers are not commonly modified by the users, we
  deliberately filtered STL and C lib headers from the ranking.
- **Compile weight.** For a given source file, how many distinct headers does it pull in
  transitively? This gives you a very rough score for how expensive that translation unit is to
  compile. This is clearly not perfect for numerous reasons but may give you a good place to start
  your own investigation.
- **Cycle detection.** Include cycles are usually a smell (and, without include guards, a compile
  error). The tool finds the strongly connected components of the graph and flags any with more than
  one member. Some cycles are false-positives, one such case being inline implementations, e.g. `.inl`
  files.

{{ figure(
    src="/img/include-graph/selection_info.png",
    alt="A selected node information window with searchable results",
    position="center",
    caption="You can inspect the information about a node in a searchable view. The figure shows the
    results for `source/blender/bmesh/bmesh.hh`.",
    caption_style="font-weight: bold; font-style: italic;"
)}}

## Pointing it at Blender

A toy graph proves nothing, so I ran the extractor over Blender's source tree. Blender is a great
target: it is large, old, written by many hands, and organised into a variety of subsystems.

I scoped the analysis to Blender's own code, the `source/` and `intern/` trees, and deliberately
left out `extern/` --- which is vendored third-party libraries that say more about those projects than
about Blender itself. After extraction, the graph for Blender's own code holds ~8,800 nodes
connected by ~58,400 include edges.

### Detected cycles

Out of 8,813 files, the graph flagged 12 inclusion cycles, touching 29 files in total. Everything
else is a clean directed acyclic graph.

The cycles that do exist are small and localised: a six-header cluster in the `mathutils` Python
bindings, a three-header knot in `bmesh`, several `.cpp`/`.h` and `*_inline.hh` implementation pairs
(in `slim`, the subdivision and multi-resolution code, the dependency graph node factory, and
Cycles' renderer image utilities), an SDL window/system pair in the GHOST windowing layer, and one
in the draw engine's image code. None of them are the kind of tangled mess that include cycles can
become --- in fact, most of these cycles are completely benign and are part of a deliberate
inclusion pattern. The specific cases where the compilation is only saved by a `#pragma once` are:

- `GHOST_WindowSDL.hh` and `GHOST_SystemSDL.hh` --- in the windowing system.
- `image_state.hh` and `image_private.hh` --- in the draw image engine.

This goes to show that the cycle detection, although not entirely reliable in its current
implementation, can surface real issues with the include graph of a codebase.

### The load-bearing headers

Ranking files by how many other files include them directly produces a good view of Blender's
architecture:

| Header | Direct includers | Subsystem |
|---|---:|---|
| `MEM_guardedalloc.h` | 1005 | guarded memory allocator |
| `BLI_listbase.h` | 813 | BlenLib (core utilities) |
| `BLI_utildefines.h` | 725 | BlenLib |
| `BKE_context.hh` | 670 | Blender Kernel |
| `RNA_access.hh` | 634 | RNA runtime data API |
| `WM_api.hh` | 622 | window manager |
| `BLI_math_vector.h` | 600 | BlenLib math |
| `BLT_translation.hh` | 599 | translation |
| `WM_types.hh` | 545 | window manager |
| `DNA_object_types.h` | 534 | DNA data model |

If you know Blender, the prefixes are a map of the whole system. The convention is consistent enough
that the include graph alone tells you how the project is layered:

- **MEM**: memory management system.
- **BLI**: the standard-library-like utility layer (`blenlib`).
- **DNA**: the data model of the structs that get written to `.blend` files (`makesdna`).
- **BKE**: the core data operations (`blenkernel`).
- **RNA**: the runtime reflection/data-access layer (`makesrna`).
- **WM**, **ED** and **UI**: window manager, editors, and interface.
- **BLT**: translation (`blentranslation`).

Having `MEM_guardedalloc.h` sitting at the top of the inclusion hierarchy is expected --- it is the
header for Blender's custom allocator, and almost everything in the program allocates memory through
it.

### Rebuild impact

Direct includers only tell you the first and easiest part of the inclusion analysis. The question a
developer actually cares about is the transitive one: "if I change this header, how much of the
codebase have I just invalidated?"

| Header | Transitive impact | Share of the graph |
|---|---:|---:|
| `BLI_assert.h` | 4446 | ~50% |
| `BLI_sys_types.h` | 4393 | ~50% |
| `BLI_utildefines_variadic.h` | 4252 | ~48% |
| `BLI_compiler_typecheck.h` | 4251 | ~48% |
| `BLI_compiler_compat.h` | 4181 | ~47% |

The headers with the widest reach are the low-level BlenLib primitives --- such as assertions,
integer types, compiler shims --- coming from `BLI_utildefines.h`. Since `BLI_utildefines.h` is
itself transitively included by nearly half the tree (~4,170 files, ~47%), everything it depends on
inherits that reach: a single edit to `BLI_assert.h` or `BLI_sys_types.h` lands in the transitive
include path of roughly 4,400 files, almost exactly half of the graph. Even `MEM_guardedalloc.h`
sits at ~46%. Fortunately, at this point in the life-cycle of the Blender codebase, a change in
these foundational headers is rarely needed.

This is the payoff of the whole exercise. The numbers put a concrete, defensible figure on
intuitions that are otherwise folklore ("don't casually touch `BLI_utildefines.h`"), and they point
right at the headers where techniques like the pimpl idiom, forward declarations, or splitting a
large header into smaller ones would pay off the most.

{{ figure(
    src="/img/include-graph/blender_stats.png",
    alt="The global statistics panel ranking files by rebuild impact and compile weight",
    position="center",
    caption="Global statistics of the internal source code of the Blender codebase.",
    caption_style="font-weight: bold; font-style: italic;"
)}}

### Compile weight: the expensive translation units

Walking the edges the other way around ranks the files that pull in the most headers, and therefore
tend to cost the most every time they are compiled:

| Translation unit | Transitive includes |
|---|---:|
| `draw/engines/overlay/overlay_instance.cc` | 450 |
| `draw/engines/overlay/overlay_engine.cc` | 448 |
| `draw/engines/select/select_instance.cc` | 448 |
| `draw/intern/draw_context.cc` | 369 |
| `editors/geometry/node_group_operator.cc` | 369 |

The draw engine's overlay and select passes dominate, each dragging in several hundred headers. If
you wanted to attack Blender's build times, this list is where you would start looking for
unnecessary includes to prune. It goes without saying that this is a rough estimation of compilation
effort and only provides a crude first tip where you should go looking for the heavy sources.

### A leak the graph points you at

Combining the rankings with a quick look at the files themselves turns up exactly the sort of issue
this tool is meant to surface. The `FreestyleConfig.h` configuration header for the Freestyle
renderer is transitively included by 481 files --- almost the entire Freestyle subsystem. Its
contents include:

```cpp
#include <string>

// ...

using namespace std;

namespace Freestyle {
namespace Config {
#ifdef WIN32
static const string DIR_SEP("\\");
// ...
```

Two bad things compound here. First, the `using namespace std;` sits at global scope inside a
header, so each one of the ~480 files that directly or transitively includes this header silently
gets the whole `std` namespace pulled into its global scope. Second, the header
unconditionally drags in `<string>`, one of the heavy-weight STL headers, spreading that compile
cost across the entire subsystem --- whether a given file touches `std::string` or not.

Neither problem is visible from any single file. You can only notice it once you see that one tiny
configuration header sits upstream of hundreds of others --- which is exactly the view the tool
hands you.

## Closing thoughts

The best-effort include path resolution we take is fundamentally approximate. Base-name matching can
fragment a header into more than one node, or merge two distinct files that happen to share a
name. The percentages above are best read as *order of magnitude*, not as exact recompilation
counts.

A real development tool would have to close that gap by talking to the build system. The clean way is
to consume the `compile_commands.json` compilation database that CMake (and others) can emit: it
records the exact flags, including every `-I`, for every translation unit. With that in hand the tool
could resolve each include the way the compiler actually does, respecting search-path order, and the
graph would go from "a faithful sketch" to "ground truth". That is the obvious next step, and the
line between this proof of concept and something genuinely useful for steering a large codebase.

What I like about this project is how little it takes to turn an invisible, implicit structure into
something you can look at and reason about. A few hundred lines to scrape the includes and a simple
UI to draw it --- and suddenly a twenty-year-old project with ~10,000 files will tell you, at a
glance, where its heavy parts and foundational headers are, and where to start investigating the
codebase.

Even as a proof of concept, the Blender numbers were enough to make me interested in the premise: a
cheap structural map, even with not-entirely-reliable results, can be a useful tool to have
when you are staring down a codebase far too large to hold in your head. You can now easily estimate
how much of an impact a given change to a file will have on the entire codebase.
