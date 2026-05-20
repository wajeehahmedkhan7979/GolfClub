# Northern Golf Club Booking Bot

## Overview
This project is a browser automation bot designed to secure tee-time bookings for the Northern Golf Club. Built as a minimal, single-file Selenium script (`main1.py`), it automatically logs into the member portal, detects available tee slots, and attempts to book a group based on a configured time window as soon as the timesheet opens.

## Features
- **Automated Login**: Securely authenticates into the member portal.
- **Tee-time Discovery**: Scans the timesheet for available "BOOK GROUP" slots.
- **Automated Booking Workflow**: Handles the entire booking process, including confirmation modals.
- **Resiliency & Retry Logic**: Continues to retry for slots around booking release time and recovers from intercepted clicks or stale elements.
- **Diagnostics**: Detailed local logging and screenshots on failure to aid in debugging.

## Prerequisites
- Python 3.x
- Required dependencies listed in `requirements 3.txt` (including `selenium`)
- A valid Northern Golf Club member account

## Setup & Installation

1. Clone or navigate to the project repository:
   ```bash
   cd GolfClub
   ```

2. Create and activate a virtual environment:
   ```bash
   python -m venv .venv
   source .venv/bin/activate  # On Windows, use `.venv\Scripts\activate`
   ```

3. Install the dependencies:
   ```bash
   pip install -r "requirements 3.txt"
   ```

4. Configure your credentials as environment variables (or modify the `.env` if provided).

## Usage
Run the main script to start the booking automation:
```bash
python main1.py
```

## Architecture
The bot operates based on a nested traversal state machine:
1. `LOGIN` -> `OPEN_EVENT_LIST` -> `COLLECT_EVENTS`
2. `OPEN_EVENT` -> `COLLECT_TEE_ROWS` -> `BOOK_FIRST_AVAILABLE`
3. `CONFIRM_BOOKING` -> `VERIFY_SUCCESS` -> `RETURN_TO_EVENT_LIST`

It relies on the official email notification as the final authoritative confirmation of a successful booking.

## Disclaimer
This script is intended solely for personal use to automate the booking of a single group. It is not designed to scrape historical data or overload the club's servers. Please use responsibly.
