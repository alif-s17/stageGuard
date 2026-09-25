# StageGuard: A Multi-Stage Runtime Security Monitor for Agentic AI Systems

StageGuard is a runtime security monitor for LLM agents that inspects **every stage** of an agent's execution pipeline — prompt ingestion, retrieval/context ingestion, tool selection, parameter construction, and a final pre-execution gate — rather than only the first prompt and the final output. It combines a deterministic rule engine, a statistical anomaly detector, argument-provenance verification, and an LLM-based semantic judge under a **confidence-gated aggregation policy** that stops the least reliable layer from silently overriding the most reliable ones.

Evaluated on [InjecAgent](https://github.com/uiuc-kang-lab/InjecAgent) and validated for benign-task behavior on [AgentDojo](https://github.com/ethz-spylab/agentdojo).

## Key results

| Defense | ASR ↓ | Hard-FPR ↓ | Utility ↑ | Latency (ms) |
|---|---|---|---|---|
| None (undefended) | 0.270 | 0.000 | 1.000 | 0 |
| Prompt hardening | 0.285 | 0.000 | 1.000 | 0 |
| Endpoint-only | 0.265 | 0.005 | 0.995 | 0.2 |
| **StageGuard (full)** | **0.035** | 0.005 | 0.995 | 705.5 |

StageGuard reduces attack success rate by 87% relative to both the undefended agent and an endpoint-only baseline, while hard-blocking only 0.5% of benign tasks, and attributes 83% of flagged actions to the correct pipeline stage (100% accuracy whenever an attack is detected at all). See the [paper](./paper/StageGuard.pdf) for full results, ablations, and limitations.


## How the pipeline works

1. **Prompt ingestion** — rule-based scan of the user instruction.
2. **Retrieval/context ingestion** — rule scan + a 14-dimensional stylometric anomaly detector (isolation forest, calibrated on a benign corpus) scoring retrieved content.
3. **Tool-selection validation** — checks the proposed tool call against a tool-risk registry (Low/Medium/High).
4. **Parameter inspection** — argument-provenance check: flags tool-call parameters that are absent from the user's own instruction but traceable to untrusted retrieved context.
5. **Pre-execution policy gate** — all signals are combined via noisy-OR aggregation; ambiguous or untrusted-source cases are escalated to an LLM semantic judge (Layer 3), merged back in with a confidence-gated rule so it can only raise risk on already-confident findings, never dilute them.

Each decision resolves to **Allow**, **Sanitize-and-Allow**, **Require Human Approval**, or **Block**, and every flagged action is logged with the stage and evidence responsible (append-only, hash-chained audit log).

## Setup

### Requirements
- Python 3.10+
- A GPU is strongly recommended for the local agent model (Qwen2.5-7B-Instruct); CPU will work but is slow.
- (Optional) A [Groq](https://groq.com/) API key if using the hosted Layer-3 judge backend instead of a local model.

### Install

```bash
git clone https://github.com/<your-username>/stageguard.git
cd stageguard
pip install -r requirements.txt
```

### Configuration

The notebook reads a profile via environment variable:

```bash
export STAGEGUARD_PROFILE=standard   # quick | standard | full
export GROQ_API_KEY=your_key_here    # optional, only if JUDGE_BACKEND=groq
```

- `quick` — small sample sizes, for fast iteration/debugging.
- `standard` — the configuration used for the results reported in the paper.
- `full` — full DEV/TEST split (374/200), all three seeds, 200 bootstrap resamples.

### Data

The notebook automatically downloads InjecAgent test cases and tool definitions from the [official InjecAgent repo](https://github.com/uiuc-kang-lab/InjecAgent) on first run. AgentDojo requires a local install (`pip install agentdojo`) for the benign-corpus validation step; it is only used to check false-positive behavior, not adversarial ASR (see Limitations in the paper).

### Running

Open `notebooks/stageguard_pipeline.ipynb` in Jupyter, Kaggle, or Colab and run cells top to bottom. Each stage is cached (content-hash keyed JSONL) so repeated runs of an unchanged configuration don't re-invoke the LLM. Outputs (policy file, audit log, result tables, figures) are written to `artifacts/`.

Policy thresholds and layer weights for all three profiles (strict/balanced/permissive) are defined in `artifacts/policy.yaml` after the setup cell runs, and the SHA-256 hash of the active policy is logged with every run for reproducibility.

## Reproducing the paper's tables

| Paper | Notebook section | Output |
|---|---|---|
| Table 4 (main comparison) | Cells 16–18, 20 | `artifacts/table_main_comparison.csv` |
| Table 5 (ablations) | Cell 16 | `artifacts/table_ablation.csv` |
| Table 6 (stage localization) | Cell 18, 22 | `artifacts/table_stage_localization.csv`, confusion matrix |
| Figure 2 (ASR vs. latency) | Cell 24 | `artifacts/fig2_asr_latency.png` |
| Figure 3 (ablation bar chart) | Cell 16 | `artifacts/fig3_ablation.png` |

## Limitations

- Most context-borne (retrieval-stage) injections that go undetected are a coverage gap in the retrieval-stage rules/anomaly detector, not a localization or aggregation failure.
- AgentDojo is used only for benign false-positive validation; adversarial ASR on AgentDojo is untested pending harness integration with a locally-hosted model.
- All results use a single agent/judge model pair (Qwen2.5-7B-Instruct); cross-model generalization is untested.
- Results are reported on InjecAgent only; generalization to other injection styles or agent frameworks is untested.

Full discussion in Section 6 of the paper.

## Roadmap

- [ ] Refactor the notebook into a modular `src/` package (rule engine, anomaly detector, provenance checker, aggregator, judge, evaluation harness) importable outside a notebook.
- [ ] Close the context-ingestion detection gap.
- [ ] Full adversarial evaluation on AgentDojo.
- [ ] Test generalization across additional base agent/judge models.

## Citation

If you use StageGuard, please cite:

```bibtex
@inproceedings{stageguard2027,
  title     = {StageGuard: A Multi-Stage Runtime Security Monitor for Agentic AI Systems},
  author    = {Mim, Nusrat Jahan and Sagar, Md. Alif Hossain and Azam, Shifat Bin},
  year      = {2027},
  note      = {Preprint}
}
```

## License

This project is licensed under the MIT License — see [LICENSE](./LICENSE).

## Acknowledgments

Built on top of [InjecAgent](https://github.com/uiuc-kang-lab/InjecAgent) (Zhan et al., 2024) and [AgentDojo](https://github.com/ethz-spylab/agentdojo) (Debenedetti et al., 2024). Agent backbone: [Qwen2.5-7B-Instruct](https://huggingface.co/Qwen/Qwen2.5-7B-Instruct).
