---
name: pythonsv
description: "PythonSV environment knowledge for running status scope. Use when: starting a PythonSV IPC session, connecting to a live platform or Simics, running status_scope.run(), accessing namednodes, reading silicon registers, running a specific plugin or group, attaching a debugger to a plugin, understanding how to set up the PythonSV environment, injecting hang or MC scenarios via PythonSV, checking IPC connection status."
---

# PythonSV — Status Scope Runtime Environment

> **Source of truth**: The authoritative PythonSV connection infrastructure lives in **Kraken** at
> C:\SVShare\Kraken\skills\pythonsv\. This skill integrates the Kraken REPL server/client for
> all PythonSV execution. If any command here conflicts with Kraken, **Kraken takes precedence**.

How to start, connect, and operate the PythonSV environment for status scope development and integration testing.

## 1. Environment Setup — Kraken REPL Server

**NEVER initialize PythonSV inline.** All PythonSV commands MUST be executed via the Kraken REPL server/client infrastructure.

### 1.1 Check if the REPL server is already running

```powershell
netstat -ano | Select-String "53009.*LISTENING"
```

If listening (or C:\Temp\pythonsv_repl.ready exists) → skip to §1.3.

### 1.2 Start the REPL server

The server MUST run under the logged-in interactive session (faceless lab account) for full access
to PythonSV directories and IPC. A scheduled task "kraken pythonsv launcher" is pre-registered
by launch_kraken.py at platform startup — any authenticated user can trigger it:

```powershell
Remove-Item "C:\Temp\pythonsv_repl.ready" -Force -ErrorAction SilentlyContinue
& "C:\SVSHARE\Kraken\skills\pythonsv\scripts\start_repl_as_user.bat"
```

Wait up to 90 seconds for the server to be ready:

```powershell
for ($i=0; $i -lt 45; $i++) {
    Start-Sleep -Seconds 2
    $p = netstat -ano 2>$null | Select-String "53009.*LISTENING"
    if ($p) { Write-Host "SERVER READY"; break } else { Write-Host "Waiting... $i" }
}
```

The server has a 2-hour idle timeout and shuts itself down automatically. Do NOT manually kill it.

### 1.3 Execute PythonSV commands via the REPL client

All commands — including status_scope — go through the client:

```powershell
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "<command>"
```

The server pre-loads sv, itp, 
amednodes, socket0. The REPL preserves state between calls,
so import status_scope once and subsequent calls can reference it directly:

```powershell
# Import status_scope (once per server session)
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "from <product>.toolext import status_scope"

# Verify
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "print(status_scope.__file__)"
```

The import path depends on the product. Ask the user or check STATUS_SCOPE_PRODUCT_ROOT.

---

## 2. Running Status Scope

### 2.1 Full run (all enabled plugins)

```powershell
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'])"
```

### 2.2 Run specific plugins

```powershell
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], analyzers=['timeout', 'sca', 'hamvf'])"
```

### 2.3 Run a plugin group

```powershell
# Groups defined in config.json: imh, coherency, scf, d2d, iio, cbb, all
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], analyzers=['imh'])"
```

### 2.4 Run with state_dump collector (required for watch windows)

```powershell
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes', 'state_dump'], analyzers=['core_analyzer', 'timeout'])"
```

### 2.5 Adaptive mode

```powershell
# Max capture — full debug data collection
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], run_params={'ADAPTIVE': 0})"

# Min capture — registers only, fast
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], run_params={'ADAPTIVE': 2})"
```

### 2.6 Restrict to one socket

```powershell
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], analyzers=['timeout[0]'])"
```

---

## 3. Opening the Report

```powershell
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "sv.report()"
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "sv.report(plugin='timeout')"
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "sv.report(show_all=True)"
```

---

## 4. Namednodes — Accessing Hardware State

**MANDATORY: Navigate top-down with getSubComponents() before any register search.**
Never hardcode paths — hierarchy varies by silicon family.

### 4.1 Discover hierarchy (always do this first)

```powershell
# Step 1: top-level components under sockets (instant)
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "namednodes.sv.sockets.getSubComponents()"

# Step 2: drill into relevant component
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "namednodes.sv.sockets.cbb0.getSubComponents()"

# Step 3: continue drilling to target IP/module
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "namednodes.sv.sockets.cbb0.compute0.getSubComponents()"
```

### 4.2 Search within narrowed scope (fast — seconds)

```powershell
# Search by register name
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "namednodes.sv.socket0.cbb0.compute0.module0.search(regexpression=r'<register_name>', searchType='registers')"

# Search by description (BEST PRACTICE — more descriptive than names)
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "namednodes.sv.socket0.<ip>.search(regexpression=r'<description>', searchType='description')"

# Get register spec (fields, access type, reset value)
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "namednodes.sv.socket0.<ip>.<register>.getspec()"
```

**LAST RESORT ONLY** — socket0-level search (60+ seconds, expensive):
`
namednodes.sv.socket0.search(regexpression=r'<name>', searchType='registers')
`
Only use this if getSubComponents() navigation cannot identify the target sub-component.

### 4.3 Read/write registers

```powershell
# Read a register value
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "sv.socket0.uncore.ubox.scratch0.read()"

# Read a specific field
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "sv.socket0.pcie.port0.lnksta.link_training.read()"

# Iterate cores
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "[(c.id, hex(c.threads.t0.ia32_eip.read())) for c in sv.socket0.cores]"
```

---

## 5. Attaching VS Code Debugger to a Plugin

```powershell
# Step 1: Load the debugpy helper from the bigcore source tree (STATUS_SCOPE_BIGCORE_ROOT)
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "exec(open(r'<STATUS_SCOPE_BIGCORE_ROOT>\core_analyzer\tests\plugin_debug.py').read())"

# Step 2: In VS Code press F5 → "Attach: Core Analyzer plugin_debug (port 5678)"
#         Set breakpoints in the plugin source

# Step 3: Trigger the run to hit the breakpoint
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], analyzers=['core_analyzer'])"
```

---

## 6. Scenario Injection (for Integration Testing)

### 6.1 Simulate a hung core (Simics)

```python
# In Simics CLI:
simics> cpu0.set-pc 0xdeadbeef
simics> cpu0.disable-breakpoints
simics> continue
```

```powershell
# Then run core_analyzer:
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], analyzers=['core_analyzer'])"
```

### 6.2 Inject a Machine Check (Simics)

```python
# In Simics:
simics> cpu0.inject-mce bank=1 status=0xbd00000000000000 addr=0x1234
simics> continue
```

```powershell
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], analyzers=['mca', 'core_analyzer'])"
```

### 6.3 Freeze a CBO TOR entry (timeout simulation)

```powershell
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "sv.socket0.uncore.cbo0.tor_entry0.valid.write(1)"
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "sv.socket0.uncore.cbo0.tor_entry0.opcode.write(0x04)"
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], analyzers=['timeout'])"
```

### 6.4 Inject ECC error

```powershell
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "sv.socket0.uncore.mc0.mcerr_inj.write(0x1)"
python C:\SVSHARE\Kraken\skills\pythonsv\scripts\pythonsv_repl_client.py "status_scope.run(collectors=['namednodes'], analyzers=['mc', 'mca', 'memory', 'ddrphy'])"
```

---

## 7. Common PythonSV Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Port 53009 not listening | REPL server not started | Run start_repl_as_user.bat; poll for port 53009 |
| Server starts but immediately exits | PythonSV init error | Check visible server console window; re-run bat manually to see output |
| Client timeout (30 s) | Command takes too long | Reduce analyzers list; use daptive_mode=2; narrow getSubComponents() scope first |
| ImportError: no module named <product> | Product package not in server's sys.path | Verify STATUS_SCOPE_PRODUCT_ROOT; check server startup log |
| OfflineMissingNode | IPC not connected or node absent on this stepping | Wrap in try/except; check sv.refresh() via client |
| status_scope.run() hangs | Collector waiting for HW data | Use daptive_mode=2 first; check IPC is live |
| Empty report | All plugins skipped | Check config.json plugin registrations; verify IPC is live |
| shelve file not found | Run did not complete | Check for partial .db file; re-run |
| Server already running warning | Stale process from prior session | Server auto-expires after 2 h; or restart via bat to force re-init |
