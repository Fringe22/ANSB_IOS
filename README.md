# IOS XR Upgrade Automation

Ansible automation for Cisco ASR9K upgrades (IOS XR `7.8.2.x` → `24.4.2.1`) using the `install add / prepare / activate / commit` workflow, with pre/post verification, node isolation, SMU handling, and FPD management. Designed to run from Ansible Tower / AAP.

---

## What this does

- Upgrades **one ASR9K at a time**, end-to-end, across 13 orchestrated phases
- Drains the node before any package work; restores it before POST verification
- Applies SMUs before the version upgrade (committed) and after (bundled with the version commit)
- Handles FPD upgrades, including field-notice-driven sequenced orders (FN74306 on 4HG-FLEX)
- Captures pre/post state snapshots per device for post-hoc diffing
- Refuses to commit if any assertion fails — everything stays rollback-capable until the final `install commit`

## What this doesn't do

- Transfer files to the router — image + SMUs must already be staged on `harddisk:`
- Coordinate multi-device rollouts — Tower Schedules handle that
- Run node-specific change work (slice config, interface shutdowns, VRF additions) — those live in a separate pre/post playbook
- Replace your change ticket / MOP approval workflow

---

## Files

| File | Purpose |
|---|---|
| `upgrade_iosxr.yml` | Main playbook — 13 phases, full upgrade orchestration |
| `smu_phase.yml` | Reusable SMU add/activate/verify block (included twice) |
| `fpd_sequenced_single.yml` | Per-FPD upgrade + poll-until-CURRENT (included in a loop for sequenced FPD ordering) |
| `inventory.yml` | Router list + management IPs + connection vars |
| `group_vars/iosxr_devices.yml` | Target version, image, SMU list, MD5s, FPD config — **fleet-wide** |
| `provision_tower.yml` | IaC playbook — creates Project, Inventory, Credential, Job Template(s) in Tower via `awx.awx` |
| `collections/requirements.yml` | Ansible collection dependencies (`cisco.iosxr`, `ansible.netcommon`) — used by Tower EE |
| `awx_survey.json` | 16-field survey form for Tower Job Template |
| `README.md` | This file |

---

## Ansible Tower / AAP setup (first time only)

1. Install required collections (Tower/AAP does this automatically if `collections/requirements.yml` is present in the project):
   ```
   ansible-galaxy collection install -r collections/requirements.yml
   ```
2. Push the repo (all files above) to git.
3. **Tower → Projects → Add** → point at the git repo. Sync.
4. **Tower → Credentials → Add** → type `Machine` → fill in the IOS XR admin user + password. Name it e.g. `iosxr-admin`.
5. **Tower → Inventories → Add** → paste `inventory.yml`, or better: add it as a source from the same git project.
6. **Tower → Templates → Add → Job Template**:
   - **Project**: the one from step 2
   - **Playbook**: `upgrade_iosxr.yml`
   - **Inventory**: from step 4
   - **Credentials**: from step 3
   - **Forks**: `1` (defensive; `serial: 1` in the playbook is the real guard)
   - **Limit**: leave empty, enable *Prompt on launch*
   - **Verbosity**: `1`
7. Open the Job Template → **Survey** tab → *Import* → paste `awx_survey.json`. Toggle **Enabled**.
8. Attach any notifications (Slack / email) for *Started / Success / Failure*.

### Smoke-test before production

Launch the Job Template manually against **one lab device** with `--tags pre` (add as Extra Variables: `ansible_tags: ['pre']`). This runs PRE-checks only — reads baseline state, verifies parsers, touches nothing. Expected output: the debug line `EVPN BGP peers Established (N): [...]` with sensible values. If that works, you're ready for production.

---

## Running an upgrade

### Manual launch (one router, specific window)

**Tower → Templates → `IOS XR Upgrade` → Launch**

1. Fill the **Limit** prompt: `pe1.lab`
2. Fill the survey fields (most come pre-filled from defaults):
   - Target IOS XR version: `24.4.2.1`
   - Base image filename: `asr9k-x64-24.4.2.1.tar`
   - Expected MD5 of image file: *(md5sum of the staged .tar — generate with `md5sum` on the file server)*
   - SMUs BEFORE: paste the three pre-upgrade RPMs, one per line
   - SMUs BEFORE MD5s: paste the filename → MD5 YAML map
   - SMUs AFTER: the post-upgrade SMU
   - SMUs AFTER MD5s: filename → MD5 YAML map
   - Sequenced FPD upgrades: FN74306 list (or `[]` for routers without 4HG-FLEX)
3. Review → **Launch**

Watch the job output in Tower. Total runtime is typically **3–5 hours per device** (see breakdown below).

### Scheduled launch (automatic, per maintenance window)

On the same Job Template: **Schedules** tab → **Add**. Create one schedule per router/window. Each schedule carries its own Limit value and its own survey answers.

Example:
| Schedule | Start | Repeats | Limit |
|---|---|---|---|
| `pe1-upgrade-2026-05` | Tue 02:00 UTC | once | `pe1.lab` |
| `pe2-upgrade-2026-05` | Wed 02:00 UTC | once | `pe2.lab` |
| `pe3-upgrade-2026-05` | Thu 02:00 UTC | once | `pe3.lab` |

---

## The 13 phases

| # | Phase | Tag | What happens |
|--:|---|---|---|
| 1 | INIT | `always` | Normalizes survey YAML-string inputs into real types |
| 2 | PRE-CHECKS | `pre` | Baseline snapshot, NTP sync, alarms, platform health, pending install ops, BGP/IS-IS neighbor capture |
| 3 | DISK-PREP | `disk_prep` | Removes core files + inactive packages from XR and admin planes |
| 4 | FPD-PREP | `fpd_prep` | Disables `fpd auto-upgrade` / `auto-reload` in both planes |
| 5 | ISOLATE | `isolate` | `shutdown Loopback1` + `set-overload-bit` on IS-IS; waits for BGP drain |
| 6 | SMU-BEFORE | `smu_before` | MD5 verify → add → prepare → activate → wait (reload if needed) → commit |
| 7 | INSTALL | `install` | MD5 verify image → `install add/prepare/activate synchronous` + reload wait |
| 8 | FPD | `fpd` | Sequenced FPD upgrades first (if any, one at a time with poll-until-CURRENT) → bulk `upgrade hw-module all fpd all` → admin reload |
| 9 | SMU-AFTER | `smu_after` | MD5 verify → add → prepare → activate (not committed) |
| 10 | UN-ISOLATE | `unisolate` | Clear overload → no-shut Lo1 → wait for **all** baseline neighbors to return |
| 11 | POST-CHECKS | `post` | Assert target version, neighbor membership matches baseline, NTP, alarms |
| 12 | SOAK | `post` | `commit_delay` seconds — operator window to observe/abort |
| 13 | COMMIT | `commit` | `install commit` — version + SMU-after committed atomically |

Everything in 6–10 is executed with the node isolated. Step 13 is the point of no return.

---

## Tags for partial runs

| Command | Behavior |
|---|---|
| `--tags pre` | Dry-run baseline check only |
| `--tags pre,disk_prep` | Safe pre-upgrade hygiene |
| `--tags isolate` | Drain only (manual maintenance) |
| `--tags unisolate` | Restore only (recovery) |
| `--skip-tags commit` | Everything up to and including POST, stops at soak — operator verifies then commits manually |
| `--skip-tags smu_before,smu_after` | Version upgrade only, no SMUs |

---

## Estimated runtime

| Phase | Typical wait |
|---|---:|
| PRE + DISK-PREP + FPD-PREP | ~10 min |
| ISOLATE (drain) | ~5 min |
| SMU-BEFORE (3 RPMs, reload) | ~25–40 min |
| INSTALL (TAR bundle, activate, reload) | ~40–60 min |
| FPD (sequenced + bulk + admin reload) | ~40–90 min |
| SMU-AFTER | ~15–25 min |
| UN-ISOLATE + neighbor reconvergence | ~10–15 min |
| POST + SOAK | ~25 min (incl. `commit_delay`) |
| COMMIT | ~2 min |
| **Total** | **~3–5.5 hours per device** |

---

## When it fails

**Core principle:** if the playbook errors out **before COMMIT**, the upgrade is not persisted. The device may be running on the new image (because `install activate` already ran) but the next reload without `install commit` will revert to the previously-committed state. So *almost all failures are recoverable with a single rollback*.

### Phase-by-phase troubleshooting

| Failing phase | Likely cause | What to do |
|---|---|---|
| **PRE** `platform not OPERATIONAL` | Line card already down, RSP unhealthy | Do **not** upgrade — investigate hardware first |
| **PRE** `NTP not synchronised` | NTP server unreachable | Fix NTP, re-run PRE |
| **PRE** `zero BGP peers at baseline` | Device already drained, or regex didn't match output | Inspect `show bgp l2vpn evpn summary` on device, check output format |
| **PRE** `pending install operation` | Previous install add/prepare/activate left uncommitted | Run `show install request` on device; `install commit` or `install prepare clean` to clear |
| **DISK-PREP** `run rm` fails | User lacks shell (`run`) auth | Either add auth via TACACS, or clean cores manually and re-run with `-e disk_prep_enabled=false` |
| **FPD-PREP** admin config fails | Admin prompt transition mismatch | Disable admin FPD auto-upgrade manually, re-run with `-e fpd_prep_enabled=false` |
| **ISOLATE** config commit hangs | Pending config session elsewhere | Check `show configuration commit list`, clear stale sessions |
| **INSTALL** `image_file_md5 is mandatory` | MD5 not supplied or not 32-char hex | Populate `image_file_md5` in group_vars or survey |
| **SMU-BEFORE/AFTER** `SMU MD5 verification is mandatory` | SMU listed without matching MD5 entry | Populate the MD5 map with one entry per SMU, or clear the SMU list |
| **SMU-BEFORE** `MD5 mismatch` | File transfer corrupted | Re-transfer the RPM, re-run |
| **SMU-BEFORE** `install add` fails | Disk space, or SMU incompatible with current version | Check `show install log`, free disk, verify SMU applies to running image |
| **INSTALL** `MD5 mismatch` on TAR | TAR transfer corrupted | Re-transfer, re-run |
| **INSTALL** `install activate` timeout | Device didn't reload in `reload_timeout` | Check console — device may be in ROMMON; follow manual recovery |
| **INSTALL** device doesn't come back | Serious — console to the device | Manual recovery via ROMMON if needed |
| **FPD** `NEED UPGD` assertion fails | One FPD didn't upgrade | Check `show hw-module fpd`; may need manual `upgrade hw-module location X/Y fpd Z force` + reload |
| **FPD** sequenced upgrade times out | FPD write is slow (Primary-BIOS can take >30 min) | Bump `fpd_upgrade_timeout` in group_vars, re-run FPD phase with `--tags fpd` |
| **SMU-AFTER** anything | Same failure modes as SMU-BEFORE | Rollback (see below) — the version upgrade has not been committed |
| **UN-ISOLATE** `neighbor missing` | BGP peer or IS-IS adjacency didn't return | Check `show bgp l2vpn evpn summary` and peer-side status. Common cause: peer policy changed, route-map rejected updates |
| **POST** version mismatch | `target_version` string doesn't match label on device | Verify with `show install active summary` on device, set `target_version` accordingly |
| **COMMIT** fails | Very rare — install subsystem broken | Open Cisco TAC case immediately |

### Rollback (before `install commit`)

If anything fails between SMU-BEFORE and POST-CHECKS, the upgrade can be reverted by rolling back to the last committed install state:

```
admin install rollback to committed
```

The device reloads and comes up on the previously-committed image (which is pre-upgrade, plus any SMU-before patches that were committed during their phase). After the rollback reload, you'll need to manually un-isolate:

```
configure
 router isis main
  no set-overload-bit
 !
 interface Loopback1
  no shutdown
 !
commit
end
```

Then verify `show bgp l2vpn evpn summary` and `show isis adjacency` show full convergence.

### Rollback (after `install commit`)

Once committed, rollback means installing the previous version as a fresh upgrade. The previous version is still in the repository (unless a later `install remove inactive all` was run), so:

```
install rollback to <old-label>     # e.g. 7.8.2
install commit
```

Same maintenance-window rules apply — isolate first, use the playbook if you can, verify neighbors return.

---

## Variables worth knowing (group_vars/iosxr_devices.yml)

| Variable | Default | Purpose |
|---|---|---|
| `target_version` | `24.4.2.1` | Must match `show install active summary` Label exactly |
| `image_file` | `asr9k-x64-24.4.2.1.tar` | File staged on `harddisk:` |
| `image_file_md5` | `<mandatory>` | Expected MD5, verified on device. Playbook refuses to run without it. |
| `smus_before` / `smus_after` | *(set)* | SMU filename lists |
| `smus_before_md5` / `smus_after_md5` | *(set)* | Filename → expected MD5. Mandatory when corresponding SMU list is non-empty. |
| `fpd_sequenced_upgrades` | FN74306 list | `[]` for routers without 4HG-FLEX |
| `reload_timeout` | `2700` (45 min) | SSH recovery window after each reload |
| `commit_delay` | `900` (15 min) | Soak before final commit |
| `disk_prep_enabled` | `true` | Core/inactive-package cleanup toggle |
| `fpd_*_enabled` | `true` | FPD phase toggles |

---

## Escalation

| Situation | Who to call |
|---|---|
| Upgrade fails pre-install (PRE, DISK-PREP, FPD-PREP) | On-call network engineer |
| Device unreachable after INSTALL reload | Console in, involve NOC |
| FPD upgrade fails with errors on specific line card | Cisco TAC + hardware replacement path |
| `install rollback` fails | Cisco TAC immediately |
| Multiple hosts failing same phase same way | Roll out paused; review common factor (file server, TACACS, NTP) |

---

## Change history

- **v1.0** (initial) — core install + isolate/unisolate + pre/post verification
- **v1.1** — SMU-before / SMU-after support with auto-detect reload type
- **v1.2** — EVPN-focused verification command set, strict neighbor membership check
- **v1.3** — DISK-PREP + FPD-PREP + FPD phases, MD5 verification, TAR bundle support, synchronous install
- **v1.4** — MD5 verification mandatory (image + every listed SMU); survey validation tightened
- **v1.5** — Bug fixes and hardening:
  - Fixed SMU file-check `failed_when` to reference current loop iteration (was using `results[-1]`)
  - Fixed FPD sequenced upgrades to upgrade+poll one FPD at a time via `fpd_sequenced_single.yml` (was firing all upgrades before polling, breaking FN74306 ordering)
  - Fixed UN-ISOLATE neighbor polls: `interval` → `delay` (task-level `until` loops ignore `interval`; was polling at 5s instead of intended 15s)
  - Added `show install request` capture in PRE to detect pending install operations
  - Added `collections/requirements.yml` for Tower EE auto-install of `cisco.iosxr` and `ansible.netcommon`
  - Moved `iosxr_devices.yml` to `group_vars/iosxr_devices.yml` for correct Ansible auto-loading
  - Aligned all defaults across playbook vars, group_vars, and survey (reload_timeout=2700, commit_delay=900)
  - Fixed SUMMARY task tag from `[post]` to `[commit]` (was printing "COMMITTED" before commit ran)
  - Updated MD5 comment from "optional" to "mandatory" (stale since v1.4)

Track subsequent changes in git history; this README should be updated when phases are added or tag names change.
