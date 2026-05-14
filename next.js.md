// middleware.ts (project root — runs on the Edge Runtime)
import { NextRequest, NextResponse } from 'next/server';

const API_KEY = 'YOUR_API_KEY';
const BASE    = 'https://api.ip-block.com/v1';

async function apiReachable(): Promise<boolean> {
  try {
    const r = await fetch(`${BASE}/ping`, { signal: AbortSignal.timeout(500) });
    const { status } = await r.json();
    return status === 'ok';
  } catch { return false; }
}

export async function middleware(req: NextRequest) {
  if (!(await apiReachable())) return NextResponse.next(); // fail open

  const ip        = req.ip ?? req.headers.get('x-forwarded-for')?.split(',')[0] ?? '';
  const userAgent = req.headers.get('user-agent') ?? '';  // optional
  const referrer  = req.headers.get('referer')     ?? '';  // optional

  const r = await fetch(`${BASE}/check`, {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      api_key:    API_KEY,
      ip,
      site_id:    'ABCDEFGHIJKL',
      user_agent: userAgent,
      referrer,
    }),
  });
  const { action } = await r.json();
  if (action === 'block') {
    return NextResponse.redirect(new URL('https://www.ip-block.com/blocked.php'));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};