# IOS XR Upgrade Automation

Ansible automation for Cisco ASR9K upgrades (IOS XR `7.8.2.x` → `24.4.2.1`) using the `install add / prepare / activate / commit` workflow, with pre/post verification, node isolation, SMU handling, and FPD management. Designed to run from AWX / AAP.

---

## What this does

- Upgrades ASR9Ks end-to-end across 13 orchestrated phases
- Supports **serial** (one at a time) or **parallel** (multiple devices at once) via survey
- Drains the node before any package work; restores it before POST verification
- Applies SMUs before the version upgrade (committed) and after (bundled with the version commit)
- Handles FPD upgrades, including field-notice-driven sequenced orders (FN74306 on 4HG-FLEX)
- Captures pre/post state snapshots per device for post-hoc diffing
- Refuses to commit if any assertion fails — everything stays rollback-capable until the final `install commit`

## What this doesn't do

- Transfer files to the router — image + SMUs must already be staged on `harddisk:`
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
| `provision_tower.yml` | IaC playbook — creates Project, Inventory, Credential, Job Template in AWX/AAP via `awx.awx` |
| `collections/requirements.yml` | Ansible collection dependencies (`cisco.iosxr`, `ansible.netcommon`, `awx.awx`) |
| `awx_survey.json` | 15-field survey form for AWX Job Template |
| `README.md` | This file |

---

## AWX / AAP setup

### Option A — Automated (recommended)

`provision_tower.yml` creates everything in one run: Project, Inventory (with hosts + group), Credential, Job Template with survey.

```bash
# Prerequisites
pip install awxkit
ansible-galaxy collection install -r collections/requirements.yml

# Connection to AWX/AAP
export CONTROLLER_HOST=https://awx.example.com
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD='...'
export CONTROLLER_VERIFY_SSL=true

# Run — prompts for IOS XR admin user + password
ansible-playbook provision_tower.yml \
    -e tower_project_scm_url=https://github.com/YOUR_ORG/ANSB_IOS.git
```

Edit `provision_tower.yml` vars to adjust: organization, inventory hosts, credential name, project branch.

### Option B — Manual (Tower UI)

1. Install collections:
   ```
   ansible-galaxy collection install -r collections/requirements.yml
   ```
2. Push repo to git.
3. **Projects → Add** → point at the git repo. Sync.
4. **Credentials → Add** → type `Machine` → IOS XR admin user + password.
5. **Inventories → Add** → add hosts, create `iosxr_devices` group with connection vars.
6. **Templates → Add → Job Template**:
   - **Playbook**: `upgrade_iosxr.yml`
   - **Forks**: `1`
   - **Limit**: enable *Prompt on launch*
7. **Survey** tab → *Import* → paste `awx_survey.json`. Toggle **Enabled**.
8. Attach notifications (Slack / email) for *Started / Success / Failure*.

### Smoke-test before production

Launch the Job Template against **one lab device** with `--tags pre`. This runs PRE-checks only — reads baseline state, verifies parsers, touches nothing. Expected output: `EVPN BGP peers Established (N): [...]` with sensible values.

---

## Running an upgrade

### Manual launch

**Templates → `IOS XR Upgrade` → Launch**

1. Fill **Limit**: `pe1.lab` (or multiple hosts for parallel)
2. Fill the survey:
   - **Parallel upgrades**: `1` for serial, `2`-`10` for parallel
   - **Target version**: `24.4.2.1`
   - **Image filename** + **MD5**
   - **SMUs BEFORE** (one line per SMU: `filename.rpm: md5hash`)
   - **SMUs AFTER** (same format)
   - Phase toggles, FPD sequencing, timeouts
3. **Launch**

### Parallel upgrades

Set **Parallel upgrades** in the survey to control how many devices run at once within a single job:

| Setting | Behavior |
|---|---|
| `1` (default) | Serial — one device completes before the next starts. Safest. |
| `2`-`10` | That many devices upgrade simultaneously. Each gets its own baseline, isolation, and verification. |

`any_errors_fatal: true` is set — if any device in a parallel batch fails, the rest of that batch stops.

### Scheduled launch

**Templates → `IOS XR Upgrade` → Schedules → Add**. Each schedule carries its own Limit and survey answers.

| Schedule | Start | Limit | Parallel |
|---|---|---|---|
| `pe1-upgrade-2026-05` | Tue 02:00 UTC | `pe1.lab` | 1 |
| `wave2-upgrade-2026-05` | Wed 02:00 UTC | `pe2.lab,pe3.lab` | 2 |

---

## The 13 phases

| # | Phase | Tag | What happens |
|--:|---|---|---|
| 1 | INIT | `always` | Normalizes survey inputs into native types |
| 2 | PRE-CHECKS | `pre` | Baseline snapshot, NTP sync, alarms, platform health, pending install ops, BGP/IS-IS neighbor capture |
| 3 | DISK-PREP | `disk_prep` | Removes core files + inactive packages from XR and admin planes |
| 4 | FPD-PREP | `fpd_prep` | Disables `fpd auto-upgrade` / `auto-reload` in both planes |
| 5 | ISOLATE | `isolate` | `shutdown Loopback1` + `set-overload-bit` on IS-IS; waits for BGP drain |
| 6 | SMU-BEFORE | `smu_before` | MD5 verify → add → prepare → activate → wait (reload if needed) → commit |
| 7 | INSTALL | `install` | MD5 verify image → `install add/prepare/activate synchronous` + reload wait |
| 8 | FPD | `fpd` | Sequenced FPD upgrades (one at a time, poll-until-CURRENT) → bulk `upgrade hw-module all fpd all` → admin reload |
| 9 | SMU-AFTER | `smu_after` | MD5 verify → add → prepare → activate (not committed) |
| 10 | UN-ISOLATE | `unisolate` | Clear overload → no-shut Lo1 → wait for **all** baseline neighbors to return |
| 11 | POST-CHECKS | `post` | Assert target version, neighbor membership matches baseline, NTP, alarms |
| 12 | SOAK | `post` | `commit_delay` seconds — operator window to observe/abort |
| 13 | COMMIT | `commit` | `install commit` — version + SMU-after committed atomically |

Everything in 6–10 is executed with the node isolated. Step 13 is the point of no return.

---

## Survey fields (15)

| # | Field | Type | Default |
|--:|---|---|---|
| 1 | Parallel upgrades | dropdown | `1` |
| 2 | Target IOS XR version | text | `24.4.2.1` |
| 3 | Image path on device | dropdown | `harddisk:` |
| 4 | Base image filename | text | `asr9k-x64-24.4.2.1.tar` |
| 5 | Image file MD5 | text | *(required)* |
| 6 | SMUs BEFORE (`filename: md5`, one per line) | textarea | *(empty)* |
| 7 | SMUs AFTER (`filename: md5`, one per line) | textarea | *(empty)* |
| 8 | Run DISK-PREP phase | dropdown | `true` |
| 9 | Run FPD-PREP phase | dropdown | `true` |
| 10 | Run FPD upgrade phase | dropdown | `true` |
| 11 | FPD admin reload after upgrade | dropdown | `true` |
| 12 | Sequenced FPD upgrades (YAML list) | textarea | `[]` |
| 13 | Reload timeout (seconds) | integer | `2700` |
| 14 | Commit soak delay (seconds) | integer | `900` |
| 15 | Backup directory | text | `./backups` |

SMU fields use a combined format — each line is `filename.rpm: md5hash`. The playbook splits this into the filename list and MD5 map automatically. Legacy CLI format (`-e smus_before=[...] -e smus_before_md5={...}`) is still supported.

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

If anything fails between SMU-BEFORE and POST-CHECKS, the upgrade can be reverted:

```
admin install rollback to committed
```

The device reloads to the previously-committed image. After the rollback reload, manually un-isolate:

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

Once committed, rollback means installing the previous version as a fresh upgrade:

```
install rollback to <old-label>     # e.g. 7.8.2
install commit
```

Same maintenance-window rules apply — isolate first, verify neighbors return.

---

## Variables worth knowing (group_vars/iosxr_devices.yml)

| Variable | Default | Purpose |
|---|---|---|
| `target_version` | `24.4.2.1` | Must match `show install active summary` Label exactly |
| `image_file` | `asr9k-x64-24.4.2.1.tar` | File staged on `harddisk:` |
| `image_file_md5` | `<mandatory>` | Expected MD5, verified on device |
| `smus_before` / `smus_after` | *(set)* | SMU filename lists |
| `smus_before_md5` / `smus_after_md5` | *(set)* | Filename → expected MD5 |
| `fpd_sequenced_upgrades` | FN74306 list | `[]` for routers without 4HG-FLEX |
| `upgrade_serial` | `1` | Devices upgraded at once (survey-controlled) |
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
  - Fixed SMU file-check `failed_when` to reference current loop iteration
  - Fixed FPD sequenced upgrades to upgrade+poll one FPD at a time via `fpd_sequenced_single.yml`
  - Fixed UN-ISOLATE neighbor polls: `interval` → `delay` (was polling at 5s instead of intended 15s)
  - Added `show install request` capture in PRE to detect pending install operations
  - Added `collections/requirements.yml` for Tower EE auto-install
  - Moved `iosxr_devices.yml` to `group_vars/` for correct Ansible auto-loading
  - Aligned all defaults across playbook vars, group_vars, and survey
  - Fixed SUMMARY task tag from `[post]` to `[commit]`
- **v1.6** — Survey improvements and parallel upgrades:
  - Merged SMU filename + MD5 into single survey field per phase (combined `filename: md5` format)
  - Added `upgrade_serial` survey field for parallel upgrades (1/2/3/5/10 devices at once)
  - Reordered survey: parallel setting at top, each item grouped with its MD5
  - Reduced survey from 16 to 15 fields
  - Added `provision_tower.yml` for automated AWX/AAP setup (single Job Template mode)
  - Added `awx.awx` to `collections/requirements.yml`

Track subsequent changes in git history.
