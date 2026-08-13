# Changelog

All notable changes to Moov are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and versions follow
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

This file starts at **v1.0.37**, the first release published from the public
repository.

Each release also has a customer-facing note under
[`docs/releases/`](docs/releases/) with upgrade instructions and the support
matrix; this file is the running technical record.

## [Unreleased]

Nothing yet.

## [1.0.38] — 2026-08-13

Supersedes v1.0.37. Full note:
[`docs/releases/RELEASE-NOTES-v1.0.38.md`](docs/releases/RELEASE-NOTES-v1.0.38.md).

### ⚠️ Breaking

- **Veeam Backup & Replication 13.1 is supported.** The REST API revision is negotiated from the server's 
  own serverInfo build version instead of being pinned, so a 13.1 backend is addressed as 13.1 and a 13.0 
  one as 13.0 — with no setting to choose. Until now the revision was fixed, and 13.1 broke the pre-flight 
  scan outright: it sends creationTime without a timezone, which the parser rejected. If serverInfo cannot 
  be read the default revision is kept, so negotiation never blocks a connection. 
  Veeam 13.0 remains the minimum.
- **Helpers older than 1.0.38 are refused.** The Core no longer dispatches work
  to them, and a helper that reports no version at all is refused too. Helpers
  speak protobuf, so an old helper silently ignores fields a new Core sends: the
  migration finishes clean and leaves the VM misconfigured — no crash, no error.
  Blocking is the only honest behaviour. Upgrade the fleet from
  **Connections › Helpers › Update all** right after upgrading the appliance.
- **Upgrade to v1.0.38 by deploying the image.** v1.0.37 has no in-place
  updater. From v1.0.38 onwards updates are applied from the console.

### Added

- **Restart hosts from the console.** The OS channel raised "reboot pending"
  and offered no way to act on it: the only restart lived in the appliance
  console, behind the hypervisor. The fleet table now has **"Restart
  selected"** next to "Install on selected", over the same host selection, for
  helpers and the appliance alike. Helpers restart first and the appliance
  last — once the Core is down it can no longer dispatch anything. Both
  restarting the Moov services and restarting the appliance ask for the
  operator's password: the first is milder than the second, but it still cuts
  every migration in flight, so neither should be one stray click away. A host
  waiting on a kernel it has already installed has nothing left to install, so
  the old "selectable = has something to install" rule left exactly the host
  you needed to pick greyed out; selection now means "there is something to do
  with this host".

- **The OS rollout is recorded as it goes.** It writes an event when it starts,
  one per host before and after touching it, and one when it closes. Those rows
  survive a Core restart, so a run that was cut in half leaves a readable trail
  — and `GET /api/settings/os-updates/rollout` reconstructs the state from
  them. It reports which hosts finished, which failed and with what error, and
  whether the run ever closed. It does not report a percentage inside a host:
  apt does not publish partial progress and Moov does not invent it.

- **In-place updates, signed end to end** — two independent channels. Moov
  binaries (Core, console, helper, appliance console) ship as an Ed25519-signed
  bundle; the appliance verifies the signature and the SHA-256 of every declared
  file before installing a byte, and refuses any file the manifest does not
  declare. Downgrades and unsupported version jumps are rejected. If the new
  Core cannot start, the appliance rolls itself back on a timer armed before the
  restart — validated by installing a deliberately broken update.
- **OS updates for the whole fleet** — the appliance mirrors the Debian archive
  for its helpers, so a helper is patched without ever reaching the internet.
  Each package is downloaded once and served on. Debian's unattended security
  updates stay **on** for the appliance (with the switch and the last-run time in
  the console) and **off** on helpers, where they were dormant for lack of
  internet rather than protecting anything.
- **Package cache management** — emptied on demand from the console, and it
  releases space on its own when the appliance's disk gets tight, not only when
  the cache reaches its own ceiling.
- **Outbound HTTP/HTTPS proxy** configurable from the console, for sites without
  direct internet access. Everything also works air-gapped by uploading the
  bundle by hand.
- **Replacing a healthy helper** — the console can retire a working helper: it
  powers off and destroys the VM and reports what actually happened. The guard
  against deleting a busy helper stays on by default.
- **Preflight** — repository selection and pagination for large inventories.
- **Search domain (DNS suffix)** configurable from the first-boot TUI and the
  console.
- **Editable per-NIC addressing** — a migrated VM's IP can be changed from the
  wizard instead of only inherited.
- **Alerts when Veeam or a destination stops responding.** Until now only a
  fallen helper raised one, so a dead backend was discovered when a migration
  was launched. The health probe that already ran every 60 seconds now feeds
  notifications, alerting after two consecutive failures and deduplicating for
  30 minutes.
- **Backup warns that it is not enough on its own.** Restoring onto a *new*
  appliance does not bring back Veeam or hypervisor credentials without the
  recovery code — the console now says so where the backup is configured.

### Changed

- **Cold migration no longer stages the whole VM.** The helper used to write a
  full local copy of every disk before sending it; it now serves the source disk
  directly and the hypervisor pulls from it. That removes an entire copy of the
  VM from both the elapsed time and the helper's disk requirement — the helper no
  longer needs room for the largest VM you intend to migrate. Cold migrations
  now leave the VM **powered off** for your own maintenance window.
- **Guest networking binds by PCI topology, not by MAC.** Moov no longer assigns
  MAC addresses; each NIC configuration binds to its slot's PCI location, which
  is what the guest actually enumerates.
- **Progress reports transferred bytes.** The bar measured how much of the disk
  had been *traversed*, not how much had been *sent*, so a mostly-empty 500 GB
  disk looked like 500 GB of work. All three hypervisors now report real
  transferred bytes, measured before the copy starts.
- **Block storage is supported on Proxmox** for both Instant and Cold (LVM,
  iSCSI, ZFS, Ceph RBD). Moov asks the storage which formats it accepts instead
  of using a hard-coded list.
- **Dates follow the appliance's configured timezone** everywhere, instead of
  raw UTC in places.
- **Preflight scans only VM image backups** — agent, database-plugin and NAS
  backups no longer pad the list. Object-storage and hardened repositories are
  classified correctly instead of falling into the manual bucket.
- **A host whose update state could not be measured is never shown as "up to
  date"** — it says so, with the reason.
- **A pending reboot is read from the host itself**, not inferred from what is
  left to install, so a kernel installed by something else is still visible.
- **The console shows the running binary's version** alongside the image
  version, and says plainly when they differ.
- **The Veeam mount-server selector is Windows-only.** Linux mount servers
  cannot serve the iSCSI path Instant migration depends on.
- **Native `<select>` elements replaced** by an in-app dropdown, so the control
  looks the same on every OS.
- **One design of "Advanced options" for all five destinations.** The helper
  wizard showed three different forms depending on the backend. Placement is now
  required on Proxmox too, as it already was on oVirt and HVM: inheriting it
  silently from the destination was the source of Moov picking the cluster's
  first node and offering storages that did not exist on it.
- **The migration wizard names what is missing** instead of leaving a grey
  button and a "Ready to launch" that was not true.
- **Production logs are English.** 96 lines were still in Spanish — including
  the first line printed at start-up and the body of the API's own 404. Logs are
  read in English, the UI is translated; a CI gate now fails the build if a
  Spanish log line comes back.
- **Connection alerts have a switch.** The "Veeam or destination unreachable"
  notice added in this release could not be turned off from Settings ›
  Notifications, unlike every other alert.
- **Errors carry their detail in a field of its own.** Thirty-four stable codes
  could not be translated: their message *was* the information — the reason the
  hypervisor gave, which node, which item, how many seconds to wait — and the
  console prefers the translated key over the message, so translating them would
  have hidden exactly that. The headline is a constant per code and is now
  translated; the detail is runtime data and travels separately. An operator who
  chose English or Portuguese no longer reads Spanish in the errors they see
  most. The `message` field is unchanged for anything consuming the API without
  the console.

### Fixed

- **Proxmox and oVirt destinations were never actually probed.** The health
  monitor called `detect_version()`, which memoises: after the first success it
  returned the cached product version without touching the network, so those
  engines reported reachable in 0 ms forever. With an engine's NIC pulled, the
  monitor logged nothing for eight minutes and the "connection unreachable"
  alert never fired — the alert added in this release covered Veeam and
  Morpheus only, and nobody could tell. Health now uses an uncached `probe()`;
  `detect_version()` keeps its cache for feature-gating, which is what it is
  for. The failure mode only appeared after a successful start, which is why a
  short test never caught it.

- **No checkbox in the console showed as ticked.** With `appearance: none` the
  browser draws nothing, and two page-level rules set the control's background
  with the same specificity as the global `:checked` rule, winning on load
  order. Toggles worked and filtered correctly; only the drawing was dead, so
  every checkbox in the console read as inert. Fixed at both sources, and the
  global rule was hardened so a stray `<class> input {}` cannot silently take
  it over again.

- **Failed sign-in attempts were labelled "system".** Ticking "hide system
  events" left rows reading "system" on screen, so the filter looked broken. A
  failed sign-in has no user because the actor is *unknown* — which is exactly
  why an auditor looks for it, and why it is exempt from that filter. It is now
  labelled as unknown.

- **Installing OS updates gave no sign of life.** No spinner, and the restart
  button stayed live so both could be started at once. Both buttons now show
  progress and lock each other out. After a restart the console polls instead
  of measuring once: the endpoint answers when the restart is *scheduled*, and
  at that moment the host is still up on its old kernel, so a single immediate
  reading returned the state from before and left the operator pressing "Check
  now" by hand.

- **The install plan asked for confirmation twice.** The plan already states
  which hosts, in what order, how much will be downloaded and that userspace
  packages cannot be rolled back. The modal repeated that with less
  information, and a confirmation that restates the previous one trains people
  to click through both.

- **Migrating onto a name already taken overwrote the existing VM.** The
  collision guard only looked at *running* VMs, so a powered-off VM with the
  same name — the one nobody is watching — was left unprotected: its image was
  overwritten and it came back with a corrupt disk and no boot, while the
  migration reported success. The destination image is now checked before it is
  created, on both RBD and directory datastores, and a failed SSH check is
  treated as "the name may be taken" instead of "the name is free".
- **No appliance in the field could take its first update, and the console said
  it was up to date.** `/etc/moov/version` stores `v1.0.38`, and the update
  channel had its own version parser that read that as `0.0.38` — so every
  announced version looked older and the appliance concluded there was nothing
  to install. There is now a single parser. An appliance *older* than the
  channel's `min_from_version` is told so by name, rather than shown the same
  "up to date" as an appliance that really is current.
- **Legacy BIOS/MBR Windows VMs** were classified as data disks and skipped the
  whole guest preparation, so they did not boot.
- **The item never moved to `running`** — the wave ran, the VM had already
  booted, and the console showed `0/2 · 0%`.
- **A cancelled wave showed 100 % in green**, and the ETA said "~8s" with half
  the migration to go: both extrapolated from *finished* items, failures
  included, instead of items that actually completed.
- **An invalid target node was discovered 4–7 minutes in**, after the Veeam
  publish and the whole guest preparation had been spent. It is validated before
  the wave starts.
- **HVM: no Morpheus backend with SSH pins configured could migrate** — the
  pre-check looked up the fingerprint of the `NodeId` (`"1"`) rather than of a
  host. A pinning failure on HVM or oVirt also reported itself as a Proxmox
  error, sending the operator to the wrong hypervisor.
- **HVM: `ovsPortGroup` networks were offered** by the console and then rejected
  by libvirt, and service-plan reuse ignored inactive plans, hit the code
  uniqueness constraint and reported "no id in response".
- **Deleting a destination orphaned its credential**: adding it back failed with
  a uniqueness violation and an opaque 500.
- **Windows guests: the QEMU guest agent answered before PnP had finished** and
  the Morpheus agent installer fired only once; the wait ceiling was also
  unreachable on Windows (300 s for something that takes longer).
- **Proxmox: the old EFI disk was left as "Unused Disk 0"** after the pivot, and
  the log announced the swap before it happened.
- **The audit log's Action filter** offered only the actions present on the
  loaded page, so an action outside the last 50 rows could not be selected. The
  options now come from the whole table — and the table itself now refreshes
  when a filter changes, which it did not: the query options were not reactive,
  so the first fix looked like it had not worked.
- **The delete-destination dialog promised to keep the credential** for reuse
  while the handler deleted it — the same orphaning that made re-adding a
  destination fail with an opaque 500. The dialog now describes what happens.
- **"Roll back" did nothing** when the update had already been applied, and the
  channel card had no spinner and blamed the network for every failure.
- **Scheduled backups never ran** on any appliance restarted more often than its
  own interval. The countdown started when the service started, so every restart
  — an update, a new kernel, a power cut — reset it; with the default 48-hour
  interval an appliance restarted every couple of days never reached the
  deadline. It failed silently: the console said "Enabled" and showed a date for
  the next one. The schedule is now measured from the last backup on disk.
- **Windows VMs lost their static IP** on every migration, Hot and Cold, and
  came back on DHCP.
- **Instant migrations to Proxmox failed** partway through; **every Instant wave
  to oVirt failed** at the pre-flight SSH check.
- **Multi-disk Windows VMs booted from the wrong disk** after a Cold migration.
- **A dropped Veeam connection never recovered by itself** — the token refresh
  returned 401 until somebody re-saved the connection by hand.
- **Migrations went to the wrong hypervisor** when two backends of the same kind
  were registered: the destination resolved by kind instead of by id.
- **`rhsrvany.exe` was left behind in `C:\Windows`** on migrated Windows guests,
  along with the first-boot helper files. Both clean themselves now, and give up
  after three attempts rather than retrying forever.
- **A wave where every item was skipped** no longer reports as a success.
- **Helper deployment** — node and storage selection in the wizard, cidata ISO
  landing on the chosen storage, `ipconfig0` validated before it reaches the
  hypervisor instead of silently degrading to DHCP, and a failed deployment no
  longer leaves a `pending` ghost row forever.
- **oVirt VM names with spaces** are sanitised per backend before the mirror.
- **Cancelling a job** now clears it on the helper side too; `active_jobs` no
  longer sticks at 1.

## [1.0.37] — 2026-07-20

Post-production hardening: audit remediation (security and quality), live UI
improvements, the helper console, Windows multi-NIC, and the bugs found in the
three-hypervisor E2E.

### Added

- **Helper console TUI** (`moov-helper-info`) on tty1: service state, mTLS
  registration, heartbeat and active jobs, in EN/ES/PT.
- **Mount-server limit "Auto from repository tasks"** — the fixed ceiling of 12
  is gone; the per-connection limit derives from the `maxTaskCount` of the
  repositories involved (manual wins, operational ceiling 64).
- **Multi-helper placement** — Auto prefers helpers on the destination
  hypervisor, falling back to the whole fleet.
- **Deploy power-cycle** — a helper VM that does not register within ~5 minutes
  is restarted automatically (PVE / oVirt / Morpheus).
- **Windows multi-NIC static IP** — netsh per NIC, matched by MAC. Previously
  only the first NIC was configured.

### Changed

- Preflight subscribes to SSE on mount and refreshes even when another session
  launched the scan.
- Recovery detail: cancel survives a reload (`cancel_requested` is server-side),
  a rerun reflects the new wave without reloading, and the running item shows
  live elapsed time.
- The dashboard's running card shows real disk percentage instead of
  VMs-done/total.

### Fixed

- **Cold → Proxmox**: `import-from` with an absolute path failed with a non-root
  token; it now uses `pvesm alloc` plus a volid, like the Instant path.
- **oVirt**: the start retry no longer deletes a VM that is running.
- Rerun no longer requires a reload (Svelte `state_unsafe_mutation`).
- Deleting a preflight scan clears its cache and the table says which scan the
  inventory belongs to.

### Security

- Anti-injection validation in the Windows netsh path (CWE-78) and escaping plus
  validation of the libvirt domain XML (CWE-91).
- Cap and GC on the login-attempt map (memory DoS, CWE-400).
- The in-guest Morpheus agent install honours the connection's TLS flag
  (CWE-295).
- systemd hardening on every unit (CWE-250).
- SHA verification of every artefact downloaded during builds (CWE-494).
- `bcrypt` 0.19.2 (RUSTSEC-2026-0199); `cargo audit` and `cargo deny` at zero.

[Unreleased]: https://github.com/VeeamHub/moov/compare/v1.0.38...main
[1.0.38]: https://github.com/VeeamHub/moov/releases/tag/v1.0.38
[1.0.37]: https://github.com/VeeamHub/moov/releases
