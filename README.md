# assignment

Production-ready Maven project showcasing instrumented divide-and-conquer algorithms, a CLI driver, CI automation, benchmarking, and visualization tooling.

## Learning goals
- Implement classic divide-and-conquer algorithms with careful recursion control and instrumentation.
- Capture performance metrics (time, comparisons, moves, recursion depth) and export them to CSV.
- Build reproducible CLI, benchmarking, and plotting workflows around the algorithms.
- Validate correctness and robustness with comprehensive unit tests and CI.
- Summarize theoretical complexity using Master Theorem / Akra–Bazzi style reasoning and compare to empirical trends.

#assignment-dnc/
├── pom.xml # Main module with shaded CLI JAR
├── bench/ # JMH benchmark module
├── src/main/java/org/assign1 # Algorithms, metrics, CLI
├── src/test/java/org/assign1 # JUnit 5 test suite
├── scripts/ # Helper automation
├── data/ # Results/plots directories
└── .github/workflows/ci.yml # GitHub Actions pipeline


Key components:
- **Algorithms (`org.assign1.alg`)**: MergeSort, QuickSort, DeterministicSelect (Median-of-Medians), and ClosestPair2D with reusable utilities (insertion sort cutoff, partition helpers, metrics, Fisher–Yates shuffle, deterministic random helpers).
- **Metrics (`org.assign1.alg.util.Metrics`)**: Single-threaded collector tracking comparisons, moves/allocations, recursion depth, and elapsed time. Provides CSV export (`algo,n,trial,time_ms,comparisons,moves_or_alloc,max_depth,notes`).
- **CLI (`org.assign1.cli.Main`)**: Argument-parsing runner supporting `--algo`, `--n`, `--trials`, `--seed`, `--csv`, `--k`, `--points`. Generates random arrays/points (or samples from a CSV of points), executes algorithms with metrics, appends to CSV, and prints run summaries.
- **Benchmarks (`bench/`)**: JMH harness comparing deterministic selection with `Arrays.sort`+index extraction across input sizes.
- **Automation**:
  - `scripts/run_all.sh`: builds shaded JAR, runs a grid of experiments (n ∈ {10k, 20k, 50k, 100k}, trials=3), appends metrics, and calls `scripts/gen_plots.py` to produce `data/plots/time_vs_n.png` and `data/plots/depth_vs_n.png`.
  - `scripts/gen_plots.py`: Matplotlib-based plotter averaging metrics grouped by algorithm and input size.
  - `.github/workflows/ci.yml`: Runs `mvn -B -q -e -DskipTests=false verify` and `mvn -B -q -pl bench -Pjmh -DskipTests -Djmh package` on pushes/PRs to `main` and `feature/**`.

## Architecture and algorithm notes
- **MergeSort**: Top-down stable merge sort with a single reusable buffer allocated once per sort. Uses an insertion sort cutoff (default 24) and counts moves during buffer copies and merges.
- **QuickSort**: Randomized three-way partitioning quicksort that always recurses into the smaller partition and iterates over the larger to bound recursion depth. Employs insertion sort for small subarrays.
- **DeterministicSelect**: Median-of-medians (MoM5) selection using in-place three-way partitioning. Recurses into only the necessary partition and continues iteratively in tail-recursive fashion.
- **ClosestPair2D**: Divide-and-conquer algorithm sorting points by x/y (stable order), splitting recursively, and checking the strip within 7 neighbors. Allocations for sorted views and strip buffers are tracked by metrics.

### Recursion depth controls
- MergeSort increments depth on entry to each recursive call; insertion sort cutoff keeps leaf depth bounded around `log₂ n`.
- QuickSort shuffles pivots per partition and tail-recurses on the larger half to keep depth `O(log n)` with high probability; the tests enforce `depth ≤ 2⌊log₂ n⌋ + 20` on random inputs.
- DeterministicSelect recurses only on one subproblem at a time (the smaller of the pivot partitions) and iterates otherwise, yielding `O(log n)` depth in the worst case.
- ClosestPair2D partitions the y-sorted arrays without additional recursion beyond `O(log n)`.

### Metrics semantics
- **comparisons**: Every call to `Metrics.cmp(int,int)` or `Metrics.cmp(double,double)` including comparator invocations counts exactly one comparison.
- **moves_or_alloc**: Tracks array moves (swaps, insertions) and major allocations. Swaps count as three moves; array copies and buffer allocations increment by their length.
- **max_depth**: Maximum recursion depth observed via balanced `enter()/exit()` calls.
- **time_ms**: Wall-clock elapsed time in milliseconds between `start()` and `stop()`.

### Complexity summaries
- **MergeSort**: `T(n) = 2T(n/2) + Θ(n)` → Master Theorem case 2 ⇒ `Θ(n log n)` comparisons, depth `Θ(log n)`.
- **Randomized QuickSort**: `E[T(n)] = E[T(U)] + E[T(n-1-U)] + Θ(n)` with uniform pivot ⇒ `Θ(n log n)` expected time, depth `Θ(log n)` with high probability.
- **Deterministic Select**: `T(n) = T(⌈n/5⌉) + T(7n/10 + O(1)) + Θ(n)` ⇒ `Θ(n)` via Akra–Bazzi with `a₁=1, a₂=1, b₁=5, b₂≈10/7`.
- **Closest Pair**: `T(n) = 2T(n/2) + Θ(n)` (strip considers ≤7 neighbors) ⇒ `Θ(n log n)`.

Key takeaways:
- MergeSort and QuickSort both show near-linearithmic growth, but quicksort enjoys better constants on random data thanks to cache locality despite slightly higher depth variance.
- DeterministicSelect’s linear growth dominates only for larger `n`; for small inputs the constant-factor overhead (grouping by five, multiple insertions) makes `Arrays.sort` competitive.
- Closest pair’s depth remains close to `log₂ n`, and runtime reflects the cost of sorting + strip scanning. Larger constants stem from object comparisons and temporary allocations.
- Moves/allocations reveal that MergeSort’s buffer copy cost is significant, while QuickSort’s swaps dominate for adversarial-like patterns but still stay within expected bounds.

## Usage
Build and test (Java 17 required):
```bash
mvn -q -DskipTests=false verify
