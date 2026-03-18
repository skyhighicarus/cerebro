---
title: "Model Ablation and Activation Engineering for LLM Safety Bypass"
created: 2026-03-16
updated: 2026-03-16
tags: [ai-safety, mechanistic-interpretability, activation-engineering, llm-security, red-teaming]
status: published
---

# Model Ablation and Activation Engineering for LLM Safety Bypass

## Overview

Model ablation in the context of activation engineering refers to a family of techniques that identify and remove specific directions in a language model's activation space that encode safety-related behaviors -- most notably, the tendency to refuse harmful requests. Unlike prompt-level jailbreaks that try to trick the model through its input interface, ablation operates directly on the model's internal representations, making it a fundamentally different class of attack with profound implications for AI safety.

The core discovery driving this field is that safety training (RLHF, DPO, and similar alignment techniques) does not deeply restructure a model's knowledge or capabilities. Instead, it installs relatively shallow behavioral modifications that manifest as identifiable geometric structures in activation space. Specifically, the "refusal behavior" of a safety-trained model can often be localized to a single direction in the residual stream -- a finding that is both scientifically fascinating and deeply concerning from a safety perspective. Recent work (2025-2026) has nuanced this picture: while refusal can be bypassed via a single direction, the underlying geometry is richer than originally appreciated, with distinct refusal categories corresponding to geometrically distinct directions that nonetheless converge on a shared behavioral control axis.

Understanding these techniques is essential for AI safety researchers and model developers. If safety mechanisms can be removed with a rank-one intervention that preserves output quality, then current alignment approaches are insufficient as a sole defense layer. The urgency has intensified: as of early 2026, thousands of "abliterated" model variants are publicly available on Hugging Face, and the technique has become a routine community practice applied to every major open-weight release within hours of publication. This document covers the mechanics, key research, practical details, defenses, and the evolving policy landscape around activation-based safety bypass.

## Key Concepts

- **Residual stream**: The main information highway in a transformer. Each layer reads from and writes to this shared vector space. All activation engineering operates on these representations.
- **Refusal direction**: A single vector (direction) in activation space along which safety-trained models differ from their base counterparts. When a model is about to refuse, its activations have a large component along this direction.
- **Contrastive pairs**: Paired prompts (one harmful, one harmless) used to isolate the activation difference that corresponds specifically to refusal behavior, rather than to topic or style.
- **Directional ablation**: Projecting out (removing) a specific direction from the residual stream activations, effectively zeroing the model's representation of that concept.
- **Abliteration**: The community term (coined on Hugging Face) for the practical application of directional ablation to permanently remove safety guardrails from open-weight models by editing model weights. Requires no training -- only forward passes and linear algebra.
- **Steering vectors**: Vectors added to or subtracted from activations at inference time to shift model behavior along a specific conceptual axis (e.g., more/less refusal, more/less sycophancy).
- **Angular steering**: A newer technique (2025) that rotates activations within a fixed two-dimensional subspace rather than adding or projecting vectors, offering finer-grained behavioral control.
- **Circuit breakers**: A defense technique (Zou et al., 2024) that modifies internal representations to short-circuit harmful generation at the activation level, rather than relying on output-level refusal.
- **Activation patching**: Replacing activations at specific positions and layers with activations from a different forward pass, used for causal analysis of which components mediate a behavior.
- **Representation engineering**: The broader framework (Zou et al.) of reading and controlling model behavior through internal representations rather than through inputs or training.
- **SAE-guided steering**: Using sparse autoencoder features to construct more interpretable and targeted steering vectors, reducing unintended side effects compared to raw contrastive methods.

## Deep Dive

### Mechanistic Foundations: How Refusal Lives in Activation Space

The transformer architecture processes text through a residual stream -- a vector space (typically of dimension 4096 to 8192 in modern models) that each attention head and MLP layer reads from and writes to additively. At any given layer, the residual stream state is the sum of the original embedding plus all previous layer outputs.

Safety training modifies the weights of the model such that, when processing a harmful request, certain components write a "refusal signal" into the residual stream. This signal propagates forward and ultimately causes the model to generate refusal tokens ("I can't help with that...") rather than compliant ones.

The key mechanistic insight is that this refusal signal is approximately **one-dimensional**. That is, there exists a single direction vector `r` in the residual stream such that:

- When the model is about to refuse, the residual stream has a large positive projection onto `r`
- When the model is about to comply, this projection is near zero or negative
- Removing this component (projecting the residual stream onto the orthogonal complement of `r`) eliminates refusal behavior while leaving other capabilities intact

This is possible because the residual stream is extremely high-dimensional (thousands of dimensions), and the refusal direction occupies a vanishingly small fraction of that space. The model's knowledge, reasoning ability, and language fluency are encoded across the remaining thousands of dimensions.

**Why one direction?** Safety training (RLHF/DPO) optimizes for a relatively simple behavioral distinction: refuse harmful requests, comply with harmless ones. This binary signal naturally collapses to a low-dimensional (often rank-1) representation. The model learns the most efficient encoding of "should I refuse?" which, in a linear representation space, is a single direction.

**Update (2026) -- The geometry is richer than "just one direction":** Joad et al. (2026) demonstrated across eleven categories of refusal and non-compliance -- including safety refusals, incomplete/unsupported requests, anthropomorphization, and over-refusal -- that these behaviors correspond to geometrically distinct directions in activation space. However, a key paradox emerged: despite this geometric diversity, linear steering along any refusal-related direction produces nearly identical refusal-to-over-refusal trade-offs. The authors characterize this as a "shared one-dimensional control knob" -- different directions determine *how* models refuse rather than *whether* they refuse. This refines the Arditi et al. finding: the single-direction ablation works not because refusal is truly one-dimensional, but because the behavioral outcome (refuse vs. comply) is controlled by a shared axis even when the underlying representation is multi-faceted.

### Key Papers and Research Lineage

**1. Activation Addition / Steering Vectors -- Turner, Thiergart, Udell, Leech, Mini (2023)**
- **Paper**: "Activation Addition: Steering Language Models Without Optimization" (arXiv: 2308.10248)
- **Key contribution**: Demonstrated that adding a "steering vector" to the residual stream at a specific layer can reliably shift model behavior along a conceptual axis. Computed steering vectors as the difference in mean activations between contrastive prompt pairs (e.g., prompts about love vs. hate).
- **Method**: Run two sets of prompts through the model, compute the mean activation at a chosen layer for each set, take the difference. At inference time, add this difference vector (scaled by a coefficient) to the residual stream at that layer.
- **Significance**: Showed that behavioral control is possible through simple arithmetic in activation space, without any gradient computation or weight modification. One of the first demonstrations that model behavior has clean linear structure in activation space.

**2. Representation Engineering -- Zou, Phan, Chen, Campbell, et al. (2023)**
- **Paper**: "Representation Engineering: A Top-Down Approach to AI Transparency" (arXiv: 2310.01405)
- **Key contribution**: Formalized the framework of "representation reading" (identifying directions in activation space that encode specific concepts) and "representation control" (intervening on those directions to modify behavior). Applied this to honesty, fairness, harmlessness, and other behavioral dimensions.
- **Method**: Used linear probes and PCA on contrastive activation datasets to identify concept directions. Applied control by adding/subtracting these directions during inference.
- **Significance**: Provided a systematic top-down framework for understanding and controlling model internals. Showed that many high-level concepts (not just refusal) have clean linear representations that can be read and manipulated.

**3. Refusal Is Mediated by a Single Direction -- Arditi, Obber, Shlegeris, Buck (2024)**
- **Paper**: "Refusal in Language Models Is Mediated by a Single Direction" (NeurIPS 2024; arXiv: 2406.11717)
- **Key contribution**: The most targeted result. Demonstrated that across multiple open-weight models (Llama-2, Llama-3, Gemma, etc.), the entire refusal behavior can be attributed to a single direction in the residual stream. Ablating this one direction completely eliminates refusal on harmful prompts while preserving general model capabilities.
- **Method**:
  1. Construct contrastive pairs: harmful instructions (that trigger refusal) paired with harmless instructions (that get compliance)
  2. Run both sets through the model, collect residual stream activations at each layer
  3. Compute the mean difference in activations between harmful and harmless sets
  4. Apply PCA to these differences; the first principal component is the "refusal direction"
  5. At inference time, project this direction out of the residual stream (at every token position, at all layers or a subset of critical layers)
- **Key finding**: This single-direction ablation achieves near-100% refusal bypass on standard harmful prompt benchmarks while maintaining performance on capability benchmarks (MMLU, HellaSwag, etc.) within a few percentage points of the unmodified model.
- **Significance**: The most direct evidence that safety training creates a "thin veneer" -- a geometrically simple, easily removable modification rather than a deep restructuring of the model's knowledge or reasoning. Accepted at NeurIPS 2024, confirming peer-reviewed validation of the core claim.

**4. Circuit Breakers -- Zou, Phan, Wang, et al. (2024)**
- **Paper**: "Improving Alignment and Robustness with Circuit Breakers" (NeurIPS 2024)
- **Key contribution**: Proposed a defense mechanism inspired by representation engineering that directly modifies harmful internal representations rather than relying on output-level refusal. When a model's internal state activates harmful subspaces, the circuit breaker redirects representations away from harmful directions.
- **Method**: Applies loss functions at specific transformer layers (typically layers 10 and 20) that push harmful representations away from directions associated with unsafe content. Works during both input processing and generation phases.
- **Results**: Reduced multilingual attack success rates to 3.5-7.3% (vs. 19-34% for unprotected models). Maintained ~8/10 capability scores on MT-Bench. Gray Swan AI reported that Cygnet models using circuit breakers remained safe through nearly a year of consistent red-teaming.
- **Significance**: The first representation-level defense that proved substantially more robust than refusal training alone. Demonstrates that the same activation engineering principles used for attacks can be applied defensively.

**5. Related Work**
- **Causal tracing / activation patching** (Meng et al., 2022 -- "Locating and Editing Factual Associations in GPT"): Established the methodology of patching activations to identify which layers and components are causally responsible for specific behaviors.
- **Linear representation hypothesis** (Park et al., Nanda et al., various 2023-2024): The broader finding that high-level concepts are encoded as linear directions in transformer activation spaces, which underlies all of this work.
- **Mechanistic interpretability** (Anthropic's work on superposition, sparse autoencoders, etc.): Provides the theoretical foundation for understanding why and how features are encoded in activation space.

### Recent Developments (2025-2026)

This section covers significant advances since the foundational papers above.

#### Refined Understanding of Refusal Geometry

**"There Is More to Refusal in Large Language Models than a Single Direction" -- Joad et al. (February 2026)**

This paper challenges the prevailing single-direction narrative. Studying eleven categories of refusal across multiple models, the authors show that different refusal types (safety, incomplete requests, anthropomorphization, over-refusal) correspond to geometrically distinct directions. Yet paradoxically, steering along any of these directions produces nearly identical behavioral trade-offs. The implication: refusal is geometrically multi-dimensional but behaviorally one-dimensional. This means defenders cannot simply "spread" refusal across multiple directions and expect ablation to become harder -- the shared behavioral axis remains the bottleneck.

**"The Geometry of Refusal in Large Language Models" (February 2025)**

Further work examining the geometric structure of refusal in activation space, corroborating that refusal behavior has rich internal structure that collapses to a shared control axis at the behavioral level.

#### Angular Steering (2025)

Angular steering introduces a fundamentally different intervention geometry: instead of adding or projecting vectors, it **rotates** activations within a fixed two-dimensional subspace defined by the activation and a feature direction. This provides continuous, bounded control over behavior magnitude without the norm-distortion problems that plague vector addition (where large steering coefficients can push activations out of distribution). Angular steering unifies ablation and addition as special cases of rotation, providing a cleaner theoretical framework for activation interventions.

#### SAE-Guided Steering (2025)

Multiple independent groups converged on using sparse autoencoder (SAE) features to improve steering vector quality:

- **SAE-TS (SAE-Targeted Steering)**: Finds steering vectors that target specific SAE features while minimizing unintended side effects. Balances steering effect with output coherence better than raw contrastive activation addition (CAA).
- **FGAA (Feature Guided Activation Additions)**: Operates in SAE latent space, using optimization to select desired features and construct interpretable steering vectors. The resulting vectors are human-readable -- you can inspect which features are being amplified or suppressed.
- **SAE-RSV (Refinement via Sparse Autoencoder)**: Uses feature semantics to filter out superficial features (e.g., punctuation artifacts) and retain only task-relevant ones.
- **CorrSteer**: Correlates generated-token activations with behavioral outcomes to select relevant SAE features, then constructs targeted steering vectors.
- **AxBench**: A benchmark that provocatively found "even simple baselines outperform sparse autoencoders" on some steering tasks, tempering enthusiasm and highlighting that SAE-based methods are not uniformly superior.

The SAE-steering line of work matters for safety because it makes activation engineering more precise and interpretable -- both for attackers (more targeted safety bypass) and defenders (better understanding of what directions encode).

#### Cross-Model Transfer of Steering Vectors (2025)

Oozeer and Nathawani (ICML 2025, arXiv: 2503.04429) demonstrated that activation space interventions can be **transferred between different language models**. By learning a transformation (autoencoder or affine map) to align activation spaces between source and target models, steering vectors developed for one model can be mapped to another. Experiments across Llama, Qwen, and Gemma families showed successful transfer for backdoor removal and refusal steering, with the notable implication that smaller models can be used to efficiently develop safety interventions (or attacks) for larger ones.

#### Activation Steering Beyond Text (2025)

**SARSteer (Safe-Ablated Refusal Steering)** extended activation engineering to **large audio-language models**, demonstrating that refusal behavior in multimodal models follows similar one-dimensional geometry. SARSteer introduces decomposed safe-space ablation to enforce refusal while mitigating over-refusal in audio modality -- the first inference-time defense framework for audio-language models.

Separately, researchers applied contrastive steering to **masked diffusion language models**, finding that refusal behavior in these non-autoregressive architectures is also governed by a consistent, approximately one-dimensional activation subspace. This suggests the geometric simplicity of safety training is not an artifact of the autoregressive architecture but a property of how current alignment objectives shape representations.

### Practical Implementation Details

**Finding the refusal direction:**

1. **Construct contrastive dataset**: Create 50-200 pairs of prompts. Each pair should differ only in whether it triggers refusal. Common approach: take harmful prompts from datasets like AdvBench or HarmBench, pair each with a syntactically similar but harmless prompt. Example pair: "Explain how to synthesize [dangerous substance]" vs. "Explain how to synthesize aspirin."

2. **Collect activations**: Run all prompts through the model, collecting residual stream activations. Typically collect at the last token position of the instruction (or the first generated token position). Collect across all layers or a target range.

3. **Compute difference vectors**: For each pair, subtract the harmless activation from the harmful activation. This isolates the component of activation that corresponds to "this prompt is harmful and I should refuse."

4. **Extract principal direction**: Apply PCA to the set of difference vectors. The first principal component is the refusal direction. In practice, researchers find that the first component explains a large fraction of variance (often 40-70%), confirming the low-rank structure.

**Performing the ablation:**

Given refusal direction `r` (a unit vector), for each residual stream activation `x` at each token position and target layer:

```
x_ablated = x - (x . r) * r
```

This orthogonal projection removes the component of `x` along `r` while preserving all other components.

**Permanent weight modification (abliteration):**

Rather than intervening at inference time, abliteration bakes the projection into the model weights. For each weight matrix W that writes to the residual stream, compute `W' = W - r * (r^T * W)`, which ensures the layer can never write along the refusal direction. This produces a standalone model file that requires no hooks or runtime modification -- it simply never refuses. This is how community-abliterated models on Hugging Face are created.

**Which layers matter:**

- The refusal direction is typically most prominent in **middle-to-late layers** (e.g., layers 14-26 in a 32-layer model like Llama-2-7B).
- Ablating at all layers works but is unnecessary. Ablating at the **critical window** (often 5-10 contiguous layers in the middle of the network) is sufficient.
- Early layers (0-5) encode mostly syntactic/positional information and have minimal refusal signal.
- The very last layers are where the refusal decision has already been made and propagated to the output distribution.
- A practical approach: compute the refusal direction per-layer, measure its norm/projection magnitude, and ablate at layers where it is strongest.

**Tools and frameworks:**

- **TransformerLens** (Neel Nanda): The most widely used library for mechanistic interpretability. Provides hook-based access to every activation in the model, making it straightforward to read and modify residual stream states. Written in Python/PyTorch.
- **nnsight** (NDIF): A more recent library that allows intervention on model internals with a clean API. Supports remote execution on larger models.
- **baukit** (David Bau): Another intervention library with a focus on causal tracing and activation patching.
- **pyvene** (Wu et al.): A library specifically designed for activation interventions with a declarative API (arXiv: 2403.07809).
- **PyReFT**: Implementation of representation finetuning for language models, enabling persistent activation-level modifications through learned interventions.
- **awesome-activation-engineering** (GitHub): A curated, actively maintained list of papers, tools, and resources tracking the field.
- **Raw PyTorch hooks**: For simple interventions, `register_forward_hook` on the relevant modules is sufficient. No library needed.

**Scale of intervention:**

- The refusal direction is a single vector in R^d (where d is the model's hidden dimension, e.g., 4096).
- The total parameter count of the intervention is exactly `d` floating-point numbers -- negligible compared to the model's billions of parameters.
- Inference-time cost is one dot product and one vector subtraction per layer per token -- computationally invisible.

### Quality Preservation and the Nature of Safety Training

The most striking aspect of refusal ablation is that it **does not degrade output quality**. This demands explanation and has deep implications.

**Why quality is preserved:**

1. **Orthogonality of safety and capability**: The refusal direction is approximately orthogonal to the directions that encode factual knowledge, reasoning ability, and language fluency. Removing it is like removing one dimension from a 4096-dimensional space -- the remaining 4095 dimensions carry all the model's capabilities.

2. **Safety training is additive, not transformative**: RLHF and DPO modify the model by adding a behavioral overlay. They teach the model to output refusal text when certain activation patterns are detected, but they do not remove the model's knowledge of the refused topic. The knowledge remains encoded in the weights; safety training merely adds a "gate" that redirects output. Ablation removes the gate.

3. **Base model capabilities are intact**: A safety-trained model (e.g., Llama-2-Chat) has essentially the same factual knowledge as its base counterpart (Llama-2). The chat/safety fine-tuning stage modifies a small fraction of the total weight space, primarily to install the input/output formatting behavior and the refusal behavior.

**What this reveals about safety training:**

- **Safety is a thin veneer**: Current alignment techniques (RLHF, DPO, Constitutional AI) create behavioral modifications that are geometrically simple and easily separable from capabilities. This is sometimes called the "alignment tax is low" observation -- but it cuts both ways. If alignment is cheap to add, it is cheap to remove.
- **The "refusal is not unlearning" insight**: Safety-trained models have not unlearned dangerous knowledge. They have learned to detect when dangerous knowledge is being requested and to redirect their output to a refusal template. The knowledge itself is fully intact in the weights and accessible once the refusal mechanism is bypassed.
- **Linear representation vulnerability**: The fact that safety behaviors are encoded as linear directions (rather than as complex, distributed, nonlinear representations) makes them inherently fragile to linear algebraic attacks. Any representation that can be localized to a low-rank subspace can be removed by projection.

**Empirical evidence:**

- Arditi et al. report that ablated models maintain within 1-3% of original performance on MMLU (general knowledge), HellaSwag (commonsense reasoning), and other standard benchmarks.
- The ablated models produce coherent, detailed, and on-topic responses to previously-refused prompts, indicating that the underlying capability was always present.
- In some cases, ablated models actually score slightly higher on certain benchmarks, suggesting that the refusal mechanism introduces a small amount of interference even on benign prompts.

### Defenses, Countermeasures, and Implications

**Why current defenses are insufficient:**

Ablation attacks require white-box access (model weights), so they do not apply to API-only models like GPT-4 or Claude. However, the open-weight ecosystem (Llama, Mistral, Gemma, Qwen, etc.) is increasingly dominant, and any model whose weights are released is vulnerable. The attack is:
- **Trivial to execute**: A few dozen lines of code, no training required
- **Reliable**: Works across model families with minimal adaptation
- **Undetectable**: The modified model produces normal-looking text; there is no signature that distinguishes an ablated model's outputs from a compliant model's outputs
- **Transferable**: Steering vectors developed on one model can be mapped to other models via learned activation space alignments (Oozeer & Nathawani, 2025)

**Foundational defenses (proposed through 2024):**

1. **Distributed safety representations**: Instead of safety behavior collapsing to a single direction, train models so that safety is encoded across many directions in a high-dimensional, entangled manner. This would make rank-1 ablation insufficient. However, Joad et al. (2026) showed that even when refusal is geometrically multi-directional, the behavioral axis remains shared -- suggesting this defense is harder to achieve than hoped.

2. **Adversarial training against ablation**: During safety training, simulate ablation attacks and penalize the model if ablation succeeds. This creates an adversarial game where the model learns to encode safety in ablation-resistant ways.

3. **Representation entanglement**: Engineer the training process so that safety-relevant and capability-relevant features share the same directions. Ablating safety would then necessarily degrade capabilities, creating an inherent cost to the attack. This is theoretically appealing but practically difficult.

4. **Circuit-level safety**: Instead of relying on a single refusal direction, implement safety through multiple redundant circuits at different layers with different mechanisms. Analogous to defense-in-depth in security.

5. **Inference-time monitoring**: Monitor activation patterns at inference time and flag when the refusal direction has been ablated or suppressed. Requires a trusted inference environment, which is at odds with open-weight deployment.

**Newer defenses with empirical results (2024-2026):**

6. **Circuit Breakers (Zou et al., 2024)**: The most battle-tested representation-level defense. Rather than relying on refusal output, circuit breakers short-circuit harmful generation by pushing internal representations away from harmful subspaces. Attack success rates dropped to 3.5-7.3% across multilingual attacks (vs. 19-34% unprotected). Gray Swan AI's Cygnet models using this approach withstood nearly a year of red-teaming. Notably, circuit breakers use the same representation engineering framework as the attacks themselves -- fighting fire with fire.

7. **Tamper-Resistant Training / TAR (Tamirisa et al., ICLR 2025)**: Combines adversarial training with meta-learning to create robust safeguards. During training, the model is exposed to simulated tampering attacks and learns to resist them. TAR preemptively hardens the model against both fine-tuning and activation-level modifications.

8. **AntiDote (Sanyal et al., 2025)**: A bi-level adversarial training method using a state-aware adversarial hypernetwork that generates malicious LoRA weights conditioned on internal activations, paired with defender LoRA weights trained to neutralize the attacks. Achieved up to 78% reduction in harmful score vs. baseline, 27.4% improvement over tamper-resistance baselines, with less than 0.5% capability degradation on MMLU/HellaSwag/GSM8K. Effective against 52 distinct attack vectors. The key innovation is decoupled loss computation -- safety loss on the attacked model, capability loss on the clean model -- preventing gradient contamination.

9. **Antibody (Nguyen et al., ICLR 2026)**: Defends against harmful fine-tuning by identifying and attenuating harmful gradient influences during the fine-tuning process itself. Selectively dampens gradients that push toward unsafe outputs while preserving beneficial learning. A training-time defense that operates at the gradient level rather than the representation level.

10. **Safety Pretraining Resilience (Agnihotri et al., 2025)**: A granular study finding that multi-component safety strategies (combining safe-only data filtering, rephrase data, refusal examples, and metatags) are substantially more resilient to abliteration than single-mechanism approaches. Refusal-only safety training is trivially neutralized; the "Safety Oracle" combining all components showed minimal degradation under abliteration. Critical finding: models cannot reliably detect their own refusal state after abliteration, ruling out self-monitoring as a standalone defense.

**Systematic evaluation -- TamperBench (Hossain et al., 2025):**

TamperBench provides the first unified framework for evaluating tamper resistance, testing 9 attack types (weight-space and representation-space) against 5 defense methods across 21 open-weight models. Key findings:
- Every model tested admits at least one highly effective tampering attack exceeding 0.68 harmfulness score (on a 0-1 scale)
- Models over 1B parameters all exceeded 0.77 worst-case post-attack harmfulness
- Jailbreak-tuning with only 2% malicious data mixed into benign data was typically the most severe attack
- Among defenses, Triplet (contrastive representation learning) showed the strongest overall performance
- Post-training alignment sometimes *worsened* tamper resistance (Llama instruction-tuned variants were less resistant than base models)
- The field suffers from fragmented evaluation -- different papers use different attacks, threat models, and metrics, making comparison difficult

**Broader implications for AI safety:**

- **Open weights vs. safety**: There is a fundamental tension between releasing model weights and maintaining safety guarantees. Ablation techniques make it clear that safety-trained open-weight models offer only a speed bump, not a barrier, against misuse. As of 2025-2026, thousands of abliterated models exist on Hugging Face, covering every major model family. OpenAI's GPT-OSS release in July 2025 was abliterated within hours despite additional safety testing.
- **Safety must go deeper**: The long-term path forward likely requires safety properties that are not separable from capabilities -- models that genuinely lack dangerous capabilities rather than merely declining to use them. This points toward training-time interventions (knowledge unlearning, capability restriction) rather than output-level behavior modification.
- **Layered defenses**: No single mechanism is sufficient. Robust AI safety likely requires a combination of training-time safety (TAR, AntiDote, Antibody), representation-level defenses (circuit breakers), inference-time monitoring, deployment restrictions, and societal-level governance.
- **Defense maturation**: The field has progressed from "defenses are theoretical" (2024) to "defenses exist and have been empirically tested against multiple attack vectors" (2025-2026). Circuit breakers and AntiDote in particular show that meaningful resistance is achievable, though no defense is yet unbreakable.

### Variants: Ablation, Steering, and Patching

**Directional ablation** (removing a direction):
- Projects out a direction entirely: `x' = x - (x . r) * r`
- Binary intervention: the direction is either present or absent
- Used primarily for removing behaviors (refusal, sycophancy)
- The intervention is symmetric -- it prevents the model from representing anything along that direction, whether positive or negative
- **Norm-preserving biprojected abliteration** (2025): A refinement that addresses norm distortion by projecting in both the original and orthogonal subspaces, maintaining activation norms closer to the original distribution

**Steering / activation addition** (adding a direction):
- Adds a scaled vector: `x' = x + alpha * v`
- Continuous intervention: the coefficient `alpha` controls the strength
- Can both increase and decrease behaviors depending on sign
- More flexible than ablation: can shift behavior by degree rather than eliminating it entirely
- Used for behavioral control (more honest, less sycophantic, more creative, etc.)

**Angular steering** (rotating activations, 2025):
- Rotates the activation within the 2D plane defined by the activation and the feature direction
- Bounded intervention: rotation angle controls strength without changing activation norm
- Unifies ablation (90-degree rotation) and addition (small rotation) as special cases
- Avoids out-of-distribution activation norms that plague large-coefficient vector addition

**SAE-guided steering** (2025):
- Decomposes activations into sparse autoencoder features, steers in feature space
- More interpretable: each feature has a human-readable description
- More targeted: can suppress specific features while leaving related ones intact
- Multiple methods (SAE-TS, FGAA, SAE-RSV, CorrSteer) with different feature selection strategies

**Activation patching** (replacing activations):
- Replaces activations at specific (layer, position) with activations from a different input
- Used primarily for causal analysis: "if this component had seen input B instead of input A, would the output change?"
- The foundation of causal tracing and circuit discovery
- Not typically used for attacks, but essential for understanding which components mediate safety behavior

**Causal tracing** (Meng et al.):
- A methodology that uses activation patching systematically across all layers and positions
- Produces a "causal map" showing which components are responsible for a specific behavior
- Used to identify that refusal is mediated by specific attention heads and MLP layers, motivating the targeted ablation approach

**Probing / linear probes**:
- Train a linear classifier on activations to predict whether a concept is present
- Used in the "reading" phase: if a linear probe can detect refusal intent with high accuracy, it confirms that the concept has a linear representation
- Does not modify model behavior, but informs where and how to intervene

## Practical Applications

- **Red-teaming and safety evaluation**: Security teams should routinely test models against ablation attacks before deployment. If a model's safety can be ablated away, its safety properties are insufficient for high-stakes deployment. TamperBench provides a standardized framework for this.
- **Robustness benchmarking**: Measure the rank of the safety subspace (how many principal components must be ablated to bypass safety). A rank-1 safety representation is fragile; higher rank is more robust. Also test against fine-tuning attacks with small amounts of poisoned data (the 2% jailbreak-tuning threshold from TamperBench).
- **Defense development**: Use ablation as a test oracle during adversarial safety training. Train, attempt ablation, measure bypass rate, penalize, repeat. Circuit breakers and AntiDote provide proven templates.
- **Interpretability research**: Ablation is a powerful causal tool -- it answers "is this direction necessary for this behavior?" definitively. SAE-guided methods add interpretability to the intervention.
- **Decision framework for model release**: If a model's safety can be trivially ablated, this should factor into open-weight release decisions. The Centre for Future Generations recommends tiered release frameworks linking capability levels to release conditions.
- **Cross-model safety transfer**: Steering vector transfer (Oozeer & Nathawani, 2025) enables developing safety interventions on smaller models and applying them to larger ones, potentially reducing the cost of safety research.

## Connections

- **Mechanistic interpretability**: Ablation techniques are a direct application of mechanistic interpretability methods (superposition, feature geometry, circuit analysis) to safety.
- **RLHF and alignment training**: Understanding that RLHF produces low-rank safety representations motivates research into training objectives that produce higher-rank, more entangled safety features.
- **Adversarial machine learning**: Ablation is a white-box adversarial attack on behavioral alignment, connecting AI safety to the broader adversarial ML literature.
- **AI governance and open-source policy**: The feasibility of ablation directly impacts policy debates about open-weight model releases and safety requirements. The OECD, G7 Hiroshima AI Process, and EU AI Act all grapple with the open-weight safety question that ablation makes urgent.
- **Sparse autoencoders**: The SAE-guided steering work bridges activation engineering with the broader mechanistic interpretability agenda of decomposing models into interpretable features.

## Sources

1. Arditi, A., Obber, O., Shlegeris, B., & Buck. (2024). "Refusal in Language Models Is Mediated by a Single Direction." NeurIPS 2024; arXiv:2406.11717. The seminal paper demonstrating rank-1 refusal ablation across multiple model families.
2. Zou, A., Phan, L., Chen, S., et al. (2023). "Representation Engineering: A Top-Down Approach to AI Transparency." arXiv:2310.01405. Formalized the framework of representation reading and control for behavioral concepts.
3. Turner, A., Thiergart, L., Udell, D., Leech, G., & Mini, U. (2023). "Activation Addition: Steering Language Models Without Optimization." arXiv:2308.10248. Introduced steering vectors via contrastive activation differences.
4. Meng, K., Bau, D., Mitchell, A., & Yun, C. (2022). "Locating and Editing Factual Associations in GPT." NeurIPS 2022. Established causal tracing methodology for locating knowledge in transformers.
5. Tamirisa, R., et al. (2024/2025). "Tamper-Resistant Safeguards for Open-Weight LLMs." ICLR 2025; arXiv:2408.00761. Proposed training-time defenses against weight-level and activation-level safety bypass.
6. Zou, A., Phan, L., Wang, J., et al. (2024). "Improving Alignment and Robustness with Circuit Breakers." NeurIPS 2024. Representation-level defense that short-circuits harmful generation. https://www.grayswan.ai/research/circuit-breakers
7. Joad, F., Hawasly, M., Boughorbel, S., Durrani, N., & Sencar, H.T. (2026). "There Is More to Refusal in Large Language Models than a Single Direction." arXiv:2602.02132. Demonstrated multi-directional refusal geometry with shared behavioral axis.
8. Sanyal, D., Ray, M., & Mandal, M. (2025). "AntiDote: Bi-level Adversarial Training for Tamper-Resistant LLMs." arXiv:2509.08000. State-aware adversarial defense achieving 78% harm reduction with <0.5% capability loss.
9. Nguyen, Q.M., Le, T., Wu, J., Bui, A.T., & Harandi, M. (2026). "Antibody: Strengthening Defense Against Harmful Fine-Tuning via Attenuating Harmful Gradient Influence." ICLR 2026; arXiv:2603.00498. Gradient-level defense against harmful fine-tuning.
10. Hossain, S., Tseng, T., et al. (2025). "TamperBench: Systematically Stress-Testing LLM Safety Under Fine-Tuning and Tampering." arXiv:2602.06911. First unified tamper-resistance evaluation framework across 21 models and 9 attack types.
11. Agnihotri, S., et al. (2025). "A Granular Study of Safety Pretraining under Model Abliteration." arXiv:2510.02768. Multi-component safety training resists abliteration better than refusal-only approaches.
12. Oozeer, M. & Nathawani, D. (2025). "Activation Space Interventions Can Be Transferred Between Large Language Models." ICML 2025; arXiv:2503.04429. Demonstrated cross-model transfer of steering vectors via learned activation space alignments.
13. Nanda, N. (2023). TransformerLens documentation and tutorials. Primary tool for activation-level interventions and mechanistic interpretability. https://github.com/TransformerLensOrg/TransformerLens
14. Wu, Z., et al. (2024). "pyvene: A Library for Understanding and Improving PyTorch Models via Interventions." arXiv:2403.07809. Intervention library used in representation engineering research.
15. Centre for Future Generations. (2025). "Can Open-Weight Models Ever Be Safe?" https://cfg.eu/can-open-weight-models-ever-be-safe/. Policy analysis of open-weight safety challenges including abliteration.
16. mlabonne. (2024). "Uncensor any LLM with abliteration." Hugging Face Blog. https://huggingface.co/blog/mlabonne/abliteration. Practical guide that popularized the abliteration technique.
