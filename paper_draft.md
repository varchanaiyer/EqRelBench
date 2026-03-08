# Accuracy Masks Inconsistency: Logical Self-Contradiction in LLMs on Abstract Relational Reasoning

---

## Abstract

Standard evaluations of logical reasoning in large language models (LLMs) measure per-question accuracy, treating each question as independent. We argue this approach is insufficient: a model can score well on individual questions while giving jointly contradictory answers when those questions are logically related. We introduce logical consistency as a complementary metric and evaluate Phi-3-mini-4k-instruct on three fundamental properties of binary relations, reflexivity, symmetry, and transitivity, using synthetic datasets of randomly-generated entity names to eliminate memorization as a confound. Three findings emerge. First, despite 73.1% per-question accuracy on transitivity questions, joint consistency across logically equivalent question variants collapses to 49.6%, a gap of 23.5 percentage points that suggests per-question accuracy substantially overstates genuine deductive ability. Second, the model answers the simplest possible logical question, "Does X equal X?", correctly only 32.3% of the time, revealing a systematic bias against self-identity that standard benchmarks do not surface. Third, depth-scaling experiments on transitive equality chains expose a qualitative behavioral shift: the model maintains a strong "no" bias at chain depth 2 (yes-rate 23%) but transitions to near-total "yes" bias by depth 10 (yes-rate 94%), abandoning chain verification in favor of a length-based heuristic. These results indicate that LLMs exploit surface-level features of individual prompts rather than applying consistent logical rules.

---

## 1. Introduction

Evaluating the logical reasoning capabilities of LLMs has become one of the central challenges in natural language processing research. Models are increasingly deployed in settings that require reliable deduction, from legal document analysis to scientific reasoning pipelines, yet the benchmarks used to assess their reasoning typically measure accuracy at the level of individual questions. This evaluation paradigm conceals an important failure mode: a model that gives the right answer to each question in isolation may simultaneously give answers that are logically incompatible when those questions are considered together.

Consider a concrete example. A model is asked "Suppose ABLM equals KFQZ. Does KFQZ equal ABLM?" and answers "yes." When asked the logically identical question with the syntactic order reversed, "Suppose KFQZ equals ABLM. Does ABLM equal KFQZ?", it answers "no." Per-question accuracy over these two queries is 50%, but the more important observation is that the model does not hold a consistent representation of the symmetry relation at all. A model that achieves high per-question accuracy through surface-level pattern matching rather than genuine rule application would behave exactly this way.

This paper studies logical consistency, defined as the degree to which a model's joint answers across logically related questions satisfy the constraints of the underlying logical property. We distinguish consistency from accuracy: a model can be consistent but systematically wrong (consistently saying "no" to all transitivity questions), or accurate-but-inconsistent (getting individual questions right by chance while giving contradictory joint answers). The latter is the failure mode we are primarily concerned with, since it means the model cannot be trusted to reason reliably even when its per-question accuracy looks acceptable.

We evaluate Phi-3-mini-4k-instruct on three foundational properties of binary relations: reflexivity (for all x, x R x), symmetry (if x R y then y R x), and transitivity (if x R y and y R z then x R z). Rather than using real-world entities, we generate datasets using random uppercase strings of four to seven characters as entity names. This design choice follows a line of work arguing that named entities introduce memorization as a confound: when a model correctly answers questions about real people or places, it may be recalling training data rather than reasoning from premises (Mirzadeh et al., 2024; Saparov and He, 2023). Abstract random names force the model to operate purely from the logical structure of the input.

Our main findings are as follows.

**Finding 1.** For transitivity, average per-question accuracy is 73.1%, but joint logical consistency across equivalent question formulations is only 49.6%. This 23.5 percentage-point gap is the central empirical finding of this paper. It means that practitioners relying on per-question accuracy to evaluate transitive reasoning ability would substantially overestimate what the model can actually do.

**Finding 2.** The model correctly answers "Does X equal X?" only 32.3% of the time across 300 randomly-named entities. Since the correct answer is always "yes," the model is systematically denying self-identity. Reflexivity is arguably the simplest logical property expressible in natural language, and its failure here is not surfaced by any of the accuracy numbers that standard benchmarks report.

**Finding 3.** On transitive equality chains of increasing depth, the model undergoes a qualitative behavioral shift. At chain depth 2, it has a strong "no" bias (yes-rate 23%) driven by near-perfect accuracy on broken chains and poor accuracy on intact ones. By depth 10, this inverts completely to a near-total "yes" bias (yes-rate 94%), where the model correctly handles intact chains but fails on 88% of broken ones. The model is not performing chain verification; it is using premise list length as a proxy for chain validity.

Section 2 reviews related work. Section 3 describes the dataset construction and evaluation methodology. Section 4 presents results. Section 5 discusses implications for benchmarking and future directions.

---

## 2. Related Work

**Logical reasoning benchmarks.** A substantial body of recent work has developed benchmarks for evaluating multi-step logical reasoning in LLMs. Multi-LogiEval (Patel et al., 2024) covers propositional, first-order, and non-monotonic logic with reasoning depths from one to five, finding that accuracy drops from roughly 68% at depth 1 to 43% at depth 5 across models including GPT-4 and Gemini Pro. JustLogic (Chen et al., 2025) introduces a synthetic deductive benchmark designed to be knowledge-independent, with fine-grained stratification by reasoning depth and argument structure; even state-of-the-art reasoning models do not reach human ceiling performance. WorldSense (Benchekroun et al., 2023) tests transitive-order reasoning over abstract entities, finding that GPT-4 and LLaMA-2 make errors with as few as three objects in the ordering. Our work differs from these benchmarks in two respects: we focus on the three structural axioms of binary relations as distinct properties, and we introduce consistency as a first-class metric alongside accuracy rather than treating all questions as independent.

**Transitive and relational reasoning.** Mehrafarin et al. (2024) present a diagnostic study of transitive reasoning in LLaMA-2 and Flan-T5, carefully controlling for surface-level confounds such as word overlap and entity familiarity. They find that both models exploit textual overlap rather than performing genuine transitive inference, a conclusion that complements our finding that the model's behavior shifts qualitatively with chain length rather than degrading smoothly. Ni et al. (2024) introduce the Generalized Associative Recall benchmark for compositional relational reasoning and use circuit-level analysis on Vicuna-33B to identify attention heads that encode abstract truth values; their analysis finds that LLMs consistently fail when multiple relation types must be composed. Khalid et al. (2025) show that LLMs and recent reasoning models handle single-path relational reasoning but collapse on multi-path disjunctive problems, indicating that apparent relational competence does not generalize to settings requiring broader search over relation compositions.

**Consistency in LLM outputs.** Several recent papers have formalized different notions of consistency in LLM behavior. Liu et al. (2024) propose an evaluation framework based on three logical properties, transitivity, commutativity, and negation invariance, and measure consistency of LLM preference judgments, finding widespread violations. Ghosh et al. (2024) introduce consistency measures for fact-checking over knowledge graphs using propositional logic operators, demonstrating that existing LLMs fail badly on complex logical queries and that fine-tuning is required to produce consistent outputs. Novikova et al. (2025) survey the landscape of LLM consistency, classifying consistency types into negational, symmetric, transitive, and additive categories and identifying critical gaps in existing benchmark coverage. Our notion of logical consistency is closest to the transitive and symmetric categories in their taxonomy, but we apply it to relational axioms rather than factual or preference judgments. Ahn and Yin (2025) identify a related failure mode where models give contradictory evaluations when asked about the same answer candidates in different polarity framings, which they call prompt-reverse inconsistency. All of these works share the core motivation that per-output accuracy is insufficient for assessing reliability, but none examines the specific relational axioms we study or the interaction between chain depth and consistency.

**Synthetic datasets and memorization confounds.** The use of synthetic data with abstract entities to isolate genuine reasoning from memorization has a clear motivation in recent empirical work. GSM-Symbolic (Mirzadeh et al., 2024) generates math problems from symbolic templates with varied surface details, finding that all tested LLMs degrade when entity names or numbers change, and that adding a single irrelevant clause causes up to 65% performance drops. Saparov and He (2023) introduce PrOntoQA, a synthetic first-order logic dataset using fictional entity names, and find that LLMs can execute individual deduction steps but follow greedy proof strategies rather than systematically exploring valid proof paths. Morishita et al. (2024) show that training on a synthetic logic corpus with diverse rules and distractors yields large gains on reasoning benchmarks without any real-world data. These results collectively support the design choice of using random entity names in our datasets.

**Depth scaling and compositional generalization.** Dziri et al. (2023) study multi-step compositional tasks including logic grid puzzles by formalizing them as computation graphs, demonstrating that Transformers reduce compositional reasoning to subgraph matching and that error probability grows exponentially with graph depth. Opedal et al. (2025) generate arithmetic problems with precisely controlled proof depth and tree structure, finding consistent and significant performance degradation as depth increases and problems become non-linear. Saparov et al. (2023) construct a programmable deductive reasoning dataset and find that LLMs generalize across compositional proof structures but consistently fail to generalize to longer proofs. Our depth-scaling experiment extends this line of work to the setting of equality chain verification, where the failure mode is not simply degradation with depth but a qualitative inversion of output bias from "no" to "yes" as chain length grows.

---

## 3. Methodology

### 3.1 Entity Name Pool

All datasets are built from a pool of 50,000 randomly-generated uppercase strings between four and seven characters long, produced with a fixed random seed of 42 for reproducibility. No real-world names, places, or objects appear in any prompt. This follows the approach of Saparov and He (2023) and Mirzadeh et al. (2024), who demonstrate that real entity names introduce memorization as a confound. Using abstract strings ensures the model cannot answer correctly by recalling training data and must instead derive answers from the logical structure of the premises alone.

<!-- TODO [STUDENTS]: Add 2-3 example names from the pool here, e.g. "KFQZ, ABLM, MRPT" to give the reader a concrete sense of what the entities look like -->

### 3.2 Dataset Construction

**Experiment 1: Consistency evaluation.** For each of the three relational properties, we construct groups of logically related questions that a consistent model must answer in agreement.

For *symmetry*, we sample 300 random entity pairs (A, B) and form two questions per pair. Q_fwd asks "Suppose A equals B. Does B equal A?" and Q_rev asks "Suppose B equals A. Does A equal B?" Both present the same logical scenario with the surface ordering reversed. A consistent model must give the same answer to both; since the correct answer in both cases is "yes," per-question accuracy and consistency are independently measurable.

For *transitivity*, we sample 250 random entity triples (A, B, C) and form three question variants per triple, all with the same premises. Q_direct asks for the conclusion from A to C. Q_reorder presents the premises in reversed order (B=C before A=B) with the same conclusion. Q_reverse asks for the conclusion from C to A, which requires both transitivity and symmetry. We report pairwise consistency between Q_direct and Q_reorder (same logical content, different surface presentation) and between Q_direct and Q_reverse (requires an additional symmetry step).

For *reflexivity*, we sample 300 individual entity names and ask "Does X equal X?" for each. Since all correct answers are "yes," this measures whether the model uniformly applies the reflexivity axiom. We also construct a negative control by asking "Does A equal B?" for 300 distinct pairs with no premises, to confirm the model is not simply biased toward "yes" for any yes/no question.

**Experiment 2: Depth scaling.** We generate equality chains of depth D, where depth is the number of premises. A depth-D chain has D+1 entities (A_0 through A_D), with D premises "Suppose A_i equals A_{i+1}" and a final question "Does A_0 equal A_D?" Positive examples use all premises intact (correct answer: yes). Negative examples replace one randomly selected premise with "does not equal" (correct answer: no). We test depths 2, 3, 4, 5, 6, 7, 8, and 10, with 200 examples per depth split evenly between positive and negative. All entity names within a single chain are distinct.

<!-- TODO [STUDENTS]: Add a small illustrative example box showing a depth-3 positive chain and a depth-3 negative chain side by side, with the broken premise italicized or underlined. This makes the task concrete for readers unfamiliar with the setup. -->

### 3.3 Prompt Format

All questions use the following fixed template:

```
You are a logical reasoning assistant.
Answer with exactly one word.

{question_text}

Answer only yes or no:
```

<!-- TODO [STUDENTS]: If you ran any ablations on prompt wording, note them briefly here. Otherwise, one sentence stating this template was held fixed across all experiments is sufficient. -->

### 3.4 Prediction Method

We use logit-based yes/no prediction rather than free-form text generation. After a single forward pass on the tokenized prompt, we extract logits at the final token position and compare the logit for the token "yes" against the logit for "no." The prediction is "yes" if the yes logit is larger. This approach is deterministic, requires no generation hyperparameters, and avoids noise from sampling or greedy decoding of multi-token sequences. It has been used in prior probing and preference-elicitation work (Kadavath et al., 2022).

<!-- TODO [STUDENTS]: Report the exact token IDs for "yes" and "no" in Phi-3-mini's tokenizer for reproducibility. These are printed in the notebook output cell that loads the model. -->

### 3.5 Consistency Metric

For a group of logically related questions G, logical consistency is the proportion of groups in which all answers are jointly valid under the relevant axiom. For symmetry, a group is consistent if Q_fwd and Q_rev receive the same answer. For transitivity, full consistency requires agreement across all three variants. For reflexivity, since each entity's question is independent, we report the fraction of questions answered "yes" as the consistency score. Per-question accuracy is computed independently per question; the accuracy-consistency gap is their difference.

<!-- TODO [STUDENTS]: If you want to add a formal definition using notation, the consistency score for a property P over N groups can be written as: Cons_P = (1/N) * sum_{i=1}^{N} 1[answers to all questions in group i are jointly P-valid] -->

### 3.6 Model

We evaluate Phi-3-mini-4k-instruct (Microsoft, 2024), a 3.8B parameter instruction-tuned language model, loaded in FP16 precision with automatic device mapping across available hardware.

<!-- TODO [STUDENTS]: Fill in the GPU used (e.g. NVIDIA T4 on Google Colab), total wall-clock inference time for each experiment, and the exact HuggingFace model revision if available. -->

---

## 4. Experiments and Results

### 4.1 Experiment 1: Logical Consistency Across Related Questions

Table 1 summarizes per-question accuracy, logical consistency, and the gap between them for all three properties.

<!-- TODO [STUDENTS]: Insert Table 1 as a LaTeX table with caption "Accuracy, consistency, and gap for Phi-3-mini-4k-instruct on three relational properties."
     Columns: Property | N (examples) | Avg. Accuracy | Consistency | Gap
     Symmetry     | 300 pairs   | 0.568 | 0.657 | -0.088
     Transitivity | 250 triples | 0.731 | 0.496 | +0.235
     Reflexivity  | 300 singles | 0.323 | 0.323 |  0.000
-->

**IMAGE: Figure 1 — `experiment1_consistency_results.png`**
Place immediately after Table 1, full column width. Caption: "Per-question accuracy (blue) vs. logical consistency (orange) for Phi-3-mini-4k-instruct on three relational properties evaluated on randomly-named abstract entities. The annotated gap shows the difference between average accuracy and consistency for each property."

**Symmetry.** Average per-question accuracy is 56.8%, just above chance. Consistency is 65.7%, producing a negative gap of -0.088. A negative gap means the model is more consistent than accurate: when it is wrong, it tends to be wrong on both question variants rather than accidentally correct on one. This suggests the model holds a relatively stable internal response to symmetry questions.

<!-- TODO [STUDENTS]: Report the failure breakdown from experiment1_results.json (failure_breakdown field under symmetry). Specifically: what fraction of inconsistent pairs are "fwd_only" (yes on forward, no on reverse) vs. "rev_only" vs. "both_no"? This reveals whether errors are directional or symmetric. -->

**Transitivity.** Average per-question accuracy is 73.1%. Joint consistency across all three question variants is 49.6%, yielding a gap of +23.5 percentage points. This is the central finding of the paper. A practitioner reporting only per-question accuracy on a transitivity benchmark would conclude the model reasons well; the consistency score reveals the opposite.

<!-- TODO [STUDENTS]: From experiment1_results.json, report violation_rate_reorder and violation_rate_reverse. These are the fraction of triples where the model said "yes" to Q_direct but "no" to Q_reorder or Q_reverse respectively. Reporting these separately shows which question variant is harder. -->

**Reflexivity.** Per-question accuracy is 32.3%, meaning the model denies self-identity ("Does X equal X?") for 67.7% of abstract entity names. The negative control yes-rate (asking "Does A equal B?" with no premise) is reported below for comparison.

<!-- TODO [STUDENTS]: Insert the neg_control_yes_rate value from experiment1_results.json here. E.g. "The model's yes-rate for unpremised two-entity questions was X%, compared to 32.3% for self-identity questions, confirming that the reflexivity failure is not simply a global 'no' bias." -->

### 4.2 Experiment 2: Accuracy vs. Chain Depth

Table 2 reports overall accuracy, positive accuracy (intact chains), negative accuracy (broken chains), and the yes-output rate at each tested depth.

<!-- TODO [STUDENTS]: Insert Table 2 as a LaTeX table with caption "Depth-scaling results for Phi-3-mini-4k-instruct on transitive equality chain verification. Pos Acc: accuracy on intact chains (label=yes). Neg Acc: accuracy on broken chains (label=no). Yes%: fraction of outputs that were 'yes'."
     Depth | Overall | Pos Acc | Neg Acc | Yes%
       2   | 0.730   | 0.460   | 1.000   | 0.230
       3   | 0.845   | 0.880   | 0.810   | 0.535
       4   | 0.725   | 0.970   | 0.480   | 0.745
       5   | 0.660   | 0.990   | 0.330   | 0.830
       6   | 0.645   | 0.970   | 0.320   | 0.825
       7   | 0.650   | 0.960   | 0.340   | 0.810
       8   | 0.595   | 1.000   | 0.190   | 0.905
      10   | 0.560   | 1.000   | 0.120   | 0.940
-->

**IMAGE: Figure 2 — `experiment2_depth_overall.png`**
Place before Table 2. Full column width. Caption: "Overall accuracy of Phi-3-mini-4k-instruct on transitive equality chain verification across chain depths 2 to 10. Each point represents 200 balanced examples (100 positive, 100 negative). The dashed red line marks random chance (50%)."

**IMAGE: Figure 3 — `experiment2_posneg_breakdown.png`**
Place after Table 2. Full column width. This is the most interpretively important figure: it shows positive accuracy, negative accuracy, overall accuracy, and the yes-output rate on the same axes. The crossover between positive and negative accuracy near depth 3 and the monotonic rise of the yes-rate are the key visual findings. Caption: "Decomposition of accuracy by example type across chain depths. Positive accuracy (green) covers intact chains; negative accuracy (red) covers chains with one broken link. The yes-output rate (orange) reveals a monotonic shift from 'no' bias at depth 2 to near-total 'yes' bias at depth 10."

**IMAGE: Figure 4 — `experiment2_degradation.png`**
Place as supplementary or at the end of Section 4.2. Caption: "Accuracy relative to depth-2 baseline. A value of 1.0 indicates equivalent performance to depth-2. Both zero-shot curves degrade, but the degradation in negative accuracy is substantially steeper."

**Bias inversion.** At depth 2, the model outputs "no" in 77% of cases, yielding 0% error on negative examples but only 54% error on positive ones. Between depths 2 and 3, its output distribution shifts toward balance. From depth 4 onward, the yes-rate rises monotonically, reaching 94% at depth 10. At that point positive accuracy is 1.0 and negative accuracy is 0.12. The model is not verifying the chain; it is treating premise-list length as a proxy for logical validity.

<!-- TODO [STUDENTS]: Include one full example prompt from depth 2 and one from depth 10, with the model's predicted output noted below each. Pull these from the "EXAMPLE PROMPTS BY DEPTH" cell output in the notebook. This concretizes the failure mode. -->

<!-- TODO [STUDENTS]: Pinpoint the crossover depth explicitly. At depth 2, pos_acc (0.46) < neg_acc (1.00). At depth 3, pos_acc (0.88) > neg_acc (0.81). State: "The model's yes-bias exceeds its no-bias between depths 2 and 3, as evidenced by the crossover of positive and negative accuracy curves in Figure 3." -->

**Overall accuracy is misleading at depth 3.** The highest overall accuracy occurs at depth 3 (84.5%), because the model's approximately balanced output at that depth happens to align with the 50/50 label split. At all other depths, a mismatch between the model's output bias and the label distribution suppresses overall accuracy. A benchmark that reports only depth-3 results would present an overly optimistic picture.

<!-- TODO [STUDENTS]: If fine-tuned model results are available from the Phi notebook checkpoint, add a third paragraph here comparing zero-shot vs. fine-tuned at each depth. If not, note this as future work. -->

---

## 5. Discussion

<!-- TODO [STUDENTS]: Write 3 to 4 paragraphs using the structure below. -->

<!-- PARAGRAPH 1 — Benchmarking implications.
     The 23.5pp gap on transitivity shows that per-question accuracy systematically overstates
     logical competence. Propose that future benchmarks report: (a) per-question accuracy,
     (b) logical consistency, and (c) positive/negative accuracy separately in depth-scaling tasks.
     Tie to Novikova et al. (2025), who identified transitive consistency as an underexplored gap. -->

<!-- PARAGRAPH 2 — The bias inversion as a heuristic switch.
     Discuss what might cause the model to shift from "no" bias to "yes" bias as chain length grows.
     One account: at depth 2, the model's prior on arbitrary entity equality is negative, since
     two random strings are rarely equal in natural text. At longer depths, the structure of the
     prompt increasingly resembles valid transitive reasoning chains from pretraining, triggering
     a pattern-match to a "yes" response rather than genuine verification.
     Connect to Mehrafarin et al. (2024) on textual overlap exploitation. -->

<!-- PARAGRAPH 3 — Reflexivity and entity-level priors.
     The 32.3% accuracy on X=X is surprising but interpretable: in pretraining data, two instances
     of the same novel string in an equality context are essentially never observed.
     The model's default prior for equality between unknown entities is strongly negative,
     and this prior overwhelms the logical axiom. This is consistent with the observation in
     GSM-Symbolic (Mirzadeh et al., 2024) that model behavior is sensitive to surface-level
     entity properties. -->

<!-- PARAGRAPH 4 — Limitations.
     (a) Single model: results for Phi-3-mini may not generalize to larger or differently-trained models.
     (b) Single relation: only equality is tested. Relations with stronger asymmetry in pretraining
         frequency (e.g. "is older than") may show different bias patterns.
     (c) Logit-based prediction vs. chain-of-thought: we do not know whether eliciting explicit
         intermediate steps would improve consistency.
     (d) Fine-tuning results at depth: not yet collected for the depth-scaling experiment.
     Keep each point to one sentence. -->

---

## 6. Conclusion

<!-- TODO [STUDENTS]: Write a conclusion of roughly 150 words covering:
     1. Core argument: per-question accuracy alone is insufficient for evaluating relational reasoning.
     2. Three findings with key numbers: transitivity gap (23.5pp), reflexivity failure (32.3%),
        bias inversion (yes-rate 23% to 94%).
     3. Practical recommendation: report consistency and decompose accuracy by positive/negative examples.
     4. One forward-looking sentence on what genuine logical consistency would require
        (e.g. training objectives that enforce axiomatic constraints, or hybrid symbolic-neural approaches). -->

---

## References

Ahn, J. J., and Yin, W. (2025). Prompt-Reverse Inconsistency: LLM Self-Inconsistency Beyond Generative Randomness and Prompt Paraphrasing. *COLM 2025*. arXiv:2504.01282.

Benchekroun, Y., Dervishi, M., Ibrahim, M., Gaya, J.-B., Martinet, X., Mialon, G., Scialom, T., Dupoux, E., Hupkes, D., and Vincent, P. (2023). WorldSense: A Synthetic Benchmark for Grounded Reasoning in Large Language Models. arXiv:2311.15930.

Chen, M. K., Zhang, X., and Tao, D. (2025). JustLogic: A Comprehensive Benchmark for Evaluating Deductive Reasoning in Large Language Models. arXiv:2501.14851.

Dziri, N., Lu, X., Sclar, M., Li, X. L., et al. (2023). Faith and Fate: Limits of Transformers on Compositionality. *NeurIPS 2023*. arXiv:2305.18654.

Ghosh, B., Hasan, S., Arafat, N. A., and Khan, A. (2024). Logical Consistency of Large Language Models in Fact-Checking. *ICLR 2025*. arXiv:2412.16100.

Khalid, I., Nourollah, A. M., and Schockaert, S. (2025). Large Language and Reasoning Models are Shallow Disjunctive Reasoners. *ACL 2025*. arXiv:2503.23487.

Liu, Y., Guo, Z., Liang, T., Shareghi, E., Vulic, I., and Collier, N. (2024). Aligning with Logic: Measuring, Evaluating and Improving Logical Preference Consistency in Large Language Models. arXiv:2410.02205.

Mehrafarin, H., Eshghi, A., and Konstas, I. (2024). Reasoning or a Semblance of it? A Diagnostic Study of Transitive Reasoning in LLMs. *EMNLP 2024*. arXiv:2410.20200.

Mirzadeh, I., Alizadeh, K., Shahrokhi, H., Tuzel, O., Bengio, S., and Farajtabar, M. (2024). GSM-Symbolic: Understanding the Limitations of Mathematical Reasoning in Large Language Models. *ICLR 2025*. arXiv:2410.05229.

Morishita, T., Morio, G., Yamaguchi, A., and Sogawa, Y. (2024). Enhancing Reasoning Capabilities of LLMs via Principled Synthetic Logic Corpus. *NeurIPS 2024*. arXiv:2411.12498.

Ni, R., Xiao, D., Meng, Q., Li, X., Zheng, S., and Liang, H. (2024). Benchmarking and Understanding Compositional Relational Reasoning of LLMs. *AAAI 2025*. arXiv:2412.12841.

Novikova, J., Anderson, C., Blili-Hamelin, B., Rosati, D., and Majumdar, S. (2025). Consistency in Language Models: Current Landscape, Challenges, and Future Directions. *ICML 2025 Workshop*. arXiv:2505.00268.

Opedal, A., Shirakami, H., Scholkopf, B., Saparov, A., and Sachan, M. (2025). MathGAP: Out-of-Distribution Evaluation on Problems with Arbitrarily Complex Proofs. *ICLR 2025*. arXiv:2410.13502.

Patel, N., Kulkarni, M., Parmar, M., Budhiraja, A., Nakamura, M., Varshney, N., and Baral, C. (2024). Multi-LogiEval: Towards Evaluating Multi-Step Logical Reasoning Ability of Large Language Models. *EMNLP 2024*. arXiv:2406.17169.

Saparov, A., and He, H. (2023). Language Models Are Greedy Reasoners: A Systematic Formal Analysis of Chain-of-Thought. *ICLR 2023*. arXiv:2210.01240.

Saparov, A., Pang, R. Y., Padmakumar, V., Joshi, N., Kazemi, S. M., Kim, N., and He, H. (2023). Testing the General Deductive Reasoning Capacity of Large Language Models Using OOD Examples. *NeurIPS 2023*. arXiv:2305.15269.

---

<!-- ================================================================
     FIGURE PLACEMENT GUIDE
     ================================================================

     Figure 1  experiment1_consistency_results.png
       Placement: End of Section 4.1, after Table 1
       Width: Full column
       Caption: "Per-question accuracy (blue) vs. logical consistency score (orange)
       for Phi-3-mini-4k-instruct on three relational properties. Each property is
       evaluated on 250-300 randomly-named entity groups. A positive gap indicates
       the model answers individual questions correctly more often than it satisfies
       the logical axiom jointly across related questions."

     Figure 2  experiment2_depth_overall.png
       Placement: Before Table 2, start of Section 4.2
       Width: Full column
       Caption: "Overall accuracy on transitive equality chain verification across
       chain depths 2 to 10. Each point represents 200 balanced examples. The red
       dashed line marks random chance (50%)."

     Figure 3  experiment2_posneg_breakdown.png
       Placement: After Table 2
       Width: Full column, two-panel layout if possible
       Caption: "Decomposition of accuracy by example type. Positive accuracy (green)
       covers intact chains; negative accuracy (red) covers chains with one broken
       link. The yes-output rate (orange) shows the monotonic shift from 'no' bias
       at depth 2 to near-total 'yes' bias at depth 10. This is the most important
       figure in the paper."

     Figure 4  experiment2_degradation.png
       Placement: Supplementary, or end of Section 4.2
       Width: Full column
       Caption: "Accuracy normalized to the depth-2 baseline. Values below 1.0
       indicate degradation relative to depth-2 performance."

     ================================================================ -->
