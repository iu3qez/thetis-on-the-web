---
artifact_contract: "ce-handoff/v1"
created_at: "2026-09-18T22:23:17Z"
title: "TOTW: deskHPSDR now has VFO copy/swap and VFO B's mode; the client does not use them yet"
summary: "deskHPSDR fixed issues 27 and 28 (PRs 29 and 30): vfo_swap_ex, vfo_a_to_b_ex, vfo_b_to_a_ex, and VFO B's mode reported with one receiver to modulation_ex clients. The station binary predates them. The TOTW side, the bench and the branch cleanup are open."
keywords: ["totw", "deskhpsdr", "vfo_swap_ex", "vfo_a_to_b_ex", "vfo_b_to_a_ex", "modulation_ex", "vfo-b", "issue-15", "pr-17", "u10"]
cwd: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/recent-handoff-81c25b"
resume_focus: "Use the new deskHPSDR VFO commands and VFO B's mode in TOTW (issue 15), after rebuilding the station; then the pending bench rows"
repository: "iu3qez/thetis-on-the-web"
repo_root_sha: "a14fbec33d24418421b76761f4bbfe888d78bbb0"
branch: "feat/totw-modular-operable-client"
head: "32f082f"
worktree_path: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/recent-handoff-81c25b"
---

# Handoff: deskHPSDR's VFO commands have landed

**Supersedes `docs/handoffs/2026-09-19-tci-fixes-vfo-model-shift-b.md` for the state of the work.** That handoff stays the reference for the VFO model confirmed by the operator, the wrong "active VFO" path, the commits of the session, the traps, and the description of the fake WebSocket. Read it first; this one only records what changed after it.

The `head` field is the last commit before this handoff; the commit adding this file sits on top of it.

## What changed

The operator reported that deskHPSDR solved both issues opened for TOTW. Verified on GitHub and in `/Users/sf/Developer/deskhpsdr` (machine-local checkout), `master` at `5900c0b4`:

- **#28, closed by PR #29** (`0506c10f`). With one receiver, a client that has queried `modulation_ex` gets VFO B's mode: `modulation:1;` and `modulation_ex:1;` are answered, and `modulation:1,<mode>;` then `modulation_ex:1,<mode>;` are sent on every change of VFO B's mode, A>B and A<>B included. Not changed: setting VFO B's mode with one receiver is ignored, and `rx_filter_band:1` is not sent with one receiver. A client that never queried `modulation_ex` sees no difference. Contract: section 5 of `documentation/deskHPSDR_TCI_Remote_Extensions.md`.
- **#27, closed by PR #30** (`986081d8`). New commands, no argument, no reply: `vfo_swap_ex;` (A<>B, like CAT `ZZVS2`), `vfo_a_to_b_ex;` (A>B, `ZZVS0`), `vfo_b_to_a_ex;` (B>A, `ZZVS1`). They move the whole `struct _vfo`. The result reaches every client, the sender included, as the messages of a GUI swap: `dds:0`, `vfo:0,0`, `vfo:0,1`, `modulation:0`, `rx_filter_band:0` (plus the DIGU/DIGL offset), receiver 1 while RX2 runs, then `tx_frequency`, `drive`, `tune_drive`, `split_enable`; with one receiver a `modulation_ex` client also gets `modulation:1`. Accepted while transmitting, like the GUI buttons. They share the 200 ms set-lock of `vfo`: a command within 200 ms of another client's `vfo` set or copy is ignored, with a log line. A command with an argument is ignored. Contract: section 6 of the same document; the sections after it moved down by one.

## Current state

| Piece | State |
|---|---|
| deskHPSDR `master` | `5900c0b4`, with PRs #29 and #30 |
| Station binary on `ubuntu.lan` | `master` at `6d0d2aa` (merge of #26), built 2026-09-18 23:43, running. **Without #29 and #30**: no `vfo_swap_ex` in the binary, and the checkout has not fetched them. Needs fetch and rebuild. |
| TOTW, PR #17 branch | `32f082f`, pushed. Does not use the new commands or VFO B's mode. |
| PR #17 | draft, description updated to this state |
| TOTW #18 | panadapter contrast, to be handled with U10 (operator's decision); records what `type=4` fixes and what the client still needs |
| Old branches | still present: the deletion was refused by the permission layer. List and commands in the previous handoff, "Branch cleanup". |
| Fake WebSocket | not in the repository; the session scratchpad that held it will not survive. Its behaviour is described in the previous handoff and must be rewritten, now with the three copy commands and VFO B's mode. |

## What the TOTW side needs (not started)

From the contract above; my reading, to be confirmed against `src/tci.c` at `5900c0b4` before coding:

- `vfoSwap()` in `totw.html` (the A→B, B→A, A⇌B buttons) sends `vfo_a_to_b_ex;`, `vfo_b_to_a_ex;`, `vfo_swap_ex;` instead of copying frequencies with `vfo:0,0` and `vfo:0,1`, and stops updating the displays locally: the result arrives as broadcasts. Because of the 200 ms set-lock, a click right after a Shift-tune from another client can be ignored; the displays then simply do not change.
- The client already queries `modulation_ex:0;` at connect, which subscribes it. It also needs `modulation_ex:1;` at connect to learn VFO B's current mode, and handlers for `modulation:1` and `modulation_ex:1` (today only index 0 is handled) to show VFO B's mode in its display. While RX2 runs, index 1 is still RX2, which listens on VFO B, so the same handler fits both cases.
- VFO B's mode cannot be set with one receiver: the model is A<>B, change the mode on A, A<>B.
- Checklist: C5 (A→B, B→A, A⇌B) to rerun with the whole VFO moving, and a row for VFO B's mode in its display, in Italian like the rest of `docs/bench/reference-checklist.md`.

## Plausible next steps

One path:

1. Rebuild deskHPSDR on the station at `master` (the operator builds; bench results only count on the latest build).
2. Implement the TOTW side above, tested against a rewritten fake WebSocket, one commit per part.
3. Bench: C5 and the new row, plus the rows still pending from the previous session: C11-C14, F7, F8, F11-F13.
4. Then U2, the removals; the panadapter contrast waits for U10 (#18).

Independent: delete the old branches with the commands in the previous handoff.
