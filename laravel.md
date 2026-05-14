// app/Http/Middleware/IpBlock.php
namespace App\Http\Middleware;

use Closure; use Illuminate\Http\Request;

class IpBlock
{
    private const API_KEY = 'YOUR_API_KEY';
    private const BASE    = 'https://api.ip-block.com/v1';

    private function apiReachable(): bool
    {
        try {
            $ctx = stream_context_create(['http' => ['timeout' => 0.5, 'ignore_errors' => true]]);
            $res = file_get_contents(self::BASE . '/ping', false, $ctx);
            return $res !== false
                && (json_decode($res, true)['status'] ?? '') === 'ok';
        } catch (\Throwable) { return false; }
    }

    public function handle(Request $request, Closure $next): mixed
    {
        if ($this->apiReachable()) {
            $ctx = stream_context_create(['http' => [
                'method'        => 'POST',
                'header'        => 'Content-Type: application/json',
                'ignore_errors' => true,
                'content'       => json_encode([
                    'api_key'    => self::API_KEY,
                    'ip'         => $request->ip(),
                    'site_id'    => 'ABCDEFGHIJKL',
                    'user_agent' => $request->userAgent() ?? '',         // optional
                    'referrer'   => $request->headers->get('referer', ''), // optional
                ]),
            ]]);
            $res    = file_get_contents(self::BASE . '/check', false, $ctx);
            $result = json_decode($res, true);
            if (($result['action'] ?? '') === 'block') {
                return redirect('https://www.ip-block.com/blocked.php');
            }
        }
        return $next($request);
    }
}

// Register in bootstrap/app.php (Laravel 11+):
// ->withMiddleware(function (Middleware $m) {
//     $m->prepend(\App\Http\Middleware\IpBlock::class);
// })