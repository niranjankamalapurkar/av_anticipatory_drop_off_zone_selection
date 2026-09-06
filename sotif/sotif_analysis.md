## 1. Item Definition

**Intended function.** While anticipatory search is active (Route Planner's enablement flag, interface_control_document.md Interface 2), Behavior Planner evaluates curb-adjacent candidates against legal, geometric, and kinematic criteria (decision_tree.md), commits to one, re-validates that commitment as the vehicle approaches, and either holds the commitment through a stop or de-commits and returns to the active lane. Once stopped, Trip Manager aggregates ground-truth signals to authorize rider egress, and Vehicle Controls enforces a local interlock before unlocking.

**Operating context.** Dense urban core, weekday morning commute hours, drop-off only, curb-adjacent lane assumption (odd.md). This item definition does not re-derive the ODD; it inherits it.

**Functional boundaries.** In scope: candidate evaluation, commit/de-commit decisioning, egress authorization, unlock interlock. Out of scope, per standing assumptions in exhaustive_failures_list.md: cloud-backend data authoring and integrity validation (assumed handled at the connectivity layer before reaching any vehicle-side consumer), and all pre-existing baseline module behavior (Perception, Prediction, Localization, Motion Planner, actuation) not newly exercised by this item.

**Interfaces relied upon.** interface_control_document.md (all numbered interfaces and baseline dependencies), system_architecture.md (component boundaries).

------------------------------
## 2. How STPA Is Used Within This SOTIF Process

ISO 21448 does not mandate a specific technique for identifying functional insufficiencies (FIs) and triggering conditions (TCs) - it defines what those things are and how they're classified, not how to go find them. STPA is used here as that identification technique, not as a substitute for SOTIF's own structure, consistent with how the literature actually combines the two: a recent SOTIF study of an MPC-based trajectory planner cites prior work applying STPA "to verify the feasibility of STPA as part of the SOTIF lifecycle," while flagging directly that "not all of them seem to be TCs in the sense of ISO 21448" [1]. STPA's outputs - unsafe control actions and loss scenarios, worked in full in stpa-full-process.md, stpa-item24-decommit-abort.md, and stpa-item31-egress-authorization.md - are a source of candidate findings here, filtered and reinterpreted against ISO 21448's own definitions before they're carried into Section 3, not relabeled one-for-one.

That filtering did real work, not just renaming. One example: the STPA screening flagged Behavior Planner's commit control action producing an unsafe outcome when it "inherits bad upstream input" (exhaustive_failures_list.md Item 20/4). Read as a functional insufficiency, that finding doesn't belong to the commit decision itself - the decision logic performs exactly as specified on the input it's given. The actual insufficiency is upstream, in the restriction-data path having no live cross-check against current ground truth (Item 1/2). So that UCA does not get its own FI entry in Section 3; it's folded in as an exposure path off FI-7 instead. A handful of STPA's screened control actions (CA-1, CA-4, CA-5, CA-7, CA-8 in stpa-full-process.md) produced no UCA at all and correspondingly contribute nothing here.

------------------------------
## 3. Functional Insufficiencies and Triggering Conditions

**FI-1 - De-commit/abort function has no duration-held, lane-clearance check on the merge target.**
* Triggering Condition: a trailing cyclist or adjacent-lane vehicle present at, or entering during, the merge maneuver.
* Triggering Condition: Perception's occlusion/clutter detection limit (exhaustive_failures_list.md Item 7) obscuring that road user from the track list the check would otherwise rely on.
* Source: stpa-item24-decommit-abort.md, UCA1-1 / UCA1-3.

**FI-2 - Egress-authorization aggregation has no staleness or accuracy bound on its ground-truth inputs.**
* Triggering Condition: Localization pose or Vehicle Controls chassis-state feedback consumed by Trip Manager after it has aged past the point it reflects current ground truth.
* Triggering Condition: severe GPS degradation and multipath in an urban canyon (Item 17) producing an on-time but inaccurate Localization reading - stale-in-accuracy, not stale-in-time.
* Triggering Condition: Localization and Vehicle Controls feedback sampled on independent cycles, read by Trip Manager as if simultaneous.
* Source: stpa-item31-egress-authorization.md, UCA2-1 / UCA2-3.

**FI-3 - Egress-authorization state has no re-evaluation trigger tied to subsequent ego-motion.**
* Triggering Condition: a repositioning micro-adjustment (rider-initiated or environment-forced) occurring after authorization is granted but before the unlock interlock consumes it.
* Source: stpa-item31-egress-authorization.md, UCA2-4.

**FI-4 - The unlock interlock has no simultaneity requirement across its three AND-gate inputs.**
* Triggering Condition: ordinary cyclic-execution jitter causing authorization, zero-speed, and clearance-signal to each be latched true at slightly different instants, with no fault present.
* Source: stpa-full-process.md, CA-9 wrong-timing finding.

**FI-5 - Commit decision has no specified minimum stopping-distance margin.**
* Triggering Condition: a valid candidate reached late enough in the approach that current speed and remaining distance leave an inadequate comfortable-stop margin.
* Source: exhaustive_failures_list.md Item 21/5; stpa-full-process.md, CA-2.

**FI-6 - Periodic re-validation cadence for a committed candidate is not bound against realistic obstruction-appearance rates.**
* Triggering Condition: a genuine obstruction (double-parked vehicle, delivery activity) appearing at a committed spot faster than the re-check interval catches it.
* Source: exhaustive_failures_list.md Item 23; stpa-item24-decommit-abort.md, UCA1-2.

**FI-7 - Restriction/legal-permissibility determination has no live cross-check against current ground truth.**
* Triggering Condition: a curb's legal status changes (new closure, lifted restriction) between the cloud prior data's last refresh and the vehicle's arrival.
* Exposure path (not a separate FI): Behavior Planner's commit decision inherits this insufficiency without introducing one of its own (Section 2 above).
* Source: exhaustive_failures_list.md Item 1/2.

------------------------------
## 4. Four-Area Scenario Classification

Area 1 (known safe) and Area 4 (unknown safe) are not enumerated here by construction - they contain scenarios where no functional insufficiency is triggered, and Area 4's contents are by definition not yet identified. Every FI/TC pair in Section 3 is, by virtue of having been identified, Area 2 (known hazardous). What distinguishes them is mitigation status, not area membership:

* **Mitigated, residual risk accepted:** none of the FI/TC pairs above currently have a purpose-built mitigation in the architecture. The one item elsewhere in this project's analysis that does - premature de-commit on a single noisy reading (Item 22), closed by the multi-cycle confirmation requirement - is a control-action-level defect already resolved by design, not one of the functional insufficiencies carried here, and is not re-listed.

* **Open, unmitigated:** FI-1 through FI-7, all seven. Each needs the requirement named in its source document (a duration-held clearance check, a staleness/accuracy bound, a re-evaluation trigger, a simultaneity requirement, a stopping-margin specification, a re-check-cadence bound, and a live restriction cross-check, respectively) before it can be argued down to accepted residual risk.

------------------------------
## 5. Area 3 - Residual Unknown-Unsafe Scenarios

This analysis does not claim to have found every functional insufficiency in this item - that is what Area 3 represents by definition, and no single analysis pass empties it. Two concrete places this pass did not reach: no full enumeration of environmental-disturbance triggering conditions was attempted beyond the urban-canyon case already surfaced for FI-2, and the STPA screening this SOTIF pass draws from covered only the control actions in stpa-full-process.md's Step 2 diagram - a different control-structure boundary (for example, one that modeled Perception's or World Model Builder's own internal control loops rather than treating them as feedback sources) could surface FIs this one structurally cannot. Closing Area 3 further is a scenario-based verification and field-monitoring activity, not another document - it is named here as an open item, not something this document resolves.

------------------------------
## References

[1] Analysis of Functional Insufficiencies and Triggering Conditions to Improve the SOTIF of an MPC-based Trajectory Planner. https://arxiv.org/html/2407.21569v1