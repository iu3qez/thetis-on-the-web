# CLAUDE.md

Thetis On The Web (TOTW) is the browser client of a remote station that runs deskHPSDR, spoken to over TCI.

## Read first

- `STRATEGY.md` decides what is in scope. It works only with deskHPSDR, targets a laptop browser, and keeps audio and Morse out of the client.
- `CONCEPTS.md` holds the project vocabulary: implementation unit, bench verification, portable and split variant, view window, zero cost.
- The newest file in `docs/handoffs/` is the current state of the work; the plan it names in `docs/plans/` defines the units.
- `docs/bench/reference-checklist.md` is the bench baseline that removals and migrations are compared against.
- `docs/solutions/` records learnings worth reusing.

## The client

- `totw.html` is the source and the portable variant: one self-contained file.
- `website/` is generated from it. Never edit it by hand: the next regeneration erases the change.
- After every change to `totw.html`, run both, and commit `website/` with `totw.html`:

  ```bash
  node scripts/build-variants.js split
  node scripts/smoke-test.js
  ```

## The server

- deskHPSDR lives in the fork `iu3qez/deskhpsdr`; a local checkout sits at `/Users/sf/Developer/deskhpsdr` (machine-local).
- Its source is authoritative on TCI semantics. Read `src/tci.c` before assuming how a command behaves; several of its commands differ from Thetis and from the TCI standard.
- A server defect found from the client becomes an issue in `iu3qez/deskhpsdr`, one per defect, in English, with the sections Symptom, Cause, Proposal and Context, source line references and the commit they refer to. Issues up to #20 are in Italian.

## Git and GitHub

- `main` is the trunk. Work happens on branches and reaches `main` through a PR.
- This clone has an `upstream` remote pointing to the original project and no default repository: always pass `-R iu3qez/thetis-on-the-web` to `gh`. The deskHPSDR clone has a `_pi` remote to piHPSDR: pass `-R iu3qez/deskhpsdr` there.
- Never reference the original project: no links, and no `owner/repo#N` in plain text in commits, PRs or issues, since GitHub turns it into a visible event on that repository.

## Station and bench

- The station runs deskHPSDR on `ubuntu.lan`, SSH alias `ubuntu`, TCI on port 50001. The command sandbox blocks the LAN: reach the station through the SSH tool.
- Bench runs are made by the operator, on the latest deskHPSDR build only, with TX on a dummy load. Results from an older build do not go into the checklist.
- Never connect the in-app browser to the real station while the operator is at the bench. Test against a fake WebSocket that answers as deskHPSDR's source does.
- Browsers serve a stale `totw.html` after a change: reload with Cmd+Shift+R or add a `?v=` parameter.
