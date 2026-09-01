## Exhaustive failures list

This document enumerates failure modes across the use case's block diagram and interfaces, identified first and classified second, to avoid the tunnel vision of searching only for hazards that fit one framework in advance. Category assigns each item to the analysis discipline it belongs to: SOTIF (ISO 21448) for failures where every component performs to its specification and the harm arises from a limitation or gap in the intended function itself; HARA (ISO 26262) for failures where some hardware, software, or data element stops performing its specified function - a fault, whether physical, systematic, or an integrity fault on an interface. 

Each item also carries a Disposition: whether the failure is new content this use case introduces, or a Pre-existing failure mode of a baseline module this use case carries forward unchanged, and whether the architecture already provides a mitigating or redundant path. A Pre-existing failure mode inherited from a baseline module's own analysis does not need to be re-derived here; it is retained in the list so the full picture stays visible in one place, with the inheritance stated rather than left implicit. Where a failure's classification genuinely depends on its root cause, it is split into separate items - one per causal path - rather than given a single ambiguous category.

------------------------------
## Assumptions
 
1. Cloud data reaching the Prior layer is corrupted or spoofed in transit or at rest.
   * Category: HARA
   * Rationale: Data reaching a safety-relevant function incorrect relative to spec - an external malfunction.
   * Disposition: New payload (Interface 1's restriction/geometry content) riding on a **Pre-existing** baseline assumption. Per the Assumptions section above, data integrity and authenticity validation at the point cloud data enters the vehicle is assumed to already exist as baseline connectivity infrastructure, the same assumption carried in the pothole-detection project. This use case adds no new requirement here and relies on that checking function being in place rather than introducing or re-verifying it.

2. Restriction data served to the Prior layer is stale - a new closure, restriction change, or lifted restriction not yet reflected.
   * Category: SOTIF
   * Rationale: The Cloud Prior Layer serves exactly the data it holds, functioning to spec; the insufficiency is the system's reliance on non-real-time data as if current - a known, foreseeable performance limitation, not a malfunction. This is distinct from Items 5 and 6 below: staleness is an inherent latency property of any prefetch-based system, not a cloud-backend content error, so it remains a vehicle-side SOTIF item rather than an out-of-scope backend issue.
   * Disposition: Behavior Planner's decision tree includes a live-perception recheck (node G, decision_tree.md) that measures actual clear length immediately before commit, independent of the cloud prior data - this catches a stale entry that misstates *physical* clearance. Real-time detection and localization of street parking-restriction signage from a moving vehicle is a demonstrated capability in current perception research - for example, Chau, Jin, Li, Hu, and Cheng demonstrate parking-sign and parking-symbol detection at 163 and 88 frames per second respectively using YOLOv5, at production-relevant accuracy (IJCAI 2022 AI for Autonomous Driving Workshop) - so a live cross-check against legal permissibility is not physically out of reach. What is not yet demonstrated at the same maturity, per that same paper, is full extraction of structured restriction semantics (time windows, permit exceptions) from sign text and delivery of that as an actionable signal to a planner - the authors explicitly flag that step as future work, not a solved, integrated capability. Concretely for this project: Perception is not currently wired into World Model Builder's restriction_class_channel at all (icd.md Interface 1 sources that channel from the Cloud Prior Layer only), so today this remains unmitigated in the deployed architecture regardless of what perception hardware is technically capable of. Whether to add a live sign/curb-marking detection path as a redundant input to restriction determination is a genuine architectural option worth deciding on explicitly, not a closed question - see Items Not Yet Classifiable.

3. Restriction-class prefetch fails to complete before the anticipatory-search trigger is reached, due to a network or hardware fault interrupting the transfer.
   * Category: HARA
   * Rationale: An external communication path fails to perform its specified function.
   * Disposition: New, and currently unmitigated. icd.md Open Item OI-1 leaves both the staleness bound and the resulting degraded-mode behavior undefined - there is no stated fallback for a total prefetch failure. This differs from Item 2: node G's live recheck only ever validates physical clearance, and a total prefetch failure removes restriction-class data entirely, not just ages it.

4. Restriction-class prefetch fails to complete because the vehicle is operating in a low-connectivity pocket of the ODD, a foreseeable condition in dense urban cores.
   * Category: SOTIF
   * Rationale: No component has failed; the system was never robust to a known-possible connectivity gap within its own operational design domain.
   * Disposition: New, and unmitigated for the same reason as Item 3 - tied to the same undefined degraded-mode behavior in OI-1.

5. Source restriction data is itself incorrect at authoring time - for example, a safety-critical restriction mislabeled as regulatory-tier.
   * Category: **Out of Scope - External (Cloud Backend)**
   * Rationale: This is a defect in the cloud backend's own map-authoring and data-quality process, not a malfunction or performance limitation of any vehicle-side or transport component - every vehicle-side element applies the data exactly as delivered. Consistent with the Assumptions section above and with the pothole-detection project's own scoping of its cloud infrastructure as external, this is not classified as a vehicle-side SOTIF item; it is the cloud backend's responsibility to resolve through its own authoring and QA controls.
   * Disposition: Not this use case's content to mitigate. Retained in this list for completeness, since the vehicle-side consequence (a wrong commit decision) is real even though the fix lives outside this use case's boundary. To the extent a live perception-based cross-check exists (see Item 2's revised disposition), it provides the same partial backstop here that it does for staleness, but that is an architectural option, not something already in place.

6. Fallback-hub location data resolves to a hub that no longer exists or is inaccessible.
   * Category: **Out of Scope - External (Cloud Backend)**
   * Rationale: A stale or incorrect entry in the cloud backend's fallback-hub database, not a vehicle-side malfunction or performance limitation. Same scoping basis as Item 5.
   * Disposition: Not this use case's content to mitigate. Its consequence is mission delay rather than physical harm, and Trip Manager's rider-negotiation flow provides an operational recovery path (the rider can be re-prompted) independent of whether the backend data is ever corrected.

------------------------------
## Perception

7. Perception fails to detect a pedestrian or obstruction under nominal operation due to occlusion, clutter, or adverse lighting.
   * Category: SOTIF
   * Rationale: The detector performs to its specification; the scenario exceeds its known performance envelope - the canonical SOTIF case.
   * Disposition: **Pre-existing** - inherited from baseline Perception's own analysis, not introduced by this use case. This use case is a new consumer of Perception's existing output (Behavior Planner's candidate evaluation) but changes neither Perception's detection performance nor introduces a new failure mode within it. The dense-urban-curb operating context this use case exercises is within Perception's already-known SOTIF envelope, not a new triggering condition, though it plausibly increases the *frequency* that envelope is exercised relative to ordinary driving - an exposure consideration, not a new failure mode.

8. A perception sensor channel freezes or stops returning data with no fault flag raised.
   * Category: HARA
   * Rationale: A physical sensing element stops performing its specified function without the system being informed.
   * Disposition: **Pre-existing**, inherited unchanged from baseline Perception. This use case adds no new consumer-specific mitigation and relies on whatever fault-detection Perception's baseline item already has - not re-derived here.

9. Perception misclassifies a tracked object - for example, a pedestrian classified as a static object and dropped from dynamic tracking - while the algorithm runs as designed.
   * Category: SOTIF
   * Rationale: Correct execution of a specified algorithm producing an insufficient result in a hard scenario.
   * Disposition: **Pre-existing**, inherited from baseline Perception, same reasoning as Item 7.

10. Perception's tracker swaps identity between two closely-spaced pedestrians in a dense curb-side cluster.
    * Category: SOTIF
    * Rationale: A known algorithmic limitation of multi-object tracking under high object density, not a component fault.
    * Disposition: **Pre-existing**, inherited; same exposure caveat as Item 7 - curb-side pedestrian clustering at drop-off plausibly increases how often this known limitation is exercised, without being a new failure mode.

------------------------------
## Prediction

11. Prediction forecasts an incorrect or late trajectory for a pedestrian who changes direction unpredictably, while the module runs as designed.
    * Category: SOTIF
    * Rationale: Trajectory forecasting is inherently probabilistic; an unpredictable but foreseeable human motion exceeding the model's performance is a SOTIF case, not a malfunction.
    * Disposition: **Pre-existing**, inherited from baseline Prediction. This use case is a new consumer of Prediction's existing output via World Model Builder, not a new failure mode within Prediction itself.

12. Prediction's output is delayed past its latency budget under peak tracked-object load, traced to a software defect rather than a load condition the budget already accounts for.
    * Category: HARA
    * Rationale: A systematic software fault causing the module to violate its own specified timing.
    * Disposition: **Pre-existing**, inherited; not reintroduced or worsened by this use case beyond being one more consumer of the same output.

13. Prediction's process crashes or stops publishing output.
    * Category: HARA
    * Rationale: Complete loss of a specified function due to a software fault.
    * Disposition: **Pre-existing**, inherited, same reasoning as Item 12.

------------------------------
## World Model Builder

14. World Model Builder misaligns the static prior grid and live occupancy layers due to a coordinate-frame transformation defect.
    * Category: HARA
    * Rationale: A systematic software fault in a specified fusion function.
    * Disposition: **Pre-existing** World Model Builder fusion-mechanism risk, inherited. Interface 1's restriction-class channel shares World Model Builder's existing grid_metadata/origin handling rather than introducing a separate fusion pathway (icd.md Interface 1), so this defect class is not newly introduced by this use case, though the new channel is one more thing affected if it occurs.

15. World Model Builder hangs and continues republishing its last fused frame, with no freshness or health indicator available downstream.
    * Category: HARA
    * Rationale: A malfunction - the module stops updating - compounded by the absence of any downstream mechanism to detect it; the detectability gap is a finding in its own right, independent of this classification.
    * Disposition: **Pre-existing** World Model Builder gap, inherited; not introduced or worsened specifically by this use case, but this use case's restriction-class content rides the same unaddressed gap as every other layer World Model Builder fuses.

16. World Model Builder's grid resolution cannot resolve a small protruding obstruction, such as a stroller wheel, even with every input correct.
    * Category: SOTIF
    * Rationale: The representation performs to its specified resolution; the limitation is inherent to that specification, not a fault.
    * Disposition: **Pre-existing** World Model Builder representation limitation, inherited unchanged.

------------------------------
## Localization

17. Localization's position estimate drifts under GPS-denied conditions in an urban canyon, a known condition within the ODD.
    * Category: SOTIF
    * Rationale: The localization stack performs to its specified accuracy envelope; degraded accuracy in a foreseeable urban-canyon condition is a known performance limitation.
    * Disposition: **Pre-existing** Localization limitation, inherited - but with a new consequence path. This use case adds a new safety-relevant consumer, Trip Manager's ground-truth arrival confirmation (Baseline Dependency P), which did not previously exist and now depends on Localization's accuracy for an egress-authorization decision. The failure mode is not new; what it can now affect is.

18. A localization sensor (GNSS receiver, IMU) hard-faults with no fault flag raised.
    * Category: HARA
    * Rationale: A physical sensing element stops performing its specified function undetected.
    * Disposition: **Pre-existing**, inherited; same new-consequence-path caveat as Item 17.

19. Localization's pose output freezes due to a process fault, with no validity or health signal available to downstream consumers to detect it.
    * Category: HARA
    * Rationale: A malfunction compounded by the same diagnosability gap as Item 15 - neither of the system's two primary fusion-input feeds currently carries a confirmed health signal.
    * Disposition: **Pre-existing** gap (also flagged as icd.md Open Item OI-2), inherited - but now more consequential than before this use case existed, since Trip Manager's egress authorization is a new safety-relevant decision resting on this same unconfirmed-health feed.
------------------------------
## Route Planner


------------------------------
## References
 
[1] H. Chau, Y. Jin, J. Li, J. Hu, and W. Cheng, "Real-Time Street Parking Sign Detection and Recognition," IJCAI 2022 Workshop on AI for Autonomous Driving.