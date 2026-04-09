## 1. Project Overview

Build a **complete, language-agnostic quantum computing SDK**. The SDK includes
a **real quantum simulation** using state-vector evolution over complex vector
spaces — no stubs, no approximations, no external quantum backends.

**Every operation must be a real computation.** Complex number arithmetic,
matrix algebra, tensor products, gate application via subspace iteration,
Born-rule sampling, and state collapse must all be implemented from scratch. Do
not shell out to numpy, BLAS, or any linear-algebra library. The quantum
computing SDK is self-contained.

**Completeness is mandatory.** The implementation MUST cover every item in this
specification in full: every gate in Tiers 0–14, every Expansion API method
covering the complete OpenQASM 3.1 surface, every transpilation stage, every
backend, every validation rule, and every test listed in Section 11. No
shortcuts, no simplifications, no omissions, no "TODO" stubs, no placeholder
implementations, no "good enough" approximations, no silently skipped features,
and no reduced test counts. If this document says a feature, gate, method,
decomposition, validation, or test exists, it MUST be implemented exactly as
specified and MUST pass its tests before the work is considered done. An
implementation that is missing any specified item — however minor — is
non-conforming, regardless of how well the rest of the SDK works. When in doubt,
implement the full specification rather than a simplified subset.

### What This SDK Does

The SDK is organized around a central `QuantumCircuit` builder class and a
pipeline that takes a circuit from construction to execution results:

```
QuantumCircuit  →  Transpiler  →  Backend  →  ExecutionResult (bitstring histogram)
```

1. **Circuit construction.** The user builds quantum programs declaratively
   through the `QuantumCircuit` class, which exposes one chainable method per
   gate (Section 3 defines the full gate catalog across Tiers 0–14) and one
   method per non-gate language feature (Section 5 defines the Expansion API
   covering the complete OpenQASM 3.1 surface). The circuit stores an ordered
   list of instructions, named classical registers, symbolic parameters, a
   scalar `globalPhase`, and all metadata required for faithful serialization.

2. **Transpilation.** A `Transpiler` interface with an `OpenQASMTranspiler`
   implementation converts the circuit into an OpenQASM 3.1 text program. The
   transpiler also includes a full compilation pipeline — unrolling, gate
   modifier expansion, SABRE layout/routing, basis-gate decomposition (ZYZ,
   RZ+SX, KAK/Weyl), and peephole optimization — so circuits can be adapted to
   hardware-constrained backends before serialization.

3. **Execution.** A `Backend` interface abstracts execution. Three
   implementations are provided:
   - `SimulatorBackend` — a noiseless state-vector simulator that runs the
     circuit locally using subspace iteration and Born-rule sampling.
   - `IBMBackend` — transpiles and serializes the circuit, then submits it to
     IBM Quantum via the Sampler V2 REST API with CRN-aware authentication.
   - `QBraidBackend` — transpiles and serializes the circuit, then submits it to
     the qBraid cloud API with device-capability discovery.

4. **Results.** Every backend returns an `ExecutionResult`: a map from
   bitstrings to percentages (0–100) summing to 100.

**Typical usage** (language-agnostic pseudocode):

```
qc = new QuantumCircuit()
qc.h(0).cx(0, 1).measure(0, 0).measure(1, 1)

sim = new SimulatorBackend()
result = sim.execute(qc, 1024)   // 1024 shots (optional, default)
print(result)                     // e.g. { "00": 50, "11": 50 }
```

### Strategy and Document Organization

The remaining sections of this specification define everything needed to build
the SDK:

- **Section 2 — Code Conventions** fixes language-agnostic rules for naming,
  immutability, qubit ordering, and the Phase Convention that governs all gate
  matrices, decompositions, and equality checks.
- **Section 3 — Gate Definitions** specifies the complete gate catalog (Tiers
  0–14, ~80 gates) plus a small set of exact ancilla-assisted synthesis
  templates. Standalone unitary gates are pure matrix-returning functions, and
  every exact construction bottoms out in Tier 0 zero-qubit/single-qubit
  primitives and CX. This is the mathematical core.
- **Section 4 — OpenQASM 3.1 Reference** provides a condensed specification
  checklist for every syntactic and semantic feature that the transpiler must
  handle.
- **Section 5 — Expansion API** defines the non-gate `QuantumCircuit` methods
  (declarations, expressions, control flow, calibration, pragmas, etc.) that
  give the builder complete OpenQASM 3.1 coverage.
- **Section 6 — IBM Backend** specifies the IBM Quantum REST protocol, payload
  structure, authentication flow, polling, and Sampler V2 result parsing.
- **Section 7 — qBraid Backend** specifies the qBraid REST protocol, device
  discovery, payload structure, status handling, and result parsing.
- **Section 8 — Implementation Plan** defines the exact file structure, classes,
  interfaces, and build order.
- **Section 9 — Simulator Engine** specifies the state-vector simulation
  algorithm.
- **Section 10 — Transpilation Pipeline** specifies each compilation stage.
- **Section 11 — Test Plan** defines the comprehensive test suite (minimum test
  counts per module, integration tests, and skip mechanisms for cloud backends).
- **Section 12 — Verification Checklist** provides a final pass/fail checklist
  for completeness.

### First Step: Ask for the Target Language

Before writing any code, **ask the user which programming language to use**. The
architecture described here is language-agnostic. Adapt idioms, project
structure, file extensions, test frameworks, and build tools to the chosen
language.

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

Every gate matrix, parameterized gate formula, decomposition rule,
controlled-gate construction, and equality check MUST follow one single phase
convention. Defining matrices alone is not sufficient unless the library also
defines how global phase, relative phase, parameter signs, and controlled
extensions are interpreted consistently across the entire system.

Reference basis for this section: OpenQASM 3.1 language specification and
standard gate library (`stdgates.inc`), used only to fix interoperable
mathematical semantics for phase-sensitive gates and identities. At the format
boundary, the source of truth is the explicit mathematical denotation given by,
in order: (1) the built-in gate and modifier definitions in the OpenQASM
language specification; (2) the mathematical standard-library gate definitions
as if expanded from `stdgates.inc`; and then only as non-authoritative
exposition, (3) surrounding prose examples, explanatory summaries, or rendered
state-action sketches. If an expository sentence, example, or rendering ever
differs from an explicit equation, matrix, modifier rule, or standard-library
gate definition, the explicit mathematical definition controls.

This section defines the library's **internal** phase semantics. It is
serializer-independent. Any external text format that uses the same gate names
with different global-phase choices must be translated at the format boundary
instead of changing the meaning of the core library APIs.

Reference notation in this section is exact. `Section 2` means the whole Code
Conventions chapter. `Phase Convention 1` through `Phase Convention 6` mean the
numbered subsections that begin at `1. Phase scalars` below. The unnumbered
rules above those numbered subsections are cited by their bold names (for
example, the **fold-safe same-scope rule**) rather than by subsection number.

Unless a later section explicitly says **textual OpenQASM** or quotes source
code from the OpenQASM language reference, gate names in this document refer to
the SDK's internal semantic objects, not raw OpenQASM tokens. The parser and
serializer form a format boundary. A textual OpenQASM gate crosses that boundary
unchanged if and only if, after expanding the OpenQASM 3.1 built-in or
standard-library definition under this section's semantics, the resulting
operator is exactly the same internal semantic object with zero expression-local
phase correction. Gates whose OpenQASM 3.1 phase convention differs from the
internal one MUST be translated phase-exactly at that boundary instead of
reinterpreting the core API. For avoidance of doubt, textual OpenQASM `rx`,
`ry`, `rz`, `p`, `phase`, `crx`, `cry`, `crz`, `cp`, `cphase`, `u1`, `sx`, and
`sxdg` are exact semantic identities under this internal convention and
therefore cross the format boundary unchanged except for ordinary syntactic
handling. Textual `U`, `cu`, `u2`, `u3`, and any other standard-library gate
whose OpenQASM 3.1 phase convention differs from the internal one do not.

Any required format-boundary translation is part of the syntactic gate
expression itself and MUST be applied before gate modifiers (`ctrl`, `negctrl`,
`inv`, `pow`), decomposition, equality-based simplification, or any
normalization that could move a relative phase across a control boundary. For
purposes of parsing, serialization, and modifier application, a gate expression
may carry an expression-local zero-qubit phase prefix `localPhase`. If the
underlying internal gate/operator is `G`, the unmodified translated expression
denotes `exp(i * localPhase) * G` on the same base arity before any outer
modifiers are applied. This expression-local prefix is semantically inside the
gate expression itself: it is neither a sibling `globalPhaseGate(...)`
instruction in the surrounding body nor the owning scope's scalar `globalPhase`.
Any parser/serializer IR that is source-preserving / exact-round-trip, or that
may later apply, infer, or expose outer gate-expression modifiers, MUST preserve
this distinction exactly, either with an explicit `localPhase` field on
gate-call nodes or by another representation that is provably equivalent under
this section's modifier rules. A normalized semantic IR that makes neither of
those promises MAY instead fold a `localPhase` into the immediate owning
ordinary scope's scalar `globalPhase`, but only when the correction is fold-safe
and the representation also satisfies the recoverability rule stated below: it
either commits that the instruction will remain a
`bare, unmodified gate instruction` for the rest of that representation's
lifetime, or it retains enough instruction-local provenance to reconstruct the
exact same correction on that same instruction deterministically. Otherwise it
too MUST preserve this distinction exactly. Unless a later subsection proves a
more specific identity, a modifier stack `m1 @ m2 @ ... @ G` denotes exact
right-nested function application on that full corrected operator:
`m1(m2(...(exp(i * localPhase) * G)))`.

Textual OpenQASM 3.1 has no generic surface syntax for an arbitrary
gate-expression-local zero-qubit phase prefix at one call site separate from the
chosen gate family itself. Therefore a serializer that targets textual OpenQASM
and starts from a semantic gate call carrying `localPhase = delta` MUST realize
that correction only by the following exact mechanisms, used individually or,
where this section later explicitly permits, in exact combination:

- emit a textual gate family whose language-defined semantics already include
  the needed instruction-local phase inside the same gate expression and under
  the same modifier stack, using the exact family-specific offset rules stated
  later in this section (for example textual `U`, `u2`, or `u3`; textual `cu` is
  covered separately and only for its documented control-subspace relative-
  phase role, not as a generic whole-expression scalar-phase channel);
- if and only if the call is a `bare, unmodified gate instruction` and the
  still-unencoded correction satisfies the fold-safe same-scope rule specified
  in this Phase Convention, including the recoverability and no-ancestor-hoist
  clauses, move some or all of that still-unencoded correction into the
  immediate owning ordinary scope's scalar/global-phase representation and then
  lower that same- scope phase as textual `gphase(...)` or another exactly
  equivalent same-scope encoding.
- when the active serializer/API contract permits introducing fresh
  declarations, synthesize an exact helper gate definition or another equally
  exact target-language helper abstraction whose own denotation already carries
  the needed zero-qubit phase before any outer modifiers are applied to the
  emitted helper call. In textual OpenQASM 3.1, this means a fresh parameterized
  `gate` definition at global scope, built only from exact encodings already
  permitted by this section, and then called with parameters and operands such
  that the emitted call itself carries zero residual instruction-local phase.

When helper-gate synthesis is available under the active serializer/API
contract, it is a first-class exact serialization mechanism rather than a mere
optimization fallback. Whenever direct call-site emission through a textual gate
family is not exact and the remaining correction cannot be discharged by the
permitted same-scope fold-safe conversion, the serializer MUST either synthesize
such a helper exactly or fail deterministically; it MUST NOT silently emit a
non-equivalent direct call, discard residual phase, or rely on a backend to
"know what was meant". Any synthesized helper gate MUST use a fresh
deterministic name for a fixed input IR and serializer configuration, MUST avoid
collision with all user declarations and any documented implementation-reserved
namespace in the emitted program, MUST be built only from exact encodings
already permitted by this section, MUST place the required phase semantically
inside the helper operand before any outer modifiers applied at the helper call
site, and MUST ensure that the emitted helper call itself carries zero residual
instruction-local phase. A serializer/API that promises source-preserving or
exact round-trip behavior MUST NOT introduce such helpers unless that contract
explicitly permits generated declarations and specifies how their presence is
represented to callers.

The family-specific encodings above do **not** create a generic extra channel
for arbitrary instruction-local scalar phase. They contribute only the fixed
phase correction or family-defined relative phase explicitly documented for that
textual family. A serializer MAY use such a family only when the full required
correction is realized exactly by that family's own semantics together with any
same-scope fold-safe compensation explicitly permitted here. Likewise, helper-
gate synthesis is a separate declaration-level encoding mechanism; it does
**not** mean that the chosen textual family itself carried an extra scalar-
phase channel. Any residual instruction-local scalar phase that would remain
inside the emitted gate expression after all permitted family offsets,
same-scope fold-safe compensation, and any permitted helper encoding are applied
is not encodable and requires serialization failure.

Whenever a later rule in this section requires a residual instruction-local
phase to be "exactly zero", that means that for the residual angle expression
`sigma`, the scalar `exp(i * sigma)` is exactly 1 under the active
representation's documented exact-expression semantics. Implementations MAY
establish this either by proving `sigma = 0` exactly or by proving
`sigma = 2*pi*n` for some exact integer-valued `n`; floating-point rounding,
epsilon tests, and heuristic algebra are forbidden. Implementations MAY
conservatively fail to prove zero and reject serialization rather than guess.

For interoperability, every public representation/API that relies on an exact
proof obligation in this section MUST document an exact-expression semantics,
and every conforming implementation MUST support at least the following minimum
exact-expression conformance profile:

- exact retention of immutable symbols, integer literals, any finite decimal
  literals that the active frontend syntax documents as exact rational values
  (for example `2.0` as exact 2), the constant `pi`, and exact rational
  constants, including those produced by exact constant folding;
- exact syntax and evaluation for unary sign, addition, subtraction,
  parenthesization, and multiplication or division by a nonzero exact integer or
  rational constant;
- deterministic normalization of affine combinations of immutable symbols and
  `pi` with exact rational coefficients using exact integer/rational arithmetic;
- exact proofs, within that normalized profile, of literal zero, normalized-
  expression equality, membership in the set `2*pi*n` for exact integer-valued
  `n`, and integer-valuedness when an expression normalizes to an integer
  literal or to an immutable symbol documented as integer-valued under the
  active type/range contract.

- Portable conformance tests and cross-implementation guarantees may rely only
  on the minimum exact-expression conformance profile above. Outside that
  minimum, two conforming implementations may differ only in whether they
  support richer exact algebra or conservatively reject a proof obligation;
  neither may guess from floating-point evaluation or heuristics.

Implementations MAY support richer exact algebra. Outside their documented
exact-expression profile, they MAY conservatively fail a proof obligation,
preserve syntax unresolved, or reject serialization/execution, but they MUST do
so deterministically and MUST NOT guess from floating-point evaluation, epsilon
tests, heuristic simplification, or undocumented backend behavior.

A serializer MUST NOT extract a `localPhase` from inside `ctrl`, `negctrl`,
`inv`, `pow`, or any other modifier stack by emitting a sibling `gphase(...)`
unless a later subsection separately proves that rewrite exact for that specific
operand. In particular, no generic such extraction rule is available for
principal-branch non-integer `pow(k)`. This prohibition does **not** forbid an
exact helper-gate encoding that keeps the phase inside the helper operand being
modified. If no exact textual encoding or permitted helper encoding is
available, serialization MUST fail rather than emit non-equivalent OpenQASM.

This section distinguishes **semantic identity** from **surface-form identity**.
A representation or API that does not promise source-preserving or exact
round-trip behavior MAY canonicalize surface spellings at the format boundary,
subject to the exact phase rules below. A representation or API that **does**
promise source-preserving or exact round-trip behavior MUST retain enough
syntax-local metadata to re-emit the same gate-family spelling,
operand/reference forms, modifier stack, parameter-expression trees, and the
same documented gate-expression-local distinctions at the same source location.
In this document, source-preserving / exact-round-trip means exact preservation
of the parsed gate-expression structure and other syntax-local distinctions that
the relevant API exposes; it does **not** by itself require byte-for-byte
whitespace fidelity unless a later API explicitly promises that stronger
guarantee. Preserving only an internal semantic gate plus `localPhase` is
insufficient for such APIs when multiple textual spellings share semantics or
differ only by a format-boundary phase correction, for example `U` versus `u3`,
`p` versus `phase`, or `cp` versus `cphase`.

Terminology used below:

- scope scalar `globalPhase` means the scalar phase field owned by one ordinary
  scope;
- explicit zero-qubit phase instruction/gate `globalPhaseGate(theta)` means the
  first-class semantic 0-qubit phase operation; the SDK constructor/function
  spelling `GlobalPhaseGate(theta)` and the textual OpenQASM spelling
  `gphase(theta)` are concrete surface forms of this same semantic instruction;
- expression-local `localPhase` is neither of the two forms above: it prefixes
  exactly one gate expression and remains semantically inside that expression
  unless the fold-safe rule below later allows conversion into that expression's
  immediate owning scope scalar.
- a **bare, unmodified gate instruction** means a gate instruction whose gate
  expression has no outer modifier nodes in the current representation. It
  denotes exactly its base gate/operator together with any instruction-local
  `localPhase`, and not an outer `ctrl`, `negctrl`, `inv`, `pow`, or any other
  gate-expression transform.

If a gate call is broadcast over registers, broadcasting denotes the width-`k`
family of scalar gate expressions obtained by zipping the register positions,
one lane per broadcast position, before any fold-safe conversion of `localPhase`
into scope scalar phase is considered. By using broadcast syntax, the program
asserts the expanded lane calls satisfy the OpenQASM 3.1 commutation
requirement. Implementations MAY choose a deterministic canonical lane order for
in-memory expansion, testing, serialization, or rewriting, but that
representational order is not semantically significant. Each expanded lane
therefore carries its own copy of the original gate expression's `localPhase`
and modifiers. In particular, a translated textual OpenQASM `U`/`u2`/`u3` used
in a width-`k` broadcast contributes its required local phase correction `k`
times, once per expanded lane, not once per syntactic broadcast statement.

A zero-qubit phase correction is **fold-safe** in a scope if and only if all of
the following hold:

- It belongs to the immediate scope, not to any ancestor or descendant scope.
- It is either an explicit `globalPhaseGate(theta)` instruction in that scope or
  the `localPhase` prefix of a `bare, unmodified gate instruction` in that
  scope.
- During one dynamic execution of that scope, its total contribution to that
  same scope's scalar phase is the same exact scalar phase on every dynamic path
  through that same scope. Implementations MAY establish this either by
  syntactic identity of the resulting phase expressions or by an exact proof
  that any two path totals differ by an exact integer multiple of `2*pi` under
  the active exact-expression semantics. This `2*pi*n` equivalence is a
  denotational property of phase scalars
  (`exp(i * alpha) = exp(i * (alpha +
  2*pi*n))`) and applies only to the
  fold-safety proof; it does **not** permit rewriting the phase expression
  itself modulo `2*pi` in the stored representation, which remains prohibited by
  Phase Convention 1 unless the rewrite is proved exact under the current
  operator semantics. This is a semantic path condition, not a mere syntactic
  test: the correction MUST NOT be skipped, repeated a path-dependent number of
  times, or otherwise altered by branches, loops, `break`, `continue`, `return`,
  `end`, or any other reachable control transfer that changes that total
  contribution. Executing exactly once on every path is the common special case.
- Its phase expression is **entry-invariant**: every referenced symbol is
  immutable and already bound at the scope entry, so the expression does not
  depend on loop indices, values first created inside the scope body after
  entry, measurement results, or any other path-dependent runtime data.
- Folding it into the scalar `globalPhase` of that same scope loses no
  **locality-sensitive meaning** that the current representation or transform
  has committed to preserve. Locality-sensitive meaning includes annotations,
  timing, calibration attachment, modifier structure, control structure,
  source-span or exact round-trip-serialization identity, and any other frontend
  metadata attached specifically to that instruction or expression. In a
  normalized non-source-preserving IR, the original instruction position of an
  otherwise anonymous fold-safe zero-qubit phase need not itself be preserved;
  in a source-preserving or round-trip IR, it MUST be preserved by keeping the
  phase explicit. For conformance, every public representation or documented
  pass that can contain zero-qubit phase MUST state whether it is
  source-preserving / round-trip or normalized semantic. A source-preserving /
  round-trip representation MUST treat source-local gate-family identity,
  modifier structure, explicit instruction boundaries, and any documented
  round-trip distinctions as locality-sensitive meaning. A normalized semantic
  representation MAY fold only when it still preserves the denotational
  semantics plus whatever metadata that API explicitly promises to preserve.

The recoverability rule referenced above is as follows: folding the `localPhase`
of a `bare, unmodified gate instruction` into the owning scope scalar is
semantics-preserving only for a representation that either commits that the
instruction will remain a `bare, unmodified gate
instruction` for the rest of
that representation's lifetime, or retains enough instruction-local provenance
to reconstruct the exact same correction on that same instruction
deterministically. Any representation that may later apply, infer, or expose
`ctrl`, `negctrl`, `inv`, `pow`, or any other gate-expression transform whose
semantics depend on the exact operand operator MUST preserve that correction in
an instruction-local losslessly recoverable form from the moment the translated
call is created; it MUST NOT rely on later recovering it from scope scalar phase
alone. A representation that has already folded such a `localPhase` without that
guarantee MUST treat later transforms on that instruction as unavailable rather
than silently changing semantics.

For conformance, "enough instruction-local provenance" and "losslessly
recoverable form" mean enough data attached to, or directly linked from, that
same gate instruction to reconstruct without consulting surrounding scope scalar
phase all of the following that the relevant API promises to preserve: the same
operand operator/gate family, parameter expressions, modifier stack, exact
instruction-local phase correction, and any source-local identity or metadata
that would otherwise be lost by folding. Information recoverable only from a
surrounding accumulated `globalPhase` does **not** satisfy this requirement.

Only a fold-safe zero-qubit phase correction MAY be folded into the immediate
owning scope's scalar `globalPhase`. Such a correction MUST NOT be hoisted
across the boundary of its immediate owning scope into any ancestor scope that
still exists in the current representation. The denotational convention that a
scope scalar `globalPhase` acts at scope entry does not by itself make such
folding illegal. Implementations MAY use any proof procedure for fold-safety,
but the proof must establish the semantic condition above; branch-free syntax
alone is not sufficient when reachable early exits or equivalent control
transfers can bypass the correction.

Later references in this section to the **fold-safe same-scope rule** mean the
conjunction of all three requirements in this subsection: the fold-safety
conditions above, the recoverability rule, and the no-ancestor-hoist
restriction. Satisfying only one subset of those clauses is insufficient. This
paragraph is the **single canonical definition** of the fold-safe same-scope
rule. All later references in this section and in subsequent sections cite it by
name rather than restating it; if a later restatement ever conflicts with this
definition, this definition controls.

After a semantics-preserving structural transform that removes, replaces, or
flattens ordinary-scope boundaries (for example gate inlining, subroutine
inlining, or block flattening), the "immediate owning scope" of any surviving
explicit or instruction-local zero-qubit phase is the ordinary scope that
contains that phase in the transformed IR. The ancestor-hoisting prohibition and
fold-safe proof obligation are therefore judged against current IR ownership,
not pre-transform source nesting, unless the active representation/API promises
source-preserving / round-trip retention of the original scope structure. A
source-preserving / round-trip representation MUST retain the original
containment relation until it intentionally transitions to a normalized semantic
representation.

This specification does **not** require one unique canonical in-memory normal
form for fold-safe zero-qubit phase. A compliant implementation MAY leave a
provably fold-safe correction explicit. If an implementation exposes or
documents a pass as a canonical normalization pass, that pass MUST choose and
apply a deterministic fold-safe proof and rewriting strategy for a fixed input
IR. Tests in this document MUST accept either explicit or folded representations
unless a tested API explicitly promises canonical normalization.

Every public API that returns, mutates, or documents an IR containing zero-qubit
phase MUST state whether callers should expect source-preserving / round-trip
behavior or normalized semantic behavior. Compliance for fold-safe phase
handling is judged against that documented contract, not against an unstated
internal implementation choice.

**1. Phase scalars**

- A phase angle `alpha` always denotes the complex scalar `exp(i * alpha)`.
- Every phase angle or phase expression in this section, including `localPhase`,
  `globalPhase`, and symbols such as `theta`, `phi`, `lambda`, `gamma`, `delta`,
  and `alpha`, denotes a real-valued angle in radians unless a later section
  explicitly states otherwise.
- For numeric branch-sensitive algorithms, reduce a phase to the canonical
  representative `wrapPhase(alpha) in (-pi, pi]` by defining
  `wrapPhase(alpha) = alpha - 2*pi * floor((alpha + pi) / (2*pi))`, and if that
  value is exactly `-pi`, return `pi` instead.
- Implementations must treat `wrapPhase` as a mathematical definition, not as a
  language `%`/remainder operation whose behavior for negative inputs may vary.
- Floating-point implementations MUST apply a deterministic branch-cut snap
  after evaluating that definition numerically: if the computed representative
  `w` satisfies `|w - pi| <= epsilon` or `|w + pi| <= epsilon`, return `pi`;
  otherwise return `w`. This avoids spurious `-pi`/`pi` disagreement and makes
  numeric branch-sensitive algorithms deterministic near the cut.
- Symbolic phase expressions are stored exactly. Do **not** reduce symbolic
  expressions modulo `2*pi` unless the implementation can prove the rewrite
  exactly.
- The same prohibition applies to gate parameters whenever this section treats
  the resulting operator exactly rather than modulo global phase.
  Implementations MUST NOT silently rewrite a parameter such as `theta` to
  `wrapPhase(theta)` or otherwise reduce angles modulo `2*pi` for `RX`, `RY`,
  `RZ`, canonical `U`, controlled gates, or decompositions unless the rewrite is
  proved exact under the current operator semantics.

**2. State, matrix, and equality semantics**

- Any standalone gate matrix defined by this SDK, any explicit operator on `n`
  qubits whose unitarity has already been established under this section, and
  any scope/region whose concrete denotation has already been established to be
  unitary on `n` qubits, denotes a `2^n x 2^n` unitary over `Complex`. A
  concrete numeric matrix or other explicit operator object whose unitarity has
  not yet been established is not implicitly promoted to a unitary by mere
  existence; it becomes a known unitary only by the exact-construction,
  exact-proof, or numeric-certification rules stated below.
- A scope/region counts as a **unitary-only scope** for purposes of this section
  only when both of the following hold: its dynamic quantum effect is unitary,
  and the current representation has established a concrete operator denotation
  for that whole region on its qubits. A syntactically measurement-free region
  that still carries unresolved exact syntax with no concrete operator
  denotation yet (for example an unresolved `pow(k)` whose exponent
  classification or non-integer power rule is still pending) is not yet a
  unitary-only scope under these rules, even though later exact proof or
  parameter binding may resolve it to one.
- A general `QuantumCircuit` or `SubroutineDefinition` may contain measurement,
  reset, or classical control and therefore need not denote a single unitary;
  such scopes are interpreted operationally over quantum state plus classical
  memory. Whenever this section refers to a "body unitary" or compares operators
  up to global phase, it applies only to unitary regions/scopes.
- In the operational semantics of a non-unitary region, pure quantum states that
  differ only by a scalar phase `exp(i * alpha)` are observationally equivalent:
  they induce the same Born-rule probabilities, the same surviving
  post-measurement branch states up to that same scalar phase, the same reset
  behavior, and therefore the same subsequent classical-control behavior driven
  only by sampled outcomes. This runtime phase-insensitivity does **not** permit
  moving an instruction-local `localPhase`, an explicit `globalPhaseGate(...)`,
  or a scope scalar `globalPhase` across a modifier boundary, a control
  boundary, or a scope boundary except where the exact fold-safe and
  controlled-lifting rules in this section explicitly allow it.
- `Matrix.equals` and ordinary gate/unitary equality are entrywise approximate
  comparisons with epsilon `1e-10`; they do **not** quotient by global phase.
  Entrywise approximate equality means: the compared matrices must have the same
  shape, and for every corresponding entry `a`, `b`, the complex-magnitude
  difference must satisfy `|a - b| <= epsilon`, where
  `|x + i*y| = sqrt(x^2 +
  y^2)`. Every later reference in this section to
  "entrywise", "`A ~= B`", or the "entrywise comparison semantics as
  `Matrix.equals`" refers to this exact rule.
- The `1e-10` epsilon is the final acceptance threshold for approximate semantic
  checks, numeric unitary certification, and numeric phase extraction defined in
  this section. Implementations MAY use tighter internal thresholds, additional
  guard bands, or more numerically stable algorithms internally, but they MUST
  NOT accept a result that fails the final `1e-10` checks prescribed here, and
  MAY conservatively reject borderline numeric cases rather than guessing.
- For purposes of the numeric fallback below, a matrix/operator is a "known
  unitary" only when the implementation has established that fact by exact
  construction, exact symbolic proof, or the explicit numeric certification
  procedure defined here. The status MUST be determined as follows: every
  standalone gate matrix and every already-established unitary-only scope
  denotation defined by this SDK is a known unitary; multiplying a known unitary
  by an exact scalar phase `exp(i * alpha)` on the same arity, including a gate
  expression's instruction-local `localPhase` correction, preserves
  known-unitary status; exact one-control lifting of a known unitary preserves
  known-unitary status, and therefore any `ctrl^n(W)` formed by Phase Convention
  §4's repeated exact lifting, as well as any mixed positive/negative control
  chain formed by that same rule together with the required X-conjugations for
  negative controls, remains a known unitary; tensor products, matrix products,
  adjoints/inverses, and integer powers of known unitaries remain known
  unitaries; a non-integer power of a known unitary is a known unitary only when
  the current implementation has established that specific result either by a
  mandatory exact rule from Phase Convention 6 or by successfully completing an
  optional Phase Convention 6 spectral-power certification for that specific
  operand, and merely being an operator form that some implementation could in
  principle support is insufficient; any other symbolic operator is a known
  unitary only by exact symbolic proof; and any other concrete numeric matrix
  becomes a known unitary only by verifying both `M† * M ~= I` and `M * M† ~= I`
  entrywise with the same `1e-10` epsilon used by `Matrix.equals`.
  Implementations MUST NOT treat an operator as known-unitary by heuristic
  pattern matching or informal expectation. Portable conformance tests and
  generic rewrites MUST NOT rely on optional Phase Convention 6 non-integer
  power support other than the mandatory zero-qubit phase rule unless a tested
  API explicitly promises that support.
- Algorithms that intentionally compare operators up to global phase must do so
  explicitly: first require that `A` and `B` have the same shape, then choose a
  deterministic pivot `(r, c)` as the lexicographically first entry maximizing
  `min(|A[r,c]|, |B[r,c]|)`.
- If the chosen pivot has `min(|A[r,c]|, |B[r,c]|) > epsilon`, let `a = A[r,c]`,
  `b = B[r,c]`, and compute `rho = a / b`. Require `| |rho| - 1 | <= epsilon`,
  and then verify `A ~= rho * B` entrywise using the same `1e-10` epsilon and
  entrywise comparison semantics as `Matrix.equals`. The extracted numeric phase
  is `wrapPhase(arg(rho))`.
- If the chosen pivot has `min(|A[r,c]|, |B[r,c]|) <= epsilon`, implementations
  MUST treat the pivot method as numerically unstable and MUST NOT divide by
  that pivot. If `A` or `B` are concrete numeric matrices whose known-unitary
  status has not yet been established, implementations MUST first run the
  numeric unitary certification procedure above. When `A` and `B` are known
  unitaries of common dimension `d`, implementations MUST use the
  unitary-specific fallback `rho = trace(A * B†) / d`, then require
  `| |rho| - 1 | <= epsilon` and verify `A ~= rho * B` entrywise using the same
  `1e-10` epsilon and entrywise comparison semantics as `Matrix.equals`. When
  `A` and `B` are not known unitaries, the comparison MUST fail rather than
  guessing a phase from unstable data. For the gate matrices and decompositions
  used by this SDK, a stable pivot is expected in normal operation.
- The pivot and trace procedures above are numeric algorithms. They are defined
  only when the compared entries and traces can be evaluated numerically under
  the same branch convention.
- For symbolic operators, implementations MAY additionally support comparison up
  to global phase only by an exact symbolic proof of common shape and
  `A = exp(i * alpha) * B`. If no such exact proof procedure is available, the
  comparison MUST fail or report "unsupported/unknown"; it MUST NOT partially
  evaluate symbolic expressions numerically, guess a branch, or silently fall
  back to approximate mixed symbolic-numeric reasoning.
- When a decomposition returns `{ globalPhase, remainder }`, it means
  `M = exp(i * globalPhase) * remainder` exactly in the symbolic case, or within
  epsilon in the numeric case. If `globalPhase` is numeric, the returned value
  MUST already be canonicalized with `wrapPhase` using the same epsilon snapping
  rule. If `globalPhase` is symbolic, it MUST be stored exactly and MUST NOT be
  reduced modulo `2*pi` unless the implementation can prove that rewrite
  exactly.

**3. Canonical meaning of scope global phase**

- The notion of an **ordinary scope** in this subsection is an internal
  denotational device of this SDK. It does **not** claim that every textual
  OpenQASM construct stores a literal scalar `globalPhase` field in source. It
  means that whenever this SDK represents or interprets such a body as an
  ordinary scope, the body's denotation is exactly the same as if it owned such
  a scalar phase field with the rules stated below.
- In this subsection, an **ordinary scope** means any ordered instruction body
  with its own entry point and dynamic execution region whose quantum denotation
  is defined by this SDK: `QuantumCircuit`, `GateDefinition`,
  `SubroutineDefinition`, and every nested non-calibration body introduced by
  control flow, boxing, or similar frontend constructs.
- Calibration scopes (`cal`, `defcal`, and companion-grammar bodies) are
  preserved by the frontend. A loaded calibration grammar counts as having
  explicitly opted into the same denotational rule only if the implementation
  has an explicit implementation-visible contract for that grammar saying so,
  and that contract states all of the following for that grammar: it defines a
  scope-owned scalar phase with the same scope-entry semantics as this
  subsection; it defines the calibration scope's qubit arity against which a
  zero-qubit phase lifts; and it defines when explicit zero-qubit phase may be
  folded, hoisted, or kept instruction-local. This contract may be provided by
  the grammar specification itself, by frontend metadata bundled with that
  grammar, or by another equally explicit mechanism documented by the API, but
  selecting `defcalgrammar "name"` alone never implies the opt-in. Unless such
  an explicit contract is present, implementations MUST treat explicit
  zero-qubit phase there as locality-sensitive and MUST NOT infer, fold, or
  hoist a scope scalar `globalPhase` inside that calibration scope.
- Absent that explicit opt-in contract, zero-qubit phase inside a calibration
  scope is an opaque calibration-grammar construct whose meaning is defined only
  by that calibration grammar or by a backend/frontend contract that explicitly
  interprets it. The ordinary-scope rules of this subsection do not apply there
  by default: generic passes outside the calibration subsystem MUST preserve the
  construct exactly when source-preserving behavior is promised, and any pass
  that requires a concrete matrix/operator, an ordinary-scope scalar
  `globalPhase`, or generic unitary reasoning inside that calibration scope MUST
  reject or defer rather than inventing semantics.
- Every ordinary scope owns a scalar `globalPhase`. An implementation MAY
  physically omit storing the scalar when its exact value is zero, but the
  semantics are as if it were present.
- Any stored scalar `globalPhase` of an ordinary scope MUST itself be valid at
  that scope's entry. Every referenced symbol must be immutable and already
  bound at that scope's entry under the active language/frontend contract. A
  scope scalar `globalPhase` MUST NOT depend on a loop index, a measurement
  result, a branch-local value first created inside that same scope after entry,
  or any other path-dependent runtime data. If a frontend cannot satisfy this
  rule, it MUST keep the phase as an explicit `globalPhaseGate(...)` instruction
  or another equally explicit locality-preserving representation instead of
  storing it in the scope scalar.
- For an ordinary scope `S`, let `scopeQubitArity(S)` be the number of distinct
  underlying quantum wires that are semantically in quantum scope at entry to
  `S`, after resolving aliases, slices, views, captures, repeated names, and any
  frontend-specific operand forms to their underlying wires. This is a semantic
  wire count, not a syntactic identifier count, and not the cardinality of an
  ambient device namespace or of all physical-qubit literals that might be
  syntactically spellable in that frontend.
- Every frontend that can create, parse, serialize, or transform an ordinary
  scope MUST define exactly, as part of its documented scoping semantics, which
  underlying quantum wires are semantically visible at that scope's entry. If a
  frontend does not define this exactly, it is not conformant for any transform,
  equality check, serialization rule, or denotational claim that depends on
  `scopeQubitArity(S)` and MUST reject or defer those operations rather than
  guess.
- For a `GateDefinition`, `scopeQubitArity(S)` is the number of distinct quantum
  wires semantically visible at gate-body entry under the gate-body language
  rules; for the core OpenQASM gate grammar this is exactly the distinct qubit-
  parameter wires after alias resolution. For a `SubroutineDefinition`, it is
  the number of distinct quantum wires semantically visible at subroutine-body
  entry under that subroutine language/frontend contract; in a frontend that
  follows the core OpenQASM scoping rules, this includes the distinct qubit-
  parameter wires after alias resolution together with any physical/hardware-
  qubit wires already semantically available in that body at entry under those
  same rules. For the OpenQASM 3.1 frontend specifically: physical-qubit
  literals such as `$0`, `$1` are globally accessible names, but a subroutine
  body's entry arity includes only those physical qubits that are explicitly
  passed as qubit-parameter arguments or that the active frontend contract
  documents as unconditionally in scope at that body's entry. A physical-qubit
  literal that is merely referenced inside the body (not passed as a parameter
  and not documented as unconditionally in scope at entry) does **not** enlarge
  `scopeQubitArity(S)` at entry; it takes effect only from the point of first
  use, analogously to a later local qubit declaration, and zero-qubit phase
  remains a scalar multiple of identity on any additional wires. If the
  implementation's frontend contract does not specify which physical qubits are
  in scope at subroutine entry, the implementation MUST conservatively treat
  `scopeQubitArity(S)` as unknown for any transform that depends on it and
  reject or defer rather than guess. For a `QuantumCircuit`, it is the number of
  distinct circuit qubit wires. For a nested ordinary body, it is the number of
  distinct underlying quantum wires visible in that body at entry, including any
  inherited wires and any physical-qubit references or equivalent frontend forms
  that are already semantically available in that body at entry under the active
  scoping rules. Mere syntactic legality of writing a physical-qubit literal in
  that scope does **not** by itself place every device qubit in scope. A body
  does **not** reduce its arity merely by mentioning only a subset of the
  quantum wires visible in that scope, and does **not** increase its entry arity
  merely because additional wires exist elsewhere on the target device or could
  be introduced later in the body.
- Later local qubit declarations inside that same ordinary body do **not**
  retroactively change `scopeQubitArity(S)`. This is exact because a zero-qubit
  phase is a scalar multiple of identity: if additional local wires are
  introduced after scope entry, then `exp(i * theta) * I_(2^m)` tensored with
  identity on those later-created wires is still the same scalar phase on the
  enlarged space.
- An ordinary scope may carry zero-qubit phase in either or both of the
  following representation forms: as the owning scalar `globalPhase`, or as an
  explicit `globalPhaseGate(theta)` instruction in the body.
- The representation forms above concern phase owned by an ordinary scope or by
  a standalone zero-qubit instruction in that ordinary scope. They do **not**
  include an expression-local `localPhase` prefix attached to a single gate
  expression per the format-boundary rule above; that prefix belongs only to
  that expression and is not itself a scope-wide representation form.
- A `localPhase` prefix may normalize into scalar phase only by the fold-safe
  rule above and only by folding it into the immediate owning ordinary scope's
  scalar `globalPhase`; it MUST NOT be hoisted into any ancestor scope or moved
  across the boundary of the body that owns the gate expression.
- An ordinary scope with scalar phase `gamma` denotes the same quantum effect as
  executing `globalPhaseGate(gamma)` exactly once at scope entry, followed by
  the scope body.
- If the scope body is unitary `V`, the combined denotation is
  `exp(i * gamma) * V`.
- `globalPhaseGate(theta)` is the zero-qubit phase gate with 1x1 matrix
  `[[exp(i * theta)]]`. In an ordinary scope `S` with `scopeQubitArity(S) = m`,
  it acts as `exp(i * theta) * I_(2^m)`.
- Ordinary-scope composition adds phase expressions. Ordinary-scope inversion
  negates the phase expression whenever the scope admits a well-defined inverse;
  non-unitary scopes are not invertible unless a separate API defines inverse
  semantics explicitly.
- In a normalized in-memory representation for an ordinary scope, only fold-safe
  zero-qubit phase may be absorbed into the owning scope's scalar `globalPhase`;
  otherwise the explicit `globalPhaseGate(...)` instruction or instruction-local
  `localPhase` prefix MUST remain explicit.
- An explicit zero-qubit phase instruction is a first-class instruction form. It
  must remain an instruction whenever the active representation preserves
  source-local identity or other locality-sensitive meaning, when the scope is a
  calibration scope that has not explicitly opted into scalar-phase folding, or
  when the implementation cannot prove fold-safety under this section.

**4. Controlled lifting rule**

- The controlled version of an `m`-qubit gate `W` is defined from the **full**
  matrix of `W`, not from an equivalence class modulo global phase.
- When the operand gate expression denotes `exp(i * delta) * W` because of an
  expression-local `localPhase` prefix or any other exact instruction-local
  scalar phase correction, the exact matrix used for controlled lifting is that
  full corrected operator `exp(i * delta) * W`, not `W` alone.
- For `n >= 1` positive controls, `ctrl(n)(W)` or `ctrl^n(W)` is defined by
  repeated exact one-control lifting of that same full corrected operand matrix.
  Equivalently, on `n` control qubits and `m` target qubits ordered controls
  first then targets, it acts as identity on every control basis state except
  the fully enabled all-1 control subspace, where that same exact corrected
  operator acts on the targets.
- For one positive control, `ctrl(W)` is block diagonal: `diag(I_(2^m), W)`
  where `W` here denotes the exact operand matrix being lifted, including any
  instruction-local scalar phase correction already attached to it.
- If `W = exp(i * alpha) * V`, then `ctrl(W) = diag(I, exp(i * alpha) * V)`. The
  scalar `exp(i * alpha)` is now a relative phase on the fully enabled control
  subspace and must **not** be discarded.
- If `W` has arity 0, `ctrl^n(W)` acts on the `n` control qubits only; there are
  no target operands. In particular, `ctrl(globalPhaseGate(theta)) = P(theta)`,
  and more generally `ctrl^n(globalPhaseGate(theta))` is the `n`-controlled
  phase gate that applies `exp(i * theta)` exactly on the fully enabled control
  subspace.
- More generally, controlling a zero-qubit phase gate yields the corresponding
  controlled phase gate under the same exact-matrix rule.
- Negative controls are defined by conjugating the selected control qubits with
  X before and after the exact positive-control construction. Mixed
  positive/negative control chains follow the same nested exact-matrix rule in
  modifier order.
- Modifier nesting is right-associative on the gate expression, and each control
  modifier prepends its own control arguments to the current argument list. For
  example, `negctrl @ ctrl @ X a, b, t` means: first form `ctrl @ X` on
  `(b, t)`, then add the outer negative control `a`; the enabled subspace is
  therefore `a = 0, b = 1`. By contrast, `ctrl @ negctrl @ X a, b, t` enables on
  `a = 1, b = 0`.
- Consequently, controlled gates must be synthesized from the exact matrix of
  the uncontrolled gate, or from a decomposition that preserves the enabled-
  subspace phase exactly.

**5. Canonical single-qubit phase-sensitive identities**

- `RX(theta) = exp(-i * theta * X / 2)`, `RY(theta) = exp(-i * theta * Y / 2)`,
  `RZ(theta) = exp(-i * theta * Z / 2)`.
- `R(theta, phi) = exp(-i * theta * (cos(phi) * X + sin(phi) * Y) / 2)`.
- `RV(vx, vy, vz) = exp(-i * (vx * X + vy * Y + vz * Z) / 2)`, with
  `RV(0, 0, 0) = I` exactly.
- `P(lambda) = diag(1, exp(i * lambda)) = exp(i * lambda / 2) * RZ(lambda)`.
- `S = P(pi/2)`, `T = P(pi/4)`, `Z = P(pi)`, `Sdg = S†`, `Tdg = T†`.
- The canonical square root of X is
  `SX = exp(i * pi / 4) * RX(pi/2) = (1/2) * [[1+i, 1-i], [1-i, 1+i]]`. Its
  adjoint is
  `SXdg = SX† = exp(-i * pi / 4) * RX(-pi/2) = (1/2) * [[1-i, 1+i], [1+i, 1-i]]`.
  Consequently `SX * SX = X` and `SX * SXdg = I`. For implementations that
  support the optional Phase Convention 6 spectral-power rule, the principal-
  branch power `X^{1/2}` (i.e. `pow(1/2) @ x`) equals `SX` exactly: `X` has
  eigenvalues `+1` and `-1` with eigenphases `0` and `pi` respectively under the
  `(-pi, pi]` branch, so `X^{1/2} = diag(exp(0), exp(i*pi/2))` in the
  eigenbasis, which equals `SX` after basis transformation. Implementations that
  do not support the optional spectral rule MUST leave `pow(1/2) @ x` unresolved
  rather than silently guessing `SX`.
- In this subsection, when disambiguation matters, write
  `U_can(theta, phi, lambda)` for the SDK's internal canonical single-qubit gate
  and `U_qasm(theta, phi, lambda)` for the textual OpenQASM 3.1 built-in family.
  This is a notation aid only; it does **not** rename any public API.
- The library's canonical single-qubit gate `U_can(theta, phi, lambda)` is
  `exp(i * (phi + lambda) / 2) * RZ(phi) * RY(theta) * RZ(lambda)` and therefore
  equals
  `[[cos(theta/2), -exp(i * lambda) * sin(theta/2)], [exp(i * phi) *
  sin(theta/2), exp(i * (phi + lambda)) * cos(theta/2)]]`.
- `U_can(theta, phi, lambda)` is the SDK's **internal canonical** single-qubit
  gate. It is not the OpenQASM 3.1 built-in gate of the same textual name.
- If `U_qasm(theta, phi, lambda)` denotes the OpenQASM 3.1 built-in gate, then
  `U_qasm(theta, phi, lambda) = exp(i * theta / 2) * U_can(theta, phi, lambda)`.
- A parser that canonicalizes textual OpenQASM `U` into this internal `U_can`
  MUST preserve semantics by materializing the extra `theta / 2` as the
  translated gate expression's expression-local zero-qubit phase prefix
  `localPhase`. Only in a normalized semantic representation that is not
  source-preserving / exact-round-trip, and only when the translated call is a
  `bare, unmodified gate instruction` and the fold-safe same-scope rule
  specified in this Phase Convention permits that folding, MAY the
  implementation instead add that phase to the immediate owning scope's scalar
  `globalPhase`.
- A serializer that emits textual OpenQASM `U` for an internal canonical `U_can`
  MUST solve the exact residual-compensation equation for the internal
  instruction's full corrected operator. If that internal instruction denotes
  `exp(i * delta) * U_can(theta, phi, lambda)`, then emitting textual
  `U(theta, phi,
  lambda)` contributes the fixed family correction
  `+theta / 2`, because
  `U_qasm(theta, phi, lambda) = exp(i * theta / 2) * U_can(theta, phi, lambda)`.
  Therefore, if the serializer preserves that same parameter tuple, it MUST
  compute the residual `sigma = delta - theta / 2`. Textual OpenQASM has no
  generic syntax for instruction-local residual `sigma` at that same call site.
  Consequently, emitting textual `U(theta, phi, lambda)` with that unchanged
  parameter tuple is exact if and only if one of the following holds:

  - `sigma` is exactly zero, so the textual family offset alone realizes the
    required corrected operator.
  - The call is a `bare, unmodified gate instruction` and all of `sigma` is
    moved into that same instruction's immediate owning scope scalar
    `globalPhase` by the fold-safe same-scope rule specified in this Phase
    Convention, leaving zero residual instruction-local phase inside the emitted
    gate expression.

  In a representation/API that does not promise source-preserving or exact
  round-trip preservation of that same gate-family spelling and parameter-
  expression tree at that source location, the serializer MAY instead emit a
  different exact textual `U(theta_e, phi_e, lambda_e)` if it can prove that
  `U_qasm(theta_e, phi_e, lambda_e) = exp(i * delta) * U_can(theta, phi, lambda)`
  exactly, so that the emitted call itself carries zero residual instruction-
  local phase. Such a proof is an exact family-specific reparameterization, not
  an extra scalar-phase channel on textual `U`.

  No serializer may behave as though it could keep an arbitrary nonzero residual
  instruction-local phase "inside" textual `U` beyond the denotation of the
  chosen `U` parameters and the only permitted same-scope fold-safe
  compensation. If residual instruction-local phase would remain after applying
  the chosen exact family denotation and that only permitted compensation, the
  serializer MUST instead use another exact encoding permitted by this section,
  such as helper-gate synthesis when the active serializer/API contract permits
  generated declarations, or otherwise fail deterministically.
- The exact boundary identities in the next bullets are normative consequences
  of the OpenQASM language-defined mathematical denotations of those textual
  standard-library gate families themselves. They are not merely conveniences of
  one particular helper expansion through other textual gate families. Any
  implementation that internally rewrites one of those families through
  built-ins such as textual `U` or `gphase` MUST still recover the same exact
  family denotation stated here before exposing semantic IR, applying outer
  modifiers, or performing equality-based rewrites.
- If `rx_qasm(theta)`, `ry_qasm(theta)`, and `rz_qasm(theta)` denote the textual
  OpenQASM standard-library gates `rx`, `ry`, and `rz`, then
  `rx_qasm(theta) = RX(theta)`, `ry_qasm(theta) = RY(theta)`, and
  `rz_qasm(theta) = RZ(theta)` exactly under this internal convention.
- If `p_qasm(lambda)` and `phase_qasm(lambda)` denote the textual OpenQASM
  standard-library gates `p` and `phase`, then
  `p_qasm(lambda) = phase_qasm(lambda) = P(lambda)` exactly under this internal
  convention.
- If `sx_qasm` and `sxdg_qasm` denote the textual OpenQASM standard-library
  gates `sx` and `sxdg`, then `sx_qasm = SX` and `sxdg_qasm = SXdg` exactly
  under this internal convention.
- Consequently, because the OpenQASM standard-library controlled rotations are
  exact controls of those textual rotations, `crx_qasm(theta) = CRX(theta)`,
  `cry_qasm(theta) = CRY(theta)`, and `crz_qasm(theta) = CRZ(theta)` exactly
  under this internal convention, with no additional zero-qubit phase
  correction.
- Consequently, because the OpenQASM standard-library controlled phase gates are
  exact controls of those textual phase gates, `cp_qasm(lambda)` and
  `cphase_qasm(lambda)` are exact semantic identities for `ctrl(P(lambda))`
  under this internal convention, with no additional zero-qubit phase
  correction.
- Parsers and serializers MUST preserve the same parameter values for
  `rx`/`ry`/`rz`, `p`/`phase`, `crx`/`cry`/`crz`, and `cp`/`cphase` at the
  format boundary, and MUST preserve `sx`/`sxdg` unchanged, before any gate
  modifiers are interpreted on the textual gate expression. This is exact
  semantic identity, not an "equal up to global phase" rewrite.

- The following are **internal semantic aliases** defined by exact matrix
  equality under the formulas above for the **internal canonical** `U_can`; they
  are **not** statements about the runtime behavior of `Matrix.equals`, and they
  are **not** the same thing as the textual OpenQASM compatibility gates
  `u1`/`u2`/`u3`: `U1(lambda) = P(lambda)`,
  `U2(phi, lambda) = U_can(pi/2, phi, lambda)`,
  `U3(theta, phi, lambda) = U_can(theta, phi, lambda)`.
- If `u1_qasm(lambda)` denotes the textual OpenQASM compatibility gate `u1`,
  then `u1_qasm(lambda) = U1(lambda) = P(lambda)` exactly under this internal
  convention.
- If `u3_qasm(theta, phi, lambda)` denotes the textual OpenQASM compatibility
  gate `u3` from `stdgates.inc`, then
  `u3_qasm(theta, phi, lambda) = exp(-i * (phi + lambda) / 2) * U_can(theta, phi, lambda)`
  under this internal convention.
- If `u2_qasm(phi, lambda)` denotes the textual OpenQASM compatibility gate
  `u2`, then
  `u2_qasm(phi, lambda) = exp(-i * (phi + lambda) / 2) * U_can(pi/2, phi, lambda)`
  under this internal convention.
- Consequently, the legacy OpenQASM 2 surface identity
  `u1(lambda) = u3(0, 0, lambda)` does **not** hold under this internal
  convention: here `u1_qasm(lambda) = P(lambda)` while
  `u3_qasm(0, 0, lambda) = RZ(lambda)`. Optimizers, canonicalizers, and
  serializers MUST NOT replace one with the other except by the exact
  format-boundary formulas in this subsection.
- A parser that canonicalizes textual OpenQASM `u2`/`u3` into these internal
  aliases MUST materialize the required expression-local zero-qubit phase
  exactly in the translated expression's `localPhase` unless, in a normalized
  semantic representation that is not source-preserving / exact-round-trip, the
  translated call is a `bare, unmodified gate instruction` and the fold-safe
  same-scope rule specified in this Phase Convention permits folding into the
  immediate owning scope's scalar `globalPhase`.
- A serializer that emits textual OpenQASM `u2`/`u3` for internal `U2`/`U3` MUST
  solve the exact residual-compensation equation for the internal instruction's
  full corrected operator. Under the identities above, textual `u2`/`u3`
  contribute the fixed family correction `-(phi + lambda) / 2` relative to the
  internal canonical `U2`/`U3`. Therefore, if the internal instruction denotes
  `exp(i * delta) * U2(phi, lambda)` or
  `exp(i * delta) * U3(theta, phi,
  lambda)`, the serializer MUST compute the
  residual `sigma = delta + (phi + lambda) / 2` if it preserves that same
  parameter tuple. Textual OpenQASM has no generic syntax for instruction-local
  residual `sigma` at that same call site. Consequently, emitting textual
  `u2`/`u3` with that unchanged parameter tuple is exact if and only if one of
  the following holds:

  - `sigma` is exactly zero, so the textual family correction alone realizes the
    required corrected operator.
  - The call is a `bare, unmodified gate instruction` and all of `sigma` is
    moved into that same instruction's immediate owning scope scalar
    `globalPhase` by the fold-safe same-scope rule specified in this Phase
    Convention, leaving zero residual instruction-local phase inside the emitted
    gate expression.

  In a representation/API that does not promise source-preserving or exact
  round-trip preservation of that same gate-family spelling and parameter-
  expression tree at that source location, the serializer MAY instead emit a
  different exact textual `u2(phi_e, lambda_e)` or
  `u3(theta_e, phi_e, lambda_e)` if it can prove that the chosen textual family
  denotation already equals the full corrected internal operator exactly, so
  that the emitted call itself carries zero residual instruction-local phase.
  Such a proof is an exact family-specific reparameterization, not an extra
  scalar-phase channel on textual `u2`/`u3`.

  No serializer may behave as though it could keep an arbitrary nonzero residual
  instruction-local phase "inside" textual `u2`/`u3` beyond the denotation of
  the chosen family parameters and the only permitted same-scope fold-safe
  compensation. If residual instruction-local phase would remain after applying
  the chosen exact family denotation and that only permitted compensation, the
  serializer MUST instead use another exact encoding permitted by this section,
  such as helper-gate synthesis when the active serializer/API contract permits
  generated declarations, or otherwise fail deterministically.
- Whenever this subsection permits a parser or serializer to replace the
  `localPhase` of a translated instruction that is a
  `bare, unmodified gate instruction` by the immediate owning scope scalar, that
  permission is representation-local only and is available only in a
  representation that has either committed that the translated instruction will
  remain a `bare, unmodified gate instruction` for the rest of that
  representation's lifetime, or stored enough instruction-local provenance to
  re-materialize the exact same correction deterministically. A pass that may
  later apply, infer, or expose outer modifiers around that translated
  instruction MUST keep the correction instruction-local, or restore it from
  that stored provenance, before performing the transform.
- Exact named-gate identities include `X = U_can(pi, 0, pi)`,
  `H = U_can(pi/2, 0, pi)`, `Y = U_can(pi, pi/2, pi/2)`.
- The controlled canonical `CU(theta, phi, lambda, gamma)` means identity on the
  control-0 subspace and `exp(i * gamma) * U_can(theta, phi, lambda)` on the
  control-1 subspace.
- If `CU_qasm(theta, phi, lambda, gamma)` denotes the OpenQASM 3.1 standard-
  library `cu`, then
  `CU_qasm(theta, phi, lambda, gamma) = CU(theta, phi, lambda, gamma + theta/2)`
  under this internal convention. Parsers/serializers MUST apply that offset
  exactly at the format boundary rather than treating textual `cu` as a synonym
  for the internal canonical `CU`.
- A parser that canonicalizes textual OpenQASM `cu(theta, phi, lambda, gamma)`
  into this internal `CU` MUST add `theta / 2` to the internal `gamma` parameter
  and MUST NOT represent that boundary correction as a whole-expression
  `localPhase`, because the offset belongs to the enabled control-1 subspace
  rather than to the full controlled operator.
- A serializer that emits textual OpenQASM `cu` for an internal canonical `CU`
  MUST solve the exact residual-compensation equation for the control-subspace
  phase parameter. If the internal instruction denotes
  `exp(i * delta) * CU(theta, phi, lambda, gamma)`, then textual
  `cu(theta, phi, lambda, gamma_qasm)` realizes
  `CU(theta, phi, lambda, gamma_qasm + theta/2)` and contributes no independent
  whole-expression scalar-phase channel. Therefore the serializer MUST set
  `gamma_qasm = gamma - theta / 2` and then separately discharge the outer
  whole-expression residual `delta`. Emitting textual `cu` directly at that call
  site is exact if and only if one of the following holds:

  - `delta` is exactly zero, so textual
    `cu(theta, phi, lambda, gamma - theta/2)` alone realizes the required
    corrected operator.
  - The call is a `bare, unmodified gate instruction` and all of `delta` is
    moved into that same instruction's immediate owning scope scalar
    `globalPhase` by the fold-safe same-scope rule specified in this Phase
    Convention, leaving zero residual whole-expression phase inside the emitted
    gate expression.

  No other direct emission of textual `cu` at that call site is exact. In
  particular, textual `cu` parameter `gamma` controls only the relative phase on
  the enabled control-1 subspace; it MUST NOT be used to encode an outer
  whole-expression zero-qubit phase prefix on the full controlled operator. If
  any such residual phase remains after applying the exact `gamma_qasm`
  translation and the only permitted same-scope fold-safe compensation, the
  serializer MUST instead use another exact encoding permitted by this section,
  such as helper-gate synthesis when the active serializer/API contract permits
  generated declarations, or otherwise fail deterministically.

**6. Phase under inversion and powers**

- For any unitary gate or unitary-only scope `W`, `inv(W)` denotes the adjoint
  `W†`. For non-unitary scopes, inverse is undefined unless a separate API
  specifies exact inverse semantics.
- When a gate expression denotes `exp(i * delta) * W`, including a translated
  gate call carrying instruction-local `localPhase = delta`, `inv` and `pow` act
  on that full corrected operator, not on `W` with `delta` discarded. In
  particular, `inv(exp(i * delta) * W) = exp(-i * delta) * W†`, and integer
  powers are powers of the same full matrix/operator.
- Modifier stacks remain right-associative under this rule. If a base gate `G`
  carries `localPhase = delta`, then `inv @ ctrl @ G` means
  `inv(ctrl(exp(i * delta) * G))`, which equals `ctrl(exp(-i * delta) * G†)`;
  equivalently, `ctrl @ inv @ G` under the same exact corrected-operator
  semantics. This equivalence is a **derived theorem** for unitaries: because
  `inv(diag(I, W)) = diag(I, W†) = ctrl(inv(W))`, `inv` and single positive
  `ctrl` commute in the modifier stack when the operand is unitary. It is not a
  general parsing axiom: implementations MUST NOT assume that `inv` commutes
  with `ctrl` for non-unitary operands or with `negctrl` in mixed-polarity
  chains without a separate proof for the specific operand.
- Likewise, `pow(2) @ ctrl @ G` means `(ctrl(exp(i * delta) * G))^2` because the
  outer `pow(2)` acts on the already-controlled operator produced by the inner
  `ctrl`. For integer powers this equals `ctrl((exp(i * delta) * G)^2)` by exact
  block-diagonal multiplication, but that equality is a derived theorem of this
  convention, not the parse rule itself.
- No implementation may assume a generic rewrite
  `pow(k) @ ctrl @ G = ctrl @ pow(k) @ G` for non-integer `k` unless that
  equality has been separately proved under the exact principal-branch power
  rule for that specific operand.
- `inv(globalPhaseGate(theta)) = globalPhaseGate(-theta)`.
- An exponent expression `k` uses the integer-power rule if and only if the
  active representation can prove, without floating-point rounding, epsilon
  tests, or heuristic algebra, that every dynamic execution of the gate call
  evaluates `k` to an integer exactly. Such proof may use literal integer
  syntax, exact constant folding, exact symbolic algebra, exact type/range
  information, or later parameter binding.
- This classification is semantic, not purely syntactic. Expressions such as
  `2.0` when the active frontend syntax documents that finite decimal literal as
  the exact rational 2, `6/3`, or a later-bound symbol may use the integer-power
  rule if that exact proof succeeds; values known only approximately, or
  expressions not exactly proved integral, MUST NOT.
- Whenever this subsection requires proving that an exponent is integer-valued,
  the proof obligation is judged under the active representation's documented
  exact-expression semantics, which MUST include at least the minimum
  conformance profile stated earlier in this section. Implementations MAY
  conservatively preserve `pow(k)` unresolved when no exact proof procedure is
  available or when such a procedure fails to discharge the obligation, but they
  MUST NOT use floating-point epsilon tests, approximate evaluation, or
  heuristic algebra to force the integer-power rule.
- If the current representation cannot yet prove whether `k` is integer-valued,
  it MUST preserve `pow(k)` as an unresolved exact modifier. Any transform that
  must choose between integer-power and non-integer-power semantics MUST defer
  or reject rather than guess from surface spelling or approximate numeric
  evaluation.
- Integer powers obey `W^0 = I` on the same arity as `W`,
  `W^k = W · W · ... · W` for positive integers `k`, and `W^(-k) = (W†)^k` for
  positive integers `k`.
- Integer powers of `globalPhaseGate(theta)` therefore satisfy
  `globalPhaseGate(theta)^k = globalPhaseGate(k * theta)` exactly.
- When `k` is exactly known to be a non-integer real, `pow(k)` has a concrete
  operator denotation in a normalized semantic representation only when this
  specification gives the gate or scope an explicit principal-branch power rule
  under this convention.
- To preserve complete OpenQASM 3.1 surface coverage, parsers and source-
  preserving / exact-round-trip representations MUST nevertheless accept and
  preserve a textual `pow(k)` modifier on any gate expression, including when
  exact classification of `k` as integer-valued or non-integer is not yet
  available, or when a known non-integer exponent does not yet have a concrete
  operator denotation under the rule above. Such an unresolved modifier is exact
  syntax, not a guessed matrix/operator.
- In a representation carrying such an unresolved `pow(k)`, any matrix-dependent
  transform or check is unavailable unless and until a later exact proof,
  parameter binding, or certification resolves both the exponent classification
  and, when needed, the concrete non-integer power rule. Equality-based
  simplification, exact matrix extraction, basis decomposition to a concrete
  gate set, simulation, and remote submission to a backend that requires a
  concrete unitary MUST therefore either preserve the unresolved modifier
  unchanged when their documented contract allows that, or reject
  deterministically.
- In this specification, the only mandatory concrete non-integer power
  denotation that every implementation MUST support is the principal-branch
  scalar-phase rule for `globalPhaseGate(theta)` and any explicit zero-qubit
  phase instruction with the same semantics.
- The rule above fixes the denotation. Eager rewriting of a non-integer power to
  another explicit gate form is optional and is allowed only when the resulting
  principal-branch phase is known exactly under the current symbolic/numeric
  information.
- An implementation MAY additionally support non-integer powers of other
  unitaries only by principal-branch spectral decomposition of the full operator
  under this same convention. For symbolic operands, this requires an exact
  derivation of the eigenphases and spectral projectors. For concrete numeric
  operands, this is a numeric algorithm on the full matrix itself, not heuristic
  parameter scaling or pattern-based angle division.
- Such support MUST preserve the phase semantics of the full operator, MUST
  treat degenerate eigenspaces in a basis-independent way, and heuristic
  parameter scaling is forbidden unless this document separately proves it as an
  explicit rule.
- For concrete numeric operands, the spectral rule is available only if the
  implementation can establish, by a fixed deterministic procedure at epsilon
  `1e-10`, a decomposition `W ~= sum_j exp(i * phi_j) * Pi_j` with pairwise
  orthogonal projectors `Pi_j` that reconstruct `W`, sum to the identity, and
  are invariant under changes of basis within each degenerate eigenspace. If
  that procedure cannot certify such a projector decomposition, including
  because a required eigenspace split is numerically ill-conditioned or
  unresolved, then the non-integer power rule is unavailable.
- For any such concrete numeric spectral rule, each eigenphase MUST first be put
  on this section's principal branch independently by applying `wrapPhase`
  before multiplication by `k`, using the same epsilon snapping rule near the
  branch cut. Equivalently, if `W = sum_j exp(i * phi_j) * Pi_j` is a certified
  numeric spectral decomposition into pairwise orthogonal projectors `Pi_j`,
  then the principal power uses `phi0_j = wrapPhase(phi_j)` and
  `W^k = sum_j exp(i * k * phi0_j) * Pi_j`. In particular, eigenvalue `-1` uses
  phase `pi`, not `-pi`.
- If no exact rule is available, the modifier MUST remain symbolic /
  syntax-preserved or raise a validation/transpilation error in any context that
  requires a concrete semantic operator. A serializer whose target is textual
  OpenQASM MUST re-emit such an unresolved modifier unchanged whenever the
  active representation still preserves enough syntax-local information to do so
  exactly. Any backend that requires a concrete numeric unitary MUST reject
  unresolved non-integer powers before execution or remote submission.
- For numeric scalar phases, the principal-branch scalar denotation is the 1x1
  instance of the same eigenphase rule: first reduce the numeric phase to
  `theta0 = wrapPhase(theta)`, then the concrete result is
  `globalPhaseGate(k * theta0)`.
- For symbolic scalar phases, the denotation is still that same principal-branch
  scalar power. Implementations MAY rewrite it eagerly to an explicit
  `globalPhaseGate(...)` only when the principal representative under the same
  `(-pi, pi]` branch is known exactly; otherwise they MUST preserve the `pow`
  operation symbolically until later evaluation or exact proof.
- The same rules apply to any explicit zero-qubit phase instruction.

## 3. Gate Definitions — Compositional Architecture

Every gate family in this section has a **pure constructor**. Fixed-family
unitary gates whose exact denotation is fully concrete at construction time
return a `Matrix`. Entries whose normative definition is given as a named-
register circuit template, that require ancilla/scratch ownership, or whose
classical preprocessing may need to be deferred are instead exact gate/template
descriptors even when their logical action is unitary. Many such entries are
called out explicitly as **exact circuit templates**, **approximate synthesis
families**, or exact semantic gate families; where a later entry is not
separately labeled, its constructor category is determined by the form of the
specification itself:

- an explicit fixed-size matrix/formula denotes a concrete matrix-returning gate
  family;
- a named-register decomposition with scratch/ancilla obligations denotes an
  exact circuit template or exact semantic gate family, not a promise of eager
  full-matrix materialization; and
- an explicitly approximate family denotes deterministic approximation
  preprocessing whose produced coefficient/angle data becomes the family's
  post-construction exact semantics.

When an entry states both a semantic matrix/formula and a named-register
decomposition, the matrix/formula fixes the exact denotation only. Unless that
entry explicitly promises eager matrix materialization, a family whose matrix
size depends on user-supplied arity, register width, or list length defaults to
an immutable exact semantic gate/template descriptor rather than a required
eager `Matrix`. This applies, for example, to higher-tier arity-dependent
families such as `DiagonalGate`, `UCR*`, `UCGate`, `UnitaryGate`, `QFTGate`, and
similar constructions. An implementation MAY still materialize such a matrix
eagerly when feasible, but that is an implementation choice, not part of the
public constructor contract.

Within Section 3, a family's **semantic input** means the complete data needed
to identify one exact gate/operator or exact gate-template instance, whether an
implementation stores that as one immutable descriptor or as a reusable gate
object plus a separate immutable application node. Unless a later entry
explicitly says otherwise, any ordered operand/register list written in a family
signature is part of that semantic input. When Section 2's MSB-first matrix
indexing, little-endian register semantics, selected-subspace rules, or
validation obligations depend on that order, equality-sensitive rewrites and
serialization MUST preserve the same ordered operand list losslessly; it is not
disposable placement metadata.

Whether an entry is a concrete matrix gate, a deferred semantic gate, or a
template, the specification must define the full logical action on the
user-visible registers, not only the special case where auxiliary outputs happen
to start in `|0...0⟩`. Implement ALL of the following — no omissions.

When a decomposition requires classical preprocessing such as ZYZ/CSD parameter
extraction, eigendecomposition, Chebyshev interpolation, or transcendental angle
evaluation (`sqrt`, `arcsin`, etc.), that preprocessing is part of the exact
specification only for concrete numeric inputs, or for symbolic inputs whose
required quantities the implementation can derive exactly. To remain consistent
with Section 2, implementations MUST NOT silently replace such steps by
heuristic approximations, floating-point branch guesses, or partially evaluated
symbolic numerics. If an exact symbolic construction is not available, the gate
MUST remain as a first-class semantic object (or another exactly equivalent
symbolic representation) until parameters are bound. Any backend or
transpilation stage that requires a concrete numeric matrix or numeric rotation
angles MUST reject unresolved instances before execution or remote submission.

### Foundational Principle: Single-Qubit + CNOT Universality with Explicit Zero-Qubit Phase

Following Nielsen & Chuang §1.3.2 — **any multi-qubit unitary can be decomposed
into single-qubit gates and CNOT (CX) gates up to global phase.** Under this
SDK's exact matrix semantics from Section 2, that theorem is lifted to exact
equality by adjoining the Tier 0 zero-qubit phase primitive
`GlobalPhaseGate(...)`. Together, the Tier 0 primitives and Tier 1 `CX` form the
exact universal basis used by this section.

This library embraces this principle architecturally: every fixed-family
compositional multi-qubit gate and every exact circuit template in this section
is specified by an exact reference synthesis over Tier 0 primitives and CX. For
compositional standalone gates, implementations compute their matrices by
composing the constituent gate matrices via matrix multiplication and tensor
products — not by hardcoding larger matrix entries. Matrix-native semantic
families such as arbitrary unitaries, isometries, and Hamiltonian evolutions
must also provide an exact reference synthesis; once the required exact
preprocessing has been carried out, that synthesis must likewise bottom out in
Tier 0 primitives and the Tier 1 CX gate.

The gates are organized into **tiers of abstraction**, where each tier normally
builds exclusively on gates from lower tiers. When a gate is explicitly marked
as a **direct synthesis** or **optimization**, it may bypass intermediate named
abstractions, but it must still be composed only from Tier 0 primitives and the
Tier 1 CX gate. No larger hardcoded matrix is introduced.

The tiers are conceptual rather than a strict build-order guarantee. A few
entries intentionally forward-reference a later named abstraction when that is
the clearest exact subroutine; such forward references do not change the
requirement that every concrete synthesis ultimately bottoms out in Tier 0
primitives and the Tier 1 CX gate.

```
Tier 0:  Zero-qubit and single-qubit primitives (1×1 / 2×2 matrix definitions)
Tier 1:  CX — the universal entangling primitive (4×4 matrix definition)
Tier 2:  Fundamental two-qubit controlled gates (from Tier 0 + Tier 1)
Tier 3:  Higher two-qubit interaction gates (from Tier 2)
Tier 4:  Three-qubit gates (from Tier 2 + Tier 3)
Tier 5:  Four+ qubit and multi-controlled gates (from Tier 4)
Tier 6:  N-qubit structural composite gates (from lower tiers)
Tier 7:  Uniformly controlled gates and general unitary synthesis (from Tier 6)
Tier 8:  Hamiltonian simulation and Pauli evolution (from Tier 6)
Tier 9:  Quantum Fourier Transform (from Tier 2)
Tier 10: Reversible classical-logic gates (from Tier 4)
Tier 11: Quantum arithmetic circuits (from Tier 9 + Tier 10)
Tier 12: Function loading and approximation (from Tier 11 + Tier 5)
Tier 13: Comparison, aggregation, and oracles (from Tier 10 + Tier 5)
Tier 14: State preparation (from lower tiers)
```

**Notation convention:** Decompositions are written in **circuit time order**
(left to right), which matches the order operations are appended to a
`QuantumCircuit` builder. The symbol `→` means "followed by." In standard matrix
multiplication, the total unitary for a circuit `A → B → C` is `C · B · A` (the
leftmost gate A is applied first, so it appears rightmost in the matrix
product).

- `→` always denotes **circuit time order** (earliest operation on the left).
- `·` and `*` denote **matrix-product order** (leftmost applied last).
- Basis-state examples for multi-qubit gates use the same MSB-first local
  argument order used by the gate matrices. For example, `|c,t⟩` for a
  controlled gate and `|c1,c2,t⟩` for a doubly-controlled gate.
- When a helper is defined algebraically, e.g. `A = RZ(phi) * RY(theta/2)`, that
  is a matrix identity; the corresponding circuit-time sequence on that qubit is
  `RY(theta/2) → RZ(phi)`.
- When a family below is written both with grouped operands/parameters (for
  example `G([c0, ..., cN-1], t)` or `G([theta_0, theta_1], [c], t)`) and with
  variadic operands/parameters (for example `G(c0, ..., cN-1, t)` or
  `G(theta_0, theta_1, c, t)`), those are two display forms for the same ordered
  semantic inputs. Bracketed lists are explanatory shorthand for one ordered
  operand/parameter list; they do **not** define a second overload unless that
  family explicitly says so.
- Unless a gate family below explicitly states another operand contract, every
  concrete multi-qubit gate call requires pairwise distinct underlying qubit
  wires after resolving aliases, slices, register elements, or equivalent
  frontend references. Reusing the same wire in multiple operand positions is
  invalid and MUST fail validation (or the gate MUST remain unresolved/opaque
  until distinctness can be decided exactly).

---

### Tier 0: Zero-Qubit and Single-Qubit Primitives (1×1 / 2×2 Matrices)

These are the atomic building blocks. Each is defined by its explicit matrix.
Together with CX, they form the universal alphabet.

**Zero-Qubit (Global) Gate:**

| Function                 | Gate   | Matrix / Description                                                  |
| ------------------------ | ------ | --------------------------------------------------------------------- |
| `GlobalPhaseGate(theta)` | GPhase | `[[exp(i*theta)]]` (1×1; multiplies the full state by `exp(i*theta)`) |

**Single-Qubit Gates (return 2×2 Matrix):**

| Function           | Gate           | Matrix                                                                               |
| ------------------ | -------------- | ------------------------------------------------------------------------------------ |
| `IGate()`          | I              | `[[1, 0], [0, 1]]`                                                                   |
| `HGate()`          | H              | `(1/sqrt(2)) * [[1, 1], [1, -1]]`                                                    |
| `XGate()`          | X              | `[[0, 1], [1, 0]]`                                                                   |
| `YGate()`          | Y              | `[[0, -i], [i, 0]]`                                                                  |
| `ZGate()`          | Z              | `[[1, 0], [0, -1]]`                                                                  |
| `PhaseGate(l)`     | P(l)           | `[[1, 0], [0, exp(i*l)]]`                                                            |
| `RGate(th, ph)`    | R(th,ph)       | `[[cos(th/2), -i*exp(-i*ph)*sin(th/2)], [-i*exp(i*ph)*sin(th/2), cos(th/2)]]`        |
| `RXGate(th)`       | RX(th)         | `[[cos(th/2), -i*sin(th/2)], [-i*sin(th/2), cos(th/2)]]`                             |
| `RYGate(th)`       | RY(th)         | `[[cos(th/2), -sin(th/2)], [sin(th/2), cos(th/2)]]`                                  |
| `RZGate(th)`       | RZ(th)         | `[[exp(-i*th/2), 0], [0, exp(i*th/2)]]`                                              |
| `SGate()`          | S              | `[[1, 0], [0, i]]`                                                                   |
| `SdgGate()`        | Sdg            | `[[1, 0], [0, -i]]`                                                                  |
| `SXGate()`         | SX             | `(1/2) * [[1+i, 1-i], [1-i, 1+i]]`                                                   |
| `SXdgGate()`       | SXdg           | `(1/2) * [[1-i, 1+i], [1+i, 1-i]]`                                                   |
| `TGate()`          | T              | `[[1, 0], [0, exp(i*pi/4)]]`                                                         |
| `TdgGate()`        | Tdg            | `[[1, 0], [0, exp(-i*pi/4)]]`                                                        |
| `UGate(th, ph, l)` | U_can(th,ph,l) | `[[cos(th/2), -exp(i*l)*sin(th/2)], [exp(i*ph)*sin(th/2), exp(i*(ph+l))*cos(th/2)]]` |

`UGate(th, ph, l)` is the public constructor name; the symbol `U_can` here
follows Section 2's notation for the SDK's internal canonical single-qubit gate
and distinguishes it from textual OpenQASM `U`.

**RVGate (Rotation-Vector Gate):**

`RVGate(vx, vy, vz)` — rotation around axis `v = (vx, vy, vz)` by angle `|v|`.

Let `norm = sqrt(vx^2 + vy^2 + vz^2)`. If `norm = 0`, return `I` exactly.
Otherwise let `half = norm/2`, `nx = vx/norm`, `ny = vy/norm`, `nz = vz/norm`:

```
RVGate(vx,vy,vz) =
  [[cos(half) - i*nz*sin(half),    (-i*nx - ny)*sin(half)],
   [(-i*nx + ny)*sin(half),          cos(half) + i*nz*sin(half)]]
```

This is the matrix exponential `exp(-i * (vx*X + vy*Y + vz*Z) / 2)`.

For concrete numeric inputs, the constructor returns this 2×2 matrix directly.
For symbolic or otherwise unresolved exact inputs, the implementation MUST apply
the generic Section 3 preprocessing rule: if it cannot prove the `norm = 0`
branch exactly or derive the nonzero branch exactly without an unresolved
division by `norm`, it MUST preserve `RVGate(vx, vy, vz)` as a first-class exact
single-qubit semantic gate until parameters are concrete enough. It MUST NOT
guess the zero/nonzero branch heuristically or divide by a symbolic quantity
that may still be zero.

**Equivalences within Tier 0:**

- `T = P(pi/4)`, `S = P(pi/2)`, `Z = P(pi)`
- `Sdg = S†`, `Tdg = T†`, `SXdg = SX†`
- `SX * SX = X`, `SX * SXdg = I`
- `X = U_can(pi, 0, pi)`, `H = U_can(pi/2, 0, pi)`, `Y = U_can(pi, pi/2, pi/2)`
- `P(l) = exp(i*l/2) * RZ(l)` (Phase Convention 5)
- `RX(th) = R(th, 0)`, `RY(th) = R(th, pi/2)`

Format-boundary note: some external circuit formats use the same textual gate
names with different global-phase choices. The formulas above define the
library's internal gates exactly. Serializer adapters must perform any required
phase compensation at the format boundary.

---

### Tier 1: The Universal Entangling Primitive — CX

The **only** primitive multi-qubit gate introduced by a hardcoded matrix
literal. Later multi-qubit entries may also state their full matrices as derived
semantic denotations or reference identities, but no other multi-qubit primitive
is introduced independently of composition. Together with the Tier 0 primitives,
CX is sufficient to construct every exact unitary denotation used in this SDK:
the single-qubit + CX universality theorem supplies the non-scalar part, and
`GlobalPhaseGate` carries any required exact zero-qubit scalar.

| Function   | Gate      | Matrix                                      |
| ---------- | --------- | ------------------------------------------- |
| `CXGate()` | CX / CNOT | `[[1,0,0,0],[0,1,0,0],[0,0,0,1],[0,0,1,0]]` |

MSB-first ordering: bit 1 = control, bit 0 = target. Identity on `|00⟩`,`|01⟩`
(control=0); X on target for `|10⟩`,`|11⟩` (control=1).

---

### General Controlled-U Construction (ABC Decomposition)

Before defining Tier 2, we establish the **master recipe** from Nielsen & Chuang
(Corollary 4.2) for building a controlled version of any single-qubit unitary.

Let `W` be an arbitrary single-qubit unitary. Write
`W = exp(i*alpha) * A * X * B * X * C` where `A * B * C = I`. Then:

```
Controlled-W(control, target) =
    C(t) → CX(c,t) → B(t) → CX(c,t) → A(t) → P(alpha)(c)
```

When control = 0: CX does nothing, target sees `A * B * C = I`. ✓ When control =
1: CX applies X, target sees `A * X * B * X * C = exp(-i*alpha) * W`, and
`P(alpha)` on control contributes `exp(i*alpha)` when c=1, giving total `W` on
the controlled subspace. ✓

**This uses exactly 2 CX gates + 4 single-qubit gates.**

For the SDK's internal canonical gate `U_can(theta, phi, lambda)`:

```
A = RZ(phi) * RY(theta/2)
B = RY(-theta/2) * RZ(-(phi+lambda)/2)
C = RZ((lambda-phi)/2)
alpha = (phi + lambda) / 2
```

Verify:
`A * B * C = RZ(phi) * RY(theta/2) * RY(-theta/2) * RZ(-(phi+lambda)/2) * RZ((lambda-phi)/2) = RZ(phi) * I * RZ(-phi) = I`
✓

For the SDK's internal canonical `CU(theta, phi, lambda, gamma)` (where `gamma`
is the additional phase on the enabled control-1 subspace, not a
whole-expression global phase):

```
CU(th,ph,l,gamma, c, t) =
    RZ((l-ph)/2)(t) → CX(c,t) → RZ(-(ph+l)/2)(t) → RY(-th/2)(t) →
    CX(c,t) → RY(th/2)(t) → RZ(ph)(t) → P(gamma + (ph+l)/2)(c)
```

---

### Tier 2: Fundamental Two-Qubit Controlled Gates

Every gate here is built **directly** from Tier 0 + Tier 1. Each function
computes its matrix by composing the constituent matrices.

**CZGate — Controlled-Z (1 CX)**

```
CZ(c, t) = H(t) → CX(c, t) → H(t)
```

Proof: When c=1, target sees `H * X * H = Z`. ✓

**CYGate — Controlled-Y (1 CX)**

```
CY(c, t) = Sdg(t) → CX(c, t) → S(t)
```

Proof: When c=1, target sees `S * X * Sdg`. Compute:
`S*X*Sdg = [[1,0],[0,i]]*[[0,1],[1,0]]*[[1,0],[0,-i]] = [[0,-i],[i,0]] = Y`. ✓

**CPhaseGate — Controlled-Phase (2 CX)**

```
CP(lambda, c, t) =
    P(lambda/2)(t) → CX(c,t) → P(-lambda/2)(t) → CX(c,t) → P(lambda/2)(c)
```

Proof: When c=0: `P(-l/2)*P(l/2) = I` on target, no phase on control. ✓ When
c=1,t=1: total phase on `|11⟩` is `exp(i*l)`. ✓

**CRZGate — Controlled-RZ (2 CX)**

```
CRZ(theta, c, t) = RZ(theta/2)(t) → CX(c,t) → RZ(-theta/2)(t) → CX(c,t)
```

Proof: When c=0: `RZ(-th/2)*RZ(th/2) = I`. ✓ When c=1:
`X*RZ(-th/2)*X = RZ(th/2)`, so total = `RZ(th)`. ✓

**CRYGate — Controlled-RY (2 CX)**

```
CRY(theta, c, t) = RY(theta/2)(t) → CX(c,t) → RY(-theta/2)(t) → CX(c,t)
```

Proof: `X*RY(-th/2)*X = RY(th/2)`, so when c=1: `RY(th)`. ✓

**CRXGate — Controlled-RX (2 CX, via CRZ)**

```
CRX(theta, c, t) = H(t) → CRZ(theta, c, t) → H(t)
```

Proof: `H*RZ(th)*H = RX(th)`, so controlled RZ becomes controlled RX. ✓
Expanding: `H(t) → RZ(th/2)(t) → CX(c,t) → RZ(-th/2)(t) → CX(c,t) → H(t)`.

**CSGate — Controlled-S (2 CX, via CP)**

```
CS(c, t) = CP(pi/2, c, t)
```

Since `S = P(pi/2)`.

**CSdgGate — Controlled-Sdg (2 CX, via CP)**

```
CSdg(c, t) = CP(-pi/2, c, t)
```

**CSXGate — Controlled-SX (2 CX)**

Since `SX = exp(i*pi/4) * RX(pi/2)`, the controlled version promotes the global
phase as a relative phase on the control:

```
CSX(c, t) = P(pi/4)(c) → CRX(pi/2, c, t)
```

Expanding:
`P(pi/4)(c) → H(t) → RZ(pi/4)(t) → CX(c,t) → RZ(-pi/4)(t) → CX(c,t) → H(t)`.

**CHGate — Controlled-Hadamard (2 CX, from ABC decomposition)**

Using the general controlled-U recipe with the ZYZ decomposition of H:
`H = exp(i*pi/2) * RZ(0) * RY(pi/2) * RZ(pi)`, so `alpha = pi/2`.

```
CH(c, t) =
    RZ(pi/2)(t) → CX(c, t) → RZ(-pi/2)(t) → RY(-pi/4)(t) →
    CX(c, t) → RY(pi/4)(t) → P(pi/2)(c)
```

**CUGate — General Controlled-U (2 CX)**

```
CU(theta, phi, lambda, gamma, c, t) =
    RZ((lambda-phi)/2)(t) → CX(c, t) →
    RZ(-(phi+lambda)/2)(t) → RY(-theta/2)(t) →
    CX(c, t) → RY(theta/2)(t) → RZ(phi)(t) →
    P(gamma + (phi+lambda)/2)(c)
```

This is identity on the control-0 subspace and
`exp(i*gamma) * U_can(theta, phi, lambda)` on the control-1 subspace.

**DCXGate — Double-CNOT (2 CX)**

```
DCX(a, b) = CX(a, b) → CX(b, a)
```

---

### Tier 3: Higher Two-Qubit Interaction Gates

Gates here are built from Tier 2 abstractions (and transitively Tier 0 + Tier
1).

**SwapGate — SWAP (3 CX)**

```
SWAP(a, b) = CX(a, b) → CX(b, a) → CX(a, b)
```

**RZZGate — ZZ Ising Interaction (2 CX)**

```
RZZ(theta, a, b) = CX(a, b) → RZ(theta)(b) → CX(a, b)
```

Produces `exp(-i*theta/2 * Z⊗Z)`. Proof on basis: CX entangles parity into qubit
b; RZ rotates by parity; CX unentangles. Verify: `|00⟩ → e^(-ith/2)|00⟩`,
`|01⟩ → e^(ith/2)|01⟩`, `|10⟩ → e^(ith/2)|10⟩`, `|11⟩ → e^(-ith/2)|11⟩`. ✓

**RXXGate — XX Ising Interaction (2 CX, via RZZ)**

```
RXX(theta, a, b) = H(a) → H(b) → RZZ(theta, a, b) → H(a) → H(b)
```

Proof: `H*Z*H = X`, so `(H⊗H) * (Z⊗Z) * (H⊗H) = X⊗X`. ✓

**RYYGate — YY Ising Interaction (2 CX, via RZZ)**

```
RYY(theta, a, b) =
    RX(-pi/2)(a) → RX(-pi/2)(b) → RZZ(theta, a, b) → RX(pi/2)(a) → RX(pi/2)(b)
```

Proof: in matrix-product order the conjugation is
`(RX(pi/2)⊗RX(pi/2)) * (Z⊗Z) * (RX(-pi/2)⊗RX(-pi/2))`, and
`RX(pi/2)*Z*RX(-pi/2) = -Y`, so the conjugated generator is `(-Y)⊗(-Y) = Y⊗Y`. ✓

**RZXGate — ZX Cross-Resonance Interaction (2 CX, via RZZ)**

```
RZX(theta, a, b) = H(b) → RZZ(theta, a, b) → H(b)
```

Proof: `I⊗H` transforms `Z⊗Z → Z⊗X`. ✓

**ECRGate — Echoed Cross-Resonance (2 CX, via RZX)**

```
ECR(a, b) = RZX(pi/2, a, b) → X(a)
```

Matrix-product order: `(X⊗I) * RZX(pi/2)`.

This SDK fixes the internal `ECR` family by that exact decomposition. In the
local MSB-first basis order `|a,b⟩`, the resulting matrix is

`(1/sqrt(2)) * [[0,0,1,i],[0,0,i,1],[1,-i,0,0],[-i,1,0,0]]`.

Implementations and serializers MUST treat that matrix as the semantic source of
truth for this SDK's `ECR`; any external backend or text format that uses a
different `ecr` convention (for example because of operand order or extra local
phases) must translate exactly at the format boundary rather than reinterpret
this internal gate family.

**iSwapGate — iSWAP (4 CX)**

```
iSWAP(a, b) = CZ(a, b) → SWAP(a, b) → S(a) → S(b)
```

Proof: `|01⟩ → CZ → |01⟩ → SWAP → |10⟩ → S⊗S → i|10⟩`. ✓
`|10⟩ → CZ → |10⟩ → SWAP → |01⟩ → S⊗S → i|01⟩`. ✓
`|11⟩ → CZ → -|11⟩ → SWAP → -|11⟩ → S⊗S → -(i·i)|11⟩ = -(-1)|11⟩ = |11⟩`. ✓

**XXPlusYYGate — Parameterized (XX+YY) Interaction (4 CX, via RXX + RYY)**

```
XXPlusYY(theta, beta, a, b) =
    RZ(-beta/2)(a) → RZ(beta/2)(b) →
    RXX(theta/2, a, b) → RYY(theta/2, a, b) →
    RZ(beta/2)(a) → RZ(-beta/2)(b)
```

Acts on the `{|01⟩, |10⟩}` subspace with phase twist beta. Identity on
`{|00⟩, |11⟩}`.

For `beta = 0`, the commuting product `RXX(theta/2) * RYY(theta/2)` equals
`exp(-i * theta/4 * (X⊗X + Y⊗Y))`, whose local MSB-first matrix is exactly

`[[1,0,0,0],[0,cos(theta/2),-i*sin(theta/2),0],[0,-i*sin(theta/2),cos(theta/2),0],[0,0,0,1]]`.

The surrounding `RZ` conjugation applies the phase twist on the excitation-
preserving `|01⟩ ↔ |10⟩` coupling, yielding the appendix matrix below exactly.

**XXMinusYYGate — Parameterized (XX-YY) Interaction (4 CX, via RXX + RYY)**

```
XXMinusYY(theta, beta, a, b) =
    RZ(-beta/2)(a) → RZ(-beta/2)(b) →
    RXX(theta/2, a, b) → RYY(-theta/2, a, b) →
    RZ(beta/2)(a) → RZ(beta/2)(b)
```

Acts on the `{|00⟩, |11⟩}` subspace with phase twist beta. Identity on
`{|01⟩, |10⟩}`.

For `beta = 0`, the commuting product `RXX(theta/2) * RYY(-theta/2)` equals
`exp(-i * theta/4 * (X⊗X - Y⊗Y))`, whose local MSB-first matrix is exactly

`[[cos(theta/2),0,0,-i*sin(theta/2)],[0,1,0,0],[0,0,1,0],[-i*sin(theta/2),0,0,cos(theta/2)]]`.

The surrounding `RZ` conjugation applies the phase twist on the pair-creation /
pair-annihilation `|00⟩ ↔ |11⟩` coupling, yielding the appendix matrix below
exactly.

---

### Tier 4: Three-Qubit Gates

**CCXGate — Toffoli / Doubly-Controlled X**

**V-decomposition** (Barenco et al., 1995): Since `V² = X` where `V = SX`:

```
CCX(c1, c2, t) =
    CSX(c1, t) → CX(c1, c2) → CSXdg(c2, t) → CX(c1, c2) → CSX(c2, t)
```

where `CSXdg(c, t)` is only local shorthand for the exact decomposition
`P(-pi/4)(c) → CRX(-pi/2, c, t)`. It denotes the exact singly-controlled lifting
of `SXdg` used by this derivation; it is **not** a separate public gate family
or additional constructor beyond this local use.

Proof (all control states, matrix-product order on target):

- c1=1,c2=1: `SX · SX = X`. ✓
- c1=1,c2=0: CX flips c2→1 then back→0; net `SXdg · SX = I`. ✓
- c1=0,c2=1: `SX · SXdg = I`. ✓
- c1=0,c2=0: `I`. ✓

CX count: 3 × CSX/CSXdg (2 CX each) + 2 CX = 8 CX total.

**Optimized T-gate decomposition (6 CX):**

```
CCX_opt(c1, c2, t) =
    H(t) → CX(c2,t) → Tdg(t) → CX(c1,t) → T(t) → CX(c2,t) →
    Tdg(t) → CX(c1,t) → T(c2) → T(t) → H(t) → CX(c1,c2) →
    T(c1) → Tdg(c2) → CX(c1,c2)
```

Both produce the same 8×8 matrix.

**CCZGate — Doubly-Controlled Z**

```
CCZ(c1, c2, t) = H(t) → CCX(c1, c2, t) → H(t)
```

Only `|111⟩` picks up −1 phase. ✓

**CSwapGate — Fredkin / Controlled-SWAP**

```
CSWAP(c, t1, t2) = CX(t2, t1) → CCX(c, t1, t2) → CX(t2, t1)
```

Proof: First CX computes `t1 ⊕ t2` into t1. CCX flips t2 conditioned on both
control and XOR. Final CX restores t1.

**RCCXGate — Relative-Phase CCX (3 CX, direct Tier 0 + Tier 1)**

```
RCCX(c1, c2, t) =
    H(t) → T(t) → CX(c2, t) → Tdg(t) → CX(c1, t) →
    T(t) → CX(c2, t) → Tdg(t) → H(t)
```

Uses only 3 CX gates. The resulting 8×8 matrix is **not** equal to exact CCX:
the active `|110⟩,|111⟩` subspace is `[[0,-i],[i,0]]`, with additional relative
phases elsewhere. Treat as a distinct gate, not an exact CCX replacement inside
larger coherent subcircuits.

---

### Tier 5: Four+ Qubit and Multi-Controlled Gates

**Phase-Safe Multi-Control Architecture (general principle):**

Section 2 makes repeated control lifting **phase-sensitive**: controls lift the
full operator, not the operator modulo global phase. Consequently, a textbook
root recursion of the form

```
C^n(U)(c1,...,cn, t) =
    C(V)(cn, t) →
    C^{n-1}(X)(c1,...,cn-1, cn) →
    C(V†)(cn, t) →
    C^{n-1}(X)(c1,...,cn-1, cn) →
    C^{n-1}(V)(c1,...,cn-1, t)
```

is exact under this SDK's convention only when a **family-specific proof**
establishes that the chosen root chain is phase-safe under the exact controlled-
lifting rules of Section 2. In particular, the naive X-root ladder
`SX, SX^{1/2}, ...` is **not** phase-safe for exact multi-controlled X here: its
promoted root phases become unwanted relative phases on partially enabled
control subspaces.

Therefore the normative exact constructions in this tier are:

- `MCPhaseGate` below for multi-controlled phase;
- `MCXGate = H(target) → MCPhase(pi) → H(target)`;
- the ancilla-assisted V-chain route from Tier 6 for arbitrary single-target
  controlled `G` when clean ancillas are available; and
- any later exact synthesis of the same repeated-control-lifted matrix (for
  example exact `UnitaryGate` synthesis) when the implementation elects not to
  commit to a smaller family-specific decomposition.

Terminology note: in this tier, `MCPhaseGate(lambda, c1, ..., cN, t)` always
means the targetful repeated control lift `ctrl^N(P(lambda))` on the ordered
operand list `(c1, ..., cN, t)`, i.e. the `(N+1)`-qubit operator
`diag(1, ..., 1, exp(i*lambda))` whose last operand is an explicit target qubit.
For the empty-control base case `N = 0`, this family is defined here as the
ordinary targetful one-qubit gate `P(lambda)(t)` on that last operand only.
Section 2 also defines the controls-only family
`ctrl^N(GlobalPhaseGate(lambda))`, which has no target operands. Those two
families are related but distinct and MUST NOT be conflated by prose, naming, or
synthesis. Only the targetful `MCPhaseGate` family is used in the
`MCX =
H(target) → MCPhase(pi) → H(target)` reductions below.

An implementation MAY still use a Barenco-style root recursion for another gate
family only when it can derive the required roots exactly under Section 2
**and** separately prove that the resulting recursion preserves the exact
enabled- subspace phase semantics. Allowed exact root sources for such a proof
include:

- the exact same-family identities `RX(theta/2)^2 = RX(theta)`,
  `RY(theta/2)^2 = RY(theta)`, `RZ(theta/2)^2 = RZ(theta)`,
  `P(lambda/2)^2 = P(lambda)`, `R(theta/2, phi)^2 = R(theta, phi)`, and
  `RV(vx/2, vy/2, vz/2)^2 = RV(vx, vy, vz)`;
- another exact symbolic 2×2 construction; or
- a concrete numeric 2×2 spectral root whose exactness/certification satisfies
  Phase Convention 6.

If an implementation cannot discharge those exact proof obligations, it MUST
keep `C^n(U)` as a first-class semantic object (or reject that synthesis step)
rather than approximate, guess a branch, or silently substitute a non-equivalent
decomposition.

**C3XGate — Triple-Controlled X**

`C3X(c1, c2, c3, t)` is the `N = 3` specialization of `MCXGate`. The exact
reference construction is the phase-safe reduction to multi-controlled phase:

```
C3X(c1, c2, c3, t) = H(t) → MCPhase(pi, c1, c2, c3, t) → H(t)
```

Expanding the recursive `MCPhase` rule below yields a circuit over `H`, `CP`,
`CX`, and `CCX` only. No phaseful X-root ladder is used in the normative exact
construction.

**C3SXGate — Triple-Controlled SX**

`C3SX(c1, c2, c3, t)` denotes the exact matrix `ctrl^3(SX)` under the repeated
controlled-lifting rule of Section 2: identity on every computational-basis
state except the fully enabled `|1110⟩, |1111⟩` subspace, where `SX` acts on the
target.

Because the naive X-root ladder recursion is not exact under this SDK's phase
convention, it is **not** a normative reference decomposition for `C3SX`.
Instead, the normative exact reference synthesis is the ancilla-assisted V-chain
specialization of Tier 6 with exactly `2` clean scratch ancillas
`anc[0], anc[1]`:

```
C3SX_ref(c1, c2, c3, t; anc[0], anc[1]) =
    CCX(c1, c2, anc[0]) →
    CCX(c3, anc[0], anc[1]) →
    CSX(anc[1], t) →
    CCX(c3, anc[0], anc[1]) →
    CCX(c1, c2, anc[0])
```

These ancillas are not additional logical operands of `C3SX`; they are clean
scratch for this exact lowering only, must be pairwise distinct and disjoint
from `c1`, `c2`, `c3`, and `t`, must start in `|00⟩`, and must be restored to
`|00⟩` at template exit. This is exactly the `k = 3`, `m = 1`, `G = SX`
specialization of the Tier 6 `MCMTGate_vchain` template.

An implementation MAY instead use any other exact synthesis of the same
repeated-control-lifted matrix, such as exact `UnitaryGate` synthesis of
`ctrl^3(SX)`, but that is an alternative exact implementation path rather than
the normative reference decomposition.

If neither exact route is available at the current stage, `C3SX` MUST remain a
first-class semantic gate rather than be approximated or replaced by the
non-phase-safe root recursion.

**C4XGate — Quadruple-Controlled X (5 qubits)**

`C4X(c1, c2, c3, c4, t)` is the `N = 4` specialization of `MCXGate`, again via
the exact phase-safe reduction

```
C4X(c1, c2, c3, c4, t) = H(t) → MCPhase(pi, c1, c2, c3, c4, t) → H(t)
```

This replaces the non-phase-safe `C3SX`-based root recursion and remains exact
under the Section 2 controlled-phase rules.

**RC3XGate — Relative-Phase C3X (6 CX, direct Tier 0 + Tier 1)**

```
RC3X(c1, c2, c3, t) =
    H(t) → T(t) → CX(c3, t) → Tdg(t) → H(t) →
    CX(c1, t) → T(t) → CX(c2, t) → Tdg(t) →
    CX(c1, t) → T(t) → CX(c2, t) → Tdg(t) → H(t) →
    T(t) → CX(c3, t) → Tdg(t) → H(t)
```

The resulting 16×16 matrix is not equal to exact C3X. The active `|1110⟩,|1111⟩`
subspace is `[[0,1],[-1,0]]`, and the gate also applies additional relative
phases on other basis states (see the derived matrix in the reference section
below). Treat it as a distinct relative-phase gate, not as exact C3X with only
its enabled two-dimensional block changed.

**MCXGate — Multi-Controlled X (variable N controls)**

For `N` controls, the ordered semantic input may be written either as
`MCX(controls[0..N-1], target)` or as `MCX(c1, ..., cN, t)`; both denote the
same ordered control list and target.

- N=0: `X(t)`.
- N=1: CX.
- N≥2: exact phase-safe reduction to multi-controlled phase:
  `MCX(c1,...,cN, t) = H(t) → MCPhase(pi, c1,...,cN, t) → H(t)`

Because `H * P(pi) * H = X`, the fully enabled subspace sees `X` exactly, and
all other basis states see identity. After that single outer `MCPhase(pi, ...)`
call is expanded, the recursion below strictly reduces the number of controls
and bottoms out in `CP` and `CX`, so this definition still composes only Tier 0
and Tier 1 primitives.

The resulting matrix is `2^(N+1) × 2^(N+1)`: identity everywhere except the last
two rows/columns where X is applied. The matrix is **computed by composing the
phase-safe recursive decomposition**, not by directly constructing the large
matrix.

**MCPhaseGate — Multi-Controlled Phase (variable N controls)**

For `N` controls with parameter `lambda`, the ordered semantic input may be
written either as `MCPhase(lambda, controls[0..N-1], target)` or as
`MCPhase(lambda, c1, ..., cN, t)`; both denote the same ordered control list and
target.

- N=0: `P(lambda)(t)`.
- N=1: CP(lambda).
- N≥2: exact phase-safe recursion

  ```
  MCPhase(lambda, c1, ..., cN, t) =
      CP(lambda/2)(cN, t) →
      MCX(c1, ..., cN-1, cN) →
      CP(-lambda/2)(cN, t) →
      MCX(c1, ..., cN-1, cN) →
      MCPhase(lambda/2, c1, ..., cN-1, t)
  ```

  where in each recursive `MCX` call the final qubit (`cN`) is the target of
  that smaller multi-controlled X. For `N = 2`, this expands to

  ```
  MCPhase(lambda, c1, c2, t) =
      CP(lambda/2)(c2, t) → CX(c1, c2) →
      CP(-lambda/2)(c2, t) → CX(c1, c2) →
      CP(lambda/2)(c1, t)
  ```

This mutual recursion with `MCX` is well-founded because every recursive call
reduces the number of controls by one. Unlike the non-phase-safe X-root ladder,
it respects Section 2's exact controlled-lifting rules at every level.

The resulting matrix is `diag(1, ..., 1, exp(i*lambda))` with `exp(i*lambda)` in
the last diagonal position only.

---

### Tier 6: N-Qubit Structural Composite Gates

**MSGate — Mølmer-Sørensen Gate**

`MSGate(theta, m)` on `m` qubits produces a `2^m × 2^m` unitary:

`MS(theta) = exp(-i * (theta/2) * sum_{j<k} X_j ⊗ X_k)`

Validation / boundary requirements:

- `m` must be a nonnegative integer-valued arity under the frontend's documented
  exact-expression semantics.
- `m = 0` denotes the exact zero-qubit identity `GlobalPhaseGate(0)`.
- `m = 1` denotes the one-qubit identity `IGate()` on the sole qubit.

Since the pairwise `X_j ⊗ X_k` terms commute, the exponential factorizes:

```
MS(theta, m) = product over all pairs (j, k) with j < k of RXX(theta, j, k)
```

Each RXX is Tier 3 (RXX → RZZ → CX + single-qubit). Total pairs: `m*(m-1)/2`.
Because the factors commute, any pair order is mathematically exact; for a
deterministic concrete builder or serializer, use lexicographic order
`(0,1), (0,2), ..., (m-2,m-1)`.

**PauliGate — Pauli String Gate**

`PauliGate(pauliString)` given a string like `"XYZ"` computes `X ⊗ Y ⊗ Z`. The
string is read **left-to-right**: the leftmost character acts on the first qubit
(MSB of gate matrix index).

Validation / boundary requirements:

- Every character must be one of `{I, X, Y, Z}` exactly.
- The empty string denotes the exact zero-qubit identity `GlobalPhaseGate(0)`.

Decomposes into independent single-qubit Pauli operations — each character maps
to a Tier 0 gate. No entanglement (CX) needed; the matrix is a tensor product.

**DiagonalGate — Diagonal Unitary**

`DiagonalGate(phases, qubits)` acts on one explicit ordered qubit list
`q[0..n-1]`, where `n = len(q[0..n-1])`. Given exactly `2^n` phase angles
`[theta_0, ..., theta_{2^n-1}]`, indexed in the local MSB-first basis order of
that same ordered qubit list:

If a frontend/API stores the ordered operand list outside the constructor call
syntax, that same ordered list is still part of the semantic input to this gate
family and to every validation, equality, and serialization obligation below.

Validation requirements:

- `phases` must contain exactly `2^n` entries, where `n = len(q[0..n-1])`.
- In particular, `n = 0` requires exactly one entry `[theta_0]`.
- If the frontend accepts another exact finite representation of the same phase
  family, the implementation MUST prove exact correspondence to one ordered list
  of exactly `2^n` phase angles for that same ordered qubit list, or keep the
  gate opaque (or fail validation).
- Implementations MUST NOT truncate or pad the phase list, infer `n` by rounding
  `log2(len(phases))`, or silently reinterpret a non-power-of-two length as some
  other diagonal family.

Matrix: `diag(exp(i*theta_0), exp(i*theta_1), ..., exp(i*theta_{2^n-1}))`

Equivalently, if the ordered qubit list carries little-endian register value
`x = sum_{i=0}^{n-1} 2^i * x_i` with qubit `q[i] = x_i` under Section 2, then
the basis state with that register value receives the phase
`exp(i * theta_{brev_n(x)})`, where `brev_n(x)` denotes the integer obtained by
reversing the `n` bits of `x`, because the matrix itself is still indexed in the
local MSB-first order.

Conceptual forward reference: the exact recursive factorization below uses the
Tier 7 `UCRZ` multiplexor as a named subroutine.

**Decomposition (exact recursive factorization):**

For `n = 0`, this reduces to the zero-qubit phase `GlobalPhaseGate(theta_0)`.

For `n = 1`:

```
DiagonalGate([theta_0, theta_1], q[0]) =
    GlobalPhaseGate((theta_0 + theta_1)/2) →
    RZ(theta_1 - theta_0)(q[0])
```

For `n ≥ 2`, choose the last qubit argument `q[n-1]` as the target. Because the
local gate-matrix ordering is MSB-first, the phase list naturally groups into
consecutive pairs `(theta_{2j}, theta_{2j+1})` for each fixed basis state `j` of
`q[0..n-2]`. Define:

```
avg_j   = (theta_{2j} + theta_{2j+1}) / 2   for j = 0..2^{n-1}-1
delta_j =  theta_{2j+1} - theta_{2j}        for j = 0..2^{n-1}-1
```

Then:

```
DiagonalGate(phases, q[0..n-1]) =
    DiagonalGate(avg, q[0..n-2]) →
    UCRZ(delta, q[0..n-2], q[n-1])
```

For a fixed control basis state `j`, the second factor contributes
`RZ(delta_j) = diag(exp(-i*delta_j/2), exp(i*delta_j/2))`, while the first
factor contributes the common scalar `exp(i*avg_j)` on both target states. Their
product is therefore exactly `diag(exp(i*theta_{2j}), exp(i*theta_{2j+1}))`.

CX count: `2^n - 2` for `n ≥ 1`. RZ count: `2^n - 1`. Zero-qubit phase count:
exactly one explicit `GlobalPhaseGate` in the reference decomposition. A
normalized non-source-preserving representation MAY instead fold that same phase
into the immediate owning scope's scalar `globalPhase` only when the **fold-safe
same-scope rule** holds and the representation also preserves the recoverability
guarantees required by Section 2.

**PermutationGate — Permutation of Basis States**

`PermutationGate(sigma, qubits)` acts on one explicit ordered qubit list
`q[0..n-1]`, where `n = len(q[0..n-1])`. Given a permutation
`sigma: {0, ..., 2^n-1} → {0, ..., 2^n-1}`:

If a frontend/API stores the ordered operand list outside the constructor call
syntax, that same ordered list is still part of the semantic input to this gate
family and to every validation, equality, and serialization obligation below.

Validation requirements:

- `sigma` must be a bijection on `{0, ..., 2^n-1}`.
- Every image `sigma(j)` must lie in `{0, ..., 2^n-1}`.
- If the frontend accepts another exact finite representation of the same
  permutation family, the implementation MUST prove exact bijectivity, exact
  range membership, and exact agreement with this same ordered qubit-list width
  `n`, or keep the gate opaque (or fail validation).
- Implementations MUST NOT silently drop repeated outputs, wrap out-of-range
  values modulo `2^n`, or coerce a non-bijection into a partial permutation.

Matrix: `P[sigma(j), j] = 1` for all `j`, zero elsewhere.

**Decomposition (exact cycle decomposition + adjacent basis transpositions):**

`sigma`, every integer basis label `a`, `b`, `u`, `v`, and every bit position
`p_s` in this construction are interpreted in the **little-endian register
value** convention of Section 2: for the ordered qubit list `q[0..n-1]`, bit `i`
of such an integer is the value of qubit `q[i]`. The matrix is still written in
the local MSB-first ordering of Section 2, so `P[sigma(j), j] = 1` means:
convert the column label `j` to its computational basis state via that
little-endian identification, apply `sigma` to the resulting register value, and
place the `1` in the row corresponding to the resulting basis state under the
same identification.

1. Decompose `sigma` into disjoint cycles using the standard convention
   `(v_0 v_1 ... v_{m-1}) : v_0 -> v_1 -> ... -> v_{m-1} -> v_0`. Because
   decompositions in this section are written in circuit time order, such a
   cycle is implemented as `T(v_0, v_1) → T(v_0, v_2) → ... → T(v_0, v_{m-1})`,
   where `T(a, b)` swaps exactly the two computational basis states `|a⟩` and
   `|b⟩`. The reverse order would implement the inverse cycle and therefore must
   not be used here.

2. To implement `T(a, b)`, let `p_1, ..., p_r` be the bit positions where `a`
   and `b` differ. Define a Gray-path of basis states `g_0 = a`,
   `g_s = a XOR (2^{p_1} XOR ... XOR 2^{p_s})` for `s = 1..r`, so `g_r = b` and
   each adjacent pair `(g_{s-1}, g_s)` differs in exactly one bit `p_s`.

3. For each adjacent pair `(u, v)` on that path, define `A(u, v)` as the
   single-bit flip on the differing bit, conditioned on all other qubits
   matching the common bit pattern of `u` and `v`. If the shared pattern
   requires a control qubit to be 0, realize that negative control by
   conjugating that qubit with `X` before and after the `MCX`, exactly as
   required by Phase Convention 4.

4. Then the endpoint transposition is:

   ```
   T(a, b) =
       A(g_0, g_1) → A(g_1, g_2) → ... → A(g_{r-1}, g_r) →
       A(g_{r-2}, g_{r-1}) → ... → A(g_0, g_1)
   ```

   This is the standard adjacent-transposition identity on a path: it swaps the
   endpoints `g_0` and `g_r` while restoring every intermediate basis state.

Each adjacent basis transposition uses one `MCX` (or one `X` when `n = 1`), so a
single endpoint transposition uses at most `2r - 1 ≤ 2n - 1` multi-controlled X
gates. A general permutation therefore uses `O(n * 2^n)` multi-controlled gates
in the worst case. When `sigma` is a linear reversible function over GF(2), use
`LinearFunction` instead (Tier 7): its normative reference synthesis is `O(n^2)`
CX-equivalent two-qubit gates, and an implementation that also provides the
optional Patel-Markov-Hayes optimization from Tier 7 may improve that to
`O(n^2 / log n)`.

**MCMTGate — Multi-Controlled Multi-Target Gate**

`MCMTGate(gate, controls, targets)` applies a single-qubit gate `G` to each
target qubit in `targets`, all controlled by the qubits in `controls`.

Validation requirements:

- `controls` must be duplicate-free.
- `targets` must be duplicate-free.
- No qubit may appear in both `controls` and `targets`.
- If either list contains duplicates, or if any qubit appears in both roles,
  validation MUST fail (or the gate MUST remain opaque). This specification does
  **not** assign a separate semantics to repeated targets or overlapping
  control/target roles.

Ancilla ownership contract:

- `MCMTGate` itself acts only on its logical `controls` and `targets`.
- Any ancillas mentioned by a concrete exact synthesis template below are clean
  scratch qubits for that lowering, not additional logical operands of the
  semantic gate.
- A frontend MAY expose such scratch through a separate low-level synthesis
  helper/API. Otherwise the implementation/backend is responsible for providing
  the required clean ancillas explicitly, or for rejecting/deferring that exact
  lowering while preserving the same semantic `MCMTGate`.

```
MCMTGate(G, controls[0..k-1], targets[0..m-1]) =
    if k = 0:
        for each target t in targets:
            G(t)
    else:
        for each target t in targets:
            C^k(G)(controls[0..k-1], t)
```

The `k = 0` case is the ordinary uncontrolled fan-out of the same single-qubit
gate onto each target. For `k >= 1`, the logical action is exact for any
single-qubit unitary `G`, but the concrete decomposition path depends on what
exact synthesis data is available:

Here and below, `C(G)` is only shorthand for the exact singly-controlled lifting
of the full 2×2 single-qubit operator `G` under Section 2's controlled-lifting
rule. It is not an additional primitive gate family beyond the Tier 2 controlled
families already defined. When `G` is a named Tier 0 family, use the
corresponding exact Tier 2 instance when one is given explicitly in this section
(`CP`, `CRX`, `CRY`, `CRZ`, `CH`, `CSX`, etc.); otherwise lower `C(G)` by the
general controlled-U construction above after expressing `G` exactly as an
internal canonical single-qubit operator. If the active stage cannot obtain that
exact single-qubit representation yet, `C(G)` (and therefore the enclosing
`MCMTGate`) MUST remain a first-class semantic object rather than be guessed or
approximated.

- In the no-ancilla route, each `C^k(G)` may use a phase-safe exact family-
  specific recursion from Tier 5 only when the required exact root chain for `G`
  is available under the rule above **and** the implementation has a separate
  proof that the chosen recursion preserves the Section 2 control-phase
  semantics.
- In the ancilla-assisted route, the V-chain optimization below is exact for
  **any** single-qubit `G`, because it uses only Toffoli ladders plus the
  singly-controlled base case `C(G)` from Tier 2.
- If neither condition holds, `MCMTGate` MUST remain as a first-class semantic
  object (or the synthesis pass MUST reject) until a later stage can supply an
  exact decomposition. Implementations MUST NOT approximate or guess square
  roots for `G`.

Whenever a concrete exact synthesis is available, the CX count is
`m * CX_count(G)` when `k = 0` (zero for Tier 0 single-qubit gates) and
`m * CX_count(C^k(G))` when `k >= 1` for the chosen exact decomposition.

**V-chain optimization:** This is an exact ancilla-assisted decomposition of the
same logical `MCMTGate`, not a distinct gate semantics. The ladder form below
assumes `k >= 2` and requires exactly `k - 1` ancillas `ancillas[0..k-2]`. Those
ancillas MUST be pairwise distinct, disjoint from all controls and targets,
initialized to `|0...0⟩`, and uncomputed back to `|0...0⟩` by the end of the
template. Under those preconditions, the multi-controlled gate can be decomposed
more efficiently using a chain of Toffoli gates to progressively compute the AND
of controls into ancillas. If `k = 0` or `k = 1`, or if such clean ancillas are
unavailable, implementations MUST use the direct definition above instead of
this ancilla-assisted variant:

```
MCMTGate_vchain(G, controls[0..k-1], targets[0..m-1], ancillas[0..k-2]) =
    // Compute AND ladder
    CCX(controls[0], controls[1], ancillas[0])
    for i = 2 to k-1:
        CCX(controls[i], ancillas[i-2], ancillas[i-1])
    // Apply G to each target, controlled by the last ancilla
    for each target t in targets:
        C(G)(ancillas[k-2], t)
    // Uncompute AND ladder (reverse)
    for i = k-1 downto 2:
        CCX(controls[i], ancillas[i-2], ancillas[i-1])
    CCX(controls[0], controls[1], ancillas[0])
```

This uses `2*(k-1)` Toffoli gates + `m` singly-controlled gates.

**PauliProductRotationGate — Pauli Tensor Rotation**

`PauliProductRotationGate(theta, paulis)` implements
`exp(-i * theta/2 * P_1 ⊗ P_2 ⊗ ... ⊗ P_n)` where each `P_k ∈ {I, X, Y, Z}`. The
ordered factors use the same contract as `PauliGate`: if `paulis` is given as a
string, it is read left-to-right; in any equivalent ordered-list form, element
`0` acts on the first qubit argument (MSB of the local gate-matrix index),
element `1` on the second, and so on.

**Decomposition:**

If every symbol in `paulis` is `I`, then the operator is `exp(-i * theta/2) * I`
on the full register. By Phase Convention 3, represent this exactly as the
explicit zero-qubit phase `GlobalPhaseGate(-theta/2)` and stop. A normalized
non-source-preserving representation MAY instead fold that same phase into the
immediate owning scope's scalar `globalPhase` only when the **fold-safe
same-scope rule** holds and the representation also preserves the recoverability
guarantees required by Section 2.

Otherwise:

1. **Basis change:** For each qubit `k` where `P_k ≠ I`:
   - If `P_k = X`: apply `H(k)` (since `H*Z*H = X`)
   - If `P_k = Y`: apply `RX(pi/2)(k)` (since `RX(-pi/2)*Z*RX(pi/2) = Y`)
   - If `P_k = Z`: no change needed

2. **CX parity ladder:** Let `active = [k for k where P_k ≠ I]`, ordered.
   ```
   for i = 0 to len(active) - 2:
       CX(active[i], active[i+1])
   ```

3. **RZ rotation:** `RZ(theta)(active[last])`

4. **Undo CX ladder (reverse order):**
   ```
   for i = len(active) - 2 downto 0:
       CX(active[i], active[i+1])
   ```

5. **Undo basis change:** Reverse of step 1: undo with `H` for X and `RX(-pi/2)`
   for Y.

CX count: if `|active| = 0`, zero CX and one zero-qubit phase; otherwise
`2 * (|active| - 1)`. Single-qubit count: if `|active| = 0`, zero; otherwise let
`basisChanged = |{ k : P_k ∈ {X, Y} }|`; the total is `2 * basisChanged + 1`
(two basis-change gates for each X/Y factor, plus the central `RZ(theta)`).

This generalizes RXX, RYY, RZZ, RZX to arbitrary Pauli strings of any length.

---

### Tier 7: Uniformly Controlled Gates and General Unitary Synthesis

**UCRZGate — Uniformly Controlled RZ**

`UCRZGate(angles, controls[0..k-1], target)` where `angles` has `2^k` entries.
The selector list is indexed in the same local MSB-first control-block order
used by the block-diagonal matrix below. Equivalently, if the ordered control
list carries little-endian register value `x = sum_{i=0}^{k-1} 2^i * c_i` with
qubit `controls[i] = c_i` under Section 2, then the selected entry is
`angles[brev_k(x)]`, where `brev_k(x)` denotes the integer obtained by reversing
the `k` bits of `x`. In other words, when the control subspace is the block with
local MSB-first index `j`, apply `RZ(angles[j])` to the target.

Matrix: block diagonal
`diag(RZ(angles[0]), RZ(angles[1]), ..., RZ(angles[2^k-1]))`. Since each `RZ` is
diagonal, the full matrix is diagonal: `D[2j, 2j] = exp(-i*angles[j]/2)`,
`D[2j+1, 2j+1] = exp(i*angles[j]/2)`.

Validation requirements (shared with `UCRYGate`, `UCRXGate`, and
`UCPauliRotGate` below):

- Let `k = len(controls)`.
- `angles` must contain exactly `2^k` entries. In particular, `k = 0` requires
  exactly one entry.
- Every angle entry must be real-valued under the frontend's documented
  exact-expression semantics.
- If the frontend accepts another exact finite representation of the same
  selector family, the implementation MUST prove exact correspondence to one
  ordered list of exactly `2^k` real angles for that same ordered control list,
  or keep the gate opaque (or fail validation).
- Implementations MUST NOT truncate or pad the selector list, infer `k` by
  rounding `log2(len(angles))`, silently broadcast a shorter list, or coerce a
  non-real angle expression into a real rotation angle.

**Decomposition (exact Gray-code multiplexor):**

For `k = 0` (0 controls, 1 angle `theta_0`):

```
UCRZ([theta_0], t) = RZ(theta_0)(t)
```

For `k = 1` (1 control, 2 angles `theta_0, theta_1`):

```
UCRZ([theta_0, theta_1], [c], t) =
    RZ((theta_0 + theta_1)/2)(t) → CX(c, t) → RZ((theta_0 - theta_1)/2)(t) → CX(c, t)
```

When c=0: target sees `RZ((th0-th1)/2) * RZ((th0+th1)/2) = RZ(th0)`. ✓ When c=1:
`X * RZ((th0-th1)/2) * X = RZ(-(th0-th1)/2)`, so target sees
`RZ(-(th0-th1)/2) * RZ((th0+th1)/2) = RZ(th1)`. ✓

For general `k >= 1`, let `g_0, ..., g_{2^k-1}` be the binary-reflected Gray
code on `k` bits and define `g_{2^k} = g_0`. For each `r`, let `flip(r)` be the
unique control index whose bit changes between `g_r` and `g_{r+1}`. Compute the
transformed rotation angles:

- Interpret every selector integer `j` and every Gray-code word `g_r` in the
  same local MSB-first control-block order used by the block diagonal
  `diag(U_0, U_1, ..., U_{2^k-1})`.
- Under Section 2's little-endian register convention for the ordered control
  list, this means Gray-code bit 0 corresponds to `controls[k-1]`, bit 1 to
  `controls[k-2]`, ..., bit `k-1` to `controls[0]`.
- Therefore, if the differing Gray-code bit between `g_r` and `g_{r+1}` is bit
  position `b`, then `flip(r) = k - 1 - b`.

```
alpha_r = (1 / 2^k) * sum_{j=0}^{2^k-1} (-1)^{popcount(g_r & j)} * angles[j]
```

Then the exact multiplexor circuit is:

```
UCRZ(angles, c[0..k-1], t) =
    for r = 0 to 2^k - 1:
        RZ(alpha_r)(t) →
        CX(c[flip(r)], t)
```

This standalone form uses `0` CX gates and `1` RZ gate when `k = 0`, and `2^k`
CX gates plus `2^k` RZ gates when `k >= 1`. Adjacent CX gates may cancel only
when this multiplexor is fused with neighboring multiplexor layers during
optimization.

**UCRYGate — Uniformly Controlled RY**

Use the same exact Gray-code multiplexor construction as UCRZGate, with the same
transformed angles `alpha_r` and the same control-bit ordering convention, but
replace every `RZ` with `RY`.

`UCRYGate` inherits the same validation requirements on `angles`, control-list
width, and exact real-valuedness as `UCRZGate`.

For `k = 0`:

```
UCRY([theta_0], t) = RY(theta_0)(t)
```

For `k = 1`:

```
UCRY([theta_0, theta_1], [c], t) =
    RY((theta_0 + theta_1)/2)(t) → CX(c, t) → RY((theta_0 - theta_1)/2)(t) → CX(c, t)
```

For `k >= 2`, use the same Gray-code construction as UCRZGate with `RY` in place
of `RZ`. CX count is `0` for `k = 0` and `2^k` for `k >= 1`. RY count is `1` for
`k = 0` and `2^k` for `k >= 1`.

**UCRXGate — Uniformly Controlled RX**

Implemented by conjugating UCRZ with Hadamard:

`UCRXGate` inherits the same validation requirements on `angles`, control-list
width, and exact real-valuedness as `UCRZGate`.

For `k = 0`:

```
UCRX([theta_0], t) = RX(theta_0)(t)
```

```
UCRX(angles, c[0..k-1], t) = H(t) → UCRZ(angles, c[0..k-1], t) → H(t)
```

For `k >= 1`, the Hadamard conjugation is exact because
`H * RZ(theta) * H =
RX(theta)`. CX count is `0` for `k = 0` and `2^k` for
`k >= 1`.

**UCPauliRotGate — Uniformly Controlled Pauli Rotation**

`UCPauliRotGate(angles, axis, controls, target)` where `axis ∈ {X, Y, Z}`.
Dispatches to UCRXGate, UCRYGate, or UCRZGate based on axis. In particular, with
an empty control list it reduces exactly to the corresponding uncontrolled
single-qubit rotation.

Validation requirements:

- `axis` must be exactly one of `{X, Y, Z}`. Invalid axis values MUST fail
  validation. Implementations MUST NOT case-fold, alias-map, or heuristically
  coerce another token into one of these three axes.
- `UCPauliRotGate` inherits the same validation requirements on `angles`,
  control-list width, and exact real-valuedness as `UCRZGate`.

**UCGate — General Uniformly Controlled Gate**

`UCGate(unitaries, controls[0..k-1], target)` where `unitaries` contains `2^k`
arbitrary 2×2 unitaries `[U_0, ..., U_{2^k-1}]`. The list is indexed in the same
local MSB-first control-block order as `UCRZ`/`UCRY`/`UCRX` and the
block-diagonal matrix below. Equivalently, if the ordered control list carries
little-endian register value `x` under Section 2, then the selected unitary is
`U_{brev_k(x)}`, where `brev_k(x)` denotes the integer obtained by reversing the
`k` bits of `x`. In other words, when the control subspace is the block with
local MSB-first index `j`, apply `U_j` to the target.

Matrix: `2^{k+1} × 2^{k+1}` block diagonal `diag(U_0, U_1, ..., U_{2^k-1})`.

Validation requirements:

- `unitaries` must contain exactly `2^k` entries.
- Every `U_j` must be a `2 × 2` matrix.
- For concrete numeric input, each `U_j` must satisfy the same known-unitary
  certification rule as Section 2: both `U_j† * U_j ≈ I` and `U_j * U_j† ≈ I`
  entrywise within epsilon `1e-10`.
- For symbolic input, unitary status of every `U_j` must already be established
  exactly. If any entry is unresolved, `UCGate` MUST remain opaque (or
  validation MUST fail) until a later stage can prove exact unitarity.
- Implementations MUST NOT silently renormalize, orthogonalize, or otherwise
  "repair" a non-unitary input list before synthesis.

Canonical single-qubit ZYZ representative used by this tier:

Whenever this tier says "compute the ZYZ decomposition" of a `2 × 2` unitary, it
means: first obtain any exact (or, for concrete numeric input, numerically
validated) tuple `(alpha_raw, beta_raw, gamma_raw, delta_raw)` satisfying

`U = exp(i*alpha_raw) * RZ(beta_raw) * RY(gamma_raw) * RZ(delta_raw)`

and then canonicalize it to `(alpha, beta, gamma, delta)` as follows:

- Whenever this canonicalization replaces an `RZ` angle `eta_1` by
  `eta = wrapPhase(eta_1)`, it MUST also compensate the scalar phase by
  `pi * n_eta` where `eta_1 = eta + 2*pi*n_eta`, because
  `RZ(eta_1) = exp(i*pi*n_eta) * RZ(eta)` exactly. The bullets below already
  include this required compensation; it MUST NOT be omitted.

- For concrete floating-point inputs, first reduce `gamma_raw` to
  `gamma0 = wrapPhase(gamma_raw)`, and let `n_gamma` be the unique integer
  satisfying `gamma_raw = gamma0 + 2*pi*n_gamma`. Because
  `RY(gamma_raw) = exp(i*pi*n_gamma) * RY(gamma0)` exactly, this wrap MUST first
  be compensated in the scalar phase. If `gamma0 < 0`, replace the raw tuple by
  the exact reflected tuple
  `(alpha_1, beta_1, gamma_1, delta_1) =
  (alpha_raw + pi*n_gamma, beta_raw + pi, -gamma0, delta_raw - pi)`,
  which satisfies the same unitary exactly. Otherwise set
  `(alpha_1, beta_1, gamma_1, delta_1) =
  (alpha_raw + pi*n_gamma, beta_raw, gamma0, delta_raw)`.
  After this step, `gamma_1 ∈ [0, pi]`.
- For concrete floating-point inputs, classify the endpoint cases with the same
  epsilon used by Section 2: if `|gamma_1| <= epsilon`, set `gamma = 0`; else if
  `|gamma_1 - pi| <= epsilon`, set `gamma = pi`; otherwise set
  `gamma = gamma_1`. Implementations MUST branch on this snapped value, not on
  an unsnapped floating-point comparison.
- For concrete floating-point inputs, in the generic case `0 < gamma < pi`, set
  `beta = wrapPhase(beta_1)` and `delta = wrapPhase(delta_1)`. Let `n_beta` and
  `n_delta` be the unique integers satisfying `beta_1 = beta + 2*pi*n_beta` and
  `delta_1 = delta + 2*pi*n_delta`; then set
  `alpha = wrapPhase(alpha_1 + pi * (n_beta + n_delta))`.
- For concrete floating-point inputs, if `gamma = 0`, only
  `sigma = beta_1 + delta_1` matters. Canonicalize by setting `beta = 0`,
  `delta = wrapPhase(sigma)`, let `n_sigma` be the unique integer satisfying
  `sigma = delta + 2*pi*n_sigma`, and set
  `alpha = wrapPhase(alpha_1 + pi * n_sigma)`.
- For concrete floating-point inputs, if `gamma = pi`, only
  `tau = beta_1 - delta_1` matters. Canonicalize by setting `delta = 0`,
  `beta = wrapPhase(tau)`, let `n_tau` be the unique integer satisfying
  `tau = beta + 2*pi*n_tau`, and set `alpha = wrapPhase(alpha_1 + pi * n_tau)`.
- For symbolic or other exact-scalar input, the same equations are available
  only when the implementation can prove every required step exactly under its
  documented exact-expression semantics: the value of each `wrapPhase(...)` call
  it intends to materialize, the corresponding integer wrap counts (`n_gamma`,
  `n_beta`, `n_delta`, `n_sigma`, `n_tau`) when used, the sign test
  `gamma0 < 0`, and the endpoint classifications `gamma = 0` or `gamma = pi`.
  Section 2's prohibition on unproved symbolic modulo-`2*pi` rewrites applies
  here in full: an implementation MUST NOT replace a symbolic angle by
  `wrapPhase(angle)` merely as syntactic normalization unless that rewrite and
  the required `alpha` compensation are themselves proved exact under the active
  exact-expression semantics.
- If those exact symbolic proof obligations cannot be discharged, the containing
  synthesis step MUST remain opaque (or reject) until parameters are concrete
  enough to choose the canonical representative exactly.

These rules define the portable representative expected by `UCGate`, the `n = 1`
case of `UnitaryGate`, `decomposeZYZ(...)`, and the associated canonicalization
tests elsewhere in this specification.

**Decomposition (Möttönen et al., 2004):**

1. For each `U_j`, compute the ZYZ decomposition:
   `U_j = exp(i*alpha_j) * RZ(beta_j) * RY(gamma_j) * RZ(delta_j)`

2. Apply four exact layers:
   ```
   UCGate(unitaries, c[0..k-1], t) =
       UCRZ({delta_j}, c, t) →
       UCRY({gamma_j}, c, t) →
       UCRZ({beta_j}, c, t) →
       DiagonalGate({alpha_j}, c)  // diagonal diag(exp(i*alpha_0), ..., exp(i*alpha_{2^k-1}))
   ```

   The last step is a `2^k × 2^k` diagonal gate on the control qubits whose
   diagonal entries are `exp(i*alpha_j)`. `DiagonalGate` takes the phase angles
   `alpha_j` themselves, not pre-exponentiated complex scalars.

CX count for `k ≥ 1`: `3 * 2^k + (2^k - 2) = 4 * 2^k - 2` (three UCR layers +
one diagonal). For `k = 0`, `UCGate` is just the single unitary `U_0`.

**UnitaryGate — Arbitrary Unitary Matrix**

`UnitaryGate(U, qubits)` applies an arbitrary `2^n × 2^n` unitary `U` to `n`
qubits.

Validation requirements:

- `U` must be square with dimension `2^n × 2^n`.
- For concrete numeric input, `U` must satisfy the same known-unitary
  certification rule as Section 2: both `U† * U ≈ I` and `U * U† ≈ I` entrywise
  within epsilon `1e-10`.
- For symbolic input, unitarity must already be established exactly. If exact
  unitarity is unresolved, the gate MUST remain opaque (or validation MUST fail)
  until a later stage can prove it exactly.

**Decomposition (reference synthesis: symbolically exact, numerically validated
in concrete mode, via a recursive CSD skeleton plus primitive-level controlled
lifting, following Shende/Bullock/Markov 2005):**

For `n = 0`, `U` is a `1 × 1` unitary `[[exp(i*alpha)]]`. Realize it as

```
UnitaryGate(U, []) = GlobalPhaseGate(alpha)
```

For concrete numeric input, the scalar angle returned by this base case MUST be
canonicalized as `wrapPhase(arg(U[0][0]))`. For symbolic input, `alpha` must be
preserved exactly when derivable; otherwise the gate MUST remain opaque (or that
synthesis step MUST reject) rather than guessing a scalar branch.

For `n = 1`, use the canonical single-qubit ZYZ representative defined above:

`U = exp(i*alpha) * RZ(beta) * RY(gamma) * RZ(delta)`

and, because this section writes circuits in circuit-time order while the
displayed factorization is in matrix-product order, realize it as

```
UnitaryGate(U, q[0]) =
    GlobalPhaseGate(alpha) → RZ(delta)(q[0]) → RY(gamma)(q[0]) → RZ(beta)(q[0])
```

The scalar phase is part of the exact unitary semantics here. In symbolic mode
it MUST be preserved exactly; in concrete numeric mode it MUST be preserved up
to the same validated numeric acceptance rules and MUST NOT be dropped as a mere
"equal up to global phase" simplification. A normalized non-source-preserving
representation MAY equivalently fold that same zero-qubit phase `alpha` into the
immediate owning scope's scalar `globalPhase` only when the **fold-safe
same-scope rule** holds and the representation also preserves the recoverability
guarantees required by Section 2.

For `n ≥ 2`: choose the first qubit argument `q[0]` (the MSB of the local
gate-matrix index under Section 2) as the pivot qubit and view `U` as a `2 × 2`
block matrix over that qubit:

```
U = [[A, B], [C, D]]
```

where each block is `2^{n-1} × 2^{n-1}`. Apply the cosine-sine decomposition:

```
U =
    [[L0, 0], [0, L1]] *
    [[C, -S], [S, C]] *
    [[R0, 0], [0, R1]]
```

with `L0, L1, R0, R1` unitary and `C = diag(cos(phi_r))`,
`S = diag(sin(phi_r))`, so `C^2 + S^2 = I`.

This CSD step fixes the target operator, not one unique public intermediate
presentation. When equal singular angles or other exact CSD freedoms occur, any
deterministic exact/validated choice of `(L0, L1, {phi_r}, R0, R1)` is
conforming provided the recomposed product equals the same input `U` exactly (or
within the prescribed numeric epsilon in concrete mode) and the subsequent
recursive synthesis uses that same factorization consistently. APIs that expose
intermediate CSD factors or require source-stable decomposition traces MUST
specify additional deterministic ordering/sign/phase conventions; otherwise
portable conformance tests for `UnitaryGate` compare only the final emitted
operator, not the particular intermediate CSD presentation.

Because this factorization is written in matrix-product order, the concrete
circuit-time sequence for one recursive synthesis step is:

```
rightFactor(q[0..n-1]) → middleFactor(q[0..n-1]) → leftFactor(q[0..n-1])
```

where `rightFactor = diag(R0, R1)` and `leftFactor = diag(L0, L1)`.

- The middle factor acts on each pair `|0⟩⊗|r⟩`, `|1⟩⊗|r⟩` as the real
  two-dimensional rotation
  `[[cos(phi_r), -sin(phi_r)], [sin(phi_r), cos(phi_r)]]`, i.e. `RY(2 * phi_r)`
  on the pivot for each fixed basis state `r` of `q[1..n-1]`.
- Because Tier 7 `UCRYGate` is defined with **controls first, target last**,
  while this CSD middle factor is written with the pivot qubit first, realize it
  exactly by calling `UCRY` with the remaining qubits as controls and the pivot
  as the target:

  ```
  middleFactor(q[0..n-1]) =
      UCRY({2 * phi_r}, q[1], ..., q[n-1], q[0])
  ```

  No physical qubit permutation is inserted here. Under Section 2, the ordered
  operand list of `UCRY` already fixes the local MSB-first control-block basis,
  so `UCRY({2 * phi_r}, q[1], ..., q[n-1], q[0])` acts as the required block
  diagonal direct sum of `RY(2 * phi_r)` on the pivot, indexed by the basis
  states `r` of `q[1..n-1]`.
- Each outer block-diagonal factor `diag(L0, L1)` or `diag(R0, R1)` is a
  selector-conditioned unitary on the remaining `n-1` qubits with selector
  `q[0]`. To synthesize such a factor exactly:
  1. Recursively synthesize concrete circuits for the smaller unitaries (`L0`,
     `L1` or `R0`, `R1`) on `q[1..n-1]`.
  2. Lower those recursive circuits completely to explicit Tier 0 zero-qubit and
     single-qubit gates plus Tier 1 `CX`, preserving any branch-local scalar
     phase exactly. If a recursive result is represented as
     `L_b = exp(i*gamma_b) * V_b` or `R_b = exp(i*gamma_b) * V_b`, materialize
     that scalar as an explicit `GlobalPhaseGate(gamma_b)` instruction inside
     the branch before adding controls; it MUST NOT be discarded or absorbed
     into an uncontrolled surrounding scope.
  3. Re-emit every instruction of the `0`-branch circuit, including any explicit
     `GlobalPhaseGate`, with an exact **negative** control on `q[0]`, and every
     instruction of the `1`-branch circuit with an exact **positive** control on
     `q[0]`.

     Because step 2 fully lowers each branch circuit to Tier 0 zero-qubit and
     single-qubit gates plus Tier 1 `CX` before any new selector control is
     added, this step never requires an unspecified higher-arity controlled-
     rotation family. Each re-emitted instruction is controlled exactly as
     follows:
     - for a `1`-branch instruction, a Tier 0 single-qubit gate becomes its
       exact singly-controlled Tier 2 form (or the corresponding exact `CU` /
       `CP` / `CR*` instance derived there); for a `0`-branch instruction, use
       that same exact positive-control construction conjugated on the selector
       qubit `q[0]` by `X` before and after, exactly as required by Phase
       Convention 4 for negative controls;
     - for a `1`-branch `GlobalPhaseGate(theta)`, emit the selector phase gate
       `P(theta)` by Phase Convention 4; for a `0`-branch
       `GlobalPhaseGate(theta)`, emit `X(q[0]) → P(theta)(q[0]) → X(q[0])`, the
       exact negative-control lifting of that zero-qubit phase; and
     - for a `1`-branch `CX`, emit `CCX`; for a `0`-branch `CX`, emit the
       corresponding negative-controlled Toffoli, realized by conjugating the
       selector qubit with `X` before and after that same `CCX`.

     If an implementation cannot complete this primitive-level lowering exactly,
     it MUST keep the surrounding block-diagonal factor as a first-class
     semantic object (or reject that synthesis stage) rather than guess a larger
     controlled decomposition.

     By Phase Convention 4, positive and negative controls are exact matrix
     liftings, and controlling `GlobalPhaseGate(theta)` yields the required
     branch-selective phase on the selector qubit. The enabled subspaces are
     therefore disjoint and the product is exactly
     `|0⟩⟨0| ⊗ L0 + |1⟩⟨1| ⊗ L1 = diag(L0, L1)` (and similarly for the right
     factor), including any relative phase difference between the two branches.
     This closes the recursion exactly: each recursive step reduces the
     uncontrolled target size from `n` qubits to `n-1` qubits, and the added
     controls apply only to Tier 0 gates and `CX`, whose exact controlled forms
     are already specified elsewhere in this section.

     This branchwise controlled-primitive lifting is the normative exact
     reference path for the outer block-diagonal factors because it makes every
     branch-local scalar phase explicit before adding the selector control, so
     Section 2's exact controlled-lifting rule is obeyed instruction by
     instruction. It is intentionally phase-transparent rather than CX-minimal.
     Lower-count multiplexor/CSD implementations from the literature are
     conforming only if they preserve the same full matrix, including
     branch-selective phase, exactly.

Informative note: some implementations may prefer a QR-style exact synthesis.
This document does **not** normatively specify such a variant. Any QR-based
alternative is conforming only if it preserves the same exact matrix and phase
semantics as the CSD reference synthesis above. The sketch below is
informational only:

1. For each column of `U` (from right to left), use uniformly controlled
   rotations to zero out entries, reducing the matrix one column at a time.
2. Each step produces uniformly controlled RY and RZ gates.
3. The circuit is the reverse sequence of these gates.

CX count for this exact reference path remains `O(4^n)`, but the exact constant
depends on the recursive branch structure and on the primitive controlled-
lowering choices inside the block-diagonal factors; this specification does not
fix a sharper universal closed form for that path. An implementation MAY
substitute another exact CSD/multiplexor realization with a proven lower CX
count only when it preserves the same full matrix and phase semantics under
Section 2.

**LinearFunction — Reversible Linear Function over GF(2)**

`LinearFunction(M, qubits)` where `M` is an `n × n` invertible binary matrix.
Implements `|x⟩ → |M·x mod 2⟩`.

Here `x` is the little-endian register-value column vector
`(x_0, ..., x_{n-1})^T` associated with the ordered qubit list `q[0..n-1]`:
`x_i` is the value of qubit `q[i]`, equivalently bit `i` of the register value
under Section 2. The output vector `M·x mod 2` is written back to that same
ordered qubit list. As elsewhere in Section 3, the gate matrix is still indexed
in the local MSB-first ordering of Section 2.

This is a permutation of computational basis states. The matrix is a permutation
matrix determined by `M`.

Validation requirements:

- `M` must have dimension `n × n`, where `n = len(qubits)`.
- `n = 0` denotes the exact zero-qubit identity `GlobalPhaseGate(0)`.
- Every entry of `M` must be exactly `0` or `1` under the frontend's documented
  exact-expression semantics.
- `M` must be invertible over GF(2).
- If the frontend accepts symbolic or generic numeric entries, the
  implementation MUST prove exact bit-valuedness and exact GF(2) invertibility,
  or keep the gate opaque (or fail validation). It MUST NOT round near-binary
  entries, reduce arbitrary integers modulo `2` implicitly, or assume
  invertibility from floating-point rank heuristics.

**Decomposition (Gaussian elimination):**

Decompose `M` into a product of elementary row operations over GF(2):

- Row swap `R_i ↔ R_j` → `SWAP(q[i], q[j])` (3 CX)
- Row addition `R_i ⊕= R_j` → `CX(q[j], q[i])` (1 CX)

Algorithm:

1. Perform Gaussian elimination on `M` to reduce it to identity, recording each
   elementary operation.
2. The circuit is the reverse sequence of the corresponding CX and SWAP gates.

This reference Gaussian-elimination synthesis uses `O(n^2)` elementary row
operations in the worst case, hence `O(n^2)` CX-equivalent two-qubit gates after
counting each SWAP as 3 CX. An implementation MAY instead use the exact
Patel-Markov-Hayes algorithm (2003), which improves the asymptotic CX count to
`O(n^2 / log n)`.

**Isometry — Canonically Completed Isometry**

To remain consistent with Section 2, the SDK's `Isometry` entry is **not** the
raw rectangular `2^n × 2^m` matrix by itself. The core semantic object is a
square `n`-qubit unitary obtained by deterministically completing that isometry.

`Isometry(V, qubits)` takes a `2^n × 2^m` isometry `V` (`m ≤ n`,
`V† * V = I_{2^m}`) and denotes a `2^n × 2^n` unitary `U_V` produced by one
fixed completion rule. That rule must specify exactly which `2^m`-dimensional
computational-basis subspace is the embedded input subspace, and it must fill
the remaining columns by a deterministic orthonormal completion. Implementations
MUST use one stable completion convention everywhere; they must not leave the
completion or column order implicit.

This specification fixes that convention to the span of the first `2^m`
computational basis vectors in the local MSB-first gate-matrix ordering of
Section 2. Equivalently, the first `2^m` columns of `U_V` are exactly the
columns of `V`, in order.

Validation requirements:

- `V` must have dimension `2^n × 2^m` with `m ≤ n`.
- For concrete numeric input, `V` must satisfy `V† * V ≈ I_{2^m}` entrywise
  within epsilon `1e-10`; otherwise validation MUST fail.
- For symbolic input, the isometry condition must already be established
  exactly. If exact isometry status is unresolved, the gate MUST remain opaque
  (or validation MUST fail) until a later stage can prove it exactly.
- Implementations MUST NOT silently orthonormalize, renormalize, or numerically
  "repair" a non-isometric input before applying the deterministic completion
  rule below.
- Because the completion rule below is part of the gate's semantics, its
  keep/discard branch decisions MUST be deterministic under the active scalar
  mode. For symbolic input or any other documented exact-scalar mode, they are
  exact zero/nonzero decisions. For concrete numeric input, they use the same
  final `1e-10` acceptance threshold from Section 2 applied componentwise to the
  residual vector computed in step 3 below: discard iff every residual entry has
  magnitude at most `1e-10`, and otherwise keep. Implementations MAY use tighter
  internal thresholds or more stable numerics, and MAY conservatively reject
  borderline numeric inputs rather than guess, but they MUST NOT use
  undocumented thresholds, platform-dependent branching, or heuristic repair to
  choose a different completion.

The remaining `2^n - 2^m` columns are fixed by the following deterministic
lexicographic Gram-Schmidt rule, which is part of the gate's semantics. For
symbolic or other exact-scalar inputs, the zero/nonzero decision in this rule is
exact. For concrete numeric inputs, it follows the fixed final `1e-10`
componentwise residual test stated above:

1. Let `e_0, e_1, ..., e_{2^n-1}` be the computational-basis column vectors in
   that same local MSB-first ordering.
2. Initialize an ordered list `cols` with the `2^m` columns of `V`, in order.
3. Scan the candidates `e_0, e_1, ..., e_{2^n-1}` in order. For each candidate
   `e_r`, subtract its projections onto every column already present in `cols`.
   If the residual is zero under the active scalar-mode rule above, discard it.
   Otherwise normalize the residual and append it to `cols`.
4. Stop when `cols` contains exactly `2^n` orthonormal vectors. These columns,
   in order, define `U_V`.

Implementations MUST NOT replace this rule by an arbitrary QR completion,
eigendecomposition-dependent basis choice, pivot-dependent column order, or any
other completion convention that could vary across languages or platforms.

On the designated embedded input subspace, `U_V` acts exactly as the intended
isometry `V`. If the frontend also exposes a helper for applying an isometry to
`m` logical qubits with fresh ancillas, that helper is only sugar for preparing
that designated subspace before applying the same square unitary `U_V`; it is
not a different core semantic object.

**Decomposition (exact synthesis of the canonical completion):**

The exact decomposition target is the completed square unitary `U_V` itself, not
merely the action of `V` on its designated input subspace. Because the
lexicographic completion above is part of the gate's semantics, every conforming
synthesis MUST realize that same `U_V` exactly on the full `2^n`-dimensional
space.

1. Apply the lexicographic Gram-Schmidt completion rule above and construct the
   full target unitary `U_V`.
2. Synthesize `U_V` exactly as an `n`-qubit `UnitaryGate(U_V, qubits)` using the
   reference `UnitaryGate` synthesis from this tier, or another exact
   square-unitary synthesis that is proven to preserve the same full matrix
   entrywise under Section 2's equality semantics. Any primitive-level lowering,
   scratch-space requirement, or defer/reject behavior inherited from
   `UnitaryGate` applies here unchanged.
3. An isometry-specific algorithm such as Iten et al. is conforming only if the
   implementation proves or verifies that the emitted circuit equals this exact
   canonical `U_V`; it is not sufficient to match `V` only on the embedded input
   subspace while leaving the orthogonal complement unspecified.
4. Preserve the chosen completion convention exactly in serialization,
   equality-sensitive tests, and transpilation.

If the implementation cannot carry out that completion exactly for symbolic
input, `Isometry(V, qubits)` MUST remain opaque until parameters are bound, and
any stage that needs an explicit `U_V` MUST reject the unresolved instance
rather than choosing an arbitrary numeric completion.

CX count: the guaranteed reference bound is therefore inherited from the chosen
exact `UnitaryGate` synthesis on `n` qubits. For the normative branchwise
reference path above, this is `O(4^n)` with no sharper universal closed form
fixed here. An implementation MAY use a lower-count exact synthesis for this
specific `U_V` only when it preserves the same completed unitary exactly.

For `m = n` (square unitary), this reduces to `UnitaryGate`.

---

### Tier 8: Hamiltonian Simulation and Pauli Evolution

**PauliEvolutionGate — Time Evolution under Pauli Hamiltonian**

`PauliEvolutionGate(hamiltonian, time, method?)` denotes the **exact** unitary
`exp(-i * t * H)` where `H = sum_k c_k * P_k` is a weighted sum of Pauli-string
operators. Every Pauli term uses the same ordered-qubit contract as `PauliGate`
and `PauliProductRotationGate`: the leftmost / element-`0` factor acts on the
first qubit argument, then the second, and so on. The semantic input therefore
includes one exact ordered qubit list / common width `n`; that width is fixed
before duplicate-term collection or exact-zero elimination, so an exactly zero
Hamiltonian still denotes the `n`-qubit identity rather than an arity-free
scalar. Accordingly, `hamiltonian` is not just a bare multiset of coefficient /
Pauli-string pairs: it MUST either intrinsically carry that exact ordered qubit
list / common width `n`, or the frontend/API must attach the same ordered list
or width losslessly as part of this gate family's semantic payload. A
representation that remembers only the surviving nonzero terms after exact
simplification is insufficient for this family. The optional `method?` parameter
is a transpiler synthesis hint descriptor only; it does **not** change the
gate's matrix or its equality semantics.

Portable hint meanings for `method?`:

- omitted / `null`: no approximation family is requested. A transpiler may keep
  the exact semantic gate intact, or may apply only another separately
  documented deterministic exact synthesis that preserves the same matrix
  exactly. Omitted / `null` by itself does **not** authorize product-formula
  approximation.
- `exact`: preserve exact semantics only. Exact commuting-term factorization is
  allowed, but product-formula approximation is not.
- `trotter1(r)`: first-order Lie-Trotter product formula with exact positive
  integer repetition count `r`.
- `trotter2(r)`: second-order symmetric Suzuki product formula `S_2` with exact
  positive integer repetition count `r`, where `S_2` is the
  retained-term-ordered symmetric formula defined below.
- `suzuki(order, r)`: higher even-order Suzuki family `S_order` with exact
  positive even `order >= 4` and exact positive integer repetition count `r`,
  using the exact recursive definition stated in the optional approximate
  synthesis subsection below.

If a frontend/API exposes another concrete representation for the same
synthesis-hint ideas, it MUST map that representation exactly to one of the
portable meanings above or document it explicitly as a non-portable extension.
Because exact Pauli sums are order-insensitive but product formulas are not,
every frontend/API that permits approximate synthesis of this family MUST also
define one deterministic retained-term order after exact duplicate-term
collection. A source-preserving API may preserve source order; a normalized
semantic API MUST choose and document a deterministic canonical order.

Validation requirements:

- The semantic input must determine one exact ordered qubit list / width `n`
  before exact duplicate-term collection and exact-zero elimination. If the
  frontend representation cannot recover that common width when some or all net
  coefficients cancel to zero, the gate MUST remain opaque (or validation MUST
  fail). Implementations MUST NOT infer arity only from the post-collection
  nonzero term list.
- After collecting identical Pauli strings and summing their coefficients
  exactly, every retained Pauli string must have the same width `n` and the same
  ordered tensor-factor interpretation on one common ordered qubit list.
  Implementations MUST reject or keep the gate opaque when terms have
  inconsistent arity or inconsistent ordered-factor semantics, and they MUST NOT
  silently pad shorter strings with identities, drop trailing identities
  heuristically, or reorder tensor factors.
- After collecting identical Pauli strings and summing their coefficients
  exactly, every resulting coefficient must be real: numerically real within
  `epsilon`, or symbolically provable to be real.
- `time` must likewise be real.
- If the inputs are symbolic and the implementation cannot prove these reality
  conditions exactly, the gate MUST remain opaque (or validation MUST fail)
  until a stage that can establish Hermiticity exactly. No stage may silently
  assume Hermiticity from unresolved symbolic data.

**Exact construction:**

1. Collect identical Pauli strings, sum their coefficients exactly, verify the
   reality conditions above, build each Pauli-string matrix as a tensor product
   of Tier 0 Pauli matrices, and obtain the full Hermitian matrix `H`.
2. If all Pauli terms commute, exact factorization is allowed:

   ```
   exp(-i*t*H) = product_k exp(-i * t * c_k * P_k)
   ```

   Because every retained `c_k` and `time` are real, each factor is unitary and
   equals a `PauliProductRotationGate` with `theta = 2 * t * c_k`.

3. If the Pauli terms do not all commute, do **not** replace the gate semantics
   by a product formula. Instead, forward the exact matrix construction to
   `HamiltonianGate(H, time)`.

**Optional approximate synthesis (transpiler only):**

For hardware-limited compilation, a transpiler MAY synthesize this gate with the
documented `trotter1(r)`, `trotter2(r)`, or `suzuki(order, r)` hint families
above. When the formulas below write `product_k` or enumerate `P_1, ..., P_K`,
that order is the exact retained-term order fixed after exact duplicate-term
collection by the frontend/API contract above. No approximation stage may
silently reorder those retained terms unless a separate documented deterministic
reordering pass first produces a new ordered term list and that new order is the
one used by the approximation stage.

First-order Trotter with `r` steps:

```
exp(-i*t*H) ≈ [product_k exp(-i * (t/r) * c_k * P_k)]^r
```

Second-order Trotter:

```
S_2(dt) =
    exp(-i*dt/2 * c_1 * P_1) *
    exp(-i*dt/2 * c_2 * P_2) * ... *
    exp(-i*dt/2 * c_{K-1} * P_{K-1}) *
    exp(-i*dt   * c_K * P_K) *
    exp(-i*dt/2 * c_{K-1} * P_{K-1}) * ... *
    exp(-i*dt/2 * c_2 * P_2) *
    exp(-i*dt/2 * c_1 * P_1)

exp(-i*t*H) ≈ [S_2(t/r)]^r
```

where `dt = t/r`. For higher even order, write `order = 2m` with `m >= 2` and
define

```
p_m = 1 / (4 - 4^(1 / (2m - 1)))

S_{2m}(dt) =
    S_{2m-2}(p_m * dt)^2 *
    S_{2m-2}((1 - 4*p_m) * dt) *
    S_{2m-2}(p_m * dt)^2
```

Then `suzuki(order, r)` means `exp(-i*t*H) ≈ [S_order(t/r)]^r` with that exact
retained-term ordering inherited recursively at every `S_2` leaf. These are
synthesis approximations only; they are not the definition of the gate matrix.

**HamiltonianGate — Time Evolution under General Hermitian Matrix**

`HamiltonianGate(H, time)` implements `exp(-i * t * H)` for a general
`2^n × 2^n` Hermitian matrix `H`.

Validation requirements:

- `H` must be square with dimension `2^n × 2^n` and Hermitian: in the concrete
  numeric case, `H† ≈ H` within `epsilon`; in the symbolic case, Hermiticity
  must be established exactly.
- `time` must be real: numerically real within `epsilon`, or symbolically
  provable to be real.
- If a concrete numeric input fails either validation check, validation MUST
  fail.
- If either obligation is unresolved for symbolic input, the gate MUST remain
  opaque (or validation MUST fail) until a later stage can establish it exactly.
  No stage may silently symmetrize `H`, discard an anti-Hermitian component, or
  assume `time` is real.

**Decomposition (reference spectral factorization: symbolically exact when
available, numerically validated in concrete mode):**

For `n = 0`, `H = [[h]]` with real scalar `h`, so realize the evolution exactly
as

```
HamiltonianGate(H, time) = GlobalPhaseGate(-time * h)
```

For `n ≥ 1`:

1. Diagonalize: `H = V * D * V†` where `D = diag(d_0, ..., d_{2^n-1})`.
2. Then `exp(-i*t*H) = V * diag(exp(-i*t*d_0), ..., exp(-i*t*d_{2^n-1})) * V†`.
3. Implement via the reference composition:
   ```
   UnitaryGate(V†) → DiagonalGate([-t*d_0, ..., -t*d_{2^n-1}]) → UnitaryGate(V)
   ```

`DiagonalGate` receives the phase angles `-t*d_j`, consistent with Phase
Convention 1, not the already-exponentiated complex numbers.

In symbolic mode this composition is exact. In concrete numeric mode it is
conforming when the validated spectral decomposition together with the
downstream `UnitaryGate` / `DiagonalGate` lowering reproduces the same operator
within the prescribed numeric epsilon.

This reference path fixes the exact target operator, not one unique public
spectral presentation. When `H` has repeated eigenvalues, any deterministic
exact basis choice within each degenerate eigenspace and any deterministic
ordering of equal eigenvalues is conforming, provided the recomposed product
`V * D * V†` equals the same input `H` exactly (or within the prescribed numeric
epsilon in concrete mode). APIs that expose eigenpairs or require source-stable
decompositions MUST specify an additional deterministic ordering/basis contract;
otherwise portable conformance tests for `HamiltonianGate` compare only the
final recomposed operator, not the intermediate `V` / `D` presentation.

This exact diagonalization step is required only when `H` is concrete numeric,
or when the implementation can carry out the spectral decomposition exactly in
its symbolic algebra system. If `H` contains unresolved symbolic entries and no
exact symbolic diagonalization is available, the gate MUST remain as
`HamiltonianGate(H, time)` (or another exactly equivalent opaque semantic
object) until parameters are bound. Any stage that needs explicit eigenvalues,
eigenvectors, or rotation angles MUST reject such unresolved instances rather
than approximating them silently.

When the diagonalization itself is available exactly, the resulting
`UnitaryGate(V)` / `UnitaryGate(V†)` subproblems inherit the exact primitive-
level lowering contract of `UnitaryGate` above. A stage that still cannot lower
those subproblems exactly MUST keep the enclosing `HamiltonianGate` semantic
object intact (or reject that stage) rather than substitute an unproved
controlled-rotation synthesis.

If `H` is already represented as a commuting Pauli sum, the exact commuting
factorization from `PauliEvolutionGate` is an equivalent decomposition.
Noncommuting Pauli/Trotter product formulas remain optional approximate
syntheses only.

---

### Tier 9: Quantum Fourier Transform

**QFTGate — Quantum Fourier Transform**

`QFTGate(n)` on ordered qubits `q[0..n-1]` is the SDK's **internal canonical
no-SWAP QFT**. This choice is required for consistency with Section 2: gate
matrices are indexed in local MSB-first argument order, while integer values
carried by the register use the little-endian convention of Section 2. Let
`brev_n(x)` denote the integer obtained by reversing the `n` bits of `x`.

Validation / boundary requirements:

- `n` must be a nonnegative integer-valued arity under the frontend's documented
  exact-expression semantics.
- `n = 0` denotes the exact zero-qubit identity `GlobalPhaseGate(0)`.
- `n = 1` reduces to `H(q[0])`.

For `n >= 0`, the `2^n × 2^n` matrix is:

```
QFT[j, k] = (1 / sqrt(2^n)) * exp(2*pi*i * j * brev_n(k) / 2^n)
```

where `j` and `k` are the row/column indices in the local MSB-first gate-matrix
ordering of Section 2. Under Section 2's little-endian register convention, the
computational-basis **input** value encoded by column `k` is `brev_n(k)`, since
the first qubit argument `q[0]` is both the MSB of the local gate-matrix index
and the LSB of the register value. The no-SWAP circuit below therefore appears
as a bit reversal on the **column** index in this local matrix ordering.
Equivalently, in local MSB-first matrix order this operator is the standard
Fourier matrix with its input basis states permuted by qubit reversal.

**Decomposition (internal canonical form):**

```
QFT(q[0..n-1]):
    for k = n-1 downto 0:
        H(q[k]) →
        for j = k-1 downto 0:
            CP(pi / 2^(k-j), q[j], q[k])
```

This descending-index circuit-time order is the exact decomposition of the
column-bit-reversed matrix above. The superficially similar ascending loop
`for k = 0 to n-1 ... H(q[k])` realizes the row-bit-reversed variant instead and
therefore is **not** this gate under Section 2's conventions.

The controlled-phase gates create the Fourier interference pattern. No trailing
SWAP network is part of the canonical gate. If an external format, textbook
presentation, or application instead wants the alternate local-matrix convention
with no input-column reversal, prepend the explicit qubit-reversal network
`SWAP(q[0], q[n-1]) → SWAP(q[1], q[n-2]) → ...` before `QFTGate`.

Gate count: `n` Hadamards + `n*(n-1)/2` CP gates. CX count: `n*(n-1)`.

The inverse QFT (`QFT†`) reverses this circuit. One exact circuit-time
description is:

```
QFT†(q[0..n-1]):
    for k = 0 to n-1:
        H(q[k]) →
        for j = k+1 to n-1:
            CP(-pi / 2^(j-k), q[k], q[j])
```

Because all `CP` gates are diagonal, commuting such phase gates within one
inverse layer is exact, so equivalent reordered presentations of those negated
phase layers are also valid. If an external adapter prepended an explicit
qubit-reversal network before `QFTGate` to remove the canonical input-column
reversal, the matching inverse must apply that same reversal after `QFT†`.

**Optional approximate synthesis (transpiler only):** For practical circuits, a
transpiler MAY expose an explicit approximation option with integer cutoff
`m_cut >= 1` for the canonical no-SWAP QFT. That cutoff is **not** part of the
semantic signature of exact `QFTGate` itself: absent a separate approximate
family or an explicit synthesis hint that carries `m_cut`, `QFTGate` always
means the exact canonical unitary defined above.

When such an explicit approximation option is used, start from the canonical
decomposition above and drop exactly those `CP(pi / 2^(k-j), q[j], q[k])` terms
with `k - j >= m_cut`; equivalently, drop phase gates whose angle is strictly
less than `pi / 2^(m_cut-1)`. The boundary is strict: a gate whose angle is
exactly `pi / 2^(m_cut-1)` is kept. This reduces the gate count to
`O(n * m_cut)`. If a frontend/API exposes this truncation as a public gate
family rather than a transpiler option, then `m_cut` MUST be an explicit part of
that family's semantic payload. Any explicit external reversal network remains a
separate optional wrapper.

---

### Tier 10: Reversible Classical-Logic Gates

These gates implement classical Boolean functions reversibly. All are unitary
permutation matrices.

**AndGate — Reversible AND**

`AndGate(n)` XORs the AND of `n` input qubits into an output qubit:

```
|x_1⟩...|x_n⟩|y⟩ → |x_1⟩...|x_n⟩|y ⊕ (x_1 ∧ x_2 ∧ ... ∧ x_n)⟩
```

Validation / boundary requirements:

- `n` must be a nonnegative integer-valued arity under the frontend's documented
  exact-expression semantics.
- `n = 0` applies the empty conjunction, so the gate is `X(out)` on the output
  qubit only.
- Clean-output special case: when `y = 0`, the last qubit becomes the Boolean
  AND value.
- `n = 1`: `CX(x_1, out)`.
- `n = 2`: Toffoli gate `CCX(x_1, x_2, out)`.
- `n ≥ 3`: `MCX(x_1, ..., x_n, out)` — multi-controlled X from Tier 5.

**OrGate — Reversible OR**

`OrGate(n)` XORs the OR of `n` input qubits into an output qubit, using De
Morgan's law: `x_1 ∨ ... ∨ x_n = ¬(¬x_1 ∧ ... ∧ ¬x_n)`.

```
|x_1⟩...|x_n⟩|y⟩ → |x_1⟩...|x_n⟩|y ⊕ (x_1 ∨ ... ∨ x_n)⟩
```

Clean-output special case: when `y = 0`, the last qubit becomes the Boolean OR
value.

Validation / boundary requirements:

- `n` must be a nonnegative integer-valued arity under the frontend's documented
  exact-expression semantics.
- `n = 0` applies the empty disjunction, so the gate is identity on `out` and
  the De Morgan template below is skipped.

```
OrGate(inputs[0..n-1], out):
    X(inputs[0]) → ... → X(inputs[n-1]) →    // negate all inputs
    MCX(inputs[0..n-1], out) →                 // AND of negations
    X(out) →                                    // negate result
    X(inputs[0]) → ... → X(inputs[n-1])        // restore inputs
```

For `n >= 1`, gate count: `2n` X gates + 1 MCX + 1 X.

**BitwiseXorGate — Bitwise XOR of Two Registers**

`BitwiseXorGate(n)` computes `|a⟩|b⟩ → |a⟩|a ⊕ b⟩` for two `n`-bit registers.

Validation / boundary requirements:

- `n` must be a nonnegative integer-valued arity under the frontend's documented
  exact-expression semantics.
- `n = 0` denotes the exact zero-qubit identity `GlobalPhaseGate(0)` on the two
  empty registers.

```
BitwiseXorGate(a[0..n-1], b[0..n-1]):
    for i = 0 to n-1:
        CX(a[i], b[i])
```

Uses exactly `n` CX gates, all independent (parallelizable).

**InnerProductGate — Inner Product mod 2**

`InnerProductGate(n)` computes `|a⟩|b⟩|r⟩ → |a⟩|b⟩|r ⊕ (a·b mod 2)⟩` where
`a·b = sum_i a_i * b_i mod 2`.

Validation / boundary requirements:

- `n` must be a nonnegative integer-valued arity under the frontend's documented
  exact-expression semantics.
- `n = 0` denotes `IGate()` on the result qubit alone, because the empty mod-2
  sum is `0`.

```
InnerProductGate(a[0..n-1], b[0..n-1], result):
    for i = 0 to n-1:
        CCX(a[i], b[i], result)
```

Each CCX XORs `a_i ∧ b_i` into `result`. After all `n` iterations, the result
wire has been toggled by `a_0*b_0 ⊕ a_1*b_1 ⊕ ... ⊕ a_{n-1}*b_{n-1}`. ✓

Uses `n` Toffoli gates.

---

### Tier 11: Quantum Arithmetic Circuits

Unless explicitly stated otherwise, every use of `QFT` and `QFT†` in this tier
means the internal canonical no-SWAP `QFTGate` from Tier 9.

**HalfAdderGate — Quantum Half Adder**

Adds two single-bit inputs by XORing the sum and carry into the output wires.

```
|a⟩|b⟩|s⟩|c⟩ → |a⟩|b⟩|s ⊕ a ⊕ b⟩|c ⊕ (a ∧ b)⟩
```

```
HalfAdder(a, b, sum, carry):
    CX(a, sum) → CX(b, sum) → CCX(a, b, carry)
```

Uses 2 CX + 1 CCX.

**FullAdderGate — Quantum Full Adder**

Adds three single-bit inputs `(a, b, carry_in)` by XORing the sum and carry-out
into the output wires.

```
|a⟩|b⟩|c_in⟩|s⟩|c_out⟩ →
    |a⟩|b⟩|c_in⟩|s ⊕ a ⊕ b ⊕ c_in⟩|c_out ⊕ majority(a,b,c_in)⟩
```

```
FullAdder(a, b, c_in, sum, c_out):
    CCX(a, b, c_out) →
    CX(a, b) →
    CCX(b, c_in, c_out) →
    CX(c_in, sum) →
    CX(b, sum) →
    CX(a, b)       // restore b
```

Uses 4 CX + 2 CCX.

**ModularAdderGate — Quantum Adder (QFT-based, Draper 2000)**

`ModularAdderGate(n)` adds two `n`-bit quantum registers:
`|a⟩|b⟩ → |a⟩|(a + b) mod 2^n⟩`.

Validation / boundary requirements:

- `n` must be a nonnegative integer-valued arity under the frontend's documented
  exact-expression semantics.
- `n = 0` denotes the exact zero-qubit identity `GlobalPhaseGate(0)` on the two
  empty addend registers.

**Decomposition (QFT adder):**

```
ModularAdder(a[0..n-1], b[0..n-1]):
    QFT(b[0..n-1]) →
    // Phase rotations: for each bit a[k], apply controlled phase shifts to b
    for j = 0 to n-1:
        for k = 0 to j:
            CP(2*pi / 2^(j-k+1), a[k], b[j]) →
    QFT†(b[0..n-1])
```

The QFT converts `b` to the Fourier basis where addition becomes phase
accumulation. Each `a[k]` contributes a phase `2^k` to the appropriate Fourier
coefficients. The inverse QFT converts back.

Gate count: 2 QFT circuits + `n*(n+1)/2` CP gates.

**Alternative exact implementation path (informative; Ripple-carry, Cuccaro et
al. 2004):** A conforming implementation MAY instead use a linear-depth
ripple-carry adder with a single clean ancilla qubit and `O(n)` Toffoli gates,
for example a MAJ/UMA-style construction rather than a chain of `FullAdderGate`
blocks, provided it preserves the same register map exactly on all basis states.
This is an alternative exact lowering, not a second underspecified semantics for
`ModularAdderGate`.

**MultiplierGate — Quantum Multiplier**

`MultiplierGate(n)` adds the product of two `n`-bit registers into a `2n`-bit
accumulator register:

```
|a⟩|b⟩|p⟩ → |a⟩|b⟩|p + a*b mod 2^{2n}⟩
```

Clean-accumulator special case: when `p = 0`, the last register becomes the
product `a*b`.

Validation / boundary requirements:

- `n` must be a nonnegative integer-valued arity under the frontend's documented
  exact-expression semantics.
- `n = 0` denotes the exact zero-qubit identity `GlobalPhaseGate(0)` on the
  empty multiplicand, multiplier, and accumulator registers.

**Decomposition (exact schoolbook multiplication via controlled Draper
adders):**

For each multiplier bit `b[j]`, add `a << j` into the suffix `product[j..2n-1]`,
conditioned on `b[j] = 1`. Use the exact QFT adder, but promote every
source-controlled phase to a doubly-controlled phase with controls `b[j]` and
the corresponding multiplicand bit `a[k]`:

```
Multiplier(a[0..n-1], b[0..n-1], product[0..2n-1]):
    for j = 0 to n-1:
        QFT(product[j..2n-1]) →
        for k = 0 to n-1:
            for r = j + k to 2n - 1:
                MCPhase(2*pi / 2^(r - (j + k) + 1), [b[j], a[k]], product[r]) →
        QFT†(product[j..2n-1])
```

When `b[j] = 1`, the controlled phases add the classical value represented by
`a << j`; when `b[j] = 0`, the whole layer is identity. Repeating this for all
`j` yields `|a⟩|b⟩|p + a * b mod 2^{2n}⟩` exactly.

Gate count: `O(n^3)` multi-controlled phase gates in the QFT formulation. An
exact ripple-carry alternative controls every gate of a Cuccaro-style adder on
`b[j]`, promoting `CX → CCX` and `CCX → C3X`; that reduces the arithmetic-gate
count to `O(n^2)`.

---

### Tier 12: Function Loading and Approximation

These gates load mathematical functions either into controlled rotations on a
target qubit or, for the amplitude-loading families below, directly into the
target qubit's amplitudes via Y-axis rotations. They all act on an `n`-bit input
register `|x⟩` (encoding integer `x ∈ {0, ..., 2^n - 1}`) and a single-qubit
target. Families that expose an `axis` parameter use configurable axis
(`axis ∈ {X, Y, Z}`; default `Y`). `LinearAmplitudeFunctionGate` and
`ExactReciprocalGate` do **not** expose `axis`; their Y-axis choice is part of
their exact amplitude semantics. Unless explicitly marked approximate, the
semantics are the exact unitary determined by the stated per-basis rotation
angles.

Shared Tier 12 validation conventions:

- For every Tier 12 family parameterized by `numStateBits`, `numStateBits` must
  be a nonnegative integer-valued arity under the frontend's documented
  exact-expression semantics unless that family states the stronger requirement
  `numStateBits >= 1`.
- For every Tier 12 family with an `axis` parameter, `axis` must be exactly one
  of `{X, Y, Z}`. Invalid axis values MUST fail validation. Implementations MUST
  NOT case-fold, alias-map, or heuristically coerce another token into one of
  these three axes.

The family-specific validation blocks below are additional requirements on top
of these shared Tier 12 rules.

Axis shorthand used throughout Tier 12:

- `R_X(theta) = RX(theta)`, `R_Y(theta) = RY(theta)`, and
  `R_Z(theta) = RZ(theta)` exactly.
- `CR_X(theta) = CRX(theta)`, `CR_Y(theta) = CRY(theta)`, and
  `CR_Z(theta) = CRZ(theta)` exactly, i.e. the Tier 2 singly-controlled
  rotations under Section 2's controlled-lifting rule.
- Any later multi-controlled or mixed-controlled `R_{axis}` term produced by
  multilinearization or piece selection denotes that same exact base rotation
  with the required outer controls added under Section 2; it is not a separate
  approximate family.

**LinearPauliRotationsGate — Linear Function Rotation**

`LinearPauliRotationsGate(slope, offset, numStateBits, axis='Y')` applies:

```
R_{axis}(2 * (slope * x + offset))
```

to the target qubit, controlled by the input register.

Validation requirements:

- `slope` and `offset` must be real-valued under the frontend's documented
  exact-expression semantics.
- If the frontend accepts generic numeric or symbolic expressions for these
  parameters, the implementation MUST prove exact reality or keep the gate
  opaque (or fail validation); it MUST NOT infer real-valuedness from
  floating-point approximation or silently discard an imaginary part.

When `numStateBits = 0`, the input register is empty and this gate reduces
exactly to the unconditional rotation `R_{axis}(2 * offset)` on the target.

**Decomposition:**

```
LinearPauliRotations(slope, offset, x[0..n-1], target):
    R_{axis}(2 * offset)(target) →
    for i = 0 to n-1:
        CR_{axis}(2 * slope * 2^i, x[i], target)
```

Each input bit `x[i]` represents `2^i` in the binary encoding of `x`. When
`x[i] = 1`, the controlled rotation adds `slope * 2^i` to the effective angle.

Uses `n` controlled rotations (each 2 CX from Tier 2) + 1 unconditional
rotation. CX count: `2n`.

**PolynomialPauliRotationsGate — Polynomial Function Rotation**

`PolynomialPauliRotationsGate(coeffs, numStateBits, axis='Y')` where
`coeffs = [c_0, c_1, ..., c_d]`. Applies:

```
R_{axis}(2 * (c_0 + c_1*x + c_2*x^2 + ... + c_d*x^d))
```

This is an exact semantic gate family in the sense of Section 3's constructor
contract. Its logical unitary is fixed exactly by the chosen polynomial and
axis, but after multilinearization the resulting terms may require exact
multi-controlled single-target rotations. A constructor or intermediate pass may
therefore return an immutable semantic descriptor until the active stage can
lower those terms by one of the exact routes stated below.

Validation requirements:

- Every coefficient `c_j` must be real-valued under the frontend's documented
  exact-expression semantics.
- If the frontend accepts generic numeric or symbolic expressions for these
  coefficients, the implementation MUST prove exact reality or keep the gate
  opaque (or fail validation); it MUST NOT infer real-valuedness from
  floating-point approximation or silently discard an imaginary part.

When `numStateBits = 0`, the input register is empty, the only basis value is
`x = 0`, and this gate reduces exactly to the unconditional rotation
`R_{axis}(2 * c_0)` on the target.

**Decomposition:**

Let `p(z) = c_0 + c_1*z + ... + c_d*z^d`, and write the register value in the
little-endian convention of Section 2 as `x = sum_{i=0}^{n-1} 2^i * x_i` with
`x_i ∈ {0,1}`. On the Boolean cube, the rotation angle is the unique multilinear
function of the input bits:

```
f(x_0, ..., x_{n-1}) = sum over subsets S of {0,...,n-1}:  a_S * product_{i in S} x_i
```

The exact subset coefficients are fixed by Möbius inversion on the subset
lattice. For each subset `S`, let `value(T) = sum_{i in T} 2^i`. Then:

```
a_S = sum over T ⊆ S of (-1)^{|S|-|T|} * p(value(T))
```

This deterministic formula is the normative multilinearization rule for this
gate. An implementation MAY derive the same `a_S` by another exact symbolic
route, but it MUST obtain coefficients that are exactly equal under Section 2's
equality semantics.

Interpret the empty product for `S = ∅` as `1`. Then:

- If exact arithmetic proves `a_∅ = 0`, the unconditional term may be omitted;
  otherwise apply the unconditional rotation `R_{axis}(2*a_∅)` to the target.
- For each non-empty subset `S`, if exact arithmetic proves `a_S = 0` the
  corresponding controlled term may be omitted; otherwise apply a
  multi-controlled `R_{axis}(2*a_S)` gate controlled on all bits in `S`.

Each resulting controlled term is a single-target exact controlled-rotation
family. Its concrete lowering follows the same exact rule as the Tier 6
`MCMTGate` specialization to one target:

- use a phase-safe family-specific exact recursion only when the implementation
  has separately proved that recursion exact under Section 2's control-phase
  semantics; or
- use the ancilla-assisted exact `MCMTGate` / V-chain route when the required
  clean scratch is available; or
- keep the term as a first-class semantic controlled-rotation object until a
  later stage can lower it exactly.

Implementations MUST NOT guess square roots, silently drop promoted control
phases, or substitute a decomposition proved only up to global phase.

This preserves the constant term `c_0` exactly. Degree-0 polynomials therefore
reduce to a single unconditional rotation.

When exact arithmetic proves `a_S = 0`, that term may be omitted. For a
degree-`d` polynomial, only subset sizes `s = 0..min(d, n)` can be nonzero, so
the number of non-empty controlled terms is bounded by
`sum_{s=1}^{min(d,n)} C(n, s)`, which is `O(n^d)` for fixed `d`.

Non-empty controlled-term count: `O(n^d)` for fixed degree `d`. The primitive CX
cost depends on the chosen exact synthesis of each resulting multi-controlled
rotation, including subset size and ancilla availability, so no sharper uniform
CX bound is asserted here.

**PiecewiseLinearPauliRotationsGate — Piecewise Linear Function Rotation**

`PiecewiseLinearPauliRotationsGate(breakpoints, slopes, offsets, numStateBits, axis='Y')`

This is an exact semantic gate family with an ancilla-assisted reference
lowering for the logical unitary that applies the piece-selected rotation to the
target qubit.

This high-level semantic gate acts only on the data register and target qubit.
The comparator ancillas `cmp[...]` and shared work register `work[...]` named
below are scratch space for one exact lowering, not additional logical operands
of the public gate family. A frontend MAY expose them only through a separate
low-level synthesis helper; otherwise the implementation must allocate them
explicitly or reject/defer this exact lowering while preserving the same
semantic gate.

The exact reference lowering below uses exactly `m - 1` clean comparator
ancillas `cmp[0..m-2]`, one for each nontrivial breakpoint predicate
`[x >= b_k]` for `k = 1..m-1`. These ancillas MUST be pairwise distinct,
disjoint from the input register and target qubit, initialized to `|0...0⟩`, and
uncomputed back to `|0...0⟩` by the end of the template.

In addition, the comparator computation itself requires the exact temporary
scratch space of the chosen comparator synthesis. If the Tier 13
`IntegerComparatorGate` reference construction is used, one shared clean work
register `work[0..numStateBits]` is sufficient: it may be reused sequentially
for all breakpoint computations, but it MUST start in `|0...0⟩`, be disjoint
from the data register, target qubit, and all `cmp` ancillas, and be restored to
`|0...0⟩` after each comparator and again at template exit.

Defined by breakpoints `[b_0, b_1, ..., b_m]` and linear parameters for each
piece:

```
f(x) = slopes[k] * x + offsets[k]    for b_k <= x < b_{k+1}
```

Validation requirements:

- Every breakpoint `b_k` must be integer-valued under the frontend's documented
  exact-expression semantics.
- The breakpoints must satisfy the exact integer chain
  `0 = b_0 < b_1 < ... < b_m = 2^n`, so every basis value `x ∈ {0, ..., 2^n-1}`
  belongs to exactly one piece.
- There must be exactly `m` slopes and `m` offsets, one for each interval.
- Every `slopes[k]` and `offsets[k]` must be real-valued under the frontend's
  documented exact-expression semantics.
- If exact breakpoint integrality/order or exact parameter reality cannot be
  proved, the gate MUST remain opaque (or validation MUST fail). Implementations
  MUST NOT round breakpoints, infer interval membership from approximate numeric
  comparison, or silently discard an imaginary component of a rotation angle.

**Decomposition:**

1. For each nontrivial breakpoint `k = 1..m-1`, compute the predicate
   `s_k = [x >= b_k]` into `cmp[k-1]` by invoking the exact
   `IntegerComparatorGate(b_k, numStateBits, geq=true)` on the data register,
   using `cmp[k-1]` as the clean result qubit and the shared scratch register
   `work` required by that comparator synthesis. Define the classical sentinels
   `s_0 = 1` and `s_m = 0`, and for `1 <= k <= m-1` identify `s_k` with
   `cmp[k-1]`.
2. For each piece `k = 0..m-1`, let `pred_k` denote the exact mixed-control
   predicate `s_k = 1` and `s_{k+1} = 0`. By Phase Convention 4, every
   `s_{k+1} = 0` condition is realized by an exact negative control, so no
   separate selector ancilla is required. Realize the selected linear rotation
   by controlling the Tier 12 linear decomposition term by term, rather than by
   treating `LinearPauliRotationsGate(...)` as one opaque controlled call:
   - apply `R_{axis}(2 * offsets[k])` to the target under `pred_k`;
   - for each state bit `i = 0..numStateBits-1`, apply
     `R_{axis}(2 * slopes[k] * 2^i)` to the target under the combined controls
     `pred_k` and `x[i] = 1`.

   Concretely, the outer piece predicate is:
   - if `k = 0`, a negative control on `cmp[0]` when `m > 1`, or no comparator
     control when `m = 1`;
   - if `0 < k < m - 1`, a positive control on `cmp[k-1]` and a negative control
     on `cmp[k]`;
   - if `k = m - 1`, a positive control on `cmp[m-2]`.

   Each resulting mixed-controlled single-target rotation follows the same exact
   lowering contract as the controlled terms of `PolynomialPauliRotationsGate`:
   use a phase-safe family-specific exact recursion only when separately proved
   exact under Section 2's control-phase semantics; otherwise use the
   ancilla-assisted exact `MCMTGate` / V-chain route when the required clean
   scratch is available; otherwise keep that mixed-controlled rotation as a
   first-class semantic object until a later stage can lower it exactly.
   Implementations MUST NOT approximate the outer predicate, silently drop
   promoted control phases, or treat the negative controls as heuristic syntax
   sugar.
3. Uncompute all comparator ancillas `cmp[0..m-2]` by reversing step 1,
   returning both `cmp` and the shared scratch register `work` to `|0...0⟩`.

Exactly one piece predicate is active for every basis value `x`, so the target
receives exactly one of the piece rotations.

**PiecewisePolynomialPauliRotationsGate — Piecewise Polynomial Rotation**

`PiecewisePolynomialPauliRotationsGate(breakpoints, coeffsList, numStateBits, axis='Y')`

Same structure as PiecewiseLinear but each piece uses a
`PolynomialPauliRotationsGate` instead of a linear one.

This is likewise an exact semantic gate family with an ancilla-assisted
reference lowering for the logical unitary determined by the chosen piecewise
polynomial.

This gate has the same ancilla-ownership contract as
`PiecewiseLinearPauliRotationsGate`: the `cmp[...]` ancillas and shared
`work[...]` register used below are scratch for one exact lowering, not
additional logical operands of the semantic gate itself.

The exact reference lowering uses the same `m - 1` clean comparator ancillas
`cmp[0..m-2]` as PiecewiseLinear, with the same preconditions: they must be
pairwise distinct, disjoint from the data and target registers, initialized to
`|0...0⟩`, and restored to `|0...0⟩` at the end. It likewise uses the same
shared comparator scratch register `work[0..numStateBits]` when instantiated via
the Tier 13 `IntegerComparatorGate` reference synthesis, with the same
clean-input and clean-output requirements.

Validation requirements:

- Every breakpoint `b_k` must be integer-valued under the frontend's documented
  exact-expression semantics.
- The breakpoints must satisfy the exact integer chain
  `0 = b_0 < b_1 < ... < b_m = 2^n`.
- `coeffsList` must contain exactly `m` coefficient lists, one for each
  interval.
- Every coefficient in every list must be real-valued under the frontend's
  documented exact-expression semantics.
- If exact breakpoint integrality/order or exact coefficient reality cannot be
  proved, the gate MUST remain opaque (or validation MUST fail). Implementations
  MUST NOT round breakpoints, infer interval membership from approximate numeric
  comparison, or silently discard an imaginary component of a rotation angle.

**Decomposition:**

1. Reuse the same comparator-bit construction as PiecewiseLinear: for each
   `k = 1..m-1`, compute `s_k = [x >= b_k]` into `cmp[k-1]` with
   `IntegerComparatorGate(b_k, numStateBits, geq=true)`, using the shared
   scratch register `work`; define sentinels `s_0 = 1` and `s_m = 0`.
2. For each piece `k = 0..m-1`, let `pred_k` be the same exact mixed-control
   predicate `s_k = 1` and `s_{k+1} = 0` described for PiecewiseLinear, and
   first multilinearize that piece polynomial by the exact Möbius rule from
   `PolynomialPauliRotationsGate`, obtaining coefficients `a_S^(k)` for subsets
   `S ⊆ {0, ..., numStateBits-1}`. Then realize that selected branch term by
   term under `pred_k`:
   - for `S = ∅`, apply `R_{axis}(2 * a_∅^(k))` to the target under `pred_k`
     unless exact arithmetic proves `a_∅^(k) = 0`;
   - for every non-empty subset `S`, apply `R_{axis}(2 * a_S^(k))` to the target
     under the combined controls `pred_k` and `x[i] = 1` for all `i ∈ S`, unless
     exact arithmetic proves `a_S^(k) = 0`.

   Because `PolynomialPauliRotationsGate` includes the empty-subset term, each
   piece's constant offset is preserved exactly. Each resulting mixed-controlled
   single-target rotation follows the same exact lowering contract as the base
   polynomial family itself: use a separately proved phase-safe family-specific
   recursion, or the ancilla-assisted exact `MCMTGate` / V-chain route, or keep
   that controlled rotation as a first-class semantic object until a later stage
   can lower it exactly. Heuristic or approximate branch-local
   multilinearization is forbidden, and the outer mixed controls must be
   realized by the exact positive/negative control rule from Section 2.
3. Uncompute all comparator ancillas `cmp[0..m-2]` by reversing step 1,
   returning both `cmp` and the shared scratch register `work` to `|0...0⟩`.

**PiecewiseChebyshevGate — Chebyshev Polynomial Approximation**

`PiecewiseChebyshevGate(f, degree, breakpoints, numStateBits, axis='Y')` where
`f` is a one-variable target-function specification, `degree` is the Chebyshev
approximation degree, and `breakpoints` define the piecewise intervals.

This family is intentionally **approximate**. Its semantics are the exact
unitary of the deterministic piecewise-Chebyshev approximation produced at gate
construction time; it is not, in general, the exact unitary of the analytic
function `f`.

For this family, the post-construction semantic payload is the derived ordinary
monomial coefficient list `coeffsList` for each interval, together with the same
`breakpoints`, `numStateBits`, and `axis`. The original `f` input, any frozen
sample data, and any Chebyshev-basis intermediate data are construction-time
provenance only unless a source-preserving / exact-round-trip API explicitly
promises to retain them in addition to that normalized semantic payload.

For avoidance of doubt, the normalized semantic payload of this family is
exactly `(breakpoints, coeffsList, numStateBits, axis)`. A normalized semantic
API MUST either materialize one immutable `coeffsList` at construction time by
steps 1-4 below or reject / keep the gate opaque; it MUST NOT defer resampling,
re-interpolation, or coefficient regeneration to later passes. Portable
conformance in concrete numeric mode may therefore rely only on that stored
payload, or on explicit frozen node data keyed to the exact reference nodes
below, rather than on re-evaluating a frontend-specific analytic `f`.

Validation requirements:

- `degree` must be a nonnegative integer-valued expression under the frontend's
  documented exact-expression semantics.
- Every breakpoint `b_k` must be integer-valued under the same exact-expression
  semantics, and the breakpoints must satisfy the exact integer chain
  `0 = b_0 < b_1 < ... < b_m = 2^n`.
- `f` must be supplied in a frontend/API form that fixes one deterministic
  real-valued input to the preprocessing below. Conforming inputs are:
  - an exact symbolic expression in one variable under the active
    exact-expression semantics; or
  - explicit frozen sample data already keyed to the exact interval/node pairs
    `(k, r)` used in step 2 below, in the same deterministic order. A raw
    host-language callback or other opaque procedure is **not** a conforming
    semantic input to this pure constructor. If a frontend offers a convenience
    helper that samples such a callback, that helper MUST run outside the core
    gate constructor, MUST deterministically materialize either the required
    node values or the final `coeffsList` first, and MUST then call
    `PiecewiseChebyshevGate` only with that frozen data. Serialization,
    equality-sensitive tests, and transpilation MUST depend only on the stored
    deterministic data, not on re-invoking user code.
- On each interval, the classical preprocessing must deterministically produce
  an ordinary monomial coefficient list `coeffsList[k] = [c_0, ..., c_d]`. Any
  Chebyshev-basis coefficients used internally are only intermediate data and
  MUST be converted into this monomial list before the gate is lowered through
  `PiecewisePolynomialPauliRotationsGate`.
- For concrete numeric inputs, the produced `coeffsList[k]` consists of the
  concrete numeric coefficients obtained by exactly steps 1-4 below under the
  API's documented scalar representation, and those stored coefficients
  themselves are the gate's concrete post-construction semantics. They MUST be
  reused directly for equality-sensitive transforms, serialization, and
  transpilation; implementations MUST NOT re-sample `f`, regenerate node values,
  or recompute the interpolation opportunistically after construction.
- Every public API that can expose such a concrete numeric `coeffsList` MUST
  document the scalar representation used for those stored coefficients. If the
  active frontend/IR can represent the produced values in its documented exact
  finite scalar domain, the implementation MUST store them there; otherwise it
  MUST store the resulting concrete numeric literals immutably and treat those
  stored values as the sole normalized semantic data for later passes.
- For symbolic inputs, the same coefficient lists must be derivable exactly and
  their entries must be exactly real-valued; otherwise the gate MUST remain
  opaque (or validation MUST fail).
- If exact breakpoint integrality/order or exact degree integrality cannot be
  established, or if the implementation cannot deterministically produce real
  coefficient lists in the active concrete/symbolic mode above, the gate MUST
  remain opaque (or validation MUST fail).

**Decomposition:**

1. On each interval `[b_k, b_{k+1})`, define the affine normalization
   `x_k(u) = ((b_k + b_{k+1}) / 2) + ((b_{k+1} - b_k) / 2) * u` from
   `u ∈ [-1, 1]` to the interval.
2. Let `d = degree` and use the first-kind Chebyshev nodes
   `u_r = cos((2r + 1) * pi / (2 * (d + 1)))` for `r = 0..d`. Obtain `y_r`
   deterministically as follows: if `f` is an exact symbolic expression, set
   `y_r = f(x_k(u_r))`; if `f` is frozen sample data, `y_r` is the stored value
   for the exact interval/node pair `(k, r)`. These are the only allowed sample
   points for this reference construction; implementations MUST NOT substitute
   an adaptive or implementation-dependent node set.
3. Compute the Chebyshev-basis coefficients

   ```
   a_s = (2 / (d + 1)) * sum_{r=0}^{d} y_r * cos(s * (2r + 1) * pi / (2 * (d + 1)))
   ```

   for `s = 0..d`, and interpret the interval approximant on normalized
   coordinate `u` as

   ```
   p_k(u) = a_0 / 2 + sum_{s=1}^{d} a_s * T_s(u)
   ```

   where `T_s` is the Chebyshev polynomial of the first kind.
4. Convert `p_k(x_k^{-1}(x))` into the ordinary monomial coefficient list
   `coeffsList[k] = [c_0, ..., c_d]` in the interval variable `x`. For concrete
   numeric inputs, the entries may be concrete numeric reals produced by this
   deterministic preprocessing; when the implementation can derive them
   symbolically, they may instead be exact symbolic reals.
5. Implement as a
   `PiecewisePolynomialPauliRotationsGate(breakpoints, coeffsList, numStateBits, axis)`,
   inheriting the exact handling of constant terms from
   `PolynomialPauliRotationsGate`.

This gate family therefore requires `f`, `degree`, `breakpoints`, and the
interval mapping to be concrete enough to evaluate the interpolation nodes and
deterministically compute the coefficient lists at construction time. For
concrete numeric inputs, those concrete coefficient lists themselves become part
of the gate's exact post-construction semantics in concrete mode. In a
source-preserving / exact-round-trip API, retaining the original `f` or frozen
sample data is additional provenance only; it does not replace the requirement
that the derived `coeffsList` be fixed deterministically and remain recoverable.
For symbolic inputs, if that classical preprocessing cannot be carried out
exactly, the gate MUST remain opaque (or validation MUST fail) rather than
substituting approximate or partially evaluated coefficients implicitly.

**LinearAmplitudeFunctionGate — Linear Amplitude Loading**

`LinearAmplitudeFunctionGate(slope, offset, domain, image, numStateBits)`

This family does not expose an `axis` parameter: the Y-axis choice is part of
its semantics because the target `|1⟩` amplitude is specified directly.

For each basis value `x`, let `x_real` be the affine map of the integer `x` from
`{0, ..., 2^n-1}` into the real interval `domain = [d_min, d_max]`:

`x_real = d_min + (d_max - d_min) * x / (2^n - 1)` for `n >= 1`.

Let `y_raw = slope * x_real + offset`, clamp it to the image interval
`image = [i_min, i_max]`, and normalize:

`f(x) = (clamp(y_raw, i_min, i_max) - i_min) / (i_max - i_min)`.

Here `clamp(y, i_min, i_max)` is part of the constructor semantics:

- For symbolic or other exact-scalar inputs, it is the mathematical clamp:
  return `i_min` if `y <= i_min` is proved exactly under the active
  exact-expression semantics, return `i_max` if `y >= i_max` is proved exactly,
  return `y` only if `i_min < y < i_max` is also proved exactly, and otherwise
  preserve the exact symbolic clamp expression (or keep the enclosing gate
  opaque) rather than choose a branch heuristically.
- For concrete numeric inputs, evaluate `y`, `i_min`, and `i_max` numerically
  and then apply a deterministic endpoint snap with Section 2's final
  `epsilon = 1e-10`: return `i_min` if `y <= i_min + epsilon`, return `i_max` if
  `y >= i_max - epsilon`, and otherwise return `y` unchanged.
- If both snapped endpoint tests would hold simultaneously, or if any compared
  quantity is non-finite or not real in the active mode, the constructor MUST
  reject or remain opaque rather than choose arbitrarily.

After computing the normalized value `f(x)`, a concrete numeric implementation
MUST likewise snap `f(x)` to `0` when `|f(x)| <= epsilon` and to `1` when
`|f(x) - 1| <= epsilon`. If the resulting value is still outside `[0, 1]`,
validation MUST fail (or the gate MUST remain opaque) rather than silently
clamping a second time.

Validation requirements:

- `slope`, `offset`, `d_min`, `d_max`, `i_min`, and `i_max` must be real-valued
  under the frontend's documented exact-expression semantics.
- `numStateBits` must be an integer-valued arity under the frontend's documented
  exact-expression semantics and must satisfy `numStateBits >= 1`.
- `i_max > i_min`.
- If the implementation eagerly evaluates any
  `theta_k = 2 * arcsin(sqrt(f(k)))`, it MUST establish that `f(k)` is real and
  lies in `[0, 1]`; it MUST NOT guess branches for `clamp`, `sqrt`, or `arcsin`
  from partially evaluated complex or approximate data.
- If exact real-valuedness of these parameters or the required preprocessing is
  unavailable, the gate MUST remain opaque (or validation MUST fail).

Under those preconditions, `f(x) ∈ [0, 1]`. The gate then applies the
single-qubit rotation `RY(2 * arcsin(sqrt(f(x))))` to the target qubit for each
computational-basis value `x`. In particular, on a clean target:

```
|x⟩|0⟩ → |x⟩(sqrt(1 - f(x))|0⟩ + sqrt(f(x))|1⟩)
```

**Decomposition:**

Use exact value-selective synthesis, mirroring `ExactReciprocalGate`:

In the selector loops below, `bit_i(k)` means bit `i` of the integer `k` in the
Section 2 little-endian register convention, i.e. the bit carried by qubit
`x[i]`.

```
LinearAmplitudeFunction(x[0..n-1], target):
    for k = 0 to 2^n - 1:
        theta_k = 2 * arcsin(sqrt(f(k)))
        for i = 0 to n-1:
            if bit_i(k) == 0:
                X(x[i]) →
        MCMTGate(RY(theta_k), x[0..n-1], [target]) →
        for i = 0 to n-1:
            if bit_i(k) == 0:
                X(x[i])
```

By Phase Convention 4, the surrounding `X` gates realize the exact negative
controls, so basis state `|k⟩` applies exactly the intended rotation and no
other basis state is affected.

This value-selective synthesis is exact for concrete numeric parameters, or for
implementations that can preserve each `theta_k = 2 * arcsin(sqrt(f(k)))`
exactly in their symbolic expression system. Otherwise the gate MUST remain as a
first-class semantic object until all required angles are numerically known, and
any backend needing concrete `RY` angles MUST reject the unresolved instance.

**ExactReciprocalGate — Exact 1/x Rotation**

`ExactReciprocalGate(numStateBits, scalingFactor)` applies a basis-dependent
single-qubit rotation to the target. This family likewise has no `axis`
parameter: the Y-axis choice is fixed by the stated amplitude-loading semantics.
For each `x ∈ {1, ..., 2^n - 1}`, the target sees `RY(2 * arcsin(C / x))`; for
`x = 0`, the target sees identity. In particular, on a clean target:

```
|x⟩|0⟩ → |x⟩(sqrt(1 - C²/x²)|0⟩ + (C/x)|1⟩)
```

for `x ∈ {1, ..., 2^n - 1}` where `C` is the real scaling factor and must
satisfy `|C| ≤ 1`, so `|C/x| ≤ 1` for every allowed nonzero basis value `x`.

Validation requirements:

- `numStateBits` must be an integer-valued arity under the frontend's documented
  exact-expression semantics and must satisfy `numStateBits >= 1`.
- `scalingFactor` must be real-valued under the frontend's documented
  exact-expression semantics.
- The implementation MUST prove the domain condition needed for every
  `arcsin(C / k)` used below: the argument must be real and lie in `[-1, 1]`.
  The simple sufficient condition `|C| ≤ 1` may be proved once up front.
- If the frontend accepts generic numeric or symbolic expressions for `C`, the
  implementation MUST prove exact reality and the required bound, or keep the
  gate opaque (or fail validation); it MUST NOT infer validity from
  floating-point approximation, silently clamp `C / k` into `[-1, 1]`, or guess
  a complex branch of `arcsin`.
- If the implementation eagerly evaluates any angle `2 * arcsin(C / k)`, it MUST
  do so only after establishing that the argument is real and in `[-1, 1]`.

**Decomposition (exact value-selective synthesis):**

For each nonzero computational-basis value `k`, the target rotation angle is
`2 * arcsin(C / k)`. Select that basis value exactly and apply the matching
rotation:

As in `LinearAmplitudeFunctionGate`, `bit_i(k)` means the little-endian bit of
integer `k` carried by qubit `x[i]` under Section 2.

```
ExactReciprocal(x[0..n-1], target):
    for k = 1 to 2^n - 1:
        for i = 0 to n-1:
            if bit_i(k) == 0:
                X(x[i]) →
        MCMTGate(RY(2 * arcsin(C / k)), x[0..n-1], [target]) →
        for i = 0 to n-1:
            if bit_i(k) == 0:
                X(x[i])
```

By Phase Convention 4, the surrounding `X` gates implement the required negative
controls exactly, so only the basis state `|k⟩` activates the corresponding
rotation. The state `|0...0⟩` is left unchanged, which is necessary because
`1/x` is undefined at `x = 0`.

As with `LinearAmplitudeFunctionGate`, this synthesis is exact for concrete
numeric `C`, or for implementations that can preserve each `2 * arcsin(C / k)`
exactly as a symbolic angle expression. Otherwise the gate MUST remain opaque
until all required angles are numerically known, and any backend needing
explicit `RY` angles MUST reject the unresolved instance.

Gate count: `2^n - 1` multi-controlled single-qubit rotations plus the
surrounding X-conjugations. A per-bit CRY approximation is allowed only as an
explicitly named approximate synthesis, not as the definition of
`ExactReciprocalGate`.

---

### Tier 13: Comparison, Aggregation, and Oracles

Unless explicitly stated otherwise, every use of `QFT` and `QFT†` in this tier
means the internal canonical no-SWAP `QFTGate` from Tier 9.

**IntegerComparatorGate — Quantum Integer Comparator**

`IntegerComparatorGate(value, numStateBits, geq=true)` is an exact ancilla-
assisted circuit template. On its logical registers it toggles the result qubit
iff the comparison predicate is true:

```
|x⟩|r⟩ → |x⟩|r ⊕ [x >= value]⟩   (if geq=true)
|x⟩|r⟩ → |x⟩|r ⊕ [x < value]⟩    (if geq=false)
```

The logical signature of this semantic gate is exactly the data register plus
the result qubit. The temporary work register `w[0..n]` used below is scratch
space for one exact lowering, not an additional logical output register. A
frontend MAY expose it only through a lower-level synthesis helper; otherwise
the implementation must provide that clean scratch explicitly or reject/defer
the exact lowering while preserving the same semantic gate.

Let `n = numStateBits`. The logical data register is therefore `x[0..n-1]`, and
the exact reference lowering below uses an `(n+1)`-qubit scratch register
`w[0..n]`.

Validation requirements:

- `numStateBits` must be a nonnegative integer-valued arity under the frontend's
  documented exact-expression semantics.
- `value` must be integer-valued under the frontend's documented
  exact-expression semantics.
- If the frontend accepts generic numeric or symbolic expressions for
  `numStateBits` or `value`, the implementation MUST prove exact integrality
  and, before selecting one of the three construction branches below, exactly
  one of `value <= 0`, `1 <= value <= 2^n - 1`, or `value > 2^n - 1` for that
  same exact `n`. If the implementation cannot prove which branch applies, the
  gate MUST remain opaque (or validation MUST fail). It MUST NOT round, floor,
  ceil, or infer the comparison from floating-point approximation.

**Decomposition (exact comparison via reversible constant addition):**

After discharging those exact proof obligations, handle boundary cases
classically at gate-construction time:

- If `value <= 0`, apply `X(result)` for `geq=true` and identity for
  `geq=false`.
- If `value > 2^n - 1`, apply identity for `geq=true` and `X(result)` for
  `geq=false`.

When `n = 0`, the nontrivial range `1 <= value <= 2^n - 1` is empty, so one of
the two boundary cases above always applies and the scratch-assisted branch
below is unreachable.

For the exactly proved nontrivial range `1 <= value <= 2^n - 1`, use an
`(n+1)`-qubit temporary work register `w[0..n]` initialized and returned to
`|0...0⟩`, and let `c = 2^n - value` (an `(n+1)`-bit classical constant). Use
the exact constant-adder specialization of `ModularAdderGate` on the work
register:

```
IntegerComparator(value, x[0..n-1], result, w[0..n]):
    for i = 0 to n-1:
        CX(x[i], w[i]) →

    // Exact in-place constant addition: w := w + c mod 2^(n+1)
    QFT(w[0..n]) →
    for each set bit j of c:
        for r = j to n:
            P(2*pi / 2^(r-j+1))(w[r]) →
    QFT†(w[0..n]) →

    CX(w[n], result) →
    if geq == false:
        X(result) →

    // Uncompute the constant addition
    QFT(w[0..n]) →
    for each set bit j of c in reverse order:
        for r = n downto j:
            P(-2*pi / 2^(r-j+1))(w[r]) →
    QFT†(w[0..n]) →

    for i = 0 to n-1:
        CX(x[i], w[i])
```

The high bit `w[n]` equals `1` exactly when `x + (2^n - value) >= 2^n`, which is
equivalent to `x >= value`. This construction is fully reversible, leaves the
input register untouched, and does not rely on placeholder borrow logic.

**QuadraticFormGate — Quadratic Form Evaluation**

`QuadraticFormGate(A, b, c, numResultBits)` adds the quadratic form value into a
result register:

```
|x⟩|r⟩ → |x⟩|r + q(x) mod 2^m⟩
```

where `x` is an `n`-bit input, `A` is an `n × n` integer matrix, `b` is an
integer `n`-vector, `c` is an integer scalar, `m = numResultBits`, and
`q(x) = x^T * A * x + b^T * x + c`.

Validation requirements:

- `A` must have dimension `n × n` for some `n`, and `b` must contain exactly `n`
  entries. That common value `n` is the width of the input register `x[0..n-1]`.
- `numResultBits` must be a nonnegative integer-valued arity under the
  frontend's documented exact-expression semantics.
- Every entry of `A`, every component of `b`, and `c` must be integer-valued
  under the frontend's documented exact-expression semantics.
- If the frontend accepts generic numeric or symbolic expressions for these
  coefficients, the implementation MUST prove exact integrality or keep the gate
  opaque (or fail validation); it MUST NOT round, truncate, or infer integrality
  from floating-point approximation.

Boundary cases:

- `numResultBits = 0` denotes identity on the input register alone, because
  addition modulo `1` is trivial.
- If `n = 0`, the input register is empty and only the constant term `c`
  remains, so the gate reduces to exact constant addition of `c mod 2^m` on the
  result register.

For binary inputs, `x_i^2 = x_i`, so implementations MUST expand the quadratic
form using

`x^T * A * x = sum_i A[i][i] * x_i + sum_{i<j} (A[i][j] + A[j][i]) * x_i * x_j`.

The synthesis therefore depends on the symmetrized off-diagonal coefficients
`A[i][j] + A[j][i]`. Implementations MUST NOT silently assume `A` is symmetric
unless that is separately validated as an API precondition.

**Decomposition:**

Use one exact QFT window on the result register and realize every additive term
as the corresponding phase polynomial in that Fourier basis. Let
`ell_i = b[i] + A[i][i]` and `q_ij = A[i][j] + A[j][i]`. Then:

```
QuadraticForm(A, b, c, x[0..n-1], result[0..m-1]):
    QFT(result[0..m-1]) →

    for r = 0 to m-1:
        P(2*pi * c / 2^(r+1))(result[r]) →

    for i = 0 to n-1:
        if exact arithmetic proves ell_i = 0:
            skip this block
        else:
            for r = 0 to m-1:
                CP(2*pi * ell_i / 2^(r+1), x[i], result[r]) →

    for i = 0 to n-1:
        for j = i+1 to n-1:
            if exact arithmetic proves q_ij = 0:
                skip this block
            else:
                for r = 0 to m-1:
                    MCPhase(2*pi * q_ij / 2^(r+1), [x[i], x[j]], result[r]) →

    QFT†(result[0..m-1])
```

This adds the constant term `c`, each linear coefficient `ell_i`, and each
quadratic coefficient `q_ij` into the result register modulo `2^m`. Signed
integer coefficients are handled directly by signed phase angles; modulo `2^m`
behavior follows from phase periodicity. All phase operations inside the QFT
window commute, so any reordering within that window is exactly equivalent. Any
linear or quadratic block may be omitted only when exact arithmetic proves the
corresponding coefficient is zero; otherwise the block MUST be emitted with that
exact coefficient (possibly symbolically) rather than removed by numeric or
heuristic simplification.

If the implementation exposes a symmetric-matrix fast path and validates
`A = A^T`, then the off-diagonal coefficient simplifies to `q_ij = 2 * A[i][j]`.

**WeightedSumGate — Weighted Sum**

`WeightedSumGate(weights, numStateBits, numSumBits)` adds a weighted sum into a
sum register:

```
|x_0⟩...|x_{n-1}⟩|s⟩ → |x_0⟩...|x_{n-1}⟩|s + sum_i w_i * x_i mod 2^m⟩
```

where `w_i` are integer weights and `m = numSumBits`. Clean-sum special case:
when `s = 0`, the last register becomes the weighted sum modulo `2^m`.

Validation requirements:

- `numStateBits` and `numSumBits` must be nonnegative integer-valued arities
  under the frontend's documented exact-expression semantics.
- `weights` must contain exactly `numStateBits` entries.
- Every `w_i` must be integer-valued under the frontend's documented
  exact-expression semantics.
- If the frontend accepts generic numeric or symbolic expressions for weights,
  the implementation MUST prove exact integrality or keep the gate opaque (or
  fail validation); it MUST NOT round, truncate, or infer integrality from
  floating-point approximation.

Boundary cases:

- `numStateBits = 0` with `weights = []` denotes identity on the sum register,
  because the empty weighted sum is `0`.
- `numSumBits = 0` denotes identity on the input register alone, because
  addition modulo `1` is trivial.

**Decomposition:**

For each input bit `x_i` whose weight is not proved exactly zero:

- Add `w_i` to the sum register, controlled on `x_i`.
- The normative exact reference decomposition below uses the QFT-based
  controlled-add route.
- An implementation MAY instead use an exact ripple-carry controlled-adder only
  if it preserves the same register map exactly on all basis states; that is an
  alternative exact implementation path, not a second underspecified reference
  decomposition.

```
WeightedSum(weights, x[0..n-1], sum[0..m-1]):
    QFT(sum) →
    for i = 0 to n-1:
        if exact arithmetic proves weights[i] = 0:
            skip this block
        else:
            // Controlled phase-add weights[i]
            for j = 0 to m-1:
                CP(2*pi * weights[i] / 2^(j+1), x[i], sum[j])
    → QFT†(sum)
```

Signed integer weights are handled directly by signed phase angles; modulo `2^m`
behavior follows from phase periodicity. A controlled-add block may be omitted
only when exact arithmetic proves the corresponding weight is zero; otherwise it
MUST be emitted with that exact signed integer coefficient. Any exact
ripple-carry alternative must realize that same map exactly, not merely up to a
discarded phase or with a different ancilla contract.

**PhaseOracleGate — Phase Oracle from Boolean Function**

`PhaseOracleGate(expression, variables?)` applies `|x⟩ → (-1)^{f(x)} |x⟩` where
`expression` is an **exact Boolean-function representation** denoting a map
`f: {0,1}^n → {0,1}` on the input register, and `variables?` is an optional
explicit ordered variable list used when that order is not already intrinsic to
`expression`.

Variable-order contract:

- The semantic input of this gate family is the pair
  `(expression, variableOrder)`, where `variableOrder` is fixed either
  intrinsically by `expression` or explicitly by `variables?`.
- The ordered input qubit list is `q[0..n-1]`. Variable/bit index `i` denotes
  qubit `q[i]` exactly.
- Therefore variable index `0` is the least-significant input bit in the Section
  2 little-endian register-value convention, while still being the first qubit
  argument in the local MSB-first gate-matrix ordering.
- If the frontend representation is positional (truth table, ANF coefficient
  list, indexed ESOP, unnamed AST leaves, etc.), that positional order is the
  variable order `0..n-1`. If `variables?` is also exposed for such a form, it
  MUST match that same positional order exactly or validation MUST fail.
- If the frontend representation is named (named AST variables, symbolic
  literals, dictionary-like forms, etc.), the frontend contract MUST either
  embed one deterministic ordered variable list `v[0..n-1]` inside `expression`,
  or require `variables?` to supply it explicitly, and variable `v[i]` denotes
  qubit `q[i]`.
- The exact ordered variable list is part of the gate's semantic payload.
  Source-preserving / exact-round-trip APIs MUST retain it explicitly;
  normalized semantic APIs MUST keep the same order losslessly recoverable.
- Implementations MUST reject or keep the gate opaque when that exact variable
  order is absent or ambiguous; they MUST NOT infer it from map iteration order,
  host-language identifier ordering, or heuristic name sorting.

Validation requirements:

- The frontend representation must denote a finite Boolean function exactly.
  Acceptable exact forms include a Boolean AST/expression tree, truth table,
  ESOP, ANF, or another representation from which the implementation can derive
  an exactly equivalent finite Boolean function deterministically.
- A host-language callback, partially evaluated predicate, or any other opaque
  runtime procedure is insufficient unless the frontend contract also provides a
  deterministic exact finite representation from which the same Boolean function
  can be recovered without approximation.
- The implementation MUST either derive an exact ESOP (or another exactly
  equivalent oracle synthesis) from that representation, or keep the gate opaque
  (or fail validation). It MUST NOT rely on heuristic Boolean simplification,
  approximate minimization, or backend-specific oracle magic.

**Reference exact decomposition via ESOP:**

1. Convert `expression` exactly to an Exclusive Sum of Products (ESOP), or use
   it directly if it is already stored in that form:
   `f(x) = t_1 ⊕ t_2 ⊕ ... ⊕ t_K` where each `t_k` is a product (AND) of
   literals. Every literal refers to a specific variable index under the
   variable-order contract above.

   Before lowering any term, normalize each product exactly: order its literals
   deterministically by variable index, delete duplicate identical literals
   within the same term, and drop any term containing both `x_i` and `¬x_i`,
   since such a product is identically `0`. This normalization is part of the
   exact Boolean semantics used by the lowering and ensures the later
   `MCZ`/`MCX` calls satisfy Section 3's pairwise-distinct-qubit rule.

2. For each remaining normalized ESOP term `t_k = l_1 ∧ l_2 ∧ ... ∧ l_m` (where
   each `l_j` is `x_{r_j}` or `¬x_{r_j}` for some variable index `r_j`):
   - If `m = 0` (constant-1 term), apply the zero-qubit phase
     `GlobalPhaseGate(pi)` exactly once for that term.
   - If `m >= 1`, apply X to each negated literal, then:
     - If `m = 1`, apply `Z` to the single literal qubit `q[r_1]`.
     - If `m >= 2`, apply multi-controlled Z on the relevant qubits. Here
       `MCZ(q[r_1], ..., q[r_m])` is only local shorthand for the exact
       decomposition
       `H(q[r_m]) → MCX(q[r_1], ..., q[r_{m-1}], q[r_m]) → H(q[r_m])`; it is not
       a separate public gate family beyond this derived use.
     - Undo X on negated literals.

3. Since each ESOP term contributes `(-1)^{t_k}` and XOR is addition mod 2, the
   phases combine correctly: `(-1)^{t_1 ⊕ ... ⊕ t_K} = (-1)^{f(x)}`. ✓

**BitFlipOracleGate — Bit-Flip Oracle from Boolean Function**

`BitFlipOracleGate(expression, variables?)` applies `|x⟩|y⟩ → |x⟩|y ⊕ f(x)⟩`,
using the same exact Boolean-expression contract and exact/opaque fallback rule
as `PhaseOracleGate`. Its semantic input is the same pair
`(expression, variableOrder)`, including the same exact variable-order contract
that binds variable index `i` to input qubit `q[i]`.

**Reference exact decomposition via ESOP:**

Use the same exact normalized ESOP analysis as `PhaseOracleGate`, but use MCX
(multi-controlled X on the output qubit `y`) instead of MCZ for each term:

1. For each remaining normalized ESOP term `t_k`:
   - If the term is constant 1 (empty product), apply `X(y)`.
   - Otherwise, apply X to negated literals, then:
     - If the term has one literal on variable index `r_1`, apply
       `CX(q[r_1], y)`.
     - If the term has two or more literals, apply `MCX(controls, y)` where
       `controls = [q[r_1], ..., q[r_m]]` are the qubits selected by those
       literals under the same variable-order contract.
     - Undo X on negated literals.

**Relationship:** If an ancilla qubit `y` is initialized to `|1⟩`, then
`H(y) → BitFlipOracle → H(y)` produces the phase oracle on the data register by
phase kickback. Equivalently, if `y` is already prepared in `|−⟩`, apply only
`BitFlipOracle` and omit both Hadamards.

---

### Tier 14: State Preparation

**GraphStateGate — Graph State Preparation**

`GraphStateGate(adjacencyMatrix)` on `n` qubits. Prepares a graph state from a
graph `G = (V, E)` defined by the adjacency matrix.

A graph state is `|G⟩ = prod_{(i,j) ∈ E} CZ(i, j) * |+⟩^{⊗n}`.

Validation requirements:

- `adjacencyMatrix` must be an `n × n` binary matrix.
- `n = 0` denotes the exact zero-qubit identity `GlobalPhaseGate(0)`.
- The diagonal must be all zeros (no self-loops).
- The matrix must be symmetric.
- If the frontend accepts symbolic or generic numeric entries, the
  implementation MUST prove exact bit-valuedness, exact zero diagonal, and exact
  symmetry, or keep the gate opaque (or fail validation). It MUST NOT round
  entries to `{0,1}`, silently symmetrize the matrix, or drop self-loops.
- Invalid inputs MUST raise a validation error; implementations MUST NOT
  silently symmetrize the matrix, drop self-loops, or coerce non-binary values.

```
GraphState(adjacencyMatrix, q[0..n-1]):
    for i = 0 to n-1:
        H(q[i])
    for each edge (i, j) where adjacencyMatrix[i][j] = 1 and i < j:
        CZ(q[i], q[j])
```

Uses `n` Hadamards + `|E|` CZ gates (each 1 CX from Tier 2).

---

### Reference: Expected Matrices

The compositional definitions above produce the following matrices. These are
**derived results** (computed by composing decompositions), not independent
definitions.

Unless a line below explicitly says otherwise, every basis-state label,
row/column index, and integer row/column number in this appendix uses the same
local **MSB-first** gate-matrix ordering from Section 2 for the ordered operand
list of that gate. Equivalently, in a two-qubit entry `|10⟩` means "first gate
argument = 1, second gate argument = 0", and integer indices such as `5`, `6`,
or `14` are interpreted in that same local gate-basis ordering even though
state-vector storage elsewhere in the SDK is little-endian.

**CX:** `[[1,0,0,0],[0,1,0,0],[0,0,0,1],[0,0,1,0]]`

**CZ:** `diag(1, 1, 1, -1)`

**CY:** `[[1,0,0,0],[0,1,0,0],[0,0,0,-i],[0,0,i,0]]`

**CP(l):** `diag(1, 1, 1, exp(i*l))`

**CRX(th):** Identity in `|00⟩,|01⟩`; RX(th) in `|10⟩,|11⟩` subspace.

**CRY(th):** Identity in `|00⟩,|01⟩`; RY(th) in `|10⟩,|11⟩` subspace.

**CRZ(th):** Identity in `|00⟩,|01⟩`; RZ(th) in `|10⟩,|11⟩` subspace.

**CS:** `diag(1, 1, 1, i)`

**CSdg:** `diag(1, 1, 1, -i)`

**CSX:** Identity in `|00⟩,|01⟩`; SX in `|10⟩,|11⟩` subspace.

**CH:** Identity in `|00⟩,|01⟩`; H in `|10⟩,|11⟩` subspace.

**CU(th,ph,l,gamma):** Identity in `|00⟩,|01⟩`; `exp(i*gamma)*U_can(th,ph,l)` in
`|10⟩,|11⟩` subspace.

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

**XXPlusYY(th,beta):** `[[1,0,0,0],[0,cos(th/2),-i*sin(th/2)*exp(-i*beta),0],`
`[0,-i*sin(th/2)*exp(i*beta),cos(th/2),0],[0,0,0,1]]`

**XXMinusYY(th,beta):** `[[cos(th/2),0,0,-i*sin(th/2)*exp(-i*beta)],`
`[0,1,0,0],[0,0,1,0],[-i*sin(th/2)*exp(i*beta),0,0,cos(th/2)]]`

**CCX (8×8):** Identity except `[6][7]=1, [7][6]=1, [6][6]=0, [7][7]=0`.

**CCZ (8×8):** Identity except `[7][7]=-1`.

**CSWAP (8×8):** Identity except rows 5,6 swapped.

**RCCX (8×8):** Identity except `[5][5]=-1`, `[6][6]=0, [6][7]=-i`,
`[7][7]=0, [7][6]=i`.

**C3X (16×16):** Identity except rows/cols 14,15 where X is applied.

**C3SX (16×16):** Identity except rows/cols 14,15 where SX is applied.

**RC3X (16×16):** Identity except `[12][12]=i`, `[13][13]=-i`,
`[14][14]=0, [14][15]=1`, `[15][15]=0, [15][14]=-1`.

**MCPhase(l, N controls) ((N+1)-qubit):** `diag(1, ..., 1, exp(i*l))`.

**QFT (n qubits):**
`QFT[j,k] = (1/sqrt(2^n)) * exp(2*pi*i * j * brev_n(k) / 2^n)` for the SDK's
internal canonical no-SWAP `QFTGate`, where `brev_n(k)` bit-reverses the `n`
bits of column index `k`.

---

### Abstraction Dependency Graph

```
Tier 14: GraphStateGate ───────→ H + CZ

Tier 13: PhaseOracleGate ─────→ GlobalPhaseGate + Z + MCX + X + H
         BitFlipOracleGate ────→ MCX + X
         IntegerComparatorGate → QFT + P + CX
         QuadraticFormGate ────→ QFT + P + CP + MCPhase
         WeightedSumGate ──────→ QFT + CP

Tier 12: LinearPauliRotationsGate ────────→ CR_{axis} + R_{axis}
         PolynomialPauliRotationsGate ────→ multilinearized multi-controlled R_{axis}
         PiecewiseLinearPauliRotationsGate → IntegerComparator + mixed-controlled linear rotation terms
         PiecewisePolynomialPauliRotationsGate → IntegerComparator + multilinearized mixed-controlled rotation terms
         PiecewiseChebyshevGate ──────────→ PiecewisePolynomialPauliRotations
         LinearAmplitudeFunctionGate ─────→ MCMT(RY) + X-conjugated selectors
         ExactReciprocalGate ─────────────→ MCMT(RY) + X-conjugated selectors

Tier 11: HalfAdderGate ───────→ CX + CCX
         FullAdderGate ────────→ CX + CCX
         ModularAdderGate ─────→ QFT + CP + QFT†
         MultiplierGate ───────→ controlled Draper adders (QFT + MCPhase)

Tier 10: AndGate ──────────────→ MCX (= CCX for n=2)
         OrGate ───────────────→ X + MCX + X
         BitwiseXorGate ───────→ CX × n
         InnerProductGate ─────→ CCX × n

Tier 9:  QFTGate ──────────────→ H + CP

Tier 8:  PauliEvolutionGate ───→ commuting PauliProductRotationGate
                                 or exact HamiltonianGate construction
         HamiltonianGate ──────→ UnitaryGate + DiagonalGate (exact diagonalization)

Tier 7:  UCRZGate ─────────────→ Gray-code multiplexor CX + RZ
         UCRYGate ─────────────→ Gray-code multiplexor CX + RY
         UCRXGate ─────────────→ H + UCRZ + H
         UCPauliRotGate ───────→ UCRX | UCRY | UCRZ
         UCGate ───────────────→ UCRZ + UCRY + UCRZ + DiagonalGate
         UnitaryGate ──────────→ recursive CSD → UCR → CX + single-qubit
         LinearFunction ───────→ Gaussian elimination → CX + SWAP
         Isometry ─────────────→ canonical unitary completion → UnitaryGate

Tier 6:  MSGate ───────────────→ RXX (pairwise)
         PauliGate ────────────→ Tier 0 (tensor products, no CX)
         DiagonalGate ─────────→ recursive DiagonalGate + UCRZ
         PermutationGate ──────→ cycle decomposition + MCX endpoint transpositions
         MCMTGate ─────────────→ repeated C^k(G) or V-chain + C(G)
         PauliProductRotationGate → basis change + CX ladder + RZ
                                     or zero-qubit global phase for all-I strings

Tier 5:  C3XGate ──────────────→ H + MCPhase(pi)
         C3SXGate ─────────────→ exact ctrl^3(SX)
                                 → V-chain(MCMT) or exact UnitaryGate synthesis
         C4XGate ──────────────→ H + MCPhase(pi)
         RC3XGate ─────────────→ CX + T/Tdg/H (direct, 6 CX)
         MCXGate ──────────────→ H + MCPhase(pi)
         MCPhaseGate ──────────→ recursive CP + smaller MCX

Tier 4:  CCXGate ──────────────→ CSX + CX (V-decomposition)
         CCZGate ──────────────→ CCX + H
         CSwapGate ────────────→ CCX + CX
         RCCXGate ─────────────→ CX + T/Tdg/H (direct, 3 CX)

Tier 3:  SwapGate ─────────────→ CX × 3
         RZZGate ──────────────→ CX + RZ
         RXXGate ──────────────→ RZZ + H (basis change)
         RYYGate ──────────────→ RZZ + RX (basis change)
         RZXGate ──────────────→ RZZ + H (partial basis change)
         ECRGate ──────────────→ RZX + X
         iSwapGate ────────────→ SWAP + CZ + S
         XXPlusYYGate ─────────→ RZ + RXX + RYY
         XXMinusYYGate ────────→ RZ + RXX + RYY

Tier 2:  CZGate ───────────────→ CX + H
         CYGate ───────────────→ CX + S + Sdg
         CPhaseGate ───────────→ CX × 2 + P
         CRZGate ──────────────→ CX × 2 + RZ
         CRYGate ──────────────→ CX × 2 + RY
         CRXGate ──────────────→ CRZ + H (basis change)
         CSGate ───────────────→ CP(π/2)
         CSdgGate ─────────────→ CP(-π/2)
         CSXGate ──────────────→ CRX(π/2) + P
         CHGate ───────────────→ ABC decomposition (2 CX)
         CUGate ───────────────→ ABC decomposition (2 CX)
         DCXGate ──────────────→ CX × 2

Tier 1:  CXGate (primitive) ←── the only hardcoded multi-qubit matrix

Tier 0:  IGate, HGate, XGate, YGate, ZGate, PhaseGate, RGate,
         RXGate, RYGate, RZGate, SGate, SdgGate, SXGate, SXdgGate,
         TGate, TdgGate, UGate, RVGate, GlobalPhaseGate
         ←── hardcoded 2×2 / 1×1 matrices
```

## 4. OpenQASM 3.1 Reference

For the complete OpenQASM 3.1 specification, see
https://openqasm.com/versions/3.1/index.html. Use this section as a checklist to
determine if you have implemented the complete specification.

================================================================================

1. # FILE-LEVEL PROGRAM STRUCTURE

program version? statementOrScope* EOF

Meaning: A source file may begin with an optional version declaration, followed
by zero or more statements and nested scopes.

---

## 1.1 Version declaration

OPENQASM M.m;

Meaning: Declares the intended OpenQASM major/minor version for the file. It may
appear only once, and only as the first non-comment line. The minor version is
optional.

Examples: OPENQASM 3; OPENQASM 3.1;

---

## 1.2 Include

include "filename";

Meaning: Includes another source file as if its contents were inserted at that
point. Valid only at global scope.

Typical example: include "stdgates.inc";

For this SDK, serialized OpenQASM 3 output MUST always include
`include "stdgates.inc";`. Emit it immediately after the version declaration
when a version is present, or as the first statement otherwise.

---

## 1.3 Defcal grammar selection

defcalgrammar "name";

Meaning: Selects the pulse/calibration grammar used inside cal/defcal blocks.
Example: defcalgrammar "openpulse";

Selecting a grammar name alone does **not** imply that calibration scopes adopt
the ordinary-scope scalar-`globalPhase` semantics from Phase Convention 3. For
that opt-in, the loaded grammar MUST carry an explicit implementation-visible
contract or metadata record stating whether it adopts those rules and, if it
does, the corresponding scope-entry arity and fold/hoist contract.

# ================================================================================ 2. COMMENTS

Single-line comment // text

Block comment /* text */

Meaning: Comments are ignored by the parser.

# ================================================================================ 3. IDENTIFIERS, PHYSICAL QUBITS, AND BASIC LEXICAL FORMS

---

## 3.1 Identifiers

identifier starts with: letter, underscore, or permitted Unicode letter class
continues with: identifier-start characters plus digits

Meaning: Names for variables, gates, subroutines, frames, ports, and so on.

Examples: theta _my_gate Δphase

---

## 3.2 Physical qubit identifiers

$0 $1 $27

Meaning: Refers to a concrete physical qubit on the target device.

---

## 3.3 Reserved keywords and built-in language words

OPENQASM include defcalgrammar def cal defcal gate extern box let break continue
if else end return for while in switch case default input output const readonly
mutable qreg qubit creg bool bit int uint float angle complex array void
duration stretch gphase inv pow ctrl negctrl #dim durationof delay reset measure
barrier true false

Meaning: These are the language-level reserved words/tokens defined by the
official grammar.

---

## 3.4 Punctuation and separators

[ ] { } ( ) : ; . , = -> @

Meaning: Used for indexing, blocks, grouping, range syntax, statement
termination, member-like lexical roles, argument separation, assignment, return
signatures, and gate modifiers.

---

## 3.5 Operators

Arithmetic / numeric:

```
+  -  *  /  %  **
```

Bitwise: ~ & ^ | << >>

Logical: ! && ||

Comparison: < <= > >= == !=

Concatenation: ++

Compound assignment: += -= *= /= &= |= ~= ^= <<= >>= %= **=

Meaning: These operators are used in expressions, assignments, concatenation,
and slicing contexts.

---

## 3.6 Operator precedence (highest to lowest)

1. () [] (type)(x)
2. **
3. ! - ~
4.

```
*  /  %
```

5.

```
+  -
```

6. << >>
7. < <= > >=
8. != ==
9. &
10. ^
11. |
12. &&
13. ||

Meaning: Defines how expressions are parsed without extra parentheses.

# ================================================================================ 4. LITERALS

Integer literals 123 0b1010 0o17 0xFF 1_000_000

Meaning: Compile-time integer constants in decimal, binary, octal, or
hexadecimal form.

Floating-point literals 1.0 .5 5. 1e3 2.5e-4

Meaning: Compile-time floating-point constants.

Imaginary literals 2im 3.5im

Meaning: Imaginary-valued constants used to build complex values.

Boolean literals true false

Meaning: Compile-time Boolean constants.

Bitstring literals "1010" "0001_1110"

Meaning: Compile-time bit array literals.

Timing literals 10ns 2us 100dt 1.5ms

Meaning: Compile-time duration literals with SI or backend-dependent units.

# ================================================================================ 5. TYPES

---

## 5.1 Quantum types

Single qubit qubit q;

Quantum register qubit[5] q;

Backward-compatible legacy form qreg q[5];

Meaning: Represents virtual qubits or fixed-size virtual qubit registers.

---

## 5.2 Classical scalar types

Bit / bit register bit b; bit[8] c;

Meaning: Single classical bit or fixed-size classical bit register.

Signed integer int i; int[32] i32;

Meaning: Signed integer value, width optional unless a fixed width is needed.

Unsigned integer uint u; uint[16] u16;

Meaning: Unsigned integer value, width optional unless a fixed width is needed.

Floating point float f; float[64] f64;

Meaning: IEEE-754 floating-point value, width optional if target-defined
precision is acceptable.

Angle angle a; angle[20] theta;

Meaning: Special angular type designed for efficient modular angle arithmetic.

Boolean bool flag;

Meaning: Boolean truth value.

Complex complex z; complex[float[32]] z32;

Meaning: Complex number whose components are floating-point values.

Duration duration t;

Meaning: Compile-time time interval.

Stretch stretch s;

Meaning: A compile-time-resolved non-negative flexible duration that can expand
to satisfy timing constraints.

Void void

Meaning: Reserved unrealizable type used only as an implicit return category for
routines with no result. It is not instantiable.

---

## 5.3 Arrays

array[base_type, dim1, dim2, ...] name; array[int[8], 4] a; array[float[64], 3,
2] b;

Meaning: Statically sized classical arrays. Arrays are global-scope declarations
in the base language, can be multidimensional, and are indexed with [].

Notes:

- Arrays are not dynamically resized.
- Negative indexing is allowed.
- Up to 7 dimensions are allowed.
- stretch is not a valid array base type in the language description.

---

## 5.4 Array-reference parameter types

readonly array[int[32], 4] a mutable array[float[64], #dim = 2] m

Meaning: Parameter/reference forms for subroutines/externs. readonly means read
access, mutable means write access. #dim may be used to constrain dimensionality
in the grammar.

---

## 5.5 Const-qualified scalar declarations

const int[32] n = 5; const angle[20] theta = pi / 2;

Meaning: Compile-time constant scalar declarations. Required in many places such
as type widths and some compile-time indexing/sizing contexts.

# ================================================================================ 6. CASTING

(type)(expression) type(expression)

Examples: int[8](3.7) angle[32](pi / 2) uint(bits) float(a)

Meaning: Explicit classical casts. The grammar models casts as type(expression).
The spec also discusses the conversion rules in detail.

# ================================================================================ 7. EXPRESSIONS

---

## 7.1 Core expression forms

Parenthesized expression (expr)

Indexing / subscripting expr[index] expr[i, j] expr[a:b] expr[a:c:b]

Power expr ** expr

Unary -expr !expr ~expr

Binary arithmetic expr * expr expr / expr expr % expr expr + expr expr - expr

Bit shifts expr << expr expr >> expr

Comparisons expr < expr expr <= expr expr > expr expr >= expr

Equality expr == expr expr != expr

Bitwise logic expr & expr expr ^ expr expr | expr

Boolean logic expr && expr expr || expr

Function/subroutine/extern call name(arg1, arg2, ...)

Duration-of block durationof({ ... })

Meaning: These are the standard expression forms used throughout the language.

---

## 7.2 Special expression-like forms used in specific contexts

Alias expression expr ++ expr ++ expr

Meaning: Used to build register aliases and concatenations in aliasing contexts.

Measure expression measure qubit_or_register

Meaning: A measurement expression that can appear in assignments, returns, and
the arrow measurement form.

Range expression a:b a:c:b :b a: :

Meaning: Used for slicing/index sets and loop ranges.

Set expression {1, 2, 4} {i, j, k}

Meaning: Used in indexing and for-loop iteration over discrete values.

Array literal {1, 2, 3} {{1, 2}, {3, 4}}

Meaning: Literal construction of arrays or set-like iteration values depending
on context.

# ================================================================================ 8. DECLARATIONS

---

## 8.1 Classical declaration

scalarType name; scalarType name = declarationExpression; arrayType name;
arrayType name = declarationExpression;

Examples: int[32] a; float[64] b = 5.0; bit[4] out = "1010"; array[int[8], 3]
arr = {1, 2, 3};

Meaning: Declares classical storage.

---

## 8.2 Quantum declaration

qubit name; qubit[size] name;

Meaning: Declares one qubit or a fixed-size register of qubits.

---

## 8.3 Legacy OpenQASM 2-style declarations

qreg q[5]; creg c[5];

Meaning: Backward-compatible declaration syntax retained by the grammar.

---

## 8.4 Alias declaration

let alias_name = aliasExpression;

Examples: let pair = q[0:1]; let both = q1 ++ q2;

Meaning: Creates an alias/reference view to qubits/registers while in scope.

---

## 8.5 Input/output declarations

input scalarType name; input arrayType name; output scalarType name; output
arrayType name;

Examples: input angle[32] theta; output bit result;

Meaning: Marks run-time input parameters and explicitly returned output
variables for parameterized circuits and near-time workflows.

# ================================================================================ 9. INDEXING, SLICING, AND CONCATENATION

---

## 9.1 Index operator

name[index] name[i, j] name[a:b] name[a:c:b] name[{1, 3, 5}]

Meaning: Selects elements, slices, or index sets depending on the target object.

---

## 9.2 Register concatenation

reg1 ++ reg2

Meaning: Combines two same-kind registers into a larger register view.

---

## 9.3 Classical value bit slicing

myInt[0] myInt[3:0] myAngle[-1] myUint[{0, 2, 4}]

Meaning: Extracts bits from int, uint, and angle values as bit arrays.

---

## 9.4 Array concatenation and slicing

arr1 ++ arr2 arr[1:4] arr[-1] arr[1, 2]

Meaning: Combines arrays by copying contents, or slices/indexes them. Array
slicing behaves differently from register slicing because arrays are value-
oriented classical containers.

# ================================================================================ 10. GATE APPLICATION SYNTAX

Basic gate call g q; g q0, q1; g(theta) q; g(theta, phi) q0, q1;

Meaning: Applies a gate to one or more qubits/registers.

Broadcast form g qreg1, qreg2;

Meaning: If one or more operands are registers, matching-length register
operands broadcast elementwise across their positions. This is semantic sugar
for the scalar lane calls obtained by zipping the register positions, one lane
per position. As in OpenQASM 3.1, use of this syntax asserts that the expanded
lane calls are mutually commuting; implementations MAY materialize them in a
deterministic canonical lane order, but that order is representational only. If
the broadcasted gate call carries `localPhase`, that same expression-local
prefix is attached to each expanded lane separately, and any folding into scope
`globalPhase` is analyzed only after this lane-wise expansion.

Special zero-qubit phase gate gphase(gamma); gphase gamma;

Meaning: Applies a global phase. It is the built-in 0-qubit gate and a
first-class instruction form distinct from a scope's scalar `globalPhase`.
Normalization may hoist it into the owning scope scalar only when the
**fold-safe same-scope rule** permits that rewrite in a representation using the
ordinary-scope scalar `globalPhase` semantics of Phase Convention 3. In
particular, an ordinary nested body determines the implicit identity size from
all distinct underlying qubit wires in scope at that body's entry after alias
resolution, not only the wires mentioned by the `gphase` statement itself.
Calibration scopes keep the explicit instruction unless the loaded companion
grammar carries the explicit opt-in contract required for those ordinary-scope
scalar semantics and for applying the same fold-safe rule. When modified by
`ctrl` or `negctrl`, a zero-qubit base gate consumes only the added control
operands; for example, `ctrl @ gphase(gamma)
c;` denotes `P(gamma)` on `c`.

# ================================================================================ 11. GATE DEFINITIONS

Named gate definition gate name qargs { body }

Parameterized gate definition gate name(params) qargs { body }

Examples below use **textual OpenQASM** built-ins, not the SDK's internal
canonical `U`: gate h q { U(pi/2, 0, pi) q; gphase(-pi/4); }

gate cphase(theta) a, b { U(0, 0, theta / 2) a; CX a, b; U(0, 0, -theta / 2) b;
CX a, b; U(0, 0, theta / 2) b; }

Meaning: Defines a gate symbol hierarchically in terms of existing gates and
built-ins.

Rules:

- Gate bodies may contain gate statements/calls and supported looping
  constructs, not arbitrary declarations.
- Gate parameters behave like angle-typed symbolic parameters.
- Qubit arguments inside gate definitions are identifiers, not indexed
  references.

# ================================================================================ 12. GATE MODIFIERS

Inverse inv @ g q;

Meaning: Applies the inverse of a gate.

Power pow(k) @ g q;

Meaning: Applies a powered/iterated modifier form of a gate.

Positive control ctrl @ g c, t; ctrl(n) @ g c1, c2, ..., target_args; ctrl @
gphase(theta) c;

Meaning: Prepends one or more positive controls to a gate.

Negative control negctrl @ g c, t; negctrl(n) @ g c1, c2, ..., target_args;
negctrl @ gphase(theta) c;

Meaning: Prepends one or more negative-polarity controls to a gate.

Modifier chaining inv @ ctrl @ g q0, q1; negctrl @ ctrl(2) @ x a, b, c, t;

Meaning: Modifiers compose as nested gate transformations on the expression to
their right; equivalently, the modifier nearest the base gate acts first and the
leftmost modifier acts last.

Rules:

- `inv` and `pow` apply to the exact matrix/operator produced by the gate and
  any modifiers to their right.
- If a gate expression carries an expression-local zero-qubit phase prefix
  `localPhase`, that prefix is part of the exact operator seen by modifiers.
  Thus `inv`, `pow`, `ctrl`, and `negctrl` act on `exp(i * localPhase) * G`, not
  on `G` alone.
- If an earlier normalization folded a bare gate call's `localPhase` into the
  owning scope scalar, any later modifier-introducing rewrite on that same call
  is legal only after the exact same correction has been re-materialized on the
  gate expression or another exactly equivalent rewrite has been proved.
- If a gate call broadcasts, each expanded lane sees its own copy of
  `localPhase` before modifier application. Modifiers therefore act on the exact
  corrected operator of each lane, not on one shared one-time phase for the
  broadcast as a whole.
- Modifiers nest on the gate expression from right to left, and each control
  modifier prepends its own new control arguments to the current quantum
  argument list. Example: `negctrl @ ctrl @ x a, b, t` enables only on
  `a = 0, b = 1`.
- For a zero-qubit base gate expression, the modified call has only the added
  control operands and no targets. Example: `ctrl @ gphase(theta) c` equals
  `P(theta)` on `c`; `ctrl(2) @ gphase(theta) a, b` equals the doubly-controlled
  phase on `a, b`.
- Integer `pow(k)` means repeated composition; negative integers use inverse.
  Non-integer real `pow(k)` follows Phase Convention 6 and may be expanded only
  when an exact principal-branch rule is defined; otherwise it must remain
  symbolic and be rejected before any backend that requires a concrete unitary.
- `ctrl` and `negctrl` lift the exact full matrix, not an equivalence class
  modulo global phase. `ctrl(n)` and mixed control chains follow the exact
  lifting rule from Phase Convention 4.

# ================================================================================ 13. BUILT-IN QUANTUM INSTRUCTIONS

---

## 13.1 Reset

reset q; reset qreg;

Meaning: Resets one qubit or a qubit register to |0>.

---

## 13.2 Measurement

target = measure source; measure source -> target; measure source;

Examples: bit b = measure q; measure q -> c; measure q;

Meaning: Measures in the Z basis. The assignment and arrow forms are both
supported, and the result may be ignored.

# ================================================================================ 14. CLASSICAL STATEMENTS AND CONTROL FLOW

---

## 14.1 Assignment

lhs = rhs; lhs += rhs; lhs -= rhs; lhs *= rhs; lhs /= rhs; lhs &= rhs; lhs |=
rhs; lhs ~= rhs; lhs ^= rhs; lhs <<= rhs; lhs >>= rhs; lhs %= rhs; lhs **= rhs;

Meaning: Assigns or updates a classical lvalue, or stores a measurement result.

---

## 14.2 Expression statement

expression;

Examples: foo(x); bar(); theta + 1; // syntactically valid expression statement

Meaning: Evaluates an expression as a statement.

---

## 14.3 If / else

if (condition) statement if (condition) statement else statement if (condition)
{ ... } else { ... }

Meaning: Conditional branching.

---

## 14.4 For

for type name in values statement for type name in values { ... }

Examples: for int i in {1, 2, 3} { ... } for int i in [0:3] { ... }

Meaning: Iterates over a discrete set, range, or expression-defined iterable
form accepted by the grammar.

---

## 14.5 While

while (condition) statement while (condition) { ... }

Meaning: Loops while the condition remains true.

---

## 14.6 Break

break;

Meaning: Exits the nearest enclosing for/while loop.

---

## 14.7 Continue

continue;

Meaning: Skips to the next iteration of the nearest enclosing for/while loop.

---

## 14.8 End

end;

Meaning: Terminates the program immediately.

---

## 14.9 Switch / case / default

switch (expr) { case value_list { ... } case value_list { ... } default { ... }
}

Meaning: Multi-branch selection based on a controlling expression.

Notes:

- case labels may contain expression lists.
- default is optional.
- The grammar includes switch, case, and default as real language constructs.

# ================================================================================ 15. SUBROUTINES

Subroutine with return value def name(parameters) -> type { body }

Subroutine without return value def name(parameters) { body }

Examples: def xmeasure(qubit q) -> bit { h q; return measure q; }

def parity(bit[8] cin) -> bit { bit c; for int i in [0:7] { c ^= cin[i]; }
return c; }

Meaning: Defines reusable routines that can take classical and quantum
parameters and optionally return one classical value.

Parameter forms accepted by the grammar:

- scalarType name
- qubitType name
- qreg/creg compatibility arguments
- readonly/mutable array reference arguments

Return statement return; return expression; return measure q;

Meaning: Returns from a subroutine, optionally with a classical value or
measurement result.

# ================================================================================ 16. EXTERN DECLARATIONS

extern declaration extern name(type1, type2, ...) -> outType; extern name;
extern name() -> outType;

Examples: extern popcount(bit[64]) -> uint[64]; extern boxcar(waveform input) ->
complex[float[64]];

Meaning: Declares externally provided classical or calibration-time
functionality that is linked by the toolchain rather than defined inside
OpenQASM itself.

# ================================================================================ 17. SCOPES AND BLOCKS

Anonymous/local block { statement* }

Meaning: Introduces a local block scope for nested statements.

Scope categories described by the spec:

- global scope
- gate/subroutine scope
- local block scope
- calibration scope (dependent on the selected calibration grammar)

# ================================================================================ 18. DIRECTIVES

---

## 18.1 Pragmas

pragma anything to end of line #pragma anything to end of line //
implementation-defined legacy support

Examples: pragma simulator noise model "qpu1.noise" #pragma vendor extension foo

Meaning: Implementation-defined directives to compilers, simulators, or
toolchains.

---

## 18.2 Annotations

@keyword @keyword extra text @keyword more text statement

Examples: @reversible gate my_gate q { ... }

@bind IOPORT[3:2] input bit[2] control_flags;

Meaning: Statement-attached metadata lines. Multiple annotations may precede a
single statement.

# ================================================================================ 19. CIRCUIT TIMING SYNTAX

---

## 19.1 Duration and stretch declarations

duration d = 10ns; stretch s;

Meaning: Represents compile-time time intervals and flexible time intervals.

---

## 19.2 durationof intrinsic

duration x = durationof({ x $0; });

Meaning: Gets the duration of the enclosed block/instruction sequence.

---

## 19.3 Delay

delay[time] q; delay[time] q0, q1; delay[time] frame_name; // in OpenPulse
contexts delay[time] frame1, frame2; // in OpenPulse contexts

Meaning: Advances time on quantum operands or frames by the specified duration.

---

## 19.4 Box

box { ... }

box[duration_expr] { ... }

Meaning: Groups operations into a boxed timing region. The optional designator
attaches a requested timing quantity to the box.

---

## 19.5 Barrier

barrier; barrier q; barrier q0, q1, q2;

Meaning: Introduces a synchronization / ordering boundary for the listed quantum
operands. In OpenPulse frame contexts, barrier also aligns frame clocks.

# ================================================================================ 20. CALIBRATION SYNTAX

---

## 20.1 Inline calibration block

cal { ... }

Meaning: A calibration-language block interpreted using the selected
defcalgrammar.

---

## 20.2 Calibration definition

defcal target(args) operands { ... }

defcal target(args) operands -> return_type { ... }

Examples: defcal rz(angle[20] theta) $0 { ... } defcal measure $0 -> bit { ... }
defcal my_measure q -> complex[float[32]] { ... }

Meaning: Defines the pulse-level or hardware-level implementation of a gate,
reset, delay, measurement, or custom named operation.

Targets allowed by the grammar:

- measure
- reset
- delay
- Identifier (custom operation name)

Operand forms:

- physical qubit identifiers such as $0
- identifier-based wildcard/general qubit operands

Important 3.1 note: In pulse grammars, wildcard-like qubit identifiers are
regular identifiers in 3.1; exact hardware qubits still use $0, $1, and so on.

# ================================================================================ 21. OPENPULSE-SPECIFIC SYNTAX (WHEN USING defcalgrammar "openpulse")

This section covers the additional syntax forms described in the OpenPulse
grammar portion of the 3.1 docs.

---

## 21.1 Port declarations and use

extern port p; extern port drive0; extern port measure0;

Meaning: A port is an exposed hardware I/O resource for driving or observing
qubits.

---

## 21.2 Frame declarations and initialization

extern frame fr; frame local_fr = newframe(port_name, frequency, phase);

Examples: extern frame xy_frame0; frame driveframe0 = newframe(drive0, 5e9,
0.0);

Meaning: A frame is a timing-plus-carrier abstraction with attached port,
frequency, phase, and implicit time.

---

## 21.3 Frame state operations

set_phase(frame fr, angle phase); shift_phase(frame fr, angle phase);
get_phase(frame fr) -> angle;

set_frequency(frame fr, float freq); shift_frequency(frame fr, float freq);
get_frequency(frame fr) -> float;

Meaning: Manipulates or reads the current carrier state of a frame.

---

## 21.4 Waveform values

waveform wf; waveform arb = [1+0im, 0+1im, 0.5+0.5im];

Meaning: Represents a pulse envelope, either as explicit samples or via template
functions.

---

## 21.5 Waveform template externs

extern gaussian(... ) -> waveform; extern gaussian_square(... ) -> waveform;
extern drag(... ) -> waveform; extern constant(... ) -> waveform; extern
sine(... ) -> waveform;

Meaning: Target-provided waveform builders described in the OpenPulse
documentation.

---

## 21.6 Waveform processing helpers

mix(wf1, wf2) -> waveform; sum(wf1, wf2) -> waveform; phase_shift(wf, ang) ->
waveform; scale(wf, factor) -> waveform;

Meaning: Creates transformed waveforms from existing ones.

---

## 21.7 Play

play(frame fr, waveform wfm);

Meaning: Schedules transmission of a waveform on a frame. Valid only in
cal/defcal OpenPulse contexts.

---

## 21.8 Capture

capture_vendor_defined(frame fr, ...);

Meaning: Schedules acquisition on a frame. The spec treats capture as a special
vendor- defined extern instruction family, with frame as the minimum required
parameter.

---

## 21.9 OpenPulse timing rules

delay[time] frame; barrier frame1, frame2; play(frame, waveform);
capture_vendor_defined(frame, ...);

Meaning: These advance or align frame clocks. Each frame maintains its own
duration-valued clock.

---

## 21.10 OpenPulse phase tracking

Meaning: Frame phase evolves both explicitly (set_phase / shift_phase) and
implicitly as time advances through delay, play, and capture.

# ================================================================================ 22. FULL STATEMENT CATALOG FROM THE REFERENCE GRAMMAR

The official grammar’s statement forms can be summarized as:

pragma annotation* aliasDeclarationStatement annotation* assignmentStatement
annotation* barrierStatement annotation* boxStatement annotation* breakStatement
annotation* calStatement annotation* calibrationGrammarStatement annotation*
classicalDeclarationStatement annotation* constDeclarationStatement annotation*
continueStatement annotation* defStatement annotation* defcalStatement
annotation* delayStatement annotation* endStatement annotation*
expressionStatement annotation* externStatement annotation* forStatement
annotation* gateCallStatement annotation* gateStatement annotation* ifStatement
annotation* includeStatement annotation* ioDeclarationStatement annotation*
measureArrowAssignmentStatement annotation* oldStyleDeclarationStatement
annotation* quantumDeclarationStatement annotation* resetStatement annotation*
returnStatement annotation* switchStatement annotation* whileStatement scope

Meaning: This is the complete parser-level list of statement categories
recognized by the official grammar page in the OpenQASM 3.1 documentation set.

# ================================================================================ 23. PRACTICAL CHECKLIST OF SYNTAX CATEGORIES

If you are implementing a parser, transpiler, or syntax checker, you should make
sure your implementation explicitly covers all of the following categories:

1. Comments
2. Optional version declaration
3. include statements
4. defcalgrammar statements
5. Identifiers and physical qubit tokens
6. All literal forms
7. All scalar types
8. qubit / qreg / bit / creg declarations
9. array declarations and array-reference parameter types
10. const declarations
11. input/output declarations
12. casts
13. indexing, slicing, ranges, set expressions, and concatenation
14. full expression precedence and associativity
15. gate applications
16. gate definitions
17. gate modifiers: inv, pow, ctrl, negctrl
18. built-in quantum instructions: reset, measure
19. assignments and compound assignments
20. if / else
21. for / while
22. break / continue / end
23. switch / case / default
24. subroutine definitions and returns
25. extern declarations and calls
26. local scopes / blocks
27. pragmas and annotations
28. timing syntax: duration, stretch, durationof, delay, box, barrier
29. cal and defcal syntax
30. OpenPulse constructs when openpulse is selected

# ================================================================================ 24. OPENQASM 3.1-SPECIFIC NOTES WORTH REMEMBERING

1. The documentation set is explicitly for OpenQASM 3.1, but the linked grammar
   page is titled "OpenQasm 3.0 Grammar". It is still the official grammar page
   used by the 3.1 docs.

2. The 3.1 release notes say that pulse-grammar wildcard identifiers are now
   regular identifiers without a leading dollar sign. Exact hardware qubits
   still use forms like $0.

3. The 3.1 release notes also note that switch/case/default are now real syntax
   in the language, not merely reserved placeholders for future expansion.

## 5. Expansion API

The idea behind the expansion API is to have a high-level API for each existing
feature in the openqasm 3.1 specification, removing the already defined gates.
This section defines the language-agnostic expansion API that complements the
gate catalog. The listed gates are treated as the supported concrete gate set,
while this API defines all other constructs required to represent, build,
analyze, serialize, and transpile OpenQASM 3.1 programs. The design goal is
semantic completeness rather than syntactic mimicry: every OpenQASM 3.1
construct should map to an explicit API object, statement, expression node, or
modifier, even if the host language uses different syntax.

The expansion API MUST support:

- file-level program structure, including version declaration, includes, and
  calibration-grammar selection;
- all classical and quantum declaration forms needed by OpenQASM 3.1;
- expressions, indexing, slicing, ranges, sets, casts, and concatenation;
- gate application, gate modifiers, and user-defined gate declarations using the
  existing gate catalog;
- built-in quantum instructions such as reset and measurement;
- classical statements and control flow, including if, for, while, break,
  continue, end, switch, case, and default;
- subroutines, return values, external declarations, block scopes, annotations,
  and pragmas;
- circuit timing constructs, including duration, stretch, durationof, delay,
  box, and barrier;
- calibration and OpenPulse constructs, including cal, defcal, ports, frames,
  waveforms, play, capture, and frame-state operations.

### 5.1 Design Principles

1. **Language-agnostic surface.** The API MUST be expressible in any target
   language without depending on language-specific syntax features.
2. **Semantic preservation.** The API MUST preserve the meaning of OpenQASM 3.1
   constructs exactly, even when serialization requires host-language
   adaptation.
3. **Structured IR model.** The API SHOULD be modeled as a typed intermediate
   representation composed of program nodes, declarations, expressions,
   statements, and calibration constructs.
4. **Round-trip fidelity.** A valid program built with this API SHOULD be
   serializable to OpenQASM 3.1 and re-importable without semantic loss, except
   where a backend explicitly documents unsupported target features.
5. **Gate separation.** The gate catalog and this expansion API are separate
   concerns. Gate names come from the supported gate set or from user-defined
   gates, while all surrounding language features are modeled here.
6. **Two-layer architecture.** The API is organized in two layers: (a) an **IR
   node model** that defines the typed AST nodes (`Expression`, `Statement`,
   `GateCall`, etc.) and (b) a **builder layer** on `QuantumCircuit` that
   appends statements and manages scope state. Expression construction is
   separated from statement appending: expression factories (§5.5) are
   standalone constructors that produce `Expression` nodes without side effects,
   while `QuantumCircuit` methods append statements that may reference those
   expressions. Implementations SHOULD provide expression factories as a
   standalone module or set of static methods (e.g. `Expr.ref("x")`,
   `Expr.binary("+", a, b)`) rather than placing them as instance methods on
   `QuantumCircuit`.
7. **Single container.** `QuantumCircuit` serves as both the top-level program
   container and the scope/body type for nested contexts (gate bodies,
   subroutine bodies, control-flow branches, box regions). Program-level
   metadata (version, includes, defcal grammar) is carried on the
   `QuantumCircuit` instance; the `Program` model in §5.2 is an abstract
   specification of those fields, not a separate class. This avoids duplication
   while keeping the `QuantumCircuit` type reusable for nested scopes (where
   program-level metadata is simply absent/ignored).

### 5.2 Core Program Model

Per Design Principle 7, `QuantumCircuit` is the concrete carrier for the program
model. The following abstract specification defines the fields that a top-level
`QuantumCircuit` instance MUST carry. These same fields exist on every
`QuantumCircuit` instance but are semantically meaningful only at the top-level
(global) scope; nested scopes (gate bodies, subroutine bodies, control-flow
branches, box regions) ignore program-level fields such as `version` and
`includes`.

```text
Program (abstract model, realized by QuantumCircuit) {
  version: VersionSpec?
  includes: IncludeSpec[]
  defcalGrammar: string?
  statements: Statement[]
  globalPhase: Expression           // scalar phase for this scope (Phase Convention 3)
  classicalRegisters: ClassicalRegister[]  // ordered named registers
  transpilationMetadata: TranspilationMetadata?  // optional post-compilation provenance
}

VersionSpec { major: integer, minor: integer? }
IncludeSpec { path: string }

TranspilationMetadata {
  initialLayout: map<integer, integer>?  // virtual qubit → physical qubit
  routingSwaps: SwapRecord[]?            // inserted SWAPs with location info
  targetDevice: string?                  // device name or identifier
  basisGateSet: string[]?                // basis gates used in compilation
  couplingMap: [integer, integer][]?     // coupling map used for routing
}

SwapRecord {
  qubit0: integer
  qubit1: integer
  insertedBeforeInstruction: integer     // index in the instruction list
}
```

Required construction operations (realized as `QuantumCircuit` methods):

```text
QuantumCircuit(globalPhase = 0) -> QuantumCircuit
qc.setProgramVersion(major, minor?)
qc.omitProgramVersion()
qc.include(path)
qc.setCalibrationGrammar(grammarName)
```

The program model MUST support global-scope statements and nested scopes. The
`transpilationMetadata` field is populated by compilation-pipeline stages and is
`null` on manually constructed circuits. Backend serializers MAY use it for
validation but MUST NOT require it for circuits that have not been transpiled.

### 5.3 Type System

The API MUST represent the OpenQASM 3.1 type system directly.

```text
Type :=
  QubitType(size?)
  BitType(width?)
  IntType(width?)
  UIntType(width?)
  FloatType(width?)
  AngleType(width?)
  BoolType()
  ComplexType(componentType?)
  DurationType()
  StretchType()
  ArrayType(baseType, dimensions[])
  VoidType()
  LegacyQRegType(size)
  LegacyCRegType(size)
```

The API MUST also support parameter-reference forms for arrays:

```text
ArrayReferenceMode := ReadOnly | Mutable

ArrayReferenceType {
  baseType: Type
  constraint: ExactDimensions(sizes[]) | RankOnly(rank)  // union type
  mode: ArrayReferenceMode
}
```

`ExactDimensions(sizes[])` corresponds to `readonly array[int[32], 4]` (exact
size known), while `RankOnly(rank)` corresponds to
`mutable array[float[64],
#dim = 2]` (only the number of dimensions is
constrained). Implementations MUST distinguish these two forms to correctly
validate call sites and serialize the `#dim` syntax.

### 5.4 Identifiers, Operands, and References

The API MUST distinguish at least the following reference categories:

```text
IdentifierRef(name)
PhysicalQubitRef(index)           // corresponds to $0, $1, ...
IndexedRef(base, indices[])
SlicedRef(base, sliceSpec)
ConcatenatedRef(parts[])
AliasRef(name)
```

The API MUST allow both virtual qubits and physical qubits to appear where
OpenQASM 3.1 permits them.

### 5.5 Literals and Expressions

The expression system MUST be rich enough to encode all classical expression
forms and OpenQASM-specific constructs.

```text
Expression :=
  // Literals
  IntegerLiteral(value, base?)        // base: decimal (default), binary, octal, hex
  FloatLiteral(value)
  ImaginaryLiteral(value)
  BooleanLiteral(value)
  BitstringLiteral(value)
  DurationLiteral(value, unit)        // unit: ns, us, ms, s, dt
  BuiltInConstant(name)               // "pi", "tau", "euler", "im"

  // References and identifiers
  IdentifierExpr(name)
  PhysicalQubitExpr(index)

  // Compound literals
  ArrayLiteral(elements[])
  SetLiteral(elements[])
  RangeExpr(start?, step?, end?)

  // Operators
  UnaryExpr(op, operand)
  BinaryExpr(op, left, right)

  // Type operations
  CastExpr(targetType, value)
  SizeOfExpr(target, dimension?)      // sizeof(target) or sizeof(target, dim)
  RealPartExpr(operand)               // real(expr) — extracts real component
  ImagPartExpr(operand)               // imag(expr) — extracts imaginary component

  // Invocations
  CallExpr(callee, args[])            // subroutine/extern/built-in function calls

  // Indexing and structure
  IndexExpr(base, selectors[])        // a[i], a[i,j], a[0:3], a[{1,3,5}]
  ConcatenationExpr(parts[])          // a ++ b ++ c

  // Quantum-specific
  MeasureExpr(source)                 // measure q as expression
  DurationOfExpr(body)                // durationof({...})

  // Grouping
  ParenthesizedExpr(inner)
```

`BuiltInConstant` represents OpenQASM 3.1 built-in named constants that are
distinct from user-declared identifiers. Implementations MUST recognize at least
`pi`, `tau` (`2*pi`), `euler` (Euler's number), and `im` (imaginary unit).

`SizeOfExpr` represents the `sizeof` intrinsic. When `dimension` is absent, it
returns the size of the first (or only) dimension; when present, it returns the
size of the specified dimension.

`RealPartExpr` and `ImagPartExpr` represent the built-in `real()` and `imag()`
extraction functions for complex-typed expressions.

`IntegerLiteral.base` is an optional source-preservation hint. The semantic
value is always the integer itself; `base` records whether the original source
used `0b`, `0o`, `0x`, or decimal notation for round-trip fidelity.

The API MUST support the following operator families:

- arithmetic: `+`, `-`, `*`, `/`, `%`, `**`
- bitwise: `~`, `&`, `^`, `|`, `<<`, `>>`
- logical: `!`, `&&`, `||`
- comparison: `<`, `<=`, `>`, `>=`, `==`, `!=`
- concatenation: `++`

The API SHOULD preserve operator precedence in the expression tree rather than
relying on source formatting.

**Built-in function names.** In addition to the expression nodes above, the
`CallExpr` node MUST support the following built-in function identifiers
recognized by OpenQASM 3.1: `arccos`, `arcsin`, `arctan`, `ceiling`, `cos`,
`exp`, `floor`, `log`, `mod`, `popcount`, `pow`, `rotl`, `rotr`, `sin`, `sqrt`,
`tan`. Implementations MUST distinguish built-in function calls from
user-defined subroutine calls during validation and serialization.

**Expression factory guidance.** Per Design Principle 6, expression nodes are
pure value objects. Implementations SHOULD provide standalone expression
factories (e.g. `Expr.int(42)`, `Expr.binary("+", a, b)`, `Expr.ref("theta")`,
`Expr.range(0, 10)`) as a separate module or set of static methods. The
`QuantumCircuit` builder methods that accept expression arguments SHOULD accept
these factory-produced nodes. The builder MAY additionally offer shorthand
convenience methods that internally construct expression nodes, but the
standalone factories MUST be available for programmatic IR construction and
round-trip deserialization.

### 5.6 Declarations and I/O

The API MUST represent all OpenQASM 3.1 declaration categories.

```text
Statement :=
  ClassicalDeclaration(type, name, initializer?)
  QuantumDeclaration(type, name)
  ConstDeclaration(type, name, initializer)
  InputDeclaration(type, name, defaultValue?)
  OutputDeclaration(type, name, initializer?)
  AliasDeclaration(name, value)
  LegacyRegisterDeclaration(kind, name, size)   // kind: QReg | CReg
  ...
```

`OutputDeclaration.initializer` is optional and corresponds to
`output bit result = 0;` forms in OpenQASM 3.1. When absent, the output variable
is uninitialized. `InputDeclaration.defaultValue` is reserved for future use by
near-time workflow extensions; implementations MAY accept it but MUST serialize
it only when the target format supports input defaults.

Required helpers:

```text
declareClassical(type, name, initializer?)
declareQuantum(name, size?)
declareConst(type, name, initializer)
declareInput(type, name, defaultValue?)
declareOutput(type, name, initializer?)
declareAlias(name, expression)
declareLegacyQReg(name, size)
declareLegacyCReg(name, size)
```

`declareLegacyQReg` and `declareLegacyCReg` emit OpenQASM 2-style `qreg`/`creg`
declarations. They are required for backward compatibility with circuits that
use the legacy syntax. Implementations MUST preserve the distinction between
legacy `creg` and modern `bit[N]` declarations through serialization, because
some backends interpret them differently.

### 5.7 Gate Application API

The gate list already defines the supported concrete gates. This expansion API
MUST provide a generic gate invocation layer instead of separate semantics for
every gate.

```text
GateName :=
  BuiltInGateName(name)           // one of the supported gate catalog entries
  UserDefinedGateName(name)

GateModifier :=
  InverseModifier()
  PowerModifier(exponent)
  ControlModifier(count?)
  NegativeControlModifier(count?)

GateCall {
  gate: GateName
  parameters: Expression[]
  operands: Operand[]
  localPhase: Expression      // default 0; expression-local zero-qubit prefix applied before modifiers
  modifiers: GateModifier[]   // outermost-first order (see below)
  surfaceName?: string        // optional original surface gate family such as "U", "u3", "phase", "cp"; required when a source-preserving / exact-round-trip API needs syntax not recoverable from semantic gate + localPhase alone
}
```

**Modifier array order convention.** The `modifiers` array stores modifiers in
**outermost-first** (syntactic left-to-right) order. For textual OpenQASM
`inv @ ctrl @ x`, the array is `[InverseModifier(), ControlModifier(1)]`. The
semantic application order is right-to-left: the last element in the array is
applied to the base gate first, then the second-to-last, and so on. This matches
the textual modifier syntax and avoids reversals during
serialization/deserialization. For example, `negctrl @ ctrl(2) @ pow(3) @ h`
stores as `[NegativeControlModifier(1), ControlModifier(2), PowerModifier(3)]`
and semantically means `negctrl(ctrl^2(pow(3)(h)))`.

Required helpers:

```text
applyGate(name, operands[], parameters? = [], modifiers? = [], localPhase? = 0, surfaceName? = null)
inv(gateCall)
pow(exponent, gateCall)
ctrl(gateCall, count? = 1)
negctrl(gateCall, count? = 1)
```

A `GateCall` with base gate/operator `G` and `localPhase = delta` denotes
`exp(i * delta) * G` before any modifiers are applied. `localPhase` defaults to
zero. It exists to preserve phase-exact gate-expression-local translations such
as textual OpenQASM `U`, `u2`, and `u3`, and it is distinct from both an
explicit `gphase`/`globalPhaseGate` instruction and the owning scope's scalar
`globalPhase`. For a zero-qubit base gate under control modifiers, `operands`
contains only the prepended controls because the base arity contributes no
target operands.

A normalized semantic IR MAY omit `surfaceName`. A source-preserving or exact
round-trip IR MUST preserve enough syntax-local metadata to recover the
documented gate-expression surface identity, exact modifier stack,
operand/reference forms, and exact parameter/localPhase expression trees
wherever those are not uniquely determined by the semantic gate/operator plus
`localPhase`; `surfaceName` is one conforming mechanism for preserving the
gate-family spelling specifically.

Any public circuit-builder or parser/deserializer surface that claims exact
OpenQASM round-trip MUST expose this generic gate-call layer or another provably
equivalent mechanism. Named convenience methods such as `u`, `p`, `cp`, and `cu`
are normalized semantic constructors under the SDK's internal phase convention;
they are insufficient on their own to preserve a parsed textual
`U`/`u2`/`u3`/`phase`/`cphase` spelling, the parsed operand/reference form,
exact parameter-expression tree, exact `localPhase` expression, or an arbitrary
modifier stack.

Final textual OpenQASM emission is stricter than semantic IR storage: textual
OpenQASM has no generic syntax for an arbitrary `GateCall.localPhase`. A
serializer may therefore lower a nonzero `localPhase` only by the exact
mechanisms permitted by this Phase Convention: an exact same-expression textual
gate-family encoding, any same-scope fold-safe compensation there explicitly
allowed for a `bare, unmodified gate instruction`, an exact helper-gate or
equivalent target-language helper encoding when the active serializer/API
contract permits introducing it, or the exact combination of those mechanisms.
If the full correction cannot be realized without residual instruction-local
phase inside the emitted call, the serializer MUST reject the circuit rather
than extract the phase as a sibling `gphase` that changes modifier semantics.

If an implementation temporarily folds the `localPhase` of a `GateCall` that
appears as a `bare, unmodified gate instruction` into the immediate owning scope
scalar under Section 2, that folded representation remains valid only while
later transforms do not need to treat that same call as the operand of outer
modifiers. Any transform that would apply or expose `ctrl`, `negctrl`, `inv`,
`pow`, or another exact gate-expression transform around that call MUST first
restore the same correction as `localPhase` on the `GateCall`, or prove another
exactly equivalent rewrite.

When `operands` broadcast over registers, the `GateCall` denotes the lane-wise
family of scalar gate calls across the broadcast positions. The stored
`localPhase` belongs to each expanded lane separately; it is not a single
statement-level phase shared across the whole broadcast. Any deterministic lane
order used by an IR is representational only and does not add semantics beyond
the broadcast's commuting-lanes contract.

The API MUST support:

- zero-qubit global phase application;
- expression-local zero-qubit phase prefixes on gate calls;
- parameterized gate calls;
- broadcasting over register operands where allowed by OpenQASM 3.1;
- chained modifiers in semantic order;
- user-defined gate invocation by identifier.

For `gphase`, the generic gate layer MUST preserve the explicit zero-qubit
instruction form even if a later normalization pass hoists eligible occurrences
into the owning scope scalar `globalPhase`. This explicit instruction form is
distinct from the `localPhase` prefix attached to another `GateCall`.

### 5.8 Gate Definitions

The API MUST allow hierarchical gate declarations that are separate from
ordinary subroutines.

```text
GateDefinition {
  name: string
  parameters: Parameter[]
  qubits: QubitParameter[]
  body: QuantumCircuit        // nested ordinary scope; owns the gate body's scalar `globalPhase`
}
```

Required helpers:

```text
defineGate(name, qubits[], body)
defineParameterizedGate(name, parameters[], qubits[], body)
```

The nested `QuantumCircuit` body is validated against the OpenQASM gate-body
grammar. It therefore carries its own scalar `globalPhase` through the
`QuantumCircuit(globalPhase = 0)` constructor, but the implementation MUST
reject declarations or any other statements forbidden inside a gate definition.
An API MAY additionally accept a raw `GateBodyStatement[]` shorthand and wrap it
as a nested `QuantumCircuit` with `globalPhase = 0`.

### 5.9 Built-In Quantum Instructions

The API MUST represent quantum instructions that are not ordinary gate calls.

```text
ResetStatement(target)

MeasureStatement {
  source: Operand                     // qubit or register being measured
  target: Operand?                    // classical bit/register receiving result (null for bare measure)
  syntax: MeasurementSyntax           // Assignment | Arrow | Bare
}

MeasurementSyntax := Assignment | Arrow | Bare
```

The unified `MeasureStatement` replaces separate `MeasureAssignment` and
`MeasureArrowStatement` IR nodes. The `syntax` field preserves round-trip
fidelity:

- `Assignment`: `bit b = measure q;` or `c = measure q;`
- `Arrow`: `measure q -> c;`
- `Bare`: `measure q;` (result ignored or used as an expression elsewhere)

When `syntax` is `Bare`, `target` MUST be null. When `syntax` is `Assignment` or
`Arrow`, `target` MUST be non-null. The `MeasureExpr` expression node from §5.5
covers the case where measurement appears inside an expression context (e.g.
`bit b = measure q;` where the right-hand side is a `MeasureExpr`).

Required helpers:

```text
reset(target)
measure(source, target?, syntax? = Assignment) -> MeasureStatement
```

The single `measure` helper covers all forms: `measure(q, c)` for assignment,
`measure(q, c, Arrow)` for arrow syntax, and `measure(q)` for bare measurement.
When used as an expression (right-hand side of an assignment), the builder
SHOULD use `MeasureExpr` from §5.5 instead.

### 5.10 Classical Statements and Control Flow

The API MUST support the full OpenQASM 3.1 control-flow surface.

```text
Assignment {
  target: Operand
  operator: AssignOp                  // "=" | "+=" | "-=" | "*=" | "/=" | "&=" | "|=" | "~=" | "^=" | "<<=" | ">>=" | "%=" | "**="
  value: Expression
}

ExpressionStatement(expression)       // bare side-effect expression, e.g. foo();
ReturnStatement(value?)               // return; or return expr; or return measure q;
IfStatement(condition, thenBody: QuantumCircuit, elseBody?: QuantumCircuit)
ForInStatement(loopType, loopVariable, iterable, body: QuantumCircuit)
WhileStatement(condition, body: QuantumCircuit)
BreakStatement()
ContinueStatement()
EndStatement()
SwitchStatement(subject, cases[], defaultBody?: QuantumCircuit)
CaseClause(values[], body: QuantumCircuit)
Block(body: QuantumCircuit)
LineComment(content)                  // // text
BlockComment(content)                 // /* text */
```

`ReturnStatement` is included here (not only in §5.11) because it is a
first-class statement that may appear in any subroutine body. When `value` is
absent, the return is void. When `value` is a `MeasureExpr`, this corresponds to
`return measure q;` in OpenQASM 3.1.

`Assignment.operator` explicitly enumerates all compound-assignment forms.
Simple assignment uses `"="`. Implementations MUST support all 13 compound
assignment operators listed above.

`LineComment` and `BlockComment` are statement-level IR nodes that preserve
comment content and positioning for round-trip fidelity. They carry no semantic
meaning and MUST be ignored by all semantic analysis, compilation, and
simulation passes.

Required helpers:

```text
assign(target, value)
assignOp(target, operator, value)
exprStatement(expression)
returnValue(value)
returnVoid()
ifThen(condition, thenBody, elseBody?)
forIn(loopType, variableName, iterable, body)
whileLoop(condition, body)
breakLoop()
continueLoop()
endProgram()
switchOn(subject, cases[], defaultBody?)
block(body)
```

Every non-calibration nested body in this subsection is a nested
`QuantumCircuit` ordinary scope and therefore owns its own scalar `globalPhase`.
An API MAY accept raw `Statement[]` shorthand for such bodies and wrap it as
`QuantumCircuit(globalPhase = 0)`.

### 5.11 Subroutines and Returns

The API MUST distinguish subroutines from gates.

```text
Parameter {
  name: string
  type: Type | ArrayReferenceType
}

SubroutineDefinition {
  name: string
  parameters: Parameter[]
  returnType: Type?
  body: QuantumCircuit
}

ReturnStatement(value?)
```

Required helpers:

```text
defineSubroutine(name, parameters[], body, returnType?)
returnValue(value)
returnVoid()
```

The API MUST support classical parameters, qubit parameters, legacy
compatibility parameters where needed, and readonly or mutable array-reference
parameters. The nested `QuantumCircuit` body is an ordinary scope and carries
the subroutine body's scalar `globalPhase`. Unlike gate bodies, subroutine
bodies may use the full statement surface allowed by OpenQASM.

### 5.12 Extern Declarations

The API MUST support declarations for externally provided functionality.

```text
ExternDeclaration {
  name: string
  parameters: Type[]
  returnType: Type?
}
```

Required helpers:

```text
declareExtern(name, parameterTypes[] = [], returnType?)
```

Externs MUST be usable from both the classical language layer and the
calibration layer whenever the target model permits it.

### 5.13 Pragmas and Annotations

The API MUST preserve implementation-defined metadata and statement annotations.

```text
Pragma(text)
Annotation(keyword, payload?)
AnnotatedStatement(annotations[], statement)
```

Required helpers:

```text
pragma(text)
annotate(statement, annotations[])
annotation(keyword, payload?)
```

Annotations MUST attach to the following statement as structured metadata, not
as discarded comments.

### 5.14 Timing and Scheduling

The API MUST represent OpenQASM timing constructs explicitly.

```text
DurationOf(body: QuantumCircuit)
DelayStatement(duration, targets[])
BoxStatement(duration?, body: QuantumCircuit)
BarrierStatement(targets[])
StretchDeclaration(name, initializer?)
DurationDeclaration(name, initializer?)
```

Required helpers:

```text
durationOf(body)
delay(durationExpr, targets[])
box(body, durationExpr?)
barrier(targets[])
declareDuration(name, initializer?)
declareStretch(name, initializer?)
```

Like other non-calibration nested bodies, `BoxStatement.body` is an ordinary
nested `QuantumCircuit` scope and therefore owns its own scalar `globalPhase`.
`DurationOf(body)` observes the timing of the supplied body without discarding
its phase metadata.

The implementation MUST preserve whether a delay or barrier targets qubits,
registers, frames, or other calibration-time timing objects.

### 5.15 Calibration and Defcal

The API MUST support inline calibration blocks and calibration definitions.

```text
CalBlock(body)
DefcalDefinition {
  target: string | BuiltInCalibrationTarget
  parameters: Parameter[]
  operands: Operand[]
  returnType: Type?
  body: CalibrationStatement[]
}

BuiltInCalibrationTarget := Measure | Reset | Delay | CustomIdentifier(name)
```

Required helpers:

```text
cal(body)
defineDefcal(target, parameters[], operands[], body, returnType?)
```

The API MUST support both exact physical qubit operands and generic
identifier-based operands where OpenQASM 3.1 allows them.

### 5.16 OpenPulse Extension Surface

When `defcalgrammar "openpulse"` is selected, the API MUST additionally support
the OpenPulse object model.

```text
PortType()
FrameType()
WaveformType()

PortDeclaration(name)
FrameDeclaration(name)
NewFrame(name, port, frequency, phase)
SetPhase(frame, phase)
ShiftPhase(frame, phase)
GetPhase(frame)
SetFrequency(frame, frequency)
ShiftFrequency(frame, frequency)
GetFrequency(frame)
WaveformDeclaration(name, initializer?)
PlayStatement(frame, waveform)
CaptureStatement(kind, frame, args[])
WaveformTransform(kind, args[])
```

Required helpers:

```text
declarePort(name)
declareFrame(name)
newFrame(name, port, frequency, phase)
setPhase(frame, phase)
shiftPhase(frame, phase)
getPhase(frame)
setFrequency(frame, frequency)
shiftFrequency(frame, frequency)
getFrequency(frame)
declareWaveform(name, initializer?)
play(frame, waveform)
capture(kind, frame, args[])
waveformOp(kind, args[])
```

The API SHOULD support the standard waveform-building and waveform-processing
patterns used by OpenPulse-capable toolchains, such as gaussian-like templates,
constant waveforms, summation, mixing, scaling, and phase shifting.

### 5.17 Scopes and Bodies

The API MUST represent scopes explicitly.

```text
ScopeKind := Global | LocalBlock | Subroutine | Gate | Calibration | Box | ControlFlow
Body(kind, statements[])
```

- `Global`: the top-level program scope.
- `LocalBlock`: an anonymous `{ ... }` block.
- `Subroutine`: a `def` body.
- `Gate`: a `gate` body (restricted to gate-body grammar).
- `Calibration`: a `cal` or `defcal` body (interpreted under the selected
  calibration grammar).
- `Box`: a `box` body (ordinary scope with optional timing designator).
- `ControlFlow`: the body of an `if`, `else`, `for`, `while`, or `switch/case`
  branch.

All scope kinds except `Calibration` are ordinary scopes under Phase Convention
3 and therefore own their own scalar `globalPhase`. This is necessary to
preserve OpenQASM validity rules, because some constructs are legal only in
specific scopes. For example, `break` and `continue` are legal only inside
`ControlFlow` scopes nested within a loop, and declarations are restricted
inside `Gate` scopes.

### 5.18 Serialization and Validation Requirements

A conforming implementation MUST provide:

```text
serializeToOpenQASM31(program) -> string
validateProgram(program) -> ValidationResult
```

**Validation result model:**

```text
ValidationResult {
  valid: boolean
  diagnostics: Diagnostic[]
}

Diagnostic {
  severity: Error | Warning | Info
  category: ValidationCategory
  message: string
  location: SourceLocation?           // null for programmatic circuits without source info
}

ValidationCategory :=
  VersionPlacement | IncludePlacement | DeclarationConsistency |
  TypeMismatch | ScopePlacement | GateBodyRestriction |
  ModifierStructure | MeasurementTarget | TimingTarget |
  CalibrationConsistency | UndefinedReference | DuplicateDeclaration |
  InvalidOperand | PhaseConventionViolation

SourceLocation {
  statementIndex: integer?            // index in the instruction/statement list
  scopePath: string[]?                // e.g. ["main", "if_body", "for_body"]
}
```

Implementations MUST collect all diagnostics rather than failing on the first
error, so that users can fix multiple issues in a single iteration. The `valid`
field is `true` if and only if no `Error`-severity diagnostics are present.
`Warning` and `Info` diagnostics do not prevent serialization or execution.

The validator MUST check at least:

- version placement and uniqueness;
- include placement restrictions;
- declaration and use-site consistency (including undefined references and
  duplicate declarations);
- type correctness for expressions, assignments, and returns;
- legal scope placement for statements (e.g. `break`/`continue` only in loops,
  `return` only in subroutines);
- legal gate-body restrictions for gate definitions;
- correct modifier application structure;
- measurement target compatibility;
- timing-target compatibility;
- defcal and OpenPulse object consistency;
- phase convention violations (e.g. non-entry-invariant scope scalar phase,
  illegal ancestor hoisting).

### 5.19 State Preparation and Initialization

The API MUST support state preparation as first-class instructions with explicit
semantics.

```text
PrepareStateStatement {
  state: StateSpec
  qubits: Operand[]?                 // if null, allocates new qubits as needed
}

InitializeStatement {
  state: StateSpec
  qubits: Operand[]
}

StateSpec :=
  AmplitudeVector(amplitudes: Complex[])   // length must be 2^n for n qubits
  BasisState(value: integer)               // computational basis state |value⟩
  BitstringState(bits: string)             // e.g. "1010"
```

`PrepareStateStatement` assumes the target qubits are already in `|0...0⟩` and
applies a unitary that maps `|0...0⟩` to the target state. It is equivalent to
synthesizing a `UnitaryGate` whose first column is the target state vector. If
`qubits` is null, the circuit allocates the required number of qubits
implicitly.

`InitializeStatement` first resets all target qubits to `|0...0⟩` (via
measurement and conditional X), then applies the same state-preparation unitary
as `PrepareStateStatement`. This makes it safe to use mid-circuit on qubits that
may not be in `|0...0⟩`.

For `AmplitudeVector`, the amplitudes MUST satisfy `sum |a_i|^2 = 1` within
epsilon `1e-10` for concrete numeric inputs; symbolic amplitudes MUST remain
opaque until bound.

For `BasisState(value)`, the state is `|value⟩` in the little-endian convention
of Section 2. The number of qubits is determined by the target register size.
For example, `BasisState(3)` on a 2-qubit register produces `|11⟩`.

For `BitstringState("10")`, the string is interpreted left-to-right as qubit 0
first (MSB of the bitstring = qubit 0). The number of qubits is the string
length.

**Decomposition.** Both instructions are ultimately lowered to gate sequences:
`BasisState` and `BitstringState` decompose to X gates on the appropriate
qubits; `AmplitudeVector` decomposes via `UnitaryGate` synthesis (Tier 7) of the
state-preparation unitary. The `InitializeStatement` additionally prepends a
reset on each target qubit.

### 5.20 Timed Operations

The API MUST support attaching timing designators to gate calls and other
operations for backends that require scheduling information.

```text
TimedOperation {
  operation: GateCall | MeasureStatement | ResetStatement
  duration: Expression                // duration expression (e.g. 100ns, 2*dt)
  qubits: Operand[]?                 // if null, inherited from operation
}
```

A `TimedOperation` wraps a single operation with an explicit duration
designator. It is semantically equivalent to a single-instruction `box` with a
duration expression:

```
box[duration] { operation; }
```

The `timed()` builder method is sugar for this construct. Unlike `delay`, which
advances time without applying any operation, a `TimedOperation` specifies that
the wrapped operation should complete within the given duration. Backends that
do not support timing constraints MUST ignore the duration designator and
execute the operation normally.

Required helpers:

```text
timed(operation, duration, qubits?)
```

When the wrapped operation is a `GateCall` with modifiers, the timing designator
applies to the full modified gate expression, not to the base gate alone.

## 6. IBM Backend

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

  isBrowser = runtime automatically detects browser environment/browser worker contexts
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

## 7. QBRaid Backend

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

---

## 8. Implementation Plan

### 8.1 Project Structure

Adapt file extensions to the target language. The logical module layout is:

```
project-root/
├── AGENTS.md               # This specification
├── README.md               # Generated after all tests pass (Section 12)
├── src/
│   ├── types.{ext}         # Shared interfaces, type aliases, enums
│   ├── complex.{ext}       # Complex number class (field ℂ)
│   ├── matrix.{ext}        # Matrix class over ℂ
│   ├── gates.{ext}         # Gate matrix constructors (pure functions, Tiers 0–14)
│   ├── expansion.{ext}     # Expansion API (Section 5): all non-gate QuantumCircuit features
│   ├── parameter.{ext}     # Symbolic parameter & expression system
│   ├── circuit.{ext}       # QuantumCircuit builder (mutable, chainable)
│   ├── transpiler.{ext}    # Transpiler interface + OpenQASMTranspiler (serialize/deserialize + compilation pipeline)
│   ├── simulator.{ext}     # SimulatorBackend: state-vector simulation engine
│   ├── backend.{ext}       # Backend interface definition
│   ├── ibm_backend.{ext}   # IBMBackend: transpile + cloud execution
│   ├── qbraid_backend.{ext}# QBraidBackend: transpile + cloud execution
│   ├── bloch.{ext}         # Bloch sphere / qubit state introspection
│   └── mod.{ext}           # Central re-export hub (public API surface)
├── tests/
│   ├── complex.test.{ext}
│   ├── matrix.test.{ext}
│   ├── gates.test.{ext}
│   ├── parameter.test.{ext}
│   ├── circuit.test.{ext}
│   ├── simulator.test.{ext}
│   ├── transpiler.test.{ext}
│   ├── ibm_backend.test.{ext}     # Execution tests skipped by default
│   ├── qbraid_backend.test.{ext}  # Execution tests skipped by default
│   ├── bloch.test.{ext}
│   └── integration.test.{ext}     # End-to-end quantum circuits
└── {build config files}
```

### 8.2 Build Order

Follow this exact order. Each step depends on the previous ones. **Every step
below is a strict write-then-test checkpoint. For each step you MUST: (1)
implement the module/tier in full according to its specification, (2) write the
tests for that step as listed in Section 11, (3) run those tests, and (4) see
every test pass before moving to the next step.** Do not batch multiple steps,
do not defer tests, do not leave failing tests "for later", and do not proceed
with any step whose predecessors have unimplemented items or failing tests.
Incremental construction with per-step test validation is a non-negotiable part
of the specification.

```
Step 1:  types           → shared types, interfaces, enums
Step 2:  complex         → Complex number class
Step 3:  matrix          → Matrix class over ℂ
Step 4:  parameter       → Symbolic parameter & expression system
Step 5:  gates           → Gate matrix constructors (Tiers 0–14), built
                            incrementally tier-by-tier (see Step 5 sub-steps)
Step 6:  expansion       → Expansion API types and node builders
Step 7:  circuit         → QuantumCircuit builder
Step 8:  backend         → Backend interface
Step 9:  simulator       → SimulatorBackend
Step 10: transpiler      → OpenQASMTranspiler (serializer + compilation pipeline)
Step 11: ibm_backend     → IBMBackend
Step 12: qbraid_backend  → QBraidBackend
Step 13: bloch           → Bloch sphere introspection
Step 14: mod             → Public API re-exports
Step 15: integration     → Full integration test suite
Step 16: README          → Documentation
```

**Step 5 is itself incremental and MUST follow the same write-then-test
checkpoint rule for every tier.** Do not implement multiple tiers in a single
pass, do not skip ahead, and do not move from tier `k` to tier `k+1` until every
gate in tier `k` is fully implemented, all of its tests from Section 11 are
written, and all of those tests pass. Each higher tier builds on the exact
denotations (and exact test coverage) of the lower tiers, so an untested lower
tier invalidates every tier above it.

```
Step 5.0:  Tier 0   → Zero-qubit and single-qubit primitives → implement + test
Step 5.1:  Tier 1   → CX (universal entangling primitive)    → implement + test
Step 5.2:  Tier 2   → Fundamental two-qubit controlled gates → implement + test
Step 5.3:  Tier 3   → Higher two-qubit interaction gates     → implement + test
Step 5.4:  Tier 4   → Three-qubit gates                      → implement + test
Step 5.5:  Tier 5   → Four+ qubit and multi-controlled gates → implement + test
Step 5.6:  Tier 6   → N-qubit structural composite gates     → implement + test
Step 5.7:  Tier 7   → Uniformly controlled + unitary synthesis → implement + test
Step 5.8:  Tier 8   → Hamiltonian simulation + Pauli evolution → implement + test
Step 5.9:  Tier 9   → Quantum Fourier Transform              → implement + test
Step 5.10: Tier 10  → Reversible classical-logic gates       → implement + test
Step 5.11: Tier 11  → Quantum arithmetic circuits            → implement + test
Step 5.12: Tier 12  → Function loading and approximation     → implement + test
Step 5.13: Tier 13  → Comparison, aggregation, and oracles   → implement + test
Step 5.14: Tier 14  → State preparation                      → implement + test
```

At each tier sub-step: implement every gate in that tier exactly as specified in
Section 3 (no omissions, no stubs, no "minimal viable" subset), write every unit
test for those gates as required by Section 11.3 (unitarity, reference matrix,
compositional verification against the lower-tier decomposition), run the tests,
and verify that all pass. Only then proceed to the next tier. The gate module is
not considered complete, and Step 6 may not begin, until every tier from 0
through 14 has passed this checkpoint.

### 8.3 Class and Interface Catalog

#### Core Value Types

**`Complex`** — Immutable complex number `a + bi`.

- Constants: `ZERO`, `ONE`, `I`, `MINUS_I`
- Factory methods: `fromPolar(r, theta)`, `exp(theta)`
- Arithmetic: `add`, `sub`, `mul`, `div`, `scale`, `conjugate`, `neg`
- Inspection: `magnitude`, `magnitudeSquared`, `phase`, `equals`, `toString`
- All methods return new instances (immutable).

**`Matrix`** — Immutable matrix over ℂ.

- Factories: `identity(size)`, `zeros(rows, cols)`
- Operations: `get`, `multiply`, `add`, `scale`, `dagger`, `tensor`, `apply`
- Inspection: `isUnitary`, `trace`, `determinant`, `equals`, `toString`
- Dimension accessors: `rows`, `cols`
- All methods return new instances (immutable).

**`Param`** — Named symbolic parameter for circuit angles.

- Arithmetic: `Param + Param`, `Param * number`, `Param / number`, etc.
- `bind(map) -> AngleExpr` — substitute known values.
- `isResolved() -> boolean` — true if pure numeric.

#### Gate Constructors (pure functions → exact gate denotations)

Every gate listed in Section 3 (Tiers 0–14) has a pure constructor. The
implementation file `gates.{ext}` exports one constructor per gate family. Each
constructor returns the gate family's immutable exact denotation: a concrete
`Matrix` when Section 3 says the denotation is fully materialized at
construction time, or an immutable exact gate/template descriptor when Section 3
permits deferred exact preprocessing, opaque matrix-native semantics, or
ancilla-aware template expansion. The complete list (grouped by tier):

When Section 3 makes an ordered operand/register list part of a family's
semantic input, an implementation MAY expose that either directly in the
constructor signature or through a separate immutable application/descriptor
layer. In either case, the same ordered operand list must remain losslessly
recoverable wherever Section 3 says it affects validation, equality, or
serialization.

**Tier 0 (1×1 / 2×2):** `GlobalPhaseGate`, `IGate`, `HGate`, `XGate`, `YGate`,
`ZGate`, `PhaseGate`, `RGate`, `RXGate`, `RYGate`, `RZGate`, `SGate`, `SdgGate`,
`SXGate`, `SXdgGate`, `TGate`, `TdgGate`, `UGate`, `RVGate`

**Tier 1 (4×4):** `CXGate`

**Tier 2:** `CZGate`, `CYGate`, `CPhaseGate`, `CRZGate`, `CRYGate`, `CRXGate`,
`CSGate`, `CSdgGate`, `CSXGate`, `CHGate`, `CUGate`, `DCXGate`

**Tier 3:** `SwapGate`, `RZZGate`, `RXXGate`, `RYYGate`, `RZXGate`, `ECRGate`,
`iSwapGate`, `XXPlusYYGate`, `XXMinusYYGate`

**Tier 4:** `CCXGate`, `CCZGate`, `CSwapGate`, `RCCXGate`

**Tier 5:** `C3XGate`, `C3SXGate`, `C4XGate`, `RC3XGate`, `MCXGate`,
`MCPhaseGate`

**Tier 6:** `MSGate`, `PauliGate`, `DiagonalGate`, `PermutationGate`,
`MCMTGate`, `PauliProductRotationGate`

**Tier 7:** `UCRZGate`, `UCRYGate`, `UCRXGate`, `UCPauliRotGate`, `UCGate`,
`UnitaryGate`, `LinearFunction`, `Isometry`

**Tier 8:** `PauliEvolutionGate`, `HamiltonianGate`

**Tier 9:** `QFTGate`

**Tier 10:** `AndGate`, `OrGate`, `BitwiseXorGate`, `InnerProductGate`

**Tier 11:** `HalfAdderGate`, `FullAdderGate`, `ModularAdderGate`,
`MultiplierGate`

**Tier 12:** `LinearPauliRotationsGate`, `PolynomialPauliRotationsGate`,
`PiecewiseLinearPauliRotationsGate`, `PiecewisePolynomialPauliRotationsGate`,
`PiecewiseChebyshevGate`, `LinearAmplitudeFunctionGate`, `ExactReciprocalGate`

**Tier 13:** `IntegerComparatorGate`, `QuadraticFormGate`, `WeightedSumGate`,
`PhaseOracleGate`, `BitFlipOracleGate`

**Tier 14:** `GraphStateGate`

#### QuantumCircuit

The central builder class. Constructor: `QuantumCircuit(globalPhase = 0)`.

Qubits are allocated implicitly. Classical memory is modeled as an ordered list
of named `ClassicalRegister` definitions. All methods return `this` for chaining
(except inspection methods).

`QuantumCircuit` is also the carrier type for ordinary nested bodies, including
control-flow branches, boxed regions, gate bodies, and subroutine bodies. Unless
an API below explicitly promises source-preserving / exact-round-trip behavior,
a manually built `QuantumCircuit` is a normalized semantic representation. A
source-preserving parser/deserializer may return the same class with additional
syntax-local metadata attached to instructions, gate calls, and annotations.

**Gate methods** — one method per gate from Section 3:

```
// Tier 0 (zero-qubit / single-qubit)
qc.globalPhaseGate(theta)
qc.id(qubit)
qc.h(qubit)
qc.x(qubit)
qc.y(qubit)
qc.z(qubit)
qc.p(lambda, qubit)
qc.r(theta, phi, qubit)
qc.rx(theta, qubit)
qc.ry(theta, qubit)
qc.rz(theta, qubit)
qc.s(qubit)
qc.sdg(qubit)
qc.sx(qubit)
qc.sxdg(qubit)
qc.t(qubit)
qc.tdg(qubit)
qc.u(theta, phi, lambda, qubit)
qc.rv(vx, vy, vz, qubit)

// Tier 1
qc.cx(control, target)

// Tier 2
qc.cz(control, target)
qc.cy(control, target)
qc.cp(lambda, control, target)
qc.crz(theta, control, target)
qc.cry(theta, control, target)
qc.crx(theta, control, target)
qc.cs(control, target)
qc.csdg(control, target)
qc.csx(control, target)
qc.ch(control, target)
qc.cu(theta, phi, lambda, gamma, control, target)
qc.dcx(qubit0, qubit1)

// Tier 3
qc.swap(qubit0, qubit1)
qc.rzz(theta, qubit0, qubit1)
qc.rxx(theta, qubit0, qubit1)
qc.ryy(theta, qubit0, qubit1)
qc.rzx(theta, qubit0, qubit1)
qc.ecr(qubit0, qubit1)
qc.iswap(qubit0, qubit1)
qc.xxPlusYY(theta, beta, qubit0, qubit1)
qc.xxMinusYY(theta, beta, qubit0, qubit1)

// Tier 4
qc.ccx(control1, control2, target)
qc.ccz(control1, control2, target)
qc.cswap(control, target1, target2)
qc.rccx(control1, control2, target)

// Tier 5
qc.c3x(control1, control2, control3, target)
qc.c3sx(control1, control2, control3, target)
qc.c4x(control1, control2, control3, control4, target)
qc.rc3x(control1, control2, control3, target)
qc.mcx(controlQubits, target)
qc.mcp(lambda, controlQubits, target)

// Tier 6
qc.ms(theta, qubits)
qc.pauli(pauliString, qubits)
qc.diagonal(phases, qubits)
qc.permutation(sigma, qubits)
qc.mcmt(gate, controlQubits, targetQubits)
qc.pauliProductRotation(theta, paulis, qubits)

// Tier 7
qc.ucrz(angles, controlQubits, target)
qc.ucry(angles, controlQubits, target)
qc.ucrx(angles, controlQubits, target)
qc.ucPauliRot(angles, axis, controlQubits, target)
qc.uc(gates, controlQubits, target)
qc.unitary(matrix, qubits)
qc.linearFunction(matrix, qubits)
qc.isometry(matrix, qubits, ancillas)

// Tier 8
qc.pauliEvolution(operator, time, qubits, trotterOrder?, trotterReps?)
qc.hamiltonianGate(matrix, time, qubits)

// Tier 9
qc.qft(qubits, doSwaps?, approximationDegree?)

// Tier 10
qc.andGate(inputs, output, ancilla?)
qc.orGate(inputs, output, ancilla?)
qc.bitwiseXor(a, b)
qc.innerProduct(a, b, output)

// Tier 11
qc.halfAdder(a, b, sum, carry)
qc.fullAdder(a, b, carryIn, sum, carryOut)
qc.modularAdder(a, b, n?)
qc.multiplier(a, b, result)

// Tier 12
qc.linearPauliRotations(slope, offset, numStateQubits, numTargetQubits, basis?)
qc.polynomialPauliRotations(coefficients, numStateQubits, numTargetQubits, basis?)
qc.piecewiseLinearPauliRotations(breakpoints, slopes, offsets, numStateQubits, numTargetQubits, basis?)
qc.piecewisePolynomialPauliRotations(breakpoints, coefficients, numStateQubits, numTargetQubits, basis?)
qc.piecewiseChebyshev(f, degree, breakpoints, numStateQubits, numTargetQubits)
qc.linearAmplitudeFunction(slope, offset, domain, image, numStateQubits, numTargetQubits)
qc.exactReciprocal(scalingFactor, numStateQubits, numTargetQubits)

// Tier 13
qc.integerComparator(value, numStateQubits, output, geq?)
qc.quadraticForm(linear, quadratic, offset, numResultQubits)
qc.weightedSum(weights, numStateQubits, numSumQubits)
qc.phaseOracle(expression, qubits)
qc.bitFlipOracle(expression, qubits, output)

// Tier 14
qc.graphState(adjacencyMatrix, qubits)
```

**Expansion API methods** — one method per feature from Section 5:

```
// Program metadata (§5.2)
qc.setProgramVersion(major, minor?)
qc.omitProgramVersion()
qc.include(path)
qc.setCalibrationGrammar(name)
qc.lineComment(content)
qc.blockComment(content)

// Classical declarations (§5.6)
qc.addClassicalRegister(name, size)
qc.declareClassicalVar(name, type, size?, initValue?)
qc.declareConst(name, type, size?, value)
qc.declareInput(name, type, size?, defaultValue?)
qc.declareOutput(name, type, size?, initializer?)
qc.declareArray(name, baseType, dimensions, initValue?)
qc.declareLegacyQReg(name, size)
qc.declareLegacyCReg(name, size)

// Aliases and declarations (§5.4, §5.6)
qc.alias(name, target)

// Classical statements (§5.10)
qc.classicalAssign(target, expression)
qc.classicalAssignOp(target, operator, expression)
qc.exprStatement(expression)
qc.returnValue(value)
qc.returnVoid()

// Non-unitary operations (§5.9)
qc.measure(qubit, clbit?, syntax?)
qc.measureRegister(qubits, clbits, syntax?)
qc.reset(target)
qc.barrier(...qubits)
qc.delay(durationExpr, qubits?)
qc.timed(operation, durationExpr, qubits?)

// Gate modifiers (§5.7)
qc.ctrl(numControls, gate, controlQubits, targetQubits)
qc.negctrl(numControls, gate, controlQubits, targetQubits)
qc.inv(gate, qubits)
qc.pow(k, gate, qubits)

// Generic gate invocation / exact round-trip support (§5.7)
qc.applyGate(name, operands, parameters?, modifiers?, localPhase?, surfaceName?)

// Custom gate definitions (§5.8)
qc.defineGate(name, params, qubits, body)

// State preparation (§5.19)
qc.prepareState(state, qubits?)
qc.initialize(state, qubits?)

// Control flow (§5.10)
qc.ifTest(condition, trueBody, falseBody?)
qc.forLoop(iterable, loopParam, body)
qc.whileLoop(condition, body)
qc.switch(target, cases, defaultBody?)
qc.breakLoop()
qc.continueLoop()
qc.end()
qc.box(body, duration?)

// Subroutines and externs (§5.11, §5.12)
qc.defineSubroutine(name, params, returnType, body)
qc.declareExtern(name, params, returnType)
qc.callSubroutine(name, args, resultVar?)

// Calibration (§5.15, §5.16)
qc.cal(body)
qc.defineCalibration(target, params, operands, returnType, body)
qc.declareCalibrationExtern(name, params, returnType)

// Pragmas and annotations (§5.13)
qc.pragma(content)
qc.annotate(content)

// Composition
qc.compose(other, qubitMapping, clbitMapping)
qc.toGate(label?)
qc.toInstruction(label?)
qc.append(operation, qubitIndices?, clbitIndices?, params?, options?)
qc.inverse()

// Inspection
qc.complexity()
qc.blochSphere(qubitIndex)

// Parameter binding
qc.run(parameters)

// Transpilation delegation
qc.transpile(backend, shots?)
```

**Expression factories** (standalone module, per Design Principle 6 in §5.1):

```
// Literals
Expr.int(value, base?)               // IntegerLiteral
Expr.float(value)                     // FloatLiteral
Expr.imaginary(value)                 // ImaginaryLiteral
Expr.bool(value)                      // BooleanLiteral
Expr.bitstring(value)                 // BitstringLiteral
Expr.duration(value, unit)            // DurationLiteral
Expr.constant(name)                   // BuiltInConstant ("pi", "tau", "euler", "im")

// References
Expr.ref(name)                        // IdentifierExpr
Expr.physicalQubit(index)             // PhysicalQubitExpr

// Compound literals
Expr.array(elements[])                // ArrayLiteral
Expr.set(elements[])                  // SetLiteral
Expr.range(start?, step?, end?)       // RangeExpr

// Operators
Expr.unary(op, operand)               // UnaryExpr
Expr.binary(op, left, right)          // BinaryExpr
Expr.concat(parts[])                  // ConcatenationExpr

// Type operations
Expr.cast(targetType, value)          // CastExpr
Expr.sizeOf(target, dimension?)       // SizeOfExpr
Expr.realPart(operand)                // RealPartExpr
Expr.imagPart(operand)                // ImagPartExpr

// Invocations
Expr.call(callee, args[])             // CallExpr

// Indexing
Expr.index(base, selectors[])         // IndexExpr

// Quantum-specific
Expr.measure(source)                  // MeasureExpr
Expr.durationOf(body)                 // DurationOfExpr

// Grouping
Expr.paren(inner)                     // ParenthesizedExpr
```

`qc.globalPhaseGate(theta)` denotes the zero-qubit gate form. Implementations
may normalize hoistable unconditional uses into the owning scope's scalar
`globalPhase` only when the **fold-safe same-scope rule** permits that rewrite
in a representation using the ordinary-scope scalar semantics of Phase
Convention 3. `qc.inverse()` is defined only for unitary-only circuits/scopes
and must reject non-unitary contents rather than invent inverse semantics.
`qc.applyGate(...)` is the required generic gate-call constructor for parsed or
programmatic instructions that carry `localPhase`, `surfaceName`, or an exact
modifier stack. Convenience methods such as `qc.u(...)`, `qc.p(...)`,
`qc.cp(...)`, and `qc.cu(...)` construct normalized semantic gate calls under
the SDK's internal phase convention; a source-preserving parser/deserializer
MUST use `qc.applyGate(...)` or another exactly equivalent lower-level mechanism
whenever those conveniences would lose exact textual identity or
gate-expression-local phase.

`qc.classicalAssignOp(target, operator, expression)` appends a compound
assignment statement. The `operator` parameter MUST be one of the 12 compound
operators
(`"+=", "-=", "*=", "/=", "&=", "|=", "~=", "^=", "<<=", ">>=", "%=",
"**="`).
For simple assignment use `qc.classicalAssign(target, expression)`.

`qc.transpile(backend, shots?)` is pure delegation to
`backend.transpileAndPackage(this, shots)`. It does not add transpilation logic
to `QuantumCircuit` itself and exists solely as a convenience entry point.

`qc.prepareState(state, qubits?)` and `qc.initialize(state, qubits?)` accept a
`StateSpec` value as defined in §5.19. The `state` parameter may be an amplitude
array (interpreted as `AmplitudeVector`), an integer (interpreted as
`BasisState`), or a bitstring (interpreted as `BitstringState`).

**Physical-qubit operands.** After transpilation (SABRE layout/routing),
circuits operate on physical qubits. Gate methods that accept integer qubit
arguments always interpret them as virtual qubit indices. To construct
instructions on physical qubits (e.g. `$0`, `$1`), use `qc.applyGate(...)` with
`PhysicalQubitRef` operands from the expression factory, or use the generic
`qc.append(...)` method. The transpilation pipeline produces circuits using
physical-qubit operands when targeting hardware backends.

**Expression factories.** The `Expr` module provides standalone constructors for
all expression nodes defined in §5.5. These are pure functions that return
`Expression` objects without modifying any circuit. Builder methods that accept
expression arguments (e.g. `qc.ifTest(condition, ...)`,
`qc.classicalAssign(...)`) accept factory-produced nodes. The builder MAY
additionally accept raw literals (integers, strings, booleans) and implicitly
wrap them as the corresponding `Expression` nodes for ergonomic use.

#### Transpiler Interface and OpenQASMTranspiler

**`Transpiler` interface:**

```
interface Transpiler {
  serialize(circuit: QuantumCircuit) -> string   // accepts normalized semantic or source-preserving circuits; may reject when no exact textual lowering exists
  deserialize(source: string) -> QuantumCircuit  // source-preserving / exact-round-trip circuit
}
```

`deserialize` is a source-preserving / exact-round-trip API. The returned
`QuantumCircuit` MUST preserve explicit zero-qubit phase instructions,
`GateCall.localPhase`, exact operand/reference forms, exact modifier stacks,
exact parameter/localPhase expression trees, and enough syntax-local metadata
(for example `surfaceName` or an exactly equivalent mechanism) to re-emit the
same documented gate-family spelling and instruction-local distinctions when
`serialize` is called without intervening normalizing transforms. `serialize`
MUST honor that metadata when present.

When `serialize` is given a normalized semantic circuit that still contains a
nonzero `GateCall.localPhase`, it MUST either lower that correction exactly as
required by this Phase Convention or reject serialization. Exact lowering means
using only the mechanisms permitted by this Phase Convention: a textual gate
family whose semantics embed part or all of the same correction inside the same
gate expression and modifier stack; for a fold-safe
`bare, unmodified gate
instruction` only, moving any remaining correction that
must leave the gate expression into the immediate owning scope's
scalar/global-phase representation and then emitting that same-scope phase as
`gphase(...)` or an exactly equivalent encoding; or, when the active
serializer/API contract permits it, an exact helper-gate or equivalent
target-language helper encoding that keeps the needed phase inside the helper
operand being emitted. If any residual instruction-local phase would remain in
the emitted call, serialization MUST reject. `serialize` MUST NOT split a
modifier-internal `localPhase` into a sibling `gphase(...)` unless a specific
exact rule for that case has been proved.

**`OpenQASMTranspiler`** implements `Transpiler` for OpenQASM 3.1. It handles
the complete language surface defined in Section 4. Its serializer MUST always
emit `include "stdgates.inc";` in generated OpenQASM 3 output. Additionally, it
exposes the full compilation pipeline as public functions:

```
// Serialization
serialize(circuit) -> string
deserialize(source) -> QuantumCircuit

// Compilation pipeline stages
transpile(circuit, target) -> QuantumCircuit
unrollComposites(circuit) -> QuantumCircuit
expandGateModifiers(circuit) -> QuantumCircuit
inlineGateDefinitions(circuit) -> QuantumCircuit
inlineSubroutines(circuit) -> QuantumCircuit
synthesizeHighLevel(circuit) -> QuantumCircuit
layoutSABRE(circuit, couplingMap) -> { circuit, layout }
routeSABRE(circuit, couplingMap, layout) -> QuantumCircuit
translateToBasis(circuit, basisGates) -> QuantumCircuit
optimize(circuit) -> QuantumCircuit

// Decomposition utilities
decomposeZYZ(matrix) -> { alpha, beta, gamma, delta }  // canonical representative from Section 3 Tier 7
decomposeToRzSx(matrix) -> { globalPhase, instructions }
decomposeKAK(matrix) -> { globalPhase, instructions }
```

Unless a method above is explicitly documented otherwise, the compilation-
pipeline functions return normalized semantic `QuantumCircuit` instances. They
may canonicalize surface spellings, inline definitions, restructure ordinary-
scope boundaries, and fold only those zero-qubit phases that Section 2 proves
fold-safe with respect to the transformed IR's current ownership relation; they
are not source-preserving / round-trip APIs.

#### Backend Interface

```
interface Backend {
  numQubits: number
  basisGates: string[]
  couplingMap: [number, number][] | null

  transpileAndPackage(circuit: QuantumCircuit, shots?: number) -> Executable
  execute(executable: Executable, shots?: number) -> ExecutionResult
}
```

Where `ExecutionResult = map<string, number>` (bitstring → percentage 0–100).
The `shots` parameter on `execute` defaults to 1024. It is a per-execution
input, not a backend capability field.

**`SimulatorBackend`** — noiseless state-vector simulator. Supports all gates.
No connectivity constraints (`couplingMap = null`). Additional method:
`getStateVector(circuit, params?) -> Complex[]`.

**`IBMBackend`** — IBM Quantum cloud execution. See Section 6.

**`QBraidBackend`** — qBraid cloud execution. See Section 7.

---

## 9. Simulator Engine

**File:** `src/simulator.{ext}`

### Constructor

```
SimulatorBackend(numShots = 1024)
```

### Properties

- `numQubits`: large (e.g. 30), limited by memory.
- `basisGates`: all gates supported.
- `couplingMap`: `null` (all-to-all).

### `transpileAndPackage(circuit, shots?) -> SimulatorExecutable`

Since the simulator supports all gates and has no connectivity constraints, this
simply validates the circuit and wraps it:

```
SimulatorExecutable {
  circuit: QuantumCircuit
  numShots: shots ?? this.numShots
}
```

### `execute(executable, shots?) -> ExecutionResult`

**State-vector simulation algorithm:**

1. Initialize state vector to `|0...0⟩ = [1, 0, 0, ..., 0]` (length
   `2^numQubits`).
2. Initialize classical memory to all zeros using the circuit's ordered named
   classical registers.
3. Record/defer the circuit's accumulated global phase.
4. For each instruction in order: a. If control flow (`if_test`, `for_loop`,
   `while_loop`, `switch`), handle accordingly. b. Otherwise apply the
   gate/operation.
5. After all instructions, apply deferred global phase if an exact state vector
   or exact comparison is requested.
6. Sample measurement outcomes.

**Gate Application — Subspace Iteration (MANDATORY):**

**Do NOT construct full 2^n × 2^n matrices.** Apply gates directly to the state
vector:

**Single-qubit gate U on qubit k:**

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

**Two-qubit gate U (4×4) on qubits k0, k1:** Group entries by bits k0, k1, apply
4×4 matrix to each group of 4.

**N-qubit gate (general):** Group into blocks of `2^m` entries differing only in
the m target qubit bits. Apply `2^m × 2^m` matrix to each block.

**Measurement (Born Rule):**

```
P(qubit k = 0) = sum of |state[j]|^2 for all j where bit k is 0
outcome = random() < P(0) ? 0 : 1
Collapse: zero out inconsistent amplitudes, renormalize.
Store outcome in classical bit.
```

**Reset:** Measure internally, if result is 1 apply X to flip to |0⟩.

**Control Flow:**

- `if_test`: evaluate classical condition, simulate matching body.
- `for_loop`: iterate, simulate body per iteration. Handle break/continue.
- `while_loop`: evaluate condition, simulate body, repeat. Handle
  break/continue. Max iterations: 10000.
- `switch`: evaluate target, find matching case, simulate body.
- `box`: execute body unconditionally.

**Barrier and Delay:** No effect on state vector. Skip.

**Sampling:**

- End-only measurements: simulate once, compute probabilities, sample `numShots`
  times.
- Mid-circuit measurements: run full simulation `numShots` times independently.

**Return:** `map<string, number>` — bitstrings to percentages (0–100).
Bitstrings concatenate classical registers in declared order, local bit 0
rightmost.

### `getStateVector(circuit, params?) -> Complex[]`

Return the raw state vector for a measurement-free circuit (or one with only
terminal measurements that can be discarded). Reject circuits with mid-circuit
measurement, reset, or classically controlled quantum evolution dependent on
sampled outcomes.

---

## 10. Transpilation Pipeline

**File:** `src/transpiler.{ext}`

The `OpenQASMTranspiler` handles both serialization (to/from OpenQASM 3.1 text)
and a full compilation pipeline for hardware-constrained backends.

### Stage 0: Initialization (Unrolling)

- Unroll composite gates (`toGate`, `toInstruction`) into primitives.
- Unroll control-flow block bodies recursively.
- Expand gate modifiers (`ctrl @`, `negctrl @`, `inv @`, `pow(k) @`) using the
  exact phase rules from Section 2; unsupported non-integer powers must remain
  symbolic until later exact handling or be rejected explicitly.
- Expand custom gate definitions (inline body with parameter substitution).
- Inline subroutine calls (up to configurable depth limit, default 100).
- Evaluate const expressions.
- Resolve input variables (if bound).
- Strip pragmas and annotations not relevant to target backend.
- Expand gate broadcasting. Each expanded lane MUST retain its own `localPhase`
  and modifiers; implementations MUST NOT first merge per-lane local phase into
  one broadcast-level phase correction. Any canonical lane order used during
  expansion is representational only.

### Stage 1: High-Level Synthesis

Replace abstract operations with concrete gate sequences:

- `SWAP → 3 CX`.
- Multi-controlled gates → sequences of 1- and 2-qubit gates.
- 3+ qubit gates → 1- and 2-qubit gates.

### Stage 2: Layout (SABRE)

Choose a mapping from virtual qubits to physical qubits:

1. Build interaction graph.
2. Score layouts by shortest-path distance on coupling map.
3. SABRE bidirectional heuristic with look-ahead.

### Stage 3: Routing (SABRE)

Insert SWAP gates so all 2-qubit gates operate on adjacent physical qubits:

1. Process front layer of ready gates.
2. Execute gates whose qubits are adjacent.
3. If none executable, insert best SWAP and update layout.

### Stage 4: Translation (Basis Gate Decomposition)

Convert every gate to the backend's basis gates:

- **ZYZ decomposition:** use the canonical Section 3 Tier 7 representative
  `M = exp(i*alpha) * Rz(beta) * Ry(gamma) * Rz(delta)`
- **RZ+SX decomposition:** `M = exp(i*eta) * Rz(a) * SX * Rz(b) * SX * Rz(c)`
- **KAK/Weyl decomposition:** Any 2-qubit unitary → at most 3 CX + single-qubit
  gates + global phase.
- **ECR backend translation:** `CX(a,b) = (S(b)*SXdg(b)) * ECR(a,b) * X(a)`
- **Gate direction reversal:** `CX(b,a) = H(a)*H(b)*CX(a,b)*H(a)*H(b)`

### Stage 5: Optimization

Iterate peephole optimizations until convergence: a) Single-qubit gate merging
(multiply 2×2 matrices, then re-decompose under the exact phase convention;
parameter normalization modulo `2*pi` is forbidden unless exact operator
equality is proved). b) Two-qubit gate cancellation (`CX*CX → I`). c)
Commutation analysis to expose cancellations. d) Identity gate removal only when
the exact operator is `I` on the same arity (`Rz(0)`, `Rx(0)`, etc.).

---

## 11. Test Plan

The test suite must contain **at minimum 900 tests** organized as follows.

### 11.1 Unit Tests per Module

| Module         | Min Tests | Notes                                                           |
| -------------- | --------- | --------------------------------------------------------------- |
| Types          | 35        | All shared types, edge cases                                    |
| Complex        | 40        | Arithmetic, constants, polar, phase                             |
| Matrix         | 45        | Multiply, tensor, dagger, unitary, determinant                  |
| Gates          | 160       | Every gate: unitarity, reference matrix, composition            |
| Parameter      | 15        | Bind, partial bind, arithmetic, isResolved                      |
| Circuit        | 100       | Every gate method, every expansion API method                   |
| Backend        | 5         | Interface contract, mock backend                                |
| Simulator      | 70        | State vector, measurement, control flow, all gates              |
| Transpiler     | 120       | Serialize/deserialize (full OpenQASM 3.1), compilation pipeline |
| IBM Backend    | 20        | Transpile tests always; execution tests skipped                 |
| qBraid Backend | 20        | Transpile tests always; execution tests skipped                 |
| Bloch          | 20        | Standard states, entangled states, mixed states                 |
| **Subtotal**   | **650**   |                                                                 |

### 11.2 Integration Tests (minimum 250)

Build and simulate complete quantum circuits, run with 1024 shots, verify output
distribution matches expectations.

**Required integration circuits (implement ALL):**

1. Identity circuit: N qubits, no gates → `{ "000...0": 100 }`.
2. Single X gate: `X(0)` → `{ "1": 100 }`.
3. Double X gate: `X(0), X(0)` → `{ "0": 100 }`.
4. Hadamard: `H(0)` → ~`{ "0": 50, "1": 50 }`.
5. Bell state: `H(0), CX(0,1)` → ~`{ "00": 50, "11": 50 }`.
6. Reverse Bell: `H(0), CX(0,1), CX(0,1), H(0)` → `{ "00": 100 }`.
7. GHZ-3: `H(0), CX(0,1), CX(0,2)` → ~`{ "000": 50, "111": 50 }`.
8. GHZ-4: extend to 4 qubits.
9. Superposition on all N qubits: uniform distribution.
10. SWAP test: prepare |01⟩, SWAP → |10⟩.
11. Double SWAP: SWAP twice → back to original.
12. iSWAP test: prepare |01⟩, iSWAP → verify i|10⟩.
13. Toffoli truth table: all 8 input combinations.
14. CCZ truth table: only |111⟩ picks up −1 phase.
15. Fredkin (CSWAP) truth table: all 8 inputs.
16. Phase kickback: `X(1), H(0), CX(0,1), H(0)`.
17. Quantum teleportation: full protocol.
18. Deutsch-Jozsa (2-qubit): constant → "0", balanced → "1".
19. Bernstein-Vazirani (3-qubit): encode secret, verify recovery. 20–40. Every
    single-qubit gate on |0⟩ and on |1⟩ (21 gates × 2 = 42 tests, minus overlaps
    ≈ 21 additional).
20. `S * S = Z` via state vector.
21. `T * T = S` via state vector.
22. `H * Z * H = X` via state vector.
23. `H * X * H = Z` via state vector.
24. `RX(pi)|0⟩ = [0, -i]` exact.
25. `RY(pi)|0⟩ = |1⟩` exact.
26. `RZ(pi) = (-i) * Z` exact.
27. Internal canonical `U(pi, 0, pi)` = X via state vector.
28. Internal canonical `U(pi/2, 0, pi)` = H.
29. Internal canonical `U(pi, pi/2, pi/2)` = Y.
30. Parameterized RX sweep: theta ∈ [0, 2π], verify prob(1) = sin²(θ/2).
31. Parameterized RY sweep.
32. Classical conditioning via `if_test`.
33. `for_loop`: repeated RX rotations.
34. `while_loop`: repeat until |1⟩.
35. `switch` statement.
36. Reset mid-circuit. 58–72. Every two-qubit gate on various inputs (CH, CY,
    CZ, DCX, ECR, CP, CRX, CRY, CRZ, CS, CSdg, CSX, CU, RXX, RYY, RZZ, RZX,
    XXPlusYY, XXMinusYY). 73–76. Three-qubit gates (CCX, CCZ, CSWAP, RCCX) on
    inputs. 77–82. Four-qubit gates (C3X, C3SX, C4X, RC3X) on inputs. 83–86.
    Multi-controlled gates with variable controls (MCX, MCP with 1, 2, 3, 4
    controls).
37. MS gate on 2-qubit system.
38. MS gate on 3-qubit system.
39. Pauli string "XYZ" on 3-qubit input.
40. Pauli string "II" on 2 qubits.
41. DiagonalGate: known phases, verify state.
42. PermutationGate: known permutation, verify state.
43. MCMTGate: multi-controlled multi-target, verify.
44. PauliProductRotationGate: known rotation, verify.
45. UCRZGate: uniformly controlled RZ, verify.
46. UCRYGate: verify.
47. UCGate: general uniformly controlled, verify.
48. UnitaryGate: custom 2×2 rotation, verify.
49. UnitaryGate: custom 4×4 (CX equivalent), verify.
50. LinearFunction: known GF(2) matrix, verify.
51. Isometry: 1→2 qubit isometry, verify.
52. PauliEvolutionGate: single Pauli term, verify.
53. PauliEvolutionGate: multi-term Hamiltonian with Trotter, verify.
54. HamiltonianGate: known Hermitian, verify.
55. QFTGate: 3-qubit canonical no-SWAP QFT, verify amplitudes against
    `QFT[j,k] = exp(2*pi*i * brev_n(j) * k / 2^n) / sqrt(2^n)`.
56. QFTGate: inverse QFT, verify exact inverse and `QFT† → QFT = I`.
57. AndGate: all 4 input combinations.
58. OrGate: all 4 input combinations.
59. BitwiseXorGate: verify.
60. InnerProductGate: verify.
61. HalfAdderGate: 0+0, 0+1, 1+0, 1+1.
62. FullAdderGate: all 8 inputs.
63. ModularAdderGate: known addition, verify.
64. MultiplierGate: known multiplication, verify.
65. LinearPauliRotationsGate: verify rotation angle.
66. PolynomialPauliRotationsGate: verify.
67. PiecewiseLinearPauliRotationsGate: verify.
68. PiecewiseChebyshevGate: approximate known function, verify deterministic
    coefficient generation on a fixed input representation.
69. LinearAmplitudeFunctionGate: verify loaded amplitudes.
70. ExactReciprocalGate: verify reciprocal rotation.
71. IntegerComparatorGate: verify comparisons.
72. QuadraticFormGate: verify.
73. WeightedSumGate: verify.
74. PhaseOracleGate: verify phase flip on target state.
75. BitFlipOracleGate: verify bit flip on target state.
76. GraphStateGate: 3-qubit graph state, verify.
77. `prepareState` with amplitudes: [1/√2, 1/√2] → ~50/50.
78. `prepareState` with integer: |3⟩ in 2-qubit system.
79. `prepareState` with bitstring: "10", verify.
80. `initialize`: reset then prepare from non-zero initial state.
81. `SX * SX = X` via circuit.
82. `SX * SXdg = I` via circuit.
83. `compose`: compose Bell state into larger circuit.
84. `toGate` + `append`: convert sub-circuit to gate, append.
85. `inverse`: create circuit, invert, compose → identity.
86. RV gate: specific axis/angle.
87. Global phase gate: verify exp(i*θ) applied to state.
88. `barrier` and `delay`: no state change.
89. `complexity()`: correct depth and gate counts.
90. `blochSphere()`: verify for |0⟩, |+⟩, Bell state qubit.
91. Round-trip serialization: build → serialize → deserialize → simulate →
    compare.
92. OpenQASM 3 with control flow: serialize/deserialize with `if_test`,
    simulate.
93. Transpile Bell state for 5-qubit CX backend: verify basis gates and
    distribution.
94. Transpile GHZ-3 for linear coupling map: verify routing SWAPs.
95. Transpile + simulate round-trip: arbitrary 3-qubit circuit.
96. Transpile for ECR backend.
97. Transpile with optimization: `CX*CX` cancellation.
98. KAK decomposition round-trip: 2-qubit circuit.
99. ZYZ decomposition round-trip: single-qubit circuit.
100. IBM payload structure: correct JSON, Sampler V2 PUB tuple.
101. IBM OpenQASM 3 in payload: valid serialization.
102. qBraid payload structure: correct JSON.
103. qBraid OpenQASM 3 in payload: valid serialization.
104. Gate modifier `ctrl @` end-to-end: simulate.
105. Gate modifier `inv @` end-to-end: simulate.
106. Gate modifier `pow(2) @ t` end-to-end: simulate.
107. Chained modifiers end-to-end: `ctrl @ inv @ s` → simulate.
108. Custom gate definition end-to-end: define, use, serialize, deserialize,
     simulate.
109. OpenQASM 3 classical types round-trip.
110. OpenQASM 3 gate modifiers round-trip.
111. OpenQASM 3 subroutine round-trip.
112. OpenQASM 3 classical expressions round-trip.
113. OpenQASM 3 pragma and annotation round-trip.
114. OpenQASM 3 calibration round-trip.
115. Transpile preserves measurement semantics with multi-register circuit.
     166–185. 20 transpilation verification circuits: build → transpile for
     constrained backend → simulate both → verify equivalent distributions.
     186–199. 14 small random circuits (2–4 qubits, 3–8 gates): compute expected
     output, verify against simulation. 200–250. Additional circuits exercising
     higher-tier gates in composed scenarios (QFT + inverse, arithmetic +
     comparison, state prep + oracle, etc.).

### 11.3 Detailed Unit Test Requirements per Module

#### Types (min 35)

- `ClassicalRegister` creation, flat offsets, declaration order.
- `ClassicalBitRef` creation and flat-index mapping.
- `Instruction` creation with flat `clbits` and named `clbitRefs`.
- `Instruction` with `modifiers` field (ctrl, negctrl, inv, pow).
- `Instruction` with `annotations` field.
- `Condition` with single bit, multi-bit, named register.
- `CircuitComplexity` defaults and field types.
- `BlochCoordinates` field ranges.
- `BackendConfiguration` with and without coupling map.
- Optional `corsProxy` defaults and modes.
- `IBMBackendConfiguration`: `serviceCrn`, `apiVersion`, exactly one auth mode.
- `ClassicalVariable` for every `ClassicalType`.
- `GateDefinition` with params, qubit args, and phase-owning `QuantumCircuit`
  body.
- `GateModifier` for each type.
- `SubroutineDefinition` with phase-owning `QuantumCircuit` body,
  `ExternDeclaration`, `ArrayType`.
- `ClassicalExpr` variants: literals, unary/binary, casts, function calls.
- `DurationExpr` variants.
- `ProgramVersion`, `IncludeDirective`, `AliasDefinition`.
- `Instruction.duration`, `MeasurementSyntax`, `SwitchCase`.
- `CalibrationScope`, `CalibrationDefinition`, `PortDeclaration`,
  `FrameDeclaration`, `WaveformDefinition`, `CalibrationInstruction`.
- `Pragma`, `Annotation`.
- Scoped control-flow with nested `QuantumCircuit` bodies.
- Shot count as per-execution input.
- CORS proxy URL rewriting.
- `numClbits` equals sum of register sizes.
- Edge cases: 0 qubits, 0 clbits, empty instructions.
- `ExecutionResult` percentages sum to 100.

#### Complex (min 40)

- Arithmetic: add, sub, mul, div with positive, negative, zero, pure-real,
  pure-imaginary operands.
- Conjugate of real, imaginary, general complex.
- Magnitude and magnitudeSquared for known values.
- Phase for all four quadrants.
- Phase on negative real axis: `(-1+0i).phase() = pi`.
- `Complex.exp` for 0, π/2, π, 3π/2, 2π.
- `Complex.fromPolar` for known values.
- Constants: ZERO, ONE, I, MINUS_I.
- Division edge cases.
- `z.mul(z.conjugate()).im ≈ 0`.
- `z.magnitude() ≈ sqrt(z.magnitudeSquared())`.
- Immutability: operations don't mutate original.
- `neg()`: `z.add(z.neg()).equals(Complex.ZERO)`.
- `equals` with various epsilon.

#### Matrix (min 45)

- Identity: `I*A = A`, `A*I = A`.
- Associativity: `(A*B)*C = A*(B*C)`.
- `A.dagger().dagger().equals(A)`.
- Tensor product dimensions.
- `I₂ ⊗ I₂ = I₄`.
- Pauli identities: `X² = I`, `Y² = I`, `Z² = I`, `XY = iZ`.
- Unitarity: `X.isUnitary()`, `H.isUnitary()`.
- Non-unitary returns `isUnitary() = false`.
- `apply` on |0⟩ and |1⟩ with X, H, Z.
- Addition commutativity.
- Scale by ZERO → zero matrix.
- Scale by ONE → same matrix.
- Trace of identity = n.
- Trace of Pauli matrices = 0.
- Determinant of identity = 1, det(X) = −1.
- `equals` symmetry.

#### Gates (min 160)

Every gate function must be:

- Called and verified unitary.
- Checked against reference matrix elements.
- Verified compositionally (multi-qubit gates match decomposition).

Additional specific tests:

- Phase hierarchy: `T ≈ P(π/4)`, `S ≈ P(π/2)`, `Z ≈ P(π)`.
- Internal canonical `U_can(π, 0, π) ≈ X`, internal canonical
  `U_can(π/2, 0, π) ≈ H`.
- `RX(π) = (−i)*X` exactly.
- `SX*SX ≈ X`, `SX*SXdg ≈ I`.
- `Sdg ≈ S†`, `Tdg ≈ T†`.
- CZ, CY, CP, CRX, CRY, CRZ, CS, CSX composition verification.
- CH, CU ABC decomposition verification.
- SWAP 3-CX decomposition.
- iSWAP composition verification.
- RZZ, RXX, RYY, RZX composition verification.
- ECR composition verification.
- XXPlusYY, XXMinusYY composition verification.
- CCX V-decomposition verification.
- CCZ, CSWAP composition verification.
- RCCX, C3X, C3SX, C4X, RC3X verification.
- MCX, MCPhase with variable controls.
- MS composition (product of RXX).
- PauliGate tensor product construction.
- DiagonalGate: small examples, verify matrix.
- PermutationGate: known permutations.
- MCMTGate: verify multi-controlled multi-target.
- PauliProductRotationGate: verify known rotations.
- UCRZGate, UCRYGate, UCRXGate: verify on small examples.
- UCGate: verify general case.
- UnitaryGate: verify reconstruction.
- LinearFunction: verify GF(2) examples.
- Isometry: verify.
- PauliEvolutionGate: single term, multi-term.
- HamiltonianGate: known Hermitian.
- QFTGate: verify matrix for small n against the canonical column-bit-reversed
  formula, not the row-bit-reversed variant.
- AndGate, OrGate, BitwiseXorGate, InnerProductGate: truth tables.
- HalfAdderGate, FullAdderGate: truth tables.
- ModularAdderGate: known additions.
- MultiplierGate: known multiplications.
- LinearPauliRotationsGate, PolynomialPauliRotationsGate: verify angles.
- IntegerComparatorGate: verify comparisons.
- QuadraticFormGate, WeightedSumGate: verify.
- PhaseOracleGate, BitFlipOracleGate: verify oracles.
- GraphStateGate: verify graph states.
- RV gate: `RV(0,0,0) = I`, `RV(π,0,0) = (−i)*X`.
- Action of every single-qubit gate on |0⟩ and |1⟩.
- Tier chain verification: trace 3+ gates per tier down to CX + single-qubit.

#### Parameter (min 15)

- Create Param, verify name.
- `Param("theta").bind({"theta": 3.14})` → 3.14.
- `Param("theta") * 2` bound → correct.
- `Param("a") + Param("b")` bound → correct.
- Partial binding leaves unbound symbolic.
- `Param("x") / 2` bound → correct.
- `Param("x") - Param("y")` → correct.
- Nested: `(Param("x") + 1) * 2` → correct.
- `isResolved()` false for unbound, true for bound.
- Numeric literal always resolved.
- Built-in constants (pi, tau, euler) preserved exactly.
- Partial binding preserves structure for `gamma + (phi + lambda) / 2`.

#### Circuit (min 100)

- Build circuit with every gate type, verify instruction count.
- Chaining: `qc.h(0).cx(0,1).measure(0,0)`.
- Implicit qubit allocation.
- Implicit classical bit → default register.
- Named classical register declaration and order preserved.
- Measurement into named register: flat + named refs.
- Measurement syntax preference preserved.
- `setProgramVersion()` stores variants.
- `include()` preserves directive order.
- Serialized OpenQASM always includes `include "stdgates.inc";`.
- Comments preserve order.
- `run()` replaces parameters.
- `run()` partial binding.
- `complexity()` returns correct metrics.
- `inverse()` reverses and daggers unitary-only circuits; rejects non-unitary
  scopes.
- `compose()` maps qubits/clbits, preserves registers.
- `toGate()`, `toInstruction()` preserve `globalPhase`.
- Control flow with nested phase-owning bodies and branch-local `globalPhase`.
- `defineGate()` and `defineSubroutine()` store phase-owning nested
  `QuantumCircuit` bodies.
- `barrier()` and `delay()` stored.
- `barrier()` zero operands → all qubits.
- `delay()` accepts `DurationExpr`.
- `timed()` preserves timing designators.
- `GateCall.localPhase` survives parse/build/serialize flows and remains
  distinct from explicit `globalPhaseGate()` and scalar `globalPhase`.
- `applyGate(...)` preserves `localPhase`, `surfaceName`, and exact modifier
  stacks when convenience methods would canonicalize them away.
- Every public IR/API carrying zero-qubit phase documents whether it is
  source-preserving / round-trip or normalized semantic.
- `GateCall.localPhase` may normalize only into the immediate owning scope's
  scalar `globalPhase`, never into an ancestor scope.
- If a bare `GateCall.localPhase` is folded into scope `globalPhase`, later
  modifier-introducing rewrites on that same call first re-materialize the exact
  same correction on the gate expression (or prove another exactly equivalent
  rewrite).
- Broadcasted `GateCall.localPhase` expands lane-wise and contributes once per
  expanded lane, not once per syntactic broadcast statement.
- Textual OpenQASM serialization of a nonzero `GateCall.localPhase` succeeds
  only via an exact same-expression gate-family encoding, a fold-safe same-
  scope zero-qubit-phase lowering, or an exact helper-gate encoding when the
  serializer contract permits introducing one; unsupported generic cases are
  rejected rather than misserialized as sibling `gphase`.
- Hoistable same-scope `globalPhaseGate()` with path-invariant total phase may
  normalize into scalar `globalPhase` in normalized non-source-preserving IRs.
- Fold-safety may be proved by exact scalar-phase equivalence, not only by
  syntactic identity of phase expressions; for example, exact path totals `0`
  and `2*pi` are fold-compatible when that equivalence is proved exactly.
- A directly stored scope scalar `globalPhase` is itself entry-invariant in that
  same scope; phases that depend on later-created or path-dependent data remain
  explicit as `globalPhaseGate(...)` or another equally explicit
  locality-preserving representation.
- The specification does not require one unique canonical folded form for
  hoistable zero-qubit phase; absent an API contract for canonical
  normalization, tests accept either the explicit or the folded representation.
- Hoisting/folding is rejected when the translated phase expression is not
  immutable or not valid at that scope's entry.
- Hoisting/folding is rejected for branch-local, loop-repeated, early-exit-
  skipped (`break`, `continue`, `return`, `end`), or other non-fold-safe
  zero-qubit phase corrections.
- Hoisting/folding is rejected when it would erase annotations, timing,
  source-span or round-trip identity, or other locality-sensitive instruction
  metadata.
- If a source-preserving or exact-round-trip parser/serializer mode is exposed,
  textual gate-family identity such as `U` versus `u3`, `p` versus `phase`, and
  `cp` versus `cphase` remains recoverable independently of internal semantic
  canonicalization; exact operand/reference forms, modifier stacks, and
  parameter/localPhase expression trees remain recoverable as well.
- Nested ordinary bodies determine zero-qubit phase arity from all distinct
  underlying quantum wires semantically visible at body entry after alias
  resolution, including any inherited wires and any physical-qubit or equivalent
  frontend forms already semantically available at entry; mere syntactic ability
  to spell additional physical-qubit literals does not enlarge that entry arity.
- After inlining or any other scope-restructuring normalization, fold legality
  is judged against the zero-qubit phase correction's current owning ordinary
  scope in the transformed IR. Source-preserving / round-trip APIs retain the
  original scope containment until they intentionally transition to normalized
  semantic form.
- Calibration scopes preserve explicit zero-qubit phase instructions and do not
  infer/fold a scalar `globalPhase` unless the loaded calibration grammar has
  the explicit implementation-visible opt-in contract required by Phase
  Convention §3; selecting a grammar name alone is insufficient, and otherwise
  such phase remains opaque to generic matrix/unitary reasoning outside that
  calibration contract.
- Symbolic `globalPhase` survives bind/compose/inverse.
- Exact non-integer powers allowed by Phase Convention 6 retain known-unitary
  status and use the deterministic `wrapPhase` principal branch for any numeric
  eigenphases.
- Optimizers do not silently wrap exact-operator gate parameters modulo `2*pi`
  unless exact equality is proved.
- Non-hoistable explicit zero-qubit phase instruction coexists with scalar
  `globalPhase`.
- Every Expansion API method tested (declarations, aliases, expressions, control
  flow, calibration, subroutines, pragmas, annotations — see Section 5 method
  list).
- `classicalAssignOp` with all 12 compound operators stores correct operator.
- `exprStatement` appends `ExpressionStatement` node.
- `returnValue` and `returnVoid` append `ReturnStatement` with/without value.
- `declareLegacyQReg` and `declareLegacyCReg` store legacy declarations distinct
  from modern `qubit[N]`/`bit[N]`.
- `declareOutput` with initializer preserves the initializer through
  serialization.
- `declareInput` with default value stores the default.
- `prepareState` with `AmplitudeVector`, `BasisState`, and `BitstringState`
  stores correct `StateSpec` variant.
- `initialize` stores `InitializeStatement` (includes reset before preparation).
- `timed` wraps a single operation with a duration expression and produces a
  `TimedOperation` node.
- `GateCall.modifiers` array stores modifiers in outermost-first order (matches
  textual left-to-right) and is interpreted right-to-left semantically.
- `Expr` factory methods produce correct expression node types.
- `Expr.constant("pi")` produces `BuiltInConstant`, not `IdentifierExpr`.
- `Expr.sizeOf`, `Expr.realPart`, `Expr.imagPart` produce correct nodes.
- `Expr.physicalQubit(0)` produces `PhysicalQubitExpr(0)`.
- `applyGate` with `PhysicalQubitRef` operands stores physical-qubit operands.
- `TranspilationMetadata` is null on manually constructed circuits and populated
  after compilation-pipeline stages.
- `ValidationResult` collects all diagnostics; `valid` is true only when no
  `Error`-severity diagnostics exist.
- `ScopeKind` correctly distinguishes `Box` and `ControlFlow` from `LocalBlock`.
- `LineComment` and `BlockComment` are preserved in instruction list and ignored
  by semantic passes.
- Edge cases: 0 instructions, only measurements, only resets.
- Qubit index validation (no duplicates).
- Every higher-tier gate method (Tiers 6–14) stores correct instruction.

#### Simulator (min 70)

- `X|0⟩` → `{ "1": 100 }`.
- `H|0⟩` → ~`{ "0": 50, "1": 50 }`.
- Bell state → ~`{ "00": 50, "11": 50 }`.
- GHZ-3 → ~`{ "000": 50, "111": 50 }`.
- `getStateVector` for H|0⟩, Bell state.
- `getStateVector` ignores terminal measurements.
- `getStateVector` rejects mid-circuit measurement.
- Every single-qubit gate on |0⟩: state vector = first column.
- Every single-qubit gate on |1⟩: state vector = second column.
- Parameterized RX(π)|0⟩ → [0, −i].
- Y|0⟩ → [0, i].
- Classical conditioning via `if_test`.
- `for_loop`, `while_loop`, `switch`.
- Reset mid-circuit.
- Multi-register circuit: histogram concatenates registers in order.
- Every two-qubit gate on input states.
- Every three-qubit gate on input states.
- Every four-qubit gate on input states.
- Multi-controlled gates with variable controls.
- Higher-tier gates: MS, Pauli, Diagonal, Permutation, MCMT,
  PauliProductRotation, UCR gates, UnitaryGate, QFT, logic gates, arithmetic
  gates, function-loading gates, comparison gates, state preparation gates.
- `prepareState`, `initialize`.
- `compose`, `inverse`.
- `globalPhaseGate`: state vector phase.
- `barrier`, `delay`: no state change.
- Multi-shot statistical tests (tolerance ±5 on percentages for 1024 shots).

#### Transpiler (min 120)

**Serialization tests (min 70):**

- Round-trip: `deserialize(serialize(circuit))` produces equivalent circuit.
- Serialize every gate type, verify output.
- Deserialize hand-written OpenQASM 3, verify source-preserving / round-trip
  circuit.
- Header format: `OPENQASM 3.1;`, `OPENQASM 3;`, omitted version.
- Multiple named classical registers as separate declarations.
- Include directives preserve order.
- `defcalgrammar` preserves order.
- Gate parameter formatting.
- Measurement formats: assignment, arrow, discarded, register-wide.
- Reset, barrier, delay formats (including bare forms).
- Timed gate calls, timed modifier-bearing calls.
- `box`, `box[duration]`.
- Global phase: hoistable, symbolic, non-hoistable, branch-local.
- `inv @ gphase`, `pow(k) @ gphase`, `ctrl @ gphase`, `ctrl(2) @ gphase`, mixed
  `negctrl/ctrl @ gphase`.
- Exact `pow(k)` exponent classification: integer-valued expressions proved
  exactly (for example `2`, `2.0` when that finite decimal literal is documented
  by the active frontend syntax as exact 2, `6/3`, or a later-bound exact
  integer) use integer-power semantics; exponents whose integrality is unproved
  remain unresolved or are rejected in contexts that require a concrete
  operator, without epsilon-based guessing.
- Principal-branch fractional powers: mandatory scalar cases such as
  `pow(1/2) @ gphase(pi)` and, if general numeric spectral powers are
  implemented, a matrix example with eigenvalue `-1` (for example
  `pow(1/2) @ z`) use the `(-pi, pi]` branch deterministically; ill-conditioned
  or uncertifiable numeric spectral cases are rejected rather than guessed.
- Textual `U`, `u2/u3`, and `cu` phase-exact handling, with `cu` limited to its
  documented control-subspace relative-phase role and not treated as a generic
  whole-expression scalar-phase channel, plus family-level exact boundary
  handling for `rx/ry/rz`, `p/phase`, `crx/cry/crz`, `cp/cphase`, `u1`, and
  `sx/sxdg`, including pre-modifier boundary translation, legal versus illegal
  use of exact same-family reparameterization when it is separately proved,
  legal versus illegal use of helper-gate synthesis when direct call-site
  lowering is impossible but declaration-based exact lowering is permitted,
  exact-zero tests by scalar identity rather than literal angle syntax, hoisting
  of local phase corrections, `U_qasm = exp(i*theta/2) * U_can`,
  `rx_qasm(theta) = RX(theta)`,`ry_qasm(theta) = RY(theta)`,`rz_qasm(theta)
    = RZ(theta)`,`crx_qasm(theta) = CRX(theta)`,`cry_qasm(theta) =
  CRY(theta)`,`crz_qasm(theta) = CRZ(theta)`,`p_qasm(lambda) =
    phase_qasm(lambda) = P(lambda)`,`cp_qasm(lambda) = cphase_qasm(lambda) =
    ctrl(P(lambda))`,`sx_qasm = SX`,`sxdg_qasm = SXdg`,`CU_qasm(theta, phi,
    lambda, gamma) = CU(theta, phi, lambda, gamma + theta/2)`,`u1_qasm(lambda) =
    P(lambda)`,`u3_qasm(theta, phi, lambda) = exp(-i*(phi + lambda)/2) *
    U_can(theta, phi, lambda)`,`u2_qasm(phi, lambda) = exp(-i*(phi + lambda)/2) *
    U_can(pi/2, phi, lambda)`,
  and the deliberate non-identity
  `u1_qasm(lambda) !=
    u3_qasm(0, 0, lambda)`.
- Serialization rejects a nonzero `GateCall.localPhase` whenever no exact
  textual lowering exists; in particular, modifier-internal corrections are not
  extracted as sibling `gphase`, and principal-branch non-integer `pow(k)`
  operands are rejected unless a specific exact textual or permitted helper
  encoding is available.
- Directly stored scope scalar `globalPhase` values are entry-invariant in their
  own scope; phases depending on loop indices, later-created branch-local
  values, or measurement results remain explicit rather than being stored in the
  scope scalar.
- Nested gate definitions and control-flow bodies keep translated `U`/`u2`/`u3`
  local-phase corrections inside their immediate owning scope; later `ctrl @`,
  `negctrl @`, `inv @`, and `pow(k) @` must still see the exact corrected
  operator, and any temporary fold into scope scalar phase is reversed before
  such a transform if needed.
- Broadcasted translated `U`/`u2`/`u3` gate calls duplicate their `localPhase`
  correction per expanded lane before any legal folding; any canonical lane
  order used internally remains representational only.
- Control flow: if/else, else-if chains, while, for (set, range, register,
  array), switch (multi-label cases), break, continue, end, return.
- Hoisting analysis rejects zero-qubit phase corrections that any reachable
  `break`, `continue`, `return`, or `end` can skip.
- Nested ordinary-body `gphase` and scalar `globalPhase` denotations use all
  distinct underlying quantum wires semantically visible at body entry after
  alias resolution, including any inherited wires and any physical-qubit or
  equivalent frontend forms already semantically available at entry; mere
  syntactic ability to spell additional physical-qubit literals does not enlarge
  that entry arity.
- Calibration-scope zero-qubit phase remains explicit through parse/serialize
  flows unless the loaded companion grammar carries the explicit opt-in contract
  required by Phase Convention 3 for the ordinary-scope scalar-phase semantics.
- `deserialize` preserves exact gate-family spellings and other syntax-local
  metadata required for round-trip, including exact operand/reference forms,
  modifier stacks, and parameter/localPhase expression trees, while compilation-
  pipeline stages return normalized semantic circuits unless explicitly
  documented otherwise.
- After inlining or other scope-restructuring normalization, fold legality is
  judged against the current owning ordinary scope in the transformed IR; a
  source-preserving / round-trip API retains original scope containment until it
  intentionally normalizes.
- Unless a tested API explicitly promises canonical normalization, phase tests
  accept either a fold-safe explicit zero-qubit phase instruction or its folded
  scalar-`globalPhase` form.
- Single-statement bodies (no braces).
- Nested control flow round-trip.
- Symbolic parameters survive round-trip.
- Multi-qubit gate serialization.
- Gate modifiers: ctrl, negctrl, inv, pow, chained.
- Custom gate definition.
- Classical types: bool, int[n], uint[n], float[n], angle[n], complex, duration,
  stretch.
- Const, input, output declarations.
- Default-output rule when no explicit output.
- Arrays, legacy qreg/creg, aliases, selectors, bit slicing.
- Classical expressions: arithmetic, bitwise, comparison, logical.
- Assignment operators.
- Built-in constants and math functions.
- Timing intrinsics: `durationof`, `sizeof`.
- Casting, measure expressions, indexing/slicing.
- Physical qubits.
- Gate broadcasting, including lane-wise `localPhase` preservation.
- Subroutines with array params and sizeof.
- Extern declarations.
- Calibration: grammar, `cal` blocks, `defcal`, resources, instructions, opaque
  fallback.
- Pragmas, annotations, comments.
- Literals: integer (hex, octal, binary), boolean, bitstring, imaginary, timing.
- `IntegerLiteral.base` round-trips correctly (hex stays hex, binary stays
  binary).
- `BuiltInConstant("pi")` serializes as `pi`, not as a numeric approximation.
- `SizeOfExpr` round-trips as `sizeof(target)` or `sizeof(target, dim)`.
- `RealPartExpr` / `ImagPartExpr` round-trip as `real(expr)` / `imag(expr)`.
- All 16 built-in function names round-trip through `CallExpr`.
- `MeasureStatement` with all three syntax variants (Assignment, Arrow, Bare)
  round-trips correctly.
- `ReturnStatement` with value, void, and `return measure q` round-trips.
- All 12 compound assignment operators round-trip through `Assignment.operator`.
- `LineComment` and `BlockComment` nodes survive round-trip at correct
  positions.
- `OutputDeclaration` with initializer round-trips.
- `LegacyRegisterDeclaration` (qreg/creg) round-trips distinct from modern
  qubit[N]/bit[N].
- `ArrayReferenceType` with `ExactDimensions` vs `RankOnly(#dim)` round-trips.
- `PrepareStateStatement` / `InitializeStatement` serialize to gate sequences.
- `TimedOperation` round-trips as `box[duration] { operation; }`.
- `GateCall.modifiers` outermost-first array order preserved through round-trip.
- `TranspilationMetadata` populated after compilation pipeline stages; null on
  input circuits.
- `ValidationResult` collects multiple diagnostics; `Error` severity prevents
  `valid = true`.
- `ScopeKind.Box` and `ScopeKind.ControlFlow` correctly restrict statement
  legality.
- Physical-qubit operands (`$0`, `$1`) in post-layout circuits round-trip.

**Compilation pipeline tests (min 50):**

- ZYZ decomposition: H, X, Y, Z, S, T, RX(π/4), arbitrary U → recompose.
- ZYZ canonicalization: ranges, tie-break rules, boundary cases.
- RZ+SX decomposition: several gates → recompose with globalPhase.
- RZ+SX anti-diagonal case.
- KAK decomposition: CX, SWAP, iSWAP, CZ, arbitrary 2-qubit → recompose.
- KAK CX count.
- High-level synthesis: SWAP → 3 CX, CCX decomposition.
- Layout: 3-qubit circuit on 5-qubit linear coupling map.
- Routing: all 2-qubit gates on adjacent qubits after routing.
- Basis translation: {H, CX} → {RZ, SX, CX}.
- Optimization: CX*CX cancels, Rz(a)*Rz(b) merges, Rz(0) removed.
- Full pipeline: Bell state for 5-qubit linear backend.
- ECR backend transpilation.
- Gate direction reversal.
- Composite unrolling.
- Gate modifier expansion: ctrl, ctrl(2), negctrl, inv, pow, chained.
- Zero-qubit phase with modifiers.
- Custom gate inlining.
- Parameterized custom gate inlining.
- Subroutine inlining.
- Const evaluation.
- Input resolution.
- Gate broadcasting expansion.
- Pragma/annotation stripping.

#### IBM Backend (min 20)

**Always-run tests (transpilation and payload):**

- Construct with mock 5-qubit configuration.
- `transpileAndPackage` on Bell state: only basis gates.
- Coupling map respected.
- OpenQASM 3 in payload valid, preserves registers.
- Payload structure: `program_id`, `backend`, `params.version`, PUB tuple with
  shots.
- Auth modes: exactly one of bearerToken/apiKey.
- CORS proxy config preserved and defaults.
- `IBMExecutable` contains all required fields.
- Polling logic: status from `status` or `state.status`, capitalized values.
- Sampler V2 result parsing: dynamic register detection, circuit-order
  reconstruction.
- Multi-register IBM results: recombined using register metadata.
- Transpile GHZ-3 for 5-qubit backend.
- Transpile for ECR-based backend.
- Transpile for CX-based backend.
- Gate direction compliance.
- Compiled circuit depth reasonable.
- Target construction from configuration.

**Skipped tests (require credentials):**

- Submit Bell state job, retrieve results.
- Percentages sum to 100.
- Bitstring format correct.
- Poll status transitions.
- Handle Failed/Cancelled.
- End-to-end.

#### qBraid Backend (min 20)

**Always-run tests (transpilation and payload):**

- Construct with mock 5-qubit configuration.
- `transpileAndPackage` on Bell state: only basis gates.
- Coupling map respected.
- OpenQASM 3 in payload valid.
- Device capability discovery/validation of `runInputTypes`.
- Payload structure: shots, deviceQrn, program.format, program.data.
- API config: headers, routes.
- CORS proxy config preserved and defaults.
- `QBraidExecutable` contains all required fields.
- Response parsing: `{ success, data }` envelope.
- Transient statuses: INITIALIZING, QUEUED, VALIDATING, RUNNING, CANCELLING.
- Terminal statuses: COMPLETED, FAILED, CANCELLED.
- UNKNOWN and HOLD handling.
- Transpile GHZ-3 for 5-qubit backend.
- Transpile for CX-based and ECR-based backends.
- Gate direction compliance.
- Target construction.

**Skipped tests (require credentials):**

- Submit Bell state job, retrieve results.
- Percentages sum to 100.
- Poll status transitions.
- Handle CANCELLING→CANCELLED, FAILED, UNKNOWN, HOLD.
- End-to-end.
- List available devices.

#### Bloch (min 20)

- |0⟩: (0, 0, 1), θ=0, r=1.
- |1⟩: (0, 0, −1), θ=π, r=1.
- H|0⟩ = |+⟩: (1, 0, 0), r=1.
- S*H|0⟩: (0, 1, 0), r=1.
- Y|0⟩ = i|1⟩: (0, 0, −1) (global phase doesn't change Bloch).
- Bell state qubit 0: ~(0, 0, 0), r≈0 (maximally mixed).
- Bell state qubit 1: also maximally mixed.
- RX(π/2)|0⟩: verify coordinates.
- RY(π/2)|0⟩: verify coordinates.
- RZ(π/2)|0⟩: still |0⟩, bloch = (0, 0, 1).
- Spherical consistency: `r ≈ sqrt(x² + y² + z²)`.
- θ ∈ [0, π], φ ∈ [0, 2π).
- 3-qubit GHZ: each qubit maximally mixed.
- Pure state: r = 1 for any single-qubit pure state.

### 11.4 Statistical Tolerance for Shot-Based Tests

- Use `numShots = 1024`.
- Allow ±5 tolerance on each percentage value.
- For deterministic circuits (e.g. `X|0⟩` → `{ "1": 100 }`), use exact
  comparison.

### 11.5 Skippable Tests (IBM & qBraid)

Tests that require real credentials **must be gated behind a skip mechanism**.

**IBM Backend:**

| Language   | Mechanism                                                         |
| ---------- | ----------------------------------------------------------------- |
| TypeScript | Check (`IBM_BEARER_TOKEN` or `IBM_API_KEY`) and `IBM_SERVICE_CRN` |
| Python     | `@pytest.mark.skipif` on same env vars                            |
| Rust       | `#[ignore]`, run with `cargo test -- --ignored`                   |
| Go         | `t.Skip(...)` if env vars missing                                 |

**qBraid Backend:**

| Language   | Mechanism                                 |
| ---------- | ----------------------------------------- |
| TypeScript | Check `QBRAID_API_KEY` env var            |
| Python     | `@pytest.mark.skipif` on `QBRAID_API_KEY` |
| Rust       | `#[ignore]`                               |
| Go         | `t.Skip(...)` if `QBRAID_API_KEY` missing |

Convention: define two test categories per cloud backend:

1. **`*_transpile` tests**: transpilation, payload, serialization — always run.
2. **`*_execute` tests**: job submission, polling, results — skipped by default.

### 11.6 Test Execution is a Build Step

Tests are not optional. The build process is:

1. Implement module.
2. Write tests for that module.
3. Run tests. All must pass.
4. Only then proceed to the next module.

After all modules are complete: 5. Run the full integration test suite. 6. All
900+ tests must pass (excluding skipped IBM/qBraid execution tests).

---

## 12. Verification Checklist

Before declaring the library done, verify every item:

### Gates

- [ ] All 19 Tier 0 single-qubit gate functions implemented and tested.
- [ ] Tier 1 CXGate implemented and tested.
- [ ] All 12 Tier 2 controlled gates implemented and tested.
- [ ] All 9 Tier 3 interaction gates implemented and tested.
- [ ] All 4 Tier 4 three-qubit gates implemented and tested.
- [ ] All 6 Tier 5 four-qubit and multi-controlled gates implemented and tested.
- [ ] All 6 Tier 6 N-qubit composite gates implemented and tested.
- [ ] All 8 Tier 7 uniformly controlled and unitary synthesis gates implemented
      and tested.
- [ ] All 2 Tier 8 Hamiltonian simulation gates implemented and tested.
- [ ] Tier 9 QFTGate implemented and tested.
- [ ] All 4 Tier 10 reversible classical-logic gates implemented and tested.
- [ ] All 4 Tier 11 quantum arithmetic gates implemented and tested.
- [ ] All 7 Tier 12 function loading gates implemented and tested.
- [ ] All 5 Tier 13 comparison/aggregation/oracle gates implemented and tested.
- [ ] Tier 14 GraphStateGate implemented and tested.
- [ ] Every gate matrix verified unitary.
- [ ] Every multi-qubit gate verified compositionally (decomposition matches
      reference matrix).

### QuantumCircuit

- [ ] One chainable method per gate (Tiers 0–14).
- [ ] Every Expansion API method (Section 5) implemented.
- [ ] Gate modifiers: `ctrl`, `negctrl`, `inv`, `pow` implemented.
- [ ] Generic gate invocation via `applyGate(...)` preserves `localPhase`,
      `surfaceName`, and exact modifier stacks when convenience methods would
      lose them.
- [ ] Custom gate definition (`defineGate`) and usage via `append`.
- [ ] Classical declarations, assignments, arrays, const/input/output.
- [ ] Subroutines, externs, calibration.
- [ ] Control flow: `ifTest`, `forLoop`, `whileLoop`, `switch`, `breakLoop`,
      `continueLoop`, `end`, `box`.
- [ ] Composition: `compose`, `toGate`, `toInstruction`, `append`, `inverse`.
- [ ] `inverse()` defined only for unitary-only circuits/scopes; non-unitary
      inputs rejected.
- [ ] State preparation: `prepareState`, `initialize`.
- [ ] Non-unitary: `measure`, `measureRegister`, `reset`, `barrier`, `delay`.
- [ ] Pragmas, annotations, comments, includes, version metadata.
- [ ] Ordered named classical-register metadata preserved through all
      transforms.
- [ ] Ordinary nested bodies (`defineGate`, `defineSubroutine`, control flow,
      `box`) are represented as phase-owning `QuantumCircuit` bodies with their
      own scalar `globalPhase`.
- [ ] Hoistable zero-qubit phase may normalize into scalar `globalPhase`;
      non-hoistable explicit zero-qubit phase stays instruction.
- [ ] Every public IR/API carrying zero-qubit phase documents whether it is
      source-preserving / round-trip or normalized semantic.
- [ ] Unless a tested API explicitly promises canonical normalization, fold-safe
      zero-qubit phase may remain either explicit or folded; any API documented
      as canonical normalization applies a deterministic fold-safe proof and
      rewrite strategy.
- [ ] If a bare translated `GateCall.localPhase` is folded into scope
      `globalPhase`, any later modifier-introducing rewrite on that same call
      first restores the exact same correction to the gate expression or proves
      another exactly equivalent rewrite.
- [ ] Fold-safety proofs may use exact scalar-phase equivalence (for example
      exact path totals `0` and `2*pi`) but never floating-point heuristics,
      epsilon tests, or approximate modulo reasoning.
- [ ] After inlining or other scope-restructuring normalization, fold legality
      is judged against the current owning ordinary scope in the transformed IR;
      source-preserving / round-trip APIs retain original scope containment
      until they intentionally normalize.
- [ ] Nested ordinary bodies determine zero-qubit phase arity from all distinct
      underlying quantum wires semantically visible at body entry after alias
      resolution, including any inherited wires and any physical-qubit or
      equivalent frontend forms already semantically available at entry; mere
      syntactic ability to spell additional physical-qubit literals does not
      enlarge that entry arity.
- [ ] Calibration scopes preserve explicit zero-qubit phase and do not infer or
      fold a scalar `globalPhase` unless the loaded calibration grammar has the
      explicit implementation-visible opt-in contract required by Phase
      Convention §3; otherwise such phase remains opaque to generic
      matrix/unitary reasoning outside that calibration contract.
- [ ] `classicalAssignOp` supports all 12 compound assignment operators.
- [ ] `exprStatement` appends `ExpressionStatement` node.
- [ ] `returnValue` and `returnVoid` append `ReturnStatement`.
- [ ] `declareLegacyQReg` and `declareLegacyCReg` preserve legacy syntax
      distinct from modern declarations.
- [ ] `declareOutput` with initializer preserves the initializer.
- [ ] `prepareState` accepts `AmplitudeVector`, `BasisState`, and
      `BitstringState` via `StateSpec` (§5.19).
- [ ] `initialize` stores `InitializeStatement` (reset + preparation).
- [ ] `timed` wraps operations with duration designators (§5.20).
- [ ] `GateCall.modifiers` stores modifiers in outermost-first order.
- [ ] `Expr` factory module produces all expression node types from §5.5.
- [ ] `BuiltInConstant` nodes for `pi`, `tau`, `euler`, `im`.
- [ ] `SizeOfExpr`, `RealPartExpr`, `ImagPartExpr` expression nodes.
- [ ] `MeasureStatement` unified model with `syntax` field (Assignment, Arrow,
      Bare).
- [ ] `LineComment` and `BlockComment` IR nodes preserved in instruction list.
- [ ] `TranspilationMetadata` null on manual circuits, populated after
      compilation.
- [ ] `ValidationResult` collects all diagnostics; `valid` requires zero
      `Error`-severity entries.
- [ ] `ScopeKind` includes `Box` and `ControlFlow`.
- [ ] Physical-qubit operands via `applyGate(...)` with `PhysicalQubitRef`.
- [ ] `ArrayReferenceType` distinguishes `ExactDimensions` from `RankOnly`.
- [ ] Built-in function names recognized in `CallExpr` validation.
- [ ] `complexity()` returns correct metrics.
- [ ] `blochSphere()` returns correct Bloch coordinates.
- [ ] `run()` binds parameters correctly.

### Transpiler

- [ ] `Transpiler` interface defined with `serialize` and `deserialize`.
- [ ] `OpenQASMTranspiler` implements complete OpenQASM 3.1
      serialize/deserialize.
- [ ] `deserialize` returns a source-preserving / exact-round-trip
      `QuantumCircuit`; `serialize` honors preserved `localPhase`,
      `surfaceName`, explicit zero-qubit phase, exact operand/reference forms,
      exact modifier stacks, and exact parameter/localPhase expression trees
      when present.
- [ ] Compilation-pipeline stages (`transpile`, `unrollComposites`,
      `expandGateModifiers`, `inline*`, `synthesizeHighLevel`, `layoutSABRE`,
      `routeSABRE`, `translateToBasis`, `optimize`) are documented and tested as
      normalized semantic APIs unless explicitly stated otherwise.
- [ ] For every circuit that `serialize` accepts, round-trip
      `deserialize(serialize(circuit))` produces an equivalent circuit; circuits
      with no exact textual lowering are rejected rather than misserialized.
- [ ] All OpenQASM 3.1 features from Section 4 handled (types, declarations,
      expressions, control flow, gate modifiers, custom gates, subroutines,
      externs, calibration, pragmas, annotations, comments, literals, timing,
      physical qubits, gate broadcasting, aliases, indexing/slicing).
- [ ] New expression nodes round-trip: `BuiltInConstant`, `SizeOfExpr`,
      `RealPartExpr`, `ImagPartExpr`, `IntegerLiteral.base`.
- [ ] All 16 built-in function names recognized and round-tripped via
      `CallExpr`.
- [ ] Unified `MeasureStatement` with `syntax` field (Assignment, Arrow, Bare)
      round-trips all three forms correctly.
- [ ] `ReturnStatement` round-trips with value, void, and `return measure q`.
- [ ] All 12 compound assignment operators round-trip via `Assignment.operator`.
- [ ] `LineComment` and `BlockComment` nodes preserved at correct positions in
      round-trip.
- [ ] `OutputDeclaration` with initializer round-trips.
- [ ] Legacy `qreg`/`creg` round-trips distinct from modern `qubit[N]`/`bit[N]`.
- [ ] `ArrayReferenceType` with `ExactDimensions` vs `RankOnly(#dim)`
      round-trips.
- [ ] `PrepareStateStatement` / `InitializeStatement` lower to gate sequences
      during synthesis.
- [ ] `TimedOperation` serializes as `box[duration] { operation; }`.
- [ ] `GateCall.modifiers` outermost-first array order preserved through
      serialize/deserialize.
- [ ] `TranspilationMetadata` populated after layout/routing stages.
- [ ] `ValidationResult` collects multiple diagnostics; `Error` severity
      prevents `valid = true`.
- [ ] `ScopeKind.Box` and `ScopeKind.ControlFlow` restrict statement legality
      correctly.
- [ ] Physical-qubit operands (`$0`, `$1`) in post-layout circuits round-trip.
- [ ] Phase-exact handling of textual `U`, `u2/u3`, and `cu`, with `cu` limited
      to its documented control-subspace relative-phase role and not treated as
      a generic whole-expression scalar-phase channel, plus family-level exact
      boundary handling of textual `rx/ry/rz`, `p/phase`, `crx/cry/crz`,
      `cp/cphase`, `u1`, and `sx/sxdg`, at the format boundary, including
      pre-modifier translation, legal-versus-illegal exact same-family
      reparameterization when it is separately proved, legal-versus-illegal
      helper-gate synthesis when direct call-site lowering is impossible but
      declaration-based exact lowering is permitted, exact-zero tests by scalar
      identity rather than literal angle syntax, legal-versus-illegal hoisting
      of local phase corrections, `U_qasm = exp(i*θ/2) * U_can`,
      `rx_qasm(theta) = RX(theta)`, `ry_qasm(theta) = RY(theta)`,
      `rz_qasm(theta) = RZ(theta)`, `crx_qasm(theta) = CRX(theta)`,
      `cry_qasm(theta) = CRY(theta)`, `crz_qasm(theta) = CRZ(theta)`,
      `p_qasm(lambda) = phase_qasm(lambda) = P(lambda)`,
      `cp_qasm(lambda) = cphase_qasm(lambda) = ctrl(P(lambda))`, `sx_qasm = SX`,
      `sxdg_qasm = SXdg`,
      `CU_qasm(theta, phi, lambda, gamma) = CU(theta, phi, lambda, gamma +
      theta/2)`,
      `u1_qasm(lambda) = P(lambda)`,
      `u3_qasm(theta, phi, lambda) = exp(-i*(phi + lambda)/2) *
      U_can(theta, phi, lambda)`,
      and
      `u2_qasm(phi, lambda) = exp(-i*(phi + lambda)/2) *
      U_can(pi/2, phi, lambda)`,
      plus the deliberate non-identity
      `u1_qasm(lambda) != u3_qasm(0, 0, lambda)` relations.
- [ ] If a translated bare `U`/`u2`/`u3` call temporarily folds its correction
      into scope scalar phase, later `ctrl`, `negctrl`, `inv`, or `pow`
      transforms on that same call re-materialize the exact correction on the
      gate expression before transforming.
- [ ] Textual OpenQASM serialization lowers a nonzero `GateCall.localPhase` only
      by an exact gate-family encoding, including any separately proved
      same-family reparameterization, a fold-safe same-scope zero-qubit-phase
      lowering, or an exact helper-gate encoding when the serializer contract
      permits it, and rejects unsupported cases rather than splitting them into
      sibling `gphase`.
- [ ] Broadcasted gate calls preserve `localPhase` per expanded lane, any
      folding/hoisting decision is made only after that lane-wise expansion, and
      any canonical lane order used internally is representational only.
- [ ] If a source-preserving or exact-round-trip parser/serializer mode is
      exposed, textual gate-family identity such as `U` versus `u3`, `p` versus
      `phase`, and `cp` versus `cphase` remains recoverable independently of
      internal semantic gate canonicalization; exact operand/reference forms,
      modifier stacks, and parameter/localPhase expression trees remain
      recoverable as well.
- [ ] Fold-safe zero-qubit phase hoisting rejects paths skipped or repeated by
      branches, loops, `break`, `continue`, `return`, `end`, or equivalent
      reachable control transfer.
- [ ] Nested ordinary-body zero-qubit phase denotations use all distinct
      underlying quantum wires semantically visible at body entry after alias
      resolution, including any inherited wires and any physical-qubit or
      equivalent frontend forms already semantically available at entry; mere
      syntactic ability to spell additional physical-qubit literals does not
      enlarge `scopeQubitArity`.
- [ ] A directly stored scope scalar `globalPhase` is entry-invariant in that
      same scope; phases that depend on loop indices, later-created branch-local
      values, or measurement results remain explicit rather than being stored in
      the scope scalar.
- [ ] Calibration-scope zero-qubit phase remains explicit through
      parse/serialize flows unless the loaded companion grammar carries the
      explicit opt-in contract required by Phase Convention 3 for the
      ordinary-scope scalar-phase semantics.
- [ ] Unless a tested API explicitly promises canonical normalization, fold-safe
      zero-qubit phase may remain either explicit or folded.
- [ ] Exact-zero tests, `2*pi*n` tests, and integer-valued exponent
      classification use a documented exact-expression semantics that includes
      at least the minimum conformance profile defined earlier in this Phase
      Convention, and unsupported cases fail or remain unresolved
      deterministically rather than being guessed from floating-point
      evaluation.
- [ ] `pow(k)` chooses integer-power semantics exactly when the exponent is
      proved integer-valued without floating-point guessing; expressions whose
      integrality is unproved remain unresolved until later exact proof/binding
      or are rejected in contexts that require a concrete operator.
- [ ] Exact non-integer powers allowed by Phase Convention 6 retain
      known-unitary status, use the deterministic `wrapPhase` principal branch
      for any implemented numeric spectral powers, reject uncertifiable
      ill-conditioned spectral cases rather than guessing, and optimizers do not
      silently wrap exact-operator gate parameters modulo `2*pi` without an
      exact proof.
- [ ] Compilation pipeline: unrollComposites, expandGateModifiers,
      inlineGateDefinitions, inlineSubroutines, synthesizeHighLevel,
      layoutSABRE, routeSABRE, translateToBasis, optimize.
- [ ] ZYZ decomposition correct with canonical ranges and tie-break rules.
- [ ] RZ+SX decomposition correct with global phase.
- [ ] KAK/Weyl decomposition correct with global phase.
- [ ] SABRE layout and routing produce valid results.
- [ ] Optimization passes reduce gate count.
- [ ] Transpiled circuits produce equivalent distributions.

### Backends

- [ ] `Backend` interface defined.
- [ ] `SimulatorBackend` implements interface, returns correct distributions.
- [ ] `getStateVector()` returns correct amplitudes.
- [ ] `IBMBackend` implements interface with correct payload, auth, polling, and
      Sampler V2 result parsing.
- [ ] `QBraidBackend` implements interface with correct payload, device
      discovery, status handling, and result parsing.
- [ ] CORS proxy configuration preserved and functional.
- [ ] IBM execution tests gated behind env vars.
- [ ] qBraid execution tests gated behind env vars.

### Core Types

- [ ] `Complex` class: all methods, immutable, fully tested.
- [ ] `Matrix` class: all methods, immutable, fully tested.
- [ ] `Param` class: arithmetic, bind, partial bind, isResolved.
- [ ] All shared types from Section 8.3 defined and exported.

### Quality

- [ ] No external math/LA dependencies.
- [ ] 900+ tests passing (excluding skipped cloud backend execution tests).
- [ ] All public API symbols exported from `mod.{ext}`.
- [ ] `README.md` exists with: installation, quick start, full public API
      reference for every exported symbol, usage examples, test commands,
      license.

---

## 13. README Documentation

**File:** `README.md`

After all modules are implemented and all tests pass, generate a comprehensive
README.

### Structure

1. **Title and Description** — one-paragraph summary.
2. **Installation** — adapted to target language.
3. **Quick Start** — Bell state: build, simulate, print results.
4. **Public API Reference** — every exported symbol, organized by category:
   - QuantumCircuit (constructor, all gate methods grouped by tier, expansion
     API methods, modifiers, composition, inspection, parameter binding).
   - Complex (constructor, constants, factories, arithmetic).
   - Matrix (constructor, factories, operations).
   - Gate Constructors (grouped by tier).
   - Param (constructor, arithmetic, bind, isResolved).
   - Backends (interface, SimulatorBackend, IBMBackend, QBraidBackend).
   - Transpiler (interface, OpenQASMTranspiler, compilation pipeline functions).
   - Bloch Sphere.
   - Types (all exported types with fields).
5. **Usage Examples** — one per scenario:
   - GHZ state
   - Parameterized circuit
   - State vector inspection
   - Bloch sphere
   - OpenQASM 3 round-trip
   - Transpilation for constrained backend
   - Circuit composition
   - Control flow with `ifTest`
   - Custom unitary
   - Complex and Matrix algebra
   - Gate modifiers (`ctrl @`, `inv @`, `pow(k) @`)
   - Custom gate definition
   - QFT circuit
   - Arithmetic circuit (adder)
6. **Running Tests** — commands for target language, including how to enable
   skipped IBM/qBraid tests.
7. **License** — MIT.

### Rules

- Every code example must be syntactically valid.
- Only document exported symbols.
- Keep method descriptions to one line in the API reference.
- The README must be accurate relative to the final implementation.
