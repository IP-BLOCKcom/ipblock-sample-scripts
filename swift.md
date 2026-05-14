// Sources/App/Middleware/IpBlockMiddleware.swift
import Vapor

struct IpBlockMiddleware: AsyncMiddleware {

    let apiKey = "YOUR_API_KEY"
    let base   = "https://api.ip-block.com/v1"

    /// Returns true if the API responds with {"status":"ok"} within 500 ms.
    func apiReachable(on req: Request) async -> Bool {
        do {
            let res = try await req.client.get(
                URI(string: "\(base)/ping"), headers: [:])
            let json = try res.content.decode([String:String].self)
            return json["status"] == "ok"
        } catch { return false }
    }

    func respond(to req: Request, chainingTo next: AsyncResponder)
        async throws -> Response {

        guard await apiReachable(on: req)
        else { return try await next.respond(to: req) } // fail open

        struct Payload: Content {
            var api_key, ip, site_id, user_agent, referrer: String
        }
        let ua  = req.headers.first(name: "User-Agent") ?? ""  // optional
        let ref = req.headers.first(name: "Referer")    ?? ""  // optional
        let res = try await req.client.post(
            URI(string: "\(base)/check")) { r in
            try r.content.encode(
                Payload(api_key:    apiKey,
                        ip:         req.remoteAddress?.hostname ?? "",
                        site_id:    "ABCDEFGHIJKL",
                        user_agent: ua,
                        referrer:   ref))
        }
        let action = try? res.content.get(String.self, at: "action")
        if action == "block" { return req.redirect(to: "https://www.ip-block.com/blocked.php") }
        return try await next.respond(to: req)
    }
}

// In configure.swift: app.middleware.use(IpBlockMiddleware())