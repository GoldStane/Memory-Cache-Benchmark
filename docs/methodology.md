# Measurement methodology

## Scope

The experiment sweeps 17 powers-of-two working-set sizes in ascending order,
from 4 KiB (4,096 bytes) through 256 MiB (268,435,456 bytes), inclusive. Each
size uses a fresh vector of `std::uint64_t` values with element count
`bytes / sizeof(std::uint64_t)`: 512 through 33,554,432 on the current target.
One scan visits every element in order and returns an unsigned 64-bit sum.
Each measured trial batches 100 scans of the same vector.

Random values do not imply random access: the access pattern is sequential.
Initialisation and warm-up touch the data before the measured trials. This is a
repeated-access experiment, not a guaranteed cold-cache measurement.

## Current procedure

For each working-set size:

1. Reserve storage for 20 durations and 20 batch checksums, and allocate the input.
2. Initialise a new `std::mt19937_64` with seed 55 and a uniform integer
   distribution over 0–1000. Accumulate a single-scan reference checksum during
   input initialisation, independently of the scan function.
3. Run one untimed warm-up scan and compare it with the single-scan reference.
4. Run 20 trials. Reset the batch checksum before the start timestamp. Between
   two `steady_clock::now()` calls, perform 100 scans and add each returned
   checksum to the batch checksum. Store duration and checksum after timing.
5. Calculate the expected batch checksum as the single-scan reference times 100
   and validate all 20 recorded batch checksums outside timing.
6. Sort the durations and print one summary row for this size. Fresh vectors and
   checksums are used for the next size.

A warm-up or batch mismatch prints an error and exits with status 1, omitting
that size's summary and all later sizes. Earlier validated rows may already have
been printed; an incomplete sweep must not be treated as a successful full run.
Successful completion returns status 0. Warm-up results are not printed.

The fixed engine seed supports repeatability within the same implementation.
The distribution's mapping is not guaranteed identical across different C++
standard-library implementations; record the toolchain for cross-machine work.

## Timing boundaries

The timed region contains 100 scan calls, batch-checksum accumulation, and inner
loop control, including any call/return and bookkeeping overhead that remains
after optimisation. Clock-reading overhead also affects the interval. Allocation,
random generation, reference generation, result-vector insertion, validation,
sorting, and printing occur outside timing.

Durations are converted to integer nanoseconds; this does not imply a clock with
one-nanosecond resolution or accuracy. Batching amortises clock overhead across
100 scans. The batch count is fixed for every size, not adaptively calibrated.
Compare empty timing intervals with batch durations and reassess the count when
changing the experiment. Batching repeats access to the same input; it does not
flush caches or create independent cold-cache scans.

## Clock-overhead diagnostic

`benchmarks/clock_overhead.cpp` is a separate executable built in Release mode.
It reserves space for 100,000 durations, then reads `steady_clock::now()` twice
consecutively per measurement, with no scan between the readings. Duration
conversion and insertion into the results vector occur after the second reading.
After all measurements, it sorts the durations and reports their minimum,
maximum, and median. For an even count, the median averages the two middle
values using floating-point division. There is no separate warm-up phase.

This measures an approximate empty timing interval, not the exact cost of two
complete clock calls or all overhead surrounding a scan. Clock resolution can
produce zero intervals; scheduling and other interruptions can produce outliers.
Compare typical empty intervals with batch durations from the same environment.
Do not automatically subtract this baseline from measured scans or infer clock
resolution from a single sample. Repeat complete runs before drawing conclusions.

No timing threshold is enforced, and this diagnostic is not registered with
CTest. If batch durations approach the empty-interval baseline, reconsider the
scans-per-trial count and verify that repeated work survives optimisation.

## Statistics and units

- Minimum and maximum: smallest and largest of the 20 batch durations per size.
- Median: average of the two middle sorted durations for the current even trial
  count, using floating-point division.
- Median ns/element: median batch duration in ns / (100 × element count).
- Useful GB/s: (100 × input bytes) / median batch duration in ns. Bytes/ns is
  numerically equivalent to decimal GB/s.

The console header reports 20 trials, 100 scans per trial, and one warm-up scan.
Rows contain size in KiB, elements, min/median/max batch ns, median ns/element,
and useful GB/s. Median batch times use one decimal place; normalised metrics
use three. These display precisions do not imply corresponding clock accuracy.
Neither the mean nor warm-up timings are reported. Min/max describe observed
spread, not confidence intervals. Repeat full sweeps to assess run-to-run variation.

Durations are sorted in place, so their chronological order is not retained.
Preserve raw trial order before sorting if later exporting per-trial data.
KiB means 1,024 bytes and MiB means 1,048,576 bytes. Useful bytes count input
element data consumed across all scans, not measured hardware memory traffic.

## Correctness and optimisation

Every recorded batch checksum is checked against 100 times the independently
accumulated single-scan reference. Individual scan results within a batch are
not stored or checked separately; compensating errors could preserve the total. This is necessary for correctness, but does not by itself guarantee
that every source-level scan survives compiler optimisation as intended.
Storing or printing results, and placing the scan in another source file, are
not universal optimisation barriers, especially with link-time optimisation.

Before making performance claims, inspect the optimised program or adopt an
appropriate benchmark harness and understand its optimisation controls. Record
whether link-time optimisation is enabled. Do not disable optimisation simply
to obtain plausible-looking timings: that changes the operation being measured.

### Assembly checks during development

On 2026-10-04, an AppleClang 21.0.0.21000334 ARM64 Release build using
`-O3 -DNDEBUG` without link-time optimisation was inspected. In the earlier single-scan baseline, the measured loop
retained 20 calls to `sequential_scan`, each between two clock calls. The scan
used vector loads and integer SIMD additions to process eight 64-bit elements
(64 bytes) per main loop iteration, followed by reduction of the partial sums.
For the 131,072-element input, this is 16,384 vector-loop iterations per scan.
Normalisation still uses 131,072 elements, not the assembly-loop iteration count.

After batching was introduced, a separate Release inspection confirmed that
the inner loop retained 100 scan calls between the clock readings, accumulating
their results rather than reusing one scan. Current normalisation counts
100 times the input element count per trial.

These observations apply to the inspected builds, not every compiler or future
configuration. Vectorisation is intended for this optimised sequential scan;
it does not remove elements from the workload. Recheck the generated code after
changes to the scan, batching, compiler options, or link-time optimisation.
Timing overhead and repeat-run stability remain separate checks.

## Reproducibility record

For each retained experiment, record:

- Source revision, including whether there are uncommitted changes.
- CPU model, operating system/version, and installed memory.
- Compiler and C++ standard-library versions.
- CMake version, generator, build configuration, effective compiler flags, and
  link-time optimisation settings.
- Input size in bytes, element count/type, seed, value range, warm-up count,
  measured trial count, and scans per trial.
- Power mode, whether the machine is on battery, and notable background load.
- Validation status and several independent runs with their summary statistics.

The program does not collect this metadata automatically. Use an optimised
Release build and keep experimental conditions reasonably consistent. A seed
alone is not a complete reproducibility record.

## Interpretation limits

Elapsed scan time includes effects of generated instructions, vectorisation,
cache behaviour, prefetching, CPU frequency, scheduling, and other system activity.
Repeated scans may be served by caches and do not directly measure RAM bandwidth.
The size sweep does not directly identify cache boundaries. Changes across sizes
can also reflect prefetching, vectorisation, CPU frequency, scheduling, and
background load. Sizes always run in ascending order, so time-dependent changes
can affect the apparent curve. One warm-up does not guarantee cache residency
or stable CPU frequency, especially for working sets larger than the caches.

The current benchmark is a learning baseline. CTest covers empty, single-element,
and multi-element scan inputs; it does not test the sweep statistics or impose
performance thresholds. The size sweep and useful throughput are implemented.
CSV export, plots, and automatic environment-metadata collection remain planned.
