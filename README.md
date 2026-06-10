# Task-Matched Agentic Teams

Task-Matched Agentic Teams is an experimental artefact repository for evaluating how agent-team configurations perform on standalone task instances exported from [TeamBench](https://github.com/ybkim95/TeamBench).

The tasks in this repository are extracted from TeamBench using an extended standalone task export pipeline, specifically [`export_standalone_tasks.py`](https://github.com/CChriz/TeamBench-task-export/blob/main/scripts/export_standalone_tasks.py) from [CChriz/TeamBench-task-export](https://github.com/CChriz/TeamBench-task-export).


This repository is intended to make TeamBench-derived task instances easier to inspect, run, and compare against task-matched multi-agent-team outputs.

## Provenance

- **Tasks:** [ybkim95/TeamBench](https://github.com/ybkim95/TeamBench)
- **Export script:** [`scripts/export_standalone_tasks.py`](https://github.com/CChriz/TeamBench-task-export/blob/main/scripts/export_standalone_tasks.py)

The exported tasks are standalone tasks, prepared so they can be run, inspected, and evaluated outside the original TeamBench execution environment.

## Repo Structure (tbu)

```text
.
├── evals/
│   ├── cross1_0_eval/
│   ├── crypto6_0_eval/
│   ├── d6_0_eval/
│   ├── dist1_0_eval/
│   ├── lh2_0_eval/
│   ├── p2_0_eval/
│   ├── spec5_0_eval/
│   ├── test3_0_eval/
│   └── trap1_0_eval/
├── task_team/
│   ├── cross1_0_team/
│   ├── crypto6_0_team/
│   ├── d6_0_team/
│   ├── p2_0_team/
│   └── spec5_0_team/
└── README.md
