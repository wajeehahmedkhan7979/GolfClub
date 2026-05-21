"""
Northern Golf Club - Saturday Booking Automation
Production-grade Selenium tee-time booking bot.
"""

from **future** import annotations

import enum
import logging
import os
import random
import re
import sys
import time
from dataclasses import dataclass, field
from datetime import datetime, time as dtime
from pathlib import Path
from typing import Optional

from selenium import webdriver
from selenium.common.exceptions import (
ElementClickInterceptedException,
NoAlertPresentException,
NoSuchElementException,
StaleElementReferenceException,
TimeoutException,
WebDriverException,
)
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.remote.webdriver import WebDriver
from selenium.webdriver.remote.webelement import WebElement
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait

# =========================================================

# CONFIG

# =========================================================

@dataclass(frozen=True)
class Config:
base_url: str = "https://www.northerngolfclub.com.au/"

    login_username: str = os.getenv("GOLF_USERNAME", "47394")
    login_password: str = os.getenv("GOLF_PASSWORD", "Northern1!")

    target_day: str = "Sat"

    preferred_start_time: dtime = field(default_factory=lambda: dtime(hour=8, minute=0))
    preferred_end_time: dtime = field(default_factory=lambda: dtime(hour=10, minute=0))

    wait_for_monday_6pm: bool = True

    page_timeout_s: int = 60
    action_timeout_s: int = 20

    headless: bool = False

    screenshot_dir: Path = field(default_factory=lambda: Path("./screenshots"))
    log_file: Path = field(default_factory=lambda: Path("./booking_bot.log"))

    # Booking resource ID found in the nav href:
    # /members/bookings/index.xsp?booking_resource_id=3000000
    # Used as a direct-navigation fallback when the nav link is hidden.
    booking_resource_id: str = os.getenv("GOLF_BOOKING_RESOURCE_ID", "3000000")

    chrome_binary: Optional[str] = None
    chromedriver_path: Optional[str] = None

# =========================================================

# LOGGING

# =========================================================

def build_logger(log_file: Path) -> logging.Logger:
logger = logging.getLogger("golf_booking_bot")
logger.setLevel(logging.INFO)
logger.propagate = False

    if not logger.handlers:
        formatter = logging.Formatter(
            "%(asctime)s | %(levelname)s | %(message)s",
            "%Y-%m-%d %H:%M:%S",
        )

        stream = logging.StreamHandler(sys.stdout)
        stream.setFormatter(formatter)
        logger.addHandler(stream)

        file_handler = logging.FileHandler(log_file, encoding="utf-8")
        file_handler.setFormatter(formatter)
        logger.addHandler(file_handler)

    return logger

# =========================================================

# STATE

# =========================================================

class State(enum.Enum):
WAIT_FOR_RELEASE = "WAIT_FOR_RELEASE"
LOGIN = "LOGIN"
OPEN_EVENT_LIST = "OPEN_EVENT_LIST"
SELECT_TARGET_EVENT = "SELECT_TARGET_EVENT"
OPEN_TARGET_EVENT = "OPEN_TARGET_EVENT"
SCAN_TEE_SHEET = "SCAN_TEE_SHEET"
BOOK_SLOT = "BOOK_SLOT"
CONFIRM_BOOKING = "CONFIRM_BOOKING"
VERIFY_SUCCESS = "VERIFY_SUCCESS"
DONE = "DONE"
FAILED = "FAILED"

# =========================================================

# MODELS

# =========================================================

@dataclass
class Slot:
time_text: str
button: WebElement = field(repr=False)

@dataclass
class SessionContext:
state: Optional[State] = None
target_event_element: Optional[WebElement] = field(default=None, repr=False)
selected_slot: Optional[Slot] = field(default=None, repr=False)
bookings_url: Optional[str] = None
last_error: Optional[str] = None

    # ------------------------------------------------------------------
    # Typed accessors — raise immediately with a clear message rather than
    # letting the caller get an AttributeError on NoneType at runtime.
    # Also satisfies Pyrefly's type narrowing so no missing-attribute
    # errors are raised at the call sites.
    # ------------------------------------------------------------------

    @property
    def slot(self) -> Slot:
        assert self.selected_slot is not None, (
            "selected_slot accessed before scan_tee_sheet() found a slot"
        )
        return self.selected_slot

    @property
    def event_element(self) -> WebElement:
        assert self.target_event_element is not None, (
            "target_event_element accessed before select_target_event() ran"
        )
        return self.target_event_element

# =========================================================

# SELECTORS

# =========================================================

class Selectors:

    MEMBERS_LOGIN = (
        By.XPATH,
        "//a[contains("
        "translate(., 'abcdefghijklmnopqrstuvwxyz', 'ABCDEFGHIJKLMNOPQRSTUVWXYZ')"
        ",'MEMBERS LOGIN')]",
    )

    USERNAME = (By.NAME, "user")

    PASSWORD = (By.NAME, "password")

    LOGIN_SUBMIT = (By.XPATH, "//input[@type='submit']")

    # Ordered most-specific → least-specific.
    # The live DOM shows:  href="/members/bookings/index.xsp?booking_resource_id=3000000"
    # The nav link is inside a collapsed mobile menu so it is NOT visible/clickable
    # in the Selenium sense; we locate by presence and JS-click it.
    BOOKINGS_CANDIDATES = [
        (By.XPATH, "//a[contains(@href,'bookings/index.xsp')]"),
        (By.XPATH, "//a[contains(@href,'booking_resource_id')]"),
        (By.XPATH, "//a[contains(@href,'booking')]"),
        (By.XPATH, "//a[contains(@href,'eventList')]"),
        (
            By.XPATH,
            "//a[contains("
            "translate(normalize-space(.), 'abcdefghijklmnopqrstuvwxyz', 'ABCDEFGHIJKLMNOPQRSTUVWXYZ')"
            ",'BOOKINGS')]",
        ),
    ]

    EVENT_ROWS = (
        By.XPATH,
        "//tr | //div[contains(@class,'row')] | //li",
    )

    SLOT_ROWS = (By.CSS_SELECTOR, "div.row.row-time")

    CONFIRM_YES = (
        By.XPATH,
        "//button[normalize-space()='Yes' or normalize-space()='YES']",
    )

    CONFIRM_BOOKING_BUTTON = (
        By.XPATH,
        "//input[contains(@value,'Confirm')] | "
        "//button[contains(.,'Confirm')] | "
        "//a[contains(.,'Confirm')]",
    )

    SUCCESS_BANNER = (
        By.XPATH,
        "//*[contains("
        "translate(., 'abcdefghijklmnopqrstuvwxyz', 'ABCDEFGHIJKLMNOPQRSTUVWXYZ')"
        ",'CONFIRMED')]",
    )

# =========================================================

# UTILS

# =========================================================

def normalize_spaces(text: str) -> str:
return " ".join(text.split()).strip()

def safe_text(element: WebElement) -> str:
try:
return normalize_spaces(element.text)
except Exception:
return ""

def parse_time(text: str) -> Optional[dtime]:
match = re.search(r"(\d{1,2}):(\d{2})", text)
if not match:
return None

    hour   = int(match.group(1))
    minute = int(match.group(2))
    lower  = text.lower()

    if "pm" in lower and hour != 12:
        hour += 12
    elif "am" in lower and hour == 12:
        hour = 0

    try:
        return dtime(hour=hour, minute=minute)
    except ValueError:
        return None

def matches_target_day(text: str, target_day: str) -> bool:
return (
re.search(rf"\b{re.escape(target_day.lower())}\b", text.lower())
is not None
)

# =========================================================

# DRIVER

# =========================================================

def create_driver(cfg: Config) -> WebDriver:
options = Options()
options.add_argument("--start-maximized")
options.add_argument("--disable-notifications")
options.add_argument("--disable-popup-blocking")
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")

    if cfg.headless:
        options.add_argument("--headless=new")

    if cfg.chrome_binary:
        options.binary_location = cfg.chrome_binary

    driver: WebDriver
    if cfg.chromedriver_path:
        service = Service(cfg.chromedriver_path)
        driver = webdriver.Chrome(service=service, options=options)
    else:
        driver = webdriver.Chrome(options=options)

    driver.set_page_load_timeout(cfg.page_timeout_s)
    return driver

# =========================================================

# BOT

# =========================================================

class BookingBot:

    def __init__(self, driver: WebDriver, cfg: Config, logger: logging.Logger) -> None:
        self.driver = driver
        self.cfg    = cfg
        self.log    = logger
        self.wait   = WebDriverWait(driver, cfg.action_timeout_s)
        self.ctx    = SessionContext()

        cfg.screenshot_dir.mkdir(parents=True, exist_ok=True)

    # ------------------------------------------------------------------
    # Public entry point
    # ------------------------------------------------------------------

    def run(self) -> None:
        try:
            self._transition(State.WAIT_FOR_RELEASE, self._wait_for_release)
            self._transition(State.LOGIN,             self._login)
            self._transition(State.OPEN_EVENT_LIST,   self._open_event_list)
            self._transition(State.SELECT_TARGET_EVENT, self._select_target_event)
            self._transition(State.OPEN_TARGET_EVENT, self._open_target_event)
            self._transition(State.SCAN_TEE_SHEET,    self._scan_tee_sheet)
            self._transition(State.BOOK_SLOT,         self._book_slot)
            self._transition(State.CONFIRM_BOOKING,   self._confirm_booking)
            self._transition(State.VERIFY_SUCCESS,    self._verify_success)

            self.ctx.state = State.DONE
            # ctx.slot uses the typed accessor — never None here
            self.log.info("BOOKING SUCCESS | slot=%s", self.ctx.slot.time_text)

        except Exception as exc:
            self.ctx.state      = State.FAILED
            self.ctx.last_error = str(exc)
            self._capture_debug("fatal_error")
            self.log.exception("Fatal error: %s", exc)

        finally:
            self.driver.quit()

    # ------------------------------------------------------------------
    # State machine helper
    # ------------------------------------------------------------------

    def _transition(self, state: State, action) -> None:
        self.ctx.state = state
        self.log.info("STATE => %s", state.value)
        action()

    # ------------------------------------------------------------------
    # States
    # ------------------------------------------------------------------

    def _wait_for_release(self) -> None:
        if not self.cfg.wait_for_monday_6pm:
            return

        now = datetime.now()

        if now.weekday() != 0:            # 0 = Monday
            self.log.info(
                "Release gate enabled but today is not Monday (%s); proceeding.",
                now.strftime("%A"),
            )
            return

        while now.hour < 18:
            self.log.info(
                "Waiting for Monday 18:00 release (now %s)…",
                now.strftime("%H:%M:%S"),
            )
            time.sleep(15)
            now = datetime.now()

        self.log.info("Release gate cleared.")

    # ------------------------------------------------------------------

    def _login(self) -> None:
        self.driver.get(self.cfg.base_url)
        self._wait_for_dom_ready()

        self._click(
            self.wait.until(EC.element_to_be_clickable(Selectors.MEMBERS_LOGIN)),
            "MEMBERS LOGIN",
        )

        time.sleep(2)

        if len(self.driver.window_handles) > 1:
            self.driver.switch_to.window(self.driver.window_handles[-1])

        self._type_into(Selectors.USERNAME, self.cfg.login_username)
        self._type_into(Selectors.PASSWORD, self.cfg.login_password)

        self._click(
            self.wait.until(EC.element_to_be_clickable(Selectors.LOGIN_SUBMIT)),
            "LOGIN SUBMIT",
        )

        time.sleep(5)
        self._wait_for_dom_ready()
        self.log.info("Login successful.")

    # ------------------------------------------------------------------

    def _open_event_list(self) -> None:
        # --- Strategy 1: locate the nav <a> by presence (not visibility) and
        #     JS-click it.  The link lives inside a collapsed mobile nav so
        #     element_to_be_clickable always times out.
        bookings: Optional[WebElement] = None

        for selector in Selectors.BOOKINGS_CANDIDATES:
            try:
                bookings = WebDriverWait(self.driver, 6).until(
                    EC.presence_of_element_located(selector)
                )
                self.log.info("Bookings link found via selector: %s", selector)
                break
            except TimeoutException:
                continue

        if bookings is not None:
            # JS click bypasses visibility/interactability constraints.
            self.driver.execute_script("arguments[0].click();", bookings)
            self.log.info("JS-clicked Bookings nav link.")
            time.sleep(4)
            self._wait_for_dom_ready()
            self.ctx.bookings_url = self.driver.current_url
            self.log.info("Opened bookings page: %s", self.ctx.bookings_url)
            return

        # --- Strategy 2: direct URL navigation using the known resource ID.
        #     Most reliable when the nav is collapsed or the DOM structure changes.
        #     URL pattern confirmed from live DOM:
        #       /members/bookings/index.xsp?booking_resource_id=3000000
        direct_url = (
            self.cfg.base_url.rstrip("/")
            + "/members/bookings/index.xsp"
            + f"?booking_resource_id={self.cfg.booking_resource_id}"
        )
        self.log.info("Nav link not found; navigating directly to: %s", direct_url)
        self.driver.get(direct_url)
        self._wait_for_dom_ready()
        self.ctx.bookings_url = self.driver.current_url
        self.log.info("Opened bookings page via direct URL: %s", self.ctx.bookings_url)

    # ------------------------------------------------------------------

    def _select_target_event(self) -> None:
        rows = self.driver.find_elements(*Selectors.EVENT_ROWS)
        self.log.info("Scanning %d event rows.", len(rows))

        for row in rows:
            try:
                text = safe_text(row)
                if not text:
                    continue

                if not matches_target_day(text, self.cfg.target_day):
                    continue

                if "OPEN" not in text.upper():
                    continue

                self.log.info("Matched Saturday OPEN row: %s", text)

                try:
                    clickable = row.find_element(
                        By.XPATH,
                        ".//a[contains(@href,'event')] | .//a | .//button",
                    )
                    self.ctx.target_event_element = clickable
                    return
                except NoSuchElementException:
                    continue

            except StaleElementReferenceException:
                continue

        raise RuntimeError(
            f"No OPEN {self.cfg.target_day} event found on the bookings page."
        )

    # ------------------------------------------------------------------

    def _open_target_event(self) -> None:
        # ctx.event_element uses the typed accessor — raises clearly if None
        element = self.ctx.event_element
        self._scroll_into_view(element)
        self._click(element, "Saturday Event")
        time.sleep(4)
        self._wait_for_dom_ready()
        self.log.info("Saturday event opened.")

    # ------------------------------------------------------------------

    def _scan_tee_sheet(self) -> None:
        rows = self.driver.find_elements(*Selectors.SLOT_ROWS)
        self.log.info("Scanning %d tee rows.", len(rows))

        for row in rows:
            try:
                row_text = safe_text(row)

                if "BOOK GROUP" not in row_text.upper():
                    continue

                time_match = re.search(
                    r"\d{1,2}:\d{2}\s*[ap]m",
                    row_text,
                    re.IGNORECASE,
                )
                if not time_match:
                    continue

                time_text = time_match.group(0)
                parsed    = parse_time(time_text)

                if parsed is None:
                    continue

                if not (self.cfg.preferred_start_time <= parsed <= self.cfg.preferred_end_time):
                    continue

                button = row.find_element(
                    By.XPATH,
                    ".//button[contains(.,'BOOK GROUP')] | "
                    ".//a[contains(.,'BOOK GROUP')]",
                )

                self.ctx.selected_slot = Slot(time_text=time_text, button=button)
                self.log.info("Selected slot: %s", time_text)
                return

            except (NoSuchElementException, StaleElementReferenceException):
                continue
            except Exception:
                continue

        raise RuntimeError(
            f"No BOOK GROUP slot found in window "
            f"{self.cfg.preferred_start_time}–{self.cfg.preferred_end_time}."
        )

    # ------------------------------------------------------------------

    def _book_slot(self) -> None:
        # ctx.slot uses the typed accessor — raises clearly if None
        slot = self.ctx.slot

        self._scroll_into_view(slot.button)
        self._click(slot.button, f"BOOK GROUP {slot.time_text}")
        self.log.info("Clicked BOOK GROUP: %s", slot.time_text)

        time.sleep(2)

        try:
            alert = self.driver.switch_to.alert
            self.log.warning("Alert detected: %s", alert.text)
            alert.accept()
        except NoAlertPresentException:
            pass

    # ------------------------------------------------------------------

    def _confirm_booking(self) -> None:
        try:
            yes_btn = WebDriverWait(self.driver, 5).until(
                EC.element_to_be_clickable(Selectors.CONFIRM_YES)
            )
            self._click(yes_btn, "YES BUTTON")
            self.log.info("Clicked YES.")
        except TimeoutException:
            self.log.info("No YES modal appeared; proceeding to confirm.")

        confirm = self.wait.until(
            EC.element_to_be_clickable(Selectors.CONFIRM_BOOKING_BUTTON)
        )
        self._click(confirm, "CONFIRM BOOKING")
        self.log.info("Clicked CONFIRM BOOKING.")
        time.sleep(4)

    # ------------------------------------------------------------------

    def _verify_success(self) -> None:
        current_url = self.driver.current_url.lower()

        if "event.msp" in current_url or "open/event" in current_url:
            self.log.info("Booking verified via URL redirect.")
            return

        banner = self.wait.until(
            EC.presence_of_element_located(Selectors.SUCCESS_BANNER)
        )
        self.log.info("Booking confirmed: %s", safe_text(banner))

    # ------------------------------------------------------------------
    # Helpers
    # ------------------------------------------------------------------

    def _wait_for_dom_ready(self) -> None:
        WebDriverWait(self.driver, self.cfg.page_timeout_s).until(
            lambda d: d.execute_script("return document.readyState") == "complete"
        )

    def _type_into(self, selector: tuple, value: str) -> None:
        element = self.wait.until(EC.presence_of_element_located(selector))
        element.clear()
        element.send_keys(value)

    def _click(self, element: WebElement, description: str) -> None:
        try:
            self._scroll_into_view(element)
            element.click()
        except (ElementClickInterceptedException, WebDriverException):
            self.log.debug(
                "Native click intercepted on '%s'; falling back to JS click.",
                description,
            )
            self.driver.execute_script("arguments[0].click();", element)

    def _scroll_into_view(self, element: WebElement) -> None:
        self.driver.execute_script(
            "arguments[0].scrollIntoView({block:'center'});",
            element,
        )
        time.sleep(random.uniform(0.1, 0.3))

    def _capture_debug(self, name: str) -> None:
        try:
            ts   = datetime.now().strftime("%Y%m%d_%H%M%S")
            path = self.cfg.screenshot_dir / f"{ts}_{name}.png"
            self.driver.save_screenshot(str(path))
            self.log.info("Saved screenshot: %s", path)
        except Exception:
            pass

# =========================================================

# MAIN

# =========================================================

def main() -> None:
cfg = Config()
logger = build_logger(cfg.log_file)

    logger.info(
        "Loaded config: target_day=%s preferred=%s–%s release_gate=%s",
        cfg.target_day,
        cfg.preferred_start_time,
        cfg.preferred_end_time,
        cfg.wait_for_monday_6pm,
    )

    driver = create_driver(cfg)
    bot    = BookingBot(driver, cfg, logger)
    bot.run()

if **name** == "**main**":
main()
