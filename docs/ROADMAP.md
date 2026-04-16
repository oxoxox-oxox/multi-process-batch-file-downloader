# Roadmap (Stage 1)

## V1.0 Sequential Downloader

- Read URLs from data/urls.txt.
- Download one by one.
- Save to downloads/.
- Print completed/total.

## V2.0 Concurrent Worker Pool

- Add fixed worker pool.
- Use queue/channel for tasks.
- Add mutex or atomic counters.
- Write failed URLs to reports/failed.txt.

## V3.0 Real-Time Progress

- Print per-file progress.
- Show global speed every 500ms.
- Keep output readable in terminal.

## V4.0 Retry + Resume

- Retry transient errors with exponential backoff (1s/2s/4s).
- Save temporary files as .part.
- Resume using Range header when supported.

## V5.0 Optional Enhancements

- Add --max-concurrency and --retry flags.
- Add download rate limit.
- Support stdin input.
- Write structured log to logs/download.log.
