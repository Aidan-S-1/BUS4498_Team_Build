# Check Appointment Details Task Specification

## Basic Information

- **Task ID:** T2
- **Task name:** Check Appointment Details
- **Task type:** Verify
- **Task owner:** SlotSaver system administrator (Orfalea Advising office)

## 1. Task Description

This task makes sure the appointment is still worth acting on and that SlotSaver can actually reach the student. It applies fixed rules: booking status must be "booked," the start time must be in the future and inside the registration window, and the record must have a student ID, a valid Cal Poly email, an advisor, and a start time. Appointments that pass go to the reminder step; anything else is flagged with the failed rule.

## 2. Inputs

### Input 1

- **Input name:** Appointment record
- **Contents and format:** Structured record with the fields listed in T1's output.
- **Source:** T1 Retrieve Upcoming Appointment

### Input 2

- **Input name:** Registration window dates
- **Contents and format:** Table of registration window IDs with start and end dates for the four-week window.
- **Source:** SlotSaver configuration file maintained by the advising office

- **If a required input is missing or invalid:** The task records which field or rule failed and sends the case to T10 Review Exception. If the status is already "cancelled" or "completed," the task records "no action needed" and sends it to T8 Record Outcome instead of T10.

## 3. Outputs

### Output 1

- **Output name:** Verified appointment
- **Contents and format:** The appointment record plus a check result of "pass" and the check timestamp.
- **Next task or recipient:** T3 Send Reminder
- **Complete when:** Every rule passes and the check result is saved to the run's working data.

### Output 2

- **Output name:** Detail exception
- **Contents and format:** Appointment ID, the failed rule or missing fields, and the record as retrieved.
- **Next task or recipient:** T10 Review Exception (or T8 Record Outcome for already-closed appointments)
- **Complete when:** The exception has been added to the coordinator's review queue with at least one failed rule named.

## 4. Planned Tools

### Tool 1

- **Tool name:** `check_appointment_details`
- **Input:** Appointment record; Registration window dates
- **Output:** Verified appointment; Detail exception
- **Implementation Route:** Functions/scripts (validation script with fixed rules)
- **Integration approach:** Direct integration
- **Role in this task:** Runs each rule against the record and returns pass or a list of failed rules. It changes no records.
- **Task timeout:** 30 seconds
- **Maximum retries:** 1
- **Retry only when:** The script crashes before returning a result. Retry immediately. The script only reads data, so a retry cannot create duplicates.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record status "check failed" with the error and send the case to T10 Review Exception. The workflow does not send a reminder.
