---
artifact_contract: "ce-handoff/v1"
created_at: "2026-09-18T22:12:05Z"
title: "TOTW: client follows the deskHPSDR TCI fixes; VFO model corrected; VFO B tuned with Shift"
summary: "The client was adapted to the TCI fixes merged in deskHPSDR on 2026-09-18 and passed E1-E4, F9, F10 at the bench; the VFO A/B model was corrected with the operator; SPLIT, the TX line and Shift tuning of VFO B are done and not yet benched; deskHPSDR issues 27 and 28 block the rest of issue 15."
keywords: ["totw", "deskhpsdr", "tci", "modulation_ex", "rx_enable", "vfo-b", "split", "tx_frequency", "shift", "bench", "pr-17"]
cwd: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/recent-handoff-81c25b"
resume_focus: "Bench of C11-C14, F7, F8, F11-F13 on the current build; the panadapter contrast reported by the operator; then issue 15 as deskHPSDR issues 27 and 28 land"
repository: "iu3qez/thetis-on-the-web"
repo_root_sha: "a14fbec33d24418421b76761f4bbfe888d78bbb0"
branch: "feat/totw-modular-operable-client"
head: "a73d9bc"
worktree_path: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/recent-handoff-81c25b"
---

# Handoff: TOTW after the deskHPSDR TCI fixes, with the VFO model corrected

**Supersedes `docs/handoffs/2026-09-18-bench-fixes-vfo-b-next.md`.** Its VFO B design (an "active VFO" selected in the client) is wrong and must not be implemented; see "VFO model" below. Its notes on U1, the libwebsockets crash and the checklist method stay valid, as do those of `2026-09-16-bench-u1-lws-fix.md`.

The `head` field is the last commit before this handoff; the commit adding this file sits on top of it. Locally the work was done on `claude/server-sweep-adapt`, tracking `origin/feat/totw-modular-operable-client`, and pushed with `git push origin HEAD:feat/totw-modular-operable-client`.

This handoff is in English, like `STRATEGY.md`, `CONCEPTS.md` and `CLAUDE.md` since 2026-09-18. The older handoffs and the bench checklist are in Italian; rows added to the checklist stay in Italian to match it.

## Operator intent and decisions

Stated by the operator in this session:

- **VFO model, confirmed**: it is deskHPSDR's. VFO B is not tied to RX2 and exists with one receiver. Band, mode and filter act on VFO A, which RX1 hears. Working on B means A<>B on the radio. A>B and B>A copy the whole VFO.
- **Keep Shift tuning of VFO B** (Shift with click, drag, wheel).
- **Start with what works on today's server**, and file the missing server features as issues in `iu3qez/deskhpsdr`: done as #27 and #28.
- Earlier the operator chose "band, mode and filter follow the active VFO" (option B), then stopped the work: "This is not as VFO A / B are supposed to work". The model above replaces it.
- Client changes first, bench after, for the TCI fixes.
- Old branches may be deleted (see "Branch cleanup").
- Open, raised at the end of the session: **the panadapter has very low contrast, "almost no information compared to deskHPSDR"**. Not investigated yet.

My own calls, not asked: the TX line is shown whenever `tx_frequency` differs from VFO A, not only with split on (deskHPSDR also transmits on VFO B when RX2 is the active receiver in its GUI, `vfo_get_tx_vfo()` in `src/vfo.c`); the VFO A drag fix as its own commit; the waterfall gets Shift click and wheel but no drag, because VFO A has no drag there either.

## VFO model, and the wrong path

Read from deskHPSDR `src/vfo.c` and `src/vfo.h` at `6d0d2aa9`:

- `struct _vfo` holds band, band stack, frequency, CTUN, RIT/XIT, mode, filter and step for both VFOs, with any number of receivers.
- RX1 always listens on VFO A; RX2, when on, on VFO B.
- `vfo_a_to_b()`, `vfo_b_to_a()`, `vfo_a_swap_b()` (`src/vfo.c:1341`, `1350`, `1359`) copy or swap the whole struct.
- TX is on VFO A, or on VFO B with split; `vfo_get_tx_vfo()` follows `active_receiver`.

**Wrong path, abandoned**: treating TCI index 1 as "VFO B", tying B's controls to RX2, and a client-side "active VFO". The TCI code uses index 1 for RX2, and with one receiver refuses it everywhere except `vfo:0,1`. That is a server limit, filed as #28, not a property of VFO B.

## Server side (iu3qez/deskhpsdr)

Merged on 2026-09-18: PR #21 (TX ownership while MOX or TUNE is queued), #22 (mode and filter reporting; closes #17, #18, #20), #23 (`rx_enable` switches RX2; closes #12), #25 (every command of a text frame runs; closes #24), #26 (per-receiver client state in one struct, the refactor; closes #10). #19 and #8 were closed as not defects. `documentation/deskHPSDR_TCI_Remote_Extensions.md` section 5 documents `modulation_ex`; its compatibility notes say narrow FM is now `NFM`.

Opened in this session, English, with line references at `6d0d2aa9`:

- **#27** TCI: no command for A>B, B>A and A<>B. Proposes `vfo_swap_ex;` (Thetis's name and form, `ramdor/Thetis` `TCIServer.cs`), `vfo_a_to_b_ex;`, `vfo_b_to_a_ex;`. Open point for the implementer: refusal while transmitting.
- **#28** TCI: with one receiver, VFO B's mode is never reported. Proposes reporting it to clients that queried `modulation_ex`; setting it is not requested.

Station, machine-local, verified over SSH at the end of the session: `~/deskhpsdr` on `ubuntu.lan` at `283cf6d` (merge of PR #25), binary built 2026-09-18 23:30, running, TCI 50001. PR #26 (merged 23:42) is **not** in that binary; it is a refactor with no protocol change intended. Bench results only count on the latest build, so rebuild before the next bench.

## Client work in this session

All on the PR 17 branch, each commit with `website/` regenerated and `scripts/smoke-test.js` passing:

| Commit | What |
|---|---|
| `11256e0` | `modulation_ex:0;` at connect; CWL/CWU from `modulation_ex` |
| `d15c35d` | RX2 chip sends the request and follows `rx_enable:1,<bool>` |
| `2b9d376` | no `rx_filter_band:0;` query after each `modulation` |
| `4fbac98` | FM button sends `NFM`; no FM name translation |
| `649a136`, `43ef402` | checklist rows E4, F9, F10, and the bench results |
| `f44ea0e` | SPLIT in the VFO panel after A→B, B→A, A⇌B |
| `43ac1d6` | `tx_frequency` kept in `S.txFreq`; red TX line on spectrum and waterfall when it differs from VFO A (`txMarkerShown()`) |
| `2b1c482` | VFO B dashed on the waterfall |
| `b55c427` | fix: VFO A mouse drag counts from the frequency at mousedown |
| `4f0362e` | Shift + click (spectrum, waterfall), drag (spectrum), wheel (both) tunes VFO B via `tuneVfo()` |
| `a73d9bc` | checklist rows C11-C14, F11-F13 |

Still frequency-only, waiting for #27: the A→B, B→A and A⇌B buttons (`vfoSwap()` in `totw.html`). VFO B's display shows no mode, waiting for #28.

## Verification

- **Bench**, operator, build of 23:30: E1-E4, F9, F10 all OK (`docs/bench/reference-checklist.md`, header records client commit and build).
- **Not yet benched**: C11-C14, F7, F8, F11-F13. F7 and F8 still carry notes from 2026-09-18 describing SPLIT in the Options panel; the next run overwrites them. C7 (drag) was OK before the drag fix; C11 retests it.
- **Fake WebSocket**, in the in-app browser: connect with CWL, GUI mode changes, TOTW mode buttons, RX2 on/off/refused in MOX, reconnect; SPLIT, TX line on and off, B on the waterfall (canvas pixels read back); Shift click, drag, wheel with `deltaX` only; VFO A drag of 50 px ends at the expected frequency (it ended 12 kHz high before the fix).

## Fragile and machine-local state

- **The fake WebSocket is not in the repository**: `fake-ws.js` and `refresh-fake.sh` in this session's scratchpad (`/private/tmp/claude-501/-Users-sf-Developer-thetis-on-the-web--claude-worktrees-recent-handoff-81c25b/2a48890f-7f57-45be-9bbc-fdf19056d2eb/scratchpad/web/`, machine-local) will not survive. It replaced `window.WebSocket` from a copy of `totw.html` with the script injected first in `<head>`, served by `.claude/launch.json` (untracked, `python3 -m http.server 8766`). It modelled `src/tci.c` at the merge of PR #26: initial state with `trx_count:1`, mode changes queued then `modulation` + `modulation_ex` (subscribed clients) + `rx_filter_band`, `rx_enable` with refusal in MOX and the RX2 state on switch-on, `tx_frequency` after VFO and split changes, echoes of every `vfo` to all clients, and only the first command of a frame (issue #24, now fixed on the server). IQ was simulated by filling `IQ.fftResult` and setting `IQ.fftReady`.
- The worktrees `aerokeyer-rp2040-analysis-f77c92` (on `feat/totw-modular-operable-client`) and `handoff-riprendere-48722d` (on `claude/bench-u1-lws-fix-90073c`) are behind the remote: pull before use.

## Traps met in this session

- **The desktop app opens `totw.html` in the browser pane after every edit** of the file. The page did not connect (host `ws://127.0.0.1:40001`), but check `S.ws` and close the tab; test only on the fake copy.
- The browser caches `fake-ws.js` across page reloads: put a timestamp on the script URL.
- deskHPSDR echoes every `vfo` to all clients, the sender included (`tci_broadcast_vfo`): a gesture that adds an offset to `S.vfoA` or `S.vfoB` during the gesture drifts.
- In a perl one-liner inside double quotes, `$(` is perl's GID variable, not shell substitution.
- zsh does not word-split `$VAR` holding a command: `S="git show ..."; $S` fails.
- Unchanged: `gh` always with `-R`; no reference to the original project.

## Branch cleanup, not done

The operator allowed deleting old branches; the permission layer refused `git branch -d` and `git push origin --delete`. All candidates are contained in `main` and checked out nowhere:

- remote: `c1/dispatch-binary-frames`, `c2/iq-samplerate-config`, `plan/deskhpsdr-client`;
- local: `c1/dispatch-binary-frames`, `c2/iq-samplerate-config`, `docs/plan-amendments-c1`, `claude/aerokeyer-rp2040-analysis-f77c92`, `claude/dove-eravamo-rimasti-288562`, `claude/handoff-riprendere-48722d`, `claude/recent-handoff-81c25b`.

Keep: `main`; `plan/deskhpsdr-client` locally (the main checkout has it checked out); `feat/totw-modular-operable-client` and `claude/bench-u1-lws-fix-90073c` (checked out in worktrees).

## Plausible next steps

One path, in order:

1. Rebuild deskHPSDR on the station at the latest `master`, then bench C11-C14, F7, F8, F11-F13.
2. Investigate the panadapter contrast against deskHPSDR's own panadapter: the dB range and gain of `drawSpec()` and `drawWF()` in `totw.html` (`DB_MIN`, `DB_MAX`, `specGain`, the waterfall palette) against how deskHPSDR scales its display. Read deskHPSDR's panadapter source before assuming.
3. When #27 lands: A⇌B, A→B and B→A through the new commands. When #28 lands: VFO B's mode in its display.
4. U2, the removals, compared against the checklist.

Independent: the "14.225 MHz" label at the bottom edge of the spectrum does not follow VFO A (seen in the fake tests, older than this session).
