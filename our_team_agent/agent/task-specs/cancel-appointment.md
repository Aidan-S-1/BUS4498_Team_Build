# Cancel Appointment Task Specification

## Basic Information

- **Task ID:** T6
- **Task name:** Cancel Appointment
- **Task type:** Act
- **Task owner:** SlotSaver system administrator (Orfalea Advising office)

## 1. Task Description

This task cancels the original booking in the scheduling system once a cancellation is authorized, and sends the student a cancellation notice. A cancellation is authorized only by one of three sources: the student's cancel response (T4), the student's cancellation or reschedule in T5, or the advisor's release decision (T9). The task applies that rule and records which source authorized it.

## 2. Inputs

### Input 1

- **Input name:** Cancellation authorization
- **Contents and format:** Structured record with appointment ID, authorization source (student via T4, student via T5, or advisor via T9), authorizing evidence (click event, reply text, or advisor decision record), and timestamp.
- **Source:** T4 Classify Student Response, T5 Resolve Unconfirmed Appointment, or T9 Approve Slot Release

### Input 2

- **Input name:** Cancellation notice template
- **Contents and format:** Approved email templates for student-initiated cancellation, reschedule, and advisor release, with placeholders for appointment details.
- **Source:** Orfalea Advising office (stored in SlotSaver configuration)

- **If a required input is missing or invalid:** No cancellation is made. The task records "cancellation not authorized" and sends the case to T10 Review Exception.

## 3. Outputs

### Output 1

- **Output name:** Cancellation record
- **Contents and format:** Appointment ID, slot ID, advisor, authorization source, cancellation timestamp from the scheduling system, and notice message ID.
- **Next task or recipient:** T7 Reopen Slot
- **Complete when:** The scheduling system shows the booking as cancelled and the cancellation timestamp is saved. This timestamp starts the one-hour reopen clock.

## 4. Planned Tools

### Tool 1

- **Tool name:** `cancel_appointment`
- **Input:** Cancellation authorization; Cancellation notice template
- **Output:** Cancellation record
- **Implementation Route:** Web API calls (scheduling system cancel endpoint; email service for the notice)
- **Integration approach:** Direct integration for the scheduling system; MCP integration (Gmail) for the notice
- **Role in this task:** Checks the current booking status, cancels the booking if it is still active, and sends one notice to the student. It changes state.
- **Task timeout:** 5 minutes
- **Maximum retries:** 2
- **Retry only when:** The cancel or send request fails with a timeout or temporary error. Wait 30 seconds. Before each retry, read the booking status; if it is already cancelled, skip the cancel call and only confirm the notice. The notice uses an idempotency key (appointment ID + "cancel notice"). If the scheduling system response is unclear, do not retry.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record status "cancellation failed" or "cancellation outcome uncertain" with the last known booking status and send the case to T10 Review Exception. The slot is not reopened.
