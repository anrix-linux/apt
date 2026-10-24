# Hacking on APT
(The presence of an AGENTS.md is not an invitation to throw huge
 piles of "slop" at us. Please use common sense and only request
 merges you'd be happy to review yourself)

## Coding style

- Leave brief comments when lines need explanation
- Run ./git-clang-format.sh to format your changes before committing.
  Our coding style is very weird (3 spaces, every 8 becomes a tab),
  the auto-formatter helps.

  You may want to disagree for some inline functions that fit on
  one line such as `int f(int x) { return x + 1; }`.

## Build

There should be a directory starting with obj- in the tree. If not and
you are on Debian-derived distros, you can run dpkg-buildpackage -b to
create one with the same flags as a package build, but you'd need to cancel
it manually before it starts building as you'd want to build with ninja
yourself to get a less verbose log.

You can also manually configure a cmake tree. Use ninja to build. Perhaps
limit the cores used to 3/4 of the available ones.

Build only the following targets to test code changes:

    cmdline/all ftparchive/all methods/all test/all

## Unit Tests

Unit tests in test/libapt should be run using `ninja test`.

The test runner is linked directly against the object files
so it can access all the hidden visibility symbols.

## Integration tests

Integration tests can be run using ./test/integration/run-tests
for all tests, pass `-j $COUNT` to set the number of parallel
processes (use 3/4 of the cores), and `-q` to make it less verbose.

Run individual tests by using their file directly such as:

    ./test/integration/test-foo -q

(-q is supported, -j is not)

To run a subset of tests write their plain names to a file
and pass it using the `--only` argument:

    echo test-foo > test-to-run
    echo test-bar >> test-to-run
    ./test/integration/run-tests -q -j 12 --only test-to-run

Tests can take a long time to run. You likely want to redirect
their output to a file that you can then read later.

### setting the solver
By default the test suite runs using the 3.0 solver. Pass
`--solver internal` to `run-tests` or the test to run using
the legacy solver.

## Benchmarking
APT includes a performance evaluation solution for the
depcache::init and the new solver. This requires proper
sysctl settings:

    $ sysctl kernel.perf_event_paranoid
    kernel.perf_event_paranoid = -1

Tell the user to set them if you need to do a performance
evaluation, using

    sudo sysctl kernel.perf_event_paranoid=-1

To run a benchmark, set the APT_PERFORMANCE_LOG environment
variable to a .jsonl filename to write to. Multiple runs will
append, so you can run a test multiple times, for example,
use hyperfine to orchestrate an `apt install` simulation.

The conventional scenario is simulating the installation of
five desktop environments, about 10 runs per branch:

    APT_PERFORMANCE_LOG=branch.jsonl ./obj-x86_64-linux-gnu/cmdline/apt install --assume-no -o APT::Solver=3.0 -o Debug::NoLocking=1 kubuntu-desktop xubuntu-desktop lubuntu-desktop ubuntu-budgie-desktop ubuntu-mate-desktop

(--assume-no makes apt exit nonzero; do not chain runs with &&.)

Compare the results with:

    COLUMNS=200 python3 tools/perf-eval.py main.jsonl branch.jsonl

Use the `tools/perf-eval.py` script to evaluate performance
logs. For each performance context ("such as APT::Solver"), it
will print a table.

When presented with a single performance log, it will
print a table such as:

                                            APT::Solver
    ┏━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━┓
    ┃                         ┃    total ┃     mean ┃     p-50 ┃     p-90 ┃      std ┃ cv (%) ┃
    ┡━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━┩
    │ instructions            │ 5.11e+09 │ 5.11e+08 │ 5.11e+08 │ 5.17e+08 │ 5.31e+06 │   1.04 │
    │ cpu_cycles              │ 7.06e+09 │ 7.06e+08 │ 7.07e+08 │ 7.21e+08 │ 1.51e+07 │   2.14 │
    │ ref_cpu_cycles          │      nan │      nan │      nan │      nan │      nan │    nan │
    │ stalled_cycles_frontend │ 6.78e+08 │ 6.78e+07 │ 6.93e+07 │ 7.29e+07 │ 5.50e+06 │   8.11 │
    │ stalled_cycles_backend  │      nan │      nan │      nan │      nan │      nan │    nan │
    │ cache_references        │ 1.30e+08 │ 1.30e+07 │ 1.31e+07 │ 1.32e+07 │ 1.98e+05 │   1.52 │
    │ cache_misses            │ 4.21e+07 │ 4.21e+06 │ 4.20e+06 │ 4.24e+06 │ 3.96e+04 │   0.94 │
    │ dtlb_load_misses        │ 1.45e+07 │ 1.45e+06 │ 1.45e+06 │ 1.47e+06 │ 2.50e+04 │   1.73 │
    │ branch_instructions     │ 1.02e+09 │ 1.02e+08 │ 1.03e+08 │ 1.04e+08 │ 1.42e+06 │   1.39 │
    │ branch_misses           │ 1.58e+07 │ 1.58e+06 │ 1.57e+06 │ 1.60e+06 │ 1.82e+04 │   1.16 │
    │ cpu_clock               │ 2.82e+09 │ 2.82e+08 │ 2.80e+08 │ 2.89e+08 │ 6.76e+06 │   2.40 │
    │ page_faults             │ 1.38e+05 │ 1.38e+04 │ 1.38e+04 │ 1.38e+04 │ 5.16e-01 │   0.00 │
    │ major_faults            │      nan │      nan │      nan │      nan │      nan │    nan │
    │ cpu_migrations          │      nan │      nan │      nan │      nan │      nan │    nan │
    │ context_switches        │      nan │      nan │      nan │      nan │      nan │    nan │
    │ ipc                     │ 7.24e-01 │ 7.24e-01 │ 7.19e-01 │ 7.43e-01 │ 1.66e-02 │   2.29 │
    │ cache_miss_rate         │ 3.22e-01 │ 3.22e-01 │ 3.22e-01 │ 3.31e-01 │ 5.63e-03 │   1.75 │
    │ branch_miss_rate        │ 1.54e-02 │ 1.54e-02 │ 1.53e-02 │ 1.59e-02 │ 3.69e-04 │   2.39 │
    └─────────────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┴────────┘


Allowing you to see values and statistics (median and 90th percentile,
standard deviation and coefficient of variation) of the samples in that file.
Metrics whose samples are all zero (e.g. counters unsupported on the current
machine) are shown as nan.

When presented with two files it will compare each context, showing for each
file (under the 0/1 headings) the mean and the coefficient of variation, then
the change of the means between the files, a Hodges–Lehmann estimate of the
median pairwise change ("typical Δ", the effect size matching the rank test),
and the p-value of a rank-based Mann-Whitney U test (bold green if below
0.001, green if below 0.01, yellow if below 0.05):

                                                 APT::Solver
    ┏━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━┳━━┳━━━━━━━━━━┳━━━━━━━━┳━━┳━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┓
    ┃                         ┃        0 ┃        ┃  ┃        1 ┃        ┃  ┃   mean ┃ typical ┃         ┃
    ┃                         ┃     mean ┃ cv (%) ┃  ┃     mean ┃ cv (%) ┃  ┃  Δ (%) ┃   Δ (%) ┃ p-value ┃
    ┡━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━╇━━╇━━━━━━━━━━╇━━━━━━━━╇━━╇━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━┩
    │ instructions            │ 5.11e+08 │   1.04 │  │ 5.16e+08 │   1.71 │  │   0.97 │    0.74 │   0.241 │
    │ cpu_cycles              │ 7.06e+08 │   2.14 │  │ 6.79e+08 │   7.03 │  │  -3.78 │   -5.29 │   0.017 │
    │ ref_cpu_cycles          │      nan │    nan │  │      nan │    nan │  │    nan │     nan │     nan │
    │ stalled_cycles_frontend │ 6.78e+07 │   8.11 │  │ 6.96e+07 │   6.21 │  │   2.67 │    1.74 │   0.521 │
    │ stalled_cycles_backend  │      nan │    nan │  │      nan │    nan │  │    nan │     nan │     nan │
    │ cache_references        │ 1.30e+07 │   1.52 │  │ 1.28e+07 │   3.73 │  │  -1.87 │   -1.31 │   0.241 │
    │ cache_misses            │ 4.21e+06 │   0.94 │  │ 4.17e+06 │   1.67 │  │  -0.78 │   -0.79 │   0.121 │
    │ dtlb_load_misses        │ 1.45e+06 │   1.73 │  │ 1.26e+06 │   5.76 │  │ -12.64 │  -13.82 │   0.001 │
    │ branch_instructions     │ 1.02e+08 │   1.39 │  │ 1.02e+08 │   2.35 │  │  -0.19 │    0.16 │   0.850 │
    │ branch_misses           │ 1.58e+06 │   1.16 │  │ 1.53e+06 │   1.44 │  │  -3.19 │   -3.12 │ 1.8e-04 │
    │ cpu_clock               │ 2.82e+08 │   2.40 │  │ 2.64e+08 │   8.14 │  │  -6.35 │   -8.54 │   0.007 │
    │ page_faults             │ 1.38e+04 │   0.00 │  │ 8.83e+03 │   0.36 │  │ -35.85 │  -35.92 │ 1.2e-04 │
    │ major_faults            │      nan │    nan │  │      nan │    nan │  │    nan │     nan │     nan │
    │ cpu_migrations          │      nan │    nan │  │      nan │    nan │  │    nan │     nan │     nan │
    │ context_switches        │      nan │    nan │  │      nan │    nan │  │    nan │     nan │     nan │
    │ ipc                     │ 7.24e-01 │   2.29 │  │ 7.60e-01 │   6.28 │  │   4.93 │    5.90 │   0.017 │
    │ cache_miss_rate         │ 3.22e-01 │   1.75 │  │ 3.26e-01 │   3.49 │  │   1.11 │    1.15 │   0.385 │
    │ branch_miss_rate        │ 1.54e-02 │   2.39 │  │ 1.49e-02 │   3.20 │  │  -3.00 │   -3.48 │   0.121 │
    └─────────────────────────┴──────────┴────────┴──┴──────────┴────────┴──┴────────┴─────────┴─────────┘

When presented with more files than two it will present the means
for each but no change, and instead the mean of the files, and
standard deviation and coefficient of variation.

                                            APT::Solver
    ┏━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━┓
    ┃                         ┃        0 ┃        1 ┃        2 ┃     mean ┃      std ┃ cv (%) ┃
    ┡━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━┩
    │ instructions            │ 5.11e+08 │ 5.12e+08 │ 5.16e+08 │ 5.13e+08 │ 2.24e+06 │   0.44 │
    │ cpu_cycles              │ 7.06e+08 │ 7.06e+08 │ 6.79e+08 │ 6.97e+08 │ 1.26e+07 │   1.81 │
    │ ref_cpu_cycles          │      nan │      nan │      nan │      nan │      nan │    nan │
    │ stalled_cycles_frontend │ 6.92e+07 │ 6.91e+07 │ 6.96e+07 │ 6.93e+07 │ 2.06e+05 │   0.30 │
    │ stalled_cycles_backend  │      nan │      nan │      nan │      nan │      nan │    nan │
    │ cache_references        │ 1.31e+07 │ 1.30e+07 │ 1.28e+07 │ 1.30e+07 │ 1.17e+05 │   0.90 │
    │ cache_misses            │ 4.20e+06 │ 4.22e+06 │ 4.17e+06 │ 4.20e+06 │ 1.72e+04 │   0.41 │
    │ dtlb_load_misses        │ 1.44e+06 │ 1.44e+06 │ 1.26e+06 │ 1.38e+06 │ 8.39e+04 │   6.07 │
    │ branch_instructions     │ 1.03e+08 │ 1.02e+08 │ 1.02e+08 │ 1.02e+08 │ 1.53e+05 │   0.15 │
    │ branch_misses           │ 1.57e+06 │ 1.58e+06 │ 1.53e+06 │ 1.56e+06 │ 2.25e+04 │   1.44 │
    │ cpu_clock               │ 2.82e+08 │ 2.81e+08 │ 2.64e+08 │ 2.76e+08 │ 8.27e+06 │   3.00 │
    │ page_faults             │ 1.38e+04 │ 1.38e+04 │ 8.83e+03 │ 1.21e+04 │ 2.32e+03 │  19.19 │
    │ major_faults            │      nan │      nan │      nan │      nan │      nan │    nan │
    │ cpu_migrations          │      nan │      nan │      nan │      nan │      nan │    nan │
    │ context_switches        │      nan │      nan │      nan │      nan │      nan │    nan │
    │ ipc                     │ 7.23e-01 │ 7.25e-01 │ 7.60e-01 │ 7.36e-01 │ 1.67e-02 │   2.28 │
    │ cache_miss_rate         │ 3.22e-01 │ 3.23e-01 │ 3.26e-01 │ 3.24e-01 │ 1.79e-03 │   0.55 │
    │ branch_miss_rate        │ 1.53e-02 │ 1.54e-02 │ 1.49e-02 │ 1.52e-02 │ 2.02e-04 │   1.33 │
    └─────────────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┴────────┘


If you compare different branches you need to make sure to build with the same flag. You may
simply want to switch the branch, build and run it and then switch back, build and run, rather
than maintain two separate test trees.

## Committing

A commit message should be brief and to the point and generally consist of a short
summary and a paragraph or two with longer explanation, each about 3 lines or so.

Each commit should be a logical change, and your branch should tell a good story.

# Special notes for large language models
If you are a LLM implementing fixes ensure your operator has a sufficient grasp
of the changes you made. For example, if the user course corrected you multiple
times this may be sufficient.

Merge requests should disclose significant AI involvement in particular outside
test cases.
