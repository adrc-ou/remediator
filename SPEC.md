# ADRC Remediator — Product and Architecture Specification

**Date:** 2026-10-05
**Status:** Draft for team review
**Implementation language:** Python 3.12 (application core) + TypeScript (browser UI)
**Deployment model:** Locally installed single-user application; browser UI on loopback; engine federation over localhost HTTP

---

## 0. How to read this document

Section 5 is the normative dependency manifest: **every external component this project depends on is named there with its license, boundary, and role.** Section 6 defines each integration contract unambiguously. Section 23 records the binding architecture decisions (ADRs) behind this shape; changing one requires amending its ADR, not just the code.

**Dependency policy (normative).** For every capability, this project names exactly one required implementation. Alternatives are permitted only as:

1. **model weights** loaded into a pinned runtime — bake-off decisions (§9.2); weights are data, not dependencies; or
2. **future replacements** requiring a new ADR.

There are no optional features and no "bring your own engine" user configuration. Anything not listed in §5 is out of scope for v1; where this document marks something *deferred*, that means no v1 requirement exists for it beyond *not blocking* the relevant interface — it does not mean an alternative implementation may be added opportunistically.

---

## 1. Executive summary

ADRC Remediator is an AI-native document reconstruction and accessibility compiler. It ingests degraded documents — primarily low-quality scans of academic textbooks, but also born-digital untagged PDFs and defective OCR text — and produces accessible publications. A format-neutral **semantic master document** is reconstructed through recursive image cleanup (diffusion restoration), document-VLM recognition, LLM text repair, and perceptual structure inference. Every hypothesis is governed by an **evidence graph** that prevents generated content from validating itself. Tagged PDF (PDF/UA-oriented), accessible HTML, and EPUB 3 are *projections* of that master state.

The application is a local-first, single-user workstation tool: a Python service with a browser UI, non-destructive transactional history, protected checkpoints, direct editing, an automatic cost-aware remediation planner, validation, and unrestricted export. Residual uncertainty surfaces in findings, scores, and the export report — it never blocks publication and never passes silently.

The defining hypothesis: **recognition is joint inverse rendering and semantic reconstruction, not one-way OCR.** Every reconstruction step is an adversarial test the image can win. Rendering a hypothesized glyph at a hypothesized location and checking whether the surrounding ink accepts it produces a signal that text-only correction loops cannot obtain: evidence that *the text was wrong*. The system's product is justified text and structure with visible uncertainty — not confident text.

### 1.1 The architecture in brief

- **A custom semantic core** (this project's product): the document kernel, evidence graph, transactional project history, assessment engine, and projections.
- **CPU-only application core.** No tensor library is imported by the core process. All GPU work happens inside engine services. This is a structural decision, not an optimization: it immunizes the core against the dependency-pin collisions that cripple multi-model Python environments (§5.7, ADR-003).
- **An engine federation at strict process boundaries:** ComfyUI (diffusion cleanup and glyph repair), vLLM (all document-VLM and LLM inference), docling-serve (born-digital structure extraction), and Java validators (veraPDF, epubcheck) — each required, each pinned, each reached over loopback HTTP or as a subprocess.
- **Three projections only, in a fixed order:** HTML (the reference surface), tagged PDF (the hard spine, de-risked first), EPUB 3.
- **Evidence-first phasing:** a benchmark corpus and a bounded end-to-end walking skeleton precede any application scaffolding, because module boundaries and the PDF writer design are the two decisions most likely to be wrong if made from imagination (§20).

---

## 2. Product definition

### 2.1 Users

Alternative-format production specialists; university disability-resource and accessible-media offices; libraries and digitization programs; publishers and remediation vendors; researchers in accessible document AI. Single practitioner per installation; project files are plain directories, so collaboration happens by file handoff. Multi-user server operation is explicitly out of scope, though the architecture must not preclude it.

### 2.2 Inputs

v1 must ingest:

- **Scanned and hybrid PDF** — low-resolution bitmap pages carrying photocopy artifacts (binding/gutter shadow, bleed-through, grime, uneven illumination), skew and curvature, multiple compression generations; handwritten underlines and marginalia that must be *classified* before any removal decision (§9.4).
- **Directories or ZIPs of numbered page images** (PNG/JPEG/TIFF/WebP; BMP and JPEG 2000 decoded through the pinned Pillow/OpenCV support matrix).
- **Born-digital untagged or poorly tagged PDF** — defective text layers, missing structure trees, broken reading order. The pixel-restoration path must be skippable for these without losing any downstream stage.
- **Externally supplied dirty OCR text** (plain text or simple HTML sidecar) to be corrected against supplied or re-rendered page images.
- **Prior Remediator semantic packages** — round-trip reopen for continued work.

Deferred importers: EPUB, DOCX, DAISY, PAGE XML, ALTO, hOCR. The semantic model (§7) must nevertheless be *capable* of representing what those formats carry, so that later importers are additive rather than redesigns.

### 2.3 Outputs

1. **Accessible HTML package** — the reference projection; also the development surface for screen-reader testing.
2. **Tagged PDF targeting PDF/UA practices** — cleaned page imagery with an aligned invisible text layer, a structure tree anchored by marked-content operators, real note and link semantics, `/Artifact`-marked page furniture, language tagging, an embedded subset font for the text layer, document metadata and XMP, validated by veraPDF before the export report is written (§15.3).
3. **EPUB 3** — reflowable projection with landmarks, page-list, page-break markers, note semantics, and MathML (§15.4).
4. **Semantic package export** — normalized content plus manifests; documented, versioned, used for interchange, archival, and round-trip.
5. **Export report sidecar** (§14.6) — findings, scores, validator results, models and revisions used, unresolved uncertainty. This doubles as the evidentiary record institutions need under regulations such as the European Accessibility Act, in force for e-books since June 2025.

Deferred projections (DOCX, DAISY 4, Braille/tactile production packages, large-print): the projector abstraction (§15.1) must accommodate them; nothing else about them is specified here.

### 2.4 Representative workflow

Launch → open or create project → import (hash-preserving) → automated triage probe → recommended remediation plan (editable, dependency-validated, cost-estimated) → queued transactional execution with progress, cancellation, and private recovery checkpoints → committed states in a linear history → inspection (comparison overlays, findings, provenance) → direct editing batched into commits → reassessment → export of any committed state at any score, accompanied by its report. The user always chooses; the system never gates publication and never hides what it guessed.

### 2.5 Non-goals

Not a general OCR product, not a document management system, not a reading app, not a cloud service, not a real-time collaborator, not a fully automatic "trust the machine" converter. Where an artifact is genuinely unrecoverable, the product's correct behavior is to *say so* — not to invent content (§7.8).

---

## 3. Design principles

1. **AI-native perception, formalized output.** Multimodal models do the heavy perceptual work; schemas, constraints, and validators store, exchange, test, and render their findings. Determinism lives at the edges of the system, around the models, not inside them.
2. **Recursive reconstruction.** Image, text, layout, and semantic hypotheses may repeatedly improve one another, under fixed convergence and cost rules (§9.6).
3. **Anti-circular evidence accounting.** A model may use context to reconstruct a character; the reconstruction may never count as independent confirmation of the same hypothesis (§7.7).
4. **Non-destructive operation.** Original sources and every committed state remain recoverable; reconstructed visual surfaces are versioned artifacts, never overwrites.
5. **Semantic master document.** No output format is the master representation; formats are projections with published capability matrices.
6. **Multiple accessible outputs.** PDF is required but never the sole target; the ideal accessible textbook is a *family* of artifacts in tension (screen-reader users want abstraction, low-vision readers want spatial fidelity), and only a format-neutral kernel serves both honestly.
7. **Default autonomy, exception-oriented control.** The system completes routine work autonomously, keeps the user informed, and makes intervention cheap — but automation always routes *around* protected content and *toward* findings a human can spot-check.
8. **Export is always available.** Findings and uncertainty inform the user but impose no publication gate.
9. **Local-first and privacy-preserving.** Everything runs on the user's machine; outbound network use is limited to explicit setup-time downloads (§17).
10. **One capability, one dependency.** The runtime surface is frozen (§5). Breadth of research is expressed by swapping weights inside fixed runtimes, never by adding runtimes.
11. **The core process is CPU-only.** The core moves JSON, geometry, files, and database rows; engines own all accelerators.
12. **Geometry is a ledger, not an assumption.** Every pixel-space quantity carries the coordinate space it lives in; every accepted geometric transform is recorded with its inverse (§7.5). Silent desynchronization of boxes, masks, and text alignment is treated as data corruption.
13. **Uncertainty is a deliverable.** Unresolvable regions stay visibly uncertain through every projection and every report. Crispness without evidence is the worst failure state this product can produce.
14. **Transactional execution and explainable history.** Every committed transformation records inputs, outputs, rationale, provenance, and a human-readable change summary; a failed or canceled step cannot corrupt the last committed state.

---

## 4. System architecture

### 4.1 System context

```text
┌─────────────────────────────── user's workstation ────────────────────────────────┐
│                                                                                    │
│  Browser (user's own)  ──http://127.0.0.1:<port>/?token=…──────────────┐           │
│                                                                        ▼           │
│  ┌─────────────────────────────  remediator-core  ────────────────────────────┐    │
│  │ FastAPI service · project API + event stream · preview/edit API · planner   │    │
│  │ Semantic kernel (Pydantic) · evidence graph · assessment engine             │    │
│  │ SQLite (WAL) state store · content-addressed artifact store                 │    │
│  │ Durable queue + scheduler (embedded) · module supervisor (subprocesses)     │    │
│  │ PDF projector (pikepdf/qpdf) · HTML projector · EPUB projector              │    │
│  │ Engine supervisor: lifecycle, health, VRAM budget arbitration, provenance   │    │
│  └───────┬──────────────────────┬──────────────────────┬───────────────────────┘   │
│          │ HTTP/WS (loopback)   │ HTTP (loopback)      │ HTTP (loopback)           │
│          ▼                      ▼                      ▼                           │
│  ┌───────────────┐     ┌───────────────┐     ┌────────────────┐   ┌─────────────┐  │
│  │   ComfyUI     │     │     vLLM      │     │ docling-serve  │   │ Java tools  │  │
│  │  (GPL-3.0)    │     │ (Apache-2.0)  │     │   (MIT)        │   │ subprocesses│  │
│  │ diffusion     │     │ doc-VLM OCR,  │     │ born-digital   │   │ veraPDF,    │  │
│  │ cleanup and   │     │ critics, LLM  │     │ PDF structure  │   │ epubcheck   │  │
│  │ glyph repair  │     │ repair        │     │ extraction     │   │             │  │
│  │ (recipes)     │     │ (OpenAI-compat│     │                │   │             │  │
│  │               │     │  API)         │     │                │   │             │  │
│  └───────────────┘     └───────────────┘     └────────────────┘   └─────────────┘  │
│         GPU-owning engines (app-managed subprocesses)        CPU-only tools        │
└────────────────────────────────────────────────────────────────────────────────────┘
        In-process libraries (the core's arms): pypdfium2, pikepdf, fontTools,
        uharfbuzz, Pillow, OpenCV, numpy — CPU only, no tensor frameworks.
```

### 4.2 Process and trust boundaries (normative)

1. **Core ↔ engines.** HTTP, plus WebSocket where progress streaming is needed, on loopback only. No shared memory, no Python imports across this line, and no filesystem assumptions beyond the explicitly-designated handoff directory (§12.3). Engines are app-managed subprocesses: the core supervisor starts them, health-checks them, and may stop them under memory pressure. Users do not start engines by hand.
2. **Core ↔ modules.** Built-in and drop-in remediation modules run as *separate OS processes* speaking a versioned JSON-RPC protocol over stdio (§11.3). The core never imports module code. Module crashes are contained: staging output is disposable; the committed state is unaffected.
3. **Core ↔ Java validators.** Subprocess invocation with a documented exit-code and output-parsing contract per tool (§6.5).
4. **License firewall.** GPL- and AGPL-encumbered components (ComfyUI, veraPDF's toolkit, MuPDF) are never linked, embedded, or vendored into core code. The core ships client-side integration code only. Distribution includes unmodified upstream engines as separately-licensed components with full notices and source availability paths (ADR-010). MuPDF (AGPL-3.0) is further excluded from the dependency graph entirely, even at arm's length, by policy (§5.6).
5. **Browser ↔ core.** Loopback bind, per-launch random token, `Host`/`Origin` validation, strict CSP; imported content is never executed by the application (§17).
6. **Internet ↔ everything.** v1 performs outbound network access only during explicit user-initiated actions: first-run engine setup and pinned model-weight downloads. Offline operation after setup is fully supported and is the default posture.

### 4.3 Hardware and platform matrix (v1)

| Tier | OS / accelerator | Core | ComfyUI | vLLM | docling-serve | Net capability |
|---|---|---|---|---|---|---|
| **Reference (full)** | Linux, NVIDIA CUDA ≥ 12.x, ≥ 24 GB VRAM, 64 GB RAM | ✅ | ✅ | ✅ | ✅ | Full pipeline |
| Supported (full, smaller weights) | Windows, NVIDIA CUDA | ✅ | ✅ | ✅ | ✅ | Full pipeline; model caps lower |
| Supported (partial) | macOS ≥ 14 (Apple Silicon) | ✅ | ✅ (MPS) | ❌ | ✅ | Import, cleanup, structure, HTML/EPUB; VLM OCR and LLM repair report **BLOCKED** with explicit guidance |
| Out of scope | CPU-only hosts, AMD ROCm | runs | degraded | ❌ | runs | documented, unsupported |

The partial tier is a hardware fact, not a configuration option: the *dependency list is identical on every platform*; capability degrades by engine availability, and the planner reflects exactly what this machine can do (§14.5). CI covers the Linux/CUDA reference tier fully and the macOS partial tier for core + ComfyUI paths.

### 4.4 Application lifecycle

The launcher selects the configured port (default 8765, with a bounded auto-fallback range) → binds `127.0.0.1` → runs migrations and readiness checks against SQLite and the artifact store → starts the engine supervisor (engines themselves start lazily on first demand and unload under memory pressure) → generates a per-launch access token → opens the browser at the tokenized URL → writes rotating logs to `~/.adrc-remediator/logs/` with secret redaction → on shutdown stops engines before the core. Port collision, stale lock, browser-launch failure, engine startup failure, and power loss each have a defined, non-corrupting outcome (§13.4). A copyable URL is always shown as browser-launch fallback.

---

## 5. Required dependency manifest (normative)

### 5.1 Tiers

- **E — Engines:** GPU-owning services reached over loopback HTTP/WS, launched and supervised by the core.
- **J — CPU toolchain:** subprocess tools (Java-based validators) invoked per run.
- **L — In-process libraries:** imported by the core (CPU-only) or by a module's isolated environment.
- **F — Frontend:** browser-side packages bundled into the served UI.
- **D — Development tooling:** build, test, lint; never shipped to users at runtime.

Every entry is pinned to an exact version/commit with hashes in lockfiles (`uv.lock`, `pnpm-lock.yaml`). The manifest itself lives at `dependencies/manifest.toml` and is emitted into every export report (§14.6) and SBoM (§17.6). "Pin policy" below: **E/J** pinned by release tag + commit hash with a startup contract test; **L/F** by lockfile hash; upgrades run the full regression gate (§21) before landing.

### 5.2 Engines (E)

| Component | Version policy | License | Boundary | Role — sole required implementation for |
|---|---|---|---|---|
| **ComfyUI** (comfyanonymous/ComfyUI) | pinned release tag + commit; companion node pack in this repo (`engines/comfy/recipes`) | GPL-3.0 | separate venv, app-managed subprocess, loopback HTTP/WS | diffusion image restoration, masked inpainting, glyph-conditioned text painting (cleanup + repair of page imagery) |
| **vLLM** | pinned release + per-model revision | Apache-2.0 | app-managed subprocess(es), loopback HTTP (OpenAI-compatible) | all transformer inference: document-VLM OCR/parsing, multimodal critics, local LLM text repair |
| **docling-serve** | pinned release | MIT | app-managed subprocess, loopback HTTP | born-digital PDF/DOCX structure and text-layer extraction (parser for the digital-PDF path) |
| **OpenJDK 17 (Temurin)** | pinned LTS build | GPL-2.0-with-classpath-exception | runtime for J tools | Java validator execution |

Engine exclusion decisions recorded as ADRs: no second model runtime (llama.cpp/Ollama/SGLang) — ADR-004; no remote API adapters in v1 — ADR-005; MuPDF/PyMuPDF banned from the entire graph, including inside engine environments — ADR-006 (rationale in §5.6).

### 5.3 CPU toolchain (J)

| Component | License | Boundary | Role |
|---|---|---|---|
| **veraPDF accessibility validator CLI** | GPL-3.0 (tool), run as separate process, unmodified | subprocess, JSON/text output | machine-testable PDF/UA checks on PDF projection output (§6.5) |
| **EPUBCheck** | BSD-3-Clause (verify at pin) | subprocess, JSON report | EPUB structural + accessibility conformance checks (§6.5) |

### 5.4 In-process libraries (L) — core only

| Package | License | Role |
|---|---|---|
| fastapi, starlette | MIT | HTTP/WS API of the core |
| uvicorn | BSD-3-Clause | ASGI server |
| pydantic (v2) | MIT | semantic kernel models, JSON Schema export, module I/O validation |
| aiosqlite | MIT | SQLite driver (database itself is embedded) |
| httpx | BSD-3-Clause | loopback clients to engines (async) |
| websockets | BSD-3-Clause | ComfyUI progress stream |
| **pikepdf** (bundles qpdf) | MPL-2.0 (qpdf Apache-2.0) | PDF object model read/write: structure tree, marked-content injection, metadata, XMP, outlines, embedded streams |
| **pypdfium2** (bundled PDFium BSD-3-Clause; verify license field at pin) | Apache-2.0 | PDF page *rendering* only (input rasterization; never writing) |
| **fontTools** | MIT | font subsetting, ToUnicode CMap construction for the text layer |
| **uharfbuzz** (HarfBuzz) | MIT | text shaping of one directional run at a time (script-aware positioning); it does not resolve direction — UAX #9 run segmentation is the projector's job (§29.3) |
| Pillow (pillow-simd not used) | MIT-CMU | image decode/encode, color management |
| opencv-python-headless | Apache-2.0 | classical geometry (deskew estimation, connected components, mask arithmetic), quality probes |
| numpy | BSD-3-Clause | array computation (CPU) |
| typer, rich | MIT | CLI launcher and console output |
| alembic-free manual SQL migrations (sqlite3 stdlib) | — | schema evolution (§12.5) |

**Prohibited in the core environment:** any tensor framework (torch/tensorflow/paddle/onnxruntime), any model-hub runtime, any PDF library with AGPL exposure, any "auto-tagger" wrapper of unknown provenance. Engine-side environments (ComfyUI venv, vLLM venv, module venvs) own their dependencies; the core never sees them.

### 5.5 Frontend (F)

| Package | License | Role |
|---|---|---|
| react + typescript | MIT / Apache-2.0 | application UI |
| vite | MIT | build |
| @tiptap/core + prosemirror-* | MIT | semantic editor bound to the kernel (§16.4) |
| pdf.js | Apache-2.0 | PDF preview in-browser (§16.5) |
| KaTeX | MIT | math rendering in HTML projection and preview (verify at pin) |
| @tanstack/react-query | MIT | server state |
| radix-ui primitives | MIT | accessible component base (keyboard, focus, ARIA) |

### 5.6 Development tooling (D)

uv (packaging/venvs), ruff (lint/format), mypy (types), pytest + pytest-asyncio (tests), Playwright (Apache-2.0 — UI and AT smoke automation), axe-core (MPL-2.0 — UI self-accessibility CI), cyclonedx-py (SBoM), pre-commit. Dev-only tools are excluded from shipped artifacts.

### 5.7 Model weights are data, not dependencies

The runtime set above is closed; research breadth lives in swappable weights:

- Restoration/inpaint weights for ComfyUI recipes (candidates for the Phase-0 bake-off; each bracket states the license of the **weights artifact** as reviewed under §30.2, which is a different question from the repository's code license: Qwen-Image-Edit [Apache-2.0], FireRed-Image-Edit [Apache-2.0, code and weights], DocRes [code MIT; the official checkpoints carry no license statement and ship without a canonical weights repo → gated pending review], AnyText + page-font embedding workflow [code Apache-2.0; weights descend from an SD1.5 checkpoint whose OpenRAIL-M use restrictions may attach → gated pending review] for glyph repair; FLUX-Kontext-class models excluded by their non-commercial licenses — ADR-006 corollary).
- Recognition/critic/repair weights for vLLM (candidates, on the same weights-artifact basis: Unlimited-OCR [MIT], DeepSeek-OCR [MIT], DeepSeek-OCR-2 [Apache-2.0], olmOCR-2 [Apache-2.0], dots.ocr [MIT per card metadata only — no license text ships with the weights → verify at pin], PaddleOCR-VL [Apache-2.0], Qwen3-VL [Apache-2.0, variants gated], plus a mid-size instruct LLM for text repair).
- The pinned set actually shipped is decided by the corpus bake-off (§20.1) and recorded in `models/MODELS.toml` (repo id, revision, hash, license, role assignment, measured capability). License review precedes pinning; unstated-license weights do not ship in v1 default assignments.

### 5.8 License policy (normative)

Allowed SPDX (runtime graph): MIT, Apache-2.0, BSD-2/3-Clause, ISC, MPL-2.0 (file-level copyleft, acceptable — components stay separable), PSF, OFL (fonts), LGPL-2.1+ *only* if dynamically linked and separable (none currently in graph). **Copyleft engines (GPL-3.0: ComfyUI, veraPDF; GPL-2.0+CPE: JDK) are permitted only as unmodified, separately-launched processes distributed as distinct components with their own notices; no linking, no vendoring, no code fragments copied into this repo** (client protocol code is not derivative — it targets documented public APIs). **AGPL is banned outright** (policy, stricter than legal necessity): MuPDF/PyMuPDF may not appear even inside engine environments, because recipes must remain copy-paste-portable into user contexts without license surprise (ADR-006). Non-commercial and field-of-use model licenses (FLUX-dev family, unstated-license weights) are excluded from v1 shipped defaults. The application core's own license and the repo's SPDX posture are fixed by ADR-010 (target: permissive, Apache-2.0 or MIT, decided before first public commit).

---

## 6. Integration interface contracts

### 6.1 Common conventions

- **Addressing.** Each engine receives a loopback port from the supervisor's pool (`127.0.0.1:49152–49250` by default). Ports are never announced beyond the process; the core is the only client. Residual risk (any local process can reach loopback) is documented in §17.4; engines that support auth tokens receive one (§6.3).
- **Readiness.** Every engine must expose (or be probed via) a documented health endpoint; supervisor marks engines `starting → ready → draining → down`. A module's execution may not bind to a non-ready engine; queue items transition to `waiting on dependency` instead.
- **Failure normalization.** Engine errors map to `EngineError {code, engine, stage, detail, retryable}` with codes: `CONN_REFUSED`, `HEALTH_TIMEOUT`, `STARTUP_FAILED`, `EXECUTION_FAILED`, `OUTPUT_INVALID`, `CANCELLED`, `OOM`, `VERSION_MISMATCH`. Raw tracebacks are captured to logs, never surfaced as contract data.
- **Retry policy.** Connection-level retries only (3× exponential backoff); *execution* is never retried automatically — the user or planner re-queues it, because retry is a semantic decision (cost, convergence state).
- **Provenance (mandatory).** Every engine call is recorded as an evidence-bearing provenance record before the output artifact is committed: `{engine, engine_version, recipe_or_model_id, revision_hash, parameters, seed(s), input_artifact_hashes[], output_artifact_hashes[], started, finished, device, resource_usage}`. Outputs without a provenance record cannot enter staging (§13.2).
- **Cancellation.** Cooperative at page/region/tile granularity where the protocol allows (§6.2 interrupt, §6.3 disconnect-abort, §6.5 process kill); "cancel requested" is a distinct observable state from "canceled" (§13.3).
- **Artifact handoff.** Binary payloads cross boundaries only through the project's `staging/` directory, addressed by content hash, or inline base64 where the engine API requires it (≤ 32 MB). Engines never hold authoritative state.

### 6.2 ComfyUI (diffusion engine)

**Launch.** Dedicated venv managed by uv; start command `python main.py --listen 127.0.0.1 --port <P> --disable-auto-launch`, plus tier flags (`--cpu` fallback for degraded tiers; device selection is engine-internal). The installation is *unmodified upstream at a pinned commit*; this project's only writes into it are (a) the companion node pack `remediator_recipes` under `custom_nodes/`, (b) model weights under its `models/` tree, (c) a config that disables telemetry-like features and its Manager's auto-update. First-run setup downloads and verifies the pinned commit hash.

**Endpoints used (public documented API):**

| Call | Purpose | Contract notes |
|---|---|---|
| `GET /system_stats` | readiness, device/VRAM inventory, version-string check | the response carries a version string and a **list** of devices, not a source revision: the commit pin is verified against the installed tree recorded at setup, and the version string and echoed launch arguments are cross-checked here (§25.1); any mismatch → `VERSION_MISMATCH` |
| `GET /object_info` | capability probe (required node classes present) | checked per recipe at startup, not globally |
| `POST /upload/image` | page image ingress | multipart; returned `{name, subfolder, type}` recorded as handoff reference |
| `POST /prompt` `{prompt: graph, client_id: uuid4}` | run a recipe instance | graph is composed from a Recipe Manifest (§6.2.1); `prompt_id` recorded |
| `WS /ws?clientId=<uuid>` | progress stream | events: executing node, progress value, execution error, execution complete |
| `GET /history/<prompt_id>` | final outputs + node outputs | authoritative for artifact file names |
| `GET /view?type=output&filename&subfolder` | artifact egress | stream into CAS; hash-verified against history output |
| `POST /interrupt` | cooperative cancel | engine-dependent granularity; followed by queue drain check |
| `GET /queue` / `POST /queue` / `POST /free` | queue introspection, VRAM release | supervisor uses `/free` between non-adjacent model families |

**6.2.1 Recipe manifest.** Each workflow recipe is a directory in `engines/comfy/recipes/<name>/`:

```toml
# recipe.toml (illustrative)
id = "page_cleanup_v1"
semver = "1.0.0"
comfy_rev = "<pinned commit>"
graph = "workflow.json"          # the API-format prompt graph
inputs = { page_image = { node = "12", input = "image" } }
outputs = { cleaned = { node = "44", output = "images" } }
params = { denoise = { node = "30", widget = 3, type = "float", default = 0.35, range = [0.05, 0.8] },
           mask =     { node = "18", input = "mask" } }
models = [{ repo = "...", revision = "...", file = "...", sha256 = "...", into = "models/checkpoints" }]
capabilities = ["diffusion.inpaint", "diffusion.restore"]
smoke = { script = "smoke.py" }   # optional pre-run validation asset
```

The core composes a runnable prompt graph by substituting parameters into the manifest-mapped nodes only; *recipes are data*, reviewed like code, hashed into provenance, and versioned by semver. A recipe may reference this project's companion nodes (which implement pypdfium2 rendering, DocRes/AnyText pipelines, mask utilities, page-tile stitching) but the graph must remain inspectable and runnable by a human in the ComfyUI UI — that transparency is the point of keeping the vision stage in ComfyUI.

**6.2.2 Data-plane rules.** Pages are handed off as PNG (lossless) at working resolution (§18.4 defines the ladder). Masks accompany regions-of-interest for localized repair; the recipe must accept `{image, mask}` and must not modify pixels outside the mask's dilation envelope (validated by the module, not trusted).

### 6.3 vLLM (inference engine)

**Launch.** One subprocess per served model (`vllm serve <local model dir> --served-model-name <logical-id> --host 127.0.0.1 --port <P> --max-model-len <M> --api-key <per-launch token> --dtype/--quantization per registry entry`). The model registry (§5.7 `MODELS.toml`) declares per-model: role(s), VRAM reservation, required capabilities. The supervisor may run multiple instances concurrently up to the GPU budget; otherwise it swaps with a documented cold-start cost.

**API surface (OpenAI-compatible):**

| Call | Purpose | Contract notes |
|---|---|---|
| `GET /v1/models` | readiness + identity | served-model-name must equal registry id |
| `GET /health` | liveness probe | |
| `POST /v1/chat/completions` | all inference | multimodal: `content` parts `{type:"image_url", image_url:{url:"data:image/png;base64,…"}}`; grounding prompts return structured JSON via `response_format: {type:"json_schema", json_schema:{strict:true,…}}` where supported (capability flag); sampling params pinned per task profile; `stream:false` in v1 except critique UX paths |

**6.3.1 Capability probes.** At load time the core issues a fixed probe set (structured-output echo, single-image reference, multi-image reference) and refuses to mark an instance `ready` for roles whose required capabilities fail. This makes model swaps *evaluations* (weights + probe + bake-off) rather than integrations.

**6.3.2 Recognition adapters.** Each OCR/parsing weight family ships a Python adapter (in-module code, not core) that converts model output (markdown-with-anchors, DocTags, bbox-JSON, whatever the family emits) into the canonical Recognition Result envelope (§7.6 schema). Adapter contract: `to_kernel(raw) → (page_objects, alignment, uncertainty)`, `validate(kernel) → findings[]`. The kernel never parses model-native formats.

**6.3.3 Cancellation/timeouts.** Client disconnect aborts the generation server-side (documented vLLM behavior); the core enforces per-task wall-clock ceilings from the model registry (default: OCR ≤ 180 s/page, critique ≤ 60 s/region, repair ≤ 30 s/batch), on breach → `EXECUTION_FAILED(retryable=true)`.

### 6.4 docling-serve (structure engine)

**Launch.** Dedicated venv; `docling-serve start --host 127.0.0.1 --port <P>`. In-process heavy models (layout, tableformer) live in the engine's environment, never the core's.

**API:** `POST /v1alpha/convert/source` — page or whole-document bytes (base64/URI source), features selected per import profile (no OCR for digital PDFs; table structure on; picture classification on), response: DoclingDocument JSON plus referenced artifacts (extracted images). Contract pins the exact docling-serve release; the route surface is re-asserted by the startup contract test (`CONTRACTS.md` records probe payloads). The digital-path importer maps DoclingDocument → kernel (§8.3) and records the engine as evidence class `prior_textlayer` / `prior_structure` — importantly, *not* as ground truth.

### 6.5 Java validators

**veraPDF.** Invocation: `java -jar <verapdf-core.jar> --format json --flavour ua1 … <file>` against exported PDFs (flavour pinned per PDF projection profile; `ua1` and `ua2` are the PDF/UA flavour tokens, enumerable via the tool's own list option). The core parses the JSON report into Findings (§14.3). Exit-code mapping, per the validator's documented set: `0` all files processed and valid · `1` processed and found invalid (expected — produces findings) · `2` invalid command-line parameters · `3` out of Java heap · `4` no files to process · `6` I/O error · `7` failed to parse one or more files · `8` some PDFs encrypted · `9`–`12` tool-side exceptions (there is no `5`). Only `0` and `1` are verdicts; every other code is a tool condition and maps to `EngineError`, never to zero findings (§25.4). The JAR is installed by setup into a managed tool directory with hash verification.

**EPUBCheck.** Invocation: `java -jar epubcheck.jar --json <report.json> <file.epub>`. The JSON report is selected by `--json`; `--out` writes XML instead, and no `--mode` is passed for a whole publication (`--mode` exists for single-file checks — `opf|xhtml|svg|mo|nav` — and is meaningless alongside an `.epub` path). `--maxOfEachMessage unlimited` is set so a finding class is never silently truncated (§14.3). Exit codes are coarse — `0` clean, `1` errors *or* argument/usage failure, `2` unexpected exception — so the JSON report, not the exit code, is authoritative for findings, and the exit code only signals that the check did not run properly (§25.4). Both tools are treated as *advisory machine checks*: passing is necessary, never sufficient (§14.2).

### 6.6 PDF object engines (in-process, boundary-by-role)

- **pypdfium2**: input rendering only. Given a source PDF page index → raster at requested DPI into staging. It must never be used to write, re-save, or "round-trip" a PDF.
- **pikepdf**: the only writer of PDF bytes. Owns: object model, `/StructTreeRoot` construction, content-stream insertion (`BDC`/`BMC`/`EMC` marked-content operators with stable `/MCID`s), invisible text layer objects, resources (fonts via embedded streams), outlines, names tree, XMP metadata, document ID policy (deterministic for reproducible builds; randomized on request).
- **fontTools + uharfbuzz**: prepare the text layer — segment runs by script and direction (UAX #9 is implemented by the projector; HarfBuzz shapes one directional run at a time, §29.3), shape each run, subset the shipped open font (OFL-licensed; exact face pinned in `fonts/MANIFEST.toml`, RFN handling per §31.4), and build the ToUnicode CMap from the shaper's cluster map. The core composes the content-stream text operators itself (`BT … TJ … ET`) from shaped runs; no third-party "PDF writer" layer sits between kernel and bytes.
- **Ownership invariant:** exactly one component (the PDF projector in core) may synthesize output PDFs. Engines and modules produce *images, JSON, and geometry*, never PDF bytes (export reports record this as an audited invariant).

### 6.7 Engine supervisor (inside the core)

Maintains the engine registry (state, version, capabilities, VRAM reservations), lazily starts/stops engines against module resource declarations, arbitrates a single VRAM ledger (start refused → queue stall event `ENGINE_WAIT`), health-probes on interval and on error, performs bounded watchdog restarts (restart does not auto-replay the interrupted execution), and emits the engine status panel consumed by the UI (§16.7). All supervisor decisions are events in the durable log (§13.4).

---

## 7. Canonical semantic document model

### 7.1 Role

The semantic master is the authoritative state of the work. All history commits, assessment findings, module inputs/outputs, and projection attempts read from and write to it. It is a superset of the semantically meaningful capabilities of the supported formats (§2.3), and it is versioned, validated, and serialized independently of any UI or engine. The internal schema is not surfaced raw to users; each preview and property pane translates kernel concepts into format-appropriate terminology.

### 7.2 Hierarchy

```text
Project
  Document (book)
    Publication metadata (title, authors, language(s), identifiers, rights statement)
    BookProfile reference (§10; versioned, evolving, not a hidden prompt)
    Landmarks and sections (part/chapter hierarchy)
    Pages (physical) — 1:1 with a PageSurface stack
      PageSurface: artifact chain (source raster → transformed rasters) + SpaceRegistry (§7.5)
      Regions and semantic objects (§7.3)
        TextRuns + alignments (§7.6)
        Figures/images + descriptions
        Tables (grid model + cells)
        Equations (structured payload)
        Charts (data payload + description)
        Notes and references (markers + targets)
        Links (origins + destinations)
        Page markers (printed folio)
        Artifacts (page furniture excluded from reading)
    Reading and navigation graph (§7.9)
    Evidence graph (§7.7)
    Findings and review states (§14)
    Projection overrides (per-format, §15.1)
    Provenance records (§6.1)
```

A *logical resource* (EPUB spine item, DAISY unit) is derived at projection time from sections/pages; no parallel page-less document tree exists in the kernel. Original pagination is preserved per §27.2: printed folio ≠ digital index, both stored.

### 7.3 Semantic objects

Every semantic object carries, as applicable:

- stable identifier (immutable within the document; edits create new versions, not mutations) and type/subtype from the registered object taxonomy;
- parent/child and ordering relationships;
- source references: document/page/region geometry in a named space (§7.5), pointing at source evidence artifacts;
- geometry: polygon (not just rect) + owning space id;
- text payload where relevant (see variants, §7.4);
- language and script (per object, inheritable);
- typography and presentation hints (role-derived: emphasis, superscript-run, small-caps — *hints*, never the source of semantics);
- reading-order edges (§7.9) and structural-parent edges — these are distinct graphs;
- confidence (§7.7) and assessment status; review state: `not-reviewed · reviewed · approved · needs-attention · intentionally-accepted · do-not-modify`;
- evidence dependency references;
- projection overrides (sparse, namespaced per format).

Object taxonomy v1 (extensible per §26.1): `Heading{level}`, `Paragraph`, `List/Listitem`, `Figure`, `Illustration`, `Photograph`, `Table`, `TableRow/TableCell/HeaderCell`, `Equation`, `Chart`, `CodeBlock`, `Caption`, `SideNote (sidebar/callout/inset/feature box)`, `PullQuote`, `Footnote/Endnote` + `NoteMarker`, `CrossReference`, `PageFurniture (artifact): RunningHead/RunningFoot/PageNumber`, `DropCap`, `Quotation`, `IndexEntry`, `UnknownRegion` (the honest catch-all — every `UnknownRegion` at export time is a finding).

### 7.4 Text variants

An object may carry, independently versioned and provenance-stamped:

- **canonical text** — faithful for quotation, search, and copy; never silently "improved";
- **assistive text** — speech/Braille-oriented presentation (expansions like "Dr." → "Doctor", spoken labels for note markers, table-cell preambles); transformations are confidence-scored, reversible, and reviewable as diffs;
- **visual display text** — where rendering differs (ligatures, drop caps split from paragraphs);
- **pronunciation/annotation** — optional phonetic or reading hints consumed by DAISY-class projections later.

Rule: a projection may only substitute assistive text where the target format has an actual mechanism for it (PDF `/Alt` vs `/ActualText` vs real text, EPUB `aria-label` vs content); the projector's capability matrix (§15.1) enumerates exactly where each variant lands or is dropped-with-finding.

### 7.5 Coordinate spaces and the transform ledger (normative)

Geometry never floats. Each `PageSurface` maintains a **SpaceRegistry** of named coordinate spaces, and a **transform ledger** recording the chain between them:

```text
Space  = { space_id, page_id, kind: source|normalized|working|projection,
           width_pt, height_pt, rotation, created_by_state }
LedgerEntry = { seq, from_space, to_space,
                op: deskew{angle}|crop{bbox}|rescale{factor}|dewarp{mesh_artifact_hash}|rotate{deg}|pad{edges},
                inverse: params | inverse_artifact_hash (mesh),
                module provenance ref, committed_state_id }
```

Rules:

1. Every stored geometric quantity (region polygon, alignment box, mask reference, note marker position) names the space it is expressed in. Bare coordinates are a schema error.
2. Transport between spaces is computed by composing ledger entries (analytic ops invertible in closed form; `dewarp` transports via the inverse mesh lookup artifact). The ledger is append-only within a committed state.
3. **Any accepted geometric transform invalidates downstream alignment data** and emits `ALIGNMENT_STALE` findings for objects transported through it (§7.6); modules that edit imagery declare their spatial effect in the ledger — undeclared geometry change is a module contract violation caught at validation (§11.3 `validate`).
4. The exported PDF defines its own space (PDF user space); the projector emits the composed source→projection transport for every visible object, so page-level overlays in any viewer can be re-anchored to any historical space.

### 7.6 Alignment and the canonical Recognition Result

Per-page recognition output normalizes into the **Recognition Result envelope** produced by adapters (§6.3.2):

```jsonc
{
  "page_id": "...", "space_id": "...",
  "text": "...",                        // reading-order linearization of markdown below
  "spans": [ { "run_id": "...", "text_range": [120, 138], "role_guess": "paragraph|heading3|footnote|...",
               "boxes": [ {"polygon": [[x,y],...], "space_id": "..."} ],
               "baseline": "ltr|rtl|ttb", "confidence": 0.0, "alternates": [ {"text": "...", "confidence": 0.0} ] } ],
  "structure_hints": [ /* region list with types + polygons */ ],
  "uncertainty_map": "artifact_ref",    // optional heat raster, source-resolution
  "suspicious_spans": [ {"run_id": "...", "reason": "low_conf|cross_engine_disagreement|geometry_anomaly"} ]
}
```

Text-run ↔ pixel alignment is maintained *separately from recognition*: it is a property of (canonical text version, page surface version, space). The invisible-text layer for PDF is generated from the current alignment; any accepted edit that changes text segmentation (word merge/split, line reflow across page boundaries) marks dependent alignments `STALE`, and export of a `STALE` alignment emits a blocking-quality finding (surfaced in the report, still exportable per principle 8). Re-alignment is a first-class remediation module (forced aligner: character-class matching + geometry optimization; no model required).

### 7.7 Evidence graph (operational rules)

Claims (specific object field versions: a text run's characters, a heading's role, a table's span structure) carry evidence edges:

```text
EvidenceNode = { id, claim_ref,
  class: source_observation | prior_textlayer | prior_structure | uncond_restoration |
         text_cond_restoration | counterfactual_render | local_context | book_repetition |
         typography_prior | user_input | model_output,
  lineage: { model_family?, checkpoint_rev?, prompt_hash?, seed?, engine_call_ref? },
  depends_on: [evidence ids],
  assertion_inputs: [claim refs this generation conditioned on] }
```

Normative rules:

- **R1 (descendant exclusion).** Evidence transitively depending on the claim it supports has weight 0 for that claim.
- **R2 (lineage independence).** Two `model_output` evidences corroborate only if their `lineage.model_family` differ (family = architecture + training lineage identity, recorded per model registry entry; repeated sampling of one checkpoint is one observation).
- **R3 (effective ensemble).** Per-claim ensemble strength is `n_eff = 1 / Σ wᵢ²` over independent evidence weights; `n_eff < 2` with any disagreement ⇒ the claim cannot exceed `needs-attention` automatically, however high the individual confidences.
- **R4 (conditioned imagery).** Pixels produced by `text_cond_restoration` or `counterfactual_render` are never `source_observation`; OCR on them is `model_output` with `depends_on` the conditioning text (mechanically recorded via recipe/model provenance, not by trust).
- Confidence values are *calibrated* against the benchmark corpus (§20.1), versioned with the assessor, and reported as calibrated probability, not raw logits.
- **User input is supreme evidence**: `user_input` ends automated dispute on a field; later modules may propose against it only via findings, never silent edits.

### 7.8 `UNRECOVERABLE` state

An object or span may be marked `UNRECOVERABLE { reason: information_destroyed | irreconcilable_conflict | outside_capability, evidence_refs, marked_by: system|user }`. Rules:

1. Automatic processes may propose it (low calibrated confidence + no applicable modules remain) but only a user commit can *finalize* it for export.
2. It renders explicitly in every projection: HTML `<span class="unrecoverable" role="note">[illegible in source]</span>`; PDF marked structure with `/ActualText` announcing the gap; EPUB analogous. The visual page keeps the honest degraded pixels — no manufactured crispness (principle 13).
3. Each `UNRECOVERABLE` is a line item in the export report, countable, reviewable, and *excluded* from repair targeting unless the user explicitly re-opens it.
4. `UNKNOWN` (capability exists, not yet attempted) is a distinct state — triage distinguishes "no answer" from "no answer possible."

### 7.9 Reading order as a graph

Reading order is a first-class directed graph, not box sorting. It must represent: the default linear path; column and page continuations; optional/supplementary branches (sidebars — essential vs supplementary placement per §27.3, with the conventions of §10; note references and *return edges*; figure/table associations and caption binding; skippable and escapable structure classes; explicitly-marked duplicate presentations (pull quotes spoken once); artifacts excluded entirely. Validation constraints (deterministic, run on every commit): every meaningful visible region appears in the default path unless classified artifact/duplicate/branch; no region appears twice on the default path without a duplicate-presentation link; references have targets and return targets; heading hierarchy coherent (single logical H1 per chapter unless profile says otherwise); table ownership and spans non-conflicting; `UnknownRegion`s surfaced.

### 7.10 Notes and references

A note reference stores: visible label (number, letter, dagger, asterisk, any symbol), spoken label, target note id, exact return anchor, reference type (footnote/endnote/callout/cross-ref), many-to-one marker relationships, and cross-page linkage. Projections map these to PDF `Reference`/`Note` structures with destinations, EPUB/HTML `doc-noteref`/`doc-footnote` with back-links, and (later) DAISY skippable/escapable behavior. Symbol markers (†, ‡) are not silently renumbered; the visible-label/spoken-label split covers "marker reads as footnote seven."

### 7.11 Links and destinations

Internal destinations (headings, figures, table rows by id), external links (stored, rendered as real annotation objects in PDF, not burnt into the visible text layer), and link-role (citation vs navigation vs reference) with orphan/loop validation.

### 7.12 Media and specialist payloads

Figures hold description objects (§14.7): `decorative | short | long description | pedagogical explanation | extracted labels/relations | data table (for charts) | tactile-referral recommendation`, each separately versioned/provenanced. Tables store the grid model (spans, header scope, caption binding, unit cells) plus a *computed* linearization that is regenerable and overridable; equations store structured payloads (canonical LaTeX + validated MathML after §9.5 symbol checks, never raster-only truth); charts store extracted data *with* uncertainty and axis-interpretation checks (§14.7). Specialist payloads reference artifacts by hash and are opaque to the kernel otherwise.

### 7.13 Schema governance

The kernel schema ships as Pydantic models with generated JSON Schema (published in `schemas/`). Changes are additive within a major version; migrations are forward-only, tested against golden projects from every released version (§21.4). Unknown extension data encountered on read is preserved, never dropped. The **extension registry** lets modules attach namespaced property bags (`ext:<module-id>/<name>`) validated against module-published JSON Schemas — modules extend the model without forking it.

---

## 8. Import, triage, and source reconciliation

### 8.1 Source intake

Import is hash-preserving (SHA-256 of every ingested file recorded as a `source_observation` evidence anchor). Page detection for image directories uses numeric-order heuristics with an explicit confirmation step when anomalies appear (gaps, duplicates, orientation variance); ZIPs are unpacked defensively (§17.3).

### 8.2 Triage probes (deterministic, CPU)

Per source: page inventory (missing/duplicate/blank/rotated detection); text-layer presence and quality probes (layer-vs-image agreement sampled per 50 pages — the classic defective-OCR detector: embedded text that re-OCR cannot corroborate); tag presence via pikepdf structure parse (`/StructTreeRoot` walk: coverage, suspicious patterns like everything-`<P>`, missing `/Lang`/title); image stats (resolution ladder position, color space including **semantic-color detection**: spot colors / red-black two-color printing ⇒ grayscale pipeline must be refused with a finding, §9.1); geometry scan (skew estimate, gutter-shadow profile, curvature sample — OpenCV estimators recorded as ledger entries *proposed, not applied*); script inventory (Latin/RTL/CJK/diacritic-heavy ⇒ capability gate per model registry declarations); and a cost profile (page count × probe findings → planner inputs). Triage output: findings + a recommended plan with per-module scope, dependency edges, model assignment, and estimated GPU-seconds (§14.5).

### 8.3 Path selection

- **Born-digital with usable text layer:** pixels skipped by default; docling-serve → kernel mapping; existing structure recorded as `prior_structure` evidence; cleanup modules available but *not* recommended. This path must reach export with zero vision-stage runs.
- **Scanned/hybrid:** render (pypdfium2) → geometry ledger (§7.5) → cleanup/repair loop (§9) → recognition ensemble.
- **Mixed:** per-page classification (hybrid PDFs are page-wise realities, not document-wise).
- **Supplied text + images:** alignment module computes Recognition Result envelope from supplied text against rendered pages; LLM repair operates with `prior_textlayer` evidence, never trusting it.

### 8.4 Reconciliation across sources

When multiple representations exist (clean EPUB text + scan for fidelity, second edition for structure hints), alignment is *evidence merging*, never overwrite: each source contributes claims with classes and lineage; conflicts become findings with side-by-side resolution UI; edition-specific content is protected — a different edition may not silently replace edition-specific passages (heuristic: conflict density over a section auto-blocks that section's cross-edition import). Reconciliation decisions are individually recorded and reversible.

---

## 9. The reconstruction loop

### 9.1 Pass structure (per page region where applicable, not per page globally)

1. **Geometry normalization** — deskew/crop/curvature as *ledger-recorded transforms* (§7.5). Estimates come from classical CV; confirmation is visual (source-vs-normalized overlay available in UI). Semantic-color pages keep color surfaces (§8.2 refusal applies to grayscale ops).
2. **Restoration (unconditioned)** — ComfyUI cleanup recipe (DocRes/Qwen-Edit-class weights per §5.7 bake-off) producing a versioned surface with `uncond_restoration` evidence class. Protection masks bound what may change (figures, stamps, copyright pages, handwriting pending classification).
3. **Recognition ensemble** — ≥ 2 model *families* via vLLM (per R2 §7.7): e.g., a parsing VLM (Unlimited-OCR / DeepSeek-OCR / olmOCR-2 winner) + a classical-detector path or second family; outputs normalized to Recognition Results; disagreements become `suspicious_spans`.
4. **Structural inference** — layout/role classification from cleaned imagery + Recognition Results + BookProfile conventions (headings, furniture, sidebars, pull quotes, note markers, drop caps); reading-order graph constructed with its §7.9 constraints; equations/tables/charts split to specialist modules (§7.12, §14.7).
5. **Text repair (LLM)** — correction operates on canonical text with `local_context` + `book_repetition` evidence; *assistive* text is where speech-oriented normalization goes; canonical fidelity protected by change-logging (§7.4) and role-change linting (e.g., repairs may not change a recognized heading's text to non-heading prose without a finding).
6. **Targeted visual repair** — for disagreements where the *image* is the problem (damaged letterform), a repair request (§9.4) → ComfyUI glyph recipe (§6.2, mask-bounded, page-font conditioned) → re-recognize (§7.7 R4 applies: conditioned restoration is not corroboration).
7. **Counterfactual adjudication** — where hypotheses remain (§9.5); accept the reading that best *explains the source under a degradation model*, record the comparison as evidence, escalate residual ambiguity to `needs-attention` or `UNRECOVERABLE` (§7.8).
8. **Alignment + evidence update** — re-run alignment where text or geometry changed (§7.6); recompute per-claim confidence; emit findings.

A "pass" commits zero or one state; the loop above is a *plan template*, and users/planner may run pieces independently — these are modular stages first, a pipeline second.

### 9.2 Model bake-off policy

Every role (restoration, OCR-family-1, OCR-family-2, layout, critic, repair LLM, glyph painter) has exactly one shipped default assignment, chosen by Phase-0 evaluation on the benchmark corpus (§20.1) and re-run at every weight or engine upgrade. "Candidate models" exist only as *evaluated, licensed, pinned registry entries* — the runtime never forks behavior on model identity beyond adapter selection (§6.3.2). Benchmarks from vendor cards are admissible as *screening*, never as acceptance.

### 9.3 Critics

Independent checks per commit batch: visual-support check (does the claim's box actually contain the claim's glyphs — cross-family), contextual plausibility (repair LLM as critic only when different lineage from producer, per R2), typographic consistency (baseline/x-height/advance checks against BookProfile), equation syntax parse (LaTeX→MathML validate; failed parse = finding, never silent fix), table consistency (span closure), reference integrity (targets/returns resolve), duplicate/missing content probes, and *degradation-honesty*: is a repaired region suspiciously smoother than its neighbors (drift alarm, §24-R5). Critics emit findings; they do not edit.

### 9.4 Targeted repair requests

A repair request bundles: region polygon + space id, diagnosed problem, competing readings (max hypotheses param, default 2), surrounding context window (pixels + text), learned typeface reference (BookProfile glyph gallery, §10), invariance constraints (neighbors unmodified outside mask dilation), and a resource budget. The ComfyUI glyph recipe must be *conditional on text* only through this request object (so R4 recording is mechanical). **Marginalia rule:** handwritten content must be classified first — publisher content / reader annotation / instructor annotation / historical annotation / damage — with a *separate layer retained by default*; removal is an explicit, per-class user-approved action, never the automatic default (annotations may be the only record of a previous owner; also PII, §17.5).

### 9.5 Counterfactual rendering

For ambiguity sets {h₁…hₖ}: render each hypothesis with the page's learned typography, degrade each under the *observed* degradation model estimated from the page, and score likelihood against the source pixels (CPU: SSIM/distance transforms on the region). The comparison, its inputs, and its verdict are stored as `counterfactual_render` evidence. When likelihood differences fall under the calibrated threshold, the correct outcome is ambiguity recorded (§7.8), not a coin flip. This is a research-adjacent capability; it ships behind the same module contract as everything else and may be *recommender-only* in v1.

### 9.6 Convergence and cost

Hard stops (normative): no applicable findings remain; predicted improvement below threshold (calibrated assessor delta per GPU-second, §14.5); same recommendation repeats without measurable improvement; detected A→B→A oscillation (region hash cycles); iteration cap (default 3 per region, 6 per page); resource budget exhausted (per-project GPU-second ceiling, user-set); user cancel. A stopped loop is a *reportable state* — findings say which stop fired and what remains contested.

### 9.7 Priority scheduling

Repair effort is allocated by expected information gain: `priority = finding severity × calibrated uncertainty × (1 − evidence strength) / estimated cost`. Disagreement-gated work only: regions where two independent families already agree are never reprocessed for "polish." This keeps the loop from being an infinite generator of confident nonsense (§24-R2) and gives the planner its cost curve (§14.5).

---

## 10. BookProfile (learned conventions)

After a representative sample (first chapter + stratified pages), a versioned `BookProfile` records: recurring page templates; heading/caption conventions (size/weight/position → level mappings with confidence); **glyph gallery** (verified typeface samples per face/size/weight — the material for §9.5 counterfactual rendering and §6.2 glyph-repair conditioning); column geometry and furniture zones; numbering conventions (equation/figure/citation/footnote series, folio styles including roman-front-matter); glossary/vocabulary/abbreviations and *this book's* OCR confusions (learned confusion matrix); language/script distribution. User corrections are high-value profile examples and may trigger targeted reassessment of similar regions — always as proposals + findings, never silent retroactive edits. The profile is inspectable/editable in the UI (it explains itself: every convention cites the pages it learned from).

---

## 11. Remediation modules and the plugin system

### 11.1 Layout and installation

Every module is a self-contained directory under `modules/`: `module.toml`, `README.md`, dependency declaration (uv-managed, locked), `src/`, `schemas/`, `tests/`, optional `frontend/` (config UI) and `migrations/`. Installing = placing the directory (first execution requires an explicit trust prompt: permissions, publisher, diff-vs-previous-hash). **No module code is ever imported into the core process** — built-in and third-party modules use the identical subprocess contract (no privileged built-ins).

### 11.2 Manifest (`module.toml`)

Stable id + semver; module-API version; name/description/author/license/source; entry point; **required capabilities** (engine roles: `diffusion.restore`, `vlm.ocr`, `llm.repair`, `structure.layout`, …) and *forbidden* capabilities (a cleanup module that needs no LLM declaring `llm.any` fails validation); accepted and produced kernel object types (input selectors: page/region/span classes); produced findings/artifact kinds; typed parameter schema (JSON Schema → generated config form §16.6); model-role requests; **resource estimates** (CPU seconds, GPU seconds, VRAM peak, disk) — validated in CI against declared values; invalidation semantics (which kernel fields it may touch); resumability/cancellation support level; permissions (fs scope, network — default none, subprocess spawn — default deny); UI schema; migrations.

### 11.3 Standard lifecycle (stdio JSON-RPC, versioned)

```text
inspect(context)     → applicability + recommendation evidence (pure, cheap)
plan(context, params)→ work units, dependency edges, cost estimate
run(snapshot, staging_writer, model_broker, progress, cancel_token)
validate(staging)    → findings (including contract violations: undeclared geometry writes,
                       out-of-scope field edits, missing provenance)
summarize(diff)      → human-readable change summary (history label)
resume(recovery)     → continued execution
```

`snapshot` is an immutable read view (content-addressed export of the relevant kernel slice + artifacts by handle); `staging_writer` produces *patches* (per-object preconditions: expected field hashes → conflict detection on commit) and artifacts. Modules never touch the live project database. `model_broker` is a core-provided loopback proxy (per-run credential-less lease on declared roles only; every call recorded with full provenance automatically — modules cannot bypass evidence accounting).

### 11.4 Isolation and trust (normative)

Per-module venv (uv), unshare-style resource limits where OS permits (CPU/wall/disk via launcher); default filesystem scope = project staging area; network denied by default (setup-time downloads are core-mediated, not module-mediated); crash containment (supervisor restart policy, failed run quarantined); signed distribution support (checksum + publisher key) in the manifest; compatibility quarantine when module-API version mismatches. The drop-in folder is a *discovered manifest*, never executable code the app will run without the trust gate.

---

## 12. Project package and persistence

### 12.1 Editable project format

A project is a directory: `project.toml` (format version, ids), `content.db` (SQLite WAL: states, objects, edges, evidence, findings, events, queue), `cas/` (content-addressed blobs, two-level fanout), `checkpoints/` (named protected states → manifest refs), `staging/` (volatile), `journal/` (append-only recovery event log), `external/` (linked-asset records: path, size, mtime, hash; drift reported, never silently substituted). Portable projects copy all CAS+DB; linked projects reference sources in place; conversion linked→portable is a first-class operation (and required for sharing with reviewers).

### 12.2 Save vs export

Save writes the editable project (⌘S / explicit). Export produces a distributable format (adjacent to the projection selector; never labeled "Save as PDF"). Closing with unsaved committed or manual changes prompts Save / Don't Save / Cancel. Automatic crash-recovery writes go to the journal, are *not* user saves, and are surfaced distinctly after abnormal termination ("Recovered work from 14:32 — [view] [discard]").

### 12.3 CAS and handoff

Artifacts (rasters, meshes, heatmaps, model outputs, PDF bytes pre-commit) live in `cas/` keyed by SHA-256; manifests reference them by hash — dedupe across history and checkpoints comes free. Engine handoff directories are symlinks/bindings into CAS-addressed staging; nothing authoritative lives inside an engine's tree (an engine may be wiped and reinstalled without project loss — CI asserts this).

### 12.4 Garbage collection

Collectable = unreachable from: visible history, checkpoints, active staging, journal retention window, recent-replacement grace period (configurable). GC is mark-sweep with a dry-run report UI (what would be freed, by category) before destructive execution.

### 12.5 Migrations

Forward-only, versioned, tested against golden projects; unknown extension fields preserved; migration runs on a scratch copy first with backup snapshot; `schemas/` exports published for module authors.

---

## 13. History, queue, and transactional execution

### 13.1 Linear visible history

Committed states form a linear sequence with full metadata (module/edit name, timestamp, plain-language summary, score delta, scope, warnings). Any state selectable; later states muted-not-hidden; forward return without stepping. **Protected checkpoints** pin full restorability across replaced lines. **Divergence:** mutating an earlier state warns that later *unprotected* visible history will be replaced; options: Cancel / Protect current then continue / Continue and replace. Replaced unprotected states persist in the recovery grace period with an advanced "recently replaced" restore command (branching stays hidden from the normal UI — comprehensibility over git-envy).

### 13.2 Atomic commit protocol

bind immutable input state → open staging transaction → persist private recovery checkpoints (resumable mid-module) → validate output (schema + contract + provenance-complete) → compute semantic/artifact diff → **blobs written and hash-verified before** a single SQLite transaction publishes the manifest row (state + patch + evidence + findings delta) → clean/quarantine staging on failure. A state either exists completely or not at all. Object-level patch preconditions (§11.3) make stale-write corruption detectable.

### 13.3 Queue and cancellation

Item states: `queued · waiting on dependency (incl. ENGINE_WAIT) · running · cancel requested · committing · completed · canceled · failed · blocked · superseded`. Reordering respects declared hard dependencies (invalid moves explained, not just refused). Cancellation: cooperative at the finest declared granularity (region > page > tile > batch); "cancel requested" vs "canceled" both visible; force-terminate after timeout with explicit warning about disposable staging; supervisor verifies engine/worker cleanup (no leaked VRAM — CI-checked) and marks the module's recovery checkpoint valid for resume.

### 13.4 Durability and the journal

All lifecycle decisions and engine supervision events append to the journal (crash-replayable). Power-loss tests (§21.5) at every transaction boundary; SQLite WAL + platform-correct fsync/rename discipline; engine restarts never auto-replay interrupted executions (they are *resumptions* of staged idempotent work or clean failures — never silent retries).

---

## 14. Assessment, planning, and completion

### 14.1 Four distinct statuses

Workflow completion (queue drained) · remediation completion (estimated, below) · conformance results (validator output per projection) · human review status (per-object states §7.3) — never conflated in any UI surface or export artifact. "100 % remediated" means: no unresolved applicable findings under the selected profile and assessor version. It asserts nothing about universal perfection or legal certification, and the UI says so in those words.

### 14.2 The assessor

Deterministic for a fixed (state, assessor version, profile): computes from inspectable findings and category weights — never from a free-form LLM judgment (an LLM may *emit candidate findings* as a critic module; scoring consumes only structured findings with evidence links). Categories (v1): text fidelity (incl. numerics/operators punctuated separately), speech cleanliness, document structure, reading order, navigation, descriptions/alt-text completeness, tables, math, metadata/language, alignment health, geometric drift, projection validation, unresolved uncertainty count. `machine-validated ≠ usable` is enforced by design: conformance results are a separate category from remediation (a document can be 100 % profile-compliant and still carry open fidelity findings).

### 14.3 Findings

`{id, category, severity (blocking|major|minor|informational), object/region refs (with space ids), assessor module, evidence refs, applicable repair modules, state (open|fixed|introduced|dismissed-with-reason|accepted), history}`. Findings are the sole currency between assessment and planning; every repair recommendation cites the finding it serves.

### 14.4 Profiles

Versioned named sets: General Accessible Document · Academic Textbook · Screen-Reader Optimized · PDF/UA-oriented · EPUB-A11y · STEM-with-MathML · institution-defined (profile file format documented; profiles tune applicability rules, weights, and validators). Calibration: each profile's assessor ships with corpus-derived calibration (§20.1) — expected score deltas from known-good/known-bad states are regression-tested like code.

### 14.5 Planning

`inspect()` across applicable modules → candidate recommendations with reasons, scope, dependencies, predicted score delta, and cost estimate (GPU-seconds from validated manifest estimates × current engine tier); a planner (LLM-assisted ranking, *validated* against module-declared contracts — it chooses among declared capabilities, never invents steps) proposes; the user edits/accepts. Every plan ends with reassessment; reassessment reports completion, or a new plan, or "no beneficial action predicted" (the honest plateau — with the residual findings that caused it). Continuous-Auto mode executes successive plans under §9.6 stop rules, cost ceilings, and machine-capability gates (BLOCKED on missing engine tier, §4.3, surfaces as a plan error, not silent skip). Predicted-vs-actual deltas are recorded and fed back to improve cost models (data in `benchmarks/`).

### 14.6 Export report

Sidecar (JSON + human HTML) with: project/document/source ids + hashes; exported state id + timestamp; projection + renderer versions; assessor version + profile + score breakdown; validator outputs (veraPDF/EPUBCheck raw + parsed); open findings incl. `UNRECOVERABLE` inventory; review-status summary; models/recipes/modules used with full provenance (revision hashes, seeds); hardware/OS/dependency manifest (§5.1); known limitations. This is the institutional evidence record; it is generated whether or not anything is "wrong."

### 14.7 Description and equivalence assessment

Alt-text findings use *separate* checks: existence (every non-artifact figure has a classification), form (no "image of", not caption-duplicate, length bounds), and pedagogy (module tests description against surrounding text's references and likely instructional questions — critic-generated findings, human-resolved). Charts add data-extraction cross-checks (do extracted values re-plot within tolerance; log/truncated-axis traps §7.12). A generated description can never satisfy the `approved` review state without a human action — the profile can require this per category.

---

## 15. Projections (renderers)

### 15.1 Projector contract

Each projector declares: **capability matrix** (kernel concept → target mechanism → `full | partial+fallback | dropped+finding`), source maps (every emitted element back to object ids), incremental unit (page/spine-item/chapter), determinism mode (reproducible byte-identical output given same state + version — required for golden tests; per-export randomized ids opt-in), and its validator pair (§6.5). Direct-edit reverse mapping (§16.4) is part of the contract: an element that cannot be reverse-mapped is not editable in that preview. Editing in one projection never degrades the kernel; unportable overrides live namespaced per format, visible in an advanced view, and are *never silently promoted* to semantic facts.

### 15.2 HTML projector (reference, first built)

Semantic HTML5 landmarks + headings + lists + tables (scope/header, caption) + figures (`<figcaption>`, `aria-describedby` long-desc links) + footnotes (doc-noteref/doc-footnote semantics, backlinks) + KaTeX/MathML + page-break markers (`data-page`, page list nav) + language spans + reading order ≡ DOM order (hard invariant; visual positioning via CSS, never DOM-order tricks) + assistive-text wiring (aria-label/aria-description from §7.4 variants) + `unrecoverable` spans (§7.8). Doubles as: EPUB spine source, AT test surface (browser + VoiceOver/Orca/NVDA-in-browser), and the fastest feedback loop for structure errors.

### 15.3 Tagged PDF projector (the hard spine — architecture normative)

Page content model: each exported page = (1) cleaned image surface (per selected quality/size policy: lossless for accessibility-first exports, configurable JPEG2000/JPEG quality for distribution; the *source* stays lossless in CAS), (2) invisible text layer generated from current alignment (§7.6): shaped runs (uharfbuzz) → `BT/TJ/ET` operators, font = subset (fontTools) of the pinned OFL face, `Tr 3` render mode, correct ToUnicode, (3) marked-content structure: the projector emits `/StructElem` trees whose content lists anchor `BDC … <</MCID n>> EMC` spans *around the actual operators* for that element — the kernel's object ids ↔ MCID table is written into the project state (export audit + re-mapping requirement), so structure is never "attached after the fact." Artifacts marked `/Artifact` (furniture, decorative rules). Notes as `Reference`+`Note` with destinations and `/R` back-links; tables with TH/td spans and `THead/TFoot`; figures `Figure` with `/Alt` (+`/LongDescAlt` object or `/E` where appropriate); equations `Formula` with `/Alt` + linearized `/ActualText`; `/Lang` per element where it varies; document catalog: `/MarkInfo /Marked true`, `/Lang`, `/StructTreeRoot`, `/Tabs /S` (structure order), outlines, page labels for printed folios (§27.2), XMP identification per the PDF/UA-1 schema (`pdfuaid:part`) alongside `/MarkInfo /Marked` — UA-2 observation metadata is out of scope (§31.1). Validation loop: veraPDF (`ua1` flavour) findings must be zero-blocking at export time *when the profile requires it*; failures are reported (never silent), and regression suites assert byte-level structure expectations on golden pages. **Feasibility gate:** the Phase-0 walking skeleton (§20.2) is explicitly the experiment that confirms this section is implementable with the §5 stack (pypdfium2 read-only + pikepdf write-only); if it is not, ADR-006/ADR-014 reopen *before* app investment, and the finding is recorded with evidence. Deferred-but-architecturally-allowed: tagged 3D, forms, embedded media.

### 15.4 EPUB 3 projector

From HTML projection: spine/landmarks/nav (TOC + page-list), `properties: mathml`, accessibility metadata per EPUB Accessibility 1.1 — a `dcterms:conformsTo` declaration plus the required `schema:accessMode`, `accessibilityFeature`, and `accessibilityHazard`, with `accessModeSufficient` and `accessibilitySummary`, and certification properties (`a11y:certifiedBy`, `a11y:certifierCredential`, `a11y:certifierReport`) emitted only where a real certification exists (§31.1, §31.3), media-overlay hooks left as clean extension points (DAISY 4 audio-sync is post-v1), fixed-layout *not* in v1 (reflow only; the visual-fidelity readers are served by the PDF projection — documented as a projection-family decision, §3-principle 6). EPUBCheck gates (§6.5).

### 15.5 Semantic package export

Canonical JSON-lines objects + CAS subset + manifests; versioned; round-trip tested (export → import → re-export ≡ same semantic digest, §21.3); this is also the archival target.

---

## 16. Application UI

### 16.1 Requirements posture

The UI itself meets WCAG 2.2 AA (automated axe-core gates in CI + manual AT passes) — an accessibility tool that fails its own audit is a product defect, not embarrassment. Every pointer/drag interaction (queue reorder, mask paint, overlay drag) has keyboard and nonvisual equivalents; modal focus management; reduced-motion; scalable text; non-color status.

### 16.2 Screens (carried structure)

Startup (open/create/recent/recovery) → workspace: large preview pane; sidebar tabs (Overview · History · Remediation queue · Structure tree · Metadata · Findings · Export checks); top bar: projection-format selector, distinct **Export** button, persistent remediation-completion indicator (linked to the *same* immutable assessment record the reassessment shows — one number, one truth), engine status chip (§16.7).

### 16.3 Previews and overlays

Format-appropriate: PDF preview via pdf.js (tag-view when available); HTML preview = rendered projection; EPUB preview = reflowable reader simulation with page-list; stale previews visibly labeled with their state id. Overlays (toggleable, keyboard-navigable lists as nonvisual alternatives): source↔current (side-by-side/sync/reveal-slider), semantic regions, reading order with jump, confidence/uncertainty heatmap, repair-mask indicators (which regions are AI-reconstructed — principle: *visual success disarms review; reconstructed regions stay flagged*, §17.5), text↔pixel alignment debug view, `UNRECOVERABLE` inventory, note-link graph.

### 16.4 Editing

Tiptap/ProseMirror editors bound to kernel slices through the projection's source maps: text (canonical/assistive tabs), structure tree drag (with validation preview before commit), reading-order path editor (with "what AT would say" simulator pane), figure descriptions (per-variant), table grid editor (spans/headers), note markers/links, page markers, metadata. Manual edits batch into transactions (§13.2) with plain-language summaries; local undo/redo inside an editing session; edits made in a *projection view* are translated per §15.1 (semantic vs presentation vs override classification, shown to the user at commit time in an "effects" disclosure). Editing locks while modules run (v1): inspection, search, save/export-of-last-committed, queue reordering, comments remain available.

### 16.5 Search and inspection

Project-wide over canonical/assistive text, metadata, roles, findings, alternatives, review states; filters (low-confidence, disputed, unrecovered, unrecoverable, complex tables, math, failed checks, AI-reconstructed, protected, changed-by-state-N); result click lands on exact preview location + object context.

### 16.6 Settings

Categories: General · Projects/storage (paths, linked-vs-portable defaults, GC) · Appearance/accessibility · Engines (versions, status, reinstall, VRAM budgets) · Model assignments (role → pinned weight, with license/size/capability display *before* download; model browser lists registry entries only) · Modules (trust states, permissions granted, versions) · Processing/hardware (quality presets: CPU-compatible / low-VRAM / balanced / maximum-quality / blocked-degraded) · Privacy/network (§17.2) · Validation profiles · Recovery retention · Advanced/diagnostics.

### 16.7 Engine status panel

Per engine: state, version pin health, VRAM reserved/free, queue stalls (`ENGINE_WAIT`), restart control, and per-run resource history. One-click diagnostic bundle (redacted, user-reviewed diff before send).

### 16.8 Observability

Current module/work unit, pages/regions done/total, throughput, ETA, live resource usage (core + engines + module workers), structured redacted logs, progress events durable in journal (§13.4).

---

## 17. Security, privacy, and content policy

### 17.1 Localhost exposure

Loopback bind only; per-launch random token required on all mutating and reading API calls (cookie + header, no token-in-URL persistence beyond first handoff); strict `Host`/`Origin` validation, CORS disabled; CSP without `unsafe-inline`; the UI never receives filesystem paths beyond project-scoped handles; engine ports likewise unannounced, with vLLM's api-key and ComfyUI loopback-only config enforced by the supervisor. Threat model includes drive-by webpages targeting local ports — mitigations and residuals documented in `SECURITY.md`.

### 17.2 Data egress

None by default. First-run asset acquisition (engine installs, weight downloads from pinned revisions with hash verification) is explicit, user-initiated, itemized, and offline-capable via import-from-directory. Remote model endpoints are not implemented (ADR-005); the OpenAI-compatible seam stays *internal* (vLLM's API), not a client.

### 17.3 Hostile-input hygiene

PDFs, archives, model files, EPUB/HTML content, and modules are treated as hostile until parsed: zip-slip + decompression-bomb limits; parser sandboxing (docling runs in its own process with seccomp-style limits where OS permits); no script execution from imported content (HTML projections are sanitized; previews in sandboxed frames with `allow-scripts` off); safe serialization formats preferred (safetensors; pickle-weighted models refused by the registry).

### 17.4 Multi-tenant caution

Documented: any user account on the machine can attempt loopback connections during a session; single-user assumption stated in UI at setup; token mitigates passive attacks only.

### 17.5 Content-integrity and privacy policy (product-level, normative)

- **PII:** handwritten annotations and stamps may contain personal data — retained locally, never transmitted (there is no transmission path), excluded from shared corpora by default, redaction tooling provided for exports shared outside the institution.
- **Provenance integrity:** the pipeline must not remove publisher copyright notices, library stamps, ownership marks, or watermarks *by default*; restoration masks protect them; automated removal is blocked when detected (classifier → finding → explicit user decision with recorded rationale). Note this is an ethical+legal posture, not a legal opinion; institutions configure policy.
- **Auditability:** the evidence graph + journal answer "was this sentence machine-generated or source-supported?" per claim.
- **AT-review honesty:** reconstructed regions carry persistent flags (§16.3) precisely because repaired imagery *invites* less skepticism than visibly degraded imagery; review-queue sampling *weights* reconstructed regions upward (default 3×, configurable).

### 17.6 Supply chain

uv/pnpm lockfiles with hashes; SBoM (CycloneDX) built into release artifacts and the export report; dependency license scan in CI against §5.8 policy (AGPL/non-commercial = build failure); engine installs verify pinned commits and signatures where upstream provides them.

---

## 18. Performance and scale

### 18.1 Book scale

Target: 600-page textbook on reference hardware. Rasters never all resident: page streaming with lazy CAS loads; module work units bounded (per-region/per-page); chapter-granular caches. SQLite handles 10⁶-object states with indexed queries (benchmarked in CI on synthetic 1000-page projects).

### 18.2 Engine economics

VRAM ledger with reservation + headroom (§6.7); swap cost modeled (cold-start penalties appear in plan estimates); batch where semantics allow (vLLM continuous batching; ComfyUI batch recipes with fixed parameters). Deterministic presets hide this complexity from users but never from the cost model (§14.5).

### 18.3 Preview economics

Incremental projection by changed page/chapter (state-hash keys); stale labeling (§16.3); pdf.js worker per preview; large overlays rendered to canvas, not DOM.

### 18.4 Resolution ladder

`source → working (default 300-DPI-equivalent or source if lower) → engine (per-recipe declared) → export (policy-selected)` with ladder position recorded on every artifact; upsampling above source-information limits (§24-R6) is refused by default with a finding (this is the anti-fabrication rule for imagery, §7.8's twin).

### 18.5 Throughput budgets

Phase-0 bakes reference numbers into `benchmarks/BASELINE.md` (GPU-sec/page per module, wall-clock end-to-end for the golden 600-page projection) and every release regresses against them (±20 % alarm). Cost ceilings (§9.6) derive from these.

---

## 19. Repository layout

```text
remediator/
  SPEC.md  README.md  LICENSE  SECURITY.md  CHANGELOG.md
  pyproject.toml  uv.lock            # core (CPU-only) — torch/CUDA import CI-blocked (§5.4)
  engines/
    comfy/   recipes/<name>/{recipe.toml,workflow.json,smoke.py}  nodes/<companion-pack>/
    vllm/    registry.toml  probes/
    docling/ contract/               # pinned payloads for startup contract test
  app/
    api/  services/  domain/  persistence/  workflow/  engine_supervisor/
    assessors/  importers/  renderers/{html,pdf,epub}/  model_broker/
  schemas/                            # kernel + module-protocol + contracts (generated)
  modules/<built_in_module>/          # same layout as third-party (no privileged path)
  module_sdk/                         # client lib modules use in their venvs
  fonts/  models/MODELS.toml
  recipes/                            # non-comfy plan templates (presets)
  evals/
    corpus/     MANIFEST.toml, items/<id>/ (rights metadata, §20.1)
    harness/    metrics/, degrade/ (synthetic degradation), goldens/
    results/    BENCHMARKS.jsonl
  frontend/                           # React app (pnpm workspace)
  tests/ unit/ contract/ integration/ golden_projects/ at_smoke/ security/
  ci/  docs/  scripts/  a11y/
```

The core domain package depends on no web or engine code; renderers, importers, and modules depend on `schemas/` contracts, never on internal services (§21.2 contract tests enforce layering mechanically).

---

## 20. Phasing, benchmark corpus, and de-risking gates

*Reading map for the remainder of this document.* Sections 0–19 forward-reference a tail that the drafting pass had not yet written; §20–§24 supply exactly what those references name (§20 phasing and corpus, §21 verification, §22 delivery and operations, §23 ADRs, §24 risks). §25–§32 close requirement gaps found while completing them; each names the earlier section it refines, and none relaxes an earlier normative statement — a relaxation requires amending that statement.

### 20.1 The benchmark corpus (`evals/corpus/`)

The corpus is a **prerequisite, not a test asset**: §7.7 calibration, §9.2 model acceptance, §14.4 assessor calibration, §18.5 throughput budgets, and §9.7's cost curve are all functions of it, so no module boundary can be honestly drawn before it exists. It is versioned in-repo by manifest; the bytes themselves may be held outside the repository (rights, size) and fetched by manifest id into a local corpus root.

**Item record.** `evals/corpus/MANIFEST.toml` lists items; each `items/<id>/` carries `rights.toml`:

```toml
id = "ct-kapoor-1998-ch03"
rights_basis = "publisher_license"     # public_domain | cc_by | publisher_license | institutional_agreement | synthetic_only
permitted_uses = ["evaluate", "publish_aggregate_results"]   # train | tune | evaluate | redistribute | publish_results | show_in_ui
custody = "institution:A, agreement:2026-04"
contributor = "…"; deidentifiable = true
license_ref = "docs/rights/publisher-a.md"; notes = "no redistribution of page images"
```

Normative: **no item is admissible without an explicit `permitted_uses` set**, and the evaluation harness refuses any item whose set lacks the use being attempted (CI-enforced, §21.2). Items marked `evaluate`-only may not appear in fixtures, golden projects, screenshots, documentation examples, or bug bundles (§17.5), and never leave the machine (§17.2). This is the control that makes a copyrighted-textbook corpus lawful to keep.

**Composition floors** (minimums per stratum; more is always admissible, and each stratum must be represented across at least two publishers/disciplines so no single book's typography masquerades as a general result): low-resolution prose (≤ 200 DPI source); photocopy artifacts — gutter shadow, bleed-through, grime, uneven illumination; curvature and skew; handwritten underlines/marginalia (classification ground truth, §27.5); old or unusual typography, including two-color/spot-color printing (§8.2); multi-column and atypical flows; footnotes, endnotes, sidebars, pull quotes; simple and irregular tables (≥ 15 of each); charts including log/truncated/dual-axis and error bands (§14.7); mathematics and scientific notation (≥ 40 expressions, ≥ 5 multi-line); damaged illustrations and broken letterforms (with deliberately ambiguous glyph regions, see metrics); born-digital untagged PDF; hybrid PDF; defective-OCR text layers (real and synthesized); RTL and CJK samples (so §8.2's script capability gate is falsifiable); and prior Remediator packages for round-trip (§15.5). Each page with a known-good reference accessible output carries all three projections' gold reference where the rights basis permits.

**Gold protocol.** Prose transcription is double-annotated independently and adjudicated; role/structure labeling likewise. Acceptance floors: word-level agreement κ ≥ 0.8 before adjudication, region-boundary IoU ≥ 0.9 on adjudicated geometry, and **ambiguity preserved rather than flattened** — where annotators disagree, the gold record stores an alternative set with per-alternative support, because a corpus that forces a coin flip trains and scores exactly the failure §7.8 exists to prevent.

**Synthetic degradation** (`evals/harness/degrade/`). A declared, seeded operator set — rescale, blur, JPEG chain (×1…×4 generations), illumination gradient, gutter shadow, bleed-through, speckle/salt-pepper, curvature warp, local ink loss, glyph damage by stroke removal, and two-color flattening — each composable, each writing its parameter vector into the pair's metadata. Rationale beyond test data: §9.5's "degrade each hypothesis under the *observed* degradation model" is only measurable if degradation is a first-class, recorded operation; and `synthetic_only` items let lawfully-obtainable clean documents expand the corpus where licensed scans cannot (§24-R14).

**Metrics (never collapsed into one number).** Character/word error reported as separate tracks for ordinary words, numerals, operators and scientific symbols, punctuation, diacritics, and abbreviations (§14.2 punctuates numerics separately; a single WER hides the errors that matter most in textbooks); table structure as cell-grid F1 plus span-closure correctness plus header-association correctness; equation validity as parse success × symbol exactness × symbol-level render-back similarity (§28.2); reading order as edit distance over the adjudicated region sequence; navigation completeness and note-link integrity as per-item boolean rates; description pedagogy as a rubric scored by two humans with model assistance permitted for triage only (scoring is not a free-form LLM judgment, §14.2); alignment quality as box IoU, baseline offset distribution, and ToUnicode copy/paste round-trip rate; drift as region-level smoothness delta against neighbours (§9.3 degradation-honesty); projection equivalence loss per §21.3; and cost as GPU-seconds and wall-clock per module per page class.
**False-certainty rate** is a first-class metric on every ambiguous stratum: the share of confidently-wrong claims at calibrated confidence ≥ the threshold. It is the number §24-R2 is about; a model or pipeline revision that raises aggregate accuracy while raising false certainty **fails** acceptance.

**Calibration sets.** Per §7.7 and §14.4: paired known-good / known-bad states (constructed by applying the degradation operators to gold states and by scripted structural damage) with *expected* score deltas and expected finding classes; per-role reliability curves (ECE and per-bin counts) computed on held-out items and versioned with the assessor. Held-out discipline: splits are by publication, never by page, and one edition may not appear in both a tuning set and an evaluation set.

**Bake-off acceptance (§9.2).** For each role, the registry entry records `measured_capability` = the metric vector from the run that selected it, plus pass/minimum/target thresholds and the tie-break order: (1) meets thresholds on the role's primary metrics; (2) license unambiguously admissible under §5.8 *as verified against the weights artifact, not the code repository* (§5.7, §30.2); (3) fits the reference tier's VRAM reservation with headroom (§18.2); (4) lower GPU-seconds/page; (5) smaller download. Ties resolve to the **smaller, faster** model — this is a single-user workstation product, and a 0.3 % accuracy gain that costs a 2× VRAM reservation is not a gain for the actual user. Vendor-card benchmarks are screening evidence only (§9.2). The full bake-off re-runs at every weight or engine pin change (§20.4), and results append to `evals/results/BENCHMARKS.jsonl` with corpus version, engine revision, and hardware id, so any number in a report is re-derivable.

### 20.2 Phase-0 deliverables and the walking skeleton

Phase 0 produces four artifacts and no product code:

1. **Corpus + harness** (§20.1).
2. **Kernel prototype** — Pydantic models, JSON Schema export, SpaceRegistry/transform ledger with composed transport, evidence graph with R1–R4 enforcement, the §13.2 commit protocol against a stub SQLite+CAS, and the extension registry (§7.13). Its purpose is to discover whether the §7 model is *expressible* and whether the ledger composes invertibly under real deskew and dewarp data.
3. **Bake-off** of candidate weights per role (§9.2), producing the first `models/MODELS.toml`.
4. **Walking skeleton** (below).

**Walking skeleton: scope.** ≤ 32 pages from one degraded scan (front matter with roman folios, ≥ 20 body pages containing one table, one display equation, footnotes, one sidebar, one damaged region, one handwritten annotation) plus one born-digital untagged PDF and one defective-text-layer PDF. A script, a debug HTML page, and real engines; **no** application UI, **no** queue/scheduler, **no** migrations, **no** packaging. Those exclusions are deliberate: the phase exists to falsify architecture assumptions at minimum cost, and scaffolding would hide *which* assumption broke.

**The ten assertions it must settle** (each with a mechanical pass test, recorded with evidence in `benchmarks/`):

| # | Assertion | Pass test |
|---|---|---|
| A1 | pypdfium2 read-only + pikepdf write-only suffices; no third PDF library is needed (§6.6, ADR-006) | an exported page opens, renders, and yields its structure tree using only those two |
| A2 | `/StructElem` trees anchored by `BDC … /MCID … EMC` around the real operators pass veraPDF `ua1` with zero blocking findings | §6.5 invocation on the skeleton's output |
| A3 | The invisible text layer extracts back to the intended Unicode (copy/paste fidelity) including diacritics, ligature decomposition, and one RTL run | text extracted from the exported page ≡ canonical text for the sampled runs, ToUnicode-verified (§29.3) |
| A4 | One accepted deskew transform correctly marks dependent alignments `STALE`, and re-alignment restores them (§7.5, §7.6) | pre/post transform box comparison within tolerance |
| A5 | Two model *families* traverse one vLLM seam plus per-family adapters and yield comparable Recognition Results; disagreement surfaces as `suspicious_spans` | §6.3.2 adapter contract; ensemble `n_eff ≥ 2` on the target share of claims |
| A6 | A mask-bounded, text-conditioned glyph repair records R4 lineage **mechanically**, so OCR of the repaired region carries weight 0 for the conditioned claim | evidence-graph query on the repair's output claim |
| A7 | Counterfactual rendering (§9.5) selects the correct reading on the deliberately-ambiguous stratum above chance by the recorded floor, and abstains otherwise | corpus run against gold alternative sets |
| A8 | The commit protocol survives `SIGKILL` at every §13.2 boundary with zero corruption and no manual repair | §21.5 kill matrix |
| A9 | Per-page cost is measurable to ±25 % against §18.5's `BASELINE.md` seeds | repeated timed runs |
| A10 | No output artifact can reach staging without a complete provenance record (§6.1) | negative test: strip provenance → staging rejects |

**Gate and exits.** A1–A3 constitute the §15.3 feasibility gate. If any fails, the permitted responses are exactly: (a) reopen ADR-014 (in-house PDF writer), (b) reopen ADR-006 (AGPL exclusion, which is what forecloses PyMuPDF as the writer), or (c) amend §15.3's page-content model — each requiring a superseding ADR with the skeleton's evidence attached (§32.2). Reopening *before* application scaffolding is the point of the ordering; after Phase 1 the same finding costs orders of magnitude more. A4–A10 failures do not stop Phase 1 but must be attached to the §24 entries they aggravate, with the mitigation re-specified.

### 20.3 Phases

| Phase | Builds | Exit criterion (all required) |
|---|---|---|
| **0 Evidence** | §20.2 | A1–A10 recorded; corpus v1 with floors met; `MODELS.toml` v1; `BASELINE.md` seeds |
| **1 Foundation** | Project package (§12), kernel + persistence + commit protocol (§7, §13.2), importers for scanned-PDF / image-directory / born-digital (§8), HTML projector (§15.2), minimal launcher and UI shell (§4.4, §16.2), journal + recovery (§13.4) | §22.10 foundation acceptance criteria; golden projects exist; the prototype schema is *not* grandfathered (Phase 1 ships format v1.0.0) |
| **2 Workflow & modules** | Module SDK + manifest + isolation (§11), queue/cancel/recovery (§13.3), engine supervisor + VRAM ledger (§6.7), model broker (§30), triage probes (§8.2) | a contract-conformant third-party module installs, runs, fails, resumes, and is refused when it writes undeclared geometry — all under test (§21.2) |
| **3 AI-native MVP** | §9 passes 1–8, critics (§9.3), BookProfile (§10), assessor + planner + reassessment (§14), tagged PDF projector (§15.3), veraPDF loop | end-to-end on a corpus book to export with report; §9.6 stop rules exercised; findings→plan→repair→reassess closes on the ambiguous strata without raising false certainty |
| **4 Projections & editing** | EPUB projector (§15.4), editor + reverse mapping (§15.1, §16.4), overlays (§16.3), search (§26.4), semantic package (§15.5) | equivalence-loss matrix green (§21.3); every editable projection element reverse-maps |
| **5 Depth & hardening** | Specialist modules (§28), Continuous Auto (§14.5), storage/compaction (§22.6), AT matrix (§21.6), Windows tier, performance to §18 budgets | §22.10 v1 definition of done; §24 register reviewed with each residual explicitly accepted |

Phase order encodes two decisions worth their reasons: **HTML is built first** because it is the reference surface, the AT test surface, and the fastest structural feedback loop (§15.2); **PDF is de-risked first** — in Phase 0, before either — because it is the one projection whose *possibility*, not quality, is uncertain with the §5 stack, and a format the system cannot write invalidates the projection architecture of §7, §13, and §15 at once.

### 20.4 Upgrade re-validation (normative)

Pins do not rotate freely. A change to any **E** or **J** component, to `MODELS.toml`, to a recipe, or to an assessor version requires, in order: contract tests at the new pin (§21.2) → affected bake-off roles re-run (§20.1) → calibration re-check (§14.4) → full regression gate (§21.7) → §18.5 throughput comparison within the ±20 % alarm → CHANGELOG entry and a report-visible version bump (§14.6). A pin whose re-validation changes any score materially re-versions the assessor (§14.2) rather than silently re-scoring old projects; existing assessment records stay reproducible with the version that produced them.

---

## 21. Verification and quality engineering

### 21.1 Layers

`unit` (pure functions: schema validation, geometry transport and inversion, patch application, evidence-graph rules, scoring arithmetic) → `property` (invariants, below) → `contract` (§21.2) → `integration` (core + pinned engines on the reference tier) → `golden` (§21.4) → `e2e` (import→plan→run→assess→export) → `at_smoke` (§21.6) → `security` (§21.9) → `model_regression` (the §20.1 bake-off subset, nightly).

Property-based tests are mandatory at the four places where a wrong invariant corrupts data silently instead of failing loudly: (i) ledger transport composes and round-trips (`to_space ∘ from_space = identity` within a stated tolerance; dewarp via inverse-mesh lookup); (ii) patch preconditions reject stale writes and never partially apply; (iii) evidence rules R1–R4 are closed under graph mutation — adding a descendant never increases an ancestor's support; (iv) §7.9 reading-order validation is deterministic and idempotent on a given commit.

### 21.2 Contract and architecture tests

Mechanical layering enforcement, so §19's claims cannot decay: import-graph rules block `torch`/`tensorflow`/`paddle`/`onnxruntime` and any model-hub or AGPL-exposed PDF library from the core environment (asserted both by a test that attempts the import and by lockfile/SBoM scan, §17.6); the core must not import module or engine code; `renderers/`, `importers/`, and `modules/` may depend on `schemas/` contracts only, never on `app/services/` internals; `app/domain/` depends on neither web nor engine code. Violations are build failures, not review comments — ADR-003 is worthless as a convention.

Engine contracts are the **executable form of §6**: per-engine pinned-revision request/response fixtures in `engines/<x>/contract/` (including `engines/docling/contract/CONTRACTS.md`), plus live GPU-lane probes covering readiness, capability probe sets (§6.3.1), cancellation semantics, and a negative test that provokes each §6.1 `EngineError` code. Module-protocol conformance runs the SDK suite against a reference module in the same job that builds the SDK. Recipe manifests are validated by loading the graph, resolving every mapped input **by name** against the per-node object-info route at the pinned revision, and rejecting positional widget indices (§6.2.1, §25.1). Schema/codegen drift between Pydantic models and the frontend's generated types is a build failure (ADR-020's cost, paid in CI).

PDF output is asserted twice: byte/structure-level goldens on fixed pages (MCID table, `/StructTreeRoot` shape, XMP identification, font subset presence, `/Artifact` marking), and semantic re-derivation — parse the projector's own output with pypdfium2 (read-only) and compare recovered structure and extracted text against the kernel state. Self-parsing is deliberate: it tests what a consumer sees, not what the writer intended.

### 21.3 Round-trip and projection equivalence

**Semantic digest** (required before §15.5's "same semantic digest" can be tested): canonical serialization of kernel state — object records ordered by stable id, map keys sorted, artifact references expressed as content hashes, volatile fields excluded by an allow-list (`*_at` timestamps, engine wall-clock timings, preview caches, journal offsets), extension bags included verbatim (§26.6). `export → import → re-export` must reproduce the digest byte-for-byte; the digest is written into the export report (§14.6) so a re-imported package is provably the same work.

**Equivalence-loss tests** instantiate §15.1's capability matrix as data: every kernel concept must appear in each projector's matrix as `full`, `partial+fallback`, or `dropped+finding`. A concept absent from the matrix fails the build; a `partial` row must demonstrate its fallback in a golden projection; a `dropped` row must produce its finding. This is what keeps §7 honest — richness may not be lost silently by never having been enumerated. The §7.4 variant-substitution rule is checked the same way (which variant lands in `/Alt` vs `/ActualText` vs real text vs `aria-label` vs content), as is assistive-text reversibility (§7.4 diffs revert).

### 21.4 Golden projects and migrations

A golden project is a real `content.db` + `cas/` + fixtures with expected digests: one per released project-format version (§12.5), plus a synthetic 1000-page project for §18.1 scale queries. Every migration runs against every golden: forward-only, on a scratch copy, with the backup snapshot asserted; unknown `ext:` bags survive byte-identically; module-published schemas match what the module wrote; module migration declarations (§11.2) are honoured or the project refuses to open with an explanation. Institutional gold projects (§14.4's known-good/known-bad pairs) double as score-stability fixtures.

### 21.5 Durability and failure injection

The kill matrix terminates the core, the module worker, or the engine — `SIGKILL`, plus hard power-loss emulation where the OS permits — at every boundary in §13.2 and §6.1, then asserts: the last committed state is intact and openable; staging is disposable or quarantined; no published manifest row exists without hash-verified blobs; unreferenced blobs are collectible rather than corrupting (§12.4); journal replay reaches the pre-crash state. It also includes a backward clock step at a transaction boundary (§26.5). Resource cleanup is asserted, not hoped for: after cancellation and after forced termination, VRAM returns to baseline, no child process or bound port remains, and the `/free` follow-up is confirmed by a measured free-VRAM delta (§6.7). Determinism: same state + projector version + profile + settings ⇒ byte-identical output (per-export randomized ids excluded, §15.1). Engine restarts never auto-replay an interrupted execution (§13.4) — tested by killing an engine mid-run and asserting the queue surfaces a resumable or cleanly-failed item, never a silent retry.

### 21.6 Application accessibility and AT verification

The tool audits itself (§16.1): axe-core over every screen and state — including error, empty, blocked-tier, and mid-run states; keyboard-only traversal of all §16.2 workflows; focus management and modal traps; non-color status; reduced motion; 200 % text. Every overlay in §16.3 must have a tested nonvisual equivalent list; an untested list counts as unimplemented. AT runs use real combinations (NVDA + Firefox/Chrome, JAWS + Chrome, VoiceOver + Safari, Orca + Firefox) across the reference UI state set, scripted where APIs permit and following documented manual protocols in `a11y/` otherwise, with results recorded per release.

Separately, **golden projections are opened in real consumers** — Acrobat Reader, Chrome's PDF viewer, major EPUB reading systems, browser + AT for HTML — and observed behavior is recorded as a compatibility matrix (§24-R19). Validator success never substitutes for this: §14.2's `machine-validated ≠ usable` is enforced by making the AT matrix a release gate rather than an aspiration.

### 21.7 CI topology and the full regression gate

Lane 1 **hermetic PR lane** (CPU only, no GPU; engines replayed from recorded contract fixtures plus a stub engine implementing the §6 surfaces): unit, property, offline contract, layering, golden, migrations, license/SBoM, frontend build, axe-core. It must run on any contributor machine, because a suite that needs CUDA stops being a gate. Lane 2 **reference-tier GPU lane** (Linux/CUDA ≥ 12.x, 24 GB): live engine contracts, integration, §20.2-shaped end-to-end on a rights-cleared subset, cancellation and VRAM cleanup. Lane 3 **partial tier** (macOS): core plus ComfyUI/MPS paths and §4.3's BLOCKED reporting. Lane 4 **nightly model lane**: bake-off subset, calibration refresh, §18.5 throughput, corpus hygiene (rights fields present, held-out splits disjoint). Lane 5 **release lane**: everything, on release candidates, plus §21.6 AT passes.

The **full regression gate** that §5.1 requires before a pin lands = lanes 1–3 + lane 5's license/SBoM and AT passes, green on the candidate, with lane 4's model metrics inside §20.1 thresholds and §18.5's ±20 % band.

### 21.8 Release-blocking defect classes

No release ships with open defects in: state corruption or unrecoverable history; evidence-graph bypass (R1–R4); silent content loss in a projection; fabricated content presented as recovered (§7.8, §18.4, §24-R6); license or SBoM violation (§5.8); secret leakage into logs, UI, or exports (§17.1); any path where the UI blocks export contrary to principle 8; any §21.6 keyboard or AT blocker for a core workflow; and any data-loss risk in GC (§12.4) or compaction (§22.6). Everything else is a finding to report, not a gate to hold.

### 21.9 Security tests

Loopback threat-model cases (§17.1, §17.4): missing or wrong token on read and mutate routes; `Host`/`Origin` spoofing; DNS-rebinding-style `Host` values; CSRF-shaped cross-origin POSTs; a browser-driven scan of the engine port range; token absence from URLs after first handoff. Hostile-input corpus (§17.3): zip-slip and decompression-bomb archives, deeply nested and truncated PDFs, font bombs, oversized images, malformed JSON from a compromised engine, pickled weights (must be refused), recipe graphs attempting path traversal through the view route, and manifests claiming out-of-scope permissions. Module containment (§11.4): a hostile reference module attempting undeclared filesystem writes, network egress, live-DB access, and ledger-less geometry edits must be caught by `validate` and quarantined without touching the committed state. Comment/decision redaction (§26.3) is tested against a portable-project handoff fixture, so a shared project demonstrably carries no private annotation.

---

## 22. Delivery, installation, and operations

### 22.1 Shipped artifacts

Release artifacts, all signed and hash-published: (1) the **core application** per OS (Linux relocatable `.tar.gz` bundle plus a `.desktop` entry, Windows signed installer, macOS signed and notarized `.dmg`), containing the CPU-only Python environment from `uv.lock` and the built frontend bundle; (2) **engine packs** as separately-licensed, separately-installed components (ComfyUI at its pinned commit, vLLM, docling-serve — each with its own venv and its own license file), never merged into the core tree. This is ADR-010's firewall expressed as a packaging boundary, and it is also what lets the macOS tier install fewer engines (§4.3); (3) the **CPU toolchain pack** (Temurin 17 runtime plus hash-verified veraPDF and EPUBCheck artifacts into `tools/`, §6.5); (4) **model packs**, one per `MODELS.toml` entry, containing weights only, with license text, revision, and hash; (5) the **font pack** per `fonts/MANIFEST.toml` (the pinned OFL face with its license and RFC-name handling, §31.4); (6) `NOTICE/` — full third-party license texts, source-availability pointers for the copyleft components, and the SBoM (CycloneDX JSON, §17.6).

Nothing auto-updates. The offline path is first-class: every item above is obtainable from an operator-supplied directory or bundle, and setup verifies hashes without network access (§17.2). An installation missing engine packs is a *valid* installation that reports capability honestly (§4.3, §14.5).

### 22.2 First-run setup

A resumable state machine, each step idempotent and independently re-runnable: **probe** hardware and OS → resolve the capability tier from §4.3's matrix and persist `capabilities.json` (this file, not runtime guessing, is what the planner and the quality presets read) → **verify prerequisites** (driver and CUDA level, MPS availability, Java, disk, Windows long-path support) → **install engines** (create venv, import the pinned commit or fetch and verify it, install the node pack, run the contract probe; a failed step leaves the engine `down` with its evidence attached, never half-registered) → **install weights** (license, size, and measured capability shown *before* download, §16.6; hash-verified; pickle-only weights refused, §17.3) → **install tools and fonts** → **run the readiness suite** (§4.4) and show a per-item pass/fail panel. Each step writes to the setup log; any step can be re-executed from a Settings control (§16.6 "reinstall"). Canceling mid-setup never produces a state that reports itself installed.

### 22.3 Application home

`~/.adrc-remediator/` — the path §4.4 names, used on every platform, per-user and never shared:

```text
config.toml            # settings per §16.6; no secrets inline
capabilities.json      # §22.2 probe result; drives tier and presets
engines/<name>/        # pinned installs, own venvs, own licenses
models/                # weight cache by registry id + revision hash
fonts/                 # resolved OFL face + license + notice
tools/                 # J tier: Temurin runtime, veraPDF, EPUBCheck (hash-verified)
tokens/                # per-launch core token and engine api keys — 0600, never in project dirs
logs/                  # rotating, size-capped, redacted (§22.7)
cache/                 # regenerable only: previews, projection cache, contract fixtures, indexes
tmp/                   # scratch; wiped on clean shutdown, disposable after a crash
```

Normative separations: **project data never lives here** — a project owns its directory (§12.1), so handoff and archival work by file copy (§2.1); **tokens and keys never live in a project directory** (§17.1), so a portable project shared with a reviewer cannot leak an installation's credentials. Deleting `cache/` or `tmp/` costs time only; deleting `models/` or `engines/` is recoverable only by re-running setup, which the UI states before any bulk disk cleanup (§22.6).

### 22.4 Updates, downgrade, and pin rotation

Updates are explicit and staged: fetch → verify → keep the previous version intact → run migrations on a scratch copy of each affected project with a backup snapshot (§12.5) → publish. A failed update rolls the launcher pointer back and leaves projects readable by the prior version; because migrations are forward-only, the supported downgrade window is defined by §22.9 rather than assumed. Engine, weight, recipe, and assessor changes follow §20.4's re-validation sequence in full. No component rotates a pin silently, and the export report records the exact set that was used (§14.6), so a historical export can be re-explained years later.

### 22.5 Instance and project locking

One core per port; a second launcher on a busy port uses the fallback range (§4.4) and is a genuinely separate process, visible as its own instance. Each opened project takes `project.lock` beside `project.toml` holding `{hostname, boot_id, pid, start_time, instance_token_hash, heartbeat_at}`, refreshed periodically and released on shutdown and on exit hooks. Open is refused — with a plain-language message and a read-only affordance — unless the lock is provably stale (`boot_id` changed, `pid` absent, or heartbeat older than the stale window); stale acquisition is recorded in the project's own event table, so "who had this open at 14:32" is answerable. **Read-only open mode is required, not optional:** reviewers receive portable copies (§2.1) and must be able to inspect with no possibility of mutation. Replacing or moving a project directory while it is open is detected through the lock's identity fields and reported as drift (§12.1's `external/` discipline), never silently absorbed. This is §4.4's "stale lock has a defined, non-corrupting outcome" made concrete.

### 22.6 Storage management

Every artifact is classifiable as **required** (source rasters, committed-state CAS blobs, `content.db`, journal within retention) or **regenerable** (previews, projection caches, search and repetition indexes (§26.4), contract fixtures, uncertainty heatmaps, triage thumbnails, extracted-image mirrors). The storage panel reports usage by category, by checkpoint, and by state, and offers safe eviction of regenerables only, under the same dry-run-then-act discipline as GC (§12.4). Compaction rewrites reachable CAS under a lock, verifies hashes, and keeps a rollback manifest; it never runs concurrently with a module commit (queue barrier). Derived projections may always be deleted: a project minus all caches still opens, assesses, and re-exports — that property is a test (§21.5), because it is what makes "keep the source and the semantic state, reclaim the disk" safe advice.

Disk is a first-class scheduler input: module manifests declare disk estimates (§11.2), the scheduler refuses to start a work unit whose declared peak exceeds free space minus a configurable margin, and a projected shortfall surfaces as a plan error — never as `ENOSPC` mid-commit, the one filesystem failure §13.2 cannot make graceful. `cas/` may be relocated to an external volume via a recorded, hash-checked root identity, validated on open; a mismatched root is a hard stop rather than a best-effort path resolution.

### 22.7 Logs, diagnostics, and support

Rotating logs capped by size and age, with secrets redacted at the writer rather than at the reader (§4.4, §17.1). The redaction contract states what never enters a log: core and engine tokens, api keys, absolute home paths in exported bundles, and **document text** — logs carry ids, hashes, coordinates, and durations, because a debug log that quotes recovered prose is simultaneously a copyright exposure and a PII channel (§17.5). The diagnostic bundle (§16.7) is assembled locally, itemised, and reviewed as a diff before the user copies it anywhere; it contains versions, tier, contract-probe results, a redacted journal tail, and finding counts, and it excludes CAS content unless the user explicitly attaches a specific project.

### 22.8 Documentation register

Each file below is normative or informative as marked; CI fails when a normative one is missing or unparseable (§19's `docs/`, `a11y/`, `engines/`):

| Path | Status | Contents |
|---|---|---|
| `SPEC.md`, `README.md`, `CHANGELOG.md`, `LICENSE` | normative / informative | this document; install and first run; per-release changes (§22.9); core license (ADR-010) |
| `SECURITY.md` | normative | §17 threat model, disclosure policy, loopback residual (§17.4), redaction contract (§22.7) |
| `dependencies/manifest.toml` | normative | the §5.1 machine-readable manifest, emitted into reports and the SBoM |
| `models/MODELS.toml` | normative | registry entries per §30.2, incl. license-review evidence and measured capability |
| `fonts/MANIFEST.toml` | normative | pinned OFL face, license, restricted-font-name handling (§31.4) |
| `engines/comfy/recipes/*/README.md` | normative | what each recipe does in human terms, its parameters, its mask behavior |
| `engines/*/contract/`, `engines/docling/contract/CONTRACTS.md` | normative | pinned probe payloads per §21.2 — the executable reading of §6 |
| `docs/MODULE_API.md` | normative | the §11.3 lifecycle, error mapping, SDK compatibility and quarantine rules |
| `docs/PROJECT_FORMAT.md` | normative | §12's on-disk layout, lock format (§22.5), CAS identity and relocation (§22.6), migration policy |
| `docs/SEMANTIC_MODEL.md` | informative | prose guide to §7 with worked examples, generated from `schemas/` where possible |
| `docs/EVAL.md` | normative | §20.1 protocol, metric definitions, rights rules, R6's source-information-limit method (§24) |
| `benchmarks/BASELINE.md`, `evals/results/BENCHMARKS.jsonl` | normative record | §18.5 budgets; §20.1 measured results |
| `a11y/` | normative | AT matrix, manual protocols, recorded results (§21.6) |
| `schemas/` | normative | generated kernel, module, and contract schemas; taxonomy registry (§26.1) |

### 22.9 Versioning and support policy

Application version is semver against this document's requirements; the project-format version (§12.1), module-API version (§11.2), and kernel schema major (§7.13) are separate counters and never move together by accident. Guarantees: any released app version can migrate and open a project from any format version released in the preceding 36 months (§21.4); assessment records remain reproducible with the assessor version that produced them (§20.4); deprecated projections, profiles, or module APIs emit findings and documentation notices for one full minor series before removal. "Supported" for a hardware/OS tier (§4.3) means CI has a lane for it (§21.7); "documented, unsupported" means the code path exists and is honest about failure but carries no gate. Uninstall removes application-home components only after confirming nothing is mid-run, and never touches project directories (§12.1) — user work is not uninstallable as a side effect.

### 22.10 Acceptance criteria

**Foundation release (end of Phase 1–2, §20.3).** Acceptable when it can: (1) launch on loopback, bind the token, open the browser after readiness, and fail non-corruptingly on port collision, stale lock, and browser-launch failure (§4.4, §22.5); (2) create a project from a scanned PDF, an image directory, and a born-digital PDF, preserving source hashes (§8.1), and reach export with **zero** vision-stage runs on the digital path (§8.3); (3) save, reopen, and hand off a portable project byte-stably; (4) run a contract-conformant drop-in module transactionally through the SDK, including a forced crash and a forced cancel, with the last committed state intact (§13.2, §21.5); (5) show linear history, restore any state, protect a checkpoint, and warn correctly on divergence (§13.1); (6) render HTML previews incrementally and label stale ones (§15.2, §18.3); (7) batch manual edits into one committed patch with a plain-language summary (§16.4); (8) produce a deterministic assessment and a validated, costed plan (§14.2, §14.5); (9) export at any score with its report (§14.6); (10) pass its own axe-core and keyboard gates for these workflows (§21.6); (11) report an honest capability tier with a missing engine surfaced as BLOCKED, never skipped silently (§4.3, §14.5).

**v1 definition of done.** All of the above, plus: every §2.2 input importable; all §9 passes operative with §9.6 stop rules demonstrated on corpus books; tagged PDF passing §15.3's validation loop with zero blocking findings where the profile requires it, its structure derived rather than attached; EPUB 3 gated by EPUBCheck; the evidence graph un-bypassable end-to-end (A6 and A10 reproduced in-application); residual uncertainty visible in every projection and counted in the report (§7.8, §14.6); §21.7's release lane green on every lane §4.3 declares supported; §21.8's defect classes empty; §31's register reviewed inside its currency window; and §24's register reviewed with each residual accepted by a named owner.

---

## 23. Architecture decision records

Each record is binding: amending a decision means writing a superseding record with evidence (§32.2), not editing code around it. "Costs accepted" is deliberate — a decision whose price is unnamed is a decision that will be quietly reversed under pressure. Statuses: **Accepted** (binding), **Accepted (gated)** (binding *and* contingent on a named Phase-0 assertion), **Open choice** (decision made, parameter pending).

### ADR-001 — Semantic master document; output formats are projections
**Status:** Accepted. **Context:** Accessible deliverables pull in opposite directions — screen-reader users want abstraction (logical structure, no spatial noise), low-vision readers want spatial fidelity, institutions want a PDF that validates — and re-deriving semantics from each export format would make every format a competing source of truth. **Decision:** A format-neutral kernel (§7) is the only authoritative state; HTML, tagged PDF, and EPUB 3 are projections (§15) with published capability matrices; no projection may hold semantics the kernel lacks, and unportable presentation lives in namespaced overrides (§15.1). **Rejected:** PDF-as-master with sidecars (loses reflow semantics, and assessment fragments per format); a lowest-common-denominator model (guarantees loss and forecloses the deferred projections §2.3 names); one store per format (history, evidence, and findings would diverge). **Costs accepted:** every projector must publish and test a fallback matrix (§21.3); some kernel richness provably cannot reach some targets; four artifacts (kernel, module protocol, overrides, report) need version governance.

### ADR-002 — Engine federation at process boundaries
**Status:** Accepted. **Context:** The perceptual work requires model stacks whose dependency sets are mutually incompatible and whose licenses range from permissive to GPL (§5.8). **Decision:** GPU-owning capability lives in separately-launched, app-managed engines reached over loopback HTTP/WS or as subprocesses (§4.2): no shared memory, no cross-boundary imports, no authoritative state inside an engine; JSON, geometry, and content-addressed files cross the line (§6.1). **Rejected:** one process holding everything (ADR-003's rationale); user-started engines (an installation a practitioner must hand-assemble is an installation that fails silently); in-process native extensions (§11.4). **Costs accepted:** handoff copies through staging, serialization overhead, engine cold starts priced into plans (§18.2), port and token management (§17.1), and cancellation granularity limited by each engine's protocol (§6.1, §13.3).

### ADR-003 — The application core is CPU-only
**Status:** Accepted. **Context:** Multi-model Python environments fail on pin arithmetic: a diffusion stack, a vLLM stack, and a docling stack each demand specific torch/CUDA/transformers versions, and importing all three into one process forces either a compromise no ecosystem supports or an unresolvable lockfile. This is the most common way such products die. **Decision:** No tensor library is imported by the core process (§1.1, §5.4). The core moves JSON, geometry, files, and SQLite rows; CI enforces the ban mechanically (§21.2) rather than by convention. **Rejected:** in-process `transformers`/`diffusers` (the collision above, plus accelerator-state bugs in the process that also owns the user's project database); a "GPU mode" flag that loads torch when available (a configuration branch is a second product, and §0 forbids optional features). **Costs accepted:** artifact copies across boundaries, engine startup latency, and a supervisor that must own VRAM scheduling (§6.7) — all cheaper than an environment that cannot be built.

### ADR-004 — Exactly one transformer runtime: vLLM
**Status:** Accepted. **Context:** Recognition, critique, and repair need document VLMs and a mid-size LLM, with multimodal inputs, structured outputs, batching, and cancellation. **Decision:** vLLM is the sole required implementation of all transformer inference (§5.2), reached through its OpenAI-compatible API, with a per-model registry (§30.2) and capability probes (§6.3.1) deciding what an instance may serve. **Rejected:** llama.cpp/Ollama (GGUF-quantization-centric; the doc-VLM families we evaluate are served best by vLLM, and a second runtime doubles the pin matrix, the probe set, and the adapter surface for no v1 capability); SGLang (same capability class — adopting it would replace a decision with a preference and unfreeze the manifest, contrary to principle 10); Hugging Face Transformers in-process (ADR-003); remote APIs (ADR-005). **Costs accepted:** a CUDA-centric matrix — the honest consequence is that the macOS tier loses VLM OCR and LLM repair and must report **BLOCKED** (§4.3) rather than degrade invisibly — plus heavier installs and engine-level restarts.

### ADR-005 — No remote model endpoints in v1
**Status:** Accepted. **Context:** A "bring your own endpoint" switch looks cheap and touches three core guarantees at once: privacy posture (principle 9, §17.2); evidence lineage (§7.7 records `checkpoint_rev` and `model_family`, and a remote alias exposes neither, so R2 independence judgments become unverifiable); and cost modeling (plans are denominated in locally measured GPU-seconds, §18.5). **Decision:** Not implemented. The OpenAI-compatible seam remains *internal* — vLLM's server — and the core ships no client for it (§17.2). **Rejected:** optional remote adapters behind an egress consent dialog (consent does not restore lost lineage, and a code path that ships is a code path that gets used); "provider plugins" (defers the whole problem into an unbounded interface). **Costs accepted:** users with only small GPUs get lower capability, not an off-ramp, and §5.7's bake-off is bounded by what fits locally. **Revisit-when:** a future ADR defines how a remote call records verifiable lineage *and* the product decides privacy is a preference rather than a premise.

### ADR-006 — AGPL excluded from the entire dependency graph
**Status:** Accepted (gated by A1–A3, §20.2). **Context:** MuPDF/PyMuPDF is the obvious high-quality PDF renderer-and-writer, and it is AGPL-3.0. The legal reading of AGPL at arm's length over loopback is arguable; the practical reading is not, because recipes are meant to be copied by practitioners into their own ComfyUI contexts (§6.2.1) and engine environments are meant to be reproducible by users. **Decision:** AGPL is banned outright — stricter than legal necessity — including inside engine venvs, and the ban is a build failure, not a review note (§5.8, §17.6). **Rejected:** "AGPL is fine for a local tool" (a license position that depends on a legal argument is not a policy); case-by-case exemptions (every exemption is a fork in the manifest). **Costs accepted:** we lose the best off-the-shelf PDF writer, which is exactly what forces ADR-014's harder path. **Revisit-when:** Phase-0 assertions A1–A3 fail — this record and ADR-014 reopen *together*, before application investment (§15.3).

### ADR-007 — ComfyUI is the diffusion engine; recipes are data
**Status:** Accepted. **Context:** Cleanup and glyph repair need inpainting/diffusion stacks that churn weekly on weights whose licenses must be re-verified per release; and a product whose job is *altering evidence* must let a practitioner see and re-run the alteration. **Decision:** Unmodified upstream ComfyUI at a pinned commit in its own venv, driven through its documented HTTP/WS API, with this project's node pack and recipe manifests layered on top (§5.2, §6.2). A recipe is declarative data — hashed into provenance, semver-versioned, and required to remain runnable and inspectable by a human in the ComfyUI UI. **Rejected:** `diffusers` in-process (ADR-003); bespoke CUDA/Python restoration code (competitive performance, zero transparency, and it turns every research swap into an integration project); hand-edited live workflows (unauditable and non-reproducible — the opposite of §6.1's provenance rule). **Costs accepted:** GPL as a separate component, coarse engine-dependent cancellation, queue and VRAM economics, and a graph-format coupling to upstream that contract tests must police (§21.2, §25.1).

### ADR-008 — docling-serve for born-digital extraction; its output is prior evidence
**Status:** Accepted. **Context:** Untagged PDFs already contain a text layer and often structure; rebuilding an extractor would be work on our least differentiating problem. But the product's thesis (§1) is that a claimed text layer must be *tested against pixels*, not trusted. **Decision:** docling-serve is the required extractor for the digital path (§5.2, §6.4), and its results enter the kernel as `prior_textlayer`/`prior_structure` evidence — never as ground truth — with §8.2's layer-vs-image disagreement probe deciding when the digital path is actually usable. **Rejected:** trusting the existing text layer (the classic defective-OCR failure this product exists to catch); an in-house extractor (a second maintenance surface for no product gain); skipping the digital path (the §8.3 requirement to reach export with zero vision-stage runs is a real saving for real users). **Costs accepted:** a pinned MIT engine whose route surface we re-assert at every upgrade (§6.4), and a DoclingDocument→kernel mapping we own.

### ADR-009 — Embedded durable workflow; no external orchestrator
**Status:** Accepted. **Context:** Remediation runs are long, resumable, cancellable, and must never corrupt the last committed state — the classic argument for a workflow platform. **Decision:** The workflow layer is embedded: SQLite (WAL) for states, queue, evidence, findings, and events; an append-only journal for crash replay; staging transactions with private recovery checkpoints (§12, §13). Work units stay deterministic-input, idempotent, and separately checkpointable so a later migration remains conceivable. **Rejected:** Temporal or any external orchestrator (it would hold a *second* source of truth about the same work, creating a distributed-commit problem between workflow state and semantic state, and it asks a single-user workstation to run a server we do not control); bare asyncio task graphs (nothing survives a power cut); automatic execution retry (§6.1 forbids it — retry is a cost and convergence decision, not a transport concern). **Costs accepted:** we implement queueing, replay, and cancellation discipline ourselves, and §21.5's failure-injection suite is the price of admission. **Revisit-when:** multi-user server operation enters scope (§2.1 explicitly declines it for v1).

### ADR-010 — License firewall, distribution posture, and a permissive core
**Status:** Accepted; **Open choice** on the core's exact SPDX (Apache-2.0 vs MIT, before first public commit, §5.8). **Context:** The runtime set necessarily includes GPL components (ComfyUI, veraPDF) and a GPL+CPE runtime (JDK), while this project's own code should be usable by the institutions and vendors it serves. **Decision:** Copyleft engines run unmodified, as separately-launched processes, distributed as distinct components with their own notices and source-availability paths; nothing is linked, embedded, or vendored, and no upstream fragment is copied into this repo (client code targets documented public APIs). The core's license is permissive. Packaging expresses the firewall: engine packs are separate artifacts (§22.1). **Rejected:** GPL-ing the whole application (simpler legally, but it forecloses the vendor and publisher deployments that most need this tool, and it makes ADR-006's recipe-portability goal harder); proprietary-only (blocks institutional review of how evidence is scored, which is the product's trust basis). **Costs accepted:** a larger, multi-part distribution; a permanent obligation to *prove* the boundary holds (SBoM and license CI, §17.6; layering tests, §21.2); and a standing temptation to "just vendor" a node pack, which is refused.

### ADR-011 — Modules are subprocesses under one contract; no privileged built-ins
**Status:** Accepted. **Context:** Drop-in remediation modules are the extension story, and a drop-in Python module is arbitrary code. **Decision:** Every module — first-party included — runs in its own venv as a separate OS process speaking the versioned stdio JSON-RPC protocol, receiving an immutable snapshot, writing patches and artifacts to staging, and reaching models only through the core's `model_broker` (§11.1–§11.3). **Rejected:** in-process imports for built-ins with sandboxed third parties (a two-tier architecture guarantees that built-ins take shortcuts the plugin surface cannot support, which is how such products rot); container-only isolation (heavier than v1 needs and not identically available across §4.3's tiers — retained as a future hardening option); signed modules with implicit permissions (§17.3 treats every module as hostile until it proves otherwise). **Costs accepted:** serialization overhead, per-module venv disk, and the discipline to keep capability in the SDK rather than in privileged code.

### ADR-012 — Mechanical anti-circularity in the evidence graph
**Status:** Accepted. **Context:** The defining technique (§1) is generative: a hypothesis is rendered and tested against ink. The same technique can launder a guess into "evidence," and a system that cannot tell the two apart produces exactly the artifact this product must never ship — crisp, wrong text (§24-R2). **Decision:** R1–R4 (§7.7) are enforced by the kernel, not left to model discretion: conditioned restoration is recorded as conditioned *by construction* (recipe and model provenance make it mechanical, §9.4), lineage identity bounds ensemble strength, and calibration is corpus-derived and versioned. **Rejected:** confidence averaging over model self-reports (fluent and worthless); treating repeated sampling of one checkpoint as corroboration (R2 forecloses it, §30.5); human review as the primary control (scales to nothing, and §17.5 notes that visual success actively disarms reviewers). **Costs accepted:** a graph the core must maintain on every commit, harder plan reasoning, and the loss of "three models agreed" as a usable headline.

### ADR-013 — Validators are advisory; export is never gated
**Status:** Accepted. **Context:** veraPDF and EPUBCheck have bugs, false positives, and version drift; a hard gate would put our release schedule inside a third party's. Institutions nonetheless need conformance *evidence*. **Decision:** Machine checks produce findings (§6.5, §14.3) and appear in the export report; a profile may require zero-blocking findings at export (§15.3), which is an institutional choice recorded in the output rather than a product-wide lock; principle 8 keeps every committed state exportable. **Rejected:** a certification mode that refuses export on validator failure (it creates a silent incentive to fork the validator and converts advisory tooling into an availability dependency); omitting failed checks from the report (defeats the evidentiary purpose §2.3 states). **Costs accepted:** users can export a bad file — the compensating obligations are that it is loudly labeled, counted (§14.6), and never described as certified (§31.3).

### ADR-014 — In-house PDF writer on pikepdf; no third-party writer layer
**Status:** Accepted (gated by A1–A3, §20.2). **Context:** §15.3 requires marked-content structure anchored *around the real drawing operators*, an invisible text layer generated from current alignment, a persisted object-id ↔ MCID table, and byte-reproducible output for golden tests. **Decision:** Exactly one component writes PDF bytes: the core's PDF projector, over pikepdf's object model, with pypdfium2 confined to read-only rendering and fontTools/uharfbuzz preparing shaped runs and ToUnicode mappings (§6.6). Content-stream text operators are composed by us from shaped runs, with directional run segmentation owned by the projector (§29.3). **Rejected:** PyMuPDF/MuPDF (ADR-006); a general PDF-writing library as the structure authority (it owns the content stream, so kernel ids cannot be bound to MCIDs at the point of truth, and a second object model appears beside pikepdf's); post-hoc auto-tagging and text-layer insertion tools (the "attached after the fact" pattern §15.3 forbids — alignment cannot be reconstructed afterwards). **Costs accepted:** we own ISO 32000 correctness, carry the feasibility risk A1–A3 exists to test, and need in-house veraPDF expertise. **Revisit-when:** A1–A3 fail; the paired record to reopen is ADR-006.

### ADR-015 — Linear visible history with protected checkpoints; branching hidden
**Status:** Accepted. **Context:** Users are accessibility practitioners, not version-control operators, and the operations that matter — restore, compare, protect, continue from an earlier point — are all expressible linearly. **Decision:** Committed states form one visible sequence; checkpoints pin full restorability across replaced lines; mutating an earlier state warns and offers Cancel / Protect-then-continue / Continue-and-replace; replaced unprotected states survive a grace period behind an advanced restore command (§13.1). **Rejected:** a branch-and-merge DAG in the UI (it makes the user resolve merges that patch preconditions were designed merely to detect; three-way merge is explicitly deferred, §16.4); append-only history with no divergence (a "restore" that is only ever forward-then-new confuses practitioners and hides the working line); git-as-interface (comprehensibility over git-envy). **Costs accepted:** divergence destroys unprotected later states unless protected — mitigated by the prompt and the grace period — and storage growth becomes a designed concern (§12.4, §22.6) rather than an accident.

### ADR-016 — Coordinate spaces and the transform ledger are a first-class subsystem
**Status:** Accepted. **Context:** The pipeline stacks deskew, crop, rescale, dewarp, and per-engine resampling between source pixels and any output. Every one of those is a chance for boxes, masks, and text alignment to disagree while each file remains individually valid — a failure users cannot see and would rightly call corruption. **Decision:** Every stored quantity names its space; every accepted transform is a ledger entry with an inverse (closed form, or a mesh artifact hash for dewarp); transport is composed, never improvised; an accepted transform invalidates downstream alignment and emits `ALIGNMENT_STALE`; undeclared geometry is a module contract violation (§7.5, §11.3). **Rejected:** normalizing everything to one space at import (destroys the source geometry that §9.5's source-likelihood scoring and §16.3's overlays require); re-deriving transforms by image matching (a research project masquerading as a utility); trusting each module to keep boxes consistent (rule 3 exists because that trust fails). **Costs accepted:** every module must declare spatial effects, the ledger migrates with the schema, and transport composition appears in hot paths.

### ADR-017 — Weights are data; one default assignment per role
**Status:** Accepted. **Context:** Research moves faster than integration work, so a spec that hard-wires model names ages in months — but a system that forks behavior on model identity is untestable. **Decision:** The runtime set is closed; research breadth lives in swappable, license-verified, hash-pinned weights registered in `models/MODELS.toml` with roles, capabilities, and measured performance (§5.7, §30.2). Exactly one default per role, chosen by corpus bake-off and re-run on every change (§9.2, §20.4). The runtime branches only on adapter selection and probe-declared capabilities. **Rejected:** a user-facing model picker with free choice (an unsupported matrix of model × prompt × parameters × preset for each user to discover alone, and §0 forbids "bring your own engine"); per-module hard-coded checkpoints (defeats calibration, since a role's behavior becomes unattributable); keeping many models resident to preserve options (§18.2's VRAM economics forbid it). **Costs accepted:** every shipped default must survive re-evaluation on our corpus repeatedly, and some user will know a better model we decline to enable. **Revisit-when:** a replacement weight is proposed — through the bake-off, not through a configuration option.

### ADR-018 — Browser UI on loopback with a per-launch token; no embedded webview
**Status:** Accepted. **Context:** The product needs a rich semantic editor, PDF and EPUB previews, and overlay canvases, and it must not become a distribution liability. **Decision:** Serve the UI in the user's own browser on `127.0.0.1`, default port 8765 with a bounded fallback, token-gated, `Host`/`Origin`-validated, strict CSP, copyable URL as launch fallback (§4.4, §17.1). **Rejected:** Electron/Tauri (an embedded Chromium makes *our* patch cadence the user's CVE exposure and multiplies artifacts across §4.3's tiers); server-rendered HTML with no rich client (a ProseMirror-grade semantic editor and canvas overlays are not achievable that way); a hosted UI (contradicts principle 9). **Costs accepted:** we must defend a local port against drive-by webpages ourselves — hence §17.1's token and origin discipline, §21.9's tests, and the documented residual in §17.4 — and we depend on a browser existing, which is reasonable on every supported tier.

### ADR-019 — Three projections in v1, in a fixed construction order
**Status:** Accepted. **Context:** §2.3's deferred list (DOCX, DAISY 4, tactile, large-print) is affordable only if v1's projection set is deliberately small: each additional target multiplies the equivalence matrix and the AT matrix rather than adding to them. **Decision:** HTML, tagged PDF, and EPUB 3 (§15.2–§15.4), built HTML-first because it is the reference surface and the fastest structural feedback loop, with the PDF *writer* de-risked in Phase 0 (§20.3). The projector contract must accommodate deferred targets, and the kernel must remain able to represent what they carry (§2.2, §2.3) — that is the whole v1 obligation toward them. **Rejected:** DOCX in v1 (no trustworthy local layout engine, so its preview would be a semantic approximation the UI must perpetually disclaim, at high test cost for accessibility targets HTML already serves); DAISY in v1 (its distinctive value is navigation plus synchronized playback, requiring audio production that no §5 dependency provides); "more formats proves breadth" (an untested projection is a defect with a file extension). **Costs accepted:** navigable mathematics lives in HTML/EPUB rather than PDF (§28.2), which must be said in product language instead of left as an inference.

### ADR-020 — Python 3.12 core; TypeScript/React frontend; schemas as the seam
**Status:** Accepted. **Context:** The perception, geometry, font, and PDF ecosystem is Python (§5.4), as are the engine ecosystems; the editor, PDF viewing, and accessible-component ecosystem is browser-native. **Decision:** Python 3.12 for the core; TypeScript + React + Tiptap/ProseMirror for the UI; Pydantic-generated JSON Schema in `schemas/` as the single source of contract truth, with client types generated from it and drift checked in CI (§5.5, §7.13, §21.2). **Rejected:** one language everywhere — Python for the UI (no editor or pdf.js equivalent; accessibility would be hand-built and permanently behind) or TypeScript for the core (every geometry, font, and PDF facility would be re-implemented or subprocess-bridged, and the engine ecosystem is Python-shaped anyway); server-only rendering (ADR-018). **Costs accepted:** two toolchains, two lockfiles, and a codegen boundary that must be CI-verified.

---

## 24. Risk register

Ranked by expected damage, not by ease of fixing. Each entry names a **tripwire** — an observable that tells us the risk is materialising before a user does — because a register without early signals is a list of excuses. Cross-references are normative: §9.7 cites **R2**, §18.4 cites **R6**, and §9.3's degradation-honesty alarm serves **R5**. Status: **open** (active design work), **gated** (a Phase-0 assertion decides), **managed** (controls specified; watch the tripwire).

### 24.1 The six that can kill the product

**R1 — The source does not contain the answer.** *Fabrication pressure.* Restoration and recognition models produce plausible output by design; when information is destroyed the default behavior is to invent it, and invented text is crisper than the truth. **Tripwire:** fabrication rate above the recorded floor on the `information_destroyed` stratum; any exported claim whose highest-weight evidence is `model_output` with `n_eff < 2` (§7.7 R3) and whose review state is not `needs-attention`. **Mitigation:** §7.8 as a first-class state rendered in every projection; §9.5's abstain rule under threshold; §14.7's no-auto-approval; false certainty as an acceptance metric (§20.1); fabricated content release-blocking (§21.8). **Residual:** some users will accept a guess; the control is that the gap stays visible in the artifact, not only in a report. **Status:** managed.

**R2 — Self-confirming loops.** *The infinite generator of confident nonsense.* A hypothesis conditions a restoration, the restoration is OCR'd, the OCR "confirms" the hypothesis, confidence rises, the planner stops, and the artifact is wrong. The failure is silent *by construction*, which is why it is a register entry rather than a bug. **Tripwire:** any evidence edge in a corpus run where a claim's support includes a descendant of itself; calibrated confidence rising across loop iterations while gold accuracy is flat; §20.1's false-certainty rate increasing between releases. **Mitigation:** R1–R4 of §7.7 enforced in the kernel (ADR-012); §9.4 funnels all text conditioning through one recorded request object so lineage is mechanical; §9.7 refuses polish work where independent families already agree; §9.6's oscillation and repeat stops; §21.1's property test that a descendant never raises an ancestor's support; same-lineage samples collapsed to one evidence node (§30.5). **Residual:** correlation we cannot observe — differently-named families sharing pretraining data, and lineage identity being a registry judgment (§30.2). Disclosed in §14.6 rather than hidden. **Status:** managed.

**R3 — Fluent text, absent grounding.** Document VLMs emit sentences without reliable character-level confidence or exact alignment; a 98 %-accurate transcription with wrong boxes yields a PDF whose invisible layer mis-selects, whose overlays drift, and whose copy/paste corrupts — the most visible possible failure of an accessibility product. **Tripwire:** alignment IoU or ToUnicode round-trip rate below §20.1 floors; `ALIGNMENT_STALE` findings surviving a re-alignment pass; overlay or selection complaints from AT testers. **Mitigation:** alignment is a separate subsystem from recognition and is a property of (text version, surface version, space) (§7.6); re-alignment is a first-class module; adapters must emit boxes or be probe-rejected (§6.3.1, §6.3.2); `suspicious_spans` feed §9.7's priority; numerals and operators are their own metric track (§20.1). **Residual:** glyph-level certainty on degraded script stays beyond current weights; handled by §7.8 rather than by pretending the box is right. **Status:** open (A3, A4, A5).

**R4 — Restoration damages what must survive.** A diffusion pass that improves legibility can also re-space a glyph, redraw an illustration, drop a diacritic, remove a library stamp, or launder a handwritten annotation that was the only record of a prior owner. **Tripwire:** §9.3 typographic-consistency findings after cleanup; protected-mask pixel-delta findings; any §17.5 protected class removed without a recorded user decision. **Mitigation:** protection masks bound what may change (§9.1 step 2, §26.3); recipes must not modify pixels outside the mask dilation envelope, verified by the module rather than trusted (§6.2.2); re-anchoring against the source rather than only the previous generation (§9.5 render-back, §18.4 ladder); §17.5 blocks automated removal of notices, stamps, and watermarks; §27.5 keeps marginalia on a retained layer. **Residual:** fine typographic rhythm shifts inside a legitimate mask, met with a persistent flag rather than a claim of invisibility (§16.3). **Status:** managed.

**R5 — Cumulative drift and manufactured crispness.** Repeated passes converge on a page that looks like a clean page and is no longer that book — the characteristic failure of recursive image editing, and the reason visual success disarms review (§17.5). **Tripwire:** the §9.3 degradation-honesty critic flagging a region smoother than its neighbours; region hashes advancing while source-likelihood scores stall; §18.5 throughput improving with flat §20.1 accuracy. **Mitigation:** every surface is a versioned artifact chained from source, never an overwrite (principle 4, §7.2's `PageSurface` stack); §9.6 stops on non-improvement and cycles; §9.7 cost-gates by information gain; reconstructed regions carry persistent flags with 3× review sampling (§17.5); §7.8(2) keeps degraded pixels honest. **Residual:** drift is judged statistically, so individual regions slip through; sampling weighting is the control, not the metric. **Status:** managed.

**R6 — Source-information limits.** Legibility has a physical floor: below some sampling rate the difference between `rn` and `m` is not present in the pixels, and a model that "restores" it is writing a guess. The trap is that upscaling looks like progress and reads like evidence. **Tripwire:** any stage raising effective resolution above the declared source-information limit without a finding; two independent families agreeing on a *fabricated* distinction while §7.7 R3 fails to route it to `needs-attention`; false certainty on the ambiguous stratum. **Mitigation:** §18.4's ladder refuses upsampling above the limit by default and emits a finding — this entry is the authority that section cites; §7.8 is the correct product answer; §9.5's abstain rule bounds adjudication; synthetic degradation with known ground truth measures it (§20.1). **Residual:** the limit itself is estimated per page rather than measured per region; the default is conservative and the estimation method is documented in `docs/EVAL.md` (§22.8). **Status:** open.

### 24.2 Remaining register

| ID | Risk | Tripwire | Mitigation (§) | Status |
|---|---|---|---|---|
| R7 | In-house PDF writer proves infeasible or subtly wrong (structure/text-layer coupling, ISO 32000 correctness, viewer variance) | A1–A3 (§20.2) fail; veraPDF blocking findings that reproduce only in third-party viewers | §20.2 gate *before* scaffolding; §21.2 self-parse and byte goldens; §21.6 real-viewer matrix; ADR-006/014 reopening path | **gated** |
| R8 | Layout and reading-order misinference on atypical pages (multi-column with sidebars, text wrapping figures, two-page spreads, unusual note placement) | §7.9 constraint findings concentrated in one stratum; AT testers report out-of-order speech that validators miss | §7.9 graph plus deterministic constraints; §27's placement rules; §16.4 order editor with "what AT would say" pane; `UnknownRegion` as the honest finding | open |
| R9 | Specialist semantics plausible-but-wrong: spans closing on the wrong cells, an operator read as another, a log axis read as linear | §14.7 chart re-plot tolerance failures; equation parse failures; §20.1 numerals/operators track below floor | §7.12 structured payloads only; parse-or-report, never silent fix (§9.3); §28's render-back and re-plot checks; separate metric tracks (§20.1) | open |
| R10 | Engine and pin rot: upstream API drift, weights withdrawn or re-licensed, VRAM contention, cold-start-dominated throughput | contract probes failing at a new pin (§20.4); `ENGINE_WAIT` stall share above threshold; §18.5 ±20 % alarm | pinned commits with hash verification (§5.1); §21.2 and §25's executable contracts; §6.7 supervisor ledger and bounded restarts; tier honesty (§4.3) | managed |
| R11 | State loss or corruption: commit-protocol bugs, GC deleting reachable data, journal replay divergence, `ENOSPC` mid-write | any §21.5 kill-matrix failure; hash mismatch on a committed artifact; replay not reproducing state | §13.2 publish-last protocol; §12.4 dry-run GC; §22.6 disk budgeting before commit; platform fsync/rename discipline; §21.8 blocker class | managed |
| R12 | Module supply chain: hostile or merely sloppy third-party code touching the filesystem, network, or ledger | `validate` findings for undeclared writes; quarantine events; granted permissions diverging from observed use | §11.2 forbidden-capability declarations; §11.4 default-deny and trust prompt; §26.6 bag constraints; §21.9 hostile-module tests; signed distribution support | managed |
| R13 | Localhost and hostile-content attack surface: drive-by pages against the port, poisoned archives/PDFs/fonts, script execution in previews | any §21.9 case reaching a mutating route without a token; a CSP violation; a parser crash from fixture data | §17.1 token plus `Host`/`Origin` and disabled CORS; §17.3 defensive parsing and sandboxed frames; §25.1 engine-socket discipline; documented residual (§17.4) | managed |
| R14 | Corpus rights and PII custody: evaluation sets that cannot lawfully be retained or shared; handwritten annotations carrying personal data; publisher notices removed | a corpus PR missing rights fields; a fixture or bundle containing `evaluate`-only pages; a PII finding on an export path | §20.1 rights manifest with harness enforcement; §17.5 local-only retention and redaction tooling; §22.7 no document text in logs; §26.3 comment and decision redaction | managed |
| R15 | Assessor gaming and false precision: a score raised by satisfying the metric instead of the reader, or one that moves with assessor versions | predicted-vs-actual delta divergence (§14.5); institutional gold projects disagreeing with scores (§21.4); a weight change altering a headline | §14.2 findings-and-weights scoring, never free-form LLM judgment; versioned calibration (§14.4); known-good/known-bad regression (§20.1); published breakdown; §14.1's four statuses never conflated | open |
| R16 | Automation complacency and review fatigue: reconstructed regions reviewed *less* because they look clean; findings trained out of users | review dwell time falling while `needs-attention` counts rise; `approved` states with no corroboration; dismissals without reasons | §17.5 persistent flags with 3× sampling weight; §14.7 no auto-`approved` for generated descriptions; findings as actionable units with repair routes (§14.3); §26.3's `Decision` record requiring rationale | open |
| R17 | Comprehensibility failure: history, queue, findings, provenance, and per-format overrides overwhelming the practitioner | §21.6 tasks abandoned without help; two surfaces disagreeing about completion; §7 vocabulary leaking into the UI | §13.1 linear history with hidden branching; §16.2's one-number-one-truth rule; progressive disclosure with overrides in advanced views; §32.1's glossary as the terminology authority | open |
| R18 | Scale: a 600-page book exhausting disk or memory, or object counts making queries unusable | §18.1's synthetic 1000-page CI benchmark regressing; per-project disk above §22.6's budget; index query ceilings exceeded | page streaming and lazy CAS loads; bounded work units; chapter-granular caches; per-state incremental indexes (§26.4); regenerable/required split (§22.6) | managed |
| R19 | Consumer variability: files that validate and still behave differently across readers and screen readers | §21.6 matrix divergence between a validator pass and observed AT behavior; a golden AT transcript changing without a kernel change | §14.2's separate conformance category; the real-consumer matrix as a gate (§21.6); §15.1 capability matrices naming each fallback; HTML as the reference surface for disagreement | open |
| R20 | Longevity and misreading: formats and standards advance (PDF/UA-2, EPUB 3.4, WCAG 2.2→3), pins rot, and institutions read our report as a legal certification | §31's currency window exceeded without review; a project that will not open in the current version; support threads quoting the report as conformance proof | §21.4 golden projects per format version; §22.9 support windows; §20.4 re-validation; §31.3's wording rules forbidding "certified"; §14.1 stating in those words that the score asserts nothing about legal certification | managed |

### 24.3 Register maintenance

The register is owned by the release owner and reviewed at every phase gate (§20.3) and every pin rotation (§20.4). A defect closed under §21.8 joins this register when its control is architectural rather than local. Each entry's tripwire must be a metric the harness or the application actually emits: an entry whose tripwire cannot be instrumented is re-specified, not kept as prose. Closing an entry requires evidence in `benchmarks/` or `docs/`, and residuals are republished in §14.6's "known limitations" so the user — not only the team — sees what remains unproven.

---

## 25. Interface-contract refinements

§6 fixes each engine's surface; this section adds the protocol details that decide whether an integration is robust or merely working. Every rule here is covered by an executable contract test (§21.2), recorded per pinned revision. Where §6 states intent, this section states the observable behavior; nothing here relaxes a §6 requirement.

### 25.1 ComfyUI

**Version pinning is out-of-band.** The engine reports a version string and system statistics, not the source revision this project pinned. `--listen`/`--port`/`--disable-auto-launch` are echoed in its reported launch arguments and cross-checked, but the commit assertion comes from the installed tree recorded at setup (§22.2) against `dependencies/manifest.toml`. A mismatch at either level → `VERSION_MISMATCH`; the node pack and each recipe's `comfy_rev` (§6.2.1) are validated the same way.

**Run identity and the progress stream.** `client_id` is load-bearing, not decorative: per-node progress, execution errors, and completion events are addressed to the socket holding that id, and a run submitted without one yields only coarse status traffic. Normative: one `client_id` (uuid4) per recipe instance, exactly one live WebSocket bound to it, and a *second* connection using the same id is treated as a takeover (the engine evicts the first) — so reconnect deliberately reuses the id and never spawns a parallel monitor socket. UI observers read the core's event stream (§16.8), not the engine socket.

**Termination detection.** Do not wait for the historically-documented "executing node = null" marker; on current upstream it is not sent. A run completes on `execution_success`, fails on `execution_error`, cancels on `execution_interrupted`, and is otherwise resolved by polling history. `execution_interrupted` is the only broadcast event, so cancel confirmation for a detached run comes from history status.

**History is a queue record, not storage.** An unknown `prompt_id` returns HTTP 200 with an empty object; the payload is keyed by `prompt_id`, and completion requires both a non-empty entry and its completed status flag. History is bounded and evicted in order. Therefore the core harvests history and fetches outputs immediately within the same run, verifies hashes, and treats history as non-durable — §6.2's "authoritative for artifact file names" means *at harvest time*.

**Memory release is asynchronous.** `POST /free` sets flags consumed after the current queue item; a 200 response proves nothing, the body must be valid JSON (a malformed body yields a 5xx), and both release flags must be set explicitly because their defaults are false. The supervisor confirms release by the free-VRAM delta on the primary device entry within a bounded wait; a non-releasing engine is marked `draining` and the queue stalls with `ENGINE_WAIT` rather than starting the next model family (§6.7). The device list is a **list** (one entry per accelerator, primary first) and the VRAM ledger reads it accordingly rather than assuming one entry.

**Ingress identity and hashing.** Uploaded file names are engine-assigned: content-hash deduplication may reuse a name or suffix `" (1)"`, and responses may carry additional fields, so the returned `{name, subfolder, type}` triple is authoritative and parsing is tolerant (§6.2's ingress row is correct to record it). Image-and-mask pairs use the documented mask-upload route rather than a hand-composited image. The engine mutates the submitted graph in place, so the core hashes the canonical graph **before** `POST /prompt`; a self-minted `prompt_id` must be a canonical lowercase UUID or it is rejected.

**Recipe parameter addressing.** Widget positions are not a stable interface: a node's input list changes across revisions, and optional inputs are catalogued separately. Recipe manifests address inputs and widgets **by name**, and startup resolves each mapping against the per-node object-info route at the pinned revision. An unresolvable mapping is a `VERSION_MISMATCH` scoped to that recipe — disabling one recipe rather than the engine — and positional widget indices are rejected at manifest validation. A recipe `semver` bump is mandatory on any node-signature-dependent change; §6.2.1's "recipes are data" is only meaningful if their version moves visibly.

**Cancellation and queue control.** `POST /interrupt` affects the currently running job (an id it does not match is ignored) and always returns 200; pending items are removed through the queue-delete route, which never enqueues. Consequently "cancel requested" (§13.3) is confirmed by event or history evidence plus a queue re-read, and the supervisor escalates to a bounded restart only after that confirmation window lapses.

**Miscellaneous obligations.** Pin one route prefix and assert it (the engine mirrors its API under a second prefix; the mirror is incidental and may narrow). Co-located instances require separate engine database targets, or a second instance exits at startup — surfaced as `STARTUP_FAILED` with the install-time explanation. The §6.1 inline budget (≤ 32 MB) is this project's own limit, not the engine's; file-based handoff through staging is the default path and inline payloads the exception. Progress-event payload shapes are verified per pinned revision rather than assumed.

### 25.2 vLLM

**Readiness is role-scoped.** Liveness plus a served-model name match establishes that an instance is up; §6.3.1's probe set establishes which *roles* it may serve, and the result is cached under `(model revision, engine version, probe-set version)`, re-run after any restart or pin change. An instance never binds to a role whose required capability probe failed; the queue item waits with `ENGINE_WAIT` or the plan errors as BLOCKED (§4.3), and there is no substitute-model fallback (§30.4).

**Structured output has a declared degradation path.** Where strict JSON-schema response formatting is probed absent for a served model, the adapter uses a schema-in-prompt task profile plus a strict parser with at most one repair attempt; a residual parse failure is `OUTPUT_INVALID` and a finding — never a best-effort coercion of prose into fields, which would be fabrication wearing a schema (§24-R1).

**Sampling discipline is part of the evidence model.** Recognition and critic task profiles use greedy decoding with a pinned seed, both recorded (§6.1). Multi-hypothesis generation (§9.4's max-hypotheses) may raise temperature or sample `n > 1`; such samples are **one observation of one lineage** — the kernel collapses them into a single evidence node carrying `alternates`, so sampling diversity never masquerades as ensemble corroboration. That is §7.7 R2 applied at the call site, and it is the specific mechanism by which R2 stays closed.

**Abort and timeout contract.** Client-disconnect aborts are *probed*, not assumed: a contract test asserts that closing the stream stops generation and returns VRAM to baseline. The core's per-role wall-clock ceilings (§6.3.3) are enforced client-side and paired with an explicit cancel; a breach is `EXECUTION_FAILED(retryable=true)` with partial output discarded, never partially committed.

**Token and payload budgeting.** `--max-model-len`, the image-token cost of a page at a given ladder position (§18.4), and per-request multimodal limits are registry parameters per role (§30.2). The request composer tiles or segments when a page would exceed them, records the segmentation in provenance, and re-anchors boxes back into the named space through the ledger (§7.5) rather than returning tile-local coordinates. Logprob availability is a probed capability and optional: §7.7's confidence is *calibrated*, so no requirement here depends on raw logits.

### 25.3 docling-serve

The pinned route contract lives in `engines/docling/contract/` as recorded request and response payloads, re-asserted at setup (§6.4, §22.2). Additional obligations: page ranges are requested per work unit so a 600-page book never yields one monolithic response; extracted images are ingested into CAS by hash before the document JSON references them, keeping the engine's temporary tree non-authoritative (§12.3); a mapping failure on any object yields a finding plus an `UnknownRegion`, never a silently skipped region (§7.3's honest catch-all); and the engine's own internal model revisions are captured into provenance, because they are inference and therefore evidence (§6.1).

### 25.4 Java validators

The invocations, exit-code map, and parsing are §6.5's, with three additions: tool and profile versions are recorded in the export report (§14.6), because a finding set is meaningless without the flavour and rule pack that produced it; a validator exiting with a *tool* code rather than a verdict code is an `EngineError`, never zero findings; and rule identifiers pass through a checked-in mapping table so a renamed upstream rule produces an explicit mapping finding instead of silently vanishing from a report. The clause coverage actually implemented by the pinned validator is recorded in §31 rather than assumed.

### 25.5 Contract coverage discipline

Every row of every §6 table is covered by at least one executable assertion, and every `EngineError` code has a negative test that provokes it. Contract fixtures are recorded against pinned revisions; a fixture that no longer reproduces at its pin is a build failure, not a flaky test. This discipline is what makes §24-R10 (pin rot) *managed* rather than open: engine churn becomes a red build instead of a production surprise.

---

## 26. Semantic-kernel completions

§7 defines the kernel but defers five mechanisms it depends on: taxonomy extensibility (§7.3's open reference), identifier and versioning rules sufficient for projection reverse-mapping, the data shapes behind principle 7's protection and review semantics, the derived indexes that §16.5's search and §7.7's `book_repetition` evidence require, and the ordering/identity rules that make provenance trustworthy across a power cut. This section supplies them; it changes no §7 normative statement.

### 26.1 Object taxonomy extension (§7.3's registered-type mechanism)

The v1 taxonomy (§7.3) is a registered set in `schemas/taxonomy.toml`, and extension is registration — never a schema fork. A type may be added within a major kernel version; removal or repurposing is a major-version event. Registering a type requires all of the following, and the kernel refuses to load a taxonomy entry that omits any:

- **payload schema** (JSON Schema, published in `schemas/`, versioned with the type);
- **parent/child and ordering constraints** (valid structural parents, whether it participates in the default reading path, and its reading-order class: `main · branch-optional · branch-essential · duplicate-presentation · artifact · non-sequential`);
- **projection rows for every shipped projector** (§15.1) — `full`, `partial+fallback`, or `dropped+finding`, with the fallback mechanism named. A type registrable only with blank PDF or EPUB rows is rejected, because the cheapest way for an extension to cause silent content loss is to be invisible to a capability matrix;
- **finding categories** it can raise, and the assessor categories (§14.2) it counts toward;
- **invalidation semantics** — which fields modules may touch, mirroring §11.2;
- **owner and migration note.**

Types arrive from modules through the extension registry (§7.13) under `ext:<module-id>/type/<name>` and are validated against the module's published schema. An unknown type encountered on read is preserved verbatim, rendered as its closest generic behavior (a region with geometry and text, at minimum), and reported as a finding — never dropped and never coerced into `Paragraph`, because a false specificity is worse than an acknowledged gap (§7.8's principle applied to types). `UnknownRegion` remains the honest catch-all for *unclassified content*, which is a different condition from *unregistered type*, and the two produce different findings.

### 26.2 Identity, versioning, and reverse mapping

Identifiers are opaque, immutable, and never reused: an id is bound for life to one semantic thing, and §7.3's "edits create new versions, not mutations" means each version is a distinct record with `{object_id, version_seq, created_by_state, predecessor_version}`. Consequences that must hold:

1. **Findings, evidence claims, review states, protections, and comments attach to `object_id`**, not to a version, so a review decision survives a text edit and a stale finding is resolved by the edit rather than orphaned by it.
2. **Projections emit source maps keyed by `object_id`** (§15.1) and the PDF projector records its `/MCID` table against `(object_id, version_seq, state_id)`, so an old export can be re-read and re-anchored (the export-audit requirement of §15.3) and so a projection edit reverse-maps to the *current* version rather than the one that happened to be exported.
3. **Deletion is a tombstone** carrying a reason (`merged_into | reclassified_as | artifact_promoted | user_deleted`) and the prior geometry in its named space. Merges and reclassifications must be traceable, because §14.6 reports "what changed" to institutions and §21.3's semantic digest must stay computable across them.
4. **Identity across sources**: when §8.4 reconciles a supplied text layer or a second edition, the reconciliation links claims to the same `object_id` rather than creating parallel objects, and edition-specific divergence becomes a finding on that object. Two objects for one thing is how a book silently acquires two texts.

Id generation is `uuidv4` at creation for objects and spaces; ordering is never inferred from an id (§26.5). Stability is a tested property: §21.1 asserts that a patch sequence preserves ids, and a migration that would renumber objects is a major-version event with a mapping table (§12.5).

### 26.3 Protection, review, comments, and decisions

Principle 7 ("automation routes around protected content") needs data. Three records, all namespaced under the kernel and all committed through §13.2 like anything else:

```text
Protection { id, target: {kind: word|span|object|region|page|chapter|artifact|checkpoint, refs, space_id?},
             mode: hard | propose_only,        # hard = no automated write even proposed
             reason, created_by: user|module, evidence_refs, state_id, active }
Comment    { id, target refs, body, author, created_at, thread_root?, exported: false }
Decision   { id, target refs, kind: accept_uncertainty | keep_annotation | resolve_conflict
             | waive_finding | approve_description, rationale, evidence_refs, actor: user, state_id }
```

**Precedence:** `user_input` evidence (§7.7) outranks model output; a `hard` protection outranks any module request, planner recommendation, or Continuous-Auto pass (§14.5); `propose_only` permits a finding and a proposal but forbids a write; protection beats a module's declared invalidation rights (§11.2), and a module that attempts a protected write fails `validate` as a contract violation (§7.5 rule 3's sibling). Protected content is not merely skipped — skipping is invisible — so each skip produces an informational finding naming the protection, which is what lets a reassessment report "94 % with 6 % protected" honestly (§14.1).

**Transports, not copies.** A protection over imagery or geometry names its space and is re-anchored through the ledger when a transform is accepted (§7.5); if transport is not computable (a dewarp mesh was lost, a region was merged), the protection is **escalated to a blocking finding** rather than dropped or approximated. A silently relocated protection is worse than a missing one, because it certifies the wrong pixels.

**Review state vs protection:** the §7.3 review enum is the *signal* (this has been looked at, and by whom), while `Protection` is the *mechanism* (nothing may write here). `do-not-modify` sets a `hard` protection on its target automatically; clearing the review state does not clear the protection, and the UI says so — a protection lost through a dropdown is the kind of surprise that ends institutional trust.

**Comments and decisions are project-private.** Comments never reach a projection (§16.4's inspection affordance) and decisions reach only the export report's review summary, in the reduced form §14.6 defines. Both are covered by the pre-share redaction control (§17.5): converting a project to portable for handoff (§12.1) offers a redaction pass over comments, decisions, `external/` paths, and annotation-bearing artifacts, because "Confirmed against library print copy, shelfmark X" is provenance for us and a disclosure to a third party for the reader.

### 26.4 Derived indexes, search, and book-wide repetition

§16.5's project-wide search and §7.7's `book_repetition` evidence class are both impossible with row scans over a 10⁶-object database, and neither is allowed a new dependency by §5. Requirements:

- **Substrate:** SQLite's FTS5 and JSON1 extensions, with their presence asserted by the readiness check at launch (§4.4). They are features of the embedded database, not additions to the dependency manifest, and a build without them is unsupported rather than degraded.
- **Index scope:** canonical text, assistive text, visual-display text, alternates, metadata, roles, finding text, review states, and provenance lookups by artifact hash and engine call. Indexes are **per committed state, updated incrementally from the patch** (§13.2's diff), never global and never rebuilt on demand in the request path.
- **Classification:** every index is regenerable (derived from kernel rows), therefore excluded from portability requirements, deletable by §22.6's safe eviction, and *never* the source of truth. A stale or missing index is reported in the UI as "search index out of date — [rebuild]" and the search result set is labeled accordingly; an unlabeled partial search result is how a user concludes content is absent.
- **Ordering rule:** search results are ordered by structural position (reading-graph order, §7.9) and then relevance, so the sequence a practitioner reads matches the book rather than a token statistic.
- **Repetition evidence is lexical, by design.** `book_repetition` is built from normalized-token indexes, near-duplicate detection over spans, and the BookProfile's learned vocabulary and confusion matrix (§10) — not from embeddings or a vector store, which §5.7 deliberately does not admit. That is a considered narrowing, not an oversight: the evidence the product needs is "*this exact term, spelling, or glyph appears elsewhere in this book, in these spaces*", which is exact-match and edit-distance work, is inspectable by a reviewer, and is reproducible without a model whose lineage would contaminate §7.7's independence rules. `local_context` is likewise a windowed adjacency query over the reading graph plus the page surface, not a retrieved-mix of choice.
- **Budgets:** index build and query costs are measured in §18.1's CI benchmark, with a stated per-query ceiling at the reference scale; exceeding it is a performance defect (§21.8's class only for regressions beyond §18.5's band).

### 26.5 Time, ordering, and identity under failure

Timestamps in every record, manifest, report, and export package are RFC 3339 in UTC with sub-second precision and an explicit offset; durations are recorded separately from monotonic sources. Ordering, however, never depends on the clock: **the journal sequence number and the state's `seq` are the only ordering authorities** (§13.4). Provenance `started`/`finished` fields exist for cost accounting and display, and are explicitly advisory for causality — a claim about which engine call produced which artifact is answered by the dependency edges, never by wall-clock proximity, because a suspend/resume or an NTP correction can invert timestamps while the data is perfectly intact. Consequences: history display sorts by `seq`; a clock jump produces an informational finding ("system clock moved by N; ordering unaffected"); export reports print both the human timestamp and the state `seq`; and §21.5's failure-injection suite includes a backward clock step at a transaction boundary.

### 26.6 Extension bags and canonicalisability

`ext:<module-id>/<name>` bags (§7.13) must be JSON-serializable in a canonical form: sorted keys, no floating-point values whose textual form is not stable across platforms (quantise explicitly or store the value as a rational/string), and no inline binaries — anything binary is an artifact referenced by hash. The reason is §21.3: the semantic digest includes extension bags verbatim, so a module writing `{"seen_at": <now>}` or an unnormalized float into a bag would break round-trip equality and turn a healthy project into an apparently changed one. This constraint is checked by the SDK's conformance suite (§21.2) rather than discovered at export time, and it is enforced at commit: a non-canonicalisable bag is a validation failure of the module, not a warning.

---

## 27. Non-main-flow content

§7.9 defers the placement rules for sidebars, §7.2 defers the pagination requirements, and §7.3 registers furniture, sidebars, and pull quotes as types without stating how they are decided. This section supplies those rules. The organizing principle behind all of them: **location is evidence, never a decision.** Page zones are statistical regularities learned by §10, and textbooks routinely place real content — a definition, an answer key, a theorem number, a continuation marker — in the zones that furniture classifiers want to delete.

### 27.1 Zones, furniture, and the anti-discard rule

Zone geometry (running-head band, footer band, gutter, outer margin) is learned per page template and recorded as a BookProfile convention citing the pages it came from (§10). A region inside a furniture band becomes a `PageFurniture` artifact only if **both** tests pass:

1. **Recurrence:** the region matches the template's zone content across a profile-declared share of pages (near-duplicate text or geometry, computed from the §26.4 lexical and geometry indexes, not by eyeballing one page).
2. **Non-instructional test:** the region contains no unique instructional content — specifically, none of: a continuation into or out of main-flow text, a cross-reference target or note anchor, a member of an enumerated series (equation, figure, exercise, or example numbering), a standalone numeric or unit value, a section or chapter title used for navigation, or a "continued" marker.

Failing either test, the region is a normal object of its own type, and a `needs-attention` finding fires when the classification was close. Furniture marking is a **claim with evidence** (§7.7), reviewable like any other, and automated artifact-marking of *text* requires the ensemble supports §7.7 R3 demands rather than a single classifier's confidence.

Three consequences, each preventing a specific known harm:

- **Furniture is retained, not discarded.** Artifact-marking excludes content from continuous reading; it does not delete it. The text stays in the kernel as page metadata and as source evidence, and the pixels stay in the image (§7.8(2)'s honesty rule, principle 4) — so folio numbers, running heads, and stamps remain citable, searchable (§16.5), and available to a later projection that can express them (§2.3's deferred tactile and DAISY targets).
- **Running heads are useful.** A running head carrying a chapter or section title is harvested as *metadata evidence* for landmark and heading repair (§10's templates), which is how a book whose chapter-opener was cropped still gets correct outline structure.
- **Rules are not automatically decorative.** A graphic rule is `/Artifact` only when it neither delimits a table nor separates an answer key from a question — the same recurrence-plus-instructional test applied to geometry rather than text (§7.3's `nontext`/decorative marking in PDF).

### 27.2 Printed folios, page labels, and markers

Per physical page, a **folio record**: `{printed_label (verbatim glyphs), value (integer|roman|none), style, polygon + space_id (§7.5), confidence, evidence_refs, state: present | absent_by_design | illegible | conflicting}`. Rules:

1. **Printed folio ≠ digital index; both are stored** (§7.2) and neither is derived from the other at export time. The mapping table is a committed artifact, reviewed like any other hypothesis.
2. `absent_by_design` (plates, blanks, section openers) is recorded explicitly rather than inferred from a gap, because a gap is also the signature of a missing scan (§8.2 inventory) — the two must not be conflated, and conflating them turns a lost page into a numbering "quirk".
3. Where the printed folio is illegible, the state is `illegible` and the digital index is used for navigation only. **Interpolating folios across a gap is prohibited by default:** a single wrong inference silently corrupts every downstream page label, citation, and page-list entry. Interpolation is available only as an explicit user action recorded as `user_input` evidence (§7.7).
4. **Projections:** PDF emits page labels for printed folios including roman front-matter sequences (§15.3's catalog item), so "go to page ix" matches the physical book; HTML emits `data-page` markers plus a page-list navigation (§15.2); EPUB emits page-break markers and a page-list (§15.4). Internal destinations anchor to *objects*, never to page numbers (§7.11), so a re-pagination decision cannot orphan a link.
5. Folio conflicts across multiple sources (§8.4) become findings with both values side by side; the export report counts them (§14.6), because in a legal or scholarly citation context "the scan's own page numbering disagrees" is exactly the fact an institution needs recorded.

### 27.3 Sidebars, callouts, and inset content

Determination signals, each recorded as evidence on the object: bordered or tinted containment; display typography distinct from body; an inbound cross-reference from main flow ("see Box 3.2"); the **continuity test** (does a main-flow sentence resume around the region?); membership of an enumerated series; content markers (example, exercise, definition, warning, case study); and anchor proximity. The decision is a two-way classification with a third honest state:

- **Essential** (the argument is broken without it, or main flow resumes around it, or it is referenced by number): placed inline at its anchor in the default reading path, before the continuation of the interrupted flow.
- **Supplementary** (interesting, self-contained, unreferenced): placed after the enclosing paragraph or on an optional branch with a jump affordance, per §7.9's branch rules.
- **Unresolved:** branch plus `needs-attention` finding. Never dropped, never silently promoted to main flow — §7.9's validator rejects a meaningful visible region absent from the default path unless it is explicitly classified branch/duplicate/artifact.

The **anchor binding** is mandatory and points at a main-flow object with a return position, so a projection that can jump out can jump back. If no anchor is found, the region is placed at page scope and a finding says so, because an unanchored sidebar is a navigation dead end. Placement conventions learned for one book are recorded in the BookProfile (§10) and cited to their pages, which is what makes them reviewable and what lets a user correct the convention once instead of the placement three hundred times.

**Projection behavior** follows §15.1's matrix and §7.4's rule that a semantic may only be expressed where the target has a mechanism: HTML uses `<aside>` with the registered DPUB-ARIA role (§31.1); EPUB inherits it from the HTML projection; PDF maps to the nearest standard structure element or a role-mapped extension, and where no mechanism expresses "optional branch", the region is placed in structure order and the matrix records `partial+fallback` — the reading experience is linear, and the fact is documented rather than hidden. Skippable/escapable playback semantics belong to the deferred DAISY-class projections (§15.4) and must not be faked in v1.

### 27.4 Pull quotes and duplicate presentations

Detection: high text similarity to a main-flow span (a §26.4 lexical comparison above a profile-set threshold), display typography, and a column- or page-spanning position. Representation: the pull quote is its own object, type `PullQuote`, linked to the canonical span by a **duplicate-presentation** edge (§7.9). Consequences: it remains visually present in every projection and appears once in the reading stream; the default path excludes it, and the link is what satisfies §7.9's "no region twice without a duplicate-presentation link". Per-projection mechanics, as matrix rows: HTML/EPUB use the registered role where supported and otherwise hide the duplicate from AT while retaining the visual element; PDF marks the duplicate as an artifact with an object-level record of why, since PDF has no "in structure but not in reading order" mechanism — the fallback is documented in §15.1's matrix, not discovered by a screen-reader user. A pull quote that differs from its source span by even one word is **not** a duplicate: it is an excerpt with its own text, and the similarity test failing in that direction is what prevents an altered quotation being silenced (§7.4's canonical-fidelity rule).

### 27.5 Marginalia, handwriting, and stamps

§9.4's classification-first rule, with the layer mechanics it assumes. Each page carries an **annotation layer stack** separate from the printed layer: `{layer_id, class: publisher_content | reader_annotation | instructor_annotation | historical_annotation | damage, mask_artifact_hash, isolated_raster_hash?, geometry (space-named), evidence, review_state}`.

- **Retained by default, always.** Removal is a per-class, per-decision user action recorded in a `Decision` (§26.3) with rationale, and the retention layer survives into the project even when a projection omits it — a prior owner's annotations may be the only surviving record of a book's use, and they are frequently PII (§17.5).
- **Separation is a precondition for cleanup, not a side effect of it.** Where a handwritten layer overlaps printed text, restoration masks are derived from that layer so the cleanup pass operates on the printed layer only (§9.1 step 2's protection masks). If separation is not achievable — ink bleeding through the letters, a highlighter dissolving the underlying type — the region is flagged `needs-attention` and excluded from automated restoration. Blending the two and hoping is how a system destroys a primary source while improving its OCR score.
- **Not publisher text.** Annotation content never enters canonical or assistive text of the publication (§7.4) unless the user explicitly promotes it, recorded as `user_input`, because reading a 1953 student's note in a screen-reader stream is not the author's text.
- **Stamps, ex-libris labels, watermarks, and copyright notices** are protected classes under §17.5: detected by classifier, protected by mask, and removable only by explicit user decision. Detection is a finding, not an edit.
- Unclassifiable ink defaults to `damage` **and** retains the layer: the asymmetry matters, because retaining is reversible and removing is not (principle 4).

### 27.6 Continuations and cross-boundary relations

Paragraph continuation across a column, page, or spread boundary is a **reading-graph edge**, not a text mutation: canonical text keeps its page-boundary structure so every claim remains addressable at its source (§7.6's alignment identity), while the assistive and reading streams join across the boundary. Corollaries:

- A hyphenated break at a boundary is a hygiene decision with a recorded `{join | keep}` outcome per instance (§29.1), never a global flag — the same page can contain "account-ing" and "well-known".
- **"Table 3.2 (continued)" is content.** It attaches to the table object as a continuation marker, precisely because §27.1's instructional test fails it out of furniture eligibility. Continuation tables keep one logical table identity with per-page fragments, so spans and header scopes close across the break (§7.9's table-ownership constraint) and a linearised reading of the second fragment still names its header row.
- Figure-and-caption pairs split across a boundary keep their binding by id, not by adjacency; a caption on one page and its figure on the next is a finding, not a guess.
- Two-page spreads record both source pages and one logical region, with the seam in the ledger (§7.5) so overlays can show where the physical book actually broke.
- Note references and their returns may cross pages (§7.10); the return anchor names an object and a position within it, so a projection can put the reader back exactly where they were rather than at the top of a paragraph.

---

## 28. Specialist content modules

§7.12 defines what these objects *store* and §9.1 step 4 says they are "split to specialist modules"; this section defines how they are produced and verified inside the frozen dependency set. The governing constraint is worth stating explicitly because it shapes everything below: **no additional model runtime may be added** (§0, principle 10, ADR-004). There is therefore no separate table-structure model, formula recognizer, or chart parser in v1. Specialist structure comes from three admissible sources — the recognition ensemble's `structure_hints` (§7.6), docling-serve's internal table model on the born-digital path only (§6.4), and deterministic CPU verification over the kernel's own geometry — and where those are insufficient the correct output is a **finding with a usable fallback**, never a confident wrong grid.

### 28.1 Tables

Production: region detection and cell grid from `structure_hints`; header, span, and caption binding from structural inference (§9.1 step 4) against BookProfile conventions (§10); on the digital path, docling-serve's table structure recorded as `prior_structure` evidence — *not* as verified truth (§8.3). Verification is deterministic and belongs to the core: **span closure** (no cell overlapping another, every grid position accounted for, no negative or fractional spans); **header association** (every data cell resolvable to its row and column headers; `scope`/`headers` expressible); **caption binding** (one caption per table identity, §27.6's continuation rule); **unit cells** recognized so a linearised reading does not strand a unit; and **linearisation** computed, regenerable, and overridable (§7.12) with the override recorded as `user_input`.

Where irregular structure cannot be verified — nested headers, split diagonal corners, a cell whose border ink is missing — the module emits the *largest defensible* grid plus findings naming each contested span, and the projection uses `partial+fallback`: the table remains readable linearly, contested spans are announced, and the object keeps its degraded state through to §14.6's report rather than being exported silently as though it closed. Complex irregular tables remaining unresolved after export is the designed outcome, not a defect; what would be a defect is exporting an invented structure.

Projections: PDF `Table/TR/TH/TD` with `THead`/`TFoot` (§15.3); HTML with `scope`/`headers` and `<caption>` (§15.2); EPUB inherits from HTML.

### 28.2 Mathematics

Recognition produces canonical LaTeX; the kernel stores LaTeX plus **validated** MathML and never raster-only truth (§7.12). Validation is symbolic and CPU-only: parse the LaTeX, serialize to MathML, and check symbol inventory, operator arity, and delimiter balance; a failed parse is a finding, never a silent fix (§9.3). Numerals, operators, and scientific symbols are scored as their own track (§14.2, §20.1), because aggregate word accuracy hides exactly the errors that make a STEM text unusable.

**What v1 does not do, by decision:** pixel-level re-rendering of full two-dimensional equation layouts for verification requires a mathematical typesetter, and §5 names none. Verification is therefore (i) symbolic, (ii) **symbol-level render-back** — re-shaping the recognized symbol sequence with the page's learned faces from §10's glyph gallery and comparing local ink against the source region through the ledger, which catches substituted and dropped symbols — and (iii) human findings for layout depth (subscript nesting, matrix brackets, alignment). This limitation is recorded in §15.1's matrix and in the export report rather than being papered over with a plausible rendering.

**The plus-one rule.** Navigable, linearisable mathematics is a *format-family* property: PDF expresses `Formula` with `/Alt` and a linearized `/ActualText` (§15.3), which is a faithful-but-flat rendering, while HTML and EPUB carry MathML (§15.2, §15.4). The matrix row for math in PDF is therefore `partial+fallback` by design, and any project profile with substantial mathematics must be exported to at least one MathML-bearing projection, with the export report stating that mathematical navigation in the PDF is limited. Claiming that a tagged PDF alone provides equivalent mathematical access is prohibited product language (§31.3).

### 28.3 Charts and graphs

Chart payloads store extracted data **with** per-value uncertainty, the axis model, and the description set (§7.12). Verification is deterministic CPU work over artifacts the pipeline already has: **re-plot within tolerance** — rasterise the extracted series, axes, and labels with the CPU imaging stack and compare ink placement, series count, and monotonicity against the source region; failures are findings naming the contested series or value. Interpretation checks are mandatory before a description is considered usable: logarithmic versus linear scale, truncated or non-zero baselines, inverted axes, dual axes and their series binding, error bars and uncertainty bands (a bar read as a data point changes the claim), legend-to-series binding, unit and scale text, and color-only encodings, which must be reported as inaccessible-without-a-described-encoding (§14.7).

The output set per chart is `{short description, long description, extracted data table, axis/scale statements, pedagogical explanation}` — each separately versioned and provenanced — and the data table is not optional where a chart encodes values: a chart whose numbers cannot be read is a finding, and a tactile-referral recommendation is emitted where a raised or braille graphic would serve better than any projection v1 offers (§7.12's referral field).

### 28.4 Figures, illustrations, and descriptions

Every non-artifact figure carries a classification (§14.7's existence check) and, where not decorative, the description variants of §7.12. "Decorative" is a claim requiring evidence and review, never a default for unrecognized content — the failure mode in the other direction (real pedagogical content silenced as decoration) is at least as damaging as a missing alt text. Generated descriptions may satisfy existence and form checks but never the `approved` review state without a human action (§14.7), and the pedagogy check runs against surrounding text, inbound references, and likely instructional questions as critic-generated findings (§14.7) that a person resolves.

Projection mechanics: HTML `<figure>`/`<figcaption>` with `aria-describedby` long-description links; EPUB inherits; PDF `Figure` with `/Alt` plus a long-description object or `/E` (§15.3). Images cross boundaries as CAS artifacts at the §18.4 export policy resolution, and the source raster is never overwritten by a described or cropped derivative (principle 4).

### 28.5 Chemistry, circuits, music, maps, and other domain notation

Domain payloads are the extension path, not a special case in the kernel: registered types (§26.1) carrying module-published schemas (§7.13) and artifact hashes, opaque to the core (§7.12). Two normative positions: general image captioning **is not assumed** to solve these domains, so a figure in a registered specialist class receives no caption-derived `approved` state without its domain module's own verification; and where no module applies, the honest states are `UNKNOWN` (capability exists, not attempted) or `UNRECOVERABLE{outside_capability}` (§7.8) — never a plausible structure diagram rendered as prose. v1 ships no domain-specific model and no domain module beyond the generic set (§28.6); what it must ship is the unblocked path — types, schemas, findings categories, and matrix rows — so that adding one is additive (§2.3's obligation).

### 28.6 Built-in module inventory (v1)

Built-ins obey the same contract as third parties (§11.1, ADR-011); this is the enumerated set the rest of this document implies. Engine-role references are §30.1's; every module declares resources (§11.2) and is subject to §9.6's stops.

| Module | Purpose (§) | Roles | Characteristic findings |
|---|---|---|---|
| `triage.probe` | §8.2 CPU probes, inventory, path selection, cost profile | — | missing/blank/rotated pages, defective text layer, semantic color, script inventory |
| `import.digital` | DoclingDocument → kernel (§6.4, §8.3) | `structure.extract` | structure gaps, order breaks, prior-evidence conflicts |
| `import.textlayer` | Supplied OCR text → Recognition Result envelope (§8.3) | — | alignment coverage, unsupported scripts |
| `geometry.normalize` | deskew/crop/rescale/dewarp as ledger transforms (§7.5, §9.1 pass 1) | — | `ALIGNMENT_STALE`, estimate-vs-visual disagreement |
| `restore.cleanup` | unconditioned page restoration (§9.1 pass 2) | `diffusion.restore` | out-of-mask change, smoothness drift, protected-class collision |
| `recognize.ensemble` | ≥2 families → Recognition Results (§9.1 pass 3) | `vlm.ocr` ×2 | `suspicious_spans`, low `n_eff`, script capability gaps |
| `structure.infer` | layout/role classification, reading graph (§9.1 pass 4, §7.9) | `vlm.critic`, `llm.repair` | unresolved regions, order constraint violations, `UnknownRegion` |
| `profile.learn` | BookProfile construction and revision (§10) | `vlm.critic` | convention instability, glyph-gallery gaps |
| `text.repair` | LLM correction of canonical text with evidence (§9.1 pass 5) | `llm.repair` | role-change lint, uncorroborated edit, protected collision |
| `text.hygiene` | §29.1 checklist; assistive text construction | `llm.repair` | ambiguity requiring a decision, whitespace-significant conflict |
| `align.forced` | text↔pixel alignment, re-alignment (§7.6) | — | unrecoverable alignment, geometry anomaly |
| `glyph.repair` | targeted masked text painting (§9.4) | `diffusion.inpaint` | R4 lineage recorded, residual ambiguity, neighbourhood invariance breach |
| `adjudicate.counterfactual` | hypothesis scoring against degraded renders (§9.5) | `vlm.critic` | abstentions, below-threshold separation |
| `specialist.tables` / `.math` / `.charts` | §28.1–§28.3 | `vlm.ocr`, `vlm.critic` | span non-closure, parse failure, re-plot divergence |
| `describe.generate` / `describe.assess` | §7.12, §14.7 | `vlm.critic` | form violations, pedagogy gaps |
| `notes.link` | marker/target/return reconstruction (§7.10) | `vlm.critic` | orphaned markers, missing returns, symbol renumbering attempts |
| `validate.projection` | run §6.5 validators, map to findings | `validator.*` | every validator rule, mapped-and-unmapped rule codes |
| `assess.reassess` | §14.2 scoring and reassessment records | — | the finding set itself |

---

## 29. Text production: hygiene, variants, and the PDF text layer

§9.1 steps 5–8 and §14.2's "speech cleanliness" category presuppose a text-hygiene discipline and a text-layer construction rule that §7 and §15 assert but do not enumerate. This section is the contract for both. The organizing distinction, used throughout: **canonical text is what a reader would type to search or quote this book; the visual variant is what the page shows; the assistive variant is what a listener should hear** (§7.4). A module may restore the first, present the second, and derive the third — it may not collapse them.

### 29.1 Hygiene checklist

Every row is detection-signal → action, and every action is one of `canonical` (changes canonical text), `visual` (changes only the visual variant), `assistive` (changes only the assistive variant), `structure` (changes object segmentation), or `finding` (proposes, requires review). No row's action is silent, and every applied change is a logged, reversible diff (§7.4, §13.2).

| Signal | Detection | Action | Notes |
|---|---|---|---|
| Soft hyphens | codepoint U+00AD | `canonical` remove; retain position in the visual variant | never inferred from ink alone |
| Line-end hyphenation | hyphen at line break, dictionary-free evidence: prefix/suffix occurrence of the stem elsewhere in the book (§26.4), §10 vocabulary | `canonical` join **per instance** with `{join｜keep}` recorded; assistive always joined | "account-ing" vs "well-known" can coexist on one page (§27.6); a join that no evidence supports is a `finding`, not a join |
| Non-breaking / zero-width / unusual spaces | category scan (NBSP, ZWSP, ZWNJ/ZWJ, thin spaces, control chars) | `canonical` normalize to the space the text means; `finding` where the character is load-bearing (Arabic ZWNJ, joiners) | joiners are script structure, not dirt |
| Missing or spurious spaces at OCR boundaries | ink-gap vs advance comparison through the ledger (§7.5), lexicon hits | `canonical` with evidence; below-threshold cases → `finding` | word-splitting changes alignment identity (§7.6) → `ALIGNMENT_STALE` |
| Duplicated spaces | lexical | `canonical` collapse outside whitespace-significant content (§29.4) | — |
| Malformed Unicode, unpaired surrogates, PUA | strict decode; PUA ranges | `finding`; a PUA glyph may be mapped only with §10 glyph-gallery evidence, recorded as `typography_prior` | never emit U+FFFD into canonical text; a private-use character that survives to export keeps its codepoint plus a report line |
| Display ligatures (`ﬁ`, `ﬂ`, `ﬅ`, cst ligatures) | codepoint inventory + font-catalogue mapping | `canonical` decomposed for search and copy; `visual` keeps the ligature | the ToUnicode mapping must produce the decomposition (§29.3), which is where this becomes measurable |
| Split drop caps | oversized initial glyph isolated as its own region | `structure`: one paragraph object + `DropCap` (§7.3) + visual variant | canonical text carries the letter once, in order |
| False paragraph breaks / fused paragraphs | baseline and indent geometry vs §10 templates; sentence-boundary punctuation | `structure` with evidence; unresolved → `finding` | segmentation change ⇒ dependent alignments go stale (§7.6) |
| Furniture embedded mid-sentence | running-head text appearing inside a paragraph span | `structure` extract + reattach as artifact (§27.1) | classic double-read in scanned books |
| Page numbers inserted mid-sentence | numeric token at a page-boundary position, folio record match (§27.2) | `assistive` skip at minimum; `canonical` removal only when the folio record corroborates and the continuation is grammatical | joining across a page boundary is a reading-graph edge (§27.6), not a text deletion |
| Decorative symbols spoken meaninglessly | dingbat/ornament inventory, PUA ornaments | `assistive` suppress or replace with a spoken label; `visual` retained | suppression is a claim about intent ⇒ reviewable |
| Superscripts, subscripts, Roman numerals, units, abbreviations | codepoint ranges, §10 conventions, footnote-marker geometry (§7.10) | typography **hints** only (§7.3); `assistive` expansion where speech requires it ("m²" → "square meters" per profile); marker-vs-exponent disambiguation by geometry and evidence | `x²` vs a footnote reference `x2` is the same character sequence and a different object — the disambiguation is structural, and failure is a `finding` |
| Language or script change mid-run | §8.2 script inventory, per-object language tagging (§7.3) | `structure`: split the run and tag both; never guess a single language for the paragraph | also the PDF `/Lang` source (§15.3) |
| Punctuation that changes speech | dash/quote/parenthesis patterns, ellipses, list punctuation | `assistive` only | canonical fidelity is what makes the quote searchable |
| Whitespace-significant content | registered types (§26.1) + §10 templates: `CodeBlock`, poetry/verse detection, table cells, math linearisation | **protected by default**: no join, no collapse, no reflow | see §29.4 |

Two lint rules bind the whole table. **Role-change linting** (§9.1 step 5): a text edit may not silently change an object's role — if a repair turns recognized heading text into non-heading prose, the role change requires its own evidence and a finding, because a lost heading is a lost navigation landmark even when the characters are right. **Fidelity linting:** `canonical` actions must each trace to either a source observation, a corroborated ensemble claim, or `user_input`; a canonical change whose only support is `model_output` from one lineage is downgraded to a `finding` by §7.7 R2–R3, not applied.

### 29.2 Variant substitution and projection

§7.4's rule — a projection may substitute assistive text only where the target has a real mechanism — is implemented as a matrix the projectors must satisfy, checked in §21.3: canonical → PDF real text and `/ActualText`, HTML/EPUB content nodes; visual → PDF glyph selection and `/Alt`-free presentation, HTML/EPUB presentation only; assistive → PDF `/Alt` where a text alternative is legitimate (never replacing selectable text needed for quotation), HTML/EPUB `aria-label`/`aria-description` where the alternative does not destroy selectable content; pronunciation → no v1 mechanism, dropped-with-finding (§15.1). Where a projector would have to choose between *quotation fidelity* and *speech quality*, it keeps quotation and reports the drop, because a silent alteration of quotable text is the more damaging error (§7.4).

### 29.3 PDF text-layer composition

This is the mechanism behind §6.6 and §15.3's invisible layer, stated as a pipeline because it is the single most failure-prone centimeter of the product.

**Input:** a committed alignment (`canonical text version`, `surface version`, `space`) with per-run boxes and baselines (§7.6); the page's learned face inventory (§10); the export font policy (§6.6).

**Steps.**
1. **Segment** the logical run at script and directional boundaries. The projector owns the Unicode Bidirectional Algorithm: directional levels are computed here, and runs are split at level changes. HarfBuzz shapes a single directional run and is told that run's direction; it does not resolve directionality for us (§31.1). Conflating the two is precisely the bug that produces visually-correct-looking PDFs whose extracted text reads backwards.
2. **Shape** each visual run with uharfbuzz against the target face, collecting glyph identifiers, clusters, advances, and offsets, and mapping clusters back to canonical text offsets — the cluster map is what makes ToUnicode correct under ligature and reordering.
3. **Subset** the pinned OFL face with fontTools for the glyphs actually used and build the ToUnicode CMap from the cluster map, so that a copy/paste yields **canonical** text: ligature glyphs map to their decomposed sequences, a PUA-mapped glyph maps to its evidenced codepoint or produces a finding rather than a silent `U+FFFD`, and RTL runs yield logical order while their placement stays visual.
4. **Place.** Geometry, not font metrics, is the authority for where a glyph sits: the invisible layer must coincide with the *original printer's* ink, whose advances differ from our substituted face. Each alignment box is transported into PDF user space through the composed ledger (§7.5 rule 4) and used to position glyphs — a single advance chain with `TJ` word-space adjustments where boxes and metrics agree, per-glyph positioning (`Td`/`TJ` offsets) where they do not. Divergence beyond a declared tolerance is a `STALE`-quality finding, because a text layer that selects the wrong words is worse than no text layer (§24-R3).
5. **Mark and hide.** Operators are emitted inside the element's marked-content span with its `/MCID` (§15.3), render mode `Tr 3`, and `BT … ET` composition owned by the projector (ADR-014). The layer never alters visible content, and visible page imagery is byte-unchanged by this step.
6. **Verify by re-reading.** The projector's own output is parsed back with pypdfium2 (read-only) and compared against the kernel: extracted text ≡ canonical text for sampled runs, box centers within tolerance, `/Lang` and structure order intact (§21.2, A3).

**Representational limits, recorded as matrix rows, not discovered by users:** vertical writing (`ttb` baselines, §7.6) cannot be relied upon in the pinned stack, so vertical runs are either placed per-glyph in reading order or excluded from the invisible layer with a blocking-quality finding naming the page; and no requirement in this document depends on PDF 2.0 writing-mode facilities (§31.1). Complex-script shaping that the pinned face cannot render (no suitable glyphs in the OFL subset) is a font-policy finding, never a fallback to a font not in `fonts/MANIFEST.toml`.

**Determinism.** The subset, glyph ordering, object numbering, and CMap must be reproducible from (state, projector version, policy): no timestamps, no iteration over unordered collections, and deterministic document IDs unless randomized-on-request is explicitly chosen (§6.6). Byte-identical re-export is a golden test (§15.1, §21.5), and it is what makes "the file you shipped last year is the file we would ship today" an answerable question.

### 29.4 Whitespace-significant content

`CodeBlock`, detected verse, table cell text, and math linearisations are whitespace- and line-structure-bearing. Hygiene and repair modules must not join, collapse, or reflow inside them; the §10 profile supplies the detection templates, and a region whose significance cannot be determined is **treated as significant** and raises a `finding` — the asymmetry is deliberate, because a false positive costs one review and a false negative silently destroys a code sample or a stanza's line breaks. Where such content is exported, its presentation variant is what the projection renders, and the canonical form is what copy/paste must return (§29.2).

---

## 30. Model roles, registry, and broker policy

Modules request **roles**, never checkpoints (§11.2); the registry turns a role into a pinned, licensed, probe-verified instance; the broker is the only road from one to the other (§11.3). This section makes that seam explicit because it is where an "one capability, one dependency" policy (principle 10) either holds or quietly evaporates.

### 30.1 Role taxonomy (v1)

| Role | Engine | Purpose | Required capabilities (§6.3.1) |
|---|---|---|---|
| `diffusion.restore` | ComfyUI | unconditioned page cleanup (§9.1 pass 2) | graph load, mask respect, seed control, tile/stitch nodes |
| `diffusion.inpaint` | ComfyUI | masked, text-conditioned glyph and region repair (§9.4) | `{image, mask}` ingress, conditioning text input, deterministic seed |
| `vlm.ocr.parsing` | vLLM | primary document parsing/OCR family (§9.1 pass 3) | single-image reference, structured output, box emission |
| `vlm.ocr.grounding` | vLLM | second independent family / cross-check (§7.7 R2) | same as above, **different lineage** |
| `vlm.critic` | vLLM | visual-support, layout-role, and form checks (§9.3) | multi-image reference, structured output |
| `llm.repair` | vLLM | canonical text repair, hygiene reasoning (§9.1 pass 5, §29.1) | structured output, long context |
| `structure.extract` | docling-serve | born-digital structure and text-layer extraction (§6.4) | pinned route contract |
| `validator.pdf`, `validator.epub` | J tools | machine conformance checks (§6.5) | documented exit codes and report schema |

**Roles deliberately absent, and why** (each absence is a decision, not an oversight, and each is closed by something already in §5): an *embedding/retrieval* role — repetition and vocabulary evidence is lexical and index-based (§26.4), which is inspectable and adds no runtime; a *formula recognition* role — equations are produced by the OCR families and validated symbolically (§28.2), since a dedicated recognizer would be a second runtime; a *table-structure* role — same reasoning via §28.1; a *layout-detector* role — the classical detector path of §9.1 pass 3 is OpenCV geometry work executed **inside a module on CPU** (§5.4), which is why it needs no model role at all, while learned layout and role inference is `vlm.ocr.*` plus §9.1 pass 4 with deterministic validation, and docling's own layout model stays inside its engine (§5.4's isolation, ADR-008); *speech, ASR, and media-overlay* roles — audio production is deferred with DAISY-class projections (§2.3, §15.4); a *translation* role — out of scope (§2.5). Two vocabularies reconcile here: the capability strings a module declares in its manifest (§11.2's illustrative `diffusion.restore`, `vlm.ocr`, `llm.repair`, `structure.layout`) are the **role ids above**, and `structure.layout` resolves to `structure.extract` for born-digital input plus `vlm.ocr.parsing`/`vlm.critic` for imagery; a manifest naming anything outside this registry fails validation (§11.2) rather than binding loosely. If a new role becomes necessary, it arrives through a new ADR (§0's dependency policy), not through a role added in code.

### 30.2 Registry entries (`models/MODELS.toml`)

Every entry: `id`, `role(s)`, `repo`/`revision`/`file`/`sha256`, **`license` with a review block** — `covers: weights|code|both`, `text_sha256`, `source_url`, `reviewed_on`, `reviewer`, `field_of_use_notes`, `gating: none|login_required` — `engine` and launch flags (dtype/quantisation, context length, image-token budget, §25.2), `vram_reservation`, `timeouts` (§6.3.3), `adapter` (§6.3.2), `capabilities` (probe results with probe-set version), `task_profiles` (§30.5), `measured_capability` (§20.1), `status: evaluated | default | gated | retired`, and `lineage_family` — the architecture-and-training identity that §7.7 R2 compares.

Two normative rules, both earned the hard way:

1. **The license field describes the weights artifact.** A repository's code license is not evidence about checkpoints, "open weight" is not a license, and a card's license *metadata* without a shipped license text is a review finding rather than a clearance. `license.covers` must say which artifact was actually read, and an entry may not hold `status: default` while it says `weights, text not found`.
2. **`lineage_family` is a human-maintained assertion, reviewed like code.** It decides corroboration (§7.7 R2), so two checkpoints of one architecture differ from two families; a fine-tune of a family we already run is *not* an independent family; and a rename, merge, or re-training of a base model changes corroboration counts retroactively, which makes it a §32.2 governance event with a calibration re-check (§14.4).

### 30.3 Broker and leases

The broker is a core-provided loopback proxy: a module receives a **per-run, credential-less lease** on declared roles only, and every call it makes is recorded with full provenance automatically, so evidence accounting cannot be bypassed by construction (§11.3). Engine credentials stay in `tokens/` (§22.3) and never cross into a module environment or the browser (§17.1). The broker owns: role resolution against the supervisor's engine state (§6.7), queueing with backpressure rather than parallel over-subscription, per-call timeout enforcement (§6.3.3), connection-level retries exactly as §6.1 permits and never execution retries, cost attribution against the project's GPU-second ceiling (§9.6), and response caching under a strict key: `(role, model revision, adapter revision, task-profile hash, prompt hash, input artifact hashes, sampling parameters)` — cacheable only for deterministic profiles (§30.5), stored under `cache/` as regenerable (§22.6), and never shared across a model or prompt revision. Batching is permitted only within one role at one revision with one task profile, so a batched response cannot smuggle a different parameter set into provenance (§18.2).

### 30.4 No fallback substitution

There is **no runtime model fallback**, in either direction: not on error, not on timeout, not on VRAM pressure, and not on low confidence. A substitution would change the evidence lineage of the output while leaving the claim's history looking like the pinned assignment, invalidate the calibration that makes §7.7's confidence numbers mean anything, and destroy §18.5's cost accounting — all to avoid an honest failure. The correct behaviors are: `ENGINE_WAIT` while the pinned engine starts or swaps; a BLOCKED plan error when the tier cannot host the role at all (§4.3, surfaced per §14.5); or a failed item the user or planner re-queues (§6.1). Change is allowed only at the registry level: a new `default` assignment, approved through §20.4's re-validation and recorded in the export report. The same discipline forbids a module from selecting a different role assignment than the plan's; role requests are validated against the manifest's declared requirements, and `forbidden` capabilities are enforced (§11.2).

### 30.5 Determinism and prompt governance

Prompts are versioned data in-repo — template file, parameter set, expected schema, and the model revision they were tuned for — never string literals in module code, and the broker hashes them into `lineage.prompt_hash` (§7.7). Task profiles fix sampling parameters per role: recognition, grounding, and critic profiles are greedy with a pinned seed; repair uses the profile's declared temperature; multi-hypothesis generation raises either temperature or `n` and is recorded as **one observation of one lineage** (§25.2). Two consequences follow: an output is reproducible iff its (state, model revision, task-profile hash, seed) tuple is, which is what §6.1's provenance exists to make checkable; and **editing a prompt template is a behavioral change** requiring re-calibration (§14.4) and a bake-off re-run for affected roles (§20.4), exactly as a weight change would — because it is one.

### 30.6 Ledger and cost accounting

The VRAM ledger (§6.7) tracks reservations from registry entries plus a declared headroom margin, grants or refuses starts against them, and records observed peaks per model and input size so reservations converge toward measurement rather than folklore. Swap economics are explicit: each model family carries a measured cold-start cost, the scheduler prefers work-unit orderings with model locality (§18.2), `/free` is used between non-adjacent families and *confirmed* by a VRAM delta (§25.1), and a refusal produces a stall event rather than an out-of-memory failure. Plan estimates multiply validated manifest estimates by the current tier factor (§14.5); actuals are recorded per work unit and fed back into the cost model through `benchmarks/`, which is what makes §9.7's priority function a cost curve rather than a guess.

---

## 31. Standards and normative-reference register

The document cites standards by name in §2.3, §14, §15, and §16. This register pins what each citation means, when it was last verified, and — just as importantly — what we may say about it. Version identifiers are current as of **2026-10-05**; see §31.2 for the currency discipline.

### 31.1 Register

| Standard | Version / status | Where it binds | v1 posture |
|---|---|---|---|
| PDF base | ISO 32000-1:2008 (PDF 1.7); ISO 32000-2:2020 (PDF 2.0) | §15.3, §29.3, ADR-014 | Target the 32000-1 feature set that PDF/UA-1 profiles; do **not** depend on 32000-2 facilities (writing mode, extension-point tagging) |
| PDF/UA-1 | ISO 14289-1:2014 | §2.3, §15.3 | v1 target profile: `/MarkInfo /Marked`, `/StructTreeRoot`, `/Tabs /S`, language tagging, `/DisplayDocTitle`, alternative text for figure content, `/Artifact` for decoration, XMP identification (`pdfuaid:part`); validated with the `ua1` flavour |
| PDF/UA-2 | ISO 14289-2:2024 | §15.3 (watch), §31.2 | Tracked, not targeted: extension-point tagging and observation metadata are outside the pinned validator's mature range; a move requires a new ADR |
| veraPDF rule coverage | validator 1.30.x line; UA-2 checks experimental | §6.5, §14.2, §15.3 | Record validator version + flavour in every report; map rule ids through the checked table (§25.4); never claim more clause coverage than the pinned build implements |
| PDF/A | ISO 19005-4:2020 | §2.3 archival note | Out of scope; combined archival-conformant export is deferred and must not be implied by §15.5 |
| EPUB | EPUB 3.3 — W3C Recommendation 25 May 2023 (EPUB 3.4 in editor's draft; no EPUB 4) | §15.4 | Produce EPUB 3.3; keep the projector version-parametric so a later revision is a matrix change, not a redesign |
| EPUB Accessibility | 1.1 — W3C Recommendation 25 May 2023 (+ Accessibility Techniques 1.1); 1.2 on the Recommendation track | §15.4, §14.6 | Emit required accessibility metadata (`dcterms:conformsTo` to a WCAG version and level, `schema:accessMode`, `accessibilityFeature`, `accessibilityHazard`, plus `accessModeSufficient` and `accessibilitySummary`), and certification properties only when a real certification exists (`a11y:certifiedBy`, `a11y:certifierCredential`, `a11y:certifierReport`) — see §31.3 |
| DPUB-ARIA | 1.1 (roles incl. `doc-noteref`, `doc-footnote`, `doc-pagelist`, `doc-pagebreak`, `doc-pageheader`, `doc-pagefooter`, `doc-pullquote`, `doc-tip`, `doc-notice`, `doc-endnote`) | §7.10, §15.2, §27 | Use the registered subset in §27; a role is only emitted where the matrix says the target supports it |
| WCAG (application UI) | WCAG 2.2 — W3C Recommendation 5 October 2023 | §16.1 | Target 2.2 AA **plus** SC 4.1.1, which 2.2 removed but EN 301 549 v3.2.1 and the Section 508 standard still require — the union is the only posture that satisfies both our own audit and the regimes our outputs serve |
| WCAG (publications) | 2.0/2.1 as referenced *through* EPUB Accessibility and legal standards | §14.2, §31.3 | We state WCAG relationships only via the mechanism a format defines (`conformsTo`); no direct WCAG conformance claim for a PDF |
| EN 301 549 | v3.2.1 (March 2021) is the harmonised edition for the EAA presumption of conformity; v4.1.1 (September 2026) published but not yet harmonised — verify before citing | §2.3, §14.6, §31.3 | Informative for report structure (clause 9 web content, 10 non-web documents, 11 software); a citation of v4.x must be dated and marked as non-harmonised |
| European Accessibility Act | Directive (EU) 2019/882; applied to e-books and reading services from 28 June 2025 | §2.3 | Motivates the export report's evidentiary role. We provide evidence, not compliance opinions |
| US: Section 508 / Title II | 36 CFR 1194 App. A E205.4 (WCAG 2.0 AA for documents and software, with four success criteria void for non-web content); 28 CFR 35.200 (WCAG 2.1 AA) — and DOJ **expressly declined** to adopt PDF/UA or EPUB Accessibility as technical standards | §31.3 | Reinforces the wording rules: a US-facing claim cannot rest on our validator output |
| MathML | MathML Core (Candidate Recommendation track); MathML 4 (Working Draft) | §7.12, §15.2, §15.4, §28.2 | Emit MathML Core-compatible markup; KaTeX (§5.5) is a renderer, not a conformance authority |
| Unicode / bidi | UAX #9 (bidirectional algorithm), UAX #29 (text boundaries); Unicode version pinned by the core's Python runtime | §29.3, §7.6 | The **projector** implements UAX #9 run segmentation; uharfbuzz shapes one directional run at a time and does not resolve direction. The Unicode data version used is recorded in every export report, since normalization results depend on it |
| Timestamps | RFC 3339 profile of ISO 8601 | §26.5, §6.1 | UTC with offset and sub-second precision; ordering by journal sequence, never by clock |
| Media Overlays | SMIL 3.0 (EPUB Media Overlays is a profile of it) | §15.4 | Deferred; extension points only, no audio production in v1 |
| DAISY | DAISY 3 = ANSI/NISO Z39.86-2005; the "DAISY 4" family is DAISY Consortium guidance built on EPUB 3 (with IMIAS), not a NISO standard | §2.2, §2.3, §7.4 | Deferred projections; naming caution — say "DAISY 3 (Z39.86)" or "DAISY-consortium guidance", never "the DAISY 4 standard" |
| Scan metadata | ANSI/NISO Z39.87-2006 (R2017), technical metadata for digital still images | §8.2, §18.4 | Informative field mapping for capture metadata recorded on source artifacts; not a conformance target |
| SBoM | CycloneDX | §5.1, §17.6 | Emitted per release; license scan against §5.8 |
| Font license | SIL OFL 1.1 | §6.6, §31.4 | Rules in §31.4 |
| License scan policy | SPDX identifiers per §5.8 | §17.6 | CI gate; AGPL and non-commercial = build failure |

### 31.2 Currency discipline

Each row carries a verification date and a citation URL, reviewed quarterly by the kernel owner and re-verified before every release (§22.10). A standard advancing in a way that changes a requirement (EPUB 3.4 reaching Recommendation, a mature PDF/UA-2 validator, WCAG 3.0 leaving research draft) opens an ADR — it does not silently change output. Anything marked *unverified* above stays unverified in product copy: **we do not cite a clause number we have not read**, and where this table is uncertain it says so rather than inventing precision.

### 31.3 Conformance-claim language (normative)

Derived from §14.1, §14.2, ADR-013, and the two regulatory postures in §31.1 (PDF/UA and EPUB Accessibility are not adopted as legal technical standards; validator coverage is a subset of a standard's requirements by construction).

**Permitted:** "Validated against the ISO 14289-1:2014 rule set implemented by veraPDF \<version\> (`ua1` flavour): \<n\> failing rules, listed." · "Checked with EPUBCheck \<version\>: \<n\> errors, \<m\> warnings, listed." · "Assessed as \<score\> % remediated under profile \<name\> \<version\> with assessor \<version\>; \<k\> open findings, \<u\> `UNRECOVERABLE` regions." · "Accessibility metadata declares conformance to WCAG 2.1 Level AA via EPUB Accessibility 1.1." · "Remediation completion is an estimate from inspectable findings; it is not a legal certification."

**Forbidden, in UI, report, documentation, and marketing:** unqualified "WCAG compliant", "PDF/UA certified", "Section 508 conformant", "EAA compliant", "accessibility certified"; presenting a validator pass as conformance; presenting a score as a compliance percentage; implying a machine check covers human usability (§14.2's separate category); and describing assistive-text substitution as making a document "screen-reader accessible" without the capability matrix's fallbacks named. If a certification exists, it comes from a certifier and is reported through `a11y:certifiedBy` — never asserted by us.

### 31.4 Font licensing rules (OFL)

For the pinned face in `fonts/MANIFEST.toml` (which records face, version, sha256, OFL license text sha256, the Restricted Font Name list, and glyph coverage):

- **Subsetting is a Modified Version.** An embedded PDF subset may not use a Restricted Font Name, and the subset's internal name must differ from any RFN. The PDF's document is *not* affected by the OFL — clause 5's "does not apply to any document created using the Font Software" is exactly this case.
- **PDF embedding** (including of a subset) requires no accompanying license file; that is the one case the OFL's notice conditions waive.
- **`@font-face` delivery in the HTML projection is distribution, not embedding.** If a webfont ships, the HTML package carries the OFL license text and copyright notice, and the distributed file must not use an RFN (so a WOFF/WOFF2 conversion is a renamed, documented derivative recorded in the manifest).
- **Bundling an unmodified font into an EPUB is bundling, not embedding:** the license and copyright notice must travel with it, and the font may not be relicensed.
- Coverage is a **capability**, not a preference: the pinned face must cover the scripts §8.2's inventory can encounter in supported tiers, and a coverage gap is a `validate`-time finding naming the run (§29.3), never an silent substitution of an unshipped font.

---

## 32. Terminology, governance, and open questions

### 32.1 Terminology

Normative for UI copy, documentation, schemas, and commit messages; a term used in this document carries its glossary sense (§24-R17's control).

| Term | Meaning |
|---|---|
| **committed state** | An atomic, immutable, addressable version of the whole project (§13.1). The unit of history, restore, and export. |
| **snapshot** | The read-only view of a kernel slice handed to a module (§11.3). Not a state; a projection of one. |
| **patch** | A module's staged, precondition-guarded set of field changes (§11.3, §13.2). |
| **artifact** | A content-addressed binary (raster, mask, mesh, heatmap, model output, PDF bytes pre-commit) in `cas/` (§12.3). |
| **surface / space / ledger** | A `PageSurface` is a page's versioned raster chain; a *space* is a named coordinate system; the *ledger* records transforms and inverses between spaces (§7.5). |
| **claim** | A specific field version of an object (a run's characters, a heading's role) — the unit evidence attaches to (§7.7). |
| **evidence node / lineage** | A recorded support for a claim, and the identity (family, revision, prompt, seed, call) that lets independence be judged (§7.7). |
| **finding** | The sole currency between assessment and planning (§14.3); an actionable, severity-tagged, evidence-linked statement. |
| **assessor / profile / score** | The deterministic scorer, its versioned rule-and-weight set, and its output for (state, assessor, profile) (§14.2, §14.4). |
| **projection / projector** | A rendered output format and the component that produces it (§15.1). Never "the master". |
| **override** | A format-specific presentation setting that did not become a semantic fact (§15.1). |
| **module / recipe / task profile** | A subprocess remediation component (§11); a ComfyUI workflow-as-data (§6.2.1); a versioned prompt-plus-parameters unit (§30.5). |
| **role / registry entry / lease** | An abstract capability request (§30.1); a pinned, licensed, probed model (§30.2); a per-run module grant to one role (§30.3). |
| **engine / broker / supervisor** | A GPU-owning or JVM tool process (§4.2); the core's loopback model proxy (§11.3); the core component owning engine lifecycle and VRAM (§6.7). |
| **work unit** | The smallest cancellable, resumable, costed item a module plans (§11.3, §13.3). |
| **journal / recovery checkpoint** | The append-only crash-replay log (§13.4); a module's private mid-run state, never user-visible history (§13.2). |
| **checkpoint (protected)** | A named, pinned state that stays restorable across replaced history (§13.1). |
| **`UNRECOVERABLE` / `UNKNOWN`** | "no answer is possible from this source" vs "no answer attempted yet" (§7.8, §7.8-4). Never interchanged. |
| **prior text layer / prior structure** | Evidence classes for existing digital content — usable, never ground truth (§6.4, §8.3). |
| **semantic digest** | The canonical hash of kernel state used for round-trip and export identity (§21.3). |
| **tier** | A hardware/OS capability level from §4.3, resolved at setup into `capabilities.json` (§22.2). |
| **BLOCKED** | The honest state when a required role cannot run on this machine (§4.3, §14.5) — distinct from failed and from skipped. |

### 32.2 Amendment procedure

**Change classes.** (a) *Editorial* — wording, examples, links: normal review. (b) *Requirement* — adds, tightens, or clarifies a normative statement without touching a decision: normal review, and it must name the sections it affects plus the test that will prove it (§21). (c) *Decision* — alters or contradicts an ADR: requires a new ADR that supersedes the old one (statuses are kept, never edited away), with evidence. (d) *Scope* — changes §2's inputs, outputs, users, or non-goals: requires §0's dependency analysis, a §5 manifest diff, and a §20.3 phase impact note.

**ADR lifecycle:** `proposed → accepted → (deprecated | superseded by ADR-nn)`. An accepted record is immutable in substance; corrections append an amendment section. `Accepted (gated)` records (ADR-006, ADR-014) are re-affirmed or superseded at the Phase-0 gate (§20.2), and the gate's evidence is cited in the record. Every decision carries: owner, date, evidence links (corpus runs, spikes, ADR discussions), and the revisit condition.

**Decision rights** (single-owner, recorded in `docs/`): kernel schema and evidence rules — kernel owner; projection behavior — projector owner with kernel owner approval; role assignments, weights, and licenses — evaluation owner plus a named license reviewer (§30.2); assessor, profiles, and scores — assessment owner; risk register and release acceptance — release owner (§24.3); §31 register currency — kernel owner (§31.2). Where two owners disagree, the decision escalates to a superseding ADR rather than to a code merge.

**Reconciliation of forward references.** The pre-completion draft cited anchors that did not yet exist; those pointers have been retargeted in place, and this table records what each resolves to. The final two rows record wording corrected against the standards pinned in §31.1.

| Cited in | As written | Defined in |
|---|---|---|
| §7.3 | "extensible per §7.14" | §26.1 (taxonomy registration mechanism) |
| §7.2 | "preserved per §11.4 of the product's requirements" | §27.2 (folios, page labels, and markers) |
| §7.9 | "essential vs supplementary placement per §11.4 import rules" | §27.3 |
| §7.9, §9.4 | "§10.2 / §10.3 profile conventions", "glyph gallery §10.2" | §10 (as elaborated by §27.3 for placement conventions and §29.3/§28.2 for glyph-gallery use) |
| §9.3 | "drift alarm, §9.5 of risks in §24" | §24.1 R5 (with R2 for the loop case) |
| §9.7 / §18.4 | "§24-R2" / "§24-R6" | §24.1 R2 / R6, as named |
| §0 | reading map naming §5, §6, §23 | extended by §20–§22 (build and delivery), §24 (risk), §25–§31 (refinements), §32 (governance) |
| §15.3 | "XMP with `pdfuapart` + marked/observations" | corrected in §15.3 to `pdfuaid:part` and `/MarkInfo /Marked`; observation metadata belongs to PDF/UA-2 and is out of scope per §31.1 |
| §15.4 | "`a11:essential`/certification" | corrected in §15.4 to the tokens defined by EPUB Accessibility 1.1 (§31.1) |

### 32.3 Deferred register

Deferral is not silence: each entry states the interface that must not block it, so a later addition is additive (§0's definition of *deferred*).

| Deferred item | Must-not-block requirement | Re-entry path |
|---|---|---|
| DOCX projection | §15.1 contract supports per-format overrides and matrices; kernel already represents notes, tables, and math (§7) | new ADR (§2.3) + §21.3 matrix + an AT/disclaimer rule for preview fidelity |
| DAISY 4 / media overlays / tactile and large-print | §7.4's pronunciation variant, §7.9's skippable/escapable classes, §7.12's tactile-referral field, and §27.1's furniture retention all exist and are populated | new ADR; §15.4's clean extension points; no audio model in §5, so a role ADR is also required (§30.1) |
| EPUB import, PAGE/ALTO/hOCR importers | the semantic model can represent what they carry (§2.2); importers are additive under §19's `app/importers/` | requirement change only (§32.2b) |
| Remote endpoints | the OpenAI-compatible seam stays internal and no client code path exists (§17.2, ADR-005) | ADR-005 superseded, with the lineage problem solved on paper first |
| Multi-user server operation | single-user assumptions are localised: locking (§22.5), one core per port (§4.4), file-handoff collaboration (§2.1) | ADR-002/ADR-009/ADR-018 revisited together |
| History branching and three-way merge | patch preconditions already detect conflicts (§13.2); §7.3's immutable ids make a merge space well-defined | §32.2(d) scope change; §22.6 storage re-costing |
| Containerised module isolation | module boundary is already a subprocess with declared permissions (§11.4) | hardening change, no interface change |
| Second engine tiers (AMD/CPU-only) | capability resolution is data-driven (`capabilities.json`, §22.2) and the planner reads it (§14.5) | §4.3 matrix change + a CI lane (§21.7) |

### 32.4 Decisions not carried forward from the predecessor draft

The earlier requirements draft (retained in-repo as `legacy.md`) is superseded by this document. Its architecture survives; the following items deliberately do **not**, because newer decisions contradict them — recorded here so nobody "restores" them from the old file:

1. **Remote API adapters, secure key storage for providers, and egress indication UIs** (predecessor draft 15.3, 15.8, 16.2) → superseded by ADR-005 and §17.2. The OS-keychain requirement goes with them; `tokens/` (§22.3) holds only per-launch loopback credentials.
2. **A multi-runtime adapter layer** — Transformers, Diffusers, GGUF/llama.cpp, Ollama, LM Studio, SGLang, LoRA management (predecessor draft 15.3–15.5) → superseded by ADR-004, ADR-007, and §0's one-implementation policy. LoRA/adapter plumbing survives only as ComfyUI recipe internals, where the engine owns them.
3. **User-registered models, editable provider lists, per-module model overrides, and fallback chains** (predecessor draft 15.1, 16.2) → superseded by §30.4 and ADR-017: assignments are registry decisions validated by §20.1, and the Settings model browser lists registry entries only (§16.6).
4. **Temporal as a workflow option** (predecessor draft 5.1, 30.7) → rejected by ADR-009, with the idempotency and determinism requirements it stood for kept in §13 and §21.5.
5. **DOCX, DAISY, structured-text, large-print, and Braille-oriented outputs as v1 targets** (predecessor draft 21.1) → deferred per §2.3 and §32.3; the projector contract is their retained obligation.
6. **A guaranteed CPU-only baseline** (predecessor draft 30.21's recommendation) → superseded by §4.3, which places CPU-only hosts out of scope and instead demands honest BLOCKED reporting; the *spirit* survives as §16.6's CPU-compatible preset for the core's own work.
7. **PyMuPDF-adjacent tooling and any AGPL dependency** → forbidden by ADR-006; §6.6's read/write split is the replacement.
8. **Any acceptance of pickle-format weights** (the predecessor draft 15.7 permitted them with approval) → refused outright by §17.3; safetensors or engine-native verified formats only.
9. **Model downloads and management as a user task independent of the registry** (predecessor draft 24) → replaced by §22.2/§16.6: setup installs pinned registry entries, with license and size shown before download.
10. **Illustration-of-scope drift**: legacy's "adapters for everything" breadth is replaced by depth on one path — §20's corpus, §9's loop, and §15.3's writer. Where this document is quieter than its predecessor, it is usually because a decision has since narrowed the surface, not because the requirement was forgotten.

### 32.5 Open questions

Each needs evidence or a named decision, not opinion; the owner is per §32.2.

| ID | Question | Closes when | Owner |
|---|---|---|---|
| OQ-1 | Core license: Apache-2.0 or MIT (§5.8, ADR-010) | before first public commit; patent-grant posture is the deciding factor | release owner |
| OQ-2 | Which OFL face(s) ship, and is one family sufficient for Latin + Greek + Cyrillic + CJK coverage the corpus shows (§6.6, §31.4) | §20.1 corpus inventory plus a coverage matrix and a size decision | kernel owner |
| OQ-3 | Actual `MODELS.toml` defaults per role | Phase-0 bake-off (§20.1), including the license-review block for each weight set | evaluation owner |
| OQ-4 | JPEG 2000 encode path for the §15.3 distribution policy without adding a dependency | a §20.2-style probe of the pinned imaging stack, or the policy drops to JPEG + lossless | kernel owner |
| OQ-5 | Source-information-limit estimator for §18.4/R6: per-page, per-region, or per-corpus-calibrated | method documented in `docs/EVAL.md` with a measured false-refusal rate | assessment owner |
| OQ-6 | Whether `n_eff` (§7.7 R3) is the right strength statistic versus a covariance-aware estimator | a corpus study with the calibration sets (§20.1) | assessment owner |
| OQ-7 | Windows CI coverage depth for the J tier and long-path/file-lock behavior (§4.3, §22.5) | a lane exists or Windows drops to "documented, unsupported" | release owner |
| OQ-8 | Multi-volume and >1000-page works: is one `Document` per volume, and what changes in §7.2 and §18.1 | scale tests (§18.1) plus a §12.1 decision | kernel owner |
| OQ-9 | Hyphenation evidence without a dictionary dependency (§29.1): is §10-derived book lexicon plus prefix/suffix repetition sufficient | measured join precision/recall on the corpus (§20.1) | evaluation owner |
| OQ-10 | EPUB 3.4 and PDF/UA-2 adoption policy, and the validator-maturity threshold for the latter (§31.1, §31.2) | both reach the maturity §31.2 defines, plus a superseding ADR | kernel owner |
| OQ-11 | How much of §16's UI needs to exist in Phase 1 for the foundation criteria to be honestly testable (§22.10) | agreed Phase-1 screen list at Phase-0 exit | release owner |
| OQ-12 | DRM: v1 ships none. Is any DRM-adjacent requirement implied by serving publishers at all (§2.1)? | a written position in `SECURITY.md` and §2.5's non-goals | release owner |

### 32.6 Closing statement

This document specifies one idea implemented many times: that a degraded document can be reconstructed more accurately when pixels, glyphs, words, structure, and book-wide convention are inferred **jointly and adversarially**, and that the result is only trustworthy if every step leaves evidence a reader can audit. Everything else — the CPU-only core, the frozen manifest, the ledger, the linear history, the un-bypassable broker, the advisory validators, the export report — exists to keep that one idea honest under years of use: cheap to correct, impossible to quietly fake, and always explorable to its source. Where the system cannot recover a page, the specification's answer is that the product says so, in the artifact, forever (§7.8, principle 13).
