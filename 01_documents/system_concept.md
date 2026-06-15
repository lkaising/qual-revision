## 1. System Overview

The operator places the probe near the selected acoustic window and chooses the target view. The robot then performs bounded local refinement and maintenance.

At every update, the system combines:

* the current ultrasound image;
* the robot’s probe pose;
* force/torque measurements;
* the estimated chest-wall normal;
* recent robot commands and image responses.

It produces a structured estimate of:

1. what view is visible;
2. whether it is diagnostically adequate;
3. why the image may be inadequate;
4. which bounded probe movement is most likely to improve it;
5. whether the estimate is reliable enough to act upon.

The controller then assigns different responsibilities to different physical directions:

* **Normal to the chest:** maintain safe acoustic contact through force control.
* **Along the chest and in probe orientation:** refine the view through image-guided motion.
* **Safety supervisor:** determine whether either controller is allowed to act.

## 2. Runtime Perception Inputs

The perception model's only runtime sensory input is $I_t$, a live 2D B-mode ultrasound image stream. The model is also given the operator-selected target view $v^*$ as a task condition:

$$v^* = \mathrm{PLAX}, \ \mathrm{PSAX}, \ \text{or} \ \mathrm{A4C}.$$

At update $t$, the model receives the most recent frame, or a short window of recent frames:

$$I_t = (B_{t-K+1}, \ldots, B_t), \qquad K \ge 1,$$

where $K$ is the number of frames: $K=1$ is a single frame and $K>1$ a short fixed window. The choice of $K$ is left to implementation. Each frame $B_t$ is a grayscale image, an $H \times W$ matrix of pixel values, so the stacked input has size $H \times W \times K$.

Each frame is preprocessed the same way as in training, and its acquisition time is recorded so output latency can be measured.

## 3. Runtime Perception Model

> Placeholder equation: $$\mathbf{y}_t = f(I_t, v^*)$$

...

## 4. Runtime Perception Output

The image-processing system outputs a structured packet:

$$\mathbf{y}_t = [\mathbf{p}^{view}_t,\; \mathbf{a}^{(v^*)}_t,\; \mathbf{c}_t,\; \mathbf{d}^{(v^*)}_t,\; u_t]$$

### A. View-identity probabilities

At each update the model estimates which standard view $I_t$ shows. The output is a probability for each view:

$$\mathbf{p}^{view}_t = [\,P(\mathrm{PLAX}),\; P(\mathrm{PSAX}),\; P(\mathrm{A4C}),\; P(\mathrm{other})\,]$$

where each entry is the probability that $I_t$ is that view. Each is between 0 and 1, and the four add up to 1:

$$P(\mathrm{PLAX}) + P(\mathrm{PSAX}) + P(\mathrm{A4C}) + P(\mathrm{other}) = 1.$$

### B. Adequacy components

The adequacy output estimates how well $I_t$ satisfies the criteria for an acceptable instance of the operator-selected target view $v^*$. It contains three component scores for $v^*$, rather than a separate set of scores for each possible target view:

$$\mathbf{a}^{(v^*)}_t = [\,a_{\text{visibility}},\; a_{\text{plane}},\; a_{\text{geometry}}\,].$$

Each component ranges from 0 to 1 and is scored separately; the components need not sum to 1:

$$0 \le a_{\text{visibility}} + a_{\text{plane}} + a_{\text{geometry}} \le 3.$$

The three components are:

- **Anatomic visibility:** whether the structures required for the view are sufficiently visible.
- **Plane and level correctness:** whether the image is at the intended plane and level.
- **Geometric fidelity:** whether the anatomy has the expected shape and proportions.

These components report which aspect of the target view is weak, not why the image is poor or how the probe should move.

### C. Image-degradation components

The image-degradation output reports evidence in $I_t$ that an acoustic condition may be impairing acquisition. It is view-agnostic: the conditions it reports do not depend on the selected target view $v^*$. The output has three components:

$$\mathbf{c}_t = [\,c_{\text{coupling}},\; c_{\text{shadow}},\; c_{\text{penetration}}\,].$$

Each component grades how strongly $I_t$ shows evidence for the corresponding degradation pattern, from 0 to 1. The components are scored separately, and because more than one degradation can be present at once, they need not sum to 1.

The three components are:

- **Coupling impairment:** broad or near-surface signal loss consistent with poor acoustic coupling.
- **Acoustic shadowing:** localized or wedge-shaped signal loss consistent with shadowing.
- **Insufficient penetration:** depth-dependent signal loss consistent with insufficient penetration.

These components report image-based evidence only; confirming the physical cause requires additional signals.

### D. Directional correction scores

...

### E. Uncertainty or action-validity output

...

## #. Later Sections...

...
