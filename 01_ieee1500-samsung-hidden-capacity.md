# Samsung GA100 Host-to-JTAG / IEEE 1500 Research Report

> [!CAUTION]
>
> ## Final conclusion: dead end for hidden-capacity activation
>
> This investigation is complete. In the examined NVIDIA MODS `455.127` and `455.229` artifacts, the Samsung GA100 model-profile, Host-to-JTAG/IEEE 1500 descriptor, and FRB/FUB paths do not expose a mechanism for changing HBM geometry, internal addressability, density, row-address width, or shadow state.
>
> The active `SamsungHbm2eGA100` catalog contains only identification, reset/TAP support, diagnostics, fuse-scan support, and lane/row-repair operations. Its methods, descriptors, and applicability construction are unchanged across the `0x43` transition from Hi4 to Hi8. The associated FRB and FUB consumers resolve to repair and field-programmable-fuse/security workflows, not framebuffer-capacity initialization.
>
> This is a bounded negative result, not proof that no hidden-capacity mechanism exists elsewhere. It closes this MODS IEEE 1500 path as a productive lead for the 40 GB to 80 GB transition. Future work should pivot to a different subsystem or to new evidence of the reported shadow-register operation.

Detailed method and transaction evidence is preserved in the [Samsung GA100 cartography](01_ieee1500-samsung-hidden-capacity_map.md).

> [!IMPORTANT]
>
> This document is the final report for the closed `research/ieee1500-samsung-hidden-capacity` branch. It records the completed NVIDIA MODS reverse engineering and the limits of its negative result; it does not claim that the broader 80 GB problem is solved.

## Research objective

The objective was to recover and classify every Samsung GA100 Host-to-JTAG / IEEE 1500 transaction in NVIDIA MODS `455.229`, then compare the complete Samsung HBM2E implementation against MODS `455.127` to identify operations introduced or changed with the `0x43` Hi8 profile.

The investigation was deliberately broader than a search for one presumed capacity-control opcode. It established the observable Samsung GA100 command surface, its callers, its model restrictions, and its behavior before evaluating operations as candidates for modifying addressability or shadow state.

## Central research question

```text
SamsungHbm2eGA100
    -> Host-to-JTAG
    -> which WIR/WDR operations exist?
    -> does any operation modify addressability or shadow state?
```

The resulting output is an evidence-backed transaction catalog, not an unlock recipe. Operations were traced from their descriptors or dynamically constructed values through the Host-to-JTAG call and, where possible, to their originating `.jse`, firmware, diagnostic, repair, or configuration workflows.

## Investigation scope

The investigation covered:

1. mapping every method in `SamsungHbm2eGA100` and its inherited Samsung HBM2E classes;
2. identifying every Host-to-JTAG call reachable from those methods;
3. extracting constant and dynamically constructed WIR and WDR values, including opcode, payload length, field layout, channel/site selection, and update/readback behavior;
4. classifying transactions as read-only, repair, test, configuration, or unknown;
5. recovering cross-references originating from `repair.jse`, `hbm.jse`, `mods.jse`, `frb_ga100.bin`, and `fub_ga100.bin` wherever the artifact format permitted it;
6. searching code, strings, tables, descriptor names, and field layouts for `shadow`, `override`, `row`, `address`, `density`, `repair`, `spare`, `redundancy`, `fuse`, and `test mode`; and
7. performing a targeted binary diff between MODS `455.127` and `455.229` around Samsung HBM2E construction, vtables, methods, descriptors, call sites, and model-applicability data.

The investigation must distinguish three separate problems:

1. selecting an 80 GB address space on the GA100 memory controller;
2. exposing additional physical rows through the Samsung HBM logic die rather than aliasing the existing 40 GB geometry; and
3. establishing refresh and timing parameters that make the expanded capacity stable.

## Current research result

Reverse engineering of NVIDIA MODS confirms the following:

- The stock CMP 170HX Samsung profile is selected by the composite key `0x4108`, formed from raw model part `0x41` and stack height 8.
- This key maps to Samsung HBM2, X-die, revision A2, 8-high, named `XA2_8HI`.
- MODS `455.229` maps raw model part `0x43` to a Samsung HBM2E, 8-high profile named `?_16GB_8HI`.
- The reconstructed consumers of these profiles are limited to device identification and specification-dependent density decoding, Host-to-JTAG/IEEE 1500 interface selection, and memory-repair functionality.
- No direct path was found from either profile to GA100 DevInit, refresh configuration, timing configuration, normal framebuffer initialization, or activation of the additional address capacity required for 80 GB.

This is a negative but substantive result: the `0x43` profile is operational data inside MODS, not merely a display string, but the recovered path does not show that selecting it can unlock capacity or configure stable 80 GB operation.

### Locally verified artifact set

The first continuation pass on this branch verified that both MODS images are present under `artifacts/mods` and that their hashes match the previously recorded acquisition data:

| Artifact | Size | SHA-256 |
| --- | ---: | --- |
| `Mods-455.iso` | 431,593,472 bytes | `7422705d2a802c7c15f94856e70c3aea8a74679be3bfee59e1c09fd91a69df45` |
| `Mods-455.229.iso` | 411,717,632 bytes | `c02f7932766af4e545be5b7466b80d581e51591db31d0421c4ad1dfff5485f1b` |

The `455.229` image was extracted into a temporary analysis directory and its packed `mods` executable was successfully decompressed with the locally stored UPX 5.2.0 tool. The resulting 170,746,192-byte ELF again hashes as `567192a497415178a2221e9dcf7cc2324e857ab7d7ea3670fd73d95ec6c85bb0`.

Additional `455.229` component identities established during this pass are:

| Embedded component | Size | SHA-256 |
| --- | ---: | --- |
| `hbm.jse` | 13,624 bytes | `501b3daae44bba662d143072f4c2e5c7298c0e51b70b40782eae3f8512f72e10` |
| `mods.jse` | 12,346 bytes | `e4858edfced6c22c813d888085622f96637536011d9c9231e76170bf8c781c89` |
| `repair.jse` | 5,677 bytes | `19dc10e8ca56d9cf92f7727182e3a8f3f55aef08e3e01471dfba67657dba44b8` |
| `frb_ga100.bin` | 4,792 bytes | `275c8f10ae84b82e31ad1bf206173269b26d7343631c998c004b81e28e74176b` |
| `fub_ga100.bin` | 5,332 bytes | `72a0c26c9e3598218375d1129477859697bb70078c41fd16cf3caf2a85fc748a` |

All five components differ from their `455.127` counterparts. The `.jse` files remain high-entropy precompiled artifacts and do not expose useful HBM identifiers through ordinary plaintext string extraction.

## Confirmed findings to preserve in the final report

### Samsung profile lookup

The Samsung lookup does not always use the raw model part as its key. MODS treats model part `0x41` as ambiguous and combines it with the numeric stack height. A stock 8-high device therefore resolves as:

```text
model part  = 0x41
stack height = 8
effective key = 0x4108
profile = HBM2 / Samsung / X / Hi8 / A2 / XA2_8HI
```

Other model parts, including `0x43`, use their raw value as the lookup key in the recovered implementation:

```text
effective key = 0x43
profile = HBM2E / Samsung / Unknown / Hi8 / Unknown / ?_16GB_8HI
```

The exact meaning of `16GB` in the internal name remains unproven. It must not be presented as direct evidence that a CMP stack physically contains or exposes 16 GB.

### Version-dependent `0x43` mapping

The Samsung `0x43` entry changed across MODS builds:

| MODS build | Samsung profile name | Stack height |
| --- | --- | ---: |
| `455.127` | `?_16GB_4HI` | 4-high |
| `455.229` | `?_16GB_8HI` | 8-high |

The `455.229` binary independently confirms the 8-high mapping reproduced in the external research article. This also proves that the uppercase Samsung entry is distinct from the mixed-case SK hynix-style `?_16GB_8Hi` string found elsewhere in the binary.

### `SamsungHbm2eGA100` vtable diff

The first complete class-level cartography pass recovered eleven virtual slots in both MODS versions. All corresponding functions have identical sizes and retain the same slot ordering. Normalized instruction comparison found no semantic method change associated with the `0x43` Hi4-to-Hi8 transition:

- nine methods are instruction-equivalent after relocation/address normalization;
- one method differs only in the relocated addresses of identical floating-point constants;
- one method retains the same control flow but follows framework object-layout offsets that changed elsewhere in the build.

No `SamsungHbm2eGA100` method was added, removed, reordered, or given a new IEEE 1500 workflow in `455.229`. The remaining diff target is below the vtable boundary: descriptor data, model-applicability sets, helper methods, native bindings, and FRB/FUB firmware.

### Selector 4 and 7 resolution

The highest-priority descriptor ambiguity is now resolved. The selector supplied to the lookup helper is a catalog key, not the literal WIR opcode:

| Selector | Descriptor | WIR | Access | WDR | Classification |
| ---: | --- | ---: | --- | ---: | --- |
| `4` | `SOFT_LANE_REPAIR` | `0x12` | Read/write | 72 bits | Repair |
| `7` | `DEVICE_ID` | `0x0e` | Read-only | 82 bits | Read-only |

Both MODS builds construct these descriptors with the same fields and the same applicability-object construction. The `0x43` Hi4-to-Hi8 change did not alter either descriptor.

The `SamsungHbm2eGA100 + 0x78` object is an `AmpereHbmInterface`. The virtual calls used by selector 7 resolve to `WriteInstr`, `WriteMode`, `Wait`, and `ReadData`. The immediate `0x0f` is a channel mask that is combined with the descriptor's opcode; it is not WIR `0x0f`.

This closes the most direct candidate path: selector 4 is strictly lane repair, and selector 7 reads device identity. Neither recovered transaction modifies addressability, density, row-address width, or shadow state.

### Complete Samsung descriptor decision

The Samsung catalog family has now been enumerated across its common, B-die, X-die, and HBM2E-GA100 vectors. The active `SamsungHbm2eGA100` catalog contains nine descriptors. Its only write-capable entries not named as lane or row repair are:

- `BYPASS`, used as IEEE 1500/TAP support by repair sequences;
- `HBM_RESET`, with no direct mapped GA100 lookup caller;
- `MODE_REGISTER_DUMP_SET`, used by test/diagnostic register-dump logic; and
- `ENABLE_FUSE_SCAN`, used to read or validate fuse-repair state around row repair.

The Samsung X-die-only `TEST_MODE_REGISTER_SET` descriptor is write-only but is not loaded by `SamsungHbm2eGA100`. No catalog entry or mapped caller refers to shadow state, density programming, address width, row-address extension, geometry, BIST, spare enablement, or redundancy configuration.

The active three-entry HBM2E-GA100 extension and its static applicability seed data are identical between `455.127` and `455.229`. The catalog-level answer is therefore negative: no write-capable non-repair operation with plausible geometry or internal-addressing semantics was recovered.

### FRB/FUB pivot result

The catalog-level negative result triggered the planned FRB/FUB pivot.

- `frb_ga100.bin` is reached through `RunRirFrb` and diagnostics explicitly identifying an RIR/fuse-repair binary. Native callers place it in the HBM row-repair persistence/check path, including warnings about repairs to already repaired fuses. This is repair infrastructure, not normal geometry initialization.
- `fub_ga100.bin` is an encrypted field-programmable-fuse binary. Native diagnostics identify FUB use cases such as GSYNC FPF programming, HULK license enable/revocation, and permission to read DRAMID. No recovered FUB use case refers to HBM capacity, density, address width, or memory geometry.
- Both artifacts are high-entropy binary containers. `frb_ga100.bin` remains 4,792 bytes in both builds and is 81.36% byte-identical at equal offsets; its changes are concentrated in block-aligned regions. `fub_ga100.bin` grows from 5,304 to 5,332 bytes, adds/repackages a 28-byte header boundary, and retains substantial payload correspondence at a `+28` alignment.

The FRB/FUB differences cannot be connected to the `0x43` Hi4-to-Hi8 table change. Their recovered native consumers instead identify row-repair/fuse and general FPF-security functions. This pivot currently provides no hidden-capacity candidate.

The complete map is maintained in [`01_ieee1500-samsung-hidden-capacity_map.md`](01_ieee1500-samsung-hidden-capacity_map.md).

### Reconstructed consumer path

The demonstrated call graph is:

```text
DEVICE_ID fields
    -> Samsung effective-key construction
    -> Samsung model-map lookup
    -> HbmModel {specification, vendor, die, height, revision, name}
    -> HBM2/HBM2E-specific DEVICE_ID density decoding
    -> model validation and Host-to-JTAG interface factory
    -> Samsung B-die, X-die, X-die-GA100, HBM2E, or HBM2E-GA100 class
    -> model-filtered IEEE 1500 command catalog
    -> memory-repair and diagnostic operations
```

The profile therefore affects real behavior in MODS. In particular, it selects how the raw density field is interpreted and which diagnostic/repair implementation and IEEE 1500 descriptors apply.

### Density decoding

MODS selects different density tables according to the recovered model's HBM specification:

- HBM2 uses a six-entry density table.
- HBM2E uses a twelve-entry density table.

This is a read-and-interpret path. It decodes the value reported by `DEVICE_ID`; it does not program density, widen row addressing, or expose additional memory.

### IEEE 1500 scope

The recovered model-specific command catalogs include identification, reset, mode-register diagnostics, lane repair, row repair, fuse scan, test-mode, and pseudo-hard-repair-related operations. The catalogs demonstrate that NVIDIA varies IEEE 1500 opcodes and payload geometries by HBM model.

No recovered descriptor has yet been connected to normal boot-time density selection, refresh programming, or activation of an additional row-address bit. A promising command name is not evidence of a hidden-capacity control.

### Negative call-graph result

Direct cross-references from the Samsung model lookup and active-site model accessors terminate in:

- `HBMDevice` construction and serialization;
- reporting and model comparison;
- density interpretation;
- Host-to-JTAG interface construction; and
- fuse, lane, and row-repair workflows.

No direct consumer was recovered in:

- GA100 DevInit;
- framebuffer initialization;
- refresh or `tREFI`/`tRFC` programming;
- normal timing generation;
- memory training;
- mode-register setup for normal operation; or
- geometry/address-decoder activation.

The final report must state this boundary clearly. Absence of a recovered path is not proof that no such mechanism exists elsewhere; it means the examined MODS profile path does not currently provide it.

### Rank-count validator resolved

The previously unresolved framebuffer/HBM consistency validator at virtual address `0xe0b290` has now been traced far enough to determine its data source.

The function obtains the framebuffer rank count through a framebuffer-object virtual method, separately requests HBM device information through a GPU-interface virtual method, and compares the returned rank count against the stack-height byte in that device-information result. The diagnostic at `0xe0b410` receives those two independently obtained values:

```text
FrameBuffer rankCount (%d) does not match HBM StackHeight (%d).
```

No call to the Samsung model-map lookup at `0xd8a8f0`, its `HBMDevice` wrapper at `0xd8aa10`, or the active-site model accessor at `0xd81420` occurs in this validator. This confirms that the validator consumes an independently decoded device representation, not the `HbmModel` returned by the Samsung `0x4108`/`0x43` map.

Consequently, this validator is not an additional downstream consumer of the Samsung profile and does not extend the recovered call graph toward geometry initialization.

### Separate `AutoRefreshValue` path

The code associated with the diagnostics `Setting the AutoRefresh register is not supported on this chip!`, `Setting AutoRefreshValue = 0x%x, reg = 0x%x`, and `Invalid AutoRefreshValue provide: 0x%x` was inspected in the same executable.

The recovered routine validates a user/configuration value, obtains a generic register-access interface, reads and updates framebuffer register fields, and reports the programmed value. Within the inspected routine there is no call to the Samsung model lookup, model accessor, density decoder, Host-to-JTAG factory, or IEEE 1500 catalog.

This establishes a second independent path: MODS can expose a generic framebuffer auto-refresh override without deriving it from the Samsung `0x41` or `0x43` profile. The exact GA100 field semantics and relationship to the refresh increase reported in the external 80 GB stability experiment still require register-level identification.

## Interpretation

The `0x41` versus `0x43` comparison does not currently provide an 80 GB unlock recipe. It establishes that NVIDIA recognizes distinct Samsung HBM2 and HBM2E models and uses those distinctions for diagnostics, density decoding, and repair-related IEEE 1500 behavior.

The result weakens the simple hypothesis that changing the detected model from `0x41` to `0x43` automatically selects a GA100 initialization profile for larger capacity. The missing operation may instead reside in a different subsystem, firmware component, generated DevInit data, a Samsung-only manufacturing flow, or an as-yet-unidentified vendor command that is not reached through this model-map consumer chain.

## Open hypotheses

The following remain hypotheses and must not be promoted to findings without new evidence:

- The CMP's Samsung DRAM dies contain native 16-Gbit capacity but report or expose an 8-Gbit geometry.
- A Samsung logic-die setting enables an additional row-address bit, commonly hypothesized as `RA[14]`.
- An undocumented IEEE 1500 instruction or test-mode field controls that setting.
- The `0x43` HBM2E profile is applicable to the physical Samsung stacks installed on the CMP 170HX.
- The `0x41` and `0x43` profiles select different normal-operation refresh or timing parameters.
- The increased refresh rate reported to stabilize an expanded configuration is the correct long-term operating solution rather than a workaround for an incomplete timing profile.

## Completed investigation plan

### Executed priority after selector resolution

The work proceeded in this order:

1. enumerate every Samsung descriptor catalog and classify every entry by name, selector, WIR, WDR length, access mode, applicability, callers, and effect;
2. prioritize write-capable entries whose names or callers suggest test, mode, fuse, row, spare, redundancy, BIST, configuration, shadow, address, or density behavior;
3. separate operations already explained by lane/row repair from non-repair writes;
4. complete semantic diffs of every Samsung descriptor and model-applicability set between `455.127` and `455.229`; and
5. decide whether any surviving non-repair operation plausibly affects memory geometry or internal addressing.

The catalog-level decision produced no candidate, so the investigation pivoted to `frb_ga100.bin` and `fub_ga100.bin`. That pivot also produced a negative result. Reconstructing every bit of the 72-bit `SOFT_LANE_REPAIR` payload may remain useful for repair cartography, but it is outside this closed hidden-capacity investigation.

The decision question is:

```text
Does the Samsung IEEE 1500 catalog contain any write-capable
non-repair operation that could plausibly alter memory geometry
or internal addressing?
```

### 1. Recover the complete Samsung HBM2E GA100 class surface

- Locate the `SamsungHbm2eGA100` RTTI object, vtable, constructors, destructors, overrides, and inherited method targets in both MODS builds.
- Assign a stable research identifier to every vtable slot and recovered method, even when no source symbol is available.
- Record method boundaries, direct callers, indirect-call sites, referenced strings, constructed objects, and adjacent diagnostics.
- Distinguish class-selection logic from operations performed after the class has been instantiated.

Expected deliverable: a two-version method and vtable map showing identical, moved, changed, added, and removed functions.

### 2. Enumerate every Host-to-JTAG transaction

- Identify all low-level Host-to-JTAG entry points and wrapper layers.
- Trace every call reachable from `SamsungHbm2eGA100` and inherited Samsung implementations.
- Recover WIR values, WDR input/output lengths, dynamically generated payloads, capture/shift/update sequencing, channel masks, site masks, and polling behavior.
- Record whether the transaction reads state, writes volatile state, invokes repair, enters a test mode, touches a fuse-related path, resets logic, or has unknown effects.
- Preserve constructed expressions and bit-field dependencies when a WIR or WDR value is not a literal constant.

Expected deliverable: one row per transaction with the following minimum fields:

```text
research ID
MODS version
class and method
caller/workflow
WIR opcode or construction
WDR length and field layout
direction and IEEE 1500 sequence
model applicability
observed or inferred effect
classification
evidence addresses
confidence
```

### 3. Classify the recovered operations

Use exactly these primary categories:

| Classification | Meaning |
| --- | --- |
| Read-only | Identification, status, capture, or readback with no observed state-changing update |
| Repair | Soft/hard lane or row repair, PPR, spare-resource handling, redundancy, or repair-state inspection |
| Test | Test-mode entry, test-register access, fuse scan, manufacturing diagnostics, or operations intended for controlled test environments |
| Configuration | A state-changing operation plausibly used for normal geometry, addressing, density, timing, refresh, mode, or shadow configuration |
| Unknown | Insufficient evidence to assign one of the above categories safely |

An operation must not be classified as configuration solely because it writes data. Repair and test operations are state-changing but belong in their own categories. Unknown operations must remain unknown until their payload and caller establish a stronger interpretation.

### 4. Trace cross-artifact consumers and producers

- Determine how `hbm.jse`, `repair.jse`, and `mods.jse` invoke native MODS APIs despite their precompiled format.
- Compare exported JavaScript bindings, property names, argument schemas, registration tables, and native handler xrefs between versions.
- Analyze `frb_ga100.bin` and `fub_ga100.bin` structure, loading path, command interface, and differences between `455.127` and `455.229`.
- Search for shared opcodes, WDR lengths, masks, field constants, error strings, and command identifiers across the native executable and all five companion artifacts.
- Record explicitly when the precompiled or firmware format prevents a reliable cross-reference rather than treating a failed plaintext search as absence.

### 5. Perform the targeted `455.127` versus `455.229` diff

The diff must cover more than the descriptive model-map entry:

- `SamsungHbm2e` and `SamsungHbm2eGA100` RTTI and vtables;
- constructor and factory branches;
- every mapped method body;
- IEEE 1500 descriptor construction and model-applicability sets;
- Host-to-JTAG wrapper functions and call sites;
- repair, test-mode, fuse, spare, and redundancy handlers;
- `.jse` native-binding tables; and
- GA100 FRB/FUB loader paths and embedded binary differences.

For each difference, determine whether it is semantic or only relocation, layout, compiler, or unrelated build drift. A change associated with the `0x43` transition from Hi4 to Hi8 is relevant only if the data flow reaches a changed consumer or command path.

Expected deliverable: a focused change report linking each semantic difference to both binary versions and stating whether it changes the available Samsung HBM2E transaction set.

### 6. Search specifically for addressability and shadow-state candidates

Search strings, descriptors, field names, constants, masks, and nearby code for:

```text
shadow
override
row
row address
address mask
address mode
density
repair
spare
redundancy
fuse
test mode
```

For every candidate, answer:

1. Is it reachable from `SamsungHbm2eGA100`?
2. Does it issue a Host-to-JTAG transaction?
3. What exact WIR/WDR data does it use?
4. Is the operation model-specific?
5. Is it present in both `455.127` and `455.229`?
6. Does it modify addressing or a shadowed state, or is it limited to identification, diagnostics, repair, or test?

### 7. Preserve secondary provenance work

- Acquire and identify the exact MODS `455.263.1` runtime referenced by the original experiment.
- Hash and inventory its `mods` executable, `.jse` files, libraries, firmware, and data files.
- Apply the same method/vtable/transaction mapping to that build once recovered.
- Recover the provenance and revision of the published source fragment.

### Closed or deprioritized paths

- The framebuffer rank-count validator is resolved as an independent HBM device-information check and is no longer a primary research target.
- The generic `AutoRefreshValue` path is independent of the Samsung model map and is no longer a primary research target.
- Refresh and stability remain relevant to a complete 80 GB result, but they should not distract from the present Host-to-JTAG transaction inventory.
- Physical-capacity and alias testing remain separate experimental work and are outside the current static-analysis phase.

## Original report structure

The following structure was used to define the intended evidence boundaries. Some hardware-dependent sections remain outside this closed static-analysis branch:

1. Executive summary and final conclusion
2. Hardware and software provenance
3. Evidence and confidence-classification method
4. Stock CMP 170HX Samsung `0x4108` profile
5. Evolution of the Samsung `0x43` profile across MODS versions
6. Reconstructed device-identification and density-decoding path
7. Host-to-JTAG/IEEE 1500 architecture and model-specific catalogs
8. Memory-repair consumers and safety implications
9. Exhaustive negative results for DevInit, geometry, timing, and refresh
10. Capacity-aliasing evidence and physical-memory validation
11. Remaining or discovered hidden-capacity mechanism
12. Refresh, timing, performance, and stability results
13. Limitations, unanswered questions, and reproducibility notes
14. Appendices containing hashes, offsets, call graphs, descriptor tables, and minimal disassembly excerpts

## Evidence-record format

Every material finding added to the report should record:

```text
claim:
classification: confirmed | strong inference | hypothesis
artifact and version:
SHA-256:
file offset / virtual address / symbol:
caller and callee:
minimal evidence:
interpretation:
alternative explanations:
reproduction status:
```

Negative results must identify the inspected artifact, search method, and boundary of the search. They must be phrased as "no path was found in the examined artifacts," not as universal proof of absence.

## Safety boundary

The present research phase is static and read-only. Do not:

- brute-force or issue unknown IEEE 1500 instructions;
- execute `HBM_RESET` or test-mode commands;
- perform hard repair, fuse, eFuse, OTP, SPI, PMBus, I2C, or voltage writes;
- write GPU MMIO registers or modify firmware; or
- assume that a diagnostic command is reversible because it is exposed by MODS.

Any future hardware experiment requires an explicit command-level understanding, a risk review, a recovery plan, and independent approval before execution.
