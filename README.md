# ubuntu2mint

High-risk, best-effort **in-place conversion** script that keeps **Ubuntu as the base OS** while adding **Linux Mint repositories + Mint desktop/tooling** to approximate a Linux Mint system **without a full reinstall**.

This project is intended for experienced Linux admins who understand APT, repo pinning, display managers, and rollback strategies.

## Two scripts, in this order

| | |
|---|---|
| **`ubuntu-to-mint-convert-v3.sh`** | Adds the Mint repositories and installs the Mint desktop stack alongside what is already there. Nothing is removed. |
| **`ubuntu-desktop-prune.sh`** | *Afterwards*, once you have booted into a working Mint session, removes the Ubuntu GNOME desktop components left behind. |

The order matters. Pruning the Ubuntu desktop before you have confirmed a
working replacement session leaves you with neither — see
[Pruning the Ubuntu desktop](#pruning-the-ubuntu-desktop).

> 🟥 **UNSUPPORTED + PROBABLY DUMB (READ THIS FIRST)**  
> There is **no guaranteed safe** way to convert Ubuntu to Linux Mint in-place and preserve *all* corporate software/agents.  
> **EDR/MDM/VPN/compliance tooling may break** and require re-enrollment.  
> A **clean install** (or a second disk/VM test) is the recommended approach.  
>  
> The script enforces an interactive “I understand” gate during `convert` and also requires `--i-accept-the-risk`.



## What this does

- Detects supported Ubuntu bases:
  - **Ubuntu 24.04 (noble)** → targets **Linux Mint 22.x** (default **22.3 “zena”**)
  - **Ubuntu 22.04 (jammy)** → targets **Linux Mint 21.x** (default **21.3 “virginia”**)
- Adds the Linux Mint package repository (`packages.linuxmint.com`)
- Keeps Ubuntu repos for the underlying base system
- Installs Mint meta-packages (desktop + tooling) and configures:
  - **LightDM** as default display manager
  - **slick-greeter** as greeter
  - Default desktop session based on `--edition`
- Applies conservative APT pinning:
  - Ubuntu remains default for overlapping base packages
  - Mint desktop stack is pinned high to avoid mixed-version dependency breakage
- Includes guardrails:
  - APT/dpkg lock detection
  - best-effort dpkg/apt repair (always on; there is no flag to disable it)
  - a pre-install APT simulation that aborts on critical-package removal or too many removals
  - disables third-party sources by default (with allowlist heuristics for common corp repos)
  - backup + rollback support
  - post-conversion validation report written into the backup directory

## What this does *not* do

- It does **not** replace your OS identity with Mint in a fully supported way.
- It does **not** guarantee your machine remains compliant in managed enterprise environments.
- It does **not** guarantee perfect package resolution—APT may still want to remove packages.
- Rollback restores `/etc/apt`, but may not remove packages installed during conversion (use snapshots/Timeshift for full reversion).
- This script is **LightDM-first**, which implies **X11 sessions**; it does not attempt to force Wayland.



## Requirements

- Ubuntu **24.04 (noble)** or **22.04 (jammy)**
- Root access (`sudo`)
- Working APT and dpkg state (the script can attempt basic remediation)
- Internet access to:
  - Ubuntu mirrors (or your corporate mirror)
  - `packages.linuxmint.com`
- Recommended:
  - **Timeshift** snapshot configured
  - Full-disk backup or VM snapshot



## Quick start

### 1) Clone and run doctor checks

```bash
git clone https://github.com/LINUXexpert-org/ubuntu2mint.git
cd ubuntu2mint
sudo bash ubuntu-to-mint-convert-v3.sh doctor
````

### 2) (Optional) Run plan mode (dry-run)

Plan mode simulates the APT changes and writes a log under `/var/log/ubuntu-to-mint/`.

```bash
sudo bash ubuntu-to-mint-convert-v3.sh plan --edition cinnamon
```

Plan mode uses a **temporary APT environment** (it does not modify your system’s APT sources).
It creates a **temporary Mint keyring** by downloading and extracting the `linuxmint-keyring` package.

> Plan mode is for decision support. It does not modify `/etc/apt` or install Mint repos onto your live system.

Plan mode is **advisory** — it writes the simulation to a log for you to
read and does not decide anything. The check that can actually stop a
conversion is the gate inside `convert` (see [Safety model](#safety-model-important)),
which runs its own simulation against your real APT configuration
immediately before installing.

### 3) Run conversion

```bash
sudo bash ubuntu-to-mint-convert-v3.sh convert --i-accept-the-risk --edition cinnamon
```

You can skip the secondary interactive confirmation prompts with `--yes`:

```bash
sudo bash ubuntu-to-mint-convert-v3.sh convert --i-accept-the-risk --edition cinnamon --yes
```

> The big red unsupported disclaimer gate is still required during `convert` even with `--yes`.

### 4) Reboot and validate

After conversion:

1. Reboot
2. Login via LightDM and confirm your chosen session loads
3. Validate:

   * VPN connectivity (GlobalProtect, etc.)
   * EDR/MDM agents + compliance posture (CrowdStrike Falcon, etc.)
   * corporate certificates / SSO
   * NetworkManager
   * printers
   * smartcards / YubiKey
   * camera/audio



## Usage

```text
sudo bash ubuntu-to-mint-convert-v3.sh doctor
sudo bash ubuntu-to-mint-convert-v3.sh plan [options]
sudo bash ubuntu-to-mint-convert-v3.sh convert --i-accept-the-risk [options]
sudo bash ubuntu-to-mint-convert-v3.sh rollback /root/ubuntu-to-mint-backup-YYYYMMDD-HHMMSS
```



## Options

### Desktop / targets

* `--edition cinnamon|mate|xfce`
  Desktop edition meta-package to install (default: `cinnamon`).

* `--target <mint_codename>`
  Override the Mint target codename. Allowed targets depend on your Ubuntu base:

  * Ubuntu `noble`: `zena`, `zara`, `xia`, `wilma`
  * Ubuntu `jammy`: `virginia`, `victoria`, `vera`, `vanessa`

* `--mint-mirror <url>`
  Override Mint mirror base URL (default: `http://packages.linuxmint.com`)

### Safety / APT behavior

* `--keep-ppas`
  Do not disable third-party APT sources (not recommended).
  **Default behavior** disables PPAs and most third-party repos during conversion and moves them into the backup folder.
  Some common enterprise repos may be automatically allowlisted.

* `--with-recommends`
  Allow installation of recommended packages (default: off for safety).

* `--max-removals N`
  Abort if the pre-install simulation would remove more than N packages
  (default: `40`). Critical packages abort regardless of this number.

* `--preserve-snap`
  Keep `snapd` and reinstall it after the Mint stack (this is the default).

* `--yes`
  Skip most interactive prompts, and allow overwriting an existing Mint
  keyring that does not match the expected key.
  (Does **not** bypass the `convert` disclaimer gate or `--i-accept-the-risk`.)

> There is no flag to disable the dpkg/apt repair pre-flight — it always
> runs. Nor is there flavor-package purging; if a conflicting Ubuntu flavor
> meta-package causes a login loop, remove it by hand.



## Safety model (important)

Guardrails included to reduce “brick your system” outcomes:

* Refuses to run unless on supported Ubuntu bases
* Detects and blocks active APT/dpkg locks
* Attempts to repair basic dpkg/apt broken states before doing anything else
* Disables PPAs by default during conversion (unless `--keep-ppas`)
* Runs an APT simulation **during `convert`, immediately before the real install**, and aborts if:

  * APT wants to remove a critical package — `sudo`, `systemd`, `libc6`, `apt`, `dpkg`, `network-manager`, `grub-*`, the kernel meta packages, `openssh-server`. There is no threshold at which removing these is acceptable, so any hit aborts outright.
  * more than `--max-removals` packages would be removed (default `40`)
  * APT cannot resolve the install at all

  Nothing is installed when the gate trips. The full simulation is written to `/var/log/ubuntu-to-mint/simulate-YYYYMMDD-HHMMSS.txt`.
* Creates a backup directory for rollback
* Post-conversion validation writes a report into the backup directory (always created)



## Logs and backups

### Logs

* Main log:

  * `/var/log/ubuntu-to-mint/ubuntu-to-mint-YYYYMMDD-HHMMSS.log`
* Plan logs:

  * `/var/log/ubuntu-to-mint/plan-YYYYMMDD-HHMMSS.txt`

### Backup directory

During `convert`, a backup directory is created, e.g.:

* `/root/ubuntu-to-mint-backup-YYYYMMDD-HHMMSS/`

Contains:

* `/etc/apt` backup
* package inventories (`apt-manual`, holds, dpkg list)
* enabled systemd services list
* snap/flatpak lists (if installed)
* disabled third-party sources (if disabled)
* **post-convert validation report**:

  * `post-convert-validation.txt`



## Rollback

Rollback restores `/etc/apt` and any disabled sources from the backup:

```bash
sudo bash ubuntu-to-mint-convert-v3.sh rollback /root/ubuntu-to-mint-backup-YYYYMMDD-HHMMSS
sudo apt-get update
sudo apt-get -f install
```

> Rollback restores APT configuration but may not remove packages installed during conversion.
> For full restoration use Timeshift / snapshot / backup.



## Troubleshooting

### APT update fails after conversion

* Check:

  * `/etc/apt/sources.list.d/official-package-repositories.list`
  * Mint mirror reachability
  * corporate proxy settings
* If needed:

  * rollback using the backup directory

### Desktop won’t start / login loop

* Boot to a TTY (`Ctrl+Alt+F3`)
* Inspect:

  * `journalctl -b -p err`
  * user session logs: `~/.xsession-errors`
* Reinstall key desktop components:

  * `sudo apt-get install --reinstall lightdm slick-greeter cinnamon-session cinnamon-settings-daemon muffin`

### Key retrieval / keyserver issues

This version avoids keyservers by default. It prefers:

* a locally installed `linuxmint-keyring` package, or
* downloading the latest `linuxmint-keyring_*.deb` from the Mint mirror and extracting the keyring

The extracted key is checked against the expected Mint key ID
(`A6616109451BBBF2`) and the run aborts if it is not present, so a
substituted keyring fails rather than being trusted.

If your existing keyring does not match, the script stops and tells you to
either re-run with `--yes` to overwrite it, or delete
`/usr/share/keyrings/linuxmint-repo.gpg` by hand and re-run.

### Known dpkg overwrite conflict (mintupdate)

Some environments hit a file conflict between `mintupdate` and
`software-properties-gtk` over an icon file. The script applies a dpkg
diversion for that file before installing, and retries the install once
with remediation if it still fails.

### Login loop after conversion

A leftover Ubuntu flavor meta-package (`ubuntucinnamon-desktop` and
friends) can fight the Mint session and produce a login loop. The script
does **not** purge these — check for them from a TTY and remove them
manually if the desktop will not start.

### Corporate VPN/EDR breaks

* Reinstall using corporate-provided packages
* Re-enroll if required
* Validate kernel modules, certificate stores, and PAM stack



## Pruning the Ubuntu desktop

`ubuntu-desktop-prune.sh` is the second half of this, and it is destructive:
it purges the Ubuntu GNOME desktop rather than adding anything.

**Only run it after you have rebooted into a working Mint session.** It
removes `gdm3`, `gnome-shell` and the Ubuntu session packages; if the
replacement desktop does not actually work yet, you will be left without
either one. `plan` first, always.

### Commands

```bash
sudo bash ubuntu-desktop-prune.sh doctor     # environment checks
sudo bash ubuntu-desktop-prune.sh plan       # simulate, change nothing
sudo bash ubuntu-desktop-prune.sh prune --yes
sudo bash ubuntu-desktop-prune.sh rollback /root/ubuntu-desktop-prune-backup-YYYYMMDD-HHMMSS
```

`prune` refuses to run without `--yes`. There is no interactive prompt —
the flag *is* the confirmation.

### Options

* `--yes` — required for `prune`.
* `--max-removals N` — abort if the simulation removes more than N packages
  (default `75`). Critical packages abort regardless of this number.
* `--with-recommends` — allow recommended packages (default: off).
* `--skip-dm-fix` — do not try to make LightDM the default display manager
  before pruning. **Rarely what you want**: the script sets LightDM up first
  precisely because `gdm3` is about to be removed, and skipping that step is
  how you end up with no display manager at all.

### What it removes

A fixed, conservative list — no wildcards — filtered to whatever is actually
installed: `ubuntu-desktop`, `ubuntu-desktop-minimal`, `ubuntu-session`,
`ubuntu-session-minimal`, `gdm3`, `gnome-shell` and its common/dock packages,
the `yaru-theme-*` set, `ubuntu-wallpapers*`, `gnome-software`, `snap-store`.

Toolkit and library packages that Cinnamon, MATE or Xfce may still depend on
are deliberately left alone.

### Safety gates

The same shape as the converter's. Before purging, it simulates
`apt-get purge --autoremove` and aborts if:

* a critical package would be removed — `sudo`, `systemd`, `systemd-sysv`,
  `network-manager`, `openssh-server`, or a kernel image. Unconditional.
* more than `--max-removals` packages would go.
* APT cannot resolve the purge.

It also sets LightDM as the default display manager *before* removing
`gdm3`, so the machine still has something to boot into.

### Backup and rollback

`prune` writes `/root/ubuntu-desktop-prune-backup-YYYYMMDD-HHMMSS/`
containing `/etc/apt`, `/etc/lightdm`, `/etc/X11/default-display-manager`,
the `display-manager.service` symlink, and inventories of installed
packages, manual selections, holds and enabled services.

`rollback` restores the APT and display-manager configuration from that
directory. It does **not** reinstall the purged packages — the inventories
are there so you can do that yourself, and a snapshot remains the only real
undo.

## Security & compliance considerations

If this is a corporate-managed device:

* Get explicit approval before modifying base OS repositories
* Confirm your organization’s standard OS images and compliance requirements
* Expect that security tooling may detect drift and require remediation



## License

Copyright (C) 2026 Coffey Labs

This project is licensed under the **GNU General Public License v3.0**.
See the `LICENSE` file or the header in `ubuntu-to-mint-convert-v3.sh`.



## Disclaimer

This project is provided as-is. You assume all risk for system instability, data loss, or compliance impact. Always have a tested restore path before running.
