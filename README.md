# MacGamePadFix

**A PlayStation controller on Bluetooth does not rumble in Windows games under
CrossOver on a Mac. This installs three patched Wine files into one CrossOver
and makes it work.**

Nothing else. It does not know about games, it does not launch anything, it does
not phone anywhere. It replaces three files inside a CrossOver you point it at,
keeps the originals beside them, and puts them back when you ask.

---

## The problem

Connect a DualSense by cable and it rumbles in every game. Connect the same pad
by Bluetooth and it never rumbles — in any game, on any bottle. The PS button
and the touchpad go quiet too. People have blamed the games, the pad, Steam and
CrossOver in turn.

It is none of them.

## What is actually wrong

A DualSense speaks a different language on each transport. Over Bluetooth it
wants output report `0x31`, 78 bytes, ending in a CRC. Over USB it wants report
`0x02`, and it **silently ignores** the wrong one — no error, no complaint, just
a pad that never buzzes.

So every Windows program that drives a controller has to know which transport it
is on, and they all ask the same question: they walk up from the HID device to
its **parent** in the device tree and look for `BTHENUM` among the parent's
compatible ids. Windows answers. Under Wine, nobody could:

- `CM_Get_Parent` was a **stub** that returned "no such device node", so there
  was no parent to ask;
- and Wine's own bus driver never named the bus a device was on anyway, so even
  reaching the parent would have got no answer.

The result was always the same: "not Bluetooth". Every client then sent the USB
report, and the pad ignored it. Steam says so in its own log, for a pad sitting
on Bluetooth:

```
Added HIDAPI device 'DualSense Wireless Controller' … bluetooth 0
```

There is a third piece, and it is why the first two are not enough on their own.
Wine wrote a device's hardware ids **only the first time it ever saw that
device**. A pad that had been plugged in once already carried the old answer in
the bottle's registry, and kept giving it no matter what the bus said now.

## What this fixes

Three patches, one per piece:

| | what it changes |
|---|---|
| `mgvf-0002` | the bus driver names the bus in a device's compatible ids: `BTHENUM…` on Bluetooth, `USB\Class_03` on a cable |
| `mgvf-0003` | `CM_Get_Parent` is implemented for HID children, so there is a parent devnode to ask |
| `mgvf-0004` | hardware and compatible ids are refreshed on **every** enumeration, not just the first sighting |

With them installed, Steam's log reads `bluetooth 1`, and every client builds
the report the pad actually wants:

**Rumble, the PS button and the touchpad work over Bluetooth.** Measured on
2026-09-08 in every title tried, on a DualSense and a DualSense Edge. Trigger
effects travel in the same report; a game that sends them over Bluetooth gets
them through.

Nothing here is specific to one game. The patches never look at which controller
it is — they key on the bus — so a DualShock 4 on Bluetooth gets the same truth
told about it.

## What it does not fix

**A game that demands a cable still demands a cable.** Some titles drive the pad
through Sony's own PC library, which only accepts a wired DualSense and refuses
a Bluetooth one before any of this comes into play. Steam knows which those are
and says so before launch: *"the developer has indicated that this game only
supports your DualSense controller over USB"*. That is the game's own limit, the
same on Windows, and plugging the cable in is the answer there.

The set also carries an **experimental** ability to present a Bluetooth pad to
Windows as though it were wired, for exactly those titles. It is **off unless
you turn it on**, per controller, with a registry value inside a bottle, and it
is not what this application installs or offers. It made adaptive triggers work
over Bluetooth in one such title; it is not finished, and it is documented in
the repository the patches live in rather than here.

**And the pad's speaker and microphone stay wired-only.** Those are USB audio
hardware on the pad. No patch reaches them, on any system.

## Which CrossOver

**Stable CrossOver 26.3.0.39832 only.** The application refuses every other
version and shows you the refusal.

That is not timidity. These are Wine binaries built from the Wine source of that
exact CrossOver, replacing three files the rest of that same Wine is compiled
against. Mixing Wine binaries across versions does not fail loudly: it gives you
a bottle that starts, runs, and then misbehaves somewhere nobody would ever
connect to a controller patch. Refusing is the only honest answer.

It refuses **while a bottle is running**, too, in both directions. Two of the
three files are loaded inside a running bottle's `winedevice.exe`. Quit Steam,
let the bottle shut down, and try again.

## What it does to CrossOver, in plain words

- Replaces `winebus.sys`, `setupapi.dll` and `ntoskrnl.exe` inside the
  application, and keeps CrossOver's own three beside them as `.mgvf-stock`.
- **Re-signs the application ad hoc**, because replacing files inside a signed
  bundle breaks its seal and macOS would otherwise call it damaged. CodeWeavers'
  signature is gone until you restore.
- **Restore** puts their three files back and re-signs again. Nothing is lost.
- **A CrossOver update overwrites all of it.** Run the install again afterwards.

## Running it the first time

The application is signed ad hoc, not notarised, so macOS will not open it on a
double-click. **Right-click it, choose Open, then Open again** in the dialog
that appears. If macOS refuses anyway, System Settings → Privacy & Security has
an *Open Anyway* button after the first attempt.

Then: pick your CrossOver, read what it says is installed now, and press
Install. Quit Steam first.

## Where the binaries come from, and the licence

The three files are **Wine**, which is **LGPL-2.1-or-later** — somebody else's
program with our patches on top. They are redistributed here because a fix that
only helps people who can build Wine is not a fix.

The patches, in full, and the script that builds and strips the binaries, live
in the sibling project:

> **<https://github.com/MathiasKowoll/MacGameVideoFix>** — `source-patches/`

`THIRD-PARTY-LICENCES.md` in this repository, and a copy inside the application
bundle, records exactly which Wine revision and which patches produced the
binaries you are running.

The application itself is **GPL-3.0-or-later**; see `LICENSE`. Its source is
one Swift file and a build script, kept beside the patches in `app-padfix/` of
that same repository, so that the three binaries have exactly one home. This
repository is where the built application is published.

## Status

**0.1.0, a pre-release.** The three patches it installs are measured and have
been in daily use on the author's machine; the application around them is new.
It has been run against a real CrossOver and a stock one, and its refusals have
been exercised, but it has not been used by anyone else yet.
