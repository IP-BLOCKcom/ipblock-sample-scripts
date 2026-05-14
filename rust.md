// Cargo.toml deps: axum, reqwest, serde_json, tokio
use axum::{
    extract::ConnectInfo, http::StatusCode,
    middleware::{self, Next}, response::Response,
    body::Body, http::Request,
};
use std::net::SocketAddr;

const API_KEY: &str = "YOUR_API_KEY";
const BASE:    &str = "https://api.ip-block.com/v1";

async fn api_reachable() -> bool {
    let client = reqwest::Client::builder()
        .timeout(std::time::Duration::from_millis(500))
        .build().unwrap();
    if let Ok(r) = client.get("https://api.ip-block.com/v1/ping").send().await {
        if let Ok(json) = r.json::<serde_json::Value>().await {
            return json["status"] == "ok";
        }
    }
    false
}

pub async fn ip_block_layer(
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
    req: Request<Body>, next: Next,
) -> Result<Response, StatusCode> {
    if !api_reachable().await {
        return Ok(next.run(req).await); // fail open
    }
    let body = serde_json::json!({
        "api_key":    API_KEY,
        "ip":         addr.ip().to_string(),
        "site_id":    "ABCDEFGHIJKL",
        "user_agent": req.headers().get("user-agent").and_then(|v| v.to_str().ok()).unwrap_or(""), // optional
        "referrer":   req.headers().get("referer").and_then(|v| v.to_str().ok()).unwrap_or("")   // optional
    });
    let resp = reqwest::Client::new()
        .post("https://api.ip-block.com/v1/check")
        .json(&body).send().await
        .map_err(|_| StatusCode::OK)?;
    let json: serde_json::Value = resp.json().await
        .map_err(|_| StatusCode::OK)?;
    if json["action"] == "block" {
        return Ok(axum::response::Redirect::to("https://www.ip-block.com/blocked.php").into_response());
    }
    Ok(next.run(req).await)
}