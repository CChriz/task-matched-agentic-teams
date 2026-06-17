# Current Status
- [x] **Jiuwen - Full Team Stream Capture**: Complete log of per-agent action traces, grouped into discrete turns, record of file content modifications, token usage...
- [x] **TeamBench Task Extraction**
- [x] **TeamBench Auto-grader on Team Output**

### TODOs
- [ ] Implment variety of Evaluation Metrics for selection
- [ ] Baseline setup of TeamBench in Jiuwen - 3-role hard separation - fixed team structure


**Task Performance Tracker**

| Task ID | Auto-Grader Score | 
|---|---|
CROSS1 | 6/10
P2 | 2/4
CRYPTO6 | 9/10
D6 | 18/18
IR2 | 7/7


---


## Project Components (TBU - current state)

This project is split across 3 related repositories: task extraction, agent-team execution, to grading and future team-behaviour analysis.
See respective Repo's for detailed breakdown of functions and usage.

### 1. Main review and evaluation artefacts

**Repository:** [CChriz/task-matched-agentic-teams](https://github.com/CChriz/task-matched-agentic-teams)

This is the main project repository. It contains the task-specific artefacts used for review, including exported task folders, task-matched team workspaces, agent-modified outputs, grading bundles, and evaluation logs.

To inspect:
- the task package given to the agent team
- the generated team/workspace for each task
- the files modified by agents
- the grading setup used to evaluate outputs
- auto-grader evaluation results


### 2. TeamBench task export and auto-grading

**Repository:** [CChriz/TeamBench-task-export](https://github.com/CChriz/TeamBench-task-export)

- extracting tasks from TeamBench
- packaging task files and workspaces
- packaging evaluation/grading bundles
- scripts for single-task and batch evaluation

In the overall pipeline, this repository is the upstream task-preparation component.


### 3. JiuwenSwarm stream capture for future metrics

**Repository:** [CChriz/jiuwenswarm-611](https://github.com/CChriz/jiuwenswarm-611)

This repository contains the stream-capture instrumentation used to record what happens during an agent-team run. It reconstructs per-agent activity from completed team runs, including reasoning, messages, tool calls, file edits, diffs, and token usage.

- capturing team-mode execution streams
- preserving per-agent action traces
- grouping interleaved events into agent turns
- recording file modifications and tool outputs
- producing trace data for future collaboration / team-behaviour metrics

In the overall pipeline, this repository is the downstream observability and metrics-capture component.


## End-to-End Workflow (TBU)

```text
TeamBench-task-export
        |
        |  exports chosen tasks and grading assets
        v
task-matched-agentic-teams
        |
        |  stores task/team artefacts, agent outputs, grading bundles and logs
        v
jiuwenswarm-611
        |
        |  full capture of per-agent execution streams for grouped logs and later team metrics
        v
future evaluation metrics

