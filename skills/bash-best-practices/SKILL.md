---
name: bash-best-practices
description: Expert guidance for writing/reviewing Bash programs with production-grade quality. Use when reviewing or writing robust shell scripts, CI/CD pipelines, or system utilities requiring fault tolerance and safety.
---

# Bash Best Practices

Comprehensive guidance for writing production-ready Bash scripts using defensive programming techniques, error handling, and safety best practices to prevent common pitfalls and ensure reliability.

## When to Use This Skill

- Writing or reviewing Bash code
- Building CI/CD pipeline scripts
- Creating system administration utilities
- Developing error-resilient deployment automation
- Writing scripts that must handle edge cases safely
- Building maintainable shell script libraries
- Implementing comprehensive logging and monitoring
- Creating scripts that must work across different platforms

## Core Defensive Principles

### Strict Mode

Enable bash strict mode at the start of every script to catch errors early.

```bash
#!/bin/bash
set -Eeuo pipefail  # Exit on error, unset variables, pipe failures
```

**Key flags:**

- `set -E`: Inherit ERR trap in functions
- `set -e`: Exit on any error (command returns non-zero)
- `set -u`: Exit on undefined variable reference
- `set -o pipefail`: Pipe fails if any command fails (not just last)

### Error Trapping and Cleanup

Implement proper cleanup on script exit on error. Example:

```bash
#!/bin/bash
set -Eeuo pipefail

trap 'echo "Error on line $LINENO"' ERR
trap 'echo "Cleaning up..."; rm -rf "$TMPDIR"' EXIT

TMPDIR=$(mktemp -d)
# Script code here
```

### Variable Safety

Always quote variables to prevent word splitting and globbing issues.

### Array Handling

Use arrays safely for complex data handling.

```bash
# Safe array iteration
declare -a items=("item 1" "item 2" "item 3")

for item in "${items[@]}"; do
  echo "Processing: $item"
  # do some work
done

# Reading output into array safely
mapfile -t lines < <(some_command)
readarray -t numbers < <(seq 1 10)
```

### Conditional Safety

Use `[[ ]]` for Bash-specific features, `[ ]` for POSIX.

```bash
# Bash - safer
if [[ -f "$file" && -r "$file" ]]; then
  content=$(<"$file")
fi

# POSIX - portable
if [ -f "$file" ] && [ -r "$file" ]; then
  content=$(cat "$file")
fi

# Test for VAR existence before operations
if [[ -z "${VAR:-}" ]]; then
  echo "VAR is not set or is empty"
fi
```

### Command Substitution

Always use `$()` instead of backticks for command substitution.

### Always Use ShellCheck

Your code should always be shellcheck compliant.

## Fundamental Patterns

### Safe Script Directory Detection

```bash
# Correctly determine script directory
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)"
SCRIPT_NAME="$(basename -- "${BASH_SOURCE[0]}")"
```

### Comprehensive Function Template

**IMPORTANT: always prefer to use local variables!** Use globals only when strictly necessary.

```bash
# Prefix for functions: handle_*, process_*, check_*, validate_*
# Include documentation and error handling

validate_file() {
  local -r file="$1"
  local -r message="${2:-File not found: $file}"

  if [[ ! -f "$file" ]]; then
    echo "ERROR: $message" >&2
    return 1
  fi
  return 0
}

process_files() {
  local -r input_dir="$1"
  local -r output_dir="$2"

  # Validate inputs
  [[ -d "$input_dir" ]] || { echo "ERROR: input_dir not a directory" >&2; return 1; }

  # Create output directory if needed
  mkdir -p "$output_dir" || { echo "ERROR: Cannot create output_dir" >&2; return 1; }

  # Process files safely
  while IFS= read -r -d '' file; do
    echo "Processing: $file"
    # Do work
  done < <(find "$input_dir" -maxdepth 1 -type f -print0)

  return 0
}
```

### Safe Temporary File Handling

```bash
trap 'rm -rf -- "$TMPDIR"' EXIT

# Create temporary directory
TMPDIR=$(mktemp -d) || { echo "ERROR: Failed to create temp directory" >&2; exit 1; }

# Create temporary files in directory
TMPFILE1="$TMPDIR/temp1.txt"
TMPFILE2="$TMPDIR/temp2.txt"

# Use temporary files
touch "$TMPFILE1" "$TMPFILE2"

echo "Temp files created in: $TMPDIR"
```

### Robust Argument Parsing

**NOTE**: if your program has a lot of options to be parsed as arguments, consider uring the Bashly framework.

```bash
set -Eeuo pipefail

# Default values
VERBOSE=false
OUTPUT_FILE=""

usage() {
  cat <<EOF
Usage: $0 [OPTIONS]

Options:
  -v, --verbose       Enable verbose output
  -o, --output FILE   Output file path
  -h, --help          Show this help message
EOF
  exit "${1:-0}"
}

# Parse arguments
while [[ $# -gt 0 ]]; do
  case "$1" in
  -v|--verbose)
    VERBOSE=true
    shift
    ;;
  -o|--output)
    OUTPUT_FILE="$2"
    shift 2
    ;;
  -h|--help)
    usage 0
    ;;
  --)
    shift
    break
    ;;
  *)
    echo "ERROR: Unknown option: $1" >&2
    usage 1
    ;;
  esac
done

# Validate required arguments
[[ -n "$OUTPUT_FILE" ]] || { echo "ERROR: -o/--output is required" >&2; usage 1; }
```

### Structured Logging

```bash
# Logging functions
# **NOTE**: the timestamp is optional.

log_info() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] INFO: $*" >&2
}

log_warn() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] WARN: $*" >&2
}

log_error() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] ERROR: $*" >&2
}

log_debug() {
  if [[ "${DEBUG:-0}" == "1" ]]; then
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] DEBUG: $*" >&2
  fi
}

# Usage
log_info "Starting script"
log_debug "Debug information"
log_warn "Warning message"
log_error "Error occurred"
```

### Process Orchestration with Signals

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

### Safe Command Substitution

```bash
set -Eeuo pipefail

# Use $() instead of backticks
name=$(<"$file")  # Modern, safe variable assignment from file
output=$(command -v python3)  # Get command location safely

# Handle command substitution with error checking
result=$(command -v node) || {
  log_error "node command not found"
  return 1
}

# For multiple lines
mapfile -t lines < <(grep "pattern" "$file")

# NUL-safe iteration
while IFS= read -r -d '' file; do
  echo "Processing: $file"
done < <(find /path -type f -print0)
```

### Dry-Run Support

```bash
set -Eeuo pipefail

DRY_RUN="${DRY_RUN:-false}"

run_cmd() {
  if [[ "$DRY_RUN" == "true" ]]; then
    echo "[DRY RUN] Would execute: $*"
    return 0
  fi

  "$@"
}

# Usage
run_cmd cp "$source" "$dest"
run_cmd rm "$file"
run_cmd chown "$owner" "$target"
```

## Best Practices Summary

1. **Always use strict mode** - `set -Eeuo pipefail`
2. **Quote all variables** - `"$variable"` prevents word splitting
3. **Use `[[ ]]` conditionals** - more robust than `[ ]`
4. **Implement error trapping** - catch and handle errors gracefully
5. **Always use functions** - with meaningful names
6. **Implement structured logging** - include levels (optionally a timestamp)
7. **Handle temporary files safely** - use mktemp, cleanup with trap

## Resources

- **Google Shell Style Guide**: <https://google.github.io/styleguide/shellguide.html>

