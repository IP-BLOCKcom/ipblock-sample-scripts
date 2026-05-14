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

// Middleware: fail open if API is unreachable
app.use(async (req, res, next) => {
  if (!(await apiReachable())) return next();

  const r = await fetch(
    "https://api.ip-block.com/v1/check",
    { method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        api_key:    "YOUR_API_KEY",
        ip:         req.ip,
        site_id:    "ABCDEFGHIJKL",
        user_agent: req.headers['user-agent'] ?? '', // optional
        referrer:   req.headers['referer']     ?? ''  // optional
      })
    }
  );
  const { action } = await r.json();
  action === "block"
    ? res.redirect('https://www.ip-block.com/blocked.php')
    : next();
});