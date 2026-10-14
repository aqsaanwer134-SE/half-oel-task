# Task 2 – Change Requests & Traceability Matrix (RTM)

## Description

After the baseline (v1.0) was approved, Mission Control sent three Change Requests. Change management means we never silently overwrite a requirement. For every CR the old text, new text and reason are recorded, a new version number is created, and the impact on other artefacts (tests, states, constraints) is identified. The RTM then shows, in one place, how each requirement flows through the lifecycle and which tests must be repeated (regression testing).

## 1. Version-Controlled Change Log

**What this table shows:** the history of each changed requirement. Each row is one version (v1.1 to v1.3), marked **Modified**, with the old wording, new wording and the reason. This is the audit trail that a real configuration-management tool (Git) would give us, and each CR corresponds to one commit.

| Version | Date | CR | Req. ID | Status | Old text | New text | Reason |
|---|---|---|---|---|---|---|---|
| v1.0 | Baseline | – | All | Approved | – | – | Baseline approved |
| v1.1 | 2026-10-14 | CR-01 | FR-07 | **Modified** | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | The robot shall enter Safe Mode within 3 seconds when chamber or battery temperature exceeds the critical threshold or battery capacity falls below the emergency level. | Old text had no time limit and no defined thresholds, so a late reaction could not be called a failure. |
| v1.2 | 2026-10-14 | CR-02 | NFR-02 | **Modified** | The system shall support communication with multiple robots simultaneously. | The system shall support at least 20 simultaneously connected robots. | "Multiple" has no number. The pilot serves 20 clinics, so 20 is the sizing target. |
| v1.3 | 2026-10-14 | CR-03 | FR-01 | **Modified** | Only authenticated operators shall be permitted to issue robot commands. | The system shall require authenticated and role-authorised operators before accepting any robot command. | Authentication alone does not stop an authenticated Viewer from issuing a dangerous command. Role-based authorisation was missing. |

Concrete values used for the CR-01 wording: critical chamber temperature below 0 °C or above 10 °C, critical battery temperature above 60 °C, emergency battery level 10 % (see README).

### Impact of each CR on other items

Impact analysis answers "what else must be updated because of this CR?". It prevents the common mistake of changing a requirement but forgetting its tests and constraints.

| CR | Affected artefacts |
|---|---|
| CR-01 | FR-07, T-10..T-14 (Safe Mode entry), CON-11, OP-07, TC-10, TC-11, TC-12 |
| CR-02 | NFR-02, NFR-01 (load figures), OP-08, TC-13, TC-14 |
| CR-03 | FR-01, FR-02, NFR-03, OP-01, OP-02, CON-13, TC-01, TC-02, TC-03 |

## 2. Why the New Wording Is More Measurable / Testable

**What this table shows:** for each CR, the weakness of the original (vague words such as *critical*, *multiple*, *authenticated*), what the new wording adds (a number, a time limit, a second condition) and a concrete way a tester can check it. A requirement is **measurable** if it contains a quantity, and **testable** if two engineers would reach the same pass/fail verdict.

| CR | Original weakness | What improved | How a tester checks it |
|---|---|---|---|
| CR-01 | "Critical" was undefined and there was no time bound. Two different test engineers could disagree on pass/fail. | Adds a **time limit (3 s)**, names the **three trigger quantities** (chamber temperature, battery temperature, battery capacity) and the **emergency level**. | Inject chamber temperature 10.5 °C at time t0 and read the timestamp when Safe Mode is reported. Pass if t − t0 ≤ 3 s. Repeat for battery temperature and battery 9 %. |
| CR-02 | "Multiple" can mean 2 or 200, so no test can fail it. | Gives a **numeric lower bound (20)** and the word "simultaneously connected" fixes the condition. | Connect 20 robot simulators at once, each sending 5 s telemetry, and verify all are connected and updated. Boundary test with 21 and 25 robots. |
| CR-03 | "Authenticated" only covers identity. Permission to do a particular command was not stated. | Two separate conditions (**authenticated AND role-authorised**) and a precise point of control ("before accepting any command"). | Test matrix of roles × commands: authenticated Admin/Dispatcher/Viewer against each command. Only allowed pairs are accepted. Unauthenticated is always rejected. |

## 3. Test Cases (scenario flow: command received → authenticated → executed → status reported → logged)


| TC ID | Scenario step | Description | Input | Expected result | Req. |
|---|---|---|---|---|---|
| TC-01 | Authenticated | Valid operator login | Correct credentials | Session created, role loaded | FR-01 |
| TC-02 | Command received → Authenticated | Command without session / wrong password | No session token; 5 wrong passwords | Command rejected (401), account locked after 5th failure, event logged | FR-01, NFR-03 |
| TC-03 | Authenticated → authorised | Viewer role sends DeliveryRequest | Viewer token + DeliveryRequest | Rejected (403), robot state unchanged, logged | FR-02, FR-01 |
| TC-04 | Command received → Executed | Dispatcher sends valid DeliveryRequest to IDLE robot | Valid command | ST-01 → ST-02 within 2 s | FR-03 |
| TC-05 | Status reported | Status after execution | After TC-04 | State NAVIGATING shown within link delay + 2 s; telemetry every 5 s | FR-10, NFR-01 |
| TC-06 | Logged | Log record for each step | After TC-04 | Four entries (received, authenticated, executed, reported) with timestamp, operator ID, robot ID, outcome | FR-10 |
| TC-07 | Executed (obstacle) | Obstacle appears during navigation and then clears | Obstacle at 3 m, then removed | T-02 then T-03, no delivery in between | FR-04 |
| TC-08 | Executed (delivery) | Destination reached | Position within 2 m | T-04 then T-05; package released only if chamber safe | FR-05, SAF-04 |
| TC-09 | Executed (battery) | Battery falls to 20 % during navigation | Battery profile 25 % → 19 % | T-06 to RETURNING, IDLE on arrival | FR-06 |
| TC-10 | Executed (Safe Mode) | Battery falls below emergency level | Battery 9 % | Safe Mode reported ≤ 3 s | FR-07 |
| TC-11 | Executed (Safe Mode) | Chamber temperature above critical | 10.5 °C | Safe Mode ≤ 3 s, alert, log | FR-07 |
| TC-12 | Executed (Safe Mode) | Battery temperature above critical | 61 °C | Safe Mode ≤ 3 s | FR-07 |
| TC-13 | Status reported | 20 simultaneous robots | 20 simulators, 30 min | All connected, telemetry received, validation ≤ 1 s for 95 % | NFR-02, NFR-01 |
| TC-14 | Status reported | Boundary above 20 | 21 and 25 simulators | System keeps all connected (≥ 20 satisfied) | NFR-02 |
| TC-15 | Executed (chamber) | Temperature out of range, correction, verification | 9 °C injected | Correction starts, in-range confirmed by sensor, otherwise protection mode after 120 s | FR-09 |
| TC-16 | Executed (crossing) | Train approaching, barrier closing | Train_Approaching = TRUE | Robot stops before crossing, alert, log | SAF-01 |
| TC-17 | Status reported (link) | Link cut for 15 s and 70 s | No heartbeat | LINK_LOST after 10 s, return after 60 s, data buffered | SAF-03, NFR-04 |
| TC-18 | Executed (door) | Door opened while NAVIGATING | Door sensor = OPEN | Alert ≤ 1 s, Safe Mode ≤ 3 s | SAF-02 |
| TC-19 | Executed (self-check) | Power-on with failed vibration sensor | Sensor fault | Monitoring not entered, alert | FR-08 |
| TC-20 | Authenticated (security) | TLS and session expiry | Plain HTTP attempt; 16 min idle | Plain refused, session expired | NFR-03 |
| TC-21 | Status reported (usability) | Operator usability study | 10 operators | ≤ 4 interactions, ≤ 60 s, ≥ 90 % success | NFR-05 |
| TC-22 | Logged (reliability) | Power loss and link loss | 100 trials | 0 lost log entries, buffered data uploaded | NFR-04 |
| TC-23 | Executed (crossing sensor fault) | Train sensor fault / inconsistent reading | Train_Sensor_Fault = TRUE | Crossing_Permission = FALSE, robot holds, alert | SAF-01 |
| TC-24 | Executed (illegal transition) | Deliver command in IDLE and in AVOIDING_OBSTACLE | Illegal events | Rejected, state unchanged, logged | SAF-04 |

## 4. Multi-Layer RTM

| Req. ID | State / Transition | Operation | Constraint | Test Case | Risk |
|---|---|---|---|---|---|
| FR-01 (v1.3) | Precondition of T-01 | OP-01, OP-02 | CON-13 | TC-01, TC-02, TC-03, TC-20 | High |
| FR-02 | T-01, rejected: T-25, T-28..T-31 | OP-02, OP-03 | CON-13 | TC-03, TC-24 | High |
| FR-03 | ST-01 → ST-02 (T-01) | OP-03, OP-04 | CON-13, CON-14 | TC-04, TC-05 | Medium |
| FR-04 | ST-02 ↔ ST-03 (T-02, T-03) | OP-06 | CON-10 | TC-07 | High |
| FR-05 | T-04, T-05 (ST-04) | OP-03, OP-22 | CON-08, CON-09 | TC-08 | High |
| FR-06 | T-06, T-07, T-08 | OP-07 | CON-15 | TC-09 | High |
| FR-07 (v1.1) | T-10..T-14, T-22 (ST-06) | OP-07, OP-23 | CON-11 | TC-10, TC-11, TC-12 | High |
| FR-08 | CT-01, CT-02 | OP-10 | CON-16 | TC-19 | High |
| FR-09 | CT-04, CT-05, CT-06 | OP-14, OP-15, OP-16, OP-17 | CON-12 | TC-15 | High |
| FR-10 | All transitions | OP-04, OP-05 | CON-14 | TC-05, TC-06 | Medium |
| NFR-01 | – | OP-02, OP-04 | – | TC-05, TC-13 | Medium |
| NFR-02 (v1.2) | – | OP-08 | – | TC-13, TC-14 | Medium |
| NFR-03 | – | OP-01, OP-02 | CON-13 | TC-02, TC-20 | High |
| NFR-04 | T-17..T-24 (ST-07) | OP-05, OP-08 | CON-14 | TC-17, TC-22 | High |
| NFR-05 | – | OP-04, OP-23 | – | TC-21 | Low |
| SAF-01 | T-18, T-19 (hold before crossing) | OP-09 | CON-01..CON-07 | TC-16, TC-23 | High |
| SAF-02 | T-11..T-14, CT-08 | OP-19, OP-22, OP-17 | CON-08, CON-10 | TC-18 | High |
| SAF-03 | T-17..T-24 | OP-08 | CON-06 | TC-17 | High |
| SAF-04 | T-09, invalid T-26, T-27 | OP-03 | CON-09, CON-10 | TC-08, TC-24 | High |

## 5. Regression: Test Cases to Re-run per CR

## Task 2 Summary

| Item | Result |
|---|---|
| Requirements modified | 3 (FR-07, NFR-02, FR-01) |
| New versions created | v1.1, v1.2, v1.3 |
| Test cases defined | 24 |
| Tests re-run for CR-01 / CR-02 / CR-03 | 8 / 5 / 7 |
| Requirements in RTM | 19 (100 % traced to a test) |

| Test Case | CR-01 | CR-02 | CR-03 | Reason |
|---|---|---|---|---|
| TC-01 | – | – | **Yes** | Login is part of the new authentication and authorisation check |
| TC-02 | – | – | **Yes** | Unauthenticated rejection now belongs to the new wording |
| TC-03 | – | – | **Yes** | Role authorisation is the core of CR-03 |
| TC-04 | – | **Yes** | **Yes** | Command acceptance path changed (CR-03). Concurrent commands from 20 robots (CR-02) |
| TC-05 | **Yes** | **Yes** | – | Safe Mode must be reported (CR-01). Status delay under 20 robots (CR-02) |
| TC-06 | **Yes** | **Yes** | **Yes** | Log must contain Safe Mode events, load volume and authorisation outcomes |
| TC-09 | **Yes** | – | – | Critical (20 %) vs emergency (10 %) battery split must stay consistent |
| TC-10 | **Yes** | – | – | Directly tests CR-01 (battery emergency, 3 s) |
| TC-11 | **Yes** | – | – | Directly tests CR-01 (chamber temperature, 3 s) |
| TC-12 | **Yes** | – | – | Directly tests CR-01 (battery temperature, 3 s) |
| TC-13 | – | **Yes** | – | Directly tests CR-02 |
| TC-14 | – | **Yes** | – | Boundary of CR-02 |
| TC-17 | **Yes** | – | – | Safe Mode vs LINK_LOST priority (T-22) |
| TC-18 | **Yes** | – | – | Door-open leads to Safe Mode in 3 s |
| TC-20 | – | – | **Yes** | Security wording change |
| TC-24 | – | – | **Yes** | Illegal commands must still be rejected by authorisation path |
| TC-07, TC-08, TC-15, TC-16, TC-19, TC-21, TC-22, TC-23 | – | – | – | Not affected (no re-run needed) |
