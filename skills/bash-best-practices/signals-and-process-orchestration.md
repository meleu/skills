# Process Orchestration with Signals

```bash
set -Eeuo pipefail

# Track background processes
BG_PIDS=()

cleanup() {
  log_info "Shutting down..."

  # Terminate all background processes
  for pid in "${BG_PIDS[@]}"; do
    if kill -0 "$pid" 2>/dev/null; then
      kill -TERM "$pid" 2>/dev/null || true
    fi
  done

  # Wait for graceful shutdown
  for pid in "${BG_PIDS[@]}"; do
    wait "$pid" 2>/dev/null || true
  done
}

trap cleanup SIGTERM SIGINT

# Start background tasks
background_task &
BG_PIDS+=($!)

another_task &
BG_PIDS+=($!)

# Wait for all background processes
wait
```
