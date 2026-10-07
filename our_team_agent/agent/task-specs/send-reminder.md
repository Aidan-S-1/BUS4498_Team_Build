# Send Reminder Task Specification

## Basic Information

- **Task ID:** T3
- **Task name:** Send Reminder
- **Task type:** Act
- **Task owner:** SlotSaver system administrator (Orfalea Advising office)

## 1. Task Description

This task sends the student a reminder 48 hours before the appointment so they either commit to it or release it while someone else can still use it. It fills an approved email template with the appointment details and three one-click options: confirm, cancel, or request a reschedule. The student can also reply in free text. A fixed template is used, so every student gets the same approved wording.

## 2. Inputs

### Input 1

- **Input name:** Verified appointment
- **Contents and format:** Structured record from T2 with a "pass" check result.
- **Source:** T2 Check Appointment Details

### Input 2

- **Input name:** Reminder template
- **Contents and format:** Approved email template with placeholders for student name, advisor, date, time, location or link, and the three response links.
- **Source:** Orfalea Advising office (stored in SlotSaver configuration)

- **If a required input is missing or invalid:** No message is sent. The task records "reminder not sent" with the reason and sends the case to T10 Review Exception.

## 3. Outputs

### Output 1

- **Output name:** Reminder sent record
- **Contents and format:** Appointment ID, recipient email, message ID from the email service, send timestamp, and the response deadline (24 hours after sending).
- **Next task or recipient:** T4 Classify Student Response
- **Complete when:** The email service returns a message ID and the record is saved, which starts the 24-hour response window.

## 4. Planned Tools

### Tool 1

- **Tool name:** `send_reminder`
- **Input:** Verified appointment; Reminder template
- **Output:** Reminder sent record
- **Implementation Route:** Web API calls (email service send endpoint)
- **Integration approach:** MCP integration (Gmail MCP server for the advising office mailbox)
- **Role in this task:** Fills the template and sends one email to the student's Cal Poly address. It changes state by sending a message.
- **Task timeout:** 2 minutes
- **Maximum retries:** 2
- **Retry only when:** The send request fails before the service accepts it (connection error or temporary server error). Wait 60 seconds. Each send carries an idempotency key made from the appointment ID and "reminder," and the task checks the sent log for that key before retrying so the student never gets two reminders. If the service may have accepted the message but returned no message ID, do not retry.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record status "reminder failed" or "send outcome uncertain" and send the case to T10 Review Exception. The response window does not start.
