# SOP-SHELVED-01: Shelved Alarms Capacity Management

## 1. Purpose

The purpose of this SOP is to manage the shelved alarms capacity. This includes monitoring the number of shelved
alarms, ensuring that the capacity is not exceeded, and taking appropriate actions when necessary.

## 2. Scope

This SOP applies to all Shelved Alarms. It issues a `SHELVED_ALARM_CAPACITY_EXCEEDED` alarm when the shelved alarm
capacity is reached or exceeded.

## 3. Roles and Responsibilities
- **Operator**: Primarily responsible for real-time triage and management of shelved alarms. The operator **MUST**
execute SOP-SHELVED-01 immediately upon receiving a `SHELVED_ALARM_CAPACITY_EXCEEDED` alarm to restore system
visibility.
- **Maintenance Team**: Responsible for providing estimated completion times for active work. **MUST** notify the 
Operator when an asset is returned to service so the shelf slot can be cleared.
- **Safety Lead**: The Safety Lead is responsible for managing the **Master Alarm Database** and has sole authority
to authorize **Maintenance Mode** overrides if a sixth critical alarm must be suppressed beyond the shelf capacity.

## 4. Procedural Steps

1. **Immediate Audit**: Open the **Shelved Alarm Summary** on the HMI.
2. **Review**: Priority should be given to unshelving Diamond [ ◇ ] icons first if a capacity advisory is issued.
3. **Triage & Re-Validate**:
   - **Step 1**: Identify any shelved alarms where the maintenance task is complete. Unshelve those **immediately**.
   - **Step 2**: For any remaining shelved alarms, verify the $Z$-Score. If any asset has returned to a **Nominal**
     state ($Z$-Score < 2.0), unshelve and clear the alarm.
4. **Prioritize**: 
   - If 5 alarms must remain shelved for active maintenance, the operator **MUST** transition to **High-Vigilance 
     Monitoring**.
   - New abnormal conditions will **NOT** be shelfable until a slot is cleared.
5. **Escalation**: If a 6th critical maintenance task is required while the capacity is full, the operator **MUST**
   notify the Safety Lead to get authorization to temporarily override **Maintenance Mode** in the **Digital Twin**, 
   which provides state-based suppression without using a shelf slot.
