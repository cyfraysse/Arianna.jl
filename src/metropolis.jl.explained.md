# metropolis.jl — Plain-language explanation

## What this file does

This file contains the **heart of the Monte Carlo algorithm**: the code that proposes a random
change to the system and decides whether to accept or reject it. It also defines the `Metropolis`
algorithm struct, which wires everything together into an Arianna-compatible algorithm.

---

## What a Monte Carlo step is

The idea: instead of solving equations of motion (like molecular dynamics), we randomly *propose*
a change to the system (e.g. move a particle), then decide whether to keep it based on how much
it changes the energy.

One step looks like this:
1. **Propose**: draw a random move (a displacement, a swap, …).
2. **Evaluate**: compute how much the energy changes: `Δ log p = log p(new) - log p(old)`.
3. **Accept or reject**: flip a biased coin with probability `min(1, exp(Δ log p))`.
   - If `Δ log p > 0`: energy went down → always accept.
   - If `Δ log p < 0`: energy went up → accept with probability `exp(Δ log p)` (less than 1).
4. **Revert** if rejected: undo the move.

This rule — called the **Metropolis criterion** — guarantees that after many steps, the
system visits configurations with the correct Boltzmann probability `p ∝ exp(-E/kT)`.

---

## Action and Policy — two separate concepts

Arianna splits a move into two objects:

| Object | What it represents | Example |
|---|---|---|
| `Action` | The *type* of move (what changes) | `Displacement` — move one particle |
| `Policy` | The *distribution* used to propose the move | `SimpleGaussian` — draw from N(0, σ²) |

For a move to be valid, the **Metropolis acceptance ratio** must include both the energy change
*and* the asymmetry of the proposal:

```
log α = Δ log p + log q(backward) - log q(forward)
```

where `q(forward)` is the probability of proposing the move you just made, and `q(backward)` is
the probability of proposing its reverse. For a symmetric proposal (Gaussian centered at 0),
these cancel. For asymmetric proposals (like `EnergyBias`), they don't, and this correction
makes the algorithm exact regardless of the proposal shape.

---

## Acceptance rules

Two rules are provided:

**MetropolisRule** (the classic):
```
α = min(1, exp(log_acceptance_ratio))
```

**BarkerRule** (smoother, less common):
```
α = 1 / (1 + exp(-log_acceptance_ratio))
```

---

## Move struct — bundling action + policy + weight

A `Move` wraps an action, a policy, its parameters, and a weight:

```julia
Move(action, policy, parameters, weight)
```

The `weight` controls how often this move is chosen in a sweep. If you have two moves with
weights `[0.7, 0.3]`, the first is chosen 70% of the time.

---

## mc_step! — one proposal-accept-reject cycle

Steps:
1. `sample_action!` — draw a proposed move.
2. `log_proposal_density(forward)` — log probability of having proposed this move.
3. `perform_action!` — apply the move; returns `(x1, x2)`, the old and new state.
4. `delta_log_target_density(x1, x2)` — compute `Δ log p`.
5. `invert_action!` — logically reverse the move (for proposal density calculation).
6. `log_proposal_density(backward)` — log probability of proposing the reverse.
7. Compute `log α = Δ log p + log q(backward) - log q(forward)`.
8. If `rand() < α`: keep (return 1). Otherwise: `revert_action!` and return 0.

Note: `invert_action!` modifies the action object (flips the displacement sign).
`revert_action!` modifies the *system* (undoes the move). These are different things.

---

## mc_sweep! — one full sweep

A sweep consists of `mc_steps` individual proposals, each picking a move randomly from the pool
(weighted by `move.weight`). In ParticlesMC, `mc_steps = N` (number of particles), so each
particle gets one chance to move per sweep on average.

---

## Metropolis struct — the Arianna algorithm

`Metropolis` is the `AriannaAlgorithm` that runs MC sweeps at every scheduled step.

Key fields:
- `pools`: one independent pool of moves per chain (avoids race conditions in parallel runs).
- `rngs`: one random number generator per chain, seeded as `seed + c - 1`. Reproducible.
- `parallel`: if true, sweeps over all chains run in parallel using Julia threads.

The `Metropolis` algorithm is **unchanged** by the restart procedure — it simply starts running
MC sweeps from `t_start + 1` as dictated by the simulation loop. The RNG is re-seeded from
scratch each job (state not saved), which is physically correct but not bit-for-bit reproducible.

---

## StoreAcceptance — tracking acceptance rates

Writes the running acceptance rate of each move type at every scheduled step.

**On restart:** `initialise` checks `simulation.t_start > 0` and opens files in append mode.
Acceptance counters (`total_calls`, `accepted_calls`) start fresh each job — the reported rate
is the within-job rate, which is what matters for tuning move parameters.
