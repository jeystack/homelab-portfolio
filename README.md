# Homelab Portfolio – IT Systems Integration & Security Basics

Documented hands-on projects from my retraining as an IT specialist for system integration (*Fachinformatiker für Systemintegration*, IHK) in Germany.

Each project shows what I built, why I made each decision, how I checked the result, and how I would roll it back.

> **Scope:** All testing and configuration work shown here was performed on personally owned equipment or isolated, authorised lab environments for educational purposes.
>
> This is a personal learning lab, not production infrastructure. Device identifiers, credentials and real network addresses have been removed or replaced with placeholders.

---

## Start here

| # | Project | What it shows | Status |
|---|---|---|---|
| 01 | [ThinkCentre M920q – Hardware Intake and BIOS Security Baseline](projects/01-thinkcentre-intake-bios-baseline/README.md) | Checking a used server before use: hardware verification, RAM/SSD health tests, BIOS/UEFI hardening, firmware lifecycle risk, rollback plan | Published |
| 02 | [Home Network Security Baseline – Router, Wi-Fi and Lab Switch](projects/02-home-network-security-baseline/README.md) | Securing the home network before the lab goes live: default credentials removed, two-factor confirmation for router changes, guest isolation, no port forwards, switch baseline | Published |
| 03 | Proxmox VE Host Deployment and Access Control | Hypervisor installation, update repositories, least-privilege admin accounts, two-factor authentication | In progress |
| 04 | Network Design and Segmentation | VLAN plan, firewall between lab networks, management network, isolation testing | Planned |

**Short on time?** Read [Project 01](projects/01-thinkcentre-intake-bios-baseline/README.md). It shows my whole working method in one example.

---

## What this lab demonstrates

| Area | Junior system administration / IT support | Junior cybersecurity / SOC / blue team |
|---|---|---|
| Networking | Router and switch administration, DHCP reservations, configuration backups | Guest isolation, disabling UPnP and port forwards, spotting clear-text management protocols |
| Hardware | Verifying delivered hardware against the order, health testing RAM and SSD | Checking a used device for leftover remote management and previous-owner settings |
| Configuration | BIOS/UEFI setup for an unattended server (virtualisation, power recovery, boot order) | Reducing attack surface: disabling unused interfaces and out-of-band management |
| Lifecycle | Checking firmware versions against vendor support | Documenting an end-of-support risk, its mitigations and when to re-evaluate it |
| Change control | Dated change notes, changelog with a rollback path for every change | Evidence-based verification after each change |
| Documentation | Clear, reproducible notes another admin could follow | Separating private lab notes from sanitised public evidence |

---

## Lab overview

- **Host:** refurbished Lenovo ThinkCentre M920q Tiny (6-core Intel i5, 32 GB RAM, 512 GB NVMe SSD)
- **Hypervisor:** Proxmox VE (installed, hardening in progress)
- **Network:** hardened home router with an isolated guest Wi-Fi; managed 8-port switch with 802.1Q VLAN support; VLAN segmentation planned with a firewall VM between lab networks
- **Planned workloads:** Windows Server (Active Directory, DNS, DHCP), Ubuntu Server, Windows 11 client, isolated security-practice VMs

---

## How I work

1. **Plan before changing.** Each change starts with a goal, a risk check and a rollback idea.
2. **Record the starting state.** Before changing settings, I document how they were found.
3. **Change one thing at a time** and note why.
4. **Verify.** Every change has a check that proves it worked.
5. **Log it.** One changelog line per change: what, why, how to roll back.
6. **Keep secrets out of documentation.** Passwords are kept in a password manager, never in notes or repositories. My private lab notebook is version-controlled with Git and scanned for secrets before each commit.
7. **Publish only sanitised work.** Private notes stay private; this repository contains rewritten, reviewed case studies.

---

## Learning background

Retraining: *Fachinformatiker für Systemintegration* (IHK), BFZ-Essen GmbH, 2026–2028.

Cisco Networking Academy **certificates of completion** (course completions, not professional certifications), completed through BFZ-Essen GmbH:

- Hardware and Upgrade Support
- Operating Systems Basics
- Networking Basics
- Introduction to Cybersecurity
- Cybersecurity Essentials / Junior Cybersecurity Analyst learning path

---

## Roadmap

1. Hardware intake and BIOS/firmware security baseline ✅
2. Home network security baseline ✅
3. Proxmox host deployment *(in progress)*
4. Network design and segmentation
5. Windows and Linux administration
6. Automation with Bash, PowerShell and Python
7. Security monitoring or an incident-response mini-lab

---

## Contact

Jeremy Njumbe – [LinkedIn](https://www.linkedin.com/in/j-njumbe)
