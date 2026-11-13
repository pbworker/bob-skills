# Map — circuits, gates, parameters, library & operators

> Verified against `qiskit` 2.4.1. Step 1 of the Qiskit pattern: turn a problem
> into **circuits** and **operators**.

## `QuantumCircuit`

```python
from qiskit import QuantumCircuit, QuantumRegister, ClassicalRegister

qc = QuantumCircuit(3)                       # 3 qubits, no classical bits
qc = QuantumCircuit(3, 3)                     # 3 qubits + 3 classical bits
qr = QuantumRegister(2, "q"); cr = ClassicalRegister(2, "c")
qc = QuantumCircuit(qr, cr)                   # named registers
```

### Gate set (methods on the circuit)
Verified present on `QuantumCircuit`:

- **1-qubit:** `h, x, y, z, s, sdg, t, tdg, rx, ry, rz, p, u`
- **2-qubit:** `cx, cy, cz, ch, cp, crx, cry, crz, swap, rxx, ryy, rzz`
- **3-qubit / multi:** `ccx` (Toffoli), `cswap` (Fredkin), `mcx` (multi-controlled X)
- **non-unitary / structural:** `measure`, `measure_all`, `reset`, `initialize`,
  `barrier`, `delay`
- **control flow:** `if_test`, `for_loop`, `while_loop`, `switch`, `box`

```python
qc.h(0); qc.cx(0, 1)            # Bell
qc.rx(1.57, 2)                  # parametric rotation (radians)
qc.measure_all()               # add a creg and measure every qubit
qc.barrier()                   # visual/transpiler barrier
```

### Inspect / compose
`qc.num_qubits`, `qc.depth()`, `qc.size()`, `qc.count_ops()`, `qc.draw("mpl")`,
`qc.copy()`, `qc.compose(other)`, `qc.append(gate, qargs)`, `qc.to_gate()`,
`qc.inverse()`, `qc.decompose()`.

> **Measurement rule:** **Sampler** circuits MUST contain measurements; **Estimator**
> circuits MUST NOT (you pass observables instead). **Bit ordering is little-endian**
> — Qiskit lists qubit *n-1 … 0*, so a counts key `"q2 q1 q0"`.

## Parameters (for variational / reusable circuits)

```python
from qiskit.circuit import Parameter, ParameterVector

theta = Parameter("θ")
qc = QuantumCircuit(1); qc.ry(theta, 0)
bound = qc.assign_parameters({theta: 0.5})        # bind one value
# Or sweep many values directly in a primitive PUB (preferred — see primitives ref)

phis = ParameterVector("φ", 4)                     # a vector of parameters
```
Primitives accept arrays of parameter values in the PUB, so you usually leave
parameters unbound and pass values at `run()` time.

## Circuit library — don't reinvent structures

`qiskit.circuit.library` ships prebuilt circuits/gates (verified present: `QFT`,
`efficient_su2`, `n_local`, `real_amplitudes`, and many more):

- **Standard gates:** `XGate`, `CXGate`, `RZGate`, … (objects for `append`)
- **N-local ansätze (variational):** `efficient_su2(...)`, `real_amplitudes(...)`,
  `n_local(...)`, `EvolvedOperatorAnsatz`, `ExcitationPreserving`
- **Fourier / arithmetic:** `QFT`, adders (`DraperQFTAdder`, `CDKMRippleCarryAdder`,
  `FullAdderGate`), multipliers, comparators
- **Data encoding / oracles:** `ZZFeatureMap`, `PauliFeatureMap`, Grover oracles
- **State prep:** `Diagonal`, `GraphState`, `PauliEvolutionGate`

```python
from qiskit.circuit.library import efficient_su2, QFT
ansatz = efficient_su2(num_qubits=4, reps=2)       # parameterized, ready to bind
qft = QFT(3)
```
(In Qiskit 2.x many of these are **functions** returning a `QuantumCircuit`, e.g.
`efficient_su2(...)`; some remain classes. Check `help()` if unsure.)

## Operators — `quantum_info`

Observables and operators for the **Estimator** and for analysis.

```python
from qiskit.quantum_info import SparsePauliOp, Pauli, Operator

# Observable as a weighted sum of Pauli strings (little-endian, left = highest qubit)
H = SparsePauliOp.from_list([("ZZ", 1.0), ("XX", 0.5), ("IZ", -0.3)])
H2 = SparsePauliOp(["IZ", "ZI"], coeffs=[1, 1])
print(H.paulis, H.coeffs, H.num_qubits)

P = Pauli("XYZ")                                   # a single Pauli
U = Operator(qc)                                   # dense unitary of a circuit
```
`SparsePauliOp` is the standard observable type; after transpiling you must
`H.apply_layout(isa_circuit.layout)` so it matches the physical qubits (see
[transpilation.md](transpilation.md)). More state/operator analysis (`Statevector`,
`DensityMatrix`, fidelity, entropy) is in
[visualization-and-postprocess.md](visualization-and-postprocess.md).

## OpenQASM interop
```python
from qiskit import qasm2, qasm3
qasm_str = qasm3.dumps(qc)                  # export
qc2 = qasm3.loads(qasm_str)                 # import (qasm2 analogous)
```
