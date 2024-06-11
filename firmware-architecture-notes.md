# SSD Firmware Architecture — Notes

> Clear, structured notes on the internal architecture of SSD firmware: how the
> controller turns unreliable, asymmetric NAND flash into a fast, reliable block
> device. Organised as the three-layer model **FE → FTL → BE**, plus the
> supporting subsystems (DRAM, multi-core, boot, power-loss, QoS).

---

## Table of Contents

1. [Why SSD Firmware Exists](#1-why-ssd-firmware-exists)
2. [Controller Hardware at a Glance](#2-controller-hardware-at-a-glance)
3. [The Three-Layer Firmware Model](#3-the-three-layer-firmware-model)
4. [Front-End (FE) Layer](#4-front-end-fe-layer)
5. [Flash Translation Layer (FTL)](#5-flash-translation-layer-ftl)
6. [Back-End (BE) Layer](#6-back-end-be-layer)
7. [DRAM Management](#7-dram-management)
8. [Multi-Core Firmware](#8-multi-core-firmware)
9. [Boot Sequence](#9-boot-sequence)
10. [Firmware Update](#10-firmware-update)
11. [Power-Loss Recovery (SPOR)](#11-power-loss-recovery-spor)
12. [QoS & Command Scheduling](#12-qos--command-scheduling)
13. [End-to-End Write Path](#13-end-to-end-write-path)

---

## 1. Why SSD Firmware Exists

An SSD is an **embedded computing system**, not a bag of NAND chips. NAND flash
has three properties that make it unusable as a raw block device:

| NAND property | Consequence |
|---------------|-------------|
| **No in-place update** | A page cannot be overwritten; the whole block must be erased first. |
| **Asymmetric granularity** | Read/write act on *pages* (~16 KB); erase acts on *blocks* (many MB). |
| **Limited endurance** | Each block survives a finite number of Program/Erase (P/E) cycles. |

Firmware **hides** all of this behind a simple contract: `LBA 0 … N-1`, arbitrary
read/write. It does so with a **log-structured** design — new data is written to
free pages sequentially, the old copy is marked stale, and a mapping table tracks
the current location of each LBA.

**The three pillars of SSD firmware:**

1. **Address Translation (FTL)** — maintain the LBA → physical mapping.
2. **Garbage Collection (GC)** — reclaim space from blocks full of stale pages.
3. **Wear Leveling (WL)** — wear all blocks at roughly the same rate.

> Scale note: production SSD firmware is 500K–2M+ lines of C/C++ running on
> multi-core ARM Cortex-R / RISC-V with hard real-time constraints. Nearly every
> SSD performance anomaly (latency spikes, write cliffs, GC stalls) traces back
> to firmware behaviour.

---

## 2. Controller Hardware at a Glance

```
+------------------------------------------------------------------+
|                     SSD Controller SoC                           |
|                                                                  |
|   Host Interface            CPU Subsystem                        |
|   (PCIe/NVMe PHY)           +------+ +------+                     |
|   Gen4 x4 lanes  <------->  |Core0 | |Core1 |   L1/L2, TCM       |
|   DMA engine                +------+ +------+                     |
|        |                    |Core2 | |Core3 |                     |
|        v                    +------+ +------+                     |
|   Command Parser /                                               |
|   NVMe Accelerator          SRAM (512KB-2MB)  <- FW code/data    |
|        |                          |                              |
|        v                          |                              |
|   DRAM Controller  <--------------+                              |
|   (DDR4/LPDDR4)                                                  |
|   256MB - 4GB                                                    |
|        |                                                         |
|   ECC Engine  <----->  Flash Controller                         |
|   (LDPC HW)            (NAND IF: CH0..CH7)                       |
+------------------------------------------------------------------+
                          |  |  |  |  ...
                       NAND Flash Channels -> NAND packages
```

| Block | Function |
|-------|----------|
| PCIe/NVMe PHY | Physical link to host (Gen3/4/5, x4 lanes typical). |
| NVMe Accelerator | HW-assisted SQ/CQ management and command parsing. |
| CPU Subsystem | 2–8 ARM Cortex-R / RISC-V cores; runs the firmware. |
| SRAM | On-chip fast memory for FW code, stack, critical structures. |
| DRAM Controller | External DRAM for L2P table, write buffer, read cache. |
| ECC Engine | Hardware LDPC/BCH encoder + decoder. |
| Flash Controller | Per-channel NAND control, DMA, timing generation. |
| Crypto Engine | HW AES-256 XTS for Self-Encrypting Drive (SED). |

**Rule of thumb:** ~1 GB DRAM per 1 TB of NAND (for a full page-level L2P table).

---

## 3. The Three-Layer Firmware Model

```
        HOST (NVMe Driver)
              |  PCIe / NVMe
   +---------------------------+
   |  FRONT-END (FE)           |  NVMe cmd processing, SQ/CQ,
   |                           |  PRP/SGL DMA, interrupts, namespaces
   +---------------------------+
              |  ftl_read / ftl_write / ftl_trim / ftl_flush
   +---------------------------+
   |  FLASH TRANSLATION (FTL)  |  L2P mapping, GC, WL, bad-block mgmt,
   |                           |  TRIM, power-loss recovery, over-provisioning
   +---------------------------+
              |  be_nand_read / program / erase / status
   +---------------------------+
   |  BACK-END (BE)            |  NAND channel control, ECC, read retry,
   |                           |  multi-plane/die scheduling, suspend/resume
   +---------------------------+
              |  NAND Flash Interface
        NAND Flash Dies
```

**Interface contracts (simplified):**

```
FE -> FTL:                          FTL -> BE:
  ftl_read(lba, len, buf)             be_nand_read(ch, die, blk, pg, buf)
  ftl_write(lba, len, buf)            be_nand_program(ch, die, blk, pg, buf)
  ftl_trim(lba, len)                  be_nand_erase(ch, die, blk)
  ftl_flush()                         be_nand_status(ch, die) -> ready/busy
```

Each layer has one job: FE speaks the host protocol, FTL owns policy, BE speaks
NAND. Clean interfaces let each layer be tested and mapped to CPU cores
independently.

---

## 4. Front-End (FE) Layer

The FE is the host-facing layer. It owns the NVMe protocol state machine.

### 4.1 NVMe Command Pipeline

```
1. Host writes SQE to Submission Queue, rings SQ Tail Doorbell
2. FE detects doorbell (interrupt or polling)
3. FE DMA-fetches the 64-byte SQE from host memory
4. FE parses: opcode, NSID, Start LBA, NLB, PRP1/PRP2 or SGL
5. FE validates NSID / LBA range / permissions
6. (Writes) FE DMA-fetches host data into DRAM write buffer via PRP/SGL
7. FE dispatches an internal request to the FTL
8. ...FTL/BE complete the operation...
9. FE assembles a 16-byte CQE (status, CID, SQ head, Phase Tag)
10. FE DMA-writes the CQE to the host Completion Queue
11. FE raises an MSI-X interrupt
12. Host consumes the CQE and rings CQ Head Doorbell
```

### 4.2 SQ / CQ Management

- SSD tracks head/tail pointers per SQ and CQ.
- **Doorbell writes** by the host signal new work (SQ) or consumed completions (CQ).
- **Phase Tag (P) bit** in each CQE lets the host detect new entries without reading pointers.
- Up to 64K queue pairs per the NVMe spec; per-CQ MSI-X vectors enable core affinity.

### 4.3 PRP vs SGL (data pointers)

| Mechanism | Description | Use |
|-----------|-------------|-----|
| **PRP** (Physical Region Page) | PRP1 + PRP2; for large transfers PRP2 points to a PRP List the FE must walk. Page-aligned. | Standard NVMe I/O. |
| **SGL** (Scatter-Gather List) | Descriptors (Data Block / Segment / Last Segment); arbitrary byte-aligned segments. | NVMe-oF, advanced hosts. |

```
PRP List walk:
SQE:  [PRP1: 0x1000] [PRP2: 0x5000 -> PRP List]
                              |
PRP List @ 0x5000:  [0x3000][0x7000][0x9000][0xB000]...
```

### 4.4 Interrupts

| Scheme | Vectors | Notes |
|--------|---------|-------|
| Pin-based | 1 | Legacy; sets a pending bit. |
| MSI | up to 32 | Controller writes to a host address. |
| **MSI-X** | up to 2048 | Per-CQ vector → target specific CPU cores (NUMA locality). |

**Optimisations:** batch SQE fetch, command reordering by LBA, pipelining
(fetch next while executing current), interrupt coalescing.

---

## 5. Flash Translation Layer (FTL)

The FTL is the heart of the firmware — it provides a rewritable block device over
write-once, erase-in-blocks NAND.

### 5.1 L2P Mapping Table

Each Logical Page Number (LPN) maps to a Physical Page Address (PPA):

```
LPN     | PPA (channel, die, plane, block, page)
--------|---------------------------------------
0       | (0, 0, 0, 100, 5)
1       | (2, 1, 0, 200, 12)
2       | UNMAPPED (0xFFFFFFFF)   <- trimmed
3       | (1, 0, 1, 50, 3)
```

**Example PPA packed into 4 bytes:**

```
[31:28] Channel (16)   [27:26] Die (4)   [25] Plane (2)
[24:11] Block (16K)    [10:0]  Page (2K)
```

**Sizing (page-level mapping, 4 KB LBAs, 4-byte entries):**

| Capacity | Entries | L2P DRAM |
|----------|---------|----------|
| 256 GB | 64 M | 256 MB |
| 1 TB | 256 M | **1 GB** |
| 2 TB | 512 M | 2 GB |

→ This is *why SSDs need DRAM*: ~1 GB DRAM per 1 TB NAND.

**DRAM-less (HMB):** budget SSDs have no DRAM. Options:
- **HMB (Host Memory Buffer):** borrow 32–64 MB of host DRAM for warm L2P entries.
- **On-SRAM caching:** small hot L2P cache in controller SRAM.
- **NAND-backed L2P:** full table in NAND, paged in on miss (adds latency).

### 5.2 Garbage Collection (GC)

When free blocks run low, GC reclaims space:

```
Victim block:  [V0][I1][V2][I3][I4][V5][I6][V7]   V=valid  I=invalid(stale)

1. Read valid pages (0,2,5,7)
2. Write them to a fresh destination block
3. Update L2P entries for the moved pages
4. Erase the victim block -> returns to the free pool

Destination:   [V0][V2][V5][V7][free][free][free][free]
```

**Victim selection strategies:**

| Strategy | Idea | Trade-off |
|----------|------|-----------|
| Greedy | Pick block with most invalid pages | Minimal copy cost; ignores wear. |
| Cost-Benefit | Weigh block age × invalidity ratio | Better WL; more complex. |
| d-Choices | Sample d random blocks, pick best | Good overhead/quality balance. |

**Write Amplification Factor (WAF)** = NAND writes / host writes.
- Ideal = 1.0. Real-world 1.5–5.0+.
- Sequential writes → ~1.0. Random writes → 3.0+.
- Low over-provisioning (OP) + random workload → WAF can spiral to 10×.

### 5.3 Wear Leveling (WL)

- **Dynamic WL:** natural side effect of GC — erased victims re-enter the free
  pool, new writes go to least-worn free blocks.
- **Static WL:** proactively moves *cold* data off low-P/E blocks so those fresh
  blocks can absorb hot writes.

```
Without WL (uneven -> premature failure)   With WL (even -> max lifetime)
5000 |     X X                              3000 | X X X X X X X X
3000 | X X X X X X                          2000 | X X X X X X X X
1000 | X X X X X X X X                            +--block index-->
```

### 5.4 Bad Block Management (BBM)

- **Factory bad blocks:** marked at manufacture, recorded in the Bad Block Table (BBT).
- **Grown bad blocks:** fail erase/program at runtime, or hit uncorrectable ECC on read.
- **Handling:** relocate valid data, map out the block, update the BBT (in reserved NAND).

### 5.5 TRIM / UNMAP

TRIM (NVMe Dataset Management, opcode `0x09`, AD bit) tells the SSD which LBAs the
host no longer needs:

1. FE parses the ranges (up to 256 ranges per command).
2. FTL sets the L2P entries to UNMAPPED and marks the physical pages INVALID.
3. GC can now skip copying those pages → lower WAF.
4. Read after TRIM: usually returns zeros without touching NAND (DZAT/DRAT determinism).

### 5.6 Over-Provisioning (OP)

Extra NAND beyond advertised capacity (e.g. 7% or 28%) gives GC room to breathe.
More OP → fewer valid-page copies per reclaimed block → lower WAF and higher
sustained performance.

---

## 6. Back-End (BE) Layer

The BE drives the physical NAND (ONFI / Toggle DDR interface).

### 6.1 NAND Scheduling & Parallelism

Four levels of parallelism to exploit:

| Level | Mechanism | Benefit |
|-------|-----------|---------|
| **Channel** | Independent buses | N channels → N× bandwidth. |
| **Die** | Die interleave on a channel (separate CE#) | Hide tPROG/tBERS busy time. |
| **Plane** | Multi-plane op in one die | ~2× throughput per die. |
| **Page** | Multi-page read (some NAND) | ~2× read bandwidth. |

```
Ch0: [Die0 PROG][Die1 READ][Die0 wait ][Die1 wait ]
Ch1: [Die0 READ][Die1 ERASE][Die0 wait][Die1 wait]
      |-- data xfer --|-- NAND busy (overlapped) --|
```

A **super-block** (same block index across all channels/dies/planes) is the usual
write-allocation unit for maximum striping.

### 6.2 ECC (LDPC) Pipeline

```
Write (encode):  Data 4KB -> [LDPC Encoder HW] -> Data + Parity -> NAND program
Read  (decode):  NAND -> Raw + Parity -> [LDPC Decoder HW] -> Corrected data
                                              |
                                    5-50 iterations; fail -> read retry
```

| Decode type | Input | Strength | Latency |
|-------------|-------|----------|---------|
| Hard-decision | Binary 0/1 | Moderate (~40–60 bits/1KB) | ~1 µs |
| Soft-decision | Multi-level Vref reliability (LLR) | Much stronger (100+ bits/1KB) | ~10–50 µs |

**ECC requirement by NAND type:** SLC → BCH; MLC → BCH/LDPC; TLC → LDPC
(mandatory); QLC → strong multi-iteration LDPC.

### 6.3 Read Retry

```
Normal read -> LDPC decode -> fail?
  -> Retry Level 1..N (shift Vref by +/- deltas), decode each
     -> still fail? -> Soft-Decision LDPC (multiple reads at shifted thresholds)
        -> still fail? -> UECC (uncorrectable; report to host / RAID recovery)
```

Causes of decode failure: charge loss (retention), P/E-cycle Vth shift, read
disturb, temperature. **Optimisations:** cache the winning retry offset per block;
proactively *refresh* (re-read + rewrite) high-retry blocks before BER explodes.

### 6.4 Program / Erase Suspend-Resume

A long tPROG (~1.5 ms TLC) can be suspended to service an urgent read:

```
Without: [======= tPROG (1.5ms) =======] READ waits -> ~2ms read latency
With:    [== tPROG ==][SUSP][tR 75us][RES][== tPROG cont ==] -> ~100us read
```

- tSUS / tRES overhead ~5–10 µs each.
- Critical for read tail latency (QoS). Excessive suspends can add 10–20% to total tPROG.
- Not all NAND supports it; suspend count per program is limited (often 1–2).

---

## 7. DRAM Management

DRAM is a shared, precious resource. Typical layout for 1 GB DRAM on a 1 TB SSD:

```
+----------------------------------+---------------+
| L2P Mapping Table                | ~512 MB (50%) |
| Write Buffer (WB)                | ~256 MB (25%) |
| Read Cache                       | ~128 MB (12.5%)|
| GC Buffer                        | ~64 MB (6.25%)|
| FW Heap / Metadata / Misc        | ~64 MB (6.25%)|
+----------------------------------+---------------+
```

- **Mapping-table caching:** for very large SSDs the full L2P won't fit; cache hot
  regions (LRU/ARC), page cold regions from NAND on demand.
- **Write buffer:** stage host writes, coalesce small writes to the same LBA,
  program a full NAND page at a time. Must be PLP-protected.
- **Read cache:** exploit temporal locality; LRU/LFU/adaptive eviction.

---

## 8. Multi-Core Firmware

Modern controllers have 2–8 cores; firmware is partitioned by role.

```
Core 0 (FE)      Core 1 (FTL)     Core 2/3 (BE)
- NVMe cmd       - L2P mgmt       - NAND channel control
- SQ/CQ          - GC engine      - ECC mgmt
- Host DMA       - WL engine      - Read retry
- Admin cmds     - TRIM / journal - NAND command issue
        \             |               /
              Shared DRAM / SRAM (L2P, buffers, queues)
```

**Inter-Processor Communication (IPC):**
- **Lock-free ring buffers** in shared SRAM (producer/consumer).
- **Hardware mailboxes / semaphores** for signalling and mutual exclusion.
- **Shared DRAM** for the L2P table and buffers.

**Critical-section discipline** — the L2P table is touched by host read (lookup),
host write (update), GC (lookup + update), and TRIM (invalidate). Options:
- Per-LBA-range fine-grained locks, or read-write locks.
- Partition the L2P by LBA range, one partition per core (avoids most locking).
- Watch for **cache coherency**: flush/invalidate D-cache when sharing via DRAM
  unless HW coherency (ACE/CHI) exists.

Real-time cores (Cortex-R) handle latency-critical FE/BE; application cores handle
FTL background work.

---

## 9. Boot Sequence

```
Power-On Reset
   -> 1. Boot ROM (immutable): init clocks/SRAM, load + verify bootloader from NAND
   -> 2. Bootloader: train DRAM, detect NAND geometry, load + verify main FW image
   -> 3. Main FW init:
          - scan bad blocks, build BBT
          - load L2P checkpoint from NAND (or replay journal if SPOR)
          - init FE (PCIe/NVMe), GC, WL, thermal
          - set CSTS.RDY = 1  (signal host: ready)
   -> 4. Ready for host commands; start background tasks (GC, WL, patrol read)
```

**Timing budget (clean boot):**

| Phase | Time |
|-------|------|
| Boot ROM | ~1–10 ms |
| Bootloader + DRAM train | ~50–200 ms |
| FW load + verify | ~20–500 ms |
| L2P rebuild (clean) | ~100–500 ms |
| L2P rebuild (after SPOR) | ~200 ms – 2 s |
| **Total** | **~0.3–3 s** |

Host waits by polling `CSTS.RDY` after `CC.EN=1`; timeout comes from `CAP.TO`
(500 ms units, typically 5–10 s). **Secure boot:** each stage verifies the next
stage's RSA/ECDSA signature.

---

## 10. Firmware Update

### Dual-Bank (A/B)

```
NAND FW area:  [ Bank A: Active v1.2 ] [ Bank B: Backup v1.1 ]

1. Host downloads new FW to Bank B (NVMe Firmware Image Download, 0x11)
2. FW verifies CRC / RSA signature
3. Host issues Firmware Commit (0x10) -> set boot pointer to Bank B
4. Reset -> boot from Bank B; Bank A becomes the backup
5. If Bank B fails to boot -> ROM falls back to Bank A (recovery)
```

**Commit Action (CA):** replace-only / activate-at-reset / replace-and-activate.
NVMe 1.3+ can activate without a reset if supported.

**Why A/B:** a corrupt or buggy new image never bricks the drive — the old image
survives. Budget in-place (single-bank) updates are riskier; a power loss mid-update
can brick the device unless a minimal recovery image is protected.

---

## 11. Power-Loss Recovery (SPOR)

Goal after a sudden power loss: preserve acknowledged writes, keep the L2P
consistent, and never let a partial program corrupt valid data.

**Strategy: Journal + Checkpoint**

```
Normal operation:
  every write   -> append journal entry {LPN, old_PPA, new_PPA, seqno, CRC}
  every N/T     -> checkpoint dirty L2P pages to a reserved NAND area

Recovery on next boot:
  1. Load last L2P checkpoint from NAND
  2. Replay journal entries after the checkpoint seqno
  3. Scan recently-open blocks for partially-written pages (ECC / seqno detect)
  4. Discard corrupted pages; rebuild a consistent L2P
  5. Resume
```

**Power-Loss Protection (PLP) hardware:** enterprise SSDs carry tantalum
supercaps holding ~10–50 ms of energy. On power dip, firmware flushes write
buffer → NAND, dirty L2P → NAND, journal → NAND. Client SSDs often skip PLP and
accept loss of in-flight (unacknowledged) writes.

**Golden rule:** update the L2P entry *only after* NAND program success —
otherwise a crash leaves a mapping pointing at an unprogrammed page. Use per-page
sequence numbers to detect partial writes during recovery. Beware MLC/TLC
paired-page effects: programming an upper page can corrupt the paired lower page.

---

## 12. QoS & Command Scheduling

The SSD serves host I/O *and* background work (GC, WL, patrol reads) on the same
NAND. Poor scheduling → multi-millisecond latency spikes.

```
Without QoS:  Host Read [====== blocked by GC writes ======][read]  ~10+ ms
With QoS:     Host Read [read][GC][read][GC][read][GC]...           ~100-200 us
```

**Priority (high → low):** host read ≈ host write(FUA) > host write(VWC) >
read-retry > GC read > GC write > background WL / SMART.

**Techniques:**
- **Rate-adaptive GC:** aggressive when idle, minimal when host is busy.
- **GC budgeting / token bucket:** cap background NAND ops per time window.
- **Program suspend/resume:** cut read tail latency during writes.
- **Watermarks:** background GC when free blocks < high watermark; urgent GC when
  < low watermark; *never* let free blocks reach zero (that forces blocking GC).
- **Multi-stream writes:** separate hot/cold data → less future GC.
- **NVMe I/O Determinism (1.4):** Deterministic / Non-Deterministic windows.

**Tail-latency targets (enterprise):** p50 ~100 µs · p99 < 500 µs · p99.9 < 1 ms ·
p99.99 < 5 ms. Good p99.99 demands meticulous scheduling + NAND suspend/resume.

**VWC & FUA:** with Volatile Write Cache the write is acknowledged from DRAM
(~10 µs) and programmed to NAND later — fast, but needs PLP. `FUA=1` forces the
data to NAND before completion (used for DB commits, FS journals). NVMe **Flush**
persists all cached writes.

---

## 13. End-to-End Write Path

Tracing a 4 KB host write to `LBA 1000`:

```
Host  ->  FE                    ->  FTL                    ->  BE            ->  NAND
 1. SQE to SQ; ring SQ doorbell
             2. detect doorbell
             3. DMA-fetch SQE
             4. parse (Write, LBA=1000, NLB=1, PRP1)
             5. DMA-fetch 4KB host data -> DRAM write buffer
             6. dispatch to FTL
                                   7. allocate free page (e.g. ch2,die0,blk500,pg42)
                                   8. update L2P: LBA1000 -> PPA; invalidate old PPA
                                   9. write journal entry (SPOR)
                                  10. hand PPA + data to BE
                                                             11. LDPC encode
                                                             12. issue PROGRAM (80h..10h)
                                                             13. wait tPROG (~1.5ms TLC)
                                                             14. check program status
                                  15. <- completion
             16. <- completion
17. FE builds CQE (success), DMA to CQ, raise MSI-X
18. Host reads CQE, rings CQ head doorbell
```

With VWC (`FUA=0`) the command can complete at step 6 (data safely in the
PLP-protected DRAM buffer) and the NAND program happens in the background.
