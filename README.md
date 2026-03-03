# Hi, I'm Ha Young (Aaron) Kim

> [🇰🇷 한국어 버전](README_KR.md)

**AI/ML Engineer** 

My recent work spans **medical imaging data pipelines**, **model quantization across 5 frameworks**, and **privacy engineering for healthcare AI**. 

---

## What I Work On

Most of my repositories are private (company/client IP). 
I'll describe **what makes each project technically interesting** — the hard problems, the design trade-offs, and the engineering decisions.

**Want to see a live demo?** I'm happy to walk through any of these projects — reach out via [aaronkim777@gmail.com](mailto:aaronkim777@gmail.com) or [https://www.linkedin.com/in/hyk1/](https://www.linkedin.com/in/hyk1/).

---

## Featured Projects

### 1. QLRO-Features — Medical Imaging DataOps Pipeline

**84.8K LOC** · **40.3K LOC tests** · 7 pipeline steps · Jan–Feb 2026

Transforms raw DICOM files into AI-ready datasets with HIPAA Safe Harbor adherence and NEMA Part 15 conformance. The interesting parts aren't the pipeline itself — it's the engineering underneath.

#### Technical deep-dives

**Geometric slice ordering with cross-product normals.** DICOM series need spatial ordering for 3D reconstruction, but the standard leaves this underspecified. I compute slice positions by extracting ImagePositionPatient (IPP) and ImageOrientationPatient (IOP), derive the slice normal via cross-product of the row/column direction cosines, then project each slice's IPP onto this normal vector. The sorted projections give physically correct Z-ordering even when InstanceNumber is wrong or missing — which happens more often than you'd expect.

**52-token canonical series labeling.** Series descriptions in DICOM are free-text chaos ("ax t1 post gad fs", "T1W_AX+C FAT SAT", etc.). I built a rule-based classifier using a custom YAML DSL with strategy/composite patterns — the engine evaluates 52 anatomical/protocol/sequence tokens against regex patterns, producing structured labels like `CT|CHEST|AXIAL|WITH_CONTRAST|SOFT_TISSUE` with 3-tier confidence scoring (HIGH/MEDIUM/LOW) and structured evidence chains. The YAML DSL supports AND/OR/NOT logic, priority ordering, and fallback chains.

**800× Presidio optimization via monkey-patching.** Presidio's `ImageRedactorEngine` calls `most_common_pixel_value()` per OCR bounding box to determine background fill color — running a full-image histogram analysis each time. I profiled this to ~700ms per image, replaced it with a single-call cached result using `collections.Counter` on the pixel array, and monkey-patched it into the Presidio class. Down to 0.9ms. The key insight was that background color doesn't change between bounding boxes in the same image.

**PaddleOCR adapter with fd-level C extension suppression.** PaddleOCR's C++ backend writes directly to stdout/stderr via native file descriptors, bypassing Python's `sys.stdout`. Suppressing this required `os.dup2()` on the actual file descriptors (fd 1/2) — not just redirecting Python streams. The adapter (698 LOC) also handles thread-safe lazy initialization with double-checked locking, uint16→uint8 DICOM pixel normalization, and dynamic model path resolution for air-gapped environments.

**Defense-in-depth PHI architecture.** Rather than trusting any single de-identification method, the pipeline layers 7 approaches: (1) all-text OCR redaction of burned-in pixels, (2) post-hoc non-PHI classification of redacted text, (3) injection of non-PHI tokens into `ImageComments` for downstream NER, (4) Presidio NER with 6 custom medical date recognizers scrubbing all free-text metadata tags, (5) SHA-256 deterministic UID replacement, (6) `.secrets/` directory isolation (never exported), (7) append-only JSONL audit trail with hashed identifiers.

**SHA-256 key-derived date shifting.** To preserve longitudinal temporal relationships within a study while removing actual dates, I derive a deterministic day-offset from `SHA256(patient_id + salt)`, apply it consistently across DA/TM/DT value representations, and ensure all instances within the same study shift by the same amount. This means you can still compute time-between-scans for AI training without ever exposing real dates.

---

### 2. Q-Hub — Multi-Sector AI Quantization & Governance Platform

**25K+ LOC** · 3 FastAPI services · 21 endpoints · Oct–Dec 2025

Backend for AI inference comparison across Healthcare (pneumothorax), Finance (fraud), and Manufacturing (defect detection), with FP32/INT8 A/B testing and IBM WatsonX governance integration.

#### Technical deep-dives

**ONNX graph surgery: Swish→ReLU6 replacement.** The EfficientNet-B0 backbone uses Swish (SiLU) activations that ONNX Runtime's INT8 quantizer handles poorly (quantization noise amplification in the sigmoid branch). I wrote a graph-level pass that pattern-matches `Mul(x, Sigmoid(x))` subgraphs, replaces them with `Clip(Relu(x), 0, 6)`, and reconnects edges — operating directly on the ONNX protobuf, not through any framework's API. This preserved accuracy while making the model quantization-friendly.

**Custom QAT fusion: Conv+BN+ReLU6.** PyTorch's built-in QAT fusion only supports Conv+BN+ReLU. Since I replaced Swish with ReLU6, I had to write a custom `ConvBnReLU6` fused module that correctly propagates the clamp through fake-quantize observers during training. Without this, the quantization-aware training would learn the wrong scale/zero-point for the activation range.

**Three-phase coordinator pattern for inference workers.** Running FP32 and INT8 models simultaneously for A/B comparison sounds simple until you account for: different inference latencies, shared GPU memory pressure, and the need to attribute individual predictions to the correct model. I used a producer→coordinator→consumer pattern with `multiprocessing.Process` for true CPU isolation, `psutil.cpu_affinity()` pinning, and IPC via `multiprocessing.Queue` — passing image file paths rather than serialized arrays (the path-not-payload pattern cut IPC overhead dramatically since serializing a batch of images through pickle was the bottleneck).

**timm's fused BatchNormAct2d decomposition.** The `timm` library fuses BatchNorm and activation into a single `BatchNormAct2d` module for inference speed. But ONNX export and quantization tooling expects them separated. I wrote a decomposition pass that walks the module tree, replaces each `BatchNormAct2d` with an explicit `BatchNorm2d` → `act_fn` sequence, and copies the learned parameters — a problem that doesn't show up until you try to export a timm model to ONNX for quantization.

**Decimal digit decomposition for finance features.** Credit card transaction amounts carry signal in their digit patterns (e.g., amounts ending in .00 vs .99 differ in fraud probability). Instead of just using the raw amount, I decompose each transaction value into per-digit features using modular arithmetic — `digit_i = (amount // 10^i) % 10` — giving the model explicit access to digit-level patterns without relying on the network to learn this decomposition from raw floats.

---

### 3. CellViT++ — Head & Neck Cancer Cell Classification

**1.2K+ LOC** · 14 W&B runs · Aug–Sep 2025

Evaluation pipeline and INT8 quantization for a 632M-parameter ViT-SAM-H cell segmentation model on private head & neck cancer pathology data.

#### Technical deep-dives

**SAM-H fused QKV decomposition for bitsandbytes.** The SAM-H vision transformer uses fused QKV projections — a single `nn.Linear(1280, 3840)` that outputs concatenated Q, K, V. bitsandbytes' `LLM.int8()` mixed-precision quantization requires knowing individual matrix dimensions to decide which rows to keep in FP16 (outlier features). I split each fused projection into three separate `nn.Linear(1280, 1280)` modules, copied the weight slices, and added 4D→3D tensor reshaping to match the expected attention input format — across all 32 transformer blocks. Without this, bitsandbytes quantized the fused matrix as a single unit and couldn't isolate outlier dimensions.

**Zero-shot domain transfer evaluation.** The model was pre-trained on PanNuke (19 tissue types, ~200K nuclei) but needed to classify head & neck cancer cells it had never seen. I built an evaluation harness that loads GeoJSON cell predictions from CellViT, maps the model's 6-class taxonomy to our 4-class schema, and computes per-class F1/recall/precision against pathologist annotations. The result — F1=0.77, Recall=91.4% — was strong enough to skip fine-tuning for the initial deployment.

**Cross-dataset label harmonization.** Six histopathology datasets (PanNuke, CoNSeP, MoNuSAC, CryoNuSeg, OCELOT, TIGER) each have different cell type taxonomies. I unified them under a 4-class schema (Neoplastic, Non-neoplastic, Inflammatory, Dead) using manually verified mapping tables, which was necessary both for training consistency and for making the evaluation metrics comparable across datasets.

---

## Contact

- aaronkim777@gmail.com
- [LinkedIn](https://www.linkedin.com/in/hyk1/)
- [arXiv Paper](https://arxiv.org/abs/2407.12514) · [IEEE Paper](https://ieeexplore.ieee.org/document/10352142)

---

*All project repositories are private (company/client IP). I maintain detailed technical reports and architecture docs for each — happy to walk through specifics in conversation.*
