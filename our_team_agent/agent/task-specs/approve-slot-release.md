# Approve Slot Release Task Specification

## Basic Information

- **Task ID:** T9
- **Task name:** Approve Slot Release
- **Task type:** Decide
- **Task owner:** Assigned advisor for the appointment

## 1. Task Description

This task keeps a person in charge of releasing a slot the student never cancelled. The advisor reads the agent's evidence summary and decides whether to release the booking for other students or hold it. This is human judgment: the advisor may know things the system does not, such as a student who emailed them directly. If the advisor does not decide by the deadline, the booking is held, so no student loses an appointment without a person's decision.

## 2. Inputs

### Input 1

- **Input name:** Agent deliverable
- **Contents and format:** T5's outbound deliverable: status, recommendation (release, hold, or undetermined), evidence summary, subtasks performed, unresolved issues, and handoff note.
- **Source:** T5 Resolve Unconfirmed Appointment

### Input 2

- **Input name:** Verified appointment
- **Contents and format:** The appointment record from T2.
- **Source:** T2 Check Appointment Details

- **If a required input is missing or invalid:** The advisor is not asked to decide on incomplete evidence. The case goes to T10 Review Exception, and the booking stays held.

## 3. Outputs

### Output 1

- **Output name:** Release decision
- **Contents and format:** Appointment ID, decision (release or hold), advisor name, optional note, and decision timestamp. A missed deadline is recorded as "hold, no decision by deadline."
- **Next task or recipient:** Release: T6 Cancel Appointment. Hold: T8 Record Outcome.
- **Complete when:** A decision is saved with the advisor's name, or the deadline passes and the default hold is saved.

## 4. Planned Tools

### Tool 1

- **Tool name:** `request_release_decision`
- **Input:** Agent deliverable; Verified appointment
- **Output:** Release decision
- **Implementation Route:** Web API calls (sends the advisor a review request with release and hold buttons; records the button click)
- **Integration approach:** MCP integration (Gmail for the request; Google Drive for the decision record)
- **Role in this task:** Delivers the evidence to the advisor and records their decision. The decision itself is made by the advisor.
- **Task timeout:** Human response deadline: the earlier of 4 business hours after the request or 2 hours before the appointment start.
- **Maximum retries:** Not applicable — manual task.
- **Retry only when:** Not applicable
- **On timeout, exhausted retries, or an error that cannot be retried:** On a missed deadline, record "hold, no decision by deadline" and send it to T8 Record Outcome. If the review request cannot be delivered, send the case to T10 Review Exception; the booking stays held.
