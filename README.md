# Cache Memory Benchmark

A C++17 learning project exploring sequential access to contiguous memory. The
benchmark sweeps 17 powers-of-two working-set sizes from 4 KiB through 256 MiB,
summing vectors of unsigned 64-bit integers in repeated batches after warm-up.

The goal is to learn how to build, validate, and interpret memory benchmarks.
Results are measurements of this scan on a particular machine and build, not a
direct measurement of RAM bandwidth or individual cache latency.

## Current behaviour

- Double the input size from 4 KiB to 256 MiB, inclusive, in ascending order.
- Allocate and initialise a fresh vector for each size, with values from 0–1000.
- Restart a Mersenne Twister engine with seed 55 for each size and calculate a
  reference checksum during initialisation.
- Run and validate one untimed warm-up scan per size.
- Measure 20 trials per size, with 100 complete scans in each timed batch.
- Validate each batch checksum against 100 times the single-scan reference;
  exit with status 1 on a mismatch.
- Print an aligned row per validated size: size in KiB, element count, minimum,
  median, and maximum batch durations in ns, median ns/element, and useful GB/s.

Timing columns describe whole 100-scan batches. Normalised metrics count all
100 scans. Warm-up timings, individual checksums, and the mean are not printed.
The size range, trial count, and scans per trial are fixed in the driver; there
are no command-line options, CSV export, or plotting scripts yet.

## Requirements

- CMake 3.20 or newer.
- A C++17-capable compiler and C++ standard library.
- A build tool supported by CMake, such as Make or Ninja.

There are no third-party library dependencies.

## Build and run

From the repository root, using a single-configuration generator such as Unix
Makefiles or Ninja:

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=ON
cmake --build build
./build/main
ctest --test-dir build --output-on-failure
```

The first command configures the project and creates the build directory. The
second builds the executables; the third runs the scan benchmark. The final
command runs the correctness tests. Use Release builds for performance
experiments. A Debug build is useful for debugging but is not directly comparable
to an optimised build.

For multi-configuration generators such as Xcode or Visual Studio, select
Release with `cmake --build build --config Release`; the executable is normally
inside the build directory's `Release` subdirectory (with `.exe` on Windows).
Run tests with `ctest --test-dir build -C Release --output-on-failure`.

For a clean build check, configure a fresh directory instead of reusing a cache
from another machine or source location.

## Clock-overhead diagnostic

After building, run the separate diagnostic:

```sh
./build/clock_overhead
```

It measures 100,000 pairs of consecutive `steady_clock::now()` calls with no
scan between them, then reports minimum, maximum, and median empty-interval
durations in nanoseconds. Compare these with batch durations to assess timing
overhead. Zero readings can occur, and occasional large readings may reflect
interruptions. Do not automatically subtract this estimate from scan timings.
The diagnostic is not a CTest test and has no timing pass/fail threshold.

## Continuous integration

The GitHub Actions workflow builds Release executables and runs CTest on Ubuntu
with GCC and Clang, and on macOS with Clang. It runs on pushes and pull requests
targeting `main`. CI checks correctness; it does not enforce performance thresholds
or run either the full sweep or the clock-overhead diagnostic.

## Source layout

- `CMakeLists.txt`: executable targets, source files, header path, and C++17 requirements.
- `src/scan.h`: declaration of the sequential checksum operation.
- `src/scan.cpp`: implementation of one sequential scan.
- `benchmarks/main.cpp`: input generation, warm-up, timing, validation, and reporting.
- `benchmarks/clock_overhead.cpp`: standalone empty timing-interval diagnostic.
- `.github/workflows/ci.yml`: Linux and macOS build and correctness checks.
- [docs/methodology.md](docs/methodology.md): measurement procedure and limitations.
- `tests/scan_test.cpp`: correctness checks for empty, single-element, and multi-element inputs.
- [LICENSE](LICENSE): MIT licence.

## Interpreting results

Use median nanoseconds per element as the main normalised metric. Minimum and
maximum batch durations provide context about variation among the 20 trials
for each size. Useful GB/s is useful bytes processed divided by median batch
duration; it does not measure hardware memory traffic. Repeat whole
runs before drawing conclusions; no performance results are claimed here yet.

Record the CPU, operating system, compiler and standard-library versions, build
configuration, compiler flags, and source revision alongside any published
results. See the [methodology](docs/methodology.md) for timing boundaries,
optimisation caveats, and reproducibility details.

## Planned next steps

1. Compare repeated sweeps for stability and record timing-overhead observations.
2. Assess whether large working sets need fewer scans per trial; currently all
   sizes use 100 scans, processing 500 GiB across the 20 trials at 256 MiB alone.
3. Recheck generated code when changing batching, compiler, or optimisation flags.
4. Export validated measurements and environment metadata to CSV and companion files.
5. Plot results against working-set size and document their interpretation.

## Licence

MIT. See [LICENSE](LICENSE).
