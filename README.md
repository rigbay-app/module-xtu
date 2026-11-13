# XTU profiles module for Rigbay

Switch Intel XTU power-limit profiles (PL1, PL2, time window) from [Rigbay](https://github.com/rigbay-app/rigbay), and read CPU
temperature and power. Version 0.1.2 (pre-release), see [CHANGELOG.md](CHANGELOG.md). **Needs Rigbay 0.1.5 or newer** (host API 0.1.4): Rigbay 0.1.0 and 0.1.1 do not check that and would offer it, and 0.1.2 and 0.1.3 refuse it, so update Rigbay first.

**For other modules (host API 0.1.4).** `rigbay xtu status` and `list` are declared read-only. `rigbay xtu apply <profile>` is declared as a write of the CPU power limits (hardware); its preview is the profile's own values ("Gaming: PL1 45 W · PL2 65 W · 28 s", from the profile list, never from what is in force now), and a run that changed something reports the profile that was in force before, so the host can put it back with `apply <that profile>`. When the old values matched no profile there is nothing to report and that step is not undone. The monitor (temperature, power, clock, usage) is also read when another module such as Watch asks for it in the background, at the asked interval but never faster than once a second; with nobody asking it is still not read at all. None of this was run with a real plan or another module: the tests use fakes.

What it does, inside the Rigbay app:

- Shows the limits the CPU is running under now (read through the Intel XTU service) and which saved XTU profile they match.
- Applies one of XTU's saved profiles, or a profile you keep in Rigbay, after you confirm the exact values in a dialog. Rigbay offers Revert afterwards.
- Hotkeys and tray items per profile, and optional automatic switching when you plug in or unplug the charger.
- CPU temperature, package power, usage, clock, PL1 and PL2 (and more per-core values if you turn them on) on Rigbay's metrics bus, so the dashboard and recorders can show them.
- `rigbay xtu status`, `rigbay xtu list` and `rigbay xtu apply <name>` on the command line.

The module runs in the Rigbay process (.NET 10) and talks to the XTU service (`XTU3SERVICE`) over its named pipes. **It contains no Intel file**: it
declares its own copies of the few contract types it uses (names and member shapes only, in `Rigbay.Module.Xtu/Intel/Contracts.cs`), so it builds and
its tests run on any Windows machine, including GitHub's hosted runners. To use it you still need Intel XTU installed and its service running.

> Looking for a small stand-alone tool that does not need Rigbay? The older app
> [minhdevtree/xtu-profile-switcher](https://github.com/minhdevtree/xtu-profile-switcher) (tray and window, the same four controls) stays
> available and is a good alternative; this module ports its logic.

## What is verified, and what is not

Verified (on one machine; the reading check on 2026-11-01 with 0.1.0, the build and tests again for 0.1.1 and 0.1.2 on 2026-11-13):

- **Build and unit tests:** 0 warnings, 0 errors; 236 tests pass (0.1.2, from a fresh clone with the submodule) with no hardware and no Intel software (fakes stand in for the XTU service and for Rigbay).
  The redesigned page of 0.1.1 (profile as the main control, tiles, chart) is covered by page tests with those fakes; it was not run against the real XTU service for this release.
  Twenty of them compare the module's contract copies with a recording of the real service's wire format (`Rigbay.Module.Xtu.Tests/wire/`, made once, read-only).
- **Reading against the real service:** loaded by the real Rigbay app, the module read the same limits and profile list as the old app (`rigbay xtu status`
  and `xtu-switcher --status` printed identical text), and the page and dashboard showed live readings matching the old app's recording
  (evidence in `docs/shots/`).

**Not verified: any write.** The module has never applied a profile to the real XTU service. Every live check was run with writes switched off, and
the write path (`ProposeChange`, `ApplyChanges`, `DiscardChanges`) is exercised only against fakes and against the serializer output of Intel's own types.
It reuses the logic of the old app, which does work, but that is not the same as having run this module. Also not seen working: hotkeys and tray items
on screen, auto-switch on plug/unplug and resume, `rigbay xtu apply` from a real command line, XTU being stopped or updated while the module runs,
and high-DPI use. The full list is in [docs/README.md](docs/README.md) ("Not verified"). If the first real apply fails, the first suspect is the contract
copy in `Intel/Contracts.cs`.

## Supported machines

- Windows 10 or 11, x64, with **Intel XTU installed** and its service running, on an Intel CPU that XTU can tune. Rigbay itself must run as administrator
  (the module declares `admin: true`).
- **Tested on one machine only:** an Acer Predator Triton 500 SE laptop (Core i7-11800H), Windows 11, XTU 7.14.2.135. Other CPUs, other XTU versions and
  desktops are untested; a new XTU version may change a contract, in which case calls fail at run time rather than at build time.
- The module's probe checks the XTU service and its two pipes and reports the machine as unsupported if they are missing.

## Safety: what the module can change

It writes exactly **four XTU controls**, nothing else, and only after you approve the exact values:

| Control | Range the module accepts |
| --- | --- |
| PL1 (long-term power limit) | 15 to 120 W |
| PL2 (short-term power limit) | from PL1 up to 150 W |
| PL2 enable | on or off |
| Time window | 1 to 128 s |

- A value outside these ranges never reaches the service. The service has its own limits on top.
- Any other control id is refused by the code (an allow-list, tested).
- Every change is confirmed first: from the page, a hotkey, a tray item or `rigbay xtu apply`. A declined or unanswerable request (for example a headless run) writes nothing.
  The one exception is auto-switch, which is off by default and applies only after you turned it on and chose its profiles.
- A **write latch**: with the environment variable `RIGBAY_XTU_NO_WRITE=1`, or a file `no-write.flag` in the module's data folder, every apply fails before it asks anyone.
- Nothing is written when the module starts. Higher limits mean more heat and fan noise; start with values the machine already runs.

The registry keys and files it reads are listed in `module.json` and shown by Rigbay before you accept the module.

## Build

Needs Windows and the .NET 10 SDK (the host contract project `Rigbay.Abstractions` targets `net8.0`; the SDK fetches its targeting pack from NuGet).
The host contract comes from the git submodule `external/rigbay` (pinned to a commit of the Rigbay repository), so clone with submodules:

```powershell
git clone --recurse-submodules https://github.com/rigbay-app/module-xtu.git
cd module-xtu
dotnet build Rigbay.Xtu.slnx --configuration Release
dotnet test Rigbay.Xtu.slnx --configuration Release
```

The build output of `Rigbay.Module.Xtu` (`Rigbay.Module.Xtu\bin\Release\net10.0`) is the module folder. `Rigbay.Abstractions.dll` is not copied into it:
the host supplies its own.

## Install into Rigbay

Rigbay reads modules from `%ProgramData%\Rigbay\modules\<id>\`, and the folder name must equal the manifest id. Copy the whole build output folder to
`%ProgramData%\Rigbay\modules\xtu\` (so that `module.json` sits at the top of it), start Rigbay as administrator and open the module; the first time it lists
what the module declares it uses and asks you to accept it. Keep an empty file named `no-write.flag` in the module's data folder (next to its `settings.json`,
(`%LOCALAPPDATA%\Rigbay\modules\xtu\`) while you are only trying it out.

## Documents and tools

- [docs/README.md](docs/README.md): design, page, metrics, commands, verification record, and the one reversible live-apply test that is still to be run.
- [docs/CLITE-PLAN.md](docs/CLITE-PLAN.md): how the build dependency on Intel's files was removed. [docs/RECORDER-NOTES.md](docs/RECORDER-NOTES.md): the module's metric ids for a recorder.
- `tools/live-read.ps1`: read-only live check through a published Rigbay app (writes stay off). `tools/dump-contracts.ps1` and `tools/record-wire.ps1`:
  print the contract metadata of an installed XTU and record the wire sample (read-only, XTU needed).

The host app is at [rigbay-app/rigbay](https://github.com/rigbay-app/rigbay); the contract this module is built against is `docs/ABSTRACTIONS.md` there.

## License

MIT, see [LICENSE](LICENSE). Intel and XTU are trademarks of Intel Corporation; this project is not affiliated with or endorsed by Intel.
