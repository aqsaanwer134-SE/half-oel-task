# Task 3: State Modelling & Transition Verification

## 1. States

| ID | State | Meaning | Entry condition | Exit condition |
|---|---|---|---|---|
| ST-01 | IDLE | Powered on, waiting at the warehouse, no mission | PowerOn with self-check passed; WarehouseReached; SafeModeReset; LinkRestored (from IDLE) | Validated DeliveryRequest; SafeModeTrigger; LinkLost |
| ST-02 | NAVIGATING | Travelling to the clinic | Accepted DeliveryRequest; ObstacleCleared (Resume = NAV); LinkRestored with operator confirmation | DestinationReached; ObstacleDetected; CriticalBattery; SafeModeTrigger; LinkLost |
| ST-03 | AVOIDING_OBSTACLE | Navigation suspended while the obstacle is handled | ObstacleDetected from NAVIGATING or RETURNING | ObstacleCleared; SafeModeTrigger |
| ST-04 | DELIVERING | Docked at the clinic bay, handing over the batch | DestinationReached from NAVIGATING (door closed, chamber OK) | DeliveryComplete; SafeModeTrigger |
| ST-05 | RETURNING | Travelling back to the warehouse | DeliveryComplete; CriticalBattery; LinkTimeout; RecoveryCommand | WarehouseReached; ObstacleDetected; SafeModeTrigger |
| ST-06 | SAFE_MODE (new, CR-01) | Motion stopped at a safe place, chamber maintained, operator alerted | SafeModeTrigger | SafeModeReset (Supervisor) or RecoveryCommand (Supervisor) |
| ST-07 | LINK_LOST (new) | Holding at a safe point with no supervision | LinkLost from IDLE or NAVIGATING (or deferred from AVOIDING_OBSTACLE) | LinkRestored; LinkTimeout; SafeModeTrigger |

Initial state: OFF, which becomes IDLE on PowerOn. The set ST-01 to ST-05 is the scenario model. ST-06 and ST-07 are the extension.

## 2. Events and Conditions

| ID | Event / condition | Description |
|---|---|---|
| E-01 | PowerOn | Robot switched on |
| E-02 | DeliveryRequest | Authenticated, authorised, valid command to deliver |
| E-03 | DestinationReached | GPS and docking confirm arrival at the clinic |
| E-04 | ObstacleDetected | Sensors report an obstacle on the path |
| E-05 | ObstacleCleared | Path clear again |
| E-06 | DeliveryComplete | Clinic confirms receipt |
| E-07 | WarehouseReached | Arrival at the warehouse |
| E-08 | CriticalBattery | Battery <= 20 % |
| E-09 | SafeModeTrigger | EmergencyBattery (< 10 %), critical temperature, critical fault, unexpected door open outside delivery, or chamber protection mode |
| E-10 | LinkLost | No heartbeat for 30 s |
| E-11 | LinkRestored | Heartbeat back |
| E-12 | LinkTimeout | 120 s in LINK_LOST |
| E-13 | SafeModeReset / RecoveryCommand | Authorised Supervisor command |

**State vs event.**
A **state** is a condition the robot remains in for a period of time. It has entry and exit conditions and the robot is "in" it. An **event** is an instantaneous occurrence (a signal, a command, a threshold crossing) that happens at one moment and may trigger a transition. "NAVIGATING" is a state. "DestinationReached" is an event. The same event can lead to different results depending on the current state, which is why the behaviour is modelled as *Current State + Event [guard] -> Next State*.

## 3. State Transition Table

Valid? legend: **Yes**, **Yes (safe)** = event deliberately ignored or deferred, **NO** = forbidden transition that must not exist. Tests are defined in Task 2.

| T-ID | Current state | Event [guard] | Next state | Req. | Valid? | Verification evidence |
|---|---|---|---|---|---|---|
| T-01 | OFF | PowerOn [self-check passed] | IDLE | FR-03 | Yes | TC-01 precondition |
| T-02 | IDLE | DeliveryRequest [auth and role OK, valid clinic, battery > 20 %, chamber OK] | NAVIGATING | FR-01, 02, 03 | Yes | TC-01 |
| T-03 | IDLE | DeliveryRequest [any guard false] | IDLE (rejected, logged) | FR-01, 02 | Yes | TC-02, 03, 04 |
| T-04 | IDLE | DestinationReached | **DELIVERING** | SR-04, FR-05 | **NO** | TC-12; model check: must be unreachable |
| T-05 | IDLE | DestinationReached | IDLE (ignored, logged) | SR-04 | Yes (safe) | TC-12 |
| T-06 | NAVIGATING | ObstacleDetected | AVOIDING_OBSTACLE [Resume = NAV] | FR-04 | Yes | TC-05A |
| T-07 | AVOIDING_OBSTACLE | ObstacleCleared [Resume = NAV, no Abandon, no LinkPending] | NAVIGATING | FR-04 | Yes | TC-05A |
| T-08 | NAVIGATING | DestinationReached [door closed, chamber OK] | DELIVERING | FR-05, SR-04 | Yes | TC-01 end to end |
| T-09 | DELIVERING | DeliveryComplete | RETURNING | FR-05 | Yes | TC-01 end to end |
| T-10 | RETURNING | WarehouseReached | IDLE | FR-05 | Yes | TC-06 |
| T-11 | NAVIGATING | CriticalBattery | RETURNING (delivery abandoned) | FR-05, 06 | Yes | TC-06 |
| T-12 | NAVIGATING | SafeModeTrigger | SAFE_MODE (<= 3 s) | FR-06 | Yes | TC-07, 08 |
| T-13 | AVOIDING_OBSTACLE | CriticalBattery | AVOIDING_OBSTACLE [Abandon = 1] | FR-06 | Yes (safe, **added**) | TC-17 |
| T-14 | AVOIDING_OBSTACLE | ObstacleCleared [Abandon = 1] | RETURNING | FR-05, 06 | Yes (**added**) | TC-17 |
| T-15 | AVOIDING_OBSTACLE | DestinationReached | AVOIDING_OBSTACLE (latched) | SR-04 | Yes (safe) | TC-22 |
| T-16 | AVOIDING_OBSTACLE | any event | **DELIVERING** | SR-04 | **NO** | TC-22; model check |
| T-17 | DELIVERING | CriticalBattery | DELIVERING [finish hand-over <= 60 s], then T-09 | FR-05, 06 | Yes (**added**) | TC-18 |
| T-18 | DELIVERING | ObstacleDetected | DELIVERING (ignored, robot docked, logged) | FR-04 | Yes (safe) | Model review |
| T-19 | RETURNING | ObstacleDetected | AVOIDING_OBSTACLE [Resume = RETURN] | FR-04 | Yes (**added**) | TC-05B |
| T-20 | AVOIDING_OBSTACLE | ObstacleCleared [Resume = RETURN] | RETURNING | FR-04 | Yes (**added**) | TC-05B |
| T-21 | RETURNING | CriticalBattery | RETURNING (no change, logged) | FR-06 | Yes | TC-06 |
| T-22 | IDLE | CriticalBattery | IDLE (new requests rejected) | FR-03, 06 | Yes | TC-06 |
| T-23 | IDLE | SafeModeTrigger | SAFE_MODE | FR-06 | Yes | TC-08 |
| T-24 | AVOIDING_OBSTACLE | SafeModeTrigger | SAFE_MODE | FR-06 | Yes | TC-07 |
| T-25 | DELIVERING | SafeModeTrigger | SAFE_MODE (door locked first) | FR-06 | Yes | TC-07 |
| T-26 | RETURNING | SafeModeTrigger | SAFE_MODE | FR-06 | Yes | TC-07 |
| T-27 | LINK_LOST | SafeModeTrigger | SAFE_MODE | FR-06 | Yes | TC-08 |
| T-28 | SAFE_MODE | SafeModeReset [Supervisor, trigger cleared, at warehouse] | IDLE | FR-01, 06 | Yes | TC-21 |
| T-29 | SAFE_MODE | RecoveryCommand [Supervisor, battery > 20 %, thermal OK, door closed] | RETURNING | FR-01, 06 | Yes | TC-21 |
| T-30 | SAFE_MODE | DeliveryRequest | SAFE_MODE (rejected, logged) | FR-01, 06 | Yes | TC-03 |
| T-31 | IDLE | LinkLost | LINK_LOST | SR-02 | Yes | TC-14 |
| T-32 | NAVIGATING | LinkLost | LINK_LOST (stop at safe holding point, not on railway) | SR-02, SR-01 | Yes | TC-14 |
| T-33 | RETURNING | LinkLost | RETURNING (autonomous, link flag set) | SR-02 | Yes | TC-23 |
| T-34 | AVOIDING_OBSTACLE | LinkLost | AVOIDING_OBSTACLE [LinkPending = 1] | SR-02 | Yes (safe) | TC-23 |
| T-35 | AVOIDING_OBSTACLE | ObstacleCleared [Resume = NAV, LinkPending] | LINK_LOST | SR-02 | Yes | TC-23 |
| T-36 | DELIVERING | LinkLost | DELIVERING (hand-over completes locally) | SR-02 | Yes | TC-23 |
| T-37 | LINK_LOST | LinkRestored [came from IDLE] | IDLE | SR-02 | Yes | TC-14 |
| T-38 | LINK_LOST | LinkRestored [came from NAV, operator ResumeConfirm] | NAVIGATING | SR-02 | Yes | TC-14 |
| T-39 | LINK_LOST | LinkTimeout (120 s) | RETURNING | SR-02 | Yes | TC-14 |
| T-40 | SAFE_MODE | LinkLost | SAFE_MODE (link flag, stays safe) | SR-02, FR-06 | Yes | Model review |

## 4. Extension: Safe Mode and Link Lost

**Which states may enter SAFE_MODE?** All of them (IDLE, NAVIGATING, AVOIDING_OBSTACLE, DELIVERING, RETURNING, LINK_LOST). A critical thermal or emergency battery condition can occur anywhere, and CR-01 requires a response within 3 s. In DELIVERING the door is locked first, then the robot stops. SAFE_MODE can be left only by an authorised Supervisor.

**Which states may enter LINK_LOST?**

| State | Enters LINK_LOST? | Why |
|---|---|---|
| IDLE | Yes (T-31) | It cannot receive requests, so it must show degraded status. |
| NAVIGATING | Yes (T-32) | The robot is unsupervised and must not continue towards a crossing. It holds at a safe point. |
| AVOIDING_OBSTACLE | Deferred (T-34, T-35) | A half-finished manoeuvre must not be abandoned. It enters LINK_LOST after ObstacleCleared. |
| DELIVERING | No (T-36) | The hand-over is local and needs no link. Stopping would leave the batch exposed. |
| RETURNING | No (T-33) | The return is already autonomous, so the robot continues. |
| SAFE_MODE | No (T-40) | It is already safe. |

**Design decision on conflict:** the scenario's CriticalBattery rule (return) and CR-01's emergency rule (Safe Mode) are two thresholds, 20 % and 10 %, as stated in Task 2 Note 1.

## 5. Verifier Investigation

### (a) Can IDLE -> DELIVERING happen? Which requirement is violated?
In the correct table, no. There is no such row (T-04 is marked forbidden and T-05 ignores the event). It can happen in an implementation if the DestinationReached handler is not guarded by the current state (for example a stale or spoofed destination signal arriving while IDLE), or if DeliveryRequest is implemented as "go to DELIVERING" when the destination equals the current location.
**Violated:** the scenario rule "a waiting robot must never begin delivering directly", i.e. **SR-04**, and **FR-05** (DELIVERING only from NAVIGATING). Constraints **C-16**. The robot could release or expose a batch without any journey, authorisation or chamber check.

### (b) What if AVOIDING_OBSTACLE has no transition back? Can the robot still deliver?
No. The robot **cannot deliver** because DELIVERING is reachable only from NAVIGATING, and the only way into NAVIGATING from AVOIDING_OBSTACLE is ObstacleCleared. This is safe against wrong delivery but it is a **liveness failure (trap state / deadlock)**: the robot stays stationary, the batch warms up, the battery drains, and the mission never completes. Even CriticalBattery cannot help if that transition is also missing.
**Fix:** guaranteed exits: ObstacleCleared (T-07, T-14, T-20, T-35), SafeModeTrigger (T-24), plus an **obstacle timeout** (for example 120 s) that raises SafeModeTrigger and alerts the operator. Verification: reachability analysis that every state can reach IDLE or SAFE_MODE.

### (c) Can AVOIDING_OBSTACLE -> DELIVERING occur?
It must not (T-16 forbidden). A realistic cause: the robot is near the destination when the obstacle appears, DestinationReached fires, and an unguarded handler delivers. This violates "must never deliver while handling an obstacle" (**SR-04**). **Correct behaviour:** the event is **latched** (T-15), the robot finishes avoidance, returns to NAVIGATING (T-07), and DestinationReached is processed there (T-08). This is tested by TC-22.

### (d) CriticalBattery in AVOIDING_OBSTACLE or DELIVERING (missing transition)
With no transition, the event is silently dropped or behaviour is undefined. The robot keeps avoiding or delivering on a nearly empty battery, may stop in the field with medicine inside, and the chamber loses power. This breaks **FR-06** and **NFR-04**.
**Added behaviour:**
- **AVOIDING_OBSTACLE:** set Abandon = 1, finish the manoeuvre, then go to RETURNING (T-13, T-14). It must **not** go to NAVIGATING and **not** to DELIVERING.
- **DELIVERING:** finish the short hand-over (limit 60 s) so the batch is not left half-delivered, then RETURNING (T-17, T-09). If the battery falls to the emergency level, SafeModeTrigger applies (T-25).

### Additional behavioural problems found
1. **Missing transition:** RETURNING + ObstacleDetected was undefined. Without it the robot drives into the obstacle. Added T-19, T-20 with a Resume variable so that avoidance returns to RETURNING, not NAVIGATING (otherwise the robot could turn back to the clinic).
2. **Conflict:** scenario "critical battery -> return" vs CR-01 "emergency -> Safe Mode" (resolved by two thresholds).
3. **Unsafe stop:** LINK_LOST or SAFE_MODE could stop the robot on the railway. Stops must happen at a holding point >= 5 m away (C-09, C-14).
4. **Completeness rule:** every state-event pair must have a defined behaviour, even if the behaviour is "ignore and log". Unhandled pairs are the source of (a) to (d).
5. **Door during motion:** an unexpected door opening in any moving state is a SafeModeTrigger (C-11).
