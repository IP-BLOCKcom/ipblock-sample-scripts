package main

import (
  "bytes"; "context"; "encoding/json"
  "net"; "net/http"; "time"
)

const (
  apiKey = "YOUR_API_KEY"
  base   = "https://api.ip-block.com/v1"
)

// apiReachable pings /v1/ping with a 500 ms deadline.
func apiReachable() bool {
  ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
  defer cancel()
  req, _ := http.NewRequestWithContext(ctx, http.MethodGet, base+"/ping", nil)
  res, err := http.DefaultClient.Do(req)
  if err != nil { return false }
  defer res.Body.Close()
  var body struct{ Status string `json:"status"` }
  json.NewDecoder(res.Body).Decode(&body)
  return body.Status == "ok"
}

// IpBlockMiddleware wraps any http.Handler.
func IpBlockMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if !apiReachable() { next.ServeHTTP(w, r); return } // fail open

    host, _, _ := net.SplitHostPort(r.RemoteAddr)
    payload, _ := json.Marshal(map[string]string{
      "api_key":    apiKey,
      "ip":         host,
      "site_id":    "ABCDEFGHIJKL",
      "user_agent": r.Header.Get("User-Agent"), // optional
      "referrer":   r.Header.Get("Referer"),     // optional
    })
    res, err := http.Post(base+"/check", "application/json", bytes.NewReader(payload))
    if err != nil { next.ServeHTTP(w, r); return }
    defer res.Body.Close()

    var body struct{ Action string `json:"action"` }
    json.NewDecoder(res.Body).Decode(&body)
    if body.Action == "block" { http.Redirect(w, r, "https://www.ip-block.com/blocked.php", http.StatusFound); return }
    next.ServeHTTP(w, r)
  })
}