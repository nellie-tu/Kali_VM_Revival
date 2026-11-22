# Reviving a 2-Year-Old Kali Linux VM: Upgrade & Troubleshooting Report

A walkthrough of bringing a Kali Linux virtual machine (Oracle VirtualBox) back to a fully updated, working state after roughly two years without updates. It covers what broke, why it broke, how each problem was diagnosed, and what I'd do differently.

> **TL;DR:** A long-dormant Kali rolling install couldn't reach the internet and couldn't update. Five separate problems stacked on top of each other: a misconfigured VirtualBox network mode, an outdated repository signing key, package file conflicts caused by two years of package renames, an outdated systemd unable to parse new config syntax, and a mid-upgrade `/tmp` change. All were resolved without reinstalling. The system now runs kernel 7.2 with all packages current.

---

## Environment

| Item | Detail |
|---|---|
| Hypervisor | Oracle VM VirtualBox |
| Guest OS | Kali Linux (rolling), last updated ~2 years prior |
| Desktop | Xfce |
| Kernel before | 6.8.11-amd64 |
| Kernel after | 7.2.15+kali-amd64 |
| Scale of upgrade | ~900 upgraded packages, ~575 new, ~170 removed, ~2 GB download |

---

## Summary of Issues

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 1 | No internet in VM (`ping` → unreachable) | VirtualBox adapter set to **NAT Network** instead of **NAT** | Switched adapter to NAT |
| 2 | `apt update` would fail with `Missing key 827C8569...` | Kali rotated its archive signing key in April 2025 | Installed the new keyring manually |
| 3 | `dpkg: error ... trying to overwrite '...', which is also in package ...` | Packages renamed or split over two years (e.g. `t64` transition); old packages still owned files | `dpkg --force-overwrite` for the specific packages, then `apt --fix-broken install` |
| 4 | `/usr/lib/sysusers.d/*.conf: Unknown modifier 'u!'` | New packages used sysusers syntax that the old systemd didn't support | Upgraded `systemd` first, then reconfigured |
| 5 | `cannot access archive '/tmp/apt-dpkg-install-...': No such file or directory` | `/tmp` contents disappeared mid-upgrade (likely the new systemd mounting a fresh `/tmp`) | Re-ran the install, which used apt's package cache |

---

## Timeline & Troubleshooting

### 0. Preparation: snapshot first
Before changing anything, I took a **VirtualBox snapshot** as a rollback point.

*Snapshot vs. clone:* a snapshot is a quick save point that depends on the original disk, which is ideal as an undo button for a risky upgrade. A clone is a full independent copy, which is better as a true backup or for running two VMs side by side.

### 1. No network connectivity

**Diagnosis, layer by layer:**

```bash
ip a                  # Does the interface have an address?
ping -c 4 8.8.8.8     # Can we reach the internet by IP?
ping -c 4 kali.org    # Does DNS work?
```

- `eth0` was `UP` with `inet 10.0.2.14/24` (an address assigned by DHCP), so the adapter itself was working.
- `ping 8.8.8.8` → **100% packet loss**; pinging the gateway → **Destination host unreachable**.
- The host laptop had working internet, so the problem sat between the VM and VirtualBox.
- `10.0.2.14` was a clue: VirtualBox's plain NAT mode normally hands out `10.0.2.15`.

**Root cause:** Adapter 1 was attached to **NAT Network**, a separate VirtualBox mode, probably left over from an old lab setup.

**Fix:** Powered off the VM and changed *Settings → Network → Adapter 1 → Attached to* to **NAT**. After booting, the ping succeeded with 0% packet loss and DNS resolved.

> `docker0` showing `DOWN` in `ip a` was a red herring. It's Docker's bridge, and it's normally down when no containers are running.

### 2. Repository signing key

In April 2025 the Kali team lost access to the repository signing key and issued a new one. Any install from before that date fails `apt update` with:

```
Missing key 827C8569F2518CC677FECA1AED65462EC8D5E4C5, which is needed to verify signature.
```

**Fix** (from the official Kali announcement):

```bash
sudo wget https://archive.kali.org/archive-keyring.gpg -O /usr/share/keyrings/kali-archive-keyring.gpg
```

> Gotcha: my first attempt failed with `wget: invalid option -- '0'`. I had typed a zero instead of a capital **O**. Copy-paste avoids this.

### 3. Package file conflicts (renames over two years)

`sudo apt update && sudo apt upgrade -y` got partway and stopped:

```
dpkg: error processing archive .../python3-pyspnego_0.10.2-4_all.deb (--unpack):
 trying to overwrite '/usr/bin/pyspnego-parse', which is also in package python3-spnego
```

**Root cause:** Over two years many packages were renamed or split, most notably Debian's **64-bit time_t (`t64`) transition** (`libssl3` → `libssl3t64`, `libgtk-3-0` → `libgtk-3-0t64`, and so on). When jumping straight from old to new, the old package still owns files the new one needs, and dpkg refuses to overwrite them.

Conflicts I hit:

| New package | Clashed with old package |
|---|---|
| `python3-pyspnego` | `python3-spnego` |
| `libboost1.92-dev` | `libboost1.74-dev` |
| `pyinstaller-hooks-contrib` | `python3-pyinstaller-hooks-contrib` |
| `openssl-provider-legacy` | `libssl3` |
| `libgtk-3-0t64` | `libgtk-3-0` |
| `libcurl4-gnutls` | `libcurl3-gnutls` |

**Fix, one package at a time:**

```bash
sudo dpkg -i --force-overwrite /var/cache/apt/archives/<package>.deb
sudo apt --fix-broken install
```

**Fix, all remaining at once** (once it was clear every conflict was a harmless rename):

```bash
sudo apt -o Dpkg::Options::="--force-overwrite" --fix-broken install
sudo apt -o Dpkg::Options::="--force-overwrite" full-upgrade
```

### 4. Outdated systemd couldn't read new config syntax

Five packages (`rtkit`, `uuid-runtime`, `rwhod`, `rwho`, `inetsim`) unpacked but failed to configure:

```
/usr/lib/sysusers.d/rtkit.conf:1: Unknown modifier 'u!'.
```

**Root cause:** These packages describe their system users in a newer `sysusers.d` format. The tool that reads those files (`systemd-sysusers`) was still the two-year-old version. Plain `apt upgrade` holds back core packages like systemd, so it hadn't been updated.

**Fix:**

```bash
sudo apt install systemd     # 255.x → 262
sudo dpkg --configure -a
```

All five packages then configured successfully.

### 5. Temporary install files vanished mid-upgrade

```
cannot access archive '/tmp/apt-dpkg-install-xtc4CM/21-python3-scipy_...deb': No such file or directory
```

**Likely cause:** Newer systemd mounts `/tmp` as a fresh tmpfs. Once the upgraded systemd took effect mid-run, apt's temporary staging directory disappeared.

**Fix:** Re-ran `sudo apt --fix-broken install`. The `.deb` files were still in `/var/cache/apt/archives/`, so nothing needed downloading again.

### 6. Finishing up

- Ran `full-upgrade` repeatedly until it reported **`Upgrading: 0, Installing: 0, Removing: 0, Not Upgrading: 0`**, and only then rebooted. Rebooting a half-upgraded system onto a new kernel risks a broken desktop or no network.
- Post-reboot checks:
  - `uname -r` → `7.2.15+kali-amd64` ✅
  - `ping -c 4 kali.org` → 0% packet loss ✅
  - VirtualBox window resizing → desktop adjusts (guest utilities working) ✅
- `sudo apt autoremove` removed 333 orphaned packages (~1.8 GB). Before confirming, I reviewed the list for real tools with a dry run:

```bash
apt -s autoremove 2>/dev/null | grep '^Remv' | awk '{print $2}' \
  | grep -vE '^(lib|python3|linux-|golang|gcc|perl|gir1|fonts|ruby)' | column
```

  The only tool removed was `crackmapexec`, which is discontinued and replaced by **NetExec** (`nxc`). I confirmed NetExec was installed with `which nxc`.

---

## Decisions Made Along the Way

| Prompt / choice | Decision | Reasoning |
|---|---|---|
| Docker: auto-delete data on purge? | **No** | Keeps data safe; irrelevant to the upgrade |
| PostgreSQL: migrate cluster 16 → 18 now? | **No** | Avoids a database migration in the middle of a huge upgrade; can run `pg_upgradecluster 16 main -v 18` later |
| Config file conflicts | **Keep local version** | Safe default |
| `upgrade` vs `full-upgrade` | **`full-upgrade`** | `upgrade` never adds or removes packages, so it held back ~900 packages that needed renames |
| Review removals before confirming | **Always** | Checked that no `kali-linux-*` / `kali-desktop-*` metapackages were removed |

To check that `full-upgrade` wouldn't remove Kali metapackages:

```bash
apt -s full-upgrade 2>/dev/null | grep '^Remv' | grep kali
```

---

## Bonus: VPN IP in the Xfce Panel

A leftover **Generic Monitor (genmon)** panel plugin was showing `(genmon)XXX` with no command configured. I repurposed it to show my TryHackMe VPN IP:

```bash
sh -c "ip -4 addr show tun0 2>/dev/null | grep -oP '(?<=inet )[\d.]+' || echo 'No VPN'"
```

Settings: Label unchecked, Period 5 s. The panel shows `No VPN` until OpenVPN connects, then the `tun0` lab IP.

---

## Lessons Learned

1. **Snapshot before any major change.** It turns a risky upgrade into a reversible one.
2. **Troubleshoot networking layer by layer:** interface → gateway → internet by IP → DNS. Each step narrows down where it fails.
3. **Read the actual error.** Every failure here named its own cause (`trying to overwrite`, `Unknown modifier 'u!'`, `No such file or directory`). Scrolling up to the first error was always the key.
4. **`upgrade` and `full-upgrade` are different tools.** Long gaps on a rolling distro need `full-upgrade`.
5. **Dry-run (`apt -s`) before removals.** It's a cheap way to check what will be removed before committing.
6. **Don't reboot a half-upgraded system.** Finish until apt reports zero pending changes.
7. **Update regularly.** A rolling release updated every couple of weeks never builds up two years of conflicts.

## Follow-ups

- [ ] Delete the pre-upgrade snapshot after a few days of stable use; take a fresh baseline snapshot
- [ ] Migrate PostgreSQL 16 → 18 if/when Metasploit's database (`msfdb`) is needed
- [ ] Investigate leftover `elastic-agent` / `ElasticEndpoint` services
- [ ] Routine maintenance: `sudo apt update && sudo apt full-upgrade` every couple of weeks

## References

- [Kali Linux: A New Kali Linux Archive Signing Key](https://www.kali.org/blog/new-kali-archive-signing-key/)
- [Kali Docs: Resolving APT errors caused by an expired signing key](https://www.kali.org/docs/general-use/gpgkey-expiry/)
- Claude (Anthropic): used as a step-by-step troubleshooting guide; all commands were run, verified, and documented by me
