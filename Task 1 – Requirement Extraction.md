# Task 1 – Requirement Extraction



## Description

The scenario describes a fleet of autonomous robots that carry vaccines in a sealed conservation chamber from a warehouse to remote clinics. Mission Control supervises the robots over a slow link (delay of several seconds). The scenario contains many sentences, but not every sentence is a usable requirement. As Software Quality Engineering Lead I did the following:


## 1. Functional Requirements (10)

**What this table shows:** the behaviour the system must perform. FR-01 and FR-02 cover the command chain (authentication, validation). FR-03 to FR-06 cover the robot life-cycle (navigation, obstacle, delivery, battery). FR-07 covers Safe Mode. FR-08 and FR-09 cover the chamber (self-check, correction with protection). FR-10 covers telemetry and logging.

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| FR-01 | Only authenticated operators shall be permitted to issue robot commands. | FR | High | Test (valid/invalid login, command without session) |
| FR-02 | The system shall validate every received command (operator role permission, command syntax, target robot ID exists, timestamp not older than 30 s) and shall reject an invalid command with an error code, without changing the robot state. | FR | High | Test (negative test set per validation rule) |
| FR-03 | The robot shall accept a valid DeliveryRequest only in IDLE, shall enter NAVIGATING within 2 s of acceptance and shall report the new state to Mission Control. | FR | High | Test (timed, instrumented robot simulator) |
| FR-04 | While NAVIGATING the robot shall check for obstacles at least every 100 ms. On detection it shall suspend navigation and enter AVOIDING_OBSTACLE, and it shall resume NAVIGATING when the obstacle is cleared. Package delivery shall not start while the obstacle is being handled. | FR | High | Test + Simulation (obstacle injection) |
| FR-05 | The robot shall enter DELIVERING only from NAVIGATING on DestinationReached (within 2 m of the destination). It shall release the package only after the chamber is verified safe, and on completion it shall enter RETURNING. | FR | High | Test + Model check of transition table |
| FR-06 | If battery capacity falls to 20 % or lower (critical) during NAVIGATING or AVOIDING_OBSTACLE, the robot shall abandon the delivery and enter RETURNING, and shall enter IDLE on arrival at the warehouse. | FR | High | Test (battery profile injection) |
| FR-07 | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | FR | High | Test (fault injection) |
| FR-08 | At power-on the chamber shall self-check temperature, humidity, vibration, door and power sensors and shall enter monitoring only if all pass. If any fails, it shall raise an operator alert and block conservation. | FR | High | Test (sensor fault injection) |
| FR-09 | When temperature or humidity leaves the loaded batch limits, the chamber shall start correction and confirm by sensor readings that the value is back in range. If not confirmed within 120 s it shall enter protection mode, reduce light exposure and alert the operator. | FR | High | Test + Simulation (thermal model) |
| FR-10 | The robot shall send telemetry (state, temperature, humidity, vibration, door, battery, position) every 5 s. The system shall log each command event (received, authenticated, executed, status reported) with timestamp, operator ID, robot ID and outcome. | FR | Medium | Test + Inspection of log records |

## 2. Non-Functional Requirements (5)

**What this table shows:** the quality attributes that decide *how well* the system works. Each NFR is tied to one of the five categories requested (performance, scalability, security, reliability, usability) and has a measurable limit (seconds, percentages, counts) so that a test can decide pass or fail.

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| NFR-01 (Performance) | Server-side authentication, authorisation and validation of a command shall complete in at most 1 s for 95 % of commands under a 20-robot load. Status shall appear on the operator console within (link delay + 2 s). | NFR | High | Performance test (load script, timestamps) |
| NFR-02 (Scalability) | The system shall support communication with multiple robots simultaneously. | NFR | Medium | Load test |
| NFR-03 (Security) | Command and telemetry channels shall use TLS 1.2 or higher. An account shall lock after 5 failed logins, a session shall expire after 15 min of inactivity and passwords shall be stored hashed with salt. | NFR | High | Security test + Inspection |
| NFR-04 (Reliability) | No committed log entry shall be lost (0 lost entries in 100 power-loss/link-loss trials). A robot shall buffer at least 24 h of telemetry during link loss and upload it on restoration. Mission Control availability shall be at least 99.5 % per month. | NFR | High | Fault-injection test + Analysis of uptime logs |
| NFR-05 (Usability) | A trained operator shall issue a delivery request in at most 4 interactions and 60 s. Critical alerts shall be shown in colour with sound within 2 s of arrival. At least 90 % of test operators shall finish the task without help. | NFR | Low | Usability test with 10 operators |

Note: NFR-02 is deliberately kept at baseline wording (not measurable); CR-02 makes it measurable.

## 3. High-Risk Requirements (4)

**What this table shows:** the requirements where a failure could cause physical harm or loss of medicine. They are *safety-critical* (railway crossing, door interlock), *real-time* (Safe Mode timing, handled in FR-07/CR-01) and *link-dependent* (link loss). These four receive the strongest verification effort: model checking, fault injection and simulation.

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| SAF-01 | The robot shall enter the railway level-crossing only when ARLCCS reports barrier open AND no train approaching, with a status message not older than 2 s. Otherwise it shall stop before the crossing and alert the operator. | Safety | High | Test + Simulation (train/barrier scenarios) |
| SAF-02 | The chamber door shall stay locked unless the robot is stopped with handover authorised. If the door opens in any other situation, the system shall alert within 1 s and enter Safe Mode within 3 s. | Safety | High | Test (door fault injection) |
| SAF-03 | If no heartbeat is received for 10 s, the robot shall enter LINK_LOST, stop at a safe location (never on the crossing), keep monitoring the chamber and buffer data. After 60 s without link it shall return autonomously to the warehouse. | Safety | High | Test (link cut-off) + Simulation |
| SAF-04 | The robot shall never begin DELIVERING directly from IDLE and never while in AVOIDING_OBSTACLE. Such requests shall be rejected and logged. | Safety | High | Model check + Test (illegal event injection) |

## Summary

The baseline contains **19 requirements**. All High-priority items are verified by test plus model check or fault injection. Three baseline statements (FR-01, FR-07, NFR-02) are intentionally the least precise ones, and they are the ones Mission Control improves in Task 2.

| Category | Count |
|---|---|
| Functional | 10 |
| Non-functional | 5 |
| High-risk (Safety) | 4 |
| **Total** | **19** |


