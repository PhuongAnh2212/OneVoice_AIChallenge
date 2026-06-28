# Vân Ngữ — Technical Deep-Dive

**Three engineering artifacts for the Qualcomm submission**
1. A defensible latency budget on a concrete Snapdragon tier
2. The context-engine design (tag substitution + constrained decoding)
3. The evaluation protocol, including the safety-set methodology

A note on numbers: hardware capabilities (TOPS, model sizes) below are from Qualcomm's published material. Per-stage **latencies are engineering estimates** derived from model size, architecture, and the chip's compute/bandwidth budget. Every one is flagged with what determines it and how to confirm it by profiling on the real device through Qualcomm AI Hub Workbench (which runs on cloud-hosted physical QCS8550 hardware). Do not present the estimates as measured results — present the *method* and the *measured* number you get back.

---

## Part 1 — Latency Budget

### 1.1 Target tier

Primary target: **Qualcomm Dragonwing QCS8550** (the IoT/industrial sibling of Snapdragon 8 Gen 2, and a supported AI Hub chipset).

| Block | Spec | Why it matters here |
|---|---|---|
| NPU (Hexagon HTP, V73) | 48 TOPS INT8 / 12 TOPS FP16, with HVX (vector) + HMX (matrix) | Runs ASR/NMT/TTS inference |
| **Dedicated Hexagon audio DSP** + Sensing Hub 3.0 | always-on, low-power | This is where VAD, denoise, feature extraction belong — **not** the CPU |
| CPU (Kryo) | 1×Cortex-X3 @3.2 GHz, 2×A715 + 2×A710, 3×A510 | Control, routing, glossary/phrase lookup, endpointing logic |
| Memory | LPDDR5X | Multi-GB; RAM capacity is **not** your bottleneck on this tier |

Lower-power wearable tier (for the hardware-tier scaling story): **QCS6490** — fewer TOPS, tighter memory, where model footprint and stage-swapping actually start to bite. Design for QCS8550, validate degradation on QCS6490.

The single most important platform signal you can send a Qualcomm jury: **put the always-on audio front-end on the Hexagon audio DSP, the heavy inference on the HTP, and control on the CPU.** Your current architecture table burns CPU on glossary lookup and never mentions the audio DSP at all.

### 1.2 The metric you must not conflate

Two different quantities, repeatedly confused in voice-AI proposals:

- **Real-Time Factor (RTF)** = compute time / audio duration. A *throughput* measure. RTF < 1 means you can keep up with a live stream. It says nothing about responsiveness.
- **Latency** = wall-clock from end-of-speech to first synthesized audio sample. This is what a worker *feels*.

You can have RTF = 0.4 and still feel sluggish, because latency is dominated by things RTF doesn't capture: the endpoint silence wait, autoregressive decode length, and vocoder time-to-first-sample. **Report both, separately.** Claiming "RTF < 1.0" as evidence of low latency is the single easiest thing for a reviewer to puncture.

### 1.3 The Whisper trap (architecture drives the budget)

Qualcomm's AI Hub Whisper-Base is published with: encoder 23.7M params / 90.7 MB float, decoder 48.9M params / 187 MB float, **fixed input 80 × 3000 = exactly 30 s of audio**, max decode 200 tokens.

That fixed 30 s window is the trap. A Whisper encoder processes 3000 mel frames **regardless of whether the worker said "stop" (0.5 s) or a full sentence (4 s)**. You pay the full encoder cost on every utterance. For short factory commands this is catastrophic for latency.

Two ways out, and you should name which you choose:

- **(A) Streaming ASR** — a chunked Conformer (CTC or RNN-T transducer) with bounded right-context. It emits tokens *during* speech, so at end-of-speech only the final chunk remains. This is the right answer for command-and-control latency. Whisper is a transcription model, not a streaming model; don't let the AI Hub catalog entry decide your architecture for you.
- **(B) Keep Whisper-Tiny but window-trim** — gate with VAD so the encoder only ever sees the active segment padded to a small fixed length (e.g. 5 s), not 30 s. Cheaper, but still non-streaming, so the whole encoder fires only *after* end-of-speech.

The budget below assumes path (A), streaming ASR, and notes the path-(B) penalty.

### 1.4 The decode-stage reality: bandwidth-bound, not compute-bound

A subtle point that will impress a technical jury. ASR decode, NMT decode, and TTS acoustic generation are **autoregressive**: one token at a time, each step reloading model weights from memory. These steps are **memory-bandwidth-bound, not compute-bound.** Your 48 TOPS barely matters for them — what matters is LPDDR5X bandwidth and how small (quantized) the weights are.

Consequence: the lever for decode latency is **quantization (smaller weights → fewer bytes per token)** and **short output sequences (phrase memory, constrained length)**, not raw TOPS. The encoder/feature stages are the compute-bound parts where TOPS helps. Say this — it shows you understand *why* the optimizations work, not just that they do.

### 1.5 The budget (time to first translated audio, short command)

Estimates for a short operational command on QCS8550, streaming ASR path. "Floor" = irreducible regardless of silicon.

| Stage | Where it runs | Estimate | Driver / how to confirm |
|---|---|---|---|
| STFT framing (25 ms / 10 ms hop) | Audio DSP | ~25–35 ms | Algorithmic; window + hop. Floor. |
| VAD + denoise | Audio DSP | ~10 ms added (pipelined w/ capture) | Streaming, overlaps capture |
| **Endpoint silence wait** | DSP/CPU | **~300–500 ms** | **Hard floor.** Trailing-silence rule to declare utterance end. No silicon removes this. |
| ASR finalize (last chunk) | HTP | ~50–150 ms | Streaming emits during speech; only tail remains. Profile on Workbench. |
| LID + phrase-memory lookup | CPU | < 20 ms | Script check + fuzzy match over small index |
| NMT decode (≈10–20 out tokens) | HTP | ~100–300 ms | Bandwidth-bound; ∝ output length. Profile. |
| Glossary tag-restore | CPU | < 5 ms | Deterministic string ops |
| TTS → **first audio sample** | HTP + CPU | ~150–400 ms | Vocoder time-to-first-chunk dominates (see 1.6) |
| Output buffering | CPU | ~50–100 ms | Playout buffer |
| **Total (time to first audio)** | | **≈ 0.8 – 1.5 s** | < 2 s feasible for short commands |

**Phrase-memory hit path** (skips NMT, optionally uses cached audio):

| Stage | Estimate |
|---|---|
| Framing + VAD + endpoint wait | ~350–550 ms |
| ASR finalize | ~50–150 ms |
| Lookup hit | < 20 ms |
| Cached safety audio (no synth) | ~10–30 ms |
| Buffer | ~50–100 ms |
| **Total** | **≈ 0.5 – 0.9 s** |

The phrase-hit path with pre-cached audio is where your "feels instant" demo lives. Make sure your demo commands are in the cache.

**Where < 2 s breaks:** long sentences. NMT and TTS are autoregressive, so a 30-token sentence roughly triples the NMT and TTS lines. Report this honestly with a per-token model rather than a single average:

$$T_{\text{e2e}} \approx T_{\text{fixed}} + T_{\text{wait}} + n_{\text{out}}\cdot(t_{\text{nmt/tok}} + t_{\text{tts/tok}})$$

A single mean latency hides the tail. **Report p50 / p95 / p99**, split by (phrase-hit vs NMT) and (short vs long).

### 1.6 Vocoder is usually the bottleneck

TTS = acoustic model (text → mel) + vocoder (mel → waveform). The vocoder dominates. Options and their cost shape:

- **LPCNet** — autoregressive, runs real-time on CPU but RTF hugs 1.0; sample-by-sample, so latency accumulates with duration.
- **GAN vocoder (HiFi-GAN class), quantized on HTP** — non-autoregressive, generates a waveform chunk in one shot; far better time-to-first-sample. Preferred for this latency target.

What the worker perceives is **time-to-first-audio-sample**, not total synthesis time — synthesis can stream and overlap with playback. Budget and report *first-sample* latency, and let the rest play out.

### 1.7 The latency lever you're not using: wait-$k$ simultaneous translation

The endpoint wait + two autoregressive stages are most of the budget. The way to actually beat them is to **not wait for end-of-speech**: begin translating after the first $k$ source tokens (wait-$k$ policy), so by the time the speaker stops you're nearly done.

This is unusually safe for **Vietnamese↔Mandarin** because both are **SVO** with mild reordering pressure, so a low-latency wait-$k$ rarely has to revise. The cost is occasional revision when ASR updates its hypothesis. Even discussing this signals you understand the difference between "fast pipeline" and "real-time interpreting."

### 1.8 Memory / model footprint budget

| Model (INT8 / w8a16) | Approx. footprint |
|---|---|
| Neural denoiser | ~2–10 MB |
| Neural VAD | < 1 MB |
| ASR (streaming Conformer, small) or Whisper-Tiny | ~15–50 MB |
| NMT (distilled small Transformer) | ~30–80 MB |
| TTS acoustic + GAN vocoder | ~10–40 MB |
| Phrase memory + bilingual glossary | a few MB |
| **Peak co-resident** | **~150–250 MB** |

On QCS8550's LPDDR5X this fits comfortably — **capacity is not the constraint.** The real costs are: (1) **NPU graph-switch overhead** when moving between ASR → NMT → TTS contexts on the HTP (if you swap rather than co-resident-load, that swap time goes *into the latency table above*); and (2) on the **QCS6490 wearable tier**, the budget genuinely tightens and you may have to swap. State whether models are co-resident or swapped, and put any swap cost in the budget.

### 1.9 What to actually submit for Part 1

- The table in §1.5 with your **own Workbench-profiled numbers** substituted in (compile each model, profile on QCS8550, paste latency + peak memory).
- RTF and latency reported **separately** (§1.2).
- A named ASR architecture choice (§1.3) and the reasoning.
- p50/p95/p99, split by path (§1.5).
- The DSP/HTP/CPU placement (§1.1) drawn correctly in your architecture diagram.

---

## Part 2 — Context-Aware Translation Engine (the mechanics)

Your proposal says context is "incorporated during translation rather than corrected after." Correct instinct, currently a black box. Here is the concrete machinery, tiered by cost, plus why this language pair makes it unusually tractable.

### 2.1 Four mechanisms, ranked by cost

**Tier 1 — Typed placeholder (tag) substitution. *Default for entities & IDs.***
Deterministic, ~zero latency, guarantees terminology fidelity. Covers the bulk of industrial terms: machine IDs, line numbers, SOP codes, named equipment, PPE terms.

1. **Detect** glossary spans in the ASR text using a per-factory gazetteer compiled into an **Aho-Corasick automaton** (or trie) over both languages — longest-match, $O(\text{text length})$.
2. **Substitute** each matched span with a **typed sentinel** that the NMT passes through untranslated, e.g. `⟨MACHINE_07⟩`, `⟨SOP_12⟩`, `⟨PPE⟩`. Keep a map `placeholder → {src surface, tgt surface}`.
3. **Translate** the masked sentence.
4. **Restore** placeholders with the **target-language** surface form from the bilingual glossary.

**Tier 2 — Lexically-constrained decoding. *For mandated wording only.***
Some terms must map to one official phrase (safety verbs, regulated terminology) and can't be blind-substituted because they must inflect/agree with the sentence. Force them into the output during beam search:
- **Grid Beam Search** (Hokamp & Liu, 2017) — guarantees constraints appear; beam grows with #constraints.
- **Dynamic Beam Allocation** (Post & Vilar, 2018) — same guarantee, **bounded** beam width; the efficient choice.

Cost: beam-search overhead fights your latency budget. **Reserve for the small mandated-phrase set**, not general vocabulary.

**Tier 3 — Whole-sentence phrase memory.** Already in your design. Highest-frequency complete commands bypass NMT entirely (see safety note 2.4).

**Tier 4 — Prompt/prefix glossary conditioning.** Only if your NMT is a small decoder-LM: prepend relevant glossary entries as context. Flexible but **bloats context → raises latency**, and gives no hard guarantee. Mention as a fallback, don't lead with it.

**Recommended stack:** Tier 3 for frequent whole commands → Tier 1 for everything entity-like → Tier 2 for the mandated safety subset. Cheapest mechanism that can handle each case handles it.

### 2.2 Why this pair is unusually friendly to tag substitution

The classic failure of placeholder substitution is **morphology and agreement** — the inserted term must inflect to fit the target grammar. But **Mandarin is morphologically near-isolating** and **Vietnamese is analytic** (no inflection, no agreement, no case). A restored surface form drops in cleanly in both directions. This is a real, defensible advantage of your specific language pair — call it out. The residual risk is **word order**, addressed next.

### 2.3 The training requirement (the part people skip)

A vanilla NMT model will mishandle sentinel tokens — drop them, duplicate them, or place them wrong. You must **fine-tune the NMT on synthetic data containing the sentinels and constraints**, so it learns to (a) preserve each placeholder exactly once and (b) position it correctly in target word order. Generate this data by templating your glossary into sentence frames and round-tripping. Without this step, Tier 1 silently fails.

### 2.4 Worked examples (using your own phrases)

**Entity (Tier 1), Vi → Zh:**
```
ASR:        Kiểm tra máy số 3
Detect:     "máy số 3"  →  ⟨MACHINE_03⟩
Masked:     Kiểm tra ⟨MACHINE_03⟩
NMT:        检查 ⟨MACHINE_03⟩
Restore:    ⟨MACHINE_03⟩ → "3号机"
Output:     检查3号机
```
The machine ID can never be mistranslated, because it never entered the model.

**Mandated safety wording (Tier 2), Vi → Zh:**
```
"Dừng dây chuyền"  must always render as the official  "停止生产线"
→ enforce as a decoding constraint, not a guess.
```

### 2.5 The fail-safe (this is also a safety mechanism)

A placeholder-count check is a free, powerful guardrail: **if the number of sentinels going into the NMT ≠ the number coming out, the translation is corrupt — reject it.** On rejection, fall back to phrase memory or a safe templated rendering and **signal uncertainty** ("không chắc — xin nhắc lại" / "不确定，请重复"). This directly addresses the safety-inversion risk: a confident wrong translation of a safety command is the worst failure mode, and the count check catches a large class of them deterministically.

### 2.6 The real moat

Vietnamese↔Mandarin is comparatively low-resource. Your translation ceiling is set by **domain parallel data**, not by the inference stack. Build it from: bilingual SOP/manual documents, glossary-templated synthetic sentences (which also serve §2.3), and back-translation augmentation of monolingual factory text. **This dataset is more of a competitive moat than the pipeline** — say so, and describe how you'd build it.

---

## Part 3 — Evaluation Protocol

No measurement plan is the biggest hole in the current proposal. For a *safety* product, "we translate factory commands" without "here is our error rate on emergency phrases and what happens when we're unsure" is the gap between a demo and a fundable product. This section is the plan.

### 3.1 Metrics per stage

**ASR**
- **CER** primary, WER secondary. CER handles Vietnamese diacritics and Mandarin characters more meaningfully than WER.
- Measured **clean *and* in factory noise** — an SNR sweep (e.g. +10, +5, 0, −5 dB) using **real factory recordings**, augmented with noise corpora (MUSAN / DEMAND). Clean-only WER is meaningless for this product.
- **Tone-confusion rate**, reported separately: a curated **minimal-pair set** where items differ only in tone; measure the substitution rate within tonal pairs. This is where aggressive denoising and INT8 quantization do their damage, and aggregate WER hides it.
- Reported **per language** and on a **code-switch set** (mixed Vi/Zh terms in one utterance).

**Language ID**
- Accuracy + confusion matrix; explicit **code-switch behavior** (does routing survive mixed input?).

**NMT**
- **chrF** and **COMET** (neural, better human correlation) as primary; **sacreBLEU** with character-level tokenization for Chinese as secondary.
- **Terminology accuracy**: % of glossary terms rendered correctly — *the metric that matters for industry*, often uncorrelated with BLEU.
- **Safety-command accuracy**: exact/semantic match on the emergency set (see 3.3).

**TTS**
- **MOS** (naturalness), native-listener panel.
- **Intelligibility** via round-trip: run synthesized audio back through ASR, measure WER.
- **Tone-correctness MOS**, scored *separately* by native listeners — natural-sounding speech with wrong tones is a safety failure, and naturalness MOS won't catch it.

**End-to-end**
- **Task success rate**: does the listener take the correct action? The real product metric.
- **Latency p50/p95/p99** for time-to-first-audio and time-to-full-audio, split by phrase-hit vs NMT and short vs long (ties back to Part 1).
- **Safety metrics** (see 3.3).

### 3.2 Test sets

| Set | Purpose |
|---|---|
| Clean read speech | Baseline upper bound |
| Factory-noise speech | Real conditions; multiple machine types, **multiple speakers incl. Northern & Southern Vietnamese**, gender/age diversity |
| Code-switch set | Mixed-language robustness |
| Tone minimal-pair set | Isolate tonal degradation from quantization/denoise |
| **Safety command set** | The critical one — see 3.3 |

### 3.3 Safety-set methodology (the part that makes it fundable)

1. **Curate** the operational safety command set (emergency shutdown, stop line, PPE, lockout, etc.) with **bilingual-reviewer-approved correct translations**.
2. **Severity tiers.** Define failure severity, e.g.:
   - *Critical*: meaning inversion or safety-instruction reversal (e.g. "don't shut off" → "shut off"). **Zero-tolerance gate.**
   - *Major*: wrong entity (wrong machine number).
   - *Minor*: dysfluency that preserves meaning.
3. **Phrase-memory false-positive rate.** The fuzzy matcher's threshold is a **safety parameter** — measure how often it returns a confident wrong cached translation, especially on near-miss minimal pairs ("đừng tắt máy" vs "tắt máy"). Report the FP rate as a function of the threshold.
4. **Fail-safe coverage.** Define and measure: when the system is wrong, how often does it correctly **signal uncertainty / ask for repeat** instead of confidently emitting the error? This is an acceptance criterion, not a nice-to-have.
5. **Human adjudication** for the critical tier; report with confidence intervals.

### 3.4 Ablations (these directly test the critique)

- **Denoise → ASR, on vs off.** Tests whether the front-end denoiser *helps or hurts* WER (enhancement artifacts can raise WER on a mismatched ASR). If it hurts, route denoising only to the human-audible path.
- **PTQ vs QAT** for ASR — measured on the **tone-confusion** set, not just aggregate WER.
- **Phrase memory on/off** — latency *and* the safety FP cost.
- **Tag substitution on/off** — terminology accuracy.
- **wait-$k$ vs full-utterance** — latency vs revision rate.

### 3.5 Statistics & harness

- Report **confidence intervals** (bootstrap) and **significance tests** when comparing configurations — don't compare bare point estimates.
- **On-device latency & peak memory** profiled on real **QCS8550** hardware via **Qualcomm AI Hub Workbench**, not on a laptop. These are the numbers that fill Part 1's table.

### 3.6 Illustrative acceptance targets (set your own, but commit to them)

| Metric | Target (example) |
|---|---|
| CER in 0 dB factory noise (per language) | below a stated threshold |
| Tone-confusion rate | below a stated threshold |
| Terminology accuracy | ≥ high-90s % |
| **Critical-tier safety errors** | **zero (hard gate)** |
| Fail-safe coverage of true errors | ≥ stated % |
| Time-to-first-audio, p95, short command | < 2 s |

---

### Sources for hardware figures
QCS8550 / Dragonwing: 48 INT8 / 12 FP16 TOPS, Hexagon HTP (HVX + HMX), dedicated Hexagon audio DSP, Sensing Hub 3.0, Kryo CPU, Adreno 740, LPDDR5X — Qualcomm Dragonwing QCS8550/QCM8550 product material and SOM vendor briefs (Lantronix Open-Q 8550CS, Thundercomm/RidgeRun, Firefly). Whisper-Base sizes (encoder 23.7M/90.7 MB, decoder 48.9M/187 MB, 80×3000 fixed input, 200-token max decode) and w8a16 quantization — Qualcomm AI Hub model pages. Constrained-decoding methods — Hokamp & Liu (2017), Post & Vilar (2018).
