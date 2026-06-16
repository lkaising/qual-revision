## 1. System Overview

The operator places the probe near the selected acoustic window and selects the target view. The robot then performs bounded local refinement and maintenance.

At each update, the system combines:

* the current ultrasound image;
* the robot’s probe pose;
* force/torque measurements;
* the estimated chest-wall normal;
* recent robot commands and image responses.

It produces a structured estimate of:

1. which view is visible;
2. whether it is diagnostically adequate;
3. why the image may be inadequate;
4. which bounded probe adjustment is most likely to improve it;
5. whether the estimate is reliable enough to act upon.

The controller then assigns responsibilities across motion directions and supervisory logic:

* **Normal to the chest:** maintain safe acoustic contact through force control.
* **Along the chest and in probe orientation:** refine the view through image-guided motion.
* **Safety supervisor:** determine whether either controller is allowed to act.

## 2. Runtime Perception Inputs

The perception model's only runtime sensory input is the live 2D B-mode ultrasound image stream, represented at update $t$ as $I_t$. The model also receives the operator-selected target view $v^*$ as a task condition:

$$v^* \in \{\mathrm{PLAX}, \mathrm{PSAX}, \mathrm{A4C}\}.$$

At update $t$, the model receives either the most recent frame or a short fixed window of recent frames:

$$I_t = (B_{t-K+1}, \ldots, B_t), \qquad K \ge 1,$$

where $K$ is the number of frames: $K=1$ is a single frame, and $K>1$ is a short fixed window. The implementation chooses $K$. Each frame $B_i$ is a grayscale $H \times W$ matrix of pixel values, so the stacked input has size $H \times W \times K$.

Frames are preprocessed the same way as in training, and acquisition times are recorded so output latency can be measured.

## 3. Runtime Perception Model

> Placeholder equation: $$\mathbf{y}_t = f(I_t, v^*)$$

...

## 4. Runtime Perception Output

The perception system outputs a structured packet:

$$\mathbf{y}_t = [\mathbf{p}^{view}_t,\; \mathbf{a}^{(v^*)}_t,\; \mathbf{c}_t,\; \mathbf{d}^{(v^*)}_t,\; \mathbf{u}_t].$$

### A. View-identity probabilities

At each update, the model estimates which standard view is shown in $I_t$. The output is a probability for each view:

$$\mathbf{p}^{view}_t = [\,P(\mathrm{PLAX}),\; P(\mathrm{PSAX}),\; P(\mathrm{A4C}),\; P(\mathrm{other})\,].$$

where each entry is the probability that $I_t$ shows that view. Every entry is between 0 and 1, and the four entries sum to 1:

$$P(\mathrm{PLAX}) + P(\mathrm{PSAX}) + P(\mathrm{A4C}) + P(\mathrm{other}) = 1.$$

### B. Adequacy components

The adequacy output estimates how well $I_t$ satisfies the adequacy criteria for the operator-selected target view $v^*$. It reports three component scores for $v^*$, rather than a separate set of scores for each possible target view:

$$\mathbf{a}^{(v^*)}_t = [\,a_{\text{visibility}},\; a_{\text{plane}},\; a_{\text{geometry}}\,].$$

Each component is scored separately from 0 to 1; together, the components can sum to any value from 0 to 3:

$$0 \le a_{\text{visibility}} + a_{\text{plane}} + a_{\text{geometry}} \le 3.$$

The three components are:

- **Anatomic visibility:** whether the structures required for the view are sufficiently visible.
- **Plane and level correctness:** whether the image is at the intended plane and level.
- **Geometric fidelity:** whether the anatomy has the expected shape and proportions.

These components report which aspect of the target view is weak, not why the image is poor or how the probe should move.

### C. Image-degradation components

The image-degradation output reports evidence in $I_t$ that acoustic conditions may be impairing acquisition. It is view-agnostic: these degradation patterns do not depend on the selected target view $v^*$. The output has three components:

$$\mathbf{c}_t = [\,c_{\text{coupling}},\; c_{\text{shadow}},\; c_{\text{penetration}}\,].$$

Each component scores the strength of evidence for the corresponding degradation pattern, from 0 to 1. Because more than one degradation can be present at once, the components are scored separately and need not sum to 1.

The three components are:

- **Coupling impairment:** broad or near-surface signal loss consistent with poor acoustic coupling.
- **Acoustic shadowing:** localized or wedge-shaped signal loss consistent with shadowing.
- **Insufficient penetration:** depth-dependent signal loss consistent with insufficient penetration.

These components report image-based evidence only; confirming a physical cause requires additional signals.

### D. Directional correction scores

The directional-correction output evaluates a fixed set of probe adjustments for local refinement using $I_t$. Each score indicates how strongly $I_t$ supports the corresponding adjustment as likely to improve the image toward the selected target view $v^*$.

The adjustments are defined in a probe-fixed frame: $x_p$ lies in the imaging plane, $y_p$ lies perpendicular to it within the probe face, and $z_p$ is normal to the probe face. The output scores translations along $x_p$ and $y_p$ and rotations about $x_p$, $y_p$, and $z_p$; translation along $z_p$ is assigned to force control. Each motion component is scored in its positive and negative directions, yielding ten scores grouped into five opposed pairs:

$$\mathbf{d}^{(v^*)}_t = [\,(d_{T,x_p}^{+},\, d_{T,x_p}^{-}),\; (d_{T,y_p}^{+},\, d_{T,y_p}^{-}),\; (d_{R,x_p}^{+},\, d_{R,x_p}^{-}),\; (d_{R,y_p}^{+},\, d_{R,y_p}^{-}),\; (d_{R,z_p}^{+},\, d_{R,z_p}^{-})\,].$$

Each score ranges from 0 to 1. The scores are not normalized, so one or more adjustments may receive support. If all adjustment scores are low, no directional correction is supported.

The five motion components are:

- **Translation along $x_p$:** sliding the probe along $x_p$.
- **Translation along $y_p$:** sliding the probe along $y_p$.
- **Rotation about $x_p$:** tilting the probe about $x_p$.
- **Rotation about $y_p$:** tilting the probe about $y_p$.
- **Rotation about $z_p$:** rotating the probe about $z_p$.

The scores indicate supported adjustment directions, not executable movement commands.

### E. Image-perception uncertainty

The uncertainty output reports disagreement among independently trained models for each output group:

$$\mathbf{u}_t =
[\,u_{\mathrm{view},t},\;
u_{\mathrm{adequacy},t}^{(v^*)},\;
u_{\mathrm{degradation},t},\;
u_{\mathrm{direction},t}^{(v^*)}\,].$$

Only the adequacy and directional uncertainty components are target-conditioned by $v^*$.

The perception system uses several models with the same architecture, trained separately with different initializations. At runtime, each model produces its own estimate from the same $I_t$. For any one output component, let the resulting estimates be $x_t^{(1)},\ldots,x_t^{(N)}$. Their average is reported as the component value:

$$x_t =
\frac{x_t^{(1)}+\cdots+x_t^{(N)}}{N}.$$

The spread of these estimates defines the component uncertainty. Because each estimate lies between 0 and 1, twice the population standard deviation lies between 0 and 1:

$$u_{x,t} =
2\,\operatorname{SD}
\left(
x_t^{(1)},\ldots,x_t^{(N)}
\right).$$

For each output group, the reported uncertainty is the largest component uncertainty in that group.

- **Low uncertainty:** the models produce similar estimates for that output group.
- **High uncertainty:** the models produce different estimates for at least one component in that output group.

Close agreement does not guarantee that an output is correct. This output describes image-perception uncertainty only; physical action validity is determined downstream.

## #. Later Sections...

...
