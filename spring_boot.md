// IpBlockInterceptor.java
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import jakarta.servlet.http.*;
import java.net.URI; import java.net.http.*;
import java.time.Duration;

@Component
public class IpBlockInterceptor implements HandlerInterceptor {

    private static final String API_KEY = "YOUR_API_KEY";
    private static final String BASE    = "https://api.ip-block.com/v1";
    private final HttpClient client = HttpClient.newBuilder()
        .connectTimeout(Duration.ofMillis(500)).build();

    private boolean apiReachable() {
        try {
            var req = HttpRequest.newBuilder()
                .uri(URI.create(BASE + "/ping"))
                .timeout(Duration.ofMillis(500)).GET().build();
            var res = client.send(req, HttpResponse.BodyHandlers.ofString());
            return res.body().contains("\"status\":\"ok\"");
        } catch (Exception e) { return false; }
    }

    @Override
    public boolean preHandle(HttpServletRequest req,
                               HttpServletResponse res, Object handler) throws Exception {
        if (!apiReachable()) return true; // fail open

        String ip  = req.getRemoteAddr();
        String ua  = req.getHeader("User-Agent") != null ? req.getHeader("User-Agent") : ""; // optional
        String ref = req.getHeader("Referer")     != null ? req.getHeader("Referer")     : ""; // optional
        String body = String.format(
            "{\"api_key\":\"%s\",\"ip\":\"%s\",\"site_id\":\"ABCDEFGHIJKL\",\"user_agent\":\"%s\",\"referrer\":\"%s\"}",
            API_KEY, ip, ua, ref);

        var request = HttpRequest.newBuilder()
            .uri(URI.create(BASE + "/check"))
            .header("Content-Type", "application/json")
            .timeout(Duration.ofSeconds(2))
            .POST(HttpRequest.BodyPublishers.ofString(body)).build();
        var response = client.send(request, HttpResponse.BodyHandlers.ofString());
        if (response.body().contains("\"action\":\"block\"")) {
            res.sendRedirect("https://www.ip-block.com/blocked.php"); return false;
        }
        return true;
    }
}

// WebConfig.java — register the interceptor:
// @Configuration
// public class WebConfig implements WebMvcConfigurer {
//     @Autowired IpBlockInterceptor ipBlock;
//     @Override public void addInterceptors(InterceptorRegistry r) {
//         r.addInterceptor(ipBlock);
//     }
// }