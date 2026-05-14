// build.gradle.kts deps: ktor-server-core, ktor-client-cio, kotlinx-serialization-json
import io.ktor.client.*
import io.ktor.client.engine.cio.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.server.application.*
import io.ktor.server.plugins.*
import io.ktor.http.*
import kotlinx.serialization.json.*

private const val API_KEY = "YOUR_API_KEY"
private const val BASE    = "https://api.ip-block.com/v1"
private val       client  = HttpClient(CIO) {
    engine { requestTimeout = 2_000 }
}

suspend fun apiReachable(): Boolean = try {
    val res  = client.get("$BASE/ping") {
        timeout { requestTimeoutMillis = 500 }
    }
    Json.parseToJsonElement(res.bodyAsText())
        .jsonObject["status"]?.jsonPrimitive?.content == "ok"
} catch (e: Exception) { false }

// Install in Application module:
fun Application.configureIpBlock() {
    intercept(ApplicationCallPipeline.Plugins) {
        if (!apiReachable()) return@intercept // fail open
        val ip  = call.request.origin.remoteHost
        val ua  = call.request.headers["User-Agent"] ?: ""  // optional
        val ref = call.request.headers["Referer"]    ?: ""  // optional
        val res = client.post("$BASE/check") {
            contentType(ContentType.Application.Json)
            setBody("""{"api_key":"$API_KEY","ip":"$ip","site_id":"ABCDEFGHIJKL","user_agent":"$ua","referrer":"$ref"}""")
        }
        val action = Json.parseToJsonElement(res.bodyAsText())
            .jsonObject["action"]?.jsonPrimitive?.content
        if (action == "block") {
            call.respondRedirect("https://www.ip-block.com/blocked.php"); finish()
        }
    }
}