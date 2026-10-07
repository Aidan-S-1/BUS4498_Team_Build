# Review Exception Task Specification

## Basic Information

- **Task ID:** T10
- **Task name:** Review Exception
- **Task type:** Decide
- **Task owner:** Orfalea Advising office coordinator

## 1. Task Description

This task handles every case the automated path cannot finish safely: missing or invalid appointment details, tool failures, uncertain send or cancel outcomes, and out-of-scope student replies. The coordinator reads the exception, checks the scheduling system and mailbox directly, and uses judgment to decide how to close it: fix the problem and resume the run, handle it by hand and close it, or stop it for further review.

## 2. Inputs

### Input 1

- **Input name:** Exception case
- **Contents and format:** Appointment ID, the task ID where the problem occurred, status recorded by that task (for example "reminder failed" or "cancellation outcome uncertain"), error message or failed rule, attempt count, and the working data available at that point.
- **Source:** Any of T1 through T9

- **If a required input is missing or invalid:** The coordinator looks up the appointment directly in the scheduling system using whatever ID is available. If the case cannot be identified, it is stopped for human review and reported to the SlotSaver system administrator.

## 3. Outputs

### Output 1

- **Output name:** Exception resolution
- **Contents and format:** Appointment ID, resolution (fixed and resumed, closed manually, or cannot resolve), what the coordinator did, any manual actions taken (for example, reminder sent by hand or slot reopened by hand), coordinator name, and timestamp.
- **Next task or recipient:** Fixed and resumed: T1 Retrieve Upcoming Appointment. Closed manually: T8 Record Outcome. Cannot resolve: stopped for human review by the SlotSaver system administrator.
- **Complete when:** A resolution is saved with the coordinator's name and at least one action or reason.

## 4. Planned Tools

### Tool 1

- **Tool name:** `retrieve_exception_case`
- **Input:** Exception case
- **Output:** Exception resolution
- **Implementation Route:** Database queries (reads the exception queue) and web API calls (records the coordinator's resolution)
- **Integration approach:** MCP integration (Google Drive for the exception queue sheet)
- **Role in this task:** Shows the coordinator the case and its evidence, and records the coordinator's resolution. The resolution itself is a human decision.
- **Task timeout:** Human response deadline: 2 business hours after assignment, or 1 hour after assignment for failed reopens (to protect the one-hour reopen target).
- **Maximum retries:** Not applicable — manual task.
- **Retry only when:** Not applicable
- **On timeout, exhausted retries, or an error that cannot be retried:** If the deadline passes, the case is escalated to the SlotSaver system administrator and stays in the queue with status "overdue." The workflow takes no automated action on the appointment while it is in review.
