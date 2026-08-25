# CMP 170HX Samsung HBM Capacity-Profile Hypothesis

> [!IMPORTANT]
>
> **Research result:** Reverse engineering of the NVIDIA MODS **Samsung HBM identification path** confirmed that the stock CMP 170HX Samsung profile is identified by the composite key `0x4108` as Samsung HBM2 `XA2_8HI`, while the Samsung model table in MODS `455.229` maps raw model part `0x43` to the Samsung HBM2E `?_16GB_8HI` profile. The reconstructed consumers of these Samsung model profiles are limited to device identification and density decoding, Host-to-JTAG/IEEE 1500, and memory-repair functionality. No direct path from either Samsung profile to GA100 DevInit, refresh, timing configuration, or activation of the additional address capacity required for 80 GB was found.

## Scope and provenance

This branch follows a research lead documented in the Chinese article ["170Hx 10G Unlock to 80G Exploration"](https://blog.kkk.rs/archives/71), published on 9 August 2026 and updated on 10 August 2026. The article contains two different kinds of evidence that must not be conflated:

1. output reportedly obtained by running NVIDIA MODS `hbm.jse` on actual cards; and
2. a fragment of an NVIDIA HBM model table that the author says was found in source code.

The article is currently the direct provenance for both items in this repository. The MODS package, its exact files, and the original source file containing the model table still need to be recovered and independently inspected. Until then, the table fragment is **reported source code**, not independently verified source provenance.

The article labels `455.263.1` as the NVIDIA driver/MODS environment in which `hbm.jse` produced usable output, and says that its `470.10` environment did not. Inspection of the article's original full-resolution photograph confirms that its `/home` directory contained `455.219`, `455.263.1`, `470.10`, and `NVMT`. These are therefore MODS runtime-directory names, not merely public display-driver versions. The exact `455.263.1` files have not yet been recovered.

### Local acquisition status

A public MODS download index referenced by the Korean [AllThatLinux MODS page](https://atl.kr/dokuwiki/doku.php/nvidia_mods) led to a Google Drive ISO named `Mods 455.iso`. It has been acquired locally as an ignored research artifact with the following identity:

```text
file          = artifacts/mods/Mods-455.iso
ISO label     = MODS 455
size          = 431,593,472 bytes
SHA-256       = 7422705d2a802c7c15f94856e70c3aea8a74679be3bfee59e1c09fd91a69df45
embedded tree = home/455.127
```

This artifact is **MODS 455.127**, not the `455.263.1` build named by the Chinese article. It therefore cannot yet reproduce the article's software environment.

The ISO does contain:

```text
home/455.127/hbm.jse  13,616 bytes
home/455.127/mods     43,483,864 bytes
```

The local `hbm.jse` has SHA-256 `e7642e208507fa273c4cc05cd21880dc479ec4c8beb55053dc7fd018813a6db0`. It is a high-entropy precompiled `.jse` artifact, consistent with NVIDIA MODS documentation describing `.jse` files as precompiled JavaScript. It must not yet be called encrypted: the container/bytecode format and any additional protection have not been identified. A normal plaintext `strings` pass does not expose the HBM model names.

The accompanying packed `mods` executable has SHA-256 `2da221573e0f8b1a436a2ae16dec8c0c319c165e7e7611ab402fdb9849bd274f`. Its initial appearance as a sectionless, single-load, static x86-64 ELF was an artifact of UPX 3.94 packing. UPX 5.2.0 recovered a 167,627,616-byte, dynamically linked ELF with SHA-256 `4b7fcbc46e9135a42513410ffb6187ffe59342f9c062e26bd8f87fe6a61369e7`. The unpacked binary exposes the HBM model strings and substantial HBM/IEEE 1500 implementation detail.

A second [public Google Drive image](https://drive.google.com/file/d/1gVJGwlKEsBnkt-WnXF7l8_qxBlIpvzyR/view), described by its distributor as MODS `455.229`, has also been acquired. Its ISO misleadingly retains the directory name `home/455.127`, exactly as the distributor warned, but the executable itself contains the version string `455.229` and differs from the real `455.127` binary:

```text
file                 = artifacts/mods/Mods-455.229.iso
ISO size             = 411,717,632 bytes
ISO SHA-256          = c02f7932766af4e545be5b7466b80d581e51591db31d0421c4ad1dfff5485f1b
embedded path        = home/455.127/mods
internal MODS string = 455.229
packed SHA-256       = 73392c7c9f750c19650f6d070b406fa5c714301ce040000f671fcb8eab9ff6df
unpacked size        = 170,746,192 bytes
unpacked SHA-256     = 567192a497415178a2221e9dcf7cc2324e857ab7d7ea3670fd73d95ec6c85bb0
```

This neighboring build resolves the 4-high/8-high question independently of the article's source fragment: MODS `455.229` already maps Samsung `0x43` to `?_16GB_8HI`. The immediate acquisition target nevertheless remains the exact `455.263.1` runtime for direct reproduction and binary diffing.

## Confirmed stock-card profile reported by MODS

For the tested stock CMP 170HX 10 GB, the article reproduces this MODS summary:

```text
Memory Size  : 10240 MB
FB Vendor    : Samsung
HBM Revision : XA2_8HI
RAM Protocol : HBM2
```

It also reproduces `DEVICE_ID` decoding for sites 0, 1, 2, 3, and 5. Every shown site reports:

```text
Model Part #   : 0x41
Vendor Name    : Samsung
Stack Height   : 8
Channels Avail : abcdefgh
Address Mode   : Only Pseudo Channel Mode Supported
Density/chan   : 8 Gb
ECC Supported  : yes
Gen2 Supported : yes
```

Therefore, the stock CMP 170HX 10 GB profile observed in that test is:

```text
model part    = 0x41
stack height  = 8
vendor        = Samsung
specification = HBM2
die/revision  = X / A2
model name    = XA2_8HI
density       = 8 Gbit per channel
```

This observation establishes what the Samsung logic die reports. It does **not**, by itself, establish the maximum physical capacity manufactured into each DRAM die.

The card's stock 10 GB framebuffer size must also not be confused with the per-stack `DEVICE_ID` geometry. The article treats the reported 8-Gbit-per-channel, 8-high profile as an 8 GB stack geometry and five such stacks as a conventional 40 GB configuration, while the CMP product exposes only 10 GB in its stock controller/product configuration.

## Research question

The reported NVIDIA table distinguishes at least the following Samsung HBM profiles:

| Lookup key | Article-reproduced source fragment | MODS `455.127` | MODS `455.229` | CMP observation |
| --- | --- | --- | --- | --- |
| `(0x41, 8)` | HBM2, X/A2, 8-high, `XA2_8HI` | Same | Same | Observed on every displayed stock CMP stack |
| `0x43` | HBM2E, unknown die/revision, 8-high, `?_16GB_8HI` | HBM2E, unknown die/revision, **4-high**, `?_16GB_4HI` | HBM2E, unknown die/revision, **8-high**, `?_16GB_8HI` | Not observed on the tested CMP |

The mapping is now a demonstrated NVIDIA binary change between `455.127` and `455.229`. In `455.127`, the separate mixed-case string `?_16GB_8Hi` belongs to a later-constructed non-Samsung model table and is consistent with SK hynix; it is not the Samsung `0x43` entry. In `455.229`, the uppercase string `?_16GB_8HI` is directly used by the Samsung map initializer for raw key `0x43` and stack-height enum `Hi8`. The article's source fragment therefore agrees with a real later NVIDIA mapping, even though the exact source revision remains unknown.

The `16GB` text in these internal names is also ambiguous without the original type definition and comments. It may denote stack capacity or die density. It must not automatically be translated into “16 Gbit per channel” or “16 GB stack” for every stack-height combination.

The important question is not merely where NVIDIA maps these identifiers to human-readable names, but what NVIDIA software does differently after identifying one profile or the other.

The objective of this research branch is to recover the code path from Samsung HBM model detection to any model-dependent configuration of HBM geometry, refresh, timings, mode registers, training, repair, or IEEE 1500 state.

## Primary hypothesis: internal Samsung geometry

The CMP 170HX 10 GB may contain more physical capacity than the 8-Gbit-per-channel geometry reported through its normal `DEVICE_ID`, while the Samsung logic die remains configured to expose an 8-Gbit geometry.

On the GA100 side, the following values select a geometry corresponding to 80 GB:

```text
CFG1 = 0x02779000
LMR  = 0x28B
```

These controller-side values may be necessary but insufficient. Published research reports that, after opening the framebuffer protection mechanisms, a shadow-register write exposed 80 GB of distinct memory. A missing Samsung-side operation may therefore configure one or more of the following:

- density;
- row-address width;
- address decoding or geometry;
- test-mode address bits;
- hidden capacity;
- redundancy or repair state;
- another shadow register that controls one of these properties.

The working hypothesis is that NVIDIA's handling of the `0x41` and `0x43` profiles may reveal this missing operation, either directly through a Samsung-specific command or indirectly through a table, initialization profile, or generated configuration.

## Secondary hypothesis: refresh and stability

Even after 80 GB is correctly and distinctly addressable, the configuration may remain unstable because the active refresh and timing profile still corresponds to the smaller geometry.

The `0x41` and `0x43` paths may select different values or profiles for:

- `tREFI`;
- `tRFC`;
- refresh multiplier or refresh mode;
- temperature-compensated refresh;
- HBM mode registers;
- write/read latency;
- other timings;
- training parameters.

An indirect selection is relevant. For example, either of the following would support the hypothesis:

```text
0x41 -> RefreshProfileA
0x43 -> RefreshProfileB
```

```text
0x41 -> TimingTable[index A]
0x43 -> TimingTable[index B]
```

## Target call graph

The desired result is a demonstrated path resembling:

```text
DEVICE_ID
    -> Samsung model detection
    -> HbmModel / internal model representation
    -> model-dependent branch or table lookup
    -> timing / refresh / MRS / IEEE 1500 / training / geometry configuration
```

Both the `0x41` and `0x43` paths must be followed as far as the locally available code and binaries permit.

### The lookup key is not always the raw model part

The reported Samsung table contains both of these entries:

```cpp
{ SamsungHbmModelKey(0x41, 4),
  HbmModel(SpecVersion::V2, Vendor::Samsung, Die::B,
           StackHeight::Hi4, Revision::B, "B_4HI") },

{ SamsungHbmModelKey(0x41, 8),
  HbmModel(SpecVersion::V2, Vendor::Samsung, Die::X,
           StackHeight::Hi8, Revision::A2, "XA2_8HI") },
```

Consequently, raw model part `0x41` is ambiguous in this implementation. For Samsung, the effective discriminator can be the tuple `(model part, stack height)`. Searches and call-graph recovery must preserve both values rather than treating `0x41` as a globally unique profile ID.

The article reports the following later-version entry:

```cpp
{ 0x43,
  HbmModel(SpecVersion::V2e, Vendor::Samsung, Die::Unknown,
           StackHeight::Hi8, Revision::Unknown, "?_16GB_8HI") },
```

This establishes a known NVIDIA Samsung HBM2E profile entry, but not that this profile is present on, accepted by, or appropriate for the CMP 170HX.

The local `455.127` binary instead constructs the `0x43` entry with:

```text
specification = V2e
vendor        = Samsung
die           = Unknown
stack height  = Hi4
revision      = Unknown
name          = ?_16GB_4HI
```

The divergence itself is now a research target. Comparisons must be made by MODS build rather than assuming the model map is invariant.

If the human-readable model name is used only for display, the investigation must follow the internal fields instead:

```text
vendor       = Samsung
specification = HBM2 or HBM2E
die          = X
revision     = A2
stack height = 8
density      = 8 or 16 Gbit
model part   = 0x41 or 0x43
```

## Initial search priorities

Search first for definitions and references related to:

```text
s_SamsungHbmModelMap
SamsungHbmModelMap
SamsungHbmModelKey
VendorHbmModelMap
HbmModel
HbmModelKey

XA2_8HI
16GB_8HI
?_16GB_8HI

Die::X
Revision::A2
SpecVersion::V2
SpecVersion::V2e
```

The constants `0x41` and `0x43` are useful only in an HBM, Samsung, model, or device-ID context because isolated searches will produce many false positives.

For every model definition found, continue through its references to determine:

1. where the model object is constructed or populated;
2. which functions call its methods;
3. which code reads die, revision, specification, density, or model part;
4. which branches or table indices depend on those values;
5. which configuration functions are ultimately called.

## High-value technical targets

Model-dependent paths should be checked for connections to:

- IEEE 1500 operations: `WIR`, `WDR`, `WBR`, `WBY`, capture, shift, and update;
- `DEVICE_ID` and WIR `0x0E`;
- mode registers, especially MR0, MR1, MR2, MR3, MR4, MR7, MR8, and MR15;
- refresh, `tREFI`, and `tRFC`;
- timing generators and generated timing state;
- density, row, column, bank, geometry, and address decoding;
- HBM training, FBPA, FBFLCN, and DevInit;
- lane or row repair, redundancy, fuses, and spare resources;
- test modes and vendor-specific Samsung operations.

Known GA100 IEEE 1500-related registers of interest include:

```text
0x009A3CB4  I1500_INSTR
0x009A3CB8  I1500_MODE
0x009A3CBC  I1500_DATA
0x009A3CC0  I1500_SHADOW_WIR
0x009A3CC4  I1500_SHADOW_WDR
0x009A3CC8  I1500_STATUS
```

CMP 170HX dumps showing `I1500_INSTR = 0xF0E` appear consistent with an all-channel `DEVICE_ID` operation and are a starting point, not evidence that unknown instructions are safe.

## Reported 40 GB aliasing result

The Chinese article reports an important experiment performed after selecting an 80 GB GA100-side address space with `CFG1` and `LMR`, but without the unidentified additional initialization:

- ranges within both approximately 5--35 GB and 45--75 GB could be written and read when tested separately;
- after writing distinct patterns to both ranges, the lower range was corrupted while the upper range remained readable;
- the author reports a concrete alias relationship between an address near 12 GB and one near `52 GB + 0xb800` after L2 eviction;
- the conclusion was that the apparent 0--80 GB address space folded onto the same physical 40 GB rather than exposing 80 GB of independent storage;
- increasing refresh did not change this aliasing behavior.

This result is **reported experimental evidence**, not yet independently reproduced in this repository. If accurate, it cleanly separates two problems:

```text
GA100-side 80 GB address-space selection
    != Samsung logic-die exposure of an additional row-address bit

logic-die capacity exposure
    != refresh/timing stability at full capacity
```

The article proposes that the missing logic-die configuration enables the highest row address bit, described as `RA[14]`, possibly through a vendor-specific IEEE 1500 WIR. That exact mechanism is still a **hypothesis**. The reported aliasing supports a missing address-decoder state, but does not prove which register, instruction, bit, or protocol changes it.

## Current evidence status

### Confirmed from the article's reproduced MODS output

- The tested stock CMP 170HX reports 10,240 MB, Samsung HBM2, and `XA2_8HI`.
- Every displayed Samsung HBM site reports model part `0x41`, stack height 8, and density 8 Gbit/channel.
- MODS `hbm.jse` reportedly worked with version `455.263.1` in the author's environment.

### Confirmed only as content printed in the article

- The displayed Samsung model table maps `(0x41, 8)` to HBM2 X-die A2 `XA2_8HI`.
- The displayed table maps `0x43` to HBM2E 8-high `?_16GB_8HI`; that mapping is now independently confirmed in the local `455.229` binary, although the displayed source fragment's own revision remains unidentified.
- The displayed table also maps `(0x41, 4)` to the distinct B-die `B_4HI` profile.

The original source fragment still needs provenance recovery. The two principal mappings are now independently present in NVIDIA binaries, but their downstream consumers must still be traced.

### Confirmed directly in local MODS `455.127`

- The unpacked executable contains the `XA2_8HI` and `?_16GB_4HI` strings at file offsets `0x34158a1` and `0x34158b1`, corresponding to virtual addresses `0x38158a1` and `0x38158b1`.
- Static initialization at virtual address `0x49f8b6` builds the composite key `0x4108`, matching `(model part 0x41, stack height 8)`, and associates it with the X/A2 HBM2 `XA2_8HI` model fields.
- Static initialization at virtual address `0x49fb28` builds raw key `0x43` and associates it with V2e/Samsung/unknown/Hi4/unknown plus `?_16GB_4HI`.
- Model objects have five integer discriminators followed by their name. Consumer code consistently accesses specification at object offset `+0x08`, vendor at `+0x0c`, die at `+0x10`, stack-height at `+0x14`, and revision at `+0x18`.
- The binary contains specialized RTTI class names for `Memory::Hbm::SamsungHbm2BDie`, `SamsungHbm2XDie`, `SamsungHbm2XDieGA100`, `SamsungHbm2e`, and `SamsungHbm2eGA100`.
- Factory/validation code at `0xad5620`, `0xad6a80`, and `0xae01e0` branches on the internal model fields and selects different Samsung HBM interface implementations. The nearby diagnostic context is memory repair and IEEE 1500, so this is confirmed model-dependent behavior but **not yet evidence of boot-time capacity or refresh configuration**.
- The binary exposes the formatter `Name(%s) Opcode(%X), RegType(%s), WdrLen(%u), HbmModels(%s)` at file offset `0x3532fa8` / virtual address `0x3932fa8`, confirming that IEEE 1500 command descriptions carry an opcode, register type, WDR length, and an HBM-model applicability set.

### Confirmed directly in local MODS `455.229`

- The executable's internal version string is `455.229`; the enclosing `home/455.127` directory is stale packaging metadata.
- The unpacked executable contains uppercase `?_16GB_8HI` at file offset `0x35ebc7f`, virtual address `0x39ebc7f`. It also contains the separate mixed-case SK hynix-style `?_16GB_8Hi` at file offset `0x35ebc9e`; the two entries must not be conflated.
- At virtual address `0x4a0a46`, the Samsung initializer builds the five HBM model fields V2e, Samsung, unknown die, **Hi8**, unknown revision. The stack-height field is assigned enum value `1`; the corresponding `455.127` initializer assigned `0` for Hi4.
- At virtual address `0x4a0aa8`, that object is inserted under raw key `0x43`. Its name storage was initialized from `?_16GB_8HI` at `0x4a09fe`.
- This proves that the Samsung `0x43` entry changed from `?_16GB_4HI`/Hi4 in `455.127` to `?_16GB_8HI`/Hi8 by `455.229`. The article's 8-high entry is therefore not merely a misplaced reference to the SK hynix string.
- The binary contains RTTI and complete vtables for `Memory::Hbm::SamsungHbm2e` and `Memory::Hbm::SamsungHbm2eGA100`. The latter type name is at virtual address `0x3e7df60`, its RTTI object begins at `0x3e7df88`, and its vtable address point is `0x3e7dfb0`.
- Factory code beginning at virtual address `0xb0ba50` consumes the internal `HbmModel` discriminators. It first branches on vendor at model offset `+0x04`; the Samsung branch beginning at `0xb0bc10` then branches on specification at offset `+0x00`. For Samsung V2e on GA100, the constructor path writes the `SamsungHbm2eGA100` vtable pointer at `0xb0bc67`. Thus, the `0x43` entry is operational input to class selection, not merely a display label.
- This factory does **not** inspect stack height while selecting the Samsung HBM2e GA100 implementation. The `Hi8` change in the `0x43` record is therefore not, by itself, a demonstrated control-flow switch at this point.
- The same factory contains the diagnostics `No Host-to-JTAG interface supported for Volta`, `No Host-to-JTAG interface supported for Ampere`, and `Unsupported HBM specification`. The selected Samsung implementations contain explicit soft/hard row-repair, fuse-check, and IEEE 1500 paths. This places the confirmed consumer in MODS' diagnostic/repair interface rather than the normal GPU boot or DevInit path.
- Elsewhere, validation code at `0xe0b290` compares framebuffer rank count against an HBM stack-height value and emits `FrameBuffer rankCount (%d) does not match HBM StackHeight (%d).` at `0xe0b410`. This proves that stack height is semantically consumed somewhere in MODS, but the recovered path has not yet demonstrated that this value comes from the same Samsung model map rather than an independently decoded `HBMDevice` representation.
- The `DEVICE_ID` decoder is now directly connected to the model map. At `0xd8aa10`, MODS extracts model part from decoded-device offset `+0x02`, vendor from `+0x13`, and stack-height enum from `+0x12`, converts the latter to 4 or 8, and enters the lookup routine at `0xd8a8f0`.
- In the Samsung lookup branch at `0xd8a9b8`, model part `0x41` is explicitly transformed into a composite 16-bit key by placing `0x41` in the high byte and the numeric stack height in the low byte. A stock 8-high device therefore selects `0x4108`; other Samsung model parts, including `0x43`, remain raw keys. This independently confirms the effective-key rule inferred from `SamsungHbmModelKey(0x41, 8)`.
- On a successful lookup, code at `0xd8a980` copies all five internal model discriminators and the name into the output object. The caller at `0xd8aa40` then uses the returned specification discriminator to decode the raw `DEVICE_ID` density code: HBM2e uses the 12-entry 16-bit table at `0x3ea5740`, while HBM2 uses the six-entry table at `0x3ea5758`. The HBM2 table decodes density codes 1--6 as 1, 2, 4, 8, 16, and 32 Gbit/channel; the HBM2e table decodes codes 1--12 as 1, 2, 4, 8, 6, 8, 0, 12, 12, 16, 18, and 24 Gbit/channel.
- This establishes another functional effect of the `0x41`/`0x43` distinction: it selects HBM2 versus HBM2e interpretation of the density field. It is still a read/interpretation path. It does not program density, expose a row-address bit, or prove that the `16GB` name is the value returned for a particular CMP device.
- Direct cross-references to the lookup and density-decoding routines in this executable are limited to `HBMDevice` construction/serialization, the `hbm.jse`-style property/reporting path, model comparison, and memory-repair setup. The active-site model accessor at `0xd81420` has only two direct callers; its external caller at `0xb0e2aa` is surrounded by `Unable to determine the HBM model`, HBM fuse-repair, and repair-command diagnostics. No direct reference from a DevInit, refresh, timing, or normal framebuffer-initialization routine has been found.
- Consequently, the current result proves a model-table change and a downstream repair/JTAG class-selection effect. No code path has yet shown that selecting `0x43`, or merely changing its stack-height field to `Hi8`, enables additional rows, changes refresh, or alters normal initialization on a Samsung `0x41` stack.

### Confirmed model-specific IEEE 1500 catalog in `455.127`

Static construction beginning near virtual address `0x46a117` creates named IEEE 1500 descriptors and groups them into model-specific catalogs. The constructor arguments expose the WIR opcode and WDR bit length. The common catalog includes:

| Name | Opcode | WDR length | Observed role |
| --- | ---: | ---: | --- |
| `BYPASS` | `0x00` | 1 bit | Standard bypass |
| `HBM_RESET` | `0x05` | 1 bit | Reset-related; never execute during this research |
| `SOFT_LANE_REPAIR` | `0x12` | 72 bits | Repair path |
| `HARD_LANE_REPAIR` | `0x13` | 72 bits | Persistent repair path; never execute |
| `MODE_REGISTER_DUMP_SET` | `0x10` | 128 bits | Mode-register diagnostic state |
| `DEVICE_ID` | `0x0e` | 82 bits | Device identification |

The Samsung X-die catalog selected by the X/A2 model includes descriptors such as:

| Name | Opcode | WDR length |
| --- | ---: | ---: |
| `HARD_REPAIR` | `0x08` | 21 bits |
| `SOFT_REPAIR` | `0x07` | 21 bits |
| `TEST_MODE_REGISTER_SET` | `0xb0` | 37 bits |
| `ENABLE_FUSE_SCAN` | `0xc0` | 26 bits |

The Samsung HBM2E catalogs use different model-dependent WDR lengths. Recovered variants include 22/23-bit soft-repair data, 24/25-bit hard-repair data, and 64/128-bit `PPR_INFO` data with opcode `0xf1`.

These values are useful static identifiers, **not candidate commands to try**. The surrounding strings and call paths explicitly concern lane repair, row repair, fuse checks, test-mode registers, and pseudo-hard-repair behavior. Several operations may be destructive or persistent. Their significance is architectural: MODS demonstrably selects different vendor command definitions and payload geometries from the internal HBM model.

No recovered descriptor has yet been tied to normal boot-time density selection, row-address-width exposure, or refresh programming. In particular, `TEST_MODE_REGISTER_SET` is only a promising name; it is not evidence that its fields control hidden capacity.

### Immediate call-graph lead

```text
DEVICE_ID fields
    -> Samsung effective key (0x41 => model part + stack height; 0x43 => raw key)
    -> Samsung model-map lookup
    -> HbmModel {spec, vendor, die, height, revision, name}
    -> specification-dependent DEVICE_ID density decoding
    -> model validation / Host-to-JTAG Samsung interface factory
    -> B-die, X-die, X-die-GA100, HBM2E, or HBM2E-GA100 implementation
    -> model-filtered IEEE 1500 command descriptions used by memory-repair paths
```

The presently recovered path terminates in diagnostic and repair behavior. The next task is to find an independent consumer of the model object in DevInit, geometry, timing, training, mode-register setup, or refresh code. The separate framebuffer/HBM consistency validator is a lead, but it is not yet connected to the Samsung map.

### Strong research lead

- The reported 40 GB folding behavior after GA100-side 80 GB geometry selection is consistent with a missing Samsung logic-die address-decoder configuration.
- A model-dependent NVIDIA initialization path is a plausible place to find density, geometry, refresh, or timing differences.

### Still hypothetical

- The CMP's physical DRAM dies are native 16-Gbit parts binned or configured down to 8 Gbit.
- Profile `0x43` reveals the operation needed to expose the CMP's hidden capacity.
- The missing operation is specifically a vendor-defined IEEE 1500 instruction that enables `RA[14]`.
- The `0x41` and `0x43` profiles select different refresh or timing parameters relevant to stable 80 GB operation.

## Analysis method

1. Inventory the locally available MODS packages, scripts, native libraries, firmware, VBIOS images, dumps, logs, symbols, and documentation.
2. Identify the exact MODS version and locate `hbm.jse` plus every module it loads.
3. Determine whether `hbm.jse` decodes the Samsung model itself or calls a native MODS API.
4. If decoding occurs in a native library, identify the library and trace string, data, and code cross-references.
5. Locate adjacent tables and compare Samsung with SK hynix handling to recover the general structure.
6. Compare the `0x41` and `0x43` branches, including stripped-binary switch statements and structure tables.
7. Follow consumers beyond diagnostics into initialization or configuration code whenever such a path exists.

Useful static-analysis tools include Ghidra, IDA, radare2, `objdump`, `readelf`, `nm`, and `strings`, according to local availability.

## Evidence standard

Every material finding should be classified as one of:

### Confirmed

Directly demonstrated by source code, symbols, a string plus a valid cross-reference, clear disassembly, or authoritative documentation.

### Strong inference

The behavior is very likely from converging evidence, but one part of the path is not directly demonstrated.

### Hypothesis

A plausible lead that still requires confirmation.

Each finding should record, when available:

```text
file
binary offset or address
symbol name
calling function
called function
minimal source or disassembly excerpt
interpretation
confidence level
```

Finding only the `XA2_8HI` display string is not a substantive result. The investigation must continue until the model influences configuration, or until the path demonstrably ends in diagnostics or crosses into a missing module.

## Safety boundary

This research is strictly static and read-only.

Do not:

- brute-force IEEE 1500 opcodes;
- issue unknown IEEE 1500 instructions;
- write GPU MMIO registers;
- perform SPI, eFuse, OTP, PMBus, I2C, or voltage writes;
- modify firmware;
- execute any hardware operation that can change HBM configuration, repair state, or test state.

Unknown vendor-specific instructions must never be assumed safe. Static inspection and read-only comparison are the only authorized operations on this branch.
