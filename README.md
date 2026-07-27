# home.lab.media.server
This is the timeline of my home lab journey
# Self-Hosted Media Server on a Proxmox Homelab

A from-scratch homelab built on a repurposed HP ProDesk 400 G3, running a
Proxmox VE hypervisor with a virtualised Ubuntu Server host for a
Dockerised Jellyfin media stack.

This project was built to develop hands-on skills in virtualisation,
Linux administration, containerisation, and — as it turned out — a fair
amount of network troubleshooting. I'm documenting the whole thing,
including the parts that didn't work the first time, because working
through those problems is where most of the real learning happened.

---

## Why this project

I'm building toward a career in **cybersecurity**, and a homelab is one of
the best ways to learn the fundamentals that security work is built on:
how machines get addresses, how services listen on ports, how isolation
between systems actually works, and how to methodically diagnose a system
that isn't behaving. You can't secure infrastructure you don't understand,
so I set out to build some.

The goal was deliberately practical: stand up a hypervisor, run an isolated
VM on it, and host a real service (Jellyfin) inside a container — so that a
misconfigured experiment can never take down the things I care about.

---

## Architecture

The design principle is layered isolation. Each layer sits inside the one
before it, so a problem in an experimental VM can't reach the media stack.

```
HP ProDesk 400 G3 (bare metal)
  └─ Proxmox VE 9.2  ............ hypervisor / management plane
       └─ Ubuntu Server 26.04 LTS (VM)  ... isolated guest
            └─ Docker  ................... container runtime
                 └─ Jellyfin  ........... media service (port 8096)
```

| Layer      | Role                        | Address / Port          |
|------------|-----------------------------|-------------------------|
| Proxmox    | Hypervisor & web management | `:8006`                 |
| Ubuntu VM  | Guest OS running containers | (own DHCP lease)        |
| Jellyfin   | Media streaming service     | `:8096`                 |

---

## Tech stack

- **Hypervisor:** Proxmox VE 9.2
- **Guest OS:** Ubuntu Server 26.04 LTS
- **Containerisation:** Docker + Docker Compose
- **Application:** Jellyfin (self-hosted media server)
- **Hardware:** HP ProDesk 400 G3 (repurposed office desktop)

---

## Build overview

1. Flashed the Proxmox VE ISO to USB and installed it to bare metal,
   wiping the previous OS.
2. Reached the Proxmox web console and created an Ubuntu Server VM.
3. Installed Ubuntu Server as a guest, enabling OpenSSH for remote admin.
4. Installed Docker and Docker Compose inside the guest.
5. Deployed Jellyfin via a Compose file (see `docker-compose.yml`).
6. Verified the service end-to-end from another device on the LAN.

### Problem 1 — Web console unreachable despite the host being "up"

**Symptom:** After install, the Proxmox dashboard at `:8006` timed out from
my laptop, even though the host was clearly running.

**Diagnosis:** Comparing addresses showed the two machines were on
different subnets — the host on `192.168.100.x`, my laptop on
`192.168.0.x`. My router (a Telstra/NBN gateway) was handing out two
different address ranges: one on its Ethernet ports and another over
Wi-Fi. `ping` between them failed, confirming they couldn't route to
each other.

**Key commands used:**
```bash
ip addr            # inspect interface addresses on the host
ip route           # confirm the default gateway
ping <target>      # test reachability between machines
```

**Wrong turn** I first tried forcing a *static* IP on
the host to match the laptop's subnet. This made things worse — it put the
host on an address that didn't match the network segment its cable was
physically plugged into, so nothing could reach it at all.

**Actual fix:** Reverted the interface to **DHCP** so the host would simply
request a valid address from whichever network it was actually connected
to. Once switched, it pulled a correct `192.168.0.x` lease, landed on the
same subnet as the laptop, and the dashboard loaded immediately.

**Lesson:** Match the host to the network it's on — don't fight the
topology. And confirm the physical layer which port feeds which subnet
before touching software config.

### Problem 2 — VM had no network connectivity

**Symptom:** Ubuntu installed fine, but had no internet —
`ping` returned "temporary failure in name resolution", and `ip addr`
showed only the `127.0.0.1` loopback, no LAN address.

**Diagnosis:** The VM had been created without a working virtual network
interface — the bridge dropdown in the VM creation wizard was rendering
empty in the browser, so no NIC was attached. I confirmed the underlying
bridge itself was healthy from the host shell:
```bash
ip link show vmbr0   # confirmed vmbr0 exists and is UP
```
So the bridge was fine; the VM just wasn't attached to it.

**Fix:** Bypassed the broken UI and attached the interface directly from
the Proxmox host CLI:
```bash
qm set <vmid> --net0 virtio,bridge=vmbr0
qm config <vmid> | grep net   # verify it took
```
After a reboot the VM pulled a DHCP lease and had full connectivity.

**Lesson:** When a GUI misbehaves, drop to the layer beneath it. Verifying
the bridge from the CLI proved the problem was the UI, not the network —
which saved me from "fixing" something that wasn't broken.

---
- **Isolation by design.** Experimental VMs are separated from the media
  service, so a compromised experiment can't reach it.
- **No inbound exposure.** The service is LAN-only. Nothing is
  port-forwarded to the internet. Remote access, when I add it, will go
  through a VPN (Tailscale) rather than exposing a port directly to the
  open internet.
- **Least privilege (in progress).** Running admin over SSH rather than at
  the console; next step is key-based auth and disabling password login.

### Planned hardening (roadmap)

- [ ] SSH key authentication + disable password login on the VM
- [ ] Non-root user with `sudo` for day-to-day admin
- [ ] DHCP reservation so the host keeps a stable address
- [ ] Network segmentation with VLANs (pfSense/OPNsense) to wall off lab VMs
- [ ] Tailscale for secure remote access without inbound port forwarding
- [ ] Automated backups of VM and container config
- [ ] Centralised logging / basic monitoring

---

## Repo contents

- `README.md` — this file
- `docker-compose.yml` — the Jellyfin service definition
- `.gitignore` — excludes local config, cache, and media

---

## What I learned

- Installing and administering a Type-1 hypervisor (Proxmox VE)
- Provisioning and managing Linux VMs
- Diagnosing layer-3 networking issues across subnets (`ip`, `ping`, routing)
- Managing Proxmox VMs from the CLI when the GUI falls short (`qm`)
- Containerising and deploying a service with Docker Compose
- Reasoning about isolation, exposure, and least privilege in a real build
