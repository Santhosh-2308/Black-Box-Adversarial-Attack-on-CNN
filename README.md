# Problem 2 - Black-Box Adversarial Attack on a CNN

**Team:** Friday  
**Members:** Santhosh Kandagatla, Bhargav Reddymalla  
**Affiliation:** Dept. of CSE (Cybersecurity), Geethanjali College of Engineering and Technology, Hyderabad, India  
**Repository:** https://github.com/Santhosh-2308/Black-Box-Adversarial-Attack-on-CNN

We generate one adversarial image for each of the 1000 clean images using only the **hard labels** returned by the pretrained CNN (`model(x)`). No gradients, parameters, architecture, intermediate outputs or class scores are used.

## Results (measured on the saved PNG files)

| Metric | Value |
|---|---|
| Attack success rate (prediction changed) | **100%** (1000/1000) |
| Misclassified against the true label | 1000/1000 |
| Mean L2 noise, pixel scale [0,1] | **0.2073** |
| Mean L2 noise, normalized-input scale | 0.8332 |
| Median / 90th pct / 99th pct L2 | 0.192 / 0.376 / 0.565 |
| Max / min L2 per image | 0.9112 / 0.0039 (one pixel value changed by one level) |
| Mean L-inf (0-255 scale) | 4.95 (median 5, 90th pct 9, 99th pct 14, max 20) |
| Images with L-inf of 8 or less / 10 or less | 85.1% / 93.1% |
| Mean PSNR (worst image) | 50.53 dB (35.68 dB) |
| Mean SSIM (worst image) | 0.9980 (0.9830) |
| Mean absolute change per pixel value | 0.645 levels out of 255 |
| Share of the 3072 values changed | about 46% on average |
| Model queries per image | 18,455 (about 18.5 M in total) |

SSIM was computed per channel with a standard 11x11 Gaussian window (sigma 1.5) on the [0,1] images; other implementations may differ in the last digits. The organizers' exact imperceptibility metric is not specified, so we report several measures.

Success was checked on the PNGs reloaded from disk, with the same `transform` as the setup cell, at batch sizes 1000, 250, 32 and 1 (all 1000/1000).

## Repository contents

```
README.md                    this file
report.pdf                   short report describing the method (IEEE two-column format)
BlackBox.ipynb               complete notebook with the final attack parameters
BlackBox.pdf                 printout of the full Colab run of that notebook
adv_images/                  1000 adversarial PNGs, named <class>_<index>_adv.png
```

## How to run

1. Open `BlackBox.ipynb` in Google Colab with a GPU runtime and upload `p2_data.zip` (set `zip_path` if needed).
2. Run the cells top to bottom. The setup cells are the organizers' originals and are unchanged.
3. The attack takes about 105 minutes for all 1000 images on a Colab GPU. For a quick test, set `SUBSET = 20` in the attack cell (set it back to `None` for the full run).
4. The save cell writes `./adv_images/` and prints the verification metrics above.

The attack is seeded. Two independent full runs produced byte-identical images.

## Method (short)

The attack is a label-only, minimum-L2 attack in the style of HopSkipJump, run in pixel space and checked on the 8-bit PNG grid.

1. **Starting points.** For each image we take the 10 nearest clean images (by pixel L2) that belong to other classes. Each one that the model labels differently is pulled toward the target by binary search until it sits just inside the adversarial region. The closest result is kept.
2. **Refinement (40 iterations).** At the current boundary point we send random probes (50 x sqrt(t) of them at iteration t, capped at 800), read only their labels, and estimate the boundary normal. We take a geometric step along it, then binary-search back toward the clean image. The noise shrinks each iteration.
3. **PNG-safe finalization.** The refined point is rounded to 8 bits. If rounding removes the adversarial effect, the perturbation is scaled up until it survives. A final 10-step binary search on the 8-bit grid then gives the smallest noise that still works as a saved PNG.
4. **Two restarts.** The attack is run twice (the second from different starting candidates) and the lower-noise valid result is kept per image. The second restart lowered the mean L2 from 0.2190 to 0.2073.

### Final parameters

| Parameter | Value |
|---|---|
| Restarts | 2 |
| Iterations per restart | 40 |
| Starting candidates per image | 10 (from the nearest 30) |
| Probes per iteration | 50 x sqrt(t), max 800 |
| Binary-search steps | 12 (10 in the final 8-bit stage) |
| Probe radius factor | 1e-3 |
| Seeds | 0 and 1 (one per restart) |

## Compliance notes

- **Black box only.** The notebook touches the model only through `model(...)` and uses only the labels it returns. It never reads weights, layers, probabilities or intermediate outputs.
- **Provided notebook.** The "do not edit" cells are untouched; only the attack and save cells are filled in.
- **Starting images.** The starting points are other clean images from the provided set, chosen using only pixel distances and the model's output labels. The rules do not mention this; we state it here for transparency.
- **Success definition.** An image counts as successful if its predicted label differs from both the true label and the clean prediction. All 1000 clean images are classified correctly, so the two definitions coincide.

## Design decisions

- **Pixel pruning was tried and rejected.** A post-processing step that moved changed pixels back toward the clean image lowered the mean L2 by about 4.4% (0.2073 to 0.1981). But it pushed images right onto the decision boundary: about half of the pruned images flipped back under tiny input noise (sigma = 1e-4), versus 14 of 1000 for the final set, and one flipped under batched CPU evaluation. Because attack success carries half of the score, we kept the more robust set.
- **Robustness checks.** Beyond the batch-size checks in the notebook, we tested the final images on CPU in float32 and float64 and under small input noise. They fooled the model in every float32/float64 setting. Under a deliberately harsh half-precision test, 4 of 200 sampled images flipped back. These extra checks were run offline as diagnostics on a copy of the model; they are not part of the notebook or the attack.
