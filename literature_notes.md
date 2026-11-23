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

This group covers the practical sorting algorithms that motivate the thesis. Together, these works explain *why* real sorting libraries combine multiple strategies, how sorting routines exploit partial order already present in the input, and why measuring only one operation (such as key comparisons) is not enough to capture real performance trade-offs.

### 1.1 Musser (1997): *Introspective Sorting and Selection Algorithms*

* **Citekey:** `Musser1997Introsort`
* **Role:** Core
* **What it is about:** Introduces **Introsort** (alongside an analogous selection algorithm, Introselect), a hybrid designed to fix Quicksort's classic weakness. Standard median-of-three Quicksort is fast on typical inputs, but Musser constructs explicit "killer" input sequences that force it into repeated lopsided splits and slow $\Theta(n^2)$ behavior. Introsort solves this by tracking the depth of recursive partitioning and switching to Heapsort whenever that depth exceeds a logarithmic limit (set empirically to $2 \lfloor \log_2 n \rfloor$ in Musser's pseudocode), guaranteeing $O(n \log n)$ worst-case time while matching Quicksort on ordinary data. To compare algorithms fairly across data types, Musser counts four separate operation categories (comparisons, assignments, pointer/iterator moves, and index arithmetic) and shows that Heapsort performs slightly fewer comparisons than Quicksort on random data yet more than twice as many total operations overall.
* **Limitations / distinctions:** Musser's worst-case theorem works for any $O(\log n)$ depth limit, so the specific depth multiplier ($2 \lfloor \log_2 n \rfloor$) and small-array insertion-sort cutoff remain empirically tuned choices rather than formally optimized parameters.
* **Relevance to thesis:** Introsort is a primary blueprint for recursive monitored policies in this thesis. Rather than hardcoding an empirically chosen depth multiplier and base-case cutoff, the framework formalizes partition-balance assumptions in Lean 4 and synthesizes cutoffs and depth limits with machine-checked worst-case resource bounds. Musser's multi-operation counting also reinforces why comparison counts alone misrank practical sorting algorithms.

### 1.2 Peters (2021 & upstream C++ implementation): *Pattern-Defeating Quicksort (`pdqsort`)*

* **Citekey:** `Peters2021Pdqsort` (paper), `Peters2021PdqsortCode` (upstream C++ implementation)
* **Role:** Core
* **What it is about:** Presents **Pattern-Defeating Quicksort (`pdqsort`)**, a widely adopted modern hybrid (used in Rust and Boost) that extends Introsort to run much faster on structured inputs such as already sorted, reverse-sorted, nearly sorted, or duplicate-heavy data without slowing down on random arrays. It introduces three main ideas: a pair of single-comparison partitioning routines that isolate repeated equal keys in $O(nk)$ time for $k$ distinct values; a fallback monitor that counts only "bad" (severely unbalanced) partitions rather than total recursion depth before switching to Heapsort, while shuffling a few elements to break adversarial patterns; and an optimistic pass that attempts a budgeted partial insertion sort whenever a partition performs no swaps and is not classified as bad, aborting quickly if too many element shifts are needed.
* **Limitations / distinctions:** In the upstream C++ implementation of `pdqsort`'s partial insertion sort, the shift limit is checked *after* finishing the insertion of each element, so a single out-of-place element can overshoot the nominal budget by up to $O(n)$ shifts before aborting. Its imbalance threshold, cutoffs, and shift cap are also hand-tuned heuristics rather than formally derived parameters.
* **Relevance to thesis:** Directly inspires the thesis's **speculative sorting** mechanism. The framework refines upstream `pdqsort`'s partial insertion sort into a strictly clamped version that prevents any budget overshoot inside the inner loop, leaves the array in a valid permutation on abort, and proves strict worst-case resource bounds inside Lean 4.

### 1.3 Brodal, Fagerberg & Moruz (2008): *On the Adaptiveness of Quicksort*

* **Citekey:** `Brodal2008AdaptivenessQuicksort`
* **Role:** Core
* **What it is about:** Examines whether standard Quicksort gets faster on nearly sorted inputs, where disorder is measured by the number of **inversions** (pairs of elements that appear in the wrong relative order). Even though Quicksort always performs $\Omega(n \log n)$ key comparisons regardless of how sorted the input already is, the authors observe experimentally that its actual running time varies by nearly a factor of two between nearly sorted and highly disordered inputs. They explain this gap theoretically by proving that randomized Quicksort performs an expected $O(n(1 + \log(1 + \mathrm{Inv}/n)))$ element swaps, which shrinks smoothly as the inversion count $\mathrm{Inv}$ drops. Hardware-counter measurements confirm that element swaps closely track both CPU branch mispredictions (misguessed conditional jumps) and total runtime, while Quicksort's cache misses barely change with disorder.
* **Limitations / distinctions:** The formal adaptive swap bound applies specifically to randomized Quicksort in the expected case (assuming distinct keys), rather than deterministic worst-case guarantees across arbitrary pivot-selection rules.
* **Relevance to thesis:** Directly motivates using a **multi-dimensional cost model** rather than counting only key comparisons. Because both adaptivity and real runtime depend heavily on memory writes and unpredictable data-dependent branches, the thesis tracks a multi-field operational event vector (loads, stores, comparisons, predictable control branches versus data-dependent branches, calls, and allocations), letting the synthesizer capture real trade-offs such as standard insertion sort versus binary insertion sort.

### 1.4 Estivill-Castro & Wood (1992): *A Survey of Adaptive Sorting Algorithms*

* **Citekey:** `EstivillCastro1992AdaptiveSurvey`
* **Role:** Supporting
* **What it is about:** A foundational survey of **adaptive sorting**, the study of algorithms whose running time improves smoothly when the input is already partially ordered. It organizes over ten mathematical measures of disorder (such as the number of inversions, the number of already-sorted segments called *runs*, and the farthest distance any element must move), explains the corresponding theoretical comparison lower bounds, and systematizes general techniques for building adaptive sorters. A recurring theme of the survey is the practical trade-off that adding extra book-keeping to adapt to more patterns can create overhead that outweighs the savings on ordinary inputs.
* **Limitations / distinctions:** Focuses on asymptotic comparison counts up to constant factors rather than fine-grained hardware operation costs, imperative memory safety, or automatic policy synthesis.
* **Relevance to thesis:** Supplies the conceptual distinction between **ghost workload properties** (like exact inversion count, which defines a nearly sorted workload class in mathematical proofs but is too expensive to compute upfront) and **observable runtime signals** (like array length, recursion depth, or a local shift counter) that a hybrid policy can check in $O(1)$ time. Its warning about adaptivity overhead also motivates explicit worst-case resource bounds on speculative passes.

### 1.5 Munro & Wild (2018): *Nearly-Optimal Mergesorts: Fast, Practical Sorting Methods That Optimally Adapt to Existing Runs*

* **Citekey:** `Munro2018Powersort`
* **Role:** Supporting
* **What it is about:** Introduces **Peeksort** and **Powersort**, two stable mergesort variants that detect already-sorted segments (*runs*) in the input and merge them in a nearly optimal order. Earlier run-adaptive sorters like TimSort relied on intricate, ad-hoc rules to decide which adjacent runs to merge, which were later shown to allow suboptimal merge orders on crafted inputs. Munro and Wild instead derive their merge rules directly from classic nearly optimal binary search tree theory, keeping only $O(\log n)$ pending runs on a stack while proving a worst-case comparison bound matching the run-entropy lower bound up to a small linear term. In Java and C++ benchmarks, both methods add virtually no overhead on random inputs while clearly outperforming standard mergesorts on partially ordered data. Powersort was later adopted as the merge policy in Python (CPython 3.11).
* **Limitations / distinctions:** Optimizes the run-merging order using a fixed handwritten base-case cutoff (switching to insertion sort for small runs), without formally verifying the implementation in a proof assistant or synthesizing cutoffs across different hardware profiles.
* **Relevance to thesis:** Serves both as a strong precedent for replacing ad-hoc hybrid control rules with mathematically derived policies, and as a natural **future extension** for adding run-adaptive merging combinators into the thesis's policy grammar.

## 2. Empirical Autotuning & Machine Learning for Sorting

This group represents the current state of the art in *automatically generating* hybrid sorting routines. These systems search over algorithmic combinations and numeric thresholds by running timed benchmarks on real hardware, achieving strong practical speedups, but without formal correctness proofs, symbolic worst-case resource bounds, or certified portability across unseen workloads.

### 2.1 Li, Garzarán & Padua (2004): *A Dynamically Tuned Sorting Library*

* **Citekey:** `Li2004DynamicallyTunedSorting`
* **Role:** Core
* **What it is about:** Builds a sorting library that automatically adapts to both the target machine and the incoming data. At installation time, the system benchmarks three algorithms (Quicksort, cache-conscious radix sort, and a cache-tuned multiway mergesort) across different parameter settings to adapt to the machine's caches and registers, and trains a lightweight linear classifier (using the Winnow learning algorithm). The authors find experimentally that relative sorting speed depends mainly on just two input properties: the number of keys and the spread of key values (measured by sampling digit-wise **entropy**), rather than the underlying statistical distribution family. At runtime, the library samples the input, uses array size and the trained classifier to pick the best algorithm, and looks up tuned merge parameters when needed, avoiding large slowdowns (up to 133% in the worst case) from picking the wrong routine.
* **Limitations / distinctions:** Selects a single sorting routine up front rather than composing nested hybrids, requires empirical benchmarking tied to specific training sizes and platforms, and offers no formal correctness proofs, worst-case resource bounds, or guarantees outside tested workloads.
* **Relevance to thesis:** One of the closest prior works in spirit: it automates the mapping from hardware and workload properties to sorting algorithm selection. Unlike this system's black-box empirical classifier and installation-time timing sweeps, the thesis derives switching policies symbolically from verified cost contracts, providing machine-checked guarantees across all valid inputs.

### 2.2 Li, Garzarán & Padua (2005): *Optimizing Sorting with Genetic Algorithms*

* **Citekey:** `Li2005SortingGeneticAlgorithms`
* **Role:** Core
* **What it is about:** Extends the authors' 2004 library from choosing a single algorithm at entry to automatically constructing multi-level hybrid sorting trees. Candidate algorithms are represented as trees built from six sorting building blocks (variants of radix sort, quicksort partitioning, and multiway merging, with base-case thresholds) and two runtime branching nodes (branching by subarray size or by sampled key entropy). Because the space of possible trees is too huge for exhaustive benchmarking, the authors use genetic programming (evolving trees via subtree crossover and mutation guided by measured runtime on training arrays) and then evolve a classifier system that assigns different hybrid trees to different regions of input size and entropy. Across seven hardware platforms, the generated hybrids outperform the best single algorithms by 36% on average and beat commercial libraries including IBM ESSL, Intel MKL, and the C++ standard library.
* **Limitations / distinctions:** Because the genetic search is heuristic and empirical, the authors explicitly note they cannot prove that the discovered hybrids are optimal. The system provides no formal correctness proofs or symbolic worst-case resource bounds, and its evolved trees reflect benchmarked training distributions.
* **Relevance to thesis:** Directly shapes the thesis's **empirical evaluation**. To measure what symbolic, proof-producing synthesis gains or loses compared to empirical autotuning, the thesis compares its Lean-synthesized policies against an empirically tuned baseline searching over the *exact same* hybrid sorting grammar on training data, then evaluates both on held-out inputs and unseen distributions.

### 2.3 Bida & Toledo (2007): *An Automatically-Tuned Sorting Library*

* **Citekey:** `Bida2007ATSL`
* **Role:** Core
* **What it is about:** Presents **ATSL**, an automatically tuned library generator that creates fast, C-callable sequential sorting routines for a given data type, machine, and compiler. ATSL defines a clean static naming notation for composing hand-tuned sorting building blocks (such as cache-aware mergesort, quicksort, samplesort, and radix variants) together with multi-threshold size switches up to tree depth 4. Search works bottom-up: ATSL first times leaf algorithms across array sizes, then times those leaves *inside* each divide-and-conquer parent (capturing how the parent changes cache state and subproblem structure), chooses the fastest candidate at each size threshold, and finally builds a top-level size selector. Across six machines and three compilers, ATSL consistently beats both standard libraries and hand-tuned research implementations, showing that even switching compilers on the same machine can change which hybrid algorithm is best.
* **Limitations / distinctions:** ATSL adapts only to input size (not input disorder or key patterns), relies entirely on finite empirical timing experiments, and provides no formal proofs of correctness, memory safety, or worst-case resource bounds.
* **Relevance to thesis:** Provides a clean point of comparison for structured hybrid sorting search over a formal composition notation. Whereas ATSL evaluates compositions empirically over finite input-size ranges, this thesis performs exhaustive structural search over a canonical policy grammar backed by symbolic cost composition and Lean 4 proofs.

### 2.4 Mankowitz et al. (2023): *Faster Sorting Algorithms Discovered Using Deep Reinforcement Learning*

* **Citekey:** `Mankowitz2023AlphaDev`
* **Role:** Supporting
* **What it is about:** Introduces **AlphaDev**, a deep reinforcement learning system (building on AlphaZero) that treats writing low-level CPU assembly code as a single-player game, adding one instruction at a time from scratch. The authors apply it to tiny fixed-size sorting routines (`sort3`, `sort4`, `sort5`, plus variants up to `sort8`) and small variable-length sorting routines (`VarSort3` to `VarSort5` that handle arrays up to a small maximum length). Candidate routines are checked on test sequences for correctness and scored using either instruction count (which closely tracks speed for branchless fixed-size routines) or actual measured CPU latency (crucial when branches are present). AlphaDev discovers clever instruction-saving tricks that shorten `sort3` and `sort5` beyond the best known human sorting-network implementations (matching `sort4`), discovers faster branching logic for `VarSort`, and had its `sort3` to `sort5` routines integrated into LLVM's standard C++ library (`libc++`).
* **Limitations / distinctions:** Targets tiny, fixed- or bounded-size assembly micro-kernels rather than general-$n$ hybrid sorting policies, checks correctness via test suites during search rather than theorem-prover synthesis, and optimizes measured latency on specific CPUs rather than symbolic worst-case resource bounds.
* **Relevance to thesis:** Highlights a clean division of labor in sorting synthesis. AlphaDev operates at the *micro-kernel* level (discovering fast low-level code for tiny sizes), whereas this thesis operates at the *policy* level (synthesizing how verified kernels are composed across all input sizes $n$). The two approaches are complementary: verified small sorting kernels of the kind AlphaDev discovers can be plugged directly into this thesis's small-input dispatch layer.

### 2.5 Rice (1976): *The Algorithm Selection Problem*

* **Citekey:** `Rice1976AlgorithmSelection`
* **Role:** Supporting
* **What it is about:** Introduces the classic conceptual framework for choosing the best algorithm for a task, breaking the problem into four parts: the *problem space* (all possible inputs), the *feature space* (compact, measurable characteristics of an input), the *algorithm space* (available methods), and the *performance space* (multiple metrics such as speed and memory). Rice also adds a *criteria space* of user-supplied weights $w$ that combine a multi-dimensional performance vector $p$ into a single score, highlighting the linear weighted sum $p \cdot w$ as the most natural rule. A key insight of the framework is that while a practical selector only gets to inspect low-dimensional features, actual performance still depends on the full input instance.
* **Limitations / distinctions:** Establishes an abstract mathematical taxonomy rather than concrete synthesis algorithms, verification methods, or recursive policy languages for sorting.
* **Relevance to thesis:** Supplies the overarching vocabulary for hybrid sorting policy synthesis (including linear weighting of multi-dimensional cost vectors), while clarifying a subtlety specific to sorting: simple observable features (like array length $n$) do not uniquely determine an algorithm's exact cost across different inputs of the same size, motivating this thesis's use of worst-case envelopes over explicitly specified, policy-independent workload classes.

## 3. Formally Verified Hybrid Sorting

This group covers works that apply interactive theorem provers or deductive verification tools to realistic, production-style hybrid sorting implementations. They demonstrate that formal guarantees for industrial sorting code are achievable, while showing that existing work verifies *fixed, handwritten* policies rather than synthesizing them.

### 3.1 Lammich (2020): *Efficient Verified Implementation of Introsort and Pdqsort*

* **Citekey:** `Lammich2020VerifiedIntrosortPdqsort`
* **Role:** Core
* **What it is about:** Uses the Isabelle proof assistant and its **Isabelle-LLVM** refinement framework to formally verify realistic, high-performance implementations of **Introsort** (modeled on the GNU C++ Standard Library) and **`pdqsort`** (modeled on the Boost C++ Libraries). Rather than proving low-level code directly from scratch, Lammich starts with abstract mathematical specifications of sorting an array slice (sorted output, exact permutation, and leaving elements outside the slice untouched) and refines them step by step into imperative LLVM code, adding support for nested data structures like arrays of strings and automatic C header generation. Benchmarked on up to $10^8$ integers and $10^7$ strings across many input distributions, the verified implementations match the speed of the original handwritten C++ libraries.
* **Limitations / distinctions:** Verifies functional correctness (not worst-case running time) of fixed, handwritten hybrid algorithms, taking their control structure, cutoffs, depth limits, and heuristic budgets as given rather than synthesizing or optimizing them.
* **Relevance to thesis:** Serves as the primary formal-methods point of comparison. This thesis takes the complementary step: instead of verifying one fixed handwritten design after the fact, it treats the hybrid control structure, cutoffs, depth limits, and speculation budgets as synthesis outputs accompanied by formal optimality and worst-case resource-bound certificates.

### 3.2 Haslbeck & Lammich (2022): *For a Few Dollars More: Verified Fine-Grained Algorithm Analysis Down to LLVM*

* **Citekey:** `Haslbeck2022VerifiedFineGrained`
* **Role:** Core
* **What it is about:** Extends Isabelle-LLVM with a formal cost-accounting mechanism based on **time credits** (a bookkeeping technique in which every low-level operation must spend a prepaid "credit" tracked in the proof), allowing both functional correctness and worst-case running-time complexity to be verified together all the way down to LLVM intermediate code. Different operation types are tracked as separate "currencies" that convert cleanly to concrete LLVM instruction costs along the refinement steps. As its main case study, the paper verifies that Lammich's Introsort implementation runs in $O(n \log n)$ worst-case time (including for non-constant-time comparisons like strings), meeting the C++11 standard library specification without slowing down the generated code.
* **Limitations / distinctions:** Proves worst-case complexity bounds for fixed, handwritten algorithms down to LLVM IR, rather than using symbolic cost models to search over and optimize a space of competing hybrid policies.
* **Relevance to thesis:** Directly informs how this thesis models low-level operational cost inside a proof assistant. Rather than verifying a single hardcoded Introsort implementation down to LLVM IR, this thesis defines an imperative micro-IR in Lean 4 that automatically counts primitive hardware-relevant events and composes verified kernel bounds to drive policy synthesis.

### 3.3 de Gouw et al. (2019): *Verifying OpenJDK's Sort Method for Generic Collections*

* **Citekey:** `DeGouw2019TimSort`
* **Role:** Supporting
* **What it is about:** Applies formal software verification (using the **KeY** verifier for Java) to OpenJDK's production implementation of **TimSort**, Java's default hybrid sorting method for objects. While trying to prove TimSort correct, the authors uncovered a real, long-standing bug in its run-merging logic (`mergeCollapse`): the code only checked the top three runs on its internal stack, which could quietly break its run-length invariant and trigger an out-of-bounds array crash on specially crafted large inputs. The same bug affected Python, Android, Apache Lucene, and Haskell (and the verification effort also spotted a latent integer-overflow edge case in binary insertion sort). The authors proposed a minimal fix checking the top four stack entries and mechanically proved **normal termination and exception-freedom (memory and bounds safety)** for both their fix and Android's alternative fix.
* **Limitations / distinctions:** After uncovering the invariant bug, the authors scaled back from full functional correctness, leaving sortedness, permutation preservation, and stability unverified, and did not address resource bounds or policy synthesis.
* **Relevance to thesis:** Provides concrete, real-world motivation for formal guarantees in hybrid sorting. Even heavily tested, widely deployed hybrid sorting policies can harbor subtle edge-case bugs in their switching, overflow, and buffer-sizing logic, reinforcing why this thesis enforces bounds safety, overflow safety, slice framing, aliasing disjointness, termination, and full sorting correctness inside Lean 4 by construction.

## 4. Program & Deductive Synthesis

This group covers core techniques for *generating* programs and algorithmic compositions automatically from specifications. It bridges empirical search (Group 2) and formal verification (Group 3), providing the architectural patterns used by this thesis's Lean 4 synthesizer.

### 4.1 Püschel et al. (2005): *SPIRAL: Code Generation for DSP Transforms*

* **Citekey:** `Pueschel2005SPIRAL`
* **Role:** Core
* **What it is about:** Presents **SPIRAL**, a landmark domain-specific code generator for digital signal processing (DSP) algorithms such as the Fast Fourier Transform (FFT). Rather than trying to synthesize arbitrary C code from scratch, SPIRAL expresses transform algorithms as structured formulas in a compact algebraic domain-specific language (**SPL**), recursively expands a transform into many alternative mathematical breakdowns, translates each formula into clean C or Fortran via templates, and uses automated search and learning to pick the fastest version for the target CPU. A striking experimental finding is that even when alternative formulas do nearly the same number of arithmetic operations (within 10%), their actual runtimes can differ by factors of 2x to 5x due to memory locality and access patterns. SPIRAL-generated code regularly matches or beats expert hand-tuned libraries like FFTW and Intel MKL.
* **Limitations / distinctions:** Targets linear numerical transforms rather than data-dependent sorting control flow, and relies on empirical benchmarking and heuristic search rather than proof-producing symbolic cost reasoning or theorem-prover certificates.
* **Relevance to thesis:** Provides the strongest architectural precedent for the thesis. Instead of attempting unrestricted general-purpose synthesis over all Lean 4 programs, the thesis restricts synthesis to a canonical, typed sorting-plan grammar built atop verified sorting kernels, replaces SPIRAL's empirical search with symbolic cost-guided optimization and Lean 4 certificates, and emits clean C code directly from the same micro-IR.

### 4.2 Drămnesc & Jebelean (2015): *Synthesis of List Algorithms by Mechanical Proving*

* **Citekey:** `Dramnesc2015ListSynthesis`
* **Role:** Core
* **What it is about:** Demonstrates automated proof-based synthesis of sorting algorithms on lists inside the **Theorema** theorem-proving system. Starting from the logical statement "for every input list, there exists a sorted permutation of it," the system constructs an inductive proof and mechanically extracts an executable functional sorting program from the proof's witness terms. Using specialized list-reasoning rules to keep the proof search manageable and varying the induction strategy, the system automatically synthesizes **five sorting algorithms** (selection sort, insertion sort, quicksort, mergesort, and a novel unbalanced mergesort). Whenever a proof gets stuck because a helper routine is missing, a **cascading** step automatically turns the stuck subgoal into a simpler synthesis task to derive auxiliary functions like `Insert`, `Merge`, and list splitting.
* **Limitations / distinctions:** Synthesizes *individual* pure functional sorting routines over linked lists from logical correctness specifications alone, without imperative array memory management, hybrid switching policies, or symbolic resource optimization.
* **Relevance to thesis:** Establishes the deductive sorting-synthesis lineage on which this thesis builds, shifting from deriving single pure sorting routines to synthesizing resource-optimal hybrid combinations of verified imperative sorting components.

### 4.3 Drămnesc & Jebelean (2021): *Synthesis of Sorting Algorithms Using Multisets in Theorema*

* **Citekey:** `Dramnesc2021MultisetSorting`
* **Role:** Core
* **What it is about:** Extends the authors' earlier Theorema sorting synthesis by formulating permutation preservation via **multiset equality** (requiring the output to contain the exact same elements with the exact same multiplicities as the input) and using strict multiset inclusion to justify termination via general well-founded (Noetherian) induction. Combined with dynamic case-splitting rules (cover sets) that avoid fixing an algorithm template in advance and automatic cascading for helper functions, a single top-level synthesis proof automatically generates **8 sorting algorithms** (including insertion sort, mergesort, bidirectional bubble sort, patience sort, and quicksort) along with **19 auxiliary functions** for inserting, splitting, and merging.
* **Limitations / distinctions:** Like the authors' 2015 work, synthesis operates purely over abstract functional lists and multisets driven solely by functional correctness, without quantitative cost models, hardware parameters, imperative array layouts, or hybrid switching policies.
* **Relevance to thesis:** Directly mirrors this thesis's functional correctness specification ($\operatorname{Sorted}(A_{\mathrm{out}}) \land \operatorname{Multiset}(A_{\mathrm{out}}) = \operatorname{Multiset}(A)$) and highlights the specific research gap this thesis targets: extending deductive sorting synthesis with formal resource modeling and hybrid policy optimization.

### 4.4 Delaware et al. (2015): *Fiat: Deductive Synthesis of Abstract Data Types in a Proof Assistant*

* **Citekey:** `Delaware2015Fiat`
* **Role:** Core
* **What it is about:** Presents **Fiat**, a framework inside the Coq proof assistant for deriving efficient, correct-by-construction implementations of abstract data types (such as database-style query tables) from high-level declarative specifications. Fiat uses **stepwise refinement**: a programmer starts with a simple, possibly nondeterministic specification describing *what* the data structure should do, and runs automated tactic scripts ("optimization scripts") that progressively replace abstract sets with concrete data structures like indexes and caches, eventually extracting executable OCaml code. Because every step is checked by Coq's proof kernel, users get **enforced modularity**: they can freely change or automate optimization heuristics without risking bugs or enlarging the trusted core.
* **Limitations / distinctions:** Targets functional refinement of relational data structures into OCaml rather than quantitative worst-case resource bounds, hardware-weight trade-offs, or imperative array sorting policies.
* **Relevance to thesis:** Demonstrates the practicality of proof-producing synthesis directly inside a dependent-type proof assistant (Coq, closely related to Lean 4). It directly informs this thesis's separation between untrusted search heuristics (which find promising candidates) and kernel-checked correctness, resource-bound, and optimality certificates in Lean 4.

### 4.5 Codish, Cruz-Filipe, Frank & Schneider-Kamp (2016): *Sorting Nine Inputs Requires Twenty-Five Comparisons*

* **Citekey:** `Codish2016SortingNetworks`
* **Role:** Core
* **What it is about:** Resolves a classic open problem (standing since 1964) about **sorting networks**, which are fixed sequences of compare-and-swap operations that sort a fixed number of inputs without any data-dependent control flow. While optimal comparator counts were already known for up to 8 inputs, proving optimality for 9 inputs was out of reach for plain Boolean satisfiability (SAT) solvers. The authors combine a symmetry-breaking search algorithm ("generate-and-prune," which discards redundant or equivalent partial networks one comparator at a time) with SAT solving to prove that no 24-comparator network can sort 9 inputs, establishing 25 comparators as optimal (verified further with independent and theorem-prover-extracted checkers). Combining this with an earlier theoretical lower bound immediately proves 29 comparators optimal for 10 inputs as well.
* **Limitations / distinctions:** Addresses static, fixed-width oblivious sorting networks rather than general-$n$ adaptive hybrid policies, and optimizes total comparator count rather than parallel network depth or dynamic dispatch overhead.
* **Relevance to thesis:** Comparator-count optimality aligns directly with this thesis's sequential event-count cost model (more closely than parallel depth optimality). In the thesis's hybrid grammar, verified small sorting networks serve as exact-size leaf building blocks behind a small-input range guard and as base-case sorters inside hybrid merges, while Codish et al. show how optimal-size networks extend to larger fixed widths.

### 4.6 Huang, Qiu, Shen & Wang (2020): *Reconciling Enumerative and Deductive Program Synthesis*

* **Citekey:** `Huang2020ReconcilingSynthesis`
* **Role:** Supporting
* **What it is about:** Introduces a **cooperative synthesis** framework (implemented in the **DryadSynth** solver) for Syntax-Guided Synthesis (SyGuS), where a synthesizer must find a program matching both a logical specification and a user-supplied grammar. Program synthesis tools typically choose between two paradigms: *deductive synthesis* (which uses logical rules to break a specification into simpler pieces quickly, when rules apply) and *enumerative search* (which systematically tries candidate programs from the grammar, working on arbitrary grammars but struggling to scale). DryadSynth combines both in a shared work queue: it uses deductive divide-and-conquer rules to split and simplify tasks whenever possible, and falls back to symbolic, height-guided enumerative search (backed by an SMT solver) on the remaining indivisible subproblems, solving significantly more benchmarks than either approach alone.
* **Limitations / distinctions:** Targets general functional synthesis over integer arithmetic using SMT-based counterexample search rather than quantitative cost optimization, unbounded-$n$ algorithmic complexity, or kernel-checked theorem-prover certificates.
* **Relevance to thesis:** Provides a close methodological precedent for this thesis's **bilevel synthesis architecture**, which splits the problem into complete in-Lean enumerative search over a finite canonical grammar of parameter-free skeletons paired with external parameter optimization and deductive Lean 4 certification lemmas.

## 5. Symbolic Cost Analysis & Hardware Models

To make synthesis *resource-aware*, the framework needs two things: a formal way to calculate and compare algorithm costs inside Lean 4, and a grounded understanding of how abstract cost models relate to real processor hardware. This group covers both.

### 5.1 Černý, Chatterjee, Henzinger, Radhakrishna & Singh (2011): *Quantitative Synthesis for Concurrent Programs*

* **Citekey:** `Cerny2011QuantitativeSynthesis`
* **Role:** Core
* **What it is about:** Introduces **quantitative synthesis** for concurrent programs. Starting from a partial program with missing synchronization choices (such as whether to use a single coarse-grained lock or finer per-element locks), a safety requirement (no data races or deadlocks), and a parametric **hardware performance model** (assigning relative costs to operations like locking, context switches, and memory or cache accesses), the synthesizer fills in the missing choices to guarantee correctness while optimizing worst-case or average-case performance. The authors reduce this problem to solving game graphs and introduce practical abstractions to keep it tractable. Across four concurrency case studies, changing the hardware cost weights automatically switches the synthesized program (for example, between coarse- and fine-grained locking), matching behavior observed on real desktop hardware.
* **Limitations / distinctions:** Studies finite-state concurrent synchronization synthesis via automata and game-theoretic methods rather than unbounded-$n$ sorting algorithms, asymptotic envelopes, or interactive theorem-prover certification.
* **Relevance to thesis:** Establishes the core synthesis paradigm of this thesis: encoding functional correctness, worst-case resource-bound constraints (such as relative worst-case cost constraints against a certified reference-cost witness), and parametric hardware-weighted cost optimization together in one formal problem, so that changing the hardware cost weights automatically yields a different certified optimal policy.

### 5.2 Guéneau, Charguéraud & Pottier (2018): *A Fistful of Dollars: Formalizing Asymptotic Complexity Claims via Deductive Program Verification*

* **Citekey:** `Gueneau2018AsymptoticComplexity`
* **Role:** Core
* **What it is about:** Builds a Coq verification framework for proving both functional correctness and **asymptotic ($O$-notation)** worst-case time bounds for imperative OCaml programs at the same time. Using separation logic extended with **time credits** (where each loop iteration and function call spends one credit), the authors formalize big-$O$ rigorously to prevent subtle flaws that commonly sneak into informal paper proofs (especially with multi-variable bounds). Rather than forcing users to guess exact cost formulas up front, interactive Coq tactics collect cost expressions and recurrence equations as the user steps through the code and discharge the final asymptotic bound at the end, demonstrated on nested loops, binary search, and Union-Find.
* **Limitations / distinctions:** Focuses on verifying coarse asymptotic big-$O$ bounds for individual programs (hiding constant factors and lower-order terms) rather than preserving explicit leading constants and second-order terms needed to optimize hybrid switching thresholds and chunk sizes.
* **Relevance to thesis:** Directly informs the thesis's **compositional cost algebra**. Building on modular deductive cost verification, this thesis preserves both leading asymptotic constants and second-order per-element excess terms (alongside exact finite-size costs) so that chunk sizes, cutoffs, and speculation parameters remain mathematically identifiable during optimization.

### 5.3 Sinn, Zuleger & Veith (2017): *Complexity and Resource Bound Analysis of Imperative Programs Using Difference Constraints*

* **Citekey:** `Sinn2017Loopus`
* **Role:** Supporting
* **What it is about:** Presents the theory and implementation of **Loopus**, an automated static analyzer that takes imperative code (via LLVM IR) and infers symbolic worst-case and amortized loop bounds without human annotations. Loopus abstracts a program into simple relational inequalities called **difference constraints** (such as $x' \le y + c$) that carefully distinguish between a counter being incremented and a counter being reset. By alternating between bounding *how often* a loop transition can run and *how large* variables can grow, Loopus infers the necessary loop invariants automatically and derives tight bounds on tricky nested and multi-path loops from real-world C code (such as parsers and OS kernel routines) where other automated analyzers lose precision.
* **Limitations / distinctions:** Targets automated loop-bound inference on non-recursive imperative code rather than recursive divide-and-conquer sorting hybrids, interactive proof-assistant certificates, or disorder-sensitive bounds like exact shift-to-inversion identities.
* **Relevance to thesis:** Defines an explicit **automation boundary** for the thesis. Automated loop-bound inference for general imperative code is a specialized research area of its own. Rather than rebuilding a standalone analyzer like Loopus, this thesis verifies tight symbolic cost contracts for a small, fixed library of sorting kernels *once* in Lean 4 and automates cost derivation *compositionally* across hybrid policy plans.

### 5.4 Hoffmann & Jost (2022): *Two Decades of Automatic Amortized Resource Analysis*

* **Citekey:** `Hoffmann2022AARA`
* **Role:** Supporting
* **What it is about:** Surveys twenty years of **Automatic Amortized Resource Analysis (AARA)**, a family of resource-aware type systems that automatically infer worst-case time or memory bounds at compile time. Based on Tarjan's potential method from amortized analysis, AARA attaches numeric "potential" annotations directly to data types (like lists or trees) so that local typing rules ensure enough stored potential is always available to pay for upcoming operations; finding the best bound then reduces to solving a linear program, and the resulting type derivation acts as a checkable proof of the bound. The survey traces how AARA grew from linear heap-space bounds for simple functional programs to polynomial, exponential, and logarithmic bounds, higher-order and imperative languages, parallel work/depth analysis, and probabilistic cost.
* **Limitations / distinctions:** Primarily embeds resource potential intrinsically into functional or specialized type systems (relying on explicit cost annotations and numeric templates), whereas hybrid imperative sorting synthesis requires exact multi-dimensional operation counts, discrete merge schedules, and optimization over structural policy grammars.
* **Relevance to thesis:** Contrasts **intrinsic** type-based cost analysis (where resource bounds are baked into the programming language's type system) with this thesis's **extrinsic** approach (where operational event counting is built automatically into the micro-IR semantics, and compositional upper bounds are proved as separate Lean 4 theorems over policy skeletons).

### 5.5 Frigo, Leiserson, Prokop & Ramachandran (1999): *Cache-Oblivious Algorithms*

* **Citekey:** `Frigo1999CacheOblivious`
* **Role:** Supporting
* **What it is about:** Introduces the **ideal-cache model** (a clean two-level memory model parameterized by cache size $Z$ and cache-line length $L$) and shows how to design **cache-oblivious** algorithms that minimize cache misses without ever knowing $Z$ or $L$. Alongside matrix multiplication and FFTs, the paper presents two cache-oblivious sorting algorithms (Funnelsort and a bucket-splitting Distribution Sort) that match the theoretical lower bound for both work and cache misses. Crucially, the authors also prove that algorithms optimal in this simple two-level ideal model remain asymptotically optimal under standard LRU replacement and across realistic multi-level cache hierarchies.
* **Limitations / distinctions:** Focuses on asymptotic cache-miss optimality for large inputs under idealized cache assumptions, rather than concrete small-array constant factors, branch-misprediction costs, or formally verified imperative sorting policies.
* **Relevance to thesis:** Clarifies an intentional modeling simplification in the thesis. While small-array cutoffs affect cache-resident subproblems, large-array sorting and merging incur substantial memory-hierarchy traffic. The thesis therefore treats its flat event cost model plus an explicit auxiliary-memory cap as a deliberate initial abstraction (whose real-world accuracy is tested empirically across sizes), rather than claiming to capture full multi-level cache behavior formally from day one.

### 5.6 Wilhelm et al. (2008): *The Worst-Case Execution-Time Problem: Overview of Methods and Survey of Tools*

* **Citekey:** `Wilhelm2008WCET`
* **Role:** Supporting
* **What it is about:** A comprehensive survey of **Worst-Case Execution Time (WCET)** analysis for safety-critical real-time systems, reviewing both static-analysis and measurement-based tools. Static WCET analyzers typically combine three stages: analyzing possible control-flow paths and loop bounds, modeling how the CPU's caches, pipelines, and branch predictors behave along those paths (often via abstract interpretation), and computing an upper bound on the longest path. A central takeaway is that on modern processors, an instruction's execution time depends heavily on execution history and hardware state; in fact, out-of-order and speculative CPUs can exhibit **timing anomalies**, where a locally faster event (such as a cache hit) actually leads to a slower overall execution. As a result, simply summing isolated instruction counts cannot guarantee exact or safe wall-clock timing on modern superscalar hardware.
* **Limitations / distinctions:** Targets binary-level timing verification of fixed real-time embedded tasks (often requiring manual loop annotations or predictable embedded processors) rather than algorithm-level policy synthesis for general-purpose sorting.
* **Relevance to thesis:** Justifies the thesis's **strict separation between formal and empirical claims**: Lean 4 certifies optimality and worst-case resource bounds with respect to the formal event model and nonnegative hardware weights, while correspondence to actual wall-clock execution time on superscalar CPUs is tested experimentally via hardware performance counters and empirical calibration. It also points to abstract cache analysis as a principled future extension for very large inputs.