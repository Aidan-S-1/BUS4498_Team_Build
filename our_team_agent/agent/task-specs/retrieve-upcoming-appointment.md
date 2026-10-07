# Retrieve Upcoming Appointment Task Specification

## Basic Information

- **Task ID:** T1
- **Task name:** Retrieve Upcoming Appointment
- **Task type:** Retrieve
- **Task owner:** SlotSaver system administrator (Orfalea Advising office)

## 1. Task Description

This task pulls the full booking record for one appointment that has just crossed the 48-hour mark so the rest of the workflow works from current data. The hourly scan supplies the appointment ID, and the task queries the scheduling system by that ID using a fixed field list. No judgment is involved: it returns exactly what the scheduling system holds.

## 2. Inputs

### Input 1

- **Input name:** Appointment trigger
- **Contents and format:** Structured record with appointment ID, scan timestamp, and registration window ID.
- **Source:** Hourly scheduled scan of the Orfalea advising scheduling system (workflow trigger).

- **If a required input is missing or invalid:** The run does not query the scheduling system. It logs the invalid trigger and sends it to T10 Review Exception.

## 3. Outputs

### Output 1

- **Output name:** Appointment record
- **Contents and format:** Structured record with appointment ID, student ID, student name, Cal Poly email, phone (if on file), advisor ID and name, start and end time, location or video link, booking status, booking created time, and retrieval timestamp.
- **Next task or recipient:** T2 Check Appointment Details
- **Complete when:** A record for the requested appointment ID has been returned and saved to the run's working data with a retrieval timestamp.

## 4. Planned Tools

### Tool 1

- **Tool name:** `retrieve_upcoming_appointment`
- **Input:** Appointment trigger
- **Output:** Appointment record
- **Implementation Route:** Web API calls (read-only request to the scheduling system's appointments endpoint)
- **Integration approach:** Direct integration
- **Role in this task:** Looks up the appointment by ID and returns its booking fields. It changes no records.
- **Task timeout:** 2 minutes
- **Maximum retries:** 3
- **Retry only when:** The request times out or returns a temporary server or rate-limit error. Wait 30 seconds before each retry. Reads change nothing, so retries cannot create duplicates.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record status "retrieval failed" with the error message and attempt count, and send the case to T10 Review Exception. The workflow does not continue to T2.
