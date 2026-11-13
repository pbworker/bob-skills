# Execute — V2 primitives, PUBs, options, execution modes

> Verified against `qiskit` 2.4.1, `qiskit-ibm-runtime` 0.47.0, `qiskit-aer`
> 0.17.2. Step 3 of the Qiskit pattern: run ISA circuits and get data.

## Two primitives, one contract

- **Sampler** → samples the circuit, returns **per-shot bitstrings / counts** from
  classical registers. Circuit **must have measurements**.
- **Estimator** → returns **expectation values** ⟨O⟩ of observables you pass.
  Circuit **must NOT have measurements**.

Both are **V2** (`SamplerV2`/`EstimatorV2`). **Always use V2** — V1 is gone. V2
takes **PUBs** (Primitive Unified Blocs, tuples) and returns one result *per PUB*.

### PUB shapes
- **Sampler PUB:** `(circuit,)` or `(circuit, parameter_values)` or
  `(circuit, parameter_values, shots)`
- **Estimator PUB:** `(circuit, observables)` or
  `(circuit, observables, parameter_values)` or `(..., precision)`

`run([...])` takes a **list of PUBs**; `result()` is indexable per PUB.

## Local execution (no account)

```python
# Exact reference primitives (qiskit) — no shot noise, great for correctness
from qiskit.primitives import StatevectorSampler, StatevectorEstimator
# Fast, scalable, shots + noise (qiskit-aer)
from qiskit_aer.primitives import SamplerV2, EstimatorV2

# Sampler
qc_m = qc.copy(); qc_m.measure_all()
res = SamplerV2().run([(qc_m,)], shots=2000).result()
counts = res[0].data.meas.get_counts()          # {'00': 982, '11': 1018}

# Estimator
from qiskit.quantum_info import SparsePauliOp
res = EstimatorV2().run([(qc, [SparsePauliOp("ZZ"), SparsePauliOp("XX")])]).result()
evs  = res[0].data.evs                            # array of expectation values
stds = res[0].data.stds                           # standard errors
```

### Reading results
- **Sampler:** `pub_result.data.<register_name>.get_counts()`. The register is
  `meas` when you used `measure_all()`; otherwise it's your `ClassicalRegister`
  name. Also `.get_bitstrings()`, `.array`.
- **Estimator:** `pub_result.data.evs` and `.stds` (NumPy arrays, one per
  observable). `pub_result.metadata` has `target_precision`, etc.
- Result is per-PUB: `result[i]` for the i-th PUB you submitted.

## Hardware execution (IBM Quantum)

```python
from qiskit_ibm_runtime import QiskitRuntimeService, EstimatorV2 as Estimator

service = QiskitRuntimeService()
backend = service.least_busy(operational=True, simulator=False)

# (transpile first — see transpilation.md)
estimator = Estimator(mode=backend)
estimator.options.default_shots = 5000
estimator.options.resilience_level = 1            # error mitigation level

job = estimator.run([(isa_circuit, isa_observables)])
print("Job ID:", job.job_id())                    # save it to retrieve later
result = job.result()
evs = result[0].data.evs
```
`SamplerV2(mode=backend)` is analogous; PUB `(isa_circuit,)`, read
`result[0].data.<creg>.get_counts()`.

### Options (set on `primitive.options`)
Common:
- `default_shots` (Sampler/Estimator), or `default_precision` (Estimator)
- `resilience_level` (Estimator: 0 none → higher = more mitigation)
- **Error suppression:** `dynamical_decoupling.enable`, `twirling.enable_gates`,
  `twirling.enable_measure`
- **Error mitigation (Estimator):** `resilience.zne_mitigation` (ZNE),
  `resilience.pec_mitigation` (PEC), `resilience.measure_mitigation`
- `environment`, `max_execution_time`

Inspect with `print(estimator.options)`; exact nested fields are in
`EstimatorOptions`/`SamplerOptions` — check `help()` for your version.

## Execution modes — Session vs Batch

```python
from qiskit_ibm_runtime import Session, Batch, SamplerV2 as Sampler

# Session: iterative/variational work — keeps a slot, minimizes inter-job queueing
with Session(backend=backend) as session:
    sampler = Sampler(mode=session)
    for params in sweep:
        job = sampler.run([(isa_circuit, params)])
        # use job.result() to decide the next params (e.g. VQE loop)

# Batch: many independent jobs run with parallel classical processing
with Batch(backend=backend) as batch:
    estimator = Estimator(mode=batch)
    jobs = [estimator.run([pub]) for pub in pubs]
```
- **Session** — best for **dependent** iterative tasks (variational algorithms);
  low latency between jobs.
- **Batch** — best for **independent** jobs you want processed efficiently in
  parallel.
- **Single job** (`mode=backend`) — fine for one-off runs.

## Aer simulator directly (alternative to primitives)

```python
from qiskit_aer import AerSimulator
sim = AerSimulator(method="statevector", seed_simulator=42)   # or density_matrix, stabilizer, matrix_product_state
isa = pm.run(qc_m)                                            # transpile for sim
counts = sim.run(isa, shots=1024).result().get_counts()
```
Build noise models from a real/fake backend with
`AerSimulator.from_backend(fake_backend)` to simulate device noise.

## Constraints recap
- **V2 only**, PUBs in, per-PUB results out.
- **Sampler needs measurements; Estimator must not have them.**
- **Transpile + `apply_layout`** before hardware.
- **Set seeds** (`seed_simulator`) for reproducible sampling.
- Hardware **queues/costs**; tune shots and use Session/Batch to be efficient, not
  to get free capacity. Save the **job ID** to retrieve results asynchronously
  (`service.job("<id>")`).
