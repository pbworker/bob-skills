# Setup, accounts & hardware — install, `QiskitRuntimeService`, backends

> Verified against `qiskit` 2.4.1, `qiskit-ibm-runtime` 0.47.0, `qiskit-aer`
> 0.17.2. When in doubt, `pip show <pkg>` and check the live docs.

## Install

```bash
pip install -U qiskit                  # the SDK (circuits, transpiler, local primitives)
pip install -U "qiskit[visualization]" # + Matplotlib drawers and plots (recommended)
pip install -U qiskit-ibm-runtime      # IBM Quantum hardware access + V2 primitives
pip install -U qiskit-aer              # local high-performance simulators
```
- Use a supported **Python** version (see classifiers on
  https://pypi.org/project/qiskit/) inside a **virtualenv**.
- **v0.x → v1.0 caveat:** Qiskit 1.0 changed the packaging structure. You **cannot**
  `pip install -U qiskit` to go from a 0.x install to 1.0+. Create a fresh
  environment and follow the 1.0 migration guide. (Upgrades from 1.0 onward are
  in-place.) Qiskit 2.0 then made API breaks — verify against your installed
  version.
- Optional metapackage extras and the C API (`qiskit-c`) exist; most users only
  need the four packages above.

## Authenticate to IBM Quantum

You need an **IBM Quantum Platform** account + **API key**; paid plans also use an
**instance CRN**. Local simulation needs none of this.

```python
from qiskit_ibm_runtime import QiskitRuntimeService

# One-time: persist credentials to ~/.qiskit (default channel = ibm_quantum_platform)
QiskitRuntimeService.save_account(
    token="<API_KEY>",
    instance="<IBM Cloud CRN or instance name>",   # optional on the Open plan
    set_as_default=True,                            # optional
    # channel="ibm_quantum_platform",               # the default; rarely set explicitly
)

# Later, anywhere:
service = QiskitRuntimeService()                    # reads the saved account
```
`QiskitRuntimeService(...)` parameters include: `channel`, `token`, `url`,
`filename`, `name`, `instance`, `proxies`, `verify`, `private_endpoint`, `region`,
`plans_preference`, `tags`. You can also construct it directly with
`QiskitRuntimeService(channel="ibm_quantum_platform", token=..., instance=...)`
instead of saving.

### Channels — important
- **`ibm_quantum_platform`** is the **default** and almost always correct (the
  new IBM Quantum Platform on IBM Cloud). Because it's the default, examples omit it.
- **`ibm_quantum`** (the classic quantum.ibm.com platform) was **sunset 1 July
  2025**. Don't use it; migrate any old `channel="ibm_quantum"` code.

### Multiple accounts
```python
QiskitRuntimeService.save_account(token="<A>", instance="<CRN_premium>", name="premium")
QiskitRuntimeService.save_account(token="<B>", instance="<CRN_open>", name="open",
                                  set_as_default=True)
service = QiskitRuntimeService(name="premium")      # pick by name
```
Loading rules (these surprise people):
- `QiskitRuntimeService(token="t", name="A")` — the **`name` wins**; `token` is
  ignored and account `A`'s saved details load.
- `QiskitRuntimeService(instance="CRN_B", name="A")` — loads account `A` but
  connects to `CRN_B` (a warning is raised on conflict).
- `QiskitRuntimeService()` with no token/name — loads the **default** saved
  account; if none is marked default, the account whose **name sorts last
  alphabetically** is used.

### Automatic instance selection
If you pass a `token` (or use saved creds) but **no `instance`**, the service
picks the most relevant instance for the requested QPU, guided by these options:
- **`plans_preference`** — priority list of plan types: `open`, `premium`,
  `pay-as-you-go`, `flex`, `on-prem`. If unset, **free plans are prioritized**
  (qiskit-ibm-runtime v0.42+).
- **`region`** — `us-east` or `eu-de`.
- **`tags`** — list of instance tag strings.

```python
service = QiskitRuntimeService(token="<KEY>", region="eu-de")
service = QiskitRuntimeService(token="<KEY>", plans_preference=["premium", "open"])
service.active_instance()        # which instance was actually selected
```
The chosen instance is also surfaced as a warning. Note: if only one instance can
reach the requested QPU and it's a **paid** plan, a **cost is incurred**.

### Local testing mode (no account, no auth)
```python
service = QiskitRuntimeService(channel="local")   # local simulation; no token/instance
```
Also, the runtime **primitives/sessions run locally automatically** when you give
them a `fake_provider` or Aer backend — see
[primitives-and-execution.md](primitives-and-execution.md). Don't save a "local"
account; just instantiate directly.

> 🔒 The API key is a secret. Prefer the saved-account file or an env var
> (`IQP_API_TOKEN`); never hardcode/commit it; don't print it; rotate if exposed.
> A persistent **401 Unauthorized** usually means you used a **bearer token where
> an API key is expected** (or vice-versa over REST).

## Choosing a backend

```python
backend = service.least_busy(operational=True, simulator=False)  # auto-pick
backend = service.backend("ibm_brisbane")                        # by name
backends = service.backends(operational=True, min_num_qubits=5)  # filter/list
print(backend.name, backend.num_qubits)
print(backend.target.operation_names)        # basis gates
print(backend.coupling_map)                  # connectivity
```
A `BackendV2`'s `target` carries everything the transpiler needs (basis gates,
coupling map, instruction properties/errors, timing).

## Local & fake backends (no account)
- **`qiskit_aer.AerSimulator`** — fast local simulator; supports shots, methods
  (`statevector`, `density_matrix`, `stabilizer`, `matrix_product_state`), and
  noise models. Use for development at scale.
- **`qiskit.primitives`** reference primitives (`StatevectorSampler`,
  `StatevectorEstimator`) — exact, no shots noise, ideal for correctness checks.
- **`qiskit_ibm_runtime.fake_provider`** (e.g. `FakeManilaV2`, `FakeBrisbane`) —
  snapshots of real devices (coupling map, basis gates, noise) to rehearse
  transpilation and noisy simulation without a queue.

```python
from qiskit_ibm_runtime.fake_provider import FakeManilaV2
fake = FakeManilaV2()                 # use as a `backend=` for transpiling & sim
```

## Constraints recap
- Local first, hardware when you need real results. Hardware **queues, has shot/
  time limits, and costs** on paid plans.
- Always transpile to the chosen backend's ISA before a hardware `run()` (see
  [transpilation.md](transpilation.md)).
