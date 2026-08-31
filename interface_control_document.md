This document defines the interfaces this use case owns. See block-diagram.md's "Basis of Design" section for the architectural reasoning and citations behind the overall module structure - not repeated here; this document's own References section covers only citations specific to interface payload design.

This use case - anticipatory stopping-zone selection for drop-off - does not introduce a standalone feature sitting on top of an otherwise-unchanged AV stack. Most of its content is a change in when and how existing modules act: new triggering conditions and decision logic layered onto interfaces the baseline AV stack already provides. It does add a small amount of genuinely new interface surface. Section 2 defines the owned interfaces. Section 3 lists the baseline capabilities this use case depends on without redefining. This framing belongs in the project's ConOps as well; it is not currently stated there.

------------------------------
## 1. Cloud Prior Layer → World Model Builder (Prior Layer: Legal Stopping Permissibility and Static Geometry)

* Protocol: Automotive Ethernet / PCIe
* Payload:
    1. timestamp
    2. grid_metadata: resolution_m, width_cells, height_cells, origin (lat, lon, heading) - one shared geo-reference for every channel below
    3. restriction_class_channel: row-major array, one restriction_class value per cell (PERMITTED | RESTRICTED_SAFETY_CRITICAL | RESTRICTED_REGULATORY)
    4. restricted_time_windows: sparse array[cell_index, active_time_window_start, active_time_window_end], present only for cells whose restriction_class is not PERMITTED
    5. drivable_surface_channel: row-major boolean array, one flag per cell
    6. curb_line_channel: row-major boolean array, one flag per cell
* Rationale: Feature-specific content this use case adds to the World Model's Prior layer: a time-of-day-resolved legal-stopping-permissibility and static-geometry raster. The payload is a stack of separately-aligned, single-purpose channels sharing one grid header (grid_metadata), rather than a per-cell struct carrying multiple fields - the representation Waymo's ChauffeurNet uses for this kind of static/semantic map content [1]. restriction_class distinguishes PERMITTED from two restricted tiers (SAFETY_CRITICAL, REGULATORY); candidate evaluation (state_chart.md `EvaluatingCandidate`) treats both restricted tiers identically - neither is ever a valid stop - so this tier split is currently carried in the payload without a consumer that acts on it differently (Open Item OI-10). Active time windows are a sparse list rather than a dense channel, since most cells carry no restriction. Prefetch is triggered by Interface 2's search-enablement decision, not queried live during search - an assumption about acceptable connectivity risk in dense urban cores (project ODD), not verified here. Freshness: no staleness bound is defined yet (Open Item OI-1). grid_metadata's origin is expressed in absolute (lat/lon) coordinates; aligning it with the vehicle's working frame is existing World Model Builder prior-layer fusion capability (block-diagram.md Section 3), not something this use case introduces.

------------------------------
## 2. Route Planner → Behavior Planner (Anticipatory-Search Enablement)

* Protocol: Internal IPC / Shared Memory
* Payload:
    1. timestamp
    2. anticipatory_search_flag (boolean)
* Rationale: New content this use case adds; a generic baseline Route Planner has no concept of it. Route Planner owns and sets the flag, since it already holds the position, destination, and ODD-eligibility knowledge the decision requires (see block-diagram.md Basis of Design for why Route Planner, not Behavior Planner, is the appropriate owner). No destination or tolerance value is carried in the payload - only the resulting boolean - since Behavior Planner does not need to re-derive timing Route Planner has already resolved (see decision-tree-and-se-roadmap.md for the effect on candidate evaluation).

------------------------------
## 3. Behavior Planner → Trip Manager (Planner Status Code)

* Protocol: Internal IPC / Shared Memory
* Payload:
    1. timestamp
    2. status_code (CORRIDOR_EXHAUSTED | CANDIDATE_COMMITTED)
    3. committed_location (present only when status_code is CANDIDATE_COMMITTED)
* Rationale: New content this use case adds. This is the only interface content Behavior Planner contributes upward, for either of the two mission-relevant events it can raise: the search corridor is exhausted with no candidate committed, or a candidate has been committed to. Both are single tactical status reports, not decisions - Behavior Planner does not negotiate with the rider, decide a resulting route, or authorize egress directly; those are mission-level decisions extending beyond a tactical module's scope (see block-diagram.md Basis of Design). Trip Manager does not act on CANDIDATE_COMMITTED alone: it independently confirms the vehicle has actually reached and stopped at `committed_location` using Localization (Baseline Dependency P) and Vehicle Controls' chassis state (Baseline Dependency Q) before issuing egress authorization - the same don't-trust-the-claim principle already applied to Vehicle Controls' own door-unlock interlock (SC6, sotif-stpa.md), applied one layer up. CORRIDOR_EXHAUSTED drives Interface 4/5's negotiation and routing path.

------------------------------
## 4. Trip Manager ↔ Rider HMI (Search-Preference Negotiation)

* Protocol: Internal IPC / Shared Memory
* Payload (Trip Manager → Rider HMI):
    1. timestamp
    2. search_status (CORRIDOR_EXHAUSTED)
    3. retry_attempts_remaining (integer)
    4. fallback_hub_distance_m
* Payload (Rider HMI → Trip Manager):
    1. timestamp
    2. rider_preference_selection (EXPAND_SEARCH | NAVIGATE_TO_HUB)
* Rationale: New content this use case adds, layered onto Trip Manager's existing rider-negotiation role (Section 4 of block-diagram.md's Basis of Design) rather than opening a new class of exchange - this is feature-specific negotiation content on a baseline capability, not a new communication channel between Trip Manager and the rider. `AwaitingRiderPreference` (state_chart.md) pauses the search and requires explicit rider input before Trip Manager issues a routing directive. `fallback_hub_distance_m` is included so the rider's choice is informed by cost, not just availability. This exchange is distinct from Baseline Dependency M's arrival-imminent notification, which is also Trip-Manager-owned but carries no decision payload and expects no reply. See Open Items for the default action if no reply is received within the response window.

------------------------------
## 5. Trip Manager → Route Planner (Search-Escalation Routing Directive)

* Protocol: Internal IPC / Shared Memory
* Payload:
    1. timestamp
    2. route_type (SWEEP | HUB_DIRECT)
    3. target_location (present only when route_type is HUB_DIRECT)
* Rationale: New content this use case adds. `route_type` distinguishes two cases Route Planner's baseline routing API does not already express: SWEEP requests an alternate route to the unchanged destination that avoids previously-scanned street segments - not itself a destination change - while HUB_DIRECT is a genuine destination change to the resolved fallback-hub coordinate. Trip Manager issues this directive once the rider responds via Interface 4 (or the default-action timeout fires); Route Planner reports the resulting route to Behavior Planner through Baseline Dependency F, the same mechanism that carries every other route update - no separate interface back to Behavior Planner is needed, since a post-negotiation reroute is not distinguishable from any other route change at Behavior Planner's consuming end.

------------------------------
## Baseline Dependencies (not defined by this use case)

**A. Perception → World Model Builder (Live Occupancy Layer)**
* Payload: timestamp, occupancy grid layer, at minimum
* Rationale: Generic live occupancy needed for any driving behavior; not specific to this use case.

**B. Perception → Prediction (Tracked Dynamic Objects)**
* Payload: tracked object list (identity, position, velocity), at minimum
* Rationale: Detection and tracking are baseline Perception functions (block-diagram.md Section 4). This use case does not perform tracking.

**C. Prediction → World Model Builder (Predicted Trajectories)**
* Payload: predicted trajectory per tracked object, at minimum
* Rationale: Trajectory forecasting is a distinct baseline module (block-diagram.md Section 4). This use case consumes it via the fused World Model output; it does not produce it.

**D. World Model Builder → Behavior Planner (Fused World Representation)**
* Payload: fused grid (drivable surface, live occupancy) and predicted-trajectory-annotated track list, at minimum
* Rationale: The general fused feed every Behavior Planner function consumes. This use case adds Interface 1's content as one layer within this feed; it does not redefine the feed itself.

**E. World Model Builder → Motion / Path Planner (Fused World Representation)**
* Payload: same fused output as Baseline Dependency D
* Rationale: Motion Planner's trajectory-cost input, consumed independently of Behavior Planner's use of the same feed.

**F. Route Planner → Behavior Planner (Route Progress and Path)**
* Payload: current planned route/path to destination, and remaining distance along it, at minimum
* Rationale: Generic, continuously-updated route output every Behavior Planner and Motion Planner function needs. Route Planner continues publishing the existing forward path once the vehicle passes the destination - the road ahead remains a legal path already being driven - rather than recomputing immediately. Route Planner receives a baseline trip-lifecycle "trip complete" signal, independent of this use case, used to distinguish an exhausted search from ordinary trip completion. Once Interface 5 delivers Trip Manager's routing directive, the resulting route reaches Behavior Planner through this same interface - no separate directive-carrying channel back to Behavior Planner is needed (see block-diagram.md Basis of Design, Item BD-3).

**G. Localization → Behavior Planner (Ego Vehicle State)**
* Payload: position, heading, speed, at minimum
* Rationale: Every Behavior Planner function needs ego state. This use case assumes the signal carries a validity/health indicator consistent with its safety relevance; not confirmed against baseline Localization's actual output (Open Item OI-2).

**H. Trip Manager → Vehicle Controls (Egress Authorization)**
* Payload: egress-ready status, at minimum
* Rationale: Baseline arrived-and-stopped signaling, required for any drop-off, issued once Trip Manager has both Behavior Planner's CANDIDATE_COMMITTED status code (Interface 3) and independent ground-truth confirmation from Localization and Vehicle Controls (Baseline Dependencies P, Q) that the vehicle has actually stopped at that location. This use case relies on it firing at whatever location the search committed to, which may not be the original requested coordinate. Sourced from Trip Manager rather than Behavior Planner directly, consistent with Trip Manager owning mission-level authorization (block-diagram.md Basis of Design). Distinct from Baseline Dependency R below - this authorizes the door/egress domain specifically, not the driving-actuation output Vehicle Controls also receives from Motion Planner.

**I. Rider HMI → Trip Manager (Rider-Initiated Unlock Request)**
* Payload: unlock request, at minimum
* Rationale: Baseline egress interaction, independent of search logic. Reaches Trip Manager first, which validates it against mission state before any authorization reaches Vehicle Controls, rather than a UI action reaching an actuation-authority module directly. Rider-initiated versus autonomous unlock is a baseline vehicle-platform decision this use case does not own (Open Item OI-3).

**J. Door-Clearance Sensor → Vehicle Controls (Road-User Clearance Signal)**
* Payload: clearance signal, at minimum
* Rationale: Baseline safety interlock (block-diagram.md Section 4). The same interlock applies regardless of where the vehicle stopped; this use case does not add a location-specific variant.

**K. Vehicle Controls → Door Actuation (Unlock / Lock Command)**
* Payload: unlock / lock command, at minimum
* Rationale: Baseline actuation, gated by Vehicle Controls' own safety logic, independent of this use case's existence.

**L. Behavior Planner → Motion / Path Planner (Stop-Fence / Target Specification)**
* Payload: reference line or path segment with an embedded stop constraint (position and/or pose) when a stop is intended; absence of an active stop constraint otherwise, at minimum
* Rationale: A pre-existing AV-stack mechanism this use case relies on rather than defines - Behavior Planner expresses a stop or hold decision as a constraint on the reference line, not a symbolic command [2][3] (see block-diagram.md Basis of Design). Assumes the mechanism supports being set, cleared, and re-set within a single trip segment (Open Item OI-7).

**M. Trip Manager → Rider HMI (Arrival-Imminent / Prepare-to-Exit Notification)**
* Payload: baseline arrival-imminent / prepare-to-exit notification, at minimum
* Rationale: Every drop-off needs this baseline notification. Sourced from Trip Manager, not Behavior Planner directly - Trip Manager is the single source of truth for rider-facing UI state, and this notification fires from the same ground-truth-confirmed arrival that drives Baseline Dependency H, just relayed to the rider instead of Vehicle Controls. This use case only changes when the notification fires - including after the vehicle commits to a candidate reached via corridor expansion or a fallback hub - not what it contains, and does not introduce an urgency-differentiated variant.

**N. Cloud Prior Layer → Trip Manager (Fallback Hub / POI Lookup)**
* Payload: designated fallback hub locations (position, capacity class), at minimum
* Rationale: Trip Manager, not World Model Builder or Route Planner, is the consumer of fallback hub location data - resolving a location for a triggered task and handing it to Route Planner is Trip Manager's existing role (block-diagram.md Basis of Design, citing Apollo's `task_manager` parking-routing precedent), and folding hub data into Interface 1's World-Model-Builder-bound grid payload would hand it to a module that has no use for it. This is treated as a baseline capability of Trip Manager's existing POI-lookup role, not new interface content this use case defines; the existence of "designated high-capacity fallback locations" as prior data is a stated design assumption, not verified against a specific cited precedent (Open Item OI-8).

**O. Trip Manager → Route Planner (Mission Goal / Destination Assignment)**
* Payload: destination or waypoint update, at minimum
* Rationale: Baseline capability this use case relies on rather than defines - Trip Manager already has some path for assigning or changing Route Planner's destination independent of this feature (e.g., trip dispatch at pickup, or an ordinary mid-trip destination change), consistent with Apollo's `task_manager` issuing routing requests to the Routing module for other triggered tasks, and with reference architectures where the routing-level module takes goal input from a layer above it rather than deriving it itself [4]. Interface 5's `route_type`/`target_location` directive is feature-specific content layered onto this same Trip-Manager-to-Route-Planner path, not a new communication channel; the exact baseline mechanism (structured API call versus a generic destination field) is not confirmed (Open Item OI-11).

**P. Localization → Trip Manager (Ego Pose vs. Goal)**
* Payload: ego pose relative to the current mission goal, at minimum
* Rationale: Baseline capability this use case relies on rather than defines - Trip Manager already needs ego-position tracking against its current goal for generic mission-lifecycle purposes (arrival detection, ETA, ordinary trip completion) independent of this feature. This use case relies on it as one of two ground-truth inputs (with Baseline Dependency Q) that let Trip Manager independently confirm arrival at Interface 3's `committed_location` rather than trusting Behavior Planner's CANDIDATE_COMMITTED status code alone (block-diagram.md Basis of Design).

**Q. Vehicle Controls → Trip Manager (Chassis State)**
* Payload: speed, gear, and door/lock status, continuously, at minimum
* Rationale: Baseline capability this use case relies on rather than defines - Vehicle Controls already reports chassis state to Trip Manager for generic mission-lifecycle tracking (e.g., confirming the vehicle is actually stopped before ending a trip), independent of this feature. This use case relies on it as Trip Manager's second ground-truth input for arrival confirmation (with Baseline Dependency P) and, once door/lock status also confirms secured, for determining drop-off conclusion - `Evt_DropOffConcluded` (state_chart.md) requires door-secured and clearance-envelope-clear facts that speed and gear alone do not supply, so door/lock status is assumed to be part of the same continuous chassis-state feed rather than a separate interface (block-diagram.md BD-4). Vehicle Controls' own zero-speed confirmation for the door-unlock interlock, separately, is sourced from its own independent vehicle-state input, not Localization - required so a shared-pipeline fault cannot compromise both the tactical decision and that safety interlock (SC6, sotif-stpa.md).

**R. Motion / Path Planner → Vehicle Controls (Actuator Commands)**
* Payload: steering, brake, and throttle commands, at minimum
* Rationale: Baseline AV-stack mechanism this use case relies on rather than defines - the execution leg of the `BP --> MP --> VC` chain, closing the path from Behavior Planner's stop-fence constraint (Baseline Dependency L) to the low-level driving actuation that carries it out (block-diagram.md Basis of Design). Trajectory-generation logic itself (how Motion Planner turns a reference-line constraint into actuator commands) is not this use case's content and is not redefined here - only the fact that the output feeds Vehicle Controls is modeled.

------------------------------
## Open Items

* OI-1 - Staleness bound for Interface 1's prefetched restriction data, and the resulting degraded-mode behavior.
* OI-2 - Whether baseline Localization (Baseline Dependency G) provides a validity/health signal this use case's kinematic check can rely on; assumed present, not confirmed.
* OI-3 - Rider-initiated versus autonomous door unlock (Baseline Dependency I) - a baseline vehicle-platform decision, referenced here because it determines whether Baseline Dependency M's notification content needs to say anything different.
* OI-4 - Time horizon for the door-clearance interlock (Baseline Dependency J, K).
* OI-5 - Scope of rider notification during the nominal (no-warning) case - referenced in the Concept of Operations.
* OI-6 - Failure mode when Interface 1's prefetch does not complete before the anticipatory-search trigger is reached.
* OI-7 - Whether the baseline stop-constraint mechanism (Baseline Dependency L) supports set-clear-reset within a single trip segment, which this use case's de-commit behavior requires and ordinary drop-off does not exercise.
* OI-8 - Whether "pre-designated, high-capacity fallback hub" locations (Baseline Dependency N) are an established Cloud Prior Layer capability or new data this use case must specify; treated here as a stated design assumption, not confirmed against a cited precedent.
* OI-9 - Default action and response-window duration if the rider does not reply to the Interface 4 preference prompt (e.g., a fixed timeout defaulting to EXPAND_SEARCH or NAVIGATE_TO_HUB) - not yet specified.
* OI-10 - Whether Interface 1's SAFETY_CRITICAL / REGULATORY restriction-tier split still earns its place in the payload now that legal relaxation is rejected and both tiers are treated identically by candidate evaluation, or whether it should collapse to a single RESTRICTED value pending a consumer that needs the distinction.
* OI-11 - Whether Baseline Dependency O (Trip Manager → Route Planner destination/goal assignment) exists in the baseline AV stack as a structured API or only as an implicit assumption; Interface 5 assumes some such path exists but does not confirm its actual form.
* OI-12 - Whether Baseline Dependency Q's chassis-state feed (Vehicle Controls → Trip Manager) already includes door/lock status alongside speed and gear, or whether that is a separate baseline signal this use case is assuming gets folded in; `Evt_DropOffConcluded` (state_chart.md) needs door/lock status specifically, and speed/gear alone do not supply it (block-diagram.md BD-4).

------------------------------
## References

[1] M. Bansal, A. Krizhevsky, and A. Ogale, "ChauffeurNet: Learning to Drive by Imitating the Best and Synthesizing the Worst," arXiv:1812.03079, 2018.

[2] H. Fan, F. Zhu, C. Liu, L. Zhang, L. Zhuang, D. Li, W. Zhu, J. Hu, H. Li, and Q. Kong, "Baidu Apollo EM Motion Planner," arXiv:1807.08048, 2018.

[3] J. Wei, J. M. Snider, T. Gu, J. M. Dolan, and B. Litkouhi, "A Behavioral Planning Framework for Autonomous Driving," in Proc. IEEE Intelligent Vehicles Symposium (IV), 2014.

[4] Autoware Foundation, "Planning Component Design," Autoware Documentation, autowarefoundation/autoware-documentation.