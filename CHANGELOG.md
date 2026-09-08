# Changelog

All notable changes to **reverse-skill** are documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

## [Unreleased]
### Added
- **CI runs remaining unwired suites** — `test-p0-friction.ps1` on the Windows leg of `routing-tests` (Windows PowerShell 5.1); `case-review/tests/test_review_case.py` in the Linux `case-contract` job. `test-workflow-title-safety.ps1` was already wired.
- **Binary Ninja route and skill** — added `binary-ninja-reverse` for HLIL/MLIL/LLIL, Python API, and an explicitly enabled loopback community MCP bridge; Binary Ninja remains a manual commercial dependency.
- **Optional Codex adapter plugin** — added `plugins/reverse-skill/` without changing the client-neutral core or auto-registering MCP servers.
- **Benchmark coverage for R40 (case-evidence-review)** — the route had a single English case; added three cases covering its previously-untested branches (Chinese `证据链`/`证据图`/`可追溯性`/`案件审查`/`案例审计`, and English `evidence.?graph`/`fixity.?check`/`case.?audit`), each verified through both the PowerShell and Bash routers.

### Fixed
- **IDA MCP HTTP stall** — `run-supervisor.py` patches stock `idalib_supervisor` onto `ThreadingHTTPServer` and accepts Streamable HTTP GET `/mcp` (patch failure is skipped, supervisor still starts). Keep-alive deadlock is **time since last healthy `tools/list`**, not process `CreationDate`; `open.ps1` holds `opening.lock` so in-flight opens are never `-Force`d. New `recover.ps1` is an immediate `-Force` path. Never `taskkill`s `ida.exe`.
- **Broken internal doc links + guard** — removed a dangling `phishing-case-study.md` reference and redirected the missing payloader `反弹shell.md` index entry to its tracked raw data. Added `skills/scripts/verify-doc-links.py`, wired into CI, so internal Markdown links are checked from Git index blobs even when Defender quarantines a working-tree payload file.
- **Windows PowerShell 5.1 encoding** — added a UTF-8 BOM to five non-ASCII `.ps1` scripts (`skills/scripts/verify-doc-facts.ps1`, `apk-reverse/scripts/frida-run.ps1`, `apk-reverse/scripts/rebuild-sign-install.ps1`, `ida-reverse/scripts/start.ps1`, `radare2/scripts/recon.ps1`). Without a BOM, PS 5.1 parses these files as the system ANSI codepage and garbles their Chinese / em-dash string literals; `verify-doc-facts.ps1` was failing four checks under 5.1 (CI only ran it under `pwsh`, which defaults to UTF-8). CI now guards every non-ASCII `.ps1` for a BOM.
- **Evidence-consolidation test fixed + two suites wired into CI** — `test-consolidate-evidence.ps1` printed its success marker but leaked exit 1: it ran `review_case.py --verify-hashes` on a case whose consolidation had (by design) rewritten `E-001.md`, so hash fixity could never pass. Dropped `--verify-hashes`, added an explicit exit-code assertion, and wired both it and `test-bootstrap-codex-encoding.ps1` into the Windows leg of `routing-tests` (both shipped in the repo but were never run by CI).

### Changed
- **Coherence clamp (identity-preserving)** — `RULES.md` hot path is `master-route` → `case-init` → PRIMARY. `routing.json` remains the only route table; `MASTER-ROUTING.md` priority order is verified against JSON. `routing.md` is advisory. `precedent-auth.md` no longer grants auth.
- **IDA open** — lock files may force a temp copy; `.i64` / `.idb` are never deleted.
- **IDA MCP keep-alive** — `start.ps1` reuses a healthy HTTP server, launches `idalib_supervisor` via windowless Python, never `taskkill`s `ida.exe` (no `/T`). A listening 13337 with `tools/list` timeout is treated as busy, not dead, so the 1-minute watchdog cannot kill a supervisor mid-`idb_open`. `open.ps1` talks ida-pro-mcp 2.x `idb_open`/`idb_list`.
- **IDA discovery** — `ToolDiscovery.ps1` now catalogs `idalib-mcp`, `ida-pro-mcp`, and `ida` with Program Files + per-user Python fallbacks.

### Added
- `ida-reverse` watchdog / scheduled-task installer / GUI launcher / supervisor wrapper (`watchdog.ps1`, `install-autostart.ps1`, `start-gui.ps1`, `run-supervisor.py`) plus portable `LOCAL-SETUP.md`.

## [1.0.1] — 2026-08-08
### Added
- **Routing single source of truth** — `skills/config/routing.json` (R0–R39 keyword rules with `must` / `mustAll` / `exclude` semantics). `master-route.ps1` now reads this file; hardcoded routing tables removed from scripts. Routing knowledge lives in one place.
- **Routing regression benchmark** — `skills/tests/routing-benchmark.json` (163 bilingual cases, 40 quick) + `skills/scripts/test-routing.ps1` runner. Any routing change must keep the benchmark green.
- **Routing keyword coverage expansion** (benchmark-driven): burp suite family, pcap/wireshark, root-detection/certificate-pinning, buffer overflow, `.so`/native/JNI, go binaries (中文), js-encrypt, webshell, privilege escalation, S3/object storage, memory dump, incident response, Bluetooth/BLE, USB, Unity/game reverse, security assessment, and more.
- **Supply-chain pin gate** — `verify-routing-coherence.ps1` now fails on any auto-install capability lacking `pinnedVersion` / `pinnedCommit` / `pinPolicy` / asset hash. Pinned: frida-tools 14.10.4, pwntools 4.15.0, agent-browser 0.31.1, ida-pro-mcp @commit, SecLists/ProxyCat @commit, nuclei v3.9.0; winget sources annotated with `winget-latest` policy.
- **Client-neutral integration contract** — routing, tests, manifests, and case workflows remain independent of Claude Code, Codex, Cursor, OpenCode, or any other client; client adapters are optional and must not define repository identity.
- **Skill navigation index** — `skills/INDEX.md` auto-generated from SKILL.md frontmatter by `extract-summaries.ps1` (`-Check` mode for CI drift detection).
- **CI pipeline** — `.github/workflows/ci.yml`: Windows + Ubuntu matrix (PowerShell shim for Linux) running test-routing / verify / smoke / INDEX check / JSON validation, plus `bash -n` syntax checks.
- **Example case** — `examples/ctf-demo/` full workflow walkthrough (route → scope gate → timeline → evidence → report).
- **frontmatter completion** — `dsl-vm-reverse/SKILL.md` gained name/description frontmatter (was the only module missing it).
- **README refresh** — updated the multilingual project overview, release badge, current capabilities, and sponsor showcase layout.
- `case-review/`: read-only Evidence Graph Review with scope, timeline, work item, Finding, Path, and optional SHA-256 fixity checks
- Domain skills R21–R27, R29–R30: `protocol-reverse`, `ghidra-reverse`, `cloud-k8s`, `windows-ad`, `digital-forensics`, `code-audit`, `threat-hunting`, `wifi-wireless`, `browser-extension-reverse`
- High-quality skills R28, R31–R38: `ot-ics`, `macos-reverse`, `thick-client`, `go-rust-reverse`, `hardware-security`, `database-security`, `email-security`, `identity-federation`, `radio-sdr`
- Wired into `MASTER-ROUTING.md`, `master-route.ps1`, routing tables, domain map, role-map, coherence tests

### Removed
- `game-reverse/` (not a product focus; Unity/IL2CPP remains via `reverse-engineering` + seed-014)

### Fixed
- **Upstream mixed-EOL files** — 3 markdown files committed with CRLF while `.gitattributes` declares `*.md eol=lf`; normalized to LF so `git status` stays clean on fresh clones.
- Routing: sigma vs malware, LLM 越狱 vs iOS 越狱, 完整渗透/打到域控 vs AD 域控, forensics vs OT ics; master-route.ps1 rewritten UTF-8 BOM for PS 5.1 CJK
- Linux/macOS bootstrap: register PentestSwarm MCP with a verified executable path after Go install or when already installed

### Security
- Core scripts do not write client-global instruction files; client-specific integration remains outside the routing core.
- Added `docs/PACKAGE-SECURITY-AUDIT.md`: static audit of package executables (no backdoor / no auto DB wipe found)
- Pin supply-chain floating tags: jshook `@0.3.4`, pentestswarm `v0.1.0`
- Bootstrap integrity: GitHub zip/jar downloads verify `assetSha256` (manifest) or GitHub API `digest`; mismatch deletes file and fails
- Pin jadx `v1.5.6` and apktool `v3.0.2` with published SHA256
- Remove shell evaluation from Kali user-home resolution and pass Frida hosts through argument arrays
- Write the Burp MCP token atomically with owner-only permissions on POSIX filesystems
- Reconnect the Burp MCP bridge when Burp starts after the bridge and parse one stdio message per line
- Enable authentication for bootstrapped Anything Analyzer MCP servers and register the bearer token with supported clients
- Stop each stale IDA MCP process individually before starting a replacement
- Add Bash case initialization, authorization guard, and a structured router that reads the same `routing.json` as PowerShell
- Verify Bash routing parity in CI without introducing a client-specific plugin manifest
- Enforce immutable Kali/Windows bootstrap sources for Frida, IDA MCP, Agent Browser, ProxyCat, Nuclei, and pwntools
- Reject path-like Bash case names and scope authorization checks to the contract's auth/network/signoff sections
- Align Bash network defaults with PowerShell: authorized URLs use `authorized_target_only`, while offline readiness requires an explicit local sample
- Pin GitHub Actions checkout to the immutable v4.2.2 commit and keep the CI case count synchronized
- Create a functional Kali `proxycat` wrapper after installing the pinned source checkout
- Scope PowerShell authorization fields to their contract sections and reject unsupported network modes in both guards
- Reject unsupported network profiles during case initialization so invalid scopes are never emitted as ready
- Reject unknown case presets in Bash and PowerShell before creating case artifacts, preventing mistyped presets from silently falling back to pending/offline defaults
- Generate `skills/INDEX.md` from tracked skills only, excluding ignored local modules so clean-clone CI stays reproducible
- Fail routing coherence when a configured skill is missing or only exists as an untracked local file

## [1.0.0] — 2026-07-18

First **formal** public release of the reverse-skill skill-router pack.

### Added

#### Ops / combat contract layer (`skills/ops/`)

- `IDENTITY.md` — product identity: lightweight skill router + bootstrap + field-journal (not a Z3r0-style platform)
- `scope-contract.md` — case scope + `network_profile`; **auth not granted → no ACT on target**
- `evidence-finding-path.md` — Evidence → Finding → Path chain
- `role-map.md` — lead / specialist role mapping and handoff
- `timeline-workitem.md` — timeline + workitem coverage
- `sandbox-profile.md` — tool profile mapping
- `skill-supply-chain.md` — Agent Skill / MCP install gate (AST10-lite)

#### PRIMARY routing & case tooling (`skills/scripts/`)

- `master-route.ps1` — PRIMARY route from task hint
- `case-init.ps1` / `case-guard.ps1` — case bootstrap + scope guard
- `append-evidence.ps1` — structured evidence append
- `smoke.ps1` — package smoke checks
- `verify-routing-coherence.ps1` — routing / ops coherence verification
- `test-p0-friction.ps1` — P0 client-side lab friction regression tests

#### Core skills & docs

- Full skill matrix: APK / IDA / radare2 / JS / .NET / mobile / malware / pwn / firmware / EDR / pentest / API / LLM / supply-chain / crypto / binary-diff / patch-diff / attack-chain / docs & diagrams
- `MASTER-ROUTING.md` + `routing.md` / `routing_zh.md` three-axis matrix
- Bootstrap + tool-index pipeline (`bootstrap-reverse.ps1` / `.sh`, `refresh-tool-index`)
- `field-journal` precedent library + completion checklist
- Multi-platform paths: Windows primary, Linux / macOS / Kali docs and scripts
- CTF-Sandbox-Orchestrator competition sub-skills
- Burp MCP extension package (`burp-mcp-full/`)

#### Quality / localization

- UTF-8 integrity for Chinese docs (`RULES_zh`, `routing_zh`, related guides)
- Client-side lab playbook and recon pipeline references for authorized testing friction reduction

### Notes

- `skills/tool-index.md` / `tool-index.json` are **machine-local** and intentionally gitignored; generate via `refresh-tool-index` after clone.
- This tag freezes the skill-router product surface at commit `9fc280b` plus this release metadata.

### Links

- Tag: `v1.0.0`
- Repository: https://github.com/zhaoxuya520/reverse-skill

[Unreleased]: https://github.com/zhaoxuya520/reverse-skill/compare/v1.0.1...HEAD
[1.0.1]: https://github.com/zhaoxuya520/reverse-skill/compare/v1.0.0...v1.0.1
[1.0.0]: https://github.com/zhaoxuya520/reverse-skill/releases/tag/v1.0.0
