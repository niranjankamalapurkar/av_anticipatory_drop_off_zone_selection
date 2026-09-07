This document defines the interfaces this use case owns. See system_architecture.md's "Basis of Design" section for the architectural reasoning and citations behind the overall module structure - not repeated here; this document's own References section covers only citations specific to interface payload design.

This use case - anticipatory stopping-zone selection for drop-off - does not introduce a standalone feature sitting on top of an otherwise-unchanged AV stack. Most of its content is a change in when and how existing modules act: new triggering conditions and decision logic layered onto interfaces the baseline AV stack already provides. It does add a small amount of genuinely new interface surface. Section 2 defines the owned interfaces. Section 3 lists the baseline capabilities this use case depends on without redefining.

------------------------------
## 1. Cloud Prior Layer → World Model Builder (Prior Layer: Legal Stopping Permissibility and Static Geometry)

* Protocol: Automotive Ethernet / PCIe
* Payload:
    1. timestamp
    2. stream_health_status (OK | LAG_WARNING | STREAM_LOST)
    3. tile_id, sequence_number
    4. grid_metadata: resolution_m, width_cells, height_cells, origin (lat, lon, heading) - one shared geo-reference for every channel below
    5. restriction_class_channel: row-major array, one restriction_class value per cell (PERMITTED | RESTRICTED_SAFETY_CRITICAL | RESTRICTED_REGULATORY)
    6. restricted_time_windows: sparse array[cell_index, active_time_window_start, active_time_window_end], present only for cells whose restriction_class is not PERMITTED
    7. drivable_surface_channel: row-major boolean array, one flag per cell
    8. curb_line_channel: row-major boolean array, one flag per cell
* Rationale: Feature-specific content this use case adds to the World Model's Prior layer: a time-of-day-resolved legal-stopping-permissibility and static-geometry raster. The payload is a stack of separately-aligned, single-purpose channels sharing one grid header (grid_metadata), rather than a per-cell struct carrying multiple fields - the representation Waymo's ChauffeurNet uses for this kind of static/semantic map content [1]. restriction_class distinguishes PERMITTED from two restricted tiers (SAFETY_CRITICAL, REGULATORY); candidate evaluation (operational_flow_chart.md `EvaluatingCandidate`) treats both restricted tiers identically - neither is ever a valid stop - so this tier split is currently carried in the payload without a consumer that acts on it differently (Open Item OI-9). Active time windows are a sparse list rather than a dense channel, since most cells carry no restriction.

Delivery is continuous, localized tile streaming along the route corridor while `anticipatory_search_flag == TRUE`, not a single prefetch fired at search-enablement - a fixed-area prefetch does not track a vehicle that passes candidates or enters an expanded-radius sweep (Interface 5's `expanded_search_radius_m`), so the served area has to move with the vehicle. `tile_id` and `sequence_number` identify which spatial tile and which update in sequence a payload belongs to, so World Model Builder can detect a skipped or out-of-order tile; `stream_health_status` reports the stream's own health independent of any single tile's content, so a consumer can distinguish "this tile is old" from "the stream itself has failed."

Freshness and degraded-mode behavior: World Model Builder retains the last-received tile per cell when `stream_health_status` moves off OK. `drivable_surface_channel` and `curb_line_channel` content is static geometry and remains usable from that cached tile independent of stream health. `restriction_class_channel` and `restricted_time_windows` content is time-of-day-dependent and becomes untrustworthy the longer the stream stays degraded - the exact bound is Open Item OI-1. If `stream_health_status` reaches STREAM_LOST and stays there past that bound, restriction determination for the affected area falls to the real-time classification channel added to Baseline Dependency A (Perception -> World Model Builder) rather than continuing to trust the stale cached restriction data. grid_metadata's origin is expressed in absolute (lat/lon) coordinates; aligning it with the vehicle's working frame is existing World Model Builder prior-layer fusion capability (system_architecture.md Section 3), not something this use case introduces.

------------------------------
## 2. Route Planner → Behavior Planner (Anticipatory-Search Enablement)

* Protocol: Internal IPC / Shared Memory
* Payload:
    1. timestamp
    2. anticipatory_search_flag (boolean)
* Rationale: New content this use case adds; a generic baseline Route Planner has no concept of it. Route Planner owns and sets the flag, since it already holds the position, destination, and ODD-eligibility knowledge the decision requires (see system_architecture.md Basis of Design for why Route Planner, not Behavior Planner, is the appropriate owner). No destination or tolerance value is carried in the payload - only the resulting boolean - since Behavior Planner does not need to re-derive timing Route Planner has already resolved (see decision_tree.md for the effect on candidate evaluation).

------------------------------
## 3. Behavior Planner → Trip Manager (Planner Status Code)

* Protocol: Internal IPC / Shared Memory
* Payload:
    1. timestamp
    2. status_code (CORRIDOR_EXHAUSTED | CANDIDATE_COMMITTED | CANDIDATE_ABORTED)
    3. committed_location (present only when status_code is CANDIDATE_COMMITTED)
    4. abort_reason (OBSTRUCTION_DETECTED | MANEUVER_FAILED; present only when status_code is CANDIDATE_ABORTED)
* Rationale: New content this use case adds. This is the only interface content Behavior Planner contributes upward, for any of the three mission-relevant events it can raise: the search corridor is exhausted with no candidate committed, a candidate has been committed to, or a previously committed candidate has been aborted. All three are single tactical status reports, not decisions - Behavior Planner does not negotiate with the rider, decide a resulting route, or authorize egress directly; those are mission-level decisions extending beyond a tactical module's scope (see system_architecture.md Basis of Design). CANDIDATE_ABORTED closes a gap the prior status-code set left open: if Behavior Planner de-commits after a dynamic obstruction appears at a committed spot (exhaustive_failures_list.md's de-commit findings), Trip Manager previously had no event to receive at all - it would only see a later CORRIDOR_EXHAUSTED or CANDIDATE_COMMITTED, with no record that a commitment had existed and been withdrawn in between. `abort_reason` distinguishes an obstruction found during re-validation from a maneuver-execution failure, since Trip Manager's response may differ. Trip Manager does not act on CANDIDATE_COMMITTED alone: it independently confirms the vehicle has actually reached and stopped at `committed_location` using Localization (Baseline Dependency P) and Vehicle Controls' chassis state (Baseline Dependency Q) before issuing egress authorization - the same don't-trust-the-claim principle already applied to Vehicle Controls' own door-unlock interlock (FI-4 in sotif_analysis.md and the CA-9 finding in stpa_full_process.md), applied one layer up. CORRIDOR_EXHAUSTED drives Interface 4/5's negotiation and routing path; CANDIDATE_ABORTED returns Behavior Planner to active search without engaging that path, unless repeated aborts eventually lead to corridor exhaustion on their own.

------------------------------
## 4. Trip Manager ↔ Rider HMI (Search-Preference Negotiation)

* Protocol: Internal IPC / Shared Memory
* Payload (Trip Manager → Rider HMI):
    1. timestamp
    2. search_status (CORRIDOR_EXHAUSTED)
    3. retry_attempts_remaining (integer)
    4. proposed_expansion_radius_m
    5. fallback_hub_distance_m
* Payload (Rider HMI → Trip Manager):
    1. timestamp
    2. rider_preference_selection (EXPAND_SEARCH | NAVIGATE_TO_HUB)
* Rationale: New content this use case adds, layered onto Trip Manager's existing rider-negotiation role (Section 4 of system_architecture.md's Basis of Design) rather than opening a new class of exchange - this is feature-specific negotiation content on a baseline capability, not a new communication channel between Trip Manager and the rider. `AwaitingRiderPreference` (operational_flow_chart.md) pauses the search and requires explicit rider input before Trip Manager issues a routing directive. `proposed_expansion_radius_m` is included so the rider's EXPAND_SEARCH choice is made against a concrete distance (e.g., "expand search up to 150m walking distance?") rather than an open-ended commitment - the same value Trip Manager subsequently carries to Route Planner as Interface 5's `expanded_search_radius_m` if the rider agrees, so the radius the rider was shown is the radius that is actually searched. `fallback_hub_distance_m` is included so the rider's choice is informed by cost, not just availability. **This exchange is distinct from Baseline Dependency M's arrival-imminent notification, which is also Trip-Manager-owned but carries no decision payload and expects no reply.** See Open Items for the default action if no reply is received within the response window.

------------------------------
## 5. Trip Manager → Route Planner (Search-Escalation Routing Directive)

* Protocol: Internal IPC / Shared Memory
* Payload:
    1. timestamp
    2. route_type (SWEEP | HUB_DIRECT)
    3. expanded_search_radius_m (present only when route_type is SWEEP)
    4. target_location (present only when route_type is HUB_DIRECT)
* Rationale: New content this use case adds. `route_type` distinguishes two cases Route Planner's baseline routing API does not already express: SWEEP requests an alternate route to the unchanged destination that avoids previously-scanned street segments - not itself a destination change - while HUB_DIRECT is a genuine destination change to the resolved fallback-hub coordinate. `expanded_search_radius_m` carries the walking-distance tolerance the rider agreed to via Interface 4's `proposed_expansion_radius_m` - without it, Route Planner has no way to know how far to extend the sweep route, and Behavior Planner has no way to know the expanded tolerance to evaluate candidates against once the new route is in effect (Interface 2's `max_walking_distance_m` is updated to this value at that point). Trip Manager issues this directive once the rider responds via Interface 4 (or the default-action timeout fires); Route Planner reports the resulting route to Behavior Planner through Baseline Dependency F, the same mechanism that carries every other route update - no separate interface back to Behavior Planner is needed, since a post-negotiation reroute is not distinguishable from any other route change at Behavior Planner's consuming end.

------------------------------
## Baseline Dependencies (not defined by this use case)

**A. Perception → World Model Builder (Live Occupancy Layer + Real-Time Restriction Classification)**
* Payload: timestamp, occupancy grid layer, at minimum. This use case adds a channel to this same interface: restriction_class_estimate (PERMITTED | RESTRICTED | UNKNOWN), confidence (0.0-1.0), source_type (SIGN_DETECTION | CURB_PAINT_CLASSIFIER).
* Rationale: Generic live occupancy needed for any driving behavior; not specific to this use case. This use case layers a new channel onto this same interface rather than opening a separate one: restriction_class_estimate/confidence/source_type, populated only when Interface 1's `stream_health_status` has been LAG_WARNING or STREAM_LOST for longer than the bound in Open Item OI-1. World Model Builder substitutes this live classification for the affected cells' `restriction_class_channel` value in place of the now-untrustworthy cached cloud-prior data. Real-time detection of parking-restriction signage and curb-paint markings from a moving vehicle is demonstrated capability at production-relevant frame rates [5]. Full extraction of structured restriction semantics - specific time windows, permit exceptions - from sign text is explicitly flagged as future work by that same source, not a solved, integrated capability, so restriction_class_estimate is a coarse classification rather than a reconstructed time window: this channel can tell World Model Builder a restriction sign or red/yellow curb marking is present, not reliably what hours it applies. UNKNOWN (occluded sign, ambiguous curb color, low confidence) and RESTRICTED are both treated identically by candidate evaluation today, consistent with Interface 1's existing SAFETY_CRITICAL/REGULATORY tier collapse (Open Item OI-9) - an unconfirmed restriction is not a valid stop. This is the fallback tier between Interface 1's cached static geometry (which remains usable during stream loss) and the conservative default of treating an unresolved candidate as not permitted (decision_tree.md node B) when neither the cloud stream nor this channel can confirm legal permissibility.

**B. Perception → Prediction (Tracked Dynamic Objects)**
* Payload: tracked object list (identity, position, velocity), at minimum
* Rationale: Detection and tracking are baseline Perception functions (system_architecture.md Section 4). This use case does not perform tracking.

**C. Prediction → World Model Builder (Predicted Trajectories)**
* Payload: predicted trajectory per tracked object, at minimum
* Rationale: Trajectory forecasting is a distinct baseline module (system_architecture.md Section 4). This use case consumes it via the fused World Model output; it does not produce it.

**D. World Model Builder → Behavior Planner (Fused World Representation)**
* Payload: fused grid (drivable surface, live occupancy) and predicted-trajectory-annotated track list, at minimum
* Rationale: The general fused feed every Behavior Planner function consumes. This use case adds Interface 1's content as one layer within this feed; it does not redefine the feed itself.

**E. World Model Builder → Motion / Path Planner (Fused World Representation)**
* Payload: same fused output as Baseline Dependency D
* Rationale: Motion Planner's trajectory-cost input, consumed independently of Behavior Planner's use of the same feed.

**F. Route Planner → Behavior Planner (Route Progress and Path)**
* Payload: current planned route/path to destination, and remaining distance along it, at minimum
* Rationale: Generic, continuously-updated route output every Behavior Planner and Motion Planner function needs. Route Planner continues publishing the existing forward path once the vehicle passes the destination - the road ahead remains a legal path already being driven - rather than recomputing immediately. Route Planner receives a baseline trip-lifecycle "trip complete" signal, independent of this use case, used to distinguish an exhausted search from ordinary trip completion. Once Interface 5 delivers Trip Manager's routing directive, the resulting route reaches Behavior Planner through this same interface - no separate directive-carrying channel back to Behavior Planner is needed (see system_architecture.md Basis of Design, Item BD-3).

**G. Localization → Behavior Planner (Ego Vehicle State)**
* Payload: position, heading, speed, at minimum
* Rationale: Every Behavior Planner function needs ego state. This use case assumes the signal carries a validity/health indicator consistent with its safety relevance; not confirmed against baseline Localization's actual output (Open Item OI-2).

**H. Trip Manager → Vehicle Controls (Egress Authorization)**
* Payload: egress-ready status, at minimum
* Rationale: Baseline arrived-and-stopped signaling, required for any drop-off, issued once Trip Manager has both Behavior Planner's CANDIDATE_COMMITTED status code (Interface 3) and independent ground-truth confirmation from Localization and Vehicle Controls (Baseline Dependencies P, Q) that the vehicle has actually stopped at that location. This use case relies on it firing at whatever location the search committed to, which may not be the original requested coordinate. Sourced from Trip Manager rather than Behavior Planner directly, consistent with Trip Manager owning mission-level authorization (system_architecture.md Basis of Design). Distinct from Baseline Dependency R below - this authorizes the door/egress domain specifically, not the driving-actuation output Vehicle Controls also receives from Motion Planner.

**I. Rider HMI → Trip Manager (Rider-Initiated Unlock Request)**
* Payload: unlock request, at minimum
* Rationale: Baseline egress interaction, independent of search logic. Reaches Trip Manager first, which validates it against mission state before any authorization reaches Vehicle Controls, rather than a UI action reaching an actuation-authority module directly. Rider-initiated versus autonomous unlock is a baseline vehicle-platform decision this use case does not own (Open Item OI-3).

**J. Door-Clearance Sensor → Vehicle Controls (Road-User Clearance Signal)**
* Payload: clearance signal, at minimum
* Rationale: Baseline safety interlock (system_architecture.md Section 4). The same interlock applies regardless of where the vehicle stopped; this use case does not add a location-specific variant.

**K. Vehicle Controls → Door Actuation (Unlock / Lock Command)**
* Payload: unlock / lock command, at minimum
* Rationale: Baseline actuation, gated by Vehicle Controls' own safety logic, independent of this use case's existence.

**L. Behavior Planner → Motion / Path Planner (Stop-Fence / Target Specification)**
* Payload: reference line or path segment with an embedded stop constraint (position and/or pose) when a stop is intended; absence of an active stop constraint otherwise, at minimum
* Rationale: A pre-existing AV-stack mechanism this use case relies on rather than defines - Behavior Planner expresses a stop or hold decision as a constraint on the reference line, not a symbolic command [2][3] (see system_architecture.md Basis of Design). Assumes the mechanism supports being set, cleared, and re-set within a single trip segment (Open Item OI-7).

**M. Trip Manager → Rider HMI (Arrival-Imminent / Prepare-to-Exit Notification)**
* Payload: baseline arrival-imminent / prepare-to-exit notification, at minimum
* Rationale: Every drop-off needs this baseline notification. Sourced from Trip Manager - Trip Manager is the single source of truth for rider-facing UI state, and this notification fires from the same ground-truth-confirmed arrival that drives Baseline Dependency H, just relayed to the rider instead of Vehicle Controls. This use case only changes when the notification fires - including after the vehicle commits to a candidate reached via corridor expansion or a fallback hub - not what it contains, and does not introduce an urgency-differentiated variant.

**N. Cloud Routing & POI Service → Trip Manager (Fallback Hub / POI Lookup)**
* Payload: designated fallback hub locations (position, capacity class), at minimum
* Rationale: Trip Manager is the consumer of fallback hub location data - resolving a location for a triggered task and handing it to Route Planner is Trip Manager's existing role (system_architecture.md Basis of Design, citing Apollo's `task_manager` parking-routing precedent). This is treated as a baseline capability of Trip Manager's existing POI-lookup role, not new interface content this use case defines.

**O. Trip Manager → Route Planner (Mission Goal / Destination Assignment)**
* Payload: destination or waypoint update, at minimum
* Rationale: Baseline capability this use case relies on rather than defines - Trip Manager already has some path for assigning or changing Route Planner's destination independent of this feature (e.g., trip dispatch at pickup, or an ordinary mid-trip destination change), consistent with Apollo's `task_manager` issuing routing requests to the Routing module for other triggered tasks, and with reference architectures where the routing-level module takes goal input from a layer above it rather than deriving it itself [4]. Interface 5's `route_type`/`target_location` directive is feature-specific content layered onto this same Trip-Manager-to-Route-Planner path, not a new communication channel; the exact baseline mechanism (structured API call versus a generic destination field) is not confirmed (Open Item OI-10).

**P. Localization → Trip Manager (Ego Pose vs. Goal)**
* Payload: ego pose relative to the current mission goal, at minimum
* Rationale: Baseline capability this use case relies on rather than defines - Trip Manager already needs ego-position tracking against its current goal for generic mission-lifecycle purposes (arrival detection, ETA, ordinary trip completion) independent of this feature. This use case relies on it as one of two ground-truth inputs (with Baseline Dependency Q) that let Trip Manager independently confirm arrival at Interface 3's `committed_location` rather than trusting Behavior Planner's CANDIDATE_COMMITTED status code alone (system_architecture.md Basis of Design).

**Q. Vehicle Controls → Trip Manager (Chassis State)**
* Payload: speed, gear, and door/lock status, continuously, at minimum
* Rationale: Baseline capability this use case relies on rather than defines - Vehicle Controls already reports chassis state to Trip Manager for generic mission-lifecycle tracking (e.g., confirming the vehicle is actually stopped before ending a trip), independent of this feature. This use case relies on it as Trip Manager's second ground-truth input for arrival confirmation (with Baseline Dependency P) and, once door/lock status also confirms secured, for determining drop-off conclusion - `Evt_DropOffConcluded` (operational_flow_chart.md) requires door-secured and clearance-envelope-clear facts that speed and gear alone do not supply, so door/lock status is assumed to be part of the same continuous chassis-state feed rather than a separate interface (system_architecture.md BD-4). Vehicle Controls' own zero-speed confirmation for the door-unlock interlock, separately, is sourced from its own independent vehicle-state input, not Localization - required so a shared-pipeline fault cannot compromise both the tactical decision and that safety interlock (FI-4 in sotif_analysis.md and the CA-9 finding in stpa_full_process.md).

**R. Motion / Path Planner → Vehicle Controls (Actuator Commands)**
* Payload: steering, brake, and throttle commands, at minimum
* Rationale: Baseline AV-stack mechanism this use case relies on rather than defines - the execution leg of the `BP --> MP --> VC` chain, closing the path from Behavior Planner's stop-fence constraint (Baseline Dependency L) to the low-level driving actuation that carries it out (system_architecture.md Basis of Design). Trajectory-generation logic itself (how Motion Planner turns a reference-line constraint into actuator commands) is not this use case's content and is not redefined here - only the fact that the output feeds Vehicle Controls is modeled.

------------------------------
## Open Items

* OI-1 - Staleness/lag bound for Interface 1's streamed restriction data - how long `stream_health_status` may remain LAG_WARNING or STREAM_LOST before cached `restriction_class_channel` content is no longer trusted and Baseline Dependency A's real-time classification channel (or the conservative default) takes over.
* OI-2 - Whether baseline Localization (Baseline Dependency G) provides a validity/health signal this use case's kinematic check can rely on; assumed present, not confirmed.
* OI-3 - Rider-initiated versus autonomous door unlock (Baseline Dependency I) - a baseline vehicle-platform decision, referenced here because it determines whether Baseline Dependency M's notification content needs to say anything different.
* OI-4 - Time horizon for the door-clearance interlock (Baseline Dependency J, K).
* OI-5 - Scope of rider notification during the nominal (no-warning) case - referenced in the Concept of Operations.
* OI-6 - Failure mode when Interface 1's stream has not yet delivered any tile for the vehicle's current position by the time the anticipatory-search trigger is reached (a stream not yet established, as distinct from an established stream degrading later, covered by OI-1).
* OI-7 - Whether the baseline stop-constraint mechanism (Baseline Dependency L) supports set-clear-reset within a single trip segment, which this use case's de-commit behavior requires and ordinary drop-off does not exercise.
* OI-8 - Default action and response-window duration if the rider does not reply to the Interface 4 preference prompt (e.g., a fixed timeout defaulting to EXPAND_SEARCH or NAVIGATE_TO_HUB) - not yet specified.
* OI-9 - Whether Interface 1's SAFETY_CRITICAL / REGULATORY restriction-tier split still earns its place in the payload now that legal relaxation is rejected and both tiers are treated identically by candidate evaluation, or whether it should collapse to a single RESTRICTED value pending a consumer that needs the distinction.
* OI-10 - Whether Baseline Dependency O (Trip Manager → Route Planner destination/goal assignment) exists in the baseline AV stack as a structured API or only as an implicit assumption; Interface 5 assumes some such path exists but does not confirm its actual form.
* OI-11 - Whether Baseline Dependency Q's chassis-state feed (Vehicle Controls → Trip Manager) already includes door/lock status alongside speed and gear, or whether that is a separate baseline signal this use case is assuming gets folded in; `Evt_DropOffConcluded` (operational_flow_chart.md) needs door/lock status specifically, and speed/gear alone do not supply it (system_architecture.md BD-4).
* OI-12 - Hysteresis/confirmation duration before World Model Builder demotes from Interface 1's stream to Baseline Dependency A's real-time classification channel, and before further demoting to the conservative default, so a brief latency spike does not cause unnecessary tier-switching mid-search.

------------------------------
## References

[1] M. Bansal, A. Krizhevsky, and A. Ogale, "ChauffeurNet: Learning to Drive by Imitating the Best and Synthesizing the Worst," arXiv:1812.03079, 2018.

[2] H. Fan, F. Zhu, C. Liu, L. Zhang, L. Zhuang, D. Li, W. Zhu, J. Hu, H. Li, and Q. Kong, "Baidu Apollo EM Motion Planner," arXiv:1807.08048, 2018.

[3] J. Wei, J. M. Snider, T. Gu, J. M. Dolan, and B. Litkouhi, "A Behavioral Planning Framework for Autonomous Driving," in Proc. IEEE Intelligent Vehicles Symposium (IV), 2014.

[4] Autoware Foundation, "Planning Component Design," Autoware Documentation, autowarefoundation/autoware-documentation.

[5] H. Chau, Y. Jin, J. Li, J. Hu, and W. Cheng, "Real-Time Street Parking Sign Detection and Recognition," IJCAI 2022 Workshop on AI for Autonomous Driving.