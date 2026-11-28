# Fleet module: overview, access, configuration and machine setup

Module `fleet` (repository `MFF-Module-FLEET`): monitoring of the facility's machines, with two actions on each Linux machine (upgrade and reboot). This describes commit `45007de` (v0.1.18).

**Status:** `prod-mff-core` reported fleet 0.1.17 loaded in production on 28/11/2026 (0.1.18 is not deployed yet). On 27/11/2026 the machine scripts (`agent/`) from commit `1fad83e` were installed, checked, uninstalled and reinstalled on one machine (`nusbf-raw4`). Two faults in the BeeGFS mount check were found there and fixed in `7781f89`. That fix has **not** yet been re-run on a machine; it has only been tested offline with fake mount tables.

## Pages
A machine may have a nickname (set on Settings). The cards, machine tabs, table, Settings and confirmations show the nickname first with the machine's name next to it; sorting and the Settings search use it too. The typed confirmation for a reboot is still the machine's name.

- **Summary:** a count per traffic light, and cards grouped into the sections Servers, GPU nodes, Storage, Workstations, Switches and Other, with problem machines first. The same data is also a sortable table. The title bar has **Refresh all**, which reads every machine again now; while it runs it shows how many are left. Each card of a machine that is read (all but Windows and shut-down ones) has its own **Refresh** button and the time of its reading ("Reading now…" while it is read). The table has a sortable "Read at" column; its rows have no Refresh button.
- **One tab per machine:** For a Linux machine (other types show health only; the buttons, identity card, port and usage charts are Linux only):
  - a title bar with two action buttons, labelled exactly `sudo apt-get update && sudo apt-get -y dist-upgrade && sudo apt-get -y autoremove` and `sudo reboot now`;
  - a Refresh button in the title bar that reads this machine again now, with the time the reading was taken;
  - an identity card, collected over SSH daily and with "Collect now", which needs only the read key on the server (also for a machine not yet verified);
  - live health, read only while the tab has focus;
  - the speed of the connected port;
  - usage charts, read from the Munin master's RRD files;
  - the action history.
- **Action log.**
- **Settings:** the machine list (add, edit, remove) with each entry's type and section, a "Shut down on purpose" tick box, a storage device's NFS exports, "Mark checked", SSH verification, disc roles, and the traffic-light limits. Each machine has an optional "Munin address": when it is set, health is read from munin-node there. SSH always uses the machine's address.
- **Page Access.**

A Refresh button skips the short time for which a reading is reused (55 s on the Summary, 8 s on a machine tab). It never starts a second probe of a machine that is already being read: it takes that result. Pressing Refresh all while it runs, by anyone, joins the same pass. It refreshes health only, not the identity card or the usage charts. Windows and shut-down machines have no Refresh button.

Page state and table sorting use core's helpers. A first start seeds 43 entries (`backend/data/seedMachines.json`): `NUSBF-GPU1` is retired; `NUSBF-GPU4` is retired too; `NUSBF-4C`, `NUSBF-20C`, `Tundra-9963484`, `WINSTORAGESERV`, `fms-sbf-shifter` (192.168.1.146, a DHCP lease) and `royale` (192.168.1.141, a DHCP lease; university address 10.64.34.80, which garcon uses for the metaljet share) are Windows; `garcon` is at `10.64.40.106` with Munin at `192.168.1.21`. In the bundled list, `ps-blw-513723` (formerly `fms-sbl-yue01`, which stays its Munin name) has the nickname `yue01`, and `fms-medc-500555` (formerly `fms-medc-500555 (coffee20)`) has the nickname `coffee20`. `NUSBF-raw1` (exports `/data`, mounted as `/home/data/qnap-nas-1`) and `NUSBF-raw2` (exports `/raw1`, `/raw2`) are QNAP storage. Switches: `Netgear-12RJ_12SFP` (192.168.1.2) and `switch-101` (192.168.1.101, unconfirmed: "Mark checked" confirms it); the user adds or edits switches on Settings. Other, ping only: `ILOCZ2348053Q` (HP iLO, .129), `idrac-D269ZH3` (Dell iDRAC, .145) and `unidentified-bmc-100`, `-108`, `-120`; their addresses are editable (they are to move to 192.168.2.x later). Other, unidentified: `unidentified-118`, `unidentified-140` and `fms-camb-522261` (.5). `fms-zenkin-001` and `fms-zenkin-003` (formerly `zenkin001`, `zenkin003`; GPU nodes) are marked as shut down; `fms-zenkin-002` is in use. Several of these addresses are DHCP leases of 28/11/2026 and may change.

An already seeded list gets five corrections (retiring `NUSBF-GPU1`; making `NUSBF-4C`, `NUSBF-20C` and `Tundra-9963484` Windows; `garcon`'s addresses) once, at the first start after the Update (marker row `correction:2026-11-28`); entries whose address was edited are left alone. A second one-time correction (marker row `correction:2026-11-28-nicknames`) renames those two entries on an existing list and sets their nicknames; entries whose address was edited are left alone, and a nickname or Munin name already set is kept. A third one-time correction (marker row `correction:2026-11-28-inventory`) makes the two raw entries QNAP storage, clears the wrong notes on them and on `fms-medc-500555`, `fms-camb-18028` and `fms-camb-18029`, renames the zenkin entries and marks them as shut down, and adds the thirteen new entries (only to a list seeded from the bundled file, and not where the name or address is already listed). Entries whose address was edited are left alone; notes written by the user are kept. Updating needs core's own Update, which adds the column `fleet_machine.munin_address`. The Update to v0.1.15 adds the table `fleet_disk_role`. The Update to v0.1.16 adds the table `fleet_host_key` and five columns of `fleet_machine`. The Update to v0.1.17 adds the column `fleet_machine.nickname`. The Update to v0.1.18 adds the columns `fleet_machine.shut_down` and `fleet_machine.nfs_exports`.

## Machine types and Shut down
Each entry has a type, set on Settings. Only Linux machines are ever logged in to: the others have no upgrade or reboot buttons, no SSH key check and no identity card, and the server refuses those requests.

| Type | Health |
|---|---|
| Linux | munin-node (see "Traffic light") |
| QNAP storage | ping, its web page (ports 8080 or 443), and its NFS exports |
| Ping only (switches, management controllers such as iDRAC and iLO) | ping: OK or DOWN |
| Unidentified | ping, always grey: never amber or red |
| Windows | not read yet (grey) |

**NFS exports of a storage device** are judged from what the Linux machines report about their own NFS mounts (Munin plugin `fleet_nfs`) in readings less than an hour old; the device's exports are those entered on Settings, or those found mounted. An export that fails is WARNING; it is CRITICAL only if it fails on two or more machines and answers on none. While no machine reports an export, that check, and so the device, is UNKNOWN. So the exports stay unknown until `fleet_nfs` is installed on the clients.

**Shut down on purpose** (tick box on Settings) is for a machine that is off deliberately: grey SHUT DOWN, not pinged, read or logged in to, no buttons and no warnings. Unticking it switches back.

## Disc roles
Each disc (a mounted file system, as Munin's `df` reports it) has a role: **OS**, **cache** or **data storage**. Only OS and cache discs turn amber or red when they fill up, in space or in inodes. A data storage disc never does, however full: filling up is its purpose. SMART, RAID and BeeGFS mount checks are unchanged. Roles belong to mounted file systems, not to the physical drives on the identity card. The role is shown on the Summary cards, in three table columns (OS discs, Cache discs, Data storage; each lists its discs and sorts by the fullest) and on the machine tab's gauges and checks. File systems kept in memory (such as `/run` and `/dev/shm`) count as cache; cards and the table list them only when they are over the limit.

Default roles, applied in this order:
1. **Cache:** memory file systems, and mount points with `scratch` or `cache` in them, or `tmp`, `temp` or `swap` as a word.
2. **OS:** `/`, `/home` itself, and `/boot`, `/efi`, `/usr`, `/var`, `/opt`, `/root` with anything under them; also BeeGFS metadata and management targets (a path with `beegfs` and `meta`, `metadata`, `mgmt` or `mgmtd`), because a full one stops the file system.
3. **Data storage:** everything else, including BeeGFS storage targets and mounts, data arrays, `/srv` and folders under `/home/`.

**Settings → Discs** (on each Linux machine's row) changes the role of any disc, or returns it to the default. It lists the discs of the machine's last reading; a change is saved at once and applies to the lights without a new reading. It needs the "Settings" right.

## Traffic light
There are six states, each with its own colour, shape, symbol and word. In rank order: **SHUT DOWN** (grey, only for machines marked as shut down), **OK**, **UNKNOWN**, **WARNING**, **CRITICAL**, **DOWN**. UNKNOWN ranks above OK, so a machine without the Fleet Munin plugins shows grey.

A machine's light is that of its worst check, and the reason text is that check's.

- **DOWN** is only for a machine that gives no answer at all. A refused connection counts as an answer, and so does a reply to ping. For a Linux machine: if neither munin-node nor SSH accepts a connection, or it only answers ping, it shows WARNING; if SSH is open, or munin-node accepts but gives no data, it shows UNKNOWN. A Ping only device that answers ping is OK. A QNAP device that answers is WARNING if it does not answer ping or its web page does not answer. An unidentified device is always grey (UNKNOWN), never DOWN. DOWN is shown only while the reading is less than 5 minutes old; an older "no answer" is grey: "No answer when last read (<time>); not read since".
- **Not yet checked:** a machine that answers is UNKNOWN (grey), or WARNING (amber) if a check found something, never red. An unchecked machine that gives no answer is DOWN; its reason reads: `Not checked yet (<that note>). No answer since <time>.` The note is the machine's "Not checked yet, because…" field on Settings.
- Windows machines are grey.
- A machine whose monitoring is switched off in Settings ("Monitored" unticked) is grey (UNKNOWN). A powered-off machine gives no answer and is DOWN, unless it is marked as shut down or is unidentified.
- A checked machine with a critical fault is still CRITICAL (red).

There is no background health timer. Machines are read while a Fleet page has focus: besides the Summary and machine tabs, the tab list on any Fleet page starts reading the machines whose result is more than about a minute old (only for users with the Summary or Machine tabs right).

Default limits (changed on Settings):

| Check | Warning | Critical |
|---|---|---|
| Load per CPU thread | 1 | 2 |
| Memory | 90 % | 95 % |
| OS and cache disc space and inodes | 85 % | 95 % |
| CPU and GPU temperature | 80 °C | 90 °C |

- **Also critical:** failed systemd services, degraded RAID, a failed SMART test, a missing BeeGFS mount.
- **Also warning:** pending updates, a reboot needed, RAID rebuilding, reallocated sectors, an expected NVIDIA GPU not reported.

Stale Munin history shows "No history since <date>".

## Verification
The buttons of a Linux machine work only once it is verified.

**With a recorded fingerprint** this is automatic, with no manual step. Fingerprints recorded at install, per expected hostname, are kept in table `fleet_host_key`, filled from `backend/data/seedMachines.json` (21 machines are bundled). At the first connection the machine must offer exactly that key, otherwise Fleet does not log in. Fleet then logs in with the read key and, if the machine reports the expected hostname, stores the key and marks the machine verified, with the time and "automatic". The record goes with the hostname, so renaming, re-adding or readdressing an entry does not drop it. If the records cannot be loaded at start-up, no new machine is verified.

**Without a recorded fingerprint** Fleet never trusts a key by itself, because a successful key login does not prove which machine answered (any SSH server can accept any key). Once the login and the hostname are right, the machine is blocked (buttons off) until someone with the "Settings" right has compared the offered fingerprint with the machine's own and accepted it on Settings → Verify. It is then verified under that person's name.

Fleet tries about 15 s after the backend starts, then hourly for unverified machines (a page showing the machine, or saving it on Settings, tries sooner, at most every 10 minutes per machine).

**A person is also needed when something is wrong:** another key than the recorded one, a changed key, or another hostname than expected. Every blocked machine, including one without a record, has a red warning on Settings and on the machine tab with the old and offered fingerprints (there is no old one where nothing was recorded or accepted). Someone with the "Settings" right accepts it on Settings → Verify, after comparing the offered fingerprint with the machine's own; for a wrong hostname, accepting also makes the reported name the expected one. A machine also unblocks if the stored key and expected hostname come back, or (hostname) if the expected hostname is corrected on Settings; a key that Fleet stored by itself must then also match the record for that name, otherwise a person accepts it. A changed key is never accepted automatically. It is noticed at the daily identity collection, with "Collect now" or "Verify now", and when a button fails on it. A refused login or no answer raises no warning: Settings shows the last attempt's message. Settings keeps the steps by hand (read host key, accept, Verify now).

At every start-up Fleet also re-checks keys that it stored by itself: one with no record, or differing from the record, is blocked at once until a person accepts it (v0.1.16 verified machines without a record). Keys accepted by a person are left alone.

## Access
There are two layers:
1. **Core's module gate:** it answers `404` to users other than developers and administrators until the module is granted at Setup → Preferences → Modules → Access.
2. **Fleet's own Page Access tab** (`summary`, `machines`, `log`, `settings`), enforced by the server with `403`.

Developers and administrators always have every page. There are no other special roles, and nusbf has no special case.

**What a grant allows:**
- A role granted "Machine tabs" can press upgrade and reboot.
- A role granted "Settings" can change machine addresses and accept SSH host keys.
- A user without "Machine tabs" sees summary cards that do not open.
- Refreshing needs no extra right. "Summary" allows Refresh all and the card buttons; "Machine tabs" allows the button on a machine tab.

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
| `BIN_PING` | `ping` | `ping` program |
| `WEB_PORTS` | `8080,443` | Ports where a QNAP device's web page may answer |
| `SEED_FILE` | | Another machine list for the very first start, for test systems |

The backend's OS user must be able to read the key files and `MUNIN_DB_DIR`, and to run `rrdtool` and `ping`.

## Security
- **Keys:** the private keys stay as files on the MFF host. Their paths and contents are never sent to the browser.
- **SSH:** run with an argument list (no shell), `-F /dev/null`, only the pinned host key, strict checking, and no agent or forwarding. The only thing sent is one fixed word: `identity`, `upgrade` or `reboot`.
- **Before the buttons work:** the machine must be verified (see "Verification").
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
- The Munin plugins `fleet_updates`, `fleet_reboot`, `fleet_services`, `fleet_raid`, `fleet_temps`, `fleet_netlink` and `fleet_nfs`. `fleet_gpu` is added only where `nvidia-smi` exists. `fleet_nfs` is added only where there are NFS mounts (it reports whether each is mounted and answering, 5 s limit per mount, all at once), `fleet_mounts` only with BeeGFS. A machine set up before `fleet_nfs` existed needs `sh install.sh --plugins-only` again from this version. `fleet_nfs` has only been tested offline with fake mount tables. The BeeGFS mount check matches every mount type starting `beegfs` (a local build shows, for example, `beegfs7-7`).
- Debian 13 starts munin-node with `ProtectHome=yes`, which hides `/home`, `/root` and `/run/user` from plugins. Only where a BeeGFS or NFS mount is in one of those, `install.sh` adds `/etc/systemd/system/munin-node.service.d/fleet.conf` with `ProtectHome=read-only`, so munin-node plugins on that machine can see those folders read-only. `uninstall.sh` removes it.

**By hand:**
- If `sshd` uses `AllowUsers` or `AllowGroups`, add both accounts.
- munin-node must allow the MFF server. If it does not, add a `cidr_allow` line in `/etc/munin/munin-node.conf`.

Not tested with `sudo-rs`.
