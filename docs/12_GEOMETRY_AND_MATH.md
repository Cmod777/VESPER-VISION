# 12 — Geometry and Math

## Overview

This module documents the mathematical, geometric, and probabilistic apparatus
used by Vesper Vision for video-space normalization, zone tolerance computation,
heatmap compression, occupancy scoring, and state fusion.

The objective is not to provide a heavy camera-calibration pipeline, but to
describe a deterministic and explainable geometric layer suitable for contextual
environmental perception.

---

## 1. Geometric Transformation and Zone Tolerance

To reduce the practical impact of perspective distortion without introducing
expensive camera calibration procedures, Vesper Vision may apply deterministic
geometry based on polygonal zone definitions and parallel-edge offsetting.

Each environmental zone is represented as a normalized polygon:

$$
P = \{(x_1, y_1), (x_2, y_2), \dots, (x_n, y_n)\}
\quad \text{with} \quad x_i, y_i \in [0, 1]
$$

Where each vertex is expressed in normalized video coordinates.

---

## Parallel Offset Method

For each segment \(S_i\) passing through:

$$
A(x_i, y_i)
\quad \text{and} \quad
B(x_{i+1}, y_{i+1})
$$

the corresponding implicit line is defined as:

$$
ax + by + c = 0
$$

with:

$$
a = y_{i+1} - y_i
$$

$$
b = x_i - x_{i+1}
$$

$$
c = x_{i+1}y_i - x_i y_{i+1}
$$

A translated parallel line can then be computed by shifting the constant term:

$$
c' = c \pm d \sqrt{a^2 + b^2}
$$

where \(d\) represents the normalized tolerance margin applied around the zone.

The sign of the offset depends on the polygon winding direction and on whether
the tolerance region is intended to expand or contract the original polygon.

---

## Vertex Reconstruction

The translated polygon vertices are obtained by intersecting consecutive
offset lines.

Given the system:

$$
\begin{cases}
a_1x + b_1y = -c'_1 \\
a_2x + b_2y = -c'_2
\end{cases}
$$

the determinant is:

$$
D = a_1b_2 - a_2b_1
$$

and the reconstructed vertex is:

$$
x = \frac{b_1c'_2 - b_2c'_1}{D}
$$

$$
y = \frac{c'_1a_2 - c'_2a_1}{D}
$$

If \(D\) approaches zero, the two translated lines are nearly parallel and the
intersection should be treated as geometrically unstable.

---

## Boundary Clamping

Because normalized coordinates must remain inside the video frame, each
computed vertex \((x', y')\) is projected back into the unit square:

$$
x_{\text{final}} = \max(0, \min(1, x'))
$$

$$
y_{\text{final}} = \max(0, \min(1, y'))
$$

This keeps all generated tolerance polygons bounded within the valid video
coordinate space.

---

## 2. Logarithmic Heatmap Compression

Footstep and presence heatmaps may suffer from stationary sampling saturation.

If a person remains in the same area for a long time, raw linear accumulation
can dominate the visualization and hide smaller movements or short transitions.

To reduce this effect, Vesper Vision may apply nonlinear logarithmic compression
using a `log1p`-style transformation:

$$
I_{\text{comp}}(x, y) =
\ln \left(1 + \alpha I_{\text{raw}}(x, y)\right)
$$

Where:

- \(I_{\text{raw}}(x, y)\) is the cumulative raw intensity at a given location.
- \(\alpha\) is the sampling intensity scale factor.
- \(I_{\text{comp}}(x, y)\) is the compressed heatmap intensity.

The addition of \(1\) ensures that when no event is present:

$$
I_{\text{raw}}(x, y) = 0
$$

the compressed output remains exactly:

$$
I_{\text{comp}}(x, y) = 0
$$

This keeps inactive background regions clean while reducing the dominance of
high-frequency stationary samples.

---

## 3. Occupancy Score and Multimodal Fusion

The final occupancy state of a critical region, such as a bed zone, should not
depend on a single perception signal.

Instead, Vesper Vision may combine multiple contextual indicators into a
weighted state-fusion score:

$$
S_{\text{bed}} =
w_{\text{vlm}} P(\text{occ}_{\text{vlm}})
+
w_{\text{pose}} \sum_{i=1}^{m} K_i
+
w_{\text{obj}} A_j
$$

Subject to the normalization constraint:

$$
\sum_k w_k = 1
$$

Where:

- \(P(\text{occ}_{\text{vlm}})\) is the visual-language occupancy confidence
  estimated on the target zone or crop.
- \(K_i\) represents the contribution of inferred human keypoints or skeletal
  evidence.
- \(A_j\) represents an object or animal activation factor when relevant.
- \(w_k\) represents the normalized contribution weight of each signal family.

This formulation is intended as an explainable fusion model rather than an
opaque black-box classifier.

---

## 4. Temporal Hysteresis and Debouncing

To avoid rapid state oscillations, Vesper Vision may apply temporal hysteresis
between occupied and free states.

A region transitions from `free` to `occupied` only when the occupancy score
exceeds a high threshold \(T_{\text{high}}\).

The return from `occupied` to `free` requires both a lower threshold
\(T_{\text{low}}\) and a temporal delay condition.

$$
\text{State}_{t} =
\begin{cases}
1 & \text{if } S_{\text{bed}} > T_{\text{high}} \\
0 & \text{if } S_{\text{bed}} < T_{\text{low}} \land \Delta t > T_{\text{delay}} \\
\text{State}_{t-1} & \text{otherwise}
\end{cases}
$$

Where:

- \(1\) represents the occupied state.
- \(0\) represents the free state.
- \(S_{\text{bed}}\) is the fused occupancy score.
- \(T_{\text{high}}\) is the activation threshold.
- \(T_{\text{low}}\) is the release threshold.
- \(\Delta t\) is the elapsed time under the release condition.
- \(T_{\text{delay}}\) is the minimum release delay.

This prevents short visual occlusions, pose changes, or temporary confidence
drops from immediately clearing the occupancy state.

---

## 5. Design Constraints

The geometry and fusion layer prioritizes:

- normalized coordinate systems
- deterministic polygon operations
- bounded tolerance expansion
- explainable score composition
- temporal state stability
- probabilistic occupancy interpretation
- runtime-safe state transitions

The mathematical layer is intentionally lightweight and designed for practical
environmental perception rather than full photogrammetric reconstruction.

---

## 6. Limitations

The described methods remain approximate.

Accuracy may be affected by:

- camera perspective
- lens distortion
- unstable polygon definitions
- incorrect zone winding
- partial occlusions
- ambiguous visual evidence
- low-confidence keypoint estimation
- environmental lighting variability
- overlapping occupancy regions

The geometry and fusion model should therefore be considered a bounded,
explainable approximation layer rather than a deterministic source of
environmental truth.
