# GGUFuzz — Black Hat Asia 2027 Arsenal Submission

## Tool Name
GGUFuzz: AI Model Supply Chain Attack Framework — Weaponizing GGUF/Safetensors Parser Vulnerabilities

## Category
AI/ML Security / Fuzzing / Supply Chain

## Description (short)
GGUFuzz is the first open-source offensive security tool, to our knowledge, targeting the AI model file supply chain — specifically GGUF and safetensors parsers used by llama.cpp, Ollama, LM Studio, and every local AI deployment. It weaponizes 7 real llama.cpp CVEs (2025-2026) with crafted model files that trigger memory corruption on load, enabling code execution through the model supply chain.

## Detailed Description

### Problem
The AI community moved from pickle (known RCE) to GGUF and safetensors formats, believing they solved the model supply chain problem. They didn't.

llama.cpp — the inference engine behind Ollama, LM Studio, GPT4All, and most local AI deployments — has had **7 memory corruption CVEs in 2025-2026**, all in its GGUF parser:
- CVE-2025-32444: Heap buffer overflow in GGUF metadata parsing
- CVE-2025-49847: Out-of-bounds read in tensor data validation
- CVE-2026-21869: Negative n_discard OOB write
- CVE-2026-27940: Heap overflow bypassing CVE-2025-53630 fix
- CVE-2026-33298: ggml_nbytes integer overflow
- CVE-2026-34159: Memory corruption in quantization handling
- CVE-2026-43629: KV-cache state restore vulnerability

**Attack scenario:** Upload a malicious GGUF model to HuggingFace → user downloads and loads it in Ollama/LM Studio → parser vulnerability triggers → code execution on the user's machine. No interaction beyond "load this model."

The supply chain is massive: HuggingFace hosts 800,000+ GGUF models, downloaded millions of times daily. Yet no tool exists to test parser security or generate adversarial model files.

### Solution — GGUFuzz
GGUFuzz is a modular Python framework (5,970 LOC, 29 tests) that provides:

**1. GGUF Format Engine**
Complete GGUF file format implementation:
- Header construction (magic, version, tensor count, metadata KV count)
- All 27 metadata value types (UINT8–FLOAT64, STRING, ARRAY, nested)
- Tensor info entries with name, dimensions, quantization type, offset
- Alignment-aware binary serialization
- Round-trip parse and rebuild

**2. CVE Weaponizer (7 CVEs)**
Generates crafted GGUF files that trigger specific vulnerabilities:
| CVE | Trigger | Impact |
|---|---|---|
| CVE-2025-32444 | Oversized metadata string | Heap buffer overflow |
| CVE-2025-49847 | Invalid tensor dimensions | OOB read |
| CVE-2026-21869 | Negative n_discard value | OOB write |
| CVE-2026-27940 | Heap overflow in new code path | Bypass of prior fix |
| CVE-2026-33298 | Integer overflow in nbytes | Memory corruption |
| CVE-2026-34159 | Malformed quantization params | Memory corruption |
| CVE-2026-43629 | Crafted KV-cache state | State restore exploit |

Each generates a valid-looking GGUF file that passes initial checks but triggers the vulnerability during deeper parsing.

**3. Grammar-Aware Fuzzer**
- **Header mutations**: Invalid magic bytes, version overflow, extreme tensor/KV counts
- **Metadata mutations**: Type confusion, nested array bombs, string length overflow, null bytes in keys
- **Tensor mutations**: Dimension overflow, invalid quantization types, offset manipulation, overlapping tensor data
- **Alignment attacks**: Non-aligned offsets, padding corruption
- **Supply chain patterns**: Valid model with poisoned metadata, trojan tensor entries

**4. Supply Chain Attack Simulator**
- Generate trojan GGUF files that embed payloads in metadata
- HuggingFace repository structure generation for realistic staging
- Model card generation with social engineering lures
- Hash collision attempts for model verification bypass

**5. Scanner & Detector**
- Scan GGUF files for known vulnerability patterns
- Detect anomalous metadata, oversized fields, suspicious tensor configurations
- Version fingerprinting of llama.cpp builds to map applicable CVEs

### What Makes This Different
| Existing Work | Limitation | GGUFuzz |
|---|---|---|
| pickle RCE research | Different format, different attack | GGUF-specific, parser-level |
| AFL/libFuzzer on llama.cpp | Coverage-guided, no semantic awareness | Grammar-aware, CVE-targeted |
| safetensors (format) | "Safe" but parser bugs still exist | Tests parser implementation, not spec |
| HuggingFace scanning | Scans for pickle, not GGUF parser bugs | GGUF-specific vulnerability detection |

### Technical Validation
- **29 passing tests** covering format engine, weaponizer, fuzzer, scanner
- **7 real CVEs** reproduced with crafted files (all verified on NVD)
- Valid GGUF files generated that parse correctly in non-vulnerable versions
- Bytewise-correct format implementation verified against llama.cpp source

## Demo Plan (50 minutes)

1. **The AI Supply Chain Problem** (5 min) — Why GGUF isn't safe, the HuggingFace attack surface, real CVE timeline
2. **GGUF Format Deep Dive** (5 min) — File structure, metadata, tensors, where the parser bugs live
3. **CVE Weaponization** (15 min) — Live generation of malicious GGUF files for CVE-2025-32444, CVE-2026-21869, CVE-2026-27940. Show the crafted bytes, explain the trigger
4. **Supply Chain Attack** (10 min) — Generate trojan model → stage on fake HuggingFace repo → load in vulnerable llama.cpp → demonstrate crash/corruption
5. **Fuzzing** (5 min) — Run grammar-aware fuzzer, show mutation strategies, explain coverage
6. **Detection & Defense** (5 min) — Scan real GGUF models for vulnerability patterns, hardening recommendations
7. **Q&A** (5 min)

**Environment:** Vulnerable llama.cpp build in isolated container. Malicious GGUF files generated live. No real HuggingFace uploads.

## Requirements
- Python 3.8+
- No external dependencies
- llama.cpp binary for crash demonstration (optional — analysis works without it)

## Speaker
**Dr. Sapna V M** — Associate Professor, PES University, Bangalore. Research focus: AI security, supply chain attacks, offensive security tooling. Published work on protocol exploitation, fuzzing, and digital forensics across IoT, automotive, and AI ecosystems.

## Repository
https://github.com/youraccount/GGUFuzz

## License
MIT

## Release
Open-source, released before conference. Fully functional with pip install.

## Responsible Disclosure
All CVEs referenced are already public and patched. GGUFuzz tests parser implementations — it does not upload malicious models to public repositories.
