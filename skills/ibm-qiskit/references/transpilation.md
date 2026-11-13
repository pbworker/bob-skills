# Optimize — transpilation & ISA circuits

> Verified against `qiskit` 2.4.1. Step 2 of the Qiskit pattern: turn abstract
> circuits into **ISA circuits** for a specific backend.

## Why: the ISA circuit

IBM hardware runs only **ISA (Instruction Set Architecture) circuits**:
- only the backend's **basis gates** (e.g. `rz`, `sx`, `x`, `ecr`/`cz`), and
- multi-qubit gates only between **physically connected** qubits (the **coupling
  map**).

An abstract circuit (arbitrary gates, all-to-all connectivity) must be
**transpiled** to meet these constraints before a hardware `run()`. Transpilation
also reduces depth/gate count to fight noise.

## The recommended path: a preset pass manager

```python
from qiskit.transpiler import generate_preset_pass_manager

pm = generate_preset_pass_manager(
    optimization_level=1,        # 0..3
    backend=backend,             # supplies target/basis_gates/coupling_map
    seed_transpiler=42,          # reproducibility (some passes are stochastic)
)
isa_circuit = pm.run(qc)
```

`generate_preset_pass_manager(...)` parameters (verified):
`optimization_level`, `backend`, `target`, `basis_gates`, `coupling_map`,
`initial_layout`, `layout_method`, `routing_method`, `translation_method`,
`scheduling_method`, `approximation_degree`, `seed_transpiler`, … You can target a
backend **or** specify `basis_gates`/`coupling_map`/`target` directly (e.g. for a
custom or fake backend).

### Optimization levels
| Level | Use |
|---|---|
| `0` | No optimization — just make it runnable. Fastest. |
| `1` | Light optimization (default-ish). Good for quick iterations. |
| `2` | Medium — more gate cancellation/commutation. |
| `3` | Heaviest — best depth/error reduction, slowest transpile. Use for hardware runs that matter. |

## Lay out the observables to match

After transpiling, an **Estimator** observable must be mapped onto the same
physical qubits as the transpiled circuit:

```python
isa_observables = [ob.apply_layout(isa_circuit.layout) for ob in observables]
```
Forgetting this gives silently wrong expectation values — it's a top gotcha.

## The simpler helper: `transpile()`

```python
from qiskit import transpile
isa = transpile(qc, backend=backend, optimization_level=3, seed_transpiler=42)
```
`transpile()` is a one-shot convenience; `generate_preset_pass_manager(...).run(...)`
is preferred when you transpile many circuits (reuse the pass manager) or want to
inspect/modify stages.

## The stages (what a preset pass manager does)
1. **Init** — unroll/normalize.
2. **Layout** — assign virtual → physical qubits (`layout_method`:
   `trivial`/`dense`/`sabre`).
3. **Routing** — insert `swap`s to satisfy the coupling map (`routing_method`:
   `sabre`/`lookahead`/`stochastic`).
4. **Translation** — rewrite to the backend's **basis gates**.
5. **Optimization** — cancel/merge gates, reduce depth.
6. **Scheduling** (optional) — add timing/delays (`scheduling_method`: `asap`/`alap`).

Inspect results: `isa_circuit.depth()`, `isa_circuit.count_ops()`,
`isa_circuit.layout`.

## Reading the target

```python
print(backend.target.operation_names)     # basis gates
print(backend.coupling_map)               # connectivity (list of qubit pairs)
print(backend.num_qubits)
```

## Custom passes & plugins (advanced)
- Build a bespoke `PassManager` from `qiskit.transpiler.passes` and `.run(qc)`.
- Register stage **plugins** (entry points) to swap layout/routing/translation
  algorithms via `generate_preset_pass_manager(..., layout_method="my_plugin")`.
- IBM's AI-powered transpiler passes live in **`qiskit-ibm-transpiler`** (separate
  package) for further hardware-aware optimization.

## Constraints recap
- **Always transpile for the target backend before hardware execution.**
- **`apply_layout` observables** after transpiling.
- Transpilation can be **stochastic** — set `seed_transpiler` for reproducibility.
- Local simulators are more forgiving, but transpiling still validates your circuit
  against a realistic ISA (especially with a `fake_provider` backend).
