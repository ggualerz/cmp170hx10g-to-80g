# cmp170hx10g-to-80g

> **80 GB physically addressable ≠ 80 GB usable as a CUDA device.**
> The HBM itself can hold data correctly across the full 80 GB while the
> RM/GSP management path breaks above ~40 GiB of real use. This distinction
> is the central finding of this research — it is not a contradiction.

> [!WARNING]
>
> ## The CMP 170HX 10 GB → 80 GB unlock has NOT been solved yet
>
> Published research reports that a CMP 170HX can expose 80 GB of distinct HBM, but the complete configuration required to reproduce this reliably and operate it stably is still unknown.

# Objective

The NVIDIA CMP 170HX 10 GB is based on GA100 and uses Samsung HBM.

The objective of this project is to understand how the card can apparently expose **80 GB of real HBM**, determine what configuration is missing from the currently known methods, and eventually reproduce a stable 80 GB configuration.

The project deliberately distinguishes between:

* **confirmed facts**
* **experimental observations**
* **strong inferences**
* **unverified hypotheses**

A larger framebuffer reported by software is not considered proof of additional physical memory.

# Research Status

Four research phases are preserved in this repository:

| Phase | Result | Document |
| --- | --- | --- |
| `00` Samsung `0x41` vs `0x43` | Confirmed the MODS profile distinction and its bounded consumers | [`00_0x41-vs-0x43-profile.md`](00_0x41-vs-0x43-profile.md) |
| `01` Samsung IEEE 1500 | Closed as a dead end for hidden-capacity activation in the examined MODS builds | [`01_ieee1500-samsung-hidden-capacity.md`](01_ieee1500-samsung-hidden-capacity.md) |
| `02` static GA100 HBM state | Closed without a complete stability fix; narrowed remaining gaps to generated-timing relatching and trained PHY state | [`02_static-ga100-hbm-state.md`](02_static-ga100-hbm-state.md) |
| `04` H2D/GSP access path | 79 GiB SM kernel fill+verify confirmed clean (WPR2-fix, no SEC2 patch). H2D writes above 40 GB **never passed** — the GSP-RM 40 GB allocation limit was never broken. Limit appears tied to the card's hardware identity (OTP fuse `FUSE_PCIE_DEVIDA=0x2082` is the primary suspect, not confirmed) → SKU table → HAL dispatch layout mismatch; 40 GB stable is the ceiling; 80 GB blocked by firmware-internal class selection (A/B on 10 GB card showed identical SKU/class between 40 GB and 80 GB, but the 8 GB→64 GB card was not tested) | [`04_h2d-gsp-access-path.md`](04_h2d-gsp-access-path.md), [`open questions`](04_h2d-gsp-access-path_to-confirm.md) |

## Phase 04 summary

**Directly observed:** a deterministic GSP-RM failure occurs whenever
the 80 GB configuration exceeds ~40 GiB of real use: H2D copies into
high mappings fault (Xid 119 → 1 → 119×2 → 154), and the crash is traced
to a **wrong-layout HAL dispatch** at firmware offset `0x5b2b724` — the
GSP-RM dispatches a method on an object whose `+0x1a8` slot is a method
pointer, but the method reads it as a data pointer and faults. By
contrast, SM-kernel fill+verify of 79 GiB completes clean (31/31 chunks,
WPR2-fix run), which is why "the memory works" and "the device breaks"
are both true at once.

**Also observed:** an A/B static-info dump (40 GB control vs 80 GB
crash) showed identical SKU/identity/class fields between both
configurations — 638 of 642 words match (on the 10 GB card only; the
8 GB card was not A/B-tested). So no host-visible field selects the bad
dispatch.

**Our explanatory model (inference):** the GSP-RM derives the faulty
internal class from the 80 GB geometry itself, and the 40 GB cap is
sized from the SKU table entry for this card's hardware identity. The
relationship between the `0x2082` identity (OTP fuse
`FUSE_PCIE_DEVIDA` is the primary suspect) and the faulty internal
class remains **inferred — not a traced code path**. Three host-side
device-ID overrides were all ignored by the GSP-RM, which is why the
fuse is suspected, but the GSP-RM code that reads it was never traced.
See [`04_h2d-gsp-access-path_to-confirm.md`](04_h2d-gsp-access-path_to-confirm.md).

The WPR2/PMA overlap bug in cmpunlocker was found and fixed (commit `e093717`): the `late-pma.patch` was registering ~141 MB of GSP-protected heap as free PMA memory. The fix clamps the registered region below WPR2.

**The FREE→NOP patch is a diagnostic workaround, NOT an unlock.** A
driver-side patch (`inject_free_skip.py` + heap override) enabled
`cudaMalloc` + `cudaFree` at 76 GiB with zero Xids, but **it does not
lift the 40 GB limit, and no H2D write above 40 GB ever passed**. It
bypasses the teardown crash only; it leaks one GSP tracking object per
skipped FREE, and it was **source-only, never installed**. Its real
value is evidential: it proves the teardown crash lives in the RPC path
(FREE / BUS_FLUSH), not in the memory hardware.

**Current assessment:** 80 GB is not achievable with any mechanism tested. 40 GB stable is the working ceiling. This reflects exhausted approaches, not a proof of impossibility.

# Known CMP 170HX Configurations

## CMP 170HX 8 GB

Community work reports a stable 64 GB configuration on the 8 GB variant:

```text
PCI ID: 20C2
HBM: SK hynix

CFG1 = 0x02779000
LMR  = 0x20B

16 × 4 GiB = 64 GiB
```

## CMP 170HX 10 GB

The 10 GB variant is:

```text
PCI ID: 2082
GPU: GA100
HBM: Samsung
```

A known 40 GB configuration uses:

```text
CFG1 = 0x02669000
LMR  = 0x28A

20 × 2 GiB = 40 GiB
```

An experimental 80 GB geometry uses:

```text
CFG1 = 0x02779000
LMR  = 0x28B

20 × 4 GiB = 80 GiB
```

These values alone are **not a complete 80 GB unlock**.

# The 8 Gb / 80 GB Paradox

MODS identifies the CMP 170HX 10 GB HBM as:

```text
Samsung
XA2_8HI
8 Gb/channel
8-high
```

while the published research reports that 80 GB of distinct memory can be accessed after additional configuration. Several explanations remain possible — the physical DRAM may genuinely be 8 Gbit, or it may contain more capacity than the normal HBM configuration exposes, or binning/repair/redundancy may be involved. None of these explanations is currently proven.

# Samsung HBM Profiles

The MODS-related material distinguishes two Samsung HBM profiles:

| Profile | HBM gen | Density | Stack |
| --- | --- | --- | --- |
| `0x41` / `XA2_8HI` | HBM2 | 8 Gb/channel | 8-high |
| `0x43` / `?_16GB_8HI` | HBM2E | 16 Gb/channel | 8-high |

The `0x43` profile is interesting because NVIDIA software distinguishes 8-Gb and 16-Gb Samsung HBM. However, `0x43` should not be treated as a Samsung commercial Device ID confirmed by a public datasheet — the evidence comes from NVIDIA/MODS-related material only. See [`00_0x41-vs-0x43-profile.md`](00_0x41-vs-0x43-profile.md).

The IEEE 1500 / JTAG investigation of the `0x43` profile is a **dead end for hidden-capacity activation** within the examined MODS path. See [`01_ieee1500-samsung-hidden-capacity.md`](01_ieee1500-samsung-hidden-capacity.md).

# Main Sources

| Source | Relevance |
| --- | --- |
| [A Canary in the Crypto Mine](https://github.com/amoghmunikote/cmpunlocker) (Jon Pry et al.) | Reported 80 GB configuration, shadow-register operation, full-capacity hash testing, stability via refresh |
| [Chinese CMP170HX / MODS investigation](https://blog.kkk.rs/archives/71) | Samsung HBM identification (`0x41`, `XA2_8HI`, 8 Gb/channel, 8-high) |
| [Consensus-Protocol/cmp170hx](https://github.com/Consensus-Protocol/cmp170hx) | GA100 register documentation, CMP/A100 dumps, FBPA, MRS, timing, IEEE1500 |
| [amoghmunikote/cmpunlocker](https://github.com/amoghmunikote/cmpunlocker) | Existing CMP unlock research, low-level GPU register access primitives |
| [CMP 170HX 80-GB GSP/DMA report](https://gist.github.com/cuddylac997/c3d80faa2430e3650cd934eda5fd65a9) | Paired kernel-fill/H2D-fill test programs, clean kernel write/read-back through 75–76 GiB, H2D/GSP failure at 38.5–44 GiB, recurring GSP fault signatures |
| [Samsung Flashbolt HBM2E](https://semiconductor.samsung.com/dram/hbm/hbm2e-flashbolt/) | Official Samsung 16 GB HBM2E products |
| [TechInsights K4C6E1K6MB](https://www.techinsights.com/blog/samsung-k4c6e1k6mb-3rd-gen-16gb-hbm2e-dram-memory-floorplan-analysis) | Physically identified Samsung 16-Gbit HBM2E die (Intel Xeon CPU Max 9462) |

# What Would Count as Success?

A real solution is not `nvidia-smi reports 80 GB`. A convincing result should demonstrate:

```text
80 GB reported
80 GB physically distinct
no address aliasing
repeatability after cold boot
stable memory stress
stable compute stress
acceptable performance
acceptable thermals
```

The configuration should also be reproducible from a known initial state.

# Current Conclusion

**The CMP 170HX 10 GB → stable 80 GB problem remains open.**

What we know:

```text
CMP 170HX 10 GB uses Samsung HBM (0x41 / XA2_8HI / 8 Gb/channel / 8-high)

Coherent 80 GB geometry is known (CFG1=0x02779000, LMR=0x28B)

DIRECTLY OBSERVED:

79 GiB of distinct HBM is accessible to SM/kernel paths (kernel fill +
verify clean, WPR2-fix run). H2D writes above the 40 GB limit NEVER
passed — every H2D attempt above it faulted.

A deterministic GSP-RM failure occurs when the 80-GB configuration
exceeds ~40 GiB of real use; the crash is traced to a wrong-layout
HAL dispatch (firmware offset 0x5b2b724).

An A/B static-info dump on this 10 GB card (40 GB control vs 80 GB
crash) showed identical SKU/identity/class fields — 638 of 642 words
match. The A/B was only run on the 10 GB card; the 8 GB card
(0x20C2 → 64 GB stable) was not tested, so a host-visible difference
for other SKU paths cannot be fully excluded.

INFERRED (explanatory model, not a traced code path):

The GSP-RM derives the faulty internal class from the 80 GB geometry
itself. The 40 GB cap is sized from the SKU table entry for this
card's hardware identity (class=9, size_index=10) — the 0x2082
identity is the primary suspect (OTP fuse FUSE_PCIE_DEVIDA), but the
GSP-RM code that reads it was never traced.

What we don't know:

```text
the exact physical Samsung DRAM die

why an 8-Gb/channel identified stack can apparently expose 80 GB

the exact shadow-register configuration used for full capacity

the correct stable Samsung 80 GB memory profile

whether the 40 GB limit can be bypassed without firmware patching

the complete reproducible unlock procedure
```
