---
name: bats-testing-patterns
description: Expert guidance for writing comprehensive tests for bash code using BATS (Bash Automated Testing System). Use when writing tests for shell scripts with BATS.
---

# Bats Testing Practices

Comprehensive guidance for writing comprehensive unit tests for shell scripts using Bats (Bash Automated Testing System), including test patterns, fixtures, and best practices for production-grade shell testing.

## BATS Availability

Try to use `bats` directly. If it's not available, try to find it at
`test/bats/bin/bats` (relative to project's root). If it's still unavailable,
check the [BATS installation instructions](./bats-installation.md).

**NOTE**: all the examples here assume you're using BATS with the helpers bats-support and bats-assert.

## Basic Test Structure

**IMPORTANT**: use `.bats` file suffix in filenames for test code to be executed with BATS.

### Simple Test File

```bash
#!/usr/bin/env bats

# Setup runs before each test
setup() {
  load 'test_helper/bats-support/load'
  load 'test_helper/bats-assert/load'

  export TMPDIR=$(mktemp -d)
}

# Teardown runs after each test
teardown() {
  rm -rf "$TMPDIR"
}

# simple assertions
@test "Function returns 0 on success" {
  run my_function "an_argument"
  assert_success
}

@test "Function returns 1 on missing argument" {
  run my_function
  assert_failure
}

@test "Function outputs correct result" {
  run my_function "test"
  assert_output "expected output"
}
```

## Setup and Teardown Patterns

### Basic Setup and Teardown

```bash
#!/usr/bin/env bats

# Setup runs before each test
setup() {
  load 'test_helper/bats-support/load'
  load 'test_helper/bats-assert/load'

  export TMPDIR=$(mktemp -d)

  # Source script under test
  source "${BATS_TEST_DIRNAME}/../bin/script.sh"
}

# Teardown runs after each test
teardown() {
  rm -rf "$TMPDIR"
}
```

### Setup with Resources

```bash
#!/usr/bin/env bats

setup() {
  load 'test_helper/bats-support/load'
  load 'test_helper/bats-assert/load'
  load 'test_helper/bats-file/load'

  # Create directory structure
  mkdir -p "$TMPDIR/data/input"
  mkdir -p "$TMPDIR/data/output"

  # Create test fixtures
  echo "line1" > "$TMPDIR/data/input/file1.txt"
  echo "line2" > "$TMPDIR/data/input/file2.txt"

  # Initialize environment
  export DATA_DIR="$TMPDIR/data"
  export INPUT_DIR="$DATA_DIR/input"
  export OUTPUT_DIR="$DATA_DIR/output"
}

teardown() {
  rm -rf "$TMPDIR/data"
}

@test "Processes input files" {
  run my_process_script "$INPUT_DIR" "$OUTPUT_DIR"
  assert_success
  assert_file_exists "$OUTPUT_DIR/file1.txt"
}
```

### Global Setup/Teardown

```bash
#!/usr/bin/env bats

# setup_file runs once before all tests
setup_file() {
  export SHARED_RESOURCE=$(mktemp -d)
  echo "Expensive setup" > "$SHARED_RESOURCE/data.txt"
}

# teardown_file runs once after all tests
teardown_file() {
  rm -rf "$SHARED_RESOURCE"
}

@test "First test uses shared resource" {
  assert_file_exists "$SHARED_RESOURCE/data.txt"
}

@test "Second test uses shared resource" {
  assert_dir_exists "$SHARED_RESOURCE"
}
```

## Assertion Patterns

### Exit Code Assertions

```bash
#!/usr/bin/env bats

@test "Command succeeds" {
  run true
  assert_success
}

@test "Command fails as expected" {
  run false
  assert_failure
}

@test "Command returns specific exit code" {
  run my_function --invalid
  assert_failure 127
}

@test "Can capture command exit status and output" {
  run echo "hello"
  assert_success
  assert_output "hello"
}
```

### Output Assertions

```bash
#!/usr/bin/env bats

setup() {
  load 'test_helper/bats-support/load'
  load 'test_helper/bats-assert/load'
}

@test "Output matches string" {
  run echo "hello world"
  assert_output "hello world"
}

@test "Output contains substring" {
  run echo "hello world"
  assert_output --partial "world"
}

@test "Output matches RegEx pattern" {
  run date +%Y
  assert_regex "$output" '^[0-9]{4}$'
}

@test "Multi-line output" {
  run printf "line1\nline2\nline3"
  assert_output "line1
line2
line3"
}

@test "Validate a line is present in a multi-line output" {
  run printf "line1\nline2\nline3"
  assert_line 'line2'
}
```

### File Assertions

**NOTE**: The assertions here depend on a BATS helper named bats-file.

```bash
#!/usr/bin/env bats

setup() {
  # bats-file is required to use the file assertions
  load 'test_helper/bats-file/load'
  load 'test_helper/bats-support/load'

  export TMPDIR=$(mktemp -d)
}

teardown() {
  rm -rf "$TMPDIR"
}

@test "File is created" {
  local file="$TMPDIR/output.txt"
  my_function > "$file"
  # this assertion also works for directories
  assert_exists "$file"
}

@test "File contents match expected" {
  local file="$TMPDIR/output.txt"
  my_function > "$file"
  assert_file_contains "$file" "expected content"
}

@test "File has correct permissions" {
  local file="$TMPDIR/test.txt"
  touch "$file"
  chmod 644 "$file"
  assert_file_permission 644 "$file"
}

@test "File size is correct" {
  local file="$TMPDIR/test.txt"
  echo -n "12345" > "$file"
  assert_file_size_equals "$file" 5
}
```

## Mocking and Stubbing Patterns

### Function Mocking

```bash
#!/usr/bin/env bats

# Create a function with the same name of the external tool to mock its behavior
my_external_tool() {
  echo "mocked output"
  return 0
}

@test "Function uses mocked tool" {
  export -f my_external_tool
  run my_function
  assert_output --partial "mocked output"
}
```

### Command Stubbing

```bash
#!/usr/bin/env bats

setup() {
  # Create stub directory
  STUBS_DIR="$TMPDIR/stubs"
  mkdir -p "$STUBS_DIR"

  # Add to PATH
  export PATH="$STUBS_DIR:$PATH"
}

create_stub() {
  local cmd="$1"
  local output="$2"
  local code="${3:-0}"

  cat > "$STUBS_DIR/$cmd" <<EOF
#!/bin/bash
echo "$output"
exit $code
EOF
  chmod +x "$STUBS_DIR/$cmd"
}

@test "Function works with stubbed curl" {
  create_stub curl "{ \"status\": \"ok\" }" 0
  run my_api_function
  assert_success
}
```

## Fixture Management

### Using Fixture Files

```bash
#!/usr/bin/env bats

# Fixture directory: tests/fixtures/

setup() {
  FIXTURES_DIR="${BATS_TEST_DIRNAME}/fixtures"
  WORK_DIR=$(mktemp -d)
  export WORK_DIR
}

teardown() {
  rm -rf "$WORK_DIR"
}

@test "Process fixture file" {
  # Copy fixture to work directory
  cp "$FIXTURES_DIR/input.txt" "$WORK_DIR/input.txt"

  # Run function
  run my_process_function "$WORK_DIR/input.txt"

  # Compare output
  assert_files_equal "$WORK_DIR/output.txt" "$FIXTURES_DIR/expected_output.txt"
}
```

### Dynamic Fixture Generation

```bash
#!/usr/bin/env bats

generate_fixture() {
  local lines="$1"
  local file="$2"

  for i in $(seq 1 "$lines"); do
      echo "Line $i content" >> "$file"
  done
}

@test "Handle large input file" {
  generate_fixture 1000 "$TMPDIR/large.txt"
  run my_function "$TMPDIR/large.txt"
  assert_success
  assert [ "$(wc -l < "$TMPDIR/large.txt")" -eq 1000 ]
}
```

## Best Practices

1. **Test one thing per test** - single responsibility principle
2. **Use descriptive test names** - clearly states what is being tested
3. **Clean up after tests** - always remove temporary files in teardown
4. **Test both success and failure paths** - don't just test happy path
5. **Mock external dependencies** - isolate unit under test
6. **Use fixtures for complex data** - makes tests more readable
7. **Run tests in CI/CD** - catch regressions early
8. **Test across shell dialects** - ensure portability
9. **Keep tests fast** - run in parallel when possible
10. **Document complex test setup** - explain unusual patterns

## Resources

- **Bats Documentation**: <https://bats-core.readthedocs.io/>
- **Bats GitHub**: <https://github.com/bats-core/bats-core>
- **Bats basic assertions**: <https://github.com/bats-core/bats-assert/blob/master/README.md>
- **Bats file assertions**: <https://github.com/bats-core/bats-file/blob/master/README.md>
