# LevelDB Agent Guidelines

This document provides guidelines for agentic coding assistants working with the LevelDB repository. It covers build commands, testing, linting, and code style conventions.

## Quick Start

1. **Build** (out‑of‑tree, debug):
   ```bash
   mkdir -p build && cd build
   cmake -DCMAKE_BUILD_TYPE=Debug ..
   cmake --build .
   ```
2. **Run all tests**:
   ```bash
   ctest --verbose
   ```
3. **Format changed files**:
   ```bash
   clang-format -i --style=file <file.cc>
   ```

## Build Commands

LevelDB uses CMake for building. The project can be built in-tree or out-of-tree; the recommended approach is out-of-tree builds.

### Standard Build (Debug)
```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Debug ..          # Debug build with assertions
cmake --build .                            # Build all targets
```

### Release Build
```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..        # Optimized release build
cmake --build .
```

### Build Options
- `-DLEVELDB_BUILD_TESTS=ON/OFF` – enable/disable unit tests (default ON)
- `-DLEVELDB_BUILD_BENCHMARKS=ON/OFF` – enable/disable benchmarks (default ON)
- `-DCMAKE_CXX_STANDARD=17` – C++ standard (default 17, required)

### Parallel Build
```bash
cmake --build . --parallel $(nproc)        # Linux/macOS
cmake --build . --parallel                 # Let CMake decide
```

## Test Commands

Tests are built into a single executable `leveldb_tests` (contains most unit tests) plus a few standalone test executables.

### Run All Tests
```bash
cd build
ctest --verbose                            # Run all tests via CTest
./leveldb_tests                            # Run directly (Google Test output)
```

### Run a Single Test Suite
```bash
./leveldb_tests --gtest_filter=DBTest.*    # Run all DBTest test cases
```

### Run a Specific Test
```bash
./leveldb_tests --gtest_filter=DBTest.Empty    # Run the "Empty" test case
```

### Standalone Test Executables
- `c_test` – C API tests
- `env_posix_test` – POSIX environment tests (Linux/macOS)
- `env_windows_test` – Windows environment tests

Run them directly:
```bash
./build/c_test
./build/env_posix_test
```

### Test Debugging
- Use `--gtest_list_tests` to list all test cases.
- Use `--gtest_repeat=N` to repeat tests N times.
- Use `--gtest_break_on_failure` with a debugger.

## Linting and Formatting

The project uses **clang-format** with a style based on the Google C++ style guide.

### Format a Single File
```bash
clang-format -i --style=file <file.cc>
```

### Format All Source Files
```bash
find . -iname '*.cc' -o -iname '*.h' -o -iname '*.h.in' | xargs clang-format -i --style=file
```

### Formatting Notes
- The `.clang-format` file in the root directory defines the style.
- Includes are sorted according to the priority rules in `.clang-format`.
- Always run formatting before submitting changes.

## Code Style

LevelDB follows the Google C++ Style Guide with a few project-specific conventions.

### Naming Conventions
- **Classes**: `CamelCase` with first letter uppercase (e.g., `DBImpl`, `VersionSet`)
- **Functions**: `CamelCase` with first letter lowercase (e.g., `openFile`, `compactMemTable`)
- **Variables**: `snake_case` for locals and parameters; member variables have a trailing underscore (e.g., `mutex_`, `versions_`)
- **Constants**: `kCamelCase` with leading `k` (e.g., `kNumLevels`, `kMaxTableFiles`)
- **Macros**: `UPPER_CASE` with `LEVELDB_` prefix (e.g., `LEVELDB_EXPORT`)
- **Namespaces**: Only the top-level `leveldb` namespace is used.

### Includes Order
1. System headers (`#include <algorithm>`, `<cstdint>` etc.)
2. Public LevelDB headers (`#include "leveldb/db.h"`)
3. Internal headers (`#include "db/dbformat.h"`)
4. Local headers (within same module)

The `.clang-format` file automatically enforces this ordering.

### Error Handling
- Use `Status` objects to report success/failure.
- Return `Status::OK()` for success.
- Use factory methods for error statuses: `Status::NotFound()`, `Status::Corruption()`, `Status::IOError()`, etc.
- Check results with `status.ok()`, `status.IsNotFound()`, etc.
- Do not use exceptions; the project is compiled with `-fno-exceptions`.

### Memory Management
- Raw pointers are used for owned objects; delete them in destructors.
- Use `std::unique_ptr` sparingly (only when standard library integration is needed).
- Manual memory management is acceptable for performance-critical code.

### Thread Safety
- The code uses a custom `port::Mutex` abstraction (see `port/port.h`).
- Annotations like `EXCLUSIVE_LOCKS_REQUIRED(mutex_)` are used for thread‑safety analysis.
- Member variables are annotated with `GUARDED_BY(mutex_)` where appropriate.

### Const Correctness
- Use `const` for parameters, methods, and member variables where possible.
- Prefer `const` references for large or non‑trivial types.

### Explicit Constructors
- Single‑argument constructors must be marked `explicit`.

### Override and Final
- Use `override` for virtual method overrides.
- Use `final` sparingly, only when a class is not meant to be derived further.

### Noexcept
- Use `noexcept` for move constructors, move assignment operators, and destructors when they cannot throw.

## Type Usage

- Use standard integer types (`int64_t`, `uint32_t`, etc.) from `<cstdint>`.
- `size_t` for sizes and indices.
- `char` for raw byte buffers; `std::string` for owned character data.
- `Slice` (leveldb::Slice) for non‑owning string/byte‑array references.

## Header Guards

Use the pattern `STORAGE_LEVELDB_<PATH>_<FILE>_H_` (e.g., `STORAGE_LEVELDB_DB_DB_IMPL_H_`).

## Example Code Snippet

```cpp
// Copyright (c) 2011 The LevelDB Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file. See the AUTHORS file for names of contributors.

#ifndef STORAGE_LEVELDB_DB_EXAMPLE_H_
#define STORAGE_LEVELDB_DB_EXAMPLE_H_

#include <cstdint>
#include <string>

#include "leveldb/status.h"
#include "db/internal_header.h"

namespace leveldb {

class Example {
 public:
  explicit Example(int64_t capacity);
  ~Example() noexcept;

  Status Insert(const Slice& key, const Slice& value);
  bool Contains(const Slice& key) const;

 private:
  int64_t capacity_;
  std::string name_;
};

}  // namespace leveldb

#endif  // STORAGE_LEVELDB_DB_EXAMPLE_H_
```

## Additional Notes

- **No Cursor or Copilot rules** – No `.cursor/`, `.cursorrules`, or `.github/copilot-instructions.md` files are present.
- **Third‑party code** resides in `third_party/` and should not be modified.
- **Public API** headers are in `include/leveldb/`; changes here must preserve backward compatibility.
- **Internal headers** are placed in the appropriate subdirectory (`db/`, `table/`, `util/`, etc.).
- **Benchmarks** are built as separate executables (`db_bench`, `db_bench_sqlite3`, `db_bench_tree_db`).
- **Portability** is important; use the abstractions in `port/port.h` for OS‑specific functionality.

## Useful Scripts

- `./build/leveldbutil` – command‑line utility for debugging and inspecting databases.
- `./build/db_bench` – run performance benchmarks.

## Resources

- [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
- [LevelDB Documentation](https://github.com/google/leveldb/blob/main/doc/index.md)
- [CMake Documentation](https://cmake.org/documentation/)

---
*This file is intended for agentic coding assistants. Keep it up‑to‑date as conventions evolve.*
