---
name: ibm-qiskit
description: >-
  Build, transpile, simulate, and run quantum programs with the IBM Qiskit
  Python SDK and IBM Quantum hardware. Use this whenever the user mentions
  Qiskit, `qiskit`, quantum circuits, qubits/gates, `QuantumCircuit`, quantum
  computing on IBM, IBM Quantum Platform, `qiskit-ibm-runtime`, the Sampler /
  Estimator primitives, transpilation / pass managers / ISA circuits, quantum
  simulators (`qiskit-aer`, AerSimulator, statevector), `SparsePauliOp` /
  observables / `quantum_info`, the circuit library, OpenQASM 2/3 interop,
  error suppression / mitigation (resilience levels), Sessions / Batch
  execution modes, local testing mode, the Qiskit Runtime REST API (IBM Quantum
  System API — backends and jobs), instance / CRN / channel setup, or
  visualizing circuits and results. Covers the Qiskit SDK (primary),
  `qiskit-ibm-runtime` (hardware access + primitives + REST), and `qiskit-aer`
  (local simulation), built around the four-step Qiskit pattern:
  Map → Optimize → Execute → Post-process.
metadata:
  enabled: true
---

# IBM Qiskit — Build · Transpile · Simulate · Run on Quantum Hardware

Authoritative, end-to-end guide for quantum programming with **IBM Qiskit**,
grounded in the real SDKs (not guesswork). Its core is the **`qiskit` SDK**
(circuits, operators, transpilation, local primitives); **`qiskit-ibm-runtime`**
runs work on **IBM Quantum** hardware via the Sampler/Estimator primitives; and
**`qiskit-aer`** provides fast local simulation.

> **Golden rule:** Qiskit moves fast and had a **major v1.0 repackaging** plus a
> **v2.0** API break. The class/method specifics here were verified live against
> **`qiskit` 2.4.1**, **`qiskit-ibm-runtime` 0.47.0**, and **`qiskit-aer` 0.17.2**.
> When a signature, import path, or option is uncertain — or you're on a
> different version — **verify against the live tool**: `pip show qiskit`,
> `python -c "import qiskit; help(qiskit.QuantumCircuit.<m>)"`, and the live docs
> (§10). Two things especially deprecate: **primitive V1 → V2** (always use
> `SamplerV2`/`EstimatorV2`) and the **`ibm_quantum` channel** (sunset 1 Jul 2025;
> use the default `ibm_quantum_platform`). Never run on real hardware a circuit
> that hasn't been transpiled to the backend's ISA.

---

## 1. Mental model — the Qiskit pattern (Map → Optimize → Execute → Post-process)

**Every** Qiskit workload follows the same four steps. Internalize this; it's the
spine of the whole SDK and every guide.

| Step | What you do | Primary API |
|---|---|---|
| **1. Map** | Turn your problem into **circuits** and **operators** | `QuantumCircuit`, `quantum_info.SparsePauliOp`, `circuit.library` |
| **2. Optimize** | **Transpile** abstract circuits to an **ISA circuit** for a specific backend (basis gates + coupling map) | `generate_preset_pass_manager(...)` → `pm.run(circuit)` |
| **3. Execute** | Run the ISA circuit with a **primitive** (Sampler → bitstrings, Estimator → expectation values), locally or on hardware | `qiskit_ibm_runtime.SamplerV2`/`EstimatorV2`, or local `qiskit.primitives` / `qiskit_aer` |
| **4. Post-process** | Turn raw results into the answer: counts, expectation values, plots, error mitigation | `result()`, `quantum_info`, `visualization` |

Three packages, one workflow:
- **`qiskit`** (the SDK) — build circuits/operators, transpile, and *local*
  reference primitives. The foundation; everything is SDK-first.
- **`qiskit-ibm-runtime`** — access IBM Quantum backends and run the **V2
  primitives** (Sampler/Estimator) with error suppression/mitigation, in Session
  or Batch modes.
- **`qiskit-aer`** — high-performance local simulators (statevector, density
  matrix, stabilizer, noise models) and Aer primitives, for testing before
  hardware.

**The concept that trips everyone up — the ISA circuit.** A circuit you build is
*abstract* (any gate, all-to-all connectivity). IBM hardware only runs **ISA
circuits**: only the backend's **basis gates**, obeying its **coupling map**. You
**must transpile** for the target backend before `run()`, and you must
`apply_layout` observables to match. Skipping this is the #1 cause of hardware
errors. (Local simulators are more forgiving, but transpiling is still good
practice.)

---

## 2. Prerequisites & install (do this first)

- **Python** — a version supported by the current `qiskit` release (check the
  "Programming Language" classifiers on https://pypi.org/project/qiskit/). Use a
  real version (`python3.12`) in a **virtualenv**.
- **Install:**
  ```bash
  pip install -U qiskit                 # the SDK
  pip install -U "qiskit[visualization]"# + Matplotlib drawers / plots (recommended)
  pip install -U qiskit-ibm-runtime     # IBM Quantum hardware + V2 primitives
  pip install -U qiskit-aer             # local high-performance simulators
  ```
  Confirm: `pip show qiskit`. **Do not `pip install qiskit` to upgrade a 0.x
  install to 1.0+** — the packaging changed; create a fresh env (see the v1.0
  migration note in the docs).
- For **hardware**, you need an **IBM Quantum Platform account + API key** and
  (for paid plans) an **instance CRN**. Local simulation needs no account.

```python
from qiskit_ibm_runtime import QiskitRuntimeService

# One-time: save credentials locally (default channel is ibm_quantum_platform)
QiskitRuntimeService.save_account(token="<API_KEY>", set_as_default=True,
                                  instance="<IBM Cloud CRN>")  # instance optional on Open plan
# Thereafter, anywhere:
service = QiskitRuntimeService()          # reads saved account
backend = service.least_busy(operational=True, simulator=False)
```

Auth options, channels (`ibm_quantum_platform` — the default — vs the retired
`ibm_quantum`), instances/CRNs, multiple saved accounts, and env-var/`channel`
patterns are in **[references/setup-and-hardware.md](references/setup-and-hardware.md)**.

> 🔒 **Treat the API key as a secret.** Read it from an env var or the saved
> account file, never hardcode it in code, notebooks, or commits. Don't print
> tokens. Rotate if exposed.

---

## 3. The workflow, in code (SDK-first)

### 3.1 Map — build circuits & operators

```python
from qiskit import QuantumCircuit
from qiskit.quantum_info import SparsePauliOp

# A Bell state
qc = QuantumCircuit(2)
qc.h(0)               # Hadamard on qubit 0
qc.cx(0, 1)           # CNOT (control 0, target 1)
# For Sampler you measure; for Estimator you do NOT measure (you pass observables)
# qc.measure_all()

# Observables as Pauli operators (for Estimator)
observables = [SparsePauliOp(l) for l in ["IZ", "IX", "ZZ", "XX"]]
```

Gate methods live on `QuantumCircuit`: `h, x, y, z, s, t, rx, ry, rz, p, cx, cz,
ccx, swap, measure, measure_all, barrier`, parametric gates via
`circuit.Parameter`, plus control flow (`if_test`, `for_loop`, `while_loop`,
`switch`, `box`). Prebuilt structures (QFT, ansätze, arithmetic, oracles) are in
`qiskit.circuit.library`. Full circuit-construction surface, parameters, and the
library are in **[references/circuits-and-operators.md](references/circuits-and-operators.md)**.

### 3.2 Optimize — transpile to an ISA circuit

```python
from qiskit.transpiler import generate_preset_pass_manager

pm = generate_preset_pass_manager(optimization_level=1, backend=backend)
isa_circuit = pm.run(qc)
# Observables must be laid out onto the same physical qubits:
isa_observables = [ob.apply_layout(isa_circuit.layout) for ob in observables]
```

`optimization_level` 0–3 (0 = minimal, 3 = heaviest optimization). The pass
manager maps layout + routing, translates to basis gates, and optimizes.
Stages, custom passes, plugins, the simpler `transpile()` helper, and reading the
`coupling_map`/`basis_gates`/`target` are in
**[references/transpilation.md](references/transpilation.md)**.

### 3.3 Execute — run with a primitive (V2)

**Local** (no account — great for development; `qiskit` ships reference
primitives, `qiskit-aer` ships fast ones):
```python
from qiskit.primitives import StatevectorSampler, StatevectorEstimator

qc_m = qc.copy(); qc_m.measure_all()
counts = StatevectorSampler().run([qc_m], shots=1024).result()[0].data.meas.get_counts()
ev = StatevectorEstimator().run([(qc, observables)]).result()[0].data.evs
```

**Hardware / IBM Quantum** (V2 primitives, PUB inputs):
```python
from qiskit_ibm_runtime import EstimatorV2 as Estimator

estimator = Estimator(mode=backend)
estimator.options.resilience_level = 1      # error mitigation
estimator.options.default_shots = 5000
# A PUB (Primitive Unified Bloc): (circuit, observables[, parameter_values])
job = estimator.run([(isa_circuit, isa_observables)])
print("Job ID:", job.job_id())
result = job.result()
evs = result[0].data.evs                     # expectation values
```
`SamplerV2` is analogous; its PUB is `(circuit[, parameter_values, shots])` and it
returns per-register bitstring data (`result[0].data.<creg>.get_counts()`). The V2
primitive contract (PUBs, options, resilience/shots, results shape), Sessions vs
Batch, and Aer primitives are in **[references/primitives-and-execution.md](references/primitives-and-execution.md)**.

### 3.4 Post-process — extract & visualize

```python
from qiskit.visualization import plot_histogram
qc.draw("mpl")                 # circuit diagram (text: print(qc.draw()))
plot_histogram(counts)         # results bar chart
```

`quantum_info` (`Statevector`, `Operator`, `DensityMatrix`, `entropy`, fidelity)
for analysis; result objects carry `.data` (per-PUB) and `.metadata`. Visualization
and state analysis are in **[references/visualization-and-postprocess.md](references/visualization-and-postprocess.md)**.

---

## 4. Choosing where to run — simulator vs hardware

- **Develop on a local simulator first.** Use `qiskit.primitives` (exact,
  statevector reference) or `qiskit_aer` (fast, scalable, supports shots & noise).
  No account, instant, free. Validate correctness here before spending hardware.
- **Use a `fake_provider` backend** (`qiskit_ibm_runtime.fake_provider`, e.g.
  `FakeManilaV2`) to rehearse transpilation and *noisy* simulation against a
  realistic coupling map/basis-gate set without a queue.
- **Go to hardware** (`QiskitRuntimeService`) when you need real quantum results.
  Pick with `service.least_busy(...)` or `service.backend("ibm_...")`. Hardware
  has **queues, shot/time limits, and cost** on paid plans — mind them.
- **Qubit count & depth matter.** Statevector simulation is exact but ~doubles
  memory per qubit (≈30 qubits is a practical laptop ceiling); hardware has many
  qubits but noise. Match the tool to the circuit.

---

## 5. Critical constraints (these cause silent failures — internalize them)

- ✅ **Transpile to the backend's ISA before hardware `run()`.** Abstract circuits
  with non-basis gates or illegal connectivity fail or are rejected. Run
  `generate_preset_pass_manager(backend=...).run(circuit)`.
- ✅ **`apply_layout` observables after transpiling.** An Estimator observable must
  match the transpiled circuit's physical-qubit layout, or results are wrong.
- ✅ **Use V2 primitives only.** `SamplerV2`/`EstimatorV2` take **PUBs** (tuples)
  and return a result *per PUB*. V1 primitives are removed/deprecated; old
  `Sampler().run(circuits, observables)` signatures no longer apply.
- ✅ **Estimator circuits must NOT have measurements; Sampler circuits MUST.**
  Estimator measures observables you pass; Sampler reads classical registers.
- ✅ **Channel: use the default `ibm_quantum_platform`.** The legacy `ibm_quantum`
  channel (quantum.ibm.com classic) was sunset 1 Jul 2025.
- ✅ **Bit ordering is little-endian.** Qiskit prints qubit *n-1 … 0*; bitstring
  `"q_{n-1}...q_1 q_0"`. Mis-reading this flips your results. (See bit-ordering.)
- ✅ **Set a seed for reproducibility** where supported (transpiler `seed_transpiler`,
  Aer `seed_simulator`) — sampling and some transpiler passes are stochastic.
- ✅ **Respect shot/time/cost limits.** Hardware jobs queue and cost on paid plans;
  large shot counts and deep circuits cost more. Batch/Session help efficiency,
  not free capacity.
- ✅ **Signatures aren't stable across major versions.** v1.0 repackaged, v2.0
  broke APIs — verify against the installed version, don't trust 0.x memory.

---

## 6. Beyond the basics (where to go next)

- **Error suppression & mitigation** — `resilience_level`, dynamical decoupling,
  measurement/gate twirling, ZNE/PEC. Configured via primitive `options`. See
  primitives reference.
- **Execution modes** — **Session** (iterative/variational, low queue latency) and
  **Batch** (parallel independent jobs). See primitives reference.
- **OpenQASM interop** — import/export circuits with `qiskit.qasm2` / `qiskit.qasm3`.
- **Addons & ecosystem** — `qiskit-addon-*` (SQD, MPF, OBP, cutting, AQC-tensor),
  Qiskit Serverless, the AI transpiler (`qiskit-ibm-transpiler`), and Qiskit
  Functions. These build on this core; pull their docs when needed.
- **Quantum information & algorithms** — `quantum_info` for states/operators;
  variational patterns (VQE-style) compose circuits + Estimator + a classical
  optimizer.

---

## 7. REST API (when you're not in Python)

The SDK is primary, but Qiskit Runtime also exposes a REST surface — the **IBM
Quantum System API** — for other languages, thin services, or curl. Same service,
same four-step pattern (you still transpile to ISA first).

- **Base URL:** `https://quantum.cloud.ibm.com/api`
- **Auth: IAM Bearer token + instance CRN.** Exchange your API key for a
  short-lived IAM token, then send `Authorization: Bearer <token>` and
  `Service-CRN: <instance CRN>`. Date-based versioning via the `IBM-API-Version`
  header (e.g. `2025-08-15`).

```bash
TOKEN=$(curl -s -X POST 'https://iam.cloud.ibm.com/identity/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d "grant_type=urn:ibm:params:oauth:grant-type:apikey&apikey=$IQP_API_TOKEN" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

curl -s "https://quantum.cloud.ibm.com/api/v1/backends" \
  -H "Authorization: Bearer $TOKEN" -H "Service-CRN: $INSTANCE_CRN"
```

Endpoints: `/v1/backends` (+ `/configuration`, `/properties`, `/lanes`),
`/v1/jobs` (`POST` to **run a primitive** — `program_id` `"sampler"`/`"estimator"`;
`GET` to list), `/v1/jobs/{id}` (`GET`/`DELETE`), `/v1/jobs/{id}/cancel`,
`/version`, `/versions`. The hard part over REST is serializing ISA circuits into
the job `params` — **if you can run Python at all, submit with the SDK** and use
REST only to poll/manage. Full endpoint map, the `RunJobBody` schema, and the
401-bearer-vs-key gotcha are in **[references/runtime-rest-api.md](references/runtime-rest-api.md)**.

---

## 8. Debugging playbook

| Symptom | Likely cause → fix |
|---|---|
| Hardware job rejected / `not an ISA circuit` | Not transpiled for the backend. `generate_preset_pass_manager(backend=...).run(qc)` first. |
| Estimator results look wrong / mismatched qubits | Observables not laid out. `ob.apply_layout(isa_circuit.layout)`. |
| `Sampler`/`Estimator` `run()` signature error | Using V1 calling style. Use V2 PUBs: `run([(circuit, observables)])` / `run([(circuit,)])`. |
| Sampler returns nothing / empty counts | Circuit has no measurements. Add `measure_all()` (Sampler) — but remove them for Estimator. |
| `ModuleNotFoundError: qiskit.providers.aer` or V1 imports | Old API. Aer is `qiskit_aer`; primitives are V2. Reinstall current packages. |
| Counts/bitstrings reversed from expectation | Little-endian ordering — Qiskit lists qubit n-1…0. Re-read or reverse. |
| `IBMNotAuthorizedError` / 401 | Bad/expired token or wrong channel/instance. Re-`save_account` on `ibm_quantum_platform`; check the CRN. |
| Old code uses `channel="ibm_quantum"` | Retired (1 Jul 2025). Drop it (default) or set `ibm_quantum_platform`. |
| REST: persistent `401 Unauthorized` | Sent the raw API key where a **Bearer token** is required, or token expired. Re-exchange the IAM token; include `Service-CRN`. |
| Wrong/ambiguous instance chosen | Pass `instance=<CRN>`, or guide auto-selection with `plans_preference`/`region`/`tags`; check `service.active_instance()`. |
| Statevector sim runs out of memory | Too many qubits (~>30). Use shots-based `AerSimulator`, a smaller circuit, or hardware. |
| `pip install -U qiskit` broke a 0.x project | v1.0 repackaging — can't upgrade in place from 0.x. Fresh venv + migration guide. |
| Non-deterministic results between runs | Stochastic sampling/transpilation. Set `seed_simulator` / `seed_transpiler`. |

More failure modes per area are in the referenced files.

---

## 9. Verify before handover

**Code that imports ≠ code that works.** Before reporting a task as done, run the
circuit and show the output — a local `StatevectorSampler`/`AerSimulator` run with
real counts, or an `Estimator` expectation value — and confirm it matches the
expected physics (e.g. a Bell state gives ~50/50 `00`/`11` and ⟨ZZ⟩ ≈ 1). For
**hardware** runs: confirm the circuit is **ISA-transpiled** for the chosen
backend, state the **queue/shot/cost implication** before submitting, and report
the **job ID**. Report honestly: "local Aer, 1024 shots → {'00':535,'11':489}", or
"submitted job `<id>` to `ibm_xxx`; ⟨ZZ⟩ = 0.97 ± 0.02".

---

## 10. Working alongside other skills

Qiskit is the **quantum compute layer**. If a TechZone environment is being
provisioned to *get* an IBM Quantum instance, that's the TechZone skill's job —
this skill picks up once you have an API key. Qiskit results (counts, expectation
values) are classical data you can then feed into any downstream analysis,
notebook, or report.

---

## 11. References (load on demand)

| File | Contents |
|------|----------|
| [references/setup-and-hardware.md](references/setup-and-hardware.md) | Install (SDK, visualization, runtime, aer), Python support, the v0.x→v1.0 repackaging caveat; `QiskitRuntimeService`, `save_account`, channels (`ibm_quantum_platform` vs retired `ibm_quantum`), instances/CRNs, multiple-account loading rules, **automatic instance selection** (`plans_preference`/`region`/`tags`, `active_instance`), **local testing mode** (`channel="local"`), choosing a backend, `fake_provider` |
| [references/circuits-and-operators.md](references/circuits-and-operators.md) | `QuantumCircuit` construction, the gate set, registers/measurement, `Parameter`/`ParameterVector`, control flow, `circuit.library` (QFT, ansätze, arithmetic, oracles), `quantum_info` operators (`SparsePauliOp`, `Pauli`, `Operator`) |
| [references/transpilation.md](references/transpilation.md) | ISA circuits, `generate_preset_pass_manager`, optimization levels, stages (layout/routing/translation/optimization), `transpile()` helper, custom passes & plugins, reading `target`/`coupling_map`/`basis_gates`, `apply_layout` |
| [references/primitives-and-execution.md](references/primitives-and-execution.md) | V2 primitives (`SamplerV2`/`EstimatorV2`), PUBs, result shapes (`.data`, `.evs`, per-register counts), options (shots, `resilience_level`, DD, twirling, ZNE), Sessions vs Batch, local `qiskit.primitives` and `qiskit_aer` primitives/noise models |
| [references/visualization-and-postprocess.md](references/visualization-and-postprocess.md) | `QuantumCircuit.draw`, `plot_histogram`, `plot_bloch_multivector`, `plot_distribution`; `quantum_info` state analysis (`Statevector`, `DensityMatrix`, fidelity, entropy); reading result metadata; bit-ordering |
| [references/runtime-rest-api.md](references/runtime-rest-api.md) | The **Qiskit Runtime REST API** (IBM Quantum System API): base URL, IAM-bearer + `Service-CRN` auth, `IBM-API-Version` date versioning, the backends/jobs endpoint map, the `RunJobBody` (`program_id` sampler/estimator) schema, when to use REST vs SDK |

### Canonical external resources (you have internet access — use them)
- **API reference (ground truth for signatures):** https://quantum.cloud.ibm.com/docs/api/qiskit (the local `REFERENCE-Qiskit-documentation-main/docs/api/qiskit` snapshot mirrors this — versioned)
- **Runtime REST API reference:** https://quantum.cloud.ibm.com/docs/api/runtime (the local `REFERENCE-Quantum/latest` is the IBM Quantum System API OpenAPI spec — backends + jobs subset)
- **Guides & tutorials:** https://quantum.cloud.ibm.com/docs (the local `docs/guides` + `docs/tutorials`)
- **PyPI / versions:** `pip show qiskit qiskit-ibm-runtime qiskit-aer`. Verified when written: `qiskit` 2.4.1, `qiskit-ibm-runtime` 0.47.0, `qiskit-aer` 0.17.2.
- **GitHub:** https://github.com/Qiskit/qiskit · https://github.com/Qiskit/qiskit-ibm-runtime · https://github.com/Qiskit/qiskit-aer

**Always prefer the live API docs / `help()` over memory when a signature is in
doubt, and always use the V2 primitives and the default `ibm_quantum_platform`
channel.** This skill was verified against `qiskit` 2.4.1.
