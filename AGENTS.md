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
- Provides introspection (state vectors, Bloch sphere coordinates, circuit
  complexity metrics).
- Supports symbolic parameters with arithmetic expressions, bound at run time.
- Supports classical control flow: `if_test`, `for_loop`, `while_loop`,
  `switch`, `break_loop`, `continue_loop`, and `box`.
- Supports circuit composition: `compose`, `to_gate`, `to_instruction`,
  `append`, `inverse`.
- Exposes a Backend interface with a full `SimulatorBackend`, `IBMBackend`, and
  `QBraidBackend` implementation (including a complete transpilation pipeline).
- Serializes/deserializes circuits to/from **OpenQASM 3** via a public
  Serializer interface.
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
Instruction {
  operation: string            // Gate/operation name (lowercase): "h", "cx", "measure", etc.
  qubits: number[]             // Qubit indices this operation acts on
  clbits: number[]             // Classical bit indices (for measure, control flow)
  params: (number | Param)[]   // Numeric or symbolic parameters (angles)
  condition?: Condition        // Classical condition (for if_test)
  label?: string               // Optional label for custom gates
}

Condition {
  register: number | number[]  // Classical bit index or register indices
  value: number                // Integer value to compare against
}

CircuitComplexity {
  numQubits: number
  numClbits: number
  depth: number
  totalOperations: number
  operationsByName: map<string, number>
  numParameters: number
  globalPhase: number
}

BlochCoordinates {
  x: number                    // Tr(rho * X)
  y: number                    // Tr(rho * Y)
  z: number                    // Tr(rho * Z)
  theta: number                // Polar angle [0, pi]
  phi: number                  // Azimuthal angle [0, 2*pi)
  r: number                    // Purity radius sqrt(x^2 + y^2 + z^2)
}

BackendConfiguration {
  name: string
  numQubits: number
  basisGates: string[]
  couplingMap: [number, number][] | null   // null = all-to-all
  maxShots: number
}

IBMBackendConfiguration extends BackendConfiguration {
  qubitProperties: map<number, QubitProperties>
  gateProperties: map<string, map<string, GateProperties>>  // gate -> qubit_tuple -> props
  maxCircuits: number | null
  apiEndpoint: string           // default: "https://quantum.cloud.ibm.com/api/v1"
  apiToken: string              // IBM Cloud API key used to mint an IAM bearer token
  serviceCrn: string            // required Service-CRN header value for IBM Quantum REST API
  apiVersion: string            // required IBM-API-Version header value (default: "2026-02-15")
}

QBraidBackendConfiguration extends BackendConfiguration {
  qubitProperties: map<number, QubitProperties>
  gateProperties: map<string, map<string, GateProperties>>  // gate -> qubit_tuple -> props
  deviceQrn: string             // qBraid Quantum Resource Name (e.g., "qbraid_qir_simulator")
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

ExecutionResult = map<string, number>  // bitstring -> percentage (0-100, e.g. { "00": 50, "11": 50 })

SerializedCircuit = string  // OpenQASM 3 program text
```

**Tests (minimum 10):**

- Verify Instruction creation with all fields populated.
- Verify Condition with single bit and multi-bit register.
- Verify CircuitComplexity struct defaults and field types.
- Verify BlochCoordinates struct field ranges.
- Verify BackendConfiguration creation with and without coupling map.
- Verify IBMBackendConfiguration includes `serviceCrn` and `apiVersion`.
- Edge cases: 0 qubits, 0 classical bits, empty instructions list.
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

### Step 4: Gate Definitions

**File:** `src/gates.{ext}`

Every gate is a **pure function** returning a `Matrix`. Implement ALL of the
following — no omissions.

#### Zero-Qubit (Global) Gate

| Function                 | Gate         | Matrix / Description                                   |
| ------------------------ | ------------ | ------------------------------------------------------ |
| `globalPhaseGate(theta)` | Global Phase | `[[exp(i*theta)]]` (1x1 matrix, multiplies full state) |

#### Single-Qubit Gates (return 2x2 Matrix)

| Function             | Gate        | Matrix Definition                                                                    |
| -------------------- | ----------- | ------------------------------------------------------------------------------------ |
| `identityGate()`     | I           | `[[1, 0], [0, 1]]`                                                                   |
| `hadamard()`         | H           | `(1/sqrt(2)) * [[1, 1], [1, -1]]`                                                    |
| `pauliX()`           | X           | `[[0, 1], [1, 0]]`                                                                   |
| `pauliY()`           | Y           | `[[0, -i], [i, 0]]`                                                                  |
| `pauliZ()`           | Z           | `[[1, 0], [0, -1]]`                                                                  |
| `pGate(lambda)`      | P(l)        | `[[1, 0], [0, exp(i*l)]]`                                                            |
| `rGate(theta, phi)`  | R(th,ph)    | `[[cos(th/2), -e^(i*ph)*sin(th/2)*i], [e^(-i*ph)*sin(th/2)*i, cos(th/2)]]`           |
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
| `rvGate(vx, vy, vz)` | RV(v)       | Rotation around axis v=(vx,vy,vz) by angle                                           |

**RV Gate formula:** Let `angle = |v|/2`, `n = v/|v|` (unit vector). If |v| = 0,
return identity.

```
[[cos(angle) - i*nz*sin(angle),    (-i*nx - ny)*sin(angle)],
 [(-i*nx + ny)*sin(angle),          cos(angle) + i*nz*sin(angle)]]
```

#### Two-Qubit Gates (return 4x4 Matrix)

| Function                     | Gate          | Description                          |
| ---------------------------- | ------------- | ------------------------------------ |
| `chGate()`                   | CH            | Controlled-Hadamard                  |
| `cxGate()`                   | CX / CNOT     | Controlled-X                         |
| `cyGate()`                   | CY            | Controlled-Y                         |
| `czGate()`                   | CZ            | Controlled-Z                         |
| `dcxGate()`                  | DCX           | Double-CNOT: CX(0,1) then CX(1,0)    |
| `ecrGate()`                  | ECR           | Echoed Cross-Resonance               |
| `swapGate()`                 | SWAP          | Swap two qubits                      |
| `iswapGate()`                | iSWAP         | iSWAP (swap with i phase)            |
| `cpGate(lambda)`             | CP(l)         | Controlled-Phase                     |
| `crxGate(theta)`             | CRX(th)       | Controlled-RX                        |
| `cryGate(theta)`             | CRY(th)       | Controlled-RY                        |
| `crzGate(theta)`             | CRZ(th)       | Controlled-RZ                        |
| `csGate()`                   | CS            | Controlled-S                         |
| `csdgGate()`                 | CSdg          | Controlled-Sdg                       |
| `csxGate()`                  | CSX           | Controlled-SX                        |
| `cuGate(th, ph, l, gamma)`   | CU(th,ph,l,g) | Controlled-U with global phase gamma |
| `cu1Gate(lambda)`            | CU1(l)        | Controlled-U1 (= CP)                 |
| `cu3Gate(th, ph, l)`         | CU3(th,ph,l)  | Controlled-U3                        |
| `rxxGate(theta)`             | RXX(th)       | XX Ising interaction                 |
| `ryyGate(theta)`             | RYY(th)       | YY Ising interaction                 |
| `rzzGate(theta)`             | RZZ(th)       | ZZ Ising interaction                 |
| `rzxGate(theta)`             | RZX(th)       | ZX interaction (cross-resonance)     |
| `xxMinusYYGate(theta, beta)` | XX-YY(th,b)   | Parameterized (XX-YY) interaction    |
| `xxPlusYYGate(theta, beta)`  | XX+YY(th,b)   | Parameterized (XX+YY) interaction    |

**All 4x4 matrices must match the exact definitions from the specification.**

Below are the explicit matrices for every two-qubit gate:

**CH:**

```
[[1, 0, 0,         0        ],
 [0, 1, 0,         0        ],
 [0, 0, 1/sqrt(2), 1/sqrt(2)],
 [0, 0, 1/sqrt(2),-1/sqrt(2)]]
```

**CX:**

```
[[1, 0, 0, 0],
 [0, 1, 0, 0],
 [0, 0, 0, 1],
 [0, 0, 1, 0]]
```

**CY:**

```
[[1, 0, 0,  0],
 [0, 1, 0,  0],
 [0, 0, 0, -i],
 [0, 0, i,  0]]
```

**CZ:**

```
[[1, 0, 0,  0],
 [0, 1, 0,  0],
 [0, 0, 1,  0],
 [0, 0, 0, -1]]
```

**DCX:**

```
[[1, 0, 0, 0],
 [0, 0, 0, 1],
 [0, 1, 0, 0],
 [0, 0, 1, 0]]
```

**ECR:**

```
(1/sqrt(2)) *
[[0,  1,  0,  i],
 [1,  0, -i,  0],
 [0,  i,  0,  1],
 [-i, 0,  1,  0]]
```

**SWAP:**

```
[[1, 0, 0, 0],
 [0, 0, 1, 0],
 [0, 1, 0, 0],
 [0, 0, 0, 1]]
```

**iSWAP:**

```
[[1, 0, 0, 0],
 [0, 0, i, 0],
 [0, i, 0, 0],
 [0, 0, 0, 1]]
```

**CP(l):**

```
[[1, 0, 0, 0         ],
 [0, 1, 0, 0         ],
 [0, 0, 1, 0         ],
 [0, 0, 0, exp(i*l)]]
```

**CRX(th):** Identity in |00>,|01> subspace; RX(th) in |10>,|11> subspace.

```
[[1, 0, 0,              0             ],
 [0, 1, 0,              0             ],
 [0, 0, cos(th/2),      -i*sin(th/2)  ],
 [0, 0, -i*sin(th/2),   cos(th/2)     ]]
```

**CRY(th):** Identity in |00>,|01> subspace; RY(th) in |10>,|11> subspace.

```
[[1, 0, 0,           0          ],
 [0, 1, 0,           0          ],
 [0, 0, cos(th/2),   -sin(th/2) ],
 [0, 0, sin(th/2),    cos(th/2) ]]
```

**CRZ(th):** Identity in |00>,|01> subspace; RZ(th) in |10>,|11> subspace.

```
[[1, 0, 0,              0            ],
 [0, 1, 0,              0            ],
 [0, 0, exp(-i*th/2),   0            ],
 [0, 0, 0,              exp(i*th/2)  ]]
```

**CS:**

```
[[1, 0, 0, 0],
 [0, 1, 0, 0],
 [0, 0, 1, 0],
 [0, 0, 0, i]]
```

**CSdg:**

```
[[1, 0, 0, 0 ],
 [0, 1, 0, 0 ],
 [0, 0, 1, 0 ],
 [0, 0, 0, -i]]
```

**CSX:** Identity in |00>,|01> subspace; SX in |10>,|11> subspace.

```
[[1, 0, 0,         0        ],
 [0, 1, 0,         0        ],
 [0, 0, (1+i)/2,   (1-i)/2  ],
 [0, 0, (1-i)/2,   (1+i)/2  ]]
```

**CU(th, ph, l, gamma):**

```
[[1, 0, 0, 0],
 [0, 1, 0, 0],
 [0, 0, exp(i*gamma)*cos(th/2), -exp(i*(gamma+l))*sin(th/2)],
 [0, 0, exp(i*(gamma+ph))*sin(th/2), exp(i*(gamma+ph+l))*cos(th/2)]]
```

Note: MSB-first ordering. Bit 1 = control, bit 0 = target. The identity acts on
the |00>,|01> subspace (control=0), and the U matrix acts on the |10>,|11>
subspace (control=1).

**CU1(l):** Same as CP(l).

**CU3(th, ph, l):** Identity in |00>,|01>; U3(th,ph,l) in |10>,|11>.

**RXX(th):**

```
[[cos(th/2),  0,             0,             -i*sin(th/2)],
 [0,          cos(th/2),     -i*sin(th/2),  0           ],
 [0,          -i*sin(th/2),  cos(th/2),     0           ],
 [-i*sin(th/2), 0,           0,             cos(th/2)   ]]
```

**RYY(th):**

```
[[cos(th/2),  0,             0,             i*sin(th/2)],
 [0,          cos(th/2),     -i*sin(th/2),  0          ],
 [0,          -i*sin(th/2),  cos(th/2),     0          ],
 [i*sin(th/2), 0,            0,             cos(th/2)  ]]
```

**RZZ(th):**

```
[[exp(-i*th/2), 0,            0,            0            ],
 [0,            exp(i*th/2),  0,            0            ],
 [0,            0,            exp(i*th/2),  0            ],
 [0,            0,            0,            exp(-i*th/2) ]]
```

**RZX(th):**

```
[[cos(th/2),     0,            -i*sin(th/2), 0            ],
 [0,             cos(th/2),    0,            i*sin(th/2)  ],
 [-i*sin(th/2),  0,            cos(th/2),    0            ],
 [0,             i*sin(th/2),  0,            cos(th/2)    ]]
```

**XX-YY(th, beta):**

```
[[cos(th/2),                    0, 0, -i*sin(th/2)*exp(-i*beta)],
 [0,                            1, 0,  0                       ],
 [0,                            0, 1,  0                       ],
 [-i*sin(th/2)*exp(i*beta),     0, 0,  cos(th/2)              ]]
```

**XX+YY(th, beta):**

```
[[1, 0,                            0,                            0],
 [0, cos(th/2),                    -i*sin(th/2)*exp(-i*beta),    0],
 [0, -i*sin(th/2)*exp(i*beta),     cos(th/2),                   0],
 [0, 0,                            0,                            1]]
```

#### Three-Qubit Gates (return 8x8 Matrix)

| Function      | Gate  | Description                    |
| ------------- | ----- | ------------------------------ |
| `ccxGate()`   | CCX   | Toffoli: doubly-controlled NOT |
| `cczGate()`   | CCZ   | Doubly-controlled Z            |
| `cswapGate()` | CSWAP | Fredkin: controlled SWAP       |
| `rccxGate()`  | RCCX  | Relative-phase CCX             |

**CCX (8x8):** Identity except rows/cols 6,7 where X is applied:
`entry [6][6] = 0, entry [6][7] = 1, entry [7][6] = 1, entry [7][7] = 0`.

**CCZ (8x8):** Identity except `entry [7][7] = -1`.

**CSWAP (8x8):** Identity except it swaps entries [5] and [6] (MSB-first:
bit2=control, bit1=target1, bit0=target2; when control=1, swap target1 and
target2, i.e. indices 5=101 and 6=110): `row 5: [0,0,0,0,0,0,1,0]`,
`row 6: [0,0,0,0,0,1,0,0]`.

**RCCX (8x8):** Identity except (MSB-first: bit2=ctrl1, bit1=ctrl2,
bit0=target): `entry [5][5] = -1`, `entry [6][6] = 0, entry [6][7] = -i`,
`entry [7][7] = 0, entry [7][6] = i`.

#### Four-Qubit Gates (return 16x16 Matrix)

| Function      | Gate    | Description                |
| ------------- | ------- | -------------------------- |
| `mcxGate()`   | MCX/C3X | Triple-controlled X        |
| `c3sxGate()`  | C3SX    | Triple-controlled SX       |
| `rcccxGate()` | RCCCX   | Relative-phase CCCX (RC3X) |

**MCX (16x16):** Identity except rows/cols 14,15 where X is applied.

**C3SX (16x16):** Identity except rows/cols 14,15 where SX is applied.

**RCCCX (16x16):** Identity except (MSB-first: bit3=ctrl1, bit2=ctrl2,
bit1=ctrl3, bit0=target): `entry [12][12] = i`, `entry [13][13] = -i`,
`entry [14][14] = 0, entry [14][15] = 1`,
`entry [15][15] = 0, entry [15][14] = -1`.

#### Multi-Controlled Gates (variable number of controls)

These are **functions** (not fixed-size matrices) that construct the appropriate
matrix at call time based on the number of control qubits:

| Function                         | Gate | Description        |
| -------------------------------- | ---- | ------------------ |
| `mcxGateN(numControls)`          | MCX  | N-controlled X     |
| `mcpGateN(lambda, numControls)`  | MCP  | N-controlled Phase |
| `mcrxGateN(theta, numControls)`  | MCRX | N-controlled RX    |
| `mcryGateN(theta, numControls)`  | MCRY | N-controlled RY    |
| `mcrzGateN(lambda, numControls)` | MCRZ | N-controlled RZ    |

For N controls, the matrix is `2^(N+1) x 2^(N+1)`: identity everywhere except
the last two rows/columns where the base gate's 2x2 matrix is applied.

#### Molmer-Sorensen Gate

| Function           | Gate | Description                      |
| ------------------ | ---- | -------------------------------- |
| `msGate(theta, m)` | MS   | Molmer-Sorensen gate on m qubits |

`MS(theta) = exp(-i * (theta/2) * sum_{j<k} X_j X_k)` — produces a `2^m x 2^m`
matrix. This creates pairwise XX interactions among all qubit pairs.

#### Pauli String Gate

| Function                 | Gate  | Description                      |
| ------------------------ | ----- | -------------------------------- |
| `pauliGate(pauliString)` | Pauli | Tensor product of Pauli matrices |

Given a string like `"XYZ"`, computes `X tensor Y tensor Z`. The string is read
**right-to-left**: the rightmost character acts on the first qubit in the list.

#### Verification Requirements for Gates

Every gate matrix must satisfy:

1. **Unitarity:** `G_dagger * G ~ I` (within epsilon).
2. **Correct dimensions.**
3. **Known eigenvalue/eigenvector relationships** where applicable.
4. **Equivalences:** `T = P(pi/4)`, `S = P(pi/2)`, `Z = P(pi)`,
   `X = U(pi, 0, pi)`, `H = U(pi/2, 0, pi)`, `SX*SX = X`, `SX*SXdg = I`,
   `U1(l) = P(l)`, `U3(th,ph,l) = U(th,ph,l)`, `CU1(l) = CP(l)`.

**Tests (minimum 80):**

- Every gate function is called and verified unitary.
- Every gate's matrix elements are checked against known values.
- Phase gate hierarchy: `tGate() ~ pGate(pi/4)`, `sGate() ~ pGate(pi/2)`,
  `pauliZ() ~ pGate(pi)`.
- `uGate(pi, 0, pi) ~ pauliX()`.
- `uGate(pi/2, 0, pi) ~ hadamard()`.
- `u1Gate(l) ~ pGate(l)` for several values.
- `u3Gate(th,ph,l) ~ uGate(th,ph,l)` for several values.
- `rxGate(pi) ~ -i*X` (up to global phase, verify action on states).
- `ryGate(pi)` applied to |0> gives |1>.
- `sxGate() * sxGate() ~ pauliX()`.
- `sxGate() * sxdgGate() ~ identityGate()`.
- `sdgGate() ~ sGate().dagger()`.
- `tdgGate() ~ tGate().dagger()`.
- `cu1Gate(l) ~ cpGate(l)` for several values.
- CX: `CX|00>=|00>`, `CX|01>=|01>`, `CX|10>=|11>`, `CX|11>=|10>`.
- CY: verify on all 4 basis states.
- CZ: verify on all 4 basis states.
- CH: verify on all 4 basis states.
- DCX: verify on all 4 basis states.
- ECR: verify unitarity and matrix elements.
- SWAP: `SWAP|01>=|10>`, `SWAP|10>=|01>`.
- iSWAP: verify `iSWAP|01>=i|10>`, `iSWAP|10>=i|01>`.
- CP(l): verify on all 4 basis states for several lambda values.
- CRX, CRY, CRZ: verify on basis states and unitarity.
- CS, CSdg, CSX: verify on basis states.
- CU: verify on basis states for specific parameters.
- Toffoli: `CCX|110>=|111>`, `CCX|111>=|110>`, all other basis states unchanged.
- CCZ: only `CCZ|111>` picks up -1 phase.
- CSWAP: verify `CSWAP|1,01>=|1,10>`, `CSWAP|1,10>=|1,01>`, control=0 no change.
- RCCX: is unitary, correct dimensions (8x8), verify specific entries.
- RXX(0) ~ I4, RZZ(0) ~ I4, RYY(0) ~ I4, RZX(0) ~ I4.
- RXX, RYY, RZZ, RZX: unitary for several values of theta.
- XX-YY and XX+YY: unitary for several parameter values, verify matrix elements.
- MCX/C3X: verify unitarity, correct dimensions (16x16), flip on |1110>.
- C3SX: verify unitarity, dimensions, apply SX on |1110>.
- RCCCX: verify unitarity, correct dimensions (16x16).
- Multi-controlled gates with variable controls: verify for 1, 2, 3 controls.
- MS gate: verify unitarity for 2 and 3 qubits.
- Pauli string: `"X"` = pauliX, `"XY"` = X tensor Y, verify dimensions and
  elements.
- RV gate: `rvGate(0,0,0) ~ I`, `rvGate(pi,0,0) ~ -i*X` (up to phase),
  unitarity.
- Action of every single-qubit gate on |0> and |1> verified against expected
  output.

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

Qubits and classical bits are allocated **implicitly**: when a gate references
qubit index N, all qubits 0..N are automatically allocated if they do not yet
exist. Same for classical bits.

#### Gate Methods — ALL must be implemented

Every gate method appends one Instruction to the circuit and returns `this` (for
chaining). Angle parameters accept `number | Param`.

**Global Phase:**

- `qc.globalPhaseGate(theta)`

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
- `qc.mcrz(lambda, controlQubits, target)`

**Special Gates:**

- `qc.ms(theta, qubits)` — Molmer-Sorensen
- `qc.pauli(pauliString, qubits)` — Pauli string gate
- `qc.unitary(matrix, qubits)` — arbitrary unitary matrix

**State Preparation:**

- `qc.prepareState(state, qubits?)` — prepare qubits in a given state
  (amplitudes, integer, or bitstring). Qubits must already be |0>.
- `qc.initialize(state, qubits?)` — reset qubits to |0> then prepare state.

**Non-Unitary Operations:**

- `qc.measure(qubit, clbit)`
- `qc.reset(qubit)`
- `qc.barrier(...qubits)` — optimization fence, no state effect
- `qc.delay(duration, qubit, unit)` — no-op in simulation

**Control Flow:**

- `qc.ifTest(condition, trueBody, falseBody?)` — conditional execution.
  `condition` is `{register, value}`. Bodies are `QuantumCircuit` instances.
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
- `qc.toGate(label?)` — convert to reusable gate (unitary only).
- `qc.toInstruction(label?)` — convert to reusable instruction.
- `qc.append(operation, qubitIndices, clbitIndices)` — low-level append.
- `qc.inverse()` — return new circuit with reversed, daggered gates. Negated
  global phase. Only unitary circuits.

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

- `qc.transpile(backend)` — calls `backend.transpileAndPackage(this)` and
  returns the opaque executable. This is the entry point for compiling a circuit
  for a specific backend.

**Tests (minimum 40):**

- Build a circuit with every single gate type and verify instruction count.
- Verify chaining: `qc.h(0).cx(0,1).measure(0,0)`.
- Verify implicit qubit allocation: using qubit 5 allocates qubits 0-5.
- Verify implicit classical bit allocation.
- Verify `run()` with symbolic parameters replaces correctly.
- Verify `run()` partial binding leaves unbound params symbolic.
- Verify `complexity()` returns correct depth, operation counts.
- Verify `inverse()` reverses and daggers gates.
- Verify `compose()` maps qubits and clbits correctly.
- Verify `toGate()` and `toInstruction()`.
- Verify control flow instructions are stored correctly.
- Verify `barrier()` and `delay()` are stored but don't affect state.
- Verify `globalPhaseGate()` is stored.
- Edge cases: circuit with 0 instructions, only measurements, only resets.
- Qubit index validation for multi-qubit gates (no duplicate qubits).
- Verify Pauli string gate with various strings.
- Verify `unitary()` stores the matrix.
- Verify `prepareState()` and `initialize()` instructions.
- Verify `ms()` stores theta and qubit list.

---

### Step 7: Backend Interface

**File:** `src/backend.{ext}`

Define the Backend interface/trait/protocol:

```
interface Backend {
  numQubits: number
  basisGates: string[]
  couplingMap: [number, number][] | null

  transpileAndPackage(circuit: QuantumCircuit) -> Executable
  execute(executable: Executable) -> ExecutionResult
}
```

Where `Executable` is an opaque, backend-specific object.

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

#### `transpileAndPackage(circuit) -> SimulatorExecutable`

Since the simulator supports all gates and has no connectivity constraints, this
simply validates the circuit and wraps it:

```
SimulatorExecutable {
  circuit: QuantumCircuit
  numShots: number
}
```

#### `execute(executable) -> ExecutionResult`

This is the core simulation engine.

**State-vector simulation algorithm:**

1. Initialize state vector to `|0...0> = [1, 0, 0, ..., 0]` (length
   `2^numQubits`).
2. Initialize classical register to all zeros (length `numClbits`).
3. Apply the circuit's global phase at the end (multiply all amplitudes by
   `exp(i * globalPhase)`).
4. For each instruction in order: a. If it's a control flow operation
   (`if_test`, `for_loop`, `while_loop`, `switch`), handle accordingly (see
   below). b. Otherwise apply the gate/operation.
5. After all instructions, sample measurement outcomes.

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
  against the condition value). If true, simulate `trueBody` on current state.
  If false and `falseBody` exists, simulate it.
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
Bitstrings are ordered with qubit 0 as rightmost bit. Sum of percentages = 100.
For example, a Bell state result would be `{ "00": 50, "11": 50 }`. Percentages
are computed from shot counts: `percentage = (count / numShots) * 100`.

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
- Parameterized circuit: `RX(theta)` with `theta = pi` on |0> should give |1>
  (up to global phase).
- Y gate on |0>: verify state vector is [0, i].
- RY(pi/2) on |0>: verify equal superposition.
- Classical conditioning via `if_test`: measure qubit 0, conditionally apply X
  to qubit 1.
- `for_loop`: repeated RX rotations with loop parameter.
- `while_loop`: measure until |1>.
- `switch`: branch on classical register value.
- Reset: after `X(0), reset(0)`, qubit 0 should be |0>.
- SWAP gate: verify qubits are exchanged.
- Toffoli gate: verify truth table across all 8 basis states.
- CY, CZ, CH: verify on various input states.
- DCX, ECR, iSWAP: verify on input states.
- CP, CRX, CRY, CRZ, CS, CSdg, CSX: verify on input states.
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

#### Stage 0: Initialization (Unrolling)

- Unroll composite gates (sub-circuits used via `toGate` or `toInstruction`)
  into their constituent primitive operations.
- Unroll control-flow block bodies recursively.

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

Where:

```
Given U = [[a, b], [c, d]], det(U) = a*d - b*c = exp(2i*alpha):
alpha = phase(det(U)) / 2
V = exp(-i*alpha) * U   (so det(V) = 1)
V = [[v00, v01], [v10, v11]]
gamma = 2 * arccos(|v00|)
if |v00| > 0 and |v10| > 0:
  beta  = phase(v10) - phase(v00)
  delta = phase(-v01) - phase(v00)
(handle edge cases where v00 or v10 are zero)
```

**Single-qubit decomposition into {RZ, SX} basis:**

```
U = Rz(a) * SX * Rz(b) * SX * Rz(c)
```

Computed from the ZYZ decomposition using:

```
Ry(gamma) = Rz(-pi/2) * Rx(gamma) * Rz(pi/2)
Rx(gamma) = Rz(-pi/2) * SX * Rz(pi - gamma) * SX * Rz(-pi/2)
```

Then merge adjacent Rz rotations. Special cases:

- If gamma = 0 (diagonal gate): only 1 RZ needed.
- If gamma = pi (anti-diagonal): only 1 SX + 2 RZ needed.

**Two-qubit decomposition (KAK / Weyl decomposition):**

Any two-qubit unitary can be decomposed into at most 3 CX gates plus
single-qubit gates:

```
U = (A1 x B1) * CX * (A2 x B2) * CX * (A3 x B3) * CX * (A4 x B4)
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
- `unrollComposites(circuit) -> QuantumCircuit` — Stage 0.
- `synthesizeHighLevel(circuit) -> QuantumCircuit` — Stage 1.
- `layoutSABRE(circuit, couplingMap) -> {circuit, layout}` — Stage 2.
- `routeSABRE(circuit, couplingMap, layout) -> QuantumCircuit` — Stage 3.
- `translateToBasis(circuit, basisGates) -> QuantumCircuit` — Stage 4.
- `optimize(circuit) -> QuantumCircuit` — Stage 5.
- `decomposeZYZ(matrix) -> {alpha, beta, gamma, delta}` — ZYZ decomposition.
- `decomposeToRzSx(matrix) -> Instruction[]` — Decompose to {RZ, SX} basis.
- `decomposeKAK(matrix) -> Instruction[]` — KAK/Weyl decomposition.

**Tests (minimum 40):**

- **ZYZ decomposition:** Decompose H, X, Y, Z, S, T, RX(pi/4), arbitrary U ->
  recompose -> verify equals original matrix (within epsilon).
- **RZ+SX decomposition:** Decompose several single-qubit gates -> recompose ->
  verify equals original.
- **KAK decomposition:** Decompose CX, SWAP, iSWAP, CZ, arbitrary 2-qubit
  unitary -> recompose using CX + single-qubit gates -> verify.
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

#### `transpileAndPackage(circuit) -> IBMExecutable`

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

1. Decompose all gates to basis gate set (Stage 4).
2. Layout and routing for coupling map via SABRE (Stages 2-3).
3. Optimize gate count (Stage 5).
4. Validate: every gate is in basis, every 2-qubit gate on a connected pair.

**Phase 3: Serialize and Package**

Serialize the compiled circuit to OpenQASM 3 using `OpenQASM3Serializer`.

Construct the full API request payload using the current Sampler V2 PUB tuple
format:

```
payload = {
  "program_id": "sampler",
  "backend": configuration.name,
  "params": {
    "version": 2,
    "pubs": [
      [serializedCircuit, null, configuration.maxShots]
    ]
  }
}
```

The PUB tuple is ordered as `[circuit, parameterValues, shots]`. For a circuit
with no symbolic parameters, use `null` for `parameterValues`.

**Authentication flow**

`configuration.apiToken` is the long-lived IBM Cloud API key, not the bearer
token sent to the runtime service. Before calling the IBM Quantum REST API,
exchange the API key for an IAM bearer token:

```
POST https://iam.cloud.ibm.com/identity/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ibm:params:oauth:grant-type:apikey&apikey=<configuration.apiToken>
```

Use the returned `access_token` as the bearer token for runtime requests. Cache
it until expiry and refresh it when needed.

Construct API configuration (default endpoint:
`https://quantum.cloud.ibm.com/api/v1`):

```
apiConfig = {
  "endpoint": configuration.apiEndpoint,  // default: "https://quantum.cloud.ibm.com/api/v1"
  "iamTokenEndpoint": "https://iam.cloud.ibm.com/identity/token",
  "apiKey": configuration.apiToken,
  "serviceCrn": configuration.serviceCrn,
  "apiVersion": configuration.apiVersion,
  "headers": {
    "Authorization": "Bearer " + bearerToken,
    "Service-CRN": configuration.serviceCrn,
    "IBM-API-Version": configuration.apiVersion,
    "Content-Type": "application/json",
    "Accept": "application/json"
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
  target: Target
  numClbits: number
}
```

#### `execute(executable) -> ExecutionResult`

1. **Acquire bearer token:** If no unexpired IAM bearer token is cached, call
   the IAM token endpoint with `configuration.apiToken` and store the returned
   `access_token` plus expiry metadata.
2. **Submit job:** POST to `endpoint + routes.submit` with payload as body.
   Every runtime request must include `Authorization`, `Service-CRN`, and
   `IBM-API-Version` headers.
3. **Poll for completion:** GET `endpoint + routes.status` with `job_id`. Read
   the status from `response.state.status` and handle at minimum: "Queued",
   "Running", "Completed", "Failed", and "Cancelled".
4. **Retrieve results:** GET `endpoint + routes.results`.
5. **Parse results:** Current Sampler V2 REST responses expose per-PUB samples
   under `response.results[0].data.meas.samples` as hex strings. Convert that
   sample list into a histogram, pad each sample to `numClbits`, and convert
   counts to percentages.

```
sampleCounts = {}
for sample in response.results[0].data.meas.samples:
  bitstring = hexToBinary(sample, executable.numClbits)
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
- Verify the OpenQASM 3 serialization is valid in the payload.
- Verify the payload structure: has `program_id`, `backend`, `params.version`,
  and Sampler V2 PUB tuples in `params.pubs`.
- Verify API config has the IAM token endpoint, required headers
  (`Authorization`, `Service-CRN`, `IBM-API-Version`), and routes.
- Verify `IBMExecutable` contains all required fields.
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
- Poll status transitions: Queued -> Running -> Completed.
- Handle failed/cancelled jobs gracefully.
- End-to-end: build circuit -> transpile -> execute -> verify distribution.

---

### Step 10b: qBraid Backend

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

#### `transpileAndPackage(circuit) -> QBraidExecutable`

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

1. Decompose all gates to basis gate set (Stage 4).
2. Layout and routing for coupling map via SABRE (Stages 2-3).
3. Optimize gate count (Stage 5).
4. Validate: every gate is in basis, every 2-qubit gate on a connected pair.

**Phase 3: Serialize and Package**

Serialize the compiled circuit to OpenQASM 3 using `OpenQASM3Serializer`.

Construct the full API request payload:

```
payload = {
  "shots": configuration.maxShots,
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

Construct API configuration (default endpoint:
`https://api-v2.qbraid.com/api/v1`):

```
apiConfig = {
  "endpoint": configuration.apiEndpoint,  // default: "https://api-v2.qbraid.com/api/v1"
  "apiKey": configuration.apiKey,
  "headers": {
    "X-API-KEY": configuration.apiKey,
    "Content-Type": "application/json"
  },
  "routes": {
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
  target: Target
  numClbits: number
}
```

#### `execute(executable) -> ExecutionResult`

1. **Submit job:** POST to `endpoint + routes.submit` with payload as body. Read
   the job identifier from `submitResult.data.jobQrn`.
2. **Poll for completion:** GET `endpoint + routes.status` with URL-encoded
   `jobQrn`. Read the status from `statusResult.data.status`. Handle statuses:
   "INITIALIZING", "QUEUED", "RUNNING", "COMPLETED", "FAILED", "CANCELLED".
3. **Retrieve results:** GET `endpoint + routes.results` with URL-encoded
   `jobQrn`.
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
- Verify the payload structure: has `shots`, `deviceQrn`, `program.format`,
  `program.data`.
- Verify API config has correct headers (`X-API-KEY`), routes.
- Verify `QBraidExecutable` contains all required fields.
- Verify nested qBraid response parsing using mock fixtures for submit, status,
  and result payloads (`data.jobQrn`, `data.status`,
  `data.resultData.measurementCounts`).
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
- Poll status transitions: QUEUED -> RUNNING -> COMPLETED.
- Handle FAILED/CANCELLED jobs gracefully.
- End-to-end: build circuit -> transpile -> execute -> verify distribution.
- List available devices (verify API connectivity).

---

### Step 11: Bloch Sphere Introspection

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

### Step 12: Serializer — OpenQASM 3

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

Implements the `Serializer` interface for OpenQASM 3 format.

**`serialize(circuit) -> string`**

Produces a valid OpenQASM 3 program:

```
OPENQASM 3.0;
include "stdgates.inc";
qubit[N] q;
bit[M] c;

// Gate instructions
rz(1.5708) q[0];
sx q[0];
cx q[0], q[1];
c[0] = measure q[0];
reset q[1];
barrier q[0], q[1];
delay[100ns] q[2];
gphase(0.7854);

// Control flow
if (c[0] == 1) {
  x q[1];
}
while (c[0] == 0) {
  h q[0];
  c[0] = measure q[0];
}
for int i in {0, 1, 2} {
  rz(i * 0.5) q[0];
}
switch (c) {
  case 0: { x q[0]; }
  default: { h q[0]; }
}
```

**Rules:**

- First line: `OPENQASM 3.0;`
- Second line: `include "stdgates.inc";`
- `qubit[N] q;` declares N qubits
- `bit[M] c;` declares M classical bits
- Gate format: `gate_name(params) qubit_args;`
  - Parameterized: `rz(1.5708) q[0];`
  - No params: `sx q[0];`
  - Two-qubit: `cx q[0], q[1];`
- Measurement: `c[i] = measure q[j];`
- Reset: `reset q[j];`
- Barrier: `barrier q[0], q[1], q[2];` (or `barrier q;` for all)
- Delay: `delay[100ns] q[0];`
- Global phase: `gphase(0.7854);`
- Control flow: `if`, `while`, `for`, `switch` with braces
- Parameters are decimal floating-point numbers
- Symbolic parameters remain as names in the output

**`deserialize(source) -> QuantumCircuit`**

Parses an OpenQASM 3 program and reconstructs a `QuantumCircuit`. Must handle:

- Header parsing (`OPENQASM 3.0;`, `include` directives)
- Qubit and classical bit declarations
- All gate instructions with parameters
- Measurement, reset, barrier, delay
- Global phase (`gphase`)
- Control flow blocks (`if`, `while`, `for`, `switch`)
- Nested control flow
- Comments (lines starting with `//`)

**Tests (minimum 25):**

- Round-trip: `deserialize(serialize(circuit))` produces equivalent circuit.
- Serialize a circuit with every gate type, verify output format.
- Deserialize a hand-written OpenQASM 3 program, verify circuit.
- Verify header format (`OPENQASM 3.0;`, `include`, declarations).
- Verify gate parameter formatting (decimal, correct precision).
- Verify measurement format: `c[i] = measure q[j];`.
- Verify reset format: `reset q[j];`.
- Verify barrier format.
- Verify delay format with units.
- Verify global phase format: `gphase(theta);`.
- Control flow: serialize/deserialize `if_test` with true and false body.
- Control flow: serialize/deserialize `while_loop`.
- Control flow: serialize/deserialize `for_loop`.
- Control flow: serialize/deserialize `switch` with multiple cases and default.
- Nested control flow round-trip.
- Symbolic parameters survive round-trip.
- Multi-qubit gates serialize correctly (cx, ccx, swap, etc.).
- Parameterized gates serialize angle values correctly.
- Simulate the deserialized circuit and verify same result as original.
- Empty circuit serializes correctly.
- Complex circuit (Bell state + measurement) round-trip.
- Parse comments in OpenQASM 3 source.

---

### Step 13: Public API Surface

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
layoutSABRE, routeSABRE, translateToBasis, optimize,
decomposeZYZ, decomposeToRzSx, decomposeKAK

// Types
Instruction, Condition, CircuitComplexity, BlochCoordinates,
BackendConfiguration, IBMBackendConfiguration, QBraidBackendConfiguration,
QubitProperties, GateProperties, Target,
ExecutionResult

// Gate constructors (all of them)
globalPhaseGate,
identityGate, hadamard, pauliX, pauliY, pauliZ,
pGate, rGate, rxGate, ryGate, rzGate,
sGate, sdgGate, sxGate, sxdgGate, tGate, tdgGate,
uGate, u1Gate, u2Gate, u3Gate, rvGate,
chGate, cxGate, cyGate, czGate, dcxGate, ecrGate,
swapGate, iswapGate,
cpGate, crxGate, cryGate, crzGate, csGate, csdgGate, csxGate,
cuGate, cu1Gate, cu3Gate,
rxxGate, ryyGate, rzzGate, rzxGate,
xxMinusYYGate, xxPlusYYGate,
ccxGate, cczGate, cswapGate, rccxGate,
mcxGate, c3sxGate, rcccxGate,
mcxGateN, mcpGateN, mcrxGateN, mcryGateN, mcrzGateN,
msGate, pauliGate
```

Verify nothing is missing from this list by cross-referencing with the gates
section.

---

### Step 14: README Documentation

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
     two-qubit, three-qubit, four-qubit, multi-controlled, special),
     measurement, reset, barrier, delay, control flow (`ifTest`, `forLoop`,
     `whileLoop`, `switch`, `breakLoop`, `continueLoop`, `box`), composition
     (`compose`, `toGate`, `toInstruction`, `append`, `inverse`), state
     preparation (`prepareState`, `initialize`), introspection (`complexity`,
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
     (`serialize`, `deserialize`).

   - **Transpiler** — All public functions: `transpile`, `unrollComposites`,
     `synthesizeHighLevel`, `layoutSABRE`, `routeSABRE`, `translateToBasis`,
     `optimize`, `decomposeZYZ`, `decomposeToRzSx`, `decomposeKAK`.

   - **Bloch Sphere** — `blochSphere` function / method, `BlochCoordinates`
     fields.

   - **Types** — All exported types/interfaces with their fields: `Instruction`,
     `Condition`, `CircuitComplexity`, `BlochCoordinates`,
     `BackendConfiguration`, `IBMBackendConfiguration`,
     `QBraidBackendConfiguration`, `QubitProperties`, `GateProperties`,
     `Target`, `ExecutionResult`.

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

- Classical register is an array of bits indexed from 0.
- Output bitstring: qubit 0 is the **rightmost** bit.
- For `if_test` condition comparison: the classical register's integer value is
  computed from the bits (bit 0 is least significant).

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

The test suite must contain **at minimum 550 tests** organized as follows:

### 9.1 Unit Tests per Module

| Module         | Min Tests                |
| -------------- | ------------------------ |
| Types          | 10                       |
| Complex        | 40                       |
| Matrix         | 45                       |
| Gates          | 80                       |
| Parameter      | 15                       |
| Circuit        | 40                       |
| Backend        | 5                        |
| Simulator      | 60                       |
| Transpiler     | 40                       |
| IBM Backend    | 20 (partially skippable) |
| qBraid Backend | 20 (partially skippable) |
| Bloch          | 20                       |
| Serializer     | 25                       |
| **Total**      | **420**                  |

### 9.2 Integration Tests — Quantum Circuit Verification (minimum 130)

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
26. **RX(pi)|0> = -i|1>**: Verify via state vector (magnitude of |1> is 1).
27. **RY(pi)|0> = |1>**: Verify via state vector.
28. **RZ(pi) ~ Z** (up to global phase): Verify via measurement statistics.
29. **U gate reproduces X**: `U(pi, 0, pi)` -> same as X.
30. **U gate reproduces H**: `U(pi/2, 0, pi)` -> same as H.
31. **U gate reproduces Y**: `U(pi, pi/2, pi/2)` -> same as Y.
32. **U1 = P**: Verify `U1(l)` matches `P(l)` for several l.
33. **U2 correctness**: Verify `U2(0, pi)` = H (up to global phase).
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
48. **CS, CSdg, CSX**: Verify on inputs.
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
    deserialize -> simulate -> compare results.
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
94. **IBM payload structure**: Build circuit -> transpile via IBMBackend ->
    verify payload JSON has correct structure, including `params.version` and a
    Sampler V2 PUB tuple in `params.pubs`.
95. **IBM OpenQASM 3 in payload**: Verify the serialized circuit in the IBM
    payload is valid OpenQASM 3.
96. **Transpile preserves measurement semantics**: Circuit with mid-circuit
    measurement -> transpile -> simulate -> same distribution.
97. **qBraid payload structure**: Build circuit -> transpile via QBraidBackend
    -> verify payload JSON has correct structure (`shots`, `deviceQrn`,
    `program.format`, `program.data`).
98. **qBraid OpenQASM 3 in payload**: Verify the serialized circuit in the
    qBraid payload is valid OpenQASM 3. 99-112. **Generate 14 additional small
    random circuits** (2-4 qubits, 3-8 gates) using various gate combinations.
    Compute expected output by manual state-vector calculation or by
    cross-verifying getStateVector, and check against simulation with 1024
    shots. 113-132. **Generate 20 transpilation verification circuits**: For
    each, build a small circuit (2-3 qubits), transpile it for a mock backend
    with a constrained coupling map and basis gates, then simulate both the
    original and transpiled circuit and verify they produce equivalent
    distributions.

### 9.3 Skippable Tests (IBM & qBraid Backends)

Tests that require real IBM or qBraid credentials or network access **must be
gated behind a skip mechanism**. The exact mechanism depends on the target
language:

**IBM Backend:**

| Language   | Mechanism                                                                                              |
| ---------- | ------------------------------------------------------------------------------------------------------ |
| TypeScript | Check for `IBM_API_KEY` and `IBM_SERVICE_CRN` env vars; skip with `Deno.test` ignore                   |
| Python     | `@pytest.mark.skipif(not os.environ.get("IBM_API_KEY") or not os.environ.get("IBM_SERVICE_CRN"), ...)` |
| Rust       | `#[ignore]` attribute, run with `cargo test -- --ignored`                                              |
| Go         | `if os.Getenv("IBM_API_KEY") == ""                                                                     |

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
   - IBM: `IBM_API_KEY`, `IBM_SERVICE_CRN`, and optionally `IBM_API_ENDPOINT` /
     `IBM_API_VERSION`
   - qBraid: `QBRAID_API_KEY` and `QBRAID_API_ENDPOINT`

The test runner output must clearly indicate which tests were skipped and why.

### 9.4 Test Execution is a Build Step

**Tests are not optional.** The build process is:

1. Implement module.
2. Write tests for that module.
3. Run tests. All must pass.
4. Only then proceed to the next module.

After all modules are complete: 5. Run the full integration test suite. 6. All
550+ tests must pass before the library is considered complete (excluding
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
- [ ] `prepareState()` and `initialize()` implemented and tested.
- [ ] `unitary()` (arbitrary unitary matrix) implemented and tested.
- [ ] `SimulatorBackend.execute()` returns correct distributions for all
      circuits.
- [ ] `getStateVector()` returns correct amplitudes.
- [ ] `blochSphere()` returns correct Bloch coordinates and purity radius.
- [ ] `Serializer` interface defined with `serialize` and `deserialize`.
- [ ] `OpenQASM3Serializer` fully implements serialize and deserialize.
- [ ] OpenQASM 3 serialization handles all gates, measurements, resets,
      barriers, delays, global phase, and control flow.
- [ ] OpenQASM 3 deserialization reconstructs circuits faithfully.
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
- [ ] Transpiler pipeline fully implemented: unroll, synthesis, layout (SABRE),
      routing (SABRE), translation (ZYZ, RZ+SX, KAK), optimization.
- [ ] ZYZ decomposition correctly decomposes any single-qubit unitary.
- [ ] RZ+SX decomposition correctly decomposes any single-qubit unitary.
- [ ] KAK/Weyl decomposition correctly decomposes any two-qubit unitary.
- [ ] SABRE layout and routing produce valid results for constrained coupling
      maps.
- [ ] Optimization passes reduce gate count (cancellation, merging, identity
      removal, commutation).
- [ ] IBM payload structure matches specification (program_id, backend,
      `params.version`, Sampler V2 PUB tuple).
- [ ] IBM executable contains API config with IAM auth flow, `Service-CRN`,
      `IBM-API-Version`, and route templates.
- [ ] qBraid payload structure matches specification (shots, deviceQrn,
      program).
- [ ] qBraid executable contains API config with X-API-KEY header, route
      templates, and nested `data` response parsing.
- [ ] Transpiled circuits produce equivalent measurement distributions to
      original circuits when simulated.
- [ ] IBM execution tests exist and are gated behind `IBM_API_KEY` and
      `IBM_SERVICE_CRN` env vars.
- [ ] qBraid execution tests exist and are gated behind `QBRAID_API_KEY` env
      var.
- [ ] Complex class has all methods, fully tested.
- [ ] Matrix class has all methods, fully tested.
- [ ] Gate matrices are all verified unitary.
- [ ] No external math/LA dependencies.
- [ ] 550+ tests passing (excluding skipped IBM and qBraid execution tests).
- [ ] All public API symbols exported from mod.{ext}.
- [ ] `README.md` exists with: installation, quick start (Bell state example),
      full public API reference for every exported symbol, 10 usage examples
      (GHZ, parameterized circuit, state vector, Bloch sphere, OpenQASM 3
      round-trip, transpilation, composition, control flow, custom unitary,
      Complex/Matrix algebra), test commands, and license.

---

## 11. What is NOT Tested (but IS Implemented)

The IBM backend (`IBMBackend`) and qBraid backend (`QBraidBackend`) are **fully
implemented** including transpilation, OpenQASM 3 serialization, payload
construction, job submission, polling, and result retrieval. However, the
**execution paths** (actual HTTP calls to the IBM and qBraid cloud APIs)
**cannot be tested without credentials**. These tests are gated behind
environment variables and are skipped by default:

- IBM: `IBM_API_KEY`, `IBM_SERVICE_CRN`, and optionally `IBM_API_ENDPOINT` /
  `IBM_API_VERSION`
- qBraid: `QBRAID_API_KEY` and `QBRAID_API_ENDPOINT`

Everything else — transpilation pipeline, payload construction, OpenQASM 3
serialization within both backend flows, coupling map handling — **is tested**
using mock configurations.

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
