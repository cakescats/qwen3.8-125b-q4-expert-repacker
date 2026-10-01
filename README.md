# qwen3.8-125b-q4-expert-repacker

`qwfn-pack` rewrites the routed experts of **Qwen3.8-Flash-Next** (125B MoE, GGUF `qwen4exp`, `UD-Q4_K_XL`) into an *expert pack*: one contiguous block per (layer, expert) instead of three slices scattered over three tensors. It is built for [QwFNfer](https://github.com/Apolog1ze-Dev/QwFNfer)-style engines that stream experts from an NVMe on every cache miss.

**Status:** the converter works and is verified. The engine-side reader is being measured in a QwFNfer fork and is not published yet, so for now the pack is something you build and check, not something an engine uses.

## Why

In the GGUF, the `gate`, `up` and `down` slices of one expert sit in three different tensors, so one expert miss costs three reads in three places on the drive. With the pack it is one read of ~3 MiB.

Measured on one machine (KIOXIA KXG8 NVMe, PCIe Gen4 x4, ext4, Linux 7.0), all accesses are misses, one run each:

| what is read | GB/s |
|---|---|
| random O_DIRECT reads, 768 KiB-1 MiB (one slice) | 6.0-6.1 |
| random O_DIRECT reads, 3 MiB (one whole expert) | 6.45-6.5 |
| an engine's miss pattern from the GGUF, 3 reads per expert, io_uring | 5.34-5.49 |
| the same pattern from the pack, 1 read per expert, io_uring | **5.91-6.01** (~+10%) |
| the same, thread-pool reader | no gain: that reader copies every block through a bounce buffer |

How much of that reaches tokens per second depends on how much of decode is spent waiting for the disk (about half on the machine above), so expect a few percent, not ten. It has not been measured on decode yet.

## Build

Linux only. Needs a C++17 compiler, CMake and an existing llama.cpp build (for `libggml-base`, the GGUF reader):

```bash
cmake -B build -DLLAMA_CPP_ROOT=$HOME/.unsloth/llama.cpp
cmake --build build -j
```

`LLAMA_CPP_ROOT` defaults to `~/.unsloth/llama.cpp`; pass `-DLLAMA_CPP_BUILD=<dir>` if `libggml-base.so` is not in `<root>/build/bin`.

## Use

```bash
build/qwfn-pack /path/to/Qwen3.8-Flash-Next-UD-Q4_K_XL-00001-of-00004.gguf /path/to/Qwen3.8-Flash-Next-UD-Q4_K_XL.qwpk
build/qwfn-pack --verify /path/to/...-00001-of-00004.gguf /path/to/....qwpk 256
```

- Any shard path works; the others are found by name.
- The pack for `UD-Q4_K_XL` is **77.1 GB** (0.13% padding). It took 38 s on the machine above. The tool checks free space first.
- The GGUF is only read and stays needed: the dense weights and the prefill still come from it.
- Reads and writes use O_DIRECT, so the page cache is left alone. The file is preallocated, written to `OUT.partial` and renamed when complete. The header goes in last, so an interrupted run never leaves a file a reader would accept.
- After writing, 64 random experts (`--verify N` to change) are compared byte for byte with the GGUF.

## Format

`src/qwfn_pack.h` documents it. In short:

- a 16 KiB header: magic `QWFNPAK1`, version, layer/expert counts, the slot alignment, and a fingerprint of the source (shard sizes plus each expert tensor's shard, offset, type and shape);
- then, per layer, `n_expert` blocks, each starting on a 4 KiB boundary;
- each block is laid out exactly as a QwFNfer RAM-tier slot: `gate | up | down`, each part padded to the engine's direct-I/O alignment, with the payload at the same in-slot offset the engine computes. For the Q4 file that alignment is 512 bytes, because GGUF aligns tensor data to only 32 bytes. A read of one block therefore lands in a slot as is, with no copy.

A reader must accept a pack only if the fingerprint matches its GGUF and the per-layer layout equals its own slot layout.

## Tested with

`unsloth/Qwen3.8-Flash-Next-GGUF`, `UD-Q4_K_XL` (4 shards). Other quants of the same model should work (the layout follows the file), but only Q4 has been run.

## License

Apache License 2.0; see `LICENSE` and `NOTICE` (parts of the code come from QwFNfer).
