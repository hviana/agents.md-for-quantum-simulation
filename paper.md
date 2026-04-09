---
title: "Specification-First, Agent-Driven Development of a Quantum Simulation SDK: The `AGENTS.md` Methodology Behind `@hviana/js-quantum`"
tags:
  - quantum computing
  - agent-based software engineering
  - AGENTS.md
  - large language models
  - reproducible research software
  - OpenQASM
  - TypeScript
authors:
  - name: Henrique Emanoel Viana
    orcid: 0000-0002-7119-5867
    affiliation: 1
affiliations:
  - name: Independent Researcher
    index: 1
date: 8 April 2026
bibliography: paper.bib
---

# Summary

`@hviana/js-quantum` [@jsquantumjsr] is a pure-TypeScript quantum computing SDK
distributed via the JSR registry. It provides approximately eighty quantum gates
organised in fifteen tiers, an OpenQASM 3.1 transpiler, a state-vector
simulator, and adapters to IBM Quantum and qBraid cloud backends, together with
an experimental high-level API submodule for declarative problem specification.
The library is unusual in how it was built: its entire source tree was generated
by Large Language Model (LLM) coding agents driven by two `AGENTS.md`
specifications, with the natural-language specification---not the TypeScript
source---treated as the canonical artefact of the project. The core
specification codifies a six-part complex-phase convention and a compositional
gate hierarchy explicitly grounded in the universality result of Nielsen &
Chuang §1.3.2 [@nielsen2010quantum]; the high-level submodule specification
describes a chainable progressive-disclosure API. In this paper we describe the
software and the specification-first methodology that produced it, and we argue
that promoting the specification to the role of primary artefact yields three
concrete benefits for scientific software: (i) mathematical determinism through
globally enforced sign and phase conventions; (ii) architectural
reproducibility, since multiple agents on multiple platforms regenerate
consistent implementations; and (iii) auditability, because the document itself
becomes the unit of peer review.

# Statement of Need

Quantum computing software stacks have grown rapidly over the past decade, with
widely used frameworks such as Qiskit [@qiskit2024], Cirq [@cirq2024] and
PennyLane [@bergholm2018pennylane] occupying the space between hardware vendors
and end users. These stacks share a common engineering challenge: quantum
semantics is unforgiving. A single sign error in a rotation gate, a forgotten
global phase under a controlled lift, or an inconsistent endianness convention
silently produces wrong results that survive unit testing because the failure
mode is numerical, not syntactic [@nielsen2010quantum; @divincenzo2000physical].

In parallel, LLM-based coding assistants have moved from auto-completion toward
end-to-end task execution [@chen2021evaluating; @vaithilingam2022expectation;
@xu2022systematic]. The community has begun to converge on a lightweight
convention---the `AGENTS.md` file---in which a project's repository ships a
plain-text instruction document that any compliant agent (Claude Code, Codex,
Cursor, Aider, Cline, etc.) consumes before writing or modifying code
[@agentsmd2025]. The convention generalises earlier practices such as
`CLAUDE.md` and editor-specific "project rules" files into a portable
specification surface.

The library `@hviana/js-quantum` [@jsquantumjsr] exploits this convention
aggressively. Two `AGENTS.md` specifications serve as the canonical source of
truth for the project: a long-form document driving the core SDK
[@agentsmdquantumsim], and a second document driving an experimental high-level
API submodule [@agentsmdhighlevel]. The TypeScript source itself is regarded as
a derivable artefact: any sufficiently capable agent fed the specification (in
whole or in sections) is expected to reproduce a behaviourally equivalent
implementation.

The software and the methodology are inseparable, and we describe them together.
We argue that the specification's two load-bearing constructs---the
complex-phase convention and the compositional gate hierarchy of Nielsen &
Chuang §1.3.2---are what make agent-driven generation viable for an application
domain in which correctness has no margin for error.

# State of the Field

## Quantum software engineering

Quantum SDKs share an unusually high specification cost. Beyond the
linear-algebra core, an SDK must commit to: a basis-gate set
[@barenco1995elementary]; conventions for endianness and matrix layout
[@cross2022openqasm3]; a parameterisation of single-qubit unitaries (the
_three-Euler-angle problem_) [@nielsen2010quantum]; treatment of global phase
under controlled lifting; and a compilation pipeline that includes layout and
routing [@li2019tackling], KAK/Cartan decomposition [@tucci2005introduction],
and basis-gate rewriting. Each of these is a place where two equally defensible
choices yield mutually incompatible code. Existing frameworks document these
choices in scattered API references [@qiskit2024; @cirq2024]; the
`@hviana/js-quantum` project instead concentrates them into a single normative
document.

## LLM-assisted scientific software

The use of LLMs as code-writing collaborators is now well documented. Empirical
studies of GitHub Copilot and similar tools report substantial productivity
gains for boilerplate-heavy tasks [@vaithilingam2022expectation;
@peng2023impact], with the caveat that the developer must remain in the loop for
semantic correctness [@barke2023grounded]. For scientific software, where
numerical correctness and reproducibility are paramount [@wilson2014best;
@sandve2013ten], naïve prompting is insufficient: the model has no anchor
against which to validate sign conventions, units, or coordinate frames. The
`AGENTS.md` convention addresses this by relocating the anchor into a versioned,
peer-reviewable file shipped with the repository [@agentsmd2025].

## Build-vs.-contribute justification

Existing quantum frameworks (Qiskit, Cirq, PennyLane) are Python-centric and do
not expose a language-agnostic normative specification that can drive autonomous
code generation. Contributing the `AGENTS.md` methodology to an existing
framework would require retrofitting a global phase convention and a
compositional gate hierarchy onto a codebase whose architectural choices are
already fixed. The scholarly contribution of `@hviana/js-quantum` is therefore
not the TypeScript implementation per se, but the demonstration that a single
specification document, when engineered with the right mathematical invariants,
can serve as the canonical artefact from which multiple behaviourally equivalent
implementations are derived by heterogeneous LLM agents. This contribution is
orthogonal to the feature sets of existing SDKs and could not have been made as
a patch to any of them.

# Software Design

## Specification as primary artefact

The repository `agents.md-for-quantum-simulation` [@agentsmdquantumsim] contains
a single long-form `AGENTS.md` file (currently ~8,900 lines). The file is
language-agnostic: it specifies types, complex arithmetic, dense matrix
operations, gate matrices, circuit construction, transpilation to OpenQASM 3.1,
a state-vector simulator backend, and adapters to IBM Quantum and qBraid cloud
backends---without committing to TypeScript syntax. The TypeScript
implementation hosted in `hviana/js-quantum` [@jsquantumrepo] is one
realisation; the methodology presupposes that other realisations (Python, Rust,
Go) can be obtained by re-running the agent loop with the same specification.

The specification mandates a strict 16-step build order respecting dependency
chains:

1. Foundational types module.
2. Complex arithmetic, implemented from scratch.
3. Dense matrix operations, without BLAS or NumPy [@harris2020array].
4. Tier-0 gate matrices.
5. Tier-1 (CX) and the universality bridge.
6. Higher-tier gates and decompositions.
7. Circuit builder data structures.
8. OpenQASM 3.1 transpiler [@cross2022openqasm3].
9. State-vector simulator with subspace iteration.
10. SABRE-style layout and routing [@li2019tackling].
11. Basis-gate rewriting via ZYZ and KAK [@tucci2005introduction].
12. Cloud backend adapters (IBM Quantum Sampler V2, qBraid REST).
13. Born-rule sampling and mid-circuit measurement.
14. Classical registers and control flow.
15. Test scaffolding (over 900 unit and integration tests).
16. Public export integration.

The specification asserts that "no module may depend on a module later in the
build order" [@agentsmdquantumsim], eliminating the cyclic-dependency failure
mode that has historically plagued LLM-generated codebases [@liu2023refining].

## The complex phase convention

Section 2 of `AGENTS.md` establishes a six-part phase convention. Its motivating
observation is the well-known fact that two unitaries differing only by a global
phase are physically indistinguishable when applied to an isolated state, but
become distinguishable when promoted to a controlled operation
[@nielsen2010quantum]. A convention that ignores this distinction will produce
code in which $\mathrm{ctrl}(W)$ disagrees with $\mathrm{ctrl}(V)$ even though
$W$ and $V$ are "the same gate."

The six conventions, in summary, fix: (1) phase angles as exact complex scalars
$e^{i\alpha}$ with no implicit modulo-$2\pi$ reduction; (2) matrix equality as
entrywise approximate within $\varepsilon = 10^{-10}$ and explicitly _not_
quotiented by global phase; (3) a per-scope scalar `globalPhase` field that acts
at scope entry, into which only fold-safe zero-qubit phase may be absorbed; (4)
controlled lifting from the _full_ corrected matrix, so that an
instruction-local phase $\delta$ in $W = e^{i\delta}V$ becomes physically
observable through the control as $\mathrm{ctrl}(W) = \mathrm{diag}(I,
e^{i\delta}V)$; (5) the canonical single-qubit gate

$$
U_{\mathrm{can}}(\theta,\phi,\lambda) =
\begin{pmatrix}
\cos(\theta/2) & -e^{i\lambda}\sin(\theta/2) \\
e^{i\phi}\sin(\theta/2) & e^{i(\phi+\lambda)}\cos(\theta/2)
\end{pmatrix}
$$

which differs from the OpenQASM 3 textual form by a fixed global phase of
$\theta/2$ that the transpiler must reconcile exactly at the format boundary;
and (6) `inv` and `pow` acting on the full corrected operator, with non-integer
powers requiring principal-branch spectral decomposition or remaining syntactic
until binding. Convention 4 is the load-bearing rule: it forbids the otherwise
tempting optimisation of "stripping global phase" before lifting, which is
precisely the failure mode that produces silent disagreement between
independently synthesised gate tiers.

The convention is normative across the entire specification: every gate matrix,
every parametric formula, every decomposition, every controlled construction,
and every equality check obeys it. From the perspective of agent-driven
generation, this is the single most important property of the document. Without
it, two agents working on adjacent tiers would inevitably make locally
reasonable but globally inconsistent choices about where to absorb phase, and
the union of their outputs would fail at the first controlled-lift test.

## Gate definitions and compositional architecture (Nielsen & Chuang §1.3.2)

The second pillar of the specification is its gate hierarchy. Section 1.3.2 of
Nielsen & Chuang [@nielsen2010quantum] establishes the universality result: any
unitary on $n$ qubits can be decomposed, _up to a global phase_, into a finite
sequence of single-qubit gates and CNOT gates. The `AGENTS.md` specification
lifts this result to _exact equality_ by adjoining a `GlobalPhaseGate`
$(\theta)$ to the Tier 0 primitives. The universal basis is therefore

$$
\mathcal{B}_{\mathrm{universal}} \;=\;
\bigl\lbrace\,\text{Tier 0 single-qubit gates}\,\bigr\rbrace\;\cup\;
\bigl\lbrace\,\mathrm{GlobalPhaseGate}(\theta)\,\bigr\rbrace\;\cup\;
\{\,\mathrm{CX}\,\}.
$$

With this addition, the universality theorem becomes constructive without any
"up-to-phase" caveat. The architectural consequence, quoted from the
specification, is that _"every fixed-family compositional multi-qubit gate and
every exact circuit template in this section is specified by an exact reference
synthesis over Tier 0 primitives and CX"_ [@agentsmdquantumsim].

The library's ~80 gates are organised into 15 tiers (Table 1). The structural
rule is that no tier may introduce a hardcoded matrix larger than Tier 1;
everything above Tier 1 must _decompose_ through the compositional bottleneck.

Table: Tier structure of the gate library. Above Tier 1 every gate is defined by
a reference synthesis through Tier 0 $\cup$ CX, not by a hardcoded matrix.

| Tier | Contents                                                | Depends on         |
| ---- | ------------------------------------------------------- | ------------------ |
| 0    | I, X, Y, Z, H, S, T, SX, RX, RY, RZ, U, GlobalPhaseGate | matrix definitions |
| 1    | CX (universal entangling primitive)                     | Tier 0             |
| 2    | CZ, CY, CRX/Y/Z, CH, CU, DCX                            | Tiers 0--1         |
| 3    | SWAP, RXX, RYY, RZZ, ECR, iSWAP                         | Tier 2             |
| 4    | Toffoli (CCX), CCZ, Fredkin (CSWAP), RCCX               | Tiers 2--3         |
| 5    | C3X, C3SX, C4X, RC3X, MCX, MCPhase                      | Tier 4             |
| 6    | MS, Pauli, Diagonal, Permutation, MCMT, PauliProductRot | Tier 5             |
| 7    | UCRX/Y/Z, UCGate, UnitaryGate, LinearFunction, Isometry | Tier 6             |
| 8    | PauliEvolutionGate, HamiltonianGate                     | Tier 6             |
| 9    | QFTGate                                                 | Tier 2             |
| 10   | And, Or, BitwiseXor, InnerProduct                       | Tier 4             |
| 11   | HalfAdder, FullAdder, ModularAdder, Multiplier          | Tiers 9--10        |
| 12   | LinearPauliRot, Polynomial, Piecewise, Chebyshev, 1/x   | Tiers 5, 11        |
| 13   | IntegerComparator, QuadraticForm, WeightedSum, Oracles  | Tiers 5, 10        |
| 14   | GraphState, structural state-preparation composites     | lower tiers        |

The benefit for agent-driven generation is twofold. First, the rule is locally
checkable: a verification agent reading any Tier-$k$ gate need only confirm that
its body references Tier-$<k$ primitives. Second, the rule is globally testable:
a single equivalence test suffices, comparing the matrix obtained by composing
the Tier-0+CX synthesis against an oracle dense matrix computed by an
independent path. The combination of compositional locality and global
testability makes the gate library tractable for an autonomous agent in a way
that an opaque collection of hardcoded $2^n \times 2^n$ matrices would not be.

## Agent orchestration

The methodology is explicitly multi-agent and platform-portable. The
specification names compatible runners: Claude Code (which auto-loads
`CLAUDE.md` $\rightarrow$ `AGENTS.md`), Codex CLI, Cursor, Windsurf, Cline,
Aider, and the web interfaces of ChatGPT and Gemini. For agents whose context
window cannot hold the entire specification, the document instructs
section-by-section ingestion in dependency order---each section being
self-contained relative to its declared prerequisites. This "feed-forward"
protocol mirrors the hierarchical-task-network discipline of classical AI
planning [@erol1994htn] and aligns with recent work on context-bounded code
generation [@zheng2023codegeex].

Multiple agents are assigned to disjoint regions of the dependency DAG, allowing
parallel synthesis whenever the build order admits it. A separate verification
agent is invoked once each tier is closed, re-reading the relevant specification
section against the produced code to detect drift. This pattern echoes the
"planner-executor-critic" decomposition explored in autonomous
software-engineering benchmarks [@jimenez2024swebench; @yang2024sweagent].

## The high-level API

The methodology proceeds in two sequential steps. The first step, described
above, runs the core `AGENTS.md` specification [@agentsmdquantumsim] to generate
the mathematically rigorous SDK. Once that artefact is in place, a _second step_
is invoked: a second `AGENTS.md` document [@agentsmdhighlevel], archived at
<https://doi.org/10.5281/zenodo.19473636>, is fed to the agent to generate an
experimental _high-level_ API---a submodule layered on top of the previously
generated core SDK. This staging is essential: the high-level specification
presupposes that the primitives, gate hierarchy, and phase conventions of the
core SDK already exist, and instructs the agent to bind to them through a
dedicated bridge layer rather than to re-derive them. Where the core
specification is concerned with mathematical correctness, the high-level
specification is concerned with ergonomics for end users who wish to express
problems without first compiling them to circuits.

The design philosophy rests on two foundational principles. The first is a
strict _classical I/O contract_: users supply only classical data (numbers,
strings, lists, matrices, functions, graphs) and receive classical results;
qubits, gates, and bitstrings never cross the public API surface. The second is
_progressive disclosure_ within a single tier: novices accomplish a task in two
to four chained calls, while experienced users access granular control through
optional configuration objects on the same methods---there is no separate
"advanced" API. The single entry function `quantum(problem, options?)` returns a
`QuantumTask` builder, where `problem` is either a plain-language string
(`"search"`, `"factoring"`, `"optimization"`, ...) or an explicit object for
advanced configuration, and `options` carry execution-environment hints (qubits,
model, backend, resources).

The builder exposes a small set of chainable methods that defer all quantum
resource use until the terminal call. `.data(role, value, options?)` ingests
classical data under a role tag (`"items"`, `"target"`, `"cost"`, `"matrix"`,
...); `.transform(kind, options?)` derives new artefacts through operations such
as `"adjoint"`, `"controlled"`, `"trotterize"`, or `"block_encode"`, recording
full lineage and storing the result symbolically when the host library lacks
native support; `.solve(strategy?, options?)` assembles the quantum pipeline
either from a preset family string, an explicit step array, or a hybrid of the
two, and auto-infers a strategy when omitted entirely; `.run(options?)` is the
_sole_ method that consumes quantum resources, returning a `ResultHandle` whose
`.answer()` delivers the interpreted classical result, `.confidence()` a
measurement-derived score, `.analyze(...)` optional post-processing, and
`.raw()` the uninterpreted quantum data for advanced users. Classical
introspection through `.inspect(...)` and native-circuit extraction through
`.circuit(...)` never touch a backend. Crucially, `.data()` may be called
multiple times to register named artefacts in an internal registry that
`.transform()` and `.solve()` reference by name, so the pipeline is a directed
acyclic graph of artefacts rather than a strictly linear chain.

The implementation phases prescribed by the secondary specification mirror those
of the core specification but are adapted to API design: (0) mandatory
reconnaissance of the host language, its package structure, type conventions,
and export mechanisms; (1) creation of an `hlapi/` submodule scaffold with fixed
subdirectories for the task builder, inputs, transforms, solver, result,
interpretation, inspection, parameter enums, algorithm-family presets, the
artefact registry, the bridge layer, and the examples; (2) full public API
implementation (`quantum`, `QuantumTask`, `ResultHandle`, and the internal
`.input()` for direct quantum-native ingestion); (3) a _bridge layer_ that
concentrates all host-library coupling in a single module responsible for
classical-to-quantum translation, quantum-to-classical interpretation, transform
dispatch, compilation, execution, resource estimation, and circuit extraction,
with a clear fallback rule that symbolic placeholders fail loudly at runtime
when unresolvable; (4) twelve labelled worked examples (A--L), one per algorithm
family---hidden-subgroup/factoring, amplitude amplification, quantum walks,
Hamiltonian simulation, linear systems, optimisation, sampling, machine
learning, error correction, measurement-based, topological, and phase
estimation---each self-contained, importing only the high-level API, and
demonstrating the classical-in/classical-out contract on a realistic (non
look-up) problem; (5) a deliberate _no-execution_ testing policy during code
generation, accepting only static analysis (type checking, import resolution,
syntactic validity) on the grounds that exponential simulation cost makes honest
end-to-end testing infeasible and a hallucinated test that happens to pass is
more dangerous than one that does not run; (6) purely additive export
integration that does not shadow existing symbols; and (7) an appended
"High-Level API (Experimental)" README section with twelve plain-language usage
snippets and a prominent experimental warning.

The bridge layer is the methodological linchpin: by isolating coupling to the
underlying SDK in a single adapter module, the high-level API becomes portable
to any host quantum library that implements the same primitives, just as the
core SDK is portable across host languages. The progressive-disclosure pattern
is well established in scientific software ergonomics [@wilson2017good], but its
agent-driven realisation here is novel: the specification is the only thing the
agent sees, and the resulting API surface is identical regardless of which agent
platform produced it. We highlight that the high-level API is itself flagged as
_experimental_, which underscores a methodological point: marking the artefact
experimental in the specification automatically propagates the warning to the
README, the docstrings, and the public exports, because all three are generated
from the same source.

## Design trade-offs

**Mathematical determinism.** The phase convention described above is the chief
contributor to determinism. Because it is normative across the entire
specification, two independent agents synthesising disjoint tiers cannot
disagree about where to put a sign. In a domain where "a wrong sign or missing
phase factor means wrong results" [@agentsmdquantumsim; @nielsen2010quantum],
this is the difference between a simulator that is correct and one that merely
appears correct on small instances.

**Architectural reproducibility.** The compositional gate hierarchy makes the
library reproducible in the strict sense of computational science
[@stodden2016enhancing]: re-running any compatible agent on the same
specification yields a behaviourally equivalent implementation, modulo cosmetic
variation in identifier choice. This is a stronger guarantee than "same input,
same output"---it is "same specification, same architecture."

**Auditability and peer review.** Reviewing the specification is plausible for a
domain expert in a way that reviewing the generated code is not. This addresses
a long-standing concern about the reviewability of research software
[@hinsen2018verifiability].

**Portability across languages and agents.** Because the specification is
language-agnostic and the agent ecosystem is plural, the methodology decouples
three things that are usually entangled: the human author, the implementation
language, and the agent platform. Any combination of the three should produce
the same library. This decoupling is particularly attractive for quantum SDKs,
which historically fragment along language lines [@qiskit2024; @cirq2024;
@bergholm2018pennylane].

## Limitations

The methodology inherits the well-known weaknesses of LLM-based code generation
[@liu2023refining; @jimenez2024swebench]: the agent may hallucinate API calls,
propagate stale assumptions across sections, or silently widen the type system.
These risks are mitigated, but not eliminated, by the strict tier ordering and
the verification agent. The methodology also assumes that the specification
author is willing to invest substantial up-front effort: the currently ~8,900
lines `AGENTS.md` is not a lightweight artefact, and writing it correctly
requires mastery of the underlying physics. Finally, the high-level API's
no-execution testing policy is a deliberate trade-off: it avoids cargo-cult
tests but defers empirical validation to the user, which is acceptable for an
experimental submodule but would be inappropriate for a numerical core.

# Research Impact Statement

The primary scholarly contribution of `@hviana/js-quantum` is methodological: it
demonstrates that a specification-first, agent-driven workflow can produce a
mathematically rigorous quantum computing SDK. The core specification
[@agentsmdquantumsim] and the high-level API specification [@agentsmdhighlevel]
are archived on Zenodo (<https://doi.org/10.5281/zenodo.19473636>), ensuring
long-term availability and citability. The library itself is published on the
JSR registry [@jsquantumjsr] and its source is openly available
[@jsquantumrepo], with over 900 unit and integration tests that serve as
reproducible verification materials. The `AGENTS.md` methodology is designed to
be community-ready: any researcher can feed the specification to a compatible
LLM agent and independently regenerate a behaviourally equivalent
implementation, providing a concrete substrate for reproducibility studies in
agent-assisted scientific software engineering.

# AI Usage Disclosure

Everything in this project---the core library, the high-level API submodule,
both `AGENTS.md` specifications, the documentation, and this article---was built
with the assistance of agents connected to Large Language Models. The human
author wrote and iteratively refined the `AGENTS.md` specifications, which
served as the normative input; LLM coding agents (including Claude Code, Codex,
Cursor, and others listed in the specification) generated the TypeScript source
code, the test suite, and drafts of the documentation and paper text. Quality
and correctness of all AI-generated content were verified through multiple
complementary mechanisms: (i) the strict 16-step build order and tier hierarchy
enforce structural correctness by construction; (ii) a dedicated verification
agent re-reads each specification section against the produced code to detect
drift; (iii) over 900 unit and integration tests validate numerical correctness,
including entrywise matrix comparisons at $\varepsilon = 10^{-10}$; and (iv) the
human author reviewed all generated artefacts against the specification and the
underlying physics before publication.

# Conclusion

We have described the development methodology behind `@hviana/js-quantum`, in
which a pure-TypeScript quantum computing SDK and an experimental high-level API
are generated by LLM coding agents from two `AGENTS.md` specifications. The
methodology's effectiveness rests on two carefully engineered constructs: a
six-part complex-phase convention that eliminates sign and global-phase
ambiguity, and a compositional gate hierarchy that lifts the universality
theorem of Nielsen & Chuang §1.3.2 to exact equality through the adjunction of
an explicit `GlobalPhaseGate`. Together these constructs make the specification
the canonical artefact of the project and turn the implementation into a
derivable consequence. We argue that this "specification-first" discipline is
particularly well suited to scientific software in domains where numerical
correctness is unforgiving, and that the `AGENTS.md` convention provides a
practical, portable substrate for it. Future work includes generating
non-TypeScript realisations of the same specification and evaluating cross-agent
variance under controlled conditions.
