# Building Google & GitHub OAuth Login From Scratch (No Passport.js)

Most tutorials wire up social login with Passport.js and call it a day. I wanted to actually understand the OAuth 2.0 Authorization Code flow, so I hand-rolled it for my app instead — about 250 lines of code across a handful of files, no OAuth library dependency at all. Here's exactly how it works, end to end.

## The problem

My app already had email/password auth: register, login, JWT access tokens, rotating refresh tokens stored as httpOnly cookies. I wanted "Continue with Google" and "Continue with GitHub" buttons that plug into that *same* session system — not a separate, parallel auth mechanism.

That constraint shaped the design: OAuth isn't a different way to get logged in, it's a different way to *prove who you are* before I hand out the exact same session tokens my password login already issues.

## The flow, at a glance

```
┌─────────┐        ┌──────────┐        ┌──────────────┐        ┌─────────┐
│ Browser │        │ My API   │        │ Google/GitHub│        │ My API  │
└────┬────┘        └────┬─────┘        └──────┬───────┘        └────┬────┘
     │  1. click "Continue with Google"        │                     │
     │  full-page navigation to                │                     │
     │  GET /api/v1/auth/google                │                     │
     ├──────────────────►│                     │                     │
     │                    │ 2. generate CSRF    │                     │
     │                    │    state, set cookie│                     │
     │                    │    302 redirect      │                     │
     │◄───────────────────┤                     │                     │
     │  3. redirected to Google's consent screen│                     │
     ├───────────────────────────────────────────►│                   │
     │  4. user approves                          │                   │
     │◄────────────────────────────────────────────┤                  │
     │  5. redirected to                           │                  │
     │  GET /auth/google/callback?code=..&state=.. │                  │
     ├──────────────────────────────────────────────────────────────►│
     │                                              6. validate state │
     │                                              7. exchange code  │
     │                                                 for profile     │
     │                                              8. find/create user│
     │                                              9. issue tokens    │
     │◄──────────────────────────────────────────────────────────────┤
     │ 10. redirected to /oauth/callback?token=...                    │
     │     (+ refresh token set as httpOnly cookie)                    │
     │ 11. frontend fetches /auth/me with token, stores session         │
```

Ten steps, three of which are just the browser bouncing between my server and the provider's consent screen. Let's go through the code for each.

## Step 1 — the button is a link, not a fetch

This tripped me up initially. My instinct was to wire the button to an `onClick` that calls `axios.post(...)`. That's wrong for OAuth — the very first step has to be a **full top-level page navigation**, because the browser needs to actually leave my site and land on Google's consent screen, cookies and all. A `fetch`/XHR request can't do that.

So the button is just a link:

```tsx
<a href={getOAuthUrl("google")} className="...">
  <GoogleLogo /> Google
</a>
```

```ts
export function getOAuthUrl(provider: "google" | "github"): string {
  const base = import.meta.env.VITE_API_URL ?? "/api/v1";
  return `${base}/auth/${provider}`;
}
```

## Step 2 — the server kicks off the handshake with a CSRF token

`GET /api/v1/auth/google` doesn't return JSON — it's a redirect. Before sending the browser off to Google, it generates a random `state` value, stashes it in a short-lived httpOnly cookie, and builds Google's authorize URL:

```ts
export async function oauthStart(req: Request, res: Response) {
  const state = crypto.randomBytes(16).toString("hex");
  res.cookie(OAUTH_STATE_COOKIE, state, {
    httpOnly: true,
    sameSite: "lax",
    maxAge: 10 * 60 * 1000,
    path: "/api/v1/auth",
  });
  res.redirect(oauthService.getAuthorizeUrl(provider, state));
}
```

The `state` parameter is the standard OAuth CSRF defense: without it, an attacker could craft a malicious link that completes an OAuth flow *for their own account* and trick a victim into clicking it, potentially linking the attacker's identity to the victim's session. By round-tripping a value only my server could have generated — and checking it against the cookie on the way back — a forged callback request gets rejected.

## Step 3–4 — the part I don't control

The browser lands on Google's actual consent screen. The user approves. Google redirects back to the **redirect URI** I registered when creating the OAuth app — `http://localhost:4000/api/v1/auth/google/callback` — with a short-lived authorization `code` and the `state` I sent, appended as query params.

## Step 5 — validating the callback

```ts
export async function oauthCallback(req: Request, res: Response) {
  const expectedState = req.cookies?.[OAUTH_STATE_COOKIE];
  res.clearCookie(OAUTH_STATE_COOKIE, { path: "/api/v1/auth" });

  const { code, state } = req.query;
  if (!code || !state || state !== expectedState) {
    res.redirect(`${FRONTEND_URL}/oauth/callback?error=invalid_state`);
    return;
  }
  // ...
}
```

If the state doesn't match — or is missing entirely — the flow is aborted and the user is bounced back to the frontend with an error, never treated as authenticated.

One important detail: **this whole handler wraps its logic in try/catch and always ends in a redirect**, never a raw JSON error. It's a browser-facing endpoint that Google navigates to directly — returning `{"error": "..."}` as a JSON body here would just show the user a wall of text instead of a proper error page.

## Step 6 — trading the code for a profile

The `code` is single-use and only proves the user approved *something* — it isn't the profile itself. The server exchanges it for an access token at Google's token endpoint, then uses that token to fetch the actual profile:

```ts
async function exchangeCode(provider, code): Promise<string> {
  const res = await fetch(config.tokenUrl, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      client_id: config.clientId,
      client_secret: config.clientSecret, // never exposed to the browser
      code,
      redirect_uri: getRedirectUri(provider),
      grant_type: "authorization_code",
    }),
  });
  const data = await res.json();
  return data.access_token;
}
```

This is *why* the Authorization Code flow needs a backend at all: the `client_secret` proves to Google that it's really my server asking, not just anyone who intercepted the code. It can never be shipped to the browser.

Google and GitHub return different shapes of profile data, so each gets normalized into the same internal shape:

```ts
interface NormalizedProfile {
  providerUserId: string;
  email: string;
  emailVerified: boolean;
  name: string;
  suggestedUsername: string;
}
```

Google's userinfo endpoint hands back `email` and `email_verified` directly. GitHub is trickier — its `/user` endpoint doesn't reliably include an email at all (users can hide it), so a second call to `/user/emails` is needed to find the verified primary address:

```ts
const emailsRes = await fetch("https://api.github.com/user/emails", { headers });
const emails = await emailsRes.json();
const primary = emails.find((e) => e.primary) ?? emails.find((e) => e.verified) ?? emails[0];
```

## Step 7 — find-or-create, with account linking

This is the part with the most interesting judgment calls. Three cases:

1. **This exact provider account has signed in before** → look up by `(provider, providerUserId)` and return the linked user.
2. **First time with this provider, but the (verified) email matches an existing account** → link this provider to that account instead of creating a duplicate. If someone registered with a password using `jane@gmail.com` and later clicks "Continue with Google" using that same Google account, they land in the *same* account, not a new one.
3. **Genuinely new** → create a fresh user with `passwordHash: null` (they can only sign in via OAuth, no password exists) and link the provider.

```ts
async function findOrCreateUser(provider, profile) {
  const existingLink = await repo.findOAuthAccount(provider, profile.providerUserId);
  if (existingLink) return repo.findUserById(existingLink.userId);

  if (profile.emailVerified) {
    const existingUser = await repo.findUserByEmail(profile.email);
    if (existingUser) {
      await repo.createOAuthAccount(existingUser.id, provider, profile.providerUserId);
      return existingUser;
    }
  }

  const username = await generateUniqueUsername(profile.suggestedUsername);
  const newUser = await repo.createOAuthUser({ fullName: profile.name, email: profile.email, username });
  await repo.createOAuthAccount(newUser.id, provider, profile.providerUserId);
  return newUser;
}
```

The `emailVerified` check matters a lot here. Linking by email is only safe because Google/GitHub already confirmed the person controls that inbox — I'm trusting *their* verification, not just trusting whatever string shows up in an API response. Skipping that check would open an account-takeover hole: anyone could self-report an unverified email matching someone else's account and get linked into it.

Database-wise, this needed a small schema change — `passwordHash` had to become nullable (OAuth-only users have none), plus a new join table:

```prisma
model OAuthAccount {
  id             String        @id @default(uuid())
  userId         String
  user           User          @relation(fields: [userId], references: [id], onDelete: Cascade)
  provider       OAuthProvider
  providerUserId String
  @@unique([provider, providerUserId])
}
```

## Step 8 — issuing the exact same session as password login

This was the whole point of building it this way. Once I have a `User` row — however it was found or created — the code path converges with regular login:

```ts
const tokens = await issueTokens(user.id, user.email); // same function password login uses
```

`issueTokens` signs a short-lived JWT access token and generates a long-lived opaque refresh token (hashed and stored in the DB, so it can be revoked). OAuth users and password users end up with structurally identical sessions — nothing downstream in the app needs to know or care how someone logged in.

## Step 9–10 — getting the token back to a single-page app

Here's a subtlety specific to SPAs: the callback is a server-side redirect, but the access token needs to end up in the *browser's* JavaScript state (a Zustand store, in my case), not just a cookie the server can read.

The refresh token goes out as an httpOnly cookie exactly like password login (invisible to JS, which is the point — it's what makes token theft via XSS much harder). But the access token gets passed via a redirect to a dedicated frontend route:

```ts
res.redirect(`${FRONTEND_URL}/oauth/callback?token=${accessToken}`);
```

That route reads the token out of the URL, immediately uses it to call `GET /auth/me` to fetch the full profile, stores everything in the app's auth store, and — importantly — replaces the URL so the token doesn't linger in browser history:

```tsx
useEffect(() => {
  const token = params.get("token");
  if (!token) return;
  setAccessToken(token);
  authService.getMe()
    .then((user) => {
      setSession(user, token);
      navigate("/dashboard", { replace: true });
    })
    .catch((err) => setError(...));
}, []);
```

## Why not just use Passport.js?

Passport abstracts almost all of this behind strategy plugins, which is genuinely convenient — but it also means trusting a library to get the state validation, token exchange, and profile normalization right, without seeing any of it. Hand-rolling it took maybe an extra hour, and the entire flow now fits in about four small, readable files with zero magic. For a project already doing its own JWT issuance instead of Passport's session middleware, it also just fit the existing shape of the code better than bolting on a whole strategy ecosystem for two providers.

## Summary

OAuth "Login with X" buttons look like a single click, but underneath it's a five-way handshake: a CSRF-protected redirect out, a provider-side approval I never see, a redirect back with a one-time code, a server-to-server exchange for a real profile (using a secret that never touches the browser), and finally converging into the same session-issuance path as every other login method. The two-line button hides all of that — which is exactly the point.
