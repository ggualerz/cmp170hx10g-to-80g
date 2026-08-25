# SamsungHbm2eGA100 Method Cartography

> [!IMPORTANT]
>
> This is the final static-analysis map for the closed `research/ieee1500-samsung-hidden-capacity` branch. Method names that are not recoverable from symbols are assigned stable identifiers `M00` through `M10`. Descriptive names are provisional unless supported by diagnostics or demonstrated data flow.

## Artifacts

| MODS version | Unpacked executable SHA-256 |
| --- | --- |
| `455.127` | `4b7fcbc46e9135a42513410ffb6187ffe59342f9c062e26bd8f87fe6a61369e7` |
| `455.229` | `567192a497415178a2221e9dcf7cc2324e857ab7d7ea3670fd73d95ec6c85bb0` |

## Class identity

The Itanium ABI RTTI name is:

```text
N6Memory3Hbm17SamsungHbm2eGA100E
```

This demangles to:

```text
Memory::Hbm::SamsungHbm2eGA100
```

| Item | MODS `455.127` | MODS `455.229` |
| --- | ---: | ---: |
| RTTI name | `0x3c8b8e0` | `0x3e7df60` |
| RTTI object | `0x3c8b908` | `0x3e7df88` |
| Vtable address point | `0x3c8b930` | `0x3e7dfb0` |
| Base RTTI | `SamsungHbm2e` | `SamsungHbm2e` |
| Object size used by factory | 128 bytes | 128 bytes |

The MODS `455.229` Samsung/V2e/GA100 factory path installs vtable address `0x3e7dfb0` at `0xb0bc67`.

## Complete recovered vtable

The class has eleven recovered virtual slots in both builds. Every corresponding function has exactly the same byte length. The entire vtable retains the same structure and slot ordering across the `0x43` Hi4-to-Hi8 model-table change.

| ID | Slot | `455.127` | `455.229` | Size | Provisional role | Classification |
| --- | ---: | ---: | ---: | ---: | --- | --- |
| `M00` | `0x00` | `0xadfe40` | `0xafae70` | 66 | Destructor | Lifecycle |
| `M01` | `0x08` | `0xadfe90` | `0xafaec0` | 71 | Deleting destructor | Lifecycle |
| `M02` | `0x10` | `0xae01e0` | `0xafb210` | 227 | Samsung model/specification validation and interface setup | Configuration support |
| `M03` | `0x18` | `0xad4d30` | `0xaefd60` | 137 | Lane-repair type dispatcher | Repair |
| `M04` | `0x20` | `0xadfdc0` | `0xafadf0` | 127 | Row-repair type dispatcher | Repair |
| `M05` | `0x28` | `0xaded20` | `0xaf9d50` | 1,225 | Repaired-row summary/reporting workflow | Read-only repair inspection |
| `M06` | `0x30` | `0xad2750` | `0xaed780` | 261 | Lane-repair payload/mask encoder | Repair helper |
| `M07` | `0x38` | `0xad2a00` | `0xaeda30` | 432 | `DEVICE_ID` IEEE 1500 read workflow | Read-only |
| `M08` | `0x40` | `0xad4100` | `0xaef130` | 389 | Active-site/channel orchestration calling a repair helper | Repair orchestration |
| `M09` | `0x48` | `0xad2860` | `0xaed890` | 364 | Active-site iteration and dispatch through virtual slot `M10` | Repair orchestration |
| `M10` | `0x50` | `0xad51c0` | `0xaf01f0` | 934 | Per-site/channel `SOFT_LANE_REPAIR` workflow | Repair |

`Lifecycle` and `Configuration support` are cartography labels, not IEEE 1500 transaction classifications. No slot is currently demonstrated to configure HBM addressability, density, or shadow state.

## Per-method evidence

### M00 — destructor

- Replaces the active vtable with a base-class vtable during destruction.
- Releases the owned interface object at object offset `+0x78`.
- Destroys internal containers and strings.
- Contains no Host-to-JTAG transaction.

### M01 — deleting destructor

- Performs the same cleanup as `M00`.
- Releases the 128-byte object allocation.
- Contains no Host-to-JTAG transaction.

### M02 — model validation and interface setup

Evidence from diagnostics:

```text
Attempt to setup Samsung HBM interface with a non-Samsung model
Unsupported Samsung HBM specification
```

- Validates the supplied internal HBM model.
- Rejects non-Samsung models.
- Selects behavior according to HBM specification.
- Installs or associates a model-specific definition set through the helper at `0xaecf70` in `455.229`.
- Contains no direct Host-to-JTAG transaction.

### M03 — lane-repair dispatcher

Evidence:

```text
Lane repair: unsupported repair type '%s'
```

- Dispatches repair type `1` to `0xaef390` in `455.229`.
- Dispatches repair type `2` to `0xaedd70`.
- Rejects other repair types.
- The dispatched helpers still require individual Host-to-JTAG mapping.

### M04 — row-repair dispatcher

Evidence:

```text
Row repair: unsupported repair type
```

- Performs a prerequisite check through `0xaf7a80` when required by an input flag.
- Dispatches repair type `1` to `0xafa220` in `455.229`.
- Dispatches repair type `2` to `0xafa8a0`.
- Rejects other repair types.
- The dispatched helpers still require individual Host-to-JTAG mapping.

### M05 — repaired-row summary

Evidence includes:

```text
== HBM Repaired Rows Summary
Site %u row repairs
Stack %u row repairs
Channel %u row repairs
Bank %u row repairs
Total repaired rows: %u
```

- Iterates sites, stacks, channels, banks, and repaired-row records.
- Formats and reports existing repair information.
- No addressability or shadow-state control has been observed.

### M06 — lane-repair payload/mask encoder

Evidence:

```text
Lane repair is not supported for lane type: %s
```

- Converts supported lane-type and lane-index fields into two 16-bit output values.
- Uses masks such as `0x00ff`, `0xff00`, and a shifted `0x00e0` pattern.
- Rejects unsupported lane types.
- Does not call Host-to-JTAG itself; its output is consumed by a higher repair workflow.

### M07 — `DEVICE_ID` read workflow

- Looks up selector `7`, which resolves to the `DEVICE_ID` descriptor.
- The descriptor specifies WIR opcode `0x0e`, read-only register type `1`, and an 82-bit WDR.
- Obtains the owned `AmpereHbmInterface` object from offset `+0x78`.
- Calls `WriteInstr`, `WriteMode`, `Wait`, and `ReadData` through virtual slots `+0x20`, `+0x28`, `+0x88`, and `+0x38`.
- Passes `0x0f` as the channel-selection argument. Descriptor helper `0xb04db0` combines this value with the opcode and explicitly treats `0x0f` as the all-channel case; it is not a WIR opcode.
- Uses two 10.0 time intervals and then invokes `Wait` with method `0` and `-1.0`, which is a timeout/sentinel value rather than descriptor data.
- Reads three 32-bit words for the 82-bit WDR, trims the unused high bits, and passes the result to `0xd8b190` for decoding/copying.
- No WDR write occurs. This transaction is read-only device identification.

### M08 — active-site/channel orchestrator

- Obtains GPU, framebuffer, and active-site information.
- Iterates active HBM sites and channel/rank dimensions.
- Calls helper `0xaeefc0` for each selected tuple in `455.229`.
- Contains no literal WIR value and no direct low-level Host-to-JTAG call.
- Requires mapping of `0xaeefc0` and the virtual GPU/framebuffer accessors.

### M09 — active-site dispatcher

- Enumerates initialized HBM sites.
- Converts the channel-availability description into channel indices `a` through `h`.
- Calls virtual slot `+0x50`, which resolves to `M10` for `SamsungHbm2eGA100`.
- Therefore, `M09 -> M10` is a demonstrated internal class edge.

### M10 — `SOFT_LANE_REPAIR` workflow

- Looks up selector `4`, which resolves to the `SOFT_LANE_REPAIR` descriptor.
- The descriptor specifies WIR opcode `0x12`, read/write register type `3`, and a 72-bit WDR.
- Acquires the Host-to-JTAG/interface object at offset `+0x78`.
- Builds per-site/channel repair records, iterates four lane groups through `0xaedbe0`, and calls helpers `0xaeee30` and `0xaefdf0` to execute and collect the repair operation.
- Any `0x0f` value propagated by this path is a channel mask, not the instruction opcode.
- This is a lane-repair transaction. No field or operation in the recovered path changes density, row-address width, general addressability, or shadow configuration.

## Recovered WIR descriptor catalog

The common descriptor vector loaded by `SamsungHbm2eGA100::M02` is at `0xa7de1e0` in `455.229`. Its initializer constructs six objects of size `0x148` through `0xb05120`. The corresponding `455.127` initializer constructs the same objects with the same literal fields and applicability construction. The complete 48-byte static seed block used by these six applicability objects is byte-identical between the builds (SHA-256 `d0af61e717f214fe35bc430f0d246e33e27d7a0b4ed0c74defe87a547bb9853a`).

| Selector | Name | WIR opcode | Register type | WDR bits | Current classification |
| ---: | --- | ---: | ---: | ---: | --- |
| `0` | `BYPASS` | `0x00` | `3` | 1 | Configuration/support |
| `1` | `HBM_RESET` | `0x05` | `2` | 1 | Configuration/support |
| `4` | `SOFT_LANE_REPAIR` | `0x12` | `3` | 72 | Repair |
| `5` | `HARD_LANE_REPAIR` | `0x13` | `3` | 72 | Repair |
| `6` | `MODE_REGISTER_DUMP_SET` | `0x10` | `3` | 128 | Test/diagnostic |
| `7` | `DEVICE_ID` | `0x0e` | `1` | 82 | Read-only |

The GA100 HBM2E-specific vector loaded immediately afterward is at `0xa7de240` in `455.229`:

| Selector | Name | WIR opcode | Register type | WDR bits | Current classification |
| ---: | --- | ---: | ---: | ---: | --- |
| `2` | `SOFT_REPAIR` | `0x07` | `2` | 22 | Repair |
| `3` | `HARD_REPAIR` | `0x08` | `2` | 22 | Repair |
| `9` | `ENABLE_FUSE_SCAN` | `0xc0` | `3` | 26 | Test/repair support |

`Register type` is the binary's own descriptor enum. Its bits are checked before WDR reads and writes: value `1` permits reads, value `2` permits writes, and value `3` permits both.

### Descriptor object layout

| Offset | Recovered field |
| ---: | --- |
| `+0x00` | Descriptor name string |
| `+0x20` | Selector/catalog key |
| `+0x24` | Actual WIR opcode byte |
| `+0x25` | Register access-type byte |
| `+0x28` | WDR bit length |
| `+0x2c` | Flags, including single-channel restriction handling |
| `+0x30` | HBM-model applicability container |

Helper `0xaebd20` is a red-black-tree lookup over the filtered descriptor map. It compares the requested selector against node key `+0x20` and returns the descriptor pointer held at node `+0x28`; its failure diagnostic is `Unknown WIR: %u`. Helper `0xaecf70` filters the source vectors by descriptor applicability before inserting them into that map.

## Complete Samsung descriptor inventory

Four vectors form the Samsung catalog family in `455.229`: the common Samsung vector and the B-die, X-die, and HBM2E-GA100 extensions. `SamsungHbm2eGA100` loads only the common vector plus the HBM2E-GA100 extension, giving it nine available descriptors.

| Catalog scope | Selector | Name | WIR | Access | WDR bits | Recovered caller/workflow | Classification |
| --- | ---: | --- | ---: | --- | ---: | --- | --- |
| Common; active on HBM2E GA100 | `0` | `BYPASS` | `0x00` | Read/write | 1 | Companion/final instruction in repair workflows | Support |
| Common; active on HBM2E GA100 | `1` | `HBM_RESET` | `0x05` | Write-only | 1 | No direct `SamsungHbm2eGA100` lookup caller recovered | Configuration/support |
| Common; active on HBM2E GA100 | `4` | `SOFT_LANE_REPAIR` | `0x12` | Read/write | 72 | `M03` soft-lane branch and `M10` | Repair |
| Common; active on HBM2E GA100 | `5` | `HARD_LANE_REPAIR` | `0x13` | Read/write | 72 | `M03` hard-lane branch | Repair |
| Common; active on HBM2E GA100 | `6` | `MODE_REGISTER_DUMP_SET` | `0x10` | Read/write | 128 | Paired test-mode/register-dump helpers on catalog variants that also provide selector `8`; no direct GA100 HBM2E caller recovered | Test/diagnostic |
| Common; active on HBM2E GA100 | `7` | `DEVICE_ID` | `0x0e` | Read-only | 82 | `M07` | Read-only |
| Samsung B-die extension | `3` | `HARD_REPAIR` | `0x08` | Write-only | 21 | Hard row-repair workflow | Repair |
| Samsung B-die extension | `9` | `ENABLE_FUSE_SCAN` | `0xc0` | Read/write | 26 | Fuse-scan enable/readback around row-repair inspection | Test/repair support |
| Samsung X-die extension | `2` | `SOFT_REPAIR` | `0x08` | Write-only | 21 | Soft row-repair workflow | Repair |
| Samsung X-die extension | `3` | `HARD_REPAIR` | `0x08` | Write-only | 21 | Hard row-repair workflow | Repair |
| Samsung X-die extension | `8` | `TEST_MODE_REGISTER_SET` | `0xb0` | Write-only | 37 | Paired with selector `6` by test-mode/register-dump workflow | Test |
| Samsung X-die extension | `9` | `ENABLE_FUSE_SCAN` | `0xc0` | Read/write | 26 | Fuse-scan enable/readback around repair inspection | Test/repair support |
| Samsung HBM2E-GA100 extension; active | `2` | `SOFT_REPAIR` | `0x07` | Write-only | 22 | `M04` soft row-repair branch | Repair |
| Samsung HBM2E-GA100 extension; active | `3` | `HARD_REPAIR` | `0x08` | Write-only | 22 | `M04` hard row-repair branch | Repair |
| Samsung HBM2E-GA100 extension; active | `9` | `ENABLE_FUSE_SCAN` | `0xc0` | Read/write | 26 | GA100 fuse-scan/readback helper used by row-repair paths and repaired-row inspection | Test/repair support |

Repeated selectors are alternative model-specific definitions, not simultaneous duplicate map entries. The applicability filter selects the definition compatible with the active HBM model before insertion.

### Targeted write-capable review

The complete active `SamsungHbm2eGA100` set contains four write-capable entries not named as lane or row repair:

- `BYPASS` is IEEE 1500/TAP support used as a companion or terminal instruction by repair flows.
- `HBM_RESET` is a one-bit reset operation. No direct lookup caller was recovered in the mapped `SamsungHbm2eGA100` workflows, and no data flow connects it to density or addressing.
- `MODE_REGISTER_DUMP_SET` is a diagnostic/test descriptor. Its recovered consumers pair it with `TEST_MODE_REGISTER_SET` on older Samsung catalog variants; selector `8` is absent from the HBM2E-GA100 catalog.
- `ENABLE_FUSE_SCAN` enables a fuse-scan/readback sequence used by row-repair and repaired-row inspection code. Its callers consume repair/fuse state rather than normal memory geometry.

The Samsung-wide X-die catalog adds the write-only `TEST_MODE_REGISTER_SET`, but that descriptor is not loaded by `SamsungHbm2eGA100`. Its recovered caller is a test-mode/register-dump workflow, not a boot-time geometry or address-decoder path.

No descriptor is named for shadow state, density selection, address width, row-address extension, BIST, spare enablement, redundancy configuration, or normal memory geometry. Searches of descriptor names and their currently mapped callers found no such field or state transition.

### Catalog diff: `455.127` versus `455.229`

- The active HBM2E-GA100 vector contains the same three entries in both builds: selectors `2`, `3`, and `9` with identical names, WIR values, access modes, WDR lengths, flags, and vector ordering.
- Its complete 56-byte static applicability seed block is byte-identical between builds (SHA-256 `35a8b23ac4bc7ce613f562302ab87aa619f650b601dc4e2b7c0e2241fa3140d5`).
- The six-entry common vector is likewise field-identical, and its 48-byte applicability seed block is byte-identical as previously recorded.
- The B-die and X-die catalog memberships and literal descriptor fields are unchanged; they do not acquire a new geometry/configuration instruction in `455.229`.

Therefore the `0x43` Hi4-to-Hi8 profile change did not alter the available `SamsungHbm2eGA100` IEEE 1500 command set or the applicability data used to select its descriptors.

### Hidden-capacity decision at the catalog boundary

At this boundary, the answer is **no**: no write-capable, non-repair Samsung HBM2E-GA100 descriptor has a recovered caller or payload semantics that plausibly modify memory geometry or internal addressing. The only non-repair writes are reset, bypass/support, diagnostic mode-dump setup, and fuse-scan support. This is a bounded static-analysis result, not proof that no mechanism exists in firmware or another subsystem.

## FRB/FUB follow-up boundary

The negative catalog decision was followed by the planned GA100 FRB/FUB inspection.

| Artifact | `455.127` size | `455.229` size | Recovered native purpose | Hidden-capacity assessment |
| --- | ---: | ---: | --- | --- |
| `frb_ga100.bin` | 4,792 | 4,792 | `RunRirFrb`; RIR/fuse-repair validation and persistence for row repair | Repair-only path |
| `fub_ga100.bin` | 5,304 | 5,332 | Encrypted field-programmable-fuse binary for named FPF/security use cases | No HBM geometry use case found |

Both blobs have approximately 7.8 bits/byte of Shannon entropy and expose no reliable plaintext command catalog. The MODS native executable explicitly contains decryption and exact-size validation paths for fuse binaries, including `Decrypted fuse binary contents`, `Fuse binary is not the correct size!`, and `RIR fuse binary is not the correct size!`.

The equal-offset `frb_ga100.bin` comparison finds 3,899 identical bytes out of 4,792 (81.36%). Differences are concentrated in several 16-byte-aligned blocks and later encrypted/container regions. The `fub_ga100.bin` comparison aligns best when the `455.229` payload is shifted by 28 bytes, consistent with its new/repacked leading header; 2,792 of the 5,304 aligned bytes are identical.

Native diagnostics bound FUB to field-programmable-fuse operations such as GSYNC enablement, HULK license enable/revocation, and DRAMID-read permissions. FRB is explicitly tied to fuse repair and warnings about attempting HBM row repair on already repaired fuses. No native caller from either loader reaches capacity selection, density programming, row-address width, shadow state, or normal GA100 framebuffer initialization.

## `AmpereHbmInterface` vtable boundary

The object stored at `SamsungHbm2eGA100 + 0x78` resolves to `Memory::Hbm::AmpereHbmInterface`. Its vtable address point is `0x3c8a9f0` in `455.127` and `0x3e7d070` in `455.229`.

| Slot | `455.127` | `455.229` | Demonstrated operation |
| ---: | ---: | ---: | --- |
| `+0x20` | `0xacfd10` | `0xaead40` | Write HBM IEEE 1500 instruction (`WriteInstr`) |
| `+0x28` | `0xacf520` | `0xaea550` | Write HBM IEEE 1500 mode (`WriteMode`) |
| `+0x38` | `0xad01e0` | `0xaeb210` | Read HBM IEEE 1500 data (`ReadData`) |
| `+0x88` | `0xace8a0` | `0xae98d0` | Wait/poll with method and timeout arguments |

The names above are supported by direct diagnostic strings in the concrete implementations. In particular, tracing `0x0f` into `WriteInstr` reaches descriptor helper `0xb04db0`, which shifts the channel selection by eight bits and ORs it with the descriptor opcode at `+0x24`. This proves that `0x0f` selects channels and cannot be interpreted as WIR `0x0f`.

## Targeted binary diff result

### Confirmed invariants

- Both versions expose exactly eleven vtable slots.
- Slot ordering is identical.
- Every corresponding method has the same byte length.
- `M00` through `M06`, `M09`, and `M10` have identical normalized instruction streams after relocation/address normalization.
- `M07` differs only in RIP-relative locations of floating-point constants; both builds use the same values `10.0` and `-1.0`.
- `M08` retains the same control flow but accesses several GPU/framework objects at different offsets and compares against relocated virtual-method addresses. These differences are consistent with surrounding object-layout/build drift, not a new Samsung HBM2E operation.

### Current conclusion

The `0x43` change from Hi4 in MODS `455.127` to Hi8 in MODS `455.229` did **not** add, remove, reorder, resize, or semantically change any recovered `SamsungHbm2eGA100` virtual method. No new method-level Host-to-JTAG or IEEE 1500 operation is visible in this class vtable.

This does not yet exclude changes in:

- helpers called by the vtable methods;
- JavaScript/native binding tables;
- dynamically constructed WIR/WDR payload data.

The complete active nine-entry catalog, its descriptor fields, and its applicability construction are unchanged. The FRB/FUB loader and consumer paths have also been bounded to fuse/row-repair and general FPF-security functions. Neither investigated boundary contains a Hi8-specific capacity or shadow-state operation.

## Current call graph

```text
Samsung/V2e/GA100 factory
    -> SamsungHbm2eGA100 object
        -> M02 model validation / definition-set setup
        -> M03 lane-repair dispatcher
            -> repair type 1 helper
            -> repair type 2 helper
        -> M04 row-repair dispatcher
            -> repair type 1 helper
            -> repair type 2 helper
        -> M05 repaired-row summary
        -> M06 lane-repair payload encoder
        -> M07 DEVICE_ID (WIR 0x0e, read 82-bit WDR)
        -> M08 active-site/channel repair orchestration
        -> M09 active-site iteration
            -> M10 SOFT_LANE_REPAIR (WIR 0x12, 72-bit WDR)
```

## Deferred work outside this closed branch

The active descriptor catalog and its applicability construction were recovered and classified without finding a hidden-capacity candidate. Further repair-specific work could still:

1. reconstruct every remaining field in the dynamically constructed lane- and row-repair payloads;
2. map deeper helpers called by `M03`, `M04`, `M08`, and `M10`; and
3. document additional repair-only semantics.

Those tasks do not currently justify continuing this hidden-capacity branch. Any renewed work should require new evidence connecting one of these paths to geometry, addressability, or shadow state.

## Safety boundary

This map is derived exclusively from static analysis. No IEEE 1500 instruction has been executed. The recovered opcodes and channel mask document binary behavior; they must not be treated as an authorization or safe procedure for issuing hardware transactions.
