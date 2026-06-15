---
name: status-scope-report
description: "Status Scope HTML report structure and analysis. Use when: opening or navigating the status scope report, understanding what sections a report contains, parsing insights and recommendations from the shelf, checking whether a plugin produced the correct output tables, validating watch windows and register dumps in a report, understanding the report_table shelf key structure, comparing report output against expected baseline."
---

# Status Scope Report Structure

How to open, navigate, and validate the HTML report produced by a status_scope run.

## 1. Opening the Report

```python
# Inside an active PythonSV session, after status_scope.run():

sv.report()                        # open full report in browser
sv.report(plugin="timeout")        # jump to timeout plugin section
sv.report(plugin="core_analyzer")  # jump to core analyzer section
sv.report(show_all=True)           # include empty/skipped sections
```

The report renders from a Python `shelve` database written during the run.

---

## 2. HTML Report Structure — Visual Navigation

When the report opens in the browser it is organized as follows:

### 2.1 Top banner
- **Run timestamp**, platform name, socket/die topology
- **Overall status badge**: CRITICAL / HIGH / MEDIUM / CLEAN
- Quick-jump links to each section below

### 2.2 Insights Summary (always first)
The most important section. Shows all insights generated across all plugins:

| Column | Meaning |
|--------|---------|
| Priority | CRITICAL → HIGH → MEDIUM |
| Type | `HW_HANG`, `HW_MCE`, `HW_UNEXPECTED_POWERDOWN`, `HW_ERR` |
| Plugin | Which analyzer produced it |
| Message | Human-readable description |
| Recommendation | What to do next |

> **Review tip**: Start here. If there are CRITICAL insights, go to that plugin section immediately.
> CLEAN = no insights generated — verify the run actually executed plugins (check shelf for `report_table:basic_check`).

### 2.3 Per-Plugin Sections
Each enabled plugin gets its own collapsible section. Sections contain:

- **Status tables** — register/state decoded into rows per socket/module/core
- **Classification result** (for core_analyzer) — `hung_with_mc`, `shutdown_without_mc`, etc.
- **Watch windows** — raw FSCP/ROB/SQ/MOB captures (only when `state_dump` collector was used)
- **Register dumps** — CRS/GPSB/PSMB raw text blocks (only for `shutdown_without_mc`)
- **Decision tree trace** — which branch was taken (timeout plugin only)

### 2.4 Empty sections
Sections may be empty because:
- Plugin was skipped (adaptive mode filtered it out) — re-run with `adaptive_mode=0`
- No data matched (no errors detected) — expected for clean systems
- Collector not included — e.g. watch windows require `collectors=["state_dump"]`

---

## 3. Shelf Key Structure

All report data is stored in the shelf. Keys follow these patterns:

```
report_table:<section_name>                 → DataFrame rendered as HTML table
insights                                    → list of Insight objects
report_table:watch_windows/S0M0C0/<name>    → watch window capture (per core)
report_table:register_dumps/S0M0/<mod>_crs  → CRS register dump
report_table:register_dumps/S0M0/<mod>_gpsb → GPSB register dump
report_table:register_dumps/S0M0/<mod>_psmb → PSMB register dump
```

Socket/Module/Core notation: `S#M#C#` where `#` is the 0-based index.

## 4. Key Report Sections per Plugin

### Core Analyzer

| Section | Shelf key | Content |
|---------|-----------|---------|
| Basic check | `report_table:basic_check` | ROB tail, EIP, core status per core |
| EIP read | `report_table:read_eip` | Instruction pointer values |
| SQ slices | `report_table:sq_slices_only` | Store queue slice occupancy |
| Patch breadcrumbs | `report_table:patch/patchload_breadcrumbs` | Microcode patch status |
| Acode table | `report_table:acode_table` | Assist code state |
| Sleep state | `report_table:sleep_state` | C-state / sleep state per core |
| Watch windows | `report_table:watch_windows/S#M#C#/<ww_name>` | FSCP, ROB, SQ, MOB captures |
| Register dumps | `report_table:register_dumps/S#M#/<module>_*` | CRS/GPSB/PSMB text |

### Timeout Plugin

| Section | Shelf key | Content |
|---------|-----------|---------|
| CBO TOR entries | `report_table:timeout/cbo_tor` | Decoded TOR entries per CBO |
| SCA correlation | `report_table:timeout/sca_correlation` | SCA TOR/RBT/WCT vs CBO |
| HAMVF mailbox | `report_table:timeout/hamvf_mailbox` | Mailbox data for timeout path |
| HIOP ITC | `report_table:timeout/hiop_itc` | ITC entries for pending completions |
| Decision tree | `report_table:timeout/decision_tree` | Classification path execution |

### PCIe Plugin

| Section | Shelf key | Content |
|---------|-----------|---------|
| Device enumeration | `report_table:pcie/devices` | All discovered PCIe devices |
| Link status | `report_table:pcie/link_status` | Per-port LNKSTA, LTSSM |
| AER errors | `report_table:pcie/aer` | AER UES, CES per device |

## 5. Insights Structure

Insights are the most important output. Each insight has:

```python
class Insight:
    type: str          # "HW_HANG", "HW_MCE", "HW_UNEXPECTED_POWERDOWN", "HW_ERR"
    priority: str      # "CRITICAL", "HIGH", "MEDIUM"
    message: str       # Human-readable description of the finding
    recommendation: str  # Actionable next step
    plugin: str        # Which plugin generated this insight
    data: dict         # Supporting data (register values, core IDs, etc.)
```

### Reading insights programmatically

```python
import shelve

with shelve.open("status_scope_report") as shelf:
    insights = shelf.get("insights", [])
    for i in insights:
        print(f"[{i.priority}] {i.type}: {i.message}")
        print(f"  → {i.recommendation}")
        print(f"  Plugin: {i.plugin}, Data: {i.data}")
```

## 6. Validating Report Output (for testing)

When validating a test run against expected baseline:

### 6.1 Check insight was generated

```python
assert any(i.type == "HW_HANG" and i.priority == "CRITICAL" for i in insights)
```

### 6.2 Check specific table exists and has data

```python
with shelve.open("status_scope_report") as shelf:
    df = shelf.get("report_table:basic_check")
    assert df is not None and not df.empty
    assert "hung_with_mc" in df["classification"].values
```

### 6.3 Check watch window was captured

```python
with shelve.open("status_scope_report") as shelf:
    ww = shelf.get("report_table:watch_windows/S0M0C0/fscp_regs")
    assert ww is not None
```

### 6.4 Check register dump was captured (shutdown scenario)

```python
with shelve.open("status_scope_report") as shelf:
    crs = shelf.get("report_table:register_dumps/S0M0/core_crs")
    assert crs is not None and len(crs) > 0
```

## 7. Common Report Issues

| Issue | Likely cause | How to check |
|-------|-------------|--------------|
| Section empty / missing | Plugin skipped due to adaptive mode | Re-run with `adaptive_mode=0` |
| No insights generated | Plugin did not detect an error condition | Check shelf for raw data tables |
| Watch windows missing | `state_dump` collector not used | Ensure `collectors=["namednodes", "state_dump"]` |
| Register dumps missing | Core not classified as `shutdown_without_mc` | Verify ROB tail classification |
| Report won't open | Shelf file corrupt or incomplete | Check for `.db`, `.dir`, `.bak` files |
