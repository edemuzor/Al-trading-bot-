"""
IQ Option martingale script (demo/testing)
- Asset: EURUSD OTC (set ASSET variable)
- Entry: 02:02 WAT (Africa/Lagos)
- Expiries: 02:03, 02:04, 02:05 (1-min expiry each)
- Action: BUY
- Martingale levels: stake doubles after each loss (1,2,4)
- Uses iqoptionapi (unofficial). Test on demo account only.
"""

import time
import datetime as dt
from pytz import timezone
import logging
from iqoptionapi.stable_api import IQ_Option

# -----------------------
# CONFIG
# -----------------------
TZ = timezone("Africa/Lagos")
EMAIL = "your_demo_email@example.com"   # replace with demo account email
PASSWORD = "your_demo_password"         # replace
MODE_DEMO = True                        # True = demo/wallet, False = real (not recommended)
ASSET = "EURUSD-OTC"                    # common IQ naming; try "EURUSD" if this doesn't exist
BASE_STAKE = 1.0                        # base stake in account currency (e.g. USD)
STAKE_MULTIPLIER = 2                    # martingale multiplier
MAX_LEVELS = 3                          # number of attempts (levels)
ENTRY_TIME = (2, 2)                     # (hour, minute) WAT for initial entry -> (02:02)
MARTINGALE_EXPIRIES = [(2,3), (2,4), (2,5)]  # expiries as requested
LOG_LEVEL = logging.INFO

# -----------------------
# Logging
# -----------------------
logging.basicConfig(level=LOG_LEVEL, format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger("iq-martingale")

# -----------------------
# Helpers
# -----------------------
def next_occurrence(hm):
    now = dt.datetime.now(TZ)
    h, m = hm
    cand = TZ.localize(dt.datetime(now.year, now.month, now.day, h, m, 0))
    if cand <= now:
        cand = cand + dt.timedelta(days=1)
    return cand

def sleep_until(ts):
    now = dt.datetime.now(TZ)
    s = (ts - now).total_seconds()
    if s > 0:
        logger.info(f"Sleeping {s:.1f}s until {ts.strftime('%Y-%m-%d %H:%M:%S %Z')}")
        time.sleep(s)

# -----------------------
# IQ Option helpers
# -----------------------
def connect_iq(email, password, demo=True, timeout=15):
    iq = IQ_Option(email, password)
    if demo:
        iq.connect()  # stable_api connects to demo by default in many forks
        # Ensure demo wallet selected (some forks provide change_balance)
        try:
            iq.change_balance("PRACTICE")  # PRACTICE = demo, REAL = real
        except Exception:
            pass
    else:
        iq.connect()
        try:
            iq.change_balance("REAL")
        except Exception:
            pass

    # wait for connection
    t0 = time.time()
    while not iq.check_connect() and time.time() - t0 < timeout:
        logger.info("Waiting for IQ connection...")
        time.sleep(0.5)
    if not iq.check_connect():
        raise ConnectionError("Unable to connect to IQ Option API")
    logger.info("Connected to IQ Option (demo=%s)", demo)
    return iq

def place_digital(iq, asset, action, amount, expiry_min=1):
    """
    Place a digital option:
    - action: 'call' or 'put' (call = BUY / up, put = SELL / down)
    - expiry_min: in minutes (1 for 1-min expiry)
    Returns (success_bool, deal_id)
    """
    # many forks expect active to be like "EURUSD-OTC" or asset id; adapt if needed
    # IQ_Option.buy(amount, asset, action, expiration) -> returns (success, id) for digital
    try:
        success, trade_id = iq.buy(amount, asset, action, expiry_min)
        return success, trade_id
    except Exception as e:
        logger.exception("Error placing trade: %s", e)
        return False, None

def check_digital_result(iq, trade_id, timeout=70):
    """
    Poll result for digital trade id until it closes.
    Returns win==True/False/None(on error)
    """
    t0 = time.time()
    while time.time() - t0 < timeout:
        try:
            res = iq.check_win(trade_id)  # returns payout or None if not closed (depends on fork)
            # In many forks check_win returns a float payout if closed, or False/None if not.
            if res is not None and res is not False:
                # positive payout indicates win (payout > 0)
                return float(res) > 0
        except Exception:
            pass
        time.sleep(0.5)
    logger.warning("Timeout while waiting for trade result for id %s", trade_id)
    return None

# -----------------------
# Orchestration
# -----------------------
def run_sequence():
    # compute datetimes
    entry_dt = next_occurrence(ENTRY_TIME)
    expiry_dts = []
    # assign expiry datetimes (ensure they are after entry)
    for h,m in MARTINGALE_EXPIRIES:
        ed = TZ.localize(dt.datetime(entry_dt.year, entry_dt.month, entry_dt.day, h, m, 0))
        if ed <= entry_dt:
            ed = ed + dt.timedelta(days=1)
        expiry_dts.append(ed)

    logger.info("Planned ENTRY at %s", entry_dt.strftime("%Y-%m-%d %H:%M:%S %Z"))
    for i,ed in enumerate(expiry_dts, start=1):
        logger.info("Level %d expiry at %s (stake x%d)", i, ed.strftime("%Y-%m-%d %H:%M:%S %Z"), STAKE_MULTIPLIER**(i-1))

    # connect
    iq = connect_iq(EMAIL, PASSWORD, demo=MODE_DEMO)

    # Wait until entry time
    sleep_until(entry_dt)

    stake = BASE_STAKE
    last_entry_time = entry_dt

    for level_idx, expiry_dt in enumerate(expiry_dts):
        level = level_idx + 1
        # entry time for this level = last_entry_time
        entry_time_for_level = last_entry_time
        stake_level = stake * (STAKE_MULTIPLIER ** (level_idx))
        logger.info("Level %d: placing BUY (CALL) stake=%.2f entry=%s expiry=%s",
                    level, stake_level,
                    entry_time_for_level.strftime("%H:%M:%S %Z"),
                    expiry_dt.strftime("%H:%M:%S %Z"))

        # place trade: call = BUY/up direction
        success, trade_id = place_digital(iq, ASSET, "call", stake_level, expiry_min=1)
        if not success:
            logger.error("Trade placement failed at level %d. Aborting sequence.", level)
            break
        logger.info("Trade placed id=%s. Waiting for result...", trade_id)

        # Wait for result - some forks return immediate known result; we poll
        win = check_digital_result(iq, trade_id)
        if win is True:
            logger.info("Level %d WIN. Stake=%.2f. Ending sequence.", level, stake_level)
            break
        elif win is False:
            logger.info("Level %d LOSS. Stake=%.2f.", level, stake_level)
            # proceed to next level if any
            if level_idx == MAX_LEVELS - 1:
                logger.info("Reached max martingale level and lost. Sequence ended.")
                break
            else:
                # next entry happens immediately at this expiry time
                last_entry_time = expiry_dt
                # continue loop
                continue
        else:
            logger.warning("Unknown result (None) for trade id %s. Ending sequence.", trade_id)
            break

    # disconnect
    try:
        iq.close()
    except Exception:
        pass
    logger.info("Sequence finished.")

# -----------------------
# MAIN
# -----------------------
if __name__ == "__main__":
    logger.info("Starting IQ Option martingale (demo=%s) for asset %s", MODE_DEMO, ASSET)
    try:
        run_sequence()
    except Exception as e:
        logger.exception("Unhandled error: %s", e)
