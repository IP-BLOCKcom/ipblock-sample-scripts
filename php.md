// 1. Confirm the API is reachable before enforcing blocks
$ctx = stream_context_create(["http" => [
  "method"  => "GET",
  "timeout" => 0.5,
  "ignore_errors" => true
]]);
$ping = file_get_contents(
  "https://api.ip-block.com/v1/ping", false, $ctx
);
$apiUp = ($ping !== false
  && (json_decode($ping, true)["status"] ?? "") === "ok");

// 2. Only enforce if the API responded; fail open otherwise
if ($apiUp) {
  $response = file_get_contents(
    "https://api.ip-block.com/v1/check",
    false,
    stream_context_create(["http" => [
      "method"  => "POST",
      "header"  => "Content-Type: application/json",
      "content" => json_encode([
        "api_key"    => "YOUR_API_KEY",
        "ip"         => $_SERVER["REMOTE_ADDR"],
        "site_id"    => "ABCDEFGHIJKL",
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "", // optional
        "referrer"   => $_SERVER["HTTP_REFERER"]    ?? ""  // optional
      ])
    ]])
  );
  $result = json_decode($response, true);
  if ($result["action"] === "block") {
    header('Location: https://www.ip-block.com/blocked.php'); exit;
  }
}