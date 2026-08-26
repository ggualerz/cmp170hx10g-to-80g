# Static GA100 HBM State: Findings and Remaining Gaps

## Result

Static analysis closes the geometry question but does **not** yet identify a complete static-state fix for coherent 80 GB operation.

The strongest result is a boundary, not a magic register:

- **CONFIRMED:** coherent 80 GB geometry is `FBPA_CFG1=0x02779000`, `LMR=0x28B`, with the 4-GiB/channel L2 decode observed as `0x10000300`. The old `80` branch actually paired CFG1 with `LMR=0x28A`; that state explains its exact 40-GiB fold.
- **CONFIRMED:** with `CONFIG0.USE_TIMING_REGS=0`, the live controller timings are the read-only `TIMING*_GEN` shadows, not the writable `TIMING0..20` bank.
- **CONFIRMED:** MODS constructs and programs the `CONFIG0..9`, MRS, and writable timing inputs from memory-clock/VBIOS state. It also exposes VBIOS-driven address training and per-subpartition/per-byte read/write barrel-shift state.
- **STRONG INFERENCE:** the GA100 hardware derives or latches `TIMING*_GEN` from those inputs. No direct software write to the generated bank was found.
- **OPEN:** the exact generated-timing transform, its relatch event, the complete generated-bank field map, and the per-byte training/VREF result addresses.
- **CONFLICT:** the Pry paper reports that lowering an unspecified refresh-interval field removed errors; the preserved local experiment wrote the visible `CONFIG4.tREFI` field, verified it on every active FBPA, and did not remove Xid instability. The experiments lack enough common provenance to call either one a reproduction of the other.

No hardware operation was performed for this report.

## Scope and artifact identity

This report is based on immutable local artifacts and targeted public documentation. The core identities are:

| Artifact | Identity |
| --- | --- |
| Repository HEAD | `2c0852dc6053116373d0aa381271fe89e25dcbd8` |
| Consensus Protocol source | commit `d923a25fb2ecb6ee3d62654983f453194b66485f` |
| Amogh `cmpunlocker` source | commit `8f3c82f438e474857edfa464d01fc8d5eaf96c5c` |
| Pry paper PDF | SHA-256 `16cb3551d5c9620ed698fb3f94a74704f398f5b40f1f37d27d6e4c0e61ef392c` |
| Zenodo metadata JSON | SHA-256 `1705c744ca156784b61520176717b74ed4f1ddf9dd7f5acb39782bc99b7effbb` |
| Chinese article capture | SHA-256 `d5941f0b558d31784c19eef2698d462de413cc602cff070e23976ccdf8ac4bc2` |
| MODS 455 ISO | SHA-256 `7422705d2a802c7c15f94856e70c3aea8a74679be3bfee59e1c09fd91a69df45` |
| MODS 455.229 ISO | SHA-256 `c02f7932766af4e545be5b7466b80d581e51591db31d0421c4ad1dfff5485f1b` |

The machine-readable evidence is observation-normalized in `data/02_static-ga100-hbm-state_registers.csv`: a repeated register with a different device, state, value, or source gets a new row. This deliberately preserves conflicts and avoids a misleading one-register/one-value table. `data/02_static-ga100-hbm-state_registers.schema.json` defines the required columns and controlled evidence vocabulary.

## Geometry is known

The static corpus supports four distinct states:

| State | CFG1 | LMR | Outcome |
| --- | --- | --- | --- |
| CMP10 stock | `0x02449000` | `0x288` | 10 GB product state |
| Supported unlock | `0x02669000` | `0x28A` | stable 40 GB |
| Old compiled `80` branch | `0x02779000` | `0x28A` | incoherent; exact 40-GiB fold |
| Later coherent experiment | `0x02779000` | `0x28B` | dense tagged readback to 77.5 GiB; Xid 154/context-lifetime failure remains |

The later result is stronger than a reported-size check: unique tags survived past the 40-GiB boundary. It does not establish a deployable 80-GB mode. The extended region ran at about 79% of peak, the top roughly 2 GiB was not covered, and the GPU failed after one CUDA context per fire.

`CSTATUS_RAMAMOUNT` is derived status, not proof of usable memory. A future claim still requires an alias-resistant write/read test.

## Active timing state

The observed stock configuration is:

```text
CONFIG0  0x009a0290 = 0x1255b93c  USE_TIMING_REGS=0
CONFIG1  0x009a0294 = 0x38d4841b  CL=27 WL=8 RD_RCD=18 WR_RCD=13
CONFIG2  0x009a0298 = 0x88130b11  tWR=19 W2R_BUS=8 R2W_BUS=8 CDLR=11
CONFIG3  0x009a029c = 0x24002b4a  tFAW=21 tCCD_L=4 tCCD_S=2
CONFIG4  0x009a02a0 = 0xc4030033  tREFI field=51
CONFIG7  0x009a02ac = 0x00c35000  ZQCS interval=12,800,000
TIMING12 0x009a0250 = 0x0bb800a1  tCKE=10 LOCKPLL=3000
```

The generated/live subset is:

```text
TIMING0_GEN  0x009a02b0  tRC=60 tRFC=441 tRAS=42
TIMING1_GEN  0x009a02b4  R2W=29 W2R=20 W2P=28
TIMING2_GEN  0x009a02b8  RD_RCD=18 WR_RCD=13 tRRD=6
TIMING4_GEN  0x009a02c0  tFAW=21
TIMING9_GEN  0x009a02d8  tCCD_L=4 tCCD_S=2
TIMING16_GEN 0x009a02e0  tRP=18
```

The important mismatch is `TIMING1_GEN`: the writable `TIMING1` copy contains turnaround values 18/13/18, while the live generated copy contains 29/20/28. Any analysis based only on `TIMING0..20` is therefore wrong for the observed mode.

### Producer trace

In `mods.unpacked` 455.229:

- the function beginning at virtual address `0x1ea5be0` constructs MRS and `CONFIG0..9` state; its caller is near `0x1eb5d22`;
- it consumes a memory-clock value (including a branch at 500,000 kHz) and state populated from VBIOS/memory tables;
- the programming path near `0x1ead794` compares and writes `CONFIG0`, then continues through `CONFIG1..9` using the common register-write helper near `0x221fc10`;
- diagnostics at string offsets `0x38d5090` onward name the `CONFIG0..9` values, and `0x38d6320` names execution of a VBIOS shadow-register script;
- the same bounded constructor region contains no direct `FBPA_CFG1` or LMR address reference.

This establishes a software producer for the inputs, not the generated outputs. Descriptor/read paths enumerate `0x2b0..0x2f0`, but the inspected code does not show software writing that bank. The best current model is:

```text
VBIOS memory-clock/profile data
        -> devinit/MODS constructs CONFIG + MRS + writable TIMING state
        -> GA100 hardware derives/latches TIMING*_GEN
        -> controller consumes TIMING*_GEN when USE_TIMING_REGS=0
```

The absence of a direct CFG1/LMR reference in one constructor is a bounded negative result. An input object could still carry a density- or rank-derived property, and another initialization path could participate. A matched snapshot is required before claiming geometry independence.

## Training and PHY state

`FBPA_TRAINING_STATUS=0` means both reported subpartitions finished. It read zero across the 15-card cohort, including an unlocked 64-GB card. This closes “training never completed” as a blanket explanation, but it does not prove that every newly addressed row has an appropriate electrical margin.

MODS provides better structural evidence than that status word:

- it can load VBIOS or internal address-training patterns;
- diagnostics explicitly name `NV_PFB_FBPA_FBIOTRNG_SUBPxBYTEx_OB_BRLSHFT1` for write/outbound delay;
- diagnostics name `...IB_BRLSHFT1` for read/inbound delay;
- both normal and skinny-partition variants are present;
- the code reports DQ/DQS termination and VREF-training configuration.

Therefore, calibrated state is at least per subpartition and byte, not summarized by `TRAINING_STATUS`. The exact GA100 addresses, iteration bounds, trained values, VREF result storage, and persistence across refire/FLR remain open. Hardware reads should wait until the chip-specific register descriptors and access types are resolved offline.

## Refresh reconciliation

The visible active refresh input is `CONFIG4.tREFI` at bits `[14:0]`. Stock `0xc4030033` is reported unchanged across CMP 8/10-GB, A100, and 40/80-GB geometry states. A clean-room experiment used `0xc403001a`, verified it on all 20 live FBPAs, flattened the bandwidth curve, reduced throughput to 54–61% of peak, and did not clear the observed instability.

The paper reports a different result: 2,796 errors at 80 GB, zero at half capacity, then zero at 80 GB after lowering an unnamed refresh-interval field, at a throughput cost from 94.6 to 64.6 TFLOPS. It does not publish the exact address, raw value, full register state, error addresses, temperature, or same-run logs.

These observations are a **CONFLICT in outcome**, not a demonstrated contradiction. Plausible differentiators include card/bin, temperature, workload, initialization path, timing profile, register/value, and whether the measured failure was retention, decode, or context teardown. None can be selected from current evidence.

Additional refresh-related candidates are not yet promoted because no active GA100 field mapping was found for per-bank refresh, temperature-compensated refresh, refresh credits, or self-refresh timing. `CONFIG7.ZQCS_INTERVAL` is active maintenance state but not evidence of a refresh-capacity fix.

## Vendor, density, and capacity confounders

The tested CMP10 identifies through Samsung's IEEE 1500 `DEVICE_ID` route as model part `0x41`, `XA2_8HI`, 8-high, 8-Gbit/channel, HBM2. `FBPA_VEND_ID_C0/C1` are both zero on all 15 cards and are closed as a vendor-identification route.

`FBPA_MRS_8` is `0x00200000` throughout the cohort, including CMP10, A100 40 GB, and A100 80 GB. It is not the product-capacity restriction. The CMP10 pair `MRS_2=0x002000cf`, `MRS_WL_RL=0x003000ea` matches A100 40 GB; A100 80 GB uses `0x00200029`/`0x003000ef`. That difference can encode profile, vendor, rank, clock, or board policy; it cannot be attributed to capacity alone.

Samsung publicly lists a 16-GB Flashbolt HBM2E stack (`KHAA84901B-MC16`) with a 1024-bit interface, 3.2-Gbit/s data rate, and 32-ms refresh. This is useful as a documentary upper-density control, not proof that a CMP package contains that stack ([Samsung product page](https://semiconductor.samsung.com/us/dram/hbm/hbm2e-flashbolt/khaa84901b-mc16/)). A TechInsights floorplan article identifies `K4C6E1K6MB` as a Samsung 16-Gbit HBM2E die, but this remains teardown-specific evidence and supplies no public MRS/training register contract ([TechInsights analysis](https://www.techinsights.com/blog/samsung-k4c6e1k6mb-3rd-gen-16gb-hbm2e-dram-memory-floorplan-analysis)).

The minimum useful capacity comparison is therefore same vendor, same clock, same board family, and independently confirmed stack identity. An SK hynix A100 80-GB timing profile must not be treated as a Samsung 80-GB answer.

## Xeon Max feasibility gate

Intel documents Xeon CPU Max as 64 GB of HBM2e across four stacks and four memory controllers, with HBM-only, Flat, and Cache modes exposed through BIOS/OS topology ([Intel technical overview](https://www.intel.com/content/www/us/en/developer/articles/technical/xeon-scalable-processor-max-series.html)). The reviewed public material exposes topology, modes, RAS, and performance behavior. It does not name a public interface for raw HBM MRS execution, refresh interval programming, per-byte training results, or PHY delay/VREF state.

**Conclusion:** do not rent Xeon Max hardware for generic register exploration. Reopen this comparison only if an Intel public or legitimately available privileged document identifies a specific HBM timing/training register and access mechanism that answers one of this report's named gaps. Architectural independence alone is not sufficient.

## Closed paths

The following should not consume more work without new evidence:

- rediscovering CFG1/LMR geometry encodings;
- treating reported CSTATUS capacity as proof of usable memory;
- `MRS_8` as the capacity limiter;
- `FBPA_VEND_ID_C0/C1` as the vendor route;
- writable `TIMING0..20` tuning while `USE_TIMING_REGS=0`;
- Samsung MODS selector 4 as hidden capacity (it is soft lane repair);
- Samsung MODS selector 7 as writable capacity (it is read-only `DEVICE_ID`);
- FRB/FUB/FPF/HULK paths as hidden HBM density selection;
- Xeon Max rental without a named low-level interface.

## Open questions, ranked

1. **What event derives or relatches `TIMING*_GEN`?** Resolve the full descriptor/producer path and compare boot stages.
2. **Does generated timing differ with geometry on the same card?** Capture matched per-FBPA snapshots at stock 10 GB, stable 40 GB, and coherent 80 GB without changing clock or vendor.
3. **Where are GA100 trained delay and VREF results?** Finish the MODS chip-dispatch trace for `FBIOTRNG ... BRLSHFT1` before touching hardware.
4. **What exactly did the paper change?** Obtain its refresh address/value, workload, error distribution, temperature, and raw logs.
5. **What changes at Xid 154?** Correlate read-only FBPA status, kernel logs, elapsed time, address range, and temperature.
6. **Can a same-vendor 40/80-GB A100 pair separate capacity from vendor/profile?** Require IEEE 1500 identity and matched clock.

## Minimal future collection

`data/02_static-ga100-hbm-state_future-read-only-collection.csv` is the executable handoff. Its first useful experiment is deliberately narrow: on one CMP10 at one clock, capture each live FBPA's `0x220..0x2f0` timing/config window plus MRS/status before and after the same initialization milestones in stock, stable-40, and coherent-80 states. Preserve broadcast and unicast values, timestamps, temperatures, CFG1/LMR/CSTATUS controls, and raw words.

Do not mix writes into that collection. Do not probe unresolved `FBIOTRNG` addresses until offline descriptor recovery establishes exact addresses, instance bounds, access types, and absence of read side effects.

## Decision

Static work has made the next hardware question small enough to be worthwhile, but it has not justified an 80-GB fix. The highest-value next artifact is a matched, per-instance, boot-stage timing snapshot—not another capacity fire and not another refresh sweep.
