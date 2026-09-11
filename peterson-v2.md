# Peterson v2: structural RM composition

The [v2 Python example](../examples/peterson_v2.py) has two independent process
classes. The shared compiler builds two persistent native `zrth.Module` objects,
combines them with `Module.compose`, reads back their atoms, and emits Lean
definitions of their relational parallel composition. There is no combined
Python `tick` function and no application scheduler parameter.

[Open the generated side-by-side comparison](peterson-v2.html). It contains the
actual Python, native RM atoms, generated Lean, and complete accepted evidence.
The earlier [FMSD99 comparison](fmsd99.md) records v1; its Peterson example is
superseded by this version.

## What changed

| Concern | v1 Peterson | v2 Peterson |
|---|---|---|
| Application code | One class updating both processes | Separate `P1` and `P2` classes, each owning local `pc` and `x` |
| Native RM | Stateless graph for a combined transition | Two persistent atoms retained by native parallel composition |
| Choice | External `run1`, `run2` arguments | Internal `AnyBool` wires select advance or sleep independently |
| Initialization | Both flags fixed to false | Both Boolean constructor arguments range over their full domains |
| Old values | Manual snapshot in the combined method | Port bindings read the round's old valuation |
| Updated values | No composition-level support | Explicit `Await(variable)` ports, with dependency ordering |
| Structural validation | No whole-module ownership check | Closed interface, unique controllers, declared dependencies, acyclic awaits |
| Lean model | A deterministic step indexed by an action | Atom relations and parallel conjunction, with relational reachability |
| Correspondence | Combined extracted function equals its graph | Each extracted component relation equals the native atom relation; equality lifts to composition |
| Match to the paper | Finite regression oracle | Lean equivalence to independently written initial and round predicates |
| Safety | One concrete initialization, arbitrary run flags | All four initial flag valuations, all choices, unbounded executions |

The source is [Alur and Henzinger, *Reactive Modules*, Figure 2 and the definitions
of atoms and parallel composition](https://www.cis.upenn.edu/~alur/FMSD99.pdf).
The process atoms read old values only; their parallel round relation therefore
conjoins their local update relations. Both may advance during the same round.
The control-location encoding is `0 = outCS`, `1 = reqCS`, `2 = inCS`; an invariant
proves that every reachable control location stays in this domain.

## User-facing specification

`Component` identifies a plain Python class, its atomic transition methods,
the global names of its controlled fields, and bindings for its scalar input
parameters. `Composition` identifies the property state schema and the components.
The schema has annotations only; it is not another implementation of the protocol.

```python
Component(P1, [P1.advance],
          controls={"pc": "pc1", "x": "x1"},
          inputs={"pc2": "pc2", "x2": "x2"},
          stutter=True)
```

An input string reads the old value. `Await("pc2")` would read the updated value
and introduce an ordering dependency; that would change this protocol's meaning.
The verifier rejects cycles, including self-await, rather than choosing a
sequential interpretation. Within one method, ordinary Python assignment order
is preserved. All of the method's final local fields become one atomic update.

The selected methods are finite alternative actions. `stutter=True` adds a sleep
alternative. Each atom makes its own choice, so a global scheduler need not be
supplied by the application. Boolean constructor parameters range over both
values by default. Other initializer parameters require explicit finite
`initial_inputs` domains; restricting a domain restricts the initial-state claim.

The optional `initial_relation` and `step_relation` are independently authored
Boolean predicates over states. They request equivalence proofs, not sample
executions. In Peterson these predicates specify the paper's permitted rounds.
They are properties, not additional transition implementations used by the compiler.

## What Lean checks

1. Every extracted constructor specialization, process method, and predicate
   agrees with its exported deterministic action graph for all typed inputs.
2. Each source atom relation agrees with its native RM atom after composition.
   This includes all initialization alternatives, action alternatives, and sleep.
3. The source composition equals the native composition translated into Lean.
   Generic `initial_parallel` and `step_parallel` theorems express conjunction.
4. The compiled atom metadata covers every state coordinate exactly once and
   respects an acyclic await order. These are kernel-checked structural facts.
5. The initial relation is nonempty and every state has a successor. Witnesses
   are computed from the exported native graphs in await order.
6. The joint invariant holds initially and is preserved by every composed round.
   Generic induction proves mutual exclusion over arbitrary finite prefixes;
   `always_safe` covers every position of any infinite execution.
7. The compiled initial and round relations are equivalent to `paper_initial`
   and `paper_round`. These equivalences also lift to extracted source execution.

The strengthening predicates `control_locations` and `priority` are proved as
part of the invariant, not assumed as axioms. Proof generation is shared with
other compositions. It does not contain a Peterson-specific proof script or
recognize process names. The await regression example uses the same machinery
for an unbounded integer incrementer and a copying component.

Accepted proofs are audited with `#print axioms`. Only `propext`,
`Classical.choice`, and `Quot.sound` are permitted; `sorryAx` and unchecked
native computation are rejected. Failure or timeout yields an unsuccessful
report, never an assumed proof. V2 does not yet generate counterexample proofs
when induction fails; such obligations remain `unknown`.

## Execution checks and limits

`test_composition.py` compares native successors with separate Python calls and
the paper relation over all 36 valid control/flag states. It checks all four
initial states, simultaneous moves, old versus awaited reads, conflicting
controllers, missing/ill-typed ports, and await cycles. It also runs the Lean
proofs, rejects an incorrect old-read pipeline invariant, and verifies that a
corrupted native initialization cannot pass source correspondence. These checks
supplement the Lean proofs.

This is a closed, finite-choice, discrete RM fragment: one atom per Python class,
scalar `int`/`bool` state, finite constructor choices, and acyclic read/await
composition for updates. Initialization inputs are independent finite choices;
initial actions that await another atom's initialization are not supported.
It rejects unowned external variables, unsupported Python syntax,
and transitions returning values instead of exposing fields. Constructor and
native choice expansion are capped at 256 alternatives per atom/block. No
fairness, liveness, hiding/refinement theory, timing, network transport, or
distributed-memory implementation is verified here. The Python methods must
execute atomically under the stated port/round semantics.

The source extractor, type/name bindings, native exporter (including the
interpretation of `AnyBool` as either Boolean), Lean semantics definitions, and
Lean kernel remain trust boundaries. Native `AnyBool` wires are exported over
their complete finite domain; these are internal choices, not external inputs.
The Rust implementation of composition itself is not proved in Lean: its actual
result is read back and checked against source composition. Integer operators
have mathematical, unbounded semantics; this is not a proof about CPython,
libtorch machine arithmetic, or arbitrary concurrent Python execution.

## Reproduce

From the repository root, with the existing formal environment installed:

```sh
formal/.venv/bin/python -m rmverify examples.peterson_v2:peterson --out formal/.rmverify/v2 --timeout 120
formal/.venv/bin/python formal/test_composition.py -v
formal/.venv/bin/python formal/composition_report.py
```

The CLI prints its evidence directory. Inspect `module.rm`, `Translation.lean`,
`Invariants.lean`, `InitialRelation.lean`, `StepRelation.lean`, and their logs.
`artifact.json` records source/tool hashes, the pinned upstream revision,
component metadata, and the actual exported graphs. The generated comparison
embeds these same files and checks source hashes before replacing the report.

For an independent kernel recheck, enter that evidence directory and run:

```sh
lake build Semantics
lake env lean -j1 Translation.lean -o .lake/build/lib/lean/Translation.olean
lake env lean -j1 Invariants.lean
lake env lean -j1 InitialRelation.lean
lake env lean -j1 StepRelation.lean
```
