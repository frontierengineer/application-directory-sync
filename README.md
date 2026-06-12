# Directory Sync — keep directories on your machines in step

Directory Sync mirrors directories across the machines connected to your Frontier instance. You define a sync pair — a source directory on one machine and a target directory on another (or the same one) — and the extension scans the source on an interval, or on demand, and copies the changes to the target, deleting target files the source no longer has. It's the workspace-spanning way to keep a folder identical in two places without thinking about it: a checkout you edit on a workstation and run on a server, a notes directory you want present on every box, an asset folder shared between machines.

It has three halves. The `server/` half is the coordinator — it owns the pair model (in the extension's own durable Store), schedules the runs, answers the UI over the bus, and drives the worker-channel protocol that actually moves the bytes. The `worker/` half is the daemon-side engine that runs on every connected machine, right next to that machine's files: it scans, hashes, reads, writes, and deletes locally. A cross-machine pair relays source → host → target in acked chunks; a same-machine pair short-circuits to one local mirror pass on that machine's worker, with no host relay. The `ui/` half is what you see: the sidebar lists your pairs with their live status, and the view lets you create, edit, enable, and run them.

## What's in here

- `server/index.ts` — the host backend: the pair Store model, the scheduler, the bus responders (`pairs.list` / `pairs.create` / `pairs.update` / `pairs.set_enabled` / `pairs.delete` / `pairs.sync_now`), and the worker-channel driver.
- `worker/index.ts` — the daemon-side engine: the `scan` / `hash` / `read` / `write.*` / `delete` / `touch` / `mirror` wire protocol over the worker channel.
- `ui/` — the React surface: the pairs sidebar, the pair editor/status view, and the bus client.
- `messages.ts` — the shared pair model + bus surface both the server and ui import as `../messages`; the constants (default interval, default exclusions, size cap) live here.
- `extension.json` — `displayName` ("Directory Sync"), `defaultColor`, `description`. It declares no `id`: an extension's on-disk id is its install directory name. The internal naming — the engine, the Store keys, the log prefixes, the protocol — stays `dir-sync` throughout (its keys are id-relative, so the data conventions are stable regardless of the install directory name); the longer "directory-sync" name is used only for this repo and the marketplace listing.

## What v1 actually does (and what it doesn't)

This is an honest description of the shipped engine — read it before you point it at anything irreplaceable:

- **One-way, whole-file mirror.** A pair has a source and a target; changes flow source → target only. A changed file is copied in full (no block/delta transfer). Target files absent from the source are deleted — the target is made to match the source, not merged with it.
- **Size + mtime change detection.** A file is considered unchanged when its size matches and its modification time matches to the second; when the size matches but the mtime differs, a content hash breaks the tie (and the mtime is restamped) so an unchanged file isn't recopied. There is no inotify/FSEvents watch — detection happens at scan time, on the interval or on a manual run.
- **Symlinks are skipped.** They're counted (`lstat` semantics, never followed) and reported, never copied or recreated.
- **Exclusions are name-based.** An excluded name matches any path segment and makes a file invisible to BOTH sides — never copied from the source, never deleted on the target. Defaults are `.git`, `node_modules`, `.frontier-worktrees`. There is a per-file size cap (oversize files are skipped and reported).
- **No conflict handling.** It's a mirror, not a two-way reconciler: edits made only on the target are overwritten or deleted on the next run.

## Where this came from, and where it could go

The original ambition for this extension was different and worth recording as a documented future direction. It was conceived as an **append-only, multi-way sync** — latest/largest wins — built specifically to sync append-heavy, UUID-named files like the JSONL session logs under `~/.claude/projects` across machines without conflicts: each machine appends to its own monotonically-growing file, and a sync that always keeps the longest/most-recent version of a given file converges every machine to the union with no merge step and no clobbering. That's a fundamentally safer model for append-only data than the whole-file, source-wins mirror shipped here.

In practice, **Frontier slots have largely solved the cross-machine-workspace need** that motivated it — a workspace's session history lives where the work runs, reachable without copying files between boxes — so the append-only engine was never the thing that shipped. The working one-way mirror above is genuinely useful for plain directory mirroring, and that's what this extension is today. An **append-only "latest/largest wins" mode is a plausible v2** if the UUID-named-append-log use case resurfaces; it would be a new sync mode alongside the mirror, not a replacement for it. This README documents the divergence deliberately: the engine here is the mirror, not the append-only design.

## How types resolve (important for a standalone repo)

Every capability imports the host contract as `import type … from '../../types'` — the exact specifier an installed extension uses. In production the host copies the extension into `<FRONTIER_DIR>/extensions/<id>/` and writes a `types.ts` shim one level up (a sibling of every extension), so `../../types` from `server/index.ts`, `worker/index.ts`, or `ui/index.tsx` resolves to `extensions/types.ts`, and `../../../types` from a file under `ui/components/` resolves to the same place. The extension's OWN shared file, `messages.ts`, sits at the repo root and is reached one level up from a capability — `server/index.ts` imports `../messages` — note the different depth: `../../types` is the host's file two levels up, `../messages` is this extension's file one level up. This repo is a flat, standalone extension (the `extension.json` is at the root), so there is no host beside it and `../../` from a capability would point above the repo. To stay byte-identical to an installed extension, the contract is vendored at the repo root — [`types.ts`](./types.ts) (a verbatim copy of the host's `backend/extensions/types.ts`, plus a one-block header) and [`workspaceTypes.ts`](./workspaceTypes.ts) (its one dependency). The imports are type-only, so esbuild erases them from the shipped bundles — nothing vendored ends up at runtime. To keep current with the host, re-copy those two files when the API moves.

The ui also imports `EmptyState` and `usePreviewClick` from `@frontierengineer/ui`, the host's shared UI primitives. The host bundler aliases that specifier to its own frontend tree at build time, so the bytes never ship here; for the local typecheck the surface this extension uses is declared in [`hostUi.d.ts`](./hostUi.d.ts), pointed at via `paths` in the verify mirror's ui tsconfig and marked external for esbuild.

Because TypeScript and esbuild won't remap a relative specifier, `npm run verify` reproduces the production directory nesting in a throwaway `.verify/` mirror (the vendored `types.ts` as a sibling of a `dir-sync` dir that symlinks this repo) and runs the checks from there — so `../../types` resolves exactly as the host resolves it, with no edits to the source.

## Verifying

```
npm install      # dev-only: TypeScript, esbuild, and @types for the local checks
npm run verify   # typecheck (host-side server+worker + ui) against the production-nested mirror, then esbuild every entry the way the host's bundler does
```

`npm run verify` is the full gate; `npm run build:check` runs just the esbuild pass. None of this is needed to use the extension — the Frontier host builds the real bundles itself when it loads the extension; these scripts only let you confirm it compiles and bundles before you publish.

## Installing from the marketplace

Open the Extensions view in Frontier, switch to the Marketplace tab, find **Directory Sync**, and install. Because it ships server-side code (the `server/` and `worker/` halves run with host/daemon access), the install trust dialog reflects that — review and approve to install. The host fetches the published tarball, verifies its pinned hash, installs it, and the Directory Sync sidebar appears; create a pair to start mirroring.

## Publishing

Publishing is open and unreviewed-by-humans: tag a release and the marketplace indexer picks it up. See the registry's [`PUBLISHING.md`](https://github.com/frontierengineer/extensions/blob/main/PUBLISHING.md). [`.github/workflows/release.yml`](./.github/workflows/release.yml) packs the extension into `extension.tgz` (minus `.git`, `.github`, `node_modules`, `data`, and the local-only `.verify`) and attaches it to a GitHub release; the registry then scans that exact tarball, pins its sha256 into `index.json`, and it's installable from the Marketplace tab.

```
git tag v1.0.0 && git push origin v1.0.0
```
