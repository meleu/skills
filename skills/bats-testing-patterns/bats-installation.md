# BATS Installation

## BATS Availability

Try to use `bats` directly. If it's not available, try to find it at `test/bats/bin/bats`
(relative to project's root).

## Installing BATS in a Project

If you're in a git repository and it doesn't already have BATS at `test/bats/bin/bats`,
install it as a submodule:

```
git submodule add \
  https://github.com/bats-core/bats-core.git \
  test/bats
```

This is useful for development and also for CI pipelines, where we can easily run the tests.

## Install BATS Helpers

Also install BATS helpers, if they're not present in the project's repository:

```
# bats-support: for more user-friendly failure/error output
git submodule add \
  https://github.com/bats-core/bats-support.git \
  test/test_helper/bats-support

# bats-assert: better assertions
git submodule add \
  https://github.com/bats-core/bats-assert.git \
  test/test_helper/bats-assert
```

Only if explicitly needed, install the helper to make filesystem related assertions.

```
# bats-file: filesystem related assertions
git submodule add \
  https://github.com/bats-core/bats-file.git \
  test/test_helper/bats-file
```

