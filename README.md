# 1BRC Polyglot

Polyglot exploration of the One Billion Row Challenge: fastest non-Java implementations, benchmarked and mined for techniques to improve [gruppera](https://github.com/andrey-usa/gruppera).

## Benchmark results (100M rows, 2-core AMD EPYC 9D25, warm cache)

| Implementation | Time | vs gruppera | Correct |
|---|---|---|---|
| **gruppera** (ours) | **0.9s** | — | ✓ |
| rust-apex | 1.9s | 2.1x slower | ✓ |
| tieliao C | 2.5s | 2.8x slower | ✓ |
| agent C++ | 3.2s | 3.6x slower | ✓ |
| agent Rust | 3.1s | 3.4x slower | ✓ |

All outputs byte-identical (sha256 `66ce8d6e...`).

## Benchmark results (1B rows, 13GB, 2-core EPYC 9D25, I/O-bound)

The 13GB file exceeds the 7GB RAM, so this tests I/O strategy, not just CPU.

| Implementation | Time | vs best | Correct |
|---|---|---|---|
| **agent C++** | **69s** | — | ✓ |
| gruppera | 81s | 1.17x slower | ✓ |
| tieliao C | 91s | 1.32x slower | ✓ |
| rust-apex | 175s | 2.5x slower | ✓ |

**Key insight**: On I/O-bound 1B, the agent C++ wins with its `F_NOCACHE` streaming design (single reader + workers). Gruppera's mmap is slower when the file doesn't fit in page cache. For the 200M CI case (2.6GB, fits in cache), gruppera wins. The optimal I/O strategy depends on the dataset size vs RAM.

## Implementations

### rust-apex (`rust-apex/`)

Source: [bonnhatnguyen/1brc-rust-apex](https://github.com/bonnhatnguyen/1brc-rust-apex)

Claims 341M rows/sec (0.293s/100M) with AVX2 SIMD, SWAR, work-stealing.

**Measured**: ~1.9s (2.1x slower than gruppera on 2-core VM). The 341M rows/sec claim does not reproduce on this hardware.

**Techniques worth mining:**
1. **AVX2 delimiter scan**: `_mm256_cmpeq_epi8` finds `;` in 32 bytes. **Tested in gruppera: neutral (1.002x)** — delimiter finding is not the bottleneck. Reverted.
2. **2-way ILP** (not 3-way): Our 3-way is better (4-way tested, lost).
3. **16K hash table** (vs our 128K): We tested 32K (neutral), 8K (worse).
4. **Multiplicative hash with rotates**: Different from our xor-fold; not tested in isolation.
5. **Prefix caching**: stores `[u64; 3]` for fast compare.

### tieliao C (`tieliao/`)

Source: [tieliao/1brc](https://github.com/tieliao/1brc)

Claims 0.39s on i7-9750H laptop (2x faster than Java winners on same hardware).

**Measured** (2-core EPYC 9D25): 2.5s (2.8x slower than gruppera). The laptop result doesn't transfer to server CPUs.

**Techniques:** Standard mmap + threads + hash table. Nothing we don't already do.

### agent C++/Rust (`agent/`)

Source: [huicongyao/1brc-for-agent](https://github.com/huicongyao/1brc-for-agent)

Designed for 1B rows that DON'T fit in page cache (I/O-bound). Uses `read(2)` with `F_NOCACHE` instead of mmap, single reader thread + workers.

**Measured**: 3.1-3.2s (3.4-3.6x slower than gruppera). The `F_NOCACHE` hurts when data fits in page cache (our 100M/200M case). Not relevant for our workload, but the SWAR parsing techniques are sound.

## Benchmark protocol

- Dataset: official 1BRC generator, 100M rows (~1.38 GB)
- Hardware: 2-core AMD EPYC 9D25 (this VM) or 4-core EPYC 7763 (CI)
- Cache: warm (one warmup + best of N)
- Correctness: sha256 of output must match gruppera's `66ce8d6eb2b9f14b57f08912c15720f772c79e2f09f43b07a3c605c20b7734c7`

## Research findings (2026-10-04)

### SWAR
The classic haszero trick (`x ^ mask; sub 0x0101...; & !x & 0x8080...`) is 5 ops per 8 bytes and is optimal. No fewer-op variant exists without false positives. Our implementation uses it correctly.

### SIMD
- **AVX2** (32-byte): Tested, **neutral** (1.002x). The 4x throughput doesn't translate to wall-time wins for short delimiter scans.
- **SSE2** (16-byte): Tested, **4% slower** than SWAR. Higher latency (load+compare+movemask) outweighs the op-count reduction.
- **AVX-512**: Literature shows 7-17% slower than AVX2 for parsing workloads (Zen 4 splits 512-bit ops, memory-bound). Not worth trying.
- **Conclusion**: SWAR is optimal for 1BRC's short station names (avg ~10 chars). SIMD only wins for longer scans (>29 bytes per rusty_json_turbo).

### CPU Branching
Research confirms: **branchless (cmov) is slower than a correctly-predicted branch** because cmov always takes the data dependency.

Our hot-loop branches are all highly predictable:
- `if (m0|m1) != 0` (fast path): 99%+ taken — predictable
- `if e.used && fp match` (hash hit): 99%+ taken after warmup — predictable  
- `if v < lo` / `if v > hi` (min/max): 99.995% not-taken (only ~12 updates per 242K rows) — predictable
- `while c.live()` (loop): predictable

**Conclusion**: Our branches are already optimal. Making them branchless would hurt.

### Overall
Gruppera's SWAR + predictable-branches + 3-way ILP + 128K table is at the practical optimum for this workload on x86-64. The remaining gap to Java native is parallel efficiency, not per-row compute.
