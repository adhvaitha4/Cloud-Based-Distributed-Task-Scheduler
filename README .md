# Distributed Task Scheduler (C++ / Linux / TCP)

A priority-based distributed task scheduling system with multithreaded worker nodes,
heartbeat failure detection, automatic task re-queuing, and JSON persistence.

---

## Architecture

```
Master Server (port 8080)
├── Priority Queue (max-heap, mutex-protected)
├── Task Dispatcher (one thread per worker)
├── In-flight Tracker (re-queue on failure)
├── Heartbeat Monitor (per-connection, timeout-based)
├── Stats Reporter (periodic summary)
└── JSON Persistence (tasks survive restart)

Worker Node
├── Connects to master
├── Receives one task
├── Sends HB ping every 3s while working
├── Executes task (simulated duration + failure)
└── Sends RESULT:OK or RESULT:FAIL
```

---

## Project Structure

```
task-scheduler/
├── src/
│   ├── master.cpp          Main server — dispatcher, re-queue, persistence
│   └── worker.cpp          Worker node — heartbeat, execution, result
├── include/
│   ├── logger.hpp          Thread-safe logger with file output
│   └── json_simple.hpp     Minimal JSON read/write (no external deps)
├── scripts/
│   └── generate_tasks.py   Dataset generator (100 tasks, varied priorities)
├── config.json             All tunable parameters
├── tasks.json              Generated task dataset (auto-created)
├── CMakeLists.txt
├── Dockerfile
├── docker-compose.yml      Master + 3 workers in containers
└── run_demo.sh             Local quick-test script
```

---

## Quick Start (Local)

**Prerequisites:** g++ (C++17), cmake, python3

```bash
# 1. Generate task dataset
python3 scripts/generate_tasks.py --count 100 --fail-rate 0.15

# 2. Build
mkdir build && cd build && cmake .. && make -j$(nproc) && cd ..

# 3. Run master (terminal 1)
./build/master config.json

# 4. Run workers (each in a new terminal)
./build/worker 127.0.0.1 8080
./build/worker 127.0.0.1 8080
./build/worker 127.0.0.1 8080

# OR: run everything with one script
bash run_demo.sh --workers 4 --tasks 30 --fail-rate 0.2
```

---

## VS Code Setup

1. Install extensions:
   - **C/C++** (Microsoft) — IntelliSense, debugging
   - **CMake Tools** (Microsoft) — build from sidebar
   - **Docker** (Microsoft) — container management

2. Open the `task-scheduler/` folder in VS Code

3. Command Palette (`Ctrl+Shift+P`):
   - `CMake: Select a Kit` → pick your GCC version
   - `CMake: Configure`
   - `CMake: Build`

4. To debug master: create `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Master",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/build/master",
      "args": ["config.json"],
      "cwd": "${workspaceFolder}",
      "MIMode": "gdb"
    }
  ]
}
```

---

## Docker

```bash
# Build image
docker build -t task-scheduler .

# Run master only (generates dataset automatically)
docker run -p 8080:8080 task-scheduler

# OR: run everything with docker-compose
docker-compose up --build

# Scale workers
docker-compose up --scale worker1=5
```

---

## Configuration (`config.json`)

| Key | Default | Description |
|---|---|---|
| `port` | 8080 | Master listen port |
| `heartbeat_interval_sec` | 3 | How often workers ping |
| `worker_timeout_sec` | 10 | Max silence before worker marked failed |
| `task_file` | tasks.json | Persistent task queue path |
| `log_file` | master.log | Log output path |
| `stats_interval_sec` | 30 | How often stats are printed |
| `max_backlog` | 20 | TCP listen backlog |

---

## Dataset Generator

```bash
python3 scripts/generate_tasks.py --help

Options:
  --count N        Number of tasks (default: 100)
  --fail-rate F    Fraction that will simulate failure (default: 0.15)
  --seed N         Random seed (default: 42)
  --output PATH    Output file (default: tasks.json)
```

Task categories: `data_processing`, `ml_training`, `media`, `system`, `analytics`,
`network`, `critical`. Each has a realistic priority range and duration.

---

## Protocol

```
Master → Worker:  "TASK:<id>|<name>|<duration_ms>|<payload_kb>|<will_fail>\0"
Worker → Master:  "HB\0"                     (heartbeat, every 3s)
Worker → Master:  "RESULT:<id>|OK\0"         (success)
Worker → Master:  "RESULT:<id>|FAIL\0"       (failure — triggers re-queue)
Master → Worker:  "NOTASK\0"                 (queue empty)
```

---

## Key Design Decisions

- **No task loss**: every dispatched task is tracked in `g_inflight`. If the worker
  disconnects or times out, the task is automatically re-pushed to the priority queue
  (up to `max_retries` times).

- **No external dependencies**: `json_simple.hpp` handles all JSON read/write without
  requiring nlohmann/json or Boost.

- **Thread-safe everything**: separate mutexes for the task queue, in-flight map, and
  heartbeat table. Atomic counters for metrics.

- **Persistence**: on clean shutdown (SIGINT/SIGTERM), remaining and in-flight tasks
  are saved back to `tasks.json` so the next run picks up where it left off.

- **SO_REUSEADDR**: server restarts instantly without waiting for TCP TIME_WAIT.

- **SIGPIPE ignored**: worker crashes don't kill the master process.
