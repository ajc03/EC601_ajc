# Machine Learning for Patent Novelty Assessment

*EC601 Phase 1 Tutorial*

---

## How to Read This Document

This tutorial is on the **computational assessment of patent novelty**, the task of deciding whether an invention described in a patent claim was already disclosed in an earlier document.

**General Note:** Patents are not one of the nine EC601 research areas. This tutorial falls under **Area 1: Machine Learning — Vision, Multimodal & Beyond**, specifically *"VLM-powered applications with rigorous evaluation"* and *"retrieval-augmented pipelines: when should a system look things up versus rely on the model — and how do you measure the difference?"* Patent documents are just the subject while retrieval and evaluation are the research content.

**Quizzes:** Five quiz blocks are embedded at section boundaries, with answers and explanations immediately following each. Attempt them before reading the answers.

**Verification Status:** Three papers were read in full by a human (AJ Chiaravalloti) first, who was then quizzed on them by Claude. Every factual claim attributed to them was checked against the papers themselves. Claims of general legal background are flagged where they were *not* verified against primary sources. See §8.

---

## 1. Motivation

### 1.1 What is a patent?

A patent gives its owner the right to exclude others from making, using, or selling an invention for a limited period. The document has two halves: a **specification**, that describes the invention in enough detail for someone skilled in the field to build it, and a set of **claims**, which are numbered sentences at the end that define the legal boundary of what is protected.

The distinction between specifications and claims is essential in this tutorial. The specification *describes*, and the claims *demarcate*. The legally relevant definition of the invention is found in the independent claims, usually claim 1, which may be short and deliberately generalized in order to keep the scope of protection as broad as possible [1].

### 1.2 What is the task?

To be patentable, an invention must be **new** or **novel**. The body of everything publicly available before the relevant date is called **prior art**, which a patent examiner must search and decide whether the claimed invention is already disclosed there.

This is difficult because:

- **The corpus is enormous** - It includes every published application and granted patent worldwide, in every language, PLUS scientific literature, product manuals, and public demonstrations.
- **The query is a document (NOT a keyword)** - You are matching an entire claim against passages inside other documents.
- **The vocabulary is adversarial** - Applicants have a financial incentive to describe their invention in terms nobody else has used, because broad and unusual language yields broad protection. A claim about "a plurality of elongate conductive members" may be describing wires.
- **The judgment is legal (NOT topical)** - Two documents about the same subject may be irrelevant to each other for novelty purposes, and a document from a different field may destroy the claim entirely.

### 1.3 Who is this for?

The same technical system ("find prior art for this claim") becomes three different engineering problems depending on who runs it.

| | **Examiner** | **Invalidity searcher** | **Freedom-to-operate analyst** |
|---|---|---|---|
| **Decision** | Can we grant or reject this application? | Can we invalidate this patent in litigation? | Can we ship this product? |
| **Stopping rule** | Stops on finding one killer reference | Must be exhaustive | Must be exhaustive |
| **Cost of a miss** | A bad patent issues; correctable later via opposition | Case lost; the patent stands | Injunction, damages, product pulled |
| **Cost of a false alarm** | Wasted examination effort | Wasted attorney hours | Wasted redesign, possibly abandoned product |
| **What to optimize** | Precision | Recall | Recall |

Only one retrieved "X" or "Y" document is enough to refuse claim 1, and for that reason the search task is focused more on precision than on recall [1]. (Will explain what "X" or "Y" documents are and what "claim 1" means later in the tutorial).

It is tempting to say "patent searches are precision-focused tasks." That is not necessarily true. Precision-dominance is a property of *the examiner's role and stopping rule*, not of patent search. For the litigator, the asymmetry inverts completely: a missed reference is the whole case. Any paper one reads is implicitly serving one of these users, and its metrics only make sense once he/she knows which.

### 1.4 What are the potential applications?

- **Examiner support tools** - Both the EPO and USPTO run internal search systems. Reducing examiner search time has direct economic value given growing global application volumes.
- **Corporate IP clearance** - Large technology companies run continuous FTO monitoring. Bosch (whose research lab produced one of the three papers cited [3]) is an example of such an organisation.
- **Drafting assistance** - Attorneys check draft claims against prior art before filing.
- **Portfolio and landscape analysis** - Investors and R&D planners use patent data to map who is working on what.
- **Litigation support** - Invalidity searches in patent disputes, where the economics justify enormous manual effort and therefore also justify automation.

### 1.5 Why it this a good research area for one semester (~12 weeks)?

The data is free and public: EPO bulk full-text and Publication Server, the EPO Register, USPTO bulk downloads, PatentsView. The evaluation is hard in ways that are legible, but the open problems do not all require large compute (see rest of tutorial).

---

### 🧩 Quiz 1 — Motivation and Framing

**Q1.1 (multiple choice) -** Where is the scope of legal protection defined?
a) The abstract
b) The detailed description
c) The independent claims
d) The classification codes

**Q1.2 (multiple choice) -** A start-up is checking whether its new sensor infringes any live patent before launch. Which error is more costly for them?
a) A false positive — flagging a patent that turns out not to matter
b) A false negative — missing a patent that does matter
c) Both are equally costly
d) Neither; FTO analysis does not use retrieval

**Q1.3 (short answer) -** A paper reports that its prior-art retrieval system achieves high precision and modest recall, and concludes that it is "suitable for deployment." What question should you ask before accepting that conclusion?

<details>
<summary><b>Answers to Quiz 1</b></summary>

**Q1.1 — (c)** The claims, specifically the independent claims. The specification teaches how to build the invention, but the claims define what is owned. Dependent claims add optional features and depend on an independent claim [1].

**Q1.2 — (b)** A missed patent means shipping a product that infringes, and potentially injunction and damages exposure. A false positive costs attorney time (not ideal, but not horrible). The asymmetry is severe and points the opposite way from the examiner's.

**Q1.3 — *Who is the user?*** High precision with modest recall is appropriate for an examiner, who stops at the first "killer" reference, but is dangerous for an invalidity or FTO searcher, who needs completeness. The given metric ("suitable for deployment") cannot be evaluated without knowing the role of the person using it.

</details>

---

## 2. Core Concepts from First Principles

### 2.1 The Anatomy of a Claim

A claim is a single sentence that defines the invention as a combination of **features**, where each feature is a component, step, or characteristic [3]. Consider the paper's illustration of a claim about a text classification method, whose features might be: converting text into tokens, applying a neural network to the tokens, or mapping embeddings into the label space [3].

Two structural facts:

**Independent versus dependent -** An independent claim stands alone. A dependent claim refers back to another claim and narrows it. For example, "2. The system according to claim 1, wherein the light source is an OLED." A patent may have several independent claims, for example a "system claim 1" and a "method claim 15" [1].

**Novelty is a property of the combination -** This is the crucial rule, and it is why the task is not text similarity:

> An invention is novel over a prior art document if **at least one** feature of the claim is not disclosed in that document [3].

A prior art document that discloses nine of ten features destroys nothing. Adding features to a claim makes it *easier* to be novel and *narrower* in protection, which is precisely why applicants add limitations when an examiner objects. This fact  will come back as a serious methodological problem in §4.3.

### 2.2 The Examination Workflow, and Where Labels Come From

Understanding the European procedure is necessary because two of the three datasets in this tutorial are built from it.

1. An application is filed and published as an **A1** document.
2. The office produces a **European Search Report (ESR)** listing cited prior art, categorizing each document and pointing at relevant passages.
3. Where a claim lacks novelty, the examiner writes a **European Search Opinion (ESOP)** explaining the rejection, reciting the claim and giving references for each feature [3].
4. The applicant **amends** the claims — typically adding limitations — possibly over several rounds.
5. If successful, the patent is granted and published as a **B1** document [3].

The citation categories in the search report carry specific meanings [1]:

| Category | Meaning |
|---|---|
| **X** | Discloses the complete invention of the claim — novelty-destroying on its own |
| **Y** | Does not disclose it fully, but in combination renders it obvious |
| **A** | Technological background; not relevant to novelty or inventive step of claim 1 |

**Every dataset in this tutorial is built from artifacts of this workflow.** Nobody in it was ever asked to label data for machine learning.

### 2.3 Novelty Versus Obviousness

Novelty (does *one* document disclose everything?) is distinct from obviousness or inventive step (would the combination have been obvious to a skilled person?). Novelty is more tractable: the single-document requirement gives it a specific structure. Obviousness requires judging what a hypothetical skilled person would have found motivating to combine.

Every dataset discussed here targets **novelty**, and each excludes obviousness explicitly: one drops "Y" citations entirely [1], and another filters out rejections based on lack of inventive step [3]. This is a sensible simplification and a real limitation, since a great deal of actual examination turns on obviousness.

### 2.4 Gold Labels Versus Proxy Labels

A **gold label** is produced for the task by someone qualified to judge the task. A **proxy label** is a cheaper correlated signal used in its place.

Proxies used in this field:

| Proxy | Stands in for | Where used |
|---|---|---|
| X citation in a search report | "This passage destroys novelty" | PatentMatch [1] |
| A citation in a search report | "This passage does not destroy novelty" | PatentMatch [1] |
| Two drawings in the same patent | "These images depict the same object" | DeepPatent [2] |
| Feature-level ESOP citation | "This passage discloses this feature" | FiNE-Patents [3] |
| Text added between A1 and B1 | "This feature is what made it novel" | FiNE-Patents [3] |

None of these are wrong, but all of them are *approximations whose error rate is unknown*. Two specific failure modes recur:

**Unreliable negatives -** "While cited passages reliably indicate disclosure, the absence of a citation does not imply non-disclosure." Negatives may not be true negatives [3]. An examiner who found one killer passage had no reason to keep listing others.

**Granularity mismatch.** An examiner citing paragraphs 27–28 against claims 1 and 3–9 made *one* legal judgment about a document. Expanding that into many independent claim–passage pairs assumes a granularity the original annotation never had.

### 2.5 Spurious Correlations

A **spurious correlation** (or shortcut) is a feature that predicts the label without being causally related to the task. If claims that were rejected are systematically shorter than claims that were granted, a model can score well on "is this claim novel?" by counting words and never looking at the prior art at all.

This is not hypothetical. It is the central finding of §4.3, and it is why the field's headline numbers must be read with care.

---

### 🧩 Quiz 2 — Core concepts

**Q2.1 (multiple choice) -** A prior art document discloses seven of the eight features of a claim. Under the novelty standard, the claim is:
a) Not novel, since most features are disclosed
b) Novel, because at least one feature is not disclosed
c) Novel only if the missing feature is in an independent claim
d) Indeterminate without knowing the obviousness analysis

**Q2.2 (multiple choice) -** Why does an applicant add limitations to a claim after a novelty objection?
a) To broaden the protection and cover more products
b) To satisfy formatting requirements imposed by the office
c) To narrow the claim so that some feature is no longer disclosed
d) To move the application into a different classification class

**Q2.3 (multiple choice) -** A dataset treats "passage not cited by the examiner" as a negative. What is the risk?
a) The negatives are too easy, since uncited passages are topically unrelated
b) An uncited passage may still disclose the feature, so negatives are unreliable
c) It makes the label distribution unbalanced and unusable
d) Uncited passages cannot be resolved to text and must be discarded

**Q2.4 (short answer) -** Explain in two sentences why "X" and "A" citations make a harder classification problem than "X" citations against randomly sampled paragraphs.

<details>
<summary><b>Answers to Quiz 2</b></summary>

**Q2.1 — (b)** Novelty requires only that at least one feature be undisclosed [3]. This is what makes claim analysis combinatorial rather than a similarity judgment.

**Q2.2 — (c)** Applicants typically add limitations to overcome novelty objections [3]. The claim becomes narrower and therefore easier to distinguish from prior art (trading scope for grant).

**Q2.3 — (b)** The absence of a citation does not imply non-disclosure because an examiner stops once the point is made [3].

**Q2.4.** An "A" document was retrieved by an expert searching the same technical territory, so it is topically adjacent by construction, a hard negative. A model therefore cannot succeed by subject-matter similarity alone, and must judge whether the passage actually discloses the claimed combination, which is much closer to the legal task.

</details>

---

## 3. The Retrieval Machinery (Summarized)

You do not need a deep IR background, but you do need the vocabulary to read the results tables.

### 3.1 Ways to Match Text

**Lexical matching (BM25, ROUGE-L overlap) -** Score documents by shared terms, weighting rare terms more heavily and normalizing for length. This is fast, interpretable, and hard to beat. Its weakness is the patent vocabulary problem: it cannot match "elongate conductive member" to "wire."

**Dense retrieval -** Encode queries and passages into vectors so semantically similar texts sit close together, then retrieve by nearest neighbour. Dense Passage Retrieval (DPR) architecture (two separate encoders, one for queries and one for passages) is the reference design, and Risch et al. report being the first to train a DPR model on patent data [1].

**Cross-encoders / text-pair classification -** Feed both texts to one model together and classify the pair. More accurate than dense retrieval because the two texts can attend to each other, but too expensive to run over a whole corpus, so it is used for re-ranking a shortlist. PatentMatch's BERT baseline is this design [1].

**LLM workflows -** Prompt a large language model with the claim and the prior art document and have it output structured judgments. This is the FiNE-Patents approach [3].

### 3.2 Negatives

**Random negatives** are easy: a randomly drawn paragraph is usually about something else entirely. **Hard negatives** are topically similar but still wrong, and they force a model to learn the actual distinction. **In-batch negatives** are a training trick where each item's positive doubles as a negative for the other items in the same batch, giving many negatives cheaply [1].

### 3.3 Metrics: How to be Misled by Them

- **Precision** — of what you returned, how much was right.
- **Recall** — of what you should have returned, how much you found.
- **F1** — their harmonic mean.
- **nDCG** — rank-sensitive, being right at position 1 beats being right at position 20.
- **mAP** — mean average precision across queries, standard in image retrieval [2].
- **Acc@K** — does the top-K set contain at least one correct item [2].
- **Soft precision/recall** — partial credit for near-misses. Knappich et al. compute these using ROUGE-L overlap between predicted and cited passages, precisely because unreliable negatives make the hard version pessimistic [3].

**Three traps that appear in the papers below:**

1. **The denominator -** "Rank 1.42" sounds excellent until you learn it is among 8 in-batch candidates rather than a full corpus [1]. You always need the candidate-pool size.
2. **The database size -** Retrieval scores fall as the search space grows. DeepPatent's mAP drops from 0.376 to 0.262 when the database goes from ~38,000 to ~350,000 images [2].
3. **The baselines that should not work -** If a model with no access to the evidence scores well, the benchmark has a shortcut [3].

---

### 🧩 Quiz 3 — Retrieval machinery

**Q3.1 (multiple choice) -** Why is a cross-encoder not used as a first-stage retriever over millions of passages?
a) It cannot handle long documents
b) It requires labelled training data, unlike dense retrieval
c) It must process every query–passage pair jointly, which is too expensive
d) It performs worse than lexical matching on technical text

**Q3.2 (multiple choice) -** A model reports an average in-batch rank of 1.42 with batch size 8. The most accurate reading is:
a) The model ranks the correct passage second or third out of eight candidates
b) The model retrieves the correct passage within the top 1.42% of the corpus
c) The model is slightly worse than random for this candidate pool
d) Average rank is not a meaningful quantity and should be ignored

**Q3.3 (short answer) -** Why might a paper report *soft* precision and recall in addition to the standard versions, and what does that choice reveal about its labels?

<details>
<summary><b>Answers to Quiz 3</b></summary>

**Q3.1 — (c)** A cross-encoder computes a joint representation per pair, so cost scales with corpus size at query time. Dense retrieval pre-computes passage vectors offline, which is what makes it deployable at scale.

**Q3.2 — (a)** With rank 0 as first position, 1.42 means roughly second-to-third out of eight [1]. It is a training diagnostic, not a retrieval result. Reporting it without the batch size would be misleading.

**Q3.3** Soft metrics give partial credit via textual overlap. Reporting them signals that the authors do not trust their negatives: an uncited passage may still be relevant, so a hard-scored "false positive" may in fact be correct [3]. It is an admission that the ground truth has holes.

</details>

---

## 4. Three Papers

The three papers below span five years and, read together, tell a coherent story about the field maturing in its treatment of supervision.

### 4.1 PatentMatch (Risch, Alder, Hewel & Krestel, 2021)

**What it is:** A dataset pairing claims from patent applications with passages from cited prior art, each labelled for whether the passage is prejudicial to the novelty of the claim [1].

**How the labels were made:** The authors did not annotate anything. They took EPO search reports (available in bulk from 2012 onward) parsed the citation entries, and resolved claim and paragraph numbers to their text. "X" citations became positives, "A" citations became negatives, and "Y" citations were excluded, on the grounds that they seemed too close to "X" in semantic relevance to generate a good training signal [1].

**Scale (what it means):** There were 6,259,703 samples, but only 297,147 distinct claim texts and 31,238 distinct applications [1]. That is roughly 21 samples per claim. The sample count wildly overstates independent information. The authors provide a stricter variant with exactly one X and one A per claim, which collapses the data to 25,340 samples [1]. That number (not 6,259,703) is the honest measure of independent signal.

**Design choices worth copying:** The train/test split is time-wise on filing date (March 29, 2017) rather than random [1]. Patent families produce near-identical claims filed years apart; a random split would scatter them across train and test and inflate scores through memorization. It also matches deployment, where tomorrow's application is not in today's index.

**Results:** A fine-tuned BERT text-pair classifier reached **54%** accuracy on the balanced variant and **52%** on the stricter one, which is barely above chance. Validation loss stopped improving after six epochs, so this result is not due to undertraining. The authors are not surprised that the task is hard, citing legal jargon and domain-specific language, that are difficult for laymen to understand, as a reason for the performance [1]. A DPR model reached an average in-batch rank of 1.42 out of 8, which they present modestly as useful for narrowing to a handful of candidates for a human expert [1].

**Known limitations.**
- References that resolve to figures, figure captions, or whole documents were discarded [1]. Prior art whose disclosure lives in drawings is systematically excluded.
- The labels are examination byproducts, expanded to a granularity the original judgment never had.
- Obviousness is out of scope by construction.

---

### 🧩 Quiz 4a — PatentMatch

**Q4.1 (multiple choice) -** Why were "Y" citations excluded from the dataset?
a) They are rare in European search reports
b) They are too semantically close to "X" to give a clean training signal
c) They refer to figures rather than to text paragraphs
d) They concern a requirement the EPO does not examine

**Q4.2 (multiple choice) -** The dataset has 6.26M samples but 297K distinct claims. What follows?
a) The duplicates should be deleted before training
b) The dataset covers only a narrow technical field
c) Sample count overstates independent information and splits must group by claim
d) The label distribution must be approximately balanced

**Q4.3 (short answer) -** The BERT baseline reaches 54% on a balanced binary task. Give two distinct explanations, and say how you would tell them apart.

<details>
<summary><b>Answers to Quiz 4a</b></summary>

**Q4.1 — (b)** "Y" documents render a claim obvious without disclosing it fully, placing them between X and A in relevance. The authors judged them too close to X for a clean binary signal [1]. The cost is that the obviousness distinction is removed from the dataset.

**Q4.2 — (c)** About 21 samples per distinct claim. Treating them as independent inflates effective sample size and leaks across splits unless grouping is enforced. The authors' stricter 25,340-sample variant is the response [1].

**Q4.3** *Explanation A:* the task is genuinely hard: legal language and abstract feature-matching defeat a general-purpose encoder, which is what the authors conclude [1]. *Explanation B:* a share of the labels are wrong, because the X/A judgment was made per document per claim-group and then expanded into pairs. *To distinguish them:* have a qualified annotator re-label a random sample of pairs and measure agreement with the derived labels. High agreement points to task difficulty and low agreement points to label noise.

</details>

---

### 4.2 DeepPatent (Kucer, Oyen, Castorena & Wu, WACV 2022)

**Why a vision paper is important:** Patent drawings often carry information more accessible than the text, yet retrieval in this domain relies primarily on text and ignores the drawings [2]. If your prior art search is text-only, you are blind to a channel that examiners actually use.

**What it is:** Over 350,000 **design** patent drawings for image retrieval, with within-domain fine-grained associations rather than cross-domain supervision [2].

**How the labels were made:** Again, no annotation. Design patents contain multiple drawings of one object from different viewpoints (on the order of ten per patent) so co-membership in a patent defines a positive pair [2]. Free supervision that captures a real invariance: the same object seen from an underside or aerial view.

**The scope restriction you must not miss:** Design patents capture the visual characteristics of an object, so their figures are object depictions, whereas utility patents contain flowcharts, plots, mathematical expressions, and text-heavy mechanical diagrams [2]. DeepPatent is therefore a clean vision benchmark and a **poor proxy for utility-patent prior art**. Utility patents are where essentially all electrical and computing subject matter lives.

**Method:** PatentNet: ResNet18/50 backbones with Generalized Mean pooling and L2 normalization, pretrained then fine-tuned by classification (each patent ID as a class) and then with triplet or contrastive retrieval loss [2].

**Four Important Results**

1. **ImageNet pretraining beats in-domain self-supervision:** A RotNet model self-supervised on the patent data itself reached 0.169 mAP against 0.291 for ImageNet weights, despite the ImageNet network never having seen a patent [2]. A corrective to the assumption that in-domain pretraining always wins.
2. **Sketch data actively hurts:** Fine-tuning on the Sketchy sketch dataset dropped mAP to 0.229 — *below* the plain ImageNet baseline [2]. Patent drawings and free-hand sketches look similar and are not the same domain. A supporting statistic: median connected components are 705 in DeepPatent against 2 in Sketchy, while ImageNet photo edge maps sit at 673 [2]. Structurally, patent drawings resemble photographs more than sketches.
3. **Scale hurts:** Expanding the search database from ~38,000 to ~350,000 images drops mAP from 0.376 to 0.262 and Top-1 from 69.1% to 55.1% [2]. And 350,000 images is roughly eighteen months of design patents alone.
4. **The label ceiling:** The authors note that some patents contain very similar images, and that in a successful retrieval example all but one of the returned images are arguably relevant [2]. Two independent patents for near-identical objects are semantically relevant but scored as non-matches. Reported mAP is a lower bound on true relevance, and the gap is unmeasured.

**Naming hazard note:** "DeepPatent" also names an unrelated 2018 text classification model, which this paper cites in its own reference list [2]. Always qualify: *the DeepPatent drawing dataset (Kucer et al., 2022)*.

---

### 🧩 Quiz 4b — DeepPatent

**Q4.4 (multiple choice) -** Why does restricting to design patents limit the dataset's usefulness for utility-patent prior art search?
a) Design patents are not published in the same bulk data channels
b) Design patent drawings are object depictions, while utility figures include flowcharts, plots, and equations
c) Design patents have only one drawing each, so no positive pairs exist
d) Design patent drawings are under copyright and cannot be redistributed

**Q4.5 (multiple choice) -** Fine-tuning on the Sketchy sketch dataset before patent retrieval:
a) Improves performance, since sketches are the nearest available domain
b) Leaves performance essentially unchanged
c) Reduces performance below the plain ImageNet baseline
d) Improves mAP while reducing Top-K accuracy

**Q4.6 (short answer).** DeepPatent defines relevance as "same patent." Name the systematic error this introduces and state its direction on reported mAP.

<details>
<summary><b>Answers to Quiz 4b</b></summary>

**Q4.4 — (b).** Design patents protect visual appearance, so the figures depict the object. Utility patents mix in diagrams, plots, and text-heavy figures [2]. The benchmark is clean but unrepresentative of where EE prior art lives.

**Q4.5 — (c).** mAP falls to 0.229 against 0.291 for plain ImageNet weights, and drops further with SBIR training [2]. The domain mismatch is real and, per their reverse-transfer experiment, runs in both directions.

**Q4.6.** Visually and semantically similar drawings from *different* patents are scored as non-matches, which are false negatives in the ground truth. The direction is downward: reported mAP **understates** true semantic retrieval quality, so it is a lower bound. The size of the gap is unknown because nobody has re-annotated a sample.

</details>

---

### 4.3 FiNE-Patents (Knappich, Hätty, Razniewski & Friedrich, SIGIR 2026)

This is the paper that makes the field's supervision problem explicit and measurable, and it is the most important of the three for anyone starting work now.

**The core argument:** Prior work treated novelty prediction as binary classification at the claim level. The authors argue this formulation is susceptible to spurious correlations and lacks the granularity needed in practice [3]. A binary "not novel" label tells a patent professional nothing about *which* part of their invention is already known, which is the only output they can act on.

**The better supervision source:** Prior datasets used European Search Reports. FiNE-Patents uses European Search **Opinions**, in which the examiner recites the rejected claim and gives precise references for every feature [3].

The comparison between the two:

> **Only 19% of the passages cited in the ESR are also cited in the ESOP** [3].

This has two causes: ESR citations are aggregated over multiple claims, losing which passage relates to which claim, and examiners merge ranges for brevity, citing paragraphs 10–20 instead of 10–13, 16–18, 20 [3]. ESOPs are the superior supervision at the cost of expensive PDF parsing [3].

Note what this does to §4.1. PatentMatch is built on ESRs. This is direct evidence that its supervision is coarse, and is a plausible partial explanation for the 54% baseline.

**Three sub-tasks instead of one:** [3]:
1. **Passage retrieval** — which passages of the prior art disclose each claim feature?
2. **Novel feature identification** — which features make the claim novel?
3. **Claim novelty prediction** — is the claim novel overall?

**Dataset construction:** 3,658 first claims from 3,163 EPO applications in CPC classes G06, G10L, H04L and H04N, filed from 2012 onward, evenly split between novel/not-novel [3]. ESOPs were parsed with a vision-language model taking image input directly, avoiding error accumulation in an OCR pipeline [3]. Novel-feature ground truth comes from a character-level diff between the A1 and B1 claim: spans added during prosecution serve as a proxy for the novel features, since applicants typically add limitations to overcome novelty objections [3].

**The filtering funnel is steep:** 364k seed applications → 132k granted → 67k with full texts → 65k with an ESOP → 19k reciting the full first claim with feature-level references → 17k successfully parsed → 4.4k after further filtering → 8.8k claims → **3,658** after length stratification [3]. Only granted applications survive, in four CPC classes. This is a precise, high-quality, and thoroughly non-representative sample.

#### The spurious correlation results:

**Shortcut 1 — claim length:** Applicants add limitations to overcome objections, so granted claims are longer. The authors stratify into 100 length bins and subsample so both classes share a length distribution [3].

**Shortcut 2 — reference numerals:** Granted claims average **12.1** reference numerals against **0.3** in initial claims [3]. Their mere presence identifies the granted version. Removed by regex.

**Shortcut 3 — everything that remains:** After both mitigations, a BERT classifier given **only the claim text, with no prior art document at all**, achieves **75.2% accuracy** [3].

Deciding novelty without seeing the prior art is impossible in principle, yet this classifier outperforms every LLM workflow in the paper. An interpretable logistic regression probe identifies what it is using: n-grams that narrow scope such as "wherein," "characterized," "in that the", IPC-class indicators for domains with imbalanced labels, the number of features, and punctuation counts [3].

**The adversarial test set:** Built by keeping only test samples the shortcut classifier gets *wrong*, then rebalancing — 278 claims [3]. The authors are careful about its use: too small for model comparison, but a strong indicator of overfitting if accuracy drops sharply [3].

**And the encouraging result:** LLM workflows show only minor accuracy loss or even gains on the adversarial subset, and show low agreement with the shortcut classifier, a negative Cohen's kappa on the adversarial set [3]. Zero-shot LLMs are not riding these shortcuts, whereas fine-tuned classifiers are.

**Cross-dataset check:** The authors train an XGBoost model on claim text alone for the PANORAMA dataset's novelty task and reach 41% macro F1, substantially outperforming most prompted LLMs in that paper's own evaluation [3]. The problem is not local to their dataset.

#### Performance results

On passage retrieval and novel feature identification, LLM workflows clearly beat lexical and embedding baselines [3]. Two findings stand out:

**Decomposition can beat scale:** On novel feature identification, a 30B model with hierarchical (feature-by-feature) examination reaches **53.7 F1**, beating a 397B model doing single-step examination at **44.7 F1** [3]. Structure mattered more than parameters.

**But not everywhere:** Hierarchical examination does not help claim-level novelty prediction, and adding feature examination summaries *reduces* accuracy to 57.9% while skewing predictions to 88.5% novel [3]. Best claim-level accuracy is 69.1%, or 70.7% with self-consistency [3] against a 75.2% shortcut classifier that cannot see the evidence.

**The bottleneck result:** Supplying the ESOP's own passage references as an oracle (perfect retrieval) **does not improve** claim novelty accuracy [3]. Finding the evidence is not the hard part. Judging disclosure from evidence already in hand is.

**What the authors themselves flag as open [3]:** how reliable ESOP documents are as supervision, how a team of human experts would perform on these tasks, and what inter-annotator agreement would be. Also: extension to multi-document settings, since novelty is formally assessed against a single document but examiners in practice compare across many and record only the best match.

---

### 🧩 Quiz 4c — FiNE-Patents

**Q4.7 (multiple choice) -** A BERT classifier with no access to the prior art reaches 75.2% accuracy. This shows:
a) Shortcuts survive in the claim text, so accuracy needn't reflect novelty reasoning
b) BERT memorized the prior art documents during pretraining
c) The stratification failed and the classes became unbalanced
d) Claim-level novelty is easier than feature-level retrieval

**Q4.8 (multiple choice) -** Only 19% of ESR-cited passages also appear in the ESOP. The main causes are:
a) ESOPs are written later and cite newer prior art
b) ESRs cite a different prior art document than the ESOP
c) ESR citations aggregate across claims and merge paragraph ranges
d) ESRs cite figures, which cannot be resolved to text

**Q4.9 (multiple choice) -** Giving the model the examiner's own passage references as an oracle does not improve claim-level accuracy. This indicates:
a) The oracle passages exceed the context window
b) Retrieval is not the bottleneck. Judging disclosure is
c) The ESOP references are unreliable and mislead the model
d) Retrieval and classification are measured on different splits

**Q4.10 (short answer) -** You are designing a new patent novelty benchmark. List three concrete checks this paper implies you must run before publishing it.

<details>
<summary><b>Answers to Quiz 4c</b></summary>

**Q4.7 — (a).** The classifier cannot reason about disclosure without the prior art, so its accuracy must come from shortcuts — narrowing n-grams, IPC-class imbalance, feature counts, punctuation [3]. It beats every LLM workflow in the paper, which is the uncomfortable part.

**Q4.8 — (c).** Aggregation across claims destroys the claim-to-passage mapping, and merged ranges over-report [3]. This is the quantified weakness of every ESR-derived dataset, PatentMatch included.

**Q4.9 — (b).** Perfect evidence does not lift accuracy, so the limiting step is the disclosure judgment itself [3]. This relocates the research problem from retrieval to reasoning.

**Q4.10.** Any three of: (i) train a classifier that sees only the query and *not* the evidence (if it performs well, you have a shortcut), (ii) check for length or formatting differences between classes and stratify, (iii) strip artifacts introduced by the document lifecycle (such as reference numerals), (iv) build an adversarial subset from the shortcut model's errors as a diagnostic, (v) run a contamination check against indexed pretraining corpora, (vi) split by time or by group rather than randomly, (vii) establish a human expert baseline and inter-annotator agreement.

</details>

---

## 5. Synthesis

| | **PatentMatch** [1] | **DeepPatent** [2] | **FiNE-Patents** [3] |
|---|---|---|---|
| Year / venue | 2021, PatentSemTech@SIGIR | 2022, WACV | 2026, SIGIR |
| Modality | Text | Images | Text (+ VLM parsing) |
| Supervision source | ESR X/A citations | Patent membership | ESOP feature references |
| Granularity | Claim ↔ passage | Drawing ↔ drawing | **Feature** ↔ passage |
| Scale | 6.26M pairs (25.3K independent) | 350K+ images | 3,658 claims |
| Headline result | BERT 54% (near chance) | mAP 0.376 → 0.262 with scale | Shortcut model 75.2% > best LLM 70.7% |
| Acknowledged weakness | Coarse labels; figures dropped | Same-patent relevance proxy | Proxy novel-feature labels; narrow domains |

**Three lessons that generalize beyond patents:**

**1. Free labels are never free -** Each dataset bought scale by repurposing an artifact made for another purpose. Each inherited a specific distortion: coarse aggregation, same-patent relevance, prosecution-amendment proxies. The distortion is the price, not a flaw in the work, and the mature move is to measure it. The 19% ESR/ESOP figure [3] is the first time anyone in this line put a number on it.

**2. Check whether your benchmark can be solved without doing the task -** The single most valuable experiment in these three papers costs almost nothing: train a model that sees only half the input. If it does well, the benchmark is broken. FiNE-Patents ran it and found a 75.2% shortcut [3] (the earlier papers did not).

**3. Every number needs its denominator -** Rank 1.42 out of 8 [1]. mAP 0.376 against a 38K database, 0.262 against 350K [2]. 75.2% on a length-stratified, numeral-stripped, four-CPC-class, granted-applications-only sample [3]. Without the conditions all three become misleading.

---

## 6. What is Still Open?

**1. There is no human baseline:** None of these papers reports how well trained experts do on its task, or how much they agree with each other. Some even name this explicitly as future work [3]. Every reported number therefore floats without a ceiling: nobody knows whether 20.9 F1 on feature-level passage retrieval is poor or near the limit of what examiner agreement would support. **This is the largest and most tractable gap in the area, and it requires domain expertise rather than compute.**

**2. The disclosure-judgment bottleneck:** The oracle result [3] says retrieval is solved enough that it is no longer the limiter. What fails is deciding whether a passage discloses a feature. That is an abstract reasoning problem with a legal standard attached.

**3. Obviousness is untouched:** Every dataset here excludes it: "Y" citations dropped [1], inventive-step rejections filtered out [3]. A large share of real examination turns on it.

**4. The figure gap is real and unoccupied:** PatentMatch discards examiner references that resolve to figures [1]. DeepPatent covers only design patents, whose figures are object depictions, explicitly unlike the flowcharts and diagrams of utility patents [2]. Existing utility-patent figure resources are small and coarse-grained, CLEF-IP 2011 covers only 211 patents for retrival, and sorts figures into nine broad image types, so **no large-scale, fine grained resource for utility-patent retrieval exists** [2].

**5. Multi-document settings:** Novelty is formally assessed against a single document, but examiners compare across many and record only the best match [3]. Real deployment requires finding the document *and* judging it.

**6. Non-patent literature:** All three datasets treat prior art as patents only. Real novelty analysis includes scientific papers, manuals, and public disclosures [3].

**7. Cross-jurisdiction generality:** Two of three datasets are EPO-derived. EPO examination is unusually high quality (each application is examined by at least three examiners to reach a joint decision [3]) so results may not transfer to offices with different procedures.

---

### 🧩 Quiz 5 — Synthesis

**Q5.1 (multiple choice) -** Across all three papers, the common structural weakness is:
a) Insufficient training data
b) Labels repurposed from artifacts created for another purpose
c) Reliance on closed-weight models that cannot be reproduced
d) Evaluation on synthetic rather than real patent documents

**Q5.2 (multiple choice) -** Why does the absence of a human expert baseline matter?
a) Reviewers require one for publication at IR venues
b) It would improve model accuracy through better training data
c) Without it, no one knows whether a given score is near the task's ceiling
d) It is needed to compute inter-annotator agreement for licensing

**Q5.3 (short answer) -** Write the two most damaging questions you would ask about a new paper claiming 85% accuracy on patent novelty prediction.

<details>
<summary><b>Answers to Quiz 5</b></summary>

**Q5.1 — (b)** ESR citations, patent membership, ESOP references, and prosecution diffs are all byproducts of processes that had nothing to do with training models [1][2][3].

**Q5.2 — (c)** Scores float without a ceiling. If trained examiners agreed with each other only 70% of the time on feature-level disclosure, a model at 65% would be nearly saturated rather than failing, and the field would be optimizing noise. This was flagged as open [3].

**Q5.3** Strongest two: *(i)* "What does a classifier that sees only the claim, without any prior art, score on your test set?" — if they haven't run it, the 85% is unattributed, and FiNE-Patents got 75.2% from shortcuts alone [3]. *(ii)* "Is the split temporal or grouped by patent family?" — a random split over patent data leaks near-identical claims between train and test. Either answer alone can account for most of the headline number.

</details>

---

## 7. References

All three primary references were retrieved and read in full during preparation of this tutorial. Claims attributed to them were checked against the paper text.

**[1] Risch, J., Alder, N., Hewel, C., & Krestel, R. (2021).** PatentMatch: A Dataset for Matching Patent Claims & Prior Art. In *Proceedings of the 2nd Workshop on Patent Text Mining and Semantic Technologies (PatentSemTech@SIGIR)*, pp. 40–44. CEUR Workshop Proceedings, Vol. 2909. https://ceur-ws.org/Vol-2909/paper5.pdf
*Verified: read in full. Supports claims about X/Y/A citation categories, exclusion of Y citations, dataset statistics (6,259,703 samples; 297,147 distinct claims; 25,340 in the stricter variant), the time-wise split at March 29 2017, the 54%/52% BERT accuracies, and the DPR in-batch rank of 1.42 with batch size 8.*
*Citation hazard: also on arXiv as 2012.13919, dated 2020. Cite the CEUR proceedings version.*

**[2] Kucer, M., Oyen, D., Castorena, J., & Wu, J. (2022).** DeepPatent: Large scale patent drawing recognition and retrieval. In *Proceedings of the IEEE/CVF Winter Conference on Applications of Computer Vision (WACV)*, pp. 2309–2318. https://openaccess.thecvf.com/content/WACV2022/papers/Kucer_DeepPatent_Large_Scale_Patent_Drawing_Recognition_and_Retrieval_WACV_2022_paper.pdf
*Verified: read in full. Supports claims about the 350,000+ design patent drawings, patent-membership supervision, the design-versus-utility distinction, RotNet 0.169 vs ImageNet 0.291 mAP, Sketchy fine-tuning degrading to 0.229, connected-component counts (705 / 2 / 244 / 673), and the database-scaling drop from 0.376 to 0.262 mAP.*
*Citation hazard: an unrelated 2018 text classification model by Li et al. in* Scientometrics *is also called DeepPatent, and is cited in this paper's own reference list. Always qualify which one you mean.*

**[3] Knappich, V., Hätty, A., Razniewski, S., & Friedrich, A. (2026).** Is It Novel and Why? Fine-Grained Patent Novelty Prediction Based on Passage Retrieval. In *Proceedings of the 49th International ACM SIGIR Conference on Research and Development in Information Retrieval (SIGIR '26)*, Melbourne, VIC, Australia, pp. 845–856. https://doi.org/10.1145/3805712.3809576
*Verified: read in full (arXiv HTML version 2605.02392v1, content matched against the ACM metadata). Supports claims about the FiNE-Patents dataset (3,658 claims, 3,163 applications, CPC G06/G10L/H04L/H04N), the 19% ESR-to-ESOP passage overlap, reference numeral counts (12.1 vs 0.3), the 75.2% claim-only BERT accuracy, the 278-sample adversarial test set, the 53.7 vs 44.7 F1 decomposition-beats-scale result, the oracle-retrieval null result, and the authors' stated open problems.*

### Consulted but not read in full

Listed for transparency. **Bibliographic details were verified; the contents were not read.** No claim in this tutorial rests on them.

**[4] Krestel, R., Chikkamath, R., Hewel, C., & Risch, J. (2021).** A survey on deep learning for patent analysis. *World Patent Information*, 65, 102035. https://doi.org/10.1016/j.wpi.2021.102035
*Peer-reviewed journal survey; the natural next reading step.*

**[5] Shomee, H. H., Wang, Z., Ravi, S. N., & Medya, S. (2025).** A Survey on Patent Analysis: From NLP to Multimodal AI. In *Proceedings of the 63rd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)*, pp. 8545–8561. https://aclanthology.org/2025.acl-long.419/
*Covers the multimodal angle relevant to open problem 4 in §6.*

### Primary sources to consult directly

Not cited above, and deliberately not paraphrased from memory. For any statutory or procedural claim, go to these rather than to this tutorial:

- **EPO Guidelines for Examination** (epo.org) — the authority on X/Y/A citation categories and European examination procedure.
- **USPTO MPEP** (uspto.gov) — US examination practice; MPEP 905.01 covers CPC structure.
- **Bulk data**: EPO Publication Server and Register, USPTO bulk data (bulkdata.uspto.gov), PatentsView.
- **Code and data for [3]**: https://github.com/boschresearch/fine-patents
