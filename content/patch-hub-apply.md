+++
title = "Contributing to patch-hub: Linux Kernel Development Workflow in the Terminal"
date = 2026-06-10
description = "Adding a single-patch apply feature to a TUI for browsing the Linux kernel mailing lists"

[taxonomies]
tags = ["programming", "linux", "kernel", "rust"]
+++

As part of [MAC5856 Open Source Development class](/mac5856), I started spending some time on
[lore.kernel.org](https://lore.kernel.org), the public archive of the Linux kernel mailing
lists. The kernel development workflow normally follows these steps: find a patchset, read the
thread in your browser or mail client, save the raw message somewhere, then drop into a terminal to
actually apply it to a local tree --- or at least that's how I did things. This entails a lot of
context switching for what is, conceptually, a very simple process.

[`patch-hub`](https://github.com/kworkflow/patch-hub) is an attempt at simplifying that iteration
loop into a single terminal application. This post covers what the tool is, the new patch-apply
feature I just implemented, and how it's laid internally.

## What is patch-hub

`patch-hub` is a terminal user interface, written in Rust, for browsing kernel mailing list
patchsets. Rather than reimplementing the machinery of talking to a public-inbox archive, it wraps
[`lei`](https://public-inbox.org/) --- the "local email interface" CLI from the public-inbox
project.  `lei` does the job querying the lore website and fetching threads, while `patch-hub` works
as the driver: parsing its JSON output, and rendering the results as a keyboard-navigable list of
threads.

A program that is similar in spirit to `patch-hub` is [`b4`](https://github.com/mricon/b4), which is
a tool to help with email-based patch workflows and is backed by the [Linux Foundation](https://www.linuxfoundation.org/).

The day-to-day workflow in `patch-hub` looks like this:

- A [Xapian](https://xapian.org/) query is issued via `lei` in order to fetch messages from the lore
  website.
- The matching threads are parsed and shown in a list view for the user to browse.
- You navigate the list, open a thread, and step through the individual messages of a patchset.

## The cousin, pet-rub

Currently, `patch-hub` is passing through an extensive rewrite of its internal systems and we're
considering an architecture that better aligns with the asynchronous actions required to drive this
kind of patch-based workflow. As part of the rewrite, we're testing a new implementation of the core
loop based on the default [ratatui](https://ratatui.rs/) template. This rewrite is being maintained
in a cousin project called [`pet-rub`](https://github.com/davidbtadokoro/pet-rub).

Architecturally, the new version application is built around an async event loop with a small set of
components that talk to each other exclusively through an `Action` message bus --- they never call
each other directly. A background task multiplexes terminal events and timer ticks into a channel;
the main loop drains that channel, turns events into actions, dispatches to every component, and
re-renders. This matters a lot for the feature below, because applying a patch is fundamentally an
asynchronous, multi-stage operation, and the action bus is a good approach for coordinating it.

Below, I'll describe the [work I've done](https://github.com/davidbtadokoro/pet-rub/pull/1) for a
new feature to the `pet-rub` codebase.

## The new feature: apply a patch with one keystroke

A part of the development workflow that was missing in `pet-rub` was actually applying a given patch
to a local copy of the kernel tree. Up to now, `pet-rub` could show you a patch, but you still had
to leave the tool to do anything with it.

For the new feature to work, you opt in by pointing `pet-rub` to a local kernel git repository:

```sh
pet-rub --kernel-tree [path-to-a-kernel-tree]
```

The new `--kernel-tree <path>` CLI argument flows from `main` into `App::new`, which only pushes the
new `Ktree` component onto the component vector when a path is actually provided. This is
intentional and makes the new feature optional for the user.

With a tree configured, the workflow becomes:

0. Fetch the mailing list content.
1. Browse to a thread and step to the message you want.
2. Press `a` to issue an "apply patch" action.
3. `pet-rub` downloads the raw patch from lore and runs `git am` against your tree.
4. A small status popup appears in the top-right corner: a spinner while applying, then a
   confirmation showing the subject line of the newly created commit.

If the patch doesn't apply cleanly --- which, with kernel patches against a moving tree, happens
constantly --- the popup displays the `git am`'s actual error message so that the developer can act
upon it. From there you can press `x` to run `git am --abort`, which rolls the tree back to a clean
state so a failed apply never leaves your repository mid-rebase.

The whole component is gated behind the CLI flag `--kernel-tree`: if you don't pass it, the apply
machinery isn't even instantiated, and `pet-rub` stays a pure read-only browser for the lore patches.

## How it's implemented

The implementation is split across two components and the action bus is reponsible for connecting
them.

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

The result of the patch application is reported back via a `KtreeStatus` over the same action bus
rather than mutating state directly from the task, which is necessary due to the architectural
constraints of the application.

On success, the task runs a follow-up `git log -1 --format=%s` to grab the subject of the commit
`git am` just created, and sends `KtreeResult(KtreeStatus::Success(subject))`. On failure it
captures the standard error stream messages and sends
`KtreeResult(KtreeStatus::Failed(stderr))`. The component's `update()` handler catches these and
flips the `Ktree` component `local_mode` accordingly, which is what drives the component rendering
code. Afterwards, in both cases the temporarily downloaded `.mbox` file is deleted.

The abort path is symmetric: pressing `x` while in `Failed` mode emits a `KtreeAbort` action, which
spawns a task running `git am --abort` and reports back `KtreeStatus::Aborted` upon completion. The
success notification auto-dismisses after a fixed number of ticks, so the popup doesn't pollute the
UI forever.

### Downloading the patch in the `Patchsets` component

The `Patchsets` component is where you actually navigate threads, so that's where the `a` keypress
lives. The tricky part is that the patch has to be fetched from lore *before* it can be handed to
`git am`. I kept this asynchronous to avoid blocking the UI while the user waits for the download to
complete.

When you press `a` on a message, the component looks up the message ID of the currently selected
thread entry and spawns a download task. It immediately tells the `Ktree` component to set its local
mode to the `Applying` state, then fetches the raw message with `curl` to a temporary file.

The lore website exposes every message at a stable `/all/{message_id}/raw` URL, which has the mbox
format, which `git am` supports by default. On a successful download the task fires a
`KtreeApply(tmp_path)`, handing control over to the `Ktree` component described above. If the
download itself fails, it short-circuits straight to a `KtreeResult(Failed(...))` so the error
surfaces the same way an apply error would --- and the `Ktree` draw method will be responsible for
reporting the error back to the developer.

A small `applying` guard flag prevents you from firing off a second apply while one is already in
flight, and it's cleared when the `KtreeResult` action comes back through the bus.

This operation could've been done via `lei` just like we do when we fetch the patchsets. However,
given that the `lei` project, as far as we know, is largely abandoned, we may want to pivot to our own
implementation. This new feature already follows this idea and also goes to show how relatively easy
it may be to replace the usage of `lei` in other places as well.

### Tests

Because `git am` behaviour is the whole point, I added integration tests with a patch example
(`tests/fixtures/single_patch.mbox`). The tests create a throwaway git repository in a temporary
directory and exercise the two paths that matter: a clean apply that produces the expected commit
subject and file, and a deliberate conflict followed by `git am --abort` that confirms the tree
returns to a clean state.

## Closing thoughts

What I like about this feature is how little new conceptual surface it added. The component and
action-bus design that was already in place did most of the work. A patch application is just a
sequence of actions (`KtreeApply` then `KtreeResult`) flowing between two components, with Tokio
tasks handling the `curl` and `git` calls off the main loop. The hard part wasn't the concurrency;
it was deciding what the *smallest* useful version of "apply a patch" looks like, and resisting the
urge to build the whole kernel-workflow kitchen sink before shipping anything.

The obvious next step is going beyond a single patch, applying an entire patchset in order, and
eventually wiring up a build step so you can go from a thread on lore to a booted kernel without
leaving the TUI.
