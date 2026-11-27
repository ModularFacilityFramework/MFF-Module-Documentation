# Fleet module: overview, access, configuration and machine setup

Module `fleet` (repository `MFF-Module-FLEET`): monitoring of the facility's machines, with two actions on each (upgrade and reboot). This describes commit `7781f89`.

**Status:** the module is not yet deployed. On 27/11/2026 the machine scripts (`agent/`) from commit `1fad83e` were installed, checked, uninstalled and reinstalled on one machine (`nusbf-raw4`). Two faults in the BeeGFS mount check were found there and fixed in `7781f89`. That fix has **not** yet been re-run on a machine; it has only been tested offline with fake mount tables.

## Pages
- **Summary:** a count per traffic light, and cards grouped into Servers, GPU nodes, Storage and Workstations, with problem machines first. The same data is also a sortable table.
- **One tab per machine:**
  - a title bar with two buttons, labelled exactly `sudo apt-get update && sudo apt-get -y dist-upgrade && sudo apt-get -y autoremove` and `sudo reboot now`;
  - an identity card, collected over SSH daily and with "Collect now";
  - live health, read only while the tab has focus;
  - the speed of the connected port;
  - usage charts, read from the Munin master's RRD files;
  - the action history.
- **Action log.**
- **Settings:** the machine list (add, edit, remove), "Mark checked", SSH verification, and the traffic-light limits.
- **Page Access.**

Page state and table sorting use core's helpers. The first start seeds 31 machines (`backend/data/seedMachines.json`).

## Traffic light
There are five states, each with its own colour, shape, symbol and word. In rank order: **OK**, **UNKNOWN**, **WARNING**, **CRITICAL**, **DOWN**. UNKNOWN ranks above OK, so a machine without the Fleet Munin plugins shows grey.

A machine's light is that of its worst check, and the reason text is that check's. A machine you have not yet checked stays red until you press "Mark checked".

Default limits (changed on Settings):

| Check | Warning | Critical |
|---|---|---|
| Load per CPU thread | 1 | 2 |
| Memory | 90 % | 95 % |
| Disk space and inodes | 85 % | 95 % |
| CPU and GPU temperature | 80 °C | 90 °C |

- **Also critical:** failed systemd services, degraded RAID, a failed SMART test, a missing BeeGFS mount.
- **Also warning:** pending updates, a reboot needed, RAID rebuilding, reallocated sectors, an expected NVIDIA GPU not reported.

Stale Munin history shows "No history since <date>".

## Access
There are two layers:
1. **Core's module gate:** it answers `404` to users other than developers and administrators until the module is granted at Setup → Preferences → Modules → Access.
2. **Fleet's own Page Access tab** (`summary`, `machines`, `log`, `settings`), enforced by the server with `403`.

Developers and administrators always have every page. There are no other special roles, and nusbf has no special case.

**What a grant allows:**
- A role granted "Machine tabs" can press upgrade and reboot.
- A role granted "Settings" can change machine addresses and accept SSH host keys.
- A user without "Machine tabs" sees summary cards that do not open.

## Configuration
All keys are optional and go in core's `.env` as `MODULE_FLEET_<KEY>`.

| Key (after `MODULE_FLEET_`) | Default | Meaning |
|---|---|---|
| `DB_CONNECTION_LIMIT` | 5 | Database connection limit |
| `READ_KEY_PATH` | | Private key of the read account; needed to verify a machine and to collect its identity |
| `ACTION_KEY_PATH` | | Private key of the action account; the buttons stay off without it |
| `SSH_READ_USER` | `fleetread` | Read account |
| `SSH_ACTION_USER` | `fleetact` | Action account |
| `SSH_PORT` | 22 | SSH port |
| `BIN_SSH` | `ssh` | `ssh` program |
| `BIN_SSH_KEYSCAN` | `ssh-keyscan` | `ssh-keyscan` program |
| `MUNIN_PORT` | 4949 | munin-node port |
| `MUNIN_DB_DIR` | `/var/lib/munin` | Munin master's data, read only |
| `BIN_RRDTOOL` | `rrdtool` | `rrdtool` program |
| `SEED_FILE` | | Another machine list for the very first start, for test systems |

The backend's OS user must be able to read the key files and `MUNIN_DB_DIR`, and to run `rrdtool`.

## Security
- **Keys:** the private keys stay as files on the MFF host. Their paths and contents are never sent to the browser.
- **SSH:** run with an argument list (no shell), `-F /dev/null`, only the pinned host key, strict checking, and no agent or forwarding. The only thing sent is one fixed word: `identity`, `upgrade` or `reboot`.
- **Before the buttons work:** a person must read the host key, compare it and accept it, and then the hostname check must pass. Changing the address removes both.
- **Confirmation:** upgrade needs a confirmation. Reboot needs the exact machine name to be typed.
- **One at a time:** one action per machine at a time.
- **Logging:** every run and every refused attempt is logged. The row is written before the command starts, and the module never deletes log rows. Command output is redacted, and the newest 1,000,000 characters are kept.
- **Backend restart during a run:** it records "Result unknown". The upgrade itself carries on in its own systemd unit (log: `/var/log/fleet/upgrade.log` on the machine).
- **Upgrade:** it is non-interactive and keeps existing configuration files.

## Machine setup (`agent/`)
Debian and Ubuntu only. An administrator runs it as root; nothing in MFF runs it. Linux only (no Windows yet). See "Status" at the top for what has been run on a machine.

Install:

```bash
sh install.sh --read-key "<pub>" --action-key "<pub>" --from "<address MFF connects from>"
```

- `--plugins-only` installs only the Munin plugins and the root-facts timer.
- `smartmontools`, `dmidecode`, `pciutils`, `sudo`, `iproute2` and `munin-node` must be installed first. The script stops and names any that are missing.

Remove:

```bash
sh uninstall.sh
```

It refuses while an upgrade is running. It keeps the packages, the munin-node configuration and `/var/log/fleet`.

**What the install adds:**
- The accounts `fleetread` (no sudo) and `fleetact`. There are no passwords. The `authorized_keys` files are root-owned, with `restrict`, `from=` and the forced command `/usr/local/sbin/fleet-dispatch`.
- Sudo rules that are literally only `/usr/local/sbin/fleet-upgrade ""` and `/sbin/reboot now`.
- A `fleet-rootfacts` systemd service and timer.
- The Munin plugins `fleet_updates`, `fleet_reboot`, `fleet_services`, `fleet_raid`, `fleet_temps` and `fleet_netlink`. `fleet_gpu` is added only where `nvidia-smi` exists, and `fleet_mounts` only with BeeGFS. The BeeGFS mount check matches every mount type starting `beegfs` (a local build shows, for example, `beegfs7-7`).
- Debian 13 starts munin-node with `ProtectHome=yes`, which hides `/home`, `/root` and `/run/user` from plugins. Only where a BeeGFS mount is in one of those, `install.sh` adds `/etc/systemd/system/munin-node.service.d/fleet.conf` with `ProtectHome=read-only`, so munin-node plugins on that machine can see those folders read-only. `uninstall.sh` removes it.

**By hand:**
- If `sshd` uses `AllowUsers` or `AllowGroups`, add both accounts.
- munin-node must allow the MFF server. If it does not, add a `cidr_allow` line in `/etc/munin/munin-node.conf`.

Not tested with `sudo-rs`.
