# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A tool for merging pairs of duplicate or related Anki notes, so that one note
holds the combined information and the spare note can be tagged or deleted. It is
planned as a batch tool that merges between pools by rule, released under
AGPL-3.0. It is the "merge notes" item under *anki addons* in
`../../project-ideas_Anki.md`, and this repo is nested inside the `Anki-projects`
umbrella. The umbrella's `CLAUDE.md` and `SAFETY.md` apply here too.

## Status: design only

There is no code and no build/lint/test setup yet. Don't invent commands. The
working documents are:

- `design-draft.md`: the design scratchsheet. It covers the decisions so far, use
  cases, the data model, how notes get paired, the merge screen, what happens to
  each note afterwards, scheduling, architecture, build order and prior art.
- `design-draft-original.md`: Kat's original draft, kept as a backup. Don't
  edit it.
- `images/merge-screen-mockup.jpg`: Kat's original notebook sketch of the
  merge screen. The table in `design-draft.md` transcribes it, and the sketch is
  the source of truth. If the two disagree, ask before changing either one.

Treat the open questions in the draft as Kat's decisions to make. Bring
trade-offs to her instead of settling them in code. The current list is under
Open Questions in `design-draft.md`.

## The design in brief

The pipeline has three stages, and each should be runnable on its own:

1. **Pair notes.** Pools are Anki search strings: either two pools (find B-notes
   that match A-notes) or a single pool (find duplicates within it). Notes are
   grouped by a key built from chosen fields, and ambiguous matches are reported
   instead of merged. With two pools, the pool-A note is always Note A; with one
   pool, role rules such as studied longest decide.
2. **Merge each pair.** A recipe has one row per Note A field, for one combination
   of note types. Fields with the same name match automatically, and the rest fill
   down in order, like Anki's Change Note Type. One B field may feed several result
   rows, so the mapping is not one-to-one. Each row is `A`, `B`, `A + B` or
   `B + A`, and means exactly that: `A + B` appends even when the texts overlap,
   and `B` replaces even when B is empty. A plan works out the result and writes
   nothing, which makes it the dry run, and an apply step then writes it. Every
   processed pair counts as merged: Note A gets the recipe's tags, and Note B gets
   the spare tag and its own tags.
3. **Act on the results.** The merged note may be unsuspended or moved to another
   deck. The spare note is tagged and suspended, and can be deleted later.

## Anki facts that constrain the design

- Scheduling belongs to **cards**, not notes. "Keeping a note's scheduling" means
  keeping that note and its cards and writing the merged fields into it. Deleting
  the spare note deletes its cards, but their review log rows stay in the database,
  attached to no card. In the first version, no cards move. The Scheduling section
  of the draft covers copying, moving and swapping, including what was tested.
- The merge never changes a notetype. Note A is the note being modified, so its
  notetype defines the result's fields, and Note B is only read from. In the
  resource-deck use case, A is the main-deck note and B is the resource-deck note
  of a different type. The merge is ordinary field writes to A, with no schema
  change and no forced sync.
- The one case that would need a notetype change: you want to keep B's cards
  for their scheduling, but you want A's notetype. Changing B's notetype is a
  schema change that forces a one-way sync (see "Never" in `SAFETY.md`). The first
  version avoids this by leaving each note's cards where they are. Don't build a
  notetype change into the merge path without Kat deciding it.
- Matching on field text has to deal with furigana syntax (` 漢字[かんじ]`), HTML,
  and `&nbsp;`. Normalising is a set of opt-in steps the user picks for each key
  part, and nothing is applied by default. Stripping furigana also removes the
  reading, which is what tells homographs like 上手 (じょうず/かみて) apart.
- The first version is a standalone script that uses the `anki` library with Anki
  closed. The merge screen comes later as an addon using `CollectionOp`, and reuses
  the same core. Both support Anki 25.07 and newer, so all code is written for
  Python 3.13 (older Anki bundles Python 3.9).

## Rules from `SAFETY.md` that matter most here

Merging writes fields and may delete notes, so every run needs all of these:

- A backup.
- A dry run against the `test` profile that prints the per-pair result.
- Writes only to notes whose content actually changed.
- Tagging spare notes so reruns skip them. Note A is never excluded, so a note
  mined again later still finds it. There is no "processed" tag.

All writes go through Anki's library and never through raw SQL.
