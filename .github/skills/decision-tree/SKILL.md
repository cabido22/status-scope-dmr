---
name: decision-tree
description: "Status Scope decision trees and scenario knowledge. Use when: classifying a hang or error scenario, determining which plugins to run for a given symptom, understanding hung_with_mc vs shutdown_without_mc vs timeout classification, choosing adaptive capture mode, correlating CBO/SCA/HAMVF/HIOP timeout path, injecting a scenario for testing, or verifying the correct classification was produced."
---

# Status Scope Decision Trees

Reference document for all silicon debug scenarios, their classification logic,
and what actions/captures each scenario triggers.

---

## 0. All Scenarios — Quick Reference

Use this table as the first lookup for any symptom. Find the matching row, then read
the detailed decision tree section below for the full classification logic.

| Symptom | Section | Expected Classification | Plugins to Run | Agent Action |
|---------|---------|------------------------|----------------|--------------|
| Core ROB tail not advancing + MCA bank set | §1 | `hung_with_mc` | `core_analyzer`, `mca` | Capture watch windows (FSCP/ROB/SQ/MOB), decode MCA banks |
| Core ROB tail not advancing, no MCA | §1 | `hung_no_mc` | `core_analyzer` | Capture status tables; investigate upstream stall |
| Core INACTIVE + MCA set | §1 | `inactive_with_mc` | `core_analyzer`, `mca` | Collect MCA data; check if MC caused power-down |
| Core INACTIVE + no MCA + shutdown detected | §1 | `shutdown_without_mc` | `core_analyzer` | Dump FSCP regs, CRS, GPSB, PSMB; check punit sequence |
| Core ROB tail corrupted / unreadable | §1 | `corrupted` | `core_analyzer` | Run TAP reset → forcereconfig → refresh → re-classify |
| HW timeout, CBOs not quiesced, READ→memory | §2 | `HW_HANG` SCA stall | `timeout`, `sca` | Dump SCA TOR/RBT/WCT entries |
| HW timeout, CBOs not quiesced, READ→MMIO | §2 | `HW_HANG` HIOP stall | `timeout`, `hiop` | Snapshot HIOP ITC pending completions |
| HW timeout, CBOs not quiesced, WRITE→memory | §2 | `HW_HANG` MC write stall | `timeout`, `mc` | Capture MC scheduler state |
| HW timeout, CBOs not quiesced, CPL waiting | §2 | `HW_HANG` HAMVF stall | `timeout`, `hamvf` | Read HAMVF mailbox data |
| HW timeout, all CBOs quiesced | §2 | `HW_HANG` upstream | `timeout` | Check UBOX scratch; look upstream of local fabric |
| PCIe link down after reset | §3 | `HW_ERR` link down | `pcie` | Check LNKSTA DLLLA bit, LTSSM state |
| PCIe AER uncorrectable error | §3 | `HW_ERR` AER | `pcie`, `error` | Decode AER UES/UESVR, check AERUEMSK |
| Correctable ECC (CE) | §4 | `HW_MCE` CE | `ddrphy`, `mc`, `mca` | Read MC_STATUS CE bit, PHY margin data |
| Uncorrectable ECC (UCE) | §4 | `HW_MCE` UCE | `mc`, `mca`, `memory` | Decode MCA bank, isolate failing DIMM |
| Thermal throttle / FIVR rail issue | §5 | `HW_ERR` power | `fivr`, `pm_imh`, `telemetry_oob` | Read throttle counters, SVID power states |
| Punit sequencer stuck | §5 | `HW_HANG` punit | `pm_imh` | Check WP/SVID state machine, PMON counters |
| UCIe PHY training failure | §6 | `HW_ERR` D2D | `uciephy`, `ula` | Read PHY training result, per-lane margin |
| D2D link down | §6 | `HW_ERR` D2D | `dda`, `ula` | Check link layer state, flow-control credits |

---

## 1. Core Analyzer — ROB Tail Decision Tree

Entry point: `core_analyzer_plugin.py` → `analyze()` reads ROB tail per core.

```
For each core discovered by adapter.discover_cores():
│
├─► ROB Tail = HUNG
│       ├─► Has MCA bank set?
│       │       YES → Classification: hung_with_mc
│       │             Captures: Watch Windows (state_dump), SQ slice check
│       │             Plugins invoked: mca_collector, ww_report_builder
│       │
│       └─► No MCA
│             Classification: hung_no_mc
│             Captures: Status tables only (no watch windows)
│
├─► ROB Tail = ADVANCING
│       └─► Has MCA bank set?
│             YES → MCA data collected
│             Plugins invoked: mca_collector
│
├─► ROB Tail = INACTIVE
│       ├─► Has MCA bank set?
│       │       YES → Classification: inactive_with_mc
│       │             Captures: MCA data
│       │
│       └─► No MCA + Shutdown detected?
│             YES → Classification: shutdown_without_mc
│             Captures: FSCP registers, CRS dump, GPSB dump, PSMB dump
│             Reporters: register_report_builder (CRS/GPSB/PMSB)
│
└─► ROB Tail = CORRUPTED
      Classification: corrupted  (or recovery_* after recovery)
      Recovery sequence:
        1. reset_tap
        2. forcereconfig
        3. refresh_itp / refresh_sv
        4. refresh_debug_core
        5. Re-classify
```

### Insight output per classification

| Classification | Insight Type | Priority |
|----------------|-------------|----------|
| `hung_with_mc` | `HW_HANG` + `HW_MCE` | CRITICAL |
| `hung_no_mc` | `HW_HANG` | HIGH |
| `inactive_with_mc` | `HW_MCE` | HIGH |
| `shutdown_without_mc` | `HW_UNEXPECTED_POWERDOWN` | HIGH |
| `corrupted` | `HW_ERR` | MEDIUM |

---

## 2. Timeout Plugin — CBO/SCA/HAMVF/HIOP Decision Tree

Entry point: `timeout.py` → `analyze()` → `_initialize_fabric()` → decision tree.

```
Step 1: Read quiesce status for all CBOs
│
├─► All CBOs quiesced?
│       YES → Check UBOX scratch register
│             → Insight: upstream hang (no local transaction stuck)
│
└─► Non-quiesced CBO found → Dump TOR entries via mailbox
        │
        ├─► TOR entry: READ request
        │       ├─► Target = MEMORY
        │       │       → Call sca_plugin: SCA TOR/RBT/WCT non-empty?
        │       │               YES → Insight: HW_HANG "Write to memory stuck in SCA"
        │       │               NO  → Insight: HW_HANG "Read pending at memory controller"
        │       │
        │       └─► Target = MMIO / IO
        │               → Call hiop_plugin: ITC snapshot
        │                       Pending ITC? → Insight: HW_HANG "Read waiting for HIOP completion"
        │
        ├─► TOR entry: WRITE request
        │       Target = MEMORY
        │               → Insight: HW_HANG "Write stall at memory controller"
        │
        └─► TOR entry: CPL (completion) waiting
                → Call hamvf_plugin: mailbox data
                        HAMVF not quiesced? → Insight: HW_HANG "Completion stuck in HAMVF"
```

### Adaptive scan selection after timeout classification

| Classification | Additional scan |
|---------------|----------------|
| SCA memory stall | SCA scan added |
| HIOP completion stall | HIOP ITC scan added |
| HAMVF completion stall | HAMVF mailbox scan added |
| MC write stall | MC scheduler scan added |

---

## 3. PCIe Scenario Map

| Symptom | Plugins to run | Key registers to check |
|---------|---------------|------------------------|
| Link down after reset | `pcie` | LNKSTA (DLLLA bit), LTSSM state |
| AER uncorrectable error | `pcie`, `error` | AER UES, UESVR, AERUEMSK |
| Gen-6 speed degraded | `pcie` | LnkCap2, LnkSta2, current link speed |
| Device not enumerated | `pcie`, `iommu` | Config space, BDF scan |
| IOMMU page fault | `iommu` | FLTSTS, INVQ occupancy |
| CXL device offline | `cxl`, `pcie` | CXL.io port state, HDM decoder |

---

## 4. Memory / ECC Scenario Map

| Symptom | Plugins | What to look for |
|---------|---------|-----------------|
| Correctable ECC errors | `ddrphy`, `mc`, `mca` | MC_STATUS[63] CE bit, PHY margin data |
| Uncorrectable ECC | `mc`, `mca`, `memory` | MC_STATUS UCE bit, MCA bank decode |
| DDR training failure | `ddrphy`, `memory` | PHY training result regs, SPD data |
| DIMM not detected | `memory` | SPD topology, slot population |
| Memory throttling | `mc`, `fivr`, `telemetry_oob` | Thermal throttle events, SVID |

---

## 5. Power / Voltage Scenario Map

| Symptom | Plugins | Key indicators |
|---------|---------|----------------|
| Thermal throttle | `fivr`, `pm_imh`, `telemetry_oob` | Throttle event counters, SVID power states |
| FIVR rail failure | `fivr` | Rail enable status, voltage readback |
| Punit sequencer stuck | `pm_imh` | WP/SVID state machine, PMON counters |
| S3M boot failure | `s3m` | Boot status register, FW version |

---

## 6. D2D / UCIe / Fabric Scenario Map

| Symptom | Plugins | Key indicators |
|---------|---------|----------------|
| UCIe PHY training fail | `uciephy`, `ula` | PHY training result, per-lane margin |
| D2D link down | `dda`, `ula` | Link layer state, flow-control credits |
| IOSF sideband stall | `iosf`, `sideband`, `iosfsb2ucie` | Credit counters, message queue occupancy |
| UBR quiesce timeout | `ubr` | UBR quiesce status, NIP/UFI credit counters |

---

## 7. Integration Test Scenarios

These are the canonical scenarios used for integration testing:

| Scenario ID | Description | Inject via | Expected classification |
|-------------|-------------|-----------|------------------------|
| `TC-HANG-MC` | Core hang with MCA | Simics hang injection + MC injection | `hung_with_mc`, CRITICAL insight |
| `TC-HANG-NOMC` | Core hang no MCA | Simics hang injection only | `hung_no_mc`, HIGH insight |
| `TC-SHUTDOWN` | Unexpected power-down, no MCA | Simics power event | `shutdown_without_mc`, HIGH insight |
| `TC-TIMEOUT-SCA` | CBO TOR stuck at SCA | CBO TOR freeze + SCA non-empty | HW_HANG timeout SCA |
| `TC-TIMEOUT-HIOP` | CBO TOR stuck at HIOP | CBO TOR freeze + HIOP ITC pending | HW_HANG timeout HIOP |
| `TC-PCIE-LINKDOWN` | PCIe link down | Simics port disable | AER insight, LNKSTA check |
| `TC-ECC-CE` | Correctable ECC error | MC error injection | MC CE insight, ddrphy tables |
| `TC-PUNIT-STUCK` | Punit sequencer stuck | Simics PM state hold | pm_imh HW_HANG insight |
