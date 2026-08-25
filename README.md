# cmp170hx10g-to-80g

> [!WARNING]
>
> ## The CMP 170HX 10 GB → 80 GB unlock has NOT been solved yet
>
> Published research reports that a CMP 170HX can expose 80 GB of distinct HBM, but the complete configuration required to reproduce this reliably and operate it stably is still unknown.
>
> This repository collects the known facts, experiments, sources, register information, and research leads related to the problem.

# Objective

The NVIDIA CMP 170HX 10 GB is based on GA100 and uses Samsung HBM.

The objective of this project is to understand how the card can apparently expose **80 GB of real HBM**, determine what configuration is missing from the currently known methods, and eventually reproduce a stable 80 GB configuration.

The project deliberately distinguishes between:

* **confirmed facts**
* **experimental observations**
* **strong inferences**
* **unverified hypotheses**

A larger framebuffer reported by software is not considered proof of additional physical memory.

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

# Published 80 GB Result

A major source for this project is:

**A Canary in the Crypto Mine: Defeating Stack Protection in a GPU Secure Coprocessor**
Jon Pry et al.

The paper reports that after opening the relevant framebuffer protection mechanisms, a shadow-register write could be used to change the memory configuration.

Most importantly, the authors report writing distinct hash patterns throughout the expanded memory space and reading them back intact.

This is the strongest currently known evidence that the 80 GB configuration can expose **distinct physical memory rather than simple address aliasing**.

The exact sequence required to reproduce that result publicly is still incomplete.

# 80 GB Stability Problem

The same paper reports that the 80 GB configuration was not initially stable.

Under `gpu-burn`, the authors reported:

```text
40 GB: clean
80 GB: 2796 errors
```

Increasing the HBM refresh frequency reportedly resulted in:

```text
0 errors
```

but reduced performance from approximately:

```text
94.6 TFLOPS
```

to:

```text
64.6 TFLOPS
```

or roughly a 32% throughput loss.

This is an important clue.

It suggests that obtaining 80 GB of distinct memory and obtaining a **correct 80 GB operating profile** may be separate problems.

# Samsung HBM Identification

A Chinese investigation used NVIDIA MODS `hbm.jse` on a CMP 170HX 10 GB.

Source:

https://blog.kkk.rs/archives/71

MODS reports information including:

```text
Vendor Name    = Samsung
Model Part #   = 0x41
HBM Revision   = XA2_8HI
Stack Height   = 8
Channels       = abcdefgh
Density/chan   = 8 Gb
Gen2 Supported = yes
```

This is a major observation.

The CMP10 HBM therefore identifies through NVIDIA's HBM tooling as a Samsung 8-high, 8-Gb/channel configuration.

# The 8 Gb / 80 GB Paradox

This creates one of the central questions of the project.

MODS sees:

```text
Samsung
XA2_8HI
8 Gb/channel
```

while the published research reports that 80 GB of distinct memory can be accessed after additional configuration.

Several explanations remain possible.

The physical DRAM may genuinely be 8 Gbit and the current interpretation of the 80 GB result may be incomplete.

Alternatively, the physical DRAM may contain more capacity than the normal HBM configuration exposes.

Another possibility is that the Samsung logic die exposes a restricted geometry even though additional physical rows exist.

Binning, repair, redundancy, or product configuration may also be involved.

None of these explanations is currently proven.

# Samsung 0x41 and 0x43

The MODS-related material shown in the Chinese investigation distinguishes the CMP profile from another Samsung profile.

The CMP profile is associated with:

```text
0x41
Samsung
HBM2
X-die
Revision A2
8-high
XA2_8HI
```

A separate profile is associated with:

```text
0x43
Samsung
HBM2E
8-high
16 GB class
?_16GB_8HI
```

This is an interesting research lead because NVIDIA software appears to distinguish Samsung 8-Gb/8-high and 16-Gb/8-high HBM profiles.

However, `0x43` should not currently be treated as a Samsung commercial Device ID mapping confirmed by a public Samsung datasheet.

The evidence currently comes from NVIDIA/MODS-related material.

# Samsung Flashbolt

Samsung officially produced 16 GB HBM2E stacks in the Flashbolt generation.

Relevant package references include:

```text
KHAA84901B-MC16
KHAA84901B-JC16
KHAA84901B-MC17
KHAA84901B-JC17
```

A 16 GB 8-high stack corresponds to:

```text
8 × 16 Gbit = 128 Gbit = 16 GB
```

Samsung reference:

https://semiconductor.samsung.com/dram/hbm/hbm2e-flashbolt/

The CMP10 has **not** been proven to contain one of these Flashbolt packages.

# K4C6E1K6MB

TechInsights identified:

```text
K4C6E1K6MB
```

as a Samsung 16-Gbit HBM2E DRAM die found in an Intel Xeon CPU Max 9462.

Reference:

https://www.techinsights.com/blog/samsung-k4c6e1k6mb-3rd-gen-16gb-hbm2e-dram-memory-floorplan-analysis

This provides a useful known Samsung 16-Gbit HBM2E reference.

However:

> **There is currently no evidence proving that K4C6E1K6MB is the DRAM die used in the CMP 170HX 10 GB.**

The possible relationship between:

```text
CMP XA2_8HI
Flashbolt KHAA84901B-*
K4C6E1K6MB
```

remains a research question.

# GA100 / A100 Register Comparison

The community project:

https://github.com/Consensus-Protocol/cmp170hx

contains a read-only register corpus covering multiple Ampere devices, including:

```text
CMP 170HX 10 GB
A100 40 GB
A100 80 GB
```

Useful HBM-related state includes registers such as:

```text
FBPA_CFG0
FBPA_CFG1

FBPA_MRS_0
FBPA_MRS_1
FBPA_MRS_2
FBPA_MRS_8
FBPA_MRS_WL_RL

FBPA_HBM_CFG0
FBPA_TRAINING_STATUS

IEEE1500-related registers
CSTATUS
```

This corpus demonstrates that CMP/A100 comparisons already exist.

The remaining opportunity is to perform a more systematic comparison of the **same CMP10** across:

```text
stock
40 GB
80 GB
```

and then compare those states against A100 40 GB and A100 80 GB.

Multiple snapshots per state are necessary to eliminate dynamic registers.

One particularly interesting pattern is:

```text
CMP40 == CMP80 == A10040
```

while:

```text
CMP80 != A10080
```

Such fields may identify parameters that remain in a 40-GB-like state when the CMP is switched to an 80 GB geometry.

They are candidates, not values that should automatically be copied.

# A100 80 GB Is Not a Complete Reference

An A100 80 GB is useful for understanding the GA100-side 80 GB geometry.

However, public A100 80 GB configurations commonly use SK hynix HBM2E, while the CMP10 under investigation uses Samsung HBM.

Therefore:

```text
A100-80 GA100 geometry
```

is useful, but:

```text
A100-80 HBM timings
A100-80 MRS
A100-80 training
A100-80 vendor-specific configuration
```

cannot automatically be assumed to be correct for the CMP.

A more useful conceptual target would be:

```text
GA100 80 GB geometry
+
appropriate Samsung HBM profile
+
training appropriate to the CMP
```

# Refresh and Timing

Refresh is a major research lead because of the published stability result.

Important parameters include:

```text
tREFI
tRFC
refresh mode
refresh multiplier
temperature-dependent refresh
```

A major hypothesis is that the 80 GB geometry may be enabled while some parameters remain appropriate for the smaller memory configuration.

Possible stale state includes:

```text
refresh
timings
MRS
WL/RL
training
PHY configuration
```

This could potentially explain why 80 GB is addressable but unstable.

It is not yet known whether this is the actual cause.

# Generated Timing State

The visible GA100 timing registers may not necessarily represent the active timings.

Existing community research indicates that the memory controller can use internally generated timing values such as:

```text
TIMING*_GEN
```

rather than only the obvious visible timing registers.

Configuration such as:

```text
USE_TIMING_REGS
```

may affect which timing source is active.

Therefore, comparing or modifying only the obvious timing registers may be insufficient.

Generated timing state is an important target for future dumps.

# Mode Registers

HBM Mode Registers are another important comparison target.

Relevant state includes:

```text
MR0
MR1
MR2
MR3
MR4
MR7
MR8
MR15
WL/RL
```

The project needs to distinguish between controller-cached MRS state and the actual configuration programmed into the HBM where possible.

# DevInit / FBFLCN / Training

Some important memory parameters may not come directly from static register values.

They may be generated during initialization by:

```text
DevInit
FBFLCN
memory training
VBIOS memory tables
PHY initialization
```

This is particularly relevant when comparing 40 GB and 80 GB states.

A correct 80 GB configuration may require a coherent initialization profile rather than one isolated register change.

# NVIDIA MODS Research Lead

NVIDIA MODS is an important research lead because it already recognizes multiple Samsung HBM models.

Useful search targets include:

```text
XA2_8HI
16GB_8HI
?_16GB_8HI

SamsungHbmModelKey
SamsungHbmModelMap
VendorHbmModelMap
HbmModel

Die::X
Revision::A2
SpecVersion::V2
SpecVersion::V2e
```

The useful question is whether the detected model is later used to select different:

```text
timings
refresh parameters
MRS
training parameters
vendor-specific initialization
```

If MODS only performs diagnostics, this path may stop there.

If it contains model-specific HBM setup information, the difference between the Samsung 8-Gb and 16-Gb profiles could provide valuable clues.

This is one research lead among several, not an assumption about where the final solution must exist.

# IEEE1500

HBM2/HBM2E exposes an IEEE1500 wrapper.

Relevant concepts include:

```text
WIR
WDR
WBR
WBY
Capture
Shift
Update
```

JEDEC defines standard HBM operations, while vendor-specific functionality may also exist.

Publicly observed GA100 registers include:

```text
0x009A3CB4 I1500_INSTR
0x009A3CB8 I1500_MODE
0x009A3CBC I1500_DATA
0x009A3CC0 I1500_SHADOW_WIR
0x009A3CC4 I1500_SHADOW_WDR
0x009A3CC8 I1500_STATUS
```

Public CMP10 dumps have shown:

```text
I1500_INSTR = 0xF0E
```

IEEE1500 remains interesting for understanding HBM identification and possible vendor-specific configuration.

However, unknown instructions must **not** be brute-forced.

IEEE1500 includes test, repair, reset, and other operations that may be unsafe.

# Xeon Max Research Lead

Intel Xeon CPU Max 9462 is interesting because TechInsights identified Samsung K4C6E1K6MB 16-Gbit HBM2E on that platform.

A bare-metal system could potentially provide a known Samsung 16-Gbit reference for:

```text
timings
refresh
memory-controller behavior
RAS/ECC
possibly Mode Register state
```

It is not known whether Sapphire Rapids exposes enough low-level HBM state to make this useful.

A Xeon Max should therefore be considered an experimental reference platform, not a confirmed path to the CMP unlock.

# Existing Code

The main existing CMP code base is:

https://github.com/amoghmunikote/cmpunlocker

It already contains low-level GPU register access primitives such as:

```text
GPU_REG_RD32()
GPU_REG_WR32()
```

This makes a targeted read-only GA100/FB/FBPA/HBM register dumper practical.

# Main Sources

### A Canary in the Crypto Mine: Defeating Stack Protection in a GPU Secure Coprocessor

Jon Pry et al.

Important for:

* the reported 80 GB configuration
* framebuffer protection
* the shadow-register operation
* full-capacity hash testing
* 80 GB stability testing
* refresh-based stabilization

### Chinese CMP170HX / NVIDIA MODS investigation

https://blog.kkk.rs/archives/71

Important for:

* Samsung HBM identification
* `0x41`
* `XA2_8HI`
* 8 Gb/channel
* 8-high stack
* MODS `hbm.jse`
* the separate Samsung 16-GB-class profile

### Consensus-Protocol/cmp170hx

https://github.com/Consensus-Protocol/cmp170hx

Important for:

* GA100 register documentation
* CMP dumps
* A100 dumps
* FBPA
* MRS
* timing state
* IEEE1500 observations

### amoghmunikote/cmpunlocker

https://github.com/amoghmunikote/cmpunlocker

Important for:

* existing CMP unlock research
* low-level GPU access
* experimental tooling

### Samsung Flashbolt HBM2E

https://semiconductor.samsung.com/dram/hbm/hbm2e-flashbolt/

Important for official Samsung 16 GB HBM2E products.

### TechInsights K4C6E1K6MB

https://www.techinsights.com/blog/samsung-k4c6e1k6mb-3rd-gen-16gb-hbm2e-dram-memory-floorplan-analysis

Important for a physically identified Samsung 16-Gbit HBM2E die.

# Open Research Leads

The main current leads are:

* determine exactly what the published shadow-register operation changes
* compare the same CMP10 in stock, 40 GB, and 80 GB states
* extend register dumps to generated timings and refresh state
* understand Samsung `XA2_8HI`
* compare Samsung 8-Gb and 16-Gb profiles known to NVIDIA MODS
* investigate MRS differences
* investigate DevInit / FBFLCN / training-generated state
* understand the GA100 IEEE1500 interface
* investigate Samsung vendor-specific HBM configuration
* determine the physical DRAM die used in the CMP
* investigate possible relationships with Flashbolt / K4C6E1K6MB
* use a known Samsung 16-Gbit platform such as Xeon Max as a possible reference
* determine whether instability is caused by physical binning or incomplete initialization

# Safety

Initial investigation should remain read-only.

Do not perform:

```text
SPI writes
eFuse / OTP writes
VRM writes
PMBus voltage writes
I2C voltage writes
arbitrary MMIO writes
unknown IEEE1500 commands
unknown repair/test operations
```

Future write experiments should only be performed after the relevant register or command is understood and should use explicit address/bit allowlists with a known recovery procedure.

# What Would Count as Success?

A real solution is not:

```text
nvidia-smi reports 80 GB
```

A convincing result should demonstrate:

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

We currently have strong evidence that:

```text
CMP 170HX 10 GB uses Samsung HBM

MODS identifies the tested memory as:
0x41 / XA2_8HI / 8 Gb/channel / 8-high

an experimental GA100 80 GB geometry is known

published research reports 80 GB of distinct memory

that 80 GB configuration showed stability problems

increasing refresh reportedly eliminated the errors
```

But we still do not know:

```text
the exact physical Samsung DRAM die

why an 8-Gb/channel identified stack can apparently expose 80 GB

the exact shadow-register configuration used for full capacity

the correct stable Samsung 80 GB memory profile

whether the instability is caused by binning or configuration

the complete reproducible unlock procedure
```

**The CMP 170HX 10 GB → stable 80 GB problem remains open.**

