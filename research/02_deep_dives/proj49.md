# proj49 Deep Dive — Heterogeneous Multi-Robot PDDL Planning & Scheduling

**Stage 1 score:** 22/25 · **Level:** A · **Tag:** 学术型 · **Tutor:** 李鹏 (ISCAS)
**Title (EN):** Heterogeneous Multi-Robot Planning and Scheduling

## 1. Target restated

A **central-dispatcher middleware** that, given (i) an officially-supplied JSON topology + task set and (ii) a fleet of heterogeneous robots (e.g. an aerial scanner-unlocker + ground transporters), emits a temporally-valid PDDL action sequence such that:

- 100 % of tasks finish (basic);
- **zero deadlock / zero collision** under physical critical sections (single-lane corridors, sole charging dock, one-way gates) — *progressive task*;
- heterogeneous-cooperation (drone unlocks → ground robot enters) is expressed as *state-dependent temporal synchronization* in the PDDL domain — *advanced task*.

Evaluation is automatic: the organiser ships a Python **State-Checker** that consumes the action stream and reports timing validity, collision, deadlock, makespan. This means the rubric is **half-deterministic** (zero-collision + makespan are numeric) and half-subjective (PDDL modelling elegance, code quality, paper).

Rubric weights (from `target`):

| Axis | Weight |
|---|---|
| Scheduling efficiency & safety (makespan, parallelism, zero deadlock) | 40 % |
| Innovation & domain modelling | 25 % |
| Engineering completeness & code quality | 20 % |
| Short-paper-style technical report | 15 % |

Level **A** + 学术型 tag signals jury expects a *publishable* idea, not just a working tool.

## 2. State of the art (May 2026 snapshot)

### 2.1 Symbolic / PDDL planners

| Tool | Why relevant |
|---|---|
| **Fast Downward** | Reference classical planner; A*+LM-cut; numeric extension via FD-numeric branch. Tutor explicitly recommends. |
| **ENHSP** | Numeric & temporal expressive PDDL 2.1 with `:durative-actions`; well integrated under Unified-Planning. |
| **Tamer** | Temporal action planner; UP-bound. |
| **PDDLStream** | Logic + geometric sampling — relevant if geometry stops being abstract. |
| **MA-PDDL** (Kovacs 2012, used by tutor in `topicreq`) | Multi-agent variant adopted by FMAP, MAP-POP. |
| **FMAP**, **MAP-POP** | Forward-chaining multi-agent / POP-based planners. Both 2015-era; still common baselines. |

### 2.2 MAPF / coupled motion planners

- **CBS** (Sharon et al., AAAI'15) and **ECBS** (focal search) — gold-standard for multi-agent pathfinding on grid graphs.
- **libMultiRobotPlanning** (whoenig, MIT) — open-source C++ implementations of A\*, A\*-ε, SIPP, **CBS**, **ECBS**, **CBS-TA**, **ECBS-TA**. YAML scenario format. Up to thousands of agents on grid maps.
- **CBS-TA** = CBS + task assignment, exactly the proj49 problem if the TA part is given as a JSON file.

### 2.3 Unified-Planning (UP) framework — recommended by tutor

AIPlan4EU UP exposes a Python API (`Fluent`, `Action`, `Problem`, `OneshotPlanner`) and binds native engines + arbitrary PDDL planners. It supports **classical, temporal, and numeric** planning. It does **not** natively expose multi-agent semantics — multi-agent must be encoded by extending the state with `(robot ?r)` typing and per-robot fluents, the same trick FMAP uses.

### 2.4 ROS-side fleet management — relevant but not required

- **Open-RMF** (Open Robotics Middleware Framework, OSRF) — multi-fleet dispatch, traffic editor, conflict tables. Used in airports/hospitals. Could be a downstream consumer of the planner.
- **Nav2 multi-robot lifecycle** (Humble/Iron/Jazzy 2024-2025): namespace-isolated nav stacks, can co-simulate multi-robot in Gazebo Harmonic.

## 3. Public benchmarks — hedge before the official set lands

The official JSON scenarios will likely arrive late (judging from prior CSCC OS tracks). We need a **public benchmark to validate end-to-end** during M1–M2 *before* the organizer release. Three plausible hedges, ranked:

1. **libMultiRobotPlanning YAML scenarios** (`map_NxM_obstA_agentsK_exJ.yaml`). MIT-licensed, directly executable. Convert YAML → MA-PDDL is a 100-line script. **Best fit** because the format is grid + agents + start/goal, matching the proj49 narrative.
2. **Open-RMF demos** (`rmf_demos`): hospital/office/airport scenarios with multi-fleet tasks. Ros2-bound, heavier. Use if the official set includes service-robot scenarios.
3. **RoboCup Logistics League** (RCLL) refbox scenarios. Real factory-logistics setting, but the rules add machine-FSM constraints that may diverge from the official problem statement. Use only if proj49 official problems involve "machines that process parts".

A safe play is to ship the prototype validated on (1), keep (2) as a stretch demo for the final answer.

## 4. Design strategy

### 4.1 Two-layer planner architecture

The tutor's task list cleanly splits into **task-allocation + sequence** (PDDL) and **collision-free motion** (MAPF). A two-layer split is more competitive than a monolithic PDDL solve:

```
JSON scene ─┬─► PDDL Domain/Problem ──► UP + FD/ENHSP ──► assignment + ordering
            │                                                     │
            └─► MAPF graph (libMRP)  ◄─── refined waypoints ◄─────┘
                                          │
                                          ▼
                                   CBS-TA → trajectories
                                          │
                                          ▼
                                   State-Checker
```

- **Layer A — symbolic PDDL** decides *who does what in which order*. Heterogeneous capabilities live here as typed actions (`(:action drone_unlock :parameters (?d - drone ?g - gate))`). Critical sections are encoded with **Mutex fluents** (`(mutex_free ?cs)`), preconditions `(mutex_free ?cs)`, effects `(not (mutex_free ?cs))` plus release. This is exactly the OS-process-scheduling analogy the tutor's `info` field calls out.
- **Layer B — geometric MAPF** turns each ordered task pair into a collision-free path via **CBS-TA** from libMultiRobotPlanning. The two layers communicate via an interface schema: PDDL emits `(robot, from, to, time-window)`, MAPF returns `(path, t_arrive)` and either commits or signals infeasibility → backtrack into PDDL.

### 4.2 Domain modelling innovations (25 % of score)

Three modelling tricks to call out in the short-paper:

1. **Critical-section = Mutex fluent**. Frame the construction explicitly as the *Operating-Systems analogy* the tutor invites in `info`. Every single-lane corridor is a binary semaphore; every charging dock a counting semaphore. Then **deadlock-avoidance = banker's-algorithm encoded as preconditions** (a robot may only enter a critical section if a hold-free ordering across all currently-held resources exists). This OS-flavoured framing is exactly what the OS-track jury wants.
2. **Heterogeneous communication as `(:durative-action)` synchronization**. Drone-unlock and ground-enter share a `(gate_unlocked ?g)` fluent with **overlapping intervals**: drone `(over all (gate_unlocked ?g))` while it hovers; ground enter `(at start (gate_unlocked ?g))` then closes after. This expresses *intent handshake* without introducing explicit comms primitives.
3. **Lifted Mutex-graph compilation**. At parse time, build the conflict graph between actions on shared resources; produce a **per-resource action ordering constraint** as a derived predicate (`:derived`) — cuts the search-space dramatically on dense conflict scenes.

### 4.3 Engineering middleware (20 %)

- **JSON → PDDL transpiler** (`scene2pddl.py`): grid + waypoint graph + robot specs → Domain + Problem. Strict schema validation (jsonschema), pytest snapshot tests.
- **UP-as-orchestrator**: choose planner per problem class — Fast Downward for purely classical, ENHSP for numeric battery/charge actions, Tamer for `:durative-actions`.
- **Plan → action-stream serializer** matching the State-Checker format.
- **CLI** with reproducible seed, deterministic tiebreaks (essential — Makespan ties must be broken by the same rule run-to-run, or the jury can't reproduce the numbers).

### 4.4 Scheduling efficiency (40 %)

Makespan is the headline number. Three accelerators:

- **Symmetry breaking**: identical robots get a lexicographic ordering constraint → cuts FD search by factor *n!*.
- **Goal decomposition**: solve per-cluster (disjoint subgraph) then stitch.
- **MAPF post-pass**: even after PDDL commits an ordering, slack inside a `:durative-action` window can be tightened by re-running CBS-TA over the committed assignment — typically 5–15 % makespan reduction.

Stretch: **LLM-assisted PDDL repair** for the report. When the planner returns "unsolvable", call a code-model with the failing problem snippet and ask for a domain-encoding fix. This is risky as a critical path but *very* publishable — exactly the "学术型" tag.

## 5. Risk register & mitigation

| Risk | Severity | Mitigation |
|---|---|---|
| Official JSON schema unknown until late | **High** | §3 hedge with libMultiRobotPlanning; build the transpiler around an internal IR so a new front-end is ≤ 1 day. |
| MA-PDDL temporal solving doesn't scale past N robots | High | Drop to **task-decomposition + per-cluster classical PDDL + CBS-TA** layer; preserve "MA-PDDL Domain" as a presentation artifact even if at runtime we use the layered solver. |
| Fast Downward incompatible with `:durative-actions` | Med | Use ENHSP or Tamer for temporal subset; FD for classical core. UP routes automatically. |
| Deadlock detection in State-Checker = adversarial | High | Implement an offline *redundant* deadlock check (Coffman conditions on the runtime resource graph) and abort plans pre-submission; never trust planner output without a sanity post-check. |
| Tie-breaking randomness → judge sees different makespan | Med | Fix all seeds; ship a `reproduce.sh` that pins planner version + seed + python hash seed. |
| "Just used Fast Downward" criticism | Med | §4.2 three modelling innovations + §4.4 MAPF post-pass + the OS-banker's-algorithm framing constitute the publishable delta. |

## 6. Four-month execution plan

- **M1**: Stand up UP + FD + ENHSP. Toy MA-PDDL domain on a 4-robot, single-corridor map from libMRP. Build State-Checker-like local validator (clone behaviour from the rubric). Lock in the IR + transpiler architecture.
- **M2**: Encode mutex / banker / heterogeneous handshake patterns. Integrate libMultiRobotPlanning CBS-TA via subprocess. Pass progressive-task (zero-deadlock) on libMRP scenarios. Start short-paper Section 3.
- **M3**: Heterogeneous drone+ground scenarios (advanced task). Symmetry-breaking + goal decomposition optimisations. Benchmark makespan vs naive sequential allocator (target ≥ 30 % reduction). Switch to official JSON schema once released; transpile + retest.
- **M4**: LLM-PDDL-repair side experiment (1 week, optional). Final short-paper write-up (5–6 pages, ICAPS/IROS workshop style). Code freeze + reproducibility kit + 3-min demo video.

## 7. Verdict

proj49 is **the lowest-risk Level-A题** on the table: a pure-software algorithm task with a deterministic eval harness (State-Checker), well-trodden tooling (UP, FD, libMRP), and a tutor (李鹏 / ISCAS) who has already supplied a roadmap in the topic text. The competitive risk is the **opposite of proj22's**: it's *too* tractable, so many teams will deliver a working pipeline and differentiation lives in (a) the OS-process-scheduling framing of critical sections, (b) the two-layer PDDL+CBS-TA architecture, and (c) the publishable short paper. With the libMRP hedge in §3 we are not gated on the organizer's release calendar.

**Recommended combination with proj22**: proj49 is the *safe academic* leg (clear scoring, deterministic eval, ICAPS-style paper outcome). proj22 is the *high-variance security/AI* leg with bigger upside if 0-day bugs land. Together they cover the framework D ("安全 + Agent") and framework E ("稳健工程") strategies from Stage 1.
