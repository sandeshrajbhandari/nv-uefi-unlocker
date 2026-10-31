# GA102M — RTX 3080 Laptop 16GB (`10de:249c`, ASUS SUBSYS `1043:1602`) — full-chain unlock on Windows

2026-11-01, ASUS ROG Strix G533QS (Cezanne APU + GA102M, **MUX-less Optimus, no MUX switch**), Windows 11,
nv-uefi-unlocker **v2.1.8 release tx build**, `-Family ga102` asset routing.

First documented `249c` run and, to our knowledge, the first MUX-less AMD-platform GA102M (whole 0x24xx mobile
span) success with an operator-visible warm-resume confirm prompt and a clean three-stage verification
(EFI transaction → in-OS reapply → in-OS read-back). Closest prior evidence: [#28](https://github.com/physicsdolphin/nv-uefi-unlocker/issues/28)
(GA104M `249d`, Cezanne, dGPU-direct). No firmware blobs are attached to this report — logs and register dumps only.

## Verification summary

- EFI transaction completed: `BOOTER wait_plm: PLM flip 0xffffff8f -> 0xffffffff after ~147000us - payload executed`,
  `fwsec ok (WPR2 verified)`, `SELECTOR RESULT: SS0=0x88888888 SS1=0x8`,
  `POST-FLR: verified (selectors survive AON)`, `CHAIN: LoadImage` of `bootmgfw.efi` (see [loader-unlock.log](loader-unlock.log)).
- Windows 616.56 took the card cleanly (no Code 43, no WHEA, no bugcheck). `nvidia-smi` power envelope shows the
  full-grade `130 W` cap (was silently clamped before); 16384 MiB intact.
- In-OS `ssdiag --reapply`: `reapply: SS0=0x88888888 SS1=0x00000008 -> OK`, exit 0.
- In-OS read-back: `VERDICT [ampere]: UNLOCKED (override values live)` (see [ssdiag-unlocked-verified.txt](ssdiag-unlocked-verified.txt)).

| Register | Locked (baseline [ssdiag-locked-baseline.txt](ssdiag-locked-baseline.txt)) | Unlocked |
| --- | --- | --- |
| FEAT_PLM 0x00823804 | `0xffffff8f` | `0xffffffff` (OPEN) |
| SS0 | `0x16547115` | `0x88888888` |
| SS1 | `0x5` | `0x00000008` |
| FEAT_READOUT_1 | `0x00400080` | `0x00000000` [full speed] |

| Item | Value |
| --- | --- |
| GPU | RTX 3080 Laptop 16 GB, `10de:249c`, BDF `01:00.0`, VBIOS `94.04.3e.00.bc` |
| Platform | AMD Cezanne: dGPU behind `1022:1633` @ 00:01.1; iGPU `1002:1638` @ 05:00.0 (panel owner) |
| VBIOS / driver TGP | `94.04.3e.00.bc` / 115 W base, 130 W boost; driver 616.56, Windows 11 |
| Secure Boot / Stick | OFF / 16 GB FAT32 EFI boot entry (no persistent NVRAM change) |
| Starting state | Warm resume after a Restart from Windows with an external unplugged; internal panel alive at confirm |

## On the observed path (things that differ on MUX-less hybrids)

1. **The BAR sweep at SAMPLE is all-ones (13/13) without read-errors** — on a laptop the dGPU parks in
   D3cold at UEFI time and BAR0 does not decode until the claim enables MEM decoding (`raw COMMAND write
   cmd=0x0007`). Do not reject a laptop run as "dGPU not brought up" on this basis; treat it as the
   D3cold norm, not the desktop-style failure it would be aotherwise.
2. **Screen dark between confirm and Windows boot** on MUX-less machines (console detached for the
   transaction, panel owner is the untouched iGPU GOP; one GOP handle surfaced, at 1920x1080).
3. **A failed prior attempt (confirm timeout) left the AMD iGPU at Code 43** — `loader-abort-attempt1.log`;
   the zero-write abort did not touch the GPU, but the broken POST state took the iGPU's panel down
   with it. `pnputil /restart-device` did NOT clear the Code 43. **Full EC discharge recovered it**
   (shutdown, unplug AC + all USB hosts, hold power 40–60 s, boot AC-only, lid open, no USB-C/DP-alt adapter).
   USB-C DP-alt-mode adapters seem to poison this machine's cold POST when attached around recovery boots.
4. **In-OS `ssdiag` can race the runtime-suspend** in the first minutes after unlock: one run saw
   `BAR0 reads all ones - device D3 / MEM decode off / driver unbound` ([ssdiag-d3cold-race.txt](ssdiag-d3cold-race.txt))
   while Device Manager and `nvlddmkm` — both already healthy. A rerun a minute later read the full FEAT
   cluster and provided the final UNLOCKED verdict. If ssdiag reports all-ones right after a boot but nvidia-smi
   shows a live WDDM card, suspect the D3cold race, not the unlock.
5. **Optional automation available**: the stick-armed marker (`i_know_what_i_am_doing.txt`) gives a 5 s auto-run
   for re-applying after cold boots, when needed — the unlock is volatile (per power cycle) and must be re-applied
   after every full power-off.

## Performance A/B (same method as [ad106-asus-z790](../ad106-asus-z790/report.md) `bench/bench_gemm.py`)

Torch `torch.mm`, contiguous `randn`, FP32 accumulate forced, warm-up + 20 timed iterations × 2 passes, median.
Unlocked side measured in the unlock boot session (ssdiag `UNLOCKED` in the same session); stock side after a full
power-down cycle with the relock re-confirmed by ssdiag `NOT_UNLOCKED` (`PLM 0xffffff8f`, `SS0 0x16547115`,
`SS1 0x5`, `READOUT_1 0x00400080` — identical to the locked baseline above). Same conditions both sides:
AC power, lid open, RUN offline, torch 2.10.0+cu130 / driver 616.56. Both sides sit at the VBIOS 130 W
board limit (`POWER-CAPPED` on every row) with throttled clocks bouncing ~1245–1785 MHz, so the per-clock
column (`TFLOPS/GHz`) is the primary comparison axis, exactly as the throughput-is-clock-limited note in the FAQ predicts.

| Case (N) | stock | unlocked | abs × | per-clock × |
| --- | --- | --- | --- | --- |
| FP32 control (4096) | 9.61 | 9.62 | **1.00×** | 0.98× |
| TF32 (4096) | 19.64 | 30.91 | 1.57× | 1.78× |
| FP16 (4096) | 39.46 | 62.24 | 1.58× | 1.82× |
| FP16 (8192) | 40.42 | 62.98 | 1.56× | 1.98× |
| BF16 (4096) | 40.03 | 69.53 | 1.74× | 2.25× |
| BF16 (8192) | 40.05 | 61.32 | 1.53× | 1.95× |

TFLOPS medians; raw data: [bench-stock.json](bench-stock.json), [bench-unlocked.json](bench-unlocked.json).

Falsifiability checks hold:

- **FP32 strict control is flat (1.00×)** — the lock/unlock signal is the tensor issue rate, not clocks or power;
- **numerical error vs FP64 identical in both states** (fp16 2.63e-4 / 2.77e-4, bf16 2.09e-3 / 2.05e-3 — same
  magnitude as the ad106 reference card);
- tensor per-clock gain band 1.78–2.25× matches the GA10x mobile reports ([#27](https://github.com/physicsdolphin/nv-uefi-unlocker/issues/27)
  ≈1.58×, [#71](https://github.com/physicsdolphin/nv-uefi-unlocker/issues/71) ≈1.83–1.86×) — `249c` ships from
  the factory with the same full-power segmentation clamp as its GA10x mobile siblings.

## Caveats / limitations of this report

- All GPU benchmark rows are power-capped at the VBIOS 130 W board limit at this chassis' throttled clock
  band; absolute TFLOPS therefore understate a derestrained desktop-style card. Per-clock is the
  faithful axis (see above). No MMA instruction-rate microbenchmark was run; the cuBLAS path is the verified one.
- The loader emitted `SCRUB: SEC2 fired but handoff stuck at 1` → `scrubber did not complete (ETimeout) - proceeding
  without it (community chain shape)`; transaction verified end-to-end regardless. Worth watching on other GA102M runs.
- `FEAT_OVR_PLM` remained AON-surviving as expected; WPR_PLM/WPR_CFG_PLM/FBPA_PLM remain closed
  (non-standard reserve writes were not part of the ga102 replay path).

## Rolling back

Deleting `v67-ga102-four-plm.bin` from the stick reverts the run to chain-replay (no unlock); restoring
`bootx64.efi.mint-backup` returns the stick to its original Linux Mint loader. Nothing was written to
the ESP or to any BIOS flash during the whole process.

## Sources

- EFI transaction log: [loader-unlock.log](loader-unlock.log) · aborted attempt #1: [loader-abort-attempt1.log](loader-abort-attempt1.log)
- ssdiag chain: [locked baseline](ssdiag-locked-baseline.txt) · [reapply refusal](ssdiag-reapply-refused.txt) (gate closed, pre-transaction) · [D3cold race](ssdiag-d3cold-race.txt) · [UNLOCKED](ssdiag-unlocked-verified.txt)
- Upstream compatibility context: [docs/compatibility.en.md](../../compatibility.en.md) (GA104M §device-12, GA102 desktop §device-9)
