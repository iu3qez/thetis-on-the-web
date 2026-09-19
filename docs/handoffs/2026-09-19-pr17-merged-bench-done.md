---
artifact_contract: "ce-handoff/v1"
created_at: "2026-09-19T03:10:43Z"
title: "TOTW: PR 17 merged; VFO copy commands, VFO B's mode and the TX line fix benched; WebSocket closes open (#22)"
summary: "The client uses deskHPSDR's vfo_*_ex commands and VFO B's mode, the TX line follows split_enable, deskHPSDR PR 32 sends tx_frequency after a vfo set, the bench passed, PR 17 is merged; open: M1-M3 and #22 (the WebSocket closes by itself, cutting RX audio with a click)."
keywords: ["totw", "deskhpsdr", "pr-17", "vfo_swap_ex", "modulation_ex", "tx_frequency", "split_enable", "bench", "issue-19", "issue-20", "issue-21", "u2", "u7", "issue-22", "websocket-close", "rx-audio"]
cwd: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/recent-handoff-81c25b"
resume_focus: "Diagnose #22 (high priority) on a new branch from main, bench M1-M3 on main, then U2"
repository: "iu3qez/thetis-on-the-web"
repo_root_sha: "a14fbec33d24418421b76761f4bbfe888d78bbb0"
branch: "docs/bench-results-and-handoff"
head: "6d76b65"
worktree_path: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/recent-handoff-81c25b"
---

# Handoff: PR 17 merged after the VFO work and the bench

**Supersedes `docs/handoffs/2026-09-19-vfo-commands-landed.md`.** `docs/handoffs/2026-09-19-tci-fixes-vfo-model-shift-b.md` stays the reference for the VFO model confirmed by the operator and the wrong "active VFO" path.

The `head` field is the last commit before this handoff; the commit adding this file sits on top of it. PR 17 was merged on 2026-09-19 at 03:13 UTC, 05:13 station time, with its branch at `910bf5e`. The D, K and L bench results and this handoff were first pushed to that branch after the merge, so they reach `main` through a follow-up PR from `docs/bench-results-and-handoff`. Times of station builds are the station's local time (CEST).

## Operator decisions in this session

- The operator deleted the last two old branches (`claude/dove-eravamo-rimasti-288562`, `claude/recent-handoff-81c25b`) with `git branch -D`; the branch cleanup of the earlier handoffs is finished, and the memory reminder for it was removed.
- The operator rebuilt deskHPSDR on the station twice: at the merge of PR 30, then at the merge of PR 32.
- Red TX line flashing with split off: the operator chose a server-only fix (deskHPSDR #31). When the bench still showed the line during tuning, I changed the client rule without a separate question; the operator then confirmed at the bench that the problem is gone.
- Filter passband moving while tuning: not fixed now, low-priority issue (#19).
- Promote PR 17: the operator chose to record the bench results before promoting, then merged it.
- Wheel defects: one issue each for zoom (#20) and tuning (#21).
- RX audio click and silence: the operator called it a big problem; opened as #22, high priority.

## Current state

| Piece | State |
|---|---|
| deskHPSDR `master` | merge of PR 32 (`31250e83`). This session: PR 29 (#28, VFO B's mode), PR 30 (#27, copy commands), PR 32 (#31, `tx_frequency` after a `vfo` set) |
| Station `ubuntu.lan` | `master` at `31250e8`, built 2026-09-19 04:43, running, TCI 50001. Verified over SSH |
| Local deskHPSDR checkout (machine-local, `/Users/sf/Developer/deskhpsdr`) | `master` still at `6d0d2aa9`; `origin/*` fetched to `31250e83`; not pulled. Read the server with `git show origin/master:src/tci.c` |
| PR 17 | merged into `main` (`b6164e0`), branch at `910bf5e`. Its description was edited after the merge; its "Not done" lists M1-M3 not run, D1 and D2 KO (#20), C10 (#21), and #22 |
| Follow-up PR | from `docs/bench-results-and-handoff`: the D, K and L bench results and this handoff |
| Repository | no CI, no GitHub Pages: a merge to `main` publishes nothing |

## Client work in this session

All on the PR 17 branch, `website/` regenerated and `scripts/smoke-test.js` passing after each change to `totw.html`:

| Commit | What |
|---|---|
| `6bb95f2` | `vfoSwap()` sends `vfo_a_to_b_ex;`, `vfo_b_to_a_ex;`, `vfo_swap_ex;`; no local display update; clears `bpIgnoreVfoUpdateUntil` and `bpIgnoreVfoBUpdateUntil` before sending |
| `21ca945` | `S.modeB`; `modulation_ex:1;` at connect; index 1 in the `modulation` and `modulation_ex` handlers; `modeFromTci(name, prev)`; `updModeB()` fills `#vfoBMode` next to the VFO B label |
| `aa60ad3` | checklist: C5 rewritten for the whole-VFO copy, new rows E5, E6 |
| `eaab31d` | `txMarkerShown()` requires `TG.split` (`totw.html:6357`) |
| `2d16a4b` | checklist: F13 also covers tuning VFO A |
| `910bf5e`; `6d76b65` in the follow-up PR | bench results and header |

## Server behaviour confirmed from source (at `31250e83`)

- Copy commands: no reply; `vfo.c` runs `vfo_vfos_changed()` then `tci_vfos_changed()` to every client, the sender included: `dds:0`, `vfo:0,0`, `vfo:0,1`, `modulation:0`(+`_ex`), `rx_filter_band:0`, then index 1 (with one receiver only `modulation:1`(+`_ex`) for subscribed clients), `tx_frequency`, `drive`, `tune_drive`, `split_enable`.
- Set-lock (`tci_set_lock_allowed()`, `src/tci.c:2338`): one slot for `vfo` sets and copy commands; the owner renews it; only another TCI client within 200 ms is refused; the GUI takes no lock; a refused copy command gets no message at all.
- `modulation_ex:1;` subscribes and is answered with `modulation_ex:1,<mode>;` only. The server does not send VFO B's current mode on subscription.
- `split_enable` is true exactly when `vfo_get_tx_vfo()` is VFO B (`tci_send_split()`), which includes RX2 active in the GUI.
- `dds:0` carries `vfo[0].frequency` (`tci_send_dds()`, `src/tci.c:1765`).
- `tci_reporter()` runs every 500 ms per client and sends only differences from the per-client `last_*` values: a message the client drops is never sent again.

## Wrong paths and traps

- The fake server inherited from the previous session sent `tx_frequency` after every `vfo` set. The real server did not (before PR 32), so the fake hid the delay. Model the server from `src/tci.c`, not from what the client expects.
- The rule "TX line when `tx_frequency` differs from VFO A" is wrong during tuning: the client moves `S.vfoA` at once, and each `tx_frequency` confirms a step already passed. Do not restore it. #19 is the same pattern with `dds:0` and `S.iqCentre`.
- `git branch -d` in the main checkout compares with its HEAD (`plan/deskhpsdr-client`), not with `main`. Check with `git merge-base --is-ancestor <branch> main`.
- The desktop app stops the preview server after a while: restart with `preview_start` name `totw-fake`. It can also open `totw.html` from file:// in the pane after an edit: check `S.ws` there and close the tab.
- PR 17 was merged while I was still pushing to its branch. Check the PR state before every commit, push or `gh pr edit` on a PR branch (`gh pr view <n> -R iu3qez/thetis-on-the-web --json state,mergedAt,headRefOid`); the operator asked for this.
- An awk scan for `tci_begin_apply()` callers matched a comment above the copy commands and listed `tci_cmd_vfo` wrongly.

## Fake WebSocket (machine-local, not in the repository)

`/private/tmp/claude-501/-Users-sf-Developer-thetis-on-the-web--claude-worktrees-recent-handoff-81c25b/63c65598-a2a0-4fa3-99dd-adf883ffa85c/scratchpad/web/fake-ws.js`, page built by `refresh-fake.sh` in the same scratchpad, served by the untracked `.claude/launch.json` (`python3 -m http.server 8766` on that directory). It models `src/tci.c` at the merge of PR 32: every command of a frame; VFO B's mode for `modulation_ex` clients; the three copy commands with the set-lock; `tx_frequency` after a `vfo` set; a 500 ms reporter per client. Helpers: `__fake.gui.{copy,mode,rx2,split,vfoB,mox}`, `__fake.other.vfo(ch, hz)` for another client holding the lock, `__fake.sent()`, `__fake.clear()`, `__fake.lockLog`. Network delay was simulated by wrapping the fake socket's `emit` with an 80 ms `setTimeout`. The scratchpad will not survive; the previous session's copy (`.../2a48890f-.../scratchpad/web/`) is older.

## Verification

- Fake: A⇌B, A→B, B→A with displays, band and mode following; a copy with the echo window open ends correct, the same command sent around `vfoSwap()` does not; a copy refused by the set-lock changes nothing; VFO B's mode at connect, after copies, in CW (`modulation:1,CW` then `modulation_ex:1,CWL` ends on CWL), with RX2, after reconnect; with 80 ms delay and split off the old TX rule showed the line in 16 samples, the new one in none.
- Bench, operator, Firefox on macOS, build of 2026-09-19 04:43: C5, C11-C14, D3, E5, E6, F7, F8, F11-F13, K1, K2, L1-L4 OK. D1 and D2 KO (#20). M1-M3 not run. `docs/bench/reference-checklist.md` holds the results and the header.

## Open issues

- `iu3qez/thetis-on-the-web`: #15 (VFO B and split; the parts listed in the earlier handoffs are done and benched, the issue is still open), #16 (RX2), #18 (panadapter contrast, U10), #19 (filter passband moves while tuning, low priority), #20 (zoom x2 per wheel event, U7), #21 (one tuning step per wheel event, U7), #22 (high priority, see below).

#22, from the station log (`~/.config/deskhpsdr/deskhpsdr.log` and `.log.1` on `ubuntu.lan`, machine-local, `tci_debug` on): the WebSocket closed 9 times in 31 minutes on the build of 04:43 and 7 times in 14 minutes on the build of 00:23, each time right after normal traffic and with no server error; the same Firefox reconnected 1.0 to 1.5 s later, which is TOTW's automatic reconnect. `onclose` calls `stopRx()`, which closes the `AudioContext` mid-waveform (`totw.html:5043`): that is the click. Who closes, and why, is not known: the client drops the close code and the server log does not name the initiator. The issue lists the suspects (Firefox rejecting a `permessage-deflate` frame from the development libwebsockets `5.0.99-sai-d7f7fdeaf`, a server timeout or write error, the Mac's network) and the diagnostics (close code in the client, `tcpdump` on port 50001).
- `iu3qez/deskhpsdr`: #27, #28, #31 closed by PRs 30, 29, 32.

## Plausible next steps

One path:

1. The follow-up PR from `docs/bench-results-and-handoff` is merged.
2. #22, high priority, on a new branch from `main`: log the close code in the client and capture one event with `tcpdump`; the fade-out on an unexpected close removes the click whatever the cause.
3. Bench M1-M3 on `main`: the checklist is the baseline for U2's removals.
4. U2, on a new branch from `main`, per `docs/plans/2026-09-16-0316-feat-totw-modular-operable-client-plan.md`.

After the follow-up PR is merged, for the operator: the remote branches `feat/totw-modular-operable-client` and `docs/bench-results-and-handoff`, the local branches `claude/server-sweep-adapt` and `docs/bench-results-and-handoff`, and the worktree `aerokeyer-rp2040-analysis-f77c92`, which has an old `feat/totw-modular-operable-client` checked out.

The client-side pattern behind the TX line and #19, a local update of `S.vfoA` ahead of the server's replies, is a candidate for `docs/solutions/` (`compound-engineering:ce-compound`).
