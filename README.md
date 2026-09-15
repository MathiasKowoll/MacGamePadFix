# MacGamePadFix

**A PlayStation controller on Bluetooth is unreliable in Windows games under
CrossOver on a Mac: rumble and the adaptive triggers do not always work, and
which titles they work in is not something you can predict. This installs ten
patched Wine files into one CrossOver and makes them work.**

Nothing else. It does not know about games, it does not launch anything, it does
not phone anywhere. It replaces ten files inside a CrossOver you point it at,
keeps the originals beside them, and puts them back when you ask.

---

## The problem

Connect a DualSense by cable and it rumbles. Connect the same pad by Bluetooth
and it is a lottery: some titles rumble, others do not, and the adaptive
triggers behave the same way. The PS button and the touchpad can go quiet too.
People have blamed the games, the pad, Steam and CrossOver in turn.

It is none of them, and the inconsistency is the clue.

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

## And rumble in games that have never heard of a DualSense

The three patches above tell the truth about the bus, which is what a game needs
if it drives the pad **as a DualSense**. Most Windows games do not. They ask
XInput for "controller 1" and expect an Xbox pad, and XInput had no motors to
offer them, because a DualSense's motors are not where an Xbox pad keeps them.

Six more patches give it some. The bus driver adds a small haptics device beside
the pad — the pad keeps every byte of its own descriptor, and the motors arrive
next door on a device of their own — `hidclass` offers that device to XInput as
well as to everything that had it before, and `xinput1_3` learns that a gamepad's
axes and buttons come in two conventions rather than one.

**So a game that only speaks XInput now rumbles a DualSense over Bluetooth.**
Measured on titles that had no rumble here at all before it.

It costs nothing in frames, and that took work to be able to say. A Bluetooth
link to a pad carries about sixty-five reports a second and no API moves it, so
a second program writing to the pad every frame is offering more than the radio
drains, and everybody pays — including the game, on the thread that carries its
own writes. The answer is not to write less often. It is to not write at all:
the motors are stamped **into a packet the game is already sending**, so nothing
of ours is ever added to the link. Measured: 92-95% of every motor change
carried that way, and the frame cost gone.

Off by default, and it is one value under the pad's key, `XInputRumble`.

It used to have to be left off for any game using Sony's own library: the small
haptics device carried the pad's vendor and product ids, that library finds its
pad by reading exactly those, and it saw two DualSense where one answered
nothing. Since 0.2.2 the device is offered to XInput and to nothing else, and the title
that would not start with this on now does.

It coexists with **Steam Input**, which was worth checking rather than assuming:
Steam presents a virtual pad that is itself an XInput device, so this adds a
second one carrying motors and no sticks, and a game could in principle pick the
wrong one. Measured on a title that reaches its controller only through Steam
Input, with both on: it rumbles and plays normally.

**And a game that uses Sony's library does not need it either way** — it drives
the pad itself and already rumbles over Bluetooth, which is what the three
patches above are for. This is for the games that have never heard of a
DualSense.

## How much of the rumble reaches the motors

A DualSense on Bluetooth is written over a link that carries about sixty-five
reports a second, so this driver holds back a motor change too small to feel
rather than spending a slot on it. That band is four parts in 255 — and until
0.2.1 it was comparing the wrong number.

The strength setting multiplies what a game asks for, and it is applied *after*
the band has already decided. So the band meant four parts in 255 at the neutral
point and forty at the top of the range: **turning the strength up made the
rumble coarser instead of stronger.** Measured, at ten times: 632 changes held
in one session, and half of them would have been ten parts or more at the
motors.

It now measures what the pad will actually receive, and it no longer holds back
a change that rides inside a packet the game was sending anyway — those cost
nothing, so there was never anything to save. Measured across two runs of one
title with only that between them: **three and a half times as much of what the
game asks now reaches the motors**, and the frame rate is unchanged.

## Which way the pad vibrates

A DualSense knows two ways to be asked. The **modern** path is the pad's own,
the one Sony's library asks for on every packet it sends, and it is the finer of
the two. The **legacy** path asks the pad to imitate a pair of rotating-mass
motors, and it is clearly harder at the same command — measured by running one
Sony title twice with nothing changed but that.

The modern path is what you get. `VibrationMode` under the pad's key asks for
the other, and `VibrationGain` is a percentage over the motors where 0 is
silence for every game at once. Both are preferences and neither is a repair.

There is no third setting hiding anywhere. Sony's own library was taken apart to
be sure of that: it writes two motor bytes and one path bit, and nothing else.

## What else is in it

Five more patches came out of using it, and they are described in full in the
repository the sources live in. Two of them change what you feel, and one of
them you should know about before you install:

- **`mgvf-0006` takes the pad for the bottle.** While a game is running, the
  game owning the controller exclusively is the right behaviour -- two
  programs writing to one pad over one Bluetooth pipe is the anomaly, and it
  was measured here as 163 write timeouts in a day. The cost is real and worth
  knowing: **while a bottle is running, macOS and its own applications cannot
  use the pad.** It is released when the bottle shuts down, which for most
  people means when they quit Steam.
- **`mgvf-0005`, `mgvf-0007` and `mgvf-0008` present a Bluetooth pad as a
  wired one**, for the titles that only accept a wired one. **Off by default,
  and leave it off unless you are experimenting**: it does make those titles
  accept the pad, and in all three of them measured here the pad then dropped
  its Bluetooth link within a minute. Making Sony's own library engage a pad
  over Bluetooth is what makes the pad leave, and that is not solved.

## What it does not fix

**A game that demands a cable still demands a cable.** Some titles drive the pad
through Sony's own PC library, which only accepts a wired DualSense and refuses
a Bluetooth one before any of this comes into play. Steam knows which those are
and says so before launch: *"the developer has indicated that this game only
supports your DualSense controller over USB"*. That is the game's own limit, the
same on Windows, and plugging the cable in is the answer there.

The experimental wired presentation described above is for exactly those
titles, and it is honest about its price: it made adaptive triggers work over
Bluetooth in one of them, and the pad left within a minute in all three.

There is a second limit worth knowing, and it belongs to the games rather than
to this. A title that reads **XInput** cannot be given a PlayStation pad by any
of this: Windows has no way to describe one through XInput, and wine's own
XInput cannot even make a DualSense rumble, because it looks for haptic usages
that no DualSense declares. Steam Input is the answer there, on this stack and
on Windows, and it costs the PlayStation button glyphs because the game
genuinely sees an Xbox pad.

**Lights are not chosen by default.** Since `mgvf-0031` the lightbar colour and
the player number can be set per pad model, with `LightbarColour` and
`PlayerLights` under the pad's key; they replace what a game or Steam Input
already sends, so a title that never sends a light change gets nothing yet.

**A pad nobody is using is asked to turn itself off.** Since `mgvf-0033`, a
DualSense on Bluetooth that the bottle holds is sent the pad's own power-off
request after 20 minutes with no stick, trigger or button input
(`IdlePowerOffMinutes` under the pad's key: 0 for never, anything else clamped
to 5 to 240). Moving only the gyro or the touchpad does not count. The same
request, sent with nothing but macOS running, turned off a DualSense and a
DualSense Edge within a tenth of a second; inside a game it is not measured yet.
It does nothing once the bottle has shut down.

**And the pad's speaker and microphone stay wired-only.** Those are USB audio
hardware on the pad. No patch reaches them, on any system.

## Which CrossOver

**Stable CrossOver 26.3.0.39832 only.** The application refuses every other
version and shows you the refusal.

That is not timidity. These are Wine binaries built from the Wine source of that
exact CrossOver, replacing ten files the rest of that same Wine is compiled
against. Mixing Wine binaries across versions does not fail loudly: it gives you
a bottle that starts, runs, and then misbehaves somewhere nobody would ever
connect to a controller patch. Refusing is the only honest answer.

It refuses **while a bottle is running**, too, in both directions. Two of the
of them are loaded inside a running bottle's `winedevice.exe`. Quit Steam,
let the bottle shut down, and try again.

## What it does to CrossOver, in plain words

- Replaces `winebus.sys`, `setupapi.dll`, `ntoskrnl.exe`, `hidclass.sys`, the
  five `xinput` DLLs and `winebus.so` inside the application, and keeps
  CrossOver's own ten beside them as `.mgvf-stock`.
- **Re-signs the application ad hoc**, because replacing files inside a signed
  bundle breaks its seal and macOS would otherwise call it damaged. CodeWeavers'
  signature is gone until you restore.
- **Restore** puts their ten files back and re-signs again. Nothing is lost.
- **A CrossOver update overwrites all of it.** Run the install again afterwards.

## Running it the first time

The application is signed ad hoc, not notarised, so macOS will not open it on a
double-click. **Right-click it, choose Open, then Open again** in the dialog
that appears. If macOS refuses anyway, System Settings → Privacy & Security has
an *Open Anyway* button after the first attempt.

Then: pick your CrossOver, read what it says is installed now, and press
Install. Quit Steam first.

## Where the binaries come from, and the licence

The ten files are **Wine**, which is **LGPL-2.1-or-later** — somebody else's
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
that same repository, so that the ten binaries have exactly one home. This
repository is where the built application is published.

## Status

**0.2.4, a pre-release.** Twenty-nine patches now. The three that tell the
truth about the bus have been in daily use on the author's machine since
September and are the settled part. The XInput rumble is newer and was measured
rather than guessed at every step — the frame cost, the stop behaviour, the two
paths and the pad's own power field each have a number behind them — but it has
been exercised on a handful of titles by one person, on one DualSense and one
DualSense Edge. The wired presentation is still explicitly unfinished.

One thing 0.2.2 also removed: the motors device used to declare a button that is
never pressed, and a game reading it could stop holding a button down — a title
here would not keep L3 held while this was on. It declares none now, and
`xinput` no longer refuses a device for having no buttons to count.

The narrow half of `mgvf-0028` is reasoned rather than exercised: the three
conditions that mark a device as carrying motors and nothing else were read from
the descriptor this project builds, and there is no real force-feedback wheel
here to check that a genuine one is left alone.
