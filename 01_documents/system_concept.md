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

The perception model's only runtime sensory input is $I_t$, the live 2D B-mode ultrasound image stream. The model is also given the operator-selected target view $v^*$ as a task condition:

$$v^* = \mathrm{PLAX}, \ \mathrm{PSAX}, \ \text{or} \ \mathrm{A4C}.$$

At update $t$, the model receives the most recent frame, or a short window of recent frames:

$$I_t = (B_{t-K+1}, \ldots, B_t), \qquad K \ge 1,$$

where $K$ is the number of frames: $K=1$ is a single frame and $K>1$ a short fixed window. The choice of $K$ is left to implementation. Each frame $B_t$ is a grayscale image, an $H \times W$ matrix of pixel values, so the stacked input has size $H \times W \times K$.

Each frame is preprocessed the same way as in training, and its acquisition time is recorded so output latency can be measured.

## 3. Runtime Perception Model

> Placeholder equation: $$\mathbf{y}_t = f(I_t)$$

...

## 4. Runtime Perception Output

The image-processing system outputs a structured packet:

$$\mathbf{y}_t =
[\mathbf{p}^{view}_t,\; \mathbf{a}_t,\; \mathbf{c}_t,\; \mathbf{d}_t,\; u_t]$$

### A. View-identity probabilities

At each update the model estimates which standard view the input $I_t$ shows. The output is a probability for each view:

$$\mathbf{p}^{view}_t =
[\,P(\mathrm{PLAX}),\; P(\mathrm{PSAX}),\; P(\mathrm{A4C}),\; P(\mathrm{other})\,]$$

where each entry is the probability that $I_t$ is that view. Each is between 0 and 1, and the four add up to 1:

$$P(\mathrm{PLAX}) + P(\mathrm{PSAX}) + P(\mathrm{A4C}) + P(\mathrm{other}) = 1.$$

### B. Adequacy components

The adequacy output estimates how well the input $I_t$ meets the criteria for an acceptable
instance of the operator-selected target view. The output is a vector of component
scores:

$$\mathbf{a}_t = [\,a_{\text{visibility}},\; a_{\text{plane}},\; a_{\text{geometry}}\,]$$

Each component is between 0 and 1. The components are scored separately and need not sum
to 1; their total can fall anywhere in

$$0 \le a_{\text{visibility}} + a_{\text{plane}} + a_{\text{geometry}} \le 3.$$

The three components are:

- **Anatomic visibility:** whether the structures required for the view are sufficiently visible.
- **Plane and level correctness:** whether the image is at the intended plane and level.
- **Geometric fidelity:** whether the anatomy has the expected shape and proportions.

---

*Defered until $v^*$ model decision point is addressed in prior sections.*

The same three components are reported for every target view, but the criteria
behind each component are defined per view, so visibility, plane correctness, and
geometric fidelity mean view-specific things for PLAX, mid-papillary PSAX, and A4C.

These components report which aspect of the target view is currently weak. They do
not identify the physical cause of a poor image or specify how the probe should
move; those are separate outputs.

A scalar summary $Q(\mathbf{a}_t)$ may be derived from the components for plotting or
for comparing conditions, but the choice of aggregation is a separate decision and
is not fixed here. The individual components remain the primary output.

### C. Image-degradation state

...

### D. Directional correction scores

...

### E. Uncertainty or action-validity output

...

## #. Later Sections...

...
