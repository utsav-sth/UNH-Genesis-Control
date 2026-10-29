# Genesis Panopticon — Ubuntu Integration Handoff

Date: October 26, 2026
Status: GUI and basic animated offline preview validated.
Physical acceptance: PENDING.

## 1. Repository and environment

Host: Ubuntu/WSL, DESKTOP-M34HHNQ
Account: utsav
Worktree:
  /home/utsav/UNH-Genesis-Control/worktrees/visualizer-integration

Branch:
  feature/visualizer-integration

Python environment:
  conda activate genesis-p1

Phoebus:
  /usr/local/bin/phoebus
  /opt/phoebus-4.7.4-SNAPSHOT/product-4.7.4-SNAPSHOT.jar

Remote graphical access:
  ssh -Y

Main display:
  gui/genesis_panopticon.bob

Generator:
  tools/make_panopticon.py

The tracked display is generated. Edit the generator rather than
manually editing the generated Panopticon.

## 2. Validation completed

- Phoebus 4.7.4 successfully rendered Panopticon on Ubuntu.
- All nine linked subsystem displays were opened successfully.
- All 13 main Panopticon actions are open_display actions.
- Linked display XML validation: 9/9 passed.
- The Panopticon is intended as a read-only overview.
- An isolated temporary preview with synthetic Phoebus PVs
  successfully displayed animated values and graphs.
- No Genesis IOC or physical device was launched for these checks.
- The feature worktree remained clean throughout validation.

Previous regression baseline:
- 22 focused tests passed; 10 skipped.
- Broader suite: 413 passed, one known outdated Microwave
  assertion, 10 skipped.
- These historical counts were not rerun during the GUI review.

Not validated:
- Full realistic multi-subsystem replay session.
- Actual physical IOC connectivity.
- Physical device control and safety acceptance.
- End-to-end DAQ startup on Ubuntu from this branch.

## 3. Existing startup components

Full-stack orchestration reference:
  scripts/live_laptop/start.sh

This script is oriented toward Prof. Long's live laptop setup.
It includes physical IOC startups and Windows/PyNMR integration.
DO NOT run it unchanged on Ubuntu without reviewing and adapting
its assumptions.

Existing GUI-only launcher:
  scripts/start_phoebus_instance.sh

Live Phoebus settings:
  configs/phoebus/panopticon_live.ini

Physical subsystem references:
  scripts/cryo_physical.sh
  configs/target_physical/
  configs/magnet_physical/
  configs/phoebus/microwave_physical.ini
  configs/phoebus/ssrf_physical.ini

The physical settings include explicit localhost EPICS ports.
They are NOT offline simulation settings.

## 4. When laboratory access is authorized

Prerequisites:

1. Confirm with lab operators that hardware is available.
2. Select exactly one DAQ hardware owner/account.
3. Confirm required physical safety interlocks and device routing.
4. Review scripts/live_laptop/start.sh for the exact startup
   sequence, required profiles, and Windows-specific assumptions.
5. Confirm the Ubuntu equivalents for every startup operation.
6. Check for running IOCs, existing device ownership, and port
   conflicts before launching anything.
7. Preserve main and unrelated feature worktrees.

Planned physical startup sequence:

A. Start the authorized Cryo physical IOCs and instrument bridges.
B. Start the Magnet control/readback stack.
C. Start the Microwave physical stack and its supervisor.
D. Start ssRF hardware services, interlocks, and controllers.
E. Start the authorized PyNMR source and online fitter.
F. Start trend-history and master archiving services.
G. Check IOC health, required PVs, alarms, and ownership.
H. Launch Panopticon with live Phoebus settings.
I. Validate physical readbacks, plots, navigation, archiving,
   failure behavior, and safe operator shutdown.

IMPORTANT: This is the required service order/checklist,
not a previously validated one-command Ubuntu startup.
Individual subsystem startup commands must be confirmed
against the then-current scripts before execution.

## 5. Launch Panopticon after the physical stack is ready

Run from the visualizer-integration worktree:

  conda activate genesis-p1

  PHOEBUS_BIN=/usr/local/bin/phoebus \
    bash scripts/start_phoebus_instance.sh \
    Panopticon \
    configs/phoebus/panopticon_live.ini \
    gui/genesis_panopticon.bob \
    /tmp/genesis_panopticon_physical.log \
    49380

Check that the selected Phoebus server port is unused first.

WARNING: This command opens live Channel Access connections
according to panopticon_live.ini. Execute ONLY after explicit
lab authorization and successful physical IOC startup.

The GUI-only launcher does not start the subsystem IOCs.

## 6. Optional future Ubuntu startup script

If the live_laptop startup cannot be used directly on Ubuntu,
create a separate Ubuntu-specific physical startup wrapper.

Requirements:
- Explicit physical-mode operator confirmation.
- Exclusive hardware ownership checks.
- Defined physical profile paths and service startup order.
- Port and process collision checks.
- Interlock and connectivity preflight.
- Clear logs and status reporting.
- Fail-closed behavior and safe service shutdown.
- No automatic switching or merging of Git branches.

Do not implement or execute this wrapper while lab devices
are occupied.

## 7. Release criteria

Before merging the feature into main:

- Reconfirm automated regression results.
- Resolve or document the outstanding Microwave assertion.
- Complete full physical acceptance with authorized access.
- Verify physical safety, all subsystem readbacks, and archiving.
- Preserve the exact accepted commit SHA and timestamped tag.
- Merge only after physical acceptance.

No physical acceptance was claimed on October 26, 2026.

## 8. October 29, 2026 — Live integration validation

Status: Partial physical integration validated.
Full physical subsystem acceptance remains pending.

The October 26 offline validation record above is retained as
historical context. The following results supersede its statements
about physical integration not having been tested.

### Validated during live operation

- Panopticon launched successfully on Ubuntu/WSL using live EPICS PVs.
- Cryo physical stack: 19 IOCs started and monitored.
- Microwave physical/control stack: monitored with RF OFF and robot disabled.
- ssRF physical stack: SMC100A, switches, Keysight WaveGen, sequencer,
  interlocks, online fitter, and archiver operated together.
- PyNMR acquisition events were accepted by the protected fitter;
  live fitter results and plots appeared in the Genesis displays.
- Trend History IOC restored historical plots in Panopticon.
- Streamlit online archive viewer successfully displayed the archive.
- Unified Master Archiver recorded Cryo, Microwave, and ssRF/NMR PVs.
- The Panopticon CA configuration was updated to include
  192.168.1.24:5064 for the Raspberry Pi Keysight IOC.

### Unified Master Archiver results

- Configured PVs: 448
  - Cryo: 168
  - Microwave: 124
  - ssRF/NMR fitter: 97
  - Magnet supply: 59
- Snapshot interval: 1 second.
- Last observed running status: 2403 snapshots, 119556 events written,
  zero dropped events, and zero snapshot overruns.
- Session data preserved: approximately 93 MB.
- Session directory:
  /home/utsav/UNH-Genesis-Control/local_data/archive/master/
  20261029T164717Z_193a0400b9f3_21169
- Magnet supply PVs were registered but reported missing, as expected
  with the magnet supply IOC offline.

### Known limitations

- Magnet power supply was not connected or physically commissioned.
- Four Microwave Siglent scope PVs were unavailable because the
  new scope was not connected for this integration test.
- Low manometer USB serial connection was unavailable.
- Roots Right flowmeter communication recovered after USB reconnection.
- No valid target polarization acceptance was performed because
  the physical target signal was not available.
- Some archived PVs were stale or unavailable; registering a PV
  does not establish that its measurement was fresh or valid.
- Cryo safety faults and disconnected instruments require separate
  operational follow-up; this test did not certify all interlocks.

### Controlled shutdown validation

- ssRF safe stop completed; SMC100A RF/FM OFF and Keysight WaveGen OFF
  were independently verified.
- PyNMR exited without a remaining acquisition process.
- Microwave safe-state verification passed before its IOCs stopped.
- Cryo coldboard output was confirmed OFF before all 19 IOCs stopped.
- Trend History IOC stopped.
- Master Archiver stopped and the session files remained preserved.
- Panopticon was closed last.
- Final inspection found no remaining Genesis screen sessions or
  Phoebus processes, and the Master Archiver reported DOWN.

### Release assessment

Live monitoring, integrated archiving, and controlled software
shutdown passed for the tested configuration.

This is not acceptance of the disconnected Magnet, unavailable
instruments, or untested physical safety and polarization workflows.
Full-system acceptance requires separate validation.

All work remained on feature/visualizer-integration.
No merge or modification of Git main was performed.
