# 12 — Geometry and Math

## Overview

This module documents the mathematical, geometric, probabilistic, and temporal
reasoning layers used by Vesper Vision for environmental interpretation,
occupancy supervision, spatial normalization, tolerance generation, heatmap
compression, and multimodal state fusion.

The architecture intentionally prioritizes deterministic and explainable
mathematical models over opaque black-box geometric reconstruction systems.

The described methods are designed for lightweight environmental cognition and
continuous runtime supervision rather than full photogrammetric reconstruction.

---

## 1. Geometric Transformation and Zone Tolerance

To reduce the practical impact of lens perspective distortion without requiring
heavy camera calibration pipelines, Vesper Vision applies deterministic polygon
geometry and parallel-edge offset reconstruction.

Each supervised environmental region is represented as a normalized polygon:

$$
P =
\{
(x_1, y_1),
(x_2, y_2),
\dots,
(x_n, y_n)
\}
\quad \text{with} \quad
x_i, y_i \in [0,1]
$$

Where each vertex is expressed in normalized video-space coordinates.

---

## Parallel Offset Reconstruction

For each polygon segment \(S_i\) passing through:

$$
A(x_i, y_i)
\quad \text{and} \quad
B(x_{i+1}, y_{i+1})
$$

the corresponding implicit line representation is:

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

To generate tolerance regions around the original polygon, the line is
translated by a normalized offset margin \(d\):

$$
c' = c \pm d \sqrt{a^2 + b^2}
$$

Where:

- \(d\) is the normalized tolerance expansion factor.
- The sign of the offset depends on polygon winding orientation and desired
  expansion direction.

This produces a translated set of parallel boundary lines used to reconstruct
the tolerance polygon.

---

## Vertex Reconstruction

The translated polygon vertices are reconstructed analytically through the
intersection of consecutive translated lines.

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

The reconstructed coordinates are then obtained through Cramer-style
deterministic reconstruction:

$$
x =
\frac{
b_1c'_2 - b_2c'_1
}{D}
$$

$$
y =
\frac{
a_1c'_2 - a_2c'_1
}{D}
$$

If:

$$
D \approx 0
$$

the translated lines become numerically unstable or nearly parallel and the
intersection should be treated as geometrically unreliable.

---

## Boundary Clamping

Because all environmental coordinates must remain inside the normalized video
domain, each reconstructed vertex:

$$
(x', y')
$$

is projected back into the unit square:

$$
x_{\text{final}} =
\max(0,\min(1,x'))
$$

$$
y_{\text{final}} =
\max(0,\min(1,y'))
$$

This guarantees that generated tolerance regions remain bounded within the
valid environmental frame.

---

## 2. Heatmap Compression and Nonlinear Normalization

Environmental trajectory heatmaps naturally suffer from stationary sampling
saturation.

If a tracked subject remains in the same spatial location for an extended
period of time, linear accumulation causes local intensity domination that may
hide short transitions, micro-movements, or low-frequency environmental events.

To reduce this effect, Vesper Vision applies nonlinear logarithmic compression
based on normalized `log1p` scaling.

Given a normalized raw environmental intensity:

$$
I_{\text{norm}}(x,y)
$$

the compressed intensity becomes:

$$
I_{\text{comp}}(x,y)
=
\frac{
\ln\left(
1 + \beta I_{\text{norm}}(x,y)
\right)
}{
\ln(1+\beta)
}
$$

Where:

- \(I_{\text{norm}}(x,y)\) is the normalized environmental accumulation map.
- \(\beta\) is the logarithmic compression scale factor.

In the current renderer implementation:

$$
\beta = 9.0
$$

This normalization guarantees:

$$
I_{\text{comp}}(x,y) \in [0,1]
$$

while compressing dominant stationary peaks and preserving weaker movement
patterns.

Additionally, when:

$$
I_{\text{norm}}(x,y)=0
$$

the compressed output remains exactly:

$$
I_{\text{comp}}(x,y)=0
$$

thereby maintaining a clean inactive environmental background.

---

## 3. Occupancy Score and Multimodal Fusion

Critical occupancy decisions should not depend on isolated perception signals.

Instead, Vesper Vision combines multiple contextual environmental indicators
through a deterministic weighted fusion layer.

The fused occupancy score is defined as:

$$
S_{\text{bed}}
=
w_{\text{vlm}}
P(\text{occ}_{\text{vlm}})
+
w_{\text{pose}}
\sum_{i=1}^{m} K_i
+
w_{\text{obj}}
A_j
$$

subject to the normalization constraint:

$$
\sum_k w_k = 1
$$

Where:

- \(P(\text{occ}_{\text{vlm}})\) represents the occupancy confidence extracted
  from visual-language environmental interpretation.
- \(K_i\) represents inferred skeletal or keypoint evidence.
- \(A_j\) represents contextual object or animal activation factors.
- \(w_k\) represents the normalized contribution weight of each modality.

This fusion model is intentionally explainable and deterministic rather than an
opaque black-box environmental classifier.

---

## 4. Temporal Hysteresis and Debouncing

To avoid rapid environmental state oscillations, Vesper Vision applies temporal
hysteresis between occupied and free states.

A supervised region transitions from `free` to `occupied` only when the fused
occupancy score exceeds the upper threshold:

$$
T_{\text{high}}
$$

The transition back to `free` requires:

- a lower release threshold
- temporal persistence under the release condition

The resulting state transition model becomes:

$$
\text{State}_t
=
\begin{cases}
1
&
\text{if }
S_{\text{bed}} > T_{\text{high}}
\\
\\
0
&
\text{if }
S_{\text{bed}} < T_{\text{low}}
\land
\Delta t > T_{\text{delay}}
\\
\\
\text{State}_{t-1}
&
\text{otherwise}
\end{cases}
$$

Where:

- \(1\) represents the occupied state.
- \(0\) represents the free state.
- \(S_{\text{bed}}\) is the fused occupancy score.
- \(T_{\text{high}}\) is the activation threshold.
- \(T_{\text{low}}\) is the release threshold.
- \(\Delta t\) is the elapsed release duration.
- \(T_{\text{delay}}\) is the minimum persistence delay before release.

This prevents temporary occlusions, pose changes, or transient confidence drops
from immediately collapsing the environmental occupancy state.

---

## Confidence Decay Dynamics

When direct visual evidence becomes unavailable, tracking confidence decays as
a function of the currently fused environmental state.

If occupancy is currently confirmed, a conservative decay coefficient is used:

$$
\gamma_{\text{occupied}} = 0.08
$$

When occupancy is not confirmed, a more aggressive decay coefficient is applied:

$$
\gamma_{\text{free}} = 0.25
$$

This asymmetric decay behavior helps preserve stable occupied states during
short-lived visibility interruptions while allowing free states to recover more
rapidly from stale environmental evidence.

---

## 5. Runtime Design Constraints

The geometry and mathematical supervision layer intentionally prioritizes:

- deterministic polygon operations
- normalized coordinate systems
- bounded tolerance expansion
- explainable score composition
- probabilistic environmental reasoning
- temporal continuity supervision
- runtime-safe state transitions
- lightweight geometric reconstruction

The architecture intentionally avoids computationally expensive photogrammetric
pipelines whenever deterministic approximations are operationally sufficient.

---

## 6. Limitations

The described methods remain approximate environmental models.

Accuracy may be affected by:

- camera perspective distortion
- lens deformation
- unstable polygon topology
- incorrect winding orientation
- environmental occlusions
- low-confidence skeletal estimation
- ambiguous environmental evidence
- overlapping occupancy regions
- unstable lighting conditions
- environmental noise
- incomplete telemetry continuity

The mathematical and geometric layer should therefore be interpreted as a
bounded and explainable approximation system rather than a deterministic source
of environmental truth.
