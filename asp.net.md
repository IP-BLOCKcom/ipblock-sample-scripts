// Program.cs — register before app.UseRouting()
using System.Net.Http.Json;

var http = new HttpClient {
    Timeout = TimeSpan.FromSeconds(2)
};
const string API_KEY = "YOUR_API_KEY";
const string BASE    = "https://api.ip-block.com/v1";

// Helper: returns true if the API is reachable
async Task<bool> ApiReachable() {
    try {
        using var cts = new CancellationTokenSource(500);
        var r = await http.GetFromJsonAsync<JsonElement>(
            BASE + "/ping", cts.Token);
        return r.GetProperty("status").GetString() == "ok";
    } catch { return false; }
}

// Middleware lambda
app.Use(async (context, next) => {
    if (!await ApiReachable()) { await next(); return; } // fail open

    var ip = context.Connection.RemoteIpAddress?.ToString();
    var payload = new {
        api_key    = API_KEY,
        ip,
        site_id    = "ABCDEFGHIJKL",
        user_agent = context.Request.Headers["User-Agent"].ToString(),  // optional
        referrer   = context.Request.Headers["Referer"].ToString()      // optional
    };
    var resp   = await http.PostAsJsonAsync(BASE + "/check", payload);
    var result = await resp.Content.ReadFromJsonAsync<JsonElement>();
    var action = result.GetProperty("action").GetString();

    if (action == "block") {
        context.Response.Redirect("https://www.ip-block.com/blocked.php");
        return;
    }
    await next();
});