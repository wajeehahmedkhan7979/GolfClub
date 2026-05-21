# Golf Booking Bot Project Context

## 1) Project Overview

This project is a **browser automation / booking bot** for the Northern Golf Club tee-time booking system.

The goal is not to scrape or store the website content in a database. The bot only needs to interact with the live booking UI fast enough to secure a slot when bookings open.

The automation must:

- log into the member portal
- open the bookings area
- detect available tee slots
- prefer a configurable tee-time window
- click a **BOOK GROUP** action on an available slot
- confirm the booking in the popup
- rely on the club's automatic confirmation email as the final success signal

---

## 2) Business / Operational Context

### Booking schedule

The user clarified the opening schedule as:

- **Monday 5pm** → Friday bookings open
- **Monday 6pm** → Saturday bookings open
- **Monday 7pm** → Sunday bookings open

### Slot behavior

The user clarified the following booking semantics:

- **LOCKED** means the timesheet is not yet open for booking
- **BOOK GROUP** means the slot is available
- once a slot is taken, it does not reopen unless someone cancels
- there is value in booking a less-than-ideal slot and cancelling later rather than missing the booking entirely

### Booking confirmation

After a successful booking, the website sends an **automatic confirmation email** to the user's personal email address. That email is the authoritative success confirmation.

### Group booking behavior

The group is already preconfigured in the system.

The popup after clicking **BOOK GROUP** asks whether to book the playing partners/group, and the user clicks **YES**.

The group size is normally 4, but the user wants the ability to change this later.

---

## 3) Screenshots / UI Context

The screenshots show a mobile browser view of the booking page.

Visible UI patterns:

- time slots such as `02:00 pm`, `02:08 pm`, `02:16 pm`
- status label `1st Tee`
- button labeled `BOOK GROUP`
- some slots marked as `LOCKED`
- a modal popup with text similar to:
  - `Would You Like To Book Your Playing Partners?`
  - list of players in the group
  - `Number of Holes: 18 Holes`
  - buttons `Yes` and `No`

This suggests the main automation target is the slot row and confirmation popup rather than broad site data extraction.

---

## 4) Scope of the Automation

### What the bot must do

- log in to the members area
- navigate to the bookings page
- scan visible tee slots
- detect whether a slot is available
- identify the slot time
- choose a slot based on a preferred start time
- click **BOOK GROUP**
- confirm with **YES** in the popup
- verify booking success
- record local logs

### What the bot does not need to do

- collect all tee sheet history
- crawl the full website
- build a database of player profiles
- store score or handicap data
- scrape unrelated pages
- capture full HTML dumps
- implement generic web scraping infrastructure

The project is a booking workflow automation tool, not a scraping pipeline.

---

## 5) Current Script Provided by the User

The user provided this `main.py` as the starting point.

### Current behavior of the script

- opens the Northern Golf Club homepage
- clicks `MEMBERS LOGIN`
- switches to a new browser tab
- logs in with hardcoded credentials
- clicks the `Bookings` link
- finds elements matching `eventStatusOpen`
- clicks each matching open item
- navigates back after each click
- exits the browser

### Current limitations

The script is only a **navigation prototype**. It does **not** yet implement the real booking workflow.

It currently lacks:

- slot parsing
- tee-time selection logic
- preferred-time filtering
- confirmation popup handling
- clicking `BOOK GROUP`
- clicking `YES`
- success verification
- email-based confirmation handling
- robust retry behavior for booking release timing

---

## 6) Product Requirements Document (PRD) for `main.py`

## 6.1 Purpose

Build a single-file Selenium script named `main.py` that can reliably automate the Northern Golf Club booking process from login through confirmation.

## 6.2 Objective

The script should secure a tee-time as quickly and reliably as possible when slots open, while remaining simple enough to operate and debug.

## 6.3 Primary Users

- the club member who owns the booking account
- the developer maintaining the automation script

## 6.4 Success Criteria

The script is considered successful when it can:

1. authenticate into the website
2. reach the bookings page
3. detect available `BOOK GROUP` slots
4. choose the best slot based on a preferred time window
5. complete the confirmation popup flow
6. log the selected tee time
7. stop after successful booking
8. fail safely and visibly when the booking cannot be completed

## 6.5 Functional Requirements

### FR1 — Login

The script must:

- open the homepage
- click the member login entry point
- enter credentials
- submit the login form
- confirm that the login succeeded before continuing

### FR2 — Navigate to bookings

The script must:

- open the bookings area after login
- wait for the bookings interface to load
- avoid brittle assumptions about page timing

### FR3 — Detect slot state

The script must inspect visible slot elements and determine:

- tee time text
- whether the slot is available
- whether the slot is locked or full
- whether a `BOOK GROUP` button exists for that slot

### FR4 — Preferred time selection

The script must support a configurable preferred time window.

Behavior:

- aim for a preferred start time such as `08:00`
- if the exact time is unavailable, choose the next closest available slot within the configured range
- if no slot is available in the range, continue retrying until success or a stop condition is reached

### FR5 — Book group action

The script must:

- click the `BOOK GROUP` button for the selected slot
- detect the confirmation popup
- click `YES`

### FR6 — Success verification

The script must:

- detect visible success indicators if present
- record the selected slot in logs
- treat the website's automatic confirmation email as the real authoritative confirmation

### FR7 — Retry handling

The script must:

- retry around booking release time if no slot is open yet
- recover from stale element references
- recover from intercepted clicks
- continue retrying without crashing on minor transient failures

### FR8 — Logging

The script must record:

- login success/failure
- bookings page open
- available slot discovery
- chosen slot
- confirmation click result
- failure reason if any

### FR9 — Diagnostics

On failure, the script should optionally:

- save a screenshot
- write a clear error message
- stop in a safe state rather than looping forever

---

## 6.6 Non-Functional Requirements

### Reliability

The script should use explicit waits and retry logic rather than heavy use of blind `sleep()` calls.

### Speed

The booking selection flow must be fast enough to act immediately when bookings open.

### Maintainability

Selectors and timing constants should be easy to update in one file.

### Simplicity

The user explicitly wants a **single `main.py` file** rather than a multi-module project.

### Observability

The script should produce logs that make it easy to understand where a failure occurred.

---

## 6.7 Constraints

- Single-file script only
- Selenium-based implementation
- No database requirement
- No scraping pipeline requirement
- No unnecessary architecture split across multiple modules
- No dependence on unrelated website data

---

## 6.8 Suggested Internal Workflow for `main.py`

A simple state machine is appropriate, even in one file.

### States

- `INIT`
- `LOGIN`
- `OPEN_BOOKINGS`
- `DISCOVER_SLOTS`
- `SELECT_SLOT`
- `CONFIRM_BOOKING`
- `VERIFY_SUCCESS`
- `RETRY`
- `DONE`
- `FAILED`

### State transitions

1. start browser
2. log in
3. open bookings page
4. discover visible slots
5. select the best available slot
6. click `BOOK GROUP`
7. confirm popup with `YES`
8. verify success
9. finish or retry

---

## 6.9 Slot Data Model

The script should internally model each slot with fields similar to:

```python
{
    "time": "08:16",
    "status": "BOOK GROUP",
    "available": True,
    "button_selector": "..."
}
```

This allows the script to:

- compare slots
- filter unavailable entries
- rank by preferred time
- click the correct control

---

## 6.10 Assumptions

- the booking page exposes visible slot time text in a consistent format
- `BOOK GROUP` is the actionable marker for an open slot
- the confirmation popup is accessible via Selenium
- the website sends a confirmation email after successful booking
- the structure of the booking page is stable enough that selectors can be maintained

---

## 6.11 Risks / Failure Modes

- DOM changes break selectors
- popup timing causes missed clicks
- stale elements appear after navigation
- click interception due to overlays
- time-based booking release races
- mobile-responsive layout changes visible structure
- confirmation email delay makes external verification slower than browser state

---

## 6.12 Acceptance Criteria

The script is acceptable when all of the following are true:

- it can log in without manual intervention
- it can navigate to bookings consistently
- it detects open slots correctly
- it can choose a slot using a preferred time window
- it can click `BOOK GROUP`
- it can confirm the popup with `YES`
- it produces readable logs
- it retries safely when booking is not yet open
- it exits cleanly after success or failure

---

## 7) Implementation Notes for the Single `main.py`

The script should remain simple and production-oriented:

- use environment variables for credentials
- use explicit waits over fixed delays where possible
- keep selectors grouped at the top of the file
- isolate parsing logic from click logic
- store screenshots on failure
- keep all booking logic in one file for easier deployment and iteration

---

## 8) Out of Scope

The following are explicitly out of scope for this `main.py` project:

- multi-user scheduling platform
- database persistence layer
- analytics dashboard
- email inbox parsing system
- distributed worker queue
- scraping other club pages
- generalized browser automation framework

---

## 9) Final Project Definition

This project is a **single-file Selenium booking automation script** for Northern Golf Club that is designed to:

- react quickly when bookings open
- choose a tee time from a preferred window
- book the configured playing group
- confirm the booking popup
- rely on the club's confirmation email as the final success result

The script should be practical, minimal, and resilient rather than abstract or over-engineered.


## 10) Latest Architectural Update: Partner-Aware Release-Time Watcher

The architecture is now optimized as a **partner-aware release-time event watcher + multi-strategy booking executor**.

### System Model & Priority Chain
When scanning the tee sheet:
1. **Priority 1 (JOIN_PARTNERS)**: If preferred partners (`Forrest, Drew`, `Forrest, Jack`, or `Candiloro, Dom`) are found in a row with a `BOOK GROUP` button (meaning only 1 slot is left to book), click it to join their group. Dismiss the partner addition modal (click 'No') and confirm booking.
2. **Priority 2 (ALREADY_COMPLETE)**: If the client (`Cuthbertson, Aaron`) AND at least 2 partners are found in the same row, all members are already booked together. Flag this via a successful outcome status and exit.
3. **Priority 3 (CLIENT_ALONE)**: If the client is booked in a row but partners are not, raise a warning flag for manual intervention (client is already booked elsewhere).
4. **Priority 4 (BOOK_FRESH)**: If no partners/client are on the sheet, search for the first empty `BOOK GROUP` in the `08:00 - 10:00` range, click it, accept the partners modal (click 'Yes'), fill out the form fields with all 3 partners' details, and confirm booking.

### State Machine Workflow
`INIT` -> `LOGIN` -> `OPEN_EVENT_LIST` -> `WAIT_FOR_SATURDAY_OPEN` -> `OPEN_SATURDAY_EVENT` -> `SCAN_TEE_SHEET` -> `CLICK_BOOK_GROUP` -> `HANDLE_PARTNERS_MODAL` -> `CONFIRM_BOOKING` -> `VERIFY_SUCCESS` -> `DONE`
