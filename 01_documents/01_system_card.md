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

One shared image encoder produces image features, and separate output heads read those features into the five packet groups. The target view $v_{\mathrm{target}}$ is routed only to the heads whose meaning depends on it.

A single encoder $E$ maps the image input to shared features:

$$\mathbf{h}_t = E(\mathcal{I}_t).$$

The view and degradation heads read these features alone, because what is visible and what is impairing the image are properties of the image itself:

$$\mathbf{p}^{\mathrm{view}}_t = h_{\mathrm{view}}(\mathbf{h}_t), \qquad \mathbf{e}^{\mathrm{deg}}_t = h_{\mathrm{degradation}}(\mathbf{h}_t).$$

The adequacy and direction heads also take $v_{\mathrm{target}}$, because adequacy and supported corrections are defined relative to the requested view:

$$\mathbf{a}^{(v_{\mathrm{target}})}_t = h_{\mathrm{adequacy}}(\mathbf{h}_t,\ v_{\mathrm{target}}), \qquad \mathbf{d}^{(v_{\mathrm{target}})}_t = h_{\mathrm{direction}}(\mathbf{h}_t,\ v_{\mathrm{target}}).$$

- **Backbone.** $E$ is a representative pretrained image encoder, such as a ResNet-based convolutional network, adapted by transfer learning. This interface is fixed; the exact backbone is a deferred implementation choice.
- **Target conditioning.** Conditioning is realized either as target-selected heads, one per view and chosen at runtime by $v_{\mathrm{target}}$, or as one head conditioned on a learned view embedding. Per-view selection is the starting point. Separate full per-view models are rejected: they duplicate features, give each model less data, and behave inconsistently. Either way, only three adequacy scores and one ten-entry directional vector are ever reported.
- **Uncertainty.** The reported uncertainty $\mathbf{u}_t$ comes from an ensemble of a small fixed number of same-architecture models trained from different initializations, summarized per output group by the rule fixed in Section 4E.

The view head stays image-only, and the conditioned heads are trained to read the image rather than the request; together these prevent target-label leakage, the failure in which the model infers the answer from the requested target rather than from the image:

- $v_{\mathrm{target}}$ is never an input to the view head, so it cannot shortcut to "the requested view is present."
- The visible-view label $y^{\mathrm{view}}_t$ and the requested target $v_{\mathrm{target}}$ are kept as distinct variables; training data are not built by setting $v_{\mathrm{target}} = y^{\mathrm{view}}_t$ on every frame.
- Conditioned heads are trained on target-image mismatches (the same image scored under different targets) and on difficult near-target negatives (foreshortened, wrong level, partial structures).
- Targets, visible views, devices, operators, and patient groups are balanced so the model learns anatomy rather than acquisition artifacts.

### B. Training Data, Labels, and Preprocessing

Training uses public, expert-annotated echocardiography datasets, each under its own labeling protocol: CAMUS as the primary image-quality-grade anchor, TMED-2 for view-type labels and a large unlabeled pool, EchoNet-Dynamic for large-scale A4C representation, and Unity Imaging for PLAX and quality coverage.

- **Splitting.** Data are split at the patient or recording level: all frames from one acquisition, subject, or study stay in a single split, and a held-out test set is reserved for final characterization. No frame, clip, patient, or study crosses splits.
- **Preprocessing.** Frames are preprocessed identically to runtime and standardized across manufacturer, device, image size, depth, gain, operator technique, and site, with ultrasound-specific preprocessing and augmentation for cross-device generalization.
- **Heterogeneous labels.** A given frame may carry a view label, adequacy labels, no degradation label, or no directional label. This is expected and handled by the masked loss in Section 3C, so a head updates only on frames that carry its label.
- **Adequacy labels.** Each adequacy component needs an expert labeling rubric, with a defined plan for datasets that lack adequacy labels (additional expert labeling, masked losses, or both). View classification and adequacy labeling are distinct tasks and are not interchangeable.
- **Directional labels.** Directional labels come from action-response transitions, generated mainly by controlled probe perturbations around expert-confirmed phantom poses. The corrective label for a perturbation is its reverse adjustment, which yields physically grounded labels without asking experts to estimate corrections from static images (Section 3C).

### C. Training, Validation, and Runtime Characterization

The model is built one output at a time, so each output is verified before the next is added:

1. Encoder $E$ and the view head from $\mathcal{I}_t$; this stage also validates the splits and preprocessing.
2. Target-conditioned adequacy heads, freezing much of the pretrained encoder first, then optionally fine-tuning jointly; this freeze-then-fine-tune sequence is the transfer-learning procedure.
3. Degradation evidence as a multi-label output, with separate sigmoid scores rather than a softmax, the same independent-score form used by the adequacy and directional heads (only the view head is a softmax over mutually exclusive classes).
4. Directional guidance from action-response data.
5. Uncertainty calibration and integration, after each output performs adequately on its own.

A masked multitask loss sums one term per output, weighted by $\lambda_{\bullet}$, with each term active only on training samples that carry the matching label (a frame for the view, adequacy, and degradation terms; a transition for the directional term):

$$\mathcal{L} = \lambda_{\mathrm{view}}\mathcal{L}_{\mathrm{view}} + \lambda_{\mathrm{adequacy}}\mathcal{L}_{\mathrm{adequacy}} + \lambda_{\mathrm{degradation}}\mathcal{L}_{\mathrm{degradation}} + \lambda_{\mathrm{direction}}\mathcal{L}_{\mathrm{direction}}.$$

The directional head is trained on transitions, not single images. A training sample pairs the images before and after one applied action $a_t$, where $a_t$ is one of the ten bounded directions, together with the resulting change in adequacy:

$$(\mathcal{I}_t,\ v_{\mathrm{target}},\ a_t,\ \mathcal{I}_{t+1}), \qquad \Delta\mathbf{a}_t = \mathbf{a}^{(v_{\mathrm{target}})}_{t+1} - \mathbf{a}^{(v_{\mathrm{target}})}_t.$$

Each action is labeled by whether the target-view adequacy improved overall, measured by the change in its aggregate $Q(\mathbf{a}^{(v_{\mathrm{target}})}_t)$ (the aggregate valued 0 to 1, defined in Section 6A). The controlled perturbations make this concrete: a perturbation from a known-good pose degrades the image, so its reverse adjustment is the improving action and supplies the positively labeled corrective direction. Each transition supervises the applied direction and, through that reverse, its opposite; the remaining directions are left unlabeled and masked. The head learns a per-action improvement score, conceptually the probability that adequacy improves given the current image, target, and action. This probability is a training-side construct only and is not reported as a calibrated probability: the reported output keeps the Section 4D meaning, how strongly the image supports each bounded adjustment.

Uncertainty is treated as a validation problem, not a single output neuron. It is characterized through probability calibration, ensemble disagreement, temporal consistency, out-of-distribution behavior, and whether high uncertainty actually flags wrong or out-of-envelope predictions. A raw network confidence value is never assumed sufficient on its own.

Runtime characterization, performed on held-out data, determines whether the output is fit for closed-loop use (Section 8): inference latency and update rate; temporal stability and abrupt-jump behavior under held conditions; response to pose change; graded response to coupling loss; generalization across devices and operators; and off-target rejection on difficult near-target cases.

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

A calibrated transform chain gives the probe pose in the robot-base frame:

$$\mathbf{T}_{\mathrm{bp},t} = \mathbf{T}_{\mathrm{bs},t}(\mathbf{q}_t)\,\mathbf{T}_{\mathrm{sp}},$$

Forward kinematics gives the base-to-sensor transform $\mathbf{T}_{\mathrm{bs},t}$ from the current joint states $\mathbf{q}_t$, and $\mathbf{T}_{\mathrm{sp}}$ is the fixed sensor-to-probe calibration. Inverse kinematics maps an approved probe correction to joint motion.

The local contact representation gives the current contact geometry:

$$\mathcal{C}_t = (\hat{\mathbf{n}}_t,\ \Pi_t,\ \tau^{\mathcal{C}}_t,\ \mathrm{valid}^{\mathcal{C}}_t),$$

where $\hat{\mathbf{n}}_t$ is the estimated local chest-surface normal, $\Pi_t$ the tangent plane, $\tau^{\mathcal{C}}_t$ the estimate time, and $\mathrm{valid}^{\mathcal{C}}_t$ whether the estimate is usable. It is re-estimated as the contact geometry changes. The estimation method and any tangent-axis convention are deferred.

The force/torque sensor observes a compensated wrench (the combined contact force and torque) in the end-effector/sensor frame, with tool weight and bias removed, providing contact force $\mathbf{f}_t$ and contact torque $\boldsymbol{\tau}_t$. Normal and tangential force are not measured directly but decomposed relative to $\hat{\mathbf{n}}_t$, with both vectors first expressed in a common frame and $\hat{\mathbf{n}}_t$ pointing from the probe into the chest, so compression gives $F_{n,t} > 0$:

$$F_{n,t} = \hat{\mathbf{n}}_t \cdot \mathbf{f}_t, \qquad \mathbf{f}^{\mathrm{tan}}_t = \mathbf{f}_t - F_{n,t}\,\hat{\mathbf{n}}_t.$$

These quantities define the physical interface consumed downstream: current probe pose, the local contact representation, and the decomposed contact loading, expressed in compatible frames.

### B. Fused State, Timing, and Action-Response History

The fused interaction state $\mathbf{z}_t$ combines the perception packet with the measured physical signals the perception model never sees, so the controller and supervisor act on one coherent object rather than on the image alone. It is the second of four representational levels along the Section 1 flow: the image-perception packet $\mathbf{o}_t$ (Section 4), the fused interaction state $\mathbf{z}_t$, the command proposal (Section 6), and safety-gated execution (Section 7).

$$\mathbf{z}_t = \Phi\big(\mathbf{o}_t,\ \mathbf{T}_{\mathrm{bp},t},\ F_{n,t},\ \mathbf{f}^{\mathrm{tan}}_t,\ \boldsymbol{\tau}_t,\ \mathcal{C}_t,\ \mathcal{H}_t,\ \alpha_t\big),$$

where $\mathbf{o}_t$ is the perception packet (Section 4), $\mathbf{T}_{\mathrm{bp},t}$ the probe pose, $F_{n,t}$, $\mathbf{f}^{\mathrm{tan}}_t$, and $\boldsymbol{\tau}_t$ the decomposed contact loading, and $\mathcal{C}_t$ the local contact representation (all from Section 5A); $\mathcal{H}_t$ is the recent action-response history and $\alpha_t$ the frame age. The fusion map $\Phi$ exposes two derived quantities used downstream: the physical-cause hypothesis $\mathbf{m}_t$ and the fused reliability $r_t$.

- **Timing and pairing.** The frame age $\alpha_t = t - \tau^{\mathrm{img}}_t$ is the delay between the acquisition time $\tau^{\mathrm{img}}_t$ of the newest frame in $\mathcal{I}_t$ (a timestamp, distinct from the contact torque $\boldsymbol{\tau}_t$) and the current update $t$. The history $\mathcal{H}_t$ stores recent commands aligned with the image and force responses they produced, so an improvement is attributed to the action that caused it.
- **Physical-cause hypothesis.** $\mathbf{m}_t$ scores the candidate physical causes of a degraded image: pose displacement, coupling impairment, attenuation or shadowing, chest motion, or uncertain. It is a separate object from the image-degradation evidence $\mathbf{e}^{\mathrm{deg}}_t$: the image reports evidence, and $\mathbf{m}_t$ forms the determination by combining that evidence with force and recent motion. This is how it separates the confusions the system must avoid:
    - coupling impairment: high $e_{\mathrm{coupling}}$ with low or marginal $F_{n,t}$;
    - insufficient penetration, as over a thick chest wall: high $e_{\mathrm{penetration}}$ with $F_{n,t}$ already adequate;
    - pose displacement off the acoustic window: falling adequacy while $e_{\mathrm{coupling}}$, $e_{\mathrm{penetration}}$, and $F_{n,t}$ stay nominal.
- **Fused reliability.** $r_t$ reports whether the fused signals can be acted on: whether contact is stable, whether the force and image signals are mutually consistent, and whether the frame is fresh (small $\alpha_t$). It is distinct from the image-model uncertainty $\mathbf{u}_t$, which reports only whether the ensemble members agree. The two can diverge: the ensemble may agree on an image while contact is unstable, or disagree on an image while the physical signals are clean.
- **Optional compliance.** $\Phi$ may track recent force-response behavior or an effective local compliance if useful, but this does not imply a complete chest-wall stiffness model, and $\mathbf{z}_t$ stays coherent without it.

Contact stability and sensor consistency are assessed here, through $r_t$, before any action is considered downstream. The working symbol $\mathbf{z}_t$, the field list above, and the command-to-response pairing in $\mathcal{H}_t$ are a reference design; the final symbol and exact fields remain to be settled.

## 6. Hybrid Force and Pose Control

### A. Control Responsibilities and Robot Command Generation

The command proposal is the third representational level: from the fused state $\mathbf{z}_t$, the controller proposes one bounded correction per update, then passes it to the safety supervisor (Section 7). Responsibility is split by direction:

- **Normal direction ($z_{\mathrm{p}}$):** assigned to force control, which regulates the normal contact force (Section 6C).
- **Tangent plane and rotations:** the translations along $x_{\mathrm{p}}$ and $y_{\mathrm{p}}$ and the rotations about $x_{\mathrm{p}}$, $y_{\mathrm{p}}$, and $z_{\mathrm{p}}$ are assigned to image-guided refinement (Section 6B).

Force and pose remain physically coupled through chest deformation, so each channel keeps its primary objective while the controller monitors the other's cross-effects. The force-controlled direction is the chest-surface normal $\hat{\mathbf{n}}_t$ (Section 5A), which coincides with the probe-face normal $z_{\mathrm{p}}$ only when the probe sits squarely on the surface; in general they differ, so the controller maps between the probe frame and the contact frame before issuing commands. Proposed corrections are expressed in the local contact or probe frame, passed through safety and kinematic filtering, and converted into bounded end-effector or joint commands with limited magnitude and velocity.

A single scalar summarizes attainment. The composite attainment scalar is

$$s_t = p_{v_{\mathrm{target}},\,t}\,\cdot\,Q\!\left(\mathbf{a}^{(v_{\mathrm{target}})}_t\right),$$

where $p_{v_{\mathrm{target}},\,t}$ is the entry of $\mathbf{p}^{\mathrm{view}}_t$ for the selected target view, and $Q$ aggregates the three adequacy components into one value from 0 to 1. So $s_t$ runs from 0 to 1, and the product is deliberately conservative: either a low view probability or a low adequacy drives $s_t$ low, so $s_t$ is high only when the image is both the requested view and an adequate instance of it. $Q$ is built so that a weak score on any one adequacy component pulls the result down, and two strong components cannot hide a weak third; its exact form is deferred, with candidates including the minimum of the three components, a smooth minimum, the geometric mean, or a mean with a floor on each component. Attainment is declared when $s_t$ reaches and holds at or above a threshold $s^*$ for the required interval (the 2 s hold of Section 7B). In Section 4E the symbol $s_t$ is a local placeholder for the ensemble average of an arbitrary output component; here $s_t$ is the composite attainment scalar, a different quantity.

The roles are divided as follows:

- $s_t$ drives attainment, stopping, maintenance thresholds, and experimental comparison (Sections 7, 8).
- The structured packet $\mathbf{o}_t$ and the fused state $\mathbf{z}_t$ drive action selection and action gating.
- The directional vector $\mathbf{d}^{(v_{\mathrm{target}})}_t$ informs which bounded move to try; it does not replace $s_t$.

The controller refines the pose by trying small bounded moves and keeping those that improve the image, rather than by computing a gradient; this is derivative-free local refinement, and $\mathbf{d}^{(v_{\mathrm{target}})}_t$ makes the bounded search more informed than an undirected one. Each update follows one propose, gate, evaluate cycle:

1. The supervisor verdict $g_t$ (Section 7A) is checked first; the controller may act only when it permits.
2. If $s_t \ge s^*$ has held for the required interval, the system is in maintenance (Section 7B); otherwise it proposes one correction.
3. A pose correction is selected from $\mathbf{d}^{(v_{\mathrm{target}})}_t$ (Section 6B) and, in parallel, the normal-force reference is managed by the force law (Section 6C); the physical-cause hypothesis $\mathbf{m}_t$ governs which channel responds.
4. One bounded command is applied, settling time is allowed, and the image and force response is evaluated, after which the action is accepted, rejected, or reversed.

The division of roles above is fixed. How $s_t$, $\mathbf{d}^{(v_{\mathrm{target}})}_t$, $\mathbf{e}^{\mathrm{deg}}_t$, $\mathbf{u}_t$, and $\mathbf{z}_t$ combine into a single accept-or-reject decision is a genuine open design decision: Sections 6B and 6C give the reference design, and the exact rule, with its thresholds and weighting, remains to be settled.

### B. Bounded Image-Guided Tangential and Angular Refinement

The reference design ranks bounded actions:

1. The candidate set is the five image-guided degrees of freedom, scored as the ten opposed directions in $\mathbf{d}^{(v_{\mathrm{target}})}_t$: translation along $x_{\mathrm{p}}$ and $y_{\mathrm{p}}$, and rotation about $x_{\mathrm{p}}$, $y_{\mathrm{p}}$, and $z_{\mathrm{p}}$.
2. Candidates are ranked by their directional scores, and unsafe or infeasible actions are removed.
3. The controller, not the model, sets the step magnitude, which is small and bounded.
4. After mechanical and imaging settling time, the post-action adequacy response is evaluated over a short sequence of frames.
5. The action is accepted, rejected, or reversed. The step size is reduced as adequacy nears attainment, and repeated motion and overshoot are prevented.

If all directional scores are low, no pose correction is supported and the controller does not move on this channel. This remains derivative-free local refinement informed by the directional output; if only coarse pose-response structure exists (Section 8), the same behavior reduces to a bounded scripted local sweep with image-quality stopping.

As an extension only, an online local response model estimates an action-response matrix $J_t$ around the current pose from recent safe perturbations,

$$\Delta\mathbf{a}_t \approx J_t\,\Delta\mathbf{x}_{\mathrm{img}},$$

where $\Delta\mathbf{x}_{\mathrm{img}}$ is a small move over the five image-guided degrees of freedom (Section 5A), $\Delta\mathbf{a}_t$ is the resulting change in the three adequacy components, and $J_t$ is the 3 by 5 matrix relating them (a response map over probe motion, not a kinematic Jacobian over joints). The controller then chooses a bounded, regularized update that raises $s_t$, subject to motion, force, and workspace limits. This is more capable and more complex than the ranked-action design, and is presented as an alternative, not a requirement.

### C. Constant and Adaptive Normal-Force Control

The normal channel uses an admittance law: a control rule that turns the force error into a commanded probe velocity, so the probe yields when it presses too hard and advances when contact is too light, tracking a bounded force reference. With the measured normal force $F_{n,t}$ from Section 5A and a commanded force reference $F_{\mathrm{ref},t}$, the force error drives a bounded normal velocity:

$$e_{F,t} = F_{\mathrm{ref},t} - F_{n,t}, \qquad M_a\,\dot{v}_{n,t} + B_a\,v_{n,t} = e_{F,t}, \qquad |v_{n,t}| \le v_{n,\max}, \qquad F_{\min} \le F_{\mathrm{ref},t} \le F_{\max},$$

where $e_{F,t}$ is the force-tracking error (a scalar force mismatch, unrelated to the degradation evidence $\mathbf{e}^{\mathrm{deg}}_t$), $v_{n,t}$ is the commanded normal velocity, $M_a$ and $B_a$ are the virtual admittance inertia and damping, $v_{n,\max}$ bounds the normal speed, and $[F_{\min},\ F_{\max}]$ bounds the reference. The gains $M_a$ and $B_a$ are fixed, chosen conservatively for the stiffest contact expected (a rib) so the normal response stays stable over soft tissue as well; the controller adapts the force reference, not these gains. The effective-compliance estimate of Section 5B could inform a bounded gain schedule later, but that is not the starting design.

This law commands continued advance whenever the measured force is below the reference, so a loss of contact must not be read as a reason to drive forward. If $F_{n,t}$ falls below a contact-detection threshold, the safety supervisor holds or unloads rather than advancing (the controlled-unload verdict of Section 7A), and a bounded normal-advance limit caps travel in every case. This contact-detection threshold sits below the low-measured-force regime that permits a coupling-driven reference increase (the table below), so lost contact triggers the interlock before any force increase is allowed. This is what prevents the probe from overshooting the acoustic window into the patient when coupling or contact is lost.

- **Constant-force mode:** $F_{\mathrm{ref},t} = F_0$, a fixed force reference, not a fixed normal position. The probe may move along the normal as the surface moves while the target force holds.
- **Adaptive-force mode:** the reference may change within bounds, $F_{\mathrm{ref},t+1} = \operatorname{clip}(F_{\mathrm{ref},t} + \Delta F_t,\ F_{\min},\ F_{\max})$.

Both modes may move along the normal; the difference is only whether the target force is fixed or adapts. Image evidence influences force only after fusion with the measured physical state, and never commands normal displacement directly: at most it requests a bounded change $\Delta F_t$ in the reference. The fused conditions that permit a force response are:

| Fused condition | Force response |
| --- | --- |
| Coupling-impairment evidence with low measured force | small bounded increase in the force reference permitted |
| Coupling-impairment evidence with adequate measured force | do not assume more force helps; hold force, try a pose adjustment, or request operator inspection or gel |
| Acoustic shadowing | no force increase; prefer tangential or angular correction |
| Insufficient penetration | limited force adjustment only if within range and evidence suggests compression helps; no repeated escalation |
| High adequacy | hold the current reference |
| Image improves after a force change | retain the new reference |
| Image worsens after a force change | reverse or reject the change |

A low adequacy or attainment score must never, by itself, drive continued force escalation.

Force adaptation stays downstream of perception because the controller has direct access to measured force and torque, owns the force-rate and abort limits, and keeps the perception model focused on image interpretation. Normal translation is excluded from $\mathbf{d}^{(v_{\mathrm{target}})}_t$ because a fixed normal displacement has an inconsistent physical effect, depending on local compliance (little force change in soft tissue, a large increase over a rib), so the normal direction is represented by a force objective rather than an image-predicted translation. For experimental comparison, the same image-guided pose strategy is used in both force modes, so the contrast isolates the added contribution of adaptive force.

Deferred tuning: the force-reference increment size, the persistence window before an adjustment, the improvement metric after an adjustment, the reversal or rejection criterion, and whether reference adaptation runs only during persistent coupling-impairment recovery or also during maintenance under strict triggers.

## 7. Safety and Runtime Operation

### A. Action Gating and Physical Limits

Safety-gated execution is the fourth and final representational level, independent of the controller: a safety supervisor decides whether either controller may act, and only it can authorize motion. Its verdict $g_t$ takes one of five values: act, hold, controlled unload, return to operator, or emergency stop.

- **Physical limits:** maximum force and force rate, and torque, translation, angle, velocity, and workspace limits.
- **Signal validity:** action is rejected under stale images, excessive latency, or communication loss, and under invalid or mutually inconsistent sensors.
- **Reduced autonomy:** under high model uncertainty $\mathbf{u}_t$ or low fused reliability $r_t$, the supervisor does less and returns authority to the operator. Uncertainty reduces autonomy; it does not expand search.
- **Non-improvement and operator authority:** repeated non-improving actions are stopped, and an operator stop is always honored.

Model uncertainty and action validity are different things: the model reports $\mathbf{u}_t$, and the supervisor decides through $g_t$ whether the robot may act.

### B. Initialization, Refinement, Attainment, Maintenance, and Recovery

One runtime sequence covers the full session:

1. Operator setup and near-window probe placement.
2. Gradual establishment of safe contact, proceeding only with detected contact, in-limit force and torque, and in-workspace pose.
3. Validation of force, pose, image stability, timing, and reliability before motion, so a single unstable frame cannot cause an action.
4. Proposal and safety gating of one bounded adjustment (Sections 6, 7A).
5. Evaluation of the resulting image and force response, then accept, reject, or reverse.
6. Sustained target attainment, declared when $s_t \ge s^*$ holds for the required interval (a 2 s sustained hold).
7. Lower-amplitude maintenance corrections, where a brief sub-threshold dip (under about 0.5 s, the maintenance tolerance of Section 8B) does not trigger motion.
8. Cause-aware recovery after persistent degradation, keyed to the physical-cause hypothesis $\mathbf{m}_t$:
   - likely pose displacement: a bounded tangential or angular correction;
   - likely coupling impairment with low force: a small bounded force-reference increase;
   - suspected coupling impairment with adequate force: a hold plus operator inspection or gel;
   - suspected attenuation or shadowing: an angular or lateral correction with no automatic force increase;
   - likely chest motion: continued force tracking of the moving surface, with no pose change;
   - high uncertainty or conflicting sensors: a hold and return of authority to the operator.

Here $s^*$ is the attainment threshold on $s_t$ defined in Section 6A, the bar both factors of $s_t$ must jointly clear.

### C. Hold, Unload, Retraction, and Operator Handoff

- Holding position and maintaining safe contact without active refinement are defined operating conditions, not failures.
- Controlled reduction of force, retraction, and hardware emergency stop are available at all times.
- Loss of imaging or communication triggers a safe response, holding position or unloading in a controlled way.
- Authority returns to the operator under persistent uncertainty or an inability to recover, with a request for gel, repositioning, or operator inspection when needed.
- Operator pause, takeover, and termination authority are preserved throughout.

## 8. Technical Requirements and Validation

### A. Latency, Stability, and Predictability Requirements

Before the perception output drives motion, it must meet timing and stability requirements, each justified by intended use and literature rather than as a bare threshold.

A classification probability or confidence score can be nonlinear, discontinuous, and prone to unexplained jumps, so the output is conditioned before it closes the loop, through temporal aggregation, a stability floor, explicit persistence requirements, abrupt-jump handling, and uncertainty gating (Sections 6, 7).

The committed signal requirements are:

- **Latency:** the median at or below 100 ms and the 95th percentile at or below 150 ms, with an update rate at or above 10 Hz. The 30 to 50 ms within-cardiac-cycle target is future, not committed.
- **Stability:** the median absolute deviation of $s_t$ at or below 0.05 within held-constant windows. This is the stability floor against which the pose-response criterion is read.
- **Graded coupling response:** a monotone decline in $s_t$ as coupling degrades, with a negative rank correlation of magnitude at or above 0.7, or a significantly negative ordinal slope.
- **Local pose-response:** an ordered decline in $s_t$ out to the search bound, plus a capture basin: a useful-sized region around the target pose within which moves that improve the image and moves that worsen it change $s_t$ by more than the stability floor.

### B. Controller, Safety, and Operating-Envelope Validation

Validation covers:

- directional action-response accuracy;
- force tracking and compliance response;
- the constant-versus-adaptive force comparison;
- pose-force coupling under different local surface conditions;
- overshoot and repeated-action behavior;
- safety stop and operator override;
- attainment, maintenance, and recovery behavior;
- stage-specific operating envelopes for phantom and volunteer use;
- fallback behavior when a requirement is not met.

The control proxy stays separate from the diagnostic-quality standard: a case where $s_t$ is high but blinded review judges the view inadequate is scored as a control-proxy failure, never a success. Two independent blinded readers score views against published echocardiography adequacy guidelines (the British Society of Echocardiography, BSE, and the American Society of Echocardiography, ASE), and their agreement is reported as Cohen's kappa.

The validation endpoints are summarized in one table (Aim 2 is the attainment experiment; Aim 3 is the maintenance-and-recovery experiment):

| Endpoint | Requirement | How measured | Response if unmet |
| --- | --- | --- | --- |
| Attainment benefit (Aim 2) | integrated-minus-pose-only difference, 95% confidence-interval lower bound at or above +10 percentage points | matched start-pose blocks, integrated versus pose-only | narrow the claim to pose refinement |
| Diagnostic-quality guardrail | 95% confidence-interval lower bound at or above -15 percentage points | two blinded readers versus BSE and ASE criteria, Cohen's kappa | report quality loss; not counted as attainment |
| Maintenance duration (Aim 3) | longer above $s^*$ than pose-only | Kaplan-Meier and restricted mean survival time to 60 s; dips under about 0.5 s tolerated | report no maintenance benefit |
| Recovery time | re-attainment within 30 s | from the $s^*$ down-crossing to a 2 s sustained hold; non-recoveries retained | retain as non-recovery |
| Respiratory robustness | benefit maintained as severity rises | sweep at 5, 10, and 15 mm peak-to-peak near 0.25 Hz, plus an irregular condition | report the severity at which benefit is lost |
| Force safety | within a conservative literature-derived bound | force and torque logging against $[F_{\min},\ F_{\max}]$ | abort and controlled unload |

The force bound is derived from obstetric probe-force literature and used as a conservative safety reference, not as a validated force range for transthoracic echocardiography (TTE). The rib-shadowing, window-geometry, and coupling-loss disturbance modules are added only if reproducible, and otherwise reported as not built.

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

### 3. Image-Perception Model and Target Conditioning

#### A. Model Architecture and Target Conditioning

> - Describe a representative pretrained image encoder, such as a ResNet-based backbone.
> - Define separate output branches or heads.
> - Keep view and degradation branches image-only.
> - Make adequacy and directional branches target-conditioned or target-selected.
> - Prevent target-label leakage into view classification.
> - Lock the output interface while leaving exact architecture choices open.

#### B. Training Data, Labels, and Preprocessing

> - Use patient- or recording-level train, validation, and test splits.
> - Prevent frame, clip, patient, and study leakage.
> - Include transfer learning and ultrasound-specific preprocessing.
> - Standardize across devices, image sizes, depth, gain, operators, and sites.
> - Separate visible-view labels from requested-target labels.
> - Define expert adequacy rubrics and handling of datasets without adequacy labels.
> - Allow branch-specific or masked losses when labels are missing.

#### C. Training, Validation, and Runtime Characterization

> - Evaluate the model on held-out data.
> - Characterize calibration, ensemble disagreement, and uncertainty behavior.
> - Measure temporal stability and abrupt output changes.
> - Measure inference latency and update rate.
> - Test generalization across devices and operators.
> - Include off-target and difficult near-target cases.
> - Evaluate whether uncertainty identifies unreliable predictions.

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

#### B. Fused State, Timing, and Action-Response History

> - Combine the perception packet, pose, force/torque, local frame, timestamps, and recent commands.
> - Include frame age and alignment of commands with resulting images.
> - Track recent image and force responses after actions.
> - Assess contact stability and sensor consistency.
> - Interpret likely pose displacement, coupling impairment, attenuation or shadowing, chest motion, and uncertain conditions.
> - Distinguish fused reliability from image-model uncertainty.
> - Mention optional local effective compliance or force-response information without claiming a complete chest model.

### 6. Hybrid Force and Pose Control

#### A. Control Responsibilities and Robot Command Generation

> - Assign the normal direction primarily to force control.
> - Assign tangent-plane translations and probe rotations primarily to image-guided refinement.
> - Acknowledge that force and pose remain physically coupled.
> - Define proposed corrections in the local contact or probe frame.
> - Apply safety and kinematic filtering before execution.
> - Convert accepted corrections into end-effector or joint commands.
> - Bound command magnitude and velocity.

#### B. Bounded Image-Guided Tangential and Angular Refinement

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

#### C. Constant and Adaptive Normal-Force Control

> - Include a representative force or admittance-control law.
> - Define constant force as a fixed force reference, not fixed normal position.
> - Define adaptive force as bounded changes to the force reference.
> - State that both modes may move normally.
> - Let image evidence influence force only after fusion with measured physical state.
> - Explain when additional force may improve coupling or penetration.
> - Explain why force should not automatically increase for shadowing, adequate-force coupling loss, or persistent poor images.
> - Include force-adjustment acceptance, retention, rejection, or reversal.
> - Use the same image-guided pose strategy in both force modes where possible.

### 7. Safety and Runtime Operation

#### A. Action Gating and Physical Limits

> - Define force magnitude and force-rate limits.
> - Define torque, translation, angle, velocity, and workspace limits.
> - Reject action under stale images, excessive latency, or communication loss.
> - Reject action under invalid or inconsistent sensors.
> - Reduce autonomy under high model uncertainty or low fused reliability.
> - Stop repeated non-improving actions.
> - Include operator stop input.
> - Distinguish model uncertainty from final action validity.

#### B. Initialization, Refinement, Attainment, Maintenance, and Recovery

> - Use one compact numbered runtime sequence: operator setup; safe contact; pre-motion validation; proposal and gating; response evaluation; sustained attainment; maintenance corrections; cause-aware recovery.
> - Avoid separate headings for each runtime state.

#### C. Hold, Unload, Retraction, and Operator Handoff

> - Define conditions for holding position.
> - Maintain safe contact without active refinement when appropriate.
> - Include controlled reduction of force, retraction, or emergency stopping.
> - Respond to loss of imaging or communication.
> - Return authority under persistent uncertainty or inability to recover.
> - Request gel, repositioning, or operator inspection when needed.
> - Preserve operator pause, takeover, and termination authority.

### 8. Technical Requirements and Validation

#### A. Latency, Stability, and Predictability Requirements

> - Include inference latency, complete-loop latency, and update rate.
> - Include temporal variation under held conditions and abrupt-jump handling.
> - Define persistence requirements before action.
> - Include uncertainty calibration and directional consistency.
> - Justify numerical ranges using intended use, engineering need, or literature.

#### B. Controller, Safety, and Operating-Envelope Validation

> - Test directional accuracy, force tracking, constant-versus-adaptive force, and pose-force coupling.
> - Test overshoot, safety stop, operator override, attainment, maintenance, and recovery.
> - Define stage-specific operating envelopes and fallback behavior when requirements are not met.
> - Prefer one compact requirements table.
