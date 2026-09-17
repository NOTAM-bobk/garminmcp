# garminmcp
Yes, you can fix these issues for free, but you'll need to make one fundamental architectural change: **stop calling Groq's API directly from the browser and instead proxy requests through a server-side route on Vercel.**

Here's how to fix each issue without spending money.

## Fix 1: Use Groq's Responses API (Not Chat Completions)

Your code uses the Chat Completions endpoint (`/chat/completions`), but remote MCP tools only work with Groq's **Responses API** (`/responses`) . The fix is straightforward: change the endpoint and request shape.

**Replace this in your client code:**

```javascript
const GROQ_API_URL = "https://api.groq.com/openai/v1/chat/completions";
```

**With this (on the server, see Fix 3):**

```javascript
const GROQ_API_URL = "https://api.groq.com/openai/v1/responses";
```

And change the payload structure from `messages` to `input`, and the tool definition format to match Groq's Responses API schema .

## Fix 2: Handle OAuth Properly (No Popup Workaround)

Your popup approach won't work because MissingMCP requires a proper OAuth 2.1 + PKCE flow . You cannot obtain a usable Groq-compatible Bearer token by simply opening a popup window.

**The realistic free path:** Build a minimal OAuth callback route on Vercel. The flow works like this :

1. Your app generates a PKCE `code_verifier` and `code_challenge` (SHA-256, base64url)
2. Redirect the user to MissingMCP's authorization endpoint with the challenge
3. MissingMCP redirects back to your `/api/auth/callback` route with an authorization code
4. Your **server-side** route exchanges the code + verifier for tokens (this is where you need CORS to be permitted by MissingMCP)
5. Store the access token server-side (session cookie or encrypted in Vercel KV/Postgres free tier)
6. Pass the token to Groq in the MCP tool definition's `headers` field

**If MissingMCP doesn't allow CORS from your origin**, the OAuth flow cannot be completed client-side. In that case, your only free option is to run your own MissingMCP instance (it's open source) or manually copy a token from a working client.

## Fix 3: Move Groq Calls to a Server Route

The Groq API blocks browser-origin requests via CORS . You must proxy through Vercel. Create `app/api/groq/route.ts`:

```typescript
export async function POST(req: Request) {
  const body = await req.json();

  const upstream = await fetch('https://api.groq.com/openai/v1/responses', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.GROQ_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });

  if (!upstream.ok || !upstream.body) {
    return new Response('upstream error', { status: upstream.status || 502 });
  }

  return new Response(upstream.body, {
    headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-store' },
  });
}
```

Then your frontend calls `/api/groq` instead of Groq directly .

## Fix 4: Secure Your API Key

Your current code hardcodes the Groq API key in the HTML (visible in the settings input). This is a serious security issue.

**Vercel free-tier fix:**
1. Add `GROQ_API_KEY` as an environment variable in Vercel dashboard (Settings → Environment Variables)
2. Never prefix it with `NEXT_PUBLIC_` — that would expose it to the browser 
3. Only reference it in server-side routes (`process.env.GROQ_API_KEY`)
4. Remove the key from client-side code entirely

## What About Vercel's 10-Second Timeout?

Vercel Hobby (free) has a 10-second serverless function timeout . For a chat with MCP tool calls, this might be tight. Options:

- **Groq is very fast** (1000+ tokens/second), so responses may complete within 10s
- If timeouts occur, you can use **Edge Functions** which have different (often longer) limits
- For a personal project, this is usually fine

## Summary: Your Free Fix Checklist

| Issue | Free Fix |
|---|---|
| Wrong Groq endpoint | Switch to `/responses` with `input` and `tools` format  |
| OAuth popup won't work | Build server-side PKCE flow; if CORS blocks, self-host MissingMCP |
| CORS blocks browser calls | Proxy through `/api/groq` route  |
| Hardcoded API key | Move to Vercel environment variables  |
| Vercel timeout | Use Edge Functions or keep requests short |

**Bottom line:** Yes, it's free — but you're essentially rebuilding the architecture. The UI you have is fine; the backend needs a complete rewrite to work with Groq's MCP support and MissingMCP's OAuth requirements.
