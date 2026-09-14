# Fuzzing: Coverage-Guided Grey-Box Fuzzing from First Principles to Production

Name: Aarush Duvvuri
Class: EC 601
Topic: Fuzzing
Date: 09/12/2026

## Abstract

Fuzzing — repeatedly executing a program with generated inputs to find bugs — has become the dominant automated vulnerability-discovery technique in the software industry. Among its three flavors (black-box, white-box, and grey-box), coverage-guided grey-box fuzzing (CGF) dominates in practice because it couples lightweight instrumentation with a fast evolutionary loop, striking the ideal balance between depth of exploration and execution throughput. This tutorial traces CGF from its conceptual foundations through three concrete research case studies, culminating in the engineering practices that power production fuzzers today. The central takeaway is that no single fuzzer configuration wins universally: integration, engineering discipline, and careful empirical validation matter as much as any algorithmic advance  [1]  [4].

---

## 1. Introduction & Motivation

### Why Fuzzing?

Software systems have grown enormously complex, and manual testing simply cannot keep pace. Every untested code path is a potential vulnerability waiting to be exploited. Fuzzing addresses this by automating input generation at machine speed and scale.

At its core, **fuzzing** is "the execution of a Program Under Test (PUT) using inputs sampled from an input space that protrudes the expected input space of the PUT" [1]. In practice, this means throwing millions of malformed, unexpected, or boundary-crossing inputs at a program and watching what breaks.

The deployment scale is remarkable: Google, Microsoft, Adobe, and Cisco all use fuzzing as part of their secure development practices [1]. Google's OSS-Fuzz platform continuously fuzzes thousands of open-source projects. Real-world impact is equally impressive — AFL++ alone has helped the community discover CVEs in VLC, SQLite, FFmpeg, Vim, and GLIBC [4], and CollAFL identified 157 vulnerabilities in a single research evaluation, of which 134 were newly unique and 95 resulted in CVEs [2].

### Why Grey-Box Won

There are three fundamental fuzzing approaches, each representing a different trade-off:

**Black-box fuzzing** treats the PUT as an opaque function, observing only inputs and outputs. It requires no instrumentation and is trivially easy to deploy, but it is essentially blind. A completely random byte stream has almost no chance of producing a valid structured input (like a PDF or MP3 file), so the fuzzer gets stuck at shallow parsing stages and never reaches deeper, more vulnerable code.

**White-box fuzzing** (Dynamic Symbolic Execution, or DSE) goes to the opposite extreme by mathematically analyzing every instruction in the PUT to precisely solve branch conditions and generate inputs that explore new paths. The coverage depth is unmatched, but the performance overhead is prohibitive — DSE must track symbolic expressions through every instruction, making it typically slower than black-box approaches and too expensive to scale on real-world programs [1].

**Grey-box fuzzing** sits between these extremes. It instruments the PUT with lightweight coverage-tracking code (typically injected at compile time), then uses the feedback from that instrumentation to evolve a population of test inputs toward greater code coverage. It achieves near-black-box execution speed while gaining meaningful insight into program behavior [1]  [3].

### Tutorial Roadmap

This tutorial builds a complete picture of CGF in four stages:

1. **Section 2** establishes core terminology and the universal fuzzing loop.
2. **Section 3** (Case Study I: CollAFL) shows why feedback accuracy matters — AFL's 64 KB bitmap causes massive hash collisions that silently degrade coverage.
3. **Section 4** (Case Study II: Dissecting AFL) shows that even AFL's well-known components have never been rigorously ablated — and some of them don't help the way everyone assumed.
4. **Section 5** (Case Study III: AFL++) shows how a community-driven engineering effort unified a fragmented research landscape into a production-ready platform.
5. **Section 6** synthesizes the lessons and surveys open problems.

---

## 2. Background: Core Concepts

### 2.1 Key Terminology

Before we can reason about fuzzing algorithms, we need a shared vocabulary.

|Term|Definition|
|---|---|
|**Program Under Test (PUT)**|The software being tested — the "target"|
|**Seed**|A well-structured, valid input used as a mutation starting point|
|**Corpus (Seed Pool)**|The evolving collection of promising seeds|
|**Mutator**|The engine that modifies seeds to produce new test cases|
|**Instrumentation**|Code injected into the PUT (at compile time or runtime) to report execution feedback|
|**Coverage**|A metric — typically edge or block coverage — tracking which parts of the PUT were executed|
|**Bug Oracle**|The mechanism that detects policy violations (crashes, memory errors, etc.)|

### 2.2 Fuzzer Taxonomy

Fuzzers are classified by the granularity of information they observe during each execution [1]:

- **Black-box:** input/output only
- **Grey-box:** lightweight execution feedback (coverage bitmaps, hitcounts)
- **White-box:** full semantic analysis (symbolic expressions, path constraints)

### 2.3 The Universal Fuzzing Loop

All modern CGF tools, regardless of their specific innovations, implement the same six-step algorithm [1]:

```
ALGORITHM: Fuzz Testing
Input:  C (set of fuzz configurations), t_limit (time budget)
Output: B (set of discovered bugs)

B ← ∅
C ← PREPROCESS(C)           // instrument PUT, select seeds, calibrate
while elapsed < t_limit:
    conf ← SCHEDULE(C)       // pick which seed to mutate next
    tcs  ← INPUTGEN(conf)    // generate test cases by mutating conf's seed
    B', infos ← INPUTEVAL(conf, tcs, Oracle)  // run PUT, check for bugs
    C ← CONFUPDATE(C, conf, infos)  // add interesting inputs to corpus
    B ← B ∪ B'
return B
```

Each of these six functions — PREPROCESS, SCHEDULE, INPUTGEN, INPUTEVAL, CONFUPDATE, and CONTINUE — is a distinct research lever. Improving any one of them can improve fuzzer performance. The three case studies in this tutorial each focus on a different lever.

### 2.4 Walkthrough: Tracing One Input

To make this concrete, let's walk a single input through the loop for a toy C program:

```c
// toy.c
int parse(char *input) {
    if (input[0] == 'F') {        // branch A
        if (input[1] == 'U') {    // branch B
            crash();              // bug!
        }
    }
    return 0;
}
```

1. **PREPROCESS:** The compiler instruments `toy.c`, injecting counters at each branch. AFL might produce a 64 KB shared bitmap where each byte tracks a specific edge.
2. **SCHEDULE:** The fuzzer picks `"hello"` from the initial corpus — it's small and fast.
3. **INPUTGEN:** The mutator flips a bit, producing `"iello"`.
4. **INPUTEVAL:** The PUT runs. `input[0] == 'i'`, so branch A is not taken. The bitmap records edge `entry→A_false`.
5. **CONFUPDATE:** This edge was seen before. The input is discarded.

Eventually, a mutation produces `"FU..."`. Branch A is taken (new edge!), then branch B is taken (another new edge!), `crash()` fires, and the bug oracle captures it. Both `"F..."` and `"FU..."` are added to the corpus as interesting seeds.

This simple loop, running at millions of executions per second with appropriate instrumentation, is the engine behind most modern vulnerability discovery.

### 2.5 Sanitizers: Expanding the Bug Oracle

Program crashes (segfaults, aborts) are the simplest bug oracle, but many bugs don't crash programs immediately. **Sanitizers** are compiler transformations that make programs crash on bugs that would otherwise be silent [1]:

| Sanitizer                              | Detects                                                           | Overhead |
| -------------------------------------- | ----------------------------------------------------------------- | -------- |
| **ASan** (AddressSanitizer)            | Spatial/temporal memory errors (buffer overflows, use-after-free) | ~73%     |
| **MSan** (MemorySanitizer)             | Uninitialized memory reads                                        | ~150%    |
| **UBSan** (UndefinedBehaviorSanitizer) | Integer overflow, null dereference, misaligned pointers           | Low      |


The overhead is real, but so is the payoff: ASan alone can catch many memory corruptions that would otherwise remain invisible.

### 2.6 A Brief Timeline

| Year      | Milestone                                                                      |
| --------- | ------------------------------------------------------------------------------ |
| 1990      | Barton Miller coins "fuzz" and shows random inputs can crash in UNIX utilities |
| 2001      | SPIKE introduces structured (model-based) fuzzing                              |
| 2007–2008 | White-box DSE (Microsoft SAGE); early grey-box tools (EFS)                     |
| 2013      | **American Fuzzy Lop (AFL)** released — establishes CGF as the modern baseline |
| 2015      | LibFuzzer (in-process, API-level fuzzing); Google's syzkaller (kernel fuzzing) |
| 2016–2018 | AFLFast (smart scheduling), CollAFL (collision-free coverage)                  |
| 2020      | **AFL++** released — consolidates the research ecosystem                       |

---

## 3. Case Study I — CollAFL: Collision-Free Feedback

_Based on: Gan et al., "CollAFL: Path Sensitive Fuzzing," IEEE S&P 2018 [2]_

### 3.1 The Problem: AFL's Bitmap Is Lossy

AFL's coverage feedback mechanism is elegant but flawed. It maintains a shared **64 KB bitmap** between the fuzzer and the PUT, where each byte tracks one edge (a branch transition between two basic blocks). To compute which byte an edge maps to, AFL hashes the current and previous basic block IDs:

```
hash(edge A→B) = cur_block_id XOR (prev_block_id >> 1)
```

The problem is that this hash isn't injective. With a 64 KB map (65,536 slots) and potentially hundreds of thousands of edges in a large program, **two different edges can map to the same slot** — a hash collision. When this happens, the fuzzer cannot distinguish between them. It might see "edge slot 4242 was hit" without knowing whether path A or path B was executed.

Gan et al. measured the prevalence of collisions and found it alarming: **up to 75% of edges collide with at least one other edge** in large programs like `libtorrent` and `libav` [2]. The table below shows a sample:

|Application|Edges|Collision Ratio|
|---|---|---|
|libtasn1|3,820|2.72%|
|tcpdump|32,656|21.2%|
|nm (binutils)|53,652|36.06%|
|vim|153,689|61.4%|
|libav|255,212|74.85%|

### 3.2 Why Collisions Hurt

Collisions cause two distinct problems:

**Path invisibility.** When a test input triggers a truly new path, but that path collides in the bitmap with a previously seen path, the fuzzer classifies the input as "not interesting" and discards it. The new path — and any vulnerabilities lurking down it — is permanently invisible.

**Bad seed selection.** Tools like AFLFast prioritize seeds that exercise "rare" paths. But if a rare path collides with a common one, AFLFast falsely believes the rare path is well-covered and under-invests in it. Coverage inaccuracy propagates into scheduling decisions.

To visualize what a collision looks like:

```
Path P1:  A → B1 → C1 → D    (hash(B1→C1) = slot 42)
Path P2:  A → B2 → C1 → D    (hash(B2→C1) = slot 42)  ← COLLISION

Fuzzer sees:  "slot 42 was hit"
Fuzzer misses: whether P1 or P2 was taken
```

### 3.3 CollAFL's Solution: Offline CFG Analysis + Hierarchical Hashing

The naive fix — simply enlarging the bitmap — doesn't work. Increasing the map from 64 KB to 4 MB reduces collisions to ~5% but causes a 60% execution speed drop-off, as the larger map no longer fits in L2 cache [2].

CollAFL takes a different approach. It analyzes the Control Flow Graph (CFG) of the PUT offline using Clang's Link-Time Optimization (LTO) and assigns hash parameters intelligently to ensure zero collisions among known edges. Three algorithms handle different edge types:

```
Fsingle(A→B):  c               // constant, for blocks with only ONE predecessor
                                //   → zero runtime cost; hash is pre-computed
Fmul(A→B):    (cur>>x) XOR (prev>>y) + z   // parameterized, solved offline
                                             //   → same cost as AFL's formula
Fhash(A→B):   hash_table[cur][prev]          // table lookup, for unsolvable cases
                                              //   → rare; only a handful per program
```

The key insight is that **over 60% of basic blocks have exactly one predecessor** [2]. For these blocks, the incoming edge is unambiguous and can be given a unique pre-computed constant at zero runtime cost. The remaining multi-predecessor blocks are handled by the parameterized Fmul formula, with parameters chosen offline via a greedy search to guarantee no collisions. Only a tiny residual set of "unsolvable" blocks falls through to the slower Fhash table lookup.

The net result: CollAFL instruments _fewer_ instructions than AFL for most programs (on average 2.93% fewer), while achieving essentially zero collisions for all statically-known edges [2].

### 3.4 Coverage-Sensitive Seed Selection

With accurate coverage in hand, CollAFL introduces three new seed prioritization policies that directly target unexplored territory:

- **CollAFL-br (untouched-branch guided):** Prioritizes seeds whose execution path has many untouched _neighboring branches_ — implying mutations have a high chance of flipping into new code.
- **CollAFL-desc (untouched-descendant guided):** Prioritizes seeds near branches leading to large unexplored subtrees.
- **CollAFL-mem (memory-access guided):** Prioritizes seeds that touch many memory operations, increasing the chance of triggering memory corruption bugs.

### 3.5 Results

Evaluated on 24 open-source applications over 200 hours [2]:

|Configuration|Avg. New Paths (vs. AFL)|Avg. New Crashes (vs. AFL)|
|---|---|---|
|CollAFL (default policy)|+9.9%|+250%|
|CollAFL-br|+20.78%|+320%|
|CollAFL-desc|+17.23%|—|
|CollAFL-mem|+15.7%|—|

CollAFL found **157 total vulnerabilities with 95 CVEs**, compared to AFL's 51 vulnerabilities over the same period. The collision-free feedback alone accounted for a substantial portion of the gain — confirming that measurement precision is not a secondary concern but a first-order variable in fuzzer effectiveness.

**Important caveat for Section 4:** As we'll see, collision-free coverage is highly beneficial for small-to-medium programs, but for very large programs, sequential edge IDs can cause a phenomenon called "eclipsing" where overflow collisions cluster in one region of the bitmap rather than being distributed — actually hurting performance in some cases [2][3].

---

## 4. Case Study II — Dissecting AFL: What Actually Works

_Based on: Fioraldi et al., "Dissecting American Fuzzy Lop: A FuzzBench Evaluation," ACM TOSEM 2023 [3]_

### 4.1 The Problem: AFL Is a Scientific Black Box

AFL has been the baseline for hundreds of fuzzing papers. Yet its internal design choices were made over years by multiple contributors, often motivated by pragmatic or historical reasons rather than rigorous experiments. Nobody had ever isolated and tested each component individually to determine whether it actually helped.

Fioraldi et al. filled this gap by performing nine carefully controlled ablation experiments on the FuzzBench platform, evaluating each AFL component in isolation against a patched alternative.

**Experimental setup:**

- **Dataset:** 25 bug-based targets + 22 coverage-based targets on FuzzBench
- **Duration:** 23 hours per run
- **Trials:** 20 independent trials per configuration
- **Statistics:** Mann-Whitney U test for significance
- **Metric:** Average normalized score (bugs found and coverage)

### 4.2 The Nine Experiments: A Summary Table

| Component               | AFL's Choice                                         | Alternative Tested                                 | Winner                               | Key Insight                                                                                                                       |
| ----------------------- | ---------------------------------------------------- | -------------------------------------------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| **Hitcounts**           | Bucket edge counts into 8 values                     | Plain edge coverage (bit: hit/not)                 | **Edge coverage** for bugs           | Hitcounts add sensitivity but can cause corpus bloat, hurting some targets                                                        |
| **Novelty vs. Fitness** | Keep any input reaching a new edge/bucket            | Replace corpus only with fitness-maximizing inputs | **AFL novelty search**               | Novelty search is far more robust; fitness can trap the fuzzer in local maxima                                                    |
| **Corpus Culling**      | Mark a min-set as "favored"; skip others 99%         | No culling; fuzz all                               | **Culling (w/o fav_factor)**         | Culling helps, but size/speed weighting (fav_factor) can actually hurt                                                            |
| **Score Calculation**   | Complex formula weighting depth, coverage, size      | Random score in [25, 1600]                         | **Random score** (narrowly)          | Random scoring (93.52) actually edges out AFL's careful score (90.51) — AFL's energy function adds little under the bugs metric.  |
| **Corpus Scheduling**   | FIFO queue                                           | LIFO / Random                                      | **LIFO** (narrowly)                  | Fuzzing the most-recently-found seeds first often outperforms AFL's queue                                                         |
| **Splicing**            | A separate "splicing stage" at end of cycle          | Splicing as a mutation in havoc                    | **Splicing as mutation**             | More frequent cross-pollination between seeds produces more bugs                                                                  |
| **Trimming**            | Minimize each corpus entry while preserving coverage | Disable trimming                                   | **No trimming**                      | For structured inputs, trimming wastes time and destroys valid structure                                                          |
| **Timeouts**            | 5× average calibration time                          | 10× (double)                                       | **Double timeout** (not significant) | Target-dependent; cloud VMs introduce enough latency that strict timeouts are harmful                                             |
| **Collisions**          | Random edge IDs (hashed)                             | Sequential IDs (collision-free)                    | **Collision-free** (mostly)          | Collision-free is better for small programs; sequential IDs can "eclipse" large ones                                              |

### 4.3 Deeper Dive: Three Surprising Results

**Hitcounts vs. Edge Coverage**

AFL's hitcount buckets (which distinguish, e.g., "this loop ran 1 time" from "this loop ran 4 times") are supposed to help detect loop-dependent bugs. And they do — on some targets. But the increased sensitivity causes AFL to accumulate more corpus entries with similar behaviors, spending execution budget exploring redundant states. On the `grok` benchmark, edge coverage outperformed hitcounts on both bugs and coverage [3]. The recommendation is to run both variants in parallel when fuzzing in ensemble mode.

**Score Calculation: The Surprising Random Baseline**

AFL's `calculate_score` routine is a careful function of execution time, coverage depth, and whether the seed is newly discovered. Improving it has been the focus of many papers — AFLFast, EcoFuzz, and others. Yet Fioraldi et al. found that **a random score within AFL's valid range often outperforms AFL's own algorithm** [3]. This doesn't mean scheduling doesn't matter; it means AFL's scoring function is not a strong baseline. Papers claiming to beat AFL's scorer may simply be beating a weak benchmark.

**Trimming: Less Is Not More**

AFL trims corpus entries to make them smaller, assuming smaller inputs run faster and are easier to mutate productively. For binary format parsers (PDF, TIFF, JPEG), this is wrong. Removing bytes from a PDF corrupts its structure, causing the trimmed input to fail validation immediately — contributing no coverage and wasting the trimming budget. For `poppler_pdf_fuzzer`, disabling trimming entirely led to dramatically more bugs and coverage without the saturation plateau seen in vanilla AFL [3].

### 4.4 The Meta-Lesson: AFL Is Not a Reliable Baseline

The paper's most important contribution may be methodological. It demonstrates that:

1. Many AFL design choices survive for historical, not empirical, reasons.
2. Simple alternatives (random scoring, LIFO scheduling, no trimming) often match or beat AFL.
3. Target-specific behavior dominates — a feature that helps on `php` may hurt on `mruby`.
4. **AFL should not be the primary baseline for new fuzzing research.** Papers need to compare against stronger, ablated baselines like the ones introduced here.

The data also validates the recommendation for **ensemble fuzzing**: because different configurations excel on different targets, running multiple AFL variants in parallel (e.g., one with hitcounts, one without; one with trimming, one without) is better than trying to find a single optimal configuration [3].

---

## 5. Case Study III — AFL++: Integration at Production Scale

_Based on: Fioraldi et al., "AFL++: Combining Incremental Steps of Fuzzing Research," WOOT 2020 [4]_

### 5.1 The Problem: A Fragmented Ecosystem

By 2019, the fuzzing research landscape had produced dozens of valuable improvements to AFL — but each lived in its own fork, incompatible with the others. A researcher wanting to combine AFLFast's scheduling with REDQUEEN's checksum bypassing and MOpt's mutation optimization had to manually port and reconcile three separate codebases. Meanwhile, AFL itself had been unmaintained for 18 months, accumulating known bugs.

Industry practitioners faced a different version of the same problem: even if a new research technique was demonstrably better for their target, the implementation effort to integrate it was prohibitive.

AFL++ solved both problems by creating a **single, production-ready, extensible platform** that incorporates the best ideas from the research ecosystem and provides APIs for adding more [4].

### 5.2 What AFL++ Incorporates

AFL++ unified the following research contributions into one codebase:

| Component              | Source                 | What It Does                                                    |
| ---------------------- | ---------------------- | --------------------------------------------------------------- |
| **AFLFast schedules**  | Böhme et al. 2016      | Power schedules prioritizing rare paths                         |
| **MOpt**               | Lyu et al. 2019        | Particle Swarm Optimization for mutation operator probabilities |
| **REDQUEEN / CmpLog**  | Aschermann et al. 2019 | Bypass hard comparisons via Input-to-State (I2S) correspondence |
| **LAF-Intel**          | 2016                   | Split multi-byte comparisons into single-byte chains            |
| **AFLSmart**           | Pham et al. 2019       | Structure-aware mutations using Peach pit input models          |
| **InsTrim**            | Hsu et al. 2018        | Dominator-tree analysis to halve instrumentation points         |
| **NeverZero**          | AFL++ novel            | Fix hitcount overflow: counters never wrap to zero              |
| **Custom Mutator API** | AFL++ novel            | Plugin interface for arbitrary research extensions              |

### 5.3 Novel Engineering Contributions

Beyond research integration, AFL++ introduced several purely engineering innovations that make a concrete difference:

**NeverZero Counters**

AFL's bitmap entries are single bytes. If an edge is executed exactly 256 times in one run, the byte wraps to 0 — identical to "never executed." AFL then incorrectly treats this input as not covering that edge, silently corrupting its feedback. AFL++ fixes this with a one-instruction patch: after incrementing, if the byte is 0, add 1. The counter is now never zero for any edge that was actually hit [4].

**CmpLog Instrumentation**

REDQUEEN's original approach logged comparison operands using software breakpoints, which is slow. AFL++ replaces this with a 256 MB shared memory table where each comparison logs the last 256 executions of both operands, allowing the fuzzer to detect input-to-state correspondences at full execution speed [4].

**Shared-Memory Persistent Mode**

AFL's persistent mode already avoids `fork()` by looping within one process. AFL++ extends this by passing test cases via shared memory instead of the filesystem, eliminating file I/O overhead. The result is an additional 2× speedup on top of persistent mode — yielding **10–20× total speedup over basic fork mode** [4].

**Custom Mutator API**

The most architecturally significant addition is the plugin API, which lets researchers and practitioners extend AFL++ without forking it:

```c
// Minimal Custom Mutator plugin skeleton (C ABI)

// Called once at startup
void *afl_custom_init(afl_state_t *afl, unsigned int seed) {
    return my_state_init(seed);
}

// Called to generate a mutated input from `buf`
size_t afl_custom_fuzz(void *data, uint8_t *buf, size_t buf_size,
                       uint8_t **out_buf, uint8_t *add_buf,
                       size_t add_buf_size, size_t max_size) {
    // Apply your custom mutation here
    return my_mutate(buf, buf_size, out_buf, max_size);
}

// Optional: a single mutation to stack inside havoc
uint8_t *afl_custom_havoc_mutation(void *data, uint8_t *buf,
                                   size_t buf_size, size_t *new_size,
                                   size_t max_size) {
    return my_havoc_step(buf, buf_size, new_size);
}

// Called at shutdown
void afl_custom_deinit(void *data) { my_state_free(data); }
```

Plugins can be written in C or Python, and a single AFL++ instance can load multiple plugins. This allows practitioners to combine, for example, a grammar-based structural mutator with RedQueen's comparison bypassing — something previously requiring a full fork [4].

### 5.4 Results: Target-Specific Performance

AFL++ evaluated six configurations on nine FuzzBench targets (20 trials each, 24 hours, median edge coverage):

|Target|Best Configuration|Key Observation|
|---|---|---|
|**libpcap**|MOpt + RedQueen|Only RedQueen could penetrate deep comparison roadblocks|
|**mbedtls**|MOpt alone|MOpt found a cluster of new paths mid-campaign; huge late-run gain|
|**libxml2**|Ngram4|Standard edge coverage stalled; Ngram4's context-sensitive feedback helped|
|**zlib**|Ngram4 + Rare|Rare scheduling improved low-frequency path exploration|
|**harfbuzz**|MOpt + RedQueen|Synergistic win; neither alone matched the combination|
|**lcms**|RedQueen alone|MOpt _hurt_ on this target — negative interaction between the two|

The lcms result is crucial: **MOpt's positive effect on some targets actively counteracted RedQueen's positive effect on lcms** [4]. No single configuration dominated across all targets. This confirms the ensemble principle: run multiple AFL++ configurations in parallel and let coverage and bug counts guide which to invest in.

When hand-tuning AFL++ configurations for 13 of 21 FuzzBench targets, the "AFL++ Optimal" setup achieved **+7% median coverage** and outperformed all other fuzzers on FuzzBench overall [4].

---

## 6. Synthesis & Open Problems

### 6.1 The Through-Line

These four works form a coherent research arc:

- **[1] Manès et al.** provided the universal model — a common vocabulary and algorithmic skeleton that applies to all fuzzing.
- **[2] Gan et al.** (CollAFL) showed that the _precision_ of feedback, not just its presence, is a first-order variable. Inaccurate coverage corrupts scheduling decisions and hides paths.
- **[3] Fioraldi et al.** (Dissecting AFL) showed that individual components need empirical justification, not just theoretical motivation. Historical reasons are not scientific ones, and AFL is a surprisingly weak baseline.
- **[4] Fioraldi et al.** (AFL++) showed that the future lies in _consolidation and combination_ — research techniques are orthogonal and synergistic, but they need to be integrated into an extensible platform to realize their full potential.

### 6.2 The Emerging Consensus

Three principles emerge from these works and the broader literature:

**No universal configuration exists.** A technique that yields +30% coverage on libpcap may yield –5% on zlib. Configuration selection must be empirically validated per target [3][4].

**Ensemble fuzzing is a practical answer.** Running multiple configurations in parallel — some with hitcounts, some without; some with trimming, some without; some with RedQueen, some with MOpt — uses parallelism to hedge against target-specific behavior [3].

**Low-level engineering ≈ algorithmic improvement.** NeverZero counters, shared-memory persistent mode, CmpLog instrumentation — none of these involve novel algorithms, yet each delivers comparable performance gains to a well-designed research technique [4].

### 6.3 Open Problems

Despite substantial progress, important challenges remain:

**Manual per-target configuration.** AFL++ Optimal requires human expertise to choose the right combination of instrumentation, mutator, and scheduler for each target. Automating this via static analysis (e.g., detecting many `strcmp` calls as a signal to enable RedQueen) is promising but unsolved [4].

**Indirect calls and unresolved CFG edges.** CollAFL's collision mitigation only works for statically-known edges. Indirect calls (function pointers, virtual dispatch) produce edges that cannot be resolved at compile time, leaving gaps in the coverage model [2].

**Multi-core and fork() scaling.** AFL's architecture uses the filesystem for test case delivery and `fork()` for isolation — both bottlenecked by kernel locks under heavy parallelism. AFL++'s Linux Kernel Module for snapshots is a first step, but true multi-threaded fuzzing at scale remains an open engineering challenge [4].

**Crash deduplication is unsound.** Stack-hash-based deduplication (grouping crashes by call stack signature) is the practical standard, but it is known to both over-group (different bugs with identical stacks) and under-group (the same bug manifesting with different stacks). Better semantic deduplication is an active area [1].

**Checksum and parsing walls.** Many file formats enforce checksums or magic bytes at the entry point. Without bypass techniques like LAF-Intel or RedQueen, fuzzers cannot penetrate past these checks. But these bypasses don't scale perfectly to all formats [1].

**The collision-speed trade-off on large programs.** Collision-free coverage (Section 3) is beneficial for small-to-medium programs but can cause "eclipsing" [3] on very large targets — where sequential IDs cluster all collisions in one bitmap region rather than distributing them uniformly. Dynamically resizing the coverage bitmap (as AFL++ does) helps but introduces its own overhead [2][3].

**Directions for further reading:** OSS-Fuzz's continuous ensemble approach [3], QEMU-mode binary fuzzing and QASan for closed-source targets [4], and syzkaller for OS kernel fuzzing [1].

---

## 7. Conclusion

Four takeaways for practitioners and researchers:

1. **Grey-box CGF is the pragmatic sweet spot.** It achieves near-blackbox speed with meaningful coverage guidance — which is why every major software vendor uses it [1].
    
2. **Accurate feedback changes results.** AFL's 64 KB bitmap introduces up to 75% edge collisions on large programs, silently corrupting scheduling decisions and hiding paths. Precision matters [2].
    
3. **Ablate, don't assume.** Many of AFL's most-used features survive for historical reasons rather than empirical ones. Even a random energy assignment can outperform AFL's careful scoring on simple targets. Every design choice deserves an experiment [3].
    
4. **Integration and engineering unlock production power.** AFL++ demonstrates that consolidating orthogonal research advances — combined with low-level engineering like NeverZero, CmpLog, and shared-memory persistence — yields gains that no single algorithmic innovation could match alone [4].
    

Fuzzing works. It works better when we measure it rigorously, question our assumptions, and build on each other's work rather than duplicating it.

---

## 8. References

**[1]** V. J. M. Manès _et al_., “The Art, Science, and Engineering of Fuzzing: A Survey,” _IEEE Transactions on Software Engineering_, vol. 47, no. 11, pp. 2312–2331, Nov. 2021, doi: 10.1109/TSE.2019.2946563.

**[2]** S. Gan, C. Zhang, X. Qin, X. Tu, K. Li, Z. Pei, and Z. Chen, “CollAFL: Path Sensitive Fuzzing,” in _2018 IEEE Symposium on Security and Privacy (SP)_, 2018, pp. 679–696, doi: 10.1109/SP.2018.00040.

**[3]** A. Fioraldi, A. Mantovani, D. Maier, and D. Balzarotti, “Dissecting American Fuzzy Lop: A FuzzBench Evaluation,” _ACM Transactions on Software Engineering and Methodology_, vol. 32, no. 2, Art. no. 52, Mar. 2023, doi: 10.1145/3580596.

**[4]** A. Fioraldi, D. Maier, H. Eißfeldt, and M. Heuse, “AFL++: Combining Incremental Steps of Fuzzing Research,” in _Proc. 14th USENIX Workshop on Offensive Technologies (WOOT)_, 2020.