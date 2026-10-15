# Task 5: Constraints, Formalisation & Violations

Short names: TP = Train_Present, TA = Train_Approaching, BO = Barrier_Open, REC = Robot_Entering_Crossing

## 1 + 2. Constraints and formal notation

| ID | Constraint (simple English) | Why necessary | Formal |
|---|---|---|---|
| C-01 | When a train is present or approaching, the barrier must not be open | Prevent collision with the train | (TP ∨ TA) → ¬BO |
| C-02 | The robot may enter the crossing only if the barrier is open and no train is approaching | Main safety rule for robot (HR-01) | REC → (BO ∧ ¬TA) |
| C-03 | The barrier may open only when the crossing is confirmed clear | Opening early puts traffic in front of the train | Barrier_Opening → (Crossing_Clear ∧ ¬TP) |
| C-04 | When a train is approaching, warning lights and alarm must be on | Warns people and robots | TA → (Lights_On ∧ Alarm_On) |
| C-05 | If the train sensor fails, the barrier must stay closed and the operator alerted | We cannot know if a train is coming (sensor failure) | Train_Sensor_Fail → (¬BO ∧ Operator_Alert) |
| C-06 | If the robot has no ARLCCS status, it must not enter the crossing | Robot cannot check safety (communication loss) | ARLCCS_Link_Lost → ¬REC |
| C-07 | If the two train sensors disagree, the barrier stays closed | A wrong reading must not open the crossing | (Sensor_A ≠ Sensor_B) → ¬BO |
| C-08 | While the robot is moving, the chamber door must be closed (robot-chamber) | Open door spoils the batch | Robot_State ∈ {NAVIGATING, AVOIDING_OBSTACLE, RETURNING} → ¬Door_Open |
| C-09 | In DELIVERING the door may be open only if the handover is authorised (robot-chamber) | Stops unauthorised access to medicine | (Robot_State = DELIVERING ∧ ¬Release_Authorised) → ¬Door_Open |
| C-10 | Within 3 s of critical temperature or emergency battery, the robot must be in SAFE_MODE | CR-01 | ((Temp > T_crit ∨ Batt < 10) ∧ t > 3) → Robot_State = SAFE_MODE |
| C-11 | If the Mission Control link is lost for more than 60 s, the robot must not be NAVIGATING (link loss) | Robot must not drive unsupervised | Link_Lost_Time > 60 → ¬(Robot_State = NAVIGATING) |
| C-12 | A batch may be released only if the chamber is safe and no protection response is active | Scenario rule for removal | Batch_Release → (Chamber_Safe ∧ ¬Protection_Active) |
| C-13 | A command may be executed only for an authenticated and role-authorised operator | CR-03 | Command_Executed → (Authenticated ∧ Role_Authorised) |

## 3 + 4. Violation scenarios, detection and response

| ID | Violation (values) | How detected | Expected response |
|---|---|---|---|
| C-01 | TP = TRUE, BO = TRUE | Safety unit compares train sensor and barrier position | Close barrier, alarm, stop robot, alert operator, log |
| C-02 | REC = TRUE, BO = FALSE, TA = TRUE | Robot checks ARLCCS status at the stop line | Robot emergency stop, Safe Mode if already on tracks, alert operator, log |
| C-03 | Barrier_Opening = TRUE, Crossing_Clear = FALSE | Safety unit checks clear status before opening | Reject open command, barrier stays closed, alert operator, log |
| C-04 | TA = TRUE, Lights_On = TRUE, Alarm_On = FALSE | Alarm output feedback check | Switch to backup alarm, alert operator, log |
| C-05 | Train_Sensor_Fail = TRUE, BO = TRUE | Sensor heartbeat missing for 2 s | Close barrier (fail-safe), robot waits, alert operator, log |
| C-06 | ARLCCS_Link_Lost = TRUE, REC = TRUE | No ARLCCS message for 2 s | Robot stops before the line, alert operator, log |
| C-07 | Sensor_A = train, Sensor_B = no train, BO = TRUE | Cross-check of the two sensors | Treat as train present, close barrier, alarm, alert operator, log |
| C-08 | Robot_State = NAVIGATING, Door_Open = TRUE | Door sensor | Chamber suspends (OP-12), robot goes to SAFE_MODE, alert operator, log |
| C-09 | Robot_State = DELIVERING, Release_Authorised = FALSE, Door_Open = TRUE | Door sensor and authorisation flag | Alert operator, robot to SAFE_MODE, log |
| C-10 | Temp = 9.5 C (limit 8 C), t = 4 s, Robot_State = NAVIGATING | Safety monitor timer | Force SAFE_MODE, alert operator, log the delay as a defect |
| C-11 | Link_Lost_Time = 75 s, Robot_State = NAVIGATING | Link timer in robot | Force RETURNING, sync log after link returns |
| C-12 | Batch_Release = TRUE, Protection_Active = TRUE | Guard inside ReleaseBatch | Refuse release, door stays locked, alert operator, log |
| C-13 | Command_Executed = TRUE, Authenticated = TRUE, Role_Authorised = FALSE | Command gate and audit log | Reject command, show access denied, log the rejection |

Robot-chamber constraints: C-08, C-09, C-12. Sensor failure or communication loss: C-05, C-06, C-07, C-11.
