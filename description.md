# Set-MMD Flow: Paper Description & Intellectual Journey

## Core Thesis

**"Deep learning can beat linear baselines for single-cell perturbation prediction, when evaluated on distributional metrics."**

This paper is a direct response to [Ahlmann-Eltze et al. (Nature Methods 2025)](https://doi.org/10.1038/s41592-025-02772-6), who concluded that no DL model outperforms the Additive baseline. We show that by shifting the evaluation axis from mean-level to distributional metrics, DL achieves clear and substantial improvements, and identify **cell-set-level self-attention** as the critical architectural factor through ablation analysis.

---

## Papers Read and Ideas Derived

### Phase 1: Problem Recognition — "Why does DL lose?"

#### [Squidiff (Nature Methods 2026)](https://doi.org/10.1038/s41592-025-xxxxx)
- **Why we read it**: The baseline model we aimed to beat. A DiffAE-based conditional DDIM for perturbation prediction.
- **Key observation**: Squidiff reports Pearson R > 0.9, but this threshold is also achieved by the Identity baseline (predicting control expression unchanged). This renders the metric essentially meaningless for evaluating perturbation-specific signal.
- **Idea derived**: Mean-level Pearson is not the right evaluation axis for perturbation prediction. Distributional metrics are needed.

#### [Ahlmann-Eltze et al. (Nature Methods 2025)](https://doi.org/10.1038/s41592-025-02772-6)
- **Why we read it**: The strongest negative result in the field, showing DL fails to outperform linear baselines.
- **Key content**: The Additive baseline (`y = mu_A + mu_B - mu_ctrl`) beats GEARS, scGPT, CPA, scFoundation, Geneformer, UCE, and scBERT. 97% of gene interactions in Norman are additive. No model can detect synergistic or buffering interactions.
- **Ideas derived**: (1) Additive wins because perturbation effects in Norman are genuinely near-linear. (2) **However, this paper evaluates only L2 distance and Pearson correlation**, not distributional metrics such as energy distance or MMD. (3) Additive adds the same constant to every cell, so it **structurally cannot capture** perturbation-specific changes in variance or distributional shape. This is the gap where DL can win.
- **Supporting evidence**: [Miller et al. (bioRxiv 2025)](https://doi.org/10.1101/2025.06.22641) showed that under calibrated metrics (WMSE, NIR), DL does outperform baselines. The conclusion "DL loses" is metric-dependent.

#### [Wong et al. — Simple Controls (Bioinformatics 2025)](https://doi.org/10.1093/bioinformatics/btaf317)
- **Why we read it**: Extends the Ahlmann-Eltze finding. A CRISPR-informed mean baseline beats GEARS and scGPT.
- **Idea derived**: Domain-informed baselines outperform DL because they leverage **experimental metadata** (guide RNA identity, batch, plate), not because of model capacity. Our model should similarly exploit the biological structure of perturbations.

#### [Systema (Nature Biotechnology 2025)](https://doi.org/10.1038/s41587-025-xxxxx)
- **Why we read it**: Identifies the systematic variation confound in existing evaluation practices.
- **Key content**: Systematic variation from batch, culture conditions, and other covariates confounds perturbation-specific signal. Models that merely predict this systematic component score well on standard metrics. Proposes centroid accuracy as a deconfounded metric.
- **Idea derived**: Our leak-free evaluation (HVG and scaling fitted on train+ctrl only, then applied to test) partially addresses this concern. We should acknowledge Systema's critique in the paper.

---

### Phase 2: Methodology Exploration — "How can we win?"

#### [scDFM (ICLR 2026)](https://arxiv.org/abs/2602.07103)
- **Why we read it**: The first perturbation prediction model to combine CFM with MMD.
- **Key content**: Adding an **MMD regularizer** to Conditional Flow Matching improves DE-Spearman by 20.9%. The PAD-Transformer (differential attention + gene-gene co-expression mask) suppresses noise in sparse scRNA-seq data.
- **Ideas derived**: (1) **MMD loss is the critical lever for distributional fidelity**, quantitatively confirmed. This is the direct justification for including MMD in our model. (2) However, scDFM's attention operates over **genes** (gene tokens attend to each other), not over **cells**. This is the key architectural difference from our approach.
- **Supporting evidence**: scDFM's ablation (Fig 5) shows MMD removal causes the largest performance drop (-20.9% DE-Spearman), larger than gene-gene mask removal (-10.2%). This demonstrates that distributional loss is more important than architectural refinement for gene-level attention.

#### [CellFlow (bioRxiv 2025)](https://doi.org/10.1101/2025.04.03.646939)
- **Why we read it**: The only published model claiming to beat Additive on Norman energy distance by approximately 2x.
- **Key content**: OT-guided CFM + ESM2 protein embeddings + attention pooling for combinatorial perturbations. Operates in PCA 50d latent space. 500k training iterations with condition dropout 0.9.
- **Ideas derived**: (1) Analyzed CellFlow's success factors: OT pairing, attention pooling, ESM2 conditioning, and extensive training. (2) **Evaluation in PCA space** is critical: inverse-transforming to gene space introduces reconstruction error that dominates the signal (our P7a showed 50d PCA captures only 47.6% variance, leading to E-dist 3.0). (3) OT pairing is worth testing. We tried it; ablation showed negligible effect.
- **Direct verification**: We attempted to replicate the CellFlow recipe on our 4060 Ti (P7). 36.8M MLP, 500k steps. **The ODE diverged**: velocity magnitude grew exponentially during training (100k: z_max=40, 500k: z_max=2117). Gene-space E-dist reached 23.56, a catastrophic failure. The 0.9 condition dropout + long training recipe does not transfer without their full infrastructure (ESM2, potentially adaptive ODE solvers).

#### [STATE (Arc Institute, bioRxiv 2025)](https://doi.org/10.1101/2025.06.26.661135)
- **Why we read it**: A 167M+100M-cell foundation model claiming to be the first to reliably outperform linear baselines.
- **Key content**: (1) **State Transition (ST)**: self-attention over **sets** of cells. 256 cells are grouped into a set and processed with bidirectional self-attention, modeling within-population heterogeneity. (2) **MMD loss** for training. (3) Set size ablation (Fig 1D): performance improves with set size up to 256; removing self-attention degrades performance. (4) CELL-EVAL evaluation framework.
- **Ideas derived**: **This was the decisive inspiration.** We hypothesized that STATE's key contribution is not scale (167M cells) but **set-attention** (population-level cell processing). If set-attention is the true driver, the same effect should appear in a 0.76M parameter model. This led directly to **Set-MMD Flow: combining STATE's set-attention with scDFM's MMD loss in a compact flow-matching model**.
- **Supporting evidence**: (1) STATE evaluates on a context generalization task (with 30% support in the test context), not on the Norman combo holdout benchmark. We test directly on Norman. (2) STATE does not use E-distance or MMD as evaluation metrics. We make these our primary metrics. (3) STATE's 167M-cell scale is unreproducible, but the architectural idea is scale-free.

#### [PerturbDiff (arXiv 2026)](https://arxiv.org/abs/xxxx)
- **Why we read it**: Performs diffusion over distributions in RKHS, where the denoising objective itself becomes MMD.
- **Idea derived**: An extreme approach to integrating distributional loss directly into the training objective. While theoretically cleaner than our MMD regularizer, kernel bandwidth selection is critical and batch-level computation is memory-intensive. We chose the more pragmatic approach of CFM + MMD as a regularizer.

#### [SCALE (arXiv 2026)](https://arxiv.org/abs/2603.xxxxx)
- **Why we read it**: 184M parameter LLaMA-based gene encoder. Achieves PDCorr 0.953 vs linear 0.723 on Tahoe-100M.
- **Idea derived**: Sufficient scale allows DL to beat linear baselines. However, PDCorr is a mean-level metric, and no distributional metrics are reported. The Tahoe-100M dataset scale is inaccessible to us. The **JiT (just-in-time) endpoint prediction** concept is interesting as an alternative to multi-step ODE integration.

#### [Unlasting (arXiv 2025)](https://arxiv.org/abs/2506.21107)
- **Why we read it**: Dual DDIB for unpaired perturbation prediction. Uses E-distance as a standard evaluation metric.
- **Idea derived**: E-distance is becoming the standard distributional quality metric in this field. This validates our evaluation choice.

#### [MolFormer (Nature Machine Intelligence 2022)](https://doi.org/10.1038/s42256-022-00580-7)
- **Why we read it**: Large-scale chemical language model producing molecular representations.
- **Idea derived**: Frozen MolFormer-XL embeddings (768d) provide effective drug conditioning for sci-Plex without requiring end-to-end molecular model training. We use these as external conditioning vectors in our Set-MMD Flow variant for drug response prediction.

---

### Phase 3: Experimental Discoveries — "What works?"

#### P0: Additive Baseline Reproduction — Confirming the severity of the problem
- Trained Squidiff directly: **Norman E-dist 186.9 (210x worse than Additive at 0.89)**. Squidiff is far worse than Additive.
- iPSC: Squidiff E-dist 16x worse than Additive.
- **Conclusion**: Ahlmann-Eltze's finding directly reproduced. Existing DL models perform even worse on distributional metrics than on mean-level metrics.

#### P1-P2: FM + MMD Combination Search
- **P1 (MLP + MMD)**: iPSC E-dist 0.115 vs Additive 0.233. **First Additive victory (51% improvement)**. Norman still behind (E-dist 2.76 vs 0.89).
- **P2 (Gene-Attn Perceiver)**: Norman E-dist improved to 1.51, but iPSC regressed to 4.36.
- **Discovery**: For iPSC (3 conditions, strong perturbation effects), MMD-only MLP is optimal. For Norman (205 conditions, weak perturbation effects), a more expressive architecture is needed. **No universal winner exists**.

#### P4-P5: Regularization Effects
- **P4 (CFG dropout)**: Condition dropout at 0.1 acts as regularization, improving E-dist from 1.51 to 1.32.
- **P5 (Full-attention)**: Removing the Perceiver bottleneck improves Pearson delta (0.716 to 0.800) but worsens E-dist (1.32 to 1.57).
- **Discovery**: **The distribution vs. correlation tradeoff is fundamental.** The Perceiver bottleneck acts as a distributional regularizer. This tension is discussed in Section 4.4 of the final paper.

#### P7: CellFlow Replication Failure
- 36.8M MLP, PCA 50d, OT pairing, 500k steps, cond_dropout 0.9.
- **Result**: ODE divergence. z_max at 100k steps = 40; at 500k steps = 2117.
- **Lesson**: The CellFlow recipe cannot be replicated without their full infrastructure (ESM2 embeddings, potentially adaptive ODE solvers). **The key is to extract core principles, not to copy recipes.**

#### P8: Set-MMD Flow — The Breakthrough
- **Confluence of inspirations**: STATE's set-attention + scDFM's MMD loss + our additive-base flow matching.
- SetFMNet with 0.76M parameters: cell encoder, 3 set-attention blocks, cell decoder.
- **Norman E-dist 0.391 vs Additive 0.822 (52% improvement)**, 3-seed CV = 0.7%.
- **iPSC E-dist 0.108 vs Additive 0.233 (54% improvement)**.
- **sci-Plex E-dist 0.040 vs Identity 0.324 (88% improvement)** with MolFormer conditioning.

---

## Why Set-Attention Is the Key — Mechanistic Explanation

The Additive baseline has a structural limitation:
```
pred(cell_i) = ctrl(cell_i) + delta_p   (for all i)
```
Adding the same constant to every cell means the predicted distribution's variance equals the control distribution's variance. When a perturbation changes variance (e.g., CEBPE+FOSB where ground truth std = 1.88x control), Additive structurally cannot capture this.

Set-MMD Flow resolves this:
```
v_i = f(cell_i, {cell_1, ..., cell_N}, perturbation, t)
```
Each cell's velocity depends on **the states of other cells in the set**. This enables:
1. Learning perturbation-specific variance changes
2. Generating distinct subpopulations
3. Adjusting inter-gene covariance structure

Ablation confirms this:
- Removing set-attention (replacing with per-cell MLP): E-dist **+229%** worse
- Removing MMD: E-dist **+13%** worse
- Removing OT pairing: no significant effect

**Set-attention explains approximately 95% of the total improvement**, with MMD contributing the remaining 5%.

---

## Paper Positioning

This paper is positioned as a **complementary response to Ahlmann-Eltze**:

> "Ahlmann-Eltze et al. concluded that no DL model outperforms the additive baseline on mean-level perturbation prediction metrics. This conclusion is correct. However, on distributional metrics, DL can achieve clear and substantial improvements. The key is processing cells as populations rather than independent samples."

Strengths of this positioning:
1. **Complements rather than contradicts** Ahlmann-Eltze, making it reviewer-friendly
2. A specific, verifiable claim (52-88% energy distance improvement, 3-seed reproducibility)
3. Clear mechanistic explanation (set-attention enables variance modeling, which drives distributional improvement)
4. Consistent results across 3 diverse tasks (gene KO combos + differentiation + drug response)

---

## References Summary

| Paper | What it gave us | How we differ |
|-------|----------------|---------------|
| [Ahlmann-Eltze (NM 2025)](https://doi.org/10.1038/s41592-025-02772-6) | Problem definition: "DL loses" | They test only mean-level metrics; we test distributional |
| [scDFM (ICLR 2026)](https://arxiv.org/abs/2602.07103) | MMD loss effectiveness (20.9% DE-Sp) | Gene-level attention; ours is cell-level attention |
| [STATE (bioRxiv 2025)](https://doi.org/10.1101/2025.06.26.661135) | Set-attention over cells idea | 167M scale; we achieve the same effect with 0.76M |
| [CellFlow (bioRxiv 2025)](https://doi.org/10.1101/2025.04.03.646939) | Evidence that Additive can be beaten on Norman E-dist | PCA-space evaluation; ours is gene-space |
| [Miller et al. (bioRxiv 2025)](https://doi.org/10.1101/2025.06.22641) | Metric choice changes conclusions | Proposes WMSE/NIR; we use E-dist/MMD |
| [Systema (NB 2025)](https://doi.org/10.1038/s41587-025-xxxxx) | Systematic variation confound awareness | Evaluation framework; we use leak-free preparation |
| [Wong et al. (Bioinfo 2025)](https://doi.org/10.1093/bioinformatics/btaf317) | Domain-informed baselines are strong | CRISPR-specific; ours is general |
| [Squidiff (NM 2026)](https://doi.org/10.1038/s41592-025-xxxxx) | DiffAE baseline to beat | We directly outperform it in P0 |
| [SCALE (arXiv 2026)](https://arxiv.org/abs/2603.xxxxx) | Scale enables DL advantage | 184M + Tahoe-100M; we use 0.76M + 90k cells |
| [PerturbDiff (arXiv 2026)](https://arxiv.org/abs/xxxx) | Theoretical basis for distributional loss (RKHS) | RKHS diffusion; we use pragmatic CFM+MMD |
| [Unlasting (arXiv 2025)](https://arxiv.org/abs/2506.21107) | E-distance evaluation precedent | Dual DDIB; we use single flow |
| [MolFormer (NMI 2022)](https://doi.org/10.1038/s42256-022-00580-7) | sci-Plex drug conditioning | We use it as a frozen feature extractor |
