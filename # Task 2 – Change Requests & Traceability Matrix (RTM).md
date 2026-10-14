# Task 2: Change Requests & Traceability Matrix (RTM)

IDs from Task 3 (ST/E/T), Task 4 (CS/OP) and Task 5 (C) are used below.

## 1. Version-Controlled Change Log

Baseline: **v1.0** (approved, `Task1_Requirements.md`). Date of changes: 2026-10-14.

| Version | CR | Req. ID | Status | Old text | New text | Reason |
|---|---|---|---|---|---|---|
| v1.1 | CR-01 | FR-06 | **Modified** | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | The robot shall enter Safe Mode **within 3 seconds** when chamber or battery temperature exceeds the critical threshold or battery capacity falls below the emergency level. | "Critical" had no value and no response time. Mission Control needs a bounded reaction for medicine safety. |
| v1.2 | CR-02 | FR-10 | **Modified** | The system shall support communication with multiple robots simultaneously. | The system shall support **at least 20 simultaneously connected robots**. | "Multiple" is unbounded. The pilot serves 20 clinics, so capacity must be a number. |
| v1.3 | CR-03 | FR-01 | **Modified** | Only authenticated operators shall be permitted to issue robot commands. | The system shall require **authenticated and role-authorised** operators before accepting any robot command. | Authentication alone lets a Viewer command a robot. Role-based authorisation was required by the scenario. |

Impacted, not edited: NFR-03 (already sized for 20 robots; test re-run), SR-02 and T-11 (see Note 1).

**Note 1 (consistency with Task 3).** The scenario says "critically low battery -> return to warehouse". CR-01 says "emergency level -> Safe Mode". These are two different thresholds: **CriticalBattery <= 20 %** (abandon delivery, return) and **EmergencyBattery < 10 %** (Safe Mode within 3 s). Both are documented in the README assumptions.

## 2. Why the New Wording Is More Testable

| CR | Original weakness | What the new wording adds | How it is tested |
|---|---|---|---|
| CR-01 | "Critical" is undefined. No time limit. A test cannot say when the robot is too slow. | (1) A **time limit of 3 s**. (2) Three explicit triggers: chamber temperature, battery temperature, battery capacity. (3) An **emergency level** distinct from the critical level. | Boundary-value tests at the threshold. Measured delay: pass at 2.9 s, fail at 3.1 s. One test per trigger (TC-07, TC-08). |
| CR-02 | "Multiple" is satisfied by 2 robots, so no test can fail it. | A **numeric minimum (20)**, tied to the pilot. "Simultaneously connected" can be measured through heartbeats and sessions. | Load test with 20 simulated robots (pass) and 21 (defined behaviour). Check acknowledgement time and no message loss (TC-11). |
| CR-03 | Only one condition. "Permitted" does not say when the check happens. | **Two conditions** (authenticated AND role-authorised). **"Before accepting any command"** fixes the order and covers every command type. | Three-case negative matrix: no auth -> reject; auth but wrong role -> reject; auth and right role -> accept. Repeated for every command type (TC-02, TC-03, TC-01). |

## 3. Test Cases (traceable to scenario steps)

Scenario steps: **S1** command received, **S2** authenticated (and authorised after CR-03), **S3** executed, **S4** status reported, **S5** logged.

| TC ID | Title | Steps | Procedure | Expected result | Req. |
|---|---|---|---|---|---|
| TC-01 | Happy path delivery request | S1-S5 | Dispatcher sends DeliveryRequest to a valid clinic while the robot is IDLE | Accepted, NAVIGATING, ack <= 10 s, log record complete | FR-01, 02, 03, 09 |
| TC-02 | Unauthenticated command | S1, S2, S5 | Send a command with no, expired or forged token | Rejected, state unchanged, rejection logged | FR-01, NFR-02 |
| TC-03 | Wrong role (new after CR-03) | S1, S2, S5 | Viewer sends DeliveryRequest | Rejected "not authorised", state unchanged, logged | FR-01 |
| TC-04 | Invalid command parameters | S1-S3, S5 | Authorised operator sends unknown clinic ID | Rejected with reason, no state change | FR-02 |
| TC-05 | Obstacle handling | S3-S5 | A: obstacle while NAVIGATING. B: obstacle while RETURNING | AVOIDING_OBSTACLE, then back to the previous activity, events logged | FR-04 |
| TC-06 | Critical battery (<= 20 %) | S3-S5 | Drop battery to 20 % in NAVIGATING, then IDLE, then RETURNING | NAVIGATING -> RETURNING -> IDLE. IDLE rejects new requests. RETURNING unchanged | FR-05, FR-06 |
| TC-07 | Emergency battery (< 10 %) | S3-S5 | Drop battery to 9 % during NAVIGATING | SAFE_MODE <= 3 s, alert sent, logged | FR-06 |
| TC-08 | Critical temperature | S3-S5 | Chamber at 12.5 C, then battery temperature above limit | SAFE_MODE <= 3 s, operator alert | FR-06, 08 |
| TC-09 | Correction and sensor verification | S3-S5 | Raise temperature to 9 C. A: correction works. B: disable cooler | A: back in range, confirmed by sensors. B: protection mode after 10 min, alert <= 5 s | FR-08, SR-03 |
| TC-10 | Status under link delay | S4, S5 | 5 s delay. Cut link for 1 h, restore | Reports <= 10 s. Buffered records uploaded in order | NFR-01, NFR-04, FR-09 |
| TC-11 | 20 simultaneous robots | S1-S5 | 20 simulated robots send telemetry, one command each. Then add a 21st | All 20 acknowledged <= 10 s, no loss. 21st handled as defined | FR-10, NFR-03 |
| TC-12 | IDLE must not deliver | S1, S3 | Inject DestinationReached while IDLE | Stays IDLE, event ignored and logged | SR-04, FR-05 |
| TC-13 | Railway crossing | S3-S5 | Barrier closed, train approaching, stale ARLCCS data | Robot stops >= 5 m before the crossing, no entry | SR-01 |
| TC-14 | Link loss | S3-S5 | Cut link > 30 s in IDLE and NAVIGATING, then > 120 s, then restore | LINK_LOST, autonomous RETURNING after 120 s, restore handled | SR-02 |
| TC-15 | Door opened during conservation | S3-S5 | Open door in CONSERVING | Conservation suspended. Resumes only after sensor and condition verification | FR-07 |
| TC-16 | Audit log completeness | S5 | Run accepted and rejected commands, then inspect the log | Every command has all FR-09 fields | FR-09 |
| TC-17 | CriticalBattery during AVOIDING_OBSTACLE | S3-S5 | Battery 20 % while avoiding | Avoidance finishes, then RETURNING (never NAVIGATING) | FR-05, 06 |
| TC-18 | CriticalBattery during DELIVERING | S3-S5 | Battery 20 % during hand-over | Hand-over completes, then RETURNING | FR-05, 06 |
| TC-19 | Vibration hold | S3-S5 | Apply vibration, then remove it | Risky activities suspended until 60 s below threshold | FR-07 |
| TC-20 | Batch removal interlock | S1-S5 | Request removal in protection mode, then when safe | Blocked in protection mode, allowed when safe | SR-03 |
| TC-21 | Safe Mode reset role | S1-S5 | Dispatcher, then Supervisor, sends reset | Dispatcher rejected, Supervisor accepted | FR-01, FR-06 |
| TC-22 | DestinationReached during AVOIDING_OBSTACLE | S3 | Inject the event while avoiding | Event latched. No jump to DELIVERING. After ObstacleCleared the robot goes to NAVIGATING and then DELIVERING | SR-04 |
| TC-23 | Link loss during AVOIDING / DELIVERING | S3-S5 | Cut link in each state | Avoidance and hand-over finish first, then deferred behaviour | SR-02 |
| TC-24 | Power loss | S3-S5 | Cut main power with and without emergency power | Emergency power switch, or incident recorded and safe shutdown | NFR-04, FR-07 |
| TC-25 | Usability trial | S1, S4 | 10 trained Dispatchers issue a command | <= 60 s, <= 5 clicks | NFR-05 |

## 4. Tests to Re-run per Change Request

| CR | Must re-run | Reason |
|---|---|---|
| CR-01 (FR-06) | **TC-07, TC-08**, TC-06, TC-17, TC-18, TC-21, TC-10, TC-16 | The timing and threshold logic changed. Neighbouring battery tests must show that critical and emergency levels still behave differently. Safe Mode reporting, logging and exit are affected. |
| CR-02 (FR-10) | **TC-11**, TC-10, TC-16, TC-01 | Capacity changed. Telemetry timing, logging and the command path must hold under 20 connections. |
| CR-03 (FR-01) | **TC-03 (new)**, TC-01, TC-02, TC-04, TC-16, TC-21, TC-11 | Every command path now has a role check. All command-issuing tests, plus log content, must be repeated. |

## 5. Multi-Layer RTM

Requirement -> State/Transition -> Operation -> Constraint -> Test Case -> Risk Level.

| Req. ID | State / Transition | Operation | Constraint | Test Case | Risk |
|---|---|---|---|---|---|
| FR-01 | T-02, T-03, T-30 | OP-R01, OP-R02 | C-15 | TC-01, 02, 03, 16, 21 | High |
| FR-02 | T-02, T-03 | OP-R03 | C-15 | TC-04 | Medium |
| FR-03 | T-01, T-02 | OP-R04, OP-R11 | C-15, C-16 | TC-01 | Medium |
| FR-04 | T-06, 07, 15, 18, 19, 20 | OP-R08 | C-11 | TC-05, 22 | Medium |
| FR-05 | T-08, 09, 10, 11, 14 (T-04, T-16 forbidden) | OP-R11, OP-R12 | C-12, C-16 | TC-01, 06, 12, 17, 18, 22 | High |
| FR-06 | T-12, 23 to 30 | OP-R07, OP-C08 | C-13 | TC-07, 08, 17, 18, 21 | High |
| FR-07 | CS-02 to CS-05, CS-08, CS-09, CS-10 | OP-C01, 02, 03, 04, 11, 12, 13, 14, 15, 16 | C-11, C-18 | TC-15, 19, 24 | High |
| FR-08 | CS-06, CS-07 | OP-C05 to C10 | C-13, C-17 | TC-09 | High |
| FR-09 | all transitions | OP-R05, OP-R06, OP-C18 | C-10, C-15 | TC-01, 10, 16 | Medium |
| FR-10 | none (session level) | OP-R13 | C-15 | TC-11 | Medium |
| NFR-01 | all | OP-R03, OP-R05 | none | TC-10 | Medium |
| NFR-02 | T-02, T-03 | OP-R01, OP-R02 | C-15 | TC-02, 03 | High |
| NFR-03 | none | OP-R13 | none | TC-11 | Medium |
| NFR-04 | CS-05, CS-10, CS-11 | OP-C15, C16, R06 | C-18 | TC-10, 24 | High |
| NFR-05 | none | OP-R05, OP-C10 | none | TC-25 | Medium |
| SR-01 | T-32 | OP-R09 | C-01 to C-09 | TC-13 | Critical |
| SR-02 | T-31 to T-40 | OP-R10 | C-07, C-14 | TC-14, 23 | Critical |
| SR-03 | CS-06 -> CS-07 | OP-C07, C08, C10, C17 | C-13, C-17 | TC-09, 20 | Critical |
| SR-04 | T-04 (forbidden), T-16 (forbidden), T-08 | OP-R11 | C-12, C-16 | TC-12, 22 | Critical |

### CR impact view

| CR | Requirement | Transitions | Operations | Constraints | Tests |
|---|---|---|---|---|---|
| CR-01 | FR-06 | T-12, T-23 to T-27 | OP-R07, OP-C08 | C-13 | TC-06, 07, 08, 10, 16, 17, 18, 21 |
| CR-02 | FR-10 (NFR-03) | none | OP-R13 | C-15 | TC-01, 10, 11, 16 |
| CR-03 | FR-01 | T-02, T-03, T-28, T-29, T-30 | OP-R01, OP-R02 | C-15 | TC-01, 02, 03, 04, 11, 16, 21 |
