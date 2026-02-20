# Memory Drainability: One-Page Summary

**Author:** Dayna Blackwell • **Year:** 2026 • **DOI:** 10.5281/zenodo.18653776

---

## Core Discovery

Production services experience unbounded RSS growth even when Valgrind reports zero leaks. Root cause: allocator granules (slabs/arenas/epochs) get pinned by mixed object lifetimes, preventing memory reclamation at coarse boundaries.

## Redis Experiment: Validating the Theory

Instrumented Redis 7.2 + jemalloc to measure drainability:

- Populated 100K keys (1KB each) → 256 slabs tracked
- Deleted 50% of keys (odd keys only) → freed 195,641 objects
- **Result: 0 out of 256 slabs became reclaimable (0% DSR)**
- Remaining keys scattered uniformly across all slabs (~813 objects/slab)

Conclusion: structural fragmentation confirmed. Valgrind reports "all freed" (no logical leaks), but allocator cannot reclaim memory due to scattered surviving allocations.

## Theoretical Framework

**Drainability:** Property that allocator granules can reclaim memory at natural boundaries

**Alignment Theorem:** Reclaimability ⟺ Drainability at boundaries

**Pinning Growth Theorem:** Sustained violation → Ω(t) unbounded growth

**Sharp dichotomy:** O(1) bounded vs Ω(t) unbounded (no intermediate regime)

**Empirical validation:** R² ≥ 0.998, 238× recycle rate differential

---

## Three Components

### 1. Theory (drainability-framework)
- 50-page formal paper with proofs
- Published with DOI: 10.5281/zenodo.18653776
- Establishes when bounded retention is possible

### 2. Measurement (libdrainprof)
- C library, <2ns overhead per allocation
- DSR metric: 0-100% (>90% healthy, <50% severe leaks)
- Thread-safe, production-ready
- Works with any coarse-grained allocator

### 3. Reference (temporal-slab)
- 2,517 LOC implementing epoch-based routing
- 238× improvement: 66.5% recycle rate (aligned) vs 0.28% (mixed)
- Lock-free fast path: 120ns p99, 340ns p999
- Bounded RSS: 0-2.4% growth under sustained churn

---

## Key Results

| Configuration | Recycle Rate | RSS Behavior |
|---------------|--------------|--------------|
| Mixed lifetimes | 0.28% | Ω(t) unbounded |
| Aligned lifetimes | 66.5% | O(1) bounded |

**Differential:** 238× improvement validates theory

**Redis validation:** 0% DSR after 50% deletion confirms structural fragmentation

**Sustained churn test:** Allocator committed bytes: 0.70 MB → 0.70 MB (0.0% drift over 2000 cycles)

---

## Impact

### For Practitioners
- Explains mysterious production RSS growth
- DSR metric for diagnosing structural leaks
- Instrumentation library (<2ns overhead)

### For Researchers
- Formal proofs (Alignment, Pinning Growth theorems)
- Reproducible validation on Redis
- Applies to all coarse-grained allocators

### For Allocator Developers
- Architectural guidance for bounded retention
- No need to adopt temporal-slab—apply principles to existing allocators
- Works with jemalloc, tcmalloc, mimalloc

---

## Applying to Your Allocator

1. **Measure:** Instrument with libdrainprof to track current DSR
2. **Route:** Group allocations by lifetime phase (request scope, frame boundaries)
3. **Reclaim:** Drain granules at phase boundaries (madvise/munmap)
4. **Validate:** Measure DSR improvement over time

---

## What Valgrind Misses

**Valgrind:** "All heap blocks were freed -- no leaks are possible"

**libdrainprof:** "DSR: 5.0% — 19 epochs pinned, structural leak detected"

Both correct: objects are freed (no logical leaks), but epochs remain pinned (structural fragmentation).

---

## Resources

**Repositories:**
- github.com/blackwell-systems/drainability (landing page)
- github.com/blackwell-systems/drainability-framework (paper)
- github.com/blackwell-systems/drainability-profiler (libdrainprof)
- github.com/blackwell-systems/temporal-slab (reference implementation)
- github.com/blackwell-systems/redis-drainprof (validation)

**Paper:** doi.org/10.5281/zenodo.18653776

**Contact:** dayna@blackwell-systems.com

---

## Citation

```bibtex
@techreport{blackwell2026drainability,
  title   = {Drainability: When Coarse-Grained Memory Reclamation
             Produces Bounded Retention},
  author  = {Blackwell, Dayna},
  year    = {2026},
  doi     = {10.5281/zenodo.18653776}
}
```

---

**Key Takeaway:** Allocation routing (not policy) determines bounded vs unbounded retention. The Redis experiment showing 0% DSR after 50% deletion demonstrates this on a production database.
