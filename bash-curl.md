#!/usr/bin/env bash
API_KEY="YOUR_API_KEY"
BASE="https://api.ip-block.com/v1"
IP="${1:-$REMOTE_ADDR}"  # pass IP as first arg or use env var
SITE_ID="ABCDEFGHIJKL"
USER_AGENT="${HTTP_USER_AGENT:-}"  # optional
REFERRER="${HTTP_REFERER:-}"      # optional

# 1. Check API is reachable (500 ms timeout)
PING=$(curl --silent --max-time 0.5 \
  "$BASE/ping")

STATUS=$(echo "$PING" | grep -o '"status":"ok"')

if [[ -z "$STATUS" ]]; then
  echo "API unreachable — failing open" >&2
  exit 0
fi

# 2. Check visitor IP
RESPONSE=$(curl --silent --max-time 2 \
  --request POST \
  --header "Content-Type: application/json" \
  --data "$(cat <<EOF
{
  "api_key":    "$API_KEY",
  "ip":         "$IP",
  "site_id":    "$SITE_ID",
  "user_agent": "$USER_AGENT",
  "referrer":   "$REFERRER"
}
EOF
)" \
  "$BASE/check")

ACTION=$(echo "$RESPONSE" | grep -o '"action":"[^"]*"' | sed 's/"action":"//;s/"//')

if [[ "$ACTION" == "block" ]]; then
  echo "Blocked: $IP"
  exit 1
fi
exit 0