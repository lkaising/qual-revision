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

The directional output evaluates a fixed set of probe adjustments for local refinement from $I_t$. Each score indicates how strongly the corresponding adjustment is supported as improving the image toward the selected target view $v^*$.

The adjustments are defined in a probe-fixed frame: $x_p$ lies in the imaging plane, $y_p$ lies perpendicular to it within the probe face, and $z_p$ is normal to the probe face. The output scores translations along $x_p$ and $y_p$ and rotations about $x_p$, $y_p$, and $z_p$; translation along $z_p$ is assigned to force control. Each motion component is scored in its positive and negative directions, yielding ten scores grouped into five opposed pairs:

$$\mathbf{d}^{(v^*)}_t = [\,(d_{T,x_p}^{+},\, d_{T,x_p}^{-}),\; (d_{T,y_p}^{+},\, d_{T,y_p}^{-}),\; (d_{R,x_p}^{+},\, d_{R,x_p}^{-}),\; (d_{R,y_p}^{+},\, d_{R,y_p}^{-}),\; (d_{R,z_p}^{+},\, d_{R,z_p}^{-})\,]$$

Each score ranges from 0 to 1. The scores are not normalized, so one or more adjustments may receive support. If all adjustment scores are low, no directional correction is supported.

The five motion components are:

- **Translation along $x_p$:** sliding the probe along $x_p$.
- **Translation along $y_p$:** sliding the probe along $y_p$.
- **Rotation about $x_p$:** tilting the probe about $x_p$.
- **Rotation about $y_p$:** tilting the probe about $y_p$.
- **Rotation about $z_p$:** rotating the probe about $z_p$.

The scores indicate supported adjustment directions, not executable movements.

### E. Image-perception uncertainty

The uncertainty output reports model-to-model disagreement for each perception branch:

$$\mathbf{u}_t =
[\,u_{\mathrm{view},t},\;
u_{\mathrm{adequacy},t}^{(v^*)},\;
u_{\mathrm{degradation},t},\;
u_{\mathrm{direction},t}^{(v^*)}\,]$$

As in the corresponding outputs, only adequacy and direction depend on the selected target view $v^*$.

The perception system uses several separately trained copies of the same model. At runtime, each copy evaluates the same $I_t$ and produces its own estimate. For one output component, let those estimates be $x_t^{(1)},\ldots,x_t^{(N)}$. Their average is reported as the component value:

$$x_t =
\frac{x_t^{(1)}+\cdots+x_t^{(N)}}{N}.$$

The spread of those estimates defines the component uncertainty. Because each estimate lies between 0 and 1, the population standard deviation is at most 0.5, so

$$u_{x,t} =
2\,\operatorname{SD}
\left(
x_t^{(1)},\ldots,x_t^{(N)}
\right).$$

Each branch reports the highest uncertainty among its components.

- **Low uncertainty:** the models produce similar estimates.
- **High uncertainty:** the models disagree substantially about at least one component.

Close agreement does not by itself establish that an output is correct. This output describes image-perception uncertainty only; it does not determine whether robot movement is physically safe or permitted.

## #. Later Sections...

...
