# StageGuard: A Multi-Stage Runtime Security Monitor for Agentic AI Systems

StageGuard is a runtime security monitor for LLM agents that inspects **every stage** of an agent's execution pipeline — prompt ingestion, retrieval/context ingestion, tool selection, parameter construction, and a final pre-execution gate — rather than only the first prompt and the final output. It combines a deterministic rule engine, a statistical anomaly detector, argument-provenance verification, and an LLM-based semantic judge under a **confidence-gated aggregation policy** that stops the least reliable layer from silently overriding the most reliable ones.

Evaluated on [InjecAgent](https://github.com/uiuc-kang-lab/InjecAgent) and validated for benign-task behavior on [AgentDojo](https://github.com/ethz-spylab/agentdojo). Full results, ablations, and limitations are reported in the accompanying paper.

## How the pipeline works

1. **Prompt ingestion** — rule-based scan of the user instruction.
2. **Retrieval/context ingestion** — rule scan + a 14-dimensional stylometric anomaly detector (isolation forest, calibrated on a benign corpus) scoring retrieved content.
3. **Tool-selection validation** — checks the proposed tool call against a tool-risk registry (Low/Medium/High).
4. **Parameter inspection** — argument-provenance check: flags tool-call parameters that are absent from the user's own instruction but traceable to untrusted retrieved context.
5. **Pre-execution policy gate** — all signals are combined via noisy-OR aggregation; ambiguous or untrusted-source cases are escalated to an LLM semantic judge (Layer 3), merged back in with a confidence-gated rule so it can only raise risk on already-confident findings, never dilute them.

Each decision resolves to **Allow**, **Sanitize-and-Allow**, **Require Human Approval**, or **Block**, and every flagged action is logged with the stage and evidence responsible (append-only, hash-chained audit log).

## What's in the notebook

The notebook (`cs-revised__6_.ipynb`) is organized as ~18 sequential, independently re-runnable cells:

1. Setup, run profile, caching, dependency install
2. Load the local Qwen2.5-7B-Instruct agent (and optional local judge)
3. Download InjecAgent test cases, tool definitions, and user cases
4. Build the external `policy.yaml` (three profiles: strict/balanced/permissive) and hash it for reproducibility
5. Tool-risk and provenance registries, plus an explicit oracle-ablation registry
6. Layer 1 — rule sets (core/full tiers) and a benign-acceptance self-test
7. Load the AgentDojo benign corpus (required source for Layer 2 calibration)
8. Layer 2 — stylometric feature extraction + isolation-forest anomaly detector
9. Provenance and data-stealing chain detector, argument-provenance scoring
10. Action generation (stochastic decoding, request-hash caching, seed self-check)
11. Layer 3 — local/Groq semantic judge, noisy-OR aggregation
12. Sanitizer (sentence-level span removal driven by actual detections)
13. Main evaluation loop across defenses and conditions
14. Offline detector metrics and DEV-only threshold tuning
15. Metrics: ASR, prevention-vs-none, hard-FPR, approval burden, clustered bootstrap CIs
16. Ablation configurations (L1-only, no-provenance, uniform-blend vs. confidence-gated aggregation, etc.)
17. AgentDojo integration for benign-task validation
18. Extended-seed runs, per-seed variance, stage-localization confusion matrix, L3-dilution diagnostic, and final result/figure export

## Running the notebook (Kaggle)

1. Upload the `.ipynb` file to [Kaggle](https://www.kaggle.com/code) as a new notebook, or open it directly if importing from GitHub.
2. **Enable GPU**: Notebook settings → Accelerator → GPU (needed to run the local Qwen2.5-7B-Instruct agent at a reasonable speed).
3. **Dependencies**: the first cell installs everything it needs at runtime from `requirements.txt` — no manual setup required.
4. **(Optional) Groq API key**: if you want to use the hosted Layer-3 judge backend instead of a local model, add a Kaggle secret named `GROQ_API_KEY` (Add-ons → Secrets), or set it as an environment variable.
5. **(Optional) AgentDojo**: only needed for the benign false-positive validation step; the notebook installs it automatically if listed, and degrades gracefully with a notice if unavailable.
6. Set the run profile before executing, e.g. at the top of the setup cell:
   ```python
   import os
   os.environ["STAGEGUARD_PROFILE"] = "standard"  # quick | standard | full
   ```
   - `quick` — small sample sizes, for fast iteration/debugging.
   - `standard` — the configuration used for the paper's reported results.
   - `full` — full DEV/TEST split (374/200), all three seeds, 200 bootstrap resamples.
7. Run all cells top to bottom. Each stage is cached (content-hash keyed JSONL) so re-running an unchanged configuration doesn't re-invoke the LLM. Result tables and figures are written to the notebook's working directory (`/kaggle/working/`) and can be downloaded from the Output tab when the run finishes.

## Limitations

- Most context-borne (retrieval-stage) injections that go undetected are a coverage gap in the retrieval-stage rules/anomaly detector, not a localization or aggregation failure.
- AgentDojo is used only for benign false-positive validation; adversarial ASR on AgentDojo is untested pending harness integration with a locally-hosted model.
- All results use a single agent/judge model pair (Qwen2.5-7B-Instruct); cross-model generalization is untested.
- Results are reported on InjecAgent only; generalization to other injection styles or agent frameworks is untested.

## Acknowledgments

Built on top of [InjecAgent](https://github.com/uiuc-kang-lab/InjecAgent) (Zhan et al., 2024) and [AgentDojo](https://github.com/ethz-spylab/agentdojo) (Debenedetti et al., 2024). Agent backbone: [Qwen2.5-7B-Instruct](https://huggingface.co/Qwen/Qwen2.5-7B-Instruct).
