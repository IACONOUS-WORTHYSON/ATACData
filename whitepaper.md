# Democratization of A.I.

*by IACONOUS WORTHYSON — Elad David Levi*

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.21545816.svg)](https://doi.org/10.5281/zenodo.21545816)

### The whole industry shrinks AI models by rounding numbers and hoping the model survives. I measured what actually breaks — and spent the memory there instead. Same size. Better model. The proof is below, and it's yours to check.

---

**Start with a kid.**

A kid with a school laptop and parents who can't spare a cent for Claude or GPT. Right now that kid is locked out of the most important tool of his generation — not because he isn't smart enough, but because the model won't fit on anything he owns, and the hardware that *would* run it costs more than his family will see in years. Hold him in your head. This entire paper is about him.

Because here's the part nobody says plainly: **modern AI is not gated by intelligence. It is gated by memory.**

## Memory is the gate

An AI model is, underneath, a giant list of numbers — billions of them. To run it, every one of those numbers has to sit in fast memory on a chip. Big model, big expensive chip. And if it doesn't fit, it doesn't run — there's no "a little slower," it simply won't start. So the real question of who gets to use frontier AI was never about algorithms. It's *can you afford the hardware.* For most of the planet, the answer is no.

Everyone's fix is the same: make the model smaller. Fine. But *how* they make it smaller is where this whole story turns.

## How you shrink a model — and yes, the good tools already do it well

The trick is called **quantization**, and the idea is simple: rounding. Store each of those billions of numbers with less precision — a number that took 16 units of memory now takes 4. Do that across the whole model and it shrinks four to eight times. Small enough, finally, to fit.

Let me be fair to the tools that already exist, because I'm not going to win by lying about them. llama.cpp, PyTorch's quantizers — they compress models, and they're *good* at it. Some of them are faster than anything I've built. I'm not here to tell you they're broken.

I'm here to tell you they all make the same decision the same wrong way.

## The lie in the ruler

Every one of these tools has to make one judgment call: *which parts of the model do I protect, and which do I round into the dirt?* You only have so much precision to spend. Where do you spend it?

The universal answer — the one the entire field runs on — is to protect the parts whose *numbers* move the most when rounded. Measure how far each number shifts, guard the big shifters, crush the rest. It sounds so obviously correct that almost nobody bothered to check it.

I checked it.

I took a model apart one piece at a time. I would damage exactly one part — round it hard — leave everything else perfect, and measure how much the model's actual *answers* changed. Then I put it back and did the next part. Every part. And then I asked the only question that matters: **does "the numbers moved a lot" predict "the answers got worse"?**

It does not. The correlation is **−0.031.** That is not weak. That is *nothing* — statistically indistinguishable from zero, and if it leans anywhere, it leans the wrong way.

Sit with that for a second. The single measurement the entire industry uses to decide where to spend precision **does not predict the thing it exists to predict.** The whole field is aiming with a ruler that doesn't measure distance.

> Think of it like judging which typo matters most by counting the letters that changed. "cat" → "car" changes one letter and means almost nothing. "now here" → "nowhere" changes a single space and flips the entire sentence. The *size* of the change tells you nothing about whether it wrecks the meaning. Model weights behave exactly like this — and the industry has been grading them by letter count.

How badly does the ruler miss? One kind of internal weight does roughly **23× more damage** when you round it than another kind — while being about **44× smaller.** The standard method lavishes its care on the big, dramatic-looking parts and starves the tiny, critical ones. It gets it precisely backwards.

## What I actually built

So I stopped guessing from the numbers and started measuring the answers — and then I spent the precision where the measurement said it truly mattered. Protect the handful of parts that break the model; round the robust majority hard. Same total size. Better model.

I want to be exact about what this is, because precision is the whole point: **it is still quantization.** I did not invent a new kind of arithmetic. What I changed is *where the care goes* — aimed by measured damage instead of by a proxy that doesn't work.

If you want the feel of it, picture the model as an ocean and a prompt as a tide moving through it. Most of the water barely stirs; a few currents carry the whole tide. You don't reinforce the entire ocean. You reinforce the currents that matter — and I found them by measuring, not by guessing.

## The proof — and it scales the right way

Same file size. My allocation against the standard method. Three models, small to large:

| Model size | Better, at the same size | Result |
|---|---|---|
| Small (0.5B) | **1.26×** average | 2 of 3 held-out tests improved |
| Medium (4B) | **1.33×** average | 2 of 3 improved |
| Large (8B) | **2.22×** average | **every** test improved — nothing regressed; best case 2.74× |

Don't just read the multipliers. Read the trend. **The bigger the model, the more this wins** — and on the largest one, nothing got worse at all. That is the pattern you want before you bet on something: it doesn't fade as the stakes rise. It grows.

In plain words: at the same size, for free, my compressed 8-billion model stayed roughly twice as faithful to the full-quality original as the standard method managed.†

## What this does for the kid

Here's the concrete version, no slogan. An 8-billion-parameter model **will not fit** on a cheap 8 GB consumer graphics card at 8-bit — a card that costs a few hundred dollars and already sits in millions of ordinary PCs. Drop it to 4-bit and it fits — and with my allocation it holds onto more of its quality on the way down.

That is the wall coming down. Today, running a real model means renting a data center — servers priced past anything the kid's family will ever touch. My work moves the floor: from a ten-million-dollar server to a single commodity graphics card. I'll be straight with you — that card is not yet literally his school laptop, and I'll be the first to say so. But it is most of the distance, and it is the part of the distance **nobody else is even measuring.**

## What I am NOT claiming — read this before you trust me

I measured the edges too, and I'll hand them to you myself. A pitch that hides its limits is hiding something worse.

- **This is not a speed play.** llama.cpp still runs inference faster than my setup. My claim is *quality per byte* — fitting more model into the same memory — not *speed per second.* Different axis. It happens to be the axis that decides whether the model fits at all.
- **This is not new hardware.** It needs none. It runs on what you already own.
- **This is not finished.** Right now I tune the allocation on a single example prompt — which is exactly why the two smaller models each had one test that didn't improve. Tuning on a handful of prompts instead of one is the obvious next step, and the clear road from *usually* wins to *always* wins.

None of that dents the core finding. The ruler is broken no matter what I build next — and that broken ruler is the ground everything here stands on.

## What I am asking

I'm asking you to fund the next stage of AI infrastructure: making any large model runnable by the people currently locked out. Not a demo — the tool. Point it at any model, get back a smaller one that is *measurably* better than today's best compression at the same size.

And I'm asking you to **not take my word for a single line of it.** Every measurement, on every model, with the raw data, is deposited publicly, timestamped to the Bitcoin blockchain, and held under my own license — free to inspect under strict terms, and provably mine. Read the fine print. Check the data. *Then* decide.

The data lives here: **[ATACData](https://github.com/IACONOUS-WORTHYSON/ATACData)** · DOI **[10.5281/zenodo.21545816](https://doi.org/10.5281/zenodo.21545816)**

## Where it goes next

The finding is done and it holds. The product is the short walk from here — multi-prompt tuning to close the last gap, then more model families, then a tool anyone can aim at a model and run.

---

**The industry rounds the model and hopes it still works. I measured what breaks, and spent the memory there.** That is the whole difference. And it is the difference between a kid who gets to run the model — and a kid who never does.

*— IACONOUS WORTHYSON*

<sub>† "2.22× / twice as faithful" is the ratio between how far each compressed model's answers drift from the full-quality original, measured the standard way researchers measure it (KL divergence). It is a real, reproducible figure; "twice as faithful" is the plain-English gloss of it.</sub>
