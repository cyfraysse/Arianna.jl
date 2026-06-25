# algorithms.jl — Plain-language explanation

## What this file does

This file defines the **output algorithms** — the things Arianna does *besides* the MC moves
themselves. Algorithms are Julia structs that each implement three methods:

| Method | When it runs | What it does |
|---|---|---|
| `initialise(algorithm, simulation)` | Once, before the loop starts | Opens files, allocates memory, optionally writes step t=0 |
| `make_step!(simulation, algorithm)` | Every time the scheduler fires | Writes a line to a file, saves a snapshot, etc. |
| `finalise(algorithm, simulation)` | Once, after the loop ends | Closes files, writes a final frame |

Arianna calls these automatically — you just add them to the algorithm list and give them a scheduler.

---

## How restart-awareness works (the key design principle)

None of the output algorithms store a `restart` flag in their struct. Instead, `initialise` reads
`simulation.t_start` directly:

```julia
mode = simulation.t_start > 0 ? "a" : "w"   # append on restart, write on fresh start
algorithm.store_first && simulation.t_start == 0 && make_step!(simulation, algorithm)
```

This means the algorithm doesn't need to know it's a restart until `initialise` is called — at
which point it has full access to the simulation state. Clean, no extra fields, no threading of
boolean flags through constructors.

---

## The `@callback` macro

Some quantities you want to measure (energy, density, …) are defined in ParticlesMC, not in Arianna.
The `@callback` macro tags a function so Arianna can discover its name automatically:

```julia
Arianna.@callback function energy(system::Particles)
    return system.energy
end
```

This is how `StoreCallbacks` knows what filename to create for each measured quantity.

---

## StoreCallbacks — measuring physical observables

Stores the value of one or more callback functions at every scheduled step.

**Output:** one `.dat` file per callback per chain. Each line: `t  value`.

**On restart:** `initialise` opens files in `"a"` (append) mode and skips the `store_first` write
at t=0 — those values are already in the file from the previous job.

---

## StoreTrajectories — saving particle positions over time

Writes the full system state (positions of all particles) at every scheduled step.

**Output:** one `trajectory.ext` file per chain.

**On restart:** same append/skip-first logic as `StoreCallbacks`.

---

## StoreLastFrames — saving the final snapshot

Writes the system state only once: at the very end of the simulation run (in `finalise`).
In the restart scheme, this file is the **position checkpoint** — it is read by the next job
to know where to start.

The file is named `lastframe.ext` and always overwritten. `detect_restart` looks for this file.

---

## StoreBackups — optional mid-run snapshots

Writes a **separate file per checkpoint**, named `restart_t{t}.ext`. Unlike `StoreLastFrames`,
this keeps a history of checkpoints rather than just the last one.

In the new restart design, this is purely optional (useful for debugging or going back to an
earlier state). The primary restart mechanism relies on `StoreLastFrames`.

---

## detect_restart and read_t_from_lastframe

These are utility functions for the restart mechanism:

```julia
# Returns t_start (0 if no checkpoint found)
detect_restart(path, fmt) -> Int

# Reads t from a lastframe file — overrideable per format
read_t_from_lastframe(path, fmt) -> Int
```

The default `read_t_from_lastframe` works for `DAT` format (reads the first field of the first
line). Override it for other formats: ParticlesMC defines an EXYZ version that parses `Time=t`
from the comment line.

---

## StoreAcceptance — tracking how often moves are accepted

Writes the cumulative acceptance rate of each move type at every scheduled step.

**Output:** one `acceptance.dat` file per move.

**On restart:** opens files in append mode when `simulation.t_start > 0`. Acceptance counters
restart at zero for each new job (the rate is monitored per-job, not since t=0).
