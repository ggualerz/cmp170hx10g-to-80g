# Static GA100 HBM State Research Plan

> [!IMPORTANT]
>
> ## Final conclusion: static research closed; no complete 80 GB stability fix found
>
> This static phase is complete. It confirms the coherent 80 GB geometry (`FBPA_CFG1=0x02779000`, `LMR=0x28B`) and establishes that GA100 consumes the read-only `TIMING*_GEN` bank while `CONFIG0.USE_TIMING_REGS=0`. NVIDIA MODS constructs and programs the writable timing/configuration inputs, but the exact hardware transform and relatch event that produce `TIMING*_GEN` remain unresolved.
>
> MODS also proves that memory-training state extends beyond the single completion register into per-subpartition/per-byte read and write delay families. Their exact GA100 addresses and VREF result storage remain open. The published successful refresh result and the later unsuccessful `CONFIG4.tREFI` experiment cannot be reconciled without the original paper's register value and raw logs.
>
> No additional static source currently justifies a stable 80 GB configuration. The next useful work is the narrowly defined, read-only, matched-state collection in `data/02_static-ga100-hbm-state_future-read-only-collection.csv`, not another broad static search or speculative register write.

## Branch

```text
02_static-ga100-hbm-state
```

## Execution status (2026-08-26)

The static phase is complete. Its outputs are:

- `02_static-ga100-hbm-state.md` - evidence-backed findings, producer/dependency analysis, conflict ledger, stop decisions, and ranked remaining gaps;
- `data/02_static-ga100-hbm-state_registers.csv` - observation-normalized register corpus;
- `data/02_static-ga100-hbm-state_registers.schema.json` - row schema and controlled vocabulary;
- `data/02_static-ga100-hbm-state_future-read-only-collection.csv` - minimal future collection manifest with controls, stop conditions, and risk.

The work found no justified static 80-GB fix. It narrowed the next useful collection to matched, per-FBPA timing/config/MRS snapshots at identical clock and initialization milestones. It also added two material refinements to the plan's initial model:

1. MODS 455.229 contains a concrete software construction/programming path for the writable `CONFIG0..9`, MRS, and timing inputs, but no demonstrated software writer for `TIMING*_GEN`; the hardware derivation and relatch event remain open.
2. MODS names per-subpartition/per-byte inbound and outbound `FBIOTRNG ... BRLSHFT1` training-delay families. This proves the completion bit is not the training-result corpus, while leaving the exact GA100 addresses and VREF result storage unresolved.

## Purpose

This phase assumes that no physical CMP 170HX is available. Its purpose is to identify the active GA100/HBM state that is still missing from the public CMP/A100 register corpus, so that later hardware work is limited to a short, evidence-backed set of read-only observations and controlled tests.

The work must remain:

```text
static
read-only
reproducible
source-backed
```

This is also intended to establish a reusable register inventory and evidence format for later CMP/A100 register research, rather than producing another one-off dump comparison.

## Safety boundary

This branch permits static analysis of documents, source, binaries, firmware images, VBIOS images, existing dumps, and archived logs only.

It does not permit:

- GPU register writes;
- IEEE 1500 execution;
- firmware or VBIOS modification;
- SPI, eFuse, OTP, PMBus, or VRM operations;
- hardware rental;
- speculative procedures presented as safe hardware instructions.

Any future collection procedure produced here must be read-only and must identify every register, range, command, privilege requirement, and expected side effect before use.

## Evidence rules

Every material claim must carry one of these labels:

- **CONFIRMED** - demonstrated directly by a primary document, inspected code/binary, or preserved dump/log;
- **STRONG INFERENCE** - the evidence supports one explanation but does not directly demonstrate it;
- **HYPOTHESIS** - plausible and testable, but not yet demonstrated;
- **CONFLICT** - sources or observations disagree and the disagreement is not yet resolved.

For each fact, preserve:

```text
source
artifact identity/version
location (page, file/line, symbol, address, or log record)
hardware/SKU/vendor context
observation date when relevant
confidence label
```

Do not silently merge these evidence classes:

1. statements reported by the Pry paper;
2. later community hardware observations preserved in the Consensus-Protocol corpus;
3. values decoded from public register headers or source;
4. values recovered from MODS, VBIOS, driver, or firmware binaries;
5. interpretation added by this repository.

## Established baseline

### CMP 170HX 10 GB identification

The tested stock card is reported by NVIDIA MODS as:

```text
vendor          Samsung
model part      0x41
model           XA2_8HI
stack height    8-high
density/channel 8 Gbit
protocol        HBM2
```

This proves what the logic die reports through `DEVICE_ID`; it does not by itself prove the maximum physical density manufactured into the DRAM dies.

### Geometry states

The following controller states must be distinguished:

| State | `FBPA_CFG1` (`0x009a0204`) | LMR (`0x00100ce0`) | Meaning/status |
| --- | --- | --- | --- |
| CMP10 stock | `0x02449000` | `0x288` | 10 GiB product configuration |
| CMP10 supported unlock | `0x02669000` | `0x28A` | 40 GiB; stable/shipping community configuration |
| CMP10 coherent 80 GiB geometry | `0x02779000` | `0x28B` | 80 GiB address geometry; experimental and unstable |
| Old community `80` branch as compiled | `0x02779000` | `0x28A` | incoherent: CFG1 requests 4 GiB/FBPA while LMR remains 40 GiB |

The old `80` branch must not be cited as evidence that `0x28B` was tested: its active build path hard-coded `0x28A` even though inert metadata said `0x28B`. Later driverless experiments did program the coherent `0x28B` state and reported dense tagged readback past 40 GiB, but also repeatable Xid 154/context-lifetime instability. Geometry selection is therefore known; stable operation is not.

Do not spend this phase rediscovering the CFG1/LMR encoding except to resolve a field definition or provenance needed by the inventory.

### Published 80 GB result

The Pry paper reports:

- a framebuffer-geometry shadow-register write after opening the relevant PLMs;
- a distinct hash pattern written across 80 GB and read back intact;
- 2,796 errors in one 80 GB `gpu-burn` run while 40 GB ran cleanly;
- zero errors after lowering the refresh-interval field of one FB-config register;
- throughput falling from 94.6 to 64.6 TFLOPS (about 32%);
- no sweep of the other timing parameters;
- failure to make a runtime HBM mode-register-set command reach the die, with FBFLCN boot code the only observed agent driving it.

These are confirmed statements in the paper, but their underlying memory-test logs, exact register sequence, and exact refresh register/value are not published in the paper. Treat those missing details as open provenance, not as reconstructed facts.

### Later refresh evidence

The newer community corpus identifies `FBPA_CONFIG4` (`0x009a02a0`) bits `[14:0]` as the visible `tREFI` field, with stock value `0xc4030033`. A coherent-geometry experiment halved the interval with `0xc403001a`; the write reached all 20 active FBPAs, reduced bandwidth, and did not clear the observed instability.

This does not contradict the Pry result unless both experiments are shown to use the same card, initialization state, workload, timing source, and register/value. Record it as a source/experiment difference. Visible `tREFI` remains relevant but must not be treated as the primary unexplored hypothesis.

### Generated timings already present in the corpus

The current corpus reports:

```text
CONFIG0.USE_TIMING_REGS = 0
CONFIG0 address          = 0x009a0290
TIMING* writable bank    = 0x009a0220-0x009a028c
TIMING*_GEN live shadows = 0x009a02b0-0x009a02f0, plus 0x009a0288
```

It also contains partial field decoding and example values for `CONFIG0` through `CONFIG4`, `CONFIG7`, and several generated timing registers. The next task is to verify completeness, provenance, producers, and coverage across devices—not to claim that the generated bank has just been discovered.

### Closed Samsung MODS paths

Previous repository work established the following for `SamsungHbm2eGA100`:

```text
selector 4  SOFT_LANE_REPAIR  WIR 0x12  WDR 72 bits  repair
selector 7  DEVICE_ID         WIR 0x0e  WDR 82 bits  read-only
0x0f                           channel mask, not a WIR opcode
```

The complete active descriptor catalog contains no demonstrated non-repair write that plausibly changes density, geometry, row width, hidden capacity, or normal address decoding. `frb_ga100.bin` is bounded to row/fuse repair persistence and validation. `fub_ga100.bin` is bounded to FPF/security uses such as GSYNC, HULK/license, and DRAMID permission.

Do not reopen these paths without new evidence connecting a specific operation to normal framebuffer initialization or active HBM state.

## Central question

> Which active GA100/HBM registers, generated values, training results, or firmware-produced structures relevant to coherent and stable 80 GiB operation are absent from, or insufficiently described by, the existing public corpus?

Do not assume missing state exists, or that any missing state explains instability. First establish its existence, producer, consumer, scope, lifetime, and dependencies.

## Workstream 1 - Build a reusable register inventory

Audit all existing CMP/A100 register tables, dumps, scripts, logs, headers, and research notes. Produce a machine-readable inventory rather than another prose-only comparison.

Planned outputs:

```text
data/02_static-ga100-hbm-state_registers.csv
data/02_static-ga100-hbm-state_registers.schema.json
```

Minimum fields:

```text
canonical_name
address
address_space
functional_block
broadcast_or_unicast
instance_scope
field_definitions
access_type
privilege_or_PLM
reset/persistence domain
initialization stage
known writer
known reader/consumer
active/inert/unknown
dynamic/static/latched/unknown
device/SKU
HBM vendor
CMP10 stock value
CMP10 40 GiB value
CMP10 coherent-80 GiB value
CMP8 64 GiB value
A100 40 GB value
A100 80 GB value
source and exact locator
confidence
notes/conflicts
```

Classify each register or state as one or more of:

```text
geometry
address decode
timing
generated timing
MRS
refresh
training
PHY
status
repair
IEEE 1500
fuse
firmware interface
unknown
```

Requirements:

- preserve broadcast and per-FBPA aliases rather than deduplicating them incorrectly;
- distinguish a register's reset value, sampled value, programmed value, and derived value;
- record whether a value survives FLR, driver reload, GPU reset, or power cycle when known;
- distinguish public header field definitions from reverse-engineered names;
- represent multiple observations instead of forcing conflicting values into one cell;
- make the schema reusable for other CMP/A100 register families.

The first analysis product is a coverage matrix showing which devices and states have comparable evidence and which cells are genuinely missing.

## Workstream 2 - Complete and verify active timing state

Verify the full GA100 map for:

```text
TIMING0-TIMING20
TIMING*_GEN
CONFIG0-CONFIGn
refresh-related registers outside those banks
```

For every entry determine:

```text
address and aliases
field widths and units
read/write status
whether it is active when USE_TIMING_REGS = 0
known values for each device/state
whether present in existing public dumps
writer/producer
reader/consumer
reset and relatch behavior
source provenance
```

Do not infer that every address in `0x009a02xx` is a timing register. Verify names and fields against primary headers, code, or independent binary behavior. Explicitly resolve the unusual generated-bank placement at `0x009a0288` and every gap in `0x009a02b0-0x009a02f0`.

Separate these questions:

1. Which bank contains readable active values?
2. Which bank is writable?
3. When are writable values sampled or copied?
4. Does toggling `USE_TIMING_REGS` merely select a bank, or trigger generation/relatching?
5. Are some fields sourced from `CONFIG*`, training output, MRS, or another indexed space?

## Workstream 3 - Identify the producer of `TIMING*_GEN`

Search all available NVIDIA artifacts for symbols, strings, field names, and raw addresses associated with:

```text
TIMING_GEN
TIMING*_GEN
USE_TIMING_REGS
CONFIG0-CONFIGn
0x009a0290-0x009a02f0
per-FBPA unicast aliases
```

Search targets, in priority order:

1. public NVIDIA register headers and open kernel modules;
2. VBIOS tables and the existing VBIOS parsers;
3. DevInit and framebuffer initialization paths;
4. FBFLCN firmware and interfaces;
5. MODS native code and scripts;
6. driver and GSP binaries or descriptor tables;
7. preserved initialization logs and register dumps.

Determine whether generated timings are:

```text
hard-coded
selected from a table/strap profile
computed from memory clock
computed from protocol/vendor parameters
computed from JEDEC-like timing inputs
derived from training
produced by firmware
copied or transformed from another register bank
```

Stop once the value source and dependency inputs are understood well enough to predict or explain the observed register values. Do not reverse unrelated initialization code merely because it is nearby.

## Workstream 4 - Test geometry and density dependency statically

Trace whether the timing-generation path consumes any representation of:

```text
FBPA_CFG1
LMR
L2/LTC address-decode state
CSTATUS_RAMAMOUNT
row count or address depth
2 GiB versus 4 GiB per FBPA
8-Gbit versus 16-Gbit density
stack height
HBM vendor/model/protocol
memory strap
```

Decision rule:

- If geometry or density is a demonstrated input, promote stale geometry-dependent generated timings to a high-priority future hardware hypothesis.
- If generation depends only on clock, vendor/protocol, strap profile, or electrical training, strongly downgrade the claim that the 40-to-80 instability comes from stale geometry-dependent `TIMING*_GEN` state.
- If the producer cannot be recovered, record the exact unresolved boundary and the smallest observation that future hardware or a new artifact would need to resolve it.

## Workstream 5 - Locate training and PHY state

Search statically for:

```text
DQ / DQS
VREF / VREFDQ / VREFCA
read leveling / write leveling
deskew
delay / tap / trim / phase
eye / margin
training offset/result/status
calibration
```

Start from the known `FBPA_TRAINING_STATUS` (`0x009a0974`) but do not mistake one completion/status register for the training-result corpus.

Candidate storage includes:

```text
FBPA or PHY registers
indexed register spaces
per-channel/per-lane apertures
FBFLCN DMEM/SRAM structures
VBIOS training tables
generated register banks
GSP/RM handoff structures
```

For each candidate record:

```text
scope: global / FBPA / stack / channel / pseudo-channel / rank / lane
writer/producer
reader/consumer
initialization stage
lifetime and reset domain
whether publicly dumpable read-only
known values/provenance
```

Then trace whether initialization or result selection depends on density, row geometry, CFG1, LMR, stack capacity, or memory strap. If training is purely electrical and independent of address depth, record that negative result and downgrade it as an explanation for the 40-to-80 transition. Do not infer independence merely because no dependency is found in one artifact.

## Workstream 6 - Reassess refresh beyond visible `tREFI`

Treat visible `CONFIG4.tREFI` as already identified and experimentally weakened, not as a new lead. Search for distinct active state involving:

```text
tRFC / refresh-cycle duration
per-bank refresh
refresh mode or multiplier
temperature-compensated refresh
refresh credits/scheduling
generated refresh fields
self-refresh entry/exit
vendor- or density-specific refresh selection
```

For each field, establish whether it is active under `USE_TIMING_REGS = 0`, where it is sourced, and whether it differs by geometry, vendor, density, or clock. Only promote fields that are demonstrably active.

Reconcile the Pry paper's successful unspecified refresh-interval change with the later `CONFIG4 = 0xc403001a` result. Acceptable conclusions include different cards/bins, different workloads, different initialization states, a different register/value, or incomplete provenance; do not choose among them without evidence.

## Workstream 7 - Separate vendor, capacity, and strap effects

Do not treat an A100 80 GB SK hynix profile as a Samsung 80 GB profile.

Classify every comparison as:

```text
GA100 generic
capacity/geometry-dependent
HBM vendor-specific
memory-strap/profile-specific
clock-dependent
unknown/confounded
```

A100 80 GB is strong evidence for GA100 controller topology and geometry. Differences in MRS, WL/RL, `tRFC`, VREF, PHY calibration, and training are not 40-versus-80 evidence unless vendor and strap are controlled or the controlling code proves the relationship.

The known MRS replay observations must be inventoried, including their provenance and failure mode, but runtime MRS reachability is not the primary target of this static phase.

## Workstream 8 - Samsung 16-Gbit documentary references

Continue source research for:

```text
model part 0x43
?_16GB_8HI
KHAA84901B-*
K4C6E1K6MB
Samsung Flashbolt HBM2E
```

The purpose is to find public Samsung 16-Gbit information relevant to MRS, refresh, timings, training, or density-dependent behavior. It is not to assert that the CMP10 contains those exact parts.

Keep these relationships explicit:

- NVIDIA MODS `455.229` mapping `0x43` to `?_16GB_8HI`: **CONFIRMED** from the inspected binary;
- Flashbolt 16 GB stack product references: **CONFIRMED** from Samsung documentation;
- `K4C6E1K6MB` as a Samsung 16-Gbit HBM2E die in Xeon Max: source-specific documentary evidence;
- any relationship from those parts to CMP `0x41`/`XA2_8HI`: **HYPOTHESIS** unless new direct evidence appears.

## Workstream 9 - Xeon Max feasibility gate

Before considering access to a Xeon Max system, determine statically:

```text
HBM IMC PCI/MMIO ranges
publicly documented registers and access methods
refresh parameter visibility
mode-register visibility
PHY/training visibility
low-level HBM maintenance interfaces
read permissions and kernel requirements
whether the exposed state is comparable to GA100 at all
```

Produce a read-only collection proposal only if the available interface can answer a named open question. Do not recommend hardware rental for generic register exploration or for data that cannot be mapped back to GA100/Samsung state.

## Workstream 10 - Build the future read-only collection plan

Convert only unresolved, observable state into a later CMP/A100 collection manifest. Each proposed read must contain:

```text
question answered
exact address/range or interface
instance count and stride
required device state and timing
expected safe access type
comparison devices needed
output format
success/failure interpretation
known hazards or uncertainty
```

Prefer a minimal matched-state collection over a broad dump:

```text
same CMP10 card: stock -> 40 GiB -> coherent 80 GiB
matched clocks and temperature where possible
before and after driver/GSP initialization when justified
A100 40 and A100 80 only for fields that are not vendor-confounded
```

No collection command should be executed on this branch.

## Deliverables

### `02_static-ga100-hbm-state.md`

The final report must contain:

1. **Confirmed** - directly demonstrated facts, with exact provenance;
2. **Open** - unresolved questions and the boundary preventing a static answer;
3. **Closed / Low Priority** - paths already exhausted or hypotheses weakened by evidence;
4. **Hardware Test Candidates** - only questions that require physical CMP/A100 access, each tied to a minimal read-only observation or controlled future test.

It must also include:

- a source/conflict ledger;
- a register-coverage summary generated from the machine-readable inventory;
- the `TIMING*_GEN` producer/dependency conclusion;
- the training/PHY storage map or exact unresolved boundary;
- refresh conclusions that distinguish the two reported experiments;
- explicit vendor/capacity confounders;
- a ranked list of remaining hardware candidates with stop/go criteria.

### Machine-readable corpus

```text
data/02_static-ga100-hbm-state_registers.csv
data/02_static-ga100-hbm-state_registers.schema.json
```

Validation must check at least:

- unique canonical identity plus address-space/instance semantics;
- valid evidence classification;
- a source locator for every non-empty observed value;
- explicit device and HBM-vendor context;
- no silent overwrite of conflicting observations;
- consistency between CSV fields and the JSON schema.

## Suggested execution order

1. Freeze artifact identities and build the source/conflict ledger.
2. Import and normalize the existing register corpus.
3. Produce the coverage matrix and identify actual gaps.
4. Complete the `CONFIG*`, `TIMING*`, and `TIMING*_GEN` map.
5. Trace generated-timing producers and dependency inputs.
6. Map training/PHY state and its producer/consumer boundary.
7. Reassess active refresh state beyond visible `tREFI`.
8. Classify vendor, capacity, clock, and strap confounders.
9. Perform targeted Samsung 16-Gbit and Xeon Max documentary research only where it closes an identified gap.
10. Write `02_static-ga100-hbm-state.md` and derive the minimal future collection plan.

## Stop rules

- Do not reverse a subsystem merely because it is interesting.
- Stop a call graph when its output and dependencies are understood well enough to answer the branch question.
- Do not reopen IEEE 1500 or FRB/FUB without a new concrete edge to geometry or active initialization.
- Do not collect more A100 values until the inventory shows which comparison cell they fill.
- Do not interpret reported framebuffer size or CSTATUS alone as proof of usable, distinct memory.
- Do not promote a timing difference until vendor, clock, strap, and initialization state are accounted for.
- Preserve negative results: proving a hypothesis irrelevant is a successful outcome.

## Success condition

This phase succeeds when it reduces later hardware work to a short list such as:

```text
candidate active register/state A
candidate generated-timing dependency B
candidate per-lane training state C
candidate refresh field D
```

Each candidate must have a demonstrated reason to matter, a missing evidence cell, and a minimal observation that can resolve it. The phase also succeeds if static analysis closes all of these categories and shows that the remaining instability is more consistent with part-specific margin/binning than with undocumented geometry-dependent controller state.

The final question for every task is:

> Will this result change what we observe or test once a CMP 170HX is physically available?
