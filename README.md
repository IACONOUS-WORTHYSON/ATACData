# ATACData

> ⊚ **FOR YOUR EYES ONLY.** This repository is published under The 75th Avenue License —
> **For-Your-Eyes-Only (FYEO) Edition v1.0**, Issuance No. **75AV-FYEO-001**. You are
> granted **one** right: to *look*. Cloning, copying, downloading for retention, forking,
> mirroring, executing, and redistribution are **prohibited** — see [`LICENSE`](LICENSE).

**Per-tensor damage tables and precision-allocation measurements for three
transformer models.** This repository is *data only* — the measured record behind a
single, testable claim:

> The metric the field allocates quantization bits by — reconstruction error,
> `‖unmap(map(W)) − W‖ / ‖W‖` — has **near-zero correlation (Spearman −0.031)**
> with the damage a tensor actually does to the model's output. Allocating a fixed
> byte budget by **measured** damage instead beats uniform quantization at **identical
> size**, and the gain **grows with model scale**.

No code lives here. These are the numbers a reviewer needs to check the claim.

---

## The headline, measured

Held-out quality-per-byte gain from spending a fixed byte budget where measured damage
says it matters, versus uniform Q4_1 at the same size. Ratio > 1 means the allocated
model is closer to full precision. "Held-out" = prompts the allocation was never fitted to.

| Model | Held-out mean gain | Prompts improved | Best single gain |
|-------|-------------------:|:----------------:|-----------------:|
| Qwen2-0.5B | 1.26× | 2 of 3 | 1.52× |
| Qwen3-4B   | 1.33× | 2 of 3 | 2.02× |
| Qwen3-8B   | **2.22×** | **3 of 3** | **2.74×** |

The win does not shrink with scale — it grows, and at 8B there is no regression.
Full per-prompt figures are in each model's `allocation.json`; the roll-up is in
[`cross_scale_summary.json`](cross_scale_summary.json).

---

## Layout

```
Precision-Allocation-Data/
├── LICENSE                      75th Avenue License, Exclusive Edition — Issuance 75AV-EX-002
├── README.md                    this file
├── MANIFEST.json                every data file with byte size + SHA-256
├── cross_scale_summary.json     the three-scale headline result
├── compression_sweep.json       150-run compression sweep (every format × every model)
├── summary.xlsx                 the same data as a 10-sheet workbook, for reading in Excel
├── activation/
│   └── zone_matrix.json         12 prompts × 5 chip regions — observed activation geography
└── data/
    ├── Qwen2-0.5B/
    │   ├── sensitivity.json      per-role mean damage + the −0.031 Spearman + tensor count
    │   └── allocation.json       uniform-vs-allocated KL, per prompt, at equal bytes
    ├── Qwen3-4B/
    │   ├── damage_table.json     full per-tensor table: damage × 4 formats, per tensor
    │   └── allocation.json
    └── Qwen3-8B/
        ├── damage_table.json     full per-tensor table
        └── allocation.json
```

---

## Data dictionary

### `data/<model>/damage_table.json` (Qwen3-4B, Qwen3-8B)
One entry per `nn.Linear` weight tensor in the model (253 each). Each entry:

| field | meaning |
|-------|---------|
| `kl`  | object `{format → KL}`. **The causal damage:** KL divergence of the model's next-token distribution from the full-precision model when *exactly this one tensor* is quantized to `format` and every other tensor is left at full precision. Higher = this tensor matters more. `Infinity` = that format is illegal for this tensor's shape. |
| `te`  | object `{format → error}`. The **reconstruction / translation error** `‖requant − W‖ / ‖W‖` for the same quantization. This is the metric the field uses. Compare it against `kl`: they disagree. |
| `n`   | number of weights in the tensor. |
| `role`| tensor role (`q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj`, `lm_head`, …). |
| `layer`| transformer layer index the tensor belongs to. |

Formats on the ladder: `q4_0`, `q4_1`, `q5_1`, `q8_0` (≈ 4, 4.5, 5.5, 8.5 bits/weight).

### `data/Qwen2-0.5B/sensitivity.json`
The 0.5B model was measured on GPU. Its full per-tensor table was not retained; this is
the distilled result:

| field | meaning |
|-------|---------|
| `per_role_mean_kl` | mean causal KL damage per role (Q4_1). Note `v_proj` carries ~23× the damage of `q_proj` while being ~44× smaller — the opposite of how uniform quantization spends bits. |
| `spearman_te_vs_kl` | **−0.031** — the rank correlation between reconstruction error and real damage across all 169 tensors. Effectively zero. This is the finding. |
| `n_tensors` | tensors ablated to build the table (169). |

### `data/<model>/allocation.json`
The result of the allocation experiment on each model.

| field | meaning |
|-------|---------|
| `budget_MB` | the fixed byte budget both models are held to (the size of the uniform-Q4_1 baseline). |
| `prompts` | per prompt, `{uniform_q4_1, allocated}` = next-token KL from full precision for each build. `fit` = the single prompt the allocation was calibrated on; `held_0/1/2` = held-out prompts. |
| `ratios` | `uniform_q4_1 / allocated` per prompt. > 1 means allocation wins. |

### `compression_sweep.json`
150 runs: every format applied uniformly to every model, with bits/weight, size, compression
factor, KL from full precision, and top-1 agreement. The raw material for the compression tables.

### `activation/zone_matrix.json`
For 12 prompts (5 complex, 7 simple), the fraction of total activation current landing in
each of 5 chip regions during a real forward pass. Supporting observational data — see caveat 4.

---

## Method & provenance

- **Damage** is KL divergence of the next-token log-probabilities from the full-precision
  model, measured by quantizing exactly one tensor at a time and leaving all others intact —
  isolating each tensor's causal contribution to output error.
- **Calibration prompt:** `"The capital of France is"`. **Held-out prompts:** an explanation,
  a poem, and a why-question — none seen during allocation.
- **Allocation** is a greedy spend of the byte budget over the format ladder, placing each
  byte where it removes the most measured damage.
- **Models:** Qwen2-0.5B, Qwen3-4B, Qwen3-8B (base weights from their public releases).
  0.5B and 4B measured in float32; 8B in bfloat16 (to fit host RAM).

## Honest caveats (read these)

1. **Single-prompt calibration.** The allocation is fitted on one prompt. At 0.5B and 4B it
   still improves two of three held-out prompts but regresses one; at 8B it improves all
   three. Multi-prompt calibration is the obvious next step and is not yet done here.
2. **Independence assumption.** The greedy allocator assumes tensors degrade independently.
   They do not — on 0.5B it predicted 6.06e-2 damage and measured 5.29e-2. Where a predicted
   and a measured number both exist, trust the measured one.
3. **Mixed dtype across scales.** 0.5B/4B damage is measured in float32, 8B in bfloat16.
   Every ratio is computed within a single dtype, so each is internally valid, but the three
   scales are not bit-identical measurement conditions.
4. **The activation map is observational, not causal.** `zone_matrix.json` records *where*
   activity concentrates during inference. Region placement is prompt-stable; it does **not**
   drive the output. Nothing in this dataset claims that a weight's position changes what the
   model computes.
5. **This is a quality-per-byte result, not a speed result.** It says a given byte budget can
   hold more model quality — not that any engine here runs faster than an existing one.

---

## License — For Your Eyes Only

Governed by **The 75th Avenue License — For-Your-Eyes-Only (FYEO) Edition v1.0**,
Issuance No. **75AV-FYEO-001** (see [`LICENSE`](LICENSE)). This is the framework's most
restrictive edition. It grants exactly one right — **sight**:

- **Permitted:** viewing, reading, and inspecting this data in place, for your own private
  evaluation.
- **Prohibited absolutely** (Article III): cloning (`git clone`, fork, or equivalent),
  copying, downloading for retention, mirroring, re-hosting, redistribution, execution,
  derivation, and **AI-training use of any kind**.

**How "view-only" is enforced — honestly.** No license and no archive can *physically*
stop someone who can read a file from copying it; a git repository is clonable by anyone who
can see it, and this instrument does not pretend otherwise (Article IV §4.1). What makes the
prohibition real is the **Immutable Deposit** (Article I): the authoritative master is
anchored by a **Zenodo DOI** (fixing exactly *what* the data is) and an **OpenTimestamps /
Bitcoin** proof (fixing exactly *when* it existed). Any copy appearing anywhere else is then
*provably* the holder's property, prior in time, and taken in breach — a self-proving
violation, dated and public before the copy could exist. Enforcement is by **proof, not by
lock**. If physical impossibility of copying is ever required, the only real route is to not
publish openly at all — keep the Deposit private and grant view access individually.

**Deposit anchors:**

- **OpenTimestamps proof: [`MANIFEST.json.ots`](MANIFEST.json.ots)** — created **2026-07-25**.
  It timestamps the SHA-256 of `MANIFEST.json`
  (`2cdfabdcae9f6ee44e925b81e63fcbadaf0928c030dfbcd7adbc676bb4b6f56a`), and because that
  manifest carries the SHA-256 of every other file, this single proof fixes the entire
  deposit — the `LICENSE` included. Submitted to the OpenTimestamps calendars
  `bob.btc.calendar.opentimestamps.org` and `btc.calendar.catallaxy.com`; the Bitcoin block
  attestation is pending (it bakes in after the next calendar aggregation — upgrade it with
  `ots upgrade MANIFEST.json.ots`, a few hours out).
  **Verify independently, trusting no one:** drop `MANIFEST.json` and `MANIFEST.json.ots`
  together at <https://opentimestamps.org>, or run `ots verify MANIFEST.json.ots`.
- Zenodo DOI: `<pending deposit>` — to be minted when/if you archive to Zenodo.

Required attribution notice, unmodified:

    ATACData — Created by IACONOUS WORTHYSON . Licensed under The 75th Avenue License —
    For-Your-Eyes-Only Edition v1.0 . Issuance No. 75AV-FYEO-001 . For your eyes only:
    view permitted, all copying forbidden.

Built upon ATAC (Alien Transformer Architecture Complete) and CAR (Composed Arithmetic
Primitives). This dataset is a separate work from the ATAC/CAR source repository.
