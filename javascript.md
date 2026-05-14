// Helper: returns true if the API is reachable
async function apiReachable() {
  try {
    const p = await fetch(
      "https://api.ip-block.com/v1/ping",
      { signal: AbortSignal.timeout(500) }
    );
    const { status } = await p.json();
    return status === "ok";
  } catch { return false; }
}

// Call before allowing a sensitive action (login, checkout, etc.)
async function isVisitorAllowed(visitorIp) {
  if (!(await apiReachable())) return true; // fail open

  const r = await fetch(
    "https://api.ip-block.com/v1/check",
    { method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        api_key:    "YOUR_API_KEY",
        ip:         visitorIp,
        site_id:    "ABCDEFGHIJKL",
        user_agent: navigator.userAgent,  // optional
        referrer:   document.referrer     // optional
      })
    }
  );
  const { action } = await r.json();
  return action !== "block";
}

// Example: guard a form submit handler
// The browser can't self-determine its public IP — fetch it from your server
document.querySelector("#login-form")
  .addEventListener("submit", async (e) => {
    e.preventDefault();
    const { ip } = await fetch("/api/my-ip").then(r => r.json());
    if (!(await isVisitorAllowed(ip))) {
      alert("Access denied."); return;
    }
    // proceed with login logic...
  });