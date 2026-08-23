# AGENTS.md

## Scope

These instructions apply to the entire `deepin-liferaft` repository.

## Project

Deepin Liferaft is a single-binary DTK 6 application. A gxde-k9 edging task in the graphical session runs it with `--hidden`; sustained Fedora-style systemd-oomd pressure or swap conditions open a macOS-style force-quit dialog. DDE `app-DDE-*` cgroups are the application boundary.

## Layout

- `main.cpp`: monitoring policy, cgroup ownership, lifecycle cleanup, and DTK UI
- `CMakeLists.txt`: build, CTest, resources, and installation
- `resources.qrc`: embedded application resources
- `data/icons/`: application icon source
- `deepin-liferaft.desktop`: DDE application entry
- `deepin-liferaft.edging`: gxde-k9 keep-alive task that starts the hidden monitor
- `deepin-liferaft.1`: command manual page
- `debian/`: Debian packaging
- `README.md`: user and operator documentation

Do not edit or commit generated content under `obj-*/`, `debian/deepin-liferaft/`, debhelper stamp/helper files, package artifacts, or CodeGraph databases.

## Safety Invariants

Memory-pressure code is safety critical. Preserve all of these invariants:

- Parse cgroup `memory.pressure` `full avg10`, not `some`.
- Treat failed or incomplete PSI, `memory.stat`, cgroup value, and `/proc/meminfo` reads as invalid samples. Never convert failure into zero usage or a trigger.
- Never freeze the cgroup containing Deepin Liferaft.
- Read `cgroup.freeze` before freezing. Do not claim or thaw a cgroup already frozen by another component.
- Track every cgroup successfully frozen by this process.
- Resume all owned cgroups before accepting a window close, normal process exit, `SIGTERM`, or `SIGINT`.
- If thaw fails, retain ownership and retry. Never silently clear the recovery set.
- After `cgroup.kill`, thaw a surviving cgroup before releasing ownership.
- Keep frozen applications visible in the table even when they fall outside the normal top-ten memory list.

Do not test freeze/kill behavior on real user applications. Use a disposable user cgroup or temporary fake control files.

## Policy

Current workstation policy:

- poll interval: 1 second
- pressure: `full avg10 > 50%` for 20 seconds
- reclaim must have occurred in the previous 30 seconds
- system memory and swap trigger: both above 90%
- swap candidate: above 5% of total swap
- post-action delay: 15 seconds

Pressure candidates sort by latest `pgscan` delta, then `memory.current`. Swap candidates sort by `memory.swap.current`. The UI may freeze up to three candidates; it does not automatically kill them.

If policy values change, update `README.md`, tests, and Debian package description in the same change.

## Resource Constraints

Keep hidden mode cheap:

- Do not construct table widgets, labels, buttons, or load desktop icons before the dialog is shown.
- Do not scan all application cgroups while pressure is low and the dialog is hidden.
- Cache desktop metadata; it is static for the lifetime of the daemon.
- Do not add a dependency for parsing or logic available through Qt, libc, procfs, or cgroup v2.

Measure changes with `/proc/PID/smaps_rollup`, not RSS alone. Report PSS and private dirty memory using equal startup timing.

## Build and Test

Run after every functional change:

```bash
cmake -S . -B obj-x86_64-linux-gnu
cmake --build obj-x86_64-linux-gnu --clean-first -j"$(nproc)"
ctest --test-dir obj-x86_64-linux-gnu --output-on-failure
./obj-x86_64-linux-gnu/deepin-liferaft --self-test
```

UI smoke test in a graphical session:

```bash
env DISPLAY=:0 ./obj-x86_64-linux-gnu/deepin-liferaft
```

Verify the window title, application icon, list layout, selection, Resume state, Force Quit behavior, and close cleanup. Verify hidden mode creates no window.

Verify signal cleanup by starting `--hidden`, sending `SIGTERM`, and checking shell wait status is `0`.

## Packaging

Build the Debian package with:

```bash
dpkg-buildpackage -us -uc -b
```

Inspect the package for the binary, gxde-k9 edging task, scalable icon, README, man page, translations, and generated debhelper maintainer snippets. Run `bash -n` on the packaged edging task and confirm it appears in `gxde-k9-chocker list --category edging`.

## Git

Keep commits scoped. Do not include build directories, generated Debian state, `.deb` files, tarballs, logs, temporary screenshots, or local CodeGraph state. Before committing, inspect `git status`, staged diff, tests, package contents, and remote configuration.
