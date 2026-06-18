# private-perception-node

> **For the implementing agent:** This file is the complete brief. Read it fully before writing code. It defines *what* to build, the *hard constraints* you must not violate, and the *architecture* to follow. This is a **portfolio demo** for the author's AI-demos blog — see §13. When in doubt, prefer the privacy-preserving / honest-physics path. Do not promise capabilities the cheap hardware cannot deliver.

---

## 1. What this is (and what it is NOT)

A **camera-free, edge-only spatial perception node**: it senses presence and motion with a Doppler sensor and builds a crude, low-resolution spatial map with a swept ultrasonic sensor, then an **LLM narrates and answers questions** about the scene — entirely on a Jetson Nano, with **no data ever leaving the device**, and with a **living governance dossier** as a first-class deliverable.

**This is NOT** a camera + computer-vision project, and it does **NOT** reconstruct photographic images. It produces an honest, coarse **occupancy map** (bearing + range + crude size) plus a **motion/speed layer** — like a topographic map that color-codes elevation, except here color codes motion. No imagery of people exists anywhere in the system.

**Thesis of the demo:** *"How do you build an AI perception system that is private, evaluable, and governed **by design** — and prove it with evidence, not claims?"* The differentiator vs. the thousand "camera + YOLO" demos is **measurable responsible AI at the edge**.

---

## 2. Hard constraints (MUST follow — safety/honesty rails, not preferences)

1. **NO CAMERA, NO IMAGES OF PEOPLE.** The system never captures, stores, or reconstructs photographic imagery. The only "image" that may exist is a **rendered occupancy/heatmap plot** of sensor returns (a radar-style PPI plot) — never a photo. Do not add a camera. Do not use generative models to "hallucinate" a realistic image from sensor data (fabricating imagery of people from sensors is an explicit anti-goal and a governance red flag).
2. **EDGE-ONLY / NO CLOUD.** All inference, storage, and evaluation run locally on the Jetson Nano (or, for offline batch eval, on a local companion machine on the same LAN — never a cloud service). No data leaves the local network.
3. **HONEST PHYSICS — NO OVERPROMISING.** The hardware is cheap and low-resolution (§4). Size is a **bucket** (small / medium / person-sized / wall), not a precise measurement. "Person-sized" means *object of roughly person width* — **NOT verified identity, NOT biometrics**. Never claim sub-beam-width resolution, precise dimensions, or human identification.
4. **PERSIST TEXT, NOT RAW SENSITIVE SIGNAL.** Store derived **events** (structured detections + narration + metadata + embeddings). The system is anonymous-by-construction (no biometric data is even collected).
5. **DETERMINISTIC GROUND TRUTH OVERRIDES THE LLM.** The raw sensor numbers (Doppler Hz→m/s, ultrasonic cm) are the source of truth. The LLM only *narrates/interprets*; a deterministic cross-check can flag the LLM when its narrative contradicts the numbers (§6, §7). The model never has unmediated authority.

---

## 3. The user's stack this must exercise (context)

This demo reinforces **AI-200** (agentic AI) and the author's study stack. Target coverage:
MCP · Evals (RAGAS, DeepEval, Promptfoo) · Langfuse · pgvector or Qdrant · Ollama · open models (Llama/Qwen/DeepSeek) · AI Governance (EU AI Act, NIST AI RMF) · multimodal (rendered-map vision, phase 2). Google ADK, Phoenix, openevals/LangSmith, and Azure AI Foundry Evaluations were **considered and dropped** — do not add them.

---

## 4. Hardware (fixed) and what it can actually do

- **Jetson Nano — original 4GB** (Maxwell 128-core, JetPack 4). Constrained. This is why there is **no VLM**: only classic signal processing + a **small text LLM** (see §5) fit. Assume event-driven, not real-time.
- **HB100** microwave Doppler module, 10.525 GHz → **motion + speed** (scalar; no bearing, no position). Needs amplification of its low-frequency Doppler output; speed ∝ Doppler frequency.
- **HC-SR04** ultrasonic distance sensor (40 kHz, ~2 cm–4 m) mounted on a **stepper motor** → swept to produce a polar **range-vs-bearing** map.

**Honest resolution envelope (encode these limits in the code and the README):**
- **Angular resolution ≈ 15°** (HC-SR04 effective beam cone). You may step the motor finer (e.g., 2°), but objects closer than ~15° apart merge. The beam "fattens" objects.
- **Range:** ~2 cm–4 m; soft/angled/small targets reflect poorly.
- **Size = geometric bucket:** consecutive angles returning ≈ same range give an *angular span*; `arc ≈ range × span(rad)` → bucket (small / medium / person-sized / wall). Not centimeters.
- **Sweep time ≈ 5–10 s** (echo timeout + stepper settling per step). Good for near-static scenes; the Doppler covers motion between sweeps.
- **Motion attribution is ambiguous with multiple objects** (HB100 has no bearing). With one object, attribute motion to it; with several, mark it **uncertain** — and feed that uncertainty into the governance cross-check.

---

## 5. Architecture / data flow

```
[HB100 Doppler] --speed/motion--┐
                                ├─► [Fusion: occupancy grid + motion layer]
[HC-SR04 + stepper sweep]───────┘         │  detections: [{bearing,range,size,moving,speed}]
                                          │  (render-able as a color-coded polar map)
                                          ▼
                          [Small text LLM (Ollama: Qwen 1.5B–3B quantized)]
                                narrates / answers   │
                          [Deterministic cross-check: narrative vs Hz/cm] ◄─ anti-hallucination
                                          ▼
                          [Event] → Qdrant or pgvector (narration text + metadata + embeddings)
                          ┌───────────────┼────────────────────┐
                          ▼               ▼                     ▼
                     [MCP server]     [Eval harness]      [GOVERNANCE.md + status panel]
                  get_presence()    Langfuse (traces)     EU AI Act risk classification
                  get_speed()       RAGAS (RAG eval)      + NIST AI RMF mapping
                  scan_space()      DeepEval (faithfulness)   (Measure = evals,
                  query_events()    Promptfoo (model bake-off) Manage = cross-check)
                          │
                          ▼
                  [Conversational agent: plain Ollama loop]
```

### Detection schema (the heart of the data model)
```python
Detection = {
  "bearing_deg": float,     # stepper angle; negative = west/left, 0 = front
  "range_m": float,         # ultrasonic time-of-flight
  "size_bucket": str,       # "small" | "medium" | "person_sized" | "wall"
  "moving": bool,           # from Doppler (attributed; may be uncertain)
  "speed_mps": float | None,
  "motion_uncertain": bool  # true when >1 object and motion can't be attributed
}
```
Example narration target: *"Small static object ~20° west at 1.4 m; a person-sized object straight ahead at 2.1 m, approaching slowly."*

---

## 6. Components

1. **Sensing layer** — Doppler reader (amplified HB100 → speed), ultrasonic sweep driver (stepper + HC-SR04 → range/bearing). Pure, deterministic, cheap.
2. **Fusion** — assemble the `Detection` list per sweep; compute size buckets geometrically; attach the Doppler motion layer; flag `motion_uncertain`.
3. **Renderer** — color-coded **polar occupancy map** (position = bearing+range, color = motion/speed like elevation-color on a topo map, marker size = size bucket). Output is an image artifact for the UI and for the phase-2 multimodal experiment — never a photo.
4. **Narrator (LLM)** — small text model via **Ollama**; input is the numeric `Detection` list (default path). Returns a natural-language scene description + per-claim confidence.
5. **Cross-check (anti-hallucination)** — deterministic rules comparing the narrative against the raw numbers (e.g., LLM says "fast-approaching large object" but Doppler ≈ 0 m/s, or ultrasonic sees nothing < 4 m → **flag inconsistency**). This is both an eval signal and the NIST *Manage* control.
6. **Event store** — Qdrant or pgvector; persist narration text + metadata + embeddings (no raw sensitive signal needed).
7. **MCP server** — exposes the node's senses: `get_presence()`, `get_speed()`, `scan_space()`, `query_events()`, `get_governance_status()`.
8. **Conversational agent** — plain Ollama loop (no ADK) that calls MCP tools to answer natural-language questions over the event history (RAG).

---

## 7. Evaluation strategy (lean set — do not add more tools)

| Job | Tool | Notes |
|---|---|---|
| Tracing / observability | **Langfuse** | every perception + every agent query |
| RAG eval (queries over events) | **RAGAS** | faithfulness, answer-relevancy, context-precision; events give verifiable ground truth |
| Narration faithfulness (LLM-as-judge, local) | **DeepEval** | does the narrative stay faithful to the `Detection` numbers? Powers the anti-hallucination metric |
| Model / prompt bake-off | **Promptfoo** | compare Qwen vs Llama vs DeepSeek (small variants) and prompt variants |

**Ground truth is cheap here:** you physically place objects at known bearings/ranges, so you can score both the deterministic fusion (size/bearing/range accuracy) and the LLM narration against measured reality. Curate a small staged test set (object at known angle/distance, empty room, two objects, moving object).

---

## 8. Governance dossier (the differentiator) — scope: EU AI Act + NIST AI RMF only

Deliver a living `GOVERNANCE.md` + a status panel. **ISO/IEC 42001 is explicitly out of scope (phase 2).**

- **EU AI Act:** honest risk classification. A camera-free, non-identifying presence/spatial sensor collects **no biometric data** and performs **no remote biometric identification** → argue **low/limited risk**, and document the *design decisions that keep it there* (no camera, no identity, no image retention, edge-only). This honesty *is* the exercise.
- **NIST AI RMF:** map the four functions to concrete system parts —
  - **Govern:** the policy + this dossier.
  - **Map:** the documented capabilities/limits envelope (§4).
  - **Measure:** the eval harness (§7).
  - **Manage:** the deterministic cross-check (§6.5) + any guardrails (e.g., refuse to assert identity).

---

## 9. Tech stack

| Layer | Choice |
|---|---|
| Language / runtime | Python 3 (whatever JetPack 4 supports on the Nano; keep deps light) |
| Sensing | GPIO/serial drivers for HB100 (amplified) + HC-SR04 + stepper |
| LLM | **Ollama**, small quantized text model (Qwen 1.5B–3B class); event-driven |
| Agent | plain Ollama loop (no framework) |
| Agent surface | custom **MCP server** |
| Vector store | **Qdrant or pgvector** (pick one; Qdrant is lighter to self-host on-device) |
| Evals | Langfuse + RAGAS + DeepEval + Promptfoo |
| Rendering / UI | matplotlib polar plot for the map; a simple local dashboard (text feed + map) |

---

## 10. Suggested repo structure

```
private-perception-node/
├── CLAUDE.md                 # this file
├── README.md                 # public-facing, blog/portfolio framing
├── GOVERNANCE.md             # living dossier: AI Act + NIST mapping
├── src/
│   ├── sensing/
│   │   ├── doppler.py         # HB100 → speed/motion
│   │   └── ultrasonic.py      # HC-SR04 + stepper sweep → range/bearing
│   ├── fusion.py             # build Detection list + size buckets + motion layer
│   ├── render.py             # color-coded polar occupancy map (NOT a photo)
│   ├── narrator/
│   │   ├── llm.py             # Ollama small-model narration
│   │   └── crosscheck.py      # narrative-vs-numbers anti-hallucination
│   ├── store/                # Qdrant/pgvector events + embeddings
│   ├── mcp_server/           # get_presence/get_speed/scan_space/query_events/...
│   └── agent/                # plain Ollama conversational loop (RAG over events)
├── eval/                     # Langfuse + RAGAS + DeepEval + Promptfoo; staged test set
├── ui/                       # text feed + rendered map
└── README assets/            # demo recordings, hero map image
```

---

## 11. Implementation order (milestones)

1. **Sensing + fusion:** drivers for HB100, HC-SR04+stepper → a clean `Detection` list per sweep. Verify against physically-placed objects.
2. **Renderer:** color-coded polar occupancy map from the `Detection` list.
3. **Narrator:** Ollama small model turns the numeric list into a scene description (default numeric path).
4. **Cross-check:** deterministic narrative-vs-numbers consistency flagging.
5. **Event store + MCP server + agent:** persist events, expose tools, answer NL queries (RAG).
6. **Eval harness:** Langfuse traces + RAGAS (RAG) + DeepEval (faithfulness) + Promptfoo (model bake-off); build the staged ground-truth set.
7. **Governance dossier:** `GOVERNANCE.md` (AI Act + NIST) + status panel wired to live metrics.
8. **(Phase 2) Multimodal experiment:** feed the *rendered map image* to a small vision model and compare "numeric input vs. rendered-image input" as an eval.

Ship 1–4 (sense → map → narrate → cross-check) before anything fancy. Every milestone should be demoable on its own.

---

## 12. Honest constraints (repeat in README, don't hide them)

- 4GB Nano → small model, event-driven, slow-ish. That's fine for the thesis.
- ~15° angular resolution; size is a bucket; ~5–10 s per sweep; multi-object motion attribution is uncertain. These limits are *part of the story* (honest engineering), not bugs to paper over.
- Privacy-by-design has a cost: not retaining raw signal limits debugging — log enough structured metadata to debug without collecting anything sensitive.

---

## 13. Portfolio integration

This project will be integrated into the **demos portfolio of the author's personal website / AI-demos blog**, as the privacy/responsible-AI counterpart to the sibling `agentic-trading-lab` project.

- `README.md` must be **public-facing**: state the honest thesis, show the hero **color-coded occupancy map**, and the live **text-narration feed**. Link a recording.
- Narrative angle: *"I gave an agent spatial senses at the edge — no camera, nothing leaves the device — and built the evidence that it's private, evaluated, and governed."*
- The **governance dossier + eval charts** are the visual payload of the blog post.
- Favor clean, well-commented, honest code; it will be read by blog visitors, not just run. **Never oversell the hardware.**
```
