# Barrier Synchronization Benchmark Suite

A set of custom barrier synchronization algorithms implemented and benchmarked across three parallel programming models: shared-memory (OpenMP), distributed-memory (MPI), and a hybrid MPI/OpenMP combination. The project focuses on how different barrier designs — centralized vs. tree-based vs. dissemination — behave as thread and process counts scale, and why.

## What This Project Demonstrates

- Implementing classic barrier synchronization algorithms (sense-reversing, tournament, dissemination, MCS tree) from first principles, without relying on the language/runtime's built-in barrier.
- Shared-memory synchronization in C using OpenMP: atomic counters, spin-waiting, cache-line alignment, and false-sharing avoidance.
- Distributed-memory synchronization in C using MPI: non-blocking point-to-point messaging (`MPI_Isend`/`MPI_Irecv`) to implement barrier semantics across processes.
- Composing a two-level hybrid barrier that synchronizes OpenMP threads within a process and MPI processes across nodes.
- Designing and running scaling experiments (2–8 threads, 2–12 processes, combined sweeps) and reasoning about the resulting performance trends in terms of algorithm structure and hardware/network behavior.

## Tech Stack

- **Language:** C (C99 / gnu99)
- **Shared-memory parallelism:** OpenMP (GCC `libgomp`)
- **Distributed-memory parallelism:** MPI (MPICH / MVAPICH2)
- **Build:** GNU Make
- **Experiment automation:** Bash + Slurm `sbatch` scripts (written for a Slurm-based HPC cluster)

See `docs/environment.md` for the tooling a local build needs.

## Implemented Components

| Component | Location | Algorithm |
|---|---|---|
| OpenMP barrier 1 | `omp/gtmp1.c` | Centralized sense-reversing barrier |
| OpenMP barrier 2 | `omp/gtmp2.c` | Tournament (binary tree) barrier |
| MPI barrier 1 | `mpi/gtmpi1.c` | Dissemination barrier (`ceil(log2(P))` rounds) |
| MPI barrier 2 | `mpi/gtmpi2.c` | MCS-style tree barrier |
| Hybrid barrier | `combined/` | OpenMP tournament barrier (intra-process) + MPI dissemination barrier (inter-process) |

Each barrier implements the same three-function interface (`*_init`, `*_barrier`, `*_finalize`), declared in `omp/gtmp.h` and `mpi/gtmpi.h`, so the algorithm implementations are swappable behind a fixed contract.

## Repository Structure

```text
.
├── README.md
├── .gitignore
├── omp/                     # OpenMP barriers, harness, Makefile, sample sbatch scripts
├── mpi/                     # MPI barriers, harness, Makefile, sample sbatch scripts
├── combined/                # Hybrid MPI/OpenMP barrier, harness, Makefile, sample sbatch script
├── examples/                # Small standalone OpenMP/MPI tutorial programs (not the barrier suite itself)
├── scripts/                 # Slurm sweep automation (see below)
│   ├── run_openmp_sweep.sh   # generates/submits sbatch jobs sweeping OpenMP thread counts
│   ├── run_mpi_sweep.sh      # generates/submits sbatch jobs sweeping MPI process counts
│   └── run_hybrid_sweep.sh   # generates/submits sbatch jobs sweeping process x thread counts
└── docs/
    ├── environment.md               # tooling a local build needs
    ├── design-notes.md              # architecture, algorithm rationale, tradeoffs
    ├── benchmark-methodology.md     # what/how measurements were taken
    └── results-analysis.md          # summary of original experimental results
```

## Build Instructions

Each component has its own `Makefile` and no shared build system is assumed. A C compiler with OpenMP support (e.g. `gcc`) and an MPI implementation providing `mpicc`/`mpiexec` (MPICH or MVAPICH2) must be on `PATH`.

```bash
# OpenMP barriers -> builds ./omp/mp1 and ./omp/mp2
cd omp && make all

# MPI barriers -> builds ./mpi/mpi1 and ./mpi/mpi2
cd mpi && make all

# Hybrid MPI/OpenMP barrier -> builds ./combined/combined
cd combined && make all
```

`make clean` is available in each directory to remove build output.

On an environment-modules-based cluster, the `mpicc` used by `mpi/Makefile` and `combined/Makefile` is expected to come from a loaded MPI module (e.g. `module load gcc/12.3.0 mvapich2/2.3.7-1`, the module set this project was originally benchmarked with). See `docs/environment.md` for general tooling requirements, and `examples/MPI/Makefile` for the alternate `mpicc.mpich` naming convention used by some MPICH installs.

## Run Instructions

**OpenMP barriers** (single machine, scale thread count):

```bash
cd omp
./mp1 8   # sense-reversing barrier, 8 threads
./mp2 8   # tournament barrier, 8 threads
```

**MPI barriers** (single machine or multiple nodes, scale process count — pass the same count to `mpiexec`/`srun` and to the binary):

```bash
cd mpi
mpiexec -n 4 ./mpi1 4   # dissemination barrier, 4 processes
mpiexec -n 4 ./mpi2 4   # MCS barrier, 4 processes
```

On a Slurm cluster, `srun ./mpi1 4` is used instead (see `mpi/mpi1.sbatch` for a worked example, or `scripts/run_mpi_sweep.sh` to generate and submit a sweep across process counts).

**Hybrid MPI/OpenMP barrier** (scale processes via `mpiexec`/`srun`, threads per process via the argument):

```bash
cd combined
mpiexec.mpich -np 4 ./combined 8   # 4 MPI processes, 8 OpenMP threads each
```

Each harness prints the average time per barrier call (nanoseconds), computed over 100,000 iterations to smooth out timer resolution and scheduling noise. See `docs/benchmark-methodology.md` for details.

## Benchmarking Methodology (Summary)

- Timing uses `clock_gettime(CLOCK_MONOTONIC, ...)` for nanosecond resolution and immunity to wall-clock adjustments.
- Each configuration runs 100,000 barrier calls in a tight loop; the reported value is total elapsed time divided by iteration count.
- OpenMP barriers were swept from 2–8 threads on a single 8-way SMP node.
- MPI barriers were swept from 2–12 processes, one process per cluster node.
- The hybrid barrier was swept across 2–8 MPI processes, each running 2–12 OpenMP threads.

Full methodology, including known sources of measurement noise, is in `docs/benchmark-methodology.md`. A summary of the original experimental results and analysis is in `docs/results-analysis.md`.

## Verification Status

- **Static review only.** No C/OpenMP/MPI toolchain (`gcc`, `mpicc`, `make`) has been available in the environments this repository has been cleaned up in, so the build and run instructions above were verified by reading the Makefiles and harness sources, not by executing them. This has not changed since the previous cleanup pass.
- The performance figures discussed in `docs/results-analysis.md` are carried over from the original experimental write-up (originally run on a Slurm-managed HPC cluster in March 2025) — they were **not freshly re-run**.
- Job output logs and compiled binaries from the original development process have been removed from the working tree (see `.gitignore`); they were single-configuration artifacts, not the full benchmark sweep, and are not needed to build or read the project.

## Limitations

- Only two barrier algorithms are implemented per programming model, matching the original project scope — this is not an exhaustive survey of barrier designs.
- There is no automated correctness test suite (e.g., no automated check that no thread/process ever proceeds past the barrier early); correctness was validated manually via the harnesses during development.
- The `scripts/run_*_sweep.sh` scripts are designed specifically for Slurm-managed HPC environments: they generate and submit `sbatch` jobs, assume `mvapich2`/`gcc` environment modules, and use a specific queue/partition (`-q coc-ice`). They are not required to build or run the project — the Makefiles and the `mpiexec` commands in **Run Instructions** above work directly on any machine with the right tooling. Moving the sweep scripts to a different cluster or scheduler would require adjusting the queue/partition name, module names, and (for a non-Slurm system) replacing `sbatch`/`srun` with that scheduler's equivalents.
- There is no environment/container definition checked into this repository; see `docs/environment.md` for the general tooling a local build needs.
- Benchmark data reflects a single cluster and a single point in time; it is not a general performance guarantee across hardware or interconnects.

## Future Improvements

- Add automated correctness tests for each barrier implementation, independent of the timing harness.
- Add CI to at least build all three targets on every change.
- Re-run the benchmark sweep on current hardware and refresh `docs/results-analysis.md` with up-to-date, clearly dated results.
- Generalize the cluster automation scripts to work with a plain hostfile in addition to Slurm.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
