# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A tool for merging pairs of duplicate or related Anki notes, so that one note
holds the combined information and the spare note can be tagged or deleted. It is
the "merge notes" item under *anki addons* in `../../project-ideas_Anki.md`, and
this repo is nested inside the `Anki-projects` umbrella. The umbrella's
`CLAUDE.md` and `SAFETY.md` apply here too.

## Status: design only

There is no code, no build/lint/test setup, and no commits yet. Don't invent
commands. The working documents are:

- `design-draft.md`: the design scratchsheet. It covers use cases, how duplicates
  get paired, the merge screen, and what happens to each note afterwards.
- `images/merge-screen-mockup.jpg`: Kat's original notebook sketch of the
  merge screen. The table in `design-draft.md` transcribes it, and the sketch is
  the source of truth. If the two disagree, ask before changing either one.

Treat the open questions in the draft as Kat's decisions to make. Bring
trade-offs to her instead of settling them in code. Two questions are still open:
how a "match" is defined (one field, several fields with AND/OR, regex, or a
custom script), and whether scheduling or note type decides which note is kept.

## The design in brief

The pipeline has three stages, and each should be runnable on its own:

1. **Pair notes.** Search either two pools (find B-notes that match A-notes) or a
   single pool (find duplicates within it).
2. **Merge each pair.** Build one row per Note A field. Fields with the same name
   match automatically, and the rest fill down in order, like Anki's Change Note
   Type. One B field may feed several result rows, so the mapping is not
   one-to-one. Each row is combined as `A`, `A + B`, or `B + A`. Scheduling uses
   "longest gap". The merge also adds extra tags to both notes for later actions.
3. **Act on the results.** The merged note may be unsuspended or moved to another
   deck. The spare note is tagged or deleted.

## Anki facts that constrain the design

- Scheduling belongs to **cards**, not notes. "Keeping a note's scheduling" means
  keeping that note and its cards and writing the merged fields into it. Deleting
  the spare note deletes its cards and their review history. Actually copying
  review state from one card to another is a separate, harder problem, and the
  draft already says so.
- The merge never changes a notetype. Note A is the note being modified, so its
  notetype defines the result's fields, and Note B is only read from. In the
  resource-deck use case, A is the main-deck note and B is the resource-deck note
  of a different type. The merge is ordinary field writes to A, with no schema
  change and no forced sync.
- The one case that would need a notetype change: you want to keep B's cards
  for their scheduling, but you want A's notetype. Changing B's notetype is a
  schema change that forces a one-way sync (see "Never" in `SAFETY.md`). This is
  the "scheduling *or* note type decides Note A" trade-off in the draft. Don't
  build a notetype change into the merge path without Kat deciding it.
- Matching on field text has to deal with furigana syntax (` 漢字[かんじ]`), HTML,
  and `&nbsp;`. Raw string equality will miss real duplicates. Normalisation
  should be its own swappable function.
- The merge screen is a UI, so the write path is probably an addon using
  `CollectionOp`. Pairing and dry runs could still be a standalone script that
  uses the `anki` library or AnkiConnect. Nothing has been decided.

## Rules from `SAFETY.md` that matter most here

Merging writes fields and may delete notes, so every run needs all of these:

- A backup.
- A dry run against the `test` profile that prints the per-pair result.
- Writes only to notes whose content actually changed.
- Tagging of processed pairs, so that reruns are incremental.

All writes go through Anki's library and never through raw SQL.
