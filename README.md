```
   ____ ____ _   _ _____               
  / ___|/ ___| | | |  ___|   _ ________
 | |  _| |  _| | | | |_ | | | |_  /_  /
 | |_| | |_| | |_| |  _|| |_| |/ / / / 
  \____|\____|\___/|_|   \__,_/___/___|
```

# GGUFuzz — AI Model Supply Chain Attack Framework

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Tests](https://img.shields.io/badge/tests-29%20passing-brightgreen.svg)]()
[![CVEs](https://img.shields.io/badge/CVEs-7%20weaponized-red.svg)]()
[![Black Hat Arsenal](https://img.shields.io/badge/Black%20Hat-Asia%202027%20Arsenal-black.svg)]()

> **First open-source offensive security tool, to our knowledge, targeting the AI model file supply chain — weaponizing GGUF/Safetensors/ONNX parser vulnerabilities in llama.cpp, Ollama, LM Studio, and every local AI deployment.**

---

## The Problem

The AI community moved from pickle to GGUF/safetensors thinking they solved the supply chain problem. **They didn't.**

llama.cpp — the inference engine behind Ollama, LM Studio, GPT4All, and most local AI — has had **7 memory corruption CVEs in 2025-2026**, all in its GGUF parser:

| CVE | Target | Bug | CVSS |
|---|---|---|---|
| CVE-2026-34159 | llama.cpp RPC | Arbitrary R/W, no bounds check | 9.8 |
| CVE-2025-32444 | vLLM | Pickle deserialization RCE | 10.0 |
| CVE-2026-43629 | llama.cpp | KV-cache heap overflow | High |
| CVE-2026-33298 | llama.cpp GGUF | Heap overflow in parser | High |
| CVE-2026-27940 | llama.cpp GGUF | Heap overflow (bypasses prior fix) | High |
| CVE-2025-49847 | llama.cpp GGUF | OOB read in tensor validation | High |
| CVE-2026-21869 | llama.cpp | OOB write via negative n_discard | High |

### The Attack

```
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │  WEAPONIZE   │───►│  DISTRIBUTE  │───►│   DELIVER    │───►│   EXPLOIT    │
  │              │    │              │    │              │    │              │
  │ Craft GGUF   │    │ Upload to    │    │ User runs    │    │ Parser bug   │
  │ with CVE     │    │ HuggingFace  │    │ ollama pull  │    │ triggers     │
  │ trigger      │    │ or Ollama    │    │ or download  │    │ Code exec    │
  └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

**Scale:** HuggingFace hosts **800,000+ GGUF models**, downloaded millions of times daily. No tool existed to test parser security or generate adversarial model files.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       GGUFuzz CLI                               │
│    fuzz · weaponize · supply-chain · validate · list-cves · demo│
├───────────┬───────────┬───────────┬───────────┬─────────────────┤
│  FORMAT   │  FUZZER   │ WEAPONIZE │  SUPPLY   │   REPORTER      │
│  ENGINE   │           │           │  CHAIN    │                 │
│           │           │           │           │                 │
│ GGUF v3   │ 12 GGUF   │ CVE-2025- │ 5-phase   │ JSON reports    │
│ Builder   │ mutations │ 32444     │ attack    │ Markdown        │
│ Safetensors│ 3 ST     │ CVE-2025- │ chain     │ reports         │
│ Builder   │ mutations │ 49847     │ Ollama    │ Terminal        │
│ ONNX      │ 6 ONNX   │ CVE-2026- │ Hugging   │ summary         │
│ Builder   │ mutations │ 21869     │ Face      │ Crash           │
│           │ Coverage- │ CVE-2026- │ Model     │ dedup           │
│ All 27    │ guided    │ 27940     │ Scope     │ Exploitability  │
│ metadata  │ AFL-style │ CVE-2026- │           │ rating          │
│ types     │ bitmap    │ 33298     │ Social    │                 │
│           │ Corpus    │ CVE-2026- │ eng.      │                 │
│           │ mgmt      │ 34159     │ lures     │                 │
│           │ Parallel  │ CVE-2026- │           │                 │
│           │ exec      │ 43629     │           │                 │
├───────────┴───────────┴───────────┴───────────┴─────────────────┤
│                   CRASH CLASSIFICATION                          │
│    ASAN Parsing · Signal Detection · Exploitability Rating      │
│    SHA-256 Dedup · EXPLOITABLE → NOT_EXPLOITABLE spectrum       │
└─────────────────────────────────────────────────────────────────┘
```

### Module Breakdown

```
GGUFuzz/
├── ggufuzz/
│   ├── formats.py         # GGUF/Safetensors/ONNX file builders (pure stdlib)
│   ├── mutators.py        # Structure-aware mutations (12 GGUF + 3 ST + 6 ONNX)
│   ├── fuzzer.py          # Subprocess-based fuzzer with crash classification
│   ├── coverage.py        # AFL-style coverage-guided fuzzer (bitmap, corpus mgmt)
│   ├── weaponizer.py      # CVE-specific weaponized model generators (7 CVEs)
│   ├── supply_chain.py    # Supply chain attack simulation (5 phases)
│   ├── reporter.py        # JSON/Markdown/terminal crash reports
│   └── cli.py             # Full CLI interface
├── modules/               # Additional attack modules
├── payloads/              # Pre-built weaponized model files
├── demo_output/           # Validated demo artifacts
├── tests/
│   └── test_ggufuzz.py    # 29 test cases
└── setup.py
```

---

## CVE Weaponization

Each weaponizer crafts a **valid-looking GGUF file** that passes initial header checks but triggers the vulnerability during deeper parsing:

| CVE | Crafted Trigger | Parser Effect |
|---|---|---|
| CVE-2025-32444 | Oversized metadata string | Heap buffer overflow |
| CVE-2025-49847 | Invalid tensor dimensions | Out-of-bounds read |
| CVE-2026-21869 | Negative `n_discard` value | Out-of-bounds write |
| CVE-2026-27940 | Heap overflow in new code path | Bypasses prior CVE-2025-53630 fix |
| CVE-2026-33298 | Integer overflow in `ggml_nbytes` | Memory corruption |
| CVE-2026-34159 | Malformed quantization params | Memory corruption |
| CVE-2026-43629 | Crafted KV-cache state | State restore exploit |

---

## Structure-Aware Fuzzing

| Format | Mutations | Targets |
|---|---|---|
| **GGUF** | 12 mutations | Magic, version, tensor count, metadata count, tensor dims, offset, dtype, string length, metadata type, overlapping tensors, empty file, alignment |
| **Safetensors** | 3 mutations | Header length, JSON header, truncation |
| **ONNX** | 6 mutations | IR version, tensor dims, data type, raw data size, protobuf lengths, truncation |

**Coverage-guided mode:** AFL-style 64KB bitmap, energy-based corpus scheduling, corpus minimization, parallel execution across multiple workers.

### Crash Classification

```
EXPLOITABLE          ← Heap overflow, arbitrary write, use-after-free
PROBABLY_EXPLOITABLE ← Stack corruption, OOB access near heap
UNKNOWN              ← Unclassified crash
NOT_EXPLOITABLE      ← Null deref, assertion failure
```

ASAN report parsing, signal-based classification (SIGSEGV/SIGABRT/SIGFPE/SIGBUS), SHA-256 deduplication.

---

## Usage

### Installation

```bash
git clone https://github.com/p17433308-cmyk/GGUFuzz-Arsenal.git
cd GGUFuzz-Arsenal
pip install -e .
```

### Quick Start — Full Demo

```bash
ggufuzz demo
```

Constructs real model files, runs mutations, generates weaponized payloads — no target binary needed.

### Individual Modules

**Fuzzing:**
```bash
# Fuzz a GGUF parser
ggufuzz fuzz --format gguf --target "llama-cli --model {input}" --output crashes/

# Coverage-guided fuzzing (parallel)
ggufuzz fuzz --format gguf --target "llama-cli --model {input}" --coverage --workers 4
```

**CVE Weaponization:**
```bash
# Generate weaponized model for a specific CVE
ggufuzz weaponize --cve CVE-2026-33298 --output evil.gguf

# Generate all weaponized models
ggufuzz weaponize --cve all --output payloads/
```

**Supply Chain Simulation:**
```bash
# Simulate full supply chain attack
ggufuzz supply-chain --cve CVE-2026-33298 --registry ollama --output sim/
```

**Model Validation:**
```bash
# Scan a model file for vulnerability patterns
ggufuzz validate --file model.gguf
```

**CVE Database:**
```bash
# List all CVEs in database
ggufuzz list-cves
```

---

## Supply Chain Attack Simulation

5-phase attack chain targeting real AI model registries:

| Phase | Action | Output |
|---|---|---|
| 1. Weaponization | Craft GGUF with CVE trigger | Malicious `.gguf` file |
| 2. Distribution | Generate fake HuggingFace/Ollama repo | Repository structure + model card |
| 3. Delivery | Social engineering lures | "SOTA model, 4-bit quantized, fastest inference" |
| 4. Exploitation | User loads model → parser bug triggers | Memory corruption → code execution |
| 5. Impact | Assessment | Exploitability rating + report |

---

## Prior Art

| Tool | Scope | GGUFuzz Advantage |
|---|---|---|
| Pickle RCE research | Different format | GGUF-specific, parser-level attacks |
| AFL/libFuzzer on llama.cpp | Coverage-guided, no semantic awareness | Grammar-aware + CVE-targeted |
| HuggingFace scanning | Checks for pickle, not GGUF bugs | GGUF parser vulnerability detection |
| safetensors (format) | "Safe" spec | Tests parser implementation, not spec |

---

## Dependencies

**None.** Pure Python standard library. Zero external packages.

## Tests

```bash
pytest tests/ -p no:cacheprovider    # 29 tests: builders, mutators, weaponizer, coverage, reporter
```

---

## Authors

- **Dr. Sapna V M** — PES University
- **Prajwal Chowdary M** — PES University
- **Prasad H B** — PES University

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

## Disclaimer

GGUFuzz is designed for **authorized security testing, AI security research, and educational purposes only**. All CVE weaponizers target already-patched vulnerabilities. This tool does not upload malicious models to public repositories. Users are responsible for ensuring compliance with all applicable laws.
