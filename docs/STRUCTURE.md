# Repository Structure

## Current layout

- cmd/downloader: CLI entrypoint.
- internal/app: app orchestration.
- internal/config: configuration and flag parsing.
- internal/input: URL/task parsing.
- internal/process: multi-process shard and launcher logic.
- internal/download: HTTP client, worker, retry, checksum.
- internal/output: file output and final report generation.
- internal/logx: logging helpers.
- data: sample input files.
- downloads: downloaded artifacts (git-ignored except .gitkeep).
- reports: run reports and failure lists (git-ignored except .gitkeep).
- logs: runtime logs (git-ignored except .gitkeep).
- tmp: temporary shard files and partial states (git-ignored except .gitkeep).

## Notes

- Keep internal package boundaries small and explicit.
- Do not put business logic in cmd/downloader/main.go.
- Add tests next to internal packages when implementation starts.
