# Post-process — visualization & state analysis

> Verified against `qiskit` 2.4.1. Step 4 of the Qiskit pattern: turn raw results
> into the answer. Most plotting needs `pip install "qiskit[visualization]"`
> (Matplotlib).

## Drawing circuits

```python
qc.draw("mpl")                 # Matplotlib figure (Jupyter shows it inline)
print(qc.draw())               # text drawing (for scripts/terminals)
qc.draw("mpl", idle_wires=False, fold=-1)   # hide unused wires; don't fold
qc.draw("latex")               # LaTeX (needs a LaTeX toolchain)
```
For a transpiled circuit, `isa_circuit.draw("mpl", idle_wires=False)` shows the
physical-qubit mapping.

## Plotting results

```python
from qiskit.visualization import (
    plot_histogram, plot_distribution,
    plot_bloch_multivector, plot_state_city, plot_state_qsphere,
)

plot_histogram(counts)                       # bar chart of Sampler counts
plot_histogram([counts_a, counts_b], legend=["A", "B"])   # compare runs
plot_distribution(counts)                    # probability distribution
```
For Estimator output (expectation values), plot with Matplotlib directly — e.g.
`plt.bar(observable_labels, result[0].data.evs)`.

## Bit ordering (read this before trusting any result)

Qiskit is **little-endian**: in a counts key, the **leftmost** character is the
**highest-index** qubit and the **rightmost** is qubit `0`. So `"10"` means
`q1=1, q0=0`. If your results look mirrored vs a textbook, this is why — reverse
the string (`key[::-1]`) or account for it in indexing.

## State & operator analysis — `quantum_info`

For exact analysis of small circuits (simulation, not hardware):

```python
from qiskit.quantum_info import (
    Statevector, DensityMatrix, Operator,
    state_fidelity, process_fidelity, entropy, partial_trace,
)

sv = Statevector(qc)                       # exact statevector of a circuit
probs = sv.probabilities_dict()            # {'00': 0.5, '11': 0.5}
dm = DensityMatrix(qc)
U  = Operator(qc)                          # dense unitary

state_fidelity(sv, Statevector.from_label("00"))   # compare states
entropy(partial_trace(dm, [1]))                    # entanglement entropy of a subsystem
```

Common uses:
- **Verify correctness** of a circuit in development (`Statevector(qc)` vs expected).
- **Fidelity** between an ideal state and a simulated/noisy one.
- **Reduced states** via `partial_trace` for subsystem analysis.
- **Expectation values offline:** `Statevector(qc).expectation_value(SparsePauliOp("ZZ"))`.

## Reading result objects (recap)
- **Sampler:** `result[i].data.<creg>.get_counts()` / `.get_bitstrings()`.
- **Estimator:** `result[i].data.evs` (expectation values) and `.stds` (errors).
- **Metadata:** `result[i].metadata` (e.g. `target_precision`), and job-level
  `result.metadata`.

## Post-processing techniques (beyond plotting)
- **Readout/measurement error mitigation** — prefer the primitive's built-in
  `resilience.measure_mitigation`; classical post-correction is also possible.
- **Marginalization** — `qiskit.result.marginal_counts(counts, indices)` to get
  counts on a subset of qubits.
- **Post-selection** — filter shots by a physical invariant (parity, particle
  number, total spin) before computing the final quantity.

## Constraints recap
- Plots need the `visualization` extra; in a non-notebook context call
  `fig.savefig(...)` or `plt.show()`.
- `quantum_info` state analysis is **exact simulation** — only feasible for small
  qubit counts (memory ~doubles per qubit). For large/hardware results, work from
  the primitive's counts/evs.
- Mind **little-endian** bit ordering when mapping bitstrings back to qubits.
