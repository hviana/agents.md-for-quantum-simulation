<p align="center">
<h1 align="center">agents.md for Quantum Simulation</h1>

<p align="center">
  <em>A comprehensive, language-agnostic specification that instructs AI coding agents<br>to build a complete quantum computing SDK from scratch.</em>
</p>

---

## 🧬 What Is This?

This repository contains **[`AGENTS.md`](./AGENTS.md)** — a ~7,700-line
specification file that tells an AI coding agent **exactly** how to build a
complete quantum computing SDK. It is not code. It is a _blueprint for code_.

When you feed this file to an AI agent, it will generate a **fully functional
quantum simulation library** in the programming language of your choice —
including:

| Component                      | Description                                                                  |
| :----------------------------- | :--------------------------------------------------------------------------- |
| ⚛️ **Real Quantum Simulation** | State-vector evolution, Born-rule sampling, mid-circuit measurement          |
| 🧮 **~80 Gate Catalog**        | Tiers 0–14: from Pauli gates to QFT, arithmetic circuits, and oracles        |
| 🔢 **Complex & Matrix Math**   | Built from scratch — no numpy, no BLAS, no external math libraries           |
| 📝 **OpenQASM 3.1 Support**    | Full transpiler: serialize, deserialize, and compile quantum programs        |
| 🏗️ **Compilation Pipeline**    | SABRE layout/routing, basis-gate decomposition (ZYZ, KAK/Weyl), optimization |
| ☁️ **Cloud Backends**          | IBM Quantum (Sampler V2) and qBraid REST API integrations                    |
| 🧪 **900+ Test Suite**         | Unit, integration, and round-trip tests with statistical validation          |

> **No stubs. No approximations. No external quantum backends.**\
> Every operation — tensor products, gate application via subspace iteration,
> state collapse — is implemented from scratch.

---

## 📐 Architecture at a Glance

```
┌──────────────────────────────────────────────────────────────┐
│                     QuantumCircuit                           │
│  qc.h(0).cx(0,1).measure(0,0).measure(1,1)                   │
└──────────────────────┬───────────────────────────────────────┘
                       │
          ┌────────────▼────────────┐
          │       Transpiler        │
          │  ┌────────────────────┐ │
          │  │ OpenQASM 3.1       │ │
          │  │ Serialize/Parse    │ │
          │  ├────────────────────┤ │
          │  │ Compilation        │ │
          │  │ ● Unroll           │ │
          │  │ ● SABRE Layout     │ │
          │  │ ● SABRE Routing    │ │
          │  │ ● Basis Decomp    │ │
          │  │ ● Optimization     │ │
          │  └────────────────────┘ │
          └────────────┬────────────┘
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
  ┌─────────┐   ┌───────────┐   ┌───────────┐
  │Simulator│   │ IBM Cloud │   │  qBraid   │
  │ Backend │   │  Backend  │   │  Backend  │
  └────┬────┘   └─────┬─────┘   └─────┬─────┘
       │              │               │
       ▼              ▼               ▼
    ExecutionResult: { "00": 50, "11": 50 }
```

---

## 🔬 Gate Tier System

The specification defines **~80 quantum gates** organized into 15 tiers of
abstraction, where every gate ultimately decomposes into **single-qubit gates +
CNOT**:

```
Tier 0   ░░░░░░░░░░░░░░░  Single-qubit primitives (I, H, X, Y, Z, RX, RY, RZ, S, T, SX, U, ...)
Tier 1   ░░░░░░░░░░░░░░░  CX — the universal entangling primitive
Tier 2   ░░░░░░░░░░░░░    Controlled gates (CZ, CY, CP, CRX, CH, CU, ...)
Tier 3   ░░░░░░░░░░░      Interaction gates (SWAP, RZZ, RXX, iSWAP, ECR, ...)
Tier 4   ░░░░░░░░░        Three-qubit gates (Toffoli, Fredkin, ...)
Tier 5   ░░░░░░░          Multi-controlled gates (MCX, MCPhase, C3X, C4X, ...)
Tier 6   ░░░░░░           N-qubit composites (Diagonal, Permutation, MCMT, ...)
Tier 7   ░░░░░            Uniformly controlled & unitary synthesis (UCGate, UnitaryGate, ...)
Tier 8   ░░░░             Hamiltonian simulation (PauliEvolution, HamiltonianGate)
Tier 9   ░░░              Quantum Fourier Transform
Tier 10  ░░░              Reversible classical logic (AND, OR, XOR, ...)
Tier 11  ░░               Quantum arithmetic (Adders, Multiplier)
Tier 12  ░░               Function loading (Chebyshev, Reciprocal, ...)
Tier 13  ░                Comparison & oracles (Phase/BitFlip Oracle, ...)
Tier 14  ░                State preparation (GraphState)
```

---

## 🚀 How to Use

### The Basic Idea

1. **Pick an AI coding agent** (see the table below)
2. **Feed it the `AGENTS.md` file**
3. **Tell it which programming language** you want
4. **Let it build the entire SDK**, module by module

The specification is designed so the agent asks you for the target language
first, then follows a strict build order — implementing each module and running
its tests before moving on.

---

## 🤖 Running with AI Coding Agents

### Quick Reference

| Tool                                      | Method                    | Complexity    |
| :---------------------------------------- | :------------------------ | :------------ |
| [Claude Code](#claude-code)               | Automatic via `CLAUDE.md` | ⭐ Easiest    |
| [Claude (Web/App)](#claude-webapp)        | Upload or paste           | ⭐⭐ Easy     |
| [Cursor](#cursor)                         | Project rules             | ⭐⭐ Easy     |
| [Windsurf](#windsurf)                     | Project rules             | ⭐⭐ Easy     |
| [GitHub Copilot](#github-copilot)         | Instructions file         | ⭐⭐ Easy     |
| [Cline](#cline-vs-code)                   | Custom instructions       | ⭐⭐ Easy     |
| [Aider](#aider)                           | Conventions file          | ⭐⭐ Easy     |
| [Roo Code](#roo-code)                     | Instructions file         | ⭐⭐ Easy     |
| [Amazon Q Developer](#amazon-q-developer) | Project context           | ⭐⭐ Easy     |
| [ChatGPT / GPT](#chatgpt)                 | Upload file               | ⭐⭐⭐ Manual |
| [Gemini](#gemini)                         | Upload or paste           | ⭐⭐⭐ Manual |
| [Codex CLI](#codex-cli)                   | Instructions file         | ⭐⭐ Easy     |
| [bolt.new / Lovable / v0](#web-builders)  | Paste in prompt           | ⭐⭐⭐ Manual |

---

### Claude Code

> **Easiest method** — Claude Code natively supports `AGENTS.md` via the
> `CLAUDE.md` pointer.

This repo is already configured. Just clone and run:

```bash
git clone https://github.com/nicedoc/agents.md-for-quantum-simulation.git
cd agents.md-for-quantum-simulation
claude
```

Claude Code will automatically read `CLAUDE.md` → which references `@AGENTS.md`
→ and follow the full specification.

**How it works:**

- `CLAUDE.md` contains `@AGENTS.md`, which tells Claude Code to load the
  specification
- The agent will ask you for your target programming language
- Then it builds everything step by step

> 💡 **Tip:** You can also use Claude Code in **VS Code** or **JetBrains** via
> the official extensions.

---

### Claude Web

For **claude.ai** or the **Claude desktop/mobile apps**:

1. Start a new conversation
2. **Attach** the `AGENTS.md` file (📎 button), or paste its contents
3. Send your first message:

```
I've attached a specification file (AGENTS.md) for building a quantum computing 
SDK. Please read it completely and then follow the implementation plan. 
I want to use TypeScript as the target language.
```

> ⚠️ **Context window:** The spec is ~7,700 lines. Claude's large context window
> handles this well. If you hit limits, you can split the conversation across
> sessions, referencing specific sections.

---

### Cursor

**Option A — Project Rules (recommended):**

1. Open the project folder in Cursor
2. Go to **Settings → Rules** (or create `.cursor/rules`)
3. Add a rule file that references the spec:

```
Read and follow the complete specification in AGENTS.md. 
It defines a quantum computing SDK to build from scratch.
Ask me for the target programming language before writing any code.
```

4. Use **Composer** (Cmd/Ctrl+I) or **Chat** to start:

```
Follow the AGENTS.md specification. Let's begin with Step 1.
```

**Option B — Direct in chat:**

1. Open `AGENTS.md` in the editor
2. Select all content → open Chat (Cmd/Ctrl+L)
3. The file is automatically included as context

---

### Windsurf

1. Open the project in Windsurf
2. Create `.windsurfrules` in the project root:

```
Read and follow the AGENTS.md specification completely.
It defines a quantum computing SDK. Follow the build order in Section 8.
Ask me for the target programming language first.
```

3. Open **Cascade** and say:

```
Follow the AGENTS.md specification in this project. Let's start building.
```

> Windsurf's Cascade can read project files directly, so it will pick up the
> full spec.

---

### Github Copilot

**For Copilot Chat in VS Code / JetBrains:**

1. Create `.github/copilot-instructions.md`:

```markdown
Read and follow the complete specification in AGENTS.md at the project root. It
defines a quantum computing SDK to implement from scratch. Follow the exact
build order in Section 8.2. Ask the user for the target programming language
before writing any code.
```

2. Open Copilot Chat and reference the file:

```
@workspace Read AGENTS.md and follow the implementation plan. 
Let's start with Step 1: types module.
```

**For GitHub Copilot Coding Agent (in GitHub):**

1. Open an issue referencing the spec:

```
Implement the quantum computing SDK as specified in AGENTS.md using TypeScript.
Start with Step 1 (types) and follow the build order in Section 8.2.
```

2. Assign the Copilot coding agent to the issue

---

### Cline VS Code

1. Install the **Cline** extension in VS Code
2. Open Settings → **Custom Instructions** and add:

```
Always read and follow the AGENTS.md file in the project root.
It is a complete specification for a quantum computing SDK.
Follow the build order in Section 8.2. Run tests after each module.
```

3. In the Cline panel:

```
Read AGENTS.md and start implementing the quantum SDK. 
I want to use Python. Begin with Step 1: types module.
```

> Cline has full file system access and can run tests automatically after each
> module.

---

### Aider

1. Create a `.aider.conf.yml` or use the `--read` flag:

```bash
aider --read AGENTS.md
```

2. Or add to `.aider.conf.yml`:

```yaml
read:
  - AGENTS.md
```

3. Start aider and give your instruction:

```
Follow the AGENTS.md specification to build a quantum computing SDK in Python. 
Start with Step 1: implement the types module in src/types.py.
```

> 💡 **Tip:** Use `--model` to pick your preferred LLM backend with Aider.

---

### Roo Code

1. Create `.roo/rules` or `.roocode/rules` in the project root with a markdown
   file:

```markdown
# Project Rules

Read and follow the complete specification in AGENTS.md. It defines a quantum
computing SDK to build from scratch. Follow the build order in Section 8.2
exactly. Run tests after implementing each module before moving to the next.
```

2. In the Roo Code panel:

```
Read AGENTS.md and implement the quantum SDK in Rust. Start with Step 1.
```

---

### Amazon Q Developer

1. Open the project in your IDE with **Amazon Q Developer** installed
2. In the Q Chat panel, reference the spec:

```
Read the file AGENTS.md in this project. It contains a complete specification 
for building a quantum computing SDK. Follow it to implement the SDK in Python.
Start with Step 1 from Section 8.2.
```

> Amazon Q can read project files when you reference them in chat.

---

### ChatGPT

**For ChatGPT (Web/App):**

1. Start a new conversation
2. **Attach** `AGENTS.md` using the 📎 button
3. Send:

```
I've uploaded a specification file called AGENTS.md. It defines a complete 
quantum computing SDK to build from scratch. Please read it entirely, then 
follow the implementation plan in Section 8. I want to use TypeScript.
```

**For OpenAI Codex CLI:**

```bash
# Set up the instructions
echo "Read and follow AGENTS.md completely." > codex-instructions.md
codex --instructions codex-instructions.md
```

> ⚠️ **Note:** Due to context limits, ChatGPT may need you to work section by
> section. Start with "Read Sections 1-3" then proceed incrementally.

---

### Codex CLI

1. Create `AGENTS.md` instructions file (already present in this repo)
2. Use the `--instructions` flag or `codex.md`:

```bash
# Option A: instructions flag
codex --instructions AGENTS.md "Build the quantum SDK in TypeScript, start with Step 1"

# Option B: Create codex.md pointing to the spec
echo "Follow the specification in AGENTS.md exactly." > codex.md
codex
```

---

### Gemini

**For Gemini (Web/App):**

1. Start a new conversation
2. **Upload** `AGENTS.md`
3. Send:

```
I've uploaded a detailed specification file. It defines a complete quantum 
computing SDK. Read it fully, then implement it step by step in Python 
following the build order in Section 8.2.
```

**For Gemini in Google AI Studio / Vertex AI:**

1. Add `AGENTS.md` as system instructions or context
2. Start the conversation with your language choice

**For Gemini Code Assist (IDE):**

1. Reference the file in your IDE's Gemini chat
2. Ask it to follow the specification

> 💡 Gemini's large context window (1M+ tokens) can handle the entire spec in
> one pass.

---

### Web Builders

For **web-based AI code generators** (bolt.new, Lovable, v0, Replit Agent,
etc.):

1. Copy the contents of `AGENTS.md`
2. Paste it into the prompt / system instructions area
3. Add your instruction:

```
Follow this specification to build a quantum computing SDK in TypeScript.
Start with the types module (Step 1) and proceed through all 16 steps.
```

> ⚠️ These tools may have smaller context windows. Consider splitting the spec
> into sections and feeding them incrementally.

---

## 💡 Tips for Best Results

### 🎯 Prompting Strategy

| Strategy                           | Description                                          |
| :--------------------------------- | :--------------------------------------------------- |
| **Be explicit about the language** | "Use TypeScript" / "Use Python" / "Use Rust"         |
| **Follow the build order**         | The spec defines Steps 1–16. Go in order.            |
| **Run tests after each module**    | The spec requires this. Remind the agent.            |
| **Reference sections directly**    | "Implement the gates from Section 3, Tier 0"         |
| **Split if needed**                | For smaller context windows, work section by section |

### 🧩 Incremental Approach (for any agent)

If the agent can't process the full spec at once:

```
Step 1: "Read Sections 1–2 of AGENTS.md (Overview & Conventions)"
Step 2: "Read Section 3, Tiers 0–1 (Single-qubit gates & CX)"
Step 3: "Now implement src/complex.{ext} and src/matrix.{ext}"
Step 4: "Read Section 3, Tiers 2–5 and implement src/gates.{ext}"
...continue through all sections...
```

### 📏 Spec Size Reference

| Metric           | Value                     |
| :--------------- | :------------------------ |
| Total lines      | ~7,760                    |
| Total sections   | 13                        |
| Gate definitions | ~80 gates across 15 tiers |
| Required tests   | 900+                      |
| Estimated tokens | ~60,000–80,000            |

---

## 📁 Repository Structure

```
.
├── 📋 AGENTS.md          # The full specification (this is the star of the show)
├── 📎 CLAUDE.md           # Pointer file for Claude Code (@AGENTS.md)
├── 📖 README.md           # You are here
└── ⚖️  LICENSE             # MIT License
```

After an agent finishes building, the repo will look like:

```
.
├── AGENTS.md
├── CLAUDE.md
├── README.md (updated with API docs)
├── LICENSE
├── src/
│   ├── types.{ext}
│   ├── complex.{ext}
│   ├── matrix.{ext}
│   ├── gates.{ext}         # ~80 gates, Tiers 0–14
│   ├── expansion.{ext}     # OpenQASM 3.1 Expansion API
│   ├── parameter.{ext}
│   ├── circuit.{ext}       # QuantumCircuit builder
│   ├── transpiler.{ext}    # OpenQASM 3.1 transpiler + compiler
│   ├── simulator.{ext}     # State-vector simulation engine
│   ├── backend.{ext}
│   ├── ibm_backend.{ext}   # IBM Quantum cloud
│   ├── qbraid_backend.{ext}# qBraid cloud
│   ├── bloch.{ext}
│   └── mod.{ext}
├── tests/
│   ├── complex.test.{ext}
│   ├── matrix.test.{ext}
│   ├── gates.test.{ext}
│   ├── ... (900+ tests)
│   └── integration.test.{ext}
└── {build config}
```

---

## 🤔 FAQ

<details>
<summary><strong>What programming languages does this support?</strong></summary>

Any language. The spec is language-agnostic. It has been designed to work with
TypeScript, Python, Rust, Go, Java, C#, and others. The agent adapts idioms,
naming conventions, and tooling to your chosen language.

</details>

<details>
<summary><strong>How long does it take to generate?</strong></summary>

It depends on the agent and the language. With a capable agent like Claude Code,
expect several hours of autonomous work. The spec is large and detailed — that's
by design.

</details>

<details>
<summary><strong>Does the generated code actually simulate quantum circuits?</strong></summary>

Yes. The simulator uses real state-vector evolution over complex vector spaces
with subspace iteration for gate application and Born-rule sampling for
measurement. It's a real quantum simulator — not a mock or stub.

</details>

<details>
<summary><strong>Can I use this to run circuits on real quantum hardware?</strong></summary>

Yes. The spec includes IBM Quantum and qBraid backend integrations. After
generating the SDK, configure your API credentials and submit circuits to real
quantum processors.

</details>

<details>
<summary><strong>Why is the spec so long?</strong></summary>

Precision. Every gate matrix, every decomposition, every phase convention, every
edge case is specified explicitly. This eliminates ambiguity and ensures the
generated code is mathematically correct. Quantum computing is unforgiving — a
wrong sign or missing phase factor means wrong results.

</details>

<details>
<summary><strong>What is the Phase Convention section about?</strong></summary>

Quantum gates that differ only by a "global phase" are physically equivalent in
isolation — but that phase becomes observable when you add controls. The Phase
Convention (Section 2) ensures every gate, decomposition, and equality check
handles phase consistently across the entire library. It's the mathematical
backbone.

</details>

<details>
<summary><strong>What if my AI agent can't handle the full file?</strong></summary>

Work incrementally. Feed it section by section, following the build order in
Section 8.2. Each module depends only on previous ones, so you can build step by
step.

</details>

---

## 📜 License

[MIT](./LICENSE) — Henrique Emanoel Viana

---

<p align="center">
  <sub>Built with ⚛️ and a very detailed specification.</sub>
</p>
