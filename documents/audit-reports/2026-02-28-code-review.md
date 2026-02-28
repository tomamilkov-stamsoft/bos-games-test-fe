# Code Quality Review — bos-games-test-fe
**Date:** 2026-02-28
**Reviewer:** Claude Sonnet 4.6 (Automated Code Review)
**Scope:** Full source review of `/src` and `/public` directories

---

## Executive Summary

The `bos-games-test-fe` repository is a React 18 / TypeScript / Vite frontend for a gaming matchmaking platform. The codebase shows clear signs of being a **rapid prototype or internal test harness** rather than a production-ready application. The code is functional, but contains a significant number of issues that would block a production release:

- **343 `console.log/warn/error` calls** left in production code across 23 files, including sensitive data such as auth tokens, VAPID keys, Firebase credentials, and full user objects.
- **Firebase API keys and VAPID keys are hardcoded** in version-controlled source files, representing a direct credential-exposure vulnerability.
- **No ESLint, Prettier, or testing framework** configured — `package.json` has no linting scripts and `tsconfig.json` is absent, meaning TypeScript is run only through Vite's loose transpilation.
- **Pervasive `any` typing** (27 occurrences) and missing return-type annotations leave the majority of data-handling code unchecked.
- **No global HTTP client or auth interceptor** — authentication headers and `sessionStorage` reads are copy-pasted into every API call across 14 separate files.
- **Duplicate business logic** — the Steam/CS2 server-connect flow (building the `steam://run/730//...` URL and calling `window.location.href`) is copy-pasted verbatim in at least 4 different places.
- **Hardcoded test credentials** (`"Test123!"`, `devCode = "111111"`) and test-only utility code (`BackgroundMessageTest`, `random.ts`, `device-id.test.ts`) are shipped with the main bundle.
- **Zero test coverage** — the single `device-id.test.ts` file is not a real test (no test runner, no assertions, it writes to `window` and uses `console.log`).

---

## Quality Scorecard

| Dimension | Score | Notes |
|---|---|---|
| TypeScript Safety | 3/10 | 27 `any` usages, no tsconfig, implicit `any` on caught errors |
| React Best Practices | 4/10 | Almost no `memo`/`useCallback`, large monolithic components, stale closure risks |
| Error Handling | 4/10 | Inconsistent, swallowed errors in many catch blocks |
| Code Duplication (DRY) | 3/10 | API auth pattern, CS2 connect logic, player stats map all duplicated |
| Security Posture | 2/10 | Hardcoded credentials, JWT in logs, no HTTPS enforcement |
| Logging Practices | 1/10 | 343 calls in production, many leak PII and secrets |
| Performance | 4/10 | No code-splitting, `NotificationBadge` polls every 30s, large components |
| Test Coverage | 1/10 | No real tests exist |
| Naming / Conventions | 6/10 | Generally consistent but some inconsistencies noted |
| Build / Tooling | 4/10 | No ESLint, Prettier, or tsconfig; missing `.env.example` |

**Overall: 3.2 / 10**

---

## Findings

---

### CRITICAL — Security

---

#### [SEC-1] Firebase API key and VAPID key hardcoded in source control
**Priority:** HIGH
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/config/firebase.ts` (lines 3–19)
**Also:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/public/firebase-messaging-sw.js` (lines 54–62)

Both the `firebaseConfig` object (including `apiKey`, `messagingSenderId`, `appId`, `measurementId`) and the FCM VAPID public key are committed directly in source. Because these files are tracked by git and likely pushed to a remote, the credentials are effectively public.

While Firebase API keys are not strictly secret (they are rate-limited by Firebase Security Rules), committing them encourages unsafe practices and exposes the project ID and sender ID which can facilitate notification spoofing or abuse quota attacks.

**Before:**
```ts
// src/config/firebase.ts
export const firebaseConfig = {
  apiKey: "AIzaSyDUsEFOdlNO8muiTUx0Em65KY59Da_V_3A",
  appId: "1:303868662210:web:8d737920e41afe8540065b",
  ...
};
export const vapidKey = "BKIlF0YS9FZix8JOnhkmfXQ2v5uItzP1wI_tNcQ14fr6yt8...";
```

**After:**
```ts
// .env (gitignored)
VITE_FIREBASE_API_KEY=AIzaSy...
VITE_FIREBASE_APP_ID=1:303868...
VITE_FIREBASE_VAPID_KEY=BKIlF0...

// src/config/firebase.ts
export const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  appId: import.meta.env.VITE_FIREBASE_APP_ID,
  ...
};
export const vapidKey = import.meta.env.VITE_FIREBASE_VAPID_KEY;
```

Add `.env.example` with placeholder values and ensure `.env` is in `.gitignore`.

---

#### [SEC-2] Auth tokens logged to the browser console
**Priority:** HIGH
**Files:**
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Dashboard.tsx` line 14: `console.log("token", token)`
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Login.tsx` line 87: `console.log("user", user)`
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/SocialAuth.tsx` line 37: `console.log("Social auth user", user)`
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/utils/push-notifications.ts` line 432: `console.error("Current VAPID key:", vapidKey)`

Logging raw JWT tokens and full user objects provides an easy attack surface via browser DevTools, browser extensions, and any monitoring tools that capture console output.

**Fix:** Remove all `console.log` statements containing token, user, or credential data. See [LOG-1] for the broader logging strategy.

---

#### [SEC-3] Hardcoded test password and email-verification bypass code
**Priority:** HIGH
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Register.tsx` (lines 19, 31)
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Login.tsx` (line 18)

```ts
// Register.tsx
const [password] = useState("Test123!");   // line 19
const devCode = "111111";                   // line 31 — email verification bypass

// Login.tsx
const [password] = useState("Test123!");   // line 18
```

The password is displayed on screen and pre-filled for the user. The `devCode = "111111"` hardcodes the expected email verification code, bypassing the verification flow. If the backend also accepts `111111` in non-development environments, this represents a complete account-takeover bypass.

**Fix:** Move these to environment variables gated on `import.meta.env.DEV`. If this page is only meant for development/demo purposes, it must be excluded from production builds using feature flags or dynamic imports.

---

#### [SEC-4] JWT stored in `sessionStorage` — accessible to all same-origin scripts
**Priority:** MEDIUM
**Files:** All page components and `src/api/auth.ts`

The application stores JWT access tokens in `sessionStorage`. While this avoids persistence across tabs, it remains accessible to any JavaScript executing on the same origin, including injected third-party scripts. The `token` is read via `sessionStorage.getItem("token")` in 16 different files without any encapsulation or access control.

**Fix:** Centralize token storage and retrieval behind a single `authStore` module or React context. If cookie-based auth is not feasible, at minimum ensure the token is never directly accessible from component code — pass it through context or a dedicated hook.

---

### HIGH — Logging

---

#### [LOG-1] 343 `console.log/warn/error` calls in production code
**Priority:** HIGH
**Files:** All 23 source files — highest concentrations in:
- `src/App.tsx`: 143 calls
- `src/utils/push-notifications.ts`: 58 calls
- `src/utils/background-message-test.ts`: 35 calls
- `src/components/MapBanningModal.tsx`: 18 calls
- `src/pages/LiveMatch.tsx`: 13 calls

This is the single most pervasive quality issue. Console logging:
1. Leaks sensitive data (tokens, user objects, VAPID keys — see SEC-2)
2. Degrades runtime performance in hot paths (push notification handlers, timer ticks)
3. Pollutes the developer console for anyone debugging
4. Signals the codebase is in a debug/prototype state

**Fix:** The project needs a structured logging layer:

```ts
// src/utils/logger.ts
const isDev = import.meta.env.DEV;

export const logger = {
  debug: (...args: unknown[]) => isDev && console.debug(...args),
  info: (...args: unknown[]) => isDev && console.info(...args),
  warn: (...args: unknown[]) => console.warn(...args),  // keep in prod
  error: (...args: unknown[]) => console.error(...args), // keep in prod
};
```

Replace all `console.log` calls with `logger.debug` or remove them entirely. `console.warn` and `console.error` may remain for genuine error conditions but should never log PII or credentials.

---

#### [LOG-2] `console.log` inside a per-second timer tick
**Priority:** HIGH
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/MapBanningModal.tsx` (line 183)

```ts
timerRef.current = setInterval(() => {
  setTimeRemaining((prev) => {
    console.log("Timer tick:", prev, "seconds remaining"); // fires every second
    ...
  });
}, 1000);
```

This fires a `console.log` every 1000ms for the duration of the map-banning phase. It saturates the console and causes measurable runtime overhead.

**Fix:** Remove this log entirely. A timer tick is not an error condition and does not need logging.

---

#### [LOG-3] `console.log` inside every render of `MapBanningModal`
**Priority:** HIGH
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/MapBanningModal.tsx` (lines 37–54)

```ts
// This executes on every render of the component
console.log("MapBanningModal debug:", {
  currentUserId,
  session: session ? { ... } : null,
  isCurrentUserTurn,
  isVisible,
  timeRemaining,
});
```

React components re-render frequently. Placing an unconditional `console.log` at the top level of a component body runs on every render cycle, logging the full session object each time.

**Fix:** Remove this debug log. If debugging is needed, use React DevTools or place the log inside a `useEffect` with a dependency array.

---

### HIGH — TypeScript Safety

---

#### [TS-1] Widespread `any` typing in state declarations
**Priority:** HIGH
**Files:** Multiple page components

```ts
// Friends.tsx lines 16–20
const [users, setUsers] = useState<any[]>([]);
const [friends, setFriends] = useState<any[]>([]);
const [received, setReceived] = useState<any[]>([]);
const [sent, setSent] = useState<any[]>([]);
const [currentUser, setCurrentUser] = useState<any>(null);

// Teams.tsx lines 23–28
const [myTeams, setMyTeams] = useState<any[]>([]);
const [users, setUsers] = useState<any[]>([]);
const [friends, setFriends] = useState<any[]>([]);
const [games, setGames] = useState<any[]>([]);
const [gameModes, setGameModes] = useState<any[]>([]);
const [invites, setInvites] = useState<any[]>([]);

// Dashboard.tsx line 10
const [userProfile, setUserProfile] = useState<any>(null);

// Parties.tsx — 11 separate `any` uses
```

Using `any` defeats TypeScript's type system entirely. Data shapes are known from the API responses — they should be modelled as interfaces.

**Fix:** Define and export interfaces for all domain objects. The `match-statistics.ts` and `live-matches.ts` files already show the correct pattern. Apply the same to `User`, `Friend`, `FriendRequest`, `Team`, `TeamInvite`, `Party`, `Game`, `GameMode`.

```ts
// Example: src/types/user.ts
export interface User {
  id: string;
  email: string;
  nickname?: string;
  firstName?: string;
  lastName?: string;
  country?: string;
}
```

---

#### [TS-2] Implicit `any` on caught errors
**Priority:** HIGH
**Files:** `src/pages/HardwareProfile.tsx` (lines 166, 203), `src/utils/push-notifications.ts` (lines 402–405), and others

```ts
// HardwareProfile.tsx — TypeScript will infer `error` as `any` here
} catch (error) {
  setMessage({
    type: "error",
    text: error.response?.data?.message || "Failed to save hardware profile",
  });
}
```

With `useUnknownInCatchVariables` (TypeScript 4.0+ strict mode), `error` in a catch block should be typed as `unknown`. Directly accessing `error.response` without a type guard will cause a compile error in strict mode.

**Fix:**
```ts
} catch (error) {
  const message =
    error instanceof Error
      ? (error as any).response?.data?.message ?? error.message
      : "Failed to save hardware profile";
  setMessage({ type: "error", text: message });
}
```

Or define a typed error helper:
```ts
// src/utils/error.ts
import axios from "axios";
export function getErrorMessage(error: unknown, fallback: string): string {
  if (axios.isAxiosError(error)) {
    return error.response?.data?.message ?? error.message ?? fallback;
  }
  if (error instanceof Error) return error.message;
  return fallback;
}
```

---

#### [TS-3] `data?: any` on the `Notification` interface
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/api/notifications.ts` (line 10)

```ts
export interface Notification {
  ...
  data?: any;   // any defeats type checking for notification payloads
  ...
}
```

The `data` field is accessed in multiple places (`notification.data?.matchId`, `notification.data?.data`). Typing it as `any` means errors in property access are invisible.

**Fix:** Define a discriminated union or at minimum a `Record<string, unknown>` with known fields:
```ts
data?: {
  matchId?: string;
  type?: string;
  [key: string]: unknown;
};
```

---

#### [TS-4] Missing `tsconfig.json` / TypeScript strict mode
**Priority:** HIGH
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/` — no `tsconfig.json` found

The project lacks a `tsconfig.json`. Vite will run TypeScript in transpile-only mode (no type checking at all), and the `build` script also performs no type checking. The entire TypeScript safety layer is effectively disabled — errors will never surface at build time.

**Fix:** Add `tsconfig.json` with strict mode and add a type-check step to the build:

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noImplicitAny": true,
    "useUnknownInCatchVariables": true,
    "noUncheckedIndexedAccess": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

```json
// package.json scripts
"typecheck": "tsc --noEmit",
"build": "tsc --noEmit && vite build"
```

---

### HIGH — Architecture / DRY Violations

---

#### [DRY-1] Auth header pattern duplicated across all 14 API files
**Priority:** HIGH
**Files:** Every file under `src/api/`

Every single API call manually constructs the `Authorization` header:
```ts
// Repeated in auth.ts, user.ts, friend.ts, team.ts, party.ts, notifications.ts,
// game-modes.ts, games.ts, hardware.ts, live-matches.ts, map-banning.ts,
// match-statistics.ts, push-tokens.ts, ratings.ts
headers: { Authorization: `Bearer ${token}` }
```

This is copy-pasted verbatim across all 14 API files. Any change to the auth scheme (e.g., adding a `X-Device-ID` header) requires editing every file.

**Fix:** Create a shared Axios instance with an interceptor:

```ts
// src/api/client.ts
import axios from "axios";

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});

apiClient.interceptors.request.use((config) => {
  const token = sessionStorage.getItem("token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      sessionStorage.clear();
      window.location.href = "/login";
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

All API functions then reduce to:
```ts
// Before (user.ts)
export async function getMe(token: string) {
  const resp = await axios.get(`${API_BASE_URL}/users/me`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return resp.data;
}

// After
export async function getMe() {
  const resp = await apiClient.get("/users/me");
  return resp.data;
}
```

This also eliminates the need to pass `token` as a parameter to every API function.

---

#### [DRY-2] `getMe` function defined twice
**Priority:** HIGH
**Files:**
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/api/auth.ts` (line 69)
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/api/user.ts` (line 13)

Both files export an identical `getMe` function hitting `/users/me`. The `Friends.tsx` page imports from `auth.ts` while `Dashboard.tsx` imports from `user.ts`. This creates inconsistency and confusion about the canonical source.

**Fix:** Keep `getMe` in `user.ts` only. Remove from `auth.ts`. Update all import sites.

---

#### [DRY-3] CS2 server connect logic copy-pasted in 4 locations
**Priority:** HIGH
**Files:**
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/utils/cs2-connection.ts` (lines 17–41)
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/LiveMatch.tsx` (lines 603–623)
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/LiveMatches.tsx` (lines 40–61)
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/ServerConnectionModal.tsx` (lines 59–86)

The same `tryConnect` pattern (building a `steam://run/730//+connect ...` URL, setting `window.location.href`, catching errors) is copy-pasted identically in the page components and the modal, despite a utility function already existing in `cs2-connection.ts`.

**Fix:** The `connectToCS2Server` function in `cs2-connection.ts` already exists for this purpose. Import and use it everywhere instead of inlining the logic.

---

#### [DRY-4] Player stats mapping duplicated for `round_completed` and `match_completed`
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/utils/push-notifications.ts` (lines 222–252 and 291–326)

The 25-line block that maps raw push-notification player stats fields (`player.stats.kills_with_headshot`, `player.stats["2ks"]`, etc.) to the camelCase `LiveMatchPlayer` shape is copy-pasted verbatim for both `round_completed` and `match_completed` handlers.

**Fix:** Extract a `mapPlayerStats(player: RawPlayerData): PlayerStats` helper function and call it from both handlers.

---

#### [DRY-5] `API_BASE_URL` constant declared in each of the 14 API files
**Priority:** MEDIUM
**Files:** All `src/api/*.ts`

```ts
// Repeated 14 times across all API files:
const API_BASE_URL = import.meta.env.VITE_API_URL;
```

Three files also have inconsistent fallbacks:
- `live-matches.ts`: `|| "http://localhost:3000"`
- `match-statistics.ts`: `|| "http://localhost:3000"`
- `hardware.ts`: `|| "http://localhost:3000"`
- `ratings.ts`: `|| "http://localhost:3000/api"` — **different path!**

The `ratings.ts` fallback includes `/api` which the others don't, creating a subtle inconsistency that will cause 404s if `VITE_API_URL` is unset.

**Fix:** Centralise in the Axios client (see [DRY-1]). The `baseURL` is set once. No module needs its own `API_BASE_URL`.

---

### HIGH — React Best Practices

---

#### [REACT-1] `App.tsx` is a 500+ line monolith with 143 console.log calls
**Priority:** HIGH
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/App.tsx`

The root `App` component handles: authentication state, push notification setup, match acceptance modals, server connection modals, map banning modals, background service worker message polling, and location updating — all in a single component with 143 `console.log` calls.

This violates the Single Responsibility Principle and makes the component extremely difficult to test, maintain, or reason about.

**Fix:** Extract responsibilities into dedicated hooks:
- `useAuth()` — authentication state and token management
- `usePushNotifications(authToken)` — Firebase initialisation and subscription
- `useMatchEvents()` — custom event listeners (`matchFound`, `round-end`, `match-end`, `player-update`)
- `useServiceWorkerMessages()` — the `navigator.serviceWorker.addEventListener("message", ...)` polling

The modal state can be managed by a `useModalManager()` hook or a small state machine.

---

#### [REACT-2] `debouncedSearch` in Friends.tsx uses IIFE + `useCallback` incorrectly
**Priority:** HIGH
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Friends.tsx` (lines 77–99)

```ts
const debouncedSearch = useCallback(
  (() => {
    let timeoutId: NodeJS.Timeout;       // captured in closure
    return (searchQuery: string) => {
      clearTimeout(timeoutId);
      timeoutId = setTimeout(async () => {
        ...
      }, 300);
    };
  })(),   // IIFE — executes immediately at render time
  [token] // dependency array is on the outer useCallback
);
```

The IIFE runs when the component first renders and creates a closure over `timeoutId`. On re-renders, `useCallback` returns the same inner function (due to `[token]` dependency), so the closure is preserved — this accidentally works. However, the pattern is confusing and non-idiomatic. When `token` changes (e.g., after a re-login), a new IIFE runs and a new `timeoutId` closure is created, but the old `timeoutId` from the previous IIFE may still fire. This is a stale-closure bug.

**Fix:** Use a `useRef` to hold the timeout ID:
```ts
const timeoutRef = useRef<NodeJS.Timeout>();

const debouncedSearch = useCallback((searchQuery: string) => {
  clearTimeout(timeoutRef.current);
  timeoutRef.current = setTimeout(async () => {
    if (!token) return;
    setSearchLoading(true);
    try {
      const results = await searchUsers(token, searchQuery, 20);
      setUsers(results);
    } catch (error) {
      setStatus("Error searching users");
    } finally {
      setSearchLoading(false);
    }
  }, 300);
}, [token]);
```

---

#### [REACT-3] `NotificationBadge` polls the full notification list every 30 seconds
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/NotificationBadge.tsx` (lines 22–23)

```ts
const response = await getMyNotifications(token, 1, 100); // fetches 100 notifications
const unreadCount = response.data.filter((n) => !n.read).length;
```

The badge fetches up to 100 notifications every 30 seconds just to count the unread ones. This is wasteful when only a count is needed. If the API provides a dedicated unread-count endpoint, it should be used. If not, a HEAD request or a small-page fetch (`limit=1`) with `meta.total` unread would suffice.

Additionally, the `token` read happens at render time (line 14) rather than inside the effect, meaning if the token is set after the initial render, the effect dependency `[token]` won't re-fire correctly because `token` is a `const` in the render scope.

**Fix:** Move the `token` read inside the effect, or use a context value:
```ts
useEffect(() => {
  const token = sessionStorage.getItem("token"); // read fresh each time
  if (!token) return;
  ...
  const interval = setInterval(fetchNotificationCount, 30_000);
  return () => clearInterval(interval);
}, []); // no dependency needed
```

---

#### [REACT-4] Team player stats block copy-pasted verbatim for Team 1 and Team 2 in LiveMatch
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/LiveMatch.tsx` (lines 291–413 and 416–537)

The JSX block rendering a player card (with Headshots, Pistol Kills, Sniper Kills, Damage, Double Kills, Triple Kills, etc.) is copy-pasted identically for Team 1 and Team 2, spanning approximately 120 lines each. The only difference is the heading colour (`text-red-800` vs `text-blue-800`).

**Fix:** Extract a `PlayerCard` component and a `TeamSection` component:
```tsx
<TeamSection team={1} players={getTeamPlayers(1)} name={getTeamName(1)} colorClass="text-red-800" />
<TeamSection team={2} players={getTeamPlayers(2)} name={getTeamName(2)} colorClass="text-blue-800" />
```

---

#### [REACT-5] Multiple `useEffect` hooks have missing or incorrect dependency arrays
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Notifications.tsx` (line 24)

```ts
useEffect(() => {
  if (token) {
    loadNotifications();
  }
}, [token, page]); // loadNotifications is not in the dependency array
```

`loadNotifications` is a function defined in the same component scope and closes over `token` and `page`. Not including it in the dependency array means stale closures could use outdated values. The eslint `react-hooks/exhaustive-deps` rule would catch this.

**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/MapBanningModal.tsx` (line 110)

```ts
useEffect(() => {
  ...
}, [session, isCurrentUserTurn, timeRemaining, currentUserId]);
// `timeRemaining` in a dependency array causes this effect to run every second
// because `timeRemaining` is the state variable updated by the timer
```

Including `timeRemaining` as a dependency causes this effect to re-run on every timer tick, which produces the verbose per-second logging and potentially resets the "not user's turn" timer display every second.

**Fix:** Add ESLint with `eslint-plugin-react-hooks` and address all `exhaustive-deps` warnings. For `MapBanningModal`, remove `timeRemaining` from the dependency array of the session-update effect (it should only react to session and user changes, not the local timer).

---

#### [REACT-6] No `React.memo`, `useMemo`, or `useCallback` anywhere except Friends.tsx
**Priority:** MEDIUM
**Files:** All component files

With the exception of the flawed `useCallback` in `Friends.tsx`, the entire codebase has zero memoisation. Components like `LiveMatch` (which re-renders on every timer tick via `setMatchStats`) and `MapBanningModal` (which re-renders every second) will cause all child components to re-render at 1Hz. As the application grows, this will cause noticeable performance degradation.

**Fix:** Apply `React.memo` to pure display components (`PlayerCard`, `RatingsDisplay`, `NotificationBadge`, map grid items). Use `useCallback` for stable event handlers passed as props.

---

### HIGH — Error Handling

---

#### [ERR-1] Errors swallowed silently in multiple catch blocks
**Priority:** HIGH
**Files:** Multiple

```ts
// MatchAcceptanceModal.tsx line 69
const handleAccept = async () => {
  setIsLoading(true);
  try {
    await onAccept();
  } catch (error) {
    console.error("Error accepting match:", error); // logged but not shown to user
    setIsLoading(false);
    // isLoading stays false but no error state is set — UI gives no feedback
  }
};
```

```ts
// Dashboard.tsx lines 18–20
.catch(() => {
  sessionStorage.clear();
  navigate("/login");
  // The error is silently consumed — user has no idea why they were logged out
})
```

```ts
// App.tsx — multiple push notification handlers
} catch (error) {
  console.error("Failed to ...", error);
  // No user-visible error state set
}
```

**Fix:** For user-initiated actions (accept match, ban map), always set an error state that renders a message to the user. For background operations, logging is acceptable but should be structured (see [LOG-1]).

---

#### [ERR-2] Floating promise in `Notifications.tsx`
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Notifications.tsx` (line 24)

```ts
useEffect(() => {
  if (token) {
    loadNotifications(); // returns a Promise, but it is not awaited
  }
}, [token, page]);
```

Returning a promise from a `useEffect` callback (without awaiting it) is a React anti-pattern. If `loadNotifications` throws after the component unmounts, it will produce an "update on unmounted component" warning or worse, an unhandled rejection.

**Fix:**
```ts
useEffect(() => {
  if (!token) return;
  let cancelled = false;

  const run = async () => {
    try {
      await loadNotifications();
    } catch (err) {
      if (!cancelled) setError("Failed to load notifications");
    }
  };

  run();
  return () => { cancelled = true; };
}, [token, page]);
```

---

#### [ERR-3] `validateAccessToken` returns `{ success: false, error }` instead of throwing
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/api/auth.ts` (lines 126–140)

```ts
export async function validateAccessToken(accessToken: string) {
  try {
    const user = await getMe(accessToken);
    sessionStorage.setItem("token", accessToken);
    sessionStorage.setItem("user", JSON.stringify(user));
    return { user, success: true };
  } catch (error) {
    return { success: false, error }; // returns error object, not thrown
  }
}
```

This mixes error-handling paradigms. The function signals failure via a return value (like a Go-style `{error}` tuple) but all callers are written in async/await style expecting thrown exceptions. This means a network failure is indistinguishable from an invalid-token response at the call site.

**Fix:** Either throw on failure (consistent with other auth functions) or establish a project-wide convention and apply it everywhere.

---

### MEDIUM — Performance

---

#### [PERF-1] No code splitting — entire application loads as a single chunk
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/vite.config.ts`

```ts
export default defineConfig({
  plugins: [react()],
  // No manual chunks, no lazy loading configured
})
```

All 14 page components are eagerly imported in `App.tsx`. Firebase (a large library) is dynamically imported in `push-notifications.ts` — which is the correct approach — but the page components themselves are not lazy loaded. For a production app with this many pages and the heavy `MatchStatistics` component, initial bundle size will be unnecessarily large.

**Fix:** Use React's `lazy` and `Suspense` for page-level code splitting:
```ts
// App.tsx
const Dashboard = lazy(() => import("./pages/Dashboard"));
const MatchStatistics = lazy(() => import("./pages/MatchStatistics"));
// etc.

// In JSX:
<Suspense fallback={<div>Loading...</div>}>
  <Routes>...</Routes>
</Suspense>
```

---

#### [PERF-2] `NotificationBadge` fetches 100 notifications every 30 seconds for every authenticated page
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/NotificationBadge.tsx` (line 22)

See [REACT-3]. The polling interval compounds with the large page fetch (100 items) and runs on every authenticated page load. For a user with the dashboard open, this is 2 API calls per minute purely for a badge count.

---

#### [PERF-3] `setTimeout` in navigation without cleanup
**Priority:** MEDIUM
**Files:**
- `Login.tsx` line 94: `setTimeout(() => navigate("/dashboard"), 1000)`
- `Register.tsx` line 77: `setTimeout(() => navigate("/login"), 2000)`
- `SocialAuth.tsx` line 40: `setTimeout(() => navigate("/dashboard"), 1500)`
- `LiveMatch.tsx` line 108: `setTimeout(() => navigate("/match-results/${matchId}"), 5000)`

None of these timeouts are cancelled if the component unmounts before they fire (e.g., user navigates away manually). This causes state updates on unmounted components. The `LiveMatch` case has a 5-second delay which is especially prone to this.

**Fix:**
```ts
useEffect(() => {
  const timer = setTimeout(() => navigate("/dashboard"), 1000);
  return () => clearTimeout(timer);
}, [navigate]);
```

---

#### [PERF-4] `getAllUsers` called on every page load in Friends and Teams
**Priority:** MEDIUM
**Files:**
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Friends.tsx` (line 38)
- `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Teams.tsx` (line 45)

Both pages fetch the full user list on mount. The `getAllUsers` endpoint (`/users`) returns a paginated response, but both pages treat it as a complete list. If the platform has many users, this will be slow and wasteful. In `Friends.tsx`, this is also called again every time the search term is cleared to empty.

**Fix:** Remove the initial `getAllUsers` call. In `Friends.tsx`, the search box already drives the user list — start with an empty list and only populate it when the user types. In `Teams.tsx`, only fetch users when the user explicitly opens an invite flow.

---

### MEDIUM — Code Smells & Antipatterns

---

#### [SMELL-1] `BackgroundMessageTest` utility class shipped in production bundle
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/utils/background-message-test.ts`

This file contains a `BackgroundMessageTest` class that simulates push notifications and service worker messages for manual testing. It is imported in `App.tsx` and registered on `window` (implicitly). It should never be in a production build.

**Fix:** Either delete this file entirely or gate it behind `if (import.meta.env.DEV)`. Remove the import from `App.tsx` for production.

---

#### [SMELL-2] `DeviceIdDisplay` debug widget shipped in production bundle
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/DeviceIdDisplay.tsx`
**Import:** `App.tsx` line 15

The `DeviceIdDisplay` widget shows device ID, platform, and raw user agent in a fixed-position overlay. It is imported and rendered in `App.tsx` unconditionally. Users in production would see a "Device ID" button in the bottom-right corner of every page.

**Fix:** Wrap the import and render with `import.meta.env.DEV`:
```tsx
{import.meta.env.DEV && <DeviceIdDisplay />}
```

---

#### [SMELL-3] `random.ts` test-data generators included in production bundle
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/utils/random.ts`

Functions `randomEmail()`, `randomNickname()`, `randomName()`, `randomCountry()` generate random test data used by `Register.tsx` to auto-populate the registration form. This is test infrastructure that should not exist in a production bundle.

**Fix:** If the registration form is intended for development/demo only, gate the entire page behind a dev flag. Otherwise, remove the `random.ts` import and require users to fill in their own details.

---

#### [SMELL-4] `window.location.reload()` used as error recovery
**Priority:** LOW
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Register.tsx` (line 196)
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/MatchSelection.tsx` (line 109)

```tsx
<button onClick={() => window.location.reload()}>Try Again</button>
```

Full page reloads discard all application state and are rarely the right recovery mechanism in a SPA. React state resets would be more appropriate.

**Fix:** For `Register.tsx`, reset `isStarted` to `false` and clear the error. For `MatchSelection.tsx`, refetch the data via the existing `fetchMatches` function.

---

#### [SMELL-5] `document.execCommand("copy")` deprecated fallback
**Priority:** LOW
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/ServerConnectionModal.tsx` (lines 43–48)

```ts
// Fallback for older browsers
const textArea = document.createElement("textarea");
document.body.appendChild(textArea);
textArea.select();
document.execCommand("copy"); // deprecated
document.body.removeChild(textArea);
```

`document.execCommand` is deprecated in all modern browsers and may be removed in future browser versions. The Clipboard API (`navigator.clipboard.writeText`) is supported in all target environments for a gaming app.

**Fix:** Remove the fallback. If it fails, show a "Please copy manually" message with the text in a selectable input.

---

#### [SMELL-6] `self.controller = navigator.serviceWorker.controller` invalid assignment
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/public/firebase-messaging-sw.js` (line 992)

```js
self.addEventListener("controllerchange", (evt) => {
  console.log("controller changed");
  self.controller = navigator.serviceWorker.controller; // this is invalid in SW context
});
```

Inside a service worker, `self` is the `ServiceWorkerGlobalScope`. It does not have a `controller` property — that property belongs to `navigator.serviceWorker` on the client side. This line is a no-op at best and would throw in strict mode.

**Fix:** Remove this listener entirely if it serves no purpose, or correct it if there is a specific intent.

---

#### [SMELL-7] `key={i}` (array index) used as React key
**Priority:** MEDIUM
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Parties.tsx` (line 386)

```tsx
{mySoloParties.map((party, i) => (
  <div key={i} ...>   // index as key
```

Using array index as `key` causes React reconciliation bugs when the list is reordered or items are inserted/removed. The `party` object likely has an `id` field that should be used.

**Fix:** `key={party.id}`

---

#### [SMELL-8] `RatingsDisplay` uses array index as key
**Priority:** LOW
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/RatingsDisplay.tsx` (line 98)

```tsx
{ratings.map((rating, index) => (
  <div key={index} ...>
```

Same issue as [SMELL-7]. Use `rating.gameModeName` or a composite key if a stable ID is not available.

---

#### [SMELL-9] `devCode` registration verification bypass exposed at component level
**Priority:** HIGH
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/Register.tsx` (line 31)

```ts
const devCode = "111111"; // Demo: Simulate code
```

This bypasses the email verification flow entirely by passing a hardcoded code `"111111"`. If the backend also accepts this code in non-development environments (or if the backend dev environment is accessible from the internet), this is a complete email verification bypass.

**Fix:** This code must never reach production. Gate behind `import.meta.env.DEV` or remove entirely.

---

### MEDIUM — Naming Conventions

---

#### [NAME-1] Inconsistent HTTP client — 10 files use `axios`, 4 use `fetch`
**Priority:** MEDIUM
**Files using `fetch`:** `live-matches.ts`, `match-statistics.ts`, `ratings.ts`, `hardware.ts` (partially — also uses axios)

The codebase mixes `axios` and native `fetch` without a consistent rationale. `axios` provides request/response interceptors (the correct fix for [DRY-1]) while `fetch` does not. Mixing both prevents a unified error handling and auth strategy.

**Fix:** Standardise on `axios` (already the majority choice) and convert the four `fetch`-based files to use the shared axios client from [DRY-1].

---

#### [NAME-2] API parameter naming inconsistency — `token` vs `authToken`
**Priority:** LOW
**Files:** `notifications.ts` uses `authToken`, all others use `token`

```ts
// notifications.ts
export const getMyNotifications = async (authToken: string, ...) => { ... }
export const markNotificationAsRead = async (notificationId: string, authToken: string) => { ... }

// user.ts, friend.ts, etc.
export async function getAllUsers(token: string) { ... }
```

**Fix:** Standardise on `token` as the parameter name everywhere. This becomes moot when moving to the shared client approach ([DRY-1]).

---

#### [NAME-3] Export style inconsistency — named vs default exports
**Priority:** LOW
**Files:** Components mix export styles

- `DeviceIdDisplay.tsx` exports as named: `export const DeviceIdDisplay`
- All other components export as default: `export default ComponentName`

**Fix:** Standardise on default exports for all page and component files.

---

### LOW — Test Coverage

---

#### [TEST-1] No test infrastructure configured
**Priority:** HIGH (for production readiness)
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/package.json`

The `package.json` has no test script, no testing framework (Vitest, Jest), no `@testing-library/react`, and no CI configuration. The `device-id.test.ts` file is named like a test but is actually a runtime utility that logs to console — it contains no assertions and would not run in any test runner.

**Critical paths with zero test coverage:**
- Authentication flow (login, register, token refresh)
- Match acceptance/decline logic
- Map banning timer logic
- Push notification event dispatch
- Error handling on API failures

**Fix:** Add Vitest (compatible with Vite) as the minimum testing setup:
```json
// package.json
"scripts": {
  "test": "vitest",
  "test:ui": "vitest --ui",
  "test:coverage": "vitest --coverage"
},
"devDependencies": {
  "vitest": "^1.0.0",
  "@testing-library/react": "^14.0.0",
  "@testing-library/user-event": "^14.0.0",
  "jsdom": "^24.0.0"
}
```

Priority test targets: `device-id.ts`, `auth.ts` (validateAccessToken), `MapBanningModal` timer logic, push notification event dispatching.

---

### LOW — Dead Code

---

#### [DEAD-1] Round history JSX block commented out in LiveMatch
**Priority:** LOW
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/pages/LiveMatch.tsx` (lines 542–590)

A 50-line JSX block rendering round history is commented out with `{/* ... */}`. If this feature is abandoned, the dead code should be removed. If it is planned, a feature flag or a TODO comment with a tracking issue is more appropriate.

---

#### [DEAD-2] `BackgroundMessageTest` exported but only used for console testing
**Priority:** LOW
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/utils/background-message-test.ts`

The `BackgroundMessageTest` class exports 5 static methods. None of them are used in the application UI — they are only called from the browser console for manual testing. The file itself is dead application code and should be removed from production (see [SMELL-1]).

---

#### [DEAD-3] `SteamIcon` component imported but its usage context is limited
**Priority:** LOW
**File:** `/Users/marin.ranchev.ai/Projects/Audits/bos-games-test-fe/src/components/SteamIcon.tsx`

`SteamIcon` is imported in `Login.tsx` but not actually rendered in the visible JSX — it was presumably used at some point for the Steam login button decoration. Verify whether it is currently rendered and remove if not.

---

## Prioritised Action Items

The following is a suggested remediation order based on impact and effort:

### Immediate (Before Any Production Deployment)

1. **[SEC-1]** Move Firebase config and VAPID key to `.env` — 30 minutes of effort, eliminates credential exposure.
2. **[SEC-3]** Remove hardcoded `"Test123!"` password and `"111111"` verification bypass from the production build — 1 hour.
3. **[SEC-2]** Remove all `console.log` statements that output tokens, user objects, or API keys — can be done with a targeted search-and-replace.
4. **[TS-4]** Add `tsconfig.json` and add `tsc --noEmit` to the build script — 30 minutes. This will immediately surface additional type errors to fix.
5. **[LOG-1]** Strip all 343 `console.log` calls from production (replace with the `logger` utility or delete).

### Sprint 1 (Within 2 Weeks)

6. **[DRY-1]** Create a shared Axios client with auth interceptor — eliminates the single biggest DRY violation and paves the way for all subsequent API refactors.
7. **[DRY-2]** Remove duplicate `getMe` from `auth.ts`.
8. **[DRY-3]** Remove inline CS2 connect logic from `LiveMatch.tsx`, `LiveMatches.tsx`, `ServerConnectionModal.tsx` — use `connectToCS2Server` utility.
9. **[TS-1]** Define `User`, `Team`, `Party`, `FriendRequest`, `GameMode` interfaces and replace all `useState<any>` declarations.
10. **[TS-2]** Add typed error handling in all catch blocks.

### Sprint 2

11. **[REACT-1]** Break `App.tsx` into custom hooks (`useAuth`, `usePushNotifications`, `useMatchEvents`).
12. **[REACT-4]** Extract `PlayerCard` and `TeamSection` components from `LiveMatch.tsx`.
13. **[SMELL-1/2/3]** Gate `BackgroundMessageTest`, `DeviceIdDisplay`, and `random.ts` behind `import.meta.env.DEV`.
14. **[PERF-3]** Add cleanup to all `setTimeout` navigation calls.
15. **[PERF-1]** Add lazy loading for page components.

### Sprint 3 / Ongoing

16. **[TEST-1]** Add Vitest + React Testing Library. Write tests for auth flow, timer logic, and push notification dispatch.
17. **[REACT-2]** Fix the `debouncedSearch` stale-closure bug in `Friends.tsx`.
18. **[PERF-2/4]** Reduce `NotificationBadge` polling footprint; remove `getAllUsers` eager load.
19. **[DRY-4/5]** Centralise `API_BASE_URL`, extract `mapPlayerStats` helper.
20. **[SMELL-6]** Fix invalid `self.controller` assignment in service worker.

---

## Appendix: File-Level Finding Summary

| File | Findings |
|---|---|
| `src/config/firebase.ts` | SEC-1 (hardcoded credentials) |
| `public/firebase-messaging-sw.js` | SEC-1, LOG-1, SMELL-6 |
| `src/App.tsx` | LOG-1 (143 calls), REACT-1, SMELL-1, SMELL-2 |
| `src/api/auth.ts` | DRY-2 (duplicate getMe), SEC-2 |
| `src/api/user.ts` | DRY-2, DRY-1 |
| `src/api/*.ts` (all 14) | DRY-1 (auth header), DRY-5 (API_BASE_URL) |
| `src/api/hardware.ts` | LOG-1 (15 calls), DRY-1 |
| `src/api/notifications.ts` | TS-3 (any on data), NAME-2 |
| `src/api/live-matches.ts` | NAME-1 (fetch vs axios) |
| `src/api/match-statistics.ts` | NAME-1 (fetch vs axios) |
| `src/api/ratings.ts` | NAME-1 (fetch vs axios), DRY-5 (inconsistent fallback) |
| `src/utils/push-notifications.ts` | LOG-1 (58 calls), SEC-2, DRY-4, TS-2 |
| `src/utils/background-message-test.ts` | SMELL-1, DEAD-2 |
| `src/utils/device-id.ts` | LOG-1, TEST-1 |
| `src/utils/device-id.test.ts` | TEST-1 (not a real test) |
| `src/utils/cs2-connection.ts` | DRY-3 |
| `src/utils/random.ts` | SMELL-3 |
| `src/pages/Register.tsx` | SEC-3, SMELL-9, PERF-3, SMELL-4 |
| `src/pages/Login.tsx` | SEC-3, SEC-2, PERF-3 |
| `src/pages/Dashboard.tsx` | SEC-2, TS-1 |
| `src/pages/Friends.tsx` | TS-1, REACT-2, PERF-4, ERR-1 |
| `src/pages/Teams.tsx` | TS-1, PERF-4, ERR-1 |
| `src/pages/Parties.tsx` | TS-1, SMELL-7, ERR-1 |
| `src/pages/Notifications.tsx` | ERR-2, REACT-5 |
| `src/pages/HardwareProfile.tsx` | TS-2, LOG-1 (6 calls), SMELL-4 |
| `src/pages/LiveMatch.tsx` | DRY-3, REACT-4, LOG-1, PERF-3, DEAD-1 |
| `src/pages/LiveMatches.tsx` | DRY-3, LOG-1 |
| `src/pages/MatchSelection.tsx` | SMELL-4 |
| `src/components/MapBanningModal.tsx` | LOG-2, LOG-3, REACT-5 |
| `src/components/MatchAcceptanceModal.tsx` | ERR-1 |
| `src/components/ServerConnectionModal.tsx` | DRY-3, SMELL-5 |
| `src/components/NotificationBadge.tsx` | REACT-3, PERF-2 |
| `src/components/RatingsDisplay.tsx` | SMELL-8 |
| `src/components/DeviceIdDisplay.tsx` | SMELL-2 |
| `package.json` | TEST-1 (no test runner), no lint scripts |
| `vite.config.ts` | PERF-1 (no code splitting) |
| *(missing)* `tsconfig.json` | TS-4 |
| *(missing)* `.env.example` | SEC-1 |
| *(missing)* `.eslintrc` | REACT-5, TS-1, TS-2 (would catch these) |
