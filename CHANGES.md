# Changes in this fork

This file lists the changes this fork makes on top of upstream DORY.

Based on: https://github.com/pulp-platform/dory at `add0d9c1be889b5f802b2606ced8c59acff8aa02`
(`master`, 305 commits after tag `v1.0`).

## Bug fixes

**MCHAN transfers over 32 KB were silently truncated on GAP8.**
Layers with a tile copy over 32 KB computed on stale L1 data on the chip while GVSOC stayed
bit-exact: GAP8's MCHAN length field is 15 bits, so a longer transfer moves `size & 0x7FFF`
bytes and still reports completion. `dory_dma.c` now splits every copy into commands of at most
32764 bytes (`MCHAN_MAX_TRANSFER_SIZE`), on whole lines if strided, all on one counter. (3d8d070)

**Channel-tiled copies hung with `SINGLE_CORE_DMA` and `ALWAYS_BLOCK_DMA_TRANSFERS`.**
A layer whose data took the 3-D or HWC-to-CHW copy path (for example a channel-tiled pooling input)
hung the cluster: only core 0 runs those per-row loops in that mode, but each row waited with
`dory_dma_barrier()`, whose team barrier the other cores never reach there. Rows now wait on the
MCHAN counter only (`dory_dma_wait_rows()`); the layer's own barrier still syncs the team. (b51c1ff)

**Loading weights from flash into HyperRAM could hang or leave a stale chunk on GAP8.**
`load_file_to_ram()` hung in some boots or left one 128-byte chunk stale, depending on timing: GAP8
readfs `pi_fs_read()` updates its position only after issuing each flash read, and a completion
handled first by the higher-priority PMSIS event task re-enters it with stale state. It now uses
`pi_fs_direct_read()` through a 4 KB L2 heap buffer and closes the file (~10x faster). (9dec4dc)

**`pulp_nn_linear` used one byte of each int32 bias.**
Fully connected layers with a bias and an 8-bit requantized output (the hidden layers of an MLP) got
wrong, usually near-zero biases: the kernel indexed the int32 bias array through its `int8_t *`
parameter. It now reads `int32_t` values, as `pulp_nn_linear_out_32` already did. The fix is in the
`pulp-nn` submodule (`IllinoisReliableAutonomyGroup/pulp-nn`, branch `master`, commit `2f46d87`).
(3d8d070)

**Square fully connected layers got transposed weights.**
A `Gemm` with `transB=1` and equal input and output sizes gave wrong results: the NEMO and Quantlab
frontends guessed the layout from `shape[0] == input_channels`, always true for a square layer, so
the backend transposed an already `[out][in]` tensor. `DORY_node.fc_weights_layout()` now reads it
from the op (`MatMul`, or `Gemm`'s `transB`), and each node's weight name is reset first. (3d8d070)

**Convolutions tiled from L3 by weights only read shifted input rows.**
The L3 template runs a layer that has a single row band through the middle-band `_L2` variant, which
was generated without top and bottom padding, so every output row came from shifted input rows. The
C parser now keeps the full padding on `_L2` when the layer is a single band. (4f9215c)

**Depthwise layers overran the L2 arena on PULP.**
A grouped convolution tiled from L3 (for example a MobileNet v1 depthwise layer at 320x240) got an
L2 weight buffer larger than the L3 tiler had planned, so `dmalloc()` returned NULL and the layer
wrote through it. `HW_node` counted a grouped layer's bias 16 times, which only Diana needs; it now
does so only for Diana targets. (4f9215c)

**Add layers that did not fit L2 hung at run time.**
The L3 check in `tiler_add.py` counted one input, so DORY generated an Add whose two inputs and
output overflowed the L2 arena; `dmalloc()` then returned NULL and the cluster hung without an
error. Both inputs are now counted, as in the L2 tiler, and code generation stops with
`Add ERROR: no L3-L2 tiling supported`. (4f9215c)

**`network_run_async()` wrote past its `args[]` array.**
The array had 4 slots (3-level target) or 5 (2-level), but 5 or 6 were written, overwriting nearby
stack data such as the cluster configuration. It is now sized 5 or 6, in the PULP network template
and in the GAP9 one. (3d8d070)

**A layer ran on a NULL L1 buffer when `pmsis_l1_malloc()` failed.**
When something else held cluster L1, the layer wrote through address 0 and produced plausible but
wrong output. `execute_layer_fork()` now skips the layer and increments `dory_l1_alloc_failed`,
which the application can read; `VERBOSE` builds also print an error. (3d8d070)

**The second `network_run()` halted GAP8.**
Each `network_run()` opens the cluster, runs and closes it; the first inference worked and the second
halted the chip. `pi_open_from_conf()` keeps a pointer to the cluster conf, which was a local of
`network_run_async()`, and `pi_cluster_close()` in `network_run_wait()` reads `conf->id` after that
frame is gone: it cleared the wrong cluster slot, so the next open reused freed driver data. The device
and its conf are now file-scope (`<prefix>cluster_dev`, `<prefix>cluster_conf`), and a failed
`pi_cluster_open()` returns a zeroed token instead of no value. (3d8d070, 546d875)

**Networks generated with `--prefix` did not compile.**
`network.h.t` declared an unprefixed `network_run_wait()` and `network.c.t` called an unprefixed
`network_run_async()`; both now use `${prefix}`. (07db4c0)

**The 2-level target's weights header did not compile under the GAP SDK.**
`weights_h_template.h` always included `<hal/pulp.h>`, which needs `PULP_CHIP_STR` from the PULP-OS
build; it is now included only for builds that do not use the GAP SDK. (3d8d070)

## Features

**Per-layer hooks.**
`dory_layer_done()` runs after each layer while its output is in L2, and `dory_layer_done_l3()`
instead when it stays in L3. Both are weak and empty; an application defines its own to use them.
(3d8d070, 4f9215c)

**`mem_init()` step hook.**
`dory_mem_init_step(step, err)` reports each step (`flash_open`, `fs_mount`, `ram_open`) and its
return code, before `mem_init()` exits on a failure. Weak and empty by default. (36edaf7)

## Build and tooling

**Submodule URLs.**
`pulp-nn` now points to `https://github.com/IllinoisReliableAutonomyGroup/pulp-nn.git` (branch
`master`), which carries the bias fix above, instead of a relative URL. `pulp-nnx` uses HTTPS, not
SSH. (3d8d070, b79b4af)
