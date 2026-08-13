# Moov v1.0.38 — Release Notes

Supersedes v1.0.37. Released **2026-08-13**.

---

## ⚠️ Read this before you upgrade

**Every helper must be updated together with the appliance.** From v1.0.38 the
Core refuses work from helpers older than 1.0.38. After upgrading, open
**Connections › Helpers** and use **"Update all"**. Until you do, migrations
that target those helpers will not start.

This is deliberate. Helpers talk to the Core over protobuf, so an *old* helper
silently ignores fields a *new* Core sends — no crash and no error, just a wrong
result. A migration would finish clean and leave the VM misconfigured (for
example without its static IP), and nobody would find out until someone tried to
use that VM. Blocking is the only honest behaviour.

A helper that reports no version at all is also blocked; that only happens on
builds that predate this field.

**If you restore a backup onto a *new* appliance, backend credentials do not
come with it.** Operators, engines, waves, the audit log, the SSH key and the CA
all survive. Veeam and hypervisor credentials do **not**: they are encrypted
with a master key that lives in the original machine's keyring and never travels
in the backup. Recover with `moov-core --recover-master-key <code>` using the
code shown once at bootstrap, or re-enter the credentials from the console.
Relevant if you plan to redeploy rather than upgrade in place.

---

## How to upgrade

**To v1.0.38: deploy the image.** v1.0.37 has no in-place updater, so this
release is delivered as an appliance image (qcow2 / OVA / VHDX) exactly like the
previous ones.

**From v1.0.38 onwards: from the console.** This release introduces in-place
updates under **Settings › Software Update** — signed, verified, and reversible.
There is no need to redeploy an appliance again for a Moov update.

---

## What's new

### In-place updates, signed end to end

Two independent channels:

- **Moov binaries** — the Core, the console, the helper and the appliance
  console are replaced in place. Every update is an Ed25519-signed bundle; the
  appliance verifies the signature and the SHA-256 of every file it contains
  before installing a single byte, and refuses anything the manifest does not
  declare. Downgrades and unsupported version jumps are rejected.
- **Operating system** — Debian and kernel updates for the appliance, applied
  from the console.

If an update leaves the Core unable to start, the appliance **rolls itself
back**: the swap is atomic, a timer is armed before the restart, and the new
Core has to prove it is healthy or the previous version is restored
automatically. Validated by deliberately installing a broken update and watching
the appliance recover on its own.

An outbound HTTP/HTTPS proxy is configurable from the console for sites without
direct internet access, and everything works air-gapped by uploading the bundle
by hand.

### Cold migration no longer stages the whole VM

The helper used to write a full local copy of every disk before sending it. It
now serves the source disk directly and the hypervisor pulls from it. That
removes a complete copy of the VM from both the elapsed time and the helper's
disk requirement — the helper no longer needs room for the largest VM you intend
to migrate.

Cold migrations now leave the VM **powered off** so you can start it in your own
maintenance window.

### Guest networking lands on the right NIC

On multi-NIC VMs the network configuration could be applied to the wrong
adapter. Binding is now done by PCI topology, which is what the guest actually
enumerates, and Windows and Linux VMs keep their addresses across the move.
Windows guests with a **static IP** now reliably come back with it instead of
falling to DHCP.

### Progress you can trust

The transfer bar measured how much of the disk had been *traversed*, not how
much had been *sent*, so a mostly-empty 500 GB disk looked like 500 GB of work.
All three hypervisors now report real transferred bytes, measured before the
copy starts. Dates everywhere follow the appliance's configured timezone.

### Keeping the OS patched, across the whole fleet

The appliance now mirrors the Debian archive for its helpers, so a helper is
patched without ever reaching the internet. Your appliance downloads each
package once and serves it to the rest.

Two things this deliberately does **not** hide from you:

- **A host whose state could not be measured is never shown as "up to date."**
  It says so, with the reason. Being told "I don't know" is worth more than a
  green tick you cannot rely on.
- **A pending reboot is read from the host itself**, not guessed from what is
  left to install. When something else installs a kernel — Debian's own
  automatic security updates, or an administrator over SSH — nothing is left
  pending and the machine keeps running the old kernel until it is restarted.
  That case is now visible.

Debian's unattended security updates stay **on** for the appliance, which is the
only host with internet access, and the console shows when they last ran and
gives you the switch. On helpers they are turned off: there they were dormant
for lack of internet, not protecting anything.

The package cache can be emptied from the console, and it now releases space on
its own when the appliance's disk gets tight — not only when the cache reaches
its own ceiling.

### Restarting hosts, from the same place you patch them

The console told you a host was waiting on a reboot and then sent you to the
hypervisor to do it. Now the fleet table has **Restart selected** next to
**Install on selected**, over the same host list — helpers and the appliance.

Helpers restart first and the appliance last: once the Core is down it cannot
dispatch anything else. Both actions ask for your password. Restarting only the
Moov services is milder than restarting the machine, but it still cuts every
migration in flight, so neither is one stray click away.

Two refusals are deliberate:

- **While an update is waiting to be confirmed, restarting is blocked
  outright.** That is the window where the new binaries are installed and the
  automatic rollback timer is armed; restarting inside it is how an appliance
  ends up unable to boot. Confirm or roll back the update first.
- **If migrations are running, the restart is refused** and tells you how many.
  You can still go ahead once you have seen the number — a stuck migration must
  never lock you out of restarting your own appliance — but it cannot happen
  silently over work in flight.

### If a patch run is interrupted, you can see where it stopped

Patching writes down what it is doing as it goes: when it started, each host
before and after it is touched, and when it closed. Those records survive the
appliance restarting, so a run that was cut in half no longer leaves the console
claiming "installing" forever. The host it stopped on is named.

What this does not do is show a percentage inside a host. `apt` does not publish
partial progress, and a bar that moves on a guess is worse than no bar.

### You are told when Veeam or a destination stops responding

Until now only a fallen helper raised an alert, so a Veeam server or a
hypervisor that had gone away was discovered when somebody launched a migration.
Moov already probed all of them every minute; that probe now raises a
notification — after two consecutive failures, so a single network hiccup does
not wake anybody, and deduplicated for 30 minutes so an outage does not flood
the bell.

### Replacing a healthy helper

The console can now retire a working helper: it powers off and destroys its VM,
and tells you what actually happened rather than returning silently. The guard
that prevents deleting a busy helper by accident stays on by default.

### Preflight

Repository selection and pagination for large inventories, and the scan now
covers only VM image backups — agent, database-plugin and NAS backups no longer
pad the list. Backups in object storage and hardened repositories are classified
correctly instead of falling into the manual bucket.

---

## Fixed

Highlights of what customers on v1.0.37 are hitting today:

- **Proxmox and oVirt destinations were shown as reachable without being
  checked.** After the first successful contact, Moov reused the answer it
  already had instead of asking again, so those destinations reported "up · 0
  ms" indefinitely — including while one was unplugged. The alert introduced in
  this release covered Veeam and HPE VM Essentials only, and there was no way to
  tell from the screen. Destinations are now genuinely probed on every cycle.

- **No checkbox in the console appeared to tick.** They worked — filters and
  selections applied — but the box never changed colour, so every one of them
  looked broken.

- **Failed sign-in attempts were listed as "system"**, which made "hide system
  events" look like it did nothing. A failed sign-in has no user because the
  actor is unknown, which is exactly why an auditor looks for it; it is now
  labelled that way and stays visible.

- **Installing OS updates showed no progress and did not refresh when it
  finished.** Both buttons now show they are working and block each other, and
  the fleet state updates on its own after a restart instead of requiring
  "Check now".

- **Migrating onto a name that was already taken overwrote the existing VM.**
  The collision guard only looked at *running* VMs, so a powered-off VM with the
  same name — the one nobody is watching — was not protected: its disk image was
  overwritten and the VM came back corrupt and unbootable, while the migration
  reported success. The destination image is now checked before it is created,
  and a check that cannot be completed is treated as "the name may be taken"
  rather than "the name is free".
- **Legacy BIOS/MBR Windows VMs** were treated as data disks, skipped the entire
  guest preparation, and did not boot.
- **A running migration could show `0/2 · 0%`** while the VM had already booted;
  **a cancelled wave showed 100 % in green**; and the ETA said "~8s" with half
  the migration still to go.
- **An invalid destination node was only discovered 4–7 minutes in**, after the
  Veeam publish and the whole guest preparation had been spent.
- **HPE VM Essentials**: no backend with SSH pinning configured could migrate,
  the console offered networks libvirt does not know, and service-plan reuse
  failed with "no id in response". A pinning failure on HVM or oVirt also
  reported itself as a Proxmox error, sending you to the wrong hypervisor.
- **Deleting a destination orphaned its credential** — adding it back failed
  with an opaque 500.
- **Windows guests**: the guest agent answered before Windows had finished
  installing drivers, and the agent installer only ever fired once.
- **The audit log's Action filter** only offered the actions visible on the
  loaded page, so anything older could not be selected.
- **"Roll back" did nothing** once an update had been applied.
- **Scheduled backups never ran** on any appliance that restarted more often
  than its own interval. The countdown started from the moment the service
  started, so every restart — an update, a new kernel, a power cut — reset it.
  With the default 48-hour interval an appliance that is restarted every couple
  of days never reached the deadline, and it failed silently: the console said
  "Enabled" and showed a date for the next one. **If you have scheduled backups
  configured, check that you actually have backup files.** The schedule is now
  measured from the last backup on disk.
- **Windows VMs lost their static IP** on every migration, Hot and Cold, and
  came back on DHCP.
- **Instant migrations to Proxmox failed** partway through, and **every Instant
  wave to oVirt failed** at the pre-flight SSH check.
- **Multi-disk Windows VMs booted from the wrong disk** after a Cold migration.
- **A Veeam connection that dropped never recovered by itself** — the token
  refresh returned 401 until somebody re-saved the connection by hand.
- **`rhsrvany.exe` was left behind in `C:\Windows`** on migrated Windows guests,
  and the first-boot helper files were never cleaned up. Both now clean
  themselves, and give up after three attempts rather than retrying forever.
- **Migrations to the wrong hypervisor** when two backends of the same type were
  registered.
- A wave where every item was skipped no longer reports as a success.
- The console now shows the version of the **running binary** alongside the
  image version, and says so plainly when they differ.

Plus the deploy wizard (node and storage selection, validation before it reaches
the hypervisor), oVirt VM names with spaces, Ceph RBD as a Cold destination, and
block storage (LVM / iSCSI / ZFS) for both Instant and Cold.

---

## Compatibility

| | Minimum | Recommended |
|---|---|---|
| **Proxmox VE** | 8.2 | 9.0+ |
| **oVirt / OLVM / RHV** | 4.5 | 4.6+ |
| **HPE VM Essentials** | 1.x | 2.x |
| **Veeam Backup & Replication** | **13.0** — required | 13.1 or newer |
| **Helpers** | **1.0.38** — required | 1.0.38 |

Veeam 13.0 and 13.1 (and newer) are both supported: Moov negotiates the REST
API version from the server rather than assuming one, so a 13.1 backend is
addressed as 13.1.

Veeam 12.3 is not supported: it does not expose the full Data Integration API
that Instant migration depends on.

The Veeam **Mount Server must be Windows**. Linux mount servers cannot serve the
iSCSI path Instant migration uses.

---

## Validated for this release

Every check below was run against a **clean appliance deployed from this
image** — not a development box — and driven from the web console.

**Migrations.** A Rocky 9 guest, one VM per wave, to five destinations in both
modes; each VM verified against its own hypervisor afterwards and then deleted.

| Destination | Instant | Cold |
|---|---|---|
| Proxmox VE | ✅ | ✅ |
| oVirt 4.5 | ✅ | ✅ |
| Oracle Linux Virtualization Manager | ✅ | ✅ |
| HPE VM Essentials 8.1 (Ceph RBD) | ✅ | ✅ |
| HPE VM Essentials 9.01 (Ceph RBD) | ✅ | ✅ |

Verified on the destination, not just reported by Moov: the guest agent
answering with its address on the Instant runs; and on Cold, the VM created and
**powered off**, its disk `virtio_scsi`, bootable, with CPU and memory matching
what preflight read from the source. On HPE VM Essentials the Cold disk is a
native Ceph RBD device, not a file path.

With two backends of the same type registered (oVirt and OLVM), the migration
landed on the one that was selected and nothing appeared on the other.

**In-place updates, against the real distribution point.** New version
discovered from the signed channel, in-app notice and e-mail, download,
signature and hash verification, apply, restart, and the anti-brick timer
committing on a healthy boot.

**Image integrity.** The contents of the shipped qcow2 were compared byte for
byte against the signed update bundle: binaries, console and helper image all
match.

### Checksums

```
81d230852e7c5638d1e72e5202313e24ecaa2469dd634f4e66ab8e205b80d44f  moov-appliance-v1.0.38.qcow2
17b9e3e3562c28c6ce1d761f302d87e351fac91e454ac79b1c9c428a1f12dc53  moov-appliance-v1.0.38.ova
d6e3d723768fe91540de38022c281cf72c039913c4042b2e1abbe86a470c4d2a  moov-appliance-v1.0.38.raw
e940d287b7fcedce5a4f5c437aa6050487bac599772c2869c98daed1e947b096  moov-appliance-v1.0.38.vhdx
```

