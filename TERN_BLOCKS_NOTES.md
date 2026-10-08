# WIP: show Tern's non-terminal blocks in Collie

Parked 2026-10-08. Nothing implemented yet; these are the probe findings (tern 0.6.2 `4b3ed42`, Linux).

## Problem

`tern ls --json` lists native blocks (file, browser, sqlite, notebook, folder) and plugin blocks
next to terminals. The adapter maps every one to a pane, but:

- native blocks report `live: false`, `program: ""`, `title: ""`, `cols/rows: 0`, so they show as
  dead, nameless shell panes;
- `tern capture <id>` answers `no such pane in this session daemon` for them, so opening one in
  Collie gives a 502 (`GET /api/pane/:id`);
- plugin blocks (e.g. `cliproxy.usage`) are `live: true`, but `capture` / `--ansi` return empty.
  Only `capture --surfaces` returns their text.

## What `ls --json` adds per kind

| Block | Extra key on the block | Example |
| --- | --- | --- |
| file / image / board / folder | `file` | `/tmp/x.md`, `/tmp/dir` |
| browser | `browser` (+ `pip: {owner, corner, stashed}` when floated) | `https://example.com/` |
| sqlite | `sqlite` | `/tmp/t.db` |
| notebook | `notebook` | `/tmp/n.ipynb` |
| plugin | none; `program` is `<plugin>.<block>` | `cliproxy.usage` |

## Ways to read them

- Plugin: `tern capture <id> --surfaces` gives the plain text (~instant).
- Browser: `tern browser '{"op":"snapshot","block":ID,"mode":"text","format":"text"}'` gives the
  URL, title and page text (~80 ms). `{"op":"state",...}` gives URL/title/loading.
- File-like: no CLI read. The plugin API has `cx.session:read(pane)` (window only), not the CLI.
  The Collie side could point at the existing Files view (ADR 0083) instead of reading the file.
- `tern ls --json` takes ~15 ms, so mapping kinds there costs nothing extra.

## Other observations

- `tern send <id> keys/text` to a file or browser block exits 0 (did nothing visible to the file).
- `tern close` / `tern focus` work on native blocks.
- `tern events` emits only `layout_changed` when a block is opened with `tern open`.
- `tern new session|tab --json` prints `{session, tab, block}`, so `createTab`/`createSpace` could
  use those ids instead of comparing snapshots.

## Rough plan

1. `protocol.ts`: type the extra keys and classify a block's kind.
2. `adapter.ts`: native blocks count as alive and get a title (file name, URL, kind); `readGrid`
   picks a read by kind (capture, `--surfaces`, browser snapshot, or a one-line summary).
3. `watch.ts`: add the kind keys to the census signature.
4. Fixture plus tests, then `bun run typecheck`, `bun run lint`, `bun test ./bridge`.
5. Fork PR to AltanS/collie with no version bump and no CHANGELOG line (CLAUDE.md).
