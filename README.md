# QueueCTL  
### A CLI-based Background Job Queue System (Go)

**QueueCTL** is a lightweight, persistent, CLI-driven job queue written in Go.  
It supports background workers, retries with exponential backoff, and a Dead Letter Queue (DLQ) — all configurable and persistent via SQLite.

---

## Features

| Category | Description |
|-----------|--------------|
| **Jobs** | Enqueue shell commands with metadata (id, retries, priority, timestamps). |
| **Workers** | Parallel background workers that process jobs concurrently. |
| **Retries** | Automatic retries with exponential backoff (`delay = base^attempts`). |
| **DLQ** | Permanently failed jobs are moved to a Dead Letter Queue. |
| **Persistence** | SQLite (WAL mode) for durable storage across restarts. |
| **Configurable** | Manage retry counts, backoff base, and more via CLI. |
| **Graceful Shutdown** | Workers finish in-flight jobs cleanly on exit. |
| **Cross-Platform** | Works on macOS, Linux, and Windows (using `cmd /C` for commands). |

---

## Tech Stack

- **Language:** Go (≥1.22)  
- **Database:** SQLite (via `modernc.org/sqlite`)  
- **CLI Framework:** [Cobra](https://github.com/spf13/cobra)

---

## Architecture Overview

```

┌────────────────────┐        ┌────────────────────┐
│   queuectl CLI     │        │   SQLite Storage   │
│  (enqueue, status, │        │ jobs / dlq / config│
│   worker, dlq)     │        └────────────────────┘
└─────────┬──────────┘                  ▲
│                             │
▼                             │
┌────────────────────┐        ┌────────────────────┐
│   Worker Pool      │  <───▶ │   Job Lifecycle    │
│  (Goroutines)      │        │ pending → processing│
│ executes commands  │        │ → completed/failed  │
└────────────────────┘        │ → DLQ if retries exhausted│
└────────────────────┘

````

---

## Job Specification

Each job record stored in SQLite looks like:

```json
{
  "id": "unique-job-id",
  "command": "echo 'Hello World'",
  "state": "pending",
  "attempts": 0,
  "max_retries": 3,
  "created_at": "2025-11-09T10:30:00Z",
  "updated_at": "2025-11-09T10:30:00Z"
}
````

### Job States

| State        | Description                          |
| ------------ | ------------------------------------ |
| `pending`    | Waiting to be picked up by a worker. |
| `processing` | Currently executing.                 |
| `completed`  | Finished successfully.               |
| `failed`     | Temporary failure — retry scheduled. |
| `dead`       | Permanently failed — moved to DLQ.   |

---

## 🛠️ Setup Instructions

### 1️⃣ Clone the repository

```bash
git clone https://github.com/<your-username>/queuectl
cd queuectl
```

### 2️⃣ Initialize dependencies

```bash
go mod tidy
```

### 3️⃣ Verify installation

```bash
go run main.go config get
```

This initializes the SQLite database (`queue.db`) and prints default configuration values.

---

## Usage Examples

### Enqueue a job

**Cross-platform (recommended)**:

```bash
go run main.go enqueue --id job1 --cmd "echo Hello QueueCTL"
```

**Windows PowerShell (safe form)**:

```powershell
go run main.go enqueue --id job2 --cmd 'powershell -Command "Write-Output ''Hello World''"'
```

### Start workers

```bash
go run main.go worker start --count 2
```

Output:

```
🚀 Starting 2 workers...
✅ Job job1 completed
   └─ Output: Hello QueueCTL
```

### View system status

```bash
go run main.go status
```

Shows number of jobs by state and active workers.

### List jobs by state

```bash
go run main.go list --state pending
```

### Check or retry DLQ jobs

```bash
go run main.go dlq list
go run main.go dlq retry job1-dlq
```

### Configuration management

```bash
go run main.go config set max-retries 5
go run main.go config get
```

---

## Retry & Backoff Logic

Each failed job is retried automatically with exponential backoff:

```
delay = base ^ attempts   (in seconds)
```

Example (base=2):

```
attempt #1 → 2s
attempt #2 → 4s
attempt #3 → 8s
```

After exceeding `max_retries`, the job is marked `dead` and moved to the DLQ.

---

## Persistence

All data (jobs, DLQ, config) lives in a single `queue.db` file.
SQLite is opened in WAL mode for safe concurrent access by multiple workers.

---

## Directory Structure

```
queuectl/
├── cmd/                   # CLI commands
│   ├── enqueue.go
│   ├── worker.go
│   ├── status.go
│   ├── list.go
│   ├── dlq.go
│   └── config.go
├── internal/
│   ├── db/                # Database setup & migrations
│   ├── queue/             # Job logic, workers, retry logic
│   └── util/              # Command execution helpers
├── main.go
├── go.mod
└── README.md
```

---

## Testing / Demo Script

Run these in sequence to demonstrate core features:

```powershell
# 1. Show config
go run main.go config get

# 2. Enqueue jobs
go run main.go enqueue --id ok1 --cmd "echo Hello"
go run main.go enqueue --id bad1 --cmd "doesnotexist"

# 3. Start workers
go run main.go worker start --count 2

# 4. View status and DLQ
go run main.go status
go run main.go dlq list
```

Expected:

* `ok1` → completed successfully
* `bad1` → retries then moved to DLQ

---

## Assumptions & Trade-offs

| Area                | Design Choice                          | Rationale                           |
| ------------------- | -------------------------------------- | ----------------------------------- |
| **Storage**         | SQLite (file-based)                    | Easy persistence, concurrency-safe. |
| **Shell execution** | `cmd /C` on Windows, `bash -c` on Unix | Full cross-platform support.        |
| **Backoff**         | Exponential (base^attempts)            | Simple + reliable.                  |
| **Concurrency**     | One DB transaction per claimed job     | Prevents double-processing.         |
| **Config**          | Stored in DB table                     | Centralized, durable configuration. |

---

## Bonus Features (Implemented / Planned)

| Feature                   | Status      |
| ------------------------- | ----------- |
| Job timeout               | ✅           |
| Job priority              | ✅           |
| Scheduled jobs (`run_at`) | ✅           |
| Job output logging        | ✅           |
| Retry & DLQ               | ✅           |

---

## Example Output

```
🚀 Starting 1 workers...
✅ Job job1 completed
   └─ Output: Hello QueueCTL
⚠️  Job bad1 failed, retrying in 2 sec
⚠️  Job bad1 failed, retrying in 4 sec
💀 Job bad1 moved to DLQ
```

---

## Design Summary

* **CLI frontend** → Cobra commands (`enqueue`, `worker`, `dlq`, etc.)
* **Core queue logic** → `internal/queue`
* **Persistence layer** → `internal/db`
* **Execution engine** → `internal/util/exec.go`
* **Configurable defaults** in `config` table

---

## License

MIT License © 2025 [Your Name]

---

## Acknowledgements

* Inspired by production job queues like Sidekiq, Celery, and Faktory.
