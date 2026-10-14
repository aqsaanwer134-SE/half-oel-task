# Task 1: Requirement Extraction (Baseline v1.0)

Format: **Req. ID | Requirement | Type | Priority | Verification Method**
Verification methods: **T** = Test, **S** = Simulation / fault injection, **I** = Inspection, **A** = Analysis, **D** = Demonstration.
Thresholds come from the global assumptions in `README.md`.

## 1. Functional Requirements (10)

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| FR-01 | Only authenticated operators shall be permitted to issue robot commands. A command from an unauthenticated session shall be rejected, the robot state left unchanged, and the rejection logged. | FR | High | T: negative tests with no, expired and forged tokens |
| FR-02 | After authentication, the robot shall validate every command (known command type, destination is one of the 20 registered clinics, parameters in range). An invalid command shall be rejected with a reason code and no state change. | FR | Medium | T: boundary and invalid-parameter tests |
| FR-03 | On a validated DeliveryRequest while IDLE with battery above CriticalBattery, the robot shall enter NAVIGATING and send a status report to Mission Control. | FR | High | T: state-transition test |
| FR-04 | While NAVIGATING or RETURNING, on obstacle detection the robot shall suspend navigation and enter AVOIDING_OBSTACLE. When the obstacle is cleared it shall resume the previous activity. | FR | Medium | T, S: obstacle injection |
| FR-05 | The robot shall enter DELIVERING only from NAVIGATING after DestinationReached. After DeliveryComplete it shall travel back to the warehouse and become IDLE on arrival. | FR | High | T, A: transition analysis |
| FR-06 | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | FR | High | S: fault injection |
| FR-07 | The chamber shall run a self-check at power-on and enter monitoring only if all essential sensors pass. It shall start conservation only when the door is closed and the batch profile is loaded. On door opening during conservation it shall suspend normal operation and resume only after conditions and sensors are verified. | FR | High | T, D |
| FR-08 | On out-of-range temperature or humidity the chamber shall start correction and confirm from sensor readings that the value is back in range. If it is not back in range within the recovery period, the chamber shall enter protection mode, reduce light exposure and alert the operator. | FR | High | S: heater/cooler fault injection |
| FR-09 | For each command the system shall report status to Mission Control and write a log record containing timestamp, robot ID, operator ID, command ID, authentication result, execution result and resulting state. | FR | Medium | T, I: log inspection |
| FR-10 | The system shall support communication with multiple robots simultaneously. | FR | High | T: multi-robot load test |

## 2. Non-Functional Requirements (5)

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| NFR-01 (Performance) | Authentication, authorisation and validation of a command shall finish in <= 500 ms on the robot. The acknowledgement and every state-change report shall reach Mission Control within <= 10 s of the triggering event with a one-way link delay of up to 5 s. | NFR | High | T: timed tests with delay emulator |
| NFR-02 (Security) | All command and telemetry traffic shall be encrypted (TLS 1.2 or higher). Operator sessions shall expire after 15 min of inactivity. 100 % of accepted commands shall carry a valid signed session token. | NFR | High | T, I: protocol capture and review |
| NFR-03 (Scalability) | Mission Control shall process telemetry from 20 robots (1 message per 10 s each) with command acknowledgement <= 10 s and CPU <= 70 %, and shall grow to 40 robots by configuration only. | NFR | Medium | T: load test with 20 and 40 simulated robots |
| NFR-04 (Reliability) | Mission Control availability shall be >= 99.5 % per month. During a link outage of up to 4 h the robot shall buffer 100 % of chamber and event records and upload them in order on reconnection. Chamber temperature shall stay within range for >= 99.5 % of mission time. | NFR | High | A: availability analysis; T: link-outage test |
| NFR-05 (Usability) | After 2 h of training, a Dispatcher shall issue a delivery command and see its confirmation in <= 60 s with <= 5 clicks. Critical alerts shall appear in a distinct colour within 2 s of receipt. | NFR | Medium | T: usability trial, 10 operators |

## 3. High-Risk (Safety-Critical) Requirements (4)

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| SR-01 (Railway crossing) | The robot shall enter the level crossing only if the latest ARLCCS message (age <= 2 s) reports Barrier_Open = TRUE and no train approaching or present. Otherwise it shall stop >= 5 m before the crossing and request clearance again. | Safety | Critical | S: ARLCCS simulator with stale, failed and conflicting inputs |
| SR-02 (Link-dependent) | If no heartbeat arrives for 30 s, the robot shall enter LINK_LOST, stop at a safe holding point (never on or within 5 m of the railway), keep the chamber controlled, and start an autonomous return to the warehouse after 120 s. | Safety | Critical | S: link-cut tests in each state |
| SR-03 (Chamber protection) | If correction fails within the recovery period (10 min), the chamber shall enter protection mode and alert the operator within 5 s. Batch removal shall be blocked while protection is active. | Safety | Critical | S: fault injection |
| SR-04 (Delivery guard) | The robot shall never begin DELIVERING from IDLE or from AVOIDING_OBSTACLE. DELIVERING shall be reachable only from NAVIGATING at the destination. | Safety | Critical | A: model checking; T: negative transition tests |

Total: 10 FR + 5 NFR + 4 high-risk = **19 requirements**.
