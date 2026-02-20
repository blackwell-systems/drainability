# Memory Drainability: Understanding Structural Memory Leaks

Production services leak memory even when Valgrind reports zero leaks. This project formalizes why this happens, provides measurement tools, and demonstrates how to achieve bounded memory retention.

## What This Is

Allocators using coarse-grained reclamation (slabs, arenas, epochs) can only return memory when entire granules become empty. A single long-lived allocation pins the entire granule, preventing reclamation even if 99% of objects are freed. Valgrind reports "no leaks" because individual objects are eventually freed, but the allocator cannot reclaim memory at granule boundaries.

This work:
- Formalizes when bounded retention is possible (Alignment Theorem, Pinning Growth Theorem)
- Provides instrumentation to measure drainability at runtime (<2ns overhead)
- Demonstrates enforcement via epoch-based routing (238× improvement in recycle rate)
- Validates theory on Redis (0% DSR after freeing 50% of data)

## Redis Validation: 0% Memory Reclaimed

Instrumented Redis 7.2 + jemalloc to measure drainability under realistic workload:

```bash
# Populate 100K keys (1KB each)
redis-cli DEBUG POPULATE 100000 key 1000
# Baseline: 256 slabs tracked, 0 drainable (DSR = 0%)

# Delete 50% of keys (odd keys only)
for i in $(seq 1 2 100000); do echo "DEL key:$i"; done | redis-cli --pipe
# Result: 195,641 objects freed (48% reduction in live data)
#         Still 256 slabs tracked, still 0 drainable (DSR = 0%)
#         0 slabs became reclaimable
```

The remaining 50K keys are scattered uniformly across all 256 slabs at ~813 objects/slab. Every slab is pinned by surviving allocations, preventing any from becoming fully empty and reclaimable.

This is what Valgrind cannot detect: all objects are eventually freed (no logical leaks), but structural fragmentation prevents memory reclamation at the allocator layer.

See full validation: [examples/redis](https://github.com/blackwell-systems/drainability-profiler/tree/main/examples/redis)

## Components

### Drainability Framework (Theory)

**Repository:** [drainability-framework](https://github.com/blackwell-systems/drainability-framework)
**Paper:** [DOI: 10.5281/zenodo.18653776](https://doi.org/10.5281/zenodo.18653776)

Formal framework proving when bounded memory retention is possible:

- **Alignment Theorem:** Reclaimability ⟺ Drainability at granule boundaries
- **Pinning Growth Theorem:** Sustained drainability violation → Ω(t) unbounded growth
- **Sharp dichotomy:** O(1) bounded vs Ω(t) unbounded (no intermediate regime)
- **Empirical validation:** R² ≥ 0.998 across multiple workloads

Key insight: allocation *routing* (which granule receives an allocation), not allocator *policy* (slab sizes, cache tuning), determines bounded vs unbounded retention.

The paper provides formal proofs and experimental validation showing a 238× differential in recycle rates between lifetime-aligned and lifetime-mixed allocation patterns.

### libdrainprof (Measurement)

**Repository:** [drainability-profiler](https://github.com/blackwell-systems/drainability-profiler)
**Performance:** <2ns overhead per allocation (1.97ns register, 1.77ns deregister)

Production-grade C library for runtime drainability measurement:

```c
#include <drainprof.h>

drainprof *prof = drainprof_create();

// Instrument your allocator
drainprof_granule_open(prof, granule_id);
drainprof_alloc_register(prof, granule_id, alloc_id, size);
drainprof_alloc_deregister(prof, granule_id, alloc_id);
int drainable = drainprof_granule_close(prof, granule_id);

// Read metrics
drainprof_snapshot_t snap;
drainprof_snapshot(prof, &snap);
printf("DSR: %.1f%%\n", snap.dsr * 100.0);  // 0-100%
```

Features:
- Thread-safe lock-free atomic operations
- <2ns overhead (production mode), ~25ns (diagnostic mode)
- Works with any coarse-grained allocator (jemalloc, tcmalloc, mimalloc)
- Real integration with temporal-slab allocator (validated in CI)

Answers: "Why does Valgrind say no leaks but my service leaks?"

### temporal-slab (Reference Implementation)

**Repository:** [temporal-slab](https://github.com/blackwell-systems/temporal-slab)
**Implementation:** 2,517 LOC in C

Lifetime-aware slab allocator demonstrating drainability enforcement via epoch-based routing:

```c
SlabAllocator* alloc = slab_allocator_create();

// Group allocations by temporal phase (epoch)
EpochId request_epoch = epoch_current(alloc);
void* session = slab_malloc_epoch(alloc, 256, request_epoch);
void* buffer = slab_malloc_epoch(alloc, 512, request_epoch);

// Process request...

// Advance to next epoch (previous epoch drains naturally)
epoch_advance(alloc);

// Close epoch to reclaim empty slabs
epoch_close(alloc, request_epoch);
```

Performance characteristics:
- **238× improvement:** 66.5% recycle rate (aligned) vs 0.28% (mixed)
- **Lock-free fast path:** 120ns p99, 340ns p999 (GitHub Actions AMD EPYC)
- **Bounded RSS:** 0-2.4% growth under sustained churn (2000+ cycles)
- **Thread-safe:** Lock-free allocation, scales to ~4 threads

Design highlights:
- 16-epoch ring buffer with era-based wraparound safety
- RAII-style epoch domains (nestable, thread-local, auto-close)
- Conservative deferred recycling (zero hot-path overhead)
- 24-bit generation counters for ABA protection (16M reuse budget)
- Comprehensive observability (global/class/epoch stats APIs)

## Validation Results

### Redis Integration

**Repository:** [redis-drainprof](https://github.com/blackwell-systems/redis-drainprof)

Instrumented Redis 7.2 + jemalloc 5.3.0 with symmetric fastpath hooks:

| Metric | Before Deletion | After 50% Deletion | Change |
|--------|----------------|-------------------|--------|
| Keys | 100,000 | 50,000 | -50% |
| Live objects | ~403K | ~208K | -48% |
| Slabs tracked | 256 | 256 | 0 |
| Drainable slabs | 0 (0% DSR) | 0 (0% DSR) | **0** |

Conclusion: structural fragmentation confirmed—freed memory cannot be reclaimed due to scattered surviving allocations. This demonstrates the phenomenon predicted by drainability theory on a production database.

See: [examples/redis/validation-output.txt](https://github.com/blackwell-systems/drainability-profiler/blob/main/examples/redis/validation-output.txt)

### temporal-slab Benchmarks

Experiment: 2000 allocation/deallocation cycles with different routing strategies

| Configuration | Recycle Rate | Net Slabs | RSS Behavior |
|---------------|--------------|-----------|--------------|
| Mixed lifetimes (violation) | 0.28% | ~493K (~1.9 GB) | Ω(t) unbounded growth |
| Isolated lifetimes (satisfaction) | 66.5% | ~2K (~8 MB) | O(1) bounded plateau |

Differential: 238× improvement validates drainability theory's predictions.

Sustained churn test results:
- Allocator committed bytes: 0.70 MB → 0.70 MB (0.0% drift)
- RSS stable after working set builds (no unbounded growth)

## Usage: Diagnosing Your Service

### Step 1: Install libdrainprof

```bash
git clone https://github.com/blackwell-systems/drainability-profiler
cd drainability-profiler
make && sudo make install
```

### Step 2: Instrument your allocator

```c
#include <drainprof.h>

drainprof *prof = drainprof_create();

// On granule allocation (slab/arena/epoch)
drainprof_granule_open(prof, granule_id);

// On object allocation
drainprof_alloc_register(prof, granule_id, alloc_id, size);

// On object deallocation
drainprof_alloc_deregister(prof, granule_id, alloc_id);

// On granule close/reclamation attempt
int drainable = drainprof_granule_close(prof, granule_id);
```

### Step 3: Read DSR metric

```c
drainprof_snapshot_t snap;
drainprof_snapshot(prof, &snap);
printf("DSR: %.1f%%\n", snap.dsr * 100.0);
```

Interpretation:
- **DSR > 90%:** Healthy (bounded retention likely)
- **DSR 50-90%:** Moderate fragmentation (monitor RSS trends)
- **DSR < 50%:** Severe structural leaks (unbounded growth expected)

See: [docs/tutorials/measuring-dsr.md](docs/tutorials/measuring-dsr.md) (coming soon)

## Reproducing Experiments

### Redis validation

```bash
git clone https://github.com/blackwell-systems/redis-drainprof
cd redis-drainprof
./build_with_drainprof.sh

# Run Redis with instrumentation
./src/redis-server --port 6380 --enable-debug-command yes

# Reproduce experiment
redis-cli -p 6380 DEBUG POPULATE 100000 key 1000
redis-cli -p 6380 INFO MEMORY | grep mem_drainability

for i in $(seq 1 2 100000); do echo "DEL key:$i"; done | redis-cli -p 6380 --pipe
redis-cli -p 6380 INFO MEMORY | grep mem_drainability
```

### temporal-slab benchmarks

```bash
git clone https://github.com/blackwell-systems/temporal-slab
cd temporal-slab/src
make
./churn_test           # Bounded RSS validation
./benchmark_accurate   # Allocation latency microbenchmarks
```

See: [temporal-slab/docs/](https://github.com/blackwell-systems/temporal-slab/tree/main/docs)

## Applying to Existing Allocators

The drainability framework applies to any coarse-grained allocator. You don't need to adopt temporal-slab—apply these principles to your existing allocator:

**1. Measure current DSR:**
- Instrument with libdrainprof
- Track drainability over time
- Identify problematic workloads

**2. Route allocations by lifetime:**
- Group short-lived allocations together
- Separate long-lived from short-lived
- Use application-provided hints (request scope, frame boundaries)

**3. Reclaim at phase boundaries:**
- Drain granules when phases end
- Return memory to OS via madvise/munmap
- Validate DSR improvement

Applicable to: jemalloc, tcmalloc, mimalloc, or any slab/arena-based allocator

## What Valgrind Misses

Live comparison showing what traditional leak detectors cannot find:

**Valgrind output:**
```
==1== HEAP SUMMARY:
==1==     in use at exit: 0 bytes in 0 blocks
==1==   total heap usage: 167 allocs, 167 frees
==1==
==1== All heap blocks were freed -- no leaks are possible
```

**libdrainprof output on same binary:**
```
Drainability Satisfaction Rate (DSR): 5.0%
  - 19 epochs pinned by session lifetimes
  - 1 epoch drainable (5.0%)
  - All objects freed (Valgrind confirms)
  - But epochs can't be reclaimed until sessions timeout

Conclusion: Structural leak detected!
```

See: [examples/temporal-slab](https://github.com/blackwell-systems/drainability-profiler/tree/main/examples/temporal-slab)

## Citation

If you use this work in research, please cite:

```bibtex
@techreport{blackwell2026drainability,
  title   = {Drainability: When Coarse-Grained Memory Reclamation
             Produces Bounded Retention},
  author  = {Blackwell, Dayna},
  year    = {2026},
  doi     = {10.5281/zenodo.18653776},
  license = {CC-BY-4.0}
}
```

For the software components:

```bibtex
@software{blackwell2026libdrainprof,
  author    = {Blackwell, Dayna},
  title     = {libdrainprof: Runtime Drainability Profiling Library},
  year      = {2026},
  url       = {https://github.com/blackwell-systems/drainability-profiler},
  license   = {MIT}
}

@software{blackwell2026temporalslab,
  author    = {Blackwell, Dayna},
  title     = {temporal-slab: Lifetime-Aware Slab Allocator},
  year      = {2026},
  url       = {https://github.com/blackwell-systems/temporal-slab},
  license   = {MIT}
}
```

## Impact

### For production systems
- Explains mysterious RSS growth in long-running services
- Provides diagnostic tools (DSR metric) for measuring structural leaks
- Shows path to bounded retention via lifetime-aligned routing

### For memory management research
- Formal framework with theorem proofs (Alignment, Pinning Growth)
- Reproducible validation on production database (Redis)
- Applies broadly to jemalloc, tcmalloc, mimalloc, any coarse-grained allocator

### For practitioners
- Actionable metric (DSR) for measuring memory health
- Instrumentation library (<2ns overhead, production-ready)
- Architectural guidance for achieving bounded memory

## Related Work

This work builds on and extends:
- **Slab allocation** (Bonwick, 1994) - Temporal fragmentation in object caches
- **Jemalloc** (Evans, 2006) - Arena-based allocation
- **mimalloc** (Leijen et al., 2019) - Free list sharding and delayed reuse
- **tcmalloc** (Ghemawat & Menage) - Thread-caching malloc

Our contribution: formalizing when coarse-grained reclamation produces bounded retention, providing measurement tools, and demonstrating practical enforcement.

## Documentation

- **[One-Page Summary](docs/one-pager.md)** - Printable overview
- **[Tutorial: Measuring DSR](docs/tutorials/measuring-dsr.md)** - Step-by-step guide (coming soon)

## Contributing

This is a research project, but contributions are welcome:
- **Bug reports:** File issues in respective repos
- **Validation experiments:** Share your DSR measurements from production
- **Integration examples:** Help instrument other allocators
- **Documentation improvements:** Submit PRs to any repo

## Author & Contact

**Dayna Blackwell**
Email: dayna@blackwell-systems.com
GitHub: [@blackwell-systems](https://github.com/blackwell-systems)

---

## Repository Organization

This is the landing page for the drainability project. Component repositories:

- **[drainability-framework](https://github.com/blackwell-systems/drainability-framework)** - Formal theory and research paper
- **[drainability-profiler](https://github.com/blackwell-systems/drainability-profiler)** - libdrainprof measurement library
- **[temporal-slab](https://github.com/blackwell-systems/temporal-slab)** - Reference allocator implementation
- **[redis-drainprof](https://github.com/blackwell-systems/redis-drainprof)** - Redis integration for validation

---

**Status:** All components production-ready • Paper published with DOI • Redis validation complete
