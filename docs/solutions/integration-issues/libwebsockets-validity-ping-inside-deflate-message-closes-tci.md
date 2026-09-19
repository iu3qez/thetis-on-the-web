---
title: libwebsockets sends the validity PING inside a permessage-deflate message, and deskHPSDR closes the TCI connection 10 s later
date: 2026-09-19
category: integration-issues
module: deskhpsdr-client
problem_type: integration_issue
component: messaging
symptoms:
  - "RX audio gives a full-scale click, then a few seconds of silence, or only silence; spectrum and S-meter freeze meanwhile"
  - "deskHPSDR logs `LWS CLOSED client=N` every 1.5 to 9 minutes, right after normal traffic, with no error"
  - "Connection lifetimes are all 10 + 40*k s (50, 90, 130, 170, 250, 530 s)"
  - "Firefox reconnects 1.0 to 1.5 s after each close, through TOTW's automatic reconnect"
  - "The close reaches Firefox as code 1006, wasClean false, no error event: a TCP FIN with no close frame"
root_cause: logic_error
resolution_type: config_change
severity: high
framework_version: libwebsockets 5.0.99-sai-d7f7fdeaf (static, built by build-libwebsockets.sh)
related_components:
  - tci_server
  - libwebsockets
  - rx_audio
  - websocket_reconnect
tags: [tci, websocket, permessage-deflate, libwebsockets, deskhpsdr, validity-ping, keepalive, reconnect, close-1006]
---

# libwebsockets sends the validity PING inside a permessage-deflate message, and deskHPSDR closes the TCI connection 10 s later

Where the cited paths resolve: `lib/...` is libwebsockets at `d7f7fdeaf`, the tree that the station's `build-libwebsockets.sh` cloned and built (the script clones the tip, with no pinned commit). `src/...` is iu3qez/deskhpsdr. `totw.html` and `docs/...` are this repository.

## Problem

deskHPSDR closed TOTW's TCI WebSocket by itself, 10 + 40·k s after each handshake, with a TCP FIN and no close frame. Each close cut TOTW's RX audio, often with a full-scale click, and froze the spectrum and the S-meter until the reconnect.

## Symptoms

Bench of 2026-09-19, Firefox 155 on macOS, TOTW with RX audio on (#22) and IQ on (iu3qez/deskhpsdr issue 33). The station's deskHPSDR links libwebsockets `5.0.99-sai-d7f7fdeaf` statically, with permessage-deflate available. That is the build adopted in `docs/solutions/integration-issues/libwebsockets-permessage-deflate-assert-crashes-deskhpsdr-ptt.md`.

- RX audio: a full-scale click, then a few seconds of silence, then normal audio. Or only the silence. The spectrum and the S-meter froze during the silence.
- `deskhpsdr.log`: one `LWS CLOSED client=N` every 1.5 to 9 minutes. Each came 7 to 37 ms after a normal `rx_smeter` exchange. No error was logged before it, and `deskhpsdr.err` was empty.
- The next `LWS HANDSHAKE` from the same Firefox came 1.0 to 1.5 s later (once 2.7 s). That is TOTW's `scheduleReconnect()` (`totw.html:4273`), whose first delay is 1000 ms (`_reconnectDelay`, `totw.html:4149`).
- Connection lifetimes from `LWS HANDSHAKE` to `LWS CLOSED`: 90.0, 50.0, 90.0, 250.0, 50.0 and 50.0 s, each within 30 ms. Earlier runs: 90, 130, 170, 250 and 530 s. All are 10 + 40·k s (iu3qez/deskhpsdr issue 33).
- Station capture, 2026-09-19 10:53 to 11:12, 6 closes (iu3qez/deskhpsdr issue 33):
  - deskHPSDR sent a WebSocket PING every 40 s. Firefox answered every PING it received within 3 to 6 ms.
  - Each close was a FIN from deskHPSDR, 50.0 s after the last PONG, or after the handshake when no PING had been answered. Neither side sent a close frame. There was no RST. Firefox sent its own FIN 2 to 6 ms later.
  - In each closed connection the PING due 40 s after the last PONG was missing, and so was its PONG.
  - Data flowed until the FIN. The largest gap between deskHPSDR's small segments was 40 ms, and between Firefox's ACKs 43 ms.
- In Firefox these closes show as code 1006, `wasClean` false, no `error` event (per iu3qez/deskhpsdr issue 33).

## What Didn't Work

**The three candidates in #22.** #22 listed three causes, none verified:

1. Firefox fails the connection on a compressed frame it cannot inflate (close code 1002 or 1007).
2. deskHPSDR closes it: a libwebsockets timeout or a write error.
3. A Wi-Fi or interface change on the macOS laptop.

The evidence at hand could not separate them. `S.ws.onclose` discarded `code`, `reason` and `wasClean`, and `onerror` logged a fixed text. The server log recorded `LWS CLOSED` without the side that started the close. The libwebsockets notice that names a validity close (`lib/core-net/wsi-timeout.c:270`) was never written, because `tci_lws_server()` called `lws_set_log_level(LLL_ERR, NULL)` (`src/tci.c:7758` before iu3qez/deskhpsdr PR 34). Candidate 2 was the right one. Only the capture and the source reading in iu3qez/deskhpsdr issue 33 showed it.

**The lifetime pattern was set aside.** #22 listed lifetimes of 130, 87, 170, 130, 250, 531, 170, 170 and 90 s and said the clustering might be a coincidence. Measured from `LWS HANDSHAKE` to `LWS CLOSED` in `deskhpsdr.log`, every lifetime of the run started 10:50:35 fell within 30 ms of 10 + 40·k s, and the earlier runs fit the same rule (iu3qez/deskhpsdr issue 33). 40 s and 50 s are the libwebsockets defaults `secs_since_valid_ping` and `secs_since_valid_hangup` (`lib/core/context.c:1426-1427`).

**The assumption that a PING always travels as a control frame.** Step 2 of the proposal in #22 relied on RFC 7692: control frames are never compressed, so a capture shows them. That holds for the protocol, not for this libwebsockets build. `rops_write_role_protocol_ws()` sends a PING written during a permessage-deflate drain as payload of the data message (see Why This Works).

**Starting the capture from the SSH tool.** `sudo` on `ubuntu.lan` asks for a password, `tcpdump` has no capabilities set, and `dumpcap` has `cap_net_raw` but is mode 750 for group `wireshark`, which the SSH user is not in. The operator ran the capture by hand from a terminal. No `tshark` was available, so the pcap was parsed with a one-off Python script on the station (session history).

**The station capture with a length filter.** The capture on `ubuntu.lan` kept FIN and RST segments and packets under 300 bytes:

```bash
ssh -t ubuntu "sudo timeout 1200 tcpdump -i any -w /tmp/tci22.pcap 'tcp port 50001 and (tcp[tcpflags] & (tcp-fin|tcp-rst) != 0 or len < 300)'"
```

It showed the PINGs that went out alone, the PONGs, and the FIN 50.0 s after the last PONG. It could not show where the missing PING went. Its 8 bytes left inside a data message fragment, and those packets were over 300 bytes, so the filter dropped them. Without the filter they would still not appear as a PING frame: they are compressed with the message (`lib/roles/ws/ops-ws.c:1880-1887`) and are visible only after inflating it. iu3qez/deskhpsdr issue 33 marked this step as a source reading not seen on the wire. The loopback test in iu3qez/deskhpsdr PR 34 confirmed it. The filter was also not small: `len < 300` keeps every pure ACK the Mac sends for the audio and IQ segments, 996518 packets in 20 minutes (session history).

**The closing comment on #22 is wrong on two points.** It says "Some of those PINGs never reached the wire", and it says the permessage-deflate suspect is ruled out. The correct version:

- The PINGs reached the wire, but not as PING frames. iu3qez/deskhpsdr PR 34 found one at the end of a binary data message: an IQ message of 8264 bytes = 64 + 8192 + 8. The 8 extra bytes, `d677b50b83000000`, are the `lws_now_usecs()` value that `rops_issue_keepalive_ws()` writes into the PING payload (`lib/roles/ws/ops-ws.c:2354-2355`). Firefox received them as message data, so it had no PING to answer.
- permessage-deflate is one of the two conditions of the defect. The write-type change happens only while `tx_draining_ext` is set, and only the permessage-deflate TX path sets it (`lib/roles/ws/ops-ws.c:1894-1901`). The other condition is the validity PING. What the evidence ruled out is candidate 1 of #22, Firefox rejecting a compressed frame: Firefox sent no close frame (it sends 1002 when it rejects a frame, per PR #24), and every close started with a FIN from deskHPSDR.

## Solution

### Server: iu3qez/deskhpsdr PR 34, merged 2026-09-19

The station runs deskHPSDR `master` at the merge of iu3qez/deskhpsdr PR 35 since 2026-09-19 16:20. Line numbers below are `src/tci.c` on that `master`.

1. No libwebsockets validity PINGs. `tci_lws_server()` passes its own retry policy with both validity times at 0, and sets the TCP keepalive values (`src/tci.c:7862-7876`, excerpt):

   ```c
     static const lws_retry_bo_t tci_lws_retry = {
       .secs_since_valid_ping   = 0,
       .secs_since_valid_hangup = 0,
     };
     /* ... */
     info.retry_and_idle_policy = &tci_lws_retry;
     info.ka_time = TCI_LWS_KA_TIME;
     info.ka_interval = TCI_LWS_KA_INTERVAL;
     info.ka_probes = TCI_LWS_KA_PROBES;
   ```

2. Dead peers are detected by TCP. `TCI_LWS_KA_TIME` is 30 s, `TCI_LWS_KA_INTERVAL` 10 s, `TCI_LWS_KA_PROBES` 3, and `TCI_LWS_DEAD_PEER_S` is their total, 60 s (`src/tci.c:87-90`). On Linux libwebsockets applies them to every accepted socket, plus `TCP_USER_TIMEOUT` = 1000 · (30 + 10 · 3) = 60000 ms (`lib/plat/unix/unix-sockets.c:132-173`, called from `lib/roles/listen/ops-listen.c:188`). The station build defines `LWS_HAVE_TCP_USER_TIMEOUT` (per iu3qez/deskhpsdr PR 34).
3. macOS. On Apple platforms libwebsockets sets only `SO_KEEPALIVE` and leaves the times to the kernel (`lib/plat/unix/unix-sockets.c:139-148`). The kernel default idle time is 7200 s (per iu3qez/deskhpsdr PR 34). `tci_lws_set_tcp_timeouts()` (`src/tci.c:7577`), called in `LWS_CALLBACK_ESTABLISHED` (`src/tci.c:7675-7677`), sets `TCP_KEEPALIVE` 30, `TCP_KEEPINTVL` 10, `TCP_KEEPCNT` 3 and `TCP_RXT_CONNDROPTIME` 60, all in seconds (`src/tci.c:7578-7583`). With `rigctl_debug` on it logs the values read back with `getsockopt()`.
4. libwebsockets logging. With `rigctl_debug` on, `tci_lws_update_log_level()` sets `LLL_ERR | LLL_WARN | LLL_NOTICE` (`src/tci.c:7629-7636`). It runs at start and on every pass of the service loop (`src/tci.c:7866`, `7923`), so the level follows the switch at run time. `tci_lws_log_emit()` (`src/tci.c:7615`) sends each line through `t_print()` to `deskhpsdr.log`, next to `LWS CLOSED client=N`. iu3qez/deskhpsdr issue 33 proposed `deskhpsdr.err`. iu3qez/deskhpsdr PR 34 did not use it, because `stderr` is fully buffered once `startup.c` reopens it.

iu3qez/deskhpsdr PR 34 verification ran on loopback with `hpsdrsim -P1 -orion` and libwebsockets `d7f7fdeaf` built with the flags of `build-libwebsockets.sh`. The client was Python `websockets` 17.1 with permessage-deflate and `iq_start:0;`. It logged every PING frame and checked each binary message against `64 + length * bytes_per_sample` from its header.

| Binary | Run | PING frames | Messages longer than the header says | Closes started by deskHPSDR |
|---|---|---|---|---|
| `master` before iu3qez/deskhpsdr PR 34 | 150 s | 2 | 1 at 40.0 s: 8264 = 64 + 8192 + 8 bytes | 1 at 50.0 s, code 1006, no close frame |
| iu3qez/deskhpsdr PR 34 branch | 150 s | 0 | 0 | 0 |
| iu3qez/deskhpsdr PR 34 branch, validity policy commented out | 140 s | 2 | 1 at 40.0 s | 1 at 50.0 s |

In the third run `deskhpsdr.log` showed the `VALIDITY TIMEOUT EXPIRED ON` notice, with `(ping=40, hangup=50)`, just before `LWS CLOSED client=1`.

iu3qez/deskhpsdr PR 35 is a follow-up found while testing. It puts the include directory of a local libwebsockets tree before `/opt/homebrew/include` in the Makefile. Without it, a macOS build with both installed compiles `src/tci.c` against the Homebrew header, links the local library, and `lws_create_context` fails.

### Client: PR #24, merged 2026-09-19

- Close record. `S.ws.onclose` (`totw.html:4246`) records `code`, `reason`, `wasClean`, whether an `error` event came first, the seconds the socket was open, and whether the operator asked for the close. `wsCloseText()` (`totw.html:4166`) formats it, and `onclose` writes it to the TCI log (`totw.html:4259`), in red when nobody asked for the close. In that case `console.warn` also gets the record (`totw.html:4254`). `WSDIAG` (`totw.html:4159`) keeps the last 50 records (`WS_CLOSES_KEPT`, `totw.html:4158`) and counts automatic reconnects (`totw.html:4281`). DIAG shows both.
- RX fade-out. `S.rxOut` is a gain node after the analyser (`totw.html:4999-5001`). `closeRxContext()` (`totw.html:5092`) ramps it to 0 over `RX_STOP_FADE_S` = 0.02 s (`totw.html:5091`), then closes the `AudioContext` after the ramp plus `outputLatency` plus 30 ms. `stopRx()` (`totw.html:5078-5082`) uses it on a lost connection and on RX AUDIO off. `startRx()` uses it for a context left open (`totw.html:4981`). In the PR #24 test against a fake TCI server with a 700 Hz tone at 0.9, the output fell linearly to 0 in 19.5 ms. The largest sample-to-sample step was 0.0824, the tone's own maximum.

PR #24 does not prevent any close. On a close, RX audio is still silent for the reconnect delay, the handshake and the RX audio restart. `onclose` still sets `S.iqOn = false` and stops the S-meter poll (`totw.html:4255`, `4259`), so the spectrum and the S-meter still freeze until the new connection starts them. PR #24 removes the click, and the next close is logged with its code, `wasClean` and `error` flag.

## Why This Works

Library references are libwebsockets `5.0.99-sai-d7f7fdeaf`, the tree that the station's `build-libwebsockets.sh` built.

The sequence that closed the connection:

1. Before iu3qez/deskhpsdr PR 34, `tci_lws_server()` set no `retry_and_idle_policy`. The vhost then took the context default (`lib/core-net/vhost.c:849-852`), and every accepted connection took the vhost's policy (`lib/core-net/adopt.c:84`). The default is `secs_since_valid_ping = 40` and `secs_since_valid_hangup = 50` (`lib/core/context.c:1426-1427`).
2. At the end of the handshake, `lws_validity_confirmed()` (`lib/roles/ws/ops-ws.c:969`) reaches `_lws_validity_confirmed_role()` (`lib/core-net/wsi-timeout.c:314-337`), which arms `sul_validity` for 40 s. A received PONG does the same (`lib/roles/ws/ops-ws.c:612`). In the ws server path nothing else calls it, so text and binary messages from the client do not reset the timer.
3. When the timer fires, `lws_validity_cb()` (`lib/core-net/wsi-timeout.c:257`) calls `rops_issue_keepalive_ws()` (`lib/roles/ws/ops-ws.c:2345`). That writes `lws_now_usecs()` into the 8-byte PING payload, sets `send_check_ping` and requests a writable callback (`ops-ws.c:2354-2359`). `lws_validity_cb()` then sets `validity_hup` and rearms itself for 50 - 40 = 10 s (`wsi-timeout.c:301-305`).
4. On the writable callback, `rops_handle_POLLOUT_ws()` writes the pending PING with `lws_write(..., 8, LWS_WRITE_PING)` (`ops-ws.c:1539-1554`). This step comes before the step that drains a permessage-deflate message (`ops-ws.c:1563-1575`), whose comment says control packets can interleave with the fragments.
5. `lws_write()` calls `rops_write_role_protocol_ws()` (`lib/core-net/output.c:237-238`, `ops-ws.c:1790`). If `tx_draining_ext` is set, that function clears it, removes the connection from the drain list, and replaces the write type with `LWS_WRITE_CONTINUATION` (`ops-ws.c:1806-1824`). The `switch` that keeps PING, PONG and CLOSE out of the extension comes later (`ops-ws.c:1880-1884`). The write type is no longer `LWS_WRITE_PING`, so the 8 bytes go to `lws_ext_cb_active(..., LWS_EXT_CB_PAYLOAD_TX, ...)` (`ops-ws.c:1887`) as payload of the message being drained.
6. `tx_draining_ext` is set only when permessage-deflate returns `PMDR_HAS_PENDING` (`ops-ws.c:1894-1901`). Its TX buffer defaults to 1024 bytes (`lib/roles/ws/ext/extension-permessage-deflate.c:241`). An RX audio message is 4160 bytes (`ws_len=4160` in `deskhpsdr.log`), and an IQ message in the iu3qez/deskhpsdr PR 34 test was 64 + 8192 bytes. Each goes out in several fragments, so the flag is set for part of the time the streams run. A PING issued in that time goes out inside the message. A PING issued outside it goes out alone, and those are the PINGs the capture saw.
7. No PING frame reaches Firefox, so no PONG comes back and `ops-ws.c:612` does not run. 10 s later `lws_validity_cb()` finds `validity_hup` set. It logs the notice at `wsi-timeout.c:270`, which `LLL_ERR` dropped, and calls `__lws_close_free_wsi(wsi, LWS_CLOSE_STATUS_NOSTATUS, "validity timeout")` (`wsi-timeout.c:278`). For `LWS_CLOSE_STATUS_NOSTATUS`, `rops_close_via_role_protocol_ws()` returns without queuing a close frame (`ops-ws.c:1713-1716`). The socket closes with a FIN only, and Firefox reports 1006, not clean, no `error` event.
8. The timer starts at the handshake and restarts at each PONG, which arrives 3 to 6 ms after its PING. A connection whose k-th PING is lost closes at 40·k + 10 s, plus a few ms per answered PING. That gives the 50, 90, 130 ... 530 s lifetimes.

On the client the 8 bytes passed without a trace. The inflate succeeded, because the bytes were compressed as part of the message, so Firefox had no frame to reject. `totw.html` derives the sample count from `byteLength` (`totw.html:5187-5188` for RX audio, `totw.html:6242-6244` for IQ) and never compares it with the header's `length` field. The extra bytes became one more stereo float32 sample or one more I/Q pair.

Why iu3qez/deskhpsdr PR 34 removes it:

- `_lws_validity_confirmed_role()` returns at `wsi-timeout.c:319` when `secs_since_valid_hangup` is 0, before it arms `sul_validity`. The only other place that arms a connection's `sul_validity` is `lws_validity_cb()` itself (`wsi-timeout.c:302-305`), which never runs if the timer was never armed. The listener arms its own `sul_validity` (`lib/roles/listen/ops-listen.c:173`), which belongs to the listen socket. So libwebsockets issues no PING, `send_check_ping` stays 0, and the close at `wsi-timeout.c:278` cannot happen.
- TCP keepalive probes are TCP segments with no WebSocket payload. They never pass through `rops_write_role_protocol_ws()` or permessage-deflate. `TCP_USER_TIMEOUT` and `TCP_RXT_CONNDROPTIME` act on unacknowledged TCP data, also below the WebSocket layer.

What changes for a dead peer:

- Before iu3qez/deskhpsdr PR 34, the validity timer closed a peer that stopped answering 50 s after its last PONG. The same timer also closed healthy connections, as above.
- After iu3qez/deskhpsdr PR 34, only TCP closes it. While TCI streams, the connection is never idle, so a dead peer shows as unacknowledged data, not as failed keepalive probes (per iu3qez/deskhpsdr PR 34). Linux drops it when `TCP_USER_TIMEOUT` (60000 ms) expires. On macOS the kernel drops it at the first retransmission timeout after 60 s of retransmission, not at exactly 60 s (per iu3qez/deskhpsdr PR 34). On an idle connection the first probe goes out after 30 s, then 2 more 10 s apart. `TCP_KEEPCNT` is the total number of probes sent before the drop (tcp(7)), so with no answer the connection is dropped at 60 s, which is `TCI_LWS_DEAD_PEER_S` (`src/tci.c:90`).
- `LWS_CALLBACK_CLOSED` releases `tci_tx_owner` (`src/tci.c:7740-7743`), together with the IQ stream owner and the RTTY owner. The same TCP limit therefore bounds how long a client that has vanished holds TX. Before iu3qez/deskhpsdr PR 34 that bound was the 50 s validity hangup. The time after iu3qez/deskhpsdr PR 34 was not measured.

## Prevention

1. **Read a Firefox close by three fields.** From Firefox's WebSocket source, as read in PR #24 and recorded at `totw.html:4152-4157`:

   | What happens | `code` on the page | `wasClean` | `error` before `close` |
   |---|---|---|---|
   | The server sends a close frame | the server's code | true | no |
   | TCP FIN with no close frame | 1006 | false | no |
   | Firefox rejects a frame (for example an inflate failure), or a socket read error | 1006 | false | yes |

   In the third case Firefox sends 1002 to the server. That code is visible only on the wire. PR #24 logs all three fields for every close, and `WSDIAG` keeps the last 50.

2. **Lifetimes of the form a + b·k s point to a timer.** Measure them from `LWS HANDSHAKE` to `LWS CLOSED` in `deskhpsdr.log`, to the millisecond. If they fall on a + b·k, read the timer defaults of the library first. Here b = 40 s is `secs_since_valid_ping` and a + b = 50 s is `secs_since_valid_hangup` (`lib/core/context.c:1426-1427`).

3. **Run a close diagnosis with `rigctl_debug` on.** `LWS CLOSED client=N` is logged only with `rigctl_debug` (`src/tci.c:7718`). Since iu3qez/deskhpsdr PR 34 the same switch puts libwebsockets warnings and notices in `deskhpsdr.log` (`src/tci.c:7629-7636`), including `VALIDITY TIMEOUT EXPIRED ON` (`wsi-timeout.c:270`).

4. **A capture that keeps only small packets cannot show a control frame written into a data message.** The filter drops the fragments that carry it. Without the filter the bytes are compressed inside the message and show only after inflating it. Do not infer from RFC 7692 what this libwebsockets build puts on the wire: it can send a PING's payload as data (`ops-ws.c:1806-1824`). The check that found the bytes compared each message's length with its header (next rule).

5. **`totw.html` does not compare `byteLength` with the header `length` field.** deskHPSDR writes `length` as a count of float32 values (`src/tci.c:800` for IQ). It is the sixth `uint32_t` of `TCI_STREAM_HEADER`, at offset 20 (`src/tci_audio.h:67-77`). A message is therefore 64 + 4 · `length` bytes. `rxAudioFrame()` reads `length` into `audioDiag.lastSampleCount` (`totw.html:5162`) but sizes the payload from `byteLength` (`totw.html:5187-5188`). The IQ path does the same (`totw.html:6242-6244`). A message longer than its header says passes with no error and no log. The iu3qez/deskhpsdr PR 34 test client found the 8 PING bytes with exactly this comparison. Comparing the two in the client would expose this class of server defect from the browser.

6. **Do not use libwebsockets validity PINGs with permessage-deflate and continuous binary streams.** At `5.0.99-sai-d7f7fdeaf`, a PING issued while a compressed message drains is sent as payload of that message (`ops-ws.c:1806-1824`). `tci_lws_server()` keeps `secs_since_valid_hangup = 0`, and the comment at `src/tci.c:7851-7861` gives the reason. Dead-peer detection stays with TCP keepalive. The rule covers the control frames libwebsockets writes itself in `rops_handle_POLLOUT_ws()` before the drain step. It does not cover a PING that deskHPSDR queues itself: `tci_send_ping()` goes through `tci_queue_frame()` (`src/tci.c:7057-7060`), and a queued frame is written from the writeable callback. `rops_handle_POLLOUT_ws()` returns while `tx_draining_ext` is set (`ops-ws.c:1570-1576`; the comment at `ops-ws.c:1566` says it blocks the writeable callback to keep payload order) and reaches the writeable callback only after the drain (`ops-ws.c:1580-1583`). The one exception is a connection already closing, `LRS_RETURNED_CLOSE`, which returns to the writeable callback before the drain check (`ops-ws.c:1559-1560`). This matters for the server ping of KTD13 in `docs/plans/2026-09-16-0316-feat-totw-modular-operable-client-plan.md:240` (source reading, not tested).

7. **macOS builds against Homebrew libwebsockets do not reproduce this.** Homebrew libwebsockets 5.0.0 defines `LWS_WITHOUT_EXTENSIONS` (`/opt/homebrew/include/lws_config.h:237` on the development Mac). It has no permessage-deflate, and `ops-ws.c:1805-1840` is not compiled. deskHPSDR logs which case applies at start: `permessage-deflate available` or `permessage-deflate not available in this libwebsockets build` (`src/tci.c:7892`, `7894`). A reproduction on a Mac needs libwebsockets built with `build-libwebsockets.sh` and the Makefile from iu3qez/deskhpsdr PR 35.

Still open:

- No long bench session on the new deskHPSDR build. Rows K3 (RX AUDIO off, no click) and K4 (a close on its own, no click, DIAG JSON copied into #22) are empty (`docs/bench/reference-checklist.md:109-110`). K4 as written waits for the #22 close every 1.5 to 9 minutes, which iu3qez/deskhpsdr PR 34 removes.
- The dead-peer time was not measured. It needs packet loss. The iu3qez/deskhpsdr PR 34 test did not run on Linux or with a real radio.
- The libwebsockets defect at `ops-ws.c:1806-1824` has not been reported to the libwebsockets maintainers (per iu3qez/deskhpsdr issue 33). The PONG and close-frame writes in `rops_handle_POLLOUT_ws()` (`ops-ws.c:1485-1487`, `1522-1523`) also come before the drain step and also go through `rops_write_role_protocol_ws()`. By the same source reading, they would be changed to `LWS_WRITE_CONTINUATION` too if `tx_draining_ext` is set. For the close frame this is narrower: `rops_handle_POLLIN_ws()` clears `tx_draining_ext` once the state is `LRS_WAITING_TO_SEND_CLOSE` or `LRS_RETURNED_CLOSE` (`ops-ws.c:1132-1143`), so the close frame goes out with the flag set only when a POLLOUT pass comes first. iu3qez/deskhpsdr PR 34 does not change any of this, and it was not tested. A browser cannot send a WebSocket PING from JavaScript, so TOTW never makes libwebsockets write a PONG. A non-browser TCI client that sends PINGs could.
- The closing comment on #22 has not been edited. It still says that some PINGs never reached the wire and that the permessage-deflate suspect is ruled out. The What Didn't Work section above gives the correct version.

## Related Issues

- #22: symptom, candidates, closing comment.
- PR #24: client close record and RX fade-out.
- iu3qez/deskhpsdr issue 33: server-side diagnosis, capture and source reading.
- iu3qez/deskhpsdr PR 34: the fix and the loopback verification.
- iu3qez/deskhpsdr PR 35: Makefile include order for a local libwebsockets on macOS.
- iu3qez/deskhpsdr issue 3: bench checks that need another machine, including a libwebsockets build with extensions off the Mac.
- `docs/solutions/integration-issues/libwebsockets-permessage-deflate-assert-crashes-deskhpsdr-ptt.md`: the RX-side permessage-deflate crash that moved the station to the same libwebsockets build.
- `docs/plans/2026-09-16-0316-feat-totw-modular-operable-client-plan.md`: KTD13 and unit U12, the server keepalive, written before iu3qez/deskhpsdr PR 34.
