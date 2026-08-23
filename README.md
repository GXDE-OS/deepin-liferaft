# Deepin Liferaft

<p align="center">
  <img src="data/icons/deepin-liferaft.svg" width="128" alt="Deepin Liferaft icon">
</p>

Deepin Liferaft provides a macOS-style "Your system has run out of application memory" dialog for Deepin. It detects sustained memory pressure, pauses application cgroups before the desktop becomes unusable, and lets the user resume or force quit an application.

## Behavior

- Polls the current `user@UID.service` cgroup every second.
- Uses Fedora's workstation systemd-oomd policy: `full avg10` above 50% for 20 seconds, with reclaim activity (`pgscan`) seen in the last 30 seconds.
- Also triggers when system memory and swap usage both exceed 90% and an application uses more than 5% of total swap.
- Lists up to 10 `app-DDE-*` cgroups by `memory.current`.
- Attributes child resources to the application cgroup. A `memhog` process launched in Deepin Terminal therefore increases the Terminal row.
- Reads localized names and icons from XDG desktop files, including the Linglong export directory at `/var/lib/linglong/entries/apps/share/applications`.
- Ranks pressure candidates by the latest `pgscan` delta and then memory, or ranks swap candidates by `memory.swap.current`.
- Freezes up to three candidates with `cgroup.freeze`; Resume thaws the selected group and Force Quit uses `cgroup.kill`.
- Waits 15 seconds after an action before showing another dialog.

Detection follows Fedora's systemd-oomd policy while interaction follows macOS. systemd-oomd kills one cgroup immediately; Deepin Liferaft pauses up to three candidates and leaves the final choice to the user.

## Safety

Deepin Liferaft only thaws cgroups it froze itself. It skips cgroups already frozen by another component and never freezes the cgroup containing its own process.

Closing the dialog thaws every owned cgroup before the window closes. If a thaw temporarily fails, the window remains open and retries. `SIGTERM` and `SIGINT`, including a K9 task stop/disable or session logout, are received through `signalfd` so owned cgroups are thawed before process exit.

Invalid PSI, `memory.stat`, `memory.current`, or `/proc/meminfo` samples do not trigger an action.

## Logging

Deepin Liferaft logs through DTK's `DLog` (journald appender). K9 redirects task stdout/stderr to `/dev/null`, so journald is the primary log destination; run from a terminal, messages also appear on stderr. Log lines carry a timestamp, level, source file, function, and line.

Inspect the daemon's status and logs:

```bash
gxde-k9-chocker status edging deepin-liferaft --system
journalctl --user _COMM=deepin-liferaft -f
```

The journal records program startup (pid and hidden/foreground mode), pressure threshold crossings, trigger decisions with pressure, duration, reclaim and memory usage, every freeze (cgroup, trigger, sort metric), every force quit (kill/thaw result), resume and thaw failures, and shutdown cleanup. Invalid samples are logged as warnings on state change rather than every second.

The K9 task inherits the DDE session environment, and DTK's journald appender writes these messages directly to the user journal.

## Resource Use

`--hidden` mode delays creation of the table, labels, buttons, and icon-theme data until the dialog is first shown. It also avoids scanning every application cgroup while pressure is low and the window is hidden.

On the development machine, measured three seconds after startup:

| Metric | Before | After |
| --- | ---: | ---: |
| RSS | 60.7 MiB | 52.7 MiB |
| PSS | 19.4 MiB | 13.0 MiB |
| Private dirty | 9.85 MiB | 7.93 MiB |

Most remaining RSS is shared DTK/Qt code. An otherwise empty `DApplication` measured about 41.6 MiB RSS on the same machine, so RSS alone overstates private memory cost.

## Requirements

- Linux cgroup v2 with `memory`, `memory.swap`, `memory.pressure`, `cgroup.freeze`, and `cgroup.kill`
- `gxde-k9` for session startup and keep-alive
- Deepin sessions that place applications in `app-DDE-*` cgroups
- Qt 6 and DTK 6 Widget development packages
- Qt 6 Linguist tools (`qt6-tools-dev`, `qt6-l10n-tools`) to compile translations
- CMake 3.16 or newer and a C++17 compiler

## Build and Test

```bash
cmake -S . -B build
cmake --build build
ctest --test-dir build --output-on-failure
```

The self-test covers PSI parsing, pressure and swap threshold boundaries, `pgscan` sampling failures, memory formatting, and freezer ownership/thaw behavior.

Run visibly for UI inspection:

```bash
./build/deepin-liferaft
```

Run as a hidden monitor:

```bash
./build/deepin-liferaft --hidden
```

## Debian Package

```bash
dpkg-buildpackage -us -uc -b
sudo apt install ../deepin-liferaft_0.1.0_amd64.deb
```

The package installs:

- `/usr/bin/deepin-liferaft`
- `/usr/share/gxde-k9/edging/deepin-liferaft.edging`
- `/usr/share/icons/hicolor/scalable/apps/deepin-liferaft.svg`
- `/usr/share/doc/deepin-liferaft/README.md.gz`
- `/usr/share/man/man1/deepin-liferaft.1.gz`
- `/usr/share/deepin-liferaft/translations/` with `zh_CN`, `en_US`, `ja_JP`, and `ko_KR` message catalogs

## K9 Session Startup

`gxde-k9` is started by the desktop session through `/etc/xdg/autostart/gxde-k9.desktop`. Its daemon scans `/usr/share/gxde-k9/edging/` every five seconds and keeps the configured `deepin-liferaft` process running. The task inherits the real graphical-session environment, so the dialog is opened in the correct user session on both X11 and Wayland.

The installed task is active by default for every user. Check it:

```bash
gxde-k9-chocker list --category edging
gxde-k9-chocker status edging deepin-liferaft --system
```

Stop the current monitor (K9 restarts it within five seconds):

```bash
gxde-k9-chocker stop edging deepin-liferaft --system
```

Disable it permanently for all users:

```bash
sudo gxde-k9-chocker disable edging deepin-liferaft --system --yes
```

Re-enable it:

```bash
sudo gxde-k9-chocker enable edging deepin-liferaft --system --yes
```

When upgrading from a package version that used the systemd user unit, the old unit's global enable links are removed. If an old instance is still running in the current session, the K9 task waits for it to stop instead of starting a second monitor; the next login starts through K9.

## Platform Differences

Linux PSI and systemd cgroups replace macOS VM-pressure and application lifecycle APIs. The dialog is therefore behaviorally similar rather than binary-identical: DDE cgroups define application boundaries, Fedora's systemd-oomd thresholds decide when to show the dialog, and DTK supplies native Deepin window styling.

## License

Deepin Liferaft is licensed under [GPL-3.0-or-later](LICENSE).
