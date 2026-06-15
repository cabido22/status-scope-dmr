# Status Scope — Copilot Agent Workspace

Two specialized agents and three shared skills for Diamond Rapids Status Scope
development and integration testing in VS Code.

## Architecture

See [.github/status-scope-architecture.md](.github/status-scope-architecture.md)
for the full system overview: source trees, plugin model, report structure, decision trees.

---

## Agents

| Agent | Invoke | Purpose |
|-------|--------|---------|
| `@status-scope-developer` | `@status-scope-developer <request>` | Create, fix, extend plugin **code** |
| `@status-scope-integrator` | `@status-scope-integrator <request>` | E2E tests, scenario injection, validation |

Both agents point to the same three source trees:
- `C:\pythonsv\diamondrapids\toolext\status_scope` — DMR ip_plugins
- `C:\pythonsv\bigcore\toolext\status_scope` — core_analyzer + project_plugins
- `C:\Python310\Lib\site-packages\pysvtools\status_scope` — framework base classes *(read-only)*

---

## Skills

| Skill | Trigger keywords | Purpose |
|-------|-----------------|---------|
| `decision-tree` | hang, timeout, MC, shutdown, scenario, classification | All scenario → action mappings |
| `status-scope-report` | report, shelf, insight, table, HTML, validate output | Report structure and validation |
| `pythonsv` | run, IPC, session, namednodes, inject, simulate | PythonSV runtime and scenario injection |

Skills are loaded automatically by the agents when relevant.
You can also invoke them directly: type `/decision-tree`, `/status-scope-report`, `/pythonsv` in chat.

---

## How to use

### Developer — code tasks

```
@status-scope-developer Create a new plugin for UPI link status that
collects per-lane error counters and registers it in config.json under the "fabric" group

@status-scope-developer The mesh plugin throws KeyError: 'mesh_credit_status' on B0
stepping — fix it

@status-scope-developer Add DMR adapter to core_analyzer — create the adapter and
project plugin following the novalake pattern

@status-scope-developer Improve the timeout plugin to also check FBLP credit stalls
in the decision tree

@status-scope-developer The pcie plugin is not generating an AER insight when
UES bits are set — investigate and fix
```

### Integrator — test and validation tasks

```
@status-scope-integrator Run TC-HANG-MC: inject a hung core with MCA on socket 0
and verify hung_with_mc classification with CRITICAL insight in the report

@status-scope-integrator Set up E2E test for TC-TIMEOUT-SCA: freeze a CBO TOR entry
targeting memory, run the timeout plugin, and validate the SCA correlation table

@status-scope-integrator Run a regression on the mesh plugin after the latest change —
cover all groups that include mesh and validate no regressions

@status-scope-integrator Simulate TC-SHUTDOWN on Simics and verify the report contains
CRS/GPSB/PSMB register dumps for shutdown_without_mc classification

@status-scope-integrator Run the full integration test suite (all TC-* scenarios)
and generate a pass/fail report
```

---

## Workspace structure

```
.github/
├── status-scope-architecture.md          ← Full system architecture doc
├── agents/
│   ├── status-scope-developer.agent.md   ← Code development agent
│   └── status-scope-integrator.agent.md  ← E2E testing agent
└── skills/
    ├── decision-tree/SKILL.md             ← All scenario → classification mappings
    ├── status-scope-report/SKILL.md       ← HTML report structure and validation
    └── pythonsv/SKILL.md                  ← PythonSV runtime and injection commands

status-scope-dmr/
└── status-scope-dmr.code-workspace       ← VS Code workspace (opens all 3 source trees)
```
