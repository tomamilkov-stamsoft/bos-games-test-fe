# Security Audit Report — bos-games-test-fe

**Date:** 2026-02-28
**Auditor:** Senior Security Auditor (Claude Sonnet 4.6)
**Codebase:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe`
**Framework:** React 18 + Vite 5 + TypeScript + Tailwind CSS
**Audit Scope:** Full frontend static analysis, dependency review, authentication patterns, API security, and information exposure

---

## Executive Summary

The `bos-games-test-fe` application is a React/Vite single-page application for a gaming matchmaking platform (CS2 / Counter-Strike 2). The codebase appears to be a **test/development dashboard** intended to facilitate backend API testing and demonstrate features such as user registration, login, friend management, team creation, party matchmaking, push notifications via Firebase, and live match tracking.

The audit identified **21 distinct findings** ranging from Critical to Informational severity. The most serious issues involve hardcoded Firebase credentials committed to version control, tokens passed as URL query parameters in OAuth flows, a hardcoded server IP address in production UI code, pervasive verbose `console.log` calls leaking sensitive data in production builds, absent HTTP security headers, no input sanitisation, and multiple vulnerable production dependencies.

The codebase demonstrates characteristics of a prototype or internal testing tool (hardcoded test passwords, auto-registration flows, raw token inputs) that must not reach production users without significant security remediation.

---

## Findings Summary Table

| # | Severity | Category | Title | File(s) |
|---|----------|----------|-------|---------|
| F-01 | CRITICAL | Sensitive Data Exposure | Firebase API key and project credentials hardcoded in source code | `src/config/firebase.ts`, `public/firebase-messaging-sw.js` |
| F-02 | CRITICAL | Authentication | OAuth tokens transmitted via URL query parameters | `src/pages/Login.tsx`, `src/pages/SocialAuth.tsx`, `src/api/auth.ts` |
| F-03 | HIGH | Authentication | Hardcoded default password in registration and login flows | `src/pages/Register.tsx`, `src/pages/Login.tsx` |
| F-04 | HIGH | Sensitive Data Exposure | JWT token partially rendered in the UI | `src/pages/Dashboard.tsx` |
| F-05 | HIGH | Sensitive Data Exposure | 343+ verbose `console.log` statements expose tokens, user data, and internal state | Multiple files |
| F-06 | HIGH | Dependency Vulnerability | React Router DOM 6.x vulnerable to XSS via open redirect | `package.json` |
| F-07 | HIGH | Dependency Vulnerability | Axios 1.x vulnerable to DoS via missing data size check | `package.json` |
| F-08 | HIGH | Dependency Vulnerability | Rollup 4.x arbitrary file write via path traversal | `package.json` |
| F-09 | HIGH | Dependency Vulnerability | glob and minimatch ReDoS / command injection vulnerabilities | `package.json` |
| F-10 | HIGH | Security Misconfiguration | No Content Security Policy (CSP) or security headers configured | `index.html`, `vite.config.ts` |
| F-11 | HIGH | Hardcoded Credential | Production game server IP hardcoded in UI navigation code | `src/App.tsx` |
| F-12 | MEDIUM | Dependency Vulnerability | esbuild development server CORS bypass | `package.json` |
| F-13 | MEDIUM | Dependency Vulnerability | Vite middleware path confusion vulnerability | `package.json` |
| F-14 | MEDIUM | Authentication | Token stored in `sessionStorage` accessible to any same-origin JavaScript | All authenticated pages |
| F-15 | MEDIUM | Input Validation | No client-side input validation or sanitisation on any user-supplied fields | `src/pages/Register.tsx`, `src/pages/Teams.tsx`, etc. |
| F-16 | MEDIUM | Open Redirect / Protocol Injection | `window.location.href` set to attacker-controlled server IP/port from push notifications | `src/App.tsx`, `src/components/ServerConnectionModal.tsx` |
| F-17 | MEDIUM | Information Exposure | Full notification payloads including match and player data rendered as raw JSON | `src/pages/Notifications.tsx` |
| F-18 | MEDIUM | API Security | Unauthenticated `getUserById` endpoint called without a token | `src/api/user.ts` |
| F-19 | MEDIUM | Authentication | No centralised Axios interceptor; token handling is manual and inconsistent | All `src/api/*.ts` files |
| F-20 | LOW | Security Misconfiguration | `http://localhost:3000` fallback hard-coded in multiple API modules | `src/api/hardware.ts`, `src/api/live-matches.ts`, `src/api/match-statistics.ts`, `src/api/ratings.ts` |
| F-21 | INFO | Design / Dev Artefact | Test utilities, debug endpoints, and simulation code shipped in production bundle | `src/utils/background-message-test.ts`, `src/components/DeviceIdDisplay.tsx` |

---

## Detailed Findings

---

### F-01 — CRITICAL: Firebase API Key and Project Credentials Hardcoded in Source Code

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/config/firebase.ts` (lines 3–9)
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/public/firebase-messaging-sw.js` (lines 54–62)

**Description:**
The Firebase project configuration, including the API key (`AIzaSyDUsEFOdlNO8muiTUx0Em65KY59Da_V_3A`), auth domain, project ID, messaging sender ID, app ID, measurement ID, and the VAPID public key for Web Push, are all committed directly in source code. Both the main app module and the service worker file contain identical copies of these values.

```typescript
// src/config/firebase.ts
export const firebaseConfig = {
  apiKey: "AIzaSyDUsEFOdlNO8muiTUx0Em65KY59Da_V_3A",
  authDomain: "bos-games-145f0.firebaseapp.com",
  projectId: "bos-games-145f0",
  storageBucket: "bos-games-145f0.firebasestorage.app",
  messagingSenderId: "303868662210",
  appId: "1:303868662210:web:8d737920e41afe8540065b",
  measurementId: "G-5NHMN87Z34",
};

export const vapidKey =
  "BKIlF0YS9FZix8JOnhkmfXQ2v5uItzP1wI_tNcQ14fr6yt8MX3da-3n7RtXiL_wzAIpSwvwmnL8qoy27cqLmPac";
```

**Attack Scenario:**
Any person with access to the public repository (or the built JS bundle) can extract these credentials. While Firebase Web API keys are technically "public" in Firebase's design model, they are intended to be protected by Firebase Security Rules, not to be considered secret. However, with the project ID and API key, an attacker can:
- Enumerate Firebase Firestore collections, Realtime Database, and Storage if rules are misconfigured.
- Register push token endpoints directly against the Firebase project.
- Abuse Firebase Authentication endpoints (password reset, anonymous sign-in, etc.) at quota-exhaustion scale.
- The VAPID key exposure allows an attacker to subscribe their own browser to push notifications on behalf of arbitrary users if the backend does not verify ownership.

**Remediation:**
1. Move Firebase client configuration values to Vite environment variables (`VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_PROJECT_ID`, etc.) read via `import.meta.env`.
2. Ensure `.env` is listed in `.gitignore` (it is currently listed, but the values are bypassed by being hardcoded).
3. For the service worker (`firebase-messaging-sw.js`), which cannot use ES module imports at bundle time, inject the config dynamically from a secure endpoint or use a server-side template at build time.
4. Audit Firebase Security Rules for the `bos-games-145f0` project and restrict access to authenticated users only.
5. Consider rotating the API key and regenerating the VAPID key pair if the repository has been public.

---

### F-02 — CRITICAL: OAuth Tokens Transmitted via URL Query Parameters

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Login.tsx` (lines 43–55)
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/SocialAuth.tsx` (lines 12–26)
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/api/auth.ts` (lines 107–123)

**Description:**
The Steam OAuth callback flow passes both `accessToken` and `refreshToken` as URL query parameters (e.g., `/login?accessToken=eyJ...&refreshToken=eyJ...`). The application reads these from `useSearchParams()`.

```typescript
// src/pages/Login.tsx:43-55
const accessToken = searchParams.get("accessToken");
const refreshToken = searchParams.get("refreshToken");
if (accessToken) {
  handleSocialAuthSuccess(accessToken, refreshToken || undefined);
}
```

**Attack Scenario:**
Tokens in URL query strings are logged in:
- Browser history (accessible to other users of shared machines, or the operating system).
- Web server access logs (if the app is behind a reverse proxy).
- Referrer headers when navigating to third-party resources (analytics scripts, CDN assets).
- Browser extensions that observe navigation events.

An attacker with access to any of these channels obtains a valid session token with no further effort.

**Remediation:**
1. Use the `fragment` (hash) portion of the URL instead of the query string for token relay — fragments are not sent to servers or logged. This is the OAuth 2.0 implicit flow convention.
2. Preferably implement the PKCE (Proof Key for Code Exchange) OAuth flow, where the backend never puts tokens in the redirect URL; instead, a short-lived authorisation code is exchanged over a back-channel HTTPS request from the frontend.
3. If the current redirect pattern must be preserved temporarily, ensure the application immediately navigates away using `history.replaceState` to remove the tokens from the URL after reading them, and before rendering any content that loads third-party scripts.

---

### F-03 — HIGH: Hardcoded Default Password in Registration and Login Flows

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Register.tsx` (line 19, 48)
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Login.tsx` (line 18)

**Description:**
The password `Test123!` is hardcoded as the default value in both the registration and login components. It is rendered as a read-only field with the label "(Auto-filled for demo)". Additionally, the email verification code `111111` is hardcoded.

```typescript
// src/pages/Register.tsx:19,31
const [password] = useState("Test123!");
const devCode = "111111";

// src/pages/Login.tsx:18
const [password] = useState("Test123!");
```

**Attack Scenario:**
Any user can view the page source or browser DevTools to recover the hardcoded credential. If this application were deployed against a production backend, an attacker could register accounts programmatically with the known password. The hardcoded OTP code (`111111`) bypasses email verification entirely for any account registered through this UI.

**Remediation:**
1. Remove hardcoded passwords and verification codes entirely from the codebase.
2. If the intent is a developer test harness, gate this functionality behind a build-time flag (`import.meta.env.DEV`) so it is stripped in production builds.
3. Require users to provide their own credentials.

---

### F-04 — HIGH: JWT Token Partially Rendered in the UI

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Dashboard.tsx` (lines 121–125)

**Description:**
The first 20 characters of the JWT access token are rendered directly in the Dashboard page HTML:

```tsx
// src/pages/Dashboard.tsx:121-125
<div className="text-xs text-gray-500">
  <p>
    <strong>JWT Token:</strong> {token.substring(0, 20)}...
  </p>
</div>
```

**Attack Scenario:**
While only a prefix of the token is shown, this practice normalises the display of authentication material in the UI. A CSS modification (via a browser extension or XSS), a screenshot, or screen recording will capture the token fragment. Moreover, the first 20 characters include the base64url-encoded header and part of the payload, which may reveal the token type and algorithm. Any future increase to the substring length (even accidentally) would expose the full token.

**Remediation:**
Remove this debug display entirely. If a developer needs to inspect the active token, they should use browser DevTools (Application > Session Storage).

---

### F-05 — HIGH: 343 Verbose `console.log` Statements Expose Sensitive Data

**Files:** 23 source files including `src/App.tsx` (143 occurrences), `src/utils/push-notifications.ts` (58 occurrences), `src/pages/Login.tsx`, `src/api/hardware.ts`, and others.

**Description:**
The codebase contains 343 `console.log` / `console.error` / `console.warn` calls, many of which output sensitive runtime data including:

- JWT access tokens: `console.log("token", token)` in `src/pages/Dashboard.tsx:14`, `src/App.tsx:87`.
- User profile objects (email, nickname, country): `console.log("user", user)` in `src/pages/Login.tsx:87`, `src/pages/SocialAuth.tsx:37`.
- Firebase push tokens and VAPID keys: `console.log("Push token registered successfully:", { token, deviceId, platform })` in `src/utils/push-notifications.ts:393`.
- Full Firebase messaging payload data including match IDs and server IPs: `src/utils/push-notifications.ts:79–84`.
- Hardware benchmark scores and system specifications: `console.log("Submitting hardware specs:", hardwareData)` in `src/api/hardware.ts:57`.
- Steam OAuth callback tokens: `console.log("Steam user", user)` in `src/pages/Login.tsx:67`.
- Map banning session data including all player and leader IDs: `src/components/MapBanningModal.tsx`.

**Attack Scenario:**
In a production deployment, any XSS vulnerability (even a minor one), a malicious browser extension, or remote debugging access would allow an attacker to harvest these logged values. Browser consoles can also be read by the operating system in enterprise environments with endpoint monitoring.

**Remediation:**
1. Remove all `console.log` statements from production code. Use a dedicated logging library (e.g., `loglevel`, `pino`, or a no-op stub) that can be configured to suppress output in production builds.
2. For Vite, add the `drop: ['console']` option to the `build.rollupOptions` configuration, or use the `vite-plugin-remove-console` plugin.
3. Never log authentication tokens, user PII, or cryptographic material at any log level in a production-facing build.

---

### F-06 — HIGH: React Router DOM Vulnerable to XSS via Open Redirect (CVE-2025 / GHSA-2w69-qvjg-hvjx)

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/package.json` (line 16)

**Description:**
The installed version of `react-router-dom` (`^6.23.0`) falls within the vulnerable range `6.0.0 – 6.30.2`. The advisory `GHSA-2w69-qvjg-hvjx` describes a stored XSS vector via open redirects: when a route redirect target is constructed from unsanitised user input, it is possible to inject `javascript:` URIs that execute in the browser context.

**Remediation:**
Upgrade `react-router-dom` to version `≥ 6.30.3` (or the latest available stable release). Run `npm install react-router-dom@latest` and verify test coverage is maintained.

---

### F-07 — HIGH: Axios Vulnerable to DoS via Missing Data Size Check (GHSA-4hjh-wcwx-xvwj)

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/package.json` (line 12)

**Description:**
The installed version of `axios` (`^1.6.7`) falls within the vulnerable range `1.0.0 – 1.13.4`. An attacker who controls a backend endpoint (e.g., through a compromised API server or a man-in-the-middle position) can return an arbitrarily large response body, causing the axios promise to buffer the entire response in memory, leading to out-of-memory conditions (DoS).

**Remediation:**
Upgrade axios to version `≥ 1.13.5`. Run `npm install axios@latest`.

---

### F-08 — HIGH: Rollup 4 Arbitrary File Write via Path Traversal (GHSA-mw96-cpmx-2vgc)

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/package.json` (Vite build dependency)

**Description:**
Rollup versions `4.0.0 – 4.58.0` (used internally by Vite 5.x) are vulnerable to arbitrary file write via a path traversal attack when processing specially crafted input files during the build process. This primarily affects CI/CD pipelines and developer workstations running `vite build` against untrusted inputs.

**Remediation:**
Upgrade Vite to a version that depends on Rollup `≥ 4.58.1`. Run `npm install vite@latest`.

---

### F-09 — HIGH: glob and minimatch ReDoS / Command Injection

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/package.json` (transitive build dependencies)

**Description:**
- `glob` versions `10.2.0 – 10.4.5` are vulnerable to command injection via the `-c/--cmd` CLI argument executing matched filenames with `shell: true` (GHSA-5j98-mcp5-4vw2).
- `minimatch` versions `9.0.0 – 9.0.6` are vulnerable to ReDoS (regular expression denial of service) via repeated wildcards in patterns (GHSA-3ppc-4f35-3m26).

These are build-time dependencies, so direct exploitation by end-users is not possible. However, they present a risk to developers and CI/CD pipelines.

**Remediation:**
Upgrade affected transitive dependencies by running `npm audit fix` or upgrading the parent packages (Vite, Rollup) that pull these in.

---

### F-10 — HIGH: No Content Security Policy or HTTP Security Headers

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/index.html`
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/vite.config.ts`

**Description:**
The `index.html` entry point contains no `Content-Security-Policy` meta tag, no `X-Frame-Options`, no `X-Content-Type-Options`, no `Referrer-Policy`, and no `Permissions-Policy` header. The `vite.config.ts` does not configure any custom response headers for the dev server or preview server.

```html
<!-- index.html — no security meta tags present -->
<head>
  <meta charset="UTF-8" />
  <link rel="icon" type="image/svg+xml" href="/vite.svg" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Bos Games Test Dashboard</title>
</head>
```

**Attack Scenario:**
Without a CSP:
- Any XSS payload can execute arbitrary JavaScript (load external scripts, exfiltrate tokens, redirect users).
- Inline scripts injected by malicious browser extensions are not blocked.

Without `X-Frame-Options` / `frame-ancestors`, the application can be embedded in an `<iframe>` on an attacker's site (clickjacking).

Without `Referrer-Policy: no-referrer`, the full URL (including any tokens in query strings — see F-02) is sent to third-party origins via the Referer header.

**Remediation:**
1. Add a `Content-Security-Policy` meta tag restricting `script-src` to `'self'` and known CDN origins (e.g., `www.gstatic.com` for Firebase).
2. Configure `X-Frame-Options: DENY` and `X-Content-Type-Options: nosniff` at the web server/CDN layer.
3. Add `<meta name="referrer" content="no-referrer">` to `index.html`.
4. If using a hosting platform (Nginx, Vercel, Netlify), add the security headers in the platform's response header configuration.

---

### F-11 — HIGH: Production Game Server IP Address Hardcoded in UI Navigation Code

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/App.tsx` (line 1366)

**Description:**
A concrete game server IP address and port are hardcoded directly in the navigation bar code that is rendered for all authenticated users:

```typescript
// src/App.tsx:1366
const steamUrl = `steam://run/730//+connect 156.146.52.210:26952 -novid`;
```

**Attack Scenario:**
- The IP address `156.146.52.210` and port `26952` are exposed in the public source and in the compiled JS bundle. Anyone can attempt to connect to the server, probe for vulnerabilities, or launch DDoS attacks targeting that endpoint.
- This hardcoded value bypasses the intended secure flow where server connection details should only be delivered to matched players via authenticated push notifications.

**Remediation:**
1. Remove this hardcoded URL entirely. Server connection details must only be delivered by the backend to authenticated, matched players via the push notification / match-started flow.
2. This appears to be a debugging shortcut that was never removed. Add a lint rule or pre-commit hook to detect raw IP addresses in source code.

---

### F-12 — MEDIUM: esbuild Development Server CORS Bypass (GHSA-67mh-4wv8-2f99)

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/package.json` (devDependency via Vite)

**Description:**
esbuild versions `≤ 0.24.2` allow any website to send cross-origin requests to the esbuild development server and read the full response. This means that if a developer has the Vite dev server running and visits a malicious website, that website can read the application source code via the dev server.

**Remediation:**
Upgrade Vite to a version that depends on esbuild `≥ 0.25.0`. This is a development-only risk but can leak source code and internal API structure to attackers who can lure developers to malicious pages.

---

### F-13 — MEDIUM: Vite Middleware Path Confusion (GHSA-g4jq-h2w9-997c)

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/package.json` (devDependency)

**Description:**
Vite versions `≤ 6.1.6` have a middleware vulnerability where files in the `public` directory can be served when the request path starts with the same string as a public directory file, bypassing intended routing. An attacker who can influence request paths may be able to access unintended static files.

**Remediation:**
Upgrade Vite to version `≥ 6.1.7` or latest stable.

---

### F-14 — MEDIUM: JWT Token Stored in `sessionStorage` — Accessible to Same-Origin JavaScript

**Files:** All authenticated pages and `src/api/auth.ts` (lines 113, 120, 132–133)

**Description:**
JWT access tokens (and in some code paths refresh tokens) are stored in `sessionStorage` under the key `"token"`:

```typescript
// src/api/auth.ts:113,115,120
sessionStorage.setItem("token", accessToken);
sessionStorage.setItem("refreshToken", refreshToken);
sessionStorage.setItem("user", JSON.stringify(user));
```

`sessionStorage` is accessible to any JavaScript running in the same origin. An XSS vulnerability — even a reflected, non-persistent one — is sufficient to extract the token via `sessionStorage.getItem("token")`.

**Remediation:**
1. For a higher security posture, use `HttpOnly`, `Secure`, `SameSite=Strict` cookies managed by the server for session tokens. These are not accessible to JavaScript.
2. If client-side token access is required (e.g., for attaching to Axios headers), consider a short-lived in-memory token store (a module-level variable) and use a refresh token in an `HttpOnly` cookie for session persistence.
3. Minimise the data stored: avoid storing the full user profile JSON in `sessionStorage` as it contains PII (email, name, country).

---

### F-15 — MEDIUM: No Client-Side Input Validation or Sanitisation

**Files:** `src/pages/Register.tsx`, `src/pages/Teams.tsx`, `src/pages/Friends.tsx`, `src/pages/Parties.tsx`

**Description:**
User-supplied inputs (team names, search queries, nickname fields) are passed directly to API calls with no client-side validation. For example:

```typescript
// src/pages/Teams.tsx:102-120
async function handleCreateTeam() {
  if (!teamName.trim() || !gameModeId) {
    setStatus("Please enter team name and select a game mode");
    return;
  }
  const team = await createTeam(teamName, gameModeId, inviteeIds, token);
```

The only check is that `teamName` is non-empty. There is no length limit, no character allow-list, and no encoding before display.

**Attack Scenario:**
While React's JSX rendering uses auto-escaping and does not execute injected HTML, a crafted team name containing special characters could affect backend processing if the backend does not also validate and sanitise. Client-side validation is a usability and first-line defence concern.

**Remediation:**
1. Add input length limits on all text fields (e.g., team name `max=50`).
2. Consider adding a character allow-list for usernames and team names (letters, numbers, hyphens, underscores).
3. Rely on the backend as the authoritative validation layer, but provide client-side feedback to reduce unnecessary API calls.

---

### F-16 — MEDIUM: Open Redirect / Protocol Injection via Push Notification Server IP

**Files:** `src/App.tsx` (lines 247, 260, 540–551, 870–883), `src/components/ServerConnectionModal.tsx` (line 73)

**Description:**
The application constructs a `steam://` protocol URL directly from `serverIp` and `serverPort` values that originate from push notification payloads and are never validated or sanitised:

```typescript
// src/components/ServerConnectionModal.tsx:60
const steamUrl = `steam://run/730//+connect ${serverIp}:${serverPort} -novid`;
window.location.href = steamUrl;
```

The `serverIp` and `serverPort` values flow from `payload.data.serverIp` in push notification handlers (`src/utils/push-notifications.ts`, `public/firebase-messaging-sw.js`) through `CustomEvent` dispatches to `App.tsx` state and then into the modal.

**Attack Scenario:**
If an attacker can send a crafted FCM push notification to a target user (e.g., by exploiting the Firebase credential exposure in F-01, or by compromising the backend), they can set `serverIp` to an arbitrary value. Depending on how the operating system handles `steam://` URIs, a crafted IP string could:
- Redirect the user to a malicious server that spoofs a legitimate CS2 game server.
- In certain Steam client versions, inject additional command-line arguments via special characters in the IP field.

Furthermore, there is no validation that `serverIp` is actually a valid IP address or that `serverPort` is a numeric value within the valid port range.

**Remediation:**
1. Validate `serverIp` against an IPv4 regex pattern and validate `serverPort` is a number between 1 and 65535 before constructing any URL.
2. Consider allowlisting known server IP ranges managed by the platform.
3. Resolve F-01 to prevent unauthorised FCM message injection.

---

### F-17 — MEDIUM: Full Notification Payload Data Rendered as Raw JSON in UI

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Notifications.tsx` (lines 188–197)

**Description:**
The Notifications page renders the raw `data` object of every notification using `JSON.stringify` inside a `<pre>` block, visible to the user on demand:

```tsx
// src/pages/Notifications.tsx:188-197
{notification.data && (
  <div className="mt-2 text-xs text-gray-500">
    <details>
      <summary className="cursor-pointer hover:text-gray-700">
        View Details
      </summary>
      <pre className="mt-2 bg-gray-100 p-2 rounded text-xs overflow-auto">
        {JSON.stringify(notification.data, null, 2)}
      </pre>
    </details>
  </div>
)}
```

**Attack Scenario:**
If backend notifications contain sensitive internal data (database IDs, internal match state, server identifiers, other users' metadata), this exposes it directly in the browser. While React's JSX rendering prevents HTML injection, the raw data surface area is unnecessarily broad.

**Remediation:**
Only expose specific, curated fields from notification data objects rather than dumping the entire payload. Remove the "View Details" raw JSON expander from the production UI.

---

### F-18 — MEDIUM: Unauthenticated `getUserById` Endpoint Called Without Token

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/api/user.ts` (lines 21–24)

**Description:**
The `getUserById` function makes an authenticated resource request without attaching any authorisation header:

```typescript
// src/api/user.ts:21-24
export async function getUserById(userId: string) {
  const resp = await axios.get(`${API_BASE_URL}/users/${userId}`);
  return resp.data;
}
```

**Attack Scenario:**
If this function is ever called in the application (it currently appears unused but is exported), unauthenticated callers can enumerate user profiles by cycling through user IDs. This constitutes an Insecure Direct Object Reference (IDOR) risk if the backend does not independently enforce authentication.

**Remediation:**
Add an `Authorization: Bearer ${token}` header to this request, consistent with all other user API calls. If the endpoint is intentionally public, document this explicitly.

---

### F-19 — MEDIUM: No Centralised Axios Interceptor; Token Handling is Manual and Inconsistent

**Files:** All files in `src/api/`

**Description:**
Every API function independently reads the token from `sessionStorage` or receives it as a function argument and manually attaches it to the `Authorization` header. There is no centralised Axios instance with a request interceptor. This pattern has several security implications:

1. Any function that forgets to attach the token will silently make an unauthenticated request.
2. Token refresh logic (the `refreshToken` function exists in `src/api/auth.ts` but is never called from an interceptor) is unused. If the access token expires, the user gets a 401 and is not automatically refreshed.
3. The `hardware.ts` module reads the token internally from `sessionStorage` rather than accepting it as a parameter, creating an inconsistent contract.

**Remediation:**
1. Create a single `axios` instance with a request interceptor that attaches the `Authorization` header from the in-memory token store.
2. Add a response interceptor that handles 401 responses by attempting token refresh and retrying the request once.
3. Remove all manual token attachment from individual API functions.

---

### F-20 — LOW: `http://localhost:3000` Hardcoded as API Fallback

**Files:**
- `src/api/hardware.ts` (line 4): `import.meta.env.VITE_API_URL || "http://localhost:3000"`
- `src/api/live-matches.ts` (line 60): `import.meta.env.VITE_API_URL || "http://localhost:3000"`
- `src/api/match-statistics.ts` (line 127): `import.meta.env.VITE_API_URL || "http://localhost:3000"`
- `src/api/ratings.ts` (line 2): `import.meta.env.VITE_API_URL || "http://localhost:3000/api"`

**Description:**
If `VITE_API_URL` is not set in the environment (e.g., in a misconfigured deployment), these modules will silently fall back to `http://localhost:3000`, which means API calls will fail silently or — if an attacker controls a service on that port on the host — be redirected to a local attacker-controlled endpoint.

**Remediation:**
1. Remove the `|| "http://localhost:3000"` fallback values. In production, an undefined `VITE_API_URL` should cause an explicit build-time error, not a silent fallback.
2. Consider adding a build-time assertion: `if (!import.meta.env.VITE_API_URL) throw new Error("VITE_API_URL is required")`.
3. Note that this inconsistency also explains why some API modules work without the env var and others do not.

---

### F-21 — INFO: Test Utilities and Debug Code Shipped in Production Bundle

**Files:**
- `src/utils/background-message-test.ts` — `BackgroundMessageTest` class with methods to simulate background push notifications and inject arbitrary match events into the application.
- `src/components/DeviceIdDisplay.tsx` — A floating UI widget (rendered in `App.tsx` for all authenticated users) that displays the device ID, detected platform, and partial user-agent string.
- `src/utils/random.ts` — Generates random test emails, nicknames, names, and countries for automated registration.

**Description:**
The `BackgroundMessageTest` class provides a public API (`simulateBackgroundMatchNotification`, `testForegroundMessage`, `testMatchStartedNotification`) that can be called from the browser console to inject arbitrary push notification payloads into the application's event system. This includes triggering match acceptance modals with attacker-specified match IDs and server connection modals with attacker-specified IP addresses.

The `DeviceIdDisplay` component exposes device fingerprint information in a floating widget visible in the production UI.

**Attack Scenario:**
While these utilities do not introduce remote code execution on their own, they significantly expand the attack surface in a live deployment. An attacker exploiting any XSS vulnerability can call `BackgroundMessageTest.simulateBackgroundMatchNotification("victim-match-id")` to trigger UI actions on behalf of the victim. The device information display also leaks browser fingerprinting data.

**Remediation:**
1. Remove `BackgroundMessageTest` from the production bundle entirely, or guard it with `if (import.meta.env.DEV)`.
2. Remove the `DeviceIdDisplay` component from the authenticated UI, or restrict it to development environments.
3. Remove the `randomEmail`, `randomNickname`, `randomName`, and `randomCountry` utilities from the production bundle if they are only used for test registration.

---

## Security Misconfiguration Summary

| Configuration | Current State | Recommended State |
|---|---|---|
| Content Security Policy | Not set | Strict CSP with `script-src 'self'` |
| X-Frame-Options | Not set | `DENY` |
| X-Content-Type-Options | Not set | `nosniff` |
| Referrer-Policy | Not set | `no-referrer` |
| Strict-Transport-Security | Not configured (deployment-level) | `max-age=63072000; includeSubDomains` |
| Permissions-Policy | Not set | Restrict camera, microphone, geolocation |
| CORS | Handled by backend (not audited) | Backend should restrict to known origins |
| Token storage | `sessionStorage` (JS-accessible) | `HttpOnly` cookie |
| Console logging | 343 calls, many with secrets | Zero in production |

---

## Dependency Vulnerability Summary

| Package | Installed Range | Vulnerability | Severity | Fix Version |
|---|---|---|---|---|
| `react-router-dom` | `^6.23.0` | XSS via open redirect (GHSA-2w69-qvjg-hvjx) | HIGH | `≥ 6.30.3` |
| `axios` | `^1.6.7` | DoS via missing data size check (GHSA-4hjh-wcwx-xvwj) | HIGH | `≥ 1.13.5` |
| `rollup` (via vite) | `~4.x` | Arbitrary file write via path traversal (GHSA-mw96-cpmx-2vgc) | HIGH | `≥ 4.58.1` |
| `glob` (transitive) | `10.2.0–10.4.5` | Command injection via CLI (GHSA-5j98-mcp5-4vw2) | HIGH | Upgrade Vite |
| `minimatch` (transitive) | `9.0.0–9.0.6` | ReDoS (GHSA-3ppc-4f35-3m26) | HIGH | Upgrade Vite |
| `esbuild` (via vite) | `≤ 0.24.2` | CORS bypass on dev server (GHSA-67mh-4wv8-2f99) | MODERATE | `≥ 0.25.0` |
| `vite` | `^5.2.0` | Path confusion in middleware (GHSA-g4jq-h2w9-997c) | MODERATE | `≥ 6.1.7` |

---

## Prioritised Remediation Roadmap

### Immediate (Before Any Production Deployment)

1. **F-01** — Rotate Firebase credentials, move all Firebase config to environment variables, update Firebase Security Rules.
2. **F-02** — Redesign the OAuth callback to use fragment or PKCE; never put tokens in query strings.
3. **F-03** — Remove hardcoded password and OTP code; gate test functionality behind `import.meta.env.DEV`.
4. **F-11** — Remove the hardcoded server IP from `App.tsx`.
5. **F-05** — Strip all `console.log` statements from production builds.

### Short-Term (Within 1–2 Sprints)

6. **F-06, F-07, F-08, F-09, F-12, F-13** — Run `npm audit fix --force` and upgrade all vulnerable dependencies.
7. **F-10** — Implement a Content Security Policy and configure standard HTTP security headers.
8. **F-04** — Remove the JWT token display from the Dashboard.
9. **F-21** — Remove test utilities and the DeviceIdDisplay widget from production.

### Medium-Term (Architecture Improvements)

10. **F-14** — Migrate from `sessionStorage` tokens to `HttpOnly` cookies with token refresh via backend.
11. **F-16** — Add server IP/port validation before constructing Steam protocol URLs.
12. **F-19** — Implement a centralised Axios instance with interceptors for auth and token refresh.
13. **F-15** — Add client-side input validation with length limits and character restrictions.
14. **F-17** — Restrict notification data display to curated fields only.
15. **F-18** — Add authentication to the `getUserById` API call.
16. **F-20** — Remove `localhost` fallback URLs; fail loudly on missing env vars.

---

## Appendix: Files Reviewed

| File | Notes |
|---|---|
| `package.json` | Dependencies and scripts |
| `vite.config.ts` | Build configuration |
| `index.html` | Entry point HTML |
| `.gitignore` | Ignore rules |
| `src/main.tsx` | React root |
| `src/App.tsx` | Root component, auth guard, event routing |
| `src/config/firebase.ts` | Firebase credentials (CRITICAL) |
| `src/api/auth.ts` | Authentication API |
| `src/api/user.ts` | User profile API |
| `src/api/friend.ts` | Friend management API |
| `src/api/team.ts` | Team management API |
| `src/api/party.ts` | Party/matchmaking API |
| `src/api/hardware.ts` | Hardware profile API |
| `src/api/notifications.ts` | Notifications API |
| `src/api/push-tokens.ts` | Push token registration API |
| `src/api/map-banning.ts` | Map banning API |
| `src/api/live-matches.ts` | Live match API |
| `src/api/match-statistics.ts` | Match statistics API |
| `src/api/games.ts` | Games list API |
| `src/api/game-modes.ts` | Game modes API |
| `src/api/ratings.ts` | Player ratings API |
| `src/pages/Login.tsx` | Login page (CRITICAL) |
| `src/pages/Register.tsx` | Registration page |
| `src/pages/SocialAuth.tsx` | OAuth callback handler |
| `src/pages/Dashboard.tsx` | Main dashboard |
| `src/pages/Friends.tsx` | Friends management |
| `src/pages/Teams.tsx` | Team management |
| `src/pages/Parties.tsx` | Party/matchmaking |
| `src/pages/Notifications.tsx` | Notifications inbox |
| `src/pages/LiveMatch.tsx` | Live match view |
| `src/pages/LiveMatches.tsx` | Live match list |
| `src/pages/MatchSelection.tsx` | Match history selector |
| `src/pages/MatchStatistics.tsx` | Match statistics |
| `src/pages/HardwareProfile.tsx` | Hardware profile form |
| `src/components/DeviceIdDisplay.tsx` | Debug device info widget |
| `src/components/MapBanningModal.tsx` | Map banning UI |
| `src/components/MatchAcceptanceModal.tsx` | Match acceptance UI |
| `src/components/ServerConnectionModal.tsx` | Server connection UI |
| `src/components/NotificationBadge.tsx` | Notification count badge |
| `src/components/SteamIcon.tsx` | Steam icon SVG |
| `src/components/RatingsDisplay.tsx` | Ratings display |
| `src/utils/push-notifications.ts` | Firebase push notification service |
| `src/utils/device-id.ts` | Device ID generation |
| `src/utils/cs2-connection.ts` | CS2 server connection utility |
| `src/utils/random.ts` | Test data generators |
| `src/utils/background-message-test.ts` | Push notification test harness |
| `public/firebase-messaging-sw.js` | Firebase service worker (CRITICAL) |

---

*Report generated by automated static analysis and manual code review on 2026-02-28.*
