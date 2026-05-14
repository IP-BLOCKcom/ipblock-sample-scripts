# ip_block/middleware.py
import requests
from django.http import HttpResponseRedirect

API_KEY = "YOUR_API_KEY"
BASE    = "https://api.ip-block.com/v1"

class IpBlockMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if self._api_reachable():
            try:
                r = requests.post(
                    f"{BASE}/check",
                    json={
                        "api_key":    API_KEY,
                        "ip":         request.META.get("REMOTE_ADDR", ""),
                        "site_id":    "ABCDEFGHIJKL",
                        "user_agent": request.META.get("HTTP_USER_AGENT", ""),  # optional
                        "referrer":   request.META.get("HTTP_REFERER",    ""),  # optional
                    },
                    timeout=2
                )
                if r.json().get("action") == "block":
                    return HttpResponseRedirect('https://www.ip-block.com/blocked.php')
            except Exception:
                pass  # fail open on check error
        return self.get_response(request)

    def _api_reachable(self):
        try:
            r = requests.get(f"{BASE}/ping", timeout=0.5)
            return r.json().get("status") == "ok"
        except Exception:
            return False

# Add to MIDDLEWARE in settings.py (before other middleware):
# 'ip_block.middleware.IpBlockMiddleware',