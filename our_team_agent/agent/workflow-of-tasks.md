# Workflow of Tasks

## 1. Workflow Goal

This workflow supports the goal in our completed [team charter](https://github.com/Aidan-S-1/BUS4498_Team_Build/blob/main/README.md).

## 2. Workflow Trigger

One run starts when a booked Orfalea undergraduate advising appointment that falls inside the four-week registration window reaches 48 hours before its start time. An hourly scheduled scan of the scheduling system finds appointments that have crossed this point and starts one run per appointment.

## 3. Completion Condition at Runtime

A run ends successfully when the appointment has a final status in the scheduling system and a matching record exists in the SlotSaver outcome log. The final status is one of: confirmed by the student, held by advisor decision, or cancelled with the slot reopened for public booking. For cancelled appointments, the outcome record must show the cancellation time and the reopen time.

## 4. General Workflow

SlotSaver retrieves the appointment (T1) and checks that it is still booked and has the details needed to contact the student (T2). It sends the student a reminder with confirm, cancel, and reschedule options (T3). When the student replies, or when 24 hours pass with no reply, SlotSaver classifies the response (T4). A confirmation goes straight to Record Outcome (T8). A cancellation goes to Cancel Appointment (T6) and Reopen Slot (T7), which puts the slot back in the public booking queue and notifies waitlisted students, and then to T8. Reschedule requests, unclear replies, and no-reply cases go to the SlotSaver agent (T5). The agent gathers evidence such as duplicate bookings, waitlist demand, and the student's registration date, may send one follow-up, and may book a new slot only when the student explicitly picks it. If the student confirms, the run goes to T8. If the student cancels or reschedules, the original slot goes through T6 and T7.

The agent never releases a slot on its own. When it recommends releasing a no-reply booking, or when it cannot reach a supported result, the assigned advisor reviews its evidence summary (T9) and decides to release (T6, T7, T8) or hold (T8). If the advisor does not decide before the deadline, the booking is held. If required appointment details are missing, or a tool times out, fails, or exhausts its retries, the case goes to the advising office coordinator (T10) with the failure evidence. The coordinator either fixes the problem and resumes the run from T1 (every task checks current status first, so nothing is sent or cancelled twice), closes the case manually and records it through T8, or stops the run for further human review.

## 5. Workflow Diagram

```mermaid
flowchart TD
    START([Booked appointment reaches 48 hours before start]) --> T1["T1: Retrieve Upcoming Appointment"]
    T1 --> T2["T2: Check Appointment Details"]
    T2 --> D1{"Details complete and still booked?"}
    D1 -->|Yes| T3["T3: Send Reminder"]
    D1 -->|No| T10["T10: Review Exception"]
    T3 --> T4["T4: Classify Student Response"]
    T4 --> D2{"Response type?"}
    D2 -->|Confirmed| T8["T8: Record Outcome"]
    D2 -->|Cancel| T6["T6: Cancel Appointment"]
    D2 -->|Reschedule request, unclear, or no reply| T5["T5: Resolve Unconfirmed Appointment"]
    T5 --> D3{"Agent result?"}
    D3 -->|Student confirmed| T8
    D3 -->|Student cancelled or rescheduled| T6
    D3 -->|Release or hold recommended, or undetermined| T9["T9: Approve Slot Release"]
    T9 --> D4{"Advisor decision?"}
    D4 -->|Release| T6
    D4 -->|Hold or no decision by deadline| T8
    T6 --> T7["T7: Reopen Slot"]
    T7 --> T8
    T8 --> END([Run complete])
    T1 -.->|Tool failure| T10
    T3 -.->|Tool failure| T10
    T4 -.->|Tool failure| T10
    T5 -.->|Tool failure or out-of-scope reply| T10
    T6 -.->|Tool failure or uncertain outcome| T10
    T7 -.->|Tool failure| T10
    T8 -.->|Tool failure| T10
    T10 --> D5{"Coordinator result?"}
    D5 -->|Fixed, resume| T1
    D5 -->|Closed manually| T8
    D5 -->|Cannot resolve| HANDOFF([Stopped for human review])
```
