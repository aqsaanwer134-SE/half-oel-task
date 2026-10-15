# Task 4: Operation Identification & Operation Schema (Conservation Chamber)

## 1. Operations

| Op ID | Operation |
|---|---|
| OP-01 | RunSelfCheck |
| OP-02 | LoadBatchProfile |
| OP-03 | StartConservation |
| OP-04 | ReadSensors |
| OP-05 | CorrectTemperature |
| OP-06 | CorrectHumidity |
| OP-07 | VerifyRecovery |
| OP-08 | EnterProtectionMode |
| OP-09 | ReduceLightExposure |
| OP-10 | AlertOperator |
| OP-11 | SuspendRiskyActivities (vibration) |
| OP-12 | SuspendOnDoorOpen |
| OP-13 | VerifyAndResume |
| OP-14 | SwitchToEmergencyPower |
| OP-15 | RecordIncidentAndShutdown |
| OP-16 | ReleaseBatch |

## 2. Operation Schemas

### OP-01 RunSelfCheck
| Field | Value |
|---|---|
| Description | Tests all essential sensors at power-on |
| Inputs | Power-on signal |
| Outputs | PASS / FAIL, list of failed sensors |
| Reads | Temperature, humidity, vibration, door, power sensors |
| Changes | Chamber state (SELF_CHECK to MONITORING) |
| Sends/Alerts | Alert to operator if a sensor fails |
| Assumes | Chamber just powered on |
| Result | All essential sensors work and chamber is in MONITORING. Otherwise it stays out of MONITORING and operator is alerted |

### OP-03 StartConservation
| Field | Value |
|---|---|
| Description | Starts active conservation of the batch |
| Inputs | Start request |
| Outputs | Conservation_Active flag |
| Reads | Door status, profile loaded flag |
| Changes | Chamber state (MONITORING to CONSERVING) |
| Sends/Alerts | Status to Mission Control |
| Assumes | Door closed AND profile loaded AND self-check passed |
| Result | Chamber is in CONSERVING. If any condition is false, nothing starts |

### OP-05 CorrectTemperature
| Field | Value |
|---|---|
| Description | Switches cooling or heating to bring temperature back in range |
| Inputs | Current temperature, profile limits |
| Outputs | Cooler/heater command, recovery timer started |
| Reads | Temperature sensor, profile |
| Changes | Cooler/heater output, chamber state (CONSERVING to CORRECTING) |
| Sends/Alerts | Warning to operator if not recovered half way |
| Assumes | Temperature outside range, power available |
| Result | Command issued and timer running. Temperature is NOT assumed to be fixed until OP-07 confirms it from sensor readings |

### OP-08 EnterProtectionMode
| Field | Value |
|---|---|
| Description | Protection response when correction fails in the recovery period |
| Inputs | Recovery timer expired |
| Outputs | Protection_Active flag |
| Reads | Temperature, humidity, recovery timer |
| Changes | Chamber state (CORRECTING to PROTECTION), light level |
| Sends/Alerts | Alert to operator, SafeModeTrigger to robot if temperature is above critical limit |
| Assumes | Value still out of range after 10 min |
| Result | Chamber in PROTECTION, light reduced, operator alerted, batch release blocked |

### OP-12 SuspendOnDoorOpen
| Field | Value |
|---|---|
| Description | Suspends normal operation when the door opens during conservation |
| Inputs | Door-open event |
| Outputs | Suspended flag |
| Reads | Door sensor, chamber state, robot state |
| Changes | Chamber state (CONSERVING to DOOR_HOLD) |
| Sends/Alerts | Operator alert, event logged, fault to robot if it is moving |
| Assumes | Chamber is in CONSERVING |
| Result | Chamber in DOOR_HOLD and normal operation stopped |

### OP-16 ReleaseBatch
| Field | Value |
|---|---|
| Description | Unlocks the batch for removal |
| Inputs | Release request from authorised user |
| Outputs | Batch_Released flag, door unlocked |
| Reads | Chamber safe flag, Protection_Active, authorisation |
| Changes | Door lock, loaded profile cleared |
| Sends/Alerts | Event logged |
| Assumes | Chamber is safe AND no protection response active AND request authorised |
| Result | Door unlocked and release logged. Otherwise release is refused and logged |

## 3. Operations changing chamber state, linked to Task 3

**Door opened during conservation (OP-12)**
- Chamber: CONSERVING to DOOR_HOLD.
- Robot: if the robot is NAVIGATING (or AVOIDING_OBSTACLE / RETURNING), an open door is a critical fault, so SafeModeTrigger fires and TR-10 moves the robot to SAFE_MODE.
- If the robot is DELIVERING with an authorised handover, there is no fault. Without authorisation, TR-10 also applies.

**Correction fails (OP-05 then OP-08)**
- Chamber: CONSERVING to CORRECTING to PROTECTION.
- Robot: if temperature is above the critical limit, SafeModeTrigger fires and TR-10 sends the robot to SAFE_MODE. OP-16 is blocked, so the robot cannot hand over the batch.

## 4. Operations that must be verified by sensors

1. **OP-05 CorrectTemperature / OP-06 CorrectHumidity (checked by OP-07).** Switching the cooler on is only a command. The cooler may be broken or the seal may leak, so the real temperature may not change. The batch is safe only if the sensor shows the value back in range. If it does not within 10 min, OP-08 starts.
2. **OP-13 VerifyAndResume (after door open).** Closing the door does not mean the chamber is safe. Warm air may have come in, or the door sensor may be faulty. The chamber resumes only after sensors confirm the door is closed and temperature and humidity are in range. Otherwise it goes back to correction.
