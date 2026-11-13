# IBM Qiskit Skill

A skill that teaches an AI assistant (e.g. **Bob**) to build on **IBM Qiskit**
correctly — constructing quantum circuits and operators, transpiling them to
hardware-native ISA circuits, executing with the Sampler/Estimator primitives
(locally or on IBM Quantum hardware), and post-processing results — using the
official `qiskit`, `qiskit-ibm-runtime`, and `qiskit-aer` Python libraries,
grounded in the real API surface rather than guesswork.

## What it covers

The whole skill is organized around the **four-step Qiskit pattern**:

- **Map** — `QuantumCircuit`, the gate set, `Parameter`s, the circuit library, and
  `quantum_info` operators (`SparsePauliOp`).
- **Optimize** — transpilation to **ISA circuits** with `generate_preset_pass_manager`,
  optimization levels, stages, and `apply_layout`.
- **Execute** — the **V2 primitives** (`SamplerV2`/`EstimatorV2`), PUBs, options
  (shots, `resilience_level`, error suppression/mitigation), Sessions vs Batch,
  local `qiskit.primitives` and `qiskit-aer` simulation.
- **Post-process** — drawing circuits, `plot_histogram`, `quantum_info` state
  analysis, and the all-important little-endian bit ordering.
- **Connect** — `QiskitRuntimeService`, `save_account`, the `ibm_quantum_platform`
  channel, instances/CRNs, automatic instance selection, local testing mode, and
  `fake_provider` backends.
- **REST API** — the Qiskit Runtime / IBM Quantum System API (backends + jobs),
  IAM-bearer + `Service-CRN` auth, for non-Python callers.

## Layout

```
ibm-qiskit/
├── SKILL.md              # the skill (the assistant loads this)
├── README.md             # this file
└── references/           # loaded on demand by the skill
    ├── setup-and-hardware.md
    ├── circuits-and-operators.md
    ├── transpilation.md
    ├── primitives-and-execution.md
    ├── visualization-and-postprocess.md
    └── runtime-rest-api.md            # IBM Quantum System REST API (backends + jobs)
```

## Grounding & versions

**Live-verified** against **`qiskit` 2.4.1**, **`qiskit-ibm-runtime` 0.47.0**, and
**`qiskit-aer` 0.17.2** (installed via pip), including actually running circuits —
a Bell state on a local simulator gave ~50/50 `00`/`11` and ⟨ZZ⟩ ≈ 1.0. Qiskit
changes fast (a v1.0 repackaging and a v2.0 API break), so the skill's golden rule
is to **verify signatures against the live docs / `help()`**, always use the **V2
primitives**, always **transpile to the backend's ISA** before hardware, and use
the default **`ibm_quantum_platform`** channel (the legacy `ibm_quantum` channel
was sunset 1 July 2025).

> 🔒 IBM Quantum API keys are secrets — store them via `save_account` or an env
> variable, never hardcode or commit them.

## Related skills

Qiskit is the **quantum compute layer**. Pair it with the **TechZone** skill when
you need to provision an IBM Quantum environment first; this skill picks up once
you have an API key. Qiskit's outputs (counts, expectation values) are ordinary
classical data for any downstream analysis or report.
