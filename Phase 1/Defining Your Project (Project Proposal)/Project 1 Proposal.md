# Project Proposal: Stress-Testing Image Retrieval on Utility-Patent Drawings

**EC601 Phase 1 · AJ Chiaravalloti**

**Note:** This Project Proposal was generated with Claude based on the instructions provided during the first class session. The "Defining Your Project" document should be taken as more authoritative than this. This document was created so that something that more closely resembled the Project Proposal task was present in the documentation. This was read and checked for accuracy, but no substantial changes were made.

**Research area:** Area 1 — Machine Learning: Vision, Multimodal & Beyond. This project follows the first Vision bullet directly: *stress-test vision-language models on a niche domain, characterize where and why they fail, then close the gap with targeted fine-tuning or retrieval.* The niche domain is utility-patent drawings.

---

## Who Is It For

The primary user is a **patent agent at a small firm running a pre-filing patentability search** on a client's electromechanical invention, with the inventor's drawings in hand and a fixed number of billable hours. The decision they make is whether the invention's structure is already disclosed in earlier patents, which determines whether the client files and how the claims are drafted to avoid what is found. **A missed reference is expensive:** the client pays to file and prosecute an application that an examiner will later reject over art that a search should have caught, or worse, a patent issues and is invalidated years later. **A false alarm is cheap:** a patent figure can be recognized as irrelevant at a glance, in seconds, far faster than reading a text passage. That asymmetry sets the design. The system optimizes **recall within the top 20 results** — the number of thumbnails an agent will realistically scan — rather than precision at rank 1, and it shows results as images so that rejecting a bad hit costs almost nothing.

---

## The Six Questions

**1. Who is the user?** A registered patent agent or attorney at a small firm, not a large firm with a dedicated search department, doing a patentability search on a mechanical or electrical invention. Technically trained, legally trained, time-constrained. They can judge relevance instantly; what they lack is a way to find visually similar prior art that uses different words.

**2. What decision do they make with the output?** Whether a prior-art figure discloses the same structure as the client's drawing. A hit changes whether to file at all, or what to claim.

**3. What does a wrong answer cost, in each direction?** A miss is high-cost and delayed: it surfaces later as an office-action rejection or an invalidity attack. A false alarm is low-cost and immediate: seconds of the agent's attention. The metric therefore weights recall heavily. Precision matters only enough to keep the top 20 worth scanning.

**4. What are the constraints of their setting?** Unfiled drawings are confidential, so they should not be sent to a third-party API. Image-retrieval models of the size used in the literature are small enough to run locally, which suits this constraint well. The agent works on a laptop and bills by the hour, so the system must return results in under a minute.

**5. Who else touches the result?** The client, who decides whether to pay for filing. And later the patent examiner, who will run their own search — increasingly with AI tools (see *Funding Agendas* below). The system's job is to surface what the examiner would find, before the examiner finds it.

**6. What exists for them today, and why isn't it enough?** Prior-art search is overwhelmingly text-based. The funded commercial products I reviewed map text prior art against claim limitations, and none of the sources I found describes figure-based search of utility patents. The USPTO has stated publicly that text and classification search alone are insufficient for visual utility applications, and is seeking an image-search tool for its own examiners [6]. A former examiner quoted in the same report attributes the gap partly to large language models handling patent figures poorly [6]. **Not yet verified:** whether free tools such as Google Patents offer usable figure search for utility patents. That is assumption A3, tested in Week 1.

---

## Problem Statement

Utility patents — where nearly all electrical and mechanical inventions are protected — contain drawings that often disclose structure more directly than their text. Yet retrieval in this domain relies primarily on text, and the one large-scale patent-drawing benchmark covers only design patents [2]. There is no public, fine-grained evaluation of how well modern image encoders retrieve relevant utility-patent figures, where they fail, or how badly the standard evaluation proxy understates their true performance.

---

## Prior Work

**PatentMatch** [1] pairs claims with examiner-cited prior-art passages from EPO search reports. During construction, every examiner reference that resolves to a figure, figure caption, or whole document is discarded. The largest text-matching dataset in the field is therefore text-only by construction, and prior art disclosed primarily through drawings is systematically excluded.

**DeepPatent** [2] provides more than 350,000 drawings from about 45,000 design patents spanning 2018 and the first half of 2019, with roughly ten drawings per patent. Positive pairs come from co-membership in the same patent — no manual labels. Three findings shape this proposal:

- **The scope is design patents only.** The authors restrict to design patents because their drawings depict the object itself, unlike utility patents, which mix in flowcharts, plots, equations, and text-heavy mechanical diagrams. Utility-patent figures are the harder, unaddressed case.
- **Pretraining choices matter in counterintuitive ways.** ImageNet-pretrained weights (0.291 mAP) beat self-supervised training on the patent data itself (0.169 mAP), and fine-tuning on free-hand sketches *hurt* performance (0.229 mAP), showing that patent drawings and sketches are different domains.
- **The evaluation proxy has an unmeasured ceiling.** Relevance is defined as "same patent," so a visually and semantically similar drawing from a *different* patent is scored as a miss. The authors show a successful retrieval in which all but one of the returned images are arguably relevant. Reported scores are therefore a lower bound, and nobody has measured by how much.

**Earlier image-based patent datasets** are small and coarse. CLEF-IP 2011 includes only 211 patents for its retrieval task and classifies figures into nine broad image types such as flowchart and chemical structure, rather than supporting fine-grained retrieval [2]. A separate concept dataset covers 1,000 drawings of eight shoe types and 2,000 mechanical drawings [2].

**FiNE-Patents** [3] is the current state of the art on the text side, evaluating novelty at the level of individual claim features. It is relevant here for two reasons. First, it documents that patent documents use reference numerals to link specific parts across the claims, description, and drawings — a hook for connecting figures to text. Second, its authors deliberately used only open-weight models because patent work demands privacy standards that are hard to guarantee through closed-model APIs, which supports the local-inference constraint above.

---

## Funding Agendas and Industry Signals

**Research funding (NSF).** NSF's Directorate for Technology, Innovation and Partnerships identified artificial intelligence as one of four primary key technology areas for near-term investment, alongside biotechnology, advanced communications, and data storage [4]. The directorate's stated emphasis is use-inspired, translational research, which this project fits.

**Government demand specific to this problem (USPTO).** The patent office is actively deploying and procuring AI search:

- Since September 2022, utility examiners have searched prior art with SimSearch, an AI-assisted similarity tool [5].
- A 2025 Automated Search Pilot uses an internal AI tool to search an application's classification codes, specification, claims, and abstract, and returns up to ten ranked documents to the applicant before examination [7].
- In May 2026 the USPTO requested information on an AI image-search tool for examiners that takes images as direct queries, citing a backlog of nearly 774,000 unexamined applications [6].

The first two tools are text-driven. The third is an explicit statement that the figure channel is not yet served.

**Industry signal (a16z and Y Combinator).** Patent AI attracted substantial venture investment in the past year. Stilta raised a $10.5 million seed round led by Andreessen Horowitz with participation from Y Combinator; in the same period Patlytics closed a $40 million Series B and Solve Intelligence a $40 million Series B [8]. Stilta's product maps prior art against each limitation of a claim in a claim chart [8]. **My reading of these signals:** capital is flowing into text-based claim charting, while the government is asking for image search. The gap between those two is where this project sits. That interpretation is mine, not a claim made by any of the sources.

---

## Proposed Approach

The project produces a **reusable evaluation**, not a product — consistent with the course's guidance that a strong Area 1 project yields an evaluation others could reuse rather than just a trained checkpoint.

**Step 1 — Build a scoped benchmark.** Select one CPC subclass whose drawings are predominantly structural rather than flowcharts. A candidate is H01R (electrical connectors), to be confirmed by the Week 1 figure-type test. DeepPatent's own construction confirms that the USPTO's weekly bulk downloads include every patent granted that week with drawings as TIF images and metadata as XML [2]; the same pipeline, filtered to utility patents in one subclass, yields the corpus. Hand-label a sample of figures by type: object depiction, schematic, flowchart, plot, other.

**Step 2 — Run baselines under the established protocol.** Evaluate several off-the-shelf image encoders (candidates include CLIP and DINOv2), a classic hand-crafted descriptor such as HOG as a floor, and a DeepPatent-style recipe — ResNet backbone, GeM pooling, triplet loss — built on the public retrieval codebase that DeepPatent's training was based on [2]. Use DeepPatent's same-patent protocol (mAP, Acc@K) so results are directly comparable to the design-patent numbers.

**Step 3 — Characterize where and why they fail.** Break every result down by figure type, and plot performance against database size. DeepPatent found mAP falling from 0.376 to 0.262 as the database grew from about 38,000 to 350,000 images [2]; this measures whether utility figures degrade faster.

**Step 4 — Measure the proxy's ceiling.** For about 50 query figures, pool the top results across all systems and judge cross-patent relevance by hand, with a second annotator on a subset to report inter-annotator agreement (Cohen's kappa). This yields the first measured number for how often "same-patent" evaluation scores a genuinely relevant retrieval as wrong — the gap DeepPatent acknowledged but did not quantify.

**Step 5 — Close the gap with one targeted intervention.** Either fine-tune on same-patent supervision within the subclass, or add text by attaching each figure's written description and reference numerals to its image embedding. Re-evaluate on both the automatic protocol and the hand-judged set.

**Compute:** DeepPatent's models are ResNet18 and ResNet50 backbones [2] — tens of millions of parameters, not billions. Inference on a single-subclass corpus is feasible on one GPU, and plausibly on a laptop for the smaller encoders.

---

## Milestones (12 Weeks)

| Weeks | Sprint | Milestone |
|---|---|---|
| 1–2 | 1 | Run validation tests A1–A3 (below). Pick the subclass. **Thin end-to-end slice:** one weekly bulk file downloaded, 100 figures extracted, one encoder run, top-20 results displayed for one query. |
| 3–4 | 2 | Full single-subclass corpus built. Figure-type labels on a sample. All baselines evaluated under the same-patent protocol. |
| 5–7 | 3 | Failure breakdown by figure type and database size. Pooled relevance judgments on ~50 queries, second annotator, agreement measured. |
| 8–10 | 4 | One targeted intervention implemented and re-evaluated on both protocols. |
| 11–12 | 5 | Benchmark, labels, and evaluation code packaged in the GitHub repo. Poster. |

### Validation before building (Week 1)

| # | Assumption | Kill criterion (written before testing) | Cheapest test |
|---|---|---|---|
| A1 | Figures in the chosen subclass are mostly structural, not flowcharts | Pivot to another subclass if under 25% are object or structural depictions; if three candidate subclasses fail, fall back to a text-side open benchmark | Pull 200 figures, label by type, count |
| A2 | Practitioners care about figure-based search | If fewer than 2 of 3 practitioners say figures matter to their searches, reframe the primary user as the patent examiner, where the USPTO's RFI documents the need | Interview 3 people — a BU technology-transfer officer, a patent agent, one more via BU alumni |
| A3 | No existing free tool already does this well | If a free tool returns the relevant figure in its top 20 for 8 of 10 test queries, reframe as an open evaluation of that tool | One hour testing Google Patents and any commercial figure search on 10 queries |

---

## What "Done" Looks Like at Week 12

1. **A public benchmark** for one utility-patent CPC subclass: corpus-build script, figure-type labels, and hand-judged cross-patent relevance for ~50 queries, with inter-annotator agreement reported.
2. **A results table** comparing at least three baselines and one intervention, broken down by figure type, alongside DeepPatent's design-patent numbers for comparison.
3. **A measured proxy ceiling:** the fraction of cross-patent retrievals scored wrong under same-patent evaluation that human judges rate relevant.
4. **A thin demo:** submit a drawing, see the top 20 prior-art figures with patent numbers and figure labels, running locally.

**Scoping rule check:** no cleanroom, no IRB (all data is public-domain patent material), no chip tapeout. Software and measurement only.

---

## References

**Research papers (read in full during Phase 1)**

[1] Risch, J., Alder, N., Hewel, C., & Krestel, R. (2021). PatentMatch: A Dataset for Matching Patent Claims & Prior Art. *Proceedings of the 2nd Workshop on Patent Text Mining and Semantic Technologies (PatentSemTech@SIGIR)*, 40–44. CEUR Workshop Proceedings Vol. 2909.

[2] Kucer, M., Oyen, D., Castorena, J., & Wu, J. (2022). DeepPatent: Large scale patent drawing recognition and retrieval. *IEEE/CVF Winter Conference on Applications of Computer Vision (WACV)*, 2309–2318.

[3] Knappich, V., Hätty, A., Razniewski, S., & Friedrich, A. (2026). Is It Novel and Why? Fine-Grained Patent Novelty Prediction Based on Passage Retrieval. *Proceedings of the 49th International ACM SIGIR Conference on Research and Development in Information Retrieval (SIGIR '26)*, 845–856.

**Funding agendas and government sources**

[4] U.S. National Science Foundation. NSF announces investment roadmap for the Technology, Innovation and Partnerships Directorate. https://www.nsf.gov/tip/updates/nsf-announces-investment-roadmap-technology-innovation

[5] U.S. Patent and Trademark Office (2025, August 14). Another USPTO AI-assisted examination tool ready for prime time. https://www.uspto.gov/about-us/news-updates/another-uspto-ai-assisted-examination-tool-ready-prime-time

[6] Will, K. S. (2026, May 20). USPTO seeking AI-driven image search tool for patent examiners. *FedScoop*. https://fedscoop.com/uspto-seeking-ai-driven-image-search-tool-for-patent-examiners/

[7] Morgan Lewis (2025, October 16). USPTO Announces Automated Search Pilot Program. https://www.morganlewis.com/pubs/2025/10/uspto-announces-automated-search-pilot-program

**Industry signals**

[8] Ambrogi, R. (2026, May 19). Stilta, a Swedish startup bringing agentic AI to patent litigation, raises $10.5M seed led by Andreessen Horowitz. *LawSites*. https://www.lawnext.com/2026/05/stilta-a-stockholm-startup-bringing-agentic-ai-to-patent-litigation-raises-10-5m-seed-led-by-andreessen-horowitz.html
