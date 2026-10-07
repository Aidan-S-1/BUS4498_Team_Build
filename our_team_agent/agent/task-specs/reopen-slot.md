# Reopen Slot Task Specification

## Basic Information

- **Task ID:** T7
- **Task name:** Reopen Slot
- **Task type:** Act
- **Task owner:** SlotSaver system administrator (Orfalea Advising office)

## 1. Task Description

This task puts the cancelled slot back into the public booking queue and notifies students waitlisted for that advisor and day, so the time can be claimed by someone else. It supports the charter target of reopening at least 80% of cancelled slots within one hour of cancellation. It follows fixed rules: reopen only if the slot start time is still at least 2 hours away and the slot is not already booked by someone else.

## 2. Inputs

### Input 1

- **Input name:** Cancellation record
- **Contents and format:** Structured record from T6 with slot ID, advisor, and cancellation timestamp.
- **Source:** T6 Cancel Appointment

- **If a required input is missing or invalid:** The slot is not reopened. The task records "reopen not attempted" and sends the case to T10 Review Exception. If the slot starts in less than 2 hours, the task records "too late to reopen" and sends it to T8 Record Outcome.

## 3. Outputs

### Output 1

- **Output name:** Reopened slot record
- **Contents and format:** Slot ID, reopen timestamp, minutes between cancellation and reopen, and the number of waitlisted students notified.
- **Next task or recipient:** T8 Record Outcome
- **Complete when:** The scheduling system shows the slot as open for public booking and the reopen timestamp is saved.

## 4. Planned Tools

### Tool 1

- **Tool name:** `reopen_slot`
- **Input:** Cancellation record
- **Output:** Reopened slot record
- **Implementation Route:** Web API calls (scheduling system availability endpoint and waitlist notification endpoint)
- **Integration approach:** Direct integration
- **Role in this task:** Marks the slot as available and triggers the scheduling system's waitlist notification. It changes state.
- **Task timeout:** 10 minutes (well inside the one-hour reopen target)
- **Maximum retries:** 3
- **Retry only when:** The request times out or returns a temporary error. Wait 60 seconds. Before each retry, read the slot status; if it is already open, skip to the waitlist notice. The waitlist notice uses an idempotency key (slot ID + "reopen") so students are not notified twice.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record status "reopen failed" with the last known slot status and send the case to T10 Review Exception so a coordinator can reopen it by hand within the hour.
