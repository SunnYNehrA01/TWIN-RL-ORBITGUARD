# TWIN-RL-ORBITGUARD [Digital Twin-Based Space Debris Collision Detection System]

## Abstract
This project implements a digital twin-based collision detection and avoidance system to identify near-term conjunctions between an operational satellite and cataloged space debris. The system provides probabilistic collision estimates and recommends fuel-aware avoidance maneuvers. 

The main innovation is the **Collision Occurrence Zone (COZ)**—a moving 3D volume around the satellite for a 24-hour horizon that restricts computation to imminent threats. Probabilistic “phantom” sampling accounts for orbital uncertainties, while the optional reinforcement-learning agent helps optimize maneuver selection without replacing deterministic physics propagation.

---

## Table of Contents
1. Concept & Motivation  
2. Goals & Success Criteria  
3. System Overview  
4. Methodology  
5. System Components & Architecture  
6. Data Sources & Formats  
7. Algorithms & Mathematics  
8. Implementation Plan & Tech Stack  
9. Testing, Validation & Metrics  
10. Performance & Scaling Considerations  
11. Project Roadmap & Milestones  
12. To-do List  
13. Contribution & License  
14. References  

---

## 1. Concept & Motivation
Near-Earth orbits are increasingly congested with operational satellites and debris. Even centimeter-scale debris moving at orbital velocities (~7–8 km/s) can destroy satellites, generate thousands of new fragments, and reduce orbital sustainability.  

This system is designed to:

- Identify near-term collision threats within a 24–72 hour window.
- Quantify collision probability while accounting for orbital uncertainty.
- Recommend fuel- and mission-aware avoidance maneuvers.
- Continuously update predictions as new observations arrive.

The Collision Occurrence Zone (COZ) reduces computational load while ensuring near-term collisions are detected reliably. Physics-based propagation remains the core of the system; AI assists only with prioritization and surrogate calculations.

---

## 2. Goals & Success Criteria

**Primary Goals**
- Detect all potential collisions with Pc above a configurable threshold.
- Provide maneuver recommendations that reduce Pc below the threshold while respecting fuel and pointing constraints.
- Maintain computation speed to handle daily conjunctions for a medium-sized constellation.

**Success Criteria**
- True positive rate for high-Pc conjunctions ≥ X% on test sets.
- Maneuver recommendations reduce Pc below threshold in >95% of flagged cases.
- End-to-end pipeline latency within operational limits (minutes) for the target hardware.

---

## 3. System Overview
The pipeline includes:

1. **Data ingestion** — TLEs, precise state vectors, sensor updates.  
2. **Broad-phase filtering** — altitude/orbit overlap, directional/time-of-crossing filters, COZ membership.  
3. **Phantom generation** — Monte Carlo sampling of position and velocity using covariance.  
4. **Digital twin propagation** — physics-based forward propagation of phantoms and satellite.  
5. **Collision detection & probability estimation** — intersection checks and Monte Carlo Pc calculation.  
6. **Maneuver planning** — evaluate Δv maneuvers, optionally assisted by RL agent.  
7. **Rolling updates** — continuous integration of new sensor data.

This design balances computational efficiency with high-fidelity physics simulation.

---

## 4. Methodology

### 4.1 Data Acquisition & State Estimation
- Accepts TLEs, precise radar or telescope state vectors, and observation streams.
- Maintains nominal state and 6×6 covariance matrix per object to represent orbital uncertainty.
- Covariance allows probabilistic simulation instead of assuming perfect knowledge of orbits.

### 4.2 Broad-Phase Filtering
- **Altitude/Orbit Overlap**: exclude debris whose orbits do not intersect the satellite altitude range.
- **Directional/Time-of-Crossing**: exclude debris with incompatible relative motion or timing.
- **COZ Membership**: only debris that may enter the satellite-centric COZ within the 24-hour horizon are considered.

### 4.3 Collision Occurrence Zone
- 3D volume around satellite trajectory: dimensions = satellite size + uncertainty buffer.
- Shape may be spherical, ellipsoidal, or elongated along velocity vector.
- Recomputed dynamically at each update or after satellite maneuvers.

### 4.4 Phantom Debris Monte Carlo
- Each candidate debris is represented by hundreds to thousands of phantoms.
- Phantoms are sampled from multivariate Gaussian distributions using nominal state and covariance.
- Captures the uncertainty in debris orbits for probabilistic collision assessment.

### 4.5 Digital Twin Propagation
- Satellite and phantom debris are forward propagated using physics-based models (two-body + perturbations).
- Intersection checks are performed at each timestep.
- Parallel computation can be used to scale Monte Carlo simulations efficiently.

### 4.6 Collision Probability Estimation
- Monte Carlo Pc = (# of phantoms colliding) / total phantoms.
- Optionally, analytic b-plane integration provides faster estimates in linearized scenarios.
- Outputs include Pc, expected miss distance, and time-of-closest-approach.

### 4.7 Maneuver Simulation & Planning
- Candidate maneuvers: Δv impulses in various directions and timings.
- Evaluated for Pc reduction and compliance with fuel, pointing, and operational constraints.
- RL agent can suggest optimized Δv policies but all outputs are validated with deterministic physics.

### 4.8 Rolling Horizon Updates
- New observations refine object states and recompute COZ.
- Only high-risk debris are propagated.
- Maneuver recommendations are re-evaluated continuously to maintain near-real-time risk assessment.

---

## 5. System Components & Architecture
- **Data Ingest Module**: parses TLEs and state vectors; normalizes reference frames.
- **Filter Module**: applies broad-phase filtering and COZ membership checks.
- **Uncertainty Module**: handles covariance propagation and phantom sampling.
- **Propagator**: SGP4 or high-fidelity numerical integrator.
- **Collision Engine**: geometric intersection and Pc calculation.
- **Maneuver Engine**: evaluates Δv maneuvers, optionally using RL agent.
- **Orchestrator**: manages batch propagation and rolling updates.
- **Visualization Module**: dashboards for collision risk and maneuver planning.
- **Persistence & Logging**: tracks events, simulations, and Pc over time.

---

## 6. Data Sources & Formats
- **TLE** — Two-Line Elements, SGP4-compatible.
- **Precise State Vectors** — ECI/ECEF position and velocity with covariance.
- **Sensor Observations** — Radar/optical measurements.
- **Debris Catalogs** — SSA public catalogs or operator data feeds.

All data include epochs and reference frame tags for consistent propagation.

---

## 7. Algorithms & Mathematics
- **SGP4 Propagation**: nominal orbital trajectories from TLE.
- **Covariance Sampling**: multivariate Gaussian for position & velocity (Cholesky decomposition).
- **Monte Carlo Collision Probability**: count phantom intersections with satellite volume.
- **Analytic b-plane Estimation**: fast approximation using projected Gaussian distribution.
- **Reinforcement Learning**: PPO/DDPG for Δv optimization under constraints.

This combination ensures high-fidelity physics with probabilistic risk assessment and maneuver optimization.

---

## 8. Implementation Plan & Tech Stack
- **Python 3.10+**
- Libraries: `numpy`, `scipy`, `pandas`, `sgp4`, `poliastro`, `matplotlib`, `pytorch`
- Optional parallelization: `ray`, `dask`, GPU acceleration via `numba` or CUDA.
- Prototype first with Python and SGP4, then scale with parallel Monte Carlo and RL.

---

## 9. Testing, Validation & Metrics
- **Unit Tests**: propagator accuracy, phantom sampling correctness, collision engine geometry checks.
- **Integration Tests**: historical conjunctions, maneuver validation.
- **Metrics**: Pc accuracy, false positive/negative rate, maneuver Δv cost, pipeline latency.

---

## 10. Performance & Scaling
- Filtering and COZ reduce debris candidates drastically.
- Batch propagation allows parallel Monte Carlo.
- Surrogate models or importance sampling can accelerate low-Pc events.
- System scales to medium-sized constellations by scheduling incremental propagation windows.

---

## 11. Project Roadmap & Milestones
1. **Weeks 0–2** — Acquire data, understand formats, validate epochs and frames.  
2. **Weeks 3–6** — Prototype ingestion, SGP4 propagation, simple COZ filter.  
3. **Weeks 7–10** — Phantom generator, Monte Carlo propagation, Pc estimation.  
4. **Weeks 11–14** — Maneuver evaluator (grid search) implementation.  
5. **Weeks 15–20** — RL agent integration for maneuver optimization.  
6. **Weeks 21–24** — Visualization dashboard, rolling updates, validation.  
7. **Beyond** — GPU acceleration, surrogate modeling, operator workflows.

---

## 12. To-do List

| To do | Description | Done |
| :--- | :--- | :---: |
| Explore data sources | Acquire TLE and state vector datasets; verify epochs and reference frames | ⬜ |
| Set up environment | Install Python 3.10+, Jupyter, `numpy`, `scipy`, `sgp4`, `pandas`, `pytorch` | ✅ |
| Implement TLE parser & SGP4 | Compute nominal ephemerides; canonicalize epochs/frames | ✅ |
| Design COZ | Generate 3D COZ for 24-hour horizon; implement unit tests | ⬜ |
| Broad-phase filters | Apply altitude, radial distance, and time-of-crossing filters | ⬜ |
| Phantom generator | Implement multivariate Gaussian sampling for debris | ⬜ |
| Monte Carlo engine | Propagate phantoms, compute intersections, calculate Pc | ⬜ |
| Analytic b-plane estimator | Implement for fast Pc approximation; validate against Monte Carlo | ⬜ |
| Maneuver evaluator | Sample Δv impulses; compute Pc reduction | ⬜ |
| RL agent prototype | Train PPO/DDPG agent for maneuver optimization; validate outputs | ⬜ |
| Validation scenarios | Reproduce historical conjunctions and verify Pc and maneuvers | ⬜ |
| Visualization dashboard | Display timeline, Pc plots, maneuver comparisons | ⬜ |
| Parallelization & scaling | GPU/cluster acceleration for Monte Carlo propagation | ⬜ |
| Integration tests | Validate pipeline correctness and reproducibility | ⬜ |
| Documentation | Prepare README, user guidance, and references | ⬜ |
| Operator approval UI | Review and approve recommended maneuvers | ⬜ |

---

## 13. Contribution & License
- **Contributing**: Fork repository → create feature branch → submit pull request with tests and documentation.  
- **Governance**: Maintain logs of simulation results and maneuver recommendations for audit.  
- **License**: Apache-2.0 (code) / ensure data is used according to provider licensing.

---

## 14. References
- NASA GMAT, ESA Space Debris Office papers.  
- Analytic Graphics Inc., SSA collision analysis documentation.  
- TLE propagation references: Vallado, *Fundamentals of Astrodynamics and Applications*.  
- Monte Carlo collision probability literature: Alfriend & Akella, “Probability of Collision” papers.  
- Open-source Python libraries: `sgp4`, `poliastro`, `numpy`, `pandas`, `scipy`, `pytorch`.  
