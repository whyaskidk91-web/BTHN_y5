# STO — Synthetic Thinking Organoid

**MOST RECENT VERSION: BTHN_y7**

A single artificial neuron with a **growing dendritic tree**, built from scratch in NumPy without backpropagation.

The goal of the project is **memory**: a neuron that can learn a new task without overwriting an old one (no "catastrophic forgetting"). Most design choices take their cue from how real dendrites work.

> Status: research prototype, single neuron.

---

**IMPORTANT**

Current research suggests that the STO takes mostly advantage of the given benchmark,

if memory and how memory actually improves remains to be tested, future results will be shared.

## Headline results

**1. Less forgetting than a backprop network**

Two tasks are learned one after the other (A, then B). Afterwards, both are tested again.
The score is `(error on B + error on A after learning B) / 2`. Lower is better.

Measured over **10 random task pairs × 5 seeds each**, against a 2-layer MLP trained with backpropagation:

| baseline (MLP) | median: tree forgets less by | worst pair |
|---|---|---|
| 2 or 4 hidden units, learning rate 0.1 | **≈ 5.7×** | ≈ 0.74× |

Tuning the MLP's size or learning rate made no difference. It forgets about the same at every setting.

**2. A single neuron solves XOR** (100 seeds, 2000 epochs)

| model | weights | XOR solved |
|---|---|---|
| **STO tree, growth off** | **4** | **98 / 100** |
| STO tree, growth on | ~8 (median) | 100 / 100 |
| MLP, 2 hidden, best learning rate | 9 | 79 / 100 |
| MLP, 4 hidden, best learning rate | 17 | 99 / 100 |

A flat single neuron can't solve XOR at all (0/100). The branched neuron matches a 17-weight backprop network with 4 weights.

---

## How the neuron works

### Structure
- The neuron owns a **bias** and decides whether to fire. Its **branches** own the weights.
- Inputs arrive at the **tips** of the dendritic tree and flow inward to the roots (like real dendrites).
- **Tips** respond with `exit = tanh(weight × entry)`.
- **Parent branches** average their children's exits and respond with a **bump**:
  `u = weight × entry`, `exit = u · exp(0.5 − 0.5·u²)`.
  The bump rises, peaks at `u = 1` and then falls again. Human cortical dendrites show the same kind of response, which lets a single neuron compute XOR (Gidon et al., *Science* 2020).
- The neuron sums its roots plus the bias (`activation`) and fires `y = s · tanh(activation)`, where `s` is its polarity (+1 or −1, see below).

### What each branch remembers
Each branch keeps a small local record: `entry`, `exit`, `gain` (how much it changed the signal), `involvement` (its share of the tree's structure, set at creation: root 1, two children ½ each, and so on), a running `activity` average and a running `avg_blame` average.

### Learning rule (no backprop)
1. The neuron computes its blame: `blame = (t − y) / gain`.
2. The blame is shared between the roots and the bias.
3. Each branch's blame = `share × involvement × reach`, where **reach** is the product of the gains of all branches above it, i.e. how strongly its signal currently gets through.
4. Weight update: `weight += lr × blame × sign(entry)`. The blame sets **how much**, and the input's sign sets **which way**. Nothing is divided by the input, so small inputs can't make it explode.

**Centered gain:** for bump branches, gain is measured relative to the peak: `gain = (exit − 1) / (entry − 1/weight)`. Left of the peak it's positive, right of it negative, so blame flows the right way on both sides of the bump. For tanh branches the center is 0, so this is the ordinary `exit / entry`.

### Self-regulation
- **Homeostasis:** if the whole neuron goes silent (running average of its root output < 0.1), all branches are woken up at once. This fixed the "dead neuron" runs (XOR went from 95/100 to 100/100).
- **Growth:** a tip that keeps getting heavily blamed (`avg_blame > 0.015`) sprouts two children. At most one sprout per ~400 steps (a recovery period), and at most 15 branches.
- **Polarity with a critical period:** the neuron decides whether it is "normal" (+1) or "flipping" (−1) from the running average of `target × root output`. Once its error falls below 0.15, it keeps deciding for 300 more steps, then **locks for life**. This mirrors Dale's principle: a real neuron's excitatory or inhibitory identity is fixed after development.

---

## What each mechanism contributed

| mechanism | effect |
|---|---|
| bump parents + centered gain | XOR: 0 → ~100 / 100 |
| homeostasis (wake all branches together) | removed dead-neuron failures |
| growth (sprouting) | memory ratio 2.8× → ~10× on the first test pair |
| polarity + critical period | median memory ratio over random pairs 4.6× → 5.7× |

## Known limitations

- **Polarity is a guess.** During task A the neuron can't know which sign task B will need. When both tasks have targets of the same sign, the guess can be wrong (worst case still ≈ 0.74× better than the MLP). Biology solves this with pairs of excitatory and inhibitory neurons, which is the multi-neuron stage.
- The memory benchmark uses **one input** and **two tasks**. Larger tests (more inputs, more tasks) are next.
- Inputs are assigned by index for now. Real positions in 3D are next.

---

## Tests in the notebook

| test | what it checks |
|---|---|
| sine wave | the learning rule learns and stays stable |
| XOR (100 seeds) | non-linear computation inside one neuron |
| memory A → B → A | task A: x in [0.1, 0.5], task B: x in [−0.5, −0.1], each with its own constant target; train A, then B, then retest A |

Baselines: a flat single neuron (`STO`) and a 2-layer MLP trained with backprop (`BENCHMARK`).

## How to run

Open `BTHN_y7.ipynb` and run all cells. Only NumPy is needed.
Every test loops over seeds (`np.random.default_rng(seed)`), so the results are reproducible.

## References / inspiration

- Gidon et al. (2020), *Dendritic action potentials and computation in human layer 2/3 cortical neurons*, Science: XOR in single human neurons.
- Cichon & Gan (2015), *Nature*: different tasks are stored on different dendritic branches.
- Dale's principle: a neuron's excitatory or inhibitory identity is fixed.
- Turrigiano: homeostatic plasticity / synaptic scaling.

## License

Hippocratic License HL3-MIL-SV:

[![Hippocratic License HL3-MIL-SV](https://img.shields.io/static/v1?label=Hippocratic%20License&message=HL3-MIL-SV&labelColor=5e2751&color=bc8c3d)](https://firstdonoharm.dev/version/3/0/mil-sv.html)
