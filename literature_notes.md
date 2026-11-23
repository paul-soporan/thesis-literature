# Literature Notes

## Context: What This Thesis Is About

Modern sorting libraries rarely rely on a single textbook algorithm. Instead, they use **hybrid sorting policies** that switch between multiple algorithms depending on the input, for example using fixed sorting networks for tiny arrays, insertion sort for small subarrays, quicksort or mergesort for general inputs, heapsort when recursion grows too deep, and quick speculative passes to catch nearly sorted data.

In practice, these hybrid policies are usually built either **by hand** (relying on heuristics and hardcoded threshold numbers) or via **empirical autotuning** (benchmarking candidates on a specific machine without formal guarantees). Meanwhile, existing formal-methods literature has already established both **deductive synthesis of individual sorting algorithms** and **formal verification of realistic handwritten hybrid sorting implementations**. What remains open is bringing these capabilities together with resource-aware synthesis of hybrid policies and formally certified optimization.

This thesis investigates a **Lean 4 framework for the proof-producing synthesis of resource-aware hybrid sorting policies**. Starting from a small library of verified sorting building blocks, workload assumptions, and hardware cost weights, the goal is to synthesize both the hybrid control structure and key switching parameters (such as size cutoffs, chunk sizes, recursion-depth limits, or speculation budgets), accompanied by machine-checked certificates of functional correctness, memory safety, worst-case resource bounds, and cost optimality.

Importantly, all formal optimality guarantees are scoped to the chosen policy grammar, parameter domain, workload definitions, and abstract operational cost model. They do not claim universal optimality across every possible sorting algorithm or exact wall-clock CPU speed; how closely the formal cost model reflects real machine performance is evaluated separately through compiled benchmarks and empirical measurements.

**Legend:** Within each thematic group below, sources are ranked by direct relevance to the thesis:

* **Core:** Directly shapes the system design, algorithms, or experimental baselines.
* **Supporting:** Establishes background concepts, scope boundaries, or planned future extensions.

## 1. Hybrid & Adaptive Sorting Algorithms

This group covers the practical sorting algorithms that motivate the thesis. Together, these works explain *why* real sorting libraries combine multiple strategies, how sorting routines exploit partial order in the input, and why measuring only one operation (such as key comparisons) is not enough to capture real performance trade-offs.

### 1.1 Musser (1997): *Introspective Sorting and Selection Algorithms*

* **Citekey:** `Musser1997Introsort`
* **Role:** Core
* **What it is about:** Introduces **Introsort**, a hybrid sorting strategy whose central contribution is depth-limited Quicksort partitioning with a fallback to Heapsort whenever recursion depth exceeds a logarithmic bound. This preserves Quicksort's fast typical behavior while guaranteeing $O(n \log n)$ worst-case time.
* **Limitations / distinctions:** Although Musser's pseudocode uses $2 \lfloor \log_2 n \rfloor$ as an empirically chosen depth limit (alongside a size threshold and a single final insertion-sort pass modeled on HP's STL), his worst-case theorem holds for *any* $O(\log n)$ depth limit rather than deriving an optimal constant factor or cutoff formally. Specific depth multipliers and insertion-sort thresholds remain heuristic implementation choices.
* **Relevance to thesis:** Introsort is a primary blueprint for recursive monitored policies in this thesis. Rather than hardcoding an empirically chosen depth multiplier and base-case cutoff, the framework aims to formalize partition-balance assumptions in Lean 4 and synthesize cutoffs and depth limits with formal worst-case resource-bound proofs.

### 1.2 Peters (2021 & upstream C++ implementation): *Pattern-Defeating Quicksort (`pdqsort`)*

* **Citekey:** `Peters2021Pdqsort` (paper), `Peters2021PdqsortCode` (upstream C++ implementation)
* **Role:** Core
* **What it is about:** Presents **Pattern-Defeating Quicksort (`pdqsort`)**, a modern hybrid sorting algorithm extending Introsort to recognize and exploit common input patterns (such as nearly sorted data, reverse-sorted data, and inputs with many equal keys). Instead of tracking raw recursion depth, it counts badly unbalanced partitions to trigger the fallback, and when a partition step performs no swaps and is not classified as a bad partition, it attempts a budgeted partial insertion sort that aborts if too many element shifts occur.
* **Limitations / distinctions:** In the upstream C++ implementation of `pdqsort`'s partial insertion sort, the shift counter is updated and checked *after* finishing the insertion of the current element, so a single adversarial element can overshoot the nominal shift threshold before aborting. The design also relies on hand-tuned heuristics rather than formally synthesized parameters.
* **Relevance to thesis:** Directly inspires the thesis's **speculative sorting** mechanism. The framework refines upstream `pdqsort`'s partial insertion sort into a strictly clamped version that enforces zero shift overshoot via inner loop-bound clamping, leaves the array in a valid permutation on abort, and proves worst-case resource bounds for speculative execution inside Lean 4.

### 1.3 Brodal, Fagerberg & Moruz (2008): *On the Adaptiveness of Quicksort*

* **Citekey:** `Brodal2008AdaptivenessQuicksort`
* **Role:** Core
* **What it is about:** Analyzes whether Quicksort adapts to nearly sorted inputs measured by the number of inversions (out-of-order pairs). The authors show that Quicksort's key-comparison count is not adaptive to inversions, while its running time still exhibits empirical adaptivity (with branch mispredictions correlating closely with element swaps and dominating Quicksort's runtime), and prove an inversion-adaptive expected element-swap bound specifically for **randomized Quicksort**.
* **Limitations / distinctions:** The formal expected adaptive swap bound applies specifically to randomized Quicksort and expected-case analysis, rather than deterministic worst-case guarantees across arbitrary pivot rules.
* **Relevance to thesis:** Directly motivates using a **multi-dimensional cost model** rather than counting only key comparisons. Because both adaptivity and real runtime depend heavily on which operations are measured (including writes and data-dependent branches), the thesis models operational cost as a multi-field event vector (tracking loads, stores, comparisons, control versus data-dependent branches, calls, and allocations), allowing the synthesizer to capture genuine trade-offs such as standard insertion sort versus binary insertion sort.

### 1.4 Estivill-Castro & Wood (1992): *A Survey of Adaptive Sorting Algorithms*

* **Citekey:** `EstivillCastro1992AdaptiveSurvey`
* **Role:** Supporting
* **What it is about:** A foundational, accessible survey of adaptive sorting theory. It catalogs mathematical measures of presortedness (such as inversion count and the number of already-sorted runs) and formalizes what it means for a sorting algorithm to be optimally adaptive with respect to those measures.
* **Relevance to thesis:** Supplies the conceptual distinction between **ghost workload properties** (like exact inversion count, which defines a nearly sorted workload class in proofs but is too expensive to compute upfront) and **observable runtime signals** (like array length, recursion depth, or a local shift counter) that a hybrid policy can inspect in $O(1)$ time.

### 1.5 Munro & Wild (2018): *Nearly-Optimal Mergesorts: Fast, Practical Sorting Methods That Optimally Adapt to Existing Runs*

* **Citekey:** `Munro2018Powersort`
* **Role:** Supporting
* **What it is about:** Introduces **Peeksort** and **Powersort**, stable mergesort variants that detect already-sorted segments (*runs*) in the input and derive nearly optimal merge orders from weighted binary search tree theory, replacing the ad-hoc merge heuristics used in TimSort. Powersort was subsequently adopted as the merge policy in CPython 3.11.
* **Relevance to thesis:** Serves both as a compelling precedent for replacing heuristic hybrid control flow with mathematically derived policies, and as a natural **future extension** for adding run-adaptive merging combinators into the thesis's policy grammar.

## 2. Empirical Autotuning & Machine Learning for Sorting

This group represents the current state of the art in *automatically generating* hybrid sorting routines. These systems search over algorithmic combinations and numeric thresholds by running benchmarks on real hardware, achieving strong practical speedups, but without formal correctness proofs, symbolic worst-case resource bounds, or certified portability across unseen workloads.

### 2.1 Li, Garzarán & Padua (2004): *A Dynamically Tuned Sorting Library*

* **Citekey:** `Li2004DynamicallyTunedSorting`
* **Role:** Core
* **What it is about:** Builds a sorting library that uses empirical installation-time search and the Winnow machine-learning algorithm to predict the best sorting algorithm (among Quicksort, CC-radix sort, and a memory-hierarchy-tuned multiway mergesort) from two runtime input characteristics (input size and sampled digit-wise key entropy), while explicitly noting that broader distribution information is unnecessary.
* **Limitations / distinctions:** Establishes effective empirical selection of sorting routines from measured machine and workload features, but provides no formal correctness proofs, no worst-case resource bounds, and no symbolic optimality certificates outside tested workloads.
* **Relevance to thesis:** One of the closest prior works in spirit: it automates the mapping from hardware and workload properties to hybrid sorting decisions. Unlike this system's black-box empirical classifiers, the thesis derives switching policies symbolically from verified cost contracts, providing machine-checked guarantees across all valid inputs.

### 2.2 Li, Garzarán & Padua (2005): *Optimizing Sorting with Genetic Algorithms*

* **Citekey:** `Li2005SortingGeneticAlgorithms`
* **Role:** Core
* **What it is about:** Extends the authors' earlier library by combining genetic algorithms and a classifier system to construct hierarchically organized hybrid sorting algorithms (selecting partitioning, merging, radix passes, base-case sorters, and switching thresholds) guided directly by measured execution times on training inputs.
* **Limitations / distinctions:** Establishes automated empirical construction of hierarchical hybrid sorting trees, but does not provide formally verified correctness or symbolic worst-case resource bounds, and its learned policies are tied to benchmarked training distributions.
* **Relevance to thesis:** Directly shapes the thesis's **empirical evaluation**. To measure what symbolic, proof-producing synthesis gains or loses compared to empirical autotuning, the thesis compares its Lean-synthesized policies against an empirically tuned baseline searching over the *exact same* hybrid sorting grammar on training data, then evaluates both on held-out inputs and unseen distributions.

### 2.3 Bida & Toledo (2007): *An Automatically-Tuned Sorting Library*

* **Citekey:** `Bida2007ATSL`
* **Role:** Core
* **What it is about:** Presents **ATSL**, a library generator that searches a structured space of basic sorting algorithms, algorithmic compositions, and numeric cutoffs to produce a fast in-memory sorting routine tailored to a specific data type and target machine via timed experiments.
* **Limitations / distinctions:** Demonstrates structured grammar-like search over sorting compositions and cutoffs, but relies entirely on finite empirical timing runs and offers no formal proofs of correctness, memory safety, or worst-case resource behavior.
* **Relevance to thesis:** Provides a clean point of comparison for structured hybrid sorting search. Whereas ATSL tunes compositions empirically over finite input-size ranges, this thesis performs exhaustive structural search over a canonical policy grammar backed by Lean 4 proofs.

### 2.4 Mankowitz et al. (2023): *Faster Sorting Algorithms Discovered Using Deep Reinforcement Learning*

* **Citekey:** `Mankowitz2023AlphaDev`
* **Role:** Supporting
* **What it is about:** Introduces **AlphaDev**, a deep reinforcement learning agent that treats assembly instruction generation as a game and discovers shorter, faster fixed-size sorting routines for `sort3` and `sort5` (matching the state-of-the-art human benchmark on `sort4`, and saving instructions on `sort6`–`sort8` with a variant) as well as faster branching variable-length routines (`VarSort3`–`VarSort5`), using instruction-length proxies for branchless sorts and measured-latency rewards when branching breaks the correlation between length and latency.
* **Relevance to thesis:** Highlights a clean division of labor in sorting synthesis. AlphaDev operates at the *micro-kernel* level (discovering fast low-level assembly for tiny sizes), whereas this thesis operates at the *policy* level (synthesizing how verified kernels are composed across all input sizes $n$). The two approaches are complementary: verified small sorting kernels of the kind AlphaDev discovers can be plugged directly into this thesis's small-input dispatch layer.

### 2.5 Rice (1976): *The Algorithm Selection Problem*

* **Citekey:** `Rice1976AlgorithmSelection`
* **Role:** Supporting
* **What it is about:** Defines the classic abstract framework for selecting the best algorithm for a problem instance, separating the *problem space* (inputs), *feature space* (measurable properties), *algorithm space* (available methods), and *performance measure space*, and extending the model with a *criteria weight vector* $w$ aggregated against the performance vector $p$ via the linear dot product $p \cdot w$.
* **Relevance to thesis:** Supplies the overarching conceptual vocabulary for hybrid sorting policy synthesis (including linear criteria weighting of multi-dimensional cost vectors), while clarifying a subtlety specific to sorting: low-dimensional observable features (like array length $n$) do not uniquely determine an algorithm's exact runtime cost, motivating this thesis's use of worst-case envelopes over explicitly specified, policy-independent workload classes.

## 3. Formally Verified Hybrid Sorting

This group covers works that apply interactive theorem provers or deductive verification tools to realistic, production-style hybrid sorting implementations. They demonstrate that formal guarantees for industrial sorting code are achievable, while showing that existing work verifies *fixed, handwritten* policies rather than synthesizing them.

### 3.1 Lammich (2020): *Efficient Verified Implementation of Introsort and Pdqsort*

* **Citekey:** `Lammich2020VerifiedIntrosortPdqsort`
* **Role:** Core
* **What it is about:** Uses the Isabelle/LLVM refinement framework to formally verify implementations of both Introsort and `pdqsort` modeled closely on the GNU C++ Standard Library and Boost C++ Libraries, achieving compiled performance comparable to handwritten C++ code.
* **Limitations / distinctions:** Establishes high-performance, end-to-end formal verification of realistic handwritten hybrid sorters down to LLVM IR, but takes the hybrid control structure, cutoffs, and heuristic budgets as fixed inputs rather than synthesizing or optimizing them.
* **Relevance to thesis:** Serves as the primary formal-methods point of comparison. This thesis takes the complementary step: instead of verifying one fixed handwritten design after the fact, it treats the hybrid control structure, cutoffs, depth limits, and speculation budgets as synthesis outputs accompanied by formal optimality and resource-bound certificates.

### 3.2 Haslbeck & Lammich (2022): *For a Few Dollars More: Verified Fine-Grained Algorithm Analysis Down to LLVM*

* **Citekey:** `Haslbeck2022VerifiedFineGrained`
* **Role:** Core
* **What it is about:** Extends Isabelle/LLVM with a separation-logic-based cost model (using resource currencies and time credits) that tracks fine-grained operational costs down to LLVM IR, demonstrating verified $O(n \log n)$ worst-case complexity bounds for realistic algorithms including Introsort.
* **Limitations / distinctions:** Proves fine-grained complexity bounds for fixed handwritten algorithms down to LLVM IR, rather than using symbolic cost models to drive automated policy search and parameter optimization across a space of hybrid designs.
* **Relevance to thesis:** Directly informs how this thesis models low-level operational cost inside a proof assistant. Rather than verifying a single hardcoded Introsort implementation down to LLVM IR, this thesis defines an imperative micro-IR in Lean 4 that automatically counts primitive hardware-relevant events and composes verified kernel bounds to drive policy synthesis.

### 3.3 de Gouw et al. (2019): *Verifying OpenJDK's Sort Method for Generic Collections*

* **Citekey:** `DeGouw2019TimSort`
* **Role:** Supporting
* **What it is about:** Reports an effort to formally verify Java's default object-sorting algorithm (**TimSort**) using the KeY verifier. During the attempt, the authors uncovered a real, long-standing bug in TimSort's run-merging invariant that could trigger an uncaught `ArrayIndexOutOfBoundsException` on specially crafted inputs; they then proposed a fix and mechanically verified **normal termination and exception-freedom (bounds and null safety)** for both their fix and Android's fix, explicitly setting aside full functional correctness (sortedness, permutation preservation, and stability) for future work.
* **Relevance to thesis:** Provides concrete, real-world motivation for formal guarantees in hybrid sorting. Even heavily tested, widely deployed hybrid sorting policies can harbor subtle edge-case bugs in their switching and buffer-sizing logic, reinforcing why this thesis enforces bounds safety, slice framing, aliasing disjointness, termination, and full sorting correctness inside Lean 4 by construction.

## 4. Program & Deductive Synthesis

This group covers core techniques for *generating* programs and algorithmic compositions automatically from specifications. It bridges empirical search (Group 2) and formal verification (Group 3), providing the architectural patterns used by this thesis's Lean 4 synthesizer.

### 4.1 Püschel et al. (2005): *SPIRAL: Code Generation for DSP Transforms*

* **Citekey:** `Pueschel2005SPIRAL`
* **Role:** Core
* **What it is about:** Presents **SPIRAL**, a landmark domain-specific code generator for linear digital signal processing (such as FFTs). Rather than synthesizing arbitrary C code from scratch, SPIRAL represents algorithms in a small algebraic domain-specific language (SPL) defined by a grammar and code-generation templates, systematically searches alternative breakdowns of a transform using empirical performance optimization, and translates the winning plan into clean low-level code.
* **Limitations / distinctions:** SPIRAL relies on empirical benchmarking and search to evaluate candidate transforms rather than proof-producing symbolic cost reasoning or machine-checked correctness and optimality certificates inside a theorem prover.
* **Relevance to thesis:** Provides the strongest architectural precedent for the thesis. Instead of attempting unrestricted general-purpose synthesis over all Lean 4 programs, the thesis restricts synthesis to a canonical, typed sorting-plan grammar built atop verified sorting kernels, replaces SPIRAL's empirical search with symbolic cost-guided optimization and Lean 4 certificates, and emits clean C code from the same micro-IR.

### 4.2 Drămnesc & Jebelean (2015): *Synthesis of List Algorithms by Mechanical Proving*

* **Citekey:** `Dramnesc2015ListSynthesis`
* **Role:** Core
* **What it is about:** Demonstrates automated proof-based synthesis of list-processing and sorting algorithms (Selection-Sort, Insertion-Sort, Quick-Sort, Merge-Sort, and a novel Unbalanced-Merge-Sort, plus auxiliary functions via cascading) inside the Theorema system. Algorithms are extracted mechanically from constructive induction proofs showing that a sorted permutation exists for every input list.
* **Limitations / distinctions:** Focuses on synthesizing *individual* pure functional sorting routines from logical correctness specifications, without imperative array memory management, hybrid switching policies, or symbolic resource optimization.
* **Relevance to thesis:** Establishes the deductive sorting-synthesis lineage on which this thesis builds, shifting from deriving single pure sorting routines to synthesizing resource-optimal hybrid combinations of verified imperative sorting components.

### 4.3 Drămnesc & Jebelean (2021): *Synthesis of Sorting Algorithms Using Multisets in Theorema*

* **Citekey:** `Dramnesc2021MultisetSorting`
* **Role:** Core
* **What it is about:** Extends mechanical constructive synthesis in Theorema by combining multiset equality (requiring the output to contain the exact same elements and multiplicities as the input), Noetherian induction, and automatic generation of cascading auxiliary functions, successfully synthesizing eight sorting algorithms and nineteen auxiliary functions.
* **Limitations / distinctions:** Like the authors' 2015 work, synthesis is driven purely by functional correctness over abstract lists and multisets rather than quantitative cost models, hardware parameters, or hybrid switching policies.
* **Relevance to thesis:** Directly mirrors this thesis's functional correctness specification ($\operatorname{Sorted}(A_{\mathrm{out}}) \land \operatorname{Multiset}(A_{\mathrm{out}}) = \operatorname{Multiset}(A)$) and highlights the specific research gap this thesis targets: extending deductive sorting synthesis with formal resource modeling and hybrid policy optimization.

### 4.4 Delaware et al. (2015): *Fiat: Deductive Synthesis of Abstract Data Types in a Proof Assistant*

* **Citekey:** `Delaware2015Fiat`
* **Role:** Core
* **What it is about:** Presents **Fiat**, a Coq framework for deriving efficient, correct-by-construction implementations of abstract data types (such as SQL-like query structures) through stepwise refinement inside an interactive proof assistant, automating routine refinement derivations via tactic-based optimization scripts (honing tactics) while guaranteeing that the generated code satisfies its formal specification.
* **Limitations / distinctions:** Targets functional refinement of relational data structures rather than quantitative worst-case resource optimization, hardware-weight trade-offs, or imperative array sorting policies.
* **Relevance to thesis:** Demonstrates the practicality of proof-producing synthesis directly inside a dependent-type proof assistant (Coq, closely related to Lean 4). It informs this thesis's separation between untrusted search heuristics (which find promising candidates) and kernel-checked correctness and optimality certificates in Lean 4.

### 4.5 Codish, Cruz-Filipe, Frank & Schneider-Kamp (2016): *Sorting Nine Inputs Requires Twenty-Five Comparisons*

* **Citekey:** `Codish2016SortingNetworks`
* **Role:** Core
* **What it is about:** Combines a **generate-and-prune** symmetry-breaking procedure with SAT solving to prove that sorting 9 inputs with an oblivious *sorting network* requires at least 25 comparators (closing the smallest open instance of the optimal-size problem, extending earlier results up to 8 inputs), and obtains optimality of Waksman's 29-comparator network for 10 inputs as a corollary via Van Voorhis's lower-bound inequality.
* **Limitations / distinctions:** Addresses static, fixed-width oblivious sorting networks rather than general-$n$ adaptive hybrid policies, and optimizes total comparator count rather than parallel network depth.
* **Relevance to thesis:** Comparator-count optimality aligns directly with this thesis's sequential event-count cost model (more closely than parallel depth optimality). In the thesis's hybrid grammar, verified small sorting networks serve as exact-size leaf building blocks behind a small-input range guard and as base-case sorters inside hybrid merges, while Codish et al. show how optimal-size networks extend to larger fixed widths.

### 4.6 Huang, Qiu, Shen & Wang (2020): *Reconciling Enumerative and Deductive Program Synthesis*

* **Citekey:** `Huang2020ReconcilingSynthesis`
* **Role:** Supporting
* **What it is about:** Introduces a cooperative Syntax-Guided Synthesis (SyGuS) framework (implemented in DryadSynth) that bridges two major synthesis paradigms: systematic *enumerative search* over a grammar and *deductive synthesis* that decomposes specifications into simpler subproblems, letting each method handle the parts it solves best.
* **Limitations / distinctions:** Targets general SyGuS functional synthesis problems using SMT-backed CEGIS rather than quantitative cost optimization or kernel-checked theorem-prover certificates.
* **Relevance to thesis:** Provides a close methodological precedent for this thesis's **bilevel synthesis architecture**, which splits the problem into complete in-Lean enumerative search over a finite canonical grammar of parameter-free skeletons paired with external parameter optimization and deductive Lean 4 certification lemmas.

## 5. Symbolic Cost Analysis & Hardware Models

To make synthesis *resource-aware*, the framework needs two things: a formal way to calculate and compare algorithm costs inside Lean 4, and a grounded understanding of how abstract cost models relate to real processor hardware. This group covers both.

### 5.1 Černý, Chatterjee, Henzinger, Radhakrishna & Singh (2011): *Quantitative Synthesis for Concurrent Programs*

* **Citekey:** `Cerny2011QuantitativeSynthesis`
* **Role:** Core
* **What it is about:** Introduces a formal framework for **quantitative synthesis**, where a synthesizer resolves missing choices in a partial program not only to guarantee functional correctness, but also to optimize a parametric performance model representing different hardware architectures (assigning weights to operations such as memory accesses or synchronization).
* **Limitations / distinctions:** Studies finite-state concurrent synchronization synthesis via automata/game-theoretic techniques rather than unbounded-$n$ sorting algorithms or interactive theorem-prover certification.
* **Relevance to thesis:** Establishes the core synthesis paradigm of this thesis: encoding functional correctness, worst-case resource-bound constraints (such as relative worst-case cost constraints against a certified reference-cost witness), and parametric hardware-weighted cost optimization together in one formal problem, so that changing the hardware cost weights automatically yields a different certified optimal policy.

### 5.2 Guéneau, Charguéraud & Pottier (2018): *A Fistful of Dollars: Formalizing Asymptotic Complexity Claims via Deductive Program Verification*

* **Citekey:** `Gueneau2018AsymptoticComplexity`
* **Role:** Core
* **What it is about:** Develops a framework inside Coq for proving asymptotic ($O$-notation) execution-time bounds for imperative programs using separation logic and time credits, handling the mathematical subtleties of asymptotic inequalities so that cost proofs remain modular and compositional.
* **Limitations / distinctions:** Focuses on verifying asymptotic big-$O$ bounds for individual algorithms rather than distinguishing leading constants from second-order terms to optimize hybrid switching thresholds.
* **Relevance to thesis:** Directly informs the thesis's **compositional cost algebra**. Building on modular deductive cost verification, this thesis tracks both leading asymptotic terms and second-order per-element excess terms (alongside exact finite-size costs) so that chunk sizes, cutoffs, and speculation parameters remain mathematically identifiable during optimization.

### 5.3 Sinn, Zuleger & Veith (2017): *Complexity and Resource Bound Analysis of Imperative Programs Using Difference Constraints*

* **Citekey:** `Sinn2017Loopus`
* **Role:** Supporting
* **What it is about:** Presents the theoretical foundations behind **Loopus**, a static analyzer that automatically infers loop iteration bounds and amortized complexity bounds for a substantial class of imperative resource-bound problems (including challenging nested and multi-path loop patterns) by abstracting programs into difference-constraint systems.
* **Relevance to thesis:** Defines an explicit **automation boundary** for the thesis. Automated loop-bound inference for general imperative code is a specialized research area of its own. Rather than rebuilding a standalone analyzer like Loopus, this thesis verifies tight symbolic cost contracts for a small, fixed library of sorting kernels *once* in Lean 4 and automates cost derivation *compositionally* across hybrid policy plans.

### 5.4 Hoffmann & Jost (2022): *Two Decades of Automatic Amortized Resource Analysis*

* **Citekey:** `Hoffmann2022AARA`
* **Role:** Supporting
* **What it is about:** A readable survey of Automatic Amortized Resource Analysis (**AARA**), a family of resource-aware type systems that automatically infer worst-case time and space bounds by embedding potential functions into program types and solving numeric constraint systems.
* **Relevance to thesis:** Contrasts **intrinsic** type-based cost analysis (where resource bounds are baked into the programming language's type system) with this thesis's **extrinsic** approach (where operational event counting is built into the micro-IR semantics, and compositional upper bounds are proved as separate Lean 4 theorems over policy skeletons).

### 5.5 Frigo, Leiserson, Prokop & Ramachandran (1999): *Cache-Oblivious Algorithms*

* **Citekey:** `Frigo1999CacheOblivious`
* **Role:** Supporting
* **What it is about:** Introduces the **ideal-cache model** (a two-level memory hierarchy parameterized by cache size and cache-line length) and shows how algorithms, including sorting, can be analyzed for cache misses without tuning to specific hardware cache sizes.
* **Relevance to thesis:** Clarifies an intentional modeling simplification in the thesis. While small-array cutoffs affect small subproblems, large-array sorting and merging incur substantial memory-hierarchy traffic. The thesis therefore treats its flat event cost model plus an explicit auxiliary-memory cap as a deliberate initial abstraction (whose real-world accuracy is tested empirically across sizes), rather than claiming to capture full multi-level cache behavior formally from day one.

### 5.6 Wilhelm et al. (2008): *The Worst-Case Execution-Time Problem: Overview of Methods and Survey of Tools*

* **Citekey:** `Wilhelm2008WCET`
* **Role:** Supporting
* **What it is about:** A comprehensive survey of Worst-Case Execution Time (**WCET**) analysis, explaining how static timing analyzers abstract complex processor hardware (such as pipelines, branch predictors, and caches via abstract-interpretation analyses classifying memory accesses as definite hits or potential misses) and why exact wall-clock timing on modern out-of-order CPUs cannot be read directly from raw instruction counts.
* **Relevance to thesis:** Justifies the thesis's **strict separation between formal and empirical claims**: Lean 4 certifies optimality and worst-case resource bounds with respect to the formal event model and nonnegative hardware weights, while correspondence to actual wall-clock execution time on superscalar CPUs is tested experimentally via hardware performance counters and empirical calibration. It also points to abstract cache analysis as a principled future extension for very large inputs.