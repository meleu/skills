---
name: bash-best-practices
description: Expert guidance for writing/reviewing Bash programs with production-grade quality. Use this skill when writing or reviewing bash scripts.
---

# Bash Best Practices

Comprehensive guidance for writing production-ready Bash scripts using defensive programming techniques, error handling, and safety best practices to prevent common pitfalls and ensure reliability.

## Core Principles

### Strict Mode

Enable bash strict mode at the start of every script to catch errors early.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
```

### Error Trapping and Cleanup

Implement proper cleanup on script exit on error. Example:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

trap 'echo ERROR: "${BASH_SOURCE[0]}:${BASH_LINENO[0]} in \`${FUNCNAME[1]}\`: $BASH_COMMAND"' ERR
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

Always use `[[ ]]`.

```bash
# Bash - safer
if [[ -f "$file" && -r "$file" ]]; then
  content=$(<"$file")
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
SCRIPT_DIR="$(cd -- "${BASH_SOURCE[0]%/*}" && pwd -P)"
# Correctly determine script name
SCRIPT_NAME="${BASH_SOURCE[0]##*/}"
```

### Comprehensive Function Template

**IMPORTANT: always prefer to use local variables!** Use globals only when strictly necessary.

```bash
# Prefix for functions: handle_*, process_*, check_*, validate_*
# Include documentation and error handling

validate_file() {
  local -r file="$1"
  local -r message="${2:-File not found: $file}"

  [[ -f "$file" ]] && return 0

  echo "ERROR: $message" >&2
  return 1
}
```

### Safe Temporary File Handling

```bash
trap 'rm -rf -- "$TMPDIR"' EXIT

# Create temporary directory
readonly TMPDIR=$(mktemp -d) || { echo "ERROR: Failed to create temp directory" >&2; exit 1; }

my_function() {
  # Create temporary files in directory
  local -r tmpfile1="$TMPDIR/temp1.txt"
  local -r tmpfile2="$TMPDIR/temp2.txt"

  # Use temporary files
  touch "$tmpfile1" "$tmpfile2"

  echo "Temp files created in: $TMPDIR"
}
```

### Robust Argument Parsing

**NOTE**: if your program has a lot of options to be parsed as arguments, consider using the Bashly framework.

```bash
set -Eeuo pipefail

# Default values
VERBOSE=false
OUTPUT_FILE=""

main() {
  parse_arguments

  # do something cool...
}

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

parse_arguments() {
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
}
```

### Process Orchestration with Signals

In case you need to run launch parallel processes in background, check [signals-and-process-orchestration.md](./signals-and-process-orchestration.md).

### Safe Command Substitution

```bash
set -Eeuo pipefail

# Use $() instead of backticks
name="$(<"$file")"  # Modern, safe variable assignment from file
output="$(command -v python3)"  # Get command location safely

# Handle command substitution with error checking
result="$(command -v node)" || {
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
6. **Handle temporary files safely** - use mktemp, cleanup with trap

## Resources

- **Google Shell Style Guide**: <https://google.github.io/styleguide/shellguide.html>
