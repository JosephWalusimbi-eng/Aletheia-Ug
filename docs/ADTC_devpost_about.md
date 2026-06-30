## Inspiration

At Soroti Regional Referral Hospital in Eastern Uganda, a single clinical 
officer may see 80 to 120 patients in a single day. That is fewer than five 
minutes per patient to take a history, examine, diagnose, and decide on 
management. Under that kind of cognitive load, critical presentations get 
missed, not because the clinician is incompetent, but because they are 
overwhelmed.

Uganda has a physician-to-patient ratio of approximately 1:25,000. The tools 
that exist to support clinical decision-making require cloud servers, fast 
internet, and expensive hardware. None of these are reliably available at 
the point of care in rural Africa. The clinicians who need AI assistance the 
most are the ones with the least access to it.

Aletheia was built to close that gap. The name comes from the ancient Greek 
ἀλήθεια - meaning truth or disclosure. The revealing of what is hidden. 
That is exactly what diagnosis is.

## What it does

Aletheia is an offline-first clinical decision support system. A clinical 
officer types in a patient's symptoms and duration. Aletheia returns:
- **Ranked differential diagnoses** with probability estimates and severity 
  ratings, prioritised for African disease epidemiology
- **Priority investigations** available at district hospital level
- **Clinical rationale** explaining the reasoning behind each diagnosis
- **Red flags** that require immediate escalation
- **Follow-up questions** to narrow the differential
- **Immediate management hints** appropriate for resource-limited settings

It covers 50 disease conditions with elevated prevalence across sub-Saharan 
Africa; including cerebral malaria, bacterial meningitis, eclampsia, 
postpartum haemorrhage, severe acute malnutrition, neonatal sepsis, 
snake envenomation, visceral leishmaniasis, and tuberculosis.
It runs entirely on an Intel Core i5, 8 GB DDR4 laptop/computer
Ubuntu 22.04; with no internet connection, no GPU, and no cloud 
dependency. Peak RAM usage is approximately 3,630 MB.

## How we built it

**Base model:** Qwen2.5-3B-Instruct was selected for its strong 
instruction-following performance at a parameter count that fits within the 
ADTC memory budget after quantisation.

**Fine-tuning:** We applied QLoRA (r=32, α=64) across all 7 linear 
projection layers, training 59,867,136 parameters (1.94% of total). 
Training ran on Google Colab Pro (NVIDIA A100-SXM4-80GB) for 1.92 hours 
across 3 epochs.

**Dataset:** 27,000 clinical reasoning samples across 3 sources:
- 18,000 Africa-weighted synthetic samples (50 conditions, 8 reasoning types)
- 6,000 MedQA-USMLE filtered questions
- 6,000 MedMCQA filtered questions

**Deployment:** The merged model was converted to GGUF format and quantised 
to Q4_K_M (1.80 GB) using llama.cpp's two-step pipeline: F16 conversion 
followed by llama-quantize. The inference engine is llama.cpp compiled for 
CPU-only operation.

**Interface:** Two interfaces are provided. A Python terminal chatbot using 
the Rich library for clean clinical formatting, with a single-query CLI 
mode for scripting and integration; and a Gradio-based web UI for clinical 
officers who prefer a browser-based interaction, running entirely on 
localhost with no internet dependency.

## Challenges we ran into

**MedMCQA path bug:** A file path error during dataset construction 
excluded 6,000 MedMCQA samples from the first training run. We identified 
this, corrected the pipeline, and retrained on the full 27,000-sample 
dataset. The corrected run improved Top-1 accuracy from 70% to 80% and 
Top-3 accuracy from 90% to 100%.

**GGUF conversion flags:** The newer llama.cpp removed the `--outtype 
q4_k_s` flag. We solved this by switching to a two-step process: convert 
to F16 GGUF first, then use `llama-quantize` for compression - a more 
robust approach that separates conversion from quantisation.

**Training loss vs accuracy tradeoff:** The second training run showed 
higher terminal loss (0.52 vs 0.31) despite better accuracy. This reflects 
the increased task diversity from MedMCQA, which exposes the model to 
broader question styles. Loss alone is an incomplete proxy for clinical 
utility - accuracy is what matters.

**CPU inference latency:** llama.cpp on CPU is slower than GPU inference. 
On the ADTC target hardware, inference takes 2–4 minutes per query — 
acceptable for clinical use where a few minutes of thinking is normal, 
but something we are optimising.

**Early-stage multilingual fine-tuning:** We began extending Aletheia to 
Kiswahili through continued fine-tuning from the existing LoRA adapter. 
Initial evaluation metrics (ROUGE, BERTScore, JSON validity) looked strong, 
but manual spot-checks against held-out clinical cases revealed the model 
was converging toward a small number of frequent diagnoses rather than 
discriminating correctly across the full condition set. This is a known 
failure mode in early-stage low-resource fine-tuning, and it taught us that 
aggregate text-similarity metrics are not sufficient evidence of clinical 
correctness — only direct case-by-case verification is. We are continuing 
this work, but English is the validated language for this submission.

## Accomplishments that we're proud of

**100% Top-3 accuracy** - the correct diagnosis appears in Aletheia's top 
3 suggestions for every single test case. In clinical practice, this means 
a clinician reviewing three ranked options will almost never miss the 
correct diagnosis.

**1.80 GB deployment** - a 3-billion parameter clinical reasoning model 
compressed to under 2 GB without meaningful quality loss. It fits on a 
USB drive.

**3,630 MB peak RAM** - 3,538 MB below the ADTC ceiling with over 3.5 GB 
to spare. This is not a tight squeeze - it is a comfortable margin that 
leaves room for the operating system, other applications, and future model 
improvements.

**BERTScore-F1 of 0.909** - the model's clinical reasoning text is 
semantically very close to expert reference answers. It is not just naming 
the right diagnosis; it is explaining the right reasoning.

**50 African clinical conditions** including conditions like visceral 
leishmaniasis, Buruli ulcer, and schistosomiasis that are almost never 
represented in Western medical AI benchmarks

## What we learned

The biggest lesson was that **loss is not the metric that matters for 
clinical AI.** A model trained on more diverse data had higher loss but 
better clinical accuracy. Evaluating AI in healthcare requires 
domain-appropriate metrics - Top-k accuracy, BERTScore, and calibration - not just training loss.

We also learned that **dataset quality matters more than dataset size.** 
The Africa-weighted synthetic dataset with careful clinical case design 
contributed more to performance than raw MCQ volume. A small, well-designed 
dataset beats a large, unfocused one.

We learned that **the deployment pipeline is as important as the 
model.** Getting from a trained model to something a clinical officer can 
actually run on an offline Ubuntu laptop required as much engineering 
effort as the training itself - quantisation, compilation, inference 
wrapping, and installation scripting.

Finally, our early Kiswahili experiments taught us that **automated 
evaluation metrics can be quietly misleading in low-resource language 
settings.** A model can score well on ROUGE and BERTScore while still 
defaulting to a narrow set of confident-sounding but incorrect diagnoses. 
For clinical AI, manual case-by-case verification against domain experts 
is not optional, it is the actual test that matters.

## What's next for Aletheia: Offline Clinical Reasoning Engine

**Prospective clinical validation** - we are planning a validation study 
across multiple health facilities in Eastern Uganda involving 50+ clinical 
officers and 500+ patient cases, subject to IRB approval from Soroti 
University and the Uganda National Council for Science and Technology.

**Expanded condition coverage** - growing from 50 to 100+ clinical 
conditions, with deeper coverage of obstetric emergencies, paediatric 
dosing, trauma triage, and mental health.

**Language support** - Kiswahili and Ateso interfaces are in active 
development. We have identified a mode-collapse issue in our first 
Kiswahili fine-tuning attempt and are working with our clinical co-authors 
to rebuild the training data with greater per-condition lexical diversity 
before the next training run. English remains the validated submission 
language for ADTC 2026.

**Desktop GUI** - a lightweight graphical interface making Aletheia 
accessible to non-technical clinical users without a terminal.

**Regulatory pathway** - submission to the Uganda National Drug Authority 
(NDA) under the software-as-a-medical-device framework.

**Commercialisation** - through Arapai Technologies International Limited, 
making Aletheia available to district hospitals and health centres across 
Uganda and the wider East African region.
