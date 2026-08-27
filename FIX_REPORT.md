# [Bug] KOReader → web reader progress bridge fails for indexed text-node XPointers (`text()[k].N`)

**Version:** v2.3.1 (docker, `ghcr.io/booklore-app/booklore:latest`, image digest from 2026-06-27)
**Area:** KOReader progress sync → BookLore reader position ("Sync progress with BookLore reader")
**Environment:** Linux host, docker compose, MariaDB 11.4.5; KOReader v2026.07.1 on PocketBook (also reproducible with plain `curl` against the kosync API — see below)

> Note: I have a minimal validated fix for this (one regex + one normalization line, tests included) and would like to submit a PR once this issue is approved.

## Summary

When a KOReader device pushes progress whose XPointer ends in an **indexed text node with a character offset** — e.g. `/text()[1].247` — the XPointer→CFI conversion throws, and the BookLore web reader position is never updated. Device↔device sync is unaffected (raw string stored fine); only the bridge to the built-in reader breaks.

KOReader emits `text().N` when a paragraph is a single text node, but `text()[k].N` whenever the paragraph contains inline markup (italics, links, spans). So the bridge works on plain paragraphs and silently fails on formatted ones — which makes the feature look intermittently broken.

## Reproduction

1. Enable KOReader sync and "sync progress with BookLore reader" for a user.
2. Push progress via the kosync API with an indexed text-node position:

```
PUT /api/koreader/syncs/progress
{
  "document": "<book hash>",
  "progress": "/body/DocFragment[15]/body/p[205]/text()[1].247",
  "percentage": 0.8959,
  "device": "x", "device_id": "y"
}
```

3. Log shows:

```
WARN  o.b.service.koreader.KoreaderService : Failed to convert xpointer to CFI: Invalid XPointer segment: text()[1].247
```

4. `user_book_progress.koreader_progress` is saved, but `epub_progress` (web reader position) is not updated.

## Root cause (verified against v2.3.1 bytecode)

In `org.booklore.util.koreader.CfiConvertor`:

- `XPOINTER_TEXT_OFFSET_PATTERN = /text\(\)\.(\d+)$` only matches the un-indexed form, so for `text()[1].247` no offset is extracted and the trailing step is not stripped from the element path.
- The leftover segment `text()[1].247` then reaches `resolveXPointerPath`, where it matches neither `XPOINTER_SEGMENT_WITH_INDEX_PATTERN = ^(\w+)\[(\d+)\]$` (`\w+` cannot match `text()`) nor `^(\w+)$`, producing `IllegalArgumentException("Invalid XPointer segment: ...")`.

## Suggested fix

Accept an optional index in the offset pattern:

```java
private static final Pattern XPOINTER_TEXT_OFFSET_PATTERN =
    Pattern.compile("/text\\(\\)(?:\\[\\d+\\])?\\.(\\d+)$");
```

and, defensively, strip any remaining trailing `text()` step (e.g. a bare `/text()[2]` with no offset) from the element path in `parseXPointer` before resolution — `TRAILING_TEXT_OFFSET_PATTERN` already matches it:

```java
elementPath = TRAILING_TEXT_OFFSET_PATTERN.matcher(elementPath).replaceAll("");
```

I patched exactly this locally against v2.3.1 and re-sent the failing payload; conversion now succeeds:

```
INFO o.b.service.koreader.KoreaderService : Converted xpointer to CFI for BookLore reader sync: epubcfi(/6/30!/4/410/1:247)
```

and the web reader opens at the pushed position.

Note: the produced CFI still uses the `/1:` first-text-node step, so for `text()[k]` with k>1 the offset lands relative to the first text node — close to, but not exactly, the pushed character. That refinement (mapping the k-th text node to its CFI step, or summing preceding sibling text lengths) would make it exact, but the parse fix above is what stops the hard failure.
