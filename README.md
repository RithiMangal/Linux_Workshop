# Linux Workshop

**Repair Anything Here With Joy**

![Linux](https://img.shields.io/badge/platform-Linux-blue?style=for-the-badge&logo=linux)
![Rust](https://img.shields.io/badge/built_with-Rust-orange?style=for-the-badge&logo=rust)
![License](https://img.shields.io/badge/license-GPL--3.0-green?style=for-the-badge)
![AppImage](https://img.shields.io/badge/distro-AppImage-red?style=for-the-badge)

• Strategic Recommendation: Frames Lubuntu as the primary native development environment for guaranteed stability.
• Technical Breakdown: Bullet points highlighting the streamlined configuration, low resource footprint, and its ability to revive older hardware with zero lag.
• Veteran Insight: Concludes with an engaging nod to the fun of managing multiple Linux environments.

Linux Workshop is a Rust/egui desktop application for inspecting Linux system health and running a predefined sequence of repair commands. It combines disk, package, service, boot, network, security, and update checks in a custom dark-neon interface and is distributed as an x86-64 AppImage.

> [!CAUTION]
> **Experimental system-modification tool.** `DEEP SCAN & FIX` is **not** a read-only scan. It runs privileged commands during the audit phase and then runs every repair stage, without per-action confirmation, dry-run mode, or a cancellation control. It can remove packages and files, truncate logs, kill processes, change permissions, modify EFI/GRUB configuration, and install a boot-time service. Back up important data and boot configuration, review the [operational risks](#operational-risks-and-known-limitations), and test on a disposable VM before using it on a machine you rely on.

## At a glance

| Area | Current implementation |
| --- | --- |
| Desktop | Linux GUI built with Rust 2021, eframe/egui 0.31 |
| Distribution | Prebuilt x86-64 AppImage; source build with Cargo |
| Workflow | One `DEEP SCAN & FIX` action: 20 audit stages followed by 10 repair stages |
| Privileges | Root, or an administrator password validated with `sudo -S -v` |
| Interface | Frameless neon window, live activity log, per-module reports, draggable scrollbars |
| Platform focus | Linux with systemd and Debian/Ubuntu-style tools; other distributions are only partially handled |
| Release | `1.0.0` in `Cargo.toml`; see [RELEASE_NOTES.md](RELEASE_NOTES.md) |

The tagline is a UI label, **not** a promise that every reported problem can be diagnosed or repaired.

## Requirements and compatibility

- **OS/architecture:** Linux. The provided AppImage and binary target **x86-64**; other architectures are not packaged here.
- **Desktop:** A graphical session and the native graphics/window-system libraries required by eframe. Native window dragging has been exercised on X11; other desktop environments need their own validation.
- **Privileges:** Run as root, or have `sudo` installed and an account authorized to use it. There is no limited-scan mode. If `sudo` is unavailable, the app will not start the workflow.
- **System tools:** Individual stages invoke available commands such as `lsblk`, `findmnt`, `fsck.ext4`, `smartctl`, `dpkg`, `apt-get`, `systemctl`, `journalctl`, `nmcli`, `efibootmgr`, `grub-install`, `update-grub`, and `chkrootkit`. Missing tools may cause stages to skip work, report incomplete results, or attempt a package install.
- **Distribution scope:** Some package checks recognize RPM, pacman, dnf, or zypper, but EFI repair and several other paths assume **Ubuntu/Debian on x86-64**. Do **not** infer safe support for other distributions, dual-boot systems, or custom boot layouts.
- **AppImage runtime:** The AppImage packages the application and desktop metadata, but it does **not** bundle every host library or system utility. Depending on the host, AppImage execution may require working FUSE support.

## Run the application

For **UI evaluation only**, launch the artifact from the project directory:

```bash
chmod +x appimage/Linux_Workshop-x86_64.AppImage
./appimage/Linux_Workshop-x86_64.AppImage
```

**There is no safe UI-only mode:** on any machine that is not a disposable VM, do **not** click **DEEP SCAN & FIX** or enter an administrator password. The button starts the privileged, system-modifying workflow. If you downloaded only the AppImage, use its own path instead. Do not store an executable AppImage in `/tmp`, `/var/tmp`, or `/dev/shm`: remediation can delete it. Do not launch the GUI with `sudo` merely to start it; the application asks for authorization when the workflow begins.

> [!IMPORTANT]
> A normal AppImage run executes from `/tmp/.mount_*`. During a **full** run, the temp-path process-kill rule targets Linux Workshop itself. The process can terminate before the final stage or boot follow-up. For controlled testing of the entire sequence, build and launch `./target/release/linux-workshop` **outside `/tmp`, `/var/tmp`, and `/dev/shm`** on a disposable VM; this avoids that specific self-kill, not the other risks below.

1. Start Linux Workshop and inspect the distribution, kernel, boot mode, and privilege indicator. The strip reports the process's effective user, so it may still say `user (limited)` after a successful sudo unlock.
2. Click **DEEP SCAN & FIX**. If not already running as root, enter your administrator password and select **UNLOCK & FULL REPAIR**. **Esc** cancels the password prompt; it does not start a scan.
3. The workflow runs audit stages and then repair stages. **SCANNING** and **REPAIRING** indicate phases. There is no per-stage approval or in-app cancellation; closing the window during a command is not a safe cancellation strategy and can leave child commands running or interrupted.
4. Watch **SCAN MODULES** and **ACTIVITY LOG**. Click an audit row to open its **MODULE REPORT**; select **‹ LOG** to return to the activity stream. Repair stages appear only as summary lines in the log; their detailed notes are not exposed in module reports. Use the mouse wheel or drag the scrollbar to inspect older entries after the run finishes.
5. Independently verify findings with suitable system tools. **ALL CLEAR** means the workflow reached its end state; it is **not** an independent health certification. The final confirmation stage reports success without a comprehensive recheck.

The **CONTACT** button opens the maintainer's GitHub repository listing in a browser. Avoid launching desktop applications as root merely to follow links. The activity log is an in-memory view capped at 300 entries. A boot-time check, when armed, writes a separate log to `/var/log/linux-workshop-bootscan.log`.

### Interface and report semantics

- The frameless title bar draws its own **close**, **minimize**, and **maximize** controls. Drag the bar to move the window using the desktop window manager; double-click the bar to toggle maximize.
- The main action changes colour while a workflow is running. The status strip shows `SYSTEM READY`, `SCANNING`, `REPAIRING`, or `ALL CLEAR` alongside aggregate counters, boot-check state, and elapsed time.
- The lower area gives roughly **46%** of its width to scan modules and the remainder to the activity log or selected module report. Both panels support wheel scrolling and draggable scrollbar thumbs where content exceeds the visible area.
- Row tags such as `CLEAN`, `FOUND`, and `FIXED` reflect the app's reported counts. A mapped repair may mark a row resolved without updating the row's own `fixed` value; neither a tag nor a counter is a substitute for independent verification.
- The progress indicator tracks UI reports and jumps to complete when the worker finishes; it is not a calibrated percentage of work or remaining time. There is no report export in this release.

## What the workflow inspects

The application currently schedules these **20 audit stages**, in order. Stage names are shown as they appear in the UI; availability and accuracy depend on the host's tools, permissions, devices, and distribution.

| # | Audit stage | Scope |
| ---: | --- | --- |
| 1 | Disk & Filesystem Audit | Block-device and mount overview, including high filesystem usage. |
| 2 | File System Check (scandisk) | Invokes `fsck.ext4 -n` through the GUI process's normal shell on mounted ext2/3/4, XFS, Btrfs, and F2FS entries (unprivileged in sudo mode, privileged only when the GUI itself runs as root); for a detected read-only mount it can create `/forcefsck` and attempt a **read-write remount**. This is not an offline filesystem check. |
| 3 | S.M.A.R.T. Disk Health | Reads `smartctl` attributes when installed. Its current thresholds can count healthy zero-valued attributes as failures. |
| 4 | Package Database Integrity | Checks package-manager database state where commands are available. |
| 5 | Broken Dependency Repair | **Modifying step inside the audit phase:** can run `apt-get -f install`, `dpkg --configure -a`, or distribution-specific alternatives. |
| 6 | Failed Systemd Services | Lists failed systemd units. |
| 7 | Bootloader & EFI Health | Reads boot mode and bootloader-related state. |
| 8 | EFI Configuration Audit | Inspects UEFI entries and EFI system-partition state where accessible. |
| 9 | Kernel & Module Audit | Looks for selected kernel/module error patterns. |
| 10 | Permission & Ownership Audit | Identifies selected world-writable files and reports privileged-file information. |
| 11 | Memory & Swap Health | Checks memory and swap availability. |
| 12 | Disk Usage & Inode Check | Reports capacity and inode-pressure indicators. |
| 13 | Journal Error Analysis | Reads recent journal errors. |
| 14 | Network Stack Health | Inspects interfaces, resolver state, and listening services. |
| 15 | Network Manager Audit | Checks NetworkManager and connection state. |
| 16 | Cron & Timer Integrity | Reviews scheduled jobs and timers. |
| 17 | Security & Firewall Audit | Checks firewall state and selected exposed listeners. |
| 18 | Malware & Rootkit Heuristics | Looks for a few suspicious library/process patterns and `chkrootkit` output if installed; **not** a signature-based malware scanner. |
| 19 | Update & Upgrade Readiness | Reports pending upgrades and related package state. |
| 20 | Final Full Verification | Checks selected systemd and failed-unit status; when launched as root, also flushes and attempts to drop caches. |

## What the repair phase changes

After the audits, the current implementation runs **all 10 repair stages unconditionally**. A clean audit result does not skip its corresponding repair. Some stages execute commands that may not change anything; a displayed `repaired` count does not prove that the original condition was fixed.

| # | Repair stage | Actions and important side effects |
| ---: | --- | --- |
| 1 | Filesystem Cache & Journal Flush | Calls `sync`, attempts to drop kernel caches, and flushes the journal. This does **not** repair on-disk filesystem corruption. |
| 2 | APT Cache & Orphan Cleanup | Runs `apt-get clean` and noninteractive `apt-get autoremove -y` if apt is available; packages may be removed. |
| 3 | Failed Service Restart | Reloads systemd state, clears failure flags for all units, and tries to restart up to eight previously failed units. |
| 4 | Stale Permission Restoration | Forces `0644` on selected `/etc` files, `0600` on `/etc/shadow` and `/etc/gshadow`, and `0755` on executable directories; also removes world-write permission from selected system files. Changing shadow-file modes may disrupt authentication helpers. |
| 5 | Log & Journal Rotation Cleanup | Vacuums the journal to size/time limits and **truncates** existing `/var/log/syslog`, `/var/log/messages`, and `/var/log/auth.log`, destroying their retained audit trail. |
| 6 | Temporary File Purge | Deletes selected entries older than seven days under `/tmp`, `/var/tmp`, and `/dev/shm`. |
| 7 | Network Manager Repair | Can start/enable services, attempt connection activation, and restart NetworkManager and systemd-resolved; connectivity can be interrupted even if the audit found no network fault. |
| 8 | EFI Configuration Repair | May install `efibootmgr` through apt, delete Ubuntu EFI entries beyond the first five, run `grub-install --bootloader-id=ubuntu` when `/boot/efi` exists, and run `update-grub` on UEFI systems. Risky on nonstandard, encrypted, Secure Boot, or dual-boot setups. |
| 9 | Malware & Rootkit Remediation | Force-kills processes launched from temporary paths and deletes **all user-executable files there, regardless of age or owner**. It may clear `/etc/ld.so.preload` (the attempted backup is not required to succeed), rename matching cron/profile files—including `/etc/crontab`—and install `chkrootkit`. These broad rules do **not** confirm malware. |
| 10 | Final System Health Confirmation | Always emits `System verified healthy` and increments the repaired count; it performs **no** independent end-to-end verification. |

When the application's aggregate `found - repaired` count remains positive, it may install and enable `/etc/systemd/system/linux-workshop-bootscan.service` and `/usr/local/sbin/linux-workshop-bootscan.sh`. On a subsequent boot the one-shot script runs **automatic-repair** commands (`fsck -A -R -a -T` for fstab entries and `fsck.ext4 -p` for **every** unmounted ext2/3/4 block device it discovers), writes to `/var/log/linux-workshop-bootscan.log`, and disables the service. The `-R` option skips the root filesystem; other operating systems' partitions or removable disks may be included. The unit has a 90-second timeout: if it interrupts an in-progress check before the final disable command, the service can remain enabled and run again at the next boot. On hosts without `systemctl`, the app may create `/forcefsck` instead. The trigger is based on aggregate counters, **not specifically on confirmed filesystem corruption**. If the counters reach zero, cleanup can also remove an existing `/forcefsck` marker and disable this service—even if the marker was created outside Linux Workshop.

## Operational risks and known limitations

> [!WARNING]
> Treat this build as an **experimental prototype**, not a production recovery utility. Make a restorable backup and ensure you have console/recovery access before touching package state or boot configuration.

- **No safe preview or rollback:** There is no dry-run, stage selection, per-change approval, or automatic undo. A password authorizes the whole sequence; closing the GUI is not a reliable way to stop child processes.
- **AppImage self-termination:** A normally mounted AppImage runs its program from `/tmp/.mount_*`. The temp-path `kill -9` rule targets Linux Workshop itself and other AppImages, so a full AppImage run can stop before repair stage 10 and before completion/boot follow-up. Executables stored directly in `/tmp`, `/var/tmp`, or `/dev/shm` can also be deleted by the next rule. Use a source-built binary outside all three temporary directories for controlled VM testing.
- **Broad deletion and configuration changes:** The rootkit remediation deletes executable files under temporary directories without checking age or ownership; it can empty `/etc/ld.so.preload` even if its attempted backup fails, and pattern-based renaming may disable `/etc/crontab` or legitimate scripts. These are **not** malware-confirmed targets.
- **Heuristics are not a security verdict:** The SMART check counts raw values of `0` through `10` in several attributes as problems; healthy drives with zero reallocated sectors can therefore be flagged. Malware checks are limited heuristics. Verify findings independently.
- **Filesystem checks are limited:** `fsck.ext4 -n` is invoked through a normal shell on mounted entries, including non-ext4 types. When authorized through sudo rather than running the GUI as root, that shell remains unprivileged; device access can fail and be counted as a finding. The kernel storage-error check is skipped in this sudo mode, while some other kernel/journal checks may lack read access. A detected read-only mount may be **remounted read-write** and `/forcefsck` created; this can worsen an already-damaged filesystem. Use an offline, filesystem-specific check instead.
- **Repair results are not verified end-to-end:** Pipelines can mask command failures; some network commands contain `tail-3` typos; and the last repair always reports `System verified healthy` plus one repaired action without checking health. Audit and repair counts use different units, so findings can remain even if the displayed difference reaches zero. `FIXED`, `repaired`, and `ALL CLEAR` are reporting labels, not guarantees.
- **Privilege lifetime:** Sudo is validated once and used noninteractively afterward. If authorization expires, later commands can fail; the UI does not re-prompt automatically.
- **Data, availability, and forensic impact:** Existing syslog/auth logs are truncated, apt may remove packages, temporary files and processes are removed, and network services are restarted. Forced `0600` modes on `/etc/shadow` and `/etc/gshadow` may disrupt helpers that rely on group access. The Arch dependency path can run noninteractive `pacman -Syu` during auditing.
- **Local security and launcher caveats:** Boot-service files are staged through a predictable `/tmp/lw-stage-<pid>` path before privileged copying, creating a potential local race/symlink risk on multi-user hosts. `AppRun` extends `LD_LIBRARY_PATH`; when the inherited value is empty, the trailing empty entry can search the current working directory for libraries. Do not run this AppImage as root from an untrusted directory.
- **Partial distribution support:** Some paths assume apt, systemd, Ubuntu EFI identifiers, and x86-64 GRUB. Review the implementation before use outside a disposable Debian/Ubuntu test environment.

A missing or failed tool should not be interpreted as a clean result. If the system has an actual disk, boot, or security incident, use appropriate offline diagnostics, vendor tools, and a recovery environment.

## Build, test, and package from source

Install a stable Rust toolchain capable of compiling the pinned dependencies and the native build dependencies required by eframe on your distribution. This project does not currently declare an MSRV (minimum supported Rust version). From the project directory:

```bash
cargo check --locked
cargo test --locked
cargo build --release --locked
./target/release/linux-workshop
```

The release profile enables optimization, LTO, a single codegen unit, and symbol stripping, so a clean release build can take substantially longer than `cargo check`.

To recreate the AppImage, install `appimagetool` separately, then run:

```bash
cp target/release/linux-workshop appimage/AppDir/usr/bin/linux-workshop
appimagetool "$(pwd)/appimage/AppDir" "$(pwd)/appimage/Linux_Workshop-x86_64.AppImage"
```

`appimage/AppDir/` contains the executable, desktop entry, launcher, and 512×512 icon. The top-level `appimage/AppRun` and `appimage/linux-workshop.desktop` are copies outside `AppDir`; keep them in sync with the packaged copies if you change launcher or desktop metadata. Packaging does not install every command the repair workflow invokes.

## Code layout

```text
Cargo.toml                         Package metadata and release profile
Cargo.lock                         Locked Rust dependencies
src/main.rs                        GUI, audit/repair workflow, sudo helpers, tests
appimage/AppDir/                   AppImage filesystem and desktop assets
appimage/Linux_Workshop-x86_64.AppImage   Packaged x86-64 build
RELEASE_NOTES.md                   Release history and current caveats
```

A background worker sends stage and report messages to the egui UI through an in-process channel. The two current unit tests cover bottom-anchored scrolling and simulated scrollbar dragging. They do **not** exercise privileged repairs, real disks, package transactions, or reboot behavior.

## Licensing

No `LICENSE` file is present in this project directory. A license and distribution policy should be added before publishing the code as an open-source project.
