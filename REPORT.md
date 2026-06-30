# Aletheia — Technical Report
### ADTC 2026 Submission · Healthcare / Medical Track

---

## 1. Problem

District hospitals and rural health centres in sub-Saharan Africa handle a broad and severe case mix — meningitis, cerebral malaria, TB, obstetric emergencies, snakebite — often with a single clinician on duty, no specialist support, and unreliable or no internet connectivity. Clinical decision support tools that exist today assume a cloud connection and a general-purpose workstation. They are unusable at the point of care in these settings.

The result is that clinicians must reason through complex, overlapping differentials entirely from memory, with no structured second opinion available. Diagnostic delays and missed investigations are a direct consequence.

**Target user:** A clinician — clinical officer, nurse, or medical officer — working alone at a district hospital or health centre in Uganda or a similar East/Central African setting. They have a basic laptop and no reliable internet. They need structured reasoning support, not a remote consultation.

**What Aletheia does not claim:** It does not diagnose. It does not prescribe. It supports the clinician's reasoning through a three-stage pipeline that ends with an advisory output explicitly framed as the clinician's decision to make.

---

## 2. Design Decisions

### Pipeline structure

Aletheia enforces a three-stage clinical reasoning flow, in order:

1. **Stage 1 — Initial assessment:** The model receives symptoms and patient demographics and returns a tentative differential plus 3–5 targeted follow-up questions. Tests are deliberately withheld at this stage — the clinician must gather more history first.

2. **Stage 2 — Investigation recommendations:** After the clinician answers the follow-up questions, the model returns a prioritised list of investigations as its primary output, with a working differential as supporting context only. The model is explicitly instructed not to state a confirmed diagnosis at this stage.

3. **Stage 3 — Clinical advisory:** After the clinician enters real investigation results, the model returns a management advisory — options for consideration, not orders. The system prompt and output schema both require the model to include an advisory note explicitly stating that the treating clinician retains all decision authority.

This structure reflects actual clinical reasoning protocol and prevents the model from short-circuiting to a diagnosis before sufficient evidence is gathered. Stage ordering is enforced in both interfaces: the web UI disables later-stage buttons until earlier stages succeed, and the terminal CLI requires non-empty input at each gate.

### Model choice

The model is a GGUF-quantized language model selected for its performance on structured JSON output and medical reasoning tasks within the constraints of a CPU-only district hospital laptop. The Q4_K_M quantization was chosen as the balance point between output quality and memory footprint — fitting within 8 GB RAM while maintaining coherent, schema-valid JSON responses.

**Alternatives evaluated:**
- Q5_K_M: better quality but pushes closer to the 8 GB RAM ceiling on context sizes above 1024 tokens; rejected for margin of safety.
- Q3_K_M: fits more comfortably in RAM but shows more frequent JSON schema violations on structured output tasks; rejected.
- Q8_0: high quality but exceeds the 8 GB RAM profile for models above ~3B parameters; rejected.

### Prompt engineering

All three pipeline stages use a `### System / ### Instruction / ### Input / ### Response` format that mirrors the instruction-tuning format used by most modern fine-tuned GGUF models. Each instruction is written to:
- Specify the exact JSON output schema with field names and types
- Explicitly constrain what the model should and should not output at each stage (e.g. "do not recommend tests yet" in Stage 1; "do not state a confirmed diagnosis" in Stage 2; "do not issue a treatment order" in Stage 3)
- Require the model to produce an advisory note that names the clinician's decision authority in Stage 3

### Offline-first architecture

The system calls `llama-cli` (the llama.cpp binary) as an external subprocess. There are no Python model-loading libraries, no network calls during inference, and no external API dependencies. Once the model file and binary are on disk, the system runs with no internet connection required.

### Interfaces

Three interfaces share a single inference layer (`inference/aletheia.py`):
- **Web UI** (`aletheia/app.py`) — Gradio-based, enforces stage ordering via disabled buttons
- **Terminal CLI** (`cli.py`) — sequential, requires non-empty input at each stage gate
- **Single-stage CLI** (`run.py`) — for scripting and profiler integration; accepts `--stage` and `--extra` arguments

---

## 3. Constraints

### Hardware

Target hardware: a consumer laptop with 4 CPU cores and 8 GB RAM, integrated graphics only. No GPU acceleration is used (`-ngl 0` is passed to `llama-cli`). The system must remain within the 8 GB RAM profile during inference, including OS overhead.

Context size is set to 1024 tokens. At this context length, the Q4_K_M quantization keeps peak RSS well within the profiler's 7 GB scoring threshold for most models in the 3–8B parameter range.

### Connectivity

Zero internet access during inference. The download script (`download_model.sh`) fetches weights once before use. After that, the system is fully air-gapped.

### Data and training constraints

Aletheia is designed for conditions prevalent in East and Central Africa: malaria, meningitis, TB, typhoid, obstetric emergencies, snakebite, severe acute malnutrition, and others that are underrepresented in models trained primarily on Western medical literature. The system prompt explicitly anchors the model to this context ("district hospitals and health centres in sub-Saharan Africa") and instructs it to prioritise conditions prevalent in the region.

### Regulatory framing

Clinical AI in Uganda operates without a formal regulatory framework for AI-based decision support as of 2026. Aletheia is presented as a research prototype, not a licensed medical device. Every output carries explicit advisory framing. This is a design constraint as much as a legal one — the system must not produce output that a clinician could mistake for a final diagnosis or treatment order.

---

## 4. Benchmarks

*Run `bash download_model.sh` then `adtc-profiler run --submission . --mode participant --output submission.json --skip-accuracy` to generate your local benchmark report before submitting. The numbers below are from the development machine used during testing.*

| Metric | Value |
|---|---|
| Development machine | TODO: CPU model, RAM, OS |
| Inference runtime | llama.cpp (CPU only, `-ngl 0`) |
| Context size | 1024 tokens |
| Threads | 4 |
| Temperature | 0.1 |
| Quantization | GGUF Q4_K_M |
| Peak RSS (Stage 1 prompt) | TODO: from `submission.json` |
| Steady-state RSS | TODO: from `submission.json` |
| Tokens per second (generation) | TODO: from `submission.json` |
| First token latency | TODO: from `submission.json` |
| Stage 1 wall-clock time | TODO: seconds |
| Stage 3 wall-clock time | TODO: seconds |

**How to fill in the benchmark rows above:**

```bash
# 1. Download model weights
bash download_model.sh

# 2. Run profiler in participant mode
pip install "git+https://github.com/Africa-Deep-Tech-Foundation/adtc-profiler.git"
adtc-profiler run --submission . --mode participant --output submission.json --skip-accuracy

# 3. Read the numbers
cat submission.json
```

Paste the values from `submission.json` into the table before submitting.

---

## 5. Ethical Considerations

Aletheia is a decision support tool, not an autonomous clinical actor. Several design choices reflect this:

- The pipeline cannot be short-circuited to a final output without passing through all prior stages.
- Stage 3 output uses the word "advisory" in both the section header and a required JSON field (`clinical_advisory_note`) that instructs the model to reiterate the clinician's decision authority.
- The system is framed as a "second opinion" and "structured reasoning support," not as a replacement for clinical training or specialist consultation.
- No patient data is stored, transmitted, or logged by the system.

The African Deep Tech Challenge's vision of locally-run, low-resource-appropriate AI is directly aligned with Aletheia's design: a tool that is useful precisely because it works in the places where cloud-dependent tools fail.
