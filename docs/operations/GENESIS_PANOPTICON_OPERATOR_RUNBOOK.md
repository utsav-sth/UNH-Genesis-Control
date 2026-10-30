# GENESIS Phase-I — Panopticon Live Operator Runbook

**Current operator baseline (30 October 2026):** `main` was advanced to `4846a4d6bae0a99a4556709e4428ea38c506bb1c` (the same software as `feature/visualizer-integration`) and successfully operated from `main`. A subsequent **documentation-only** update advanced `main` to `fd67507` while `feature/visualizer-integration` remains at `4846a4d`. The tracked operating software was unchanged by that documentation commit.
**Repository:** `UNH-Genesis-Control`
**Working directory:** Any **explicitly approved Genesis worktree**. Recommended default: `/home/utsav/UNH-Genesis-Control/worktrees/main`. The runbook uses `$GENESIS_WT` instead of hard-coding a working tree.
**Historical physical integration basis:** 29 October 2026, source commit `fa3262e7a5c0117102beac216024e4e652606676`; immutable partial-validation tag `panopticon-integration-partial-validated-20261029T173322Z`. The subsequent `4846a4d` commit adds operator documentation; **the historical tag is not moved**.
**Operating host:** Ubuntu/WSL on `DESKTOP-M34HHNQ`, user `utsav`
**Conda environments:** `genesis-p1` for Genesis; `pynmr` for the separate PyNMR application
**Scope:** Live viewing, authorized subsystem startup, software health, archiving, troubleshooting, and controlled shutdown. October 30 confirms operation from `main` but does **not** independently certify Magnet ramp/heater control, RF burning, scientifically valid polarization, or all safety interlocks.

> [!CAUTION]
> **PHYSICAL SAFETY FIRST.** This is an operating aid for trained, authorized laboratory personnel, not permission to energize equipment. Do not operate an unattended cryogenic, high-field, high-power microwave or RF system without the responsible experiment operator and laboratory SOP. If the physical state cannot be positively verified, **STOP and retain monitoring/ownership**. Never bypass shutdown interlocks just to release a screen session or port. Do not switch Git worktrees/branches or make Git changes during a live hardware run; working in `main` is now permitted **for operation**, not unrestricted live editing.

## How to use this document

### Visually distinguished command types (applies to **every subsystem**)

| Marker in this runbook | Who uses it | Meaning |
|---|---|---|
| **🟩 OPERATOR / RUN, STOP, STATUS** | Authorized trained operator | Actual routine startup, shutdown, GUI launch or normal `./scripts/genesisctl.sh ...` status/readiness command |
| **🟦 EXPERT / DIAGNOSTIC** | Troubleshooter | Read-only process, log, port, socket, Git, network, EPICS or archive investigation. **Not an operational start/stop step.** |
| **🟧 SPECIALIST / CONTROLLED SERVICE** | Hardware/DAQ lead | A safety-critical IOC lifecycle, instrument-control step, device-clock action or a targeted forced stop requiring additional authorization. **The routine, manually managed Trend History IOC is 🟩 OPERATOR, not specialist.** |
| **⬜ REFERENCE / OUTPUT** | Everyone | Historical log snippets, paths or expected output; **not a command to execute** |

Color is reinforced by text labels for terminals and Markdown renderers that do not display color emoji. **Every executable command block below is preceded by one of these labels.** Blocks containing both routine status and expert diagnostics explicitly identify both. Only commands marked **OPERATOR / RUN, STOP, STATUS** or a separately authorized **SPECIALIST** step should be used for normal operation.

- **[29-OCT RUN]** means physically exercised in the original October 29 integration run.
- **[30-OCT OBSERVED]** means observed in the user's October 30 `main` run / shutdown console logs. A cleared local port or screen is **not** proof that the physical instrument was safe.
- **[SOURCE]** means found in source or earlier startup instructions, **not** independently physically validated by this runbook.
- **[CHECK]** is read-only inspection, not an authorization to change hardware.
- **[OPERATOR]** is an intentional physical action or safety decision by a qualified person.
- All shell commands are for Ubuntu/WSL as `utsav`, except those explicitly run via SSH on a named Raspberry Pi. Do not paste Ubuntu local commands into the Pi shell.

### Choose the current worktree once in each Genesis terminal

This initialization works **from inside any Genesis worktree directory** (including a nested folder); if you start outside the repository, it defaults to `main`. To deliberately use another worktree, set `GENESIS_WT` to its absolute path **before** running the block. This is terminal-local; repeat in every new Genesis terminal. The **PyNMR** terminal instead enters `/home/utsav/jlab_pynmr` and activates `pynmr`.

**🟩 OPERATOR / WORKTREE & ENVIRONMENT SETUP:**

```bash
conda activate genesis-p1
if [ -z "${GENESIS_WT:-}" ]; then
  GENESIS_WT="$(git rev-parse --show-toplevel 2>/dev/null || printf '%s' /home/utsav/UNH-Genesis-Control/worktrees/main)"
fi
export GENESIS_WT
cd "$GENESIS_WT"
printf 'Account=%s\nWorktree=%s\nBranch=%s\nSHA=%s\n' \
  "$(id -un)" "$PWD" "$(git branch --show-current)" "$(git rev-parse --short HEAD)"
```

**Require `utsav`, an explicitly approved branch and the expected software release before connecting instruments.** As of the October 30 run, `feature/visualizer-integration` remains at `4846a4d`, while `main` is at `fd67507` after a **runbook-only** commit. Those commits have identical tracked software outside this documentation file. Future code changes or other worktrees require their own review; an old detached `repo/` checkout is **not** automatically the current baseline. Never run two physical owners from different worktrees at the same time.

### Quick operator command map (full safety conditions are in the numbered sections)

| Subsystem | 🟩 Routine operation / status | 🟧 Specialist / manual service |
|---|---|---|
| Panopticon | `/usr/local/bin/phoebus -settings ... -resource ...` (§2); normal GUI Exit (§12.9) | Verified own-process `TERM` fallback only when necessary (§12.9) |
| Cryo | `./scripts/genesisctl.sh cryo-status`, `cryo-start`, `cryo-stop`, `cryo-phoebus` (§3, §12.6) | Pi USB/bridge recovery only under hardware SOP |
| Trend History | **Start:** `screen -dmS genesis-hist ...`; **status:** `screen -ls | grep genesis-hist`; **stop:** `screen -S genesis-hist -X quit` (§4, §12.7) | Owner/port/log troubleshooting only if routine status fails |
| Microwave | `./scripts/genesisctl.sh mw-status`, `mw-start`, `mw-rf-ready`, `mw-stop` (§5, §12.4) | Hardware VDI/UCA/USB sequence; independent SIG scope IOC release (§12.4) |
| PyNMR | `python pynmr_main.py -p Deuteron`; GUI Stop/Exit (§6, §12.3) | Event/FPGA/IPC debugging (§6, §7) |
| ssRF | `./scripts/genesisctl.sh ssrf-physical-status`, `./scripts/ssrf-start-auto-clock.sh`, `./scripts/genesisctl.sh ssrf-stop` (§7, §12.2) | Pi clock sync is **state-changing**; specialist RF/burn procedures not covered |
| Magnet | Read-only GUI/readbacks (§8); authorized monitor status | Ellie’s explicit `genesis-magnet` monitor shutdown, verified child exit, and physical sign-off (§8.4, §12.5); **not magnet ramp-down/power-off** |
| Master Archiver | `./scripts/genesisctl.sh master-archive-start`, `master-archive-status`, `master-archive-stop` (§9, §12.8) | Per-PV quality/CSV/JSONL diagnostics (§9) |
| Streamlit | `./scripts/genesisctl.sh archive-viewer`; foreground `Ctrl+C` (§10, §12.1) | Port `8501` owner diagnostics (§10) |
| Complete-system check | `./scripts/genesisctl.sh status` and per-subsystem status | `screen`, `ps`, `ss`, `ip`, `curl`, archive forensic checks (§11, §12.10–12.11) |

**The table is an index, not a shortcut around interlocks, operator prompts or physical preflight.**

---

## 0. The entire sequence at a glance

**Startup (controlled):**

1. Confirm authority, equipment readiness, **approved current worktree/SHA**, and exclusive device ownership. Run the per-terminal `$GENESIS_WT` setup.
2. **Open Panopticon** using its **own** live Phoebus settings. Some widgets are magenta while IOCs are still down; this is expected.
3. Physically prepare Cryo devices, the shared Raspberry Pi, and connections. Start **all 19 Cryo IOCs** and inspect `cryo-status`. Optionally open the dedicated Cryo GUI.
4. **Operator:** start **Trend History IOC**, CA port `5130`, with §4’s `screen -dmS genesis-hist ...`; validate that the Panopticon time-history charts populate. This is a routine run/status/stop action, not expert maintenance.
5. Physically prepare the Microwave UCA/VDI/USB/robot connections. Start Microwave under its supervised operator prompt; confirm RF OFF and robot disabled.
6. Start **PyNMR** in its own terminal; check its hardware route, stream configuration, protected-event socket, and ensure there is no competing fitter or PyNMR producer. **Do not start a standalone fitter.**
7. Validate Keysight and ssRF hardware, correct its **10 ms waveform period** and Raspberry Pi clock conditions; run `ssrf-start-auto-clock.sh` **once**, which starts the full ssRF stack in **operator burn mode** (inspect current script arguments if it changes). Its **ssRF-owned protected fitter** must bind to the running PyNMR process.
8. Verify completed PyNMR acquisitions are **accepted** by the protected fitter and live fitter/plot PVs update. Do not confuse a noise spectrum with a meaningful polarization measurement.
9. **Magnet (optional, specialist only):** with Magnet operator authorization, start the IPS120 IOC **read-only** if permitted. It was offline on October 29, but a separate read-only IOC session was observed on October 30. Magnet release requires physical HOLD/heater/monitoring sign-off (§8.4, §12.5).
10. Start **Unified Master Archiver** (`448` configured PVs in the October 29 release); check rising snapshots/events and quality. Open the **Streamlit viewer** and inspect its live archive session.

**Shutdown (reverse/safe):** close Streamlit; safe-stop **ssRF first**, then PyNMR; safe-stop Microwave and complete its physical shutdown; stop any **independent Siglent scope** after measurement release; stop optional **read-only Magnet IOC only under its physical specialist procedure**; safe-stop Cryo after confirming coldboard OFF and independent monitoring; **operator-stop Trend History** (`screen -S genesis-hist -X quit`); request/verify Master Archiver finalization; close **Panopticon LAST**. Verify both screen **and child-process/port** exit, plus archive persistence. During network loss use §12.11 rather than force-clearing ports.

**Never use** `./scripts/genesisctl.sh stop` or `force-stop` as generic shutdowns; they are deliberately disabled. Never run Ellie's `scripts/live_laptop/start.sh` or `stop.sh` unreviewed on any live worktree: that launcher has a different environment/layout, starts a standalone fitter, can **edit Magnet and LS455 YAML**, and includes aggressive stop behavior. Use the subsystem-specific commands below.

---

## 1. Identity, terminal layout, and release protection

### 1.1 Physical and software permission checklist [OPERATOR]

Before energizing or starting **anything**, confirm that the responsible experiment operator has authorized the run; the Cryo and Magnet equipment are physically safe for the intended activity; the RF path, absorbers/loads, robot travel and emergency stops are inspected; only the **`utsav`** Ubuntu account will own the hardware; and nobody on `sliferlab`, Windows LabVIEW, another WSL user, or another Pi runtime is competing for a device. `sliferlab` is **not** the operator account for this specific checkout. Do not start a second hardware owner merely because the screen appears blank.

Use five terminals, preferably keeping them open for the session:

| Terminal | Account/environment | Purpose |
|---|---|---|
| 1 | Ubuntu `utsav`, `genesis-p1` | Phoebus / Panopticon **foreground** GUI |
| 2 | Ubuntu `utsav`, `genesis-p1` | Genesis control, `screen`, logs, network and archive checks |
| 3 | Ubuntu `utsav`, `pynmr` | **Foreground PyNMR** (`/home/utsav/jlab_pynmr`) |
| 4 | Ubuntu `utsav`, `genesis-p1` | Optional read-only fitter diagnostics, plots, archive checks |
| 5 | Ubuntu `utsav`, `genesis-p1` | Streamlit viewer **foreground** |

**Do not launch** the standalone fitter in Terminal 4. The supported online fitter is created by the complete ssRF stack.

### 1.2 Confirm the current approved release [OPERATOR STATUS + EXPERT PROVENANCE]

First run the terminal setup in **How to use this document**, then confirm the selected checkout and the supported commands. No branch checkout, pull or Git mutation is required for operation.

**🟩 OPERATOR / HELP & CURRENT WORKTREE STATUS:**

```bash
cd "$GENESIS_WT"
printf 'Account=%s\nPath=%s\nBranch=%s\nCommit=%s\n' \
  "$(id -un)" "$PWD" "$(git branch --show-current)" "$(git rev-parse HEAD)"
./scripts/genesisctl.sh help
```

**🟦 EXPERT / GIT & HISTORICAL TAG PROVENANCE (READ-ONLY):**

```bash
git status --short
git rev-parse HEAD
git rev-parse 'panopticon-integration-partial-validated-20261029T173322Z^{}'
git log -3 --oneline
```

As of October 30, `main` and `feature/visualizer-integration` point to **`4846a4d`**. The October 29 validation tag resolves to **`fa3262e`** and intentionally **does not match current HEAD** because the operator-runbook commit came afterward. Do **not** move the tag to make SHAs match. A clean Git status is desirable, but a modified doc should be reviewed rather than erased with `git clean`, checkout or reset. `local_data/` remains outside Git.

### 1.3 Establish read-only baseline and ownership [CHECK]

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
./scripts/genesisctl.sh status
screen -ls
ps -eo user,pid,stat,args --cols 300 | grep -Ei \
  '[m]aster_ioc.py|[i]oc_manager.py|[o]nline_fitter.epics_run|[p]ynmr_main.py|[p]hoebus|[m]aster_service|[s]treamlit' || true
sudo ss -ltnp | grep -E ':(5010|5030|5064|5100|5130|5131|8501|8777|8778|8779)([[:space:]]|$)' || true
./scripts/genesisctl.sh master-archive-status
```

**Do not terminate anything from these results.** Identify which account, process start time, and subsystem own a listener. `screen -ls` identifies sessions **only for the current account**; a missing session is not evidence that another user or the Pi does not own the device. If another run appears active, coordinate handoff with the owner and use that subsystem's safe stop—never start over it.

### 1.4 Ports and addresses — quick reference

| Host / port | Purpose / ownership | Notes |
|---|---|---|
| WSL `192.168.1.6` | DAQ / Ubuntu operating host | Actual route/interface must be checked; WSL may have two subnets |
| `127.0.0.1:5111–5129` | Cryo EPICS Channel Access ports | See `configs/target_physical/ioc_manager/settings.yaml` |
| `127.0.0.1:5130` (PVA `5230`) | Trend History IOC | Manually started `genesis-hist`; **not** Master Archiver |
| `127.0.0.1:5131` (PVA `5231`) | Optional Magnet IPS120 IOC | Ellie magnet source; inactive Oct 29 |
| `127.0.0.1:5100` | ssRF-owned protected online fitter EPICS | **Do not run a second standalone fitter** |
| `127.0.0.1:8779` | Protected PyNMR spectrum TCP receiver | ssRF fitter **listens**, PyNMR sends; idle connection retries can occur |
| `~/.local/state/genesis/nmr-ipc/events.sock` | Local authenticated completed-event IPC | Unix socket; check owner and active process before any cleanup |
| `127.0.0.1:8777` | **Legacy LabVIEW fitter receiver** | Should not be used in PyNMR-only commissioning |
| Pi `192.168.1.24:8778` | Historical Keysight/bridge port | Was **FREE** during Oct 29 full Keysight IOC run; do not start competing bridge |
| Pi `192.168.1.24:5064` | Active Keysight WaveGen EPICS IOC | Explicitly added to `panopticon_live.ini` Oct 29 |
| Pi `192.168.1.24:5010` | Roots Right / Teledyne Hastings serial bridge | USB device required; Pi is shared with Keysight |
| Pi `192.168.1.24:5030` | Low-manometer serial bridge | FTDI `/dev/ttyUSB0` required, unavailable Oct 29 |
| `192.168.1.251:5012` / `:5013` | Left flow and separator flow bridges | Verify local configuration and host routing |
| Pi `192.168.1.23` | Microwave Vaunix/green-box Pi | Dedicated separate microwave hardware |
| `192.168.1.19` | UCA power supply | Physical connection and operation must be checked |
| `192.168.1.250:5001` | Magnet IPS120 via TXI017 serial-to-Ethernet | **Single hardware client**; 9600 8N2 in physical profile |
| `192.168.0.50` | R&S generator in Oct 29 PyNMR log | Requires active NMR subnet; **not** the SMC100A EPICS host |
| `192.168.0.72` | NMR FPGA reference in Ellie preflight | NMR subnet must be reachable; confirm current device inventory |
| `localhost:8501` | Streamlit viewer (common default) | Not a hardware control service |
| Phoebus Java `-server 4918` observed Oct 29 | Instance control port | **Observed runtime detail**, not fixed configuration; inspect active process |

Don't infer a TCP service from a PV prefix. Cryo, Microwave and ssRF have **different EPICS server ports**. Inspect `configs/phoebus/panopticon_live.ini` to confirm `org.phoebus.pv.ca/auto_addr_list=false` and the complete `addr_list` before troubleshooting magenta widgets.

---

## 2. Panopticon FIRST: launch and verify GUI [29-OCT RUN]

**Why first?** The operator gets one overview; disconnected IOCs initially appear magenta. **Do not** turn on physical devices or assume an alarm is harmless just to make a display green.

### 2.1 GUI prerequisites

At Terminal 1, ensure a graphical session exists (`echo "$DISPLAY"`, or use a supported local WSLg / authorized X-forwarding method) and **do not** use a stale Phoebus instance with different settings.

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
conda activate genesis-p1
cd "$GENESIS_WT"
ls -l /usr/local/bin/phoebus gui/genesis_panopticon.bob configs/phoebus/panopticon_live.ini
echo "DISPLAY=$DISPLAY"
ps -eo user,pid,args | grep -E '[p]roduct-4.7.4-SNAPSHOT.jar|[g]enesis_panopticon.bob' || true
```

If the existing Java process is another operator's or has unknown settings, **STOP** and coordinate; don't launch a duplicate or kill it. If **your own stale Panopticon** is running, close it normally before restarting with the correct settings.

### 2.2 Start live Panopticon in Terminal 1

**🟩 OPERATOR / LAUNCH PANOPTICON GUI:**

```bash
cd "$GENESIS_WT"
/usr/local/bin/phoebus \
  -settings "$PWD/configs/phoebus/panopticon_live.ini" \
  -resource "file://$PWD/gui/genesis_panopticon.bob"
```

Keep Terminal 1 open; a Java process should appear in `ps`. Do not treat the shell's initial prompt or harmless JavaFX warnings as proof that the GUI failed—look for the actual Phoebus window.

**Critical Oct 29 issue — Keysight magenta:** The source configuration previously omitted `192.168.1.24:5064`. This was corrected in commit `fa3262e…` and is included in `main` at `4846a4d`. If other PVs work but the WaveGen widget is magenta, check that the **selected worktree's actual configuration** is loaded, and that the Keysight IOC is running on the Pi. **Merely closing a display tab can reuse the old Java process and its old CA address list**. Exit the entire *correctly identified* Panopticon Java application, verify its process is gone, then relaunch with the absolute settings file. Do not modify a YAML or rewire a Pi to fix a Phoebus configuration problem.

### 2.3 Panopticon page navigation and plot expectations

Open the linked Cryo, Microwave, NMR/fitter, ssRF, and Magnet views using their existing display links. The generator-based `.bob` is tracked; **do not edit `gui/genesis_panopticon.bob` manually**. The October 26 handoff records nine linked displays and 13 `open_display` actions. If a linked display fails, verify its file exists and opens without modifying control logic.

At initial startup, **trends may be blank even when readbacks exist**. Panopticon's live XY/history plots depend on a separate **Trend History IOC**, which we start in §4 **after Cryo**. Starting Master Archiver alone does **not** bring those XY trends online. Confirm the trend/history service as soon as the underlying Cryo PVs are available.

**Normal failure presentation:** while all IOCs are stopped, many widgets and Magnet supply PVs are magenta/unavailable. Magenta means **no current EPICS connection**, not a safe physical readback. An old or stale `OK` string must never be accepted as evidence that the hardware is currently healthy.

---

## 3. Cryo devices, shared Raspberry Pi, and 19 IOCs [29-OCT RUN]

### 3.1 Physical preflight [OPERATOR]

The Cryo stack includes instrument readbacks, safety/alarm summaries, archiving and **two** serial devices on a Raspberry Pi that also owns the Keysight WaveGen. The operator must turn on/enable **only the instruments permitted by the current experiment SOP** and verify their displays, power, network, pressure and temperature sensors. **Do not turn on coldboard output** merely to pass startup; its safe shutdown requires `Coldboard_Mode=0`. Before touching a cryostat, confirm independent pressure/temperature supervision, valves, pumps, cooling state and the responsible cryogenic operator.

Known Cryo components: Siglent coldboard supply, MKS902B cell gauge, TPG262, IONIVAC gauge and ion pump, LS455 fringe-field instrument, Lakeshore LS224 fridge and magnet units, Roots Right/Left/Separator flowmeters, fridge and magnet LHe monitors, low manometer, safety interlock, target status, save/restore, subsystem archiver and alarm summary. On Oct 29 **19 Cryo IOCs** started; device power and network availability may vary per run.

**Network only, before starting:**

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
ping -c 1 -W 2 192.168.1.24 || true
ping -c 1 -W 2 192.168.1.12 || true
ping -c 1 -W 2 192.168.1.13 || true
ssh -o ConnectTimeout=5 unhpoltarg@192.168.1.24 'hostname; date -u; lsusb; ls -l /dev/serial/by-id/* /dev/ttyUSB* /dev/ttyACM* 2>&1'
```

A failed ping may reflect ICMP policy, so do not conclude a device is dead from ping alone. Verify the router reservation/device inventory and the Cryo configuration rather than guessing a new IP.

### 3.2 Shared Pi: two Cryo serial devices — inspect BEFORE startup

On Oct 29, `192.168.1.24` hosted the Keysight **and** these two Cryo USB peripherals:

| Cryo instrument | USB identity / serial device | Pi bridge | Oct 29 finding |
|---|---|---|---|
| Roots Right, Teledyne Hastings Digital | `05ff:2001`; `/dev/serial/by-id/usb-Microchip_Technology_Inc._Teledyne_Hastings_Digital-if00` → `/dev/ttyACM0` | TCP `5010` | Recovered after reconnecting to a working USB port, then `RPump_Status=OK`, live `RPump_SLM` |
| Low manometer, FTDI FT231X | `0403:6015`, serial `D307BRO7`; previously `/dev/ttyUSB0` | TCP `5030` | Missing USB device; bridge was listening but repeatedly reported `/dev/ttyUSB0` nonexistent |
| Keysight DSO-X 3024T WaveGen | `2a8d:1766`; USB/VISA `USB0::10893::5990::MY57252012::0::INSTR` | EPICS `5064` | Working; **do not disturb during ssRF operation** |

**The confirmed failure:** Kernel `xhci-hcd.1` reported `xHCI host controller not responding`, `HC died; cleaning up`, and the two serial devices disappeared from Linux USB **bus 3**. The Keysight remained on **bus 1**. After a cable reconnection Roots Right re-enumerated on bus 1, while the FTDI manometer did not. This was **not fixed** by restarting Cryo IOCs. `vcgencmd get_throttled` was `0x0` at inspection; it does **not** rule out an earlier transient electrical/controller fault. Linux **bus 3** is a bus number, not proof that the cable occupied a blue physical USB3 socket.

**Read-only Pi diagnosis [CHECK]:**

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
ssh unhpoltarg@192.168.1.24 '
  echo "USB inventory"; lsusb; lsusb -t;
  echo "Serial devices"; ls -l /dev/serial/by-id/* /dev/ttyUSB* /dev/ttyACM* 2>&1;
  echo "Bridges"; sudo ss -ltnp | grep -E ":5010|:5030" || true;
  echo "Recent kernel USB events"; sudo dmesg | tail -45
'
```

If Roots Right is missing, **and only after the hardware operator approves**, inspect/reseat **only its own USB cable** in a known functional port, without disturbing Keysight or a shared hub. Then inspect `lsusb`, the `/dev/serial/by-id/` link, and the bridge. If the low-manometer FTDI is absent, **do not restart IOCs hoping to recreate `/dev/ttyUSB0`**. Inspect/reseat only the dedicated FTDI cable if safe; if its controller is dead, stop and schedule recovery **after ssRF safe shutdown**. **Never reboot/reset/unbind the Pi USB controller during an active Keysight session**; this may also interrupt RF safety/ownership state. Do not delete/recreate device nodes, force-start a second serial bridge, or switch another person's USB connection.

A bridge can listen while the actual sensor is absent. A `COMM=0`, `NaN`, or stale archived `OK` is a **failed instrument** until verified. There are no read-only software steps that can certify physical vacuum safety from a missing manometer.

### 3.3 Start Cryo IOCs (Terminal 2)

**🟩 OPERATOR / CRYO START & STATUS:**

```bash
cd "$GENESIS_WT"
./scripts/genesisctl.sh cryo-status
./scripts/genesisctl.sh cryo-start
./scripts/genesisctl.sh cryo-status
```

**Expected Oct 29 pattern:** 19 detached `genesis-cryo-*` `screen` sessions; Cryo local archiver reports **168 PVs**; individual instrument `COMM` states reflect actual physical connectivity; right/left/separator flows and fridge/magnet thermometry update if devices are reachable. Cryo safety interlocks are **report-only** in this release; `Safe_Permit=0` / `Safe_Fault=1` must still be investigated by the responsible operator, not suppressed.

Verify screens and readbacks:

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
screen -ls | grep genesis-cryo || true
./scripts/genesisctl.sh cryo-status
```

Optional dedicated Cryo GUI (only when desired; Panopticon already links Cryo):

**🟩 OPERATOR / OPTIONAL DISPLAY LAUNCH:**

```bash
./scripts/genesisctl.sh cryo-phoebus
```

Do not launch a second physical Cryo IOC manager because a GUI is blank. Check first whether the display is using the correct `target_physical.ini` and whether a separate Phoebus Java process is already running.

### 3.4 Cryo validation and known issues

- **[RUN]** Oct 29 `CRYO:FLOW:RIGHT:COMM=1`, `RPump_Status=OK`, `RPump_SLM≈−0.30` after reconnect. A fresh **negative** flow is still not a calibrated/physically plausible positive-flow acceptance: verify sign and instrument direction separately.
- **[RUN]** Low manometer `VAC:LOW:COMM=0`, `Mano_PI=nan`; historic `Mano_Status=OK` was **stale** and must not be trusted as live.
- **[RUN]** Cryo coldboard `COMM=0` in one pre-shutdown status; do not assume its output state. The managed shutdown later positively read `Coldboard_Mode=0` and passed.
- **[RUN]** Cryo safety fault reason `F1_TI 301.9 K >= 4.5 K` during a warm/nonproducing system. This was **not** a clearance for a real cooldown. Also `Alarm_Count=24`, `Alarm_Worst=3`, `Alarm_Missing=24`: review every applicable alarm in an actual cooldown.
- **[SOURCE, not Oct 29 corrected]** Ellie’s older live-laptop script contained a local LS455 scaling patch (`tesla_scale` set to `1.0` because instrument reported tesla); the October 29 validation did **not** establish a need to reapply that patch. **Do not run the old `sed -i`** or modify the checked-out source. Verify unit conversion with the magnet/LS455 specialist if a fringe-field readback is implausible.

---

## 4. Trend History IOC and Panopticon time-series plots [29-OCT RUN]

**Problem observed:** Cryo was live but Panopticon Magnet/fridge XY-history plots were blank. **Fix:** start the **dedicated Trend History IOC** on CA `5130` (not the Master Archiver). This run restored chart history, monitored **37 PVs**, and backfilled approximately **883 points** from existing Cryo CSV.

The operator starts, checks, and stops Trend History like any other routine monitoring subsystem. It has **no `genesisctl.sh` wrapper in this baseline**, so the **exact `screen` commands are the routine operator commands**. It is read-only and does not actuate hardware. Start only after Cryo is running and do not create a second owner.

**🟩 OPERATOR / TREND HISTORY STATUS (READ-ONLY):**

```bash
screen -ls | grep genesis-hist || echo '[DOWN] No local Trend History screen'
```

If a local `genesis-hist` is already active, **reuse it**. If no local screen exists, an unexpected occupied port requires an expert owner check, not a second launch:

**🟦 EXPERT / CHECK UNEXPECTED HISTORY PORT OWNER (READ-ONLY, IF NEEDED):**

```bash
ss -ltnp | grep -E ':(5130|5230)\b' || true
```

If **neither** the session nor either port is in use, launch from the current approved worktree:

**🟩 OPERATOR / START TREND HISTORY IOC:**

```bash
cd "$GENESIS_WT"
screen -dmS genesis-hist \
  env EPICS_CAS_SERVER_PORT=5130 EPICS_PVAS_SERVER_PORT=5230 \
  bash -c 'python -u master_ioc.py -s configs/target_physical/trend_history -i trend_history >> logs/trend_history_live.log 2>&1'
```

Verify the routine service state and plot updates:

**🟩 OPERATOR / TREND HISTORY STATUS:**

```bash
screen -ls | grep genesis-hist || echo '[DOWN] Trend History not running locally'
```

**🟦 EXPERT / PORT AND LOG DIAGNOSTICS (ONLY IF STATUS/PLOTS FAIL):**

```bash
ss -ltnp | grep -E ':(5130|5230)\b' || true
tail -40 "$GENESIS_WT/logs/trend_history_live.log"
```

In Panopticon, check the Cryo pressure/flow and fridge/magnet plots: **history traces should fill and update**. For a Phoebus plot/databrowser `.plt` linked from a subsystem screen, use the existing linked plot file; do not invent a PV name or open an older simulation plot. Discover installed plot resources if needed:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
find gui configs -type f \( -name '*.plt' -o -name '*.bob' \) | grep -Ei 'trend|hist|cryo|fridge|magnet|fitter|pq' | head -60
```

If plots stay blank: check `genesis-hist` screen and `5130`; see `logs/trend_history_live.log`; make sure the source Cryo IOCs have **fresh** readings; verify Panopticon was fully restarted with the correct `panopticon_live.ini`. **Do not** start a second history IOC on an occupied port. The live chart history IOC and the Unified Master Archiver are **different services** with different outputs and shutdown commands.

---
## 5. Microwave: supervised physical startup, Pi and RF-safe checks [29-OCT RUN]

### 5.1 Physical startup — human at apparatus [OPERATOR]

**Microwave is not a software-only startup.** Have the microwave operator at the rack. Identify and inspect the **UCA programmable supply** (`192.168.1.19`), **VDI-175T green box**, **Vaunix/LabBrick source**, **PCL602 robot**, the **Microwave Raspberry Pi** (`192.168.1.23`) and the USB path from the green box to that Pi. Check interlocks, RF termination/reflected-power path, robot clearances, emergency stops and cables before typing `READY`.

Established physical sequence from validated October procedures:

1. Inspect the hardware and cables. **Do not connect the USB to the green box until the authorized physical power-up sequence is ready.** Check the VDI–Pi USB cable, connectors and that the Raspberry Pi is actually running; check the network separately from device enumeration.
2. Prepare the UCA power supply at the known safe **approximately 5 V** setting under the authorized SOP. The startup script configures setpoints and verifies them; do not independently improvise voltage/current limits.
3. Verify the **VDI-175T green light ON** and that its supply and interlock are ready. **Only then connect the authorized green-box USB path** and confirm the source/bridge can enumerate. Confirm the robot is disabled/idle.
4. At the `mw-start` prompt, **read** the full safety instructions. Type `READY` **only if** the operator has positively confirmed the expected physical state, connection, RF route and output OFF. Do not treat this as a formality.
5. Microwave startup may request the authorized Pi SSH password. Never store passwords in the runbook or shell history. A password prompt may signal that the SSH key is missing; don't repeatedly retry until locking the account.

**The presence of VDI green light and an RF-ready software status is not permission to transmit RF.** Hardware reflected power must remain within the established procedure, and the physically connected SIG readout/reflector conditions must be confirmed for any future intentional RF test. On Oct 29, we **did not energize microwave RF for this integrated check**.

### 5.2 Read-only network and ownership preflight [CHECK]

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
cd "$GENESIS_WT"
ping -c 1 -W 2 192.168.1.23 || true
ping -c 1 -W 2 192.168.1.19 || true
ssh -o ConnectTimeout=5 unhpoltarg@192.168.1.23 'hostname; date -u' || true
./scripts/genesisctl.sh mw-status
screen -ls | grep -Ei 'mw_|ioc-manager|archiver' || true
```

A missing PV when the IOCs are **not yet started** is normal. If a Microwave IOC manager or Vaunix owner is already running under another account, **STOP** and coordinate the owner; never run another `mw-start` over it.

### 5.3 Start and validate [29-OCT RUN]

**🟩 OPERATOR / SUPERVISED MICROWAVE START:**

```bash
cd "$GENESIS_WT"
./scripts/genesisctl.sh mw-start
```

**🟩 OPERATOR / MICROWAVE STATUS & READINESS (READ-ONLY):**

```bash
./scripts/genesisctl.sh mw-status
./scripts/genesisctl.sh mw-rf-ready
```

**Expected on Oct 29:**

| Readback | Safe / observed baseline |
|---|---|
| UCA | `COMM=1`, `V≈5.001 V`, output `1` |
| Vaunix source | connected `1`; output approximately **140 GHz**; power **setting** `+10 dBm` |
| Actual RF | `RF=0`, `Supervisor RF=0` — **OFF** |
| Source health | `RF valid=1`, `control owned=1`, `PLL=1` |
| Robot | `COMM=1`, `CONTROL_MODE=0`, X busy `0`, Y busy `0` |
| Supervisor | connected `1`, RF permissive `1`, but supervisor RF output **0** |
| Subsystem archiver | active; Oct 29 status reported **82 PVs** |

`mw-rf-ready` was run **read-only** and reported readiness for a future, separately authorized RF/SIG procedure. It also printed **manual confirmations** (VDI green light, SIG readout visibility, aluminum/foil reflector, reflected power below 10%, UCA 5 V, and SIG initial check near 4 V—not below 3.5 V). The Siglent/SIG **new scope was not connected during Oct 29**; its 4 missing Master Archive PVs were correctly classified as an **expected instrumentation gap**, not a required repair for this run. Do not claim RF or SIG physical acceptance on the basis of `mw-rf-ready` alone.

The `RF state=1` status field was not the same as actual RF output. We verified OFF with the **explicit** `RF=0`, `Supervisor RF=0` and readiness results. Never infer RF state from one ambiguous status integer.

Optional independent Microwave GUI (may be unnecessary when navigating through Panopticon):

**🟩 OPERATOR / OPTIONAL DISPLAY LAUNCH:**

```bash
./scripts/start_microwave_phoebus.sh
```

**Troubleshooting:** If no source/robot data, check the hardware Pi and UCA connections, power sequence, running sessions and Microwave logs **before** restarting anything. If `mw-start` says another owner exists, resolve ownership by the **managed safe stop** with the current operator's permission—not by `pkill`. An unused SIG scope can remain missing without affecting source RF OFF and robot safety; document its absence.

---

## 6. PyNMR: exclusive Deuteron producer and port/IPC preflight [29-OCT RUN]

### 6.1 This release's correct topology

**PyNMR (Ubuntu foreground) → localhost TCP `8779` + authenticated completed-event IPC → ssRF-owned protected online fitter (EPICS CA `5100`) → ssRF recipe/GUI → Panopticon plots**.

The **full ssRF startup** creates the protected fitter, with the current PyNMR producer **PID and start-ticks** pinned. The fitter must know the physical NMR switch IOC and connection route. **Do not start** a standalone `python -m online_fitter.epics_run`, a `python -m online_fitter.run` legacy GUI, a separate `genesis-fitter` from Ellie’s script, a LabVIEW receiver, or a second PyNMR source. On Oct 29, a standalone fitter connected to TCP but **rejected completed events** because the NMR-switch route/IOC was missing. Stopping that standalone fitter and running the **complete ssRF workflow** fixed the admission path; the GUI/plots then populated.

**NMR physical switch:** Mini-Circuits RC-2SPDT-A18 NMR switch, PyNMR on **A1/B1**, COM to target, A2/B2 terminated. Verified disconnected/baseline mapping: **mask 0 = NMR connected**. Separate ssRF RF switch **mask 3 = RF disconnected**. Do not swap mask meaning based on an older lab note; check physical cables and the correct switch device serial/readbacks.

### 6.2 Check old users, process owners and busy ports [CHECK]

In Terminal 2 **before launching PyNMR**:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
cd "$GENESIS_WT"
id -un
ps -eo user,pid,ppid,lstart,args --cols 340 | grep -Ei \
  '[p]ynmr_main.py|[o]nline_fitter.epics_run|[o]nline_fitter.run|[g]enesis-ssrf-fitter' || true
sudo ss -ltnp | grep -E ':(8777|8778|8779|5100|5101)([[:space:]]|$)' || true
screen -ls | grep -Ei 'genesis-fitter|genesis-ssrf|nmr' || true
ls -ld ~/.local/state/genesis/nmr-ipc 2>&1 || true
ls -l ~/.local/state/genesis/nmr-ipc/events.sock 2>&1 || true
```

**Interpretation:**

- An old `online_fitter.run` or detached `genesis-fitter` may bind `8777`, `8779`, or `5100`, or **accept TCP but reject events**. Confirm which exact PID/account created it and whether it is genuinely stopped. If another operator owns it, **STOP** and obtain a planned handoff.
- `8779` being **free before ssRF startup is expected**: it is the protected fitter's receiver, **not** proof that PyNMR streamer is broken. PyNMR may say `protected idle connection unavailable` until the fitter starts.
- A socket path that exists is **not automatically stale**; a live process may own it. Never `rm -f ~/.local/state/genesis/nmr-ipc/events.sock` while a matching fitter may be active. Inspect `ss -xlpn`, ownership and process identity; use the supported `ssrf-stop` only after safety confirmation if a previous ssRF instance must be released.
- `8777` is legacy LabVIEW receive path and **should not** be a live competing source in the PyNMR-only acceptance.
- If the other WSL account/Windows DAQ/old Linux service owns FPGA, R&S or Keysight, do **not** resolve by killing it—coordinate device handoff.

### 6.3 Start PyNMR in Terminal 3 [29-OCT RUN]

**🟩 OPERATOR / START PYNMR GUI:**

```bash
conda activate pynmr
cd /home/utsav/jlab_pynmr
python pynmr_main.py -p Deuteron
```

Terminal 3 stays open; do **not** start PyNMR through `genesisctl.sh` or from a Mac. The October 29 source reported local PyNMR branch `unh_dev/genesis_dual_account_ipc`, with local session/history file changes; these are separate runtime artifacts and **not** part of the Genesis `fa3262e` validation commit. Do not `git clean` or reset the PyNMR directory to make it look tidy.

**Observed Oct 29:** Deuteron profile (`daq_type: FPGA`, default `Deuteron5T`), R&S generator at `192.168.0.50`, startup log `Data streamer sending to 127.0.0.1:8779`; real acquisition did several **640-sweep events** (~17 s each). A recurring event-file exception **`'NoneType' object has no attribute 'write'`** appeared at `EVENT_FINISHED`, and an optional LabJack `libLabJackM.so` import warning appeared, but that day the protected path **later accepted** a finished event and plots populated. This does **not** mean the event-file error is resolved: preserve the error for follow-up and verify your own run's data are actually admitted and archived.

At this stage, check the GUI and hardware connections; ideally wait to start repeated sweeps until the complete ssRF fitter is online (§7). If acquisition starts before the receiver is ready, log retries/rejected events are possible—**do not** treat them as valid fitted events. The physical FPGA/Q-meter and R&S must be available and tuned per the NMR operator's procedure, not inferred from a graphical waveform.

Read-only cross-check in Terminal 2:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
ps -eo user,pid,stat,args --cols 300 | grep '[p]ynmr_main.py' || true
sudo ss -ltnp | grep ':8779 ' || true
```

Record the **actual** PyNMR PID from this session. **Do not reuse the October 29 PID `14103`**; that process exited at shutdown.

### 6.4 What to do when PyNMR refuses to start

1. Read the **exact first error** in Terminal 3. Confirm `conda activate pynmr` and correct `/home/utsav/jlab_pynmr` path.
2. If it reports FPGA or R&S timeout, verify NMR subnet (`ip route`, `ip addr`, `ping` when applicable); the WSL host may have **both `192.168.1.x` and `192.168.0.x` routes**. Check target devices and cables; do not swap to another generator without permission.
3. If a port is in use, use `sudo ss -ltnp` to identify the **exact owning PID/user**, and `ps -p <PID> -o user,pid,lstart,args` to confirm. Do not use a global `fuser -k`, `pkill python`, or delete IPC files to force a connection.
4. If logs show `Error in run loop: timed out`, **stop acquisition in the PyNMR GUI**, inspect physical FPGA/DAQ and network, then retry only after the operator confirms instrument readiness. Oct 29 one early run received only 64 of 640 sweeps before a timeout; later runs reached 640. Do not present a truncated sweep as successful.
5. If already running, **do not launch a second GUI**. Reuse the correct operator's instance, or stop acquisition and close it normally as part of an authorized one-owner handoff.

---

## 7. ssRF complete physical workflow, Pi time, protected fitter and burn-mode [29-OCT RUN]

### 7.1 Physical preflight — RF remains disabled [OPERATOR]

Have the ssRF/NMR operator confirm the source, target/termination, Mini-Circuits switches, SMC100A, Keysight DSO-X 3024T, WaveGen output and all interlocks. **No RF enable, FM enable, WaveGen output enable, ARM, or burn should be issued just to make PVs green.** For an actual burn, the separate hardware/RF safety procedure and correctly terminated target path must be approved; Oct 29 did **not** establish complete Single/Continuous Burn physical acceptance.

Known safe baseline from Oct 29:

| Device / state | Safe physical value |
|---|---|
| R&S SMC100A carrier | `32.50928747 MHz` (observed snapshot; **not** a universal fixed Larmor setpoint) |
| SMC power | `−120 dBm`; power mode `CW`; RF **False**; FM **False** |
| NMR switch | Model RC-2SPDT-A18, serial `02206270097`, **mask 0** / PyNMR A1/B1 to target COM |
| ssRF RF switch | Model RC-2SPDT-A18, serial `02308310195`, **mask 3** / RF disconnected |
| Keysight | DSO-X 3024T, serial `MY57252012`, **WaveGen output OFF**, ARB function, period **0.010 s**, VHIGH `+0.5 V`, VLOW `−0.5 V` |
| Keysight hardware resource | `USB0::10893::5990::MY57252012::0::INSTR` |
| Pi clock | Ubuntu–Pi clock difference within preflight tolerance; Oct 29 final verification was **218 ms** |

**Critical Oct 29 correction:** Before the full ssRF could start, the Keysight WaveGen was at **0.1 s**, not **0.01 s**; the operator physically corrected it to **10 ms**, after which ssRF startup passed. Do not silently change output parameters over an unverified control connection. A trained operator must inspect physical settings and correct the documented safe baseline **with output OFF**, then repeat the independent read-only preflight. The validated ARB length from prior ssRF work was **8192 points**.

### 7.2 Inspect Pi hardware ownership and clock [CHECK]

In Terminal 2:

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
cd "$GENESIS_WT"
./scripts/genesisctl.sh ssrf-physical-status
ssh -o ConnectTimeout=5 unhpoltarg@192.168.1.24 'date -u; lsusb; sudo ss -ltnp | grep -E ":5064|:8778" || true'
```

**Expected before startup:** no competing `genesis-keysight-bridge` at `8778`; if the full Keysight IOC already owns USB (EPICS `5064`), the read-only script **skips a direct USB probe** to avoid controller contention. Its statement `Direct USB probe skipped` is an **intentional safety action**, not proof that WaveGen is OFF. Later, after a managed ssRF shutdown released the owner, direct probe independently verified WaveGen OFF and 10 ms period.

**Keysight Pi clock issue:** The Pi's clock can drift from Ubuntu and fail ssRF authenticated/preflight checks. The supported Oct 29 launcher is `scripts/ssrf-start-auto-clock.sh`, which can **disable Pi NTP, synchronize Pi clock to WSL and start the physical ssRF stack**. It is **not read-only** and should be used only after operator approval, when no other device/user depends on the Pi time service. The script verified a clock delta within its startup threshold (historically 2 s) and allowed safe startup. The separate `ssrf-physical-status` read-only script accepts clock agreement within **10 seconds**; these are **different checks**. An observed 0.2 s difference does not justify another time change once the stack is running.

**Do not** manually run `date -s`, `timedatectl set-time`, restart NTP, reboot Pi, or kill its Keysight process during ssRF operation. Any time-service policy restoration after the experiment must be coordinated with the Pi maintainer; the October 29 session does **not** prove Pi NTP was restored automatically.

### 7.3 Start complete ssRF in operator mode (Terminal 2) [29-OCT RUN]

**Required prerequisite:** PyNMR's **one and only** foreground process is already running (§6). There must be no active standalone fitter or partial ssRF owner. Check current command help/script if the checkout differs from the tag.

**🟩 OPERATOR / CHECK ssRF READ-ONLY STATUS:**

```bash
cd "$GENESIS_WT"
./scripts/genesisctl.sh ssrf-physical-status
```

**🟧 SPECIALIST / AUTHORIZED CLOCK SYNC + ssRF START:** This launcher changes the Raspberry Pi clock/NTP configuration and starts physical device owners. Never use it solely to repair a display or while another owner is active.

```bash
./scripts/ssrf-start-auto-clock.sh
```

The Oct 29 `ssrf-start-auto-clock.sh` ran clock synchronization and the equivalent of `genesisctl.sh ssrf-start --burn-mode operator`. The accepted stack reported:

- `genesis-ssrf-smc`, `genesis-ssrf-nmr-sw`, `genesis-ssrf-rf-sw`, `genesis-ssrf-interlock`, `genesis-ssrf-sequencer`, `genesis-ssrf-recipe`, **`genesis-ssrf-fitter`** and `genesis-ssrf-archiver` detached sessions.
- Physical Keysight IOC on Pi `192.168.1.24:5064` (separate from the Ubuntu `screen` sessions).
- EPICS `COMM=1`, `UNH:GEN:SSRF:ILK:INPUTS_OK=1`, `SEQ:EXECUTION_ENABLED=1`, `SEQ:EXECUTION_ARMED=0`, `SEQ:PREPARED=0`, `FIT:FITTER_UP=1`, `RECIPE:STATE=OPERATOR_MODE`.
- ssRF subsystem archiver collecting **97 PVs**. PyNMR fitter EPICS on `5100`, spectrum TCP receiver `8779`, local authenticated completed-event IPC at `~/.local/state/genesis/nmr-ipc/events.sock`.
- **No burn armed.** SMC RF/FM OFF, RF switch mask 3, WaveGen output OFF.

Read-only verification:

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
screen -ls | grep genesis-ssrf || true
sudo ss -ltnp | grep -E ':(8779|5100)([[:space:]]|$)' || true
./scripts/genesisctl.sh ssrf-physical-status
./scripts/genesisctl.sh status
```

Use Panopticon's ssRF link or the dedicated ssRF GUI, **without starting new IOCs**. Optional display launcher (display only):

**🟩 OPERATOR / OPTIONAL DISPLAY LAUNCH:**

```bash
./scripts/start_ssrf_phoebus.sh
```

**Do not run both GUI launchers if they attach to the same Phoebus process in an uncontrolled way; prefer the Panopticon linked view.** A display is **not** an independent fitting/hardware service.

### 7.4 Explicit fitter admission check — do not accept TCP alone

Start a **short authorized** Deuteron acquisition from the PyNMR GUI only after `genesis-ssrf-fitter` is up. Observe a complete event; inspect the fitter screen or its actual log filename (find first, do not assume one):

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
screen -ls | grep genesis-ssrf-fitter || true
ps -eo user,pid,args --cols 300 | grep -E '[o]nline_fitter.epics_run|[p]ynmr_main.py' || true
ls -lt logs/*fitter* 2>/dev/null | head -15
rg -n 'GENESIS ADMISSION|GENESIS IPC|accepted=|pynmr_commissioning_unverified|route_epoch' logs 2>/dev/null | tail -35
```

**Positive Oct 29 evidence (shape of log):**

**⬜ REFERENCE / OUTPUT OR PATH (NOT EXECUTABLE):**

```text
[GENESIS ADMISSION] adapter accepted=True route={'selected_source':'pynmr', 'route_epoch':2, 'nmr_mask':0, 'nmr_comm':1, 'stream_connected':True, 'post_switch_ready':True, ...}
[GENESIS IPC] accepted=True reason=accepted
```

A successful TCP connection **alone is not proof of acceptance**. Require the correct route (`pynmr` only), `nmr_mask=0`, NMR switch IOC COMM=1, `stream_connected=True`, `post_switch_ready=True`, accepted completed-event IPC, new fit timestamp and new plot data. The fitter pin is tied to the producer's **PID and process start identity**; after PyNMR is restarted, the existing fitter may reject its events until the **entire ssRF stack is stopped safely and restarted** to pin the new producer. Do not bypass admission checks, spoof PID, or delete IPC sockets.

**Plot test:** open Panopticon → ssRF/fitter linked display; confirm the incoming raw spectrum and processed spectrum plots update. The fitter P/Q databrowser plot file was referenced in earlier milestones as `genesis_ssrf_fitter_pq.plt`; locate the exact path in this checkout with `find` before opening it. A flat/garbage spectrum with target stick disconnected **does not validate polarization**; on Oct 29 `P_FIT` values near zero and varying were merely connectivity evidence.

**Known misleading startup line:** A generic fitter startup message said fitting stays blocked until trusted event binding, but the live authenticated IPC subsequently logged `accepted=True`. Trust actual protected-runtime admission and plot timestamps, not just the startup banner.

**Known independent warning:** PyNMR may report `'NoneType' object has no attribute 'write'` on finished events; this must be documented and investigated, even if accepted fitter events are observed. A previous wrong path showed `Genesis rejected completed event; TCP suppressed`. Correct root cause was **standalone fitter without required NMR switch IOC**, not just a bad plot widget.

### 7.5 Operator burn mode is NOT authorization to burn

The operator-mode startup leaves `SEQ:EXECUTION_ARMED=0`, `SEQ:PREPARED=0`, RF/FM/WaveGen **OFF**. The GUI supports recipes and operator-gated actions, but **burn execution requires an entirely separate trained operator's hardware readiness and target/RF authorization**. Do **not** click `ARM`, `RF ON`, `BURN ONCE`, continuous burn, manual RF, or enable WaveGen as a troubleshooting step. The October 29 integrated test **did not** physically validate a meaningful Single/Continuous Burn with the target signal. The prior separate ssRF burn-commissioning worktree is not the relevant approved checkout; do not mistake its offline tests for this branch's physical burn acceptance.

If a later approved procedure does use burn features, the critical contractual properties to verify **without bypass** are fit-gated rejection of invalid fit, ungated prepared-burn authorization, one-shot and continuous mode ownership, STOP/ABORT, failed-cleanup inhibit, RF termination and physical interlocks. A fault or unknown cleanup state means **no subsequent RF enable** and a supervised fail-safe stop. Use the established subsystem operator SOP, not an improvised `caput` sequence.

### 7.6 Typical ssRF failures / recovery logic

| Symptom | Likely cause | Allowed first action |
|---|---|---|
| Startup fails 10 ms check | Keysight ARB period 0.1 s instead of 0.01 s | RF/WaveGen OFF; qualified operator correct physical period; rerun read-only `ssrf-physical-status` before startup |
| Pi clock skew/preflight fail | Pi NTP/time differs from WSL | Authorized, **pre-start only** `ssrf-start-auto-clock.sh`; do not manually adjust running Pi clock |
| `FITTER_UP=0` or TCP `8779` absent | Full stack not fully started or port already owned | Inspect `screen -ls`, `ss`, fitter logs; preserve owners; stop safely and start complete stack when authorized |
| `accepted=False`, no fitter plots | Wrong producer PID, lost IPC identity, route epoch/mask fault, standalone fitter | Check one PyNMR PID, NMR switch COMM/mask 0, `genesis-ssrf-fitter`, accepted IPC log. If restart needed, use **managed `ssrf-stop`** before a fresh complete start |
| NMR fit says `n/a` or spectra noise | No valid target signal/calibration/tuning | Don't report polarization; confirm raw acquisition/event acceptance; seek NMR expert for physical tuning |
| `/tmp/genesis-ssrf-physical/lifecycle.lock Permission denied` | Cross-user ACL/ownership or active owner | Inspect lock ownership and live processes; coordinate handoff. **Do not chmod/chown or delete** an active lock; this runbook assumes `utsav` only |
| Keysight direct USB probe skipped | Keysight IOC already owns device | Expected and safe. Use IOC readbacks; direct read-only USB probe after managed shutdown |
| Pi Keysight widget magenta | Missing CA address or IOC offline | Check approved `panopticon_live.ini` contains `192.168.1.24:5064`; inspect Pi IOC status; restart **entire** correctly owned Phoebus GUI only if it loaded an old settings file |
| `Port 8778` already in use | Competing legacy bridge or other app | Identify owner. Do **not** start competing Keysight bridge or force-reset device; authorized owner resolves conflict |
| Device disconnect after USB fault | Pi xHCI controller/USB enumeration problem | **Do not reboot/reset Pi** during ssRF. Finish safe ssRF stop first; follow Cryo USB diagnostic §3 |

---

## 8. Magnet supply — specialist read-only IOC [30-OCT OBSERVED; controlled shutdown]

The Magnet power supply was **OFF/disconnected** in the October 29 integration validation; the Master Archive recorded **59 missing Magnet supply PVs**. Cryo magnet thermometry remained a separate recorded group. On **October 30**, the operator independently launched the **read-only Magnet IPS120 IOC** from the `visualizer-integration` worktree and later stopped the associated `genesis-magnet` screen while running commands from `main`. The final local screen and TCP-port checks passed. **Those checks verify only software termination; the supplied console does not independently establish the IPS120's physical operating state.**

**Important:** Magnet current/persistent-switch control is specialist-only. The IOC guidance is adapted from Ellie's source and the checked-in `configs/magnet_physical/ips120/settings.yaml`. **Do not connect or disconnect its monitor unless the Magnet lead confirms the present physical state, single client ownership, and required independent monitoring.** Never energize a supply merely to populate a dashboard.

### 8.1 Source files and safety settings [CHECK]

- [Ellie `STARTUP_INSTRUCTIONS.md`, §3/§5](https://raw.githubusercontent.com/ellie-long/UNH-Genesis-Control/dev-visualizer/STARTUP_INSTRUCTIONS.md) — explicitly starts `genesis-magnet` on `5131/5231`; §5 closes it with `screen -S genesis-magnet -X quit` during a whole-stack software shutdown.
- [Ellie `scripts/live_laptop/start.sh`](https://raw.githubusercontent.com/ellie-long/UNH-Genesis-Control/dev-visualizer/scripts/live_laptop/start.sh) — read-only Magnet IOC launch, **but also has an opt-in edit that enables Magnet writes**; do **not** execute the whole script in this controlled checkout.
- [Ellie `scripts/live_laptop/stop.sh`](https://raw.githubusercontent.com/ellie-long/UNH-Genesis-Control/dev-visualizer/scripts/live_laptop/stop.sh) — her exact Magnet shutdown implementation: `quit_session genesis-magnet "master_ioc.py -s configs/magnet_physical/ips120"`, which first quits the matching screen and then sends `kill` to matching processes. **Treat the `pgrep -f` / broad-match fallback as historical source, not a safe command to copy into this runbook.**
- `configs/magnet_physical/ips120/settings.yaml` — Oxford IPS120-10, `192.168.1.250:5001`, `write_enabled: false` by default; physical network serial server at TXI017, 9600 8N2.
- `configs/magnet_physical/ioc_manager/settings.yaml` — IPS120 IOC CA `5131`, manager `5132`, `ips120.autostart: false`.
- `magnet_ioc/PORTING.md`, `magnet_ioc/interlocks.py`, `magnet_ioc/device.py` — review before control or operation.

In Terminal 2, after confirming Cryo monitors exist:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
cd "$GENESIS_WT"
grep -nE 'write_enabled:|ip:|port:' configs/magnet_physical/ips120/settings.yaml
screen -ls | grep genesis-magnet || true
sudo ss -ltnp | grep ':5131 ' || true
ping -c 1 -W 2 192.168.1.250 || true
```

**Required:** `write_enabled: false` and **no active competing magnet/serial owner**. A successful ping to the TXI017 serial server does not prove the IPS120 is powered, responsive or safe. **Do not** send raw serial commands, `nc`, telnet probes, or control-capable IOC writes to test connection; never start two clients to `192.168.1.250:5001`.

### 8.2 Read-only start — only with Magnet operator approval [SOURCE]

Manual read-only launch pattern, from the **currently selected worktree** (tested as an IOC lifecycle on October 30):

**🟧 SPECIALIST / MANUALLY START MAGNET READ-ONLY IOC:**

```bash
cd "$GENESIS_WT"
screen -dmS genesis-magnet \
  env EPICS_CAS_SERVER_PORT=5131 EPICS_PVAS_SERVER_PORT=5231 \
  bash -c 'python -u master_ioc.py -s configs/magnet_physical/ips120 -i ips120 >> logs/magnet_ips120_live.log 2>&1'
```

**Do not run if the `genesis-magnet` screen/`5131`/physical serial port is already owned.** The Magnet physical profile is read-only by default, but a specialist must still verify current branch/configuration and that connecting to this particular instrument will not interfere with active operation. Verify without control writes:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
screen -ls | grep genesis-magnet || true
tail -50 logs/magnet_ips120_live.log
sudo ss -ltnp | grep ':5131 ' || true
```

For a connected, safe, read-only session, expect `UNH:GEN:MAG:COMM` to become active and readbacks like `CURRENT`, `FIELD`, `HEATER`, `QUENCH`, `PERMIT`, `PERMIT_REASON` and `WRITE_ENABLED` to appear in Panopticon. **Do not assume** a nominal field/current or a permit state. The source profile includes LHe, LS224 and IONIVAC heartbeats, temperatures, shell vacuum and persistent-current safeguards; without live valid Cryo data, Magnet operations cannot be assumed permitted.

**No magnet power operations, sweeps, heater switching, current setpoints, quench response, or write-enable procedure is provided here.** Those require Prof. Long's active supervision and their own approved detailed magnet operating procedure. In particular, **never** run Ellie's old automatic `sed -i` enabling `write_enabled: true` as part of an ordinary read-only startup.

### 8.3 Magnet not connected / blocked port

If the IOC starts but `COMM=0`, or logs show the serial server is already in use, **do not repeatedly restart the IOC**. First verify power/physical availability and single ownership with the magnet lead; inspect `logs/magnet_ips120_live.log`, the network, and known producer sessions. If another client holds port 5001, do not force it off. Leave the Magnet **offline/missing** in the Master Archive, as on Oct 29, until authorized.

A manual `genesis-magnet` IOC has **no dedicated verified `genesisctl.sh magnet-stop` command** here. Shutdown is a separately authorized specialist action with explicit front-panel inspection, not a generic software stop. See **§8.4** and **§12.5**. If safe release cannot be established, retain the IOC and independent monitoring.


### 8.4 Magnet IOC shutdown / release, with Ellie's original stop instructions [SOURCE + SPECIALIST]

**What Ellie's repository actually instructs:** Her `STARTUP_INSTRUCTIONS.md` §5 lists `screen -S genesis-magnet -X quit` in the full-system stop. Her `scripts/live_laptop/stop.sh` calls `quit_session genesis-magnet "master_ioc.py -s configs/magnet_physical/ips120"`. That helper closes the `screen` and attempts to terminate surviving matching processes. **Neither source gives a physical IPS120 ramp-down, heater transition, persistent-mode conversion or magnet power-off sequence.** Those are not implicit parts of a software shutdown. **Do not execute Ellie's entire `stop.sh` here**: its generic process matching and Cryo fallback bypass the identity and safety checks of the current worktree. Use the verified, targeted steps below instead.

**This is a monitoring-service shutdown, not a physical magnet shutdown.** Do not change current setpoints, ramp rates, heater state, persistent-mode state, or power. The IOC's current profile explicitly says `write_enabled: false`; confirm that live configuration before proceeding. The October 30 `caget` check could not run because `caget` was not installed, so missing CLI results were **not** evidence that the magnet was holding.

**Magnet lead's physical release checklist (required):** independently confirm on the IPS120 front panel and laboratory instrumentation (1) supply activity is **HOLD/no active ramp**, (2) no heater switching or transition is active, (3) no unresolved quench, abnormal voltage/current or interlock condition, (4) cryogenic/safety monitoring will continue by an independent approved route, and (5) the operator approves disconnecting the read-only EPICS monitor. If any item cannot be verified, **leave `genesis-magnet` running** and arrange a specialist handoff.

**🟦 EXPERT / IDENTIFY THE EXACT MAGNET IOC AND LOG (READ-ONLY):**

```bash
screen -ls | grep genesis-magnet || true
ps -eo user,pid,ppid,lstart,args --cols 300 | grep '[m]aster_ioc.py -s configs/magnet_physical/ips120' || true
ss -ltnp | grep -E ':(5131|5231)\b' || true
# Replace ACTUAL_MAGNET_PID only with the live Python PID found above:
# readlink -f /proc/ACTUAL_MAGNET_PID/cwd
# tail -50 /proc/ACTUAL_MAGNET_PID/cwd/logs/magnet_ips120_live.log
```

**🟧 SPECIALIST / ELLIE'S MAGNET IOC STOP, AFTER PHYSICAL SIGN-OFF (SOFTWARE ONLY):**

```bash
# This is the actual screen shutdown shown in Ellie's STARTUP_INSTRUCTIONS.md §5.
# Use only after the physical release checklist above and ownership check pass.
screen -S genesis-magnet -X quit
```

**🟦 EXPERT / VERIFY EXIT AND PORT RELEASE (READ-ONLY):**

```bash
screen -ls | grep genesis-magnet || echo '[PASS] No local Magnet screen'
ps -eo user,pid,args --cols 300 | grep '[m]aster_ioc.py -s configs/magnet_physical/ips120' || true
ss -ltnp | grep -E ':(5131|5231)\b' || echo '[PASS] Magnet CA/PVA ports not listening'
```

**Ellie's `stop.sh` goes one step further:** after the screen quit, it searches for processes matching `master_ioc.py -s configs/magnet_physical/ips120` and sends `SIGTERM`. **Do not copy its `pgrep -f` loop or run it as a whole-stack stop.** The screen may disappear before its Python child releases a socket. If a matching IOC persists, first verify its **current** PID, user, start time and `/proc/<PID>/cwd`; inspect logs and confirm specialist approval before a **targeted graceful SIGTERM**. Never paste an old PID from October 30 (`41853` was historical), use `kill -9`, or kill a different Magnet/LabVIEW owner. Recheck the ports after termination. If the IOC was being used for required monitoring, do not release it even if only two ports remain occupied.

**October 30 specific observation:** a script requiring the literal `MAGNET SAFE` exited with `[STOP] Magnet monitoring retained` because that physical confirmation was not entered; later `screen -S 41847.genesis-magnet -X quit` removed the screen, and `5131/5231` were confirmed free. The final software state was checked, **not** the physical HOLD/heater state. Future operators must not treat those historical commands as blanket clearance.

---
## 9. Unified Master Archiver: start, verify, quality, session files [29-OCT RUN]

The **Master Archiver** reads PVs across multiple subsystems; it is **not** an IOC manager, has no device-control authority, and does not replace the separate Cryo/Microwave/ssRF archivers or the Trend History IOC. Start it once the intended IOCs are running, so an operating session shows fresh data from the start. It can keep running across subsystem shutdowns so the shutdown transition is recorded.

### 9.1 Before starting [CHECK]

**🟩 OPERATOR / ROUTINE STATUS OR READINESS (READ-ONLY):**

```bash
cd "$GENESIS_WT"
./scripts/genesisctl.sh master-archive-status
./scripts/genesisctl.sh cryo-status
./scripts/genesisctl.sh mw-status
./scripts/genesisctl.sh ssrf-physical-status
```

The latter three status checks may show timeouts for **intentionally stopped** or unavailable subsystems. Do **not** start the Magnet solely to eliminate its missing PVs; the 59 Magnet values should be recorded as missing in an otherwise valid archive. Check disk availability; the 29 October run had approximately **1 TB** free at startup.

### 9.2 Start recorder [29-OCT RUN]

**🟩 OPERATOR / MASTER ARCHIVER START AND STATUS:**

```bash
./scripts/genesisctl.sh master-archive-start
./scripts/genesisctl.sh master-archive-status
```

**Expected:** `[UP] master recorder PID <new_pid>`, a newly created `local_data/archive/master/<session_id>/` directory, `configured_pvs: 448`, `snapshot_interval_s: 1.0`, `state: running`. The very first status can show `snapshots: 0` and `accepted_events: 0` because it was queried immediately. This is **normal only at initial startup**. Run status again once real data have had time to arrive; the snapshot and event counters must increase. **Do not run a second `master-archive-start` if a recorder already owns the session**.

**October 29 final observed running result (historical example; not target values for every run):**

| Metric | Observed |
|---|---:|
| Configured PVs | 448 |
| Cryo / Microwave / ssRF / Magnet | 168 / 124 / 97 / 59 |
| Snapshot interval | 1.0 s |
| Last pre-stop snapshots | 2403 |
| Accepted events | 119,556 |
| Written events | 119,556 |
| Dropped events | **0** |
| Snapshot overruns | **0** |
| Maximum observed lateness | about 0.011 s |
| Session size at stop | about **93 MB** |

The archiver read **Cryo, Microwave and ssRF/NMR** values during live operation. Magnet supply was offline, so all 59 of its PVs were **`missing`**. The new Microwave SIG scope was not connected, so four `UNH:GEN:MW:SCOPE:*` PVs were **`missing`**. Low manometer had old/stale data. These do **not** mean the archiver ignored those configured fields.

### 9.3 Where the archive actually lives

Base directory:

**⬜ REFERENCE / OUTPUT OR PATH (NOT EXECUTABLE):**

```text
/home/utsav/UNH-Genesis-Control/local_data/archive/master/
```

**October 29 validated session:**

**⬜ REFERENCE / OUTPUT OR PATH (NOT EXECUTABLE):**

```text
/home/utsav/UNH-Genesis-Control/local_data/archive/master/
  20261029T164717Z_193a0400b9f3_21169/
    session.json
    2026-10-29/
      metadata.json
      genesis_master.csv
      genesis_master.jsonl
```

A different run creates a **different** timestamped session. Never hard-code the Oct 29 path as the destination for a new run; do not copy over or delete old sessions. `session.json` provides startup provenance such as git SHA/branch, CA address list and registry summary; **it did not contain the October 29 final event counts or an explicit `finished` field**. Current running-state counters are provided by `master-archive-status`; preserve the terminal report in the shift log if needed.

To locate the newest session **read-only**:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
BASE=/home/utsav/UNH-Genesis-Control/local_data/archive/master
find "$BASE" -mindepth 1 -maxdepth 1 -type d -printf '%T@ %p\n' | sort -nr | head -8
du -sh "$BASE"/* 2>/dev/null | tail -12
```

To view files inside the selected session, replace `<SESSION_NAME>` with the real directory name (type it explicitly):

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
SESSION=/home/utsav/UNH-Genesis-Control/local_data/archive/master/REPLACE_WITH_ACTUAL_SESSION_NAME
find "$SESSION" -maxdepth 2 -type f -printf '%P  %s bytes\n' | sort
```

**CSV layout:** snapshot rows contain `timestamp_utc`, `timestamp_local` plus **five columns per PV**: `|value`, `|quality`, `|age_s`, `|received_utc`, `|source_utc`. **JSONL layout:** individual `pv_update` objects include `pv`, `value`, `connected`, `received_utc`, `source_utc`. **Metadata JSON:** `schema_version`, `timezone`, `timestamp_standard`, `snapshot_interval_s`, and `pv_registry`.

Do not assume the archive stores physical safety truth. If a sensor goes offline after a successful read, its **last known value may remain nonblank but `quality=stale`** and its `age_s` grows. A stale `Mano_Status=OK` is **not** a live good status; a `missing` Magnet field is **not** zero magnet current.

### 9.4 Periodic quality/coverage check [CHECK]

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
./scripts/genesisctl.sh master-archive-status
ls -lt /home/utsav/UNH-Genesis-Control/local_data/logs/master-archive.log
```

The following compact Python diagnostic reads the **latest CSV snapshot from the newest session on the current UTC day**, without writing to the archive. It prints per-prefix freshness and missing counts; it intentionally **does not treat unchanged/stale values as newly verified physical data**. It finds a session by modification time, so double-check the printed directory before interpreting the output:

**🟦 EXPERT / READ-ONLY ANALYSIS OR CONNECTIVITY CHECK:**

```bash
python - <<'PY'
import csv
from collections import Counter, deque
from pathlib import Path
base = Path('/home/utsav/UNH-Genesis-Control/local_data/archive/master')
sessions = [p for p in base.iterdir() if p.is_dir()]
if not sessions:
    raise SystemExit('No master archive sessions found')
session = max(sessions, key=lambda p: p.stat().st_mtime)
csvs = sorted(session.glob('*/genesis_master.csv'))
if not csvs:
    raise SystemExit(f'No snapshot CSV under {session}')
path = max(csvs, key=lambda p: p.stat().st_mtime)
with path.open(newline='') as f:
    last = deque(csv.DictReader(f), maxlen=1)
if not last:
    raise SystemExit('Snapshot CSV has no rows')
row = last[0]
print('Session:', session)
print('Snapshot UTC:', row['timestamp_utc'])
counts = {}
for col, value in row.items():
    if not col.endswith('|quality'):
        continue
    pv = col[:-8]
    if pv.startswith('UNH:GEN:CRYO:'): group = 'Cryo'
    elif pv.startswith('UNH:GEN:MW:'): group = 'Microwave'
    elif pv.startswith('UNH:GEN:SSRF:'): group = 'ssRF/Fitter'
    elif pv.startswith('UNH:GEN:MAG:'): group = 'Magnet'
    else: group = 'Other'
    counts.setdefault(group, Counter())[value or 'blank'] += 1
for group, c in sorted(counts.items()):
    print(f'{group:14s} total={sum(c.values()):3d} quality={dict(c)}')
PY
```

**Oct 29 quality snapshot (16:50 UTC, historical):** Cryo `168`: 68 valid/100 stale; Microwave `124`: 9 valid/111 stale/4 missing; ssRF `97`: 7 valid/90 stale; Magnet `59`: all missing. Many stale values were status/constant PVs that did not emit frequent monitor updates. **Stale is not automatically disconnected**, and `valid` is not automatically a scientifically valid sensor result. For important channels compare actual value changes, source timestamps, quality, age and independent equipment status over a reasonable time window.

**Oct 29 live change check:** Over a 30-second comparison, 57 Cryo PV values, 5 Microwave PVs (mostly timestamps), and 4 ssRF/fitter values changed. NMR `P_FIT` and `LAST_TS` changed with accepted events. The Master Archiver had real data, but that alone did not certify a physical target polarization.

### 9.5 Archive-source diagnostics and mismatch troubleshooting

- If `configured_pvs` differs from **448** on the Oct 29/30 `4846a4d` baseline, inspect `archive/master_sources.yaml` and `session.json` **read-only**, record which source was added/omitted; don't silently repair a frozen checkout during a run.
- If `snapshots` remains zero after initial startup, inspect recorder process/PID, CA address list and `/home/utsav/UNH-Genesis-Control/local_data/logs/master-archive.log`. Verify no second owner.
- If events are zero but snapshots rise, PV monitors may all be unreachable. Check active Cryo/MW/ssRF IOC states and CA address configuration. **Do not** start all hardware just to satisfy event counts.
- If one subsystem records stale/missing, examine `|quality`, `|age_s` and **instrument** `COMM` first; distinguish a display/EPICS connection fault from a dead physical serial device.
- If `dropped_events` or `overruns` becomes nonzero, document values and preserve files; review disk space and overloaded services. Do **not** change snapshot frequency or archive code during the frozen run.
- `CA.Client.Exception: Virtual circuit disconnect` immediately after deliberately shutting down an IOC is expected. Repeated current-time disconnects **during** otherwise stable operation should be investigated by source port/process/PV, not ignored.

---

## 10. Streamlit online archive viewer [29-OCT RUN]

The viewer is a **read-only** browser of Master and subsystem archive files. It does not start/stop IOCs and does not substitute for live physical instrumentation. **Launch it after Master Archiver starts** and make sure it selects the correct new session rather than an older successful cooldown.

In Terminal 5:

**🟦 EXPERT / CHECK STREAMLIT PORT OWNERSHIP (IF NEEDED):**

```bash
ss -ltnp | grep ':8501 ' || true
```

**🟩 OPERATOR / START VIEWER (IN FOREGROUND):**

```bash
cd "$GENESIS_WT"
./scripts/genesisctl.sh archive-viewer
```

Then use the WSL/Windows browser (on the same host) at:

**⬜ REFERENCE / OUTPUT OR PATH (NOT EXECUTABLE):**

```text
http://localhost:8501
```

**Port 8501 is the observed/default launch expectation, not a guarantee**; if the viewer prints a different actual URL, use the printed address. If `8501` is already occupied, inspect the owner with `ss -ltnp` and `ps -p "$PID" ...` (after setting `PID` to the observed integer). Reuse the existing correct instance if it is the authorized viewer; **do not kill an unknown Streamlit/Python process to free the port**. Stop any stale viewer from its **own launching terminal** after confirming it is yours.

### Viewer acceptance checklist

1. Select the current session from `local_data/archive/master` (for Oct 29: `20261029T164717Z_193a0400b9f3_21169`).
2. Verify there are **448 configured PVs** in metadata (168 Cryo, 124 Microwave, 97 ssRF/NMR, 59 Magnet) for this frozen release.
3. Plot at least one live-varying **Cryo** channel (for example a connected temperature/flow), one **Microwave** timestamp/readback, and **ssRF fitter** `P_FIT`/`LAST_TS` when a legitimate accepted NMR event was produced.
4. Verify missing Magnet PVs are clearly marked missing, not shown as measured zero; also check stale low-manometer data and absent SIG scope fields.
5. If a plot looks flat, compare its quality and timestamps; **no variation in a constant status PV is not automatically an archiver defect**.
6. Confirm the data file size grows while the Master Archiver runs. After clean stop the session should remain readable.

If the viewer fails, **leave all physical subsystems and Master Archiver alone**. Check the viewer terminal traceback, its conda environment, the correct archive path and port. Try closing/reopening **only the viewer**, not the instrument owners or recording service. Do not write/change an archived CSV while an active recorder owns it.

---

## 11. Diagnostic commands, failure signatures, and recovery decision tree

Use this section when an operator sees a magenta PV, blank trend, stale value, missing instrument, busy port, or process conflict. **First decide whether the fault is GUI-only, EPICS/IOC, instrument/USB, or an unsafe physical condition.** A failed **safety** condition is never solved by removing software checks.

### 11.1 Universal read-only inventory [CHECK]

In Terminal 2, from `$GENESIS_WT`:

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
cd "$GENESIS_WT"
printf 'User=%s  branch=%s  SHA=%s\n' "$(id -un)" "$(git branch --show-current)" "$(git rev-parse --short HEAD)"
screen -ls
ps -eo user,pid,ppid,lstart,args --cols 340 | grep -Ei \
  '[p]hoebus|[p]ynmr_main.py|[m]aster_ioc.py|[i]oc_manager.py|[o]nline_fitter|[s]treamlit|[m]aster_service' || true
sudo ss -ltnp
./scripts/genesisctl.sh status
./scripts/genesisctl.sh master-archive-status
```

To inspect an **identified** PID, set the variable to the actual current PID first (the example `12345` is **not** an identified running process; change it):

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
PID=12345  # REPLACE with the actual current PID from ss/ps
ps -p "$PID" -o user,pid,ppid,stat,lstart,etime,args
readlink -f "/proc/$PID/cwd"
```

If `ps` returns nothing, the process may have exited; **do not** reuse an old PID. Identical PID numbers can be recycled; check owner, start time, command line and working directory before a managed action. **No PID printed in this document (`14103`, `19501`, `21169`, etc.) should ever be reused as a current control target.**

### 11.2 Quick fault-to-action matrix

| What you see | Verify first | Bounded solution / stop condition |
|---|---|---|
| Panopticon partly magenta before starting IOCs | Matching IOC started? CA address list? | Expected while offline. Start authorized subsystem, not a second GUI/owner. |
| WaveGen magenta, all other IOCs visible | Pi IOC on `192.168.1.24:5064`, correct `panopticon_live.ini` | Fully exit/restart **your** old Phoebus process if it cached pre-fix settings. |
| Blank history charts although Cryo values update | `genesis-hist`, CA `5130`, Trend History log | Start **only** Trend History if absent; confirm 37-PV history and backfill. |
| Cryo right flow `NaN`, bridge 5010 listening | `lsusb`, Teledyne symlink, `ttyACM0` | Bridge listening is not sensor available; physical USB inspection/reseat only when Keysight-safe. |
| Low manometer `/dev/ttyUSB0` missing | FTDI `0403:6015` in `lsusb`, recent `dmesg` | Known Oct 29 missing device / possible dead xHCI controller; defer Pi recovery while Keysight active. |
| `xHCI host controller not responding`, `HC died` | Pi `dmesg`, bus enumeration, active Keysight ownership | **STOP USB software recovery** while ssRF active; controlled hardware maintenance after safe shutdown. |
| `CRYO:SAFE:Safe_Fault=1` or `Safe_Permit=0` | Actual active reasons, Cryo operator, independent readbacks | Never bypass; confirm physical cryo state and laboratory procedure. |
| Microscope/SIG scope four PVs missing | `MW:SCOPE:*`, physical SIG attached? | Expected if new scope unused; record as unavailable, do not claim full Microwave scope acceptance. |
| Microwave UCA/VDI/robot disconnected | UCA power/voltage, VDI green, Pi `.23`, USB path, hardware owner | Authorized physical startup sequence; do not turn on RF for connectivity test. |
| Microwave RF status ambiguous | Explicit `RF=0`, `Supervisor RF=0`, `mw-rf-ready` and physical output | Never infer from `RF state=1` alone. |
| PyNMR opens, `8779` not listening | `genesis-ssrf-fitter` started? | Expected until **full ssRF** stack starts. Do not launch standalone fitter to fill port. |
| `accepted=False`, no fitter plot | PyNMR producer identity, switch mask/COMM, IPC, route epoch | Inspect logs; when authorized stop **ssRF** safely, then restart with current PyNMR PID. No spoof/bypass. |
| PyNMR `'NoneType' ... 'write'` | Completed event & accepted IPC separately | Known error not fixed in this branch; preserve log; do not assume data valid. |
| PyNMR only 64/640 sweeps | FPGA/network timeout | Stop acquisition in GUI; confirm DAQ/R&S/FPGA physically; do not accept partial spectra. |
| SMC/Keysight preflight fails | Safe physical switches/outputs and 10-ms period | Physically correct OFF baseline under operator supervision; rerun read-only check. |
| ssRF clock fail | Pi/Ubuntu time, Pi NTP service ownership | Authorized **pre-start** auto-clock script only; not during live runs. |
| Lock `Permission denied` or port `Address already in use` | `ls -l` lock, `ss`, `ps`, `screen`, account | Ownership/handoff issue. **Never** `sudo rm`, chmod/chown, `pkill` or start duplicate stack. |
| Magnet 59 missing PVs | Magnet intentionally offline? IOC `5131`? | Expected Oct 29; no power-up for archiving, ask magnet lead for authorization. |
| Master Archiver has zero events | Status, source IOCs, local CA, log | Verify configured sources and active producers, do not restart all hardware. |
| Master shows many `stale` values | `|age_s`, last received/source timestamps, update frequency | `stale` alone doesn't prove a disconnected device; never accept stale statuses as live safety. |
| Immediately after Master `stop`, status still `[UP]` | Same PID; watch status after graceful stop request | **Wait and recheck**; Oct 29 the process exited later. Don't kill or delete lock. |
| `CA.Client.Exception Virtual circuit disconnect` during shutdown | Same timestamp as IOC stop? | Expected when IOC intentionally disappears; persistent unexpected disconnects during run need investigation. |
| JavaFX warnings/hostname stack traces | Does GUI display? Are PVs actually connecting? | Oct 29 GUI still worked; investigate only when function is affected. |

### 11.3 Finding the real log and process, not guessing filenames

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
cd "$GENESIS_WT"
find logs -maxdepth 2 -type f -printf '%TY-%Tm-%Td %TH:%TM %p\n' 2>/dev/null | sort | tail -45
find /home/utsav/UNH-Genesis-Control/local_data/logs -maxdepth 1 -type f -printf '%p\n' 2>/dev/null | sort | tail -35
ls -lt logs/*fitter* logs/*trend* logs/*ssrf* 2>/dev/null | head -30
```

- Trend History log: `logs/trend_history_live.log` (manual `genesis-hist` launch).
- Ellie optional Magnet log: `logs/magnet_ips120_live.log` (manual `genesis-magnet` launch).
- Master Archiver log: `/home/utsav/UNH-Genesis-Control/local_data/logs/master-archive.log`.
- ssRF, Cryo and Microwave logs: name/date depend on the launched scripts; **discover with `find`/`ls` first**. Avoid assuming another branch's log name.
- PyNMR run: Terminal 3 printed logs; its event data/history live under `/home/utsav/jlab_pynmr` (profile referenced `data-d`, `deuteron_session`, `deuteron_history`). Do not delete these runtime files.
- Pi USB/kernel logs: obtained through the **read-only** SSH commands in §3; do not use a Python/USB direct probe if a running IOC already owns Keysight.

### 11.4 Port conflict: safe SOP

1. Print `sudo ss -ltnp | grep ':<PORT> '` and identify the listener **PID/user/program**. A local socket such as IPC may instead require `ss -xlpn`.
2. Use `ps -p "$PID" ...` (after setting `PID` to the observed integer) and inspect the specific account, process command, start time and project cwd. Check both local `screen -ls` and the remote Pi if the service lives there.
3. Identify its owner and intended role. If it is already the correct running service, **reuse it**; do not start a duplicate. If owned by another person/account, **stop** and arrange a safe handoff.
4. If it is a **previous partially started ssRF/Microwave/Cryo stack under your own account**, verify physical safe state and use its **matching managed safe-stop command**, then recheck listener/screens/PVs. A single partial `screen` is not grounds for a blanket kill.
5. If the service remains after its managed stop, **record logs and stop**; investigate the program-specific shutdown. Do not escalate to `kill -9`, `fuser -k`, `pkill -f python`, `screen -wipe`, mass process kill, raw USB reset, or deleting locks without a separate senior-operator recovery procedure.
6. For SSH failures, validate route and credentials (`ssh -o ConnectTimeout=5 ...`), avoid exposing private keys/passwords, and do not bypass host-key warnings. The Keysight Pi may use a dedicated SSH identity on the `utsav` account; the October 21 cross-account incident required correcting direct-IP key config **for another account**. No account changes are implied by this procedure.

---
## 12. Controlled shutdown — one subsystem at a time, Panopticon LAST [29-OCT RUN]

**Do not use a generic stop-all script.** The purpose is to prove hardware returns to a verified safe baseline **before** releasing software ownership, while still letting the Master Archiver observe shutdown transitions. An active cooldown, pressurized cryogenic system, high-field magnet motion, RF burn, or unsafe state **overrides this order**: retain the responsible instrumentation and follow the emergency/specialist procedure.

**Shutdown baseline before any stop:** Master Archive still running, Panopticon still open, all subsystem owners identified, and each physical subsystem operator present. If hardware cannot be positively made safe, **do not issue a stop merely to clear screens**.

### 12.1 Close Streamlit viewer FIRST

In Terminal 5, exit the Streamlit foreground process with `Ctrl+C`. Closing its browser tab alone may leave its server running. Check that its owned `8501` listener has exited; **do not kill somebody else's Streamlit process**.

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
ss -ltnp | grep ':8501 ' || echo '[INFO] No Streamlit listener on 8501'
```

Master Archiver must continue running at this stage.

### 12.2 ssRF: verified safe stop while Keysight/SMC/switch ownership is active [29-OCT RUN]

**Before** stop, confirm no authorized burn or RF activity is in progress; notify the ssRF operator. Read-only:

**🟩 OPERATOR / ROUTINE STATUS OR READINESS (READ-ONLY):**

```bash
cd "$GENESIS_WT"
./scripts/genesisctl.sh ssrf-physical-status
```

On Oct 29: SMC RF **False**, FM **False**, power `−120 dBm`; NMR mask `0`; RF mask `3`. The pre-stop script **did not directly probe Keysight USB** because the Pi IOC owned it. **Do not assume WaveGen OFF from the skipped probe**; managed stop will verify the post-release baseline. If burn active or output state unknown, use the approved controller **ABORT** under operator supervision, not a Unix kill.

Perform the supported managed stop:

**🟩 OPERATOR / SAFETY-GUARDED SSRF STOP:**

```bash
./scripts/genesisctl.sh ssrf-stop
```

**Expected Oct 29:** `[PASS] safe baseline confirmed`, `[PASS] Final ssRF software stack stopped`, then a physical **read-only post-stop probe** that verified Keysight DSO-X 3024T identity, **ARB** mode, **output=0**, **period=0.01**, **high=+0.5**, **low=−0.5**, along with SMC RF/FM OFF and RF switch disconnected. Pi Keysight IOC `genesis_ssrf_wgen` process exited; `8778` remained free. No general Pi reboot/reset needed.

**STOP condition:** any unsafe SMC RF/FM, WaveGen ON, wrong RF switch, or verification failure. Keep operator engaged; do not remove RF hardware/USB, terminate owner by force, or proceed to leave unattended.

Post-stop check (read-only):

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
screen -ls | grep genesis-ssrf || echo '[PASS] No ssRF screen sessions'
./scripts/genesisctl.sh ssrf-physical-status
```

### 12.3 PyNMR: stop acquisition, then close GUI [29-OCT RUN]

In Terminal 3, if sweeps are active, use the **PyNMR GUI's normal Stop** and allow the active acquisition to finish/clean up. Close the PyNMR window normally; keep Terminal 3 available to read cleanup messages. **After** the ssRF fitter has stopped is the validated order. A crash or segmentation fault should be recorded, not treated as a normal exit. No blanket Python process termination.

Read-only check from Terminal 2:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
ps -eo user,pid,stat,args --cols 300 | grep '[p]ynmr_main.py' || true
ss -ltnp | grep ':8779 ' || echo '[PASS] No protected NMR TCP receiver'
```

Oct 29: prior PyNMR PID `14103` exited; no remaining `pynmr_main.py`, port `8779` free. **Never use the old PID in a stop command.**

### 12.4 Microwave: safe state, managed stop, THEN physical disconnection [29-OCT RUN]

While the hardware remains connected and powered, verify:

**🟩 OPERATOR / ROUTINE STATUS OR READINESS (READ-ONLY):**

```bash
./scripts/genesisctl.sh mw-status
./scripts/genesisctl.sh mw-rf-ready
```

**Required safe readbacks:** `RF=0`, `Supervisor RF=0`, robot `CONTROL_MODE=0`, X busy/Y busy `0`, UCA ≈5 V. `mw-rf-ready` is read-only; its manual testing instructions are **not** authorization to transmit. If safe state fails, **do not** unplug USB or turn off the VDI/ UCA just to bypass the test—get the Microwave lead.

Then:

**🟩 OPERATOR / SAFETY-GUARDED MICROWAVE STOP:**

```bash
./scripts/genesisctl.sh mw-stop
```

Oct 29 managed stop explicitly set/read back Supervisor RF disable, source RF OFF, UCA safe VSET `5.0`, robot disabled; verified fresh safe-state readbacks; stopped supervisor, robot, Vaunix, UCA IOC, Microwave archive and manager sessions.

**Physical finishing sequence, after software safe stop [OPERATOR]:**

1. Confirm the UCA was at **5 V** during safe verification and source RF was OFF.
2. Disconnect the specified USB at the **VDI green box** (not another Pi/Keysight cable).
3. Turn **OFF the VDI-175T**, restore the required USB/input marker/sticker if applicable.
4. Turn **OFF the UCA power supply** as the final step under the Microwave SOP.

The Oct 29 operator confirmed unplugging cables and turning off VDI. The subsequent `mw-status` printed EPICS **timeouts for every PV**, which are **expected because the IOCs were stopped** and are **not a failure**. Verify session/process absence, not nonexistent PVs:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
screen -ls | grep -E 'mw_|ioc-manager|archiver' || echo '[INFO] No Microwave screen sessions shown'
```

**Important:** ensure the physical UCA power supply is also OFF; its final off state was a separate operator checklist item.


### 12.4.1 Independent Microwave Siglent scope IOC [30-OCT OBSERVED]

The new Siglent scope IOC may be launched separately from the Microwave control stack, potentially from a **different worktree**. The October 30 `mw-stop`/Microwave session cleanup left `mw_siglent_sds814x` running from `visualizer-integration`. This is **read-only scope monitoring**, not an RF safety controller, but confirm no necessary measurement/monitoring is being interrupted.

**🟦 EXPERT / IDENTIFY SCOPE OWNER AND PORTS:**

```bash
screen -ls | grep mw_siglent_sds814x || true
ps -eo user,pid,ppid,lstart,args --cols 300 | grep '[m]aster_ioc.py -s configs/commissioning/microwave_physical/siglent_sds814x' || true
ss -ltnp | grep -E ':(5095|5096)\b' || true
```

**🟧 SPECIALIST / CLOSE ONLY THE VERIFIED SCOPE SCREEN (AFTER MEASUREMENT HANDOFF):**

```bash
screen -S mw_siglent_sds814x -X quit
```

**🟦 EXPERT / CONFIRM THE PYTHON PROCESS AND SOCKETS HAVE ALSO EXITED:**

```bash
screen -ls | grep mw_siglent_sds814x || echo '[PASS] No local scope screen'
ps -eo user,pid,args --cols 300 | grep '[m]aster_ioc.py -s configs/commissioning/microwave_physical/siglent_sds814x' || true
ss -ltnp | grep -E ':(5095|5096)\b' || echo '[PASS] Scope IOC CA/PVA ports not listening'
```

**October 30 lesson:** immediately after the screen was closed, Python PID `39803` still appeared on `5095/5096`; a later check found it had exited. Do not mistake screen disappearance for port release, and do not reuse historical PIDs. If the *same identified child* persists, escalate to the subsystem owner for a targeted graceful termination, not blanket process killing.

### 12.5 Magnet IOC: Ellie's shutdown sequence, safely adapted [SOURCE; 30-OCT OBSERVED]

**Source cross-check:** [Ellie manual stop (§5)](https://raw.githubusercontent.com/ellie-long/UNH-Genesis-Control/dev-visualizer/STARTUP_INSTRUCTIONS.md) explicitly uses `screen -S genesis-magnet -X quit`. [Ellie automatic stop (`quit_session`)](https://raw.githubusercontent.com/ellie-long/UNH-Genesis-Control/dev-visualizer/scripts/live_laptop/stop.sh) then attempts a broad matching-process `SIGTERM`; that fallback is **not** automatically safe in a shared WSL/DAQ host and is not reproduced here. **Stop the actual `genesis-magnet` software monitor only after specialist release approval. There is no verified physical magnet power-down sequence in either Ellie source.**

**Never confuse release of a read-only IOC with safe ramp-down, heater transition, or powering off the magnet.** The IPS120 power system can remain energized while a monitor is stopped. A software stop is allowed **only after** the responsible Magnet operator independently confirms the supply is holding (not ramping), no switch-heater transition/abnormal alarm is active, and independent field/current/cryogenic safety monitoring remains in place. The current config is `write_enabled: false`; confirm that in the active worktree. If an operation is active or the state is unverified, **retain the Magnet IOC** and follow the Magnet specialist SOP; do not use the steps below.

**🟦 EXPERT / READ-ONLY MAGNET IDENTITY, LOG, AND PORT CHECK:**

```bash
cd "$GENESIS_WT"
grep -n 'write_enabled:' configs/magnet_physical/ips120/settings.yaml
screen -ls | grep genesis-magnet || true
ps -eo user,pid,ppid,stat,args --cols 300 | grep '[m]aster_ioc.py -s configs/magnet_physical/ips120' || true
ss -ltnp | grep -E ':(5131|5231)\b' || true
```

For a manual `genesis-magnet` screen, identify its actual process working directory with `readlink -f /proc/<CURRENT_PID>/cwd`, then inspect **that worktree's** `logs/magnet_ips120_live.log` (do not guess that `main/logs/` contains it). The Oct 30 Magnet IOC ran from `visualizer-integration` while subsequent commands ran from `main`. If `caget` is absent, do not infer safe state; independent front-panel confirmation remains required.

**🟧 SPECIALIST / ELLIE'S MAGNET MONITOR STOP (ONLY AFTER REQUIRED APPROVAL):**

```bash
screen -S genesis-magnet -X quit
```

**🟦 EXPERT / VERIFY MAGNET IOC EXIT:**

```bash
screen -ls | grep genesis-magnet || echo '[PASS] Magnet screen exited'
ps -eo user,pid,args --cols 300 | grep '[m]aster_ioc.py -s configs/magnet_physical/ips120' || true
ss -ltnp | grep -E ':(5131|5231)\b' || echo '[PASS] Magnet IOC ports released'
```

If the screen exits but the child process survives, do **not** infer completion. Verify PID/user/start time/exact command and owner, check the physical release authorization again, and escalate for a **targeted SIGTERM** if explicitly approved. Never use `kill -9`, generic `pkill`, `fuser -k`, or remove locks. The October 30 screen and port release is a **software observation only**; no physical Magnet control acceptance is implied. See §8.4 for the complete checklist.

### 12.6 Cryo: coldboard check, clean archive stop, 19 IOCs and Pi bridges [29-OCT RUN]

Confirm first: **cryostat warm/idle or independently monitored**, no critical operation will be left unsupervised, responsible cryo operator approves release, and the coldboard output can be *positively* read as OFF. The Oct 29 Cryo safety report had faults (`Safe_Permit=0`, `Safe_Fault=1`); this is not general authorization to stop a live cooldown.

Read-only status:

**🟩 OPERATOR / ROUTINE STATUS OR READINESS (READ-ONLY):**

```bash
./scripts/genesisctl.sh cryo-status
./scripts/genesisctl.sh master-archive-status
```

Then run the guarded stop **only after the conditions above are met**:

**🟩 OPERATOR / SAFETY-GUARDED CRYO STOP:**

```bash
./scripts/genesisctl.sh cryo-stop
```

**Observed Oct 29 sequence:** script checked `UNH:GEN:CRYO:COLDBOARD:Coldboard_Mode = 0`, passed **coldboard output OFF**, disabled Cryo archiver before shutdown, closed the **dedicated Cryo Phoebus** (not Panopticon), stopped **all 19** IOC sessions in its defined reverse order, then stopped both Pi serial bridges (`flow-right` and `manometer-low`). No hardware ON/OFF command was issued by Cryo stop.

**Safety guard:** the script uses `set -euo pipefail`, and its `coldboard_safe_to_release()` refuses to continue if the output PV is **ON or unreadable**. If it refuses, **STOP**; do **not** override with Ellie's older `--cryo-readonly-kill` mode or manually screen-kill safety/monitor IOCs. The fact that `COLDBOARD:COMM=0` appeared in an earlier status and stop later passed `Coldboard_Mode=0` is a reminder to check the **actual current readback**, not a stale overview.

Verify session count:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
screen -ls | grep genesis-cryo || echo '[PASS] No Cryo screen sessions'
```

Do not power off the shared Pi: it is the host for Keysight and Cryo bridges. The managed Cryo stop handled **bridges** only, not a Pi reboot.

### 12.7 Trend History IOC: routine operator stop after Cryo [29-OCT RUN]

**Trend History start/status/stop is 🟩 OPERATOR, not 🟦 EXPERT or 🟧 SPECIALIST.** This read-only plotting IOC runs as the dedicated `genesis-hist` screen, without a `genesisctl.sh` wrapper. The actual startup command is in §4. Keep the Unified Master Archiver distinct: the latter has its own `master-archive-stop` command and should continue recording shutdown until the end of the sequence.

**🟩 OPERATOR / CONFIRM TREND HISTORY IS RUNNING:**

```bash
screen -ls | grep genesis-hist || echo '[DOWN] Trend History already stopped'
```

**🟩 OPERATOR / STOP TREND HISTORY (ONLY THE OWNED SESSION):**

```bash
screen -S genesis-hist -X quit
```

**🟩 OPERATOR / VERIFY STOP:**

```bash
screen -ls | grep genesis-hist || echo '[PASS] Trend History screen stopped'
```

**🟦 EXPERT / ONLY IF THE SCREEN IS GONE BUT A PORT OR CHILD REMAINS:**

```bash
ss -ltnp | grep -E ':(5130|5230)\b' || echo '[PASS] History ports released'
ps -eo user,pid,ppid,args --cols 260 | grep '[m]aster_ioc.py -s configs/target_physical/trend_history' || true
```

If the history screen has already stopped, no further operator action is needed. An unexpectedly held port requires identity/owner diagnostics; **do not kill an unidentified IOC or another user's service**.

### 12.8 Unified Master Archiver: clean stop and finalization [29-OCT RUN]

**Leave Master Archiver running until after the subsystem stop sequence** so it records the availability transitions. Do not close it with a process kill. First capture last running status:

**🟩 OPERATOR / ROUTINE STATUS OR READINESS (READ-ONLY):**

```bash
./scripts/genesisctl.sh master-archive-status
```

Then request graceful finalization:

**🟩 OPERATOR / STOP MASTER ARCHIVER:**

```bash
./scripts/genesisctl.sh master-archive-stop
./scripts/genesisctl.sh master-archive-status
```

**Important October 29 behavior:** the first status immediately after `master-archive-stop` still said `[UP]` and `finished=false`; the stop printed `[STOP REQUESTED]`. A later read-only check showed `[DOWN]`, and the original PID no longer existed. This is **not** evidence that stop failed; it was asynchronous cleanup. **Do not call `kill`, delete the lock file, or repeat stop immediately.** Recheck after the process has had time to finish:

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
./scripts/genesisctl.sh master-archive-status
ps -eo user,pid,stat,args --cols 300 | grep '[a]rchive.master_service' || true
tail -30 /home/utsav/UNH-Genesis-Control/local_data/logs/master-archive.log
```

After `[DOWN]`, inspect the actual session directory selected from §9.3:

**🟦 EXPERT / READ-ONLY DIAGNOSTIC OR STATUS INSPECTION:**

```bash
BASE=/home/utsav/UNH-Genesis-Control/local_data/archive/master
find "$BASE" -mindepth 1 -maxdepth 1 -type d -printf '%T@ %p\n' | sort -nr | head -5
```

Ensure `session.json`, daily `metadata.json`, `genesis_master.csv`, `genesis_master.jsonl` remain and sizes are plausible. Oct 29 final session size was **93 MB**; no archive data was deleted. `session.json` contained source config/provenance, but not independently verified final stop counters; retain the last console status and shift notes if final counter provenance is required. If the master process stays alive abnormally, **stop here**, inspect logs and relevant source—**do not** kill/force-drop archive buffers.

### 12.9 Panopticon: close LAST [29-OCT RUN]

After the Master Archiver reports `[DOWN]`, and after all critical monitoring has been safely handed off, close Panopticon using its normal **Exit** command or close its owned application window. **Closing one display tab is not enough if Phoebus Java continues to run.** The remaining widgets will likely be magenta while the IOCs are stopped—this is the expected end-of-session state.

Verify:

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
ps -eo user,pid,stat,args --cols 340 | grep -Ei \
  '[p]hoebus|[g]enesis_panopticon.bob|[p]roduct-4.7.4-SNAPSHOT.jar' || true
screen -ls
./scripts/genesisctl.sh master-archive-status
```

On Oct 29 the Java GUI remained as PID **19501** after closing window(s), so the operator sent a **normal TERM signal to that exact verified own-account GUI process**. The immediate `ps` still showed it; a repeat after a few seconds showed it gone. **Do not reuse PID 19501.** If and only if **your own Panopticon** is the remaining exact process, you may use the same **guarded, PID-specific graceful** fallback (no `kill -9`):

**🟧 SPECIALIST / IDENTIFIED OWNED-PROCESS GRACEFUL FALLBACK:**

```bash
# Replace 12345 with the ACTUAL PID observed in the current ps output.
PID=12345
if [ "$(ps -p "$PID" -o user= | xargs)" = 'utsav' ] && \
   [ -r "/proc/$PID/cmdline" ] && \
   tr '\0' ' ' < "/proc/$PID/cmdline" | \
     grep -Fq 'configs/phoebus/panopticon_live.ini'; then
    kill -TERM "$PID"
    echo '[INFO] Graceful TERM sent to verified Panopticon'
else
    echo '[STOP] Process identity not verified; no signal sent'
fi
```

This does **not** operate on other Phoebus or Python processes. If it stays alive, examine it rather than escalating blindly. Once it exits, `ps` should show no Panopticon, `screen -ls` should show no local Genesis sessions, and Master status should be `[DOWN]`.

### 12.10 Final platform and archive check

**🟩 OPERATOR / ROUTINE STATUS + 🟦 EXPERT / OWNERSHIP DIAGNOSTICS (READ-ONLY):**

```bash
cd "$GENESIS_WT"
printf 'Branch: '; git branch --show-current
printf 'SHA: '; git rev-parse HEAD
git status --short
screen -ls
ps -eo user,pid,args --cols 300 | grep -Ei \
  '[p]ynmr_main.py|[g]enesis_panopticon.bob|[o]nline_fitter.epics_run|[m]aster_ioc.py|[i]oc_manager.py' || true
ss -ltnp | grep -E ':(5095|5096|5131|5231|5130|5230|5100|8779)\b' || true
./scripts/genesisctl.sh master-archive-status
find /home/utsav/UNH-Genesis-Control/local_data/archive/master \
  -mindepth 1 -maxdepth 1 -type d -printf '%T@ %p\n' | sort -nr | head -4
```

**Expected after a complete shutdown:** no local Genesis `screen` sockets, no residual Panopticon Java process, no unapproved IOC listeners, and Master `[DOWN]`. As observed on October 30, Siglent and Magnet may have **independently launched IOCs** beyond the normal Microwave/Cryo/ssRF stop scripts. Confirm both screen and Python child exit and relevant ports are free. The Git branch may legitimately be `main` or another approved worktree. **No software port/status check proves that Windows, remote Pi, cryostat, Magnet or RF equipment is physically powered off.**


### 12.11 Network interruption during ssRF/Microwave shutdown [30-OCT OBSERVED]

During October 30 shutdown, `ssrf-stop` refused at `cleanup_verify_sweep_disarmed` and `mw-stop` refused because its UCA `COMM=0` or other fresh-readback verification did not pass. Both **retained physical owners as designed**. The failure was not caused by running `main` instead of `visualizer-integration`: both pointed to `4846a4d`. The Ubuntu `eth0` route recovered, but distinct devices became reachable at different times: SMC `192.168.1.16:5025`, UCA `192.168.1.19:5025`, Vaunix Pi `.23`, Keysight Pi `.24` and Mini-Circuits switches `.8/.11`. Before recovery, logs reported `Network is unreachable`, `osiLocalAddr(): only loopback found`, stale UCA readings, and switch GET timeouts. **No Git modification, broad kill, Pi reboot, lock deletion, or port-force should be the first response.**

1. **🟧 SPECIALIST PHYSICAL CHECK:** At the apparatus, independently verify ssRF RF/FM/WaveGen OFF, sweep/burn stopped, the NMR and RF switch safe routes, Microwave RF OFF, robot idle and UCA in its prescribed shutdown baseline. If these are unknown, the correct decision is to retain owners and escalate.
2. **🟦 EXPERT NETWORK CHECK:** Differentiate loss of Internet/WAN from loss of the **local lab LAN**. `ip neigh FAILED` can be transient; a subsequent device-specific readback is more useful than assuming the state. Do not use ping alone as RF safety evidence.

**🟦 EXPERT / NETWORK, TCP AND CURRENT OWNER DIAGNOSTICS (READ-ONLY):**

```bash
ip -br addr show eth0
ip route get 192.168.1.24
for ip in 192.168.1.16 192.168.1.19 192.168.1.8 192.168.1.11 192.168.1.23 192.168.1.24; do
  echo "===== $ip ====="; ip neigh show "$ip"; ping -c 1 -W 1 "$ip" || true
done
python - <<'PYTCP'
import socket
for name, host, port in [
    ('SMC', '192.168.1.16', 5025),
    ('UCA', '192.168.1.19', 5025),
    ('Vaunix Pi SSH', '192.168.1.23', 22),
    ('Keysight Pi SSH', '192.168.1.24', 22),
]:
    try:
        with socket.create_connection((host, port), timeout=3):
            print('[REACHABLE]', name, host, port)
    except OSError as exc:
        print('[UNREACHABLE]', name, host, port, exc)
PYTCP
screen -ls
ss -ltnp | grep -E ':(5068|5071|5072|5073|5074|5090|5092|5100|8778|8779)\b' || true
```

3. **🟩 OPERATOR / RE-RUN FRESH STATUS AFTER NETWORK RECOVERY:** A TCP connection alone does not prove physical safety. The current SMC, switch and WaveGen states must be read, and Microwave `RF valid`/UCA `COMM` checked anew. In the October 30 record the SMC later reported `RF=False`, `FM=False`, `−120 dBm`, while the initial switch HTTP request timed out. A later `/MN?` request responded with device model text prefixed `-99 unknown command`; that showed network response, **not safe switch routing**.

**🟩 OPERATOR / REPEAT NORMAL SAFETY-VERIFIED STOP ONLY WHEN ALLOWED:**

```bash
cd "$GENESIS_WT"
./scripts/genesisctl.sh ssrf-physical-status
./scripts/genesisctl.sh mw-status
```

After independent safe-readback and switch-route verification, **in this order** and one command at a time:

**🟩 OPERATOR / PROTECTED ssRF STOP:**

```bash
./scripts/genesisctl.sh ssrf-stop
```

**🟩 OPERATOR / PROTECTED MICROWAVE STOP:**

```bash
./scripts/genesisctl.sh mw-stop
```

4. **🟦 EXPERT FINAL VERIFICATION:** Confirm no remaining screen/IOC owner for the stopped systems, no relevant listeners, Master Archiver state as intended, and physical equipment state independently. If either guarded stop still fails, **do not** use a generalized force-stop. Preserve its owner for a specialist recovery and document the actual failed verification. A request to release ports is not authorization to bypass physical checks.

---

## 13. Operator shift checklist (print or complete in Markdown)

**Shift date:** ______  **Operator:** ______  **Responsible cryo/RF/magnet leads:** ______  **Experiment / target state:** ______
**Branch/SHA:** ______  **Run notes/log location:** ______

### Before operation

- [ ] Current experiment authorization and independent safety monitoring confirmed.
- [ ] Ubuntu user `utsav`; selected approved worktree (`main` by default), SHA and historical validation tag recorded; **no Git changes during live hardware operation**.
- [ ] All prior device/port owners identified; no uncontrolled duplicate PyNMR, fitter, Pi bridge or IOC.
- [ ] Pi `192.168.1.24` Keysight and Cryo cables inspected; USB serial devices listed, missing devices recorded.
- [ ] Pi `192.168.1.23` Microwave Vaunix/green-box USB and UCA/VDI physical readiness checked.
- [ ] Keysight/SMC/ssRF NMR/RF switches and terminations verified **safe**; output OFF.
- [ ] Magnet explicitly classified: **offline** OR specialist-authorized **read-only**; current ramp/heater state and independent monitoring handoff documented.

### Startup and live acceptance

- [ ] Panopticon opened with approved `panopticon_live.ini`; linked displays work.
- [ ] Cryo 19/19 sessions assessed, archiver running, alarm/safety state recorded; no unsafe activity bypassed.
- [ ] Trend History `genesis-hist`/5130 live and Panopticon history plots populate.
- [ ] Microwave supervised startup completed; UCA/VDI/USB correct, RF OFF, robot disabled, relevant archiver active.
- [ ] One PyNMR Deuteron GUI/source, current PID recorded: ______.
- [ ] ssRF physical preflight passed; Keysight 10 ms, ±0.5 V, output OFF; clock sync script performed once if authorized.
- [ ] Full ssRF `operator` mode: all 8 sessions, protected fitter, fit/recipe and interlocks; RF not armed.
- [ ] Authenticated NMR event accepted (`[GENESIS IPC] accepted=True`), new fitter timestamps and spectrum plots; bad/absent target explicitly labeled.
- [ ] Optional Magnet read-only IOC connected **only** with specialist approval, otherwise archived as missing.
- [ ] Master Archiver 448 configured PVs (for frozen release); snapshots/events increasing, dropped/overrun recorded.
- [ ] Streamlit viewer shows **current** session and PV quality; Cryo/MW/ssRF plot samples inspected.
- [ ] Known issues logged: USB low manometer, field/coldboard, SIG scope, fit, Magnet, alarms, Pi clock/NTP and any recovered LAN/EPICS readback failures.

### Shutdown and transfer

- [ ] Streamlit process closed; Master Archiver still running.
- [ ] ssRF safe stop completed; independent Keysight readback output OFF, 10 ms; SMC OFF, RF switch disconnected.
- [ ] PyNMR acquisition stopped normally; GUI/process and TCP receiver ended.
- [ ] Microwave safe stop passed, THEN physical USB disconnected, VDI OFF, UCA OFF.
- [ ] Optional Magnet physical HOLD/no-ramp/heater state and independent monitoring **confirmed by Magnet lead**; read-only IOC released only when approved, or explicitly handed off with monitoring retained.
- [ ] Cryo coldboard read **OFF** and independent safety monitoring confirmed; 19 IOCs/bridges stopped cleanly.
- [ ] Trend History `genesis-hist` stopped.
- [ ] Master Archive stop requested; later confirmed `[DOWN]`, files preserved and readable.
- [ ] Independent Siglent scope IOC stopped after approved measurement handoff; screens **and child processes** exited; ports `5095/5096` released.
- [ ] Magnet IOC `5131/5231` released **only if** specialist sign-off permits monitoring shutdown.
- [ ] Panopticon closed **LAST**, no stale Java process, no unaccounted local Genesis `screen` sessions or listeners.
- [ ] New archive session path, final observed counters and deviations documented; **no experimental files deleted**.

**Archive session:** ______________________________
**Last snapshot/event counters:** __________________
**Events dropped / overruns:** _____________________
**Unresolved faults and equipment still energized:** ____________________________________________
**Operator handoff / sign-off:** ____________________

---

## 14. Frozen release artifacts, reference documents, and maintenance boundaries

### 14.1 Files to read locally

| File | Purpose |
|---|---|
| `docs/PANOPTICON_UBUNTU_HANDOFF.md` | October 26 offline and October 29 live integration record |
| `configs/phoebus/panopticon_live.ini` | CA addresses, includes `192.168.1.24:5064` |
| `gui/genesis_panopticon.bob` | Generated primary GUI |
| `tools/make_panopticon.py` | Panopticon generator; engineering work only, **not** a startup step |
| `scripts/genesisctl.sh` | Supported subsystem start/stop/status commands |
| `scripts/cryo_physical.sh` | Cryo owned IOCs, guarded safe stop and Pi bridges |
| `scripts/microwave_physical.sh` | UCA, robot, source and safe-start/stop |
| `scripts/ssrf-start-auto-clock.sh` | **State-changing** Pi clock sync and complete ssRF startup |
| `scripts/check_ssrf_physical_readonly.sh` | Hardware/clock/readback inspection |
| `configs/target_physical/trend_history/` | Trend History IOC profile on port 5130 |
| `configs/magnet_physical/ips120/settings.yaml` | **Default read-only** Magnet physical profile |
| `magnet_ioc/PORTING.md` | Magnet implementation and safety reference |
| `scripts/live_laptop/start.sh` | Historical Ellie startup source. Do not run wholesale; may enable magnet hardware writes |
| `scripts/live_laptop/stop.sh` | Historical Ellie Magnet software-close source (`quit_session genesis-magnet ...`); **do not run wholesale** or copy broad `pgrep -f` kill logic |
| `docs/operations/MASTER_ARCHIVER_OPERATOR.md` | Archiver operator guide |
| `archive/master_sources.yaml` | PV registry / subsystem sources |
| `docs/nmr_dual_source_commissioning.md` | Historical protected NMR route/IPC; **not** permission to use LabVIEW source |
| `docs/ssrf_pynmr_session_2026_10_16.md` | PyNMR/fitter predecessor validation |

All relative file paths refer to **`$GENESIS_WT`**, the current approved worktree, not an assumed feature checkout. A different source SHA requires release review. Discover file paths with `git ls-files` if needed; do not silently substitute old runtime logs from another worktree.

### 14.2 What was **not** physically validated on October 29

- Magnet supply power/control/ramps/persistent-switch sequence and its 59 live PV readbacks.
- Connected low-manometer FTDI or recovery of the failed Pi xHCI controller.
- New SIG oscilloscope acquisition (four Microwave scope PVs missing).
- Complete single/continuous ssRF physical burn sequence, fit-gated/ungated authorization, and target/RF acceptance.
- Scientifically valid target polarization from connected target stick; the tested spectra were noise/invalid polarization.
- All cryogenic safety interlocks as safety-certified controls; several faults and alarm statuses were present.
- Restore of Pi NTP after the auto-clock startup (check with Pi maintainer during post-run maintenance).
- Production multi-user (`sliferlab`) ownership handoff for this exact working tree.

These are explicit **open items**, not reasons to bypass safeguards. Do not claim full-system operational acceptance merely because Panopticon or an archiver works.

### 14.3 Maintaining this runbook in `main` or another approved worktree

This document may be maintained on the **current `main` worktree** or another specifically approved development worktree. Editing it is a **documentation task** and should take place with Genesis hardware safely stopped. Do **not** change hardware control code, merge branches or force-push as part of deploying this Markdown file. Preserve the immutable historical October 29 validation tag and distinguish subsequent documentation-only commits from physically accepted tests.

**🟦 EXPERT / REVIEW CURRENT WORKTREE BEFORE DOC UPDATE (READ-ONLY):**

```bash
cd "$GENESIS_WT"
test "$(id -un)" = 'utsav' || exit 1
printf 'Branch=%s SHA=%s\n' "$(git branch --show-current)" "$(git rev-parse --short HEAD)"
git status --short
git diff -- docs/operations/GENESIS_PANOPTICON_OPERATOR_RUNBOOK.md
```

After copying/reviewing the new Markdown, inspect `git diff --check` and the document-specific diff. A **separate, explicitly authorized Git commit/push** can publish the documentation later; it is not part of normal instrument operations. Do not move the October 29 tag or claim that a docs-only commit constitutes a new RF/Magnet hardware validation.

### 14.4 Provenance, October 30 update and known limits

The original runbook was compiled from the October 29 startup/diagnostic/shutdown transcript, October 26/29 handoff notes, operator cheatsheet, and selected Ellie/Prof. Long startup-script, Magnet and PyNMR excerpts. This revision explicitly cross-checked Ellie's publicly accessible `dev-visualizer` `STARTUP_INSTRUCTIONS.md` (§5) and `scripts/live_laptop/stop.sh` to distinguish her actual Magnet IOC close from an undocumented physical magnet shutdown; it also classifies Trend History start/status/stop as routine operator actions. This edition additionally incorporates the **October 30** user-supplied Git promotion and successful `main` run report, network-loss/failed-safe-stop console output, read-only status checks, manual Magnet/Siglent screen release, and final local port/screen/archiver verification. The October 30 output **does not independently show physical Magnet HOLD/heater verification, all hardware RF-safe readbacks at final exit, or a new complete burn acceptance**. No hardware was operated or Git branch changed by authoring this Markdown. Source-derived commands and profiles should be rechecked in the actual live worktree before use.

**Operator principle:** Prefer guarded `genesisctl` status/start/stop, exact identity/port/log diagnostics, fresh physical device readbacks and a responsible human check over deletion of locks, clock/USB resets, force-killing owners, or unsafe RF/Magnet control. A clear port and a visible GUI are neither physical safety evidence nor hardware acceptance.
