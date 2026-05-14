// IpBlockFilter.java — register as a servlet filter in web.xml or @WebFilter
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import java.io.*; import java.net.*; import java.nio.charset.StandardCharsets;
import org.json.*;

public class IpBlockFilter implements Filter {

  private static final String API_KEY = "YOUR_API_KEY";
  private static final String BASE    = "https://api.ip-block.com/v1";

  /** Ping /v1/ping; returns true if API responds with {"status":"ok"} */
  private boolean apiReachable() {
    try {
      HttpURLConnection c = (HttpURLConnection)
        new URL(BASE + "/ping").openConnection();
      c.setConnectTimeout(500); c.setReadTimeout(500);
      c.setRequestMethod("GET");
      if (c.getResponseCode() != 200) return false;
      String body = new String(
        c.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
      return "ok".equals(new JSONObject(body).optString("status"));
    } catch (Exception e) { return false; }
  }

  public void doFilter(ServletRequest req, ServletResponse res,
                         FilterChain chain) throws IOException, ServletException {
    if (!apiReachable()) { chain.doFilter(req, res); return; } // fail open

    HttpServletRequest httpReq = (HttpServletRequest) req;
    String ip  = req.getRemoteAddr();
    String ua  = httpReq.getHeader("User-Agent");  // optional
    String ref = httpReq.getHeader("Referer");     // optional
    String payload = "{\"api_key\":\""    + API_KEY +
                     "\",\"ip\":\""       + ip +
                     "\",\"site_id\":\"ABCDEFGHIJKL\"" +
                     ",\"user_agent\":\"" + (ua  != null ? ua  : "") + "\"" +
                     ",\"referrer\":\""   + (ref != null ? ref : "") + "\"}";

    HttpURLConnection c = (HttpURLConnection)
      new URL(BASE + "/check").openConnection();
    c.setRequestMethod("POST");
    c.setRequestProperty("Content-Type", "application/json");
    c.setDoOutput(true);
    c.getOutputStream().write(payload.getBytes(StandardCharsets.UTF_8));

    String body = new String(
      c.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
    String action = new JSONObject(body).optString("action", "allow");

    if ("block".equals(action)) {
      ((HttpServletResponse) res).sendRedirect("https://www.ip-block.com/blocked.php");
    } else {
      chain.doFilter(req, res);
    }
  }
}