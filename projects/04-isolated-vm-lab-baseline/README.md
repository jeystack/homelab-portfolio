# Isolated Virtual Lab – Network Isolation, Verified Installers and Server Baselines

**Project type:** Personal homelab (laptop, VMware Workstation) · **Date:** October 2026 · **Status:** Part 1 complete (isolated network, verified installers, two server VMs with snapshots). Active Directory, DHCP, a Windows client, an attack VM and the final isolation test follow in part 2.

> **Scope:** All testing and configuration work shown here was performed on personally owned equipment or isolated, authorised lab environments for educational purposes.
>
> This was a personal learning project on my own laptop. It is not production infrastructure. IP addresses, user names, file paths and credentials have been removed.

---

## 1. Summary

Before I build an Active Directory lab and practise with security tools, I wanted a place where mistakes cannot reach any real network. I built a second lab, separate from the Proxmox server, on my laptop with VMware Workstation, and **made the isolation the first step instead of the last**.

Part 1 covers:

1. **An isolated virtual network** with no DHCP, no internet route and no adapter on the host, created before any VM existed.
2. **Verified installers:** official sources only, integrity checked by hash where the vendor publishes one and by digital signature where it does not.
3. **Windows Server 2025 (evaluation)** as the future domain controller, installed, named and given a fixed address.
4. **Ubuntu Server LTS** as a Linux server, installed in UEFI mode with a fixed address.
5. **Communication test between the two servers**, including a narrow firewall rule when Windows blocked the test.
6. **Clean snapshots** of both machines as return points.

The most useful results were the things that went wrong and that I caught: a wrong installer, a network card that silently changed type, a Linux install in the wrong firmware mode, and a firewall that made a working connection look broken.

---

## 2. Objective

- Have a lab where vulnerable systems and scanning tools can never touch my home network or the internet.
- Prove that the installers are the genuine files before I run them.
- Build repeatable server baselines (name, fixed address, snapshot) for the later Active Directory work.
- Document every deviation from the plan and how I found it.

---

## 3. Environment and scope

| Item | Details |
|---|---|
| Host | Windows 11 Pro laptop, 32 GB RAM, VMware Workstation Pro (free for personal use), host disk encrypted |
| Lab network | One custom host-only virtual network: **no DHCP service, no host adapter, no NAT, no bridge** |
| VM 1 | Windows Server 2025 Standard, evaluation, Desktop Experience; 2 vCPU, 4 GB RAM, 60 GB NVMe disk (thin) |
| VM 2 | Ubuntu Server LTS; 1 vCPU, 2 GB RAM, 25 GB NVMe disk (thin) |
| References | BSI IT-Grundschutz (SYS.1.5 virtualisation, OPS.1.1.5 logging, CON.3 backup concept), CIS Controls v8 |

**In scope:** isolated network, installers, base installation of two servers, fixed addresses, snapshots.

**Out of scope (part 2):** Active Directory, DNS, DHCP, a Windows client, a Kali VM and the full isolation test (see section 11).

---

## 4. My work / implementation

### 4.1 Isolated network first

- Created a new virtual network in the Virtual Network Editor as **host-only**, with *"use local DHCP service"* and *"connect a host virtual network adapter"* both switched **off**. The editor shows this combination as type *Custom*.
- The default networks stayed untouched, and I did not use *Restore Defaults* later.
- **Check on the host** (positive and negative control): `ipconfig` lists the adapters of the two default networks, and a search for the new subnet finds nothing. So the check can see adapters, and the lab network is not one of them.
- The VMware library was empty, so no existing VM depended on the default networks.

### 4.2 Installers from official sources, integrity verified

| Installer | Source | How verified |
|---|---|---|
| Ubuntu Server LTS | ubuntu.com | `(Get-FileHash <file>).Hash -eq $expected` against the published SHA-256 → `True` |
| Kali Linux (for part 2) | kali.org | Same hash comparison → `True` |
| Windows Server and Windows 11 (evaluation) | microsoft.com | The vendor publishes no hash. Downloaded over HTTPS, mounted the ISO read-only and ran `Get-AuthenticodeSignature` on the installer → **Valid**, signed by Microsoft Corporation |

- I compare hashes with `-eq`, never by eye.
- **Accepted residual risk:** the signature covers the installer, not every byte of the ISO.
- **Mistakes caught before use:** my first Windows Server download was the *Languages and Optional Features* ISO (not installable), and my first Kali download was a *weekly* build instead of the release. I cancelled both. Lesson: check the **file name**, not just the website.

### 4.3 Windows Server VM

- Created with the *Custom* wizard and *install OS later* (Easy Install would hide choices), UEFI with Secure Boot, NVMe thin disk, **network card on the lab network only**, sound card and printer removed (fewer virtual devices).
- Chose **Desktop Experience** on purpose for learning; Server Core and Desktop Experience cannot be switched later without reinstalling.
- Boot problems, all resolved:
  1. I missed the "press any key" prompt and the VM fell through to a network boot. It failed because there is no DHCP on the lab network. That was an **unplanned proof of the isolation**.
  2. The installer found "no media" because the ISO path had been saved relative to an old location. Fix: select the ISO again with its full path.
  3. Shift+F10 in setup opened a command prompt with SYSTEM rights. A reminder why physical access to a machine must be protected (BIOS password, boot order, disk encryption), as in [Project 01](../01-thinkcentre-intake-bios-baseline/README.md).
- Diagnostic data set to **Required only** (data minimisation, GDPR Art. 5).
- Installed VMware Tools, renamed the computer to `DC01` with `Rename-Computer`. I had not noted the old name, but it was recoverable from the System log (**event ID 6011**).
- **Fixed address** with `New-NetIPAddress` and `Set-DnsClientServerAddress`, **deliberately without a gateway** (the lab has no route out). Before: APIPA address, which confirmed again that no DHCP exists on the lab network. After: address state *Preferred*. Right after creation it showed *Invalid* while the duplicate-address check was still running.
- IPv6 left on, because Microsoft advises against disabling it on domain controllers.

### 4.4 Ubuntu Server VM

- Same wizard approach: UEFI, thin NVMe disk, network card on the lab network only. I removed the extra CD/DVD drive set to *auto detect* (it links to the laptop's physical drive) and switched off 3D acceleration (no GUI needed).
- Installation choices: guided **LVM without LUKS** (the host disk is already encrypted, and LUKS would need a passphrase at every boot), a non-default user name, **OpenSSH server only**, no snaps, no Ubuntu Pro.
- **Firmware mistake found:** the installer's summary does not show the firmware mode, and my first install ran in **BIOS mode**. The check `[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS` printed `BIOS`. I switched the VM to UEFI and reinstalled; the second check printed `UEFI` and the disk now has an EFI system partition.
- **Known limitation:** Secure Boot could not be enabled for this VM type. I accepted it for an isolated lab and will retry later.
- The installer's default root volume uses only about half of the volume group. This is a planned exercise for `lvextend` and `resize2fs`.
- On first boot the machine waited about two minutes for a network address and then continued without one. This is correct on a network without DHCP.
- **Fixed address with netplan:** I made a backup of the file, added `dhcp4: false` and the address, and applied it with **`netplan try`**, which rolls back automatically after 120 seconds if I lose the connection. YAML mistakes I caught by reading the file (`sudo cat`) before applying: lines at the wrong indentation level and `dhcp:` instead of `dhcp4:`. In YAML the indentation is the meaning.
- The first SSH connection will need a host key fingerprint check, as in [Project 03](../03-proxmox-host-access-control/README.md).

### 4.5 Connection test between the servers, and the firewall

| Test | Result | Meaning |
|---|---|---|
| Windows server pings Linux server | 4/4 replies, TTL 64 | Network and addresses are correct; TTL 64 is typical for Linux |
| Linux server pings Windows server | **100 % loss** | Not a network fault: the Windows firewall rule for ICMP echo is **disabled by default** |

"No reply" does not mean "no connection". Instead of switching the firewall off, I created a **narrow rule**:

```powershell
New-NetFirewallRule -DisplayName "Lab: Allow ICMPv4 echo from <lab-subnet>" `
  -Direction Inbound -Protocol ICMPv4 -IcmpType 8 `
  -RemoteAddress <lab-subnet> -Action Allow -Profile Any
```

It allows only echo requests, only from the lab subnet (least privilege). After that the Linux server received replies with TTL 128, which is typical for Windows. **Rollback:** `Remove-NetFirewallRule -DisplayName "Lab: Allow ICMPv4 echo from <lab-subnet>"`.

### 4.6 Snapshots

- Took a snapshot named `00-clean` of each VM **while powered off**: installed system, tools, name and fixed address, no roles yet.
- A snapshot is **not a backup**: it lives on the same storage as the VM, so a damaged disk file loses both. A real backup (Windows Server Backup to a second disk, a written backup concept) is a later exercise.
- The snapshot of the Windows server was taken before the firewall rule; the rule is documented separately with its rollback.

---

## 5. Key decisions and rationale

| Decision | Alternative considered | Why I chose this |
|---|---|---|
| Isolation before the first VM | Add isolation later | A wrong default on the first VM would already be a leak |
| Host-only network without host adapter or DHCP | NAT or default host-only network | No route out, nothing from the host reachable, and a missing lease proves the isolation |
| No default gateway on the servers | Gateway for updates | Updates go through a separate, deliberate process, never "just this once" |
| Hash where published, signature where not | Trust the website | Integrity and, with a signature, authenticity |
| Custom wizard, OS installed later | Easy Install | Every virtual device and setting is chosen consciously |
| Desktop Experience for the first server | Server Core | Better for learning; Core is for later PowerShell-only exercises |
| LVM without LUKS | Full-disk encryption in the VM | The host disk is encrypted; LUKS would need a passphrase at each boot |
| Narrow firewall rule | Firewall off, or allow all ICMP | Least privilege, easy to remove |
| Snapshot while powered off | While running | Consistent state, no memory dump to store |

---

## 6. Validation and evidence

| Check | Result |
|---|---|
| Lab network adapter on the host | None (positive control: default adapters listed) ✅ |
| Hash comparison, Ubuntu and Kali | `True` ✅ |
| Signature of Microsoft installers | Valid, signer Microsoft Corporation ✅ |
| Network card of each VM | Lab network only (checked in the final summary and the hardware list) ✅ |
| Network boot without DHCP | Failed, as intended ✅ |
| Linux server firmware mode | `UEFI` after the reinstall ✅ |
| Fixed addresses | Active on both servers ✅ |
| Windows → Linux ping | 4/4 ✅ |
| Linux → Windows ping before the rule | 100 % loss (firewall) → explained ✅ |
| Linux → Windows ping after the rule | Replies ✅ |
| Snapshots | `00-clean` exists on both and is marked as current ✅ |

Screenshots of this part contain addresses and local paths and are therefore not published; the results are recorded as text.

### Rollback plan

| Component | Rollback |
|---|---|
| Lab network | Virtual Network Editor → remove the network |
| A VM | Right-click → Manage → Delete from Disk |
| Computer name / address | `Rename-Computer` back; remove the address or set the adapter to DHCP |
| Linux address | Restore the netplan backup and `netplan apply` |
| Firewall rule | `Remove-NetFirewallRule` with the rule name |
| Any state | *Go to snapshot* `00-clean` |

---

## 7. Outcome

- A lab network that has no DHCP, no gateway, no NAT and no adapter on the host.
- Installers whose integrity or authenticity I verified.
- Two server baselines with fixed addresses and clean snapshots.
- Verified server-to-server communication, with a minimal firewall exception.
- A list of mistakes I found myself, each with its cause and a check that would have found it earlier.

---

## 8. Defender's view

| Weakness | How it could be abused | Now |
|---|---|---|
| Installer from an unofficial mirror or a wrong build | Tampered or unsupported image | Official source, hash or signature check, file name check |
| VM network card defaulting to NAT or bridge | A vulnerable or scanning VM reaches the home network or the internet | Lab network only, checked in the final summary of each VM, full isolation test planned |
| Convenience features (shared clipboard, host drives, auto-detect CD) | A channel out of the isolation | Extra drives removed; clipboard sharing to be decided per VM |
| Disabled firewall "to make the ping work" | Permanent hole that nobody remembers | One narrow, named rule with a rollback |
| Physical access to a machine in setup mode | A SYSTEM shell before the OS is installed | Disk encryption on the host, boot order and BIOS password as in Project 01 |
| Snapshot treated as backup | Data loss when the disk file is damaged | Documented as "not a backup"; real backup is on the list |

---

## 9. Skills demonstrated

**System administration / IT support**
- Virtual network design (host-only, no DHCP, no host adapter), VM hardware choices
- Windows Server installation, naming, fixed addressing with PowerShell, Windows Firewall rules
- Ubuntu Server installation (UEFI, LVM), netplan with `netplan try`
- Event log analysis (event ID 6011), firmware mode check, snapshots
- Change documentation with rollback

**Security / blue team**
- Isolation as a design principle, with positive and negative controls
- Installer integrity (hash) versus authenticity (digital signature)
- Least-privilege firewall rule instead of disabling a control
- Reading a failed test correctly ("no reply is not no connection")
- Data minimisation setting (GDPR)

**FISI exam topics touched:** IPv4 addressing and subnets, DHCP, APIPA, TTL, ICMP, host-only versus NAT versus bridged networking, UEFI versus BIOS, partitioning and LVM, encryption (hash versus signature), firewall rules, backup versus snapshot, technical documentation.

---

## 10. Lessons learned

- **Check the final summary, not just each step.** Going back in the VM wizard reset the network page, and a newly added network card defaults to NAT.
- **A summary can hide the setting that matters.** The Ubuntu installer does not show the firmware mode. I verified it on the running system.
- **Paths saved by a tool can be relative.** A moved folder broke the ISO link.
- **Read the file before applying it.** Two YAML mistakes were visible to the eye and would have broken the network configuration. `netplan try` is the safety net, not the plan.
- **Hash is integrity; signature is integrity and authenticity.**
- **A failed ping needs a diagnosis,** not a firewall switched off.
- **A snapshot is a return point, not a backup.**
- **A missing note can often be recovered from logs** (the old computer name from event 6011), but it is better to write it down first.

---

## 11. What I would improve next

- [ ] Promote the Windows server to a domain controller, check DNS, take a snapshot.
- [ ] Add DHCP with a scope, authorise it in Active Directory.
- [ ] Install a Windows 11 client, get a lease from the domain controller, join the domain.
- [ ] Import the Kali VM on the lab network and decide about clipboard sharing.
- [ ] **Isolation test:** from the attack VM no internet and no host reachable, scans find only lab machines.
- [ ] Retry Secure Boot on the Linux VM; extend the LVM root volume.
- [ ] Real backup of the domain controller (second disk, restore test).

---

**Previous project:** [03 – Proxmox VE Host](../03-proxmox-host-access-control/README.md) · **Next project:** Network Design and Segmentation *(planned)* · [Back to overview](../../README.md)
