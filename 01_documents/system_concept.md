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

## 2. Runtime Perception Input

The perception model’s only runtime input is the live 2D B-mode ultrasound image stream. At update $t$, the model receives the most recent frame, or a short window of recent frames:

$$I_t = (B_{t-K+1}, \ldots, B_t), \qquad K \ge 1,$$

where $K$ is the number of frames: $K=1$ is a single frame and $K>1$ a short fixed window. The choice of $K$ is left to implementation. Each frame $B_t$ is a grayscale image, an $H \times W$ matrix of pixel values, so the stacked input has size

$$H \times W \times K.$$

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

...

### C. Image-degradation state

...

### D. Directional correction scores

...

### E. Uncertainty or action-validity output

...

## #. Later Sections...

...
