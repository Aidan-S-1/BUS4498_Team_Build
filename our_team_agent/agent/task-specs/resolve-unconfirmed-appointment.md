# Resolve Unconfirmed Appointment Task Specification

```yaml
# BASIC INFORMATION
task_id: "T5"
task_name: "Resolve Unconfirmed Appointment"
task_owner: "SlotSaver system administrator (Orfalea Advising office); assigned advisor approves any release"
```

## 1. Task Goal

- **Objective:** For a booking the student has not clearly confirmed or cancelled, reach a supported outcome early enough that the slot can still be used: the student confirms, the student cancels or picks a new time, or the assigned advisor receives an evidence-backed recommendation to release or hold the slot.

## 2. Inbound Inputs

### Input 1

- **Input name:** Classified response
- **What it contains:** Appointment ID, label (reschedule request, unclear, or no reply), confidence score, reply text if any, and classification timestamp.
- **Source:** T4 Classify Student Response

### Input 2

- **Input name:** Verified appointment
- **What it contains:** The full appointment record that passed T2, including student contact fields, advisor, start time, and booking status.
- **Source:** T2 Check Appointment Details

### Input 3

- **Input name:** Reminder sent record
- **What it contains:** Message ID, send timestamp, and response deadline for the first reminder.
- **Source:** T3 Send Reminder

## 3. Tool Permissions and Boundaries

### Task-Wide Limits

- **Total task timeout:** 20 hours from the start of T5, and never later than 4 hours before the appointment start time, whichever comes first. This includes waiting for a follow-up reply.
- **Maximum tool calls:** 15 across all tools, retries included.

### Tool 1

- **Tool name:** `retrieve_booking_history`
- **Tool type:** API request (read-only, scheduling system)
- **Supports these permitted subtasks:** Review Booking History
- **Allowed use:** Read this student's other current bookings and their attended, cancelled, and no-show history for the current academic year.
- **Prohibited use:** Reading other students' records; changing any booking.
- **Approval required:** None within the allowed use
- **Timeout per call:** 30 seconds
- **Maximum retries per call:** 2
- **Retry conditions and failure response:** Retry after 15 seconds on a timeout or temporary error. If retries run out, continue without history and note it as an unresolved issue.

### Tool 2

- **Tool name:** `retrieve_registration_date`
- **Tool type:** Database query (read-only, registration date list supplied by the advising office)
- **Supports these permitted subtasks:** Check Registration Timing
- **Allowed use:** Read this student's registration appointment date for the current term.
- **Prohibited use:** Reading grades, holds, course plans, or any other student record field.
- **Approval required:** None within the allowed use
- **Timeout per call:** 30 seconds
- **Maximum retries per call:** 2
- **Retry conditions and failure response:** Retry after 15 seconds on a timeout or temporary error. If retries run out, continue and note the missing date as an unresolved issue.

### Tool 3

- **Tool name:** `retrieve_waitlist_demand`
- **Tool type:** API request (read-only, scheduling system waitlist)
- **Supports these permitted subtasks:** Check Waitlist Demand
- **Allowed use:** Read the count of students waitlisted for this advisor and this day.
- **Prohibited use:** Reading waitlisted students' identities or contacting them.
- **Approval required:** None within the allowed use
- **Timeout per call:** 30 seconds
- **Maximum retries per call:** 2
- **Retry conditions and failure response:** Retry after 15 seconds on a timeout or temporary error. If retries run out, treat demand as unknown and note it.

### Tool 4

- **Tool name:** `send_follow_up`
- **Tool type:** API request (email service, plus SMS gateway if a phone number is on file)
- **Supports these permitted subtasks:** Send Follow-Up
- **Allowed use:** Send at most one follow-up message to the student's Cal Poly email and, if on file, their phone, using the approved follow-up template. The agent may add one short line answering the student's own question (for example, listing open times they asked about).
- **Prohibited use:** Messaging anyone other than the student; more than one follow-up per run; advising on course selection; telling the student the appointment is cancelled.
- **Approval required:** None within the allowed use
- **Timeout per call:** 60 seconds
- **Maximum retries per call:** 1
- **Retry conditions and failure response:** Retry once after 60 seconds only if the service did not accept the message. Every send uses an idempotency key (appointment ID + "follow-up") and the sent log is checked first. If the outcome is uncertain, do not resend; hand off to T10 Review Exception.

### Tool 5

- **Tool name:** `classify_follow_up_reply`
- **Tool type:** Language-model call (temperature 0, fixed label set)
- **Supports these permitted subtasks:** Interpret Follow-Up Reply
- **Allowed use:** Read the student's reply to the follow-up and return confirmed, cancel, selected new slot (with slot ID), or unclear, with a confidence score.
- **Prohibited use:** Sending messages or changing records.
- **Approval required:** None within the allowed use
- **Timeout per call:** 30 seconds
- **Maximum retries per call:** 2
- **Retry conditions and failure response:** Retry after 15 seconds on a timeout, error, or out-of-set label. If retries run out, treat the reply as unclear.

### Tool 6

- **Tool name:** `find_open_slots`
- **Tool type:** API request (read-only, scheduling system availability)
- **Supports these permitted subtasks:** Offer Reschedule Options
- **Allowed use:** Read open slots for the same advisor or advising team between now and the student's registration date.
- **Prohibited use:** Holding or booking slots.
- **Approval required:** None within the allowed use
- **Timeout per call:** 30 seconds
- **Maximum retries per call:** 2
- **Retry conditions and failure response:** Retry after 15 seconds on a timeout or temporary error. If retries run out, skip reschedule options and note it.

### Tool 7

- **Tool name:** `book_selected_slot`
- **Tool type:** API request (scheduling system booking endpoint)
- **Supports these permitted subtasks:** Book Selected Slot
- **Allowed use:** Book one new slot for this student, only the exact slot the student named in a reply classified as "selected new slot."
- **Prohibited use:** Booking without an explicit student selection; booking more than one slot; cancelling the original booking (that is T6's job).
- **Approval required:** The student's written selection of the specific slot, recorded before the call.
- **Timeout per call:** 30 seconds
- **Maximum retries per call:** 1
- **Retry conditions and failure response:** Before retrying, check whether the booking already exists for this student and slot. Retry once only if it does not. If the slot was taken or the outcome is uncertain, stop and hand off to T10 Review Exception.

### Tool 8

- **Tool name:** `draft_release_recommendation`
- **Tool type:** Language-model call
- **Supports these permitted subtasks:** Prepare Release Recommendation
- **Allowed use:** Summarize the evidence gathered in this run into a recommendation (release or hold) for the assigned advisor.
- **Prohibited use:** Releasing or cancelling the slot; contacting the advisor or student directly.
- **Approval required:** None within the allowed use; the recommendation itself is reviewed by the advisor in T9.
- **Timeout per call:** 60 seconds
- **Maximum retries per call:** 1
- **Retry conditions and failure response:** Retry once after 15 seconds on an error. If it still fails, hand off to T9 with the raw evidence and status "undetermined."

## 4. How the Agent Should Reason

### Permitted Subtask 1

- **Subtask name:** Review Booking History
- **Subtask description:** Looks for duplicate bookings by the same student this window and past no-shows. Finding a second booking for the same purpose strongly suggests this one will be abandoned.
- **Subtask boundary:** Read only. Only this student's records.
- **Retry limits:** 1 additional attempt

### Permitted Subtask 2

- **Subtask name:** Check Registration Timing
- **Subtask description:** Compares the student's registration date to the appointment time. An appointment after the student has already registered is less likely to be needed.
- **Subtask boundary:** Read only. Registration date field only.
- **Retry limits:** 1 additional attempt

### Permitted Subtask 3

- **Subtask name:** Check Waitlist Demand
- **Subtask description:** Checks how many students are waiting for this advisor and day, which sets how urgent a decision is.
- **Subtask boundary:** Read only. Counts only, no identities.
- **Retry limits:** 1 additional attempt

### Permitted Subtask 4

- **Subtask name:** Send Follow-Up
- **Subtask description:** Sends one follow-up to the student, through a second channel if available, asking them to confirm, cancel, or pick a new time. Used when there was no reply or an unclear reply.
- **Subtask boundary:** Only once per run. Must not imply the appointment is cancelled.
- **Retry limits:** 0 (one send per run; tool-level retry only for a send the service did not accept)

### Permitted Subtask 5

- **Subtask name:** Interpret Follow-Up Reply
- **Subtask description:** Classifies the student's reply to the follow-up or to the original reminder if it arrives late.
- **Subtask boundary:** Classification only. A low-confidence result counts as unclear.
- **Retry limits:** 1 additional attempt per reply

### Permitted Subtask 6

- **Subtask name:** Offer Reschedule Options
- **Subtask description:** When the student asked to reschedule, finds open slots before their registration date and includes up to three in the follow-up.
- **Subtask boundary:** Read open slots only; offering is not booking.
- **Retry limits:** 1 additional attempt

### Permitted Subtask 7

- **Subtask name:** Book Selected Slot
- **Subtask description:** Books the specific new slot the student chose, then reports "student rescheduled" so the original slot goes to T6 and T7.
- **Subtask boundary:** Requires an explicit student selection of one slot. Never books on inference.
- **Retry limits:** 0

### Permitted Subtask 8

- **Subtask name:** Prepare Release Recommendation
- **Subtask description:** When the student stays silent or unclear, combines the evidence into a release-or-hold recommendation with the reasons.
- **Subtask boundary:** Recommendation only; the advisor decides in T9.
- **Retry limits:** 1 additional attempt

- **Decision guidance:** After each subtask, use its findings to select the permitted subtask most likely to resolve the most important remaining uncertainty. Do not follow a fixed sequence. For example, a reschedule request points first to Offer Reschedule Options and Send Follow-Up; a no-reply case with a duplicate booking found points to Prepare Release Recommendation without waiting long; a no-reply case with no history issues points to Send Follow-Up first. If no permitted subtask can make useful progress, stop and hand the case to a person.

## 5. When to Stop or Hand Off to a Human

- **Stop successfully when:** The student's own reply, classified at or above 0.8 confidence, shows a confirmation, a cancellation, or a completed booking of a new slot they selected. Confidence alone is not enough; there must be a student reply or booking record behind the result.
- **Hand off early when:** The student has not given a clear answer and the evidence supports releasing the slot; the follow-up has been sent and no clear answer arrives by the task deadline; a reply raises something outside scope (course-selection questions, complaints, accommodation requests); a booking or send outcome is uncertain; or the tool-call or time budget runs out.
- **Hand off to:** Assigned advisor through T9 Approve Slot Release for release or hold decisions; advising office coordinator through T10 Review Exception for tool failures, uncertain actions, and out-of-scope replies.

Stop at the first applicable budget limit or handoff condition. While awaiting review, take no further autonomous action.

## 6. Outbound Deliverable

- **Status:** Completed (student confirmed, student cancelled, or student rescheduled) or escalated to human (release recommended, hold recommended, or undetermined).
- **Result or recommendation:** The student's decision, or the release/hold recommendation. If escalated before reaching a supported result, write undetermined.
- **Evidence summary:** The most important evidence, such as the student's reply text, duplicate bookings found, registration date versus appointment date, waitlist count, and follow-up send time.
- **Subtasks performed:** Permitted subtasks completed, including repeated attempts.
- **Unresolved issues:** Remaining uncertainties, such as missing history or an unanswered follow-up; use none only if no unresolved issue remains.
- **Handoff note:** Reason for stopping, unresolved questions, and what the advisor or coordinator needs to decide; write "Not applicable" for a completed task.
- **Next task or recipient:** Student confirmed: T8 Record Outcome. Student cancelled or rescheduled: T6 Cancel Appointment. Release recommended, hold recommended, or undetermined: T9 Approve Slot Release. Tool failure or out-of-scope reply: T10 Review Exception.
