## 1. System Concept and Scope

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
3. why the image may show evidence of degradation;
4. which bounded probe adjustment is most likely to improve it;
5. whether the estimate is reliable enough to act upon.

The controller then assigns responsibilities across motion directions and supervisory logic:

* **Normal to the chest:** maintain safe acoustic contact through force control.
* **Along the chest and in probe orientation:** refine the view through image-guided motion.
* **Safety supervisor:** determine whether either controller is allowed to act.

## 2. Runtime Inputs to the Perception Model

The perception model's only runtime sensory input is the live 2D B-mode ultrasound image stream, represented at update $t$ as $\mathcal{I}_t$. The model also receives the operator-selected target view $v_{\mathrm{target}}$ as a task condition:

$$v_{\mathrm{target}} \in \{\mathrm{PLAX}, \mathrm{PSAX}, \mathrm{A4C}\}.$$

At update $t$, the model receives either the most recent frame or a short fixed window of recent frames:

$$\mathcal{I}_t = (B_{t-K+1}, \ldots, B_t), \qquad K \ge 1,$$

where $K$ is the number of frames: $K=1$ is a single frame, and $K>1$ is a short fixed window. The implementation chooses $K$. Each frame $B_i$ is a grayscale $H \times W$ matrix of pixel values, so the stacked input has size $H \times W \times K$.

Frames are preprocessed the same way as in training, and acquisition times are recorded so output latency can be measured.

## 3. Image-Based Perception Model

> Placeholder equation: $$\mathbf{o}_t = f(\mathcal{I}_t, v_{\mathrm{target}})$$

...

## 4. Structured Perception Outputs

The perception system outputs a structured packet:

$$\mathbf{o}_t = [\mathbf{p}^{\mathrm{view}}_t,\; \mathbf{a}^{(v_{\mathrm{target}})}_t,\; \mathbf{e}^{\mathrm{deg}}_t,\; \mathbf{d}^{(v_{\mathrm{target}})}_t,\; \mathbf{u}_t].$$

### A. View Classification Probabilities

At each update, the model estimates which standard view is shown in $\mathcal{I}_t$. The output is a probability for each view:

$$\mathbf{p}^{\mathrm{view}}_t = [\,p_{\mathrm{PLAX}},\; p_{\mathrm{PSAX}},\; p_{\mathrm{A4C}},\; p_{\mathrm{other}}\,].$$

where each entry is the probability that $\mathcal{I}_t$ shows that view. Every entry is between 0 and 1, and the four entries sum to 1:

$$p_{\mathrm{PLAX}} + p_{\mathrm{PSAX}} + p_{\mathrm{A4C}} + p_{\mathrm{other}} = 1.$$

### B. Target-View Adequacy Scores

The adequacy output estimates how well $\mathcal{I}_t$ satisfies the adequacy criteria for the operator-selected target view $v_{\mathrm{target}}$. It reports three component scores for $v_{\mathrm{target}}$, rather than a separate set of scores for each possible target view:

$$\mathbf{a}^{(v_{\mathrm{target}})}_t = [\,a_{\mathrm{visibility}},\; a_{\mathrm{plane}},\; a_{\mathrm{geometry}}\,].$$

Each component is scored separately from 0 to 1; together, the components can sum to any value from 0 to 3:

$$0 \le a_{\mathrm{visibility}} + a_{\mathrm{plane}} + a_{\mathrm{geometry}} \le 3.$$

The three components are:

- **Anatomic visibility:** whether the structures required for the view are sufficiently visible.
- **Plane and level correctness:** whether the image is at the intended plane and level.
- **Geometric fidelity:** whether the anatomy has the expected shape and proportions.

These components report which aspect of the target view is weak, not why the image is poor or how the probe should move.

### C. Image Degradation Evidence Scores

The image-degradation output reports evidence in $\mathcal{I}_t$ that acoustic conditions may be impairing acquisition. It is view-agnostic: these degradation patterns do not depend on the selected target view $v_{\mathrm{target}}$. The output has three components:

$$\mathbf{e}^{\mathrm{deg}}_t = [\,e_{\mathrm{coupling}},\; e_{\mathrm{shadow}},\; e_{\mathrm{penetration}}\,].$$

Each component scores the strength of evidence for the corresponding degradation pattern, from 0 to 1. Because more than one degradation can be present at once, the components are scored separately and need not sum to 1.

The three components are:

- **Coupling impairment:** broad or near-surface signal loss consistent with poor acoustic coupling.
- **Acoustic shadowing:** localized or wedge-shaped signal loss consistent with shadowing.
- **Insufficient penetration:** depth-dependent signal loss consistent with insufficient penetration.

These components report image-based evidence only; confirming a physical cause requires additional signals.

### D. Probe Adjustment Direction Scores

The directional-correction output evaluates a fixed set of probe adjustments for local refinement using $\mathcal{I}_t$. Each score indicates how strongly $\mathcal{I}_t$ supports the corresponding adjustment as likely to improve the image toward the selected target view $v_{\mathrm{target}}$.

The adjustments are defined in a probe-fixed frame: $x_{\mathrm{p}}$ lies in the imaging plane, $y_{\mathrm{p}}$ lies perpendicular to it within the probe face, and $z_{\mathrm{p}}$ is normal to the probe face. The output scores translations along $x_{\mathrm{p}}$ and $y_{\mathrm{p}}$ and rotations about $x_{\mathrm{p}}$, $y_{\mathrm{p}}$, and $z_{\mathrm{p}}$; translation along $z_{\mathrm{p}}$ is assigned to force control. Each motion component is scored in its positive and negative directions, yielding ten scores grouped into five opposed pairs:

$$\mathbf{d}^{(v_{\mathrm{target}})}_t = [\,(d_{\mathrm{T},x_{\mathrm{p}}}^{+},\, d_{\mathrm{T},x_{\mathrm{p}}}^{-}),\; (d_{\mathrm{T},y_{\mathrm{p}}}^{+},\, d_{\mathrm{T},y_{\mathrm{p}}}^{-}),\; (d_{\mathrm{R},x_{\mathrm{p}}}^{+},\, d_{\mathrm{R},x_{\mathrm{p}}}^{-}),\; (d_{\mathrm{R},y_{\mathrm{p}}}^{+},\, d_{\mathrm{R},y_{\mathrm{p}}}^{-}),\; (d_{\mathrm{R},z_{\mathrm{p}}}^{+},\, d_{\mathrm{R},z_{\mathrm{p}}}^{-})\,].$$

Each score ranges from 0 to 1. The scores are not normalized, so one or more adjustments may receive support. If all adjustment scores are low, no directional correction is supported.

The five motion components are:

- **Translation along $x_{\mathrm{p}}$:** sliding the probe along $x_{\mathrm{p}}$.
- **Translation along $y_{\mathrm{p}}$:** sliding the probe along $y_{\mathrm{p}}$.
- **Rotation about $x_{\mathrm{p}}$:** tilting the probe about $x_{\mathrm{p}}$.
- **Rotation about $y_{\mathrm{p}}$:** tilting the probe about $y_{\mathrm{p}}$.
- **Rotation about $z_{\mathrm{p}}$:** rotating the probe about $z_{\mathrm{p}}$.

The scores indicate supported adjustment directions, not executable movement commands.

### E. Image-Based Perception Uncertainty

The uncertainty output reports disagreement among independently trained models for each output group:

$$\mathbf{u}_t = [\,u_{\mathrm{view}},\; u_{\mathrm{adequacy}}^{(v_{\mathrm{target}})},\; u_{\mathrm{degradation}},\; u_{\mathrm{direction}}^{(v_{\mathrm{target}})}\,].$$

Only the adequacy and directional uncertainty components are target-conditioned by $v_{\mathrm{target}}$.

The perception system uses several models with the same architecture, trained separately with different initializations. At runtime, each model produces its own estimate from the same $\mathcal{I}_t$. For any one output component, let the resulting estimates be $s_t^{(1)},\ldots,s_t^{(N)}$. Their average is reported as the component value:

$$s_t =
\frac{s_t^{(1)}+\cdots+s_t^{(N)}}{N}.$$

The spread of these estimates defines the component uncertainty. Because each estimate lies between 0 and 1, twice the population standard deviation lies between 0 and 1:

$$u_{s,t} = 2\,\operatorname{SD} \left(s_t^{(1)},\ldots,s_t^{(N)} \right).$$

For each output group, the reported uncertainty is the largest component uncertainty in that group.

- **Low uncertainty:** the models produce similar estimates for that output group.
- **High uncertainty:** the models produce different estimates for at least one component in that output group.

Close agreement does not guarantee that an output is correct. This output describes image-perception uncertainty only; physical action validity is determined downstream.

## #. Later Sections...

...
