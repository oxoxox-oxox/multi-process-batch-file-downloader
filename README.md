# Multi-Process Batch File Downloader (Go)

This project downloads files in batches from a list of URLs using multiple processes. Each process runs a worker pool for concurrent downloads.

This README provides a complete guide to finish the project, including setup, structure, implementation steps, and troubleshooting.

## Overview

The downloader should:

- Read tasks from a text or CSV file (URL, optional output filename, optional checksum).
- Split tasks across multiple child processes.
- Run multiple workers in each process.
- Support retries, request timeouts, resume (HTTP Range), and progress logging.
- Generate a final success/failure report.

## Requirements

- Go 1.22 or later
- Windows, Linux, or macOS
- Network access to download targets

Check your Go version:

```powershell
go version
```

## Suggested Project Layout

```text
multi-process-batch-file-downloader/
  cmd/
    downloader/
      main.go
  internal/
    app/
      run.go
    config/
      config.go
    input/
      parser.go
    process/
      shard.go
      launcher.go
    download/
      worker.go
      client.go
      retry.go
      checksum.go
    output/
      writer.go
      report.go
    logx/
      log.go
  data/
    tasks.csv
  downloads/
  reports/
  go.mod
  README.md
```

## Input File Format

Use a CSV file with the following header:

```csv
url,filename,sha256
https://example.com/a.zip,a.zip,
https://example.com/b.iso,b.iso,7f83b1657ff1fc53...
```

Rules:

- `url` is required.
- `filename` is optional. If empty, generate one automatically.
- `sha256` is optional. If provided, verify it after download.

## Recommended CLI

Implement the CLI like this:

```text
downloader \
  --input ./data/tasks.csv \
  --out ./downloads \
  --report ./reports/result.json \
  --processes 4 \
  --workers 8 \
  --retries 3 \
  --timeout 60s \
  --rate-limit 0 \
  --resume true
```

Flag descriptions:

- `--input`: path to the input CSV file.
- `--out`: output directory for downloaded files.
- `--report`: path to the final report file.
- `--processes`: number of child processes.
- `--workers`: number of worker goroutines per process.
- `--retries`: retry count for transient errors.
- `--timeout`: request timeout per download.
- `--rate-limit`: bytes per second per process (`0` means unlimited).
- `--resume`: resume partial files when possible.

## Implementation Plan

### Step A: Parse and validate tasks

1. Read CSV rows.
2. Validate URL format.
3. Normalize output filenames.
4. Deduplicate tasks by URL + filename.

### Step B: Shard tasks for multi-process execution

1. Set shard count to `processes`.
2. Split tasks evenly (round-robin or chunk-based).
3. Write temporary shard files (for example, `tmp/shard_0.csv`, `tmp/shard_1.csv`).
4. Start one child process per shard using `os/exec`.

Use an internal child mode flag such as:

```text
--child --shard ./tmp/shard_0.csv
```

### Step C: Create a worker pool in each process

1. Create a buffered task channel.
2. Start `workers` goroutines.
3. Each worker should:
   - download the file
   - retry on retryable status/errors
   - verify checksum when provided
   - emit a result event
4. Collect all worker results and return a process summary.

### Step D: Add download robustness

Implement the following:

- Request timeout with `http.Client{Timeout: ...}`.
- Exponential backoff retries for `429`, `5xx`, and temporary network errors.
- Resume support with the `Range` header when the server supports partial content.
- Atomic finalization:
  - write to `filename.part`
  - rename to final filename only after success

### Step E: Generate a report

Write a machine-readable report in JSON format, for example:

```json
{
  "total": 100,
  "success": 97,
  "failed": 3,
  "duration_seconds": 41.2,
  "items": [
    {
      "url": "https://example.com/a.zip",
      "file": "downloads/a.zip",
      "status": "ok",
      "attempts": 1,
      "bytes": 123456
    }
  ]
}
```

## Build and Run

### Build

```powershell
go mod tidy
go build -o downloader.exe ./cmd/downloader
```

### Run

```powershell
.\downloader.exe --input .\data\tasks.csv --out .\downloads --report .\reports\result.json --processes 4 --workers 8 --retries 3 --timeout 60s --resume true
```

### Run without building (go run)

```powershell
go run ./cmd/downloader --input ./data/tasks.csv --out ./downloads --report ./reports/result.json --processes 4 --workers 8
```

## Logging Guidelines

Include these fields in logs:

- process ID
- worker ID
- URL
- destination file
- attempt number
- HTTP status code
- elapsed time

Example:

```text
level=info pid=12240 worker=3 file=a.zip status=200 bytes=123456 elapsed=1.23s
```

## Error Handling Checklist

- Invalid URL in input
- Permission denied for output directory
- Disk full
- HTTP `403/404` (non-retryable)
- HTTP `429/5xx` (retryable)
- TLS/connection reset timeout
- Checksum mismatch
- Duplicate output filename conflict

## Performance Tuning

- Start with `processes = CPU cores / 2` and `workers = 4 to 16`.
- Increase `workers` if the network is the bottleneck and CPU is mostly idle.
- Reduce `workers` if the remote server applies aggressive rate limits.
- Prefer fewer processes and more goroutines unless post-processing is CPU-heavy.

## Minimum Test Cases

- Single small-file download succeeds.
- Batch download succeeds (50+ URLs).
- Retry scenario works (mock `500` then `200`).
- Resume works for partial downloads.
- Checksum validation passes and fails correctly.
- Duplicate rows in input are handled correctly.
- Report counts and failed items are correct.

## Common Commands

```powershell
# Format
gofmt -w ./cmd ./internal

# Static checks
go vet ./...

# Tests
go test ./... -v
```

## Definition of Done

The project is complete when:

- Batch downloads run reliably.
- Multi-process mode works and can be measured.
- Failures are retried and clearly reported.
- Final outputs are deterministic and not corrupted.
- A final report is always generated.

## Future Improvements

- Per-domain concurrency limits
- Proxy support
- Scheduled download windows
- Pause/resume controls via signals
- Prometheus metrics endpoint
