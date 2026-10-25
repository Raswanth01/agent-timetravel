# Neural Nexus

### A time-travel debugger for AI agents

**Inspect an agent run. Rewind to a specific step. Change the input or prompt. Replay the run and compare the result.**

Neural Nexus is a developer tool for understanding how AI agents behave at runtime. It captures execution traces, presents them as an interactive timeline, and helps developers investigate unexpected results by replaying recorded operations or branching from a chosen point.

The project uses **MongoDB as its persistence layer** for trace data and debugging sessions. It is designed around local development workflows and OpenTelemetry-compatible instrumentation, so developers can inspect agent behavior without introducing a permanent proxy into their production traffic path.

---

## Why Neural Nexus?

Debugging an AI agent is more complicated than debugging a conventional function. A single run can contain model requests, tool calls, intermediate steps, and generated outputs. When the final answer changes, ordinary logs often do not make it easy to identify the exact step that caused the difference.

Neural Nexus provides a workflow for investigating that behavior:

- Capture execution traces from instrumented agent applications.
- Explore traces and spans in a timeline.
- Replay a recorded execution using stored responses.
- Branch from a selected step and continue with live model or tool calls.
- Compare a modified run against the original execution.
- Save checkpoints and resume debugging from a known state.
- Run evaluation suites against candidate behavior.
- Persist trace and debugging records in MongoDB.

## Core idea: capture versus replay

Neural Nexus treats trace capture and replay as two separate responsibilities.

```text
CAPTURE
Agent application
      |
      | OpenTelemetry / OpenInference spans
      v
OTLP receiver
      |
      v
MongoDB

REPLAY
Recorded trace
      |
      v
Replay engine
      |
      +---- Frozen replay: use recorded responses
      |
      +---- Branch replay: resume live execution from a chosen step
                         |
                         v
                Compare with original run
```

**Capture is passive:** the application emits telemetry after operations complete. Neural Nexus receives and stores the spans.

**Replay is active:** during a debugging session, the replay layer uses stored responses for earlier operations or allows execution to continue live from a selected point. This makes it possible to test a changed prompt, input, or configuration against the original run.

The receiver and replay engine share the same MongoDB-backed storage layer, so the debugger can work with traces collected earlier.

---

## Features

### Trace collection

- Ingest OpenTelemetry trace data through an OTLP receiver.
- Preserve trace and span relationships for later inspection.
- Store timing, status, attributes, and relevant operation metadata.
- Keep capture separate from the application's normal request path.

### Interactive timeline

- View spans in execution order.
- Inspect individual operations and their metadata.
- Identify the point where a run behaved unexpectedly.
- Navigate from a trace to the operation selected for replay.

### Frozen replay

Re-run the recorded path with responses retrieved from the stored trace instead of making the same outbound model calls again. This is useful for reproducing a run and inspecting its behavior without repeatedly paying for inference.

### Branch replay

Start from a selected span and continue execution with live calls. This makes it possible to experiment with a changed prompt or input while retaining the original execution as a baseline.

### Checkpointing

Save debugging state at useful points and restore it later, subject to the state supported by the agent integration. Checkpoint records are persisted in MongoDB.

### Diff and evaluation

Compare an original execution with a branch, and run evaluation suites to check candidate behavior against a baseline. Store evaluation metadata and results alongside the related debugging records.

### Framework integrations

Neural Nexus is designed to work with OpenTelemetry/OpenInference traces and replay adapters for supported agent and model frameworks. Optional framework dependencies may be required. Consult the adapter documentation in the repository for the integrations supported by the current build.

### MongoDB persistence

MongoDB stores the trace and debugging data used by the receiver, replay engine, checkpoints, branches, and evaluation workflows. The document-oriented model supports storing flexible span attributes and framework-specific metadata without requiring every attribute to share a fixed relational schema.

---

## Architecture

```text
┌─────────────────────────┐
│   Instrumented Agent    │
│  LLM calls / Tool calls │
└────────────┬────────────┘
             │ OTel / OpenInference
             v
┌─────────────────────────┐
│      OTLP Receiver      │
│ Validate and ingest     │
└────────────┬────────────┘
             │
             v
┌─────────────────────────┐
│   MongoDB Storage       │
│ traces, spans, sessions │
│ checkpoints, branches   │
│ evaluations             │
└────────────┬────────────┘
             │
       ┌─────┴───────────┐
       v                 v
┌───────────────┐  ┌────────────────┐
│ Replay Engine │  │ Evaluation     │
│ Frozen/Branch │  │ and Diff       │
└───────┬───────┘  └───────┬────────┘
        └─────────┬────────┘
                  v
        ┌──────────────────┐
        │  Web Workbench   │
        │ Timeline / Debug │
        └──────────────────┘
```

### Main components

| Component | Responsibility |
|---|---|
| Agent instrumentation | Emits telemetry for model calls, tool calls, and other operations |
| OTLP receiver | Accepts trace data and forwards it to storage |
| MongoDB storage layer | Persists trace data and debugging records |
| Replay engine | Reproduces recorded calls or branches execution from a selected point |
| Checkpoint manager | Saves and restores supported debugging state |
| Diff engine | Compares a candidate run with its baseline |
| Evaluation runner | Executes evaluation suites and records results |
| Web workbench | Displays traces and provides the interactive debugging workflow |
| CLI | Starts services and exposes the supported replay and evaluation commands |

---

## MongoDB data model

Neural Nexus uses MongoDB collections to organize the information needed by the debugger. Collection names below describe the project's logical data model; they should match the names configured in the storage implementation.

| Collection | Stored information |
|---|---|
| `traces` | Trace identifiers, run metadata, timestamps, and trace-level status |
| `spans` | Individual operations, parent-child relationships, attributes, events, and timing |
| `sessions` | Interactive debugging sessions and session state |
| `checkpoints` | Saved execution state and references to the relevant trace or session |
| `branches` | Parent trace, branch point, and metadata for alternative executions |
| `evaluations` | Evaluation suite metadata, candidate results, and baseline comparisons |

### Relationships

- A trace contains multiple spans.
- A span can reference a parent span, allowing the debugger to reconstruct the execution tree.
- A branch references the original trace and the span where the new execution begins.
- A checkpoint references the trace or session whose state it represents.
- An evaluation record links a candidate run to its suite and baseline.

### Indexing

Indexes should be created for fields used frequently by the receiver and UI, such as:

- `trace_id`
- `span_id`
- `parent_span_id`
- `session_id`
- `branch_id`
- timestamps used for sorting or filtering

Compound indexes should follow the actual query patterns in the storage layer. Unique constraints should be used only where the data model guarantees uniqueness.

---

## Quick start

### Prerequisites

- Python version supported by `pyproject.toml`
- MongoDB Community Server or a MongoDB Atlas cluster
- Git
- Node.js and pnpm/npm for frontend development, if required by your checkout

### 1. Clone the repository

```bash
git clone <YOUR_REPOSITORY_URL>
cd <YOUR_REPOSITORY_DIRECTORY>
```

Replace the placeholders with your repository URL and local directory name.

### 2. Create a virtual environment

**Windows PowerShell**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -e ".[dev]"
```

**macOS / Linux**

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -e ".[dev]"
```

### 3. Configure MongoDB

Create a `.env` file in the project root, or set the equivalent environment variables using the configuration mechanism supported by the application.

```dotenv
MONGODB_URI=mongodb://127.0.0.1:27017
MONGODB_DATABASE=neural_nexus
```

For MongoDB Atlas, use the connection string from your Atlas deployment and configure network access and database credentials there.

**Keep secrets out of Git.** Add `.env` to `.gitignore`, and never commit a real connection string containing a username or password.

### 4. Start the receiver

Start the OTLP receiver using the CLI entry point configured by the project. For a local installation, the command follows this pattern:

```bash
agent-timetravel serve --port 4318
```

The receiver should be configured to write incoming trace data to MongoDB.

### 5. Configure your agent

Set the OTLP endpoint in the environment where the instrumented agent runs:

```dotenv
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318
```

Run the instrumented application to send traces to Neural Nexus.

### 6. Open the web workbench

Start the UI with the CLI command configured for the project:

```bash
agent-timetravel ui --port 8484
```

Then open:

```text
http://127.0.0.1:8484/ui
```

Select a trace to inspect its spans and use the replay workflow to investigate a specific operation.

> **Configuration note:** the upstream project uses the `agent-timetravel` CLI and SQLite-oriented `--db` arguments. In this MongoDB edition, the receiver, CLI, replay engine, and evaluation runner must all load the MongoDB configuration instead of opening a SQLite database. Keep the commands above aligned with the CLI implementation in this repository.

---

## Register an agent

Neural Nexus is designed to support a decorator-first workbench API. The following illustrates the intended usage pattern; confirm imports and supported arguments against the package in your checkout.

```python
from agent_timetravel import TimeTravelContext, timetravel

@timetravel.agent(description="Answer a question")
async def answer(
    question: str,
    context: TimeTravelContext | None = None,
) -> str:
    # Replace this body with your agent's actual logic.
    return question
```

A workbench run can provide debugging context to the registered agent while ordinary direct calls remain part of the application's normal execution flow.

For a custom registry or title, use the `TimeTravel` object exposed by the installed package:

```python
from agent_timetravel import TimeTravel

timetravel = TimeTravel(title="Neural Nexus")
```

Check the package examples for the exact entry-point and CLI syntax supported by the current version.

---

## Replay a recorded trace

Neural Nexus supports two main replay modes.

### Frozen replay

Frozen replay uses responses from the stored trace instead of making the same outbound model calls again.

```bash
agent-timetravel replay <trace_id> --mode frozen
```

Use this mode to reproduce and inspect the recorded execution path.

### Branch replay

Branch replay starts from a selected span and continues with live execution.

```bash
agent-timetravel replay <trace_id> --branch-at 4 --mode branch
```

Use this mode to test a modified prompt, input, or supported configuration and compare the resulting behavior with the original trace.

The commands above show the intended workflow. If your CLI still requires a SQLite `--db` option, update its configuration path so replay reads from MongoDB before using these commands.

---

## Replay from Python

The replay API is intended to let an application enter a controlled replay context around an agent invocation.

```python
from agent_timetravel.replay import replay
from agent_timetravel.storage import TraceStore

# The MongoDB-backed TraceStore should read from the configured database.
store = TraceStore()

# Frozen replay: use recorded responses for the selected trace.
with replay(store, trace_id="<trace_id>", mode="frozen"):
    agent.run()

# Branch replay: replay the recorded prefix and continue live from a chosen span.
with replay(store, trace_id="<trace_id>", branch_at=4, mode="branch"):
    agent.run()
```

The constructor and context-manager arguments must match the actual MongoDB-enabled API in the checkout. The key requirement is that `TraceStore`, the replay engine, and the receiver use the same configured MongoDB database.

---

## Framework adapters

Neural Nexus is intended to support replay integrations for selected agent and model frameworks. Depending on the framework, an adapter may intercept calls automatically during a workbench run or may need to be attached explicitly.

Illustrative adapter usage:

```python
# LangGraph-style adapter
from agent_timetravel.adapters.langgraph import replay_chat_model
wrapped_model = replay_chat_model(real_chat_model)

# CrewAI-style adapter
from agent_timetravel.adapters.crewai import replay_llm
wrapped_llm = replay_llm(real_llm)
```

Use only adapters present in the current source tree. Install the optional dependencies required by each integration, and consult the repository's adapter documentation for supported versions and limitations.

---

## Evaluation

Evaluation suites can be used to compare candidate runs against a baseline and detect behavior changes.

```bash
agent-timetravel eval suite.yaml --suite-name regression-check
```

An evaluation run should record its inputs, baseline references, results, and status in MongoDB. The exact suite schema and exit codes depend on the CLI implementation; refer to the examples and tests in the repository.

Evaluation is most useful when it is repeatable: keep the same test inputs and evaluation criteria when comparing two agent versions.

---

## Development

### Install development dependencies

```bash
pip install -e ".[dev]"
```

### Run tests

```bash
python -m pytest tests --no-cov -q
```

### Run quality checks

Run the checks configured by the repository:

```bash
ruff check src/agent_timetravel tests
mypy --strict src/agent_timetravel
```

If the project configures additional checks, such as Pylint or a security scanner, run those as well.

### Frontend development

The UI uses the frontend toolchain configured in the `web/` directory. For example:

```bash
cd web
pnpm install
pnpm dev
```

Use the scripts in `web/package.json` as the source of truth for development and production builds.

### MongoDB integration tests

Before merging storage changes, test the complete flow:

1. Start a test MongoDB instance or use a dedicated test database.
2. Send a trace to the OTLP receiver.
3. Verify that trace and span documents are stored correctly.
4. Retrieve the trace through the API or UI.
5. Run frozen replay and confirm that stored responses are used.
6. Run branch replay and confirm new operations are linked to the correct branch.
7. Save and restore a checkpoint.
8. Run an evaluation suite and verify that results persist.
9. Confirm that tests do not accidentally use a developer's production database.

---

## Project structure

The source tree may evolve as development continues. The main areas in the upstream project include:

```text
src/agent_timetravel/
  adapters/       Framework-specific replay adapters
  receiver.py     OTLP ingestion
  replay.py       Frozen and branch replay
  checkpoint.py   State checkpointing
  diff.py         Trace comparison
  eval_api.py     Evaluation runner
  cli.py          Command-line interface

tests/             Automated tests
web/               React / Vite / TypeScript UI
docs/              Design notes and phase documentation
examples/          Example applications and integrations
scripts/           Development and maintenance scripts
pyproject.toml     Python package and tool configuration
```

Keep the project structure section synchronized with the actual files in this fork, especially if the MongoDB storage implementation introduces new modules.

---

## Security and privacy

Agent traces can contain prompts, generated responses, tool arguments, user input, and other sensitive information.

- Never commit API keys, database credentials, or real `.env` files.
- Restrict MongoDB network access and use a least-privilege database user.
- Avoid collecting sensitive fields unless they are needed for debugging.
- Define retention rules for stored traces and evaluation results.
- Do not expose the receiver or debugging UI publicly without appropriate authentication and network controls.
- Use a dedicated test database for automated tests.

Neural Nexus is a debugging tool, not a substitute for application authorization, secrets management, or production monitoring controls.

---

## Design goals

- **Developer-controlled debugging:** inspect and replay selected runs on demand.
- **Low integration overhead:** use existing telemetry instrumentation wherever possible.
- **Reproducibility:** retain the data needed to understand a previous execution.
- **Safe experimentation:** branch from a recorded point rather than overwriting the original run.
- **Flexible persistence:** use MongoDB to store trace attributes and related debugging records.
- **Clear separation of concerns:** keep trace ingestion, storage, replay, evaluation, and UI responsibilities distinct.

## Roadmap

Potential areas for continued development include:

- Strengthen MongoDB-backed integration and regression tests.
- Improve trace filtering and search for large datasets.
- Add clearer side-by-side branch comparison.
- Expand replay adapters and compatibility coverage.
- Improve checkpoint recovery and replay diagnostics.
- Add configurable retention and export workflows.

Only mark a roadmap item as completed once it is implemented and verified in the repository.
