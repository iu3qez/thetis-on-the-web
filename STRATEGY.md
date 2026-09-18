---
name: Thetis On The Web
last_updated: 2026-09-13
---

# Thetis On The Web Strategy

Thetis On The Web is the browser interface of the remote station described in the deskHPSDR requirements; this document covers only the client.

## Purpose

The operator wants to use their own HPSDR station remotely, from a laptop, with a radio interface that lives entirely in the browser. The crux is the link: existing clients assume a LAN, and full-resolution spectrum and audio cost megabits that a 4G link does not have.

## Positioning

We own both ends of the link: the server computes what the client needs to show and sends only that, so the client stays a browser file that works even over 4G. The critical paths, audio and Morse, do not go through here; everything else is best effort.

## Users

**Primary:** The radio amateur who owns the station, alone, away from the shack - Relies on Thetis On The Web to run a QSO as if sitting at the radio: see the band, tune, drive the station.

## Boundaries

- Compatibility with Thetis or other TCI servers: the client works only with deskHPSDR.
- iPad, phones and iOS: the target is a laptop with a desktop browser.
- Audio and Morse in the browser: they travel over their own transports, outside this client.
- Multiple operators: not at this stage.
- Inherited features that call external services: no investment; they are removed if they cost bandwidth or cause problems.
- The original project: not followed and not referenced.

_Resist a change when:_ it brings a critical path into the client, ties the client to a server other than deskHPSDR, or spends 4G bandwidth on something the station does not use.

## Key metrics

- **Complete remote QSOs** - contacts completed without going back to the shack; from the station log.
- **Sessions lost to the link** - remote sessions abandoned because spectrum, audio or control were unusable; noted by the operator.
- **Bandwidth per session** - average kbit/s of a typical session with the bin spectrum, capped at 50; from the client's network monitor.
- **Command latency over 4G** - time between an action at the console and the server's confirmation; measurement still to be introduced.

## Tracks

### Low-bandwidth spectrum

The spectrum computed by the server arrives as bins and feeds the trace: a bin stream, a span that follows the zoom, an explicit choice between bins and IQ.

_Why it serves the approach:_ it is the work moved to the server, and it is what brings the spectrum under the bandwidth cap.

### Physical panel

The station is driven from a MIDI console as if sitting at the radio: jog on the VFO, two-state controls, status LEDs.

_Why it serves the approach:_ band and attenuator go through the extended commands that exist only because the server is ours.

## Brand

**One-liner:** Fast and lean
