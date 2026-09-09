# The Wine binaries in this application, and their licence

`winebus.sys`, `setupapi.dll`, `ntoskrnl.exe` and `winebus.so` — shipped
inside the application bundle as `engine-controller-winebus.sys`,
`engine-controller-setupapi.dll`, `engine-controller-ntoskrnl.exe` and
`engine-controller-winebus.so` — are **Wine**, and Wine is
**LGPL-2.1-or-later**.

They are not ours in the sense the licence cares about. They are somebody else's
program with eight patches of ours applied, and both halves of that sentence
carry obligations. They are redistributed because a fix that only reaches people
who can build Wine is not a fix; the licence permits that and requires this
notice to travel with the binaries, which is why a copy of this file sits inside
the application bundle and not only in this repository.

## What they were built from

- **Upstream project**: Wine — <https://www.winehq.org/>
- **Licence**: GNU Lesser General Public License, version 2.1 or later.
  The full text is `COPYING.LIB` in any Wine source tree, and is published at
  <https://www.gnu.org/licenses/old-licenses/lgpl-2.1.html>.
- **Source tree**: the Wine source of **CrossOver 26.3.0.39832**, revision
  **`wine-11.0-8726-g2e2f5fca349`**. That build string is recorded beside the
  binaries in `engine-controller-built-for.json`, inside the bundle, so a copy
  of the application always says what it was made from. CrossOver's Wine sources
  are published by CodeWeavers, who distribute CrossOver.
- **Patches applied on top**, all eight of them ours:

  | patch | what it changes |
  |---|---|
  | `mgvf-0002` | `winebus.sys` names the bus a device is on in its compatible ids, so `BTHENUM` is there for a client to find |
  | `mgvf-0003` | `setupapi.dll` answers `CM_Get_Parent` for HID children, which was a stub returning "no such device node" |
  | `mgvf-0004` | `ntoskrnl.exe` refreshes a device's hardware and compatible ids on every enumeration, not only the first time that device is ever seen |
  | `mgvf-0005` | `winebus.sys` can present a DualSense on Bluetooth as if it were on USB — per device, off unless a registry value asks for it |
  | `mgvf-0007` | that presentation keeps requests for the pad's wired-only audio hardware off the wire |
  | `mgvf-0008` | and answers a feature write as successful without sending it, which was an experiment and is described as one |
  | `mgvf-0006` | `winebus.so` takes a Bluetooth Sony pad from macOS while a bottle holds it, so only one program writes to it |
  | `mgvf-0009` | `winebus.sys` can rewrite a game's choice of vibration path to the stronger legacy motors, and scale the motors by a percentage; both off unless asked for |

- **Built and stripped by** `scripts/build-controller-bus.sh`, which refuses to
  stamp a binary unless its export and import tables are identical before and
  after stripping.

## Where the patches are

In full, as `source-patches/mgvf-0002-*.patch` … `mgvf-0009-*.patch`, in the
sibling project:

> **<https://github.com/MathiasKowoll/MacGameVideoFix>**

They modify LGPL code and are LGPL themselves. `mgvf-0001` is not among them:
it belongs to the sibling project's media work and is not in these binaries.

Anyone who wants to rebuild these four files needs the CrossOver Wine source
above, the eight patches, and the build script. Nothing in this application is
required to do it, and nothing about it is secret.

## The application itself

`MacGamePadFix.app`'s own code — everything that is not one of those four
binaries — is **GPL-3.0-or-later**; see `LICENSE`. It is one Swift file and a
build script, and for now they live beside the patches, in
`app-padfix/` of the MacGameVideoFix repository above, because the build copies
the four binaries straight out of that tree and one copy of a binary is better
than two.

## The icon

The application icon comes from MacGameVideoFix and is covered by that
project's licence.
