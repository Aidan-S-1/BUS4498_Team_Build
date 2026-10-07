# Classify Student Response Task Specification

## Basic Information

- **Task ID:** T4
- **Task name:** Classify Student Response
- **Task type:** Sense
- **Task owner:** SlotSaver system administrator (Orfalea Advising office)

## 1. Task Description

This task turns whatever the student did with the reminder into one routing label. Link clicks map directly to "confirmed," "cancel," or "reschedule request." Free-text replies ("can't make it, sorry," "is Thursday open instead?") go to a language-model classifier that returns one label and a confidence score. Replies below the confidence threshold are labeled "unclear." If the 24-hour window closes with nothing from the student, the label is "no reply." The task runs when a response arrives or when the window closes, whichever comes first.

## 2. Inputs

### Input 1

- **Input name:** Reminder sent record
- **Contents and format:** Structured record from T3, including message ID and response deadline.
- **Source:** T3 Send Reminder

### Input 2

- **Input name:** Student response
- **Contents and format:** Either a link-click event (appointment ID, option clicked, timestamp) or an email reply (sender address, message ID it replies to, body text, timestamp). May be empty if the window closed with no response.
- **Source:** Student, through the reminder links or the advising office mailbox

- **If a required input is missing or invalid:** A missing student response at the deadline is a valid "no reply" case and goes to T5. A reply from an address that does not match the student is not used; it is labeled "unclear" with a note. A missing reminder sent record sends the case to T10 Review Exception.

## 3. Outputs

### Output 1

- **Output name:** Classified response
- **Contents and format:** Appointment ID, label (confirmed, cancel, reschedule request, unclear, or no reply), confidence score for free-text replies, the reply text if any, and classification timestamp.
- **Next task or recipient:** Confirmed: T8 Record Outcome. Cancel: T6 Cancel Appointment. Reschedule request, unclear, or no reply: T5 Resolve Unconfirmed Appointment.
- **Complete when:** Exactly one label is assigned and saved with its evidence (click event, reply text, or deadline timestamp).

## 4. Planned Tools

### Tool 1

- **Tool name:** `classify_student_response`
- **Input:** Reminder sent record; Student response
- **Output:** Classified response
- **Implementation Route:** Functions/scripts (maps link clicks) and web API calls (language-model call for free-text replies, temperature 0, fixed label set, confidence threshold of 0.8)
- **Integration approach:** Direct integration
- **Role in this task:** Reads the response and returns one label. It changes no records and sends nothing.
- **Task timeout:** 3 minutes from the response arriving or the window closing
- **Maximum retries:** 2
- **Retry only when:** The language-model call times out or returns a temporary error or output outside the label set. Wait 15 seconds. Classification changes nothing, so retries cannot create duplicates.
- **On timeout, exhausted retries, or an error that cannot be retried:** Label the response "unclear," attach the error, and send it to T5 Resolve Unconfirmed Appointment so a confirmed or cancelling student is not ignored. If the mailbox or click log cannot be read at all, send the case to T10 Review Exception.
