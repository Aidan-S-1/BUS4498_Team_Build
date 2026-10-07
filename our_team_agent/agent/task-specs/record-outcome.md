# Record Outcome Task Specification

## Basic Information

- **Task ID:** T8
- **Task name:** Record Outcome
- **Task type:** Remember
- **Task owner:** SlotSaver system administrator (Orfalea Advising office)

## 1. Task Description

This task writes one final record per run to the SlotSaver outcome log. The log is how the team measures the charter targets: the attend-to-booking rate and the share of cancelled slots reopened within one hour. It copies fixed fields from earlier task outputs; no judgment is involved.

## 2. Inputs

### Input 1

- **Input name:** Final run result
- **Contents and format:** Appointment ID, final status (confirmed, held, cancelled and reopened, cancelled too late to reopen, rescheduled, no action needed, or closed manually), path taken (task IDs), decision source (student, advisor, or coordinator), and key timestamps (reminder sent, response, cancellation, reopen).
- **Source:** T2 Check Appointment Details, T4 Classify Student Response, T5 Resolve Unconfirmed Appointment, T7 Reopen Slot, T9 Approve Slot Release, or T10 Review Exception

- **If a required input is missing or invalid:** The task writes what it has, marks the missing fields, and sends the case to T10 Review Exception to complete the record.

## 3. Outputs

### Output 1

- **Output name:** Outcome log entry
- **Contents and format:** One row in the SlotSaver outcome log with the fields above plus a run ID and write timestamp.
- **Next task or recipient:** SlotSaver outcome log (Google Sheet owned by the advising office); run complete
- **Complete when:** The row exists in the log and its run ID matches the current run.

## 4. Planned Tools

### Tool 1

- **Tool name:** `record_outcome`
- **Input:** Final run result
- **Output:** Outcome log entry
- **Implementation Route:** Web API calls (append a row to the outcome log sheet)
- **Integration approach:** MCP integration (Google Drive)
- **Role in this task:** Appends one row per run. It changes state by writing a record.
- **Task timeout:** 2 minutes
- **Maximum retries:** 3
- **Retry only when:** The write times out or returns a temporary error. Wait 30 seconds. Before each retry, search the log for the run ID; if the row already exists, stop, so no duplicate rows are created.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record status "outcome not logged" in the run's working data and send the case to T10 Review Exception. The run is not marked complete.
