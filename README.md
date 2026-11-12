# TallyDirector

Tally lights for the sources of a video switcher. A video switcher or a show controller tells **TallyDirector** which source is live, and the tally of that source turns red (Program), green (Preview) or blue (Aux).

This repository holds the installers and the API documentation. The apps update themselves from here.

## The two apps

| App | Runs on | What it does |
|---|---|---|
| **TallyDirector** | One machine on the show network | Follows a video switcher or takes OSC, and tells every tally what to show. This is where tallies get their number and name. |
| **TallyHelper** | Any machine on the network with tallies on USB | Finds the Director on the network and passes its commands to every tally plugged into its machine. Sits in the menu bar (macOS) or the system tray (Windows). |

A tally can also connect over WiFi straight to the Director. Then it needs no TallyHelper, only power.

## Download

Get the newest version from [Releases](https://github.com/markvesterskov/TallyDirector-releases/releases/latest).

| File | For |
|---|---|
| `TallyDirector_<version>_universal.dmg` | TallyDirector on macOS 11 or newer (Apple silicon and Intel) |
| `TallyDirector_<version>_x64-setup.exe` | TallyDirector on Windows 10 or 11 |
| `TallyHelper_<version>_universal.dmg` | TallyHelper on macOS 11 or newer (Apple silicon and Intel) |
| `TallyHelper_<version>_x64-setup.exe` | TallyHelper on Windows 10 or 11 |

The other files (`.json`, `.sig`, `.tar.gz`) are for the built in updater. You do not need them.

## Install

**macOS:** open the `.dmg` and drag the app to Applications. The first time each app starts, macOS asks for access to the local network. Allow it, otherwise the apps cannot find each other.

**Windows:** run the installer. The installers are not signed yet, so SmartScreen shows a warning: choose *More info*, then *Run anyway*. When Windows asks, allow the app through the firewall on private networks.

## First use

1. Start TallyDirector on the Director machine.
2. Start TallyHelper on each machine that gets tallies and plug them into its USB ports. The Helper finds the Director by itself. Its icon turns green when it is connected and has a tally.
3. In TallyDirector, every tally appears under *Devices* with the lowest free device number. Change the number and name there.
4. Under *Settings*, pick your video switcher and enter its IP address, then choose the switcher input on each tally's card. Or send OSC to the Director.

If the Helper stays red, the network probably blocks discovery (mDNS). Type the Director's IP address into the Helper's window.

## Updates

Both apps check for a new version when they start. TallyDirector shows it under *Settings*, TallyHelper in its window. Nothing installs until you click, because an update restarts the app and the tallies show OFFLINE for a few seconds.

New tally firmware comes with TallyDirector. Plug the tally into the Director machine, open *Setup tally* and click *Update firmware*.

## Network

Everything stays on the local network. Keep the Director, the machines running TallyHelper and the WiFi tallies on the same network and allow these ports to the Director machine:

| Port | Use |
|---|---|
| UDP 9000 | OSC into TallyDirector (can be changed in Settings) |
| TCP 9100 | TallyHelpers and WiFi tallies |
| TCP 9101 | The API for other software |
| UDP 5353 | Discovery (mDNS) |

There are no passwords on these ports. Use a network that only the show equipment is on.

## API

Other software (a Bitfocus Companion module, a show controller, your own scripts) can list the tallies, follow their state live and control them. See [docs/api.md](docs/api.md).
