+++
title = "patch-hub: Linux Kernel Patch Development in the Terminal"
date = 2026-06-10
description = "Adding a single-patch apply feature to a TUI for browsing the Linux kernel mailing lists"

[taxonomies]
tags = ["programming", "linux", "kernel", "rust"]
+++

As part of my journey through the [MAC5856 Open Source Development class](/mac5856), I started
spending some time on [lore.kernel.org](https://lore.kernel.org) --- the public archive of the Linux
kernel mailing lists. If you've ever tried to follow kernel development, you know the ritual: find a
patchset, read the thread in your browser, save the raw message somewhere, then drop into a terminal
to actually apply it to a local tree --- or at least that's how I did things. It's a lot of context
switching for what is, conceptually, a very simple process.

[`patch-rub`](https://github.com/kworkflow/patch-hub) is an attempt at collapsing that loop into a
single terminal application. This post covers what the tool is, the new patch-apply feature I just
implemented, and how it's laid internally.

## What patch-hub is

`patch-hub` is a terminal user interface (TUI), written in Rust, for browsing kernel mailing list
patchsets. Rather than reimplementing the machinery of talking to a public-inbox archive, it wraps
[`lei`](https://public-inbox.org/) --- the "local email interface" CLI from the public-inbox
project.  `lei` does the job querying the lore website and fetching threads, while `patch-hub`
drives it, parses its JSON output, and renders the results as a keyboard-navigable list of threads.

A program that is similar in spirit to `patch-hub` is [`b4`](https://github.com/mricon/b4), which is
a tool to help with email-based patch workflows and is backed by the Linux Fondation.

The day-to-day workflow looks like this:

- `patch-hub` issues a [Xapian](https://xapian.org/) query via `lei` in order to fetch messages from
  the Linux Kernel lore website.
- The matching threads are parsed and shown in a list view for the user to browse.
- You navigate the list, open a thread, and step through the individual messages of a patchset.

Architecturally, the application is built around an async event loop with a small set of components
that talk to each other exclusively through an `Action` message bus --- they never call each other
directly. A background task multiplexes terminal events and timer ticks into a channel; the main
loop drains that channel, turns events into actions, dispatches to every component, and
re-renders. This matters a lot for the feature below, because applying a patch is fundamentally an
asynchronous, multi-stage operation, and the action bus is a good approach for coordinating it.

## The current state of patch-hub and its cousin, pet-rub

Currently, `patch-hub` is passing through an extensive rewrite of its internal systems and we're
considering an architecture that better aligns with the asynchronous actions required to drive this
kind of patch-based workflow. As part of the rewrite, we're testing a new implementation of the core
loop based on the default [ratatui](https://ratatui.rs/) template. This rewrite is being maintained
in a project called [`pet-rub`](https://github.com/davidbtadokoro/pet-rub). All of the
implementation below describes my new
[pull-request](https://github.com/davidbtadokoro/pet-rub/pull/1) to `pet-rub` implementing the
patch-apply feature to the codebase.

## The new feature: apply a patch with one keystroke

The piece that was always missing: actually applying a patch to a local development copy of the
kernel tree. Up to now, `pet-rub` could show you a patch, but you still had to leave the tool to do
anything with it. The new feature closes that gap.

You opt in by pointing `pet-rub` at a local kernel git repository:

```sh
pet-rub --kernel-tree <path-to-a-kernel-tree>
```

With a tree configured, the workflow becomes:

1. Browse to a thread and step to the message you want.
2. Press `a` to issue an "apply patch" action.
3. `pet-rub` downloads the raw patch from lore and runs `git am` against your tree.
4. A small status popup appears in the top-right corner: a spinner while applying, then a green
   confirmation showing the subject line of the newly created commit.

If the patch doesn't apply cleanly --- which, with kernel patches against a moving tree, happens
constantly --- the popup turns into a red, double-bordered error box showing `git am`'s error
message. From there you can press `x` to run `git am --abort`, which rolls the tree back to a clean
state so a failed apply never leaves your repository mid-rebase.

The whole component is gated behind the CLI flag `--kernel-tree`: if you don't pass it, the apply
machinery isn't even instantiated, and `pet-rub` stays a pure read-only browser for the lore patches.

## How it's implemented

The implementation is split across two components and the action bus that connects them.

### `Ktree` state machine

The `Ktree` component owns a `LocalMode` enum that captures the current state of the kernel tree
patch application process:

```rust
pub enum LocalMode {
    Idle,
    Applying,
    Success,
    Failed,
    Aborting,
}
```

When it receives a `KtreeApply` action carrying the path to a downloaded `.mbox` file via the
`Patchset` component, it transitions to `Applying` and spawns a [Tokio](https://tokio.rs/) task that
runs `git am` in the kernel tree.

The result of the patch application is reported back via a `KtreeStatsu` over the same action bus
rather than mutating state directly from the task.

On success, the task runs a follow-up `git log -1 --format=%s` to grab the subject of the commit
`git am` just created, and sends `KtreeResult(KtreeStatus::Success(subject))`. On failure it
captures the standard error stream messages and sends
`KtreeResult(KtreeStatus::Failed(stderr))`. The component's `update()` handler catches these and
flips the `Ktree` component `local_mode` accordingly, which is what drives the component rendering
code. Afterwards, in both cases the temporarily downloaded `.mbox` file is deleted.

The abort path is symmetric: pressing `x` while in `Failed` mode emits a `KtreeAbort` action, which
spawns a task running `git am --abort` and reports back `KtreeStatus::Aborted`. The success
notification auto-dismisses after a fixed number of ticks, so the popup doesn't pollute the UI
forever.

The rendering is deliberately unobtrusive. When the component is `Idle` with no message, `draw()`
returns early and paints nothing at all. Otherwise it computes a popup sized to roughly half the
width, anchors it in the top-right corner, clears the region underneath, and renders a bordered
paragraph --- rounded yellow/green borders for in-progress and success, a double red border for the
error state.

### Downloading the patch in the `Patchsets` component

The `Patchsets` component is where you actually navigate threads, so that's where the `a` keypress
lives. The tricky part is that the patch has to be fetched from lore *before* it can be handed to
`git am`. I kept this asynchronous to avoid blocking the UI while the user waits for the download to
complete.

When you press `a` on a message, the component looks up the message ID of the currently selected
thread entry and kicks off a download task. It immediately tells the `Ktree` component to show its
`Applying` state, then fetches the raw message with `curl` to a temporary file.

The lore website exposes every message at a stable `/all/{message_id}/raw` URL, which has the mbox
format, which `git am` supports by default. On a successful download the task fires a
`KtreeApply(tmp_path)` --- handing control over to the `Ktree` component described above. If the
download itself fails, it short-circuits straight to a `KtreeResult(Failed(...))` so the error
surfaces the same way an apply error would.

A small `applying` guard flag prevents you from firing off a second apply while one is already in
flight; it's cleared when the `KtreeResult` action comes back through the bus.

This operation could've been done via `lei` just like we fetch the patchsets. However, given that
the project, as far as we know, is largely abandoned, we may want to pivot to our own
implementation. This new feature already follows this idea and also goes to show how relatively easy
it may be to replace the usage of `lei`.

### Tying it together

The remaining changes are plumbing:

- The `Action` enum gained `KtreeSetMode`, `KtreeAbort`, and `KtreeResult` variants alongside the
  existing `KtreeApply`.
- A new `--kernel-tree <path>` CLI argument flows from `main` into `App::new`, which only pushes the
  `Ktree` component onto the component vector when a path is actually provided.
- I extracted the spinner animation frames into a shared `SPINNER` constant in `components.rs`, since
  both the `lei` fetch indicator and the new apply indicator use it.

### Tests

Because `git am` behaviour is the whole point, I added integration tests with a real patch fixture
(`tests/fixtures/single_patch.mbox`). They spin up throwaway git repositories in a temp directory and
exercise the two paths that matter: a clean apply that produces the expected commit subject and file,
and a deliberate conflict followed by `git am --abort` that confirms the tree returns to a clean
state (no leftover `.git/rebase-apply`). These run the actual `git` binary rather than mocking it,
which is the only way I'd trust a test of this feature.

## Closing thoughts

What I like about this feature is how little new conceptual surface it added. The component +
action-bus design that was already in place did most of the work --- a patch apply is just a
sequence of actions (`KtreeApply` → `KtreeResult`) flowing between two components, with Tokio tasks
handling the blocking `curl` and `git` calls off the main loop. The hard part wasn't the
concurrency; it was deciding what the *smallest* useful version of "apply a patch" looks like, and
resisting the urge to build the whole kernel-workflow kitchen sink before shipping anything.

The obvious next step is going beyond a single patch --- applying an entire patchset in order, and
eventually wiring up a build step so you can go from "interesting thread on lore" to "booted kernel"
without leaving the TUI.
