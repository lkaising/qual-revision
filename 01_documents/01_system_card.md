## 1. System Concept, Scope, and Information Flow

The operator places the probe near the selected acoustic window, selects the target view, applies gel, initially orients the probe, authorizes and monitors robot assistance, and may pause or stop it at any time. The robot performs only bounded local refinement and maintenance; it does not fully autonomously scan or choose a new acoustic window.

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

## 2. Runtime Inputs to Image Perception

The image-perception model uses the live 2D brightness mode ultrasound image stream as its only runtime sensory input, represented at update $t$ as $\mathcal{I}_t$. The model also receives the operator-selected target view $v_{\mathrm{target}}$ as a task condition:

$$v_{\mathrm{target}} \in \{\mathrm{PLAX}, \mathrm{PSAX}, \mathrm{A4C}\}.$$

Force, robot pose, controller state, and recent actions are not inputs to the image-perception model; they are added later in the fused interaction state.

At update $t$, the model receives either the most recent frame or a short fixed window of recent frames:

$$\mathcal{I}_t = (B_{t-K+1}, \ldots, B_t), \qquad K \ge 1,$$

where $K$ is the number of frames: $K=1$ is a single frame, and $K>1$ is a short fixed window. The implementation chooses $K$. Each frame $B_i$ is a grayscale $H \times W$ matrix of pixel values, so the stacked input has size $H \times W \times K$.

Frames are preprocessed the same way as in training, and acquisition times are recorded so output latency can be measured.

## 3. Image-Perception Model and Target Conditioning

### A. Model Architecture and Target Conditioning

> - Describe a representative pretrained image encoder, such as a ResNet-based backbone.
> - Define separate output branches or heads.
> - Keep view and degradation branches image-only.
> - Make adequacy and directional branches target-conditioned or target-selected.
> - Prevent target-label leakage into view classification.
> - Lock the output interface while leaving exact architecture choices open.

### B. Training Data, Labels, and Preprocessing

> - Use patient- or recording-level train, validation, and test splits.
> - Prevent frame, clip, patient, and study leakage.
> - Include transfer learning and ultrasound-specific preprocessing.
> - Standardize across devices, image sizes, depth, gain, operators, and sites.
> - Separate visible-view labels from requested-target labels.
> - Define expert adequacy rubrics and handling of datasets without adequacy labels.
> - Allow branch-specific or masked losses when labels are missing.

### C. Training, Validation, and Runtime Characterization

> - Evaluate the model on held-out data.
> - Characterize calibration, ensemble disagreement, and uncertainty behavior.
> - Measure temporal stability and abrupt output changes.
> - Measure inference latency and update rate.
> - Test generalization across devices and operators.
> - Include off-target and difficult near-target cases.
> - Evaluate whether uncertainty identifies unreliable predictions.

## 4. Structured Image-Perception Packet

The perception packet organizes the image model output into five groups: view probabilities $\mathbf{p}^{\mathrm{view}}_t$, target-view adequacy scores $\mathbf{a}^{(v_{\mathrm{target}})}_t$, image-degradation evidence $\mathbf{e}^{\mathrm{deg}}_t$, probe-adjustment direction scores $\mathbf{d}^{(v_{\mathrm{target}})}_t$, and image-based uncertainty $\mathbf{u}_t$.

Together, these quantities form the structured output:

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

$$s_t = \frac{s_t^{(1)}+\cdots+s_t^{(N)}}{N}.$$

The spread of these estimates defines the component uncertainty. Because each estimate lies between 0 and 1, twice the population standard deviation lies between 0 and 1:

$$u_{s,t} = 2\,\operatorname{SD} \left(s_t^{(1)},\ldots,s_t^{(N)} \right).$$

For each output group, the reported uncertainty is the largest component uncertainty in that group.

- **Low uncertainty:** the models produce similar estimates for that output group.
- **High uncertainty:** the models produce different estimates for at least one component in that output group.

Close agreement does not guarantee that an output is correct. This output describes image-perception uncertainty only; physical action validity is determined downstream.

## 5. Physical Platform and Fused Interaction State

### A. Robot, Force Sensing, and Local Coordinate Frames

A robotic manipulator holds the probe and controls it in all six degrees of freedom. Five are image-guided in the probe-fixed frame:

$$\Delta \mathbf{x}_{\mathrm{img}} = [\Delta x_{\mathrm{p}},\ \Delta y_{\mathrm{p}},\ \Delta\theta_{x_{\mathrm{p}}},\ \Delta\theta_{y_{\mathrm{p}}},\ \Delta\theta_{z_{\mathrm{p}}}].$$

The sixth degree of freedom, translation along the probe-face normal $z_{\mathrm{p}}$, is controlled through force regulation.

The robot-base, end-effector/sensor, and probe-fixed frames carry these quantities through a calibrated chain:

$$\mathbf{T}_{\mathrm{bp},t} = \mathbf{T}_{\mathrm{bs},t}(\mathbf{q}_t)\,\mathbf{T}_{\mathrm{sp}},$$

with $\mathbf{T}_{\mathrm{bs},t}(\mathbf{q}_t)$ the forward kinematics from joint states $\mathbf{q}_t$ and $\mathbf{T}_{\mathrm{sp}}$ the fixed sensor-to-probe calibration. Inverse kinematics maps an approved bounded Cartesian probe correction to joint-space motion.

The local contact representation gives the current contact geometry:

$$\mathcal{C}_t = (\hat{\mathbf{n}}_t,\ \Pi_t,\ \tau^{\mathcal{C}}_t,\ \mathrm{valid}^{\mathcal{C}}_t),$$

where $\hat{\mathbf{n}}_t$ is the estimated local chest-surface normal, $\Pi_t$ the tangent plane, $\tau^{\mathcal{C}}_t$ the estimate time, and $\mathrm{valid}^{\mathcal{C}}_t$ whether the estimate is usable. It is re-estimated as the contact geometry changes. The estimation method and any tangent-axis convention are deferred.

The force/torque sensor observes a compensated wrench in the end-effector/sensor frame, with tool weight and bias removed, providing contact force $\mathbf{f}_t$ and contact torque $\boldsymbol{\tau}_t$. Normal and tangential force are not measured directly but decomposed relative to $\hat{\mathbf{n}}_t$:

$$F_{n,t} = \hat{\mathbf{n}}_t \cdot \mathbf{f}_t, \qquad \mathbf{f}^{\mathrm{tan}}_t = \mathbf{f}_t - F_{n,t}\,\hat{\mathbf{n}}_t.$$

These quantities define the physical interface consumed downstream: current probe pose, the local contact representation, and the decomposed contact loading, expressed in compatible frames.

### B. Fused State, Timing, and Action-Response History

> - Combine the perception packet, pose, force/torque, local frame, timestamps, and recent commands.
> - Include frame age and alignment of commands with resulting images.
> - Track recent image and force responses after actions.
> - Assess contact stability and sensor consistency.
> - Interpret likely pose displacement, coupling impairment, attenuation or shadowing, chest motion, and uncertain conditions.
> - Distinguish fused reliability from image-model uncertainty.
> - Mention optional local effective compliance or force-response information without claiming a complete chest model.

## 6. Hybrid Force and Pose Control

### A. Control Responsibilities and Robot Command Generation

> - Assign the normal direction primarily to force control.
> - Assign tangent-plane translations and probe rotations primarily to image-guided refinement.
> - Acknowledge that force and pose remain physically coupled.
> - Define proposed corrections in the local contact or probe frame.
> - Apply safety and kinematic filtering before execution.
> - Convert accepted corrections into end-effector or joint commands.
> - Bound command magnitude and velocity.

### B. Bounded Image-Guided Tangential and Angular Refinement

> - Use a fixed set of candidate tangential and angular adjustments.
> - Rank or score candidates using the directional output.
> - Remove unsafe or infeasible actions.
> - Let the controller choose step magnitude.
> - Allow mechanical and imaging settling time.
> - Evaluate post-action adequacy response.
> - Accept, reject, or reverse actions.
> - Reduce step size near attainment.
> - Prevent repeated motion and overshoot.
> - Explain this as derivative-free local refinement.

### C. Constant and Adaptive Normal-Force Control

> - Include a representative force or admittance-control law.
> - Define constant force as a fixed force reference, not fixed normal position.
> - Define adaptive force as bounded changes to the force reference.
> - State that both modes may move normally.
> - Let image evidence influence force only after fusion with measured physical state.
> - Explain when additional force may improve coupling or penetration.
> - Explain why force should not automatically increase for shadowing, adequate-force coupling loss, or persistent poor images.
> - Include force-adjustment acceptance, retention, rejection, or reversal.
> - Use the same image-guided pose strategy in both force modes where possible.

## 7. Safety and Runtime Operation

### A. Action Gating and Physical Limits

> - Define force magnitude and force-rate limits.
> - Define torque, translation, angle, velocity, and workspace limits.
> - Reject action under stale images, excessive latency, or communication loss.
> - Reject action under invalid or inconsistent sensors.
> - Reduce autonomy under high model uncertainty or low fused reliability.
> - Stop repeated non-improving actions.
> - Include operator stop input.
> - Distinguish model uncertainty from final action validity.

### B. Initialization, Refinement, Attainment, Maintenance, and Recovery

> - Use one compact numbered runtime sequence:
>   1. operator setup and near-window placement;
>   2. gradual establishment of safe contact;
>   3. validation of force, pose, image, timing, and reliability;
>   4. proposal and gating of one bounded adjustment;
>   5. evaluation of resulting image and force response;
>   6. sustained target attainment;
>   7. lower-amplitude maintenance corrections;
>   8. cause-aware recovery after persistent degradation.
> - Avoid separate headings for each runtime state.

### C. Hold, Unload, Retraction, and Operator Handoff

> - Define conditions for holding position.
> - Maintain safe contact without active refinement when appropriate.
> - Include controlled reduction of force.
> - Include retraction or emergency stopping.
> - Respond to loss of imaging or communication.
> - Return authority under persistent uncertainty or inability to recover.
> - Request gel, repositioning, or operator inspection when needed.
> - Preserve operator pause, takeover, and termination authority.

## 8. Technical Requirements and Validation

### A. Latency, Stability, and Predictability Requirements

> - Include inference latency.
> - Include complete-loop latency.
> - Include update rate.
> - Include temporal variation under held conditions.
> - Include detection or handling of abrupt score jumps.
> - Define persistence requirements before action.
> - Include uncertainty calibration.
> - Include directional consistency.
> - Justify numerical ranges using intended use, engineering need, or literature.

### B. Controller, Safety, and Operating-Envelope Validation

> - Test directional action-response accuracy.
> - Test force tracking and compliance response.
> - Compare constant and adaptive force modes.
> - Test pose and force coupling under different local surface conditions.
> - Test overshoot and repeated-action behavior.
> - Test safety stop and operator override.
> - Test attainment, maintenance, and recovery behavior.
> - Define stage-specific operating envelopes for phantom and volunteer use.
> - Define fallback behavior when requirements are not met.
> - Prefer one compact table:
>
> | Property | Requirement | How measured | Response if unmet |
> | -------- | ----------- | ------------ | ----------------- |

---

## Archived Notes & Guidelines

### 1. System Concept, Scope, and Information Flow

> - Define the operator-selected target view and near-window probe placement.
> - State operator responsibilities: gel, initial orientation, authorization, monitoring, pause, and stop.
> - Distinguish supervised local refinement from full autonomous scanning.
> - Show the full flow: perception, fusion, control proposal, safety approval, robot action, observed response.
> - State what the robot automates and what remains under operator authority.

### 2. Runtime Inputs to Image Perception

> - Define the current B-mode frame or short image window.
> - Define the operator-selected target view as a task condition.
> - Distinguish sensory input from task conditioning.
> - Include preprocessing consistency and acquisition timestamps.
> - Explicitly exclude force, robot pose, and controller state from the perception model.

### 4. Structured Image-Perception Packet

> - View output answers what is visible.
> - Adequacy output answers which aspect of the selected target is weak.
> - Degradation output answers what image pattern may be impairing acquisition.
> - Direction output answers which bounded geometric adjustment is supported.
> - Uncertainty output answers how much the image models disagree.
> - Do not let packet outputs become physical-state determinations, robot commands, or safety decisions.

### 5. Physical Platform and Fused Interaction State

#### A. Robot, Force Sensing, and Local Coordinate Frames

> - Define the robotic arm and controlled probe degrees of freedom.
> - State the basic role of forward and inverse kinematics.
> - Define wrist force/torque sensing or equivalent force estimation.
> - Include measured normal force, tangential force, and contact torque.
> - Define robot-base, end-effector, probe-fixed, and local chest-contact frames.
> - Define the estimated local chest-surface normal and tangent plane.
> - Note that the local frame may change over a curved, moving, or deformable chest.
> - Explain how local-frame or probe-frame corrections become robot commands.
