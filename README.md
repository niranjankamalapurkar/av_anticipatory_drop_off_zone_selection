# Anticipatory Stopping-Zone Selection for Autonomous Vehicle Drop-Off in Dense Urban Morning Commute Conditions

## 1. Motivation (The "Why")

A human driver approaching an office-district drop-off during the morning commute doesn't wait until the exact address to start thinking about where to stop. They already know, from general familiarity with the area and the time of day, that curb space near office entrances is heavily contested between roughly 7 and 9:30am - every other rideshare, taxi, and delivery vehicle is converging on the same few blocks at the same time, many curb restrictions that don't apply in the evening are actively enforced during this specific window, and a rider heading to work has less tolerance for a long, indecisive search than a rider with no fixed schedule. So the human driver starts scanning early, evaluates candidates as they pass, and - critically - keeps going past the exact requested address if nothing suitable has appeared yet, rather than stopping in the travel lane and creating exactly the obstruction the whole search was meant to avoid.

An autonomous vehicle that only begins evaluating stopping options once it reaches the destination coordinate has already given up the lead time a human driver uses productively. This isn't a hypothetical gap: current autonomous parking and drop-off technology has been documented, in its own supporting patent literature, as searching only *after* arrival - spending time hunting for a spot the vehicle could have started evaluating blocks earlier.

**The Business & Experience Impact:**

- **Rider Experience Under Time Pressure:** A commuter with a fixed start time is more sensitive to an indecisive, late-starting search than a leisure rider would be. Anticipatory search converts "the car seems lost near my destination" into "the car dropped me off smoothly a short walk from where I asked," which matters disproportionately during exactly this time window.

- **Traffic Impact & Regulatory Standing:** Research on rideshare drop-off behavior has found a large share of pickups and drop-offs in dense urban areas occur in restricted zones - a documented, current source of both citations and congestion. A vehicle that searches too late and settles for the nearest available spot regardless of legality contributes directly to this problem; a vehicle that anticipates and evaluates candidates earlier has a materially better chance of finding a genuinely legal one in time.

- **Operational Efficiency:** A vehicle that reaches the destination coordinate only to discover it must now search - potentially idling, circling, or blocking a lane while it does - costs fleet time and creates exactly the kind of visible, publicly-noticed obstruction that erodes trust in autonomous operation, independent of whether any specific rule was technically violated.

## 2. Problem Statement (The "What")

Current drop-off behavior, per the reviewed literature, evaluates stopping locations only once the vehicle reaches the vicinity of the destination, rather than anticipating the need to search based on known context (area density, time of day) before arrival. This is most consequential in dense, high curb-contention urban cores during the morning commute - the combination of enforced time-of-day restrictions, concentrated simultaneous demand, and rider time-sensitivity that makes late, reactive searching most costly and anticipatory searching most valuable.

**Core Objective:**

We need to design the onboard decision logic that governs how an autonomous vehicle searches for and commits to a legal, unobstructed, walkably-close stopping location for drop-off - beginning that search before reaching the exact destination coordinate when context indicates it will be needed, and continuing the search past the original coordinate if no suitable candidate has appeared, rather than defaulting to the exact requested point regardless of conditions. The system must fulfill a dual mandate, organized around the two planning layers this project focuses on:

1. **Route-Level Anticipatory Trigger:** Given a destination known, from route and map context, to fall within a dense, high curb-contention area during the morning commute window, the system shall initiate active stopping-zone evaluation before the vehicle reaches the destination coordinate, rather than waiting until arrival.

2. **Behavior-Level Candidate Evaluation and Commit/Continue Decision:** While actively searching, the system shall evaluate each candidate curb location the vehicle passes against legal stopping permissibility (time-of-day-aware, distinguishing whether a brief passenger stop is currently permitted - not merely whether the zone is restricted at all), physical availability (unobstructed, per live perception), and proximity to the original destination (within an assumed rider walking tolerance) - and decide, for each candidate in turn, whether to commit to it or continue searching, including continuing past the original destination coordinate if nothing suitable has yet appeared.

---

## Scope

**In scope:**
- Dense, high curb-contention urban core areas - the kind of environment where multiple vehicles regularly compete for the same limited curb space, not areas with generally available parking.
- The morning commute window specifically (approximately 7:00–9:30am on business days) - chosen because time-of-day-dependent curb restrictions, concentrated simultaneous demand, and rider time-sensitivity are all simultaneously at their peak in this window, making it the condition where anticipatory search delivers the most value.
- Drop-off only, not pickup - pickup introduces the additional complexity of locating the rider's own current position, a distinct problem this project does not take on.
- The legal-status taxonomy needed as an input to the decision tree (time-of-day-aware, distinguishing "brief passenger stop permitted" from "not permitted," per the No Parking / No Standing / No Stopping / metered-parking distinctions this project's research surfaced) - defined as far as needed to specify the decision tree's inputs, not built out as a full mapping system.

**Explicitly out of scope:**
- Building or populating the known-restriction database itself. This is assumed to exist (see Assumptions) - it is also the part of this problem space already well-covered by existing patent literature; re-deriving it here would not be a differentiated contribution.
- Motion/Path Planner's trajectory execution - the actual steering and pull-in maneuver once a spot is committed to. Baseline, consistent with how this project has treated actuation-level execution throughout its prior work.
- Fleet-level or infrastructure-level curb orchestration (centralized systems assigning vehicles to designated curb spaces). This is a real, actively-developed alternative approach found during this project's research - this project is deliberately the onboard, single-vehicle-reactive alternative to that, not a contribution to it.
- Non-dense areas (suburban, low curb-contention) and non-morning-commute time windows (evening, weekend, off-peak). The restriction taxonomy's time-of-day dimension and the traffic-pressure dynamics this project models are most relevant and most tractable in the scoped window - generalizing beyond it is a natural future extension, not attempted here.
- Reduced-mobility or accessibility accommodation. This project assumes a rider capable of walking the full assumed tolerance distance unassisted (see Assumptions) - a real system would need a separate accommodation path for riders who cannot, which this project does not design.

## Assumptions

- A time-of-day-aware, legally-categorized curb-restriction database already exists and is queryable by the decision tree - populated and maintained by systems outside this project's scope, consistent with the precedent found in existing patent literature for exactly this kind of mapping.
- The rider is in good general health and mobility, able to walk approximately 50 meters from the actual stop location to the original requested destination without difficulty. This is a stated simplification, not a general design constraint - a real system would need this to be a configurable, rider-specific value.
- Live perception (detecting whether a specific curb location is physically obstructed right now - a double-parked vehicle, active construction) is available and reasonably reliable at the short range relevant to curb-side evaluation. This project does not re-derive perception reliability.
- The vehicle's own current position and heading, needed to determine whether a candidate location is still ahead of the vehicle or has already been passed, are available via Localization - the same baseline dependency and geometric reasoning already established in this project's prior work.
- The Motion/Path Planner can faithfully execute whatever specific stop location the Behavior Planner commits to. Actuation-level execution fidelity is baseline and out of scope, the same discipline applied throughout this project's prior work.