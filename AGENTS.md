# AGENTS.md — Quantum Circuit Simulation Library

## 1. Project Overview

Build a **complete, language-agnostic quantum circuit simulation library**. The
library performs **real quantum simulation** using state-vector evolution over
complex vector spaces — no stubs, no approximations, no external quantum
backends.

### First Step: Ask for the Target Language

Before writing any code, **ask the user which programming language to use**. The
architecture described here is language-agnostic. Adapt idioms, project
structure, file extensions, test frameworks, and build tools to the chosen
language.

### What This Library Does

- Builds quantum circuits declaratively via a builder/chaining API.
- Simulates them shot-by-shot using state-vector math (Born rule measurement).
- Returns measurement outcome percentages (bitstring histograms).
- Supports ordered, named classical registers and preserves them through circuit
  construction, serialization, transpilation, simulation, and backend result
  parsing.
- Provides introspection (state vectors, Bloch sphere coordinates, circuit
  complexity metrics).
- Supports symbolic parameters with arithmetic expressions, bound at run time.
- Supports classical control flow: `ifTest`, `forLoop`, `whileLoop`, `switch`,
  `breakLoop`, `continueLoop`, and `box`.
- Supports circuit composition: `compose`, `toGate`, `toInstruction`, `append`,
  `inverse`.
- Exposes a Backend interface with a full `SimulatorBackend`, `IBMBackend`, and
  `QBraidBackend` implementation (including a complete transpilation pipeline).
- Serializes/deserializes circuits to/from **OpenQASM 3** via a public
  Serializer interface with **comprehensive OpenQASM 3 language coverage**
  including: classical typed variables (`bool`, `int[n]`, `uint[n]`, `float[n]`,
  `angle[n]`, `const`), `input`/`output` declarations, custom `gate`
  definitions, gate modifiers (`ctrl @`, `negctrl @`, `inv @`, `pow(k) @`),
  classical expressions (arithmetic, bitwise, comparison, logical operators),
  `array` types, subroutines (`def`), `extern` declarations,
  `pragma`/annotations, timing literals, built-in constants (`pi`, `tau`,
  `euler`), built-in math functions, casting, indexing/slicing, gate
  broadcasting, and physical qubit references (`$0`).
- Exposes the underlying Complex and Matrix algebra as public API.

### Non-Negotiable Constraint

**Every operation must be a real computation.** Complex number arithmetic,
matrix algebra, tensor products, gate application via subspace iteration,
Born-rule sampling, and state collapse must all be implemented from scratch. Do
not shell out to numpy, BLAS, or any linear-algebra library. The whole point is
that the simulation engine is self-contained.

---

## 2. Code Conventions

- Use the idiomatic style of the target language (e.g., camelCase for TS/JS,
  snake_case for Python/Rust).
- All public symbols must have doc comments explaining their purpose,
  parameters, and return values.
- Immutability by default: `Complex` and `Matrix` instances are immutable value
  objects. Methods return new instances.
- Pure functions everywhere except the simulator engine (which is stateful per
  shot).
- No external runtime dependencies for core functionality. Test frameworks and
  dev tools are fine.
- Use strict type checking (e.g., `strict: true` in TS, type hints in Python,
  etc.).
- Floating-point tolerance: use `1e-10` as the default epsilon for all
  approximate equality checks.
- All angle parameters are in **radians**.
- Qubit ordering is **little-endian** (least-significant-bit-first): in an
  n-qubit system, the state vector index `j` encodes qubit `k` as bit `k` of
  `j`. For example, in a 3-qubit system, index 5 = binary 101 means qubit 0 = 1,
  qubit 1 = 0, qubit 2 = 1.
- Gate matrix ordering is **MSB-first**: in an m-qubit gate matrix, bit `(m-1)`
  of the row/column index corresponds to the first qubit argument, bit `(m-2)`
  to the second, and so on. For example, in a 2-qubit gate `G(control, target)`,
  the 4x4 matrix index has bit 1 = control, bit 0 = target. In a 3-qubit gate
  `G(ctrl1, ctrl2, target)`, bit 2 = ctrl1, bit 1 = ctrl2, bit 0 = target.

### Phase Convention

Use a single, explicit phase convention throughout the entire library. Every
gate definition, decomposition, controlled-gate construction, transpilation
pass, serializer, and simulator update must preserve this convention. The rules
below are authoritative; if any gate matrix or decomposition elsewhere in this
document appears to conflict, these rules take precedence.

Within this section, plain multiplication/juxtaposition (for example,
`A * B * C`) denotes standard matrix-product order, so the rightmost factor acts
first. The arrow notation `A → B → C` denotes circuit-time order, so the total
unitary is `C * B * A`.

#### 1. Physical Equivalence vs. Exact Equality

Quantum states are defined up to global phase: two states `|ψ⟩` and `e^(i*φ)|ψ⟩`
are physically indistinguishable, and two operators `U` and `e^(i*φ)*U` produce
identical measurement statistics. However, this library tracks global phase
explicitly:

- **Exact comparisons** (`Complex.equals`, `Matrix.equals`, state-vector
  assertions) must match every matrix element and every amplitude including
  global phase factors. They must **not** silently discard or normalize away
  global phase.
- **Physical-equivalence checks** (used only when explicitly requested, e.g.,
  comparing two operators or state vectors up to global phase after
  transpilation) may ignore a uniform scalar `e^(i*φ)` multiplying the entire
  operator or state. Such checks must be called out by name (e.g.,
  `equalsUpToGlobalPhase`) and must never be the default comparison path.
  Matching measurement distributions alone is a weaker property and does not
  prove operator equality up to global phase.

#### 2. Rotation Gates — Negative-Exponent Physics Convention

All Pauli rotation gates use the physics convention with a **negative sign** in
the exponential:

```
R_P(θ) = exp(-i * θ * P / 2) = cos(θ/2) * I  -  i * sin(θ/2) * P
```

where `P ∈ {X, Y, Z}` is the Pauli generator. Concretely:

- `RX(θ) = cos(θ/2)*I - i*sin(θ/2)*X`
  `= [[cos(θ/2), -i*sin(θ/2)], [-i*sin(θ/2), cos(θ/2)]]`
- `RY(θ) = cos(θ/2)*I - i*sin(θ/2)*Y`
  `= [[cos(θ/2), -sin(θ/2)], [sin(θ/2), cos(θ/2)]]`
- `RZ(θ) = cos(θ/2)*I - i*sin(θ/2)*Z` `= [[exp(-i*θ/2), 0], [0, exp(i*θ/2)]]`

The general rotation gate `R(θ, φ)` follows the same convention for an axis in
the XY-plane:

```
R(θ, φ) = exp(-i * θ * (cos(φ)*X + sin(φ)*Y) / 2)
```

The arbitrary-axis rotation gate `RV(vx, vy, vz)` uses a **rotation vector**
`v = (vx, vy, vz)`, not a unit axis. The direction of `v` is the rotation axis
and the magnitude `|v| = sqrt(vx^2 + vy^2 + vz^2)` is the rotation angle:

```
RV(v) = exp(-i * (vx*X + vy*Y + vz*Z) / 2)
```

Equivalently, for a unit axis `n = (nx, ny, nz)` and an angle `θ`,
`RV(θ*nx, θ*ny, θ*nz) = exp(-i * θ * (nx*X + ny*Y + nz*Z) / 2)`.

**This convention must not be mixed with the positive-exponent convention**
(`exp(+iθP/2)`) anywhere in the library. Every function that accepts a rotation
angle `θ` interprets increasing `θ` as rotation in the same direction dictated
by the negative exponent.

#### 3. Phase Gate — Computational-Basis Convention

The phase gate is defined in the computational basis as:

```
P(λ) = diag(1, exp(i*λ))  =  [[1, 0], [0, exp(i*λ)]]
```

It applies a phase `exp(i*λ)` to the `|1⟩` state and leaves `|0⟩` unchanged. The
Clifford phase hierarchy descends from P:

- `Z = P(π) = diag(1, -1)`
- `S = P(π/2) = diag(1, i)`
- `T = P(π/4) = diag(1, exp(i*π/4))`
- `Sdg = P(-π/2) = S†`, `Tdg = P(-π/4) = T†`

#### 4. Relationship Between RZ and P

`RZ(θ)` and `P(θ)` differ by a global phase factor:

```
RZ(θ) = exp(-i*θ/2) * P(θ)
```

Equivalently, `P(θ) = exp(i*θ/2) * RZ(θ)`. Both produce identical measurement
statistics for any `θ`, but they are **not** matrix-equal. Code must use the
correct one: `RZ` when the negative-exponent rotation semantics are needed
(e.g., inside decompositions that rely on `RZ` angle addition), and `P` when the
computational-basis diagonal semantics are needed (e.g., controlled-phase gates,
the `S`/`T` hierarchy). Never silently substitute one for the other.

#### 5. General Single-Qubit Unitary — Canonical U Gate and ZYZ Decomposition

The named three-parameter `U(θ, φ, λ)` gate is the standard representative of an
arbitrary single-qubit unitary **up to global phase**:

```
U(θ, φ, λ) = exp(i*(φ+λ)/2) * RZ(φ) * RY(θ) * RZ(λ)
```

which produces the matrix:

```
[[cos(θ/2),              -exp(i*λ)*sin(θ/2)       ],
 [exp(i*φ)*sin(θ/2),     exp(i*(φ+λ))*cos(θ/2)   ]]
```

An exact arbitrary single-qubit unitary `M ∈ U(2)` requires one additional phase
parameter:

```
M = exp(i*α) * U(θ, φ, λ)
```

Equivalently,

```
M = exp(i*(α + (φ+λ)/2)) * RZ(φ) * RY(θ) * RZ(λ)
```

The factor `exp(i*(φ+λ)/2)` is the **global phase built into the named `U`
gate**. It reconciles the RZ convention (which introduces `exp(∓iθ/2)` factors)
with the U-gate convention. Under the canonical decomposition ranges defined
below, this also makes the `[0][0]` entry `cos(θ/2)` real and nonnegative. This
factor must be included whenever the named `U(θ, φ, λ)` gate is recomposed
exactly. More generally, exact ZYZ recomposition of an arbitrary single-qubit
unitary must also track the separate overall phase parameter `α`; omitting
either contribution causes exact matrix comparisons to fail.

For deterministic exact ZYZ decomposition of
`M = exp(i*α) * RZ(β) * RY(γ) * RZ(δ)`, normalize outputs to:

- `γ ∈ [0, π]`
- `β, δ ∈ [-π, π]`
- `α` is fixed by the principal half-argument of the determinant:
  `α = wrapToPi(arg(det(M)) / 2)`, where `arg` returns the principal phase in
  `(-π, π]`. Equivalently, choose the unique representative `α ∈ (-π/2, π/2]`
  compatible with `det(M) = exp(2iα)`.
- If `0 < γ < π`, compute `β` and `δ` with `wrapToPi`, so the generic branch
  returns `β, δ ∈ (-π, π]`.
- If `γ = 0` within epsilon, let `V = exp(-i*α) * M`, which is diagonal.
  Exactness depends on the individual diagonal-entry phase, not only on their
  ratio. Let `ζ = arg(V[1][1]) ∈ (-π, π]`, and set `β = ζ`, `δ = ζ`, so
  `RZ(β) * RZ(δ) = RZ(2ζ)` exactly. In particular, `V = -I` yields `β = δ = π`,
  not `β = δ = 0`.
- If `γ = π` within epsilon, let `V = exp(-i*α) * M`, which is anti-diagonal.
  Exactness depends on the individual off-diagonal-entry phase, not only on the
  phase ratio. Let `ζ = arg(V[1][0]) ∈ (-π, π]`, and set `β = ζ`, `δ = -ζ`. When
  `ζ = π`, the canonical pair is `(β, δ) = (π, -π)`; this is why the closed
  interval `[-π, π]` is required at the boundary.

These normalization rules apply to decomposition/transpilation outputs so they
are deterministic. User-facing gate constructors may accept any real angles and
must preserve the exact matrix implied by the supplied inputs rather than
silently renormalizing them.

Aliases: `U1(λ) = P(λ)`, `U3(θ,φ,λ) = U(θ,φ,λ)`, `U2(φ,λ) = U(π/2, φ, λ)`.

#### 6. Global Phase Tracking

The library tracks an exact hoistable phase expression `globalPhase` on each
phase-bearing scope: `QuantumCircuit`, custom gate definitions, subroutine
definitions, and nested control-flow bodies. Its effect is to multiply every
quantum amplitude exiting that scope by `exp(i * globalPhase)`. For a purely
unitary scope this is equivalent to multiplying the scope unitary by that scalar
after the body acts. For scopes that also contain measurement or classical
control flow, the same scalar still multiplies every branch amplitude uniformly
and therefore does not change measurement probabilities, collapse statistics, or
branch predicates. It is still required for `getStateVector` correctness and for
exact matrix/state comparisons on unitary subscopes. `globalPhase` is stored
exactly as an angle expression, not only as a floating-point number. However, it
stores only phase that is hoistable to the entry of the owning scope without
changing semantics. A phase is hoistable only if it is unmodified, unannotated,
and its expression depends only on symbols that are already in scope at that
scope's entry and remain immutable throughout that scope. If unresolved symbols
remain, any operation that needs a concrete complex scalar must bind them first.

The semantic scalar is `exp(i * globalPhase)`. Therefore purely numeric phase
expressions may be reduced modulo `2π` only when that reduction is exact.
Implementations are **not** required to prove that an arbitrary symbolic
expression equals `0` or `2πk`; if they cannot prove that the scalar is exactly
unity, they must preserve and serialize the stored expression. In this document,
"normalized" hoistable global phase means "at most one leading bare scope-level
`gphase(...)` per scope, emitted in the implementation's canonical `AngleExpr`
form"; it does **not** require aggressive symbolic rewriting or inexact
floating-point simplification.

The `globalPhaseGate(θ)` operation increments the owning scope's global phase by
`θ`, where `θ` may itself be symbolic. Its "matrix" is the 1×1 scalar
`[[exp(i*θ)]]`. In the programmatic builder API, a bare unmodified
`globalPhaseGate(θ)` denotes hoistable scope-global phase and is stored
separately from the ordinary instruction stream rather than as a qubit
instruction. When circuits/scopes are composed, their global phases add. When a
circuit/scope is inverted, its global phase is negated. `compose`, `toGate`,
`toInstruction`, gate-definition bodies, subroutine bodies, and QASM
round-tripping must preserve this scalar exactly.

A bare `gphase(θ);` parsed from OpenQASM 3 is folded into the owning scope's
`globalPhase` only when it is hoistable by the rule above and no source-order
information would be lost. Otherwise it remains an ordinary zero-qubit `gphase`
instruction in the statement stream, preserving expression structure,
annotations, and evaluation order. Modifier-bearing forms of
`gphase`/`globalPhaseGate` are first normalized by modifier expansion:
`inv @ gphase(θ)` becomes bare `gphase(-θ)` and `pow(k) @ gphase(θ)` becomes
bare `gphase(k*θ)`. If the normalized result is then bare, unannotated, and
hoistable, it may fold into the owning scope's scalar. Any remaining
control-bearing, annotation-bearing, or otherwise non-hoistable forms are not
scope-global: after control promotion they become relative-phase operators and
must remain ordinary instructions (or equivalent desugared phase operators).

#### 7. Global-to-Relative Phase Promotion in Controlled Gates

A gate's global phase is unobservable in isolation, but it becomes a **relative
phase** when the gate is applied only on an enabled control subspace. If
`U = exp(i*α) * V` (where `V` has `det(V) = 1` or some other normalized form),
then the controlled version of `U` is **not** the same as the controlled version
of `V`: when the control condition is satisfied, the target sees
`U = exp(i*α) * V`, and the `exp(i*α)` factor applies only to that enabled
subspace.

For the **single positive-control** case, the standard controlled-U construction
handles this by applying `P(α)` to the control qubit:

```
Controlled-U(c, t) = C(t) → CX(c,t) → B(t) → CX(c,t) → A(t) → P(α)(c)
```

where `U = exp(i*α) * A * X * B * X * C` and `A * B * C = I`.

Concrete examples of this promotion:

- `SX = exp(i*π/4) * RX(π/2)`, so `CSX = P(π/4)(c) → CRX(π/2, c, t)`. The
  `P(π/4)` on the control promotes the `exp(i*π/4)` global phase of SX into a
  relative phase.
- `SXdg = exp(-i*π/4) * RX(-π/2)`, so `CSXdg = P(-π/4)(c) → CRX(-π/2, c, t)`.
- The `CU(θ,φ,λ,γ)` gate includes an explicit `γ` parameter for an additional
  controlled global phase: `P(γ + (φ+λ)/2)` on the control.

For **multiple positive controls**, the compensating phase must act only on the
fully enabled control subspace. Implement it as the exact control-register phase
operator that multiplies only `|11...1⟩` of the active positive-control wires by
`exp(i*α)`; equivalently, use an exact multi-controlled phase synthesis on the
controls themselves and then apply the controlled version of `V`. Applying
`P(α)` to just one control wire is generally incorrect because it also phases
partially enabled control states.

For **negative controls**, conjugate each negative-control wire by `X` before
and after both the enabled-subspace phase synthesis and the corresponding
positive-control construction so that the promoted phase lands on the intended
enabled subspace.

**Rule:** When constructing a controlled gate, preserve the base gate's global
phase exactly on the enabled control subspace. For one positive control, this is
`P(α)` on the control. For multiple positive controls, use the exact
fully-enabled-control phase operator on the active control register. For mixed
positive/negative controls, first conjugate negative controls by `X`, apply that
same exact enabled-subspace phase construction plus the controlled `V`, then
undo the `X` conjugations. Omitting this compensation produces the wrong
controlled unitary by changing relative phases.

#### 8. Ising Interaction Gates

The two-qubit Ising interaction gates follow the same negative-exponent
convention applied to tensor-product Pauli generators:

```
RZZ(θ) = exp(-i * θ * Z⊗Z / 2) = diag(e^(-iθ/2), e^(iθ/2), e^(iθ/2), e^(-iθ/2))
RXX(θ) = exp(-i * θ * X⊗X / 2)
RYY(θ) = exp(-i * θ * Y⊗Y / 2)
RZX(θ) = exp(-i * θ * Z⊗X / 2)
```

These are obtained by basis-change conjugation of `RZZ`:

- `RXX(θ) = (H⊗H) · RZZ(θ) · (H⊗H)` because `H·Z·H = X`
- `RYY(θ) = (RX(π/2)⊗RX(π/2)) · RZZ(θ) · (RX(-π/2)⊗RX(-π/2))` because
  `RX(π/2)·Z·RX(-π/2) = -Y` and `(-Y)⊗(-Y) = Y⊗Y`
- `RZX(θ) = (I⊗H) · RZZ(θ) · (I⊗H)` because `H·Z·H = X` on the second qubit

The sign in the exponent must be consistent with the single-qubit rotation
convention. Do not use `exp(+iθZ⊗Z/2)` for RZZ while using `exp(-iθZ/2)` for RZ.

#### 9. Transpilation and Decomposition Phase Preservation

All transpilation passes must preserve the exact phase-aware semantics of the
circuit. For purely unitary inputs, that means preserving the total unitary
including global phase. For circuits/scopes that contain measurement, reset, or
classical control, passes must instead preserve branch behavior,
measurement/collapse statistics, classical side effects, executable-statement
ordering, and both scope-global and statement-level phase behavior exactly; they
must not assume that one unitary exists for the whole scope:

- **ZYZ decomposition:** Given a 2×2 unitary `M`, extract `α, β, γ, δ` such that
  `M = exp(i*α) * RZ(β) * RY(γ) * RZ(δ)`. Return the canonical ranges from
  Section 2.5. The phase `α` must be tracked explicitly in the returned
  decomposition and, when lowering to instructions, added to the owning scope's
  `globalPhase` expression (equivalently, emitted as a canonical leading
  `globalPhaseGate(α)` that normalizes into that scalar) — not discarded or
  silently hidden inside a neighboring named gate.
- **RZ+SX decomposition:** The decomposition
  `M = exp(i*η) * RZ(a) * SX * RZ(b) * SX * RZ(c)` inherits phase from the ZYZ
  form. Therefore `decomposeToRzSx` must return both the instruction sequence
  and the extra phase `η` (or emit an explicit leading `globalPhaseGate(η)`).
  Since `SX` carries a global phase `exp(i*π/4)` relative to `RX(π/2)`, the
  angle arithmetic and the returned `η` must account for the accumulated
  `exp(i*π/2)` from the two `SX` gates. Verify by recomposing
  `exp(i*η) * RZ(a) * SX * RZ(b) * SX * RZ(c)` and checking exact equality with
  the original.
- **KAK/Weyl two-qubit decomposition:** The decomposition result must carry an
  explicit global phase (for example, `globalPhase` alongside the instruction
  list, or a returned phase-bearing `QuantumCircuit`). The single-qubit gates
  flanking the CX gates together with that returned phase must recompose the
  original 4×4 matrix exactly (within epsilon). Do not drop the global phase of
  the two-qubit unitary.
- **Gate substitution:** When replacing one gate with an equivalent sequence
  (e.g., `H` ≈ `RY(π/2) * RZ(π)` up to global phase), record the compensating
  global phase explicitly in the owning scope's `globalPhase` expression (or as
  a canonical leading `globalPhaseGate`). Transpilation must not silently change
  the circuit's unitary or rely on undocumented phase absorption into
  neighboring named gates.

#### 10. Serialization Phase Preservation

The OpenQASM 3 serializer must preserve both hoistable scope-global phase and
statement-level `gphase` operations. A hoistable `globalPhase` is considered
"nonzero" only when the implementation can prove that its scalar
`exp(i * globalPhase)` is not exactly `1`. Exact constant folding and exact
modulo-`2π` reduction of purely numeric literals are allowed; implementations
are not required to prove that arbitrary symbolic expressions equal `2πk`, and
if they cannot prove unity they must preserve and emit the stored expression.
"Normalized" here means "emit at most one leading bare, unmodified, unannotated
`gphase(θ);` per scope for the hoistable scalar, using the implementation's
canonical stored `AngleExpr` form"; it does not require aggressive symbolic
rewriting or inexact floating simplification. When a scope's hoistable
`globalPhase` is nonzero, emit at most one normalized leading bare, unmodified,
unannotated `gphase(θ);` for that scalar. For a braced scope, emit it
immediately after that scope's opening brace. For the top-level program scope,
emit it immediately after the `OPENQASM ...;` line and any required `include`
directives, and before declarations or executable statements.

The deserializer must parse bare, unmodified `gphase(θ);` wherever it is allowed
in a scope. It may fold such a statement into that scope's scalar only when the
statement is hoistable by Section 2.6. Otherwise preserve it as an ordinary
zero-qubit instruction in source order, together with any attached annotations
and exact expression structure. Modified or annotation-bearing forms such as
`ctrl @ gphase(θ)`, `negctrl @ gphase(θ)`, or `@tag gphase(θ)` are not
scope-global; they must be preserved as instructions (or equivalently desugared
to exact enabled-subspace phase operators) and must not be folded into the scope
scalar. `inv @ gphase(θ)` and `pow(k) @ gphase(θ)` are handled by modifier
expansion first; only if the normalized result is still non-hoistable does it
remain as an ordinary instruction. Round-tripping
(`deserialize(serialize(circuit))`) must preserve exact phase-aware semantics.
For purely unitary scopes, this means the same total unitary including global
phase. For non-unitary scopes, it means the same declarations,
executable-statement order, branch-local phase behavior, and evaluation order of
any non-hoistable `gphase`.

---

## 3. Project Structure

Adapt file extensions to the target language. The logical module layout is:

```
project-root/
├── AGENTS.md
├── README.md
├── src/
│   ├── types.{ext}            # Shared interfaces, type aliases, enums
│   ├── complex.{ext}          # Complex number class (field C)
│   ├── matrix.{ext}           # Matrix class over C
│   ├── gates.{ext}            # Gate matrix constructors (pure functions)
│   ├── parameter.{ext}        # Symbolic parameter & expression system
│   ├── circuit.{ext}          # QuantumCircuit builder (mutable, chainable)
│   ├── backend.{ext}          # Backend interface definition
│   ├── simulator.{ext}        # SimulatorBackend: state-vector simulation engine
│   ├── transpiler.{ext}       # Transpilation pipeline (decomposition, routing, optimization)
│   ├── ibm_backend.{ext}      # IBMBackend: transpile + cloud execution
│   ├── qbraid_backend.{ext}   # QBraidBackend: transpile + cloud execution
│   ├── bloch.{ext}            # Bloch sphere / qubit state introspection
│   ├── serializer.{ext}       # Serializer interface + OpenQASM 3 implementation
│   └── mod.{ext}              # Central re-export hub (public API surface)
├── tests/
│   ├── complex.test.{ext}
│   ├── matrix.test.{ext}
│   ├── gates.test.{ext}
│   ├── parameter.test.{ext}
│   ├── circuit.test.{ext}
│   ├── simulator.test.{ext}
│   ├── transpiler.test.{ext}
│   ├── ibm_backend.test.{ext} # Skipped by default (requires credentials)
│   ├── qbraid_backend.test.{ext} # Skipped by default (requires credentials)
│   ├── bloch.test.{ext}
│   ├── serializer.test.{ext}
│   └── integration.test.{ext}  # End-to-end quantum circuits
└── {build config files}
```

---

## 4. Build & Test Commands

Determine the appropriate commands for the target language. Examples:

| Language   | Build / Check     | Run Tests                | Lint                |
| ---------- | ----------------- | ------------------------ | ------------------- |
| TypeScript | `deno check src/` | `deno test --allow-read` | `deno lint`         |
| Python     | `mypy src/`       | `pytest tests/ -v`       | `ruff check src/`   |
| Rust       | `cargo build`     | `cargo test`             | `cargo clippy`      |
| Go         | `go build ./...`  | `go test ./... -v`       | `golangci-lint run` |

**After every module is written, run tests immediately.** Do not move to the
next module until all tests for the current one pass.

---

## 5. Restrictions

- **Do NOT use any external math/linear-algebra library** for complex numbers,
  matrices, or gate operations. Implement everything from scratch.
- **Do NOT skip or stub any gate, operation, or API endpoint** listed in this
  document. Every single one must be fully implemented.
- **Do NOT generate fewer tests than specified.** The test plan below defines
  minimum counts. Exceed them if possible.
- **Do NOT change the public API surface** (function names, parameter order,
  return types) unless strictly required by the target language's idioms, and
  even then, keep the semantic mapping 1:1.
- **Do NOT use `any` types** (TypeScript) or equivalent type-erasure in other
  languages.
- **Do NOT commit code that doesn't compile or pass tests.**

---

## 6. Implementation Plan — Build Order

Follow this exact order. Each step depends on the previous ones.

---

### Step 1: Types

**File:** `src/types.{ext}`

Define all shared types/interfaces. The exact representation depends on the
target language; the semantic content must match.

```
ClassicalRegister {
  name: string
  size: number
  start: number               // Starting flat classical-bit index in concatenated register order
}

ClassicalBitRef {
  register: string            // Classical register name
  index: number               // Bit index within that register
  flatIndex: number           // Derived absolute classical-bit index
}

AngleExpr = number | Param    // Exact angle/phase expression; minimum support is numeric literals and Param, and implementations may widen this to a richer exact expression tree for parsed OpenQASM 3 expressions

Instruction {
  operation: string            // Gate/operation name (lowercase): "h", "cx", "gphase", "measure", etc.
  qubits: number[]             // Qubit indices this operation acts on
  clbits: number[]             // Flat classical-bit indices in concatenated register order
  clbitRefs: ClassicalBitRef[] // Named classical-bit references aligned with clbits; empty when unused
  params: AngleExpr[]          // Numeric or symbolic angle/phase expressions
  condition?: Condition        // Classical condition (for if_test)
  label?: string               // Optional label for custom gates
  modifiers?: GateModifier[]   // Gate modifiers: ctrl @, negctrl @, inv @, pow(k) @
  annotations?: Annotation[]   // Annotations attached to this instruction
}

Condition {
  register: number | number[] | string | string[]  // Flat bit index/indices or named register(s) in declared order
  value: number                // Integer value to compare against
}

CircuitComplexity {
  numQubits: number
  numClbits: number
  numClassicalRegisters: number
  depth: number
  totalOperations: number
  operationsByName: map<string, number>
  numParameters: number
  globalPhase: AngleExpr
}

BlochCoordinates {
  x: number                    // Tr(rho * X)
  y: number                    // Tr(rho * Y)
  z: number                    // Tr(rho * Z)
  theta: number                // Polar angle [0, pi]
  phi: number                  // Azimuthal angle [0, 2*pi)
  r: number                    // Purity radius sqrt(x^2 + y^2 + z^2)
}

CorsProxyConfiguration {
  enabled: boolean             // default: false
  mode: "browser-only" | "always"  // default: "browser-only"
  baseUrl: string              // default: "https://proxy.corsfix.com/?"
}

BackendConfiguration {
  name: string
  numQubits: number
  basisGates: string[]
  couplingMap: [number, number][] | null   // null = all-to-all
  corsProxy?: CorsProxyConfiguration | null // optional HTTP CORS proxy configuration for cloud backends
}

IBMBackendConfiguration extends BackendConfiguration {
  qubitProperties: map<number, QubitProperties>
  gateProperties: map<string, map<string, GateProperties>>  // gate -> qubit_tuple -> props
  maxCircuits: number | null
  apiEndpoint: string           // default: "https://quantum.cloud.ibm.com/api/v1"
  bearerToken?: string          // Direct bearer token for IBM Quantum runtime requests; mutually exclusive with apiKey
  apiKey?: string               // IBM Cloud API key used to mint an IAM bearer token; mutually exclusive with bearerToken
  serviceCrn: string            // required Service-CRN header value for IBM Quantum REST API
  apiVersion: string            // required IBM-API-Version header value (default: "2026-02-15")
}

QBraidBackendConfiguration extends BackendConfiguration {
  qubitProperties: map<number, QubitProperties>
  gateProperties: map<string, map<string, GateProperties>>  // gate -> qubit_tuple -> props
  deviceQrn: string             // qBraid Quantum Resource Name returned by the v2 device endpoints (e.g., "qbraid:qbraid:sim:qir-sv")
  apiEndpoint: string           // default: "https://api-v2.qbraid.com/api/v1"
  apiKey: string                // X-API-KEY header value
}

QubitProperties {
  t1: number             // T1 relaxation time in microseconds
  t2: number             // T2 dephasing time in microseconds
  frequency: number      // Qubit resonance frequency in GHz
  readoutError: number   // Measurement error probability
}

GateProperties {
  errorRate: number      // Probability of gate error
  duration: number       // Gate execution time in nanoseconds
}

Target {
  numQubits: number
  gates: map<string, map<string, GateProperties>>  // gate -> qubit_tuple -> props
}

ExecutionResult = map<string, number>  // bitstring -> percentage (0-100); if multiple classical registers exist, flatten by concatenating register segments in declared order

SerializedCircuit = string  // OpenQASM 3 program text

// --- OpenQASM 3 Classical Type System ---
// These types model the full OpenQASM 3 classical type system for the
// serializer/deserializer. They are used internally by OpenQASM3Serializer
// and exposed publicly so users can inspect deserialized programs.

ClassicalType = "bit" | "bool" | "int" | "uint" | "float" | "angle"
                | "complex" | "duration" | "stretch"

ClassicalVariable {
  name: string
  type: ClassicalType
  size: number | null          // Bit width (e.g., 32 for int[32]); null for unsized
  isConst: boolean             // true for `const` declarations
  isInput: boolean             // true for `input` declarations
  isOutput: boolean            // true for `output` declarations
  initValue: any | null        // Initial value expression, if present
}

GateDefinition {
  name: string                 // Gate name
  params: string[]             // Parameter names (angle parameters)
  qubits: string[]             // Qubit argument names
  body: Instruction[]          // Gate body instructions
  globalPhase: AngleExpr       // Accumulated phase for this gate body scope
}

GateModifier {
  type: "ctrl" | "negctrl" | "inv" | "pow"
  argument: number | null      // Number of controls for ctrl/negctrl; exponent for pow; null for inv
}

SubroutineDefinition {
  name: string
  params: SubroutineParam[]    // Classical and quantum parameters
  returnType: ClassicalType | null  // null for void
  body: Instruction[]
  globalPhase: AngleExpr       // Accumulated phase for this subroutine body scope
}

SubroutineParam {
  name: string
  type: ClassicalType | "qubit" | "qubit[]"
  size: number | null
  isMutable: boolean           // For array params: readonly vs mutable
}

ExternDeclaration {
  name: string
  params: SubroutineParam[]
  returnType: ClassicalType | null
}

ArrayType {
  baseType: ClassicalType      // int, uint, float, complex, angle, bool, duration
  dimensions: number[]         // Up to 7 dimensions
}

Pragma {
  content: string              // Raw pragma text after "pragma " keyword
}

Annotation {
  content: string              // Raw annotation text after "@" symbol
}
```

The base `Instruction` record above describes the shared fields present on
ordinary operations. Scoped/control-flow operations (`ifTest`, `forLoop`,
`whileLoop`, `switch`, `box`) additionally carry nested `QuantumCircuit` body
objects, or language-idiomatic tagged-union/subclass equivalents, rather than
flattening those bodies into plain `Instruction[]`. This is required so every
nested scope can own its own `globalPhase`, declarations, and classical-register
metadata.

The circuit model must preserve `ClassicalRegister[]` in declaration order. Flat
classical-bit indices are a derived view obtained by concatenating those
registers in order, then the bits within each register in ascending local index.

Canonical phase representation:

- `QuantumCircuit`, `GateDefinition`, and `SubroutineDefinition` each own a
  scalar `globalPhase` expression (`AngleExpr`) separate from their ordered
  `Instruction[]`. This scalar stores only hoistable scope-global phase
  metadata; not every surface-syntax `gphase` statement is representable solely
  by this field.
- `globalPhaseGate(theta)` adds to the owning scope's scalar; in the normalized
  in-memory representation a hoistable **bare, unmodified** zero-qubit phase is
  not stored as an ordinary instruction. If a lower-level representation or
  parser applies modifiers first, `inv @ gphase(theta)` and
  `pow(k) @
  gphase(theta)` may normalize to a bare phase and then follow the
  same hoisting rule.
- When parsing OpenQASM 3, fold only hoistable bare, unmodified, unannotated
  `gphase(theta);` statements encountered in a given scope into that scope's
  scalar by addition. A statement is hoistable only if its expression can be
  moved to the scope entry without changing meaning: all referenced symbols are
  already in scope there, remain immutable throughout the scope, and no
  annotation or source-order information would be lost. Non-hoistable bare
  `gphase` remains an ordinary zero-qubit instruction in source order.
  Control-flow bodies are nested `QuantumCircuit` scopes, so branch-local phases
  remain branch-local. Modifier-bearing forms such as `ctrl @ gphase(theta)`,
  and annotated bare forms such as `@tag gphase(theta)`, are not scope-global
  and must remain ordinary relative-phase instructions (or be desugared to
  equivalent controlled-phase operations).
- When serializing, emit at most one normalized leading bare `gphase(theta);`
  per scope for the scalar `globalPhase`. Emit any remaining statement-level
  bare `gphase` instructions in their original relative order. For braced
  scopes, emit the scalar immediately after the opening brace and before the
  rest of the body statements. For the top-level program scope, emit the scalar
  immediately after the `OPENQASM` version line and any required `include`
  directives, and before declarations or executable statements; because the
  scalar is hoistable by construction, it must not depend on names declared
  later in that scope. "Normalized" here means one leading hoistable
  `gphase(theta);` per scope in the implementation's canonical stored
  `AngleExpr` form; it does not require proving symbolic `2πk = 0` identities
  beyond the exact simplifications the implementation already supports.

Shot count is a **per-execution / per-submitted-job input**, not a backend
capability field. Do not store it on `BackendConfiguration`; carry it in the
language-idiomatic execution/packaging call and in the resulting executable/job
payload.

HTTP proxying is transport configuration, not backend capability metadata. If
present, `corsProxy` applies only to outbound HTTP requests made by cloud
backends. For TypeScript/JavaScript implementations, when
`corsProxy.enabled = true`, automatically detect whether the code is running in
a browser. If `mode = "browser-only"`, only browser-side fetches are rewritten
to `corsProxy.baseUrl + originalUrl`; Node/Deno/server runtimes continue to use
the original URL. Use `https://proxy.corsfix.com/?` as the default proxy base
URL for TypeScript/JavaScript implementations. The mechanism must be opt-in and
explicitly disableable.

**Tests (minimum 15):**

- Verify `ClassicalRegister` creation, flat offsets, and declaration order.
- Verify `ClassicalBitRef` creation and flat-index mapping.
- Verify Instruction creation with flat `clbits` and named `clbitRefs`.
- Verify Instruction with `modifiers` field (ctrl, negctrl, inv, pow).
- Verify Instruction with `annotations` field.
- Verify Condition with single bit, multi-bit register, and named register.
- Verify CircuitComplexity struct defaults and field types, including
  `numClassicalRegisters`.
- Verify BlochCoordinates struct field ranges.
- Verify BackendConfiguration creation with and without coupling map.
- Verify optional `corsProxy` configuration defaults to disabled and supports
  `browser-only` and `always` modes.
- Verify IBMBackendConfiguration includes `serviceCrn`, `apiVersion`, and
  exactly one of `bearerToken` or `apiKey`.
- Verify `ClassicalVariable` creation for each `ClassicalType` (bit, bool, int,
  uint, float, angle, complex, duration, stretch) with size, const, input, and
  output flags.
- Verify `GateDefinition` creation with params, qubit args, and symbolic or
  numeric `globalPhase`.
- Verify `GateModifier` for each type: ctrl(n), negctrl(n), inv, pow(k).
- Verify `SubroutineDefinition` and `ExternDeclaration` with params and return
  type, including symbolic or numeric `SubroutineDefinition.globalPhase`.
- Verify `ArrayType` with base types and up to 7 dimensions.
- Verify `Pragma` and `Annotation` creation.
- Verify scoped control-flow instructions store nested `QuantumCircuit` bodies
  so branch-local `globalPhase` is representable.
- Verify shot count is modeled as a per-execution/per-job input rather than a
  `BackendConfiguration` field.
- Verify TypeScript/JavaScript browser-runtime URL rewriting uses
  `https://proxy.corsfix.com/?<originalUrl>` only when
  `corsProxy.enabled =
  true`, and remains direct when disabled.
- Verify total `numClbits` equals the sum of named classical register sizes.
- Edge cases: 0 qubits, 0 classical bits, empty instructions list, empty
  classical-register list.
- Verify ExecutionResult percentages summing to 100.
- Type validation edge cases.

---

### Step 2: Complex Numbers

**File:** `src/complex.{ext}`

Implement a `Complex` class representing a complex number `a + bi`. All methods
return new instances (immutable).

#### Constructor and Constants

- `Complex(re: number, im: number)` — create from real and imaginary parts.
- `Complex.ZERO` = `0 + 0i`
- `Complex.ONE` = `1 + 0i`
- `Complex.I` = `0 + 1i`
- `Complex.MINUS_I` = `0 - 1i`
- `Complex.fromPolar(r: number, theta: number) -> Complex` —
  `r * (cos(theta) + i*sin(theta))`
- `Complex.exp(theta: number) -> Complex` — Euler's formula:
  `e^(i*theta) = cos(theta) + i*sin(theta)`

#### Arithmetic Methods

- `add(other: Complex) -> Complex` — `(a+bi) + (c+di) = (a+c) + (b+d)i`
- `sub(other: Complex) -> Complex` — `(a+bi) - (c+di) = (a-c) + (b-d)i`
- `mul(other: Complex) -> Complex` — `(a+bi)(c+di) = (ac-bd) + (ad+bc)i`
- `div(other: Complex) -> Complex` — division by complex number
- `scale(scalar: number) -> Complex` — multiply by real scalar
- `conjugate() -> Complex` — `(a+bi)* = a - bi`
- `magnitude() -> number` — `|a+bi| = sqrt(a^2 + b^2)`
- `magnitudeSquared() -> number` — `|a+bi|^2 = a^2 + b^2` (used in Born rule —
  avoid sqrt)
- `phase() -> number` — `atan2(b, a)`
- `neg() -> Complex` — `-(a+bi) = -a - bi`
- `equals(other: Complex, epsilon?) -> boolean` — approximate equality
- `toString() -> string` — human-readable representation

**Tests (minimum 40):**

- Arithmetic: add, sub, mul, div with positive, negative, zero, pure-real,
  pure-imaginary operands.
- Conjugate of real, imaginary, general complex.
- Magnitude and magnitudeSquared for known values (3+4i -> 5, 25).
- Phase for all four quadrants.
- `Complex.exp(0) = 1`, `Complex.exp(pi/2) = i`, `Complex.exp(pi) = -1`,
  `Complex.exp(3*pi/2) = -i`, `Complex.exp(2*pi) = 1`.
- `Complex.fromPolar(1, 0) = 1`, `Complex.fromPolar(1, pi/2) = i`.
- Constants: ZERO, ONE, I, MINUS_I verify real and imaginary parts.
- Division by non-zero, division edge cases.
- `z.mul(z.conjugate()).im ~ 0` for any z (result is real).
- `z.magnitude() ~ sqrt(z.magnitudeSquared())` for various z.
- Immutability: calling add/sub/mul does not mutate the original.
- `neg()`: verify `z.add(z.neg()).equals(Complex.ZERO)`.
- `equals` with various epsilon values.

---

### Step 3: Matrix Operations

**File:** `src/matrix.{ext}`

Implement a `Matrix` class for matrices over C. Internally store a flat or 2D
array of `Complex`. All methods return new instances.

#### Constructor and Factories

- `Matrix(rows: number, cols: number, data: Complex[][])` — create from 2D
  array.
- `Matrix.identity(size: number) -> Matrix` — identity matrix of given size.
- `Matrix.zeros(rows: number, cols: number) -> Matrix` — zero matrix.

#### Operations

- `get(row, col) -> Complex` — element access.
- `multiply(other: Matrix) -> Matrix` — standard matrix multiplication.
- `add(other: Matrix) -> Matrix` — element-wise addition.
- `scale(scalar: Complex) -> Matrix` — multiply every element by a complex
  scalar.
- `dagger() -> Matrix` — conjugate transpose (dagger).
- `tensor(other: Matrix) -> Matrix` — Kronecker/tensor product.
- `apply(stateVector: Complex[]) -> Complex[]` — matrix-vector multiplication.
- `isUnitary(epsilon?) -> boolean` — check if `M_dagger * M ~ I`.
- `trace() -> Complex` — sum of diagonal elements.
- `determinant() -> Complex` — determinant (at least for 2x2; general via LU or
  cofactor expansion for small matrices).
- `equals(other: Matrix, epsilon?) -> boolean` — element-wise approximate
  equality.
- `toString() -> string` — human-readable.
- `rows` / `cols` — dimension accessors.

**Tests (minimum 45):**

- Identity: `I*A = A`, `A*I = A` for several A.
- `(A*B)*C = A*(B*C)` associativity for small matrices.
- `A.dagger().dagger().equals(A)` for several matrices.
- `(A tensor B)` dimensions: (m x n) tensor (p x q) = (mp x nq).
- Tensor product of identity matrices: `I2 tensor I2 = I4`.
- Pauli matrices: `X^2 = I`, `Y^2 = I`, `Z^2 = I`, `X*Y = iZ`, etc.
- `X.isUnitary() = true`, `H.isUnitary() = true`.
- Non-unitary matrix returns `isUnitary() = false`.
- `apply` on |0> and |1> basis states with X, H, Z.
- Matrix addition commutativity and element correctness.
- Scale by Complex.ZERO gives zero matrix.
- Scale by Complex.ONE gives same matrix.
- Trace of identity = n.
- Trace of Pauli matrices = 0.
- Determinant of identity = 1, det of Pauli X = -1.
- `equals` symmetry: `A.equals(B) <-> B.equals(A)`.

---

### Step 4: Gate Definitions — Compositional Architecture

**File:** `src/gates.{ext}`

Every gate is a **pure function** returning a `Matrix`. Implement ALL of the
following — no omissions.

#### Foundational Principle: Universality of Single-Qubit + CNOT

Following Nielsen & Chuang §1.3.2 — **any multi-qubit unitary can be decomposed
into single-qubit gates and CNOT (CX) gates.** These two ingredients form a
universal gate set for quantum computation.

This library embraces this principle architecturally: all multi-qubit gates are
**defined as compositions** of single-qubit gates and CX. The gate functions
compute their matrices by composing the constituent gate matrices via matrix
multiplication and tensor products — not by hardcoding matrix entries.

The gates are organized into **tiers of abstraction**, where each tier normally
builds exclusively on gates from lower tiers. When a gate is explicitly marked
as a **direct synthesis** or **optimization**, it may bypass intermediate named
abstractions, but it must still be composed only from Tier 0 single-qubit gates
and the Tier 1 CX gate. No larger hardcoded matrix is introduced.

```
Tier 0: Single-qubit primitives (matrix definitions)
Tier 1: CX — the universal entangling primitive (matrix definition)
Tier 2: Fundamental two-qubit compositions (from Tier 0 + Tier 1)
Tier 3: Higher two-qubit compositions (from Tier 2)
Tier 4: Three-qubit compositions (from Tier 2 + Tier 3)
Tier 5: Four-qubit and multi-controlled compositions (from Tier 4)
Tier 6: N-qubit composite gates (from lower tiers)
```

**Implementation:** Each multi-qubit gate function builds its result by
composing lower-tier gate matrices, or, when explicitly noted, by a direct Tier
0 + Tier 1 synthesis. For example, `czGate()` internally calls `hadamard()` and
`cxGate()`, constructs the appropriate tensor products, and multiplies them. No
4x4, 8x8, or 16x16 matrix is hardcoded.

**Notation convention:** Decompositions are written in **circuit time order**
(left to right), which matches the order operations are appended to a
`QuantumCircuit` builder. The symbol `→` means "followed by." In standard matrix
multiplication, the total unitary for a circuit `A → B → C` is `C · B · A` (the
leftmost gate A is applied first, so it appears rightmost in the matrix
product). Proofs in this document write matrix products in the standard
mathematical convention (leftmost factor is applied last).

To keep the notation unambiguous throughout this section:

- `→` always denotes **circuit time order** (earliest operation on the left).
- `·` and `*` denote **matrix-product order**.
- If a helper block is defined algebraically, for example
  `A = Rz(phi) * Ry(theta/2)`, that is a matrix identity; the corresponding
  circuit-time sequence on that qubit is `Ry(theta/2) → Rz(phi)`.
- Basis-state examples for multi-qubit gates are written in the same MSB-first
  local argument order used by the gate matrices. For example, `|c,t⟩` for a
  controlled gate and `|c1,c2,t⟩` for a doubly-controlled gate.

---

#### Tier 0: Single-Qubit Primitives (2×2 Matrices — defined from scratch)

These are the atomic building blocks. Each is defined by its explicit 2×2
matrix. Together with CX, they form the universal alphabet.

**Zero-Qubit (Global) Gate:**

| Function                 | Gate         | Matrix / Description                                   |
| ------------------------ | ------------ | ------------------------------------------------------ |
| `globalPhaseGate(theta)` | Global Phase | `[[exp(i*theta)]]` (1×1 matrix, multiplies full state) |

**Single-Qubit Gates (return 2×2 Matrix):**

| Function             | Gate        | Matrix Definition                                                                    |
| -------------------- | ----------- | ------------------------------------------------------------------------------------ |
| `identityGate()`     | I           | `[[1, 0], [0, 1]]`                                                                   |
| `hadamard()`         | H           | `(1/sqrt(2)) * [[1, 1], [1, -1]]`                                                    |
| `pauliX()`           | X           | `[[0, 1], [1, 0]]`                                                                   |
| `pauliY()`           | Y           | `[[0, -i], [i, 0]]`                                                                  |
| `pauliZ()`           | Z           | `[[1, 0], [0, -1]]`                                                                  |
| `pGate(lambda)`      | P(l)        | `[[1, 0], [0, exp(i*l)]]`                                                            |
| `rGate(theta, phi)`  | R(th,ph)    | `[[cos(th/2), -i*e^(-i*ph)*sin(th/2)], [-i*e^(i*ph)*sin(th/2), cos(th/2)]]`          |
| `rxGate(theta)`      | RX(th)      | `[[cos(th/2), -i*sin(th/2)], [-i*sin(th/2), cos(th/2)]]`                             |
| `ryGate(theta)`      | RY(th)      | `[[cos(th/2), -sin(th/2)], [sin(th/2), cos(th/2)]]`                                  |
| `rzGate(theta)`      | RZ(th)      | `[[exp(-i*th/2), 0], [0, exp(i*th/2)]]`                                              |
| `sGate()`            | S           | `[[1, 0], [0, i]]`                                                                   |
| `sdgGate()`          | Sdg         | `[[1, 0], [0, -i]]`                                                                  |
| `sxGate()`           | SX          | `(1/2) * [[1+i, 1-i], [1-i, 1+i]]`                                                   |
| `sxdgGate()`         | SXdg        | `(1/2) * [[1-i, 1+i], [1+i, 1-i]]`                                                   |
| `tGate()`            | T           | `[[1, 0], [0, exp(i*pi/4)]]`                                                         |
| `tdgGate()`          | Tdg         | `[[1, 0], [0, exp(-i*pi/4)]]`                                                        |
| `uGate(th, ph, l)`   | U(th,ph,l)  | `[[cos(th/2), -exp(i*l)*sin(th/2)], [exp(i*ph)*sin(th/2), exp(i*(ph+l))*cos(th/2)]]` |
| `u1Gate(lambda)`     | U1(l)       | Equivalent to `pGate(l)`: `[[1, 0], [0, exp(i*l)]]`                                  |
| `u2Gate(phi, l)`     | U2(ph,l)    | `(1/sqrt(2)) * [[1, -exp(i*l)], [exp(i*ph), exp(i*(ph+l))]]`                         |
| `u3Gate(th, ph, l)`  | U3(th,ph,l) | Identical to `uGate(th, ph, l)`                                                      |
| `rvGate(vx, vy, vz)` | RV(v)       | Rotation around axis v=(vx,vy,vz) by angle \|v\|                                     |

**RV Gate formula:** Let `angle = |v|/2`, `n = v/|v|` (unit vector). If |v| = 0,
return identity.

```
[[cos(angle) - i*nz*sin(angle),    (-i*nx - ny)*sin(angle)],
 [(-i*nx + ny)*sin(angle),          cos(angle) + i*nz*sin(angle)]]
```

**Equivalences within Tier 0:**

- `T = P(pi/4)`, `S = P(pi/2)`, `Z = P(pi)`
- `Sdg = S†`, `Tdg = T†`, `SXdg = SX†`
- `SX * SX = X`, `SX * SXdg = I`
- `U1(l) = P(l)`, `U3(th,ph,l) = U(th,ph,l)`
- `X = U(pi, 0, pi)`, `H = U(pi/2, 0, pi)`, `Y = U(pi, pi/2, pi/2)`

---

#### Tier 1: The Universal Entangling Primitive — CX

The **only** multi-qubit gate defined by an explicit matrix. Together with the
Tier 0 single-qubit gates, CX is sufficient to construct **any** unitary
operation on any number of qubits (Nielsen & Chuang, Corollary 4.2).

| Function   | Gate      | Matrix                                      |
| ---------- | --------- | ------------------------------------------- |
| `cxGate()` | CX / CNOT | `[[1,0,0,0],[0,1,0,0],[0,0,0,1],[0,0,1,0]]` |

MSB-first ordering: bit 1 = control, bit 0 = target. Identity on |00⟩,|01⟩
(control=0); X on target for |10⟩,|11⟩ (control=1).

---

#### General Controlled-U Construction (ABC Decomposition)

Before defining Tier 2, we establish the **master recipe** from Nielsen & Chuang
(Corollary 4.2) for building a controlled version of any single-qubit unitary.

For any single-qubit U, write `U = exp(i*alpha) * A * X * B * X * C` where
`A * B * C = I`. Then:

```
Controlled-U(control, target) =
    C(t) → CX(c,t) → B(t) → CX(c,t) → A(t) → P(alpha)(c)
```

**When control = 0:** CX does nothing, target sees `A * B * C = I`. ✓ **When
control = 1:** CX applies X, target sees
`A * X * B * X * C = exp(-i*alpha) * U`, and `P(alpha)` on control contributes
`exp(i*alpha)` when c=1, so the total action on the controlled subspace is
exactly U. ✓

Note on notation: decompositions are written in **circuit time order** (left to
right). In standard matrix multiplication, the leftmost gate is applied first,
and the total unitary is the rightmost matrix times … times the leftmost. So
`C → CX → B → CX → A` means C acts first, A acts last; when control = 0 the
resulting matrix on the target is `A · B · C = I` (reading right-to-left in
matrix product).

**This uses exactly 2 CX gates + 4 single-qubit gates.**

For the specific case of `U(theta, phi, lambda)`:

```
A = Rz(phi) * Ry(theta/2)
B = Ry(-theta/2) * Rz(-(phi+lambda)/2)
C = Rz((lambda-phi)/2)
alpha = (phi + lambda) / 2
```

Here `A`, `B`, and `C` are defined in matrix-product order. Therefore the actual
circuit-time implementation is:

- `C` as written
- then `Rz(-(phi+lambda)/2) → Ry(-theta/2)` for `B`
- then `Ry(theta/2) → Rz(phi)` for `A`

Verify:
`A * B * C = Rz(phi) * Ry(theta/2) * Ry(-theta/2) * Rz(-(phi+lambda)/2) * Rz((lambda-phi)/2) = Rz(phi) * I * Rz(-phi) = I`
✓

For `CU(theta, phi, lambda, gamma)` (with extra controlled global phase gamma):

```
CU(th,ph,l,gamma, c, t) =
    Rz((l-ph)/2)(t) →                  // C on target
    CX(c,t) →
    Rz(-(ph+l)/2)(t) → Ry(-th/2)(t) → // B on target
    CX(c,t) →
    Ry(th/2)(t) → Rz(ph)(t) →          // A on target
    P(gamma + (ph+l)/2)(c)             // phase on control
```

This single-positive-control construction is used by every Tier 2 controlled
gate below.

---

#### Tier 2: Fundamental Two-Qubit Compositions

Every gate in this tier is built **directly** from Tier 0 single-qubit gates and
the Tier 1 CX gate. Each function computes its matrix by composing the
constituent matrices.

**CZ — Controlled-Z (1 CX)**

```
CZ(c, t) = H(t) → CX(c, t) → H(t)
```

Proof: When control=1, target sees `H * X * H = Z`. ✓

**CY — Controlled-Y (1 CX)**

```
CY(c, t) = Sdg(t) → CX(c, t) → S(t)
```

Proof: When control=1, target sees `S * X * Sdg = Y`. ✓ Verify:
`S*X*Sdg = [[1,0],[0,i]]*[[0,1],[1,0]]*[[1,0],[0,-i]] = [[0,-i],[i,0]] = Y`. ✓

**CP — Controlled-Phase (2 CX)**

```
CP(lambda, c, t) =
    P(lambda/2)(t) → CX(c,t) → P(-lambda/2)(t) → CX(c,t) → P(lambda/2)(c)
```

Proof: When c=0: `P(-l/2)*P(l/2) = I` on target, no phase on control. ✓ When
c=1,t=1: target sees `X*P(-l/2)*X*P(l/2)` (4 operations), and `P(l/2)` on
control contributes `exp(i*l/2)`, for total `exp(i*l)` phase on |11⟩. ✓

**CRZ — Controlled-RZ (2 CX)**

```
CRZ(theta, c, t) = RZ(theta/2)(t) → CX(c,t) → RZ(-theta/2)(t) → CX(c,t)
```

Proof: When c=0: `RZ(-th/2)*RZ(th/2) = I`. ✓ When c=1:
`X*RZ(-th/2)*X = RZ(th/2)`, so total = `RZ(th/2)*RZ(th/2) = RZ(th)`. ✓

**CRY — Controlled-RY (2 CX)**

```
CRY(theta, c, t) = RY(theta/2)(t) → CX(c,t) → RY(-theta/2)(t) → CX(c,t)
```

Proof: `X*RY(-th/2)*X = RY(th/2)`, so when c=1: `RY(th/2)*RY(th/2) = RY(th)`. ✓

**CRX — Controlled-RX (abstraction on CRZ, 2 CX)**

```
CRX(theta, c, t) = H(t) → CRZ(theta, c, t) → H(t)
```

Proof: `H*RZ(th)*H = RX(th)`, so controlled version transforms CRZ→CRX. ✓
Expanding: `H(t) → RZ(th/2)(t) → CX(c,t) → RZ(-th/2)(t) → CX(c,t) → H(t)`.

**CS — Controlled-S (abstraction on CP, 2 CX)**

```
CS(c, t) = CP(pi/2, c, t)
```

Since `S = P(pi/2)`, controlled-S is controlled-phase with `lambda = pi/2`.

**CSdg — Controlled-Sdg (abstraction on CP, 2 CX)**

```
CSdg(c, t) = CP(-pi/2, c, t)
```

**CU1 — Controlled-U1 (abstraction on CP, 2 CX)**

```
CU1(lambda, c, t) = CP(lambda, c, t)
```

Since `U1 = P`.

**CSX — Controlled-SX (2 CX)**

Since `SX = exp(i*pi/4) * RX(pi/2)`, the controlled version picks up the global
phase as a relative phase on the control:

```
CSX(c, t) = P(pi/4)(c) → CRX(pi/2, c, t)
```

Expanding:
`P(pi/4)(c) → H(t) → RZ(pi/4)(t) → CX(c,t) → RZ(-pi/4)(t) → CX(c,t) → H(t)`.

**CSXdg — Controlled-SXdg (2 CX)**

Since `SXdg = exp(-i*pi/4) * RX(-pi/2)`, the controlled version is:

```
CSXdg(c, t) = P(-pi/4)(c) → CRX(-pi/2, c, t)
```

Expanding:
`P(-pi/4)(c) → H(t) → RZ(-pi/4)(t) → CX(c,t) → RZ(pi/4)(t) → CX(c,t) → H(t)`.

**CH — Controlled-Hadamard (2 CX, from ABC decomposition)**

Using the general controlled-U recipe with the ZYZ decomposition of H:
`H = exp(i*pi/2) * Rz(0) * Ry(pi/2) * Rz(pi)`, so `alpha = pi/2`, `beta = 0`,
`gamma_ry = pi/2`, `delta = pi`.

```
A = Ry(pi/4)
B = Ry(-pi/4) * Rz(-pi/2)
C = Rz(pi/2)

CH(c, t) =
    Rz(pi/2)(t) →                      // C = Rz(pi/2)
    CX(c, t) →
    Rz(-pi/2)(t) → Ry(-pi/4)(t) →     // B (Rz first, then Ry in time order)
    CX(c, t) →
    Ry(pi/4)(t) →                      // A
    P(pi/2)(c)                          // phase
```

**CU — General Controlled-U (2 CX)**

```
CU(theta, phi, lambda, gamma, c, t) =
    Rz((lambda-phi)/2)(t) →
    CX(c, t) →
    Rz(-(phi+lambda)/2)(t) → Ry(-theta/2)(t) →
    CX(c, t) →
    Ry(theta/2)(t) → Rz(phi)(t) →
    P(gamma + (phi+lambda)/2)(c)
```

**CU3 — Controlled-U3 (abstraction on CU, 2 CX)**

```
CU3(theta, phi, lambda, c, t) = CU(theta, phi, lambda, 0, c, t)
```

**DCX — Double-CNOT (2 CX)**

```
DCX(a, b) = CX(a, b) → CX(b, a)
```

Directly composed from two CX gates with swapped roles.

---

#### Tier 3: Higher Two-Qubit Compositions

Gates in this tier are built from Tier 2 abstractions (and transitively from
Tier 0 + Tier 1). They compose controlled gates and other Tier 2/3 gates.

**SWAP (3 CX)**

```
SWAP(a, b) = CX(a, b) → CX(b, a) → CX(a, b)
```

The canonical 3-CNOT SWAP decomposition.

**RZZ — ZZ Ising Interaction (2 CX)**

```
RZZ(theta, a, b) = CX(a, b) → RZ(theta)(b) → CX(a, b)
```

Proof: CX entangles the parity of both qubits into qubit b. RZ rotates based on
that parity. CX unentangles. The net effect is `exp(-i*theta/2 * Z⊗Z)`. Verify
on basis states:

- |00⟩ → CX → |00⟩ → RZ(th)|0⟩ = e^(-ith/2)|00⟩ → CX → e^(-ith/2)|00⟩ ✓
- |01⟩ → CX → |01⟩ → RZ(th)|1⟩ = e^(ith/2)|01⟩ → CX → e^(ith/2)|01⟩ ✓
- |10⟩ → CX → |11⟩ → e^(ith/2)|11⟩ → CX → e^(ith/2)|10⟩ ✓
- |11⟩ → CX → |10⟩ → e^(-ith/2)|10⟩ → CX → e^(-ith/2)|11⟩ ✓

**RXX — XX Ising Interaction (abstraction on RZZ, 2 CX)**

```
RXX(theta, a, b) = H(a) → H(b) → RZZ(theta, a, b) → H(a) → H(b)
```

Proof: `H*Z*H = X`, so `(H⊗H) * (Z⊗Z) * (H⊗H) = X⊗X`. Conjugating
`exp(-i*th/2 * Z⊗Z)` by `H⊗H` gives `exp(-i*th/2 * X⊗X)`. ✓

**RYY — YY Ising Interaction (abstraction on RZZ, 2 CX)**

```
RYY(theta, a, b) = RX(pi/2)(a) → RX(pi/2)(b) → RZZ(theta, a, b) → RX(-pi/2)(a) → RX(-pi/2)(b)
```

Proof: `RX(pi/2)*Z*RX(-pi/2) = -Y` (conjugation), so
`(RX(pi/2)⊗RX(pi/2)) * (Z⊗Z) * (RX(-pi/2)⊗RX(-pi/2)) = (-Y)⊗(-Y) = Y⊗Y`. ✓

**RZX — ZX Cross-Resonance Interaction (abstraction on RZZ, 2 CX)**

```
RZX(theta, a, b) = H(b) → RZZ(theta, a, b) → H(b)
```

Proof: Conjugation by `I⊗H` transforms `Z⊗Z → Z⊗X`. ✓ Expanding:
`H(b) → CX(a,b) → RZ(theta)(b) → CX(a,b) → H(b)`.

**ECR — Echoed Cross-Resonance (abstraction on RZX, 2 CX)**

```
ECR(a, b) = RZX(pi/2, a, b) → X(a)
```

This is the exact matrix identity: `ECR = (X⊗I) * RZX(pi/2)`. Expanding:
`H(b) → CX(a,b) → RZ(pi/2)(b) → CX(a,b) → H(b) → X(a)`.

**iSWAP (abstraction on SWAP + CZ, 4 CX)**

```
iSWAP(a, b) = CZ(a, b) → SWAP(a, b) → S(a) → S(b)
```

Proof on basis states:

- |01⟩ → CZ → |01⟩ → SWAP → |10⟩ → S⊗S → i|10⟩ ✓
- |10⟩ → CZ → |10⟩ → SWAP → |01⟩ → S⊗S → i|01⟩ ✓
- |00⟩ → CZ → |00⟩ → SWAP → |00⟩ → S⊗S → |00⟩ ✓
- |11⟩ → CZ → -|11⟩ → SWAP → -|11⟩ → S⊗S → -i²|11⟩ = |11⟩ ✓

**XX+YY — Parameterized (XX+YY) Interaction (abstraction on CRX, 4 CX)**

Acts on the {|01⟩, |10⟩} subspace with a phase twist beta.

```
XX+YY(theta, beta, a, b) =
    RZ(beta)(b) → CX(a, b) → CRX(theta, b, a) → CX(a, b) → RZ(-beta)(b)
```

Proof: CX(a,b) maps {|01⟩,|10⟩} to {|01⟩,|11⟩}. In this subspace, b=1 for both
states, so CRX(theta, b, a) applies RX(theta) to qubit a. The second CX unmaps
back. RZ(±beta) applies the phase twist.

- |00⟩: CX→|00⟩, CRX(b=0)→|00⟩, CX→|00⟩ → identity ✓
- |11⟩: CX→|10⟩, CRX(b=0)→|10⟩, CX→|11⟩ → identity ✓
- |01⟩: CX→|01⟩, CRX(b=1) applies RX(th) to a, CX unmaps ✓
- |10⟩: CX→|11⟩, CRX(b=1) applies RX(th) to a, CX unmaps ✓

**XX-YY — Parameterized (XX-YY) Interaction (abstraction on CRX, 4 CX)**

Acts on the {|00⟩, |11⟩} subspace with a phase twist beta.

```
XX-YY(theta, beta, a, b) =
    RZ(-beta)(b) → X(b) → CX(a, b) → CRX(theta, b, a) → CX(a, b) → X(b) → RZ(beta)(b)
```

Proof: X(b) flips qubit b, mapping {|00⟩,|11⟩} to {|01⟩,|10⟩}. Then CX maps to
{|01⟩,|11⟩} where b=1, enabling CRX. The sequence reverses the mappings.

- |00⟩: X→|01⟩, CX→|01⟩, CRX(b=1): RX(th) on a, CX→result, X→result ✓
- |11⟩: X→|10⟩, CX→|11⟩, CRX(b=1): RX(th) on a, CX→result, X→result ✓
- |01⟩: X→|00⟩, CX→|00⟩, CRX(b=0): identity, CX→|00⟩, X→|01⟩ → identity ✓
- |10⟩: X→|11⟩, CX→|10⟩, CRX(b=0): identity, CX→|11⟩, X→|10⟩ → identity ✓

---

#### Tier 4: Three-Qubit Compositions

The canonical exact three-qubit gates in this tier are composed from Tier 2 and
Tier 3 gates. Relative-phase optimized variants may also be given as direct Tier
0 + Tier 1 syntheses when explicitly marked as such.

**CCX — Toffoli / Doubly-Controlled X**

**V-decomposition** (Barenco et al., 1995): Since `V² = X` where `V = SX`, the
doubly-controlled X decomposes as:

```
CCX(c1, c2, t) =
    CSX(c1, t) →        // Λ(V) controlled by c1
    CX(c1, c2) →
    CSXdg(c2, t) →      // Λ(V†) controlled by c2
    CX(c1, c2) →
    CSX(c2, t)           // Λ(V) controlled by c2
```

Proof (trace through all control states):

Basis states below are written as `|c1,c2,t⟩`. The net target action is written
in matrix-product order (last-applied factor on the left):

- c1=1,c2=1: Step 1 applies `SX` to `t`. Step 2 flips `c2` to 0, so Step 3 is
  skipped. Step 4 restores `c2` to 1. Step 5 applies `SX` again. Net target
  action: `SX · SX = X`. ✓
- c1=1,c2=0: Step 1 applies `SX`. Step 2 flips `c2` to 1, so Step 3 applies
  `SXdg`. Step 4 restores `c2` to 0. Step 5 is skipped. Net target action:
  `SXdg · SX = I`. ✓
- c1=0,c2=1: Step 1 is skipped. Steps 2 and 4 do nothing because `c1=0`. Step 3
  applies `SXdg`, and Step 5 applies `SX`. Net target action: `SX · SXdg = I`. ✓
- c1=0,c2=0: No target operation is applied. Net target action: `I`. ✓

CX count: 3 × CSX/CSXdg (2 CX each) + 2 CX = 8 CX total.

**Optimized T-gate decomposition (6 CX):** For implementations that need minimum
CX count, the Toffoli can also be decomposed as:

```
CCX_opt(c1, c2, t) =
    H(t) → CX(c2,t) → Tdg(t) → CX(c1,t) → T(t) → CX(c2,t) →
    Tdg(t) → CX(c1,t) → T(c2) → T(t) → H(t) → CX(c1,c2) →
    T(c1) → Tdg(c2) → CX(c1,c2)
```

Both decompositions produce the same 8×8 matrix. The V-decomposition is the
**canonical** definition (cleaner abstraction); the T-gate version is provided
as an allowed direct-synthesis optimization. The implementation may use either,
but neither may hardcode the 8×8 matrix.

**CCZ — Doubly-Controlled Z (abstraction on CCX)**

```
CCZ(c1, c2, t) = H(t) → CCX(c1, c2, t) → H(t)
```

Proof: `H * X * H = Z`, so controlled-controlled-X with H conjugation =
controlled-controlled-Z. Only |111⟩ picks up −1 phase. ✓

**CSWAP — Fredkin / Controlled-SWAP (abstraction on CCX + CX)**

```
CSWAP(c, t1, t2) = CX(t2, t1) → CCX(c, t1, t2) → CX(t2, t1)
```

Proof: The first CX computes `t1 ⊕ t2` into t1. The CCX flips t2 conditioned on
both the control and the XOR result. The final CX restores t1.

- c=1,t1=0,t2=1: CX→|1,1,1⟩, CCX flips t2→|1,1,0⟩, CX→|1,1,0⟩ ✓ (swapped)
- c=1,t1=1,t2=0: CX→|1,1,0⟩, CCX→|1,1,1⟩, CX→|1,0,1⟩ ✓ (swapped)
- c=0: CCX does nothing → no swap ✓

**RCCX — Relative-Phase CCX (3 CX, directly from Tier 0 + Tier 1)**

A more efficient **relative-phase** variant of CCX. It preserves the classical
Toffoli truth table up to phase under computational-basis measurement, but it is
**not** the exact CCX unitary:

```
RCCX(c1, c2, t) =
    H(t) → T(t) → CX(c2, t) → Tdg(t) → CX(c1, t) →
    T(t) → CX(c2, t) → Tdg(t) → H(t)
```

Uses only 3 CX gates. The resulting 8×8 matrix is not equal to CCX: the active
`|110⟩,|111⟩` subspace is transformed by `[[0,-i],[i,0]]`, and additional
relative phases (for example on `|101⟩`) appear elsewhere. Treat it as a
distinct gate, not as an exact CCX replacement inside larger coherent
subcircuits.

---

#### Tier 5: Four-Qubit and Multi-Controlled Compositions

Exact four-qubit and multi-controlled gates use the recursive Barenco
constructions below. Relative-phase optimized variants may also be given as
direct Tier 0 + Tier 1 syntheses when explicitly marked as such.

**Recursive Barenco Decomposition (general principle):**

For an n-controlled gate `Cⁿ(U)`, find `V` such that `V² = U`. Then:

```
Cⁿ(U)(c1,...,cn, t) =
    C(V)(cn, t) →
    Cⁿ⁻¹(X)(c1,...,cn-1, cn) →
    C(V†)(cn, t) →
    Cⁿ⁻¹(X)(c1,...,cn-1, cn) →
    Cⁿ⁻¹(V)(c1,...,cn-1, t)
```

This reduces an n-controlled-U to two (n-1)-controlled-X plus controlled-V
gates, applied recursively until reaching 1-controlled (CX) gates.

**MCX / C3X — Triple-Controlled X (abstraction on CCX + CSX)**

Apply the Barenco decomposition with V = SX (since SX² = X):

```
MCX(c1, c2, c3, t) =
    CSX(c3, t) →
    CCX(c1, c2, c3) →
    CSXdg(c3, t) →
    CCX(c1, c2, c3) →
    C²SX(c1, c2, t)    // Cⁿ⁻¹(V) = C²(SX), i.e., doubly-controlled SX (NOT CCX)
```

Note: The last term `C²(SX)` (doubly-controlled SX) is itself recursive. To keep
the construction closed under the lower tiers, define the internal single-qubit
helper `W = SX^{1/2} = exp(i*pi/8) * RX(pi/4)`, so `W² = SX`. Then build `C(W)`
with the Tier 2 controlled-U / ABC recipe, and expand `C²(SX)` via the same
Barenco pattern:

```
C²(SX)(c1, c2, t) =
    C(W)(c2, t) → CX(c1, c2) → C(W†)(c2, t) → CX(c1, c2) → C(W)(c1, t)
```

This is a recursive expansion of `C²(SX)`, not a new primitive gate.

If an ancilla qubit initialized to `|0⟩` is available, a lower-CX relative-phase
synthesis can also be used:

```
Ancilla-assisted relative-phase synthesis for MCX(c1, c2, c3, t):
    RCCX(c1, c2, ancilla) → RCCX(ancilla, c3, t) →
    RCCX(c1, c2, ancilla) → RCCX(ancilla, c3, t)
```

This acts on five qubits `(c1, c2, c3, ancilla, t)` and restores the ancilla,
but it is **not** the literal canonical `16×16` matrix definition of
`MCX(c1, c2, c3, t)` above. Treat it as a synthesis strategy for
implementations/transpilers when ancilla and relative phases are acceptable.

**C3SX — Triple-Controlled SX**

Same recursive structure as MCX but with `V² = SX`:

```
C3SX(c1, c2, c3, t) =
    C(V)(c3, t) → CCX(c1, c2, c3) → C(V†)(c3, t) → CCX(c1, c2, c3) → C²(V)(c1, c2, t)
```

Where `V = SX^{1/2} = exp(i*pi/8) * RX(pi/4)` (the fourth root of X), `C(V)` is
built with the Tier 2 controlled-U / ABC recipe, and `C²(V)` is obtained by the
same recursive Barenco construction.

**RCCCX — Relative-Phase C3X (directly from CX + single-qubit)**

A more efficient 4-qubit **relative-phase** variant of C3X. It preserves the
classical multi-controlled-NOT truth table up to phase under computational-basis
measurement, but it is **not** the exact C3X unitary:

```
RCCCX(c1, c2, c3, t) =
    H(t) → T(t) → CX(c3, t) → Tdg(t) → H(t) →
    CX(c1, t) → T(t) → CX(c2, t) → Tdg(t) →
    CX(c1, t) → T(t) → CX(c2, t) → Tdg(t) → H(t) →
    T(t) → CX(c3, t) → Tdg(t) → H(t)
```

Uses 6 CX gates, fewer than the exact C3X. The resulting 16×16 matrix is not
equal to exact C3X: the active `|1110⟩,|1111⟩` subspace is transformed by
`[[0,1],[-1,0]]`, and additional relative phases appear elsewhere.

**Multi-Controlled Gates (variable number of controls):**

| Function                        | Gate | Description        |
| ------------------------------- | ---- | ------------------ |
| `mcxGateN(numControls)`         | MCX  | N-controlled X     |
| `mcpGateN(lambda, numControls)` | MCP  | N-controlled Phase |
| `mcrxGateN(theta, numControls)` | MCRX | N-controlled RX    |
| `mcryGateN(theta, numControls)` | MCRY | N-controlled RY    |
| `mcrzGateN(theta, numControls)` | MCRZ | N-controlled RZ    |

All use the recursive Barenco decomposition:

For `Cⁿ(U)` with N controls:

- N=1: Use the Tier 2 controlled gate (CX, CRX, CRY, CRZ, CP).
- N=2: Use the corresponding exact doubly-controlled `C²(U)` case. For `U = X`
  this is Tier 4 `CCX`; for a general 1-qubit `U`, apply the same Barenco/ABC
  construction specialized to two controls, including exact enabled-subspace
  phase promotion for any extracted global phase.
- N≥3: Apply the recursive scheme:
  `Cⁿ(U) = C(V)(cn,t) · Cⁿ⁻¹(X)(c1..cn-1,cn) · C(V†)(cn,t) · Cⁿ⁻¹(X)(c1..cn-1,cn) · Cⁿ⁻¹(V)(c1..cn-1,t)`
  where V²=U. If `U` or any recursively chosen root `V` carries a global phase
  `exp(i*α)`, promote that phase exactly onto the currently active enabled
  control subspace before continuing the recursion.

For N controls, the resulting matrix is `2^(N+1) × 2^(N+1)`: identity everywhere
except the last two rows/columns where the base gate's 2×2 matrix is applied.
**The matrix is computed by composing the recursive decomposition**, not by
directly constructing the large matrix.

---

#### Tier 6: N-Qubit Composite Gates

**Mølmer-Sørensen Gate**

| Function           | Gate | Description                      |
| ------------------ | ---- | -------------------------------- |
| `msGate(theta, m)` | MS   | Mølmer-Sørensen gate on m qubits |

`MS(theta) = exp(-i * (theta/2) * sum_{j<k} X_j X_k)` — produces a `2^m × 2^m`
matrix.

Since the pairwise `X_j ⊗ X_k` terms commute (they share eigenstates), the
exponential factorizes into a product of pairwise RXX interactions:

```
MS(theta, m) = product over all pairs (j, k) with j < k of RXX(theta, j, k)
```

Each RXX is a Tier 3 gate (built from RZZ, which is built from CX + RZ). This is
a direct application of **abstractions upon abstractions**: MS → RXX → RZZ →
CX + single-qubit.

**Pauli String Gate**

| Function                 | Gate  | Description                      |
| ------------------------ | ----- | -------------------------------- |
| `pauliGate(pauliString)` | Pauli | Tensor product of Pauli matrices |

Given a string like `"XYZ"`, computes `X ⊗ Y ⊗ Z`. The string is read
**left-to-right**: the leftmost character acts on the first qubit in the list
(MSB of the gate matrix index).

This decomposes trivially into independent single-qubit operations — each
character maps to a Tier 0 Pauli gate applied to its respective qubit. No
entanglement (CX) is needed; the matrix is constructed via tensor products of
single-qubit matrices.

---

#### Reference: Expected Matrices

The compositional definitions above produce the following matrices. These are
**derived results** (computed by composing the decompositions), not independent
definitions. Tests verify that the composition yields these exact matrices.

**CX:** `[[1,0,0,0],[0,1,0,0],[0,0,0,1],[0,0,1,0]]`

**CZ:** `diag(1, 1, 1, -1)`

**CY:** `[[1,0,0,0],[0,1,0,0],[0,0,0,-i],[0,0,i,0]]`

**CP(l):** `diag(1, 1, 1, exp(i*l))`

**CRX(th):** Identity in |00⟩,|01⟩; RX(th) in |10⟩,|11⟩ subspace.

**CRY(th):** Identity in |00⟩,|01⟩; RY(th) in |10⟩,|11⟩ subspace.

**CRZ(th):** Identity in |00⟩,|01⟩; RZ(th) in |10⟩,|11⟩ subspace.

**CS:** `diag(1, 1, 1, i)`

**CSdg:** `diag(1, 1, 1, -i)`

**CSX:** Identity in |00⟩,|01⟩; SX in |10⟩,|11⟩ subspace.

**CSXdg:** Identity in |00⟩,|01⟩; SXdg in |10⟩,|11⟩ subspace.

**CH:** Identity in |00⟩,|01⟩; H in |10⟩,|11⟩ subspace.

**CU(th,ph,l,gamma):** Identity in |00⟩,|01⟩; `exp(i*gamma)*U(th,ph,l)` in
|10⟩,|11⟩ subspace.

**CU1(l):** Same as CP(l).

**CU3(th,ph,l):** Identity in |00⟩,|01⟩; U3(th,ph,l) in |10⟩,|11⟩.

**DCX:** `[[1,0,0,0],[0,0,1,0],[0,0,0,1],[0,1,0,0]]`

**SWAP:** `[[1,0,0,0],[0,0,1,0],[0,1,0,0],[0,0,0,1]]`

**iSWAP:** `[[1,0,0,0],[0,0,i,0],[0,i,0,0],[0,0,0,1]]`

**ECR:** `(1/sqrt(2))*[[0,0,1,i],[0,0,i,1],[1,-i,0,0],[-i,1,0,0]]`

**RZZ(th):** `diag(exp(-i*th/2), exp(i*th/2), exp(i*th/2), exp(-i*th/2))`

**RXX(th):** `[[cos(th/2),0,0,-i*sin(th/2)],[0,cos(th/2),-i*sin(th/2),0],`
`[0,-i*sin(th/2),cos(th/2),0],[-i*sin(th/2),0,0,cos(th/2)]]`

**RYY(th):** `[[cos(th/2),0,0,i*sin(th/2)],[0,cos(th/2),-i*sin(th/2),0],`
`[0,-i*sin(th/2),cos(th/2),0],[i*sin(th/2),0,0,cos(th/2)]]`

**RZX(th):** `[[cos(th/2),-i*sin(th/2),0,0],[-i*sin(th/2),cos(th/2),0,0],`
`[0,0,cos(th/2),i*sin(th/2)],[0,0,i*sin(th/2),cos(th/2)]]`

**XX+YY(th,beta):** `[[1,0,0,0],[0,cos(th/2),-i*sin(th/2)*exp(-i*beta),0],`
`[0,-i*sin(th/2)*exp(i*beta),cos(th/2),0],[0,0,0,1]]`

**XX-YY(th,beta):** `[[cos(th/2),0,0,-i*sin(th/2)*exp(-i*beta)],`
`[0,1,0,0],[0,0,1,0],` `[-i*sin(th/2)*exp(i*beta),0,0,cos(th/2)]]`

**CCX (8×8):** Identity except `[6][7]=1, [7][6]=1, [6][6]=0, [7][7]=0`.

**CCZ (8×8):** Identity except `[7][7]=-1`.

**CSWAP (8×8):** Identity except rows 5,6 swapped.

**RCCX (8×8):** Identity except `[5][5]=-1`, `[6][6]=0, [6][7]=-i`,
`[7][7]=0, [7][6]=i`.

**MCX (16×16):** Identity except rows/cols 14,15 where X is applied.

**C3SX (16×16):** Identity except rows/cols 14,15 where SX is applied.

**RCCCX (16×16):** Identity except `[12][12]=i`, `[13][13]=-i`,
`[14][14]=0, [14][15]=1`, `[15][15]=0, [15][14]=-1`.

---

#### Abstraction Dependency Graph

```
Tier 6: MS ──────────→ RXX (pairwise)
        Pauli ────────→ Tier 0 (tensor products)

Tier 5: MCX/C3X ─────→ CCX + CSX + recursive C²SX
        C3SX ─────────→ CCX + controlled SX^(1/2) (recursive)
        RCCCX ────────→ CX + single-qubit (direct, 6 CX)
        mcxGateN ─────→ recursive Barenco → Tier 4 → Tier 2 → CX
        mcpGateN ─────→ recursive Barenco → CP → CX
        mcrxGateN ────→ recursive Barenco → CRX → CRZ → CX
        mcryGateN ────→ recursive Barenco → CRY → CX
        mcrzGateN ────→ recursive Barenco → CRZ → CX

Tier 4: CCX ──────────→ CSX + CX (V-decomposition)
        CCZ ──────────→ CCX + H
        CSWAP ────────→ CCX + CX
        RCCX ─────────→ CX + T/Tdg/H (direct, 3 CX)

Tier 3: SWAP ─────────→ CX × 3
        RZZ ──────────→ CX + RZ
        RXX ──────────→ RZZ + H (basis change)
        RYY ──────────→ RZZ + RX (basis change)
        RZX ──────────→ RZZ + H (partial basis change)
        ECR ──────────→ RZX + X
        iSWAP ────────→ SWAP + CZ + S
        XX+YY ────────→ CX + CRX
        XX-YY ────────→ CX + CRX + X

Tier 2: CZ ───────────→ CX + H
        CY ───────────→ CX + S + Sdg
        CP ───────────→ CX × 2 + P
        CRZ ──────────→ CX × 2 + RZ
        CRY ──────────→ CX × 2 + RY
        CRX ──────────→ CRZ + H (basis change)
        CS ───────────→ CP(π/2)
        CSdg ─────────→ CP(-π/2)
        CSX ──────────→ CRX(π/2) + P
        CSXdg ────────→ CRX(-π/2) + P
        CH ───────────→ ABC decomposition (2 CX)
        CU ───────────→ ABC decomposition (2 CX)
        CU1 ──────────→ CP
        CU3 ──────────→ CU(γ=0)
        DCX ──────────→ CX × 2

Tier 1: CX (primitive) ← the only hardcoded multi-qubit matrix

Tier 0: I, H, X, Y, Z, P, R, RX, RY, RZ, S, Sdg, SX, SXdg, T, Tdg,
        U, U1, U2, U3, RV, GlobalPhase ← hardcoded 2×2 / 1×1 matrices
```

---

#### Verification Requirements for Gates

Every gate matrix must satisfy:

1. **Unitarity:** `G† * G ≈ I` (within epsilon).
2. **Correct dimensions.**
3. **Compositional correctness:** The matrix produced by composing the
   decomposition equals the expected reference matrix (within epsilon).
4. **Known eigenvalue/eigenvector relationships** where applicable.
5. **Tier 0 equivalences:** `T = P(pi/4)`, `S = P(pi/2)`, `Z = P(pi)`,
   `X = U(pi, 0, pi)`, `H = U(pi/2, 0, pi)`, `SX*SX = X`, `SX*SXdg = I`,
   `U1(l) = P(l)`, `U3(th,ph,l) = U(th,ph,l)`, `CU1(l) = CP(l)`.
6. **Compositional equivalences:**
   - `CZ = (I⊗H) · CX · (I⊗H)`
   - `CY = (I⊗S) · CX · (I⊗Sdg)`
   - `SWAP = CX(a,b) · CX(b,a) · CX(a,b)`
   - `CCX(c1,c2,t) = CSX(c2,t) · CX(c1,c2) · CSXdg(c2,t) · CX(c1,c2) · CSX(c1,t)`
     (matrix-product order; the circuit-time-order version appears above)
   - `CCZ = (I⊗I⊗H) · CCX · (I⊗I⊗H)`
   - `RXX(th) = (H⊗H) · RZZ(th) · (H⊗H)`

**Tests (minimum 80):**

Unless a test bullet explicitly uses `→`, algebraic identities in this section
use matrix-product order (`·`). When both forms are shown, the `·` expression
and the `→` sequence refer to the same decomposition under the notation
convention above.

- Every gate function is called and verified unitary.
- Every gate's matrix elements are checked against the reference matrices.
- **Compositional verification:** For each multi-qubit gate, verify that the
  matrix produced by composing the decomposition matches the reference matrix.
- Phase gate hierarchy: `tGate() ~ pGate(pi/4)`, `sGate() ~ pGate(pi/2)`,
  `pauliZ() ~ pGate(pi)`.
- `uGate(pi, 0, pi) ~ pauliX()`.
- `uGate(pi/2, 0, pi) ~ hadamard()`.
- `u1Gate(l) ~ pGate(l)` for several values.
- `u3Gate(th,ph,l) ~ uGate(th,ph,l)` for several values.
- `rxGate(pi)` exactly equals `(-i) * X`; verify by exact matrix or exact
  state-vector comparison, not only by measurement statistics.
- `ryGate(pi)` applied to |0> gives |1>.
- `sxGate() * sxGate() ~ pauliX()`.
- `sxGate() * sxdgGate() ~ identityGate()`.
- `sdgGate() ~ sGate().dagger()`.
- `tdgGate() ~ tGate().dagger()`.
- `cu1Gate(l) ~ cpGate(l)` for several values.
- **CZ composition:** verify `(I⊗H) · CX · (I⊗H)`; equivalently, circuit-time
  `H(t) → CX → H(t)` produces the CZ matrix.
- **CY composition:** verify `(I⊗S) · CX · (I⊗Sdg)`; equivalently, circuit-time
  `Sdg(t) → CX → S(t)` produces the CY matrix.
- **CP composition:** verify 2-CX decomposition matches `diag(1,1,1,exp(il))`.
- **CRZ composition:** verify decomposition matches reference for several theta.
- **CRY composition:** verify decomposition matches reference for several theta.
- **CRX composition:** verify `(I⊗H) · CRZ(th) · (I⊗H)` matches the CRX
  reference.
- **CS as CP(pi/2):** verify equivalence.
- **CSX composition:** verify `(P(pi/4) on control) · CRX(pi/2)`; equivalently,
  circuit-time `P(pi/4)(c) → CRX(pi/2)` matches the reference.
- CX: `CX|00>=|00>`, `CX|01>=|01>`, `CX|10>=|11>`, `CX|11>=|10>`.
- CY: verify on all 4 basis states.
- CZ: verify on all 4 basis states.
- CH: verify ABC decomposition produces correct matrix and on all 4 basis
  states.
- DCX: verify on all 4 basis states.
- ECR: verify matrix-product order `X(a) · RZX(pi/2, a, b)`; equivalently,
  circuit-time `RZX(pi/2, a, b) → X(a)` produces the ECR matrix. Verify
  unitarity.
- **SWAP composition:** verify 3-CX decomposition: `SWAP|01>=|10>`,
  `SWAP|10>=|01>`.
- **iSWAP composition:** verify matrix-product order `(S⊗S) · SWAP · CZ`;
  equivalently, circuit-time `CZ → SWAP → S⊗S` produces the iSWAP matrix.
  `iSWAP|01>=i|10>`, `iSWAP|10>=i|01>`.
- **RZZ composition:** verify `CX · RZ(th) · CX` matches reference for several
  theta.
- **RXX composition:** verify `H⊗H · RZZ(th) · H⊗H` matches reference.
- **RYY composition:** verify the documented circuit-time sequence
  `RX(pi/2)⊗RX(pi/2) → RZZ → RX(-pi/2)⊗RX(-pi/2)` matches the reference.
- **RZX composition:** verify `H(b) · RZZ · H(b)` matches reference.
- CP(l): verify on all 4 basis states for several lambda values.
- CRX, CRY, CRZ: verify on basis states and unitarity.
- CS, CSdg, CSX, CSXdg: verify on basis states.
- CU: verify ABC decomposition on basis states for specific parameters.
- **CCX V-decomposition:** verify
  `CSX(c1,t) → CX(c1,c2) → CSXdg(c2,t) → CX(c1,c2) → CSX(c2,t)` produces the
  Toffoli matrix. `CCX|110>=|111>`, `CCX|111>=|110>`, all others unchanged.
- **CCZ composition:** verify `H · CCX · H` produces CCZ matrix. Only `CCZ|111>`
  picks up -1 phase.
- **CSWAP composition:** verify `CX(t2,t1) → CCX(c,t1,t2) → CX(t2,t1)` produces
  the Fredkin matrix. `CSWAP|1,01>=|1,10>`, `CSWAP|1,10>=|1,01>`, control=0 no
  change.
- RCCX: is unitary, correct dimensions (8×8), verify specific entries. Verify
  3-CX decomposition produces correct matrix.
- RXX(0) ~ I4, RZZ(0) ~ I4, RYY(0) ~ I4, RZX(0) ~ I4.
- RXX, RYY, RZZ, RZX: unitary for several values of theta.
- **XX+YY composition:** verify the documented circuit-time pattern
  `RZ(beta) → CX → CRX(swapped control/target) → CX → RZ(-beta)` produces the
  correct matrix for several theta, beta. Verify on basis states.
- **XX-YY composition:** verify the documented circuit-time pattern
  `RZ(-beta) → X → CX → CRX(swapped control/target) → CX → X → RZ(beta)`
  produces the correct matrix for several theta, beta. Verify on basis states.
- MCX/C3X: verify unitarity, correct dimensions (16×16), flip on |1110>. Verify
  recursive decomposition produces correct matrix.
- C3SX: verify unitarity, dimensions, apply SX on |1110>.
- RCCCX: verify unitarity, correct dimensions (16×16), and reference-matrix
  entries/relative phases on the `|1110⟩,|1111⟩` subspace.
- Multi-controlled gates with variable controls: verify for 1, 2, 3 controls.
  Verify recursive Barenco decomposition produces correct matrices.
- **MS composition:** verify `product of RXX` produces MS matrix for 2 and 3
  qubits. Verify unitarity.
- Pauli string: `"X"` = pauliX, `"XY"` = X ⊗ Y, verify dimensions and elements.
  Verify tensor product construction.
- RV gate: `rvGate(0,0,0) = I` exactly, `rvGate(pi,0,0) = (-i) * X` exactly,
  unitarity.
- Action of every single-qubit gate on |0> and |1> verified against expected
  output.
- **Tier chain verification:** Pick at least 3 gates from each tier and trace
  the full decomposition down to CX + single-qubit, verifying that the final
  composed matrix matches at every intermediate tier.

---

### Step 5: Symbolic Parameters

**File:** `src/parameter.{ext}`

Implement a symbolic parameter system that allows circuit angles to be specified
as either numeric literals or named parameters (possibly combined in
expressions).

#### Param class

- `Param(name: string)` — a named symbolic parameter.
- Supports arithmetic: `Param + Param`, `Param * number`, `Param / number`,
  `number * Param`, `Param + number`, `Param - Param`, etc.
- `bind(params: map<string, number>) -> number | Param` — substitutes known
  values. If all symbols are resolved, returns a number. Otherwise returns a
  partially-bound expression.
- `isResolved() -> boolean` — true if the expression is a pure number.

#### Expression types

Support at minimum: `Add`, `Sub`, `Mul`, `Div`, `Neg`, `Literal(number)`,
`Symbol(name)`.

The expression system must handle arbitrarily nested expressions like
`"2 * theta + phi / 3"`.

Implementations may extend this expression tree with additional nodes such as
function calls, named constants, casts, indexing/slicing, or other
OpenQASM-3-specific forms as needed for full serializer/deserializer coverage.
Bare `gphase(...)` and gate-angle expressions parsed from OpenQASM 3 must
preserve that richer structure exactly until binding or evaluation.

The same exact expression type is used for gate angles, scope `globalPhase`, and
instruction-level `gphase(...)` expressions. Scope `globalPhase` stores only
hoistable expressions as defined in Section 2.6; non-hoistable parsed
`gphase(...)` keeps the same exact expression tree but remains an ordinary
zero-qubit instruction. Serialization, binding, composition, and inversion must
preserve these expressions symbolically; only operations that need a concrete
complex scalar may require full binding first.

**Tests (minimum 15):**

- Create a Param, verify name.
- `Param("theta").bind({"theta": 3.14})` returns 3.14.
- `Param("theta") * 2` bound with theta=1.5 gives 3.0.
- `Param("a") + Param("b")` bound with a=1, b=2 gives 3.
- Partial binding: `Param("a") + Param("b")` bound with only a=1 is still
  symbolic.
- `Param("x") / 2` bound gives correct result.
- `Param("x") - Param("y")` for various values.
- Nested: `(Param("x") + 1) * 2` bound with x=3 gives 8.
- `isResolved()` returns false for unbound, true for bound.
- Expression with no symbols is always resolved.
- Verify that numeric literal angles pass through unchanged.

---

### Step 6: Circuit Builder

**File:** `src/circuit.{ext}`

Implement the `QuantumCircuit` class.

#### Constructor

```
QuantumCircuit(globalPhase = 0)
```

`globalPhase` is an exact phase expression (`AngleExpr`), defaulting to the
numeric zero.

Qubits are allocated **implicitly**: when a gate references qubit index N, all
qubits 0..N are automatically allocated if they do not yet exist.

Classical memory is modeled as an ordered list of named `ClassicalRegister`
definitions plus a derived flat-index view. The circuit must preserve register
names, sizes, and declaration order end to end. If the user references only flat
classical-bit indices and never declares a named classical register, materialize
those bits into one default register (for example `c`) so the register structure
still exists for serialization and backend result parsing.

#### Gate Methods — ALL must be implemented

Every gate method except a bare `globalPhaseGate` appends one Instruction to the
circuit and returns `this` (for chaining). `globalPhaseGate` increments the
circuit's scalar `globalPhase` and also returns `this`. Angle and phase
parameters accept `AngleExpr`.

**Global Phase:**

- `qc.globalPhaseGate(theta)`

A bare `qc.globalPhaseGate(theta)` normalizes directly into the circuit's
`globalPhase` expression rather than being stored as an ordinary instruction.
This builder API denotes hoistable scope-global phase. Deserialized OpenQASM
bare `gphase(theta)` that is not hoistable under Section 2.6 must instead remain
an ordinary zero-qubit instruction. A control-bearing form such as
`ctrl @ gphase(theta)` is not scope-global and must be represented as an
instruction (or desugared to the exact enabled-subspace phase operator) rather
than absorbed into `globalPhase`.

**Single-Qubit Gates:**

- `qc.id(qubit)`
- `qc.h(qubit)`
- `qc.x(qubit)`
- `qc.y(qubit)`
- `qc.z(qubit)`
- `qc.p(lambda, qubit)`
- `qc.r(theta, phi, qubit)`
- `qc.rx(theta, qubit)`
- `qc.ry(theta, qubit)`
- `qc.rz(theta, qubit)`
- `qc.s(qubit)`
- `qc.sdg(qubit)`
- `qc.sx(qubit)`
- `qc.sxdg(qubit)`
- `qc.t(qubit)`
- `qc.tdg(qubit)`
- `qc.u(theta, phi, lambda, qubit)`
- `qc.u1(lambda, qubit)`
- `qc.u2(phi, lambda, qubit)`
- `qc.u3(theta, phi, lambda, qubit)`
- `qc.rv(vx, vy, vz, qubit)`

**Two-Qubit Gates:**

- `qc.ch(control, target)`
- `qc.cx(control, target)`
- `qc.cy(control, target)`
- `qc.cz(control, target)`
- `qc.dcx(qubit0, qubit1)`
- `qc.ecr(qubit0, qubit1)`
- `qc.swap(qubit0, qubit1)`
- `qc.iswap(qubit0, qubit1)`
- `qc.cp(lambda, control, target)`
- `qc.crx(theta, control, target)`
- `qc.cry(theta, control, target)`
- `qc.crz(theta, control, target)`
- `qc.cs(control, target)`
- `qc.csdg(control, target)`
- `qc.csx(control, target)`
- `qc.csxdg(control, target)`
- `qc.cu(theta, phi, lambda, gamma, control, target)`
- `qc.cu1(lambda, control, target)`
- `qc.cu3(theta, phi, lambda, control, target)`
- `qc.rxx(theta, qubit0, qubit1)`
- `qc.ryy(theta, qubit0, qubit1)`
- `qc.rzz(theta, qubit0, qubit1)`
- `qc.rzx(theta, qubit0, qubit1)`
- `qc.xxMinusYY(theta, beta, qubit0, qubit1)`
- `qc.xxPlusYY(theta, beta, qubit0, qubit1)`

**Three-Qubit Gates:**

- `qc.ccx(control1, control2, target)`
- `qc.ccz(control1, control2, target)`
- `qc.cswap(control, target1, target2)`
- `qc.rccx(control1, control2, target)`

**Four-Qubit Gates:**

- `qc.mcx(control1, control2, control3, target)` (3-control version)
- `qc.c3sx(control1, control2, control3, target)`
- `qc.rcccx(control1, control2, control3, target)`

**Multi-Controlled Gates (variable controls):**

- `qc.mcxN(controlQubits, target)`
- `qc.mcp(lambda, controlQubits, target)`
- `qc.mcrx(theta, controlQubits, target)`
- `qc.mcry(theta, controlQubits, target)`
- `qc.mcrz(theta, controlQubits, target)`

**Special Gates:**

- `qc.ms(theta, qubits)` — Molmer-Sorensen
- `qc.pauli(pauliString, qubits)` — Pauli string gate
- `qc.unitary(matrix, qubits)` — arbitrary unitary matrix

**Gate Modifiers (OpenQASM 3):**

- `qc.ctrl(numControls, gate, controlQubits, targetQubits)` — apply `ctrl @`
  modifier. Creates a controlled version of any gate. `numControls` defaults
  to 1. The modifier is stored on the instruction and expanded during
  transpilation or simulation.
- `qc.negctrl(numControls, gate, controlQubits, targetQubits)` — apply
  `negctrl @` modifier. Conditions on control qubits being |0> instead of |1>.
- `qc.inv(gate, qubits)` — apply `inv @` modifier. Replaces gate U with U†.
- `qc.pow(k, gate, qubits)` — apply `pow(k) @` modifier. Applies gate to the kth
  power. Positive integer k repeats; negative k repeats inverse.

Gate modifiers can be chained: `ctrl @ inv @ U` means controlled-inverse-U.
Multiple modifiers are stored in order on the instruction's `modifiers` field
and applied from right to left (innermost first) during simulation.

**Custom Gate Definitions (OpenQASM 3):**

- `qc.defineGate(name, params, qubits, body)` — define a custom gate from a
  `QuantumCircuit` body. Parameters are angle-typed. This is the programmatic
  equivalent of the OpenQASM 3 `gate` statement and stores a `GateDefinition`.
  Defined gates can be used via `qc.append(name, ...)`.

**State Preparation:**

- `qc.prepareState(state, qubits?)` — prepare qubits in a given state
  (amplitudes, integer, or bitstring). Qubits must already be |0>.
- `qc.initialize(state, qubits?)` — reset qubits to |0> then prepare state.

**Non-Unitary Operations:**

- `qc.addClassicalRegister(name, size)` — append a named classical register and
  return its metadata/reference.
- `qc.measure(qubit, clbit)` — measure into a classical bit identified either by
  flat index or by `{ register, index }` in the target language's idiomatic
  representation. Stored instructions must retain both the flat index and the
  named-register reference.
- `qc.reset(qubit)`
- `qc.barrier(...qubits)` — optimization fence, no state effect
- `qc.delay(duration, qubit, unit)` — no-op in simulation

**Classical Variable Declarations (OpenQASM 3):**

These methods declare classical typed variables for OpenQASM 3 serialization and
classical expression support. In simulation, `bit`/`bit[n]` map to the existing
classical register system; other types are tracked as metadata and participate
in classical expressions and control flow conditions.

- `qc.declareClassicalVar(name, type, size?, initValue?)` — declare a classical
  variable. `type` is one of the `ClassicalType` values. `size` is the bit width
  (required for `int`, `uint`, `float`, `angle`; null for `bool`, `duration`,
  `stretch`). Optional initial value.
- `qc.declareConst(name, type, size?, value)` — declare a compile-time constant.
  Equivalent to `const type[size] name = value;` in OpenQASM 3.
- `qc.declareInput(name, type, size?)` — declare an input variable. Equivalent
  to `input type[size] name;` in OpenQASM 3. Input values are provided at
  execution time (analogous to symbolic `Param` but using the OpenQASM 3 input
  mechanism).
- `qc.declareOutput(name, type, size?)` — declare an output variable. Equivalent
  to `output type[size] name;`. Specifies which classical variables constitute
  the program output.
- `qc.declareArray(name, baseType, dimensions, initValue?)` — declare a
  classical array. Equivalent to `array[baseType, d1, d2, ...] name;`. Up to 7
  dimensions. Base types: `int`, `uint`, `float`, `complex`, `angle`, `bool`,
  `duration`.

**Classical Assignment and Expressions (OpenQASM 3):**

- `qc.classicalAssign(target, expression)` — assign a value or expression result
  to a classical variable. Supports compound assignment operators (`+=`, `-=`,
  `*=`, `/=`, `%=`, `**=`, `&=`, `|=`, `^=`, `<<=`, `>>=`).

**Subroutines and Externs (OpenQASM 3):**

- `qc.defineSubroutine(name, params, returnType, body)` — define a subroutine
  (`def` in OpenQASM 3). `body` is a `QuantumCircuit`. Parameters can be
  classical types or qubit references. Return type is a classical type or null
  for void.
- `qc.declareExtern(name, params, returnType)` — declare an external function
  (`extern` in OpenQASM 3). The implementation is provided by the runtime
  environment.
- `qc.callSubroutine(name, args, resultVar?)` — call a previously defined
  subroutine or extern. `resultVar` is the optional variable to assign the
  return value to.

**Pragma and Annotations (OpenQASM 3):**

- `qc.pragma(content)` — add a pragma directive. Stored as-is and serialized as
  `pragma <content>;`.
- `qc.annotate(content)` — attach an annotation to the next instruction.
  Serialized as `@<content>` preceding the annotated statement.

**Control Flow:**

- `qc.ifTest(condition, trueBody, falseBody?)` — conditional execution.
  `condition` is `{register, value}`, where `register` may be a flat classical
  bit selection or a named classical register selection. Bodies are
  `QuantumCircuit` instances.
- `qc.forLoop(indexSet, loopParam, body)` — iterate over integer values.
- `qc.whileLoop(condition, body)` — repeat while condition is true.
- `qc.switch(target, cases)` — multi-way branch with `{value, body}` pairs and
  optional DEFAULT.
- `qc.breakLoop()` — break from enclosing loop.
- `qc.continueLoop()` — continue to next iteration.
- `qc.box(body)` — atomic scoped block (optimization barrier).

**Circuit Composition:**

- `qc.compose(other, qubitMapping, clbitMapping)` — append another circuit's
  instructions into this one, mapped by indices. Adds `other`'s globalPhase.
- `qc.toGate(label?)` — convert to reusable gate (unitary only), preserving its
  exact globalPhase in the produced reusable definition/metadata.
- `qc.toInstruction(label?)` — convert to reusable instruction, preserving the
  exact globalPhase of the wrapped body.
- `qc.append(operation, qubitIndices, clbitIndices)` — low-level append.
- `qc.inverse()` — return new circuit with reversed, daggered gates. Negated
  global phase. Only unitary circuits.

Low-level append, compose, inversion, `toGate`, `toInstruction`, and parameter
binding must preserve the ordered `ClassicalRegister[]` metadata, every
instruction's named classical bit references, and every scope's exact
`globalPhase` expression. Do **not** collapse multiple named registers into one
anonymous bit array during any circuit transformation.

**Informational:**

- `qc.complexity()` — returns `CircuitComplexity` (see types).

**Inspection:**

- `qc.blochSphere(qubitIndex)` — returns `BlochCoordinates`. Simulates the
  circuit via state-vector evolution, computes reduced density matrix.

**Parameter Binding:**

- `qc.run(parameters)` — returns a new circuit with symbolic parameters replaced
  by values from the map. Unmatched parameters remain symbolic. Original circuit
  is not modified.

**Transpilation:**

- `qc.transpile(backend, shots?)` — calls
  `backend.transpileAndPackage(this, shots)` and returns the opaque executable.
  `shots` is a per-execution input used when the backend packages a submitted
  job. This is the entry point for compiling a circuit for a specific backend.

**Tests (minimum 55):**

- Build a circuit with every single gate type and verify instruction count.
- Verify chaining: `qc.h(0).cx(0,1).measure(0,0)`.
- Verify implicit qubit allocation: using qubit 5 allocates qubits 0-5.
- Verify implicit classical bit allocation materializes a default named
  register.
- Verify named classical register declaration and insertion order are preserved.
- Verify measurement into a named register bit stores both flat and named
  classical-bit references.
- Verify `run()` with symbolic parameters replaces correctly.
- Verify `run()` partial binding leaves unbound params symbolic.
- Verify `complexity()` returns correct depth, operation counts.
- Verify `inverse()` reverses and daggers gates.
- Verify `compose()` maps qubits and clbits correctly and preserves/remaps
  classical-register structure in order.
- Verify `toGate()` and `toInstruction()` preserve exact `globalPhase`.
- Verify control flow instructions are stored correctly, with nested
  `QuantumCircuit` bodies preserving branch-local `globalPhase`.
- Verify `barrier()` and `delay()` are stored but don't affect state.
- Verify `globalPhaseGate()` accumulates on the circuit's scalar `globalPhase`,
  is not stored as an ordinary qubit instruction in the normalized
  representation, and survives compose/inverse/serialization.
- Verify symbolic `globalPhase` expressions survive bind/compose/inverse without
  numeric coercion.
- Verify deserialized hoistable bare `gphase` statements normalize into the
  scope `globalPhase` only when safe to hoist.
- Verify deserialized declaration-dependent, mutable-state-dependent, or
  annotated bare `gphase` statements remain ordinary zero-qubit instructions in
  source order rather than being folded into the scope `globalPhase`.
- Verify `inv @ gphase(...)` and `pow(k) @ gphase(...)` normalize first and are
  folded into the scope `globalPhase` only if the resulting bare phase is
  hoistable.
- Verify control-bearing `gphase` operations, and any `gphase` still non-bare or
  non-hoistable after modifier expansion, are stored as ordinary instructions or
  desugared relative-phase operations, not folded into the scope `globalPhase`.
- Edge cases: circuit with 0 instructions, only measurements, only resets.
- Qubit index validation for multi-qubit gates (no duplicate qubits).
- Verify Pauli string gate with various strings.
- Verify `unitary()` stores the matrix.
- Verify `prepareState()` and `initialize()` instructions.
- Verify `ms()` stores theta and qubit list.
- Verify `ctrl()` modifier: apply `ctrl @ h` stores instruction with ctrl
  modifier and correct control/target qubits.
- Verify `negctrl()` modifier: stores negctrl modifier on instruction.
- Verify `inv()` modifier: stores inv modifier on instruction.
- Verify `pow(k)` modifier: stores pow modifier with exponent.
- Verify chained modifiers: `ctrl @ inv @ U` stores modifiers in order.
- Verify `defineGate()` stores a `GateDefinition` with name, params, body.
- Verify defined gates can be appended via `append(gateName, ...)`.
- Verify `declareClassicalVar()` for each type: bool, int[32], uint[8],
  float[64], angle[16], duration, stretch.
- Verify `declareConst()` stores const flag and initial value.
- Verify `declareInput()` stores input flag.
- Verify `declareOutput()` stores output flag.
- Verify `declareArray()` stores base type and dimensions.
- Verify `classicalAssign()` stores assignment with target and expression.
- Verify `defineSubroutine()` stores subroutine with params and body.
- Verify `declareExtern()` stores extern declaration.
- Verify `callSubroutine()` stores call instruction with args.
- Verify `pragma()` stores pragma content.
- Verify `annotate()` attaches annotation to next instruction.

---

### Step 7: Backend Interface

**File:** `src/backend.{ext}`

Define the Backend interface/trait/protocol:

```
interface Backend {
  numQubits: number
  basisGates: string[]
  couplingMap: [number, number][] | null

  transpileAndPackage(circuit: QuantumCircuit, shots?: number) -> Executable
  execute(executable: Executable) -> ExecutionResult
}
```

Where `Executable` is an opaque, backend-specific object.

`shots` is a per-execution input. Backends that submit remote jobs must carry it
in the packaged executable/job payload rather than in `BackendConfiguration`.

This file defines only the interface. Concrete implementations are in separate
files.

**Tests (minimum 5):**

- Verify interface contract (all required properties/methods exist).
- Verify that a mock/stub backend can be created implementing the interface.

---

### Step 8: Simulator Engine

**File:** `src/simulator.{ext}`

Implement `SimulatorBackend` that implements the `Backend` interface.

#### Constructor

```
SimulatorBackend(numShots = 1024)
```

#### Properties

- `numQubits`: Large number (e.g., 30), limited by memory.
- `basisGates`: All gates supported.
- `couplingMap`: `null` (all-to-all).

#### `transpileAndPackage(circuit, shots?) -> SimulatorExecutable`

Since the simulator supports all gates and has no connectivity constraints, this
simply validates the circuit and wraps it:

```
SimulatorExecutable {
  circuit: QuantumCircuit
  numShots: shots ?? this.numShots
}
```

#### `execute(executable) -> ExecutionResult`

This is the core simulation engine.

**State-vector simulation algorithm:**

1. Initialize state vector to `|0...0> = [1, 0, 0, ..., 0]` (length
   `2^numQubits`).
2. Initialize classical memory to all zeros using the circuit's ordered named
   classical registers. Maintain both the ordered register layout and the
   derived flat classical-bit array (length `numClbits`).
3. Record/defer the circuit's accumulated global phase; do not apply it during
   gate evolution because it is unobservable until an exact state vector or
   exact matrix/state comparison is needed.
4. For each instruction in order: a. If it's a control flow operation
   (`if_test`, `for_loop`, `while_loop`, `switch`), handle accordingly (see
   below). b. Otherwise apply the gate/operation.
5. After all instructions, apply the deferred global phase to the final state
   vector (multiply all amplitudes by `exp(i * globalPhase)`) if an exact state
   vector or exact state/unitary comparison is requested. It may remain deferred
   for pure measurement sampling because it does not affect probabilities.
6. After all instructions, sample measurement outcomes.

**Gate Application — Subspace Iteration (MANDATORY):**

**Do NOT construct full 2^n x 2^n matrices.** Apply gates directly to the state
vector using the subspace iteration technique:

**Single-qubit gate U = [[u00,u01],[u10,u11]] on qubit k:**

```
step = 2^k
for group_start in range(0, 2^n, 2 * step):
  for j in range(group_start, group_start + step):
    idx0 = j
    idx1 = j + step
    a = state[idx0]
    b = state[idx1]
    state[idx0] = u00 * a + u01 * b
    state[idx1] = u10 * a + u11 * b
```

**Two-qubit gate U (4x4) on qubits k0, k1:** For each group of 4 indices
differing only in bits k0 and k1:

```
idx00 = base_index                         // k0=0, k1=0
idx01 = base_index | (1 << k1)             // k0=0, k1=1
idx10 = base_index | (1 << k0)             // k0=1, k1=0
idx11 = base_index | (1 << k0) | (1 << k1) // k0=1, k1=1
v = [state[idx00], state[idx01], state[idx10], state[idx11]]
v_new = U * v   // 4x4 times 4x1
write v_new back
```

**N-qubit gate (general):** Group state-vector entries into blocks of `2^m`
entries differing only in the m target qubit bits. Apply the `2^m x 2^m` matrix
to each block.

**Measurement (Born Rule):**

```
P(qubit k = 0) = sum of |state[j]|^2 for all j where bit k is 0
outcome = random() < P(0) ? 0 : 1
Collapse: zero out amplitudes inconsistent with outcome, renormalize
Store outcome in classical bit.
```

**Reset:**

1. Measure qubit k (internally, without recording to classical register).
2. If result is 1, apply X gate on qubit k to flip it to |0>.

**Global Phase:** After all gates, multiply every amplitude by
`exp(i * totalGlobalPhase)`. (This does not affect measurement probabilities but
is needed for `getStateVector` correctness.)

**Control Flow:**

- `if_test`: Evaluate the classical condition (compare classical register value
  against the condition value). Conditions may reference flat classical bits or
  named classical registers, but the integer comparison must use the preserved
  register ordering. If true, simulate `trueBody` on current state. If false and
  `falseBody` exists, simulate it.
- `for_loop`: For each value in `indexSet`, bind the loop parameter (if any),
  simulate the body. Handle `breakLoop` and `continueLoop`.
- `while_loop`: Evaluate condition, simulate body, repeat. Handle
  break/continue.
- `switch`: Evaluate target's classical value, find matching case, simulate
  body.
- `breakLoop` / `continueLoop`: Signal the enclosing loop.
- `box`: Execute the body unconditionally.

**Barrier and Delay:** No effect on state vector. Skip.

**Sampling (producing the final result):**

For circuits with measurements only at the end (no mid-circuit measurements):

1. Simulate once to get the pre-measurement state vector.
2. Compute probabilities: `prob[j] = |state[j]|^2`.
3. Sample `numShots` times from this distribution.

For mid-circuit measurements: Run the full simulation `numShots` times
independently.

**Return value:** Dictionary mapping bitstrings to percentages (0-100).
Bitstrings are built from the classical memory, not directly from qubit order:
concatenate each classical register's bitstring in declared register order, and
within each register treat local bit 0 as the rightmost bit. Sum of percentages
= 100. For example, a Bell state result would be `{ "00": 50,
"11": 50 }`.
Percentages are computed from shot counts:
`percentage = (count / numShots) * 100`.

#### Additional Public Methods

- `getStateVector(circuit, params?) -> Complex[]` — Run the circuit without
  measurement collapse (skip measure and reset). Returns the raw state vector.
  Useful for testing and introspection.

**Tests (minimum 60):**

- Simulate `X|0>` -> measure -> `{ "1": 100 }`.
- Simulate `H|0>` -> measure -> ~`{ "0": 50, "1": 50 }` (within tolerance over
  4096 shots).
- Bell state `H(0), CX(0,1)` -> ~`{ "00": 50, "11": 50 }`.
- GHZ state `H(0), CX(0,1), CX(0,2)` -> ~`{ "000": 50, "111": 50 }`.
- `getStateVector` for `H|0>` ~ `[1/sqrt(2), 1/sqrt(2)]`.
- `getStateVector` for Bell state ~ `[1/sqrt(2), 0, 0, 1/sqrt(2)]`.
- Every single-qubit gate applied to |0>: verify `getStateVector` matches gate's
  first column.
- Every single-qubit gate applied to |1>: verify `getStateVector` matches gate's
  second column.
- Parameterized circuit: `RX(theta)` with `theta = pi` on |0> should give the
  exact state vector `[0, -i]`.
- Y gate on |0>: verify state vector is [0, i].
- RY(pi/2) on |0>: verify equal superposition.
- Classical conditioning via `if_test`: measure qubit 0, conditionally apply X
  to qubit 1.
- `for_loop`: repeated RX rotations with loop parameter.
- `while_loop`: measure until |1>.
- `switch`: branch on classical register value.
- Reset: after `X(0), reset(0)`, qubit 0 should be |0>.
- Multi-register circuit: verify final histogram concatenates named classical
  registers in declaration order.
- SWAP gate: verify qubits are exchanged.
- Toffoli gate: verify truth table across all 8 basis states.
- CY, CZ, CH: verify on various input states.
- DCX, ECR, iSWAP: verify on input states.
- CP, CRX, CRY, CRZ, CS, CSdg, CSX, CSXdg: verify on input states.
- CU, CU1, CU3: verify on input states.
- RXX, RYY, RZZ, RZX: verify on input states.
- XX-YY, XX+YY: verify on input states.
- CCZ, CSWAP: verify on input states.
- RCCX, MCX, C3SX, RCCCX: verify on input states.
- Multi-controlled gates with variable controls.
- MS gate: verify on 2-qubit system.
- Pauli string gate: verify "XY" on |00>.
- Unitary gate: custom 2x2 matrix applied.
- `prepareState`: prepare specific amplitudes and verify.
- `initialize`: reset + prepare.
- `compose`: compose two circuits and simulate.
- `inverse`: verify inverse circuit undoes the original.
- `globalPhaseGate`: verify state vector phase.
- `barrier` and `delay`: verify no state change.
- Multi-shot statistical tests (see Section 9).

---

### Step 9: Transpilation Pipeline

**File:** `src/transpiler.{ext}`

Implement the full transpilation pipeline used by backends that have hardware
constraints (basis gates, coupling maps). The transpiler is a collection of
passes that transform a `QuantumCircuit` to satisfy a `Target`'s constraints.

The transpiler is used internally by `IBMBackend.transpileAndPackage()` but is
also publicly exposed so users can invoke individual passes.

#### Stage 0: Initialization (Unrolling and OpenQASM 3 Feature Expansion)

- Unroll composite gates (sub-circuits used via `toGate` or `toInstruction`)
  into their constituent primitive operations.
- Unroll control-flow block bodies recursively.
- **Expand gate modifiers** (`ctrl @`, `negctrl @`, `inv @`, `pow(k) @`):
  - `inv @ U` → replace U with U† (conjugate transpose of the gate matrix).
  - `pow(k) @ U` → for positive integer k, repeat U k times; for negative k,
    repeat U† |k| times; for fractional k, compute U^k via matrix
    diagonalization.
  - `ctrl(n) @ U` → build the (n+m)-qubit controlled-U gate matrix: identity on
    all states where any control qubit is |0>, apply U when all control qubits
    are |1>, and preserve any global phase of U as a relative phase only on that
    fully enabled control subspace. If `U = exp(i*α) * V`, synthesize the exact
    enabled-subspace phase operator on the active control pattern together with
    the controlled version of `V`; for all-positive controls this is the exact
    fully-enabled-control phase on the control register itself.
  - `negctrl(n) @ U` → same as ctrl but conditioned on control qubits being |0>.
    Implement as X on each negctrl qubit, then the positive-control construction
    above (including enabled-subspace phase promotion), then X on each negctrl
    qubit.
  - Chained modifiers are applied right-to-left (innermost first):
    `ctrl @ inv @ U` → first compute inv(U) = U†, then build controlled-U†.
  - A zero-qubit `gphase(expr)` may be folded into the owning scope's
    `globalPhase` only after all modifiers have been expanded and only if the
    result is still a bare, unannotated, hoistable scope phase as defined in
    Section 2.6. `inv @ gphase(expr)` becomes `gphase(-expr)`,
    `pow(k) @ gphase(expr)` becomes `gphase(k*expr)`. If the resulting bare
    phase is declaration-dependent, mutable-state-dependent, or otherwise
    non-hoistable, keep it as a zero-qubit instruction.
    `ctrl/negctrl @
    gphase(expr)` synthesize exact enabled-subspace phase
    operators that must **not** be folded into the scope scalar.
- **Expand custom gate definitions** (`gate` bodies): inline the gate body with
  parameter substitution. Recursive gate definitions are expanded iteratively
  until only primitive gates remain.
- **Inline subroutine calls** (`def` bodies): replace `callSubroutine`
  instructions with the subroutine body, substituting arguments. Qubit arguments
  are mapped by reference. Classical arguments are substituted by value.
  Recursive subroutines are expanded up to a configurable depth limit (default:
  100).
- **Evaluate const expressions**: Replace `const` variable references with their
  compile-time values.
- **Resolve input variables**: If `input` values are provided (via parameter
  binding), substitute them. Unresolved inputs remain symbolic.
- **Strip pragmas and annotations**: Remove pragma directives and annotations
  that are not relevant to the target backend. Preserve backend-specific pragmas
  if recognized.
- **Expand gate broadcasting**: If a gate is applied to a register (multiple
  qubits) rather than individual qubits, expand into per-qubit gate
  applications.

#### Stage 1: High-Level Synthesis

Replace abstract/high-level operations with concrete gate sequences:

- `SWAP(a,b) = CX(a,b); CX(b,a); CX(a,b)`
- Multi-controlled gates (MCX, etc.) are decomposed into sequences of 1-qubit
  and 2-qubit gates using known decomposition schemes.
- 3+ qubit gates are decomposed into 1- and 2-qubit gates.

#### Stage 2: Layout (Qubit Mapping — SABRE)

Choose a mapping from circuit "virtual" qubits to backend "physical" qubits.

**Algorithm (simplified SABRE-based):**

1. Build an interaction graph: for each 2-qubit gate, record which pairs of
   virtual qubits interact and how often.
2. Score candidate layouts by summing shortest-path distance (on the coupling
   map) between each interacting pair's assigned physical qubits, weighted by
   interaction frequency.
3. SABRE bidirectional heuristic: a. Start with a trivial or random initial
   layout. b. Traverse the circuit forward, inserting SWAP gates when a 2-qubit
   gate's qubits are not adjacent in the coupling map. c. Choose SWAP gates that
   minimize a cost function:
   ```
   cost(SWAP) = sum over "front layer" gates of
     distance_after_swap(gate.qubit0, gate.qubit1)
   ```
   With look-ahead: also consider the next few layers of gates weighted by a
   decay factor. d. Reverse the circuit and repeat from the layout reached at
   the end. e. Use the layout from the reverse pass as the starting layout for a
   second forward pass. This bidirectional approach produces better layouts.
4. The final layout maps each virtual qubit to a physical qubit.

#### Stage 3: Routing (SWAP Insertion — SABRE)

Using the layout from Stage 2, traverse the circuit instruction by instruction:

```
front_layer = set of gates whose dependencies are all satisfied
while front_layer is not empty:
  executable = gates in front_layer whose qubits are adjacent
  if executable is not empty:
    apply all executable gates
    remove them from front_layer, add newly-ready gates
  else:
    for each SWAP candidate (any edge in coupling map
          involving a qubit used in front_layer):
      score = sum of new distances for front_layer gates after SWAP
      // look-ahead: also score next few layers with decaying weight
    choose SWAP with minimum score
    insert SWAP into circuit
    update qubit layout mapping
```

SWAP decomposition: `SWAP(a,b) = CX(a,b); CX(b,a); CX(a,b)`.

#### Stage 4: Translation (Basis Gate Decomposition)

Convert every gate to the backend's basis gates.

**Single-qubit decomposition (ZYZ):**

For any single-qubit unitary U:

```
U = exp(i*alpha) * Rz(beta) * Ry(gamma) * Rz(delta)
```

Where, with `wrapToPi(x)` meaning normalization to `(-pi, pi]` and `phase`
returning the principal argument in `(-pi, pi]`:

```
Given U = [[a, b], [c, d]], det(U) = a*d - b*c = exp(2i*alpha):
alpha = wrapToPi(phase(det(U)) / 2)   // unique principal half-angle, so alpha ∈ (-pi/2, pi/2]
V = exp(-i*alpha) * U   (so det(V) = 1)
V = [[v00, v01], [v10, v11]]
gamma = clamp(2 * arccos(|v00|), 0, pi)
if gamma is not approximately 0 or pi:
  beta  = wrapToPi(phase(v10) - phase(v00))
  delta = wrapToPi(phase(-v01) - phase(v00))
else if gamma ≈ 0:
  zeta  = phase(v11)
  beta  = zeta
  delta = zeta
else: // gamma ≈ pi
  zeta  = phase(v10)
  beta  = zeta
  delta = -zeta    // if zeta = pi, keep delta = -pi exactly for exact equality
```

These tie-break rules are mandatory so `decomposeZYZ` is deterministic and
matches the canonical ranges from Section 2.5.

**Single-qubit decomposition into {RZ, SX} basis (exact, phase-aware):**

```
M = exp(i*eta) * Rz(a) * SX * Rz(b) * SX * Rz(c)
```

Computed from the exact ZYZ decomposition
`M = exp(i*alpha) * Rz(beta) * Ry(gamma) * Rz(delta)` using:

```
Ry(gamma) = Rz(pi/2) * Rx(gamma) * Rz(-pi/2)
Rx(gamma) = exp(-i*pi/2) * Rz(-pi/2) * SX * Rz(pi - gamma) * SX * Rz(-pi/2)
```

Therefore one canonical exact choice is:

```
eta = alpha - pi/2
a = beta
b = pi - gamma
c = delta - pi
```

Then merge adjacent Rz rotations if desired, while preserving the returned
`eta`. Special cases:

- If gamma = 0 (diagonal gate): no `SX` gates are needed; return the exact phase
  and a single merged `RZ`.
- If gamma = pi (anti-diagonal): only 1 `SX` + 2 `RZ` gates are needed, with the
  omitted `SX` absorbed into the returned exact phase.

**Two-qubit decomposition (KAK / Weyl decomposition):**

Any two-qubit unitary can be decomposed into at most 3 CX gates plus
single-qubit gates and one explicit global phase:

```
U = exp(i*zeta) * (A1 x B1) * CX * (A2 x B2) * CX * (A3 x B3) * CX * (A4 x B4)
```

Number of CX gates needed depends on entangling content:

- 0 CX: `U = A x B` (tensor product, no entanglement)
- 1 CX: gates like CNOT, CZ
- 2 CX: gates like iSWAP, SWAP-like
- 3 CX: most general 2-qubit unitaries

Algorithm outline:

1. Compute `M = U^T * (Y x Y) * U * (Y x Y)` (magic basis transform).
2. Diagonalize M to find eigenvalues -> Weyl chamber coordinates (c0, c1, c2).
3. Based on (c0, c1, c2), determine minimum CX count and compute single-qubit
   gates.
4. Return `zeta` together with the gate sequence (or an equivalent phase-bearing
   circuit object); exact recomposition must include `exp(i*zeta)`.

**ECR-based backends:** For backends using ECR instead of CX:

```
CX(a,b) = (S(b) * SXdg(b)) * ECR(a,b) * X(a)
```

Or use ECR decomposition directly: any 2-qubit gate -> at most 3 ECR +
single-qubit gates.

**Gate direction handling:** If CX(a,b) is available but not CX(b,a):

```
CX(b,a) = H(a) * H(b) * CX(a,b) * H(a) * H(b)
```

#### Stage 5: Optimization

Apply peephole optimizations iteratively until convergence or max iterations:

a) **Single-qubit gate merging:** Consecutive single-qubit gates on the same
qubit are merged (multiply 2x2 matrices), then re-decomposed:
`Rz(a) * Rz(b) -> Rz(a+b)`, `Rx(a) * Rx(b) -> Rx(a+b)`.

b) **Two-qubit gate cancellation:** Consecutive inverse gates cancel:
`CX(a,b) * CX(a,b) -> Identity (removed)`.

c) **Commutation analysis:** Reorder commuting gates to expose cancellations:
`CX(0,1) * Rz(a,0) * CX(0,1) = Rz(a,0)` (Rz on control commutes through CX).

d) **Remove identity gates:** `Rz(0)`, `Rx(0)`, etc. are removed.

e) Iterate until no more reductions or max iterations reached.

#### Public Functions

- `transpile(circuit, target) -> QuantumCircuit` — run the full pipeline.
- `unrollComposites(circuit) -> QuantumCircuit` — Stage 0 (includes gate
  modifier expansion, custom gate inlining, subroutine inlining, const
  evaluation, input resolution, pragma stripping, and gate broadcasting).
- `expandGateModifiers(circuit) -> QuantumCircuit` — expand `ctrl @`,
  `negctrl @`, `inv @`, `pow(k) @` modifiers into primitive gates.
- `inlineGateDefinitions(circuit) -> QuantumCircuit` — inline custom `gate`
  definitions.
- `inlineSubroutines(circuit) -> QuantumCircuit` — inline `def` subroutine
  calls.
- `synthesizeHighLevel(circuit) -> QuantumCircuit` — Stage 1.
- `layoutSABRE(circuit, couplingMap) -> {circuit, layout}` — Stage 2.
- `routeSABRE(circuit, couplingMap, layout) -> QuantumCircuit` — Stage 3.
- `translateToBasis(circuit, basisGates) -> QuantumCircuit` — Stage 4.
- `optimize(circuit) -> QuantumCircuit` — Stage 5.
- `decomposeZYZ(matrix) -> {alpha, beta, gamma, delta}` — ZYZ decomposition.
- `decomposeToRzSx(matrix) -> { globalPhase, instructions }` — Exact phase-aware
  decomposition to the {RZ, SX} basis.
- `decomposeKAK(matrix) -> { globalPhase, instructions }` — Exact phase-aware
  KAK/Weyl decomposition.

**Tests (minimum 55):**

- **ZYZ decomposition:** Decompose H, X, Y, Z, S, T, RX(pi/4), arbitrary U ->
  recompose -> verify equals original matrix (within epsilon).
- **ZYZ canonicalization:** Verify returned `gamma ∈ [0, pi]`,
  `beta/delta ∈ [-pi, pi]`, `alpha = wrapToPi(phase(det(U)) / 2)` (therefore
  `alpha ∈ (-pi/2, pi/2]`), and the `gamma ≈ 0` / `gamma ≈ pi` tie-break rules
  are followed exactly, including exact boundary cases such as `-I` and
  `-RY(pi)`.
- **RZ+SX decomposition:** Decompose several single-qubit gates -> recompose
  using the returned `globalPhase` -> verify exact equality with the original
  matrix.
- **KAK decomposition:** Decompose CX, SWAP, iSWAP, CZ, arbitrary 2-qubit
  unitary -> recompose using the returned `globalPhase` + CX + single-qubit
  gates -> verify exact equality with the original matrix.
- **KAK CX count:** CX needs 1 CX, SWAP needs 3, identity tensor needs 0.
- **High-level synthesis:** SWAP decomposes into 3 CX. CCX decomposes into 1+2
  qubit gates.
- **Layout:** Simple 3-qubit circuit on a linear 5-qubit coupling map produces a
  valid layout.
- **Routing:** After routing, all 2-qubit gates operate on adjacent qubits in
  the coupling map.
- **Basis translation:** Circuit with {H, CX} gates translates to {RZ, SX, CX}
  basis. All gates in output are basis gates.
- **Optimization:** `CX * CX` cancels. `Rz(a) * Rz(b)` merges. `Rz(0)` is
  removed.
- **Full pipeline:** Transpile a Bell state circuit for a 5-qubit linear backend
  -> verify output uses only basis gates, respects coupling map, and simulating
  the transpiled circuit gives the same distribution as the original.
- **ECR backend:** Transpile for ECR-based basis -> verify decomposition.
- **Gate direction:** Verify CX direction reversal with Hadamards.
- **Composite unrolling:** Circuit with `toGate` sub-circuit -> unroll -> verify
  flattened.
- **Identity removal:** Verify Rz(0) and similar are removed after optimization.
- **Gate modifier expansion — `ctrl @`:** `ctrl @ h` on 2 qubits produces a
  CH-equivalent circuit. Verify by simulating on all basis states.
- **Gate modifier expansion — `ctrl(2) @`:** `ctrl(2) @ x` on 3 qubits produces
  a CCX-equivalent circuit. Verify Toffoli truth table.
- **Gate modifier expansion — `negctrl @`:** `negctrl @ x` on 2 qubits flips
  target when control is |0>. Verify on all basis states.
- **Gate modifier expansion — `inv @`:** `inv @ s` produces Sdg. Verify matrix
  equivalence.
- **Gate modifier expansion — `pow(k) @`:** `pow(2) @ s` produces Z. Verify
  matrix equivalence. `pow(2) @ t` produces S.
- **Chained modifiers:** `ctrl @ inv @ s` produces controlled-Sdg. Verify on
  basis states.
- **Gate modifier expansion — modified `gphase`:** `ctrl @ gphase(theta)` stays
  a relative-phase operation on the enabled control subspace and is not folded
  into the enclosing scope `globalPhase`.
- **Custom gate inlining:** Define a gate `bell(q0, q1) { h q0; cx q0, q1; }`,
  use it, unroll -> verify equivalent to H + CX.
- **Parameterized custom gate inlining:** Define
  `myrz(theta, q) { rz(theta) q; }`, bind parameter, unroll -> verify.
- **Subroutine inlining:** Define a subroutine with quantum operations, call it,
  inline -> verify expanded instructions.
- **Const evaluation:** Circuit with const-declared angles -> verify const
  values are substituted during unrolling.
- **Input resolution:** Circuit with `input` variables -> bind values -> verify
  substitution.
- **Gate broadcasting expansion:** Gate applied to register of 3 qubits ->
  unroll -> verify 3 individual gate applications.
- **Pragma stripping:** Circuit with pragmas -> unroll -> verify pragmas removed
  from transpiled output.
- **Annotation stripping:** Circuit with annotations -> unroll -> verify
  annotations removed from transpiled output.

---

### Step 10: IBM Backend

**File:** `src/ibm_backend.{ext}`

Implement `IBMBackend` that implements the `Backend` interface. This backend
represents a real quantum processor accessed via a cloud API.

#### Constructor

```
IBMBackend(configuration: IBMBackendConfiguration)
```

#### Properties (from configuration)

- `numQubits`: from configuration.
- `basisGates`: from configuration (typically `["ecr", "id", "rz", "sx", "x"]`
  or `["cx", "id", "rz", "sx", "x"]`).
- `couplingMap`: from configuration.

**Configuration naming contract**

Use `apiVersion` everywhere for the IBM API-version field and `bearerToken`
everywhere for direct bearer-token authentication. Do not introduce alternate
names such as `ibmApiVersion` or `apiToken` in configuration structs, executable
metadata, helper functions, or tests.

**Optional corsfix transport (TypeScript/JavaScript)**

For TypeScript/JavaScript implementations, add a request-URL helper used by
every outbound fetch call (IAM token exchange, job submit, job status, and job
results):

```
buildRequestUrl(originalUrl) {
  proxy = configuration.corsProxy
  if proxy is null or proxy.enabled is not true:
    return originalUrl

  isBrowser = runtime automatically detects browser environment
  if proxy.mode == "browser-only" and not isBrowser:
    return originalUrl

  return proxy.baseUrl + originalUrl
}
```

Use `https://proxy.corsfix.com/?` as the default `proxy.baseUrl`. The original
fully-qualified URL string is appended directly after the `?`, for example
`https://proxy.corsfix.com/?https://quantum.cloud.ibm.com/api/v1/jobs`. This
mechanism must be explicitly configured, may be disabled at any time, and must
not alter the HTTP method, headers, body, or response parsing semantics. Other
languages should preserve the configuration shape even if they do not need a
browser-specific CORS workaround.

#### `transpileAndPackage(circuit, shots = 1024) -> IBMExecutable`

**Phase 1: Build Target Description**

Construct a `Target` from the configuration:

```
target = new Target()
target.numQubits = configuration.numQubits
for each gate in configuration.basisGates:
  if 1-qubit gate:
    for each qubit q:
      add GateProperties(error, duration) for (q,)
  if 2-qubit gate:
    for each [q0, q1] in couplingMap:
      add GateProperties(error, duration) for (q0, q1)
```

**Phase 2: Compile Circuit**

Use the transpiler module:

1. Unroll composite gates — sub-circuits, `toGate`, `toInstruction` (Stage 0).
2. Synthesize high-level operations — decompose 3+ qubit gates, multi-controlled
   gates, and SWAP into 1- and 2-qubit gates (Stage 1).
3. Layout and routing for coupling map via SABRE (Stages 2-3).
4. Decompose all gates to basis gate set (Stage 4).
5. Optimize gate count (Stage 5).
6. Validate: every gate is in basis, every 2-qubit gate on a connected pair.

**Phase 3: Serialize and Package**

Serialize the compiled circuit to OpenQASM 3 using `OpenQASM3Serializer`. The
serializer must emit every named classical register declaration separately and
in compiled-circuit order. Do **not** collapse multiple classical registers into
one synthetic `bit[M]` declaration when targeting IBM Sampler V2.

Construct the full API request payload using the current Sampler V2 PUB tuple
format:

```
payload = {
  "program_id": "sampler",
  "backend": configuration.name,
  "params": {
    "version": 2,
    "pubs": [
      [serializedCircuit, null, shots]
    ]
  }
}
```

The PUB tuple is ordered as `[circuit, parameterValues, shots]`. For a circuit
with no symbolic parameters, use `null` for `parameterValues`. Use
`params.version = 2` or `params.version = "2"` depending on the target
language's JSON conventions; tests should accept either representation as long
as the Sampler V2 contract is preserved. `shots` is a per-submitted-job value
carried in the PUB tuple; it must not be modeled as a backend capability field
on `BackendConfiguration`.

**Authentication flow**

Support both IBM authentication modes:

- If `configuration.bearerToken` is provided, use it directly for runtime
  requests.
- If `configuration.apiKey` is provided, exchange it for an IAM bearer token
  before calling the IBM Quantum REST API.

Exactly one of `bearerToken` or `apiKey` must be supplied. Reject configurations
where both are supplied or both are missing.

IAM exchange flow:

```
POST https://iam.cloud.ibm.com/identity/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ibm:params:oauth:grant-type:apikey&apikey=<configuration.apiKey>
```

Use the returned `access_token` as the bearer token for runtime requests. Cache
it until expiry and refresh it when needed.

Construct API configuration (default endpoint:
`https://quantum.cloud.ibm.com/api/v1`):

```
apiConfig = {
  "endpoint": configuration.apiEndpoint,  // default: "https://quantum.cloud.ibm.com/api/v1"
  "iamTokenEndpoint": "https://iam.cloud.ibm.com/identity/token",
  "bearerToken": configuration.bearerToken ?? null,
  "apiKey": configuration.apiKey ?? null,
  "corsProxy": configuration.corsProxy ?? {
    "enabled": false,
    "mode": "browser-only",
    "baseUrl": "https://proxy.corsfix.com/?"
  },
  "serviceCrn": configuration.serviceCrn,
  "apiVersion": configuration.apiVersion,
  "headers": {
    "Authorization": "Bearer " + resolvedBearerToken,
    "Service-CRN": configuration.serviceCrn,
    "Accept": "application/json",
    "Content-Type": "application/json",
    "IBM-API-Version": configuration.apiVersion,
  },
  "routes": {
    "submit": "/jobs",
    "status": "/jobs/{job_id}",
    "results": "/jobs/{job_id}/results"
  }
}
```

Return:

```
IBMExecutable {
  payload: object              // Complete JSON-serializable job submission body
  apiConfig: object            // Auth, endpoint, headers, route templates
  compiledCircuit: QuantumCircuit  // For optional inspection
  classicalRegisters: ClassicalRegister[]  // Ordered named classical-register layout used for Sampler V2 result reconstruction
  target: Target
  numClbits: number
}
```

#### `execute(executable) -> ExecutionResult`

Before each fetch, pass the fully-qualified request URL through
`buildRequestUrl(...)` when the TypeScript/JavaScript implementation supports
the optional `corsProxy` configuration described above.

1. **Resolve bearer token:** If `configuration.bearerToken` is present, use it
   directly. Otherwise, if no unexpired IAM bearer token is cached, call the IAM
   token endpoint with `configuration.apiKey` and store the returned
   `access_token` plus expiry metadata.
2. **Submit job:** POST to `endpoint + routes.submit` with payload as body.
   Every runtime request must include `Authorization`, `Service-CRN`, `Accept`,
   `Content-Type`, and `IBM-API-Version` headers.
3. **Poll for completion:** GET `endpoint + routes.status` with `job_id`. Read
   the status from `response.status` when present, otherwise from
   `response.state.status`. Handle at minimum the real capitalized API values:
   "Queued", "Running", "Completed", "Failed", and "Cancelled". Treat "Queued"
   and "Running" as transient states; treat "Completed", "Failed", and
   "Cancelled" as terminal states.
4. **Retrieve results:** GET `endpoint + routes.results`.
5. **Parse results:** Sampler V2 REST responses are PUB-oriented. Inspect each
   PUB result's `data` object and reconstruct each shot from **all** classical
   registers in the compiled circuit's classical-register order. The final
   histogram **must** be rebuilt from those per-register samples; do **not**
   hardcode `meas`, do **not** treat a single
   `response.results[0].data.<register>.samples` payload as the full result for
   a multi-register circuit, and do **not** accept a legacy flat `data.counts`
   object as the canonical source of truth. IBM Runtime REST / Sampler V2 is
   header-driven (`Authorization`, `Service-CRN`, `IBM-API-Version`) and
   register-oriented, so the only valid histogram reconstruction path is to
   concatenate every classical-register sample in circuit order before
   converting counts to percentages.

```
pubResults = response.results or []
sampleCounts = {}
orderedRegisters = executable.classicalRegisters

for pubResult in pubResults:
  data = pubResult.data or {}
  registerEntries = []

  for register in orderedRegisters:
    registerData = data.get(register.name)
    if registerData does not have field "samples":
      raise result parsing error
    registerEntries.append((register.name, register.size, registerData.samples))

  if registerEntries is empty:
    raise result parsing error

  shotCount = number of samples in first register
  for (_, _, samples) in registerEntries:
    if number of samples != shotCount:
      raise result parsing error
  for shotIndex in range(shotCount):
    bitstring = combineRegisterSamplesInRegisterOrder(
      registerEntries,
      shotIndex,
      executable.numClbits
    )
    sampleCounts[bitstring] = sampleCounts.get(bitstring, 0) + 1

totalShots = sum of all sampleCounts values
percentages = {}
for bitstring, count in sampleCounts.items():
  percentages[bitstring] = (count / totalShots) * 100
return percentages
```

#### Coupling Map Details

A typical IBM coupling map is a heavy-hex lattice:

- 5-qubit example:
  `[[0,1],[1,0],[1,2],[2,1],[2,3],[3,2],[3,4],[4,3],[1,3],[3,1]]`
- 27-qubit: grid-like, each qubit connects to 2-3 neighbors.
- 127-qubit: heavy-hex topology, max 3 neighbors per qubit.
- May be asymmetric: ECR(a,b) available but not ECR(b,a).

#### Error Characteristics (informational, for noise-aware layout)

- Single-qubit gate errors: 0.01%-0.1%
- Two-qubit gate errors: 0.5%-5%
- Measurement errors: 0.5%-5%
- T1: 50-300 us, T2: 30-200 us
- Single-qubit duration: 20-60 ns, Two-qubit: 200-800 ns, Measurement: 500-5000
  ns

The target stores per-qubit and per-gate error rates, allowing layout/routing to
prefer lower-error qubits and connections.

**Tests (minimum 20):**

IBM backend tests involve real API calls and require credentials. All IBM
backend tests **must be gated behind a skip mechanism** (see Section 9.3).

Tests that can run without credentials (using the transpilation pipeline only):

- Construct `IBMBackend` with a mock configuration (5-qubit linear chain).
- `transpileAndPackage` on a Bell state circuit: verify the compiled circuit
  uses only basis gates.
- Verify coupling map is respected: all 2-qubit gates on connected pairs.
- Verify the OpenQASM 3 serialization is valid in the payload and preserves
  named classical-register declarations in order.
- Verify the payload structure: has `program_id`, `backend`, `params.version`
  (`2` or `"2"`), and Sampler V2 PUB tuples in `params.pubs` whose third element
  is the submitted job's `shots` value.
- Verify API config supports exactly one authentication mode at a time: direct
  `bearerToken` auth or `apiKey` + IAM token exchange; reject both/neither,
  expose the IAM token endpoint, and include the required headers
  (`Authorization`, `Service-CRN`, `Accept`, `Content-Type`, `IBM-API-Version`)
  plus routes.
- Verify optional `corsProxy` config is preserved in API config, defaults to
  disabled, and for TypeScript/JavaScript rewrites IAM and IBM runtime request
  URLs to `https://proxy.corsfix.com/?<originalUrl>` only in browser runtime
  when enabled.
- Verify `IBMExecutable` contains all required fields, including ordered named
  classical-register metadata for result reconstruction.
- Verify polling logic accepts status from either `status` or `state.status` and
  handles the real capitalized API values ("Queued", "Running", "Completed",
  "Failed", "Cancelled").
- Verify Sampler V2 result parsing dynamically detects all classical register
  names, preserves compiled-circuit register order, rebuilds the final histogram
  by reconstructing each shot from all `results[0].data.<register>.samples`
  payloads in that order, and does not assume a hardcoded `data.meas.samples`,
  `data.c.samples`, a single-register shortcut, or a legacy `data.counts` field.
- Verify mock multi-register IBM results (for example `alpha` + `beta`) are
  recombined into final bitstrings using executable register metadata rather
  than the lexical order of JSON keys.
- Transpile a 3-qubit GHZ circuit for a 5-qubit backend: verify routing inserts
  SWAPs if needed.
- Transpile for ECR-based backend: verify ECR gates in output.
- Transpile for CX-based backend: verify CX gates in output.
- Verify gate direction compliance with asymmetric coupling map.
- Verify the compiled circuit depth is reasonable (not wildly inflated).
- Construct Target from configuration: verify all gates/qubits populated.

Tests that require credentials (skipped by default):

- Submit a Bell state job to a real IBM backend and retrieve results.
- Verify returned percentages sum to 100.
- Verify bitstring format is correct.
- Poll status transitions using the real capitalized API values (for example,
  Queued -> Running -> Completed).
- Handle Failed/Cancelled jobs gracefully.
- End-to-end: build circuit -> transpile -> execute -> verify distribution.

---

### Step 11: qBraid Backend

**File:** `src/qbraid_backend.{ext}`

Implement `QBraidBackend` that implements the `Backend` interface. This backend
represents a quantum device accessed via the qBraid cloud API.

#### Constructor

```
QBraidBackend(configuration: QBraidBackendConfiguration)
```

#### Properties (from configuration)

- `numQubits`: from configuration.
- `basisGates`: from configuration.
- `couplingMap`: from configuration.

**Optional corsfix transport (TypeScript/JavaScript)**

For TypeScript/JavaScript implementations, add the same `buildRequestUrl` helper
described in the IBM backend section and use it for every outbound fetch call
(device discovery, submit, status, and results). When `configuration.corsProxy`
is enabled, browser runtimes prepend `proxy.baseUrl` to the fully-qualified
request URL; when it is disabled, or when `mode = "browser-only"` and the code
is not running in a browser, use the original URL directly. Use
`https://proxy.corsfix.com/?` as the default `proxy.baseUrl`.

#### `transpileAndPackage(circuit, shots = 1024) -> QBraidExecutable`

**Phase 0: Discover Device Capabilities**

Before assuming `program.format = "qasm3"` or any other job input shape, query
`GET /devices/{device_qrn}` (or validate against an equivalent cached device
description) and inspect `data.runInputTypes`. The backend must only emit a job
payload format that is explicitly supported by the target device. If the device
supports `qasm3`, use the OpenQASM 3 payload shape shown below. If it does not,
either transpile/serialize to another supported input type or raise a clear
compatibility error. Do not hardcode a single payload shape for all qBraid
devices.

**Phase 1: Build Target Description**

Construct a `Target` from the configuration (identical approach to IBM):

```
target = new Target()
target.numQubits = configuration.numQubits
for each gate in configuration.basisGates:
  if 1-qubit gate:
    for each qubit q:
      add GateProperties(error, duration) for (q,)
  if 2-qubit gate:
    for each [q0, q1] in couplingMap:
      add GateProperties(error, duration) for (q0, q1)
```

**Phase 2: Compile Circuit**

Use the transpiler module:

1. Unroll composite gates — sub-circuits, `toGate`, `toInstruction` (Stage 0).
2. Synthesize high-level operations — decompose 3+ qubit gates, multi-controlled
   gates, and SWAP into 1- and 2-qubit gates (Stage 1).
3. Layout and routing for coupling map via SABRE (Stages 2-3).
4. Decompose all gates to basis gate set (Stage 4).
5. Optimize gate count (Stage 5).
6. Validate: every gate is in basis, every 2-qubit gate on a connected pair.

**Phase 3: Serialize and Package**

Serialize the compiled circuit to OpenQASM 3 using `OpenQASM3Serializer`.

Construct the full API request payload for a device whose `runInputTypes`
include `qasm3`:

```
payload = {
  "shots": shots,
  "deviceQrn": configuration.deviceQrn,
  "program": {
    "format": "qasm3",
    "data": serializedCircuit
  },
  "name": "quantum-sim-job",
  "tags": {},
  "runtimeOptions": {}
}
```

As with IBM, `shots` is a per-submitted-job value carried in the payload, not a
backend capability field on `BackendConfiguration`.

Construct API configuration (default endpoint:
`https://api-v2.qbraid.com/api/v1`):

```
apiConfig = {
  "endpoint": configuration.apiEndpoint,  // default: "https://api-v2.qbraid.com/api/v1"
  "apiKey": configuration.apiKey,
  "corsProxy": configuration.corsProxy ?? {
    "enabled": false,
    "mode": "browser-only",
    "baseUrl": "https://proxy.corsfix.com/?"
  },
  "headers": {
    "X-API-KEY": configuration.apiKey,
    "Content-Type": "application/json"
  },
  "routes": {
    "device": "/devices/{device_qrn}",
    "submit": "/jobs",
    "status": "/jobs/{job_qrn}",
    "results": "/jobs/{job_qrn}/result"
  }
}
```

Return:

```
QBraidExecutable {
  payload: object              // Complete JSON-serializable job submission body
  apiConfig: object            // Auth, endpoint, headers, route templates
  compiledCircuit: QuantumCircuit  // For optional inspection
  classicalRegisters: ClassicalRegister[]  // Ordered named classical-register layout (for consistency with IBMExecutable; qBraid result parsing uses flat measurementCounts but this preserves register metadata for the caller)
  target: Target
  numClbits: number
}
```

#### `execute(executable) -> ExecutionResult`

All qBraid API v2 responses are wrapped in a `{ success, data }` envelope. Do
**not** assume flattened top-level response fields.

Before each fetch, pass the fully-qualified request URL through
`buildRequestUrl(...)` when the TypeScript/JavaScript implementation supports
the optional `corsProxy` configuration described above.

1. **Submit job:** POST to `endpoint + routes.submit` with payload as body.
   Parse the submission response from its `{ success, data }` envelope, verify
   `submitResult.success`, and read the job identifier from
   `submitResult.data.jobQrn`.
2. **Poll for completion:** GET `endpoint + routes.status` with URL-encoded
   `jobQrn`. Parse the polling response from its `{ success, data }` envelope,
   verify `statusResult.success`, read the status from
   `statusResult.data.status`, and recognize all currently documented qBraid API
   v2 job statuses: "INITIALIZING", "QUEUED", "VALIDATING", "RUNNING",
   "CANCELLING", "CANCELLED", "COMPLETED", "FAILED", "UNKNOWN", and "HOLD".
   Treat "COMPLETED", "FAILED", and "CANCELLED" as terminal states. Treat
   "INITIALIZING", "QUEUED", "VALIDATING", "RUNNING", and "CANCELLING" as
   transient states and continue polling. Add an explicit branch for "UNKNOWN"
   and "HOLD" so they do not break polling: either continue polling as
   non-terminal states or surface a clear status-handling path that preserves
   the observed status for the caller/logs.
3. **Retrieve results:** GET `endpoint + routes.results` with URL-encoded
   `jobQrn`. Parse the response from its `{ success, data }` envelope, verify
   `resultsData.success`, and read the result payload from `resultsData.data`.
4. **Parse results:** Read measurement counts from
   `resultsData.data.resultData.measurementCounts` and convert them to bitstring
   percentages.

```
counts = resultsData.data.resultData.measurementCounts
totalShots = sum of all counts values
percentages = {}
for bitstring, count in counts.items():
  percentages[bitstring] = (count / totalShots) * 100
return percentages
```

Note: The `jobQrn` must be URL-encoded when used in route paths (use
`encodeURIComponent` or equivalent).

**Tests (minimum 20):**

qBraid backend tests involve real API calls and require credentials. All qBraid
backend tests **must be gated behind a skip mechanism** (see Section 9.3).

Tests that can run without credentials (using the transpilation pipeline only):

- Construct `QBraidBackend` with a mock configuration (5-qubit linear chain).
- `transpileAndPackage` on a Bell state circuit: verify the compiled circuit
  uses only basis gates.
- Verify coupling map is respected: all 2-qubit gates on connected pairs.
- Verify the OpenQASM 3 serialization is valid in the payload.
- Verify device capability discovery/validation reads `data.runInputTypes`
  before packaging and rejects unsupported `program.format` assumptions.
- Verify the payload structure for a mock qasm3-capable device: has `shots`,
  `deviceQrn`, `program.format`, `program.data`, and uses the submitted job's
  shot count.
- Verify API config has correct headers (`X-API-KEY`), device-discovery route,
  and job routes.
- Verify optional `corsProxy` config is preserved in API config, defaults to
  disabled, and for TypeScript/JavaScript rewrites device discovery and qBraid
  job request URLs to `https://proxy.corsfix.com/?<originalUrl>` only in browser
  runtime when enabled.
- Verify `QBraidExecutable` contains all required fields.
- Verify qBraid response parsing uses the `{ success, data }` envelope rather
  than assuming flattened responses, using mock fixtures for submit, status, and
  result payloads (`data.jobQrn`, `data.status`,
  `data.resultData.measurementCounts`).
- Verify transient polling statuses include `INITIALIZING`, `QUEUED`,
  `VALIDATING`, `RUNNING`, and `CANCELLING`.
- Verify terminal polling statuses include `COMPLETED`, `FAILED`, and
  `CANCELLED`.
- Verify `UNKNOWN` and `HOLD` are handled explicitly without breaking polling,
  either as non-terminal retry states or via a clear surfaced status branch.
- Transpile a 3-qubit GHZ circuit for a 5-qubit backend: verify routing inserts
  SWAPs if needed.
- Transpile for CX-based backend: verify CX gates in output.
- Transpile for ECR-based backend: verify ECR gates in output.
- Verify gate direction compliance with asymmetric coupling map.
- Verify the compiled circuit depth is reasonable (not wildly inflated).
- Construct Target from configuration: verify all gates/qubits populated.

Tests that require credentials (skipped by default):

- Submit a Bell state job to a real qBraid device and retrieve results.
- Verify returned percentages sum to 100.
- Verify bitstring format is correct.
- Poll documented status transitions such as INITIALIZING -> QUEUED ->
  VALIDATING -> RUNNING -> COMPLETED.
- Handle CANCELLING -> CANCELLED, FAILED, UNKNOWN, and HOLD states gracefully.
- End-to-end: build circuit -> transpile -> execute -> verify distribution.
- List available devices (verify API connectivity).

---

### Step 12: Bloch Sphere Introspection

**File:** `src/bloch.{ext}`

#### `blochSphere(stateVector, qubitIndex, numQubits) -> BlochCoordinates`

**Algorithm:**

1. From the full state vector |psi>, compute the full density matrix:
   `rho_full = |psi><psi|` (outer product).

2. Compute the reduced density matrix `rho` (2x2) for the target qubit by
   partial trace over all other qubits:
   ```
   rho[i][j] = 0
   for each combination of the other (n-1) bits ("env"):
     row_index = insert bit i at position k in env
     col_index = insert bit j at position k in env
     rho[i][j] += psi[row_index] * conj(psi[col_index])
   ```

3. Compute Bloch coordinates:
   ```
   x = 2 * Re(rho[0][1])
   y = 2 * Im(rho[1][0])
   z = Re(rho[0][0] - rho[1][1])
   ```

4. Compute spherical coordinates:
   ```
   r = sqrt(x^2 + y^2 + z^2)
   theta = arccos(z / r) if r > 0, else 0
   phi = atan2(y, x) if (x != 0 or y != 0), else 0
   if phi < 0: phi += 2*pi
   ```

5. Return `{x, y, z, theta, phi, r}`.

**Tests (minimum 20):**

- `|0>`: bloch = (0, 0, 1), theta = 0, r = 1.
- `|1>`: bloch = (0, 0, -1), theta = pi, r = 1.
- `H|0> = |+>`: bloch = (1, 0, 0), r = 1.
- `S*H|0>`: bloch should be (0, 1, 0), r = 1.
- `Y|0> = i|1>`: bloch = (0, 0, -1) (global phase doesn't change Bloch).
- Bell state `(|00>+|11>)/sqrt(2)`, qubit 0: bloch ~ (0, 0, 0), r ~ 0 (maximally
  mixed).
- Bell state, qubit 1: also maximally mixed.
- `RX(pi/2)|0>`: verify Bloch coordinates.
- `RY(pi/2)|0>`: verify Bloch coordinates.
- `RZ(pi/2)|0>`: still |0> (Z rotation on |0> is a phase), bloch = (0, 0, 1).
- Verify spherical coordinates consistency: `r ~ sqrt(x^2 + y^2 + z^2)`.
- Verify `theta` in [0, pi] and `phi` in [0, 2*pi).
- Multi-qubit: 3-qubit GHZ state, each qubit is maximally mixed.
- Pure state: verify r = 1 for any single-qubit pure state.

---

### Step 13: Serializer — OpenQASM 3

**File:** `src/serializer.{ext}`

Define a `Serializer` interface and implement an `OpenQASM3Serializer`.

#### Serializer Interface

```
interface Serializer {
  serialize(circuit: QuantumCircuit) -> string
  deserialize(source: string) -> QuantumCircuit
}
```

This is a public interface so users can implement their own serializers for
other formats.

#### OpenQASM3Serializer

Implements the `Serializer` interface for OpenQASM 3 format. Must handle the
**complete OpenQASM 3 language** as defined below — not just the subset used by
the circuit builder's gate methods.

**`serialize(circuit) -> string`**

Produces a valid OpenQASM 3 program. Example showing many features:

```
OPENQASM 3.0;
include "stdgates.inc";
gphase(0.7854);

// Type declarations
qubit[4] q;
bit[1] flag;
bit[1] data;
bool outcome;
int[32] counter = 0;
uint[8] idx;
float[64] angle_val = 1.5708;
const float[64] MY_PI = 3.14159265358979;
input angle[32] param1;
output bit[2] result;
array[int[32], 4] counts = {0, 0, 0, 0};

// Custom gate definition
gate mygate(theta) q0, q1 {
  rz(theta) q0;
  cx q0, q1;
}

// Subroutine definition
def apply_bell(qubit q0, qubit q1) {
  h q0;
  cx q0, q1;
}

// Gate modifiers
ctrl @ h q[0], q[1];
ctrl(2) @ x q[0], q[1], q[2];
negctrl @ x q[0], q[1];
inv @ s q[0];
pow(2) @ t q[0];
ctrl @ inv @ s q[0], q[1];

// Standard gate instructions
rz(1.5708) q[0];
sx q[0];
cx q[0], q[1];
flag[0] = measure q[0];
data[0] = measure q[1];
reset q[1];
barrier q[0], q[1];
delay[100ns] q[2];

// Physical qubit references
cx $0, $1;

// Classical expressions and assignment
counter = counter + 1;
counter += 1;
idx = counter % 4;
outcome = (flag[0] == 1) && (data[0] == 0);

// Control flow
if (flag[0] == 1) {
  x q[1];
} else if (outcome) {
  y q[1];
} else {
  z q[1];
}
while (flag[0] == 0) {
  h q[0];
  flag[0] = measure q[0];
}
for int i in {0, 1, 2} {
  rz(i * 0.5) q[0];
}
for uint j in [0:3] {
  x q[j];
}
switch (data) {
  case 0: { x q[0]; }
  case 1: { y q[0]; }
  default: { h q[0]; }
}

// Subroutine call
apply_bell q[0], q[1];

// Custom gate use
mygate(pi/4) q[2], q[3];

// Gate broadcasting
h q;

// Pragma and annotations
pragma ibm.noise_model depolarizing
@openqasm.scheduled
cx q[0], q[1];
```

**Complete Serialization Rules:**

**Header and declarations:**

- First line: `OPENQASM 3.0;`
- Second line: `include "stdgates.inc";`
- If the top-level circuit has nonzero hoistable `globalPhase`, emit a
  normalized leading bare `gphase(theta);` immediately after the last required
  `include` directive and before any declarations or executable statements.
- Any remaining statement-level bare `gphase(...)` instructions are emitted in
  source order and are not moved ahead of declarations.
- `qubit[N] q;` declares N qubits (virtual qubits)
- `bit[M] name;` declares one named classical register
- Emit one `bit[...] name;` declaration per classical register in circuit order.
  If a circuit contains multiple named classical registers, preserve them as
  separate declarations; do **not** merge them into a single synthetic register
  during serialization.

**Classical type declarations:**

- `bool name;` or `bool name = value;`
- `int[size] name;` or `int[size] name = value;`
- `uint[size] name;` or `uint[size] name = value;`
- `float[size] name;` or `float[size] name = value;`
- `angle[size] name;` or `angle[size] name = value;`
- `complex[float[size]] name;`
- `duration name = value;` (with timing units: `ns`, `us`/`µs`, `ms`, `s`, `dt`)
- `stretch name;`
- `const type[size] name = value;`
- `input type[size] name;`
- `output type[size] name;`
- `array[baseType, dim1, dim2, ...] name;` (up to 7 dimensions)

**Built-in constants:**

- `pi` / `π` — 3.14159... (also `tau` / `τ` = 2π, `euler` / `ℇ` = e)
- Constants are recognized in parameter expressions and emitted by name.

**Integer literals:**

- Decimal: `42`, hex: `0xFF`, octal: `0o77`, binary: `0b1010`
- Underscores for readability: `1_000_000`

**Bit string literals:** `"01010101"`

**Timing literals:** integer/float + unit: `100ns`, `1.5us`, `200dt`

**Custom gate definitions:**

- `gate name(params) qargs { body }` — serialize gate definitions before use
- Parameters behave as angle types
- Empty body = identity gate
- If a gate body has nonzero hoistable `globalPhase`, serialize it as a
  normalized leading bare `gphase(theta);` immediately after `{` inside the gate
  body.
- Any remaining statement-level bare `gphase(...)` inside the body stays in
  source order.

**Gate modifiers:**

- `ctrl @ gate qubits;` — controlled gate (1 control qubit)
- `ctrl(n) @ gate qubits;` — n-controlled gate
- `negctrl @ gate qubits;` — negative-controlled gate (condition on |0>)
- `negctrl(n) @ gate qubits;` — n negative controls
- `inv @ gate qubits;` — inverse of gate
- `pow(k) @ gate qubits;` — gate to the kth power
- Modifiers can be chained: `ctrl @ inv @ gate qubits;`
- Control-bearing or otherwise still-non-hoistable global phase stays a gate
  instruction, e.g. `ctrl @ gphase(theta) q[0];`; it is **not** normalized into
  the leading scope `gphase(...)`. By contrast, `inv @ gphase(theta)` and
  `pow(k) @ gphase(theta)` are normalized by modifier expansion first and may
  hoist only if the resulting bare phase is hoistable.

**Gate instructions:**

- Gate format: `gate_name(params) qubit_args;`
  - Parameterized: `rz(1.5708) q[0];`
  - No params: `sx q[0];`
  - Two-qubit: `cx q[0], q[1];`
- Gate broadcasting: `h q;` applies H to every qubit in register `q`

**Physical qubit references:**

- `$0`, `$1`, etc. — hardware qubit identifiers
- Used in `defcal` and low-level programs targeting specific hardware

**Non-unitary operations:**

- Measurement: `name[i] = measure q[j];`
- Reset: `reset q[j];`
- Barrier: `barrier q[0], q[1], q[2];` (or `barrier q;` for all)
- Delay: `delay[100ns] q[0];`
- Bare scope global phase: `gphase(0.7854);` For the hoistable scope-global
  scalar, the serializer emits at most one normalized leading bare `gphase(...)`
  statement per scope: after the include block for the top-level program, or
  immediately after `{` for braced scopes. Non-hoistable bare `gphase(...)`
  statements remain in source order instead.

**Classical expressions and operators:**

Arithmetic: `+`, `-`, `*`, `/`, `%`, `**` Bitwise: `&`, `|`, `^`, `~`, `<<`,
`>>` Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=` Logical: `&&`, `||`, `!` Set
membership: `in` (e.g., `i in {0, 3}`) Assignment operators: `=`, `+=`, `-=`,
`*=`, `/=`, `%=`, `**=`, `&=`, `|=`, `^=`, `<<=`, `>>=`

**Built-in math functions** (used in expressions):

`arccos`, `arcsin`, `arctan`, `ceiling`, `cos`, `exp`, `floor`, `log`, `mod`,
`popcount`, `rotl`, `rotr`, `sin`, `sqrt`, `tan`

**Casting:** `type(expression)` — explicit type conversion

**Indexing and slicing:**

- Element access: `array[i]`, `register[i]`
- Range slicing: `register[start:stop]`, `register[start:step:stop]`
- Negative indexing: `array[-1]` (last element)

**Control flow:**

- `if (condition) { body }` — conditional
- `if (condition) { body } else { body }` — if/else
- `if (condition) { body } else if (condition) { body } else { body }` — else-if
  chains
- `while (condition) { body }` — while loop
- `for type var in range/set { body }` — for loop
  - Set iteration: `for int i in {0, 1, 2} { ... }`
  - Range iteration: `for uint i in [0:10] { ... }` or `[start:step:stop]`
- `switch (target) { case value: { body } default: { body } }` — switch
- `break;` — break from loop
- `continue;` — continue to next iteration
- `return;` or `return expression;` — return from subroutine
- Register references in conditions must preserve the original
  classical-register names
- If a control-flow body has nonzero hoistable `globalPhase`, serialize it as a
  normalized leading bare `gphase(theta);` inside that body scope.
- Any non-hoistable bare `gphase(...)` inside the body remains in source order.

**Subroutines:**

- `def name(params) { body }` or `def name(params) -> returnType { body }`
- Parameters: classical types passed by value, qubits by reference
- Array params: `readonly array[type, #dim=N] name` or `mutable array[...]`
- `sizeof(array)` or `sizeof(array, dim)` — query array dimensions
- If a subroutine body has nonzero hoistable `globalPhase`, serialize it as a
  normalized leading bare `gphase(theta);` inside the subroutine body.
- Any non-hoistable bare `gphase(...)` inside the subroutine body remains in
  source order.

**Extern declarations:**

- `extern name(params) -> returnType;`

**Pragma:**

- `pragma namespace.name additional_text`
- Preserved during serialization; ignored if unrecognized during deserialization

**Annotations:**

- `@namespace.name` — attached to the immediately following statement
- Multiple annotations may precede a single statement

**Comments:**

- Line comments: `// comment`
- Block comments: `/* comment */`

**Symbolic parameters** remain as names in the output.

**`deserialize(source) -> QuantumCircuit`**

Parses a **complete OpenQASM 3 program** and reconstructs a `QuantumCircuit`.
Must handle all features listed above:

- Header parsing (`OPENQASM 3.0;`, `include` directives)
- Qubit declarations (virtual `qubit[N]` and physical `$n` references) and
  multiple named classical-register declarations in source order
- Classical type declarations: `bool`, `int[n]`, `uint[n]`, `float[n]`,
  `angle[n]`, `complex[float[n]]`, `duration`, `stretch`
- `const`, `input`, `output` variable modifiers
- `array` declarations with dimensions
- All gate instructions with parameters
- Custom gate definitions (`gate name(params) qargs { body }`)
- Gate modifiers (`ctrl @`, `negctrl @`, `inv @`, `pow(k) @`) including chains
- Gate broadcasting (gate applied to register)
- Measurement, reset, barrier, delay
- Bare, unmodified global phase (`gphase`), folded into the current scope's
  scalar `globalPhase`. After modifier expansion, `inv @ gphase` /
  `pow(k) @
  gphase` may also fold if the resulting bare phase is hoistable;
  control- bearing or otherwise non-hoistable `gphase` stays an ordinary
  instruction or equivalent relative-phase operator
- Classical expressions: arithmetic, bitwise, comparison, logical operators
- Assignment operators (`=`, `+=`, `-=`, `*=`, etc.)
- Built-in constants (`pi`, `tau`, `euler`) and math functions (`sin`, `cos`,
  `sqrt`, `exp`, `arccos`, `arcsin`, `arctan`, `ceiling`, `floor`, `log`, `mod`,
  `popcount`, `rotl`, `rotr`, `tan`)
- Explicit type casting
- Indexing and slicing (element access, ranges, negative indices)
- Integer literals (decimal, hex, octal, binary), bit string literals, timing
  literals
- Control flow blocks (`if`, `else`, `else if`, `while`, `for`, `switch`)
- `break`, `continue`, `return` statements
- Nested control flow
- Subroutine definitions (`def`) and calls
- Extern declarations (`extern`)
- Pragma directives
- Annotations (single and multiple per statement)
- Line comments (`//`) and block comments (`/* */`)

**Tests (minimum 45):**

- Round-trip: `deserialize(serialize(circuit))` produces equivalent circuit.
- Serialize a circuit with every gate type, verify output format.
- Deserialize a hand-written OpenQASM 3 program, verify circuit.
- Verify header format (`OPENQASM 3.0;`, `include`, declarations).
- Verify multiple named classical registers serialize as separate `bit[...]`
  declarations in circuit order.
- Verify gate parameter formatting (decimal, correct precision).
- Verify measurement format: `name[i] = measure q[j];`.
- Verify reset format: `reset q[j];`.
- Verify barrier format.
- Verify delay format with units.
- Verify hoistable bare global phase format: `gphase(theta);`.
- Verify a nonzero top-level hoistable `globalPhase` serializes immediately
  after the include block as a normalized leading bare `gphase(theta);`.
- Verify gate-definition and subroutine-body hoistable `globalPhase` round-trip
  through normalized bare `gphase(theta);` statements without phase loss.
- Verify hoistable symbolic bare `gphase(theta + phi/2);` round-trips without
  numeric coercion.
- Verify exact numeric `globalPhase` values may be reduced modulo `2*pi` when
  doing so preserves the scalar exactly, but symbolic hoistable phases are
  emitted in canonical stored form unless exact simplification proves unity.
- Verify a declaration-dependent or otherwise non-hoistable bare `gphase(expr);`
  remains in statement order and is not folded into the surrounding scope
  `globalPhase`.
- Verify an annotated bare `@tag gphase(theta);` round-trips as a statement and
  is not folded into the surrounding scope `globalPhase`.
- Verify `inv @ gphase(theta);` and `pow(2) @ gphase(theta);` normalize first
  and serialize as hoisted bare `gphase(...)` only when the normalized result is
  hoistable.
- Verify `ctrl @ gphase(theta) q[0];` round-trips as a modifier-bearing
  instruction and is not folded into the surrounding scope `globalPhase`.
- Control flow: serialize/deserialize `if_test` with true and false body.
- Control flow: serialize/deserialize `else if` chains.
- Control flow: serialize/deserialize `while_loop`.
- Control flow: serialize/deserialize `for_loop` with set `{0, 1, 2}` and range
  `[0:10]` and stepped range `[0:2:10]`.
- Control flow: serialize/deserialize `switch` with multiple cases and default.
- Control flow: serialize/deserialize `break` and `continue` in loops.
- Control flow: serialize/deserialize `return` in subroutines.
- Nested control flow round-trip.
- Symbolic parameters survive round-trip.
- Multi-qubit gates serialize correctly (cx, ccx, swap, etc.).
- Parameterized gates serialize angle values correctly.
- Simulate the deserialized circuit and verify same result as original.
- Empty circuit serializes correctly.
- Complex circuit with multiple named classical registers round-trip.
- Deserialize preserves classical-register names and declaration order.
- Parse line comments (`//`) and block comments (`/* */`).
- **Gate modifiers:** serialize/deserialize `ctrl @ h q[0], q[1];`.
- **Gate modifiers:** serialize/deserialize chained
  `ctrl @ inv @ s q[0], q[1];`.
- **Gate modifiers:** serialize/deserialize `negctrl @`, `pow(k) @`.
- **Custom gate definition:** serialize/deserialize
  `gate mygate(theta) q0, q1 { ... }`.
- **Classical types:** serialize/deserialize `bool`, `int[32]`, `uint[8]`,
  `float[64]`, `angle[16]` declarations.
- **Const:** serialize/deserialize `const float[64] PI = 3.14159;`.
- **Input/output:** serialize/deserialize `input angle[32] theta;` and
  `output bit[2] result;`.
- **Array:** serialize/deserialize `array[int[32], 4] counts;`.
- **Classical expressions:** serialize/deserialize arithmetic (`a + b * c`),
  bitwise (`a & b`, `a << 2`), comparison (`a == b`), logical (`a && b`).
- **Assignment operators:** serialize/deserialize `counter += 1;`, `idx %= 4;`.
- **Built-in constants:** serialize/deserialize `pi`, `tau`, `euler` in
  expressions.
- **Built-in math functions:** serialize/deserialize `sin(theta)`, `cos(theta)`,
  `sqrt(2.0)`, `exp(x)`.
- **Casting:** serialize/deserialize `int[32](float_var)`.
- **Indexing/slicing:** serialize/deserialize `register[2:5]`, `array[i, j]`,
  `register[-1]`.
- **Physical qubits:** serialize/deserialize `cx $0, $1;`.
- **Gate broadcasting:** serialize/deserialize `h q;` (H on entire register).
- **Subroutine:** serialize/deserialize `def apply_h(qubit q0) { h q0; }` and
  its call.
- **Extern:** serialize/deserialize `extern parity(bit[n]) -> bit;`.
- **Pragma:** serialize/deserialize `pragma ibm.noise_model depolarizing`.
- **Annotations:** serialize/deserialize `@openqasm.scheduled` before a gate.
- **Integer literals:** deserialize hex `0xFF`, octal `0o77`, binary `0b1010`.
- **Bit string literals:** deserialize `"01010101"`.
- **Timing literals:** deserialize `100ns`, `1.5us`, `200dt`.

---

### Step 14: Public API Surface

**File:** `src/mod.{ext}`

Re-export all public symbols:

```
// Core
QuantumCircuit

// Backend
Backend (interface)
SimulatorBackend
IBMBackend
QBraidBackend

// Bloch
blochSphere (or as method on QuantumCircuit)

// Serialization
Serializer (interface)
OpenQASM3Serializer

// Parameter system
Param

// Algebra
Complex, Matrix

// Transpiler
transpile, unrollComposites, synthesizeHighLevel,
expandGateModifiers, inlineGateDefinitions, inlineSubroutines,
layoutSABRE, routeSABRE, translateToBasis, optimize,
decomposeZYZ, decomposeToRzSx, decomposeKAK

// Types — Core
ClassicalRegister, ClassicalBitRef,
Instruction, Condition, CircuitComplexity, BlochCoordinates,
CorsProxyConfiguration,
BackendConfiguration, IBMBackendConfiguration, QBraidBackendConfiguration,
QubitProperties, GateProperties, Target,
ExecutionResult, SerializedCircuit

// Types — OpenQASM 3 Classical Type System
ClassicalType, ClassicalVariable,
GateDefinition, GateModifier,
SubroutineDefinition, SubroutineParam, ExternDeclaration,
ArrayType, Pragma, Annotation

// Gate constructors (all of them)
globalPhaseGate,
identityGate, hadamard, pauliX, pauliY, pauliZ,
pGate, rGate, rxGate, ryGate, rzGate,
sGate, sdgGate, sxGate, sxdgGate, tGate, tdgGate,
uGate, u1Gate, u2Gate, u3Gate, rvGate,
chGate, cxGate, cyGate, czGate, dcxGate, ecrGate,
swapGate, iswapGate,
cpGate, crxGate, cryGate, crzGate, csGate, csdgGate, csxGate, csxdgGate,
cuGate, cu1Gate, cu3Gate,
rxxGate, ryyGate, rzzGate, rzxGate,
xxMinusYYGate, xxPlusYYGate,
ccxGate, cczGate, cswapGate, rccxGate,
mcxGate, c3sxGate, rcccxGate,
mcxGateN, mcpGateN, mcrxGateN, mcryGateN, mcrzGateN,
msGate, pauliGate
```

Verify nothing is missing from this list by cross-referencing with the gates
section and the OpenQASM 3 type system section.

---

### Step 15: README Documentation

**File:** `README.md`

After all modules are implemented, all tests pass, and the public API surface is
finalized, generate a comprehensive `README.md` at the project root.

#### Structure

The README must contain the following sections in order:

1. **Title and Description** — Library name, one-paragraph summary of what it
   does (self-contained quantum circuit simulation with no external math
   dependencies).

2. **Installation** — How to install or include the library in a project,
   adapted to the target language (e.g., `npm install`, `pip install`,
   `cargo add`, `go get`, or just cloning the repo).

3. **Quick Start** — A complete, runnable example that:
   - Creates a Bell state circuit (`H(0)`, `CX(0,1)`, measure both qubits).
   - Simulates it with `SimulatorBackend`.
   - Prints the measurement results (bitstring histogram).
   - This example must be copy-pasteable and work out of the box.

4. **Public API Reference** — Document **every** public symbol exported from
   `mod.{ext}`, organized by category:

   - **QuantumCircuit** — Constructor, all gate methods (grouped: single-qubit,
     two-qubit, three-qubit, four-qubit, multi-controlled, special), gate
     modifiers (`ctrl`, `negctrl`, `inv`, `pow`), custom gate definitions
     (`defineGate`), measurement, reset, barrier, delay, control flow (`ifTest`,
     `forLoop`, `whileLoop`, `switch`, `breakLoop`, `continueLoop`, `box`),
     composition (`compose`, `toGate`, `toInstruction`, `append`, `inverse`),
     state preparation (`prepareState`, `initialize`), classical variable
     declarations (`declareClassicalVar`, `declareConst`, `declareInput`,
     `declareOutput`, `declareArray`), classical assignment (`classicalAssign`),
     subroutines (`defineSubroutine`, `declareExtern`, `callSubroutine`),
     pragma/annotations (`pragma`, `annotate`), introspection (`complexity`,
     `blochSphere`), parameter binding (`run`), transpilation (`transpile`). For
     each method: signature, one-line description, parameter types.

   - **Complex** — Constructor, constants (`ZERO`, `ONE`, `I`, `MINUS_I`),
     factory methods (`fromPolar`, `exp`), all arithmetic methods. For each:
     signature and one-line description.

   - **Matrix** — Constructor, factories (`identity`, `zeros`), all operations.
     For each: signature and one-line description.

   - **Gate Constructors** — List every gate function grouped by qubit count
     (single, two, three, four, multi-controlled, special). For each: function
     name, gate name, parameter list, and a one-line description.

   - **Param** — Constructor, arithmetic operators, `bind`, `isResolved`.

   - **Backends** — `Backend` interface, `SimulatorBackend` (constructor,
     `execute`, `getStateVector`), `IBMBackend` (constructor, configuration
     fields, transpilation and execution flow), `QBraidBackend` (constructor,
     configuration fields, transpilation and execution flow).

   - **Serialization** — `Serializer` interface, `OpenQASM3Serializer`
     (`serialize`, `deserialize`). Document the complete list of OpenQASM 3
     features supported: classical types, const/input/output, arrays, gate
     definitions, gate modifiers, classical expressions, built-in constants and
     math functions, casting, indexing/slicing, physical qubits, gate
     broadcasting, subroutines, externs, pragma, annotations, all control flow
     including `else if`/`break`/`continue`/`return`, for-loop ranges,
     integer/bit-string/timing literals, and block comments.

   - **Transpiler** — All public functions: `transpile`, `unrollComposites`,
     `expandGateModifiers`, `inlineGateDefinitions`, `inlineSubroutines`,
     `synthesizeHighLevel`, `layoutSABRE`, `routeSABRE`, `translateToBasis`,
     `optimize`, `decomposeZYZ`, `decomposeToRzSx`, `decomposeKAK`.

   - **Bloch Sphere** — `blochSphere` function / method, `BlochCoordinates`
     fields.

   - **Types** — All exported types/interfaces with their fields:
     `ClassicalRegister`, `ClassicalBitRef`, `Instruction`, `Condition`,
     `CircuitComplexity`, `BlochCoordinates`, `CorsProxyConfiguration`,
     `BackendConfiguration`, `IBMBackendConfiguration`,
     `QBraidBackendConfiguration`, `QubitProperties`, `GateProperties`,
     `Target`, `ExecutionResult`, `SerializedCircuit`, `ClassicalType`,
     `ClassicalVariable`, `GateDefinition`, `GateModifier`,
     `SubroutineDefinition`, `SubroutineParam`, `ExternDeclaration`,
     `ArrayType`, `Pragma`, `Annotation`.

5. **Usage Examples** — One short, self-contained code example for each of the
   following scenarios:
   - **GHZ state**: Build a 3-qubit GHZ circuit, simulate, print results.
   - **Parameterized circuit**: Create an `RX(theta)` circuit using `Param`,
     bind different values, simulate each.
   - **State vector inspection**: Use `getStateVector` to inspect amplitudes
     without measurement collapse.
   - **Bloch sphere**: Compute and display Bloch coordinates for a qubit.
   - **OpenQASM 3 round-trip**: Build a circuit, serialize to OpenQASM 3, print
     the source, deserialize back, simulate.
   - **Transpilation**: Build a circuit, transpile for a constrained backend
     (e.g., linear coupling map with `{rz, sx, cx}` basis), simulate the
     transpiled circuit.
   - **Circuit composition**: Build two sub-circuits, compose them, simulate.
   - **Control flow**: A circuit using `ifTest` (measure then conditionally
     apply a gate).
   - **Custom unitary**: Apply a user-defined 2x2 unitary matrix.
   - **Complex and Matrix algebra**: Demonstrate `Complex` arithmetic and
     `Matrix` operations (multiply, tensor, dagger).
   - **Gate modifiers**: Use `ctrl @`, `inv @`, and `pow(k) @` modifiers via the
     circuit builder, serialize to OpenQASM 3, verify output.
   - **Custom gate definition**: Define a reusable gate with `defineGate`, use
     it in a circuit, serialize to OpenQASM 3.

6. **Running Tests** — The exact command(s) to run the full test suite for the
   target language, including how to enable the skipped IBM and qBraid backend
   tests via environment variables.

7. **License** — MIT.

#### Rules

- Every code example must be syntactically valid in the target language.
- Do not document internal/private symbols — only what is exported from
  `mod.{ext}`.
- Keep method descriptions to one line each in the API reference. Longer
  explanations belong in the doc comments in the source code.
- Use consistent formatting: code blocks with language annotation, tables for
  gate lists where appropriate.
- The README must be accurate relative to the final implementation — do not
  document features that were not built, and do not omit features that were.

---

## 7. Patterns & Recipes

### Adding a New Gate

1. Add the matrix constructor in `gates.{ext}`.
2. Add the builder method in `circuit.{ext}`.
3. Wire it in the simulator's gate dispatch in `simulator.{ext}`.
4. Add unit tests for the matrix (unitarity, known values, action on basis
   states).
5. Add a circuit test that uses the gate and verifies simulation output.

### Statistical Tolerance for Shot-Based Tests

For tests comparing simulation results against expected probability
distributions:

- Use `numShots = 4096` (or 8192 for tighter bounds).
- Allow +/-5 tolerance on each percentage value.
- Example: if expected is `{ "00": 50, "11": 50 }`, accept 45-55 for each.
- For deterministic circuits (e.g., X|0> -> `{ "1": 100 }`), use exact
  comparison.

### Classical Bit Ordering

- Classical memory is an ordered list of named registers. Each register is an
  array of bits indexed from 0.
- Flat classical-bit indices are derived by concatenating registers in
  declaration order, then bits within each register in ascending local index.
- Output bitstrings are built from classical registers, not directly from
  qubits: concatenate each register's bitstring in declaration order, with local
  bit 0 as the **rightmost** bit inside its register segment.
- For `if_test` condition comparison: flatten the referenced bits/registers
  using that same ordering, then interpret the rightmost bit as least
  significant.

---

## 8. Edge Cases & Error Handling

- **Invalid qubit index for multi-qubit gates:** If duplicate qubit indices are
  provided (e.g., `cx(0, 0)`), raise an error.
- **Unitary validation in `unitary()` method:** Verify the matrix is unitary
  before accepting.
- **Parameter binding:** If `run()` is called but some parameters remain
  unbound, the returned circuit is still parameterized — do not error.
- **State preparation:** Validate that the state vector is normalized.
- **Circuit too large:** If num_qubits exceeds memory limits, raise a clear
  error before attempting allocation.
- **Division by zero in Complex:** Raise an error.
- **Matrix dimension mismatches:** Raise errors for incompatible operations.
- **OpenQASM 3 parse errors:** Raise clear errors with line numbers.
- **Infinite `while_loop`:** Implement a maximum iteration count (e.g., 10000)
  to prevent hangs. Raise an error if exceeded.

---

## 9. Test Plan — Complete Specification

The test suite must contain **at minimum 617 tests** organized as follows:

### 9.1 Unit Tests per Module

| Module         | Min Tests                |
| -------------- | ------------------------ |
| Types          | 15                       |
| Complex        | 40                       |
| Matrix         | 45                       |
| Gates          | 80                       |
| Parameter      | 15                       |
| Circuit        | 55                       |
| Backend        | 5                        |
| Simulator      | 60                       |
| Transpiler     | 55                       |
| IBM Backend    | 20 (partially skippable) |
| qBraid Backend | 20 (partially skippable) |
| Bloch          | 20                       |
| Serializer     | 45                       |
| **Total**      | **475**                  |

### 9.2 Integration Tests — Quantum Circuit Verification (minimum 142)

Build and simulate complete quantum circuits, run with 1024 or 4096 shots, and
verify output distribution matches expectations.

**Required circuits (implement ALL):**

1. **Identity circuit**: N qubits, no gates, measure all ->
   `{ "000...0": 100 }`.
2. **Single X gate**: `X(0)` -> `{ "1": 100 }`.
3. **Double X gate**: `X(0), X(0)` -> `{ "0": 100 }` (X^2 = I).
4. **Hadamard**: `H(0)` -> ~`{ "0": 50, "1": 50 }`.
5. **Bell state**: `H(0), CX(0,1)` -> ~`{ "00": 50, "11": 50 }`.
6. **Reverse Bell**: `H(0), CX(0,1), CX(0,1), H(0)` -> `{ "00": 100 }`.
7. **GHZ-3**: `H(0), CX(0,1), CX(0,2)` -> ~`{ "000": 50, "111": 50 }`.
8. **GHZ-4**: extend to 4 qubits.
9. **Superposition on all qubits**: `H` on all N qubits -> uniform distribution.
10. **SWAP test**: prepare |01>, SWAP -> |10>.
11. **Double SWAP**: SWAP twice -> back to original.
12. **iSWAP test**: prepare |01>, iSWAP -> verify i|10>.
13. **Toffoli truth table**: Test all 8 input combinations.
14. **CCZ truth table**: Only |111> picks up -1 phase.
15. **Fredkin (CSWAP) truth table**: Test all 8 inputs.
16. **Phase kickback**: `X(1), H(0), CX(0,1), H(0)` -> qubit 0 picks up phase.
17. **Quantum teleportation**: Full protocol. Verify qubit 2 receives the state.
18. **Deutsch-Jozsa** (2-qubit): constant oracle -> "0", balanced oracle -> "1".
19. **Bernstein-Vazirani** (3-qubit): encode secret string, verify recovery.
20. **Every single-qubit gate in isolation on |0>**: 21 gates (id, h, x, y, z,
    p, r, rx, ry, rz, s, sdg, sx, sxdg, t, tdg, u, u1, u2, u3, rv).
21. **Every single-qubit gate on |1>**: Prepare |1> with X, apply gate, verify.
22. **S * S = Z**: Apply S twice, compare state to Z.
23. **T * T = S**: Apply T twice, compare state to S.
24. **H * Z * H = X**: Verify this identity via state vector.
25. **H * X * H = Z**: Verify this identity.
26. **RX(pi)|0> = -i|1>**: Verify the exact state vector `[0, -i]`.
27. **RY(pi)|0> = |1>**: Verify via state vector.
28. **RZ(pi) = (-i) * Z**: Verify by exact matrix or exact state-vector
    comparison, and separately verify equality up to global phase with `Z`.
29. **U gate reproduces X**: `U(pi, 0, pi)` -> same as X.
30. **U gate reproduces H**: `U(pi/2, 0, pi)` -> same as H.
31. **U gate reproduces Y**: `U(pi, pi/2, pi/2)` -> same as Y.
32. **U1 = P**: Verify `U1(l)` matches `P(l)` for several l.
33. **U2 correctness**: Verify `U2(0, pi)` = `H` exactly.
34. **Parameterized RX sweep**: Sweep theta from 0 to 2*pi, verify prob(1)
    follows sin^2(theta/2).
35. **Parameterized RY sweep**: Similar.
36. **Classical conditioning via if_test**: Measure q0, conditionally X q1.
37. **for_loop**: Apply RX(pi/4) in a loop 4 times = RX(pi).
38. **while_loop**: Repeat until measurement gives 1.
39. **switch statement**: Branch on 2-bit register value.
40. **Reset mid-circuit**: Apply X, reset, measure -> always "0".
41. **CH gate**: Controlled-H on various inputs.
42. **CY gate**: Controlled-Y on various inputs.
43. **CZ gate**: Controlled-Z on various inputs.
44. **DCX gate**: Verify on basis states.
45. **ECR gate**: Verify on basis states and entanglement.
46. **CP gate**: Verify for several lambda values.
47. **CRX, CRY, CRZ**: Verify on inputs.
48. **CS, CSdg, CSX, CSXdg**: Verify on inputs.
49. **CU gate**: Verify with specific parameters.
50. **CU1 gate**: Verify CU1(l) = CP(l).
51. **CU3 gate**: Verify on inputs.
52. **RXX gate**: Verify entanglement for known theta.
53. **RYY gate**: Verify on known inputs.
54. **RZZ gate**: Verify phase accumulation.
55. **RZX gate**: Verify on known inputs.
56. **XX-YY gate**: Verify on known inputs.
57. **XX+YY gate**: Verify on known inputs.
58. **RCCX gate**: Verify on basis states.
59. **MCX (C3X)**: Verify flip only when all 3 controls are |1>.
60. **C3SX gate**: Verify SX applied only when all 3 controls are |1>.
61. **RCCCX gate**: Verify on basis states.
62. **Multi-controlled X (4 controls)**: Verify flip only when all 4 are |1>.
63. **Multi-controlled Phase**: Verify phase applied correctly.
64. **Multi-controlled RX**: Verify rotation applied only with all controls |1>.
65. **MS gate**: Verify entanglement on 2 and 3 qubits.
66. **Pauli string "XYZ"**: Verify on 3-qubit input.
67. **Pauli string "II"**: Identity on 2 qubits.
68. **Unitary gate**: Custom 2x2 rotation, verify result.
69. **Unitary gate**: Custom 4x4 (CX equivalent), verify.
70. **prepareState with amplitudes**: Prepare [1/sqrt(2), 1/sqrt(2)], measure
    ~50/50.
71. **prepareState with integer**: Prepare |3> in 2-qubit system.
72. **prepareState with bitstring**: Prepare "10", verify.
73. **initialize**: Reset then prepare, verify from non-zero initial state.
74. **SX * SX = X**: Via circuit, verify state vector.
75. **SX * SXdg = I**: Via circuit, verify state is unchanged.
76. **compose**: Compose Bell state circuit into larger circuit.
77. **toGate + append**: Convert small circuit to gate, append to larger.
78. **inverse**: Create circuit, invert, compose both -> identity.
79. **RV gate**: Verify rotation for specific axis/angle.
80. **Global phase gate**: Verify exp(i*theta) applied to state.
81. **Barrier**: Verify no state change.
82. **Delay**: Verify no state change.
83. **complexity()**: Verify correct depth and gate counts for known circuit.
84. **blochSphere()**: Verify for |0>, |+>, Bell state qubit.
85. **Round-trip serialization**: Build -> serialize to OpenQASM 3 ->
    deserialize -> simulate -> compare results, including circuits with multiple
    named classical registers.
86. **OpenQASM 3 with control flow**: Serialize/deserialize circuit with
    if_test, simulate.
87. **Transpile Bell state for 5-qubit CX backend**: Verify output uses only
    basis gates and produces correct distribution when simulated.
88. **Transpile GHZ-3 for linear coupling map**: Verify routing inserts SWAPs
    and result is still correct.
89. **Transpile + simulate round-trip**: Arbitrary 3-qubit circuit -> transpile
    for constrained backend -> simulate transpiled circuit -> compare with
    original simulation.
90. **Transpile for ECR backend**: Verify ECR-based decomposition.
91. **Transpile with optimization**: Verify `CX * CX` cancellation reduces gate
    count.
92. **KAK decomposition round-trip**: 2-qubit circuit -> decompose -> simulate
    -> verify same result.
93. **ZYZ decomposition round-trip**: Single-qubit circuit -> decompose ->
    simulate -> verify.
94. **IBM payload structure**: Build a circuit with multiple named classical
    registers -> transpile via IBMBackend -> verify payload JSON has correct
    structure, including `params.version` and a Sampler V2 PUB tuple in
    `params.pubs` whose third element is the submitted job's `shots` value.
95. **IBM OpenQASM 3 in payload**: Verify the serialized circuit in the IBM
    payload is valid OpenQASM 3 and preserves separate classical-register
    declarations in order.
96. **Transpile preserves measurement semantics**: Circuit with mid-circuit
    measurement and multiple named classical registers -> transpile -> simulate
    -> same distribution.
97. **qBraid payload structure**: Build circuit -> transpile via QBraidBackend
    for a mock device whose `runInputTypes` include `qasm3` -> verify payload
    JSON has correct structure (`shots`, `deviceQrn`, `program.format`,
    `program.data`).
98. **qBraid OpenQASM 3 in payload**: After validating that the mock device
    supports `qasm3`, verify the serialized circuit in the qBraid payload is
    valid OpenQASM 3. 99-112. **Generate 14 additional small random circuits**
    (2-4 qubits, 3-8 gates) using various gate combinations. Compute expected
    output by manual state-vector calculation or by cross-verifying
    getStateVector, and check against simulation with 1024 shots. 113-132.
    **Generate 20 transpilation verification circuits**: For each, build a small
    circuit (2-3 qubits), transpile it for a mock backend with a constrained
    coupling map and basis gates, then simulate both the original and transpiled
    circuit and verify they produce equivalent distributions.
99. **Gate modifier `ctrl @` end-to-end**: Build circuit with `ctrl @ rz(pi/4)`,
    expand modifiers via transpiler, simulate -> verify controlled-phase
    behavior.
100. **Gate modifier `inv @` end-to-end**: Build circuit with `inv @ s`, expand
     -> verify Sdg behavior via simulation.
101. **Gate modifier `pow(2) @ t` end-to-end**: Expand -> verify S behavior.
102. **Chained modifiers end-to-end**: `ctrl @ inv @ s` -> simulate -> verify
     controlled-Sdg on all basis states.
103. **Custom gate definition end-to-end**: Define `bell(q0, q1)`, use it,
     serialize to OpenQASM 3, deserialize, simulate -> verify Bell state.
104. **OpenQASM 3 classical types round-trip**: Serialize circuit with `bool`,
     `int[32]`, `const`, `input`, `output` -> deserialize -> verify preserved.
105. **OpenQASM 3 gate modifiers round-trip**: Serialize circuit with
     `ctrl @ h`, `inv @ s`, `pow(2) @ t` -> deserialize -> verify modifiers
     preserved.
106. **OpenQASM 3 subroutine round-trip**: Serialize circuit with `def` and
     subroutine call -> deserialize -> verify preserved.
107. **OpenQASM 3 classical expressions round-trip**: Serialize circuit with
     arithmetic and bitwise expressions -> deserialize -> verify preserved.
108. **OpenQASM 3 pragma and annotation round-trip**: Serialize circuit with
     pragma and annotations -> deserialize -> verify preserved.

### 9.3 Skippable Tests (IBM & qBraid Backends)

Tests that require real IBM or qBraid credentials or network access **must be
gated behind a skip mechanism**. The exact mechanism depends on the target
language:

**IBM Backend:**

| Language   | Mechanism                                                                                                                                           |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| TypeScript | Check for (`IBM_BEARER_TOKEN` or `IBM_API_KEY`) and `IBM_SERVICE_CRN`; skip with `Deno.test` ignore                                                 |
| Python     | `@pytest.mark.skipif((not os.environ.get("IBM_BEARER_TOKEN") and not os.environ.get("IBM_API_KEY")) or not os.environ.get("IBM_SERVICE_CRN"), ...)` |
| Rust       | `#[ignore]` attribute, run with `cargo test -- --ignored`                                                                                           |
| Go         | `if (os.Getenv("IBM_BEARER_TOKEN") == "" && os.Getenv("IBM_API_KEY") == "") \|\| os.Getenv("IBM_SERVICE_CRN") == "" { t.Skip(...) }`                |

The skip gate only determines whether IBM credentials are present at all. Once
enabled, the IBM execution tests must still assert that **exactly one** of
`IBM_BEARER_TOKEN` or `IBM_API_KEY` is set.

**qBraid Backend:**

| Language   | Mechanism                                                        |
| ---------- | ---------------------------------------------------------------- |
| TypeScript | Check for `QBRAID_API_KEY` env var; skip with `Deno.test` ignore |
| Python     | `@pytest.mark.skipif(not os.environ.get("QBRAID_API_KEY"), ...)` |
| Rust       | `#[ignore]` attribute, run with `cargo test -- --ignored`        |
| Go         | `if os.Getenv("QBRAID_API_KEY") == "" { t.Skip(...) }`           |

**Convention:** Define two test categories for each cloud backend:

1. **`ibm_transpile` / `qbraid_transpile` tests**: Test transpilation, payload
   construction, and serialization using a mock configuration. These run always.
2. **`ibm_execute` / `qbraid_execute` tests**: Test actual job submission,
   polling, and result retrieval. These are **skipped by default** and only run
   when the respective environment variables are set:
   - IBM: exactly one of `IBM_BEARER_TOKEN` or `IBM_API_KEY`, plus
     `IBM_SERVICE_CRN`, and optionally `IBM_API_ENDPOINT` / `IBM_API_VERSION`.
     These tests should validate CRN-aware authentication, the `IBM-API-Version`
     header, Sampler V2 PUB tuples with per-job `shots`, capitalized runtime
     statuses, and result parsing from register-based `<register>.samples`
     payloads reconstructed in compiled-circuit register order.
   - qBraid: `QBRAID_API_KEY` and `QBRAID_API_ENDPOINT`. These tests should
     validate device discovery and `runInputTypes` handling, `{ success, data }`
     response envelopes, `data.jobQrn`, `data.status`, the full documented
     qBraid status set, explicit handling for `UNKNOWN` and `HOLD`, and
     `data.resultData.measurementCounts`.

The test runner output must clearly indicate which tests were skipped and why.

### 9.4 Test Execution is a Build Step

**Tests are not optional.** The build process is:

1. Implement module.
2. Write tests for that module.
3. Run tests. All must pass.
4. Only then proceed to the next module.

After all modules are complete: 5. Run the full integration test suite. 6. All
617+ tests must pass before the library is considered complete (excluding
skipped IBM and qBraid execution tests).

---

## 10. Verification Checklist

Before declaring the library done, verify:

- [ ] All 21 single-qubit gate functions implemented and tested.
- [ ] All 24 two-qubit gate functions implemented and tested.
- [ ] All 4 three-qubit gate functions (CCX, CCZ, CSWAP, RCCX) implemented and
      tested.
- [ ] All 3 four-qubit gate functions (MCX/C3X, C3SX, RCCCX) implemented and
      tested.
- [ ] Multi-controlled gate constructors (variable controls) for MCX, MCP, MCRX,
      MCRY, MCRZ implemented and tested.
- [ ] MS (Molmer-Sorensen) gate implemented and tested.
- [ ] Pauli string gate implemented and tested.
- [ ] RV (arbitrary axis rotation) gate implemented and tested.
- [ ] Global phase gate implemented and tested.
- [ ] All circuit builder methods wired up (every gate, measure, reset, barrier,
      delay, control flow, composition, inverse, complexity, blochSphere, run).
- [ ] Gate modifier methods (`ctrl`, `negctrl`, `inv`, `pow`) implemented and
      tested on the circuit builder.
- [ ] Custom gate definition (`defineGate`) and usage via `append` implemented.
- [ ] Classical variable declarations (`declareClassicalVar`, `declareConst`,
      `declareInput`, `declareOutput`, `declareArray`) implemented.
- [ ] Classical assignment (`classicalAssign`) with compound operators
      implemented.
- [ ] Subroutine definitions (`defineSubroutine`), extern declarations
      (`declareExtern`), and subroutine calls (`callSubroutine`) implemented.
- [ ] Pragma and annotation methods implemented on circuit builder.
- [ ] Ordered named classical-register metadata is preserved through circuit
      construction, composition, transpilation, serialization, simulation, and
      backend execution.
- [ ] `prepareState()` and `initialize()` implemented and tested.
- [ ] `unitary()` (arbitrary unitary matrix) implemented and tested.
- [ ] `SimulatorBackend.execute()` returns correct distributions for all
      circuits.
- [ ] `getStateVector()` returns correct amplitudes.
- [ ] `blochSphere()` returns correct Bloch coordinates and purity radius.
- [ ] `Serializer` interface defined with `serialize` and `deserialize`.
- [ ] `OpenQASM3Serializer` fully implements serialize and deserialize.
- [ ] OpenQASM 3 serialization handles all gates, measurements, resets,
      barriers, delays, global phase, control flow, and multiple named
      classical-register declarations in order.
- [ ] OpenQASM 3 serialization/deserialization handles gate modifiers (`ctrl @`,
      `negctrl @`, `inv @`, `pow(k) @`) including chained modifiers.
- [ ] OpenQASM 3 serialization/deserialization handles custom `gate` definitions
      with parameters and qubit arguments.
- [ ] OpenQASM 3 serialization/deserialization handles classical typed
      variables: `bool`, `int[n]`, `uint[n]`, `float[n]`, `angle[n]`,
      `complex[float[n]]`, `duration`, `stretch`.
- [ ] OpenQASM 3 serialization/deserialization handles `const`, `input`, and
      `output` variable modifiers.
- [ ] OpenQASM 3 serialization/deserialization handles `array` declarations.
- [ ] OpenQASM 3 serialization/deserialization handles classical expressions:
      arithmetic, bitwise, comparison, logical operators, assignment operators.
- [ ] OpenQASM 3 serialization/deserialization handles built-in constants (`pi`,
      `tau`, `euler`) and math functions (`sin`, `cos`, `sqrt`, `exp`, `arccos`,
      `arcsin`, `arctan`, `ceiling`, `floor`, `log`, `mod`, `popcount`, `rotl`,
      `rotr`, `tan`).
- [ ] OpenQASM 3 serialization/deserialization handles casting, indexing,
      slicing (including negative indices and range syntax).
- [ ] OpenQASM 3 serialization/deserialization handles physical qubit references
      (`$0`, `$1`).
- [ ] OpenQASM 3 serialization/deserialization handles gate broadcasting.
- [ ] OpenQASM 3 serialization/deserialization handles subroutine definitions
      (`def`), extern declarations (`extern`), and subroutine calls.
- [ ] OpenQASM 3 serialization/deserialization handles `else if` chains,
      `break`, `continue`, `return` statements.
- [ ] OpenQASM 3 serialization/deserialization handles for-loop range syntax
      `[start:stop]` and `[start:step:stop]`.
- [ ] OpenQASM 3 serialization/deserialization handles pragma directives and
      annotations.
- [ ] OpenQASM 3 serialization/deserialization handles integer literals (hex,
      octal, binary), bit string literals, and timing literals.
- [ ] OpenQASM 3 serialization/deserialization handles block comments (`/* */`).
- [ ] OpenQASM 3 deserialization reconstructs circuits faithfully, including
      classical-register names and order.
- [ ] Round-trip `deserialize(serialize(circuit))` produces equivalent circuits.
- [ ] Symbolic parameters (`Param`) work with arithmetic expressions.
- [ ] `run()` correctly binds parameters.
- [ ] Control flow (`if_test`, `for_loop`, `while_loop`, `switch`, `break`,
      `continue`, `box`) all implemented and simulated correctly.
- [ ] Circuit `compose()`, `toGate()`, `toInstruction()`, `append()`,
      `inverse()` all work.
- [ ] `complexity()` returns correct metrics.
- [ ] `Backend` interface defined.
- [ ] `SimulatorBackend` implements `Backend` interface.
- [ ] `IBMBackend` implements `Backend` interface.
- [ ] `QBraidBackend` implements `Backend` interface.
- [ ] Transpiler pipeline fully implemented: unroll (including gate modifier
      expansion, custom gate inlining, subroutine inlining, const evaluation,
      input resolution, pragma/annotation stripping, gate broadcasting),
      synthesis, layout (SABRE), routing (SABRE), translation (ZYZ, RZ+SX, KAK),
      optimization.
- [ ] `expandGateModifiers` correctly expands `ctrl @`, `negctrl @`, `inv @`,
      `pow(k) @` and chained modifiers into primitive gates.
- [ ] `inlineGateDefinitions` inlines custom `gate` definitions with parameter
      substitution.
- [ ] `inlineSubroutines` inlines `def` subroutine calls with argument
      substitution.
- [ ] ZYZ decomposition correctly decomposes any single-qubit unitary.
- [ ] RZ+SX decomposition correctly decomposes any single-qubit unitary.
- [ ] KAK/Weyl decomposition correctly decomposes any two-qubit unitary.
- [ ] SABRE layout and routing produce valid results for constrained coupling
      maps.
- [ ] Optimization passes reduce gate count (cancellation, merging, identity
      removal, commutation).
- [ ] IBM payload structure matches specification (program_id, backend,
      `params.version` as `2` or `"2"`, Sampler V2 PUB tuple, per-job `shots`
      carried in the tuple rather than `BackendConfiguration`, and serializer
      output that preserves named classical registers in order).
- [ ] IBM executable contains API config with exactly one authentication mode:
      direct `bearerToken` support or IAM auth flow from `apiKey`, required
      `Authorization`, `Service-CRN`, `Accept`, `Content-Type`, and
      `IBM-API-Version` headers, route templates, capitalized status handling
      from `status` or `state.status`, ordered named classical-register metadata
      on the executable, and all-register Sampler reconstruction from
      `results[0].data.<register>.samples` in compiled-circuit register order to
      rebuild the final histogram.
- [ ] qBraid payload structure matches specification for the selected supported
      `runInputType` (for example, a qasm3-capable device uses per-job `shots`,
      `deviceQrn`, and `program`).
- [ ] Optional backend `corsProxy` configuration exists, defaults to disabled,
      is preserved in executable/API config, and for TypeScript/JavaScript
      browser runtimes rewrites outbound fetch URLs to
      `https://proxy.corsfix.com/?<originalUrl>` only when enabled.
- [ ] qBraid executable contains API config with X-API-KEY header,
      device-discovery route, `runInputTypes` validation/discovery before
      packaging, `{ success, data }` response parsing, `data.jobQrn`,
      `data.status`, `data.resultData.measurementCounts`, full qBraid status
      handling (`INITIALIZING`, `QUEUED`, `VALIDATING`, `RUNNING`, `CANCELLING`,
      `CANCELLED`, `COMPLETED`, `FAILED`, `UNKNOWN`, `HOLD`), terminal-state
      classification for `COMPLETED` / `FAILED` / `CANCELLED`, transient-state
      classification for `INITIALIZING` / `QUEUED` / `VALIDATING` / `RUNNING` /
      `CANCELLING`, and explicit handling for `UNKNOWN` / `HOLD`.
- [ ] Transpiled circuits produce equivalent measurement distributions to
      original circuits when simulated.
- [ ] IBM execution tests exist and are gated behind exactly one of
      `IBM_BEARER_TOKEN` or `IBM_API_KEY`, plus `IBM_SERVICE_CRN` env vars.
- [ ] qBraid execution tests exist and are gated behind `QBRAID_API_KEY` env
      var.
- [ ] Complex class has all methods, fully tested.
- [ ] Matrix class has all methods, fully tested.
- [ ] Gate matrices are all verified unitary.
- [ ] No external math/LA dependencies.
- [ ] 617+ tests passing (excluding skipped IBM and qBraid execution tests).
- [ ] All public API symbols exported from mod.{ext}.
- [ ] `README.md` exists with: installation, quick start (Bell state example),
      full public API reference for every exported symbol, 12 usage examples
      (GHZ, parameterized circuit, state vector, Bloch sphere, OpenQASM 3
      round-trip, transpilation, composition, control flow, custom unitary,
      Complex/Matrix algebra, gate modifiers, custom gate definition), test
      commands, and license.

---

## 11. What is NOT Tested (but IS Implemented)

The IBM backend (`IBMBackend`) and qBraid backend (`QBraidBackend`) are **fully
implemented** including transpilation, OpenQASM 3 serialization, payload
construction, job submission, polling, and result retrieval. However, the
**execution paths** (actual HTTP calls to the IBM and qBraid cloud APIs)
**cannot be tested without credentials**. These tests are gated behind
environment variables and are skipped by default:

- IBM: exactly one of `IBM_BEARER_TOKEN` or `IBM_API_KEY`, plus
  `IBM_SERVICE_CRN`, and optionally `IBM_API_ENDPOINT` / `IBM_API_VERSION`
- qBraid: `QBRAID_API_KEY` and `QBRAID_API_ENDPOINT`

Everything else — transpilation pipeline, payload construction, IBM CRN-aware
auth configuration, IBM Sampler V2 PUB formatting with per-job `shots`, IBM
register-based sample parsing logic that rebuilds the final histogram from all
named classical registers in circuit order using preserved executable metadata,
qBraid device-discovery and `runInputTypes` validation, qBraid
`{ success, data }` envelope parsing, OpenQASM 3 serialization within both
backend flows, coupling map handling — **is tested** using mock configurations.

## 12. Out of Scope

The following are explicitly out of scope:

- **Noise simulation**: The simulator is a perfect, noiseless state-vector
  simulator.
- **Scheduling**: Gate timing and delay scheduling are not simulated (delay is a
  no-op).
- **Additional hardware backends** beyond IBM and qBraid: The `Backend`
  interface allows users to implement their own, but only `SimulatorBackend`,
  `IBMBackend`, and `QBraidBackend` are provided.

---

## 13. Author & License

This AGENTS.md defines the complete specification for building a quantum circuit
simulation library. The resulting library should be released under the **MIT
License**.
