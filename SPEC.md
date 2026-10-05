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
| **uharfbuzz** (HarfBuzz) | MIT | text shaping for content-stream runs (script-aware positioning, bidi) |
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

- Restoration/inpaint weights for ComfyUI recipes (candidates for the Phase-0 bake-off, license-verified: DocRes [MIT], Qwen-Image-Edit [Apache-2.0], FireRed-Image-Edit [Apache-2.0], AnyText + page-font embedding workflow [Apache-2.0] for glyph repair; FLUX-Kontext-class models excluded by their non-commercial licenses — ADR-006 corollary).
- Recognition/critic/repair weights for vLLM (candidates: Unlimited-OCR [MIT], DeepSeek-OCR / -2 [MIT], olmOCR-2 [Apache-2.0], dots.ocr [MIT], PaddleOCR-VL [license unstated → gated pending review], Qwen3-VL [Apache-2.0, variants gated], plus a mid-size instruct LLM for text repair).
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
| `GET /system_stats` | readiness, version pin enforcement | reject mismatched commit → `VERSION_MISMATCH` |
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

**veraPDF.** Invocation: `java -jar <verapdf-core.jar> --format json --flavour ua1 … <file>` against exported PDFs (flavour pinned per PDF projection profile). The core parses the JSON report into Findings (§14.3); exit-code mapping: `0` valid · `1` invalid (expected, produces findings) · `2` usage/encrypted input · `≥3` tool error → `EngineError`. The JAR is installed by setup into a managed tool directory with hash verification.

**EPUBCheck.** Invocation: `java -jar epubcheck.jar --out <json> --mode epub <file.epub>`; findings mapped likewise. Both tools are treated as *advisory machine checks*: passing is necessary, never sufficient (§14.2).

### 6.6 PDF object engines (in-process, boundary-by-role)

- **pypdfium2**: input rendering only. Given a source PDF page index → raster at requested DPI into staging. It must never be used to write, re-save, or "round-trip" a PDF.
- **pikepdf**: the only writer of PDF bytes. Owns: object model, `/StructTreeRoot` construction, content-stream insertion (`BDC`/`BMC`/`EMC` marked-content operators with stable `/MCID`s), invisible text layer objects, resources (fonts via embedded streams), outlines, names tree, XMP metadata, document ID policy (deterministic for reproducible builds; randomized on request).
- **fontTools + uharfbuzz**: prepare the text layer — shape runs (script/bidi aware), subset the shipped open font (OFL-licensed; exact face pinned in `fonts/MANIFEST.toml`), build the ToUnicode CMap. The core composes the content-stream text operators itself (`BT … TJ … ET`) from shaped runs; no third-party "PDF writer" layer sits between kernel and bytes.
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

A *logical resource* (EPUB spine item, DAISY unit) is derived at projection time from sections/pages; no parallel page-less document tree exists in the kernel. Original pagination is preserved per §11.4 of the product's requirements: printed folio ≠ digital index, both stored.

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

Object taxonomy v1 (extensible per §7.14): `Heading{level}`, `Paragraph`, `List/Listitem`, `Figure`, `Illustration`, `Photograph`, `Table`, `TableRow/TableCell/HeaderCell`, `Equation`, `Chart`, `CodeBlock`, `Caption`, `SideNote (sidebar/callout/inset/feature box)`, `PullQuote`, `Footnote/Endnote` + `NoteMarker`, `CrossReference`, `PageFurniture (artifact): RunningHead/RunningFoot/PageNumber`, `DropCap`, `Quotation`, `IndexEntry`, `UnknownRegion` (the honest catch-all — every `UnknownRegion` at export time is a finding).

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

Reading order is a first-class directed graph, not box sorting. It must represent: the default linear path; column and page continuations; optional/supplementary branches (sidebars — essential vs supplementary placement per §11.4 import rules below and §10.3 profile conventions); note references and *return edges*; figure/table associations and caption binding; skippable and escapable structure classes; explicitly-marked duplicate presentations (pull quotes spoken once); artifacts excluded entirely. Validation constraints (deterministic, run on every commit): every meaningful visible region appears in the default path unless classified artifact/duplicate/branch; no region appears twice on the default path without a duplicate-presentation link; references have targets and return targets; heading hierarchy coherent (single logical H1 per chapter unless profile says otherwise); table ownership and spans non-conflicting; `UnknownRegion`s surfaced.

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

Independent checks per commit batch: visual-support check (does the claim's box actually contain the claim's glyphs — cross-family), contextual plausibility (repair LLM as critic only when different lineage from producer, per R2), typographic consistency (baseline/x-height/advance checks against BookProfile), equation syntax parse (LaTeX→MathML validate; failed parse = finding, never silent fix), table consistency (span closure), reference integrity (targets/returns resolve), duplicate/missing content probes, and *degradation-honesty*: is a repaired region suspiciously smoother than its neighbors (drift alarm, §9.5 of risks in §24). Critics emit findings; they do not edit.

### 9.4 Targeted repair requests

A repair request bundles: region polygon + space id, diagnosed problem, competing readings (max hypotheses param, default 2), surrounding context window (pixels + text), learned typeface reference (BookProfile glyph gallery, §10.2), invariance constraints (neighbors unmodified outside mask dilation), and a resource budget. The ComfyUI glyph recipe must be *conditional on text* only through this request object (so R4 recording is mechanical). **Marginalia rule:** handwritten content must be classified first — publisher content / reader annotation / instructor annotation / historical annotation / damage — with a *separate layer retained by default*; removal is an explicit, per-class user-approved action, never the automatic default (annotations may be the only record of a previous owner; also PII, §17.5).

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

Page content model: each exported page = (1) cleaned image surface (per selected quality/size policy: lossless for accessibility-first exports, configurable JPEG2000/JPEG quality for distribution; the *source* stays lossless in CAS), (2) invisible text layer generated from current alignment (§7.6): shaped runs (uharfbuzz) → `BT/TJ/ET` operators, font = subset (fontTools) of the pinned OFL face, `Tr 3` render mode, correct ToUnicode, (3) marked-content structure: the projector emits `/StructElem` trees whose content lists anchor `BDC … <</MCID n>> EMC` spans *around the actual operators* for that element — the kernel's object ids ↔ MCID table is written into the project state (export audit + re-mapping requirement), so structure is never "attached after the fact." Artifacts marked `/Artifact` (furniture, decorative rules). Notes as `Reference`+`Note` with destinations and `/R` back-links; tables with TH/td spans and `THead/TFoot`; figures `Figure` with `/Alt` (+`/LongDescAlt` object or `/E` where appropriate); equations `Formula` with `/Alt` + linearized `/ActualText`; `/Lang` per element where it varies; document catalog: `/MarkInfo /Marked true`, `/Lang`, `/StructTreeRoot`, `/Tabs /S` (structure order), outlines, page labels for printed folios, XMP with `pdfuapart` + marked/observations. Validation loop: veraPDF (`ua1` flavour) findings must be zero-blocking at export time *when the profile requires it*; failures are reported (never silent), and regression suites assert byte-level structure expectations on golden pages. **Feasibility gate:** the Phase-0 walking skeleton (§20.2) is explicitly the experiment that confirms this section is implementable with the §5 stack (pypdfium2 read-only + pikepdf write-only); if it is not, ADR-006/ADR-014 reopen *before* app investment, and the finding is recorded with evidence. Deferred-but-architecturally-allowed: tagged 3D, forms, embedded media.

### 15.4 EPUB 3 projector

From HTML projection: spine/landmarks/nav (TOC + page-list), `properties: mathml`, accessibility metadata (`a11:essential`/certification per EPUB-A11y 1.1), media-overlay hooks left as clean extension points (DAISY 4 audio-sync is post-v1), fixed-layout *not* in v1 (reflow only; the visual-fidelity readers are served by the PDF projection — documented as a projection-family decision, §3-principle 6). EPUBCheck gates (§6.5).

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
