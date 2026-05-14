import requests

API_KEY = "YOUR_API_KEY"
BASE    = "https://api.ip-block.com/v1"

# Helper: returns True if the API is reachable
def api_reachable():
    try:
        r = requests.get(
            f"{BASE}/ping", timeout=0.5
        )
        return r.json().get("status") == "ok"
    except Exception:
        return False

# Check visitor IP; returns False if blocked
def visitor_allowed(ip, site_id="ABCDEFGHIJKL", user_agent="", referrer=""):
    if not api_reachable():
        return True  # fail open
    r = requests.post(
        f"{BASE}/check",
        json={
            "api_key":    API_KEY,
            "ip":         ip,
            "site_id":    site_id,
            "user_agent": user_agent,  # optional
            "referrer":   referrer     # optional
        },
        timeout=2
    )
    return r.json().get("action") != "block"

# Flask example
from flask import request, abort

@app.before_request
def check_ip():
    if not visitor_allowed(
        request.remote_addr,
        user_agent=request.headers.get("User-Agent", ""),  # optional
        referrer=request.headers.get("Referer", "")        # optional
    ):
        from flask import redirect as flask_redirect
        return flask_redirect('https://www.ip-block.com/blocked.php')