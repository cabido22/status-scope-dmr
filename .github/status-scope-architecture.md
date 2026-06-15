# Status Scope — Architecture Document

## 1. Purpose

Status Scope is a silicon debug framework used across Intel silicon products.
It orchestrates a set of **IP plugins** to collect, decode, and correlate hardware state from a live
or post-silicon platform (IPC session, Simics, crashdump) and produces a structured HTML report
with insights and actionable recommendations.

---

## 2. Source Trees

All logic lives across three directories that must be treated as a single system.
Paths are configured via environment variables:

| Layer | Environment Variable | Role |
|-------|----------------------|------|
| **Product ip_plugins** | `STATUS_SCOPE_PRODUCT_ROOT` | IP-specific analyzers, `config.json`, README |
| **Bigcore core_analyzer** | `STATUS_SCOPE_BIGCORE_ROOT` | Cross-project core-level analysis framework |
| **pysvtools framework** | `STATUS_SCOPE_PYSVTOOLS_ROOT` | Runner, collectors, base classes *(read-only)* |

---

## 3. Plugin Model

### 3.1 IpPlugin Base Class (`pysvtools/status_scope/ip_plugin.py`)

Every analyzer plugin subclasses `IpPlugin` and implements:

```
pre_execution(ctx)   → claim resources, set adaptive capture requirements
analyze(ctx)         → collect data, populate shelf with DataFrames / insights
```

Plugin lifecycle:
```
runner.run() → discover plugins via config.json → pre_execution (all) → analyze (all) → post_processors → summarizer
```

### 3.2 Plugin Categories

| Category | Location | Examples |
|----------|----------|---------|
| Product ip_plugins | `$STATUS_SCOPE_PRODUCT_ROOT/ip_plugins/` | pcie, mc, mca, timeout, mesh, sca … |
| Bigcore project plugins | `$STATUS_SCOPE_BIGCORE_ROOT/core_analyzer/project_plugins/` | novalake_core_analyzer |
| Post-processors | registered in config.json post_processors | crashdump, known_sightings, summary |
| pysvtools built-ins | `$STATUS_SCOPE_PYSVTOOLS_ROOT` (read-only) | openipc, python_packages, state_dump_raw |

### 3.3 config.json Structure

```json
{
  "analyzers":    { "socket": { "all": { "all": ["module.path:ClassName", ...] } } },
  "post_processors": { ... },
  "collectors":   { "socket": { "all": { "all": { "state_dump": {...} } } } },
  "groups":       { "socket": { "all": { "all": { "imh": [...], "coherency": [...] } } } }
}
```

Groups allow partial runs: `status_scope.run(analyzers=["imh"])` runs all 20 IMH plugins.

---

## 4. Runtime Pipeline

```
pythonsv IPC session active
        │
        ▼
status_scope.run(collectors=["namednodes"], analyzers=["timeout"])
        │
        ├─► CollectorManager.collect()       ← gathers raw HW state via namednodes
        │
        ├─► IpPlugin.pre_execution()         ← all plugins declare what data they need
        │       (adaptive capture decisions happen here)
        │
        ├─► IpPlugin.analyze()               ← each plugin reads collected data,
        │       builds DataFrames, writes to shelf                     produces insights
        │
        ├─► PostProcessors.run()             ← crashdump decode, known_sightings match
        │
        └─► Summarizer.run()                 ← product Summarizer aggregates → HTML report
```

### 4.1 Adaptive Capture Modes

| Mode | Value | Behaviour |
|------|-------|-----------|
| Dynamic | 1 (default) | Captures only structures that are active/non-empty |
| Max | 0 | Captures everything regardless of state |
| Min | 2 | Registers only — no watch windows or state dumps |

---

## 5. Report Structure

Reports are stored in a Python `shelve` database. All `DataFrame` objects stored with
the key prefix `report_table:` are auto-rendered as HTML tables.

### 5.1 Top-level Sections

| Section | Shelf key pattern | Content |
|---------|-------------------|---------|
| Insights | `insights` | List of `Insight(type, priority, message, recommendation)` |
| Status Tables | `report_table:basic_check` etc. | Per-plugin DataFrames |
| Watch Windows | `report_table:watch_windows/S#M#C#/<ww_name>` | FSCP, ROB, SQ, MOB captures |
| Register Dumps | `report_table:register_dumps/S#M#/<module>_crs\|gpsb\|psmb` | CRS/GPSB/PSMB text |
| Summary | `report_table:summary` | Executive aggregation |

### 5.2 Insight Types & Priorities

```
HW_HANG              CRITICAL / HIGH
HW_MCE               CRITICAL / HIGH / MEDIUM
HW_UNEXPECTED_POWERDOWN  HIGH
HW_ERR               HIGH / MEDIUM
```

### 5.3 Opening the Report

```python
# After status_scope.run():
sv.report()                      # opens HTML in browser
sv.report(plugin="timeout")      # opens specific plugin section
```

---

## 6. Decision Trees — Scenario Map

### 6.1 Core Analyzer (ROB Tail Check)

| ROB Tail | Has MC? | Shutdown? | Classification | Triggered Captures |
|----------|---------|-----------|----------------|--------------------|
| Hung | Yes | — | `hung_with_mc` | Watch Windows (state_dump), SQ slice check |
| Hung | No | — | `hung_no_mc` | Status tables only |
| Advancing | Yes | — | `advancing_with_mc` | MCA data |
| Inactive | Yes | — | `inactive_with_mc` | MCA data |
| Inactive | No | Yes | `shutdown_without_mc` | FSCP regs, CRS dump, GPSB dump, PSMB dump |
| Corrupted | — | — | `corrupted / recovery_*` | reset_tap → forcereconfig → refresh → re-classify |

### 6.2 Timeout Decision Tree (CBO/SCA/HAMVF/HIOP)

```
Non-quiesced CBO found?
├── Yes → Dump TOR entries
│         ├── Read request → target memory → stuck in SCA?  → HW_HANG: SCA memory stall
│         ├── Read request → target MMIO  → stuck in HIOP? → HW_HANG: HIOP completion stall
│         ├── Write request → target memory               → HW_HANG: MC write stall
│         └── CPL waiting → HAMVF not quiesced           → HW_HANG: HAMVF completion stall
└── No  → All CBOs quiesced → check UBOX scratch          → HW_HANG: upstream hang
```

### 6.3 PCIe Scenario Map

| Symptom | Plugin | Key Registers |
|---------|--------|---------------|
| Link down after reset | `pcie` | LNKSTA, LTSSM state |
| AER uncorrectable error | `pcie` + `error` | AER UES, UESVR |
| Gen-6 degraded | `pcie` | LnkCap2, LnkSta2 |
| IOMMU fault | `iommu` | FLTSTS, INVQ |

### 6.4 Memory / ECC Scenario Map

| Symptom | Plugins | Key Registers |
|---------|---------|---------------|
| Correctable ECC | `ddrphy` + `mc` + `mca` | MC_STATUS, PHY margin |
| Training failure | `ddrphy` + `memory` | PHY training result regs |
| DIMM not detected | `memory` | SPD data, topology |

---

## 7. Agent & Skill Architecture

### 7.1 Skills

| Skill | File | Purpose |
|-------|------|---------|
| `decision-tree` | `.github/skills/decision-tree/SKILL.md` | All hang/error scenarios, what triggers what captures |
| `status-scope-report` | `.github/skills/status-scope-report/SKILL.md` | HTML report structure, shelf keys, how to parse/navigate |
| `pythonsv` | `.github/skills/pythonsv/SKILL.md` | How to run PythonSV, IPC sessions, namednodes, status_scope.run() |

### 7.2 Agents

| Agent | File | Skills Used | Purpose |
|-------|------|-------------|---------|
| `status-scope-developer` | `.github/agents/status-scope-developer.agent.md` | all 3 | Code enhancement — create, fix, extend plugins and framework |
| `status-scope-integrator` | `.github/agents/status-scope-integrator.agent.md` | all 3 | E2E testing — scenario injection, simulation, integration validation |

### 7.3 Skill → Agent Mapping

```
decision-tree  ──► developer  (understands what the code must handle)
               ──► integrator (knows what scenarios to inject and verify)

status-scope-report ──► developer  (validates report output of new code)
                    ──► integrator (verifies correct report sections appear)

pythonsv       ──► developer  (runs the code to validate changes)
               ──► integrator (primary tool for running all E2E tests)
```

---

## 8. Development Workflow (status-scope-developer)

1. Read `ip_plugin.py` base class to understand interface contracts
2. Look at existing plugin in `ip_plugins/` for pattern reference
3. Implement `pre_execution()` — declare adaptive captures
4. Implement `analyze()` — collect data, build DataFrames, add insights
5. Register in `config.json` under the correct group
6. Run via PythonSV: `status_scope.run(analyzers=["new_plugin"])`
7. Open report and validate output sections

---

## 9. Integration Testing Workflow (status-scope-integrator)

1. Identify the scenario to test (use `decision-tree` skill)
2. Set up the simulation/injection (Simics hang injection, MC injection script)
3. Connect PythonSV IPC session (use `pythonsv` skill)
4. Run the relevant plugin group or full status_scope run
5. Open report (use `status-scope-report` skill) and validate:
   - Correct insight type and priority generated
   - Correct tables populated
   - Correct watch windows / register dumps captured
6. Compare against expected baseline
7. Log test result and file bug if mismatch
