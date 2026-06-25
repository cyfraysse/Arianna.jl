# simulation.jl — Plain-language explanation

## What this file does

This file defines the **central object of Arianna**: the `Simulation` struct and the `run!` function.
Think of `Simulation` as the "conductor" of the orchestra: it holds the physical system (the particles),
the list of algorithms to run, the schedule telling each algorithm when to fire, and the loop that
advances time step by step.

---

## The Simulation struct — what it stores

| Field | Type | Meaning |
|---|---|---|
| `chains` | `Vector{S}` | Independent copies of the physical system (one per "chain") |
| `algorithms` | tuple | The list of things to do at each scheduled step (store trajectories, do MC moves, etc.) |
| `steps` | `Int` | Total number of MC steps the simulation will run across all jobs |
| `t` | `Int` | The current step (updated at every iteration of the main loop) |
| `t_start` | `Int` | The step to resume from — 0 for a fresh simulation, >0 on restart |
| `t_restart` | `Union{Int,Nothing}` | Steps per job chunk; `nothing` means the full simulation runs in one job |
| `schedulers` | tuple | For each algorithm, the sorted list of steps at which it fires |
| `counters` | `Vector{Int}` | For each algorithm, a pointer to the next scheduled step |
| `path` | `String` | The folder where all output files are written |
| `verbose` | `Bool` | Whether to print progress messages |

---

## What a scheduler is

A scheduler is just a sorted list of integers, like `[100, 200, 300, 1000]`. It tells an algorithm
"fire at step 100, then at step 200, then at 300, then at 1000". When the main loop reaches step 100,
it sees that `counters[k] = 1` points to `schedulers[k][1] = 100`, fires algorithm `k`, then
increments the counter to 2 so the next event is at step 200.

Scheduler values can exceed `steps` without error — events beyond the job window simply never fire.
This is important for restarted jobs, where schedulers are built for the full simulation but each
job only runs a subset.

---

## Constructor — what happens when you create a Simulation

```julia
Simulation(chains, algorithms, schedulers, steps; t_start=0, t_restart=nothing, path="data", verbose=false)
```

Three checks are made:
1. There must be as many schedulers as algorithms.
2. All scheduler values must be ≥ 0.
3. Each scheduler must be sorted.
4. `t_start` must be between 0 and `steps-1`.

Then:
- `t` is initialised to `t_start`.
- Each counter is initialised with `findfirst(x -> x > t_start, scheduler)` — this **fast-forwards**
  past all events that already happened before `t_start`. Without this, the simulation would try to
  fire events from the past.

### Auto-adding StoreLastFrames

The factory function (which takes a list of algorithm constructors rather than instances) has one
extra job: if `t_restart` is set and `StoreLastFrames` is not already in the algorithm list, it
adds it automatically. This guarantees that a position checkpoint always exists at the end of each
job, which is required for the next job to find where to resume from.

---

## The main loop — run!

```julia
job_end = isnothing(simulation.t_restart) ? simulation.steps :
          min(simulation.steps, simulation.t_start + simulation.t_restart)

for simulation.t in (simulation.t_start + 1):job_end
```

On a fresh single-job run (`t_restart = nothing`, `t_start = 0`): this is just `1:steps`.

On a restarted job (`t_start = 600`, `t_restart = 300`): this runs from 601 to 900.

`run!` returns `true` if the simulation is complete (`simulation.t == simulation.steps`),
`false` if more jobs are needed. The caller (the CLI) uses this to decide whether to exit with
code 0 (done) or 1 (resubmit needed).

---

## detect_restart — finding the last checkpoint

`detect_restart(path, fmt)` checks whether a `lastframe` file exists:

```julia
function detect_restart(path, fmt)
    lastframe_path = joinpath(path, "chains", "1", "lastframe$(fmt.extension)")
    isfile(lastframe_path) || return 0
    return read_t_from_lastframe(lastframe_path, fmt)
end
```

If the file exists, it reads `t` from it (the step at which the job ended). That `t` becomes the
`t_start` for the next job. If no file exists, returns 0 (fresh start).

`read_t_from_lastframe(path, fmt)` is a generic function — the default implementation works for
the `DAT` format. Other formats (like EXYZ) override it.

---

## build_schedule — helper functions

These create the sorted list of integers (the scheduler) from a compact description:

| Call | Produces |
|---|---|
| `build_schedule(steps, burn, Δt)` | Linear: `burn, burn+Δt, burn+2Δt, …, steps` |
| `build_schedule(steps, burn, base::Float)` | Logarithmic: denser at the start, sparser later |
| `build_schedule(steps, burn, block::Vector)` | Repeating block pattern |
| `build_schedule(steps, MultiOrigins(tw, N))` | Multi-tau: log-spaced bursts starting at every `tw` |

The **MultiOrigins** scheduler is important for computing correlation functions efficiently: you want
many measurement points both at short times (to see fast dynamics) and at long times (to see slow
relaxation), so you repeat a log-spaced pattern starting from many different "time origins".
