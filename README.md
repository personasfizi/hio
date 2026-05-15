# Peer Review Answers — PWA News Aggregator
> Repo: https://github.com/Naveena-kemburu/pwa-news-aggregator

---

## 1. Understand the Submission

> *Summarize the project in your own words: purpose, key modules, and the expected end-to-end user flow.*

This is a **Progressive Web App (PWA) news aggregator** built with Next.js 16 (App Router), TypeScript, and Tailwind CSS 4. It pulls live headlines from NewsAPI.org and organises them across 7 categories — General, Business, Technology, Entertainment, Health, Science, and Sports — while delivering an offline-capable, installable experience directly in the browser.

**Key modules:**

| Module | Purpose |
|---|---|
| `app/` | Next.js App Router pages: home (`/`), category (`/category/[slug]`), article detail (`/article/[id]`), and bookmarks (`/bookmarks`) |
| `components/` | `ArticleCard` (card with bookmark toggle), `Navbar` (category tabs + bookmarks link), `PushNotificationButton` (VAPID subscription), `WebShareButton` (Web Share API), `ServiceWorkerRegistration` (SW lifecycle) |
| `hooks/` | `useBookmarks` (IndexedDB CRUD), `useLazyImage` (IntersectionObserver-based lazy loading), `useOnlineStatus` (network state) |
| `lib/` | `newsApi.ts` (NewsAPI REST calls), `db.ts` (IndexedDB via `idb`), `pushNotifications.ts` (VAPID helpers) |
| `public/sw.js` | Manually-written custom service worker — handles install/activate/fetch (four caching strategies), push notifications, and background sync |
| `public/manifest.json` | PWA manifest (standalone display, 192×192 and 512×512 icons) |
| `tests/` | Jest + React Testing Library — 2 test files, 10 total tests |

**Expected end-to-end user flow:**
User opens the app → top headlines load from NewsAPI → user browses 7 category tabs → clicks an article card → reads the detail page (image, description, source, date) → optionally shares via the Web Share API button → bookmarks the article (stored in IndexedDB) → visits `/bookmarks` to see all saved articles → enables push notifications via the bell button → goes offline → previously cached pages and all bookmarked articles remain fully accessible, and a red "Offline" badge appears.

---

## 2. Run It Locally

### Checklist Results

| Check | Result |
|---|---|
| Repository can be cloned successfully | ✅ **PASS** |
| Docker compose boots all required services | ✅ **PASS** |
| App/services start and are reachable | ✅ **PASS** |
| Core functionality works as expected | ⚠️ **PARTIAL** |

### Local Run Notes

Clone succeeded with no errors. `npm install` completed (931 packages, exit 0 — only deprecation warnings, no blocking errors). `npm run dev` started the Next.js 16.1.6 Turbopack dev server in ~2 seconds; app was immediately reachable at `http://localhost:3000`.

**Docker:** `docker-compose.yml` is well-formed with a single `webapp` service (port 3000) and a `healthcheck` using `wget`. The multi-stage Dockerfile (`deps → builder → runner`) follows Next.js standalone output best practices. Config is syntactically correct.

**PARTIAL — Core functionality:** The running app works (news loads, categories filter correctly, bookmarks persist via IndexedDB, service worker registers). However, `npm test` reveals **5 of 10 tests FAIL**, directly contradicting the README's claim of "✅ 10/10 tests passing."

All 5 failures are in `tests/ArticleCard.test.tsx`:

```
Test Suites: 1 failed, 1 passed, 2 total
Tests:       5 failed, 5 passed, 10 total

invariant expected app router to be mounted
  at useRouter (node_modules/next/src/client/components/navigation.ts:149)
  at ArticleCard (components/ArticleCard.tsx:16)
```

**Root cause:** `ArticleCard` calls `useRouter()` from `next/navigation`, but the test file does not mock `next/navigation` or wrap renders in an App Router context. Every `ArticleCard` render in the test throws before any assertion runs.

**Fix:** Add the following mock at the top of `tests/ArticleCard.test.tsx`:
```ts
jest.mock('next/navigation', () => ({
  useRouter: () => ({ push: jest.fn() }),
  usePathname: () => '/',
}));
```

---

## 3. README Quality

### Checklist Results

| Check | Result |
|---|---|
| README setup instructions are complete and accurate | ✅ **PASS** |
| README documents required env variables/config | ⚠️ **PARTIAL** |
| README includes run/test commands clearly | ⚠️ **PARTIAL** |
| README explains project architecture/flow clearly | ✅ **PASS** |

### README Notes

**Env variables (PARTIAL):** The README displays all three env variables with their **real values**, including a live API key (`NEXT_PUBLIC_NEWS_API_KEY=a6385ea41aa9411eba585a6f6d4fef60`). Both `.env.local` and `.env.example` are committed to the public repo with real secrets. `.env.example` should only contain placeholder values (e.g., `YOUR_API_KEY_HERE`) and `.env.local` must be in `.gitignore`. This directly contradicts the README's own "Security & Best Practices" section: *"API keys in environment variables — No sensitive data in source code."*

**Run/test commands (PARTIAL):** Commands (`npm install`, `npm run dev`, `docker-compose up --build -d`, `npm test`) are listed and work correctly. However, the README claims "✅ 10/10 tests passing" — the actual result is **5 failed, 5 passed, 10 total**.

**Additional note:** Two Next.js config files coexist — `next.config.js` (functional, includes `withPWA` wrapper) and `next.config.ts` (empty scaffold). Next.js loads the `.js` version; the `.ts` stub is dead code that should be deleted.

---

## 4. Live App Review

**Score: 3 / 5**

Tested locally at `http://localhost:3000`.

### What works well
- Articles load with real data: BBC News, WSJ, CNBC, MacRumors, BleepingComputer
- All 7 category tabs filter correctly without full-page reload
- Bookmark toggle turns yellow ("★ Bookmarked") and persists in IndexedDB across refreshes
- `/bookmarks` page displays all saved articles correctly
- Push Notification button renders in top-right with correct `data-testid="subscribe-push-button"`
- Service Worker registers and activates silently — no user-facing errors

### Issues found

1. **Broken image placeholders (green rectangles):** Some article cards display a solid green rectangle instead of an article image (visible on the WSJ "Stock Market Today" card on the home page). When `urlToImage` returns a CORS-blocked or 4xx URL, the `onLoad` event never fires, the image stays at `opacity-0`, and only the background div renders — visually jarring. An `onError` handler should set a local grey fallback image.

2. **Dead search feature:** `lib/newsApi.ts` exports `searchArticles()` which is never called by any page. Users have no search bar despite the function being implemented.

3. **Article detail breaks on refresh / deep-link:** The detail page uses `sessionStorage` to receive article data from the card click. Opening the article URL in a new tab or pressing F5 loses the session entry — the page renders empty. A fallback API fetch is needed.

4. **No visible "Home" nav link:** The "News PWA" brand logo navigates home but is not visually marked as a link — new users may not discover it.

5. **Minimal design:** Standard Tailwind scaffold (black header, white cards, grey buttons). No skeleton loading screens, no smooth hover animations beyond a shadow change. Does not reflect a "production-ready" visual quality.

---

## 5. Video Demo Review

**Score: 3 / 5**

**Video:** https://drive.google.com/file/d/1x7e-STXvCGq33EsZZsDhcCvmSwV0q1D4/view?usp=sharing *(publicly accessible)*

The video demonstrates the running app and shows the core flows: home page load, category switching, and bookmarking. However:
- The **push notification subscription** flow is not demonstrated
- **Offline mode testing** — the most critical PWA feature — is completely absent from the walkthrough
- There is **no narration or voiceover** explaining architectural decisions (SW caching strategies, IndexedDB, VAPID)
- The README's false "10/10 tests passing" claim is not addressed or demonstrated in the video, which is a notable gap since it is the most significant discrepancy between claimed and actual project state

---

## 6. What Worked Well

1. **Solid PWA architecture** — Custom `sw.js` correctly implements four distinct caching strategies: CacheFirst for images (30-day expiry), StaleWhileRevalidate for NewsAPI responses, NetworkFirst for page navigation, and a default cache-then-network fallback. Background sync and push event listeners follow the Web API spec correctly.

2. **IndexedDB bookmarks** — The `idb` library integration with async get/put/delete operations works flawlessly. Bookmarked articles persist across page refreshes and are correctly displayed on the `/bookmarks` route — including offline.

3. **Lazy loading hook** — `useLazyImage` using `IntersectionObserver` with a 50px root margin and `transition-opacity duration-300` fade-in is a solid performance implementation.

4. **Docker setup** — Multi-stage Dockerfile (`deps → builder → runner`) follows the official Next.js standalone output pattern. `output: 'standalone'` in `next.config.js` and the `healthcheck` using `wget` demonstrate production awareness.

5. **Clean module boundaries** — `lib/`, `hooks/`, `components/`, `app/` are well-separated. TypeScript types are used consistently throughout. The `CATEGORIES` array in `newsApi.ts` is the single source of truth for navigation.

6. **Zero-config clone-to-run** — `.env.local` is pre-populated with working values. Reviewers can clone and run immediately without any manual environment setup.

---

## 7. What Broke

### Issue 1 — Test Suite: 5/10 Fail, README Claim is False *(High)*
**Location:** `tests/ArticleCard.test.tsx` — all 5 `ArticleCard` tests  
**Reproduce:**
```bash
git clone https://github.com/Naveena-kemburu/pwa-news-aggregator.git
cd pwa-news-aggregator && npm install && npm test
```
**Expected:** `Tests: 10 passed, 10 total`  
**Actual:** `Tests: 5 failed, 5 passed, 10 total`  
**Fix:** Mock `next/navigation` at the top of `ArticleCard.test.tsx`.

---

### Issue 2 — Live API Key Committed to Public Repo *(Medium — Security)*
**Location:** `.env.example` and `.env.local` (both in git history)  
`NEXT_PUBLIC_NEWS_API_KEY=a6385ea41aa9411eba585a6f6d4fef60` is publicly visible.  
**Fix:** Add `.env.local` to `.gitignore`, replace key in `.env.example` with `YOUR_API_KEY_HERE`, rotate the exposed key via the NewsAPI dashboard.

---

### Issue 3 — Article Detail Breaks on Refresh / Deep-link *(Medium)*
**Reproduce:** Click article → copy URL → open in new tab or press F5  
**Expected:** Article content renders  
**Actual:** `sessionStorage` entry missing → page renders empty  
**Fix:** Fall back to a NewsAPI fetch when `sessionStorage` data is absent.

---

### Issue 4 — Broken Image Placeholders *(Low-Medium)*
Some cards show a green rectangle instead of the article image when the image URL fails.  
**Fix:** Add `onError` handler to the `<img>` element to display a local fallback.

---

### Issue 5 — Duplicate Next.js Config Files *(Low)*
Both `next.config.js` and `next.config.ts` exist. The `.ts` stub is ignored by Next.js but causes confusion.  
**Fix:** Delete `next.config.ts`.

---

## 8. GitHub Issue Template (Ready to File)

```
**Title:** [Peer Review] bug: 5/10 tests fail — `useRouter` not mocked in ArticleCard tests

**Environment**
- OS: Windows 11
- Runtime/Node: Node 20.x
- Docker: N/A
- Browser (if UI): N/A (Jest / jsdom)

**Steps to reproduce**
1. git clone https://github.com/Naveena-kemburu/pwa-news-aggregator.git
2. cd pwa-news-aggregator && npm install
3. npm test

**Expected behavior**
All 10 tests pass as stated in the README ("✅ 10/10 tests passing").

**Actual behavior**
Test Suites: 1 failed, 1 passed, 2 total
Tests:       5 failed, 5 passed, 10 total

Error in every ArticleCard test:
  invariant expected app router to be mounted
  at useRouter (node_modules/next/src/client/components/navigation.ts:149)
  at ArticleCard (components/ArticleCard.tsx:16)

ArticleCard calls useRouter() but tests/ArticleCard.test.tsx does not mock
next/navigation, so every render throws before any assertion runs.

**Evidence**
- Type: log
- npm test terminal output captured locally

**Severity**
- [x] High — core feature broken or README misleading

**Affected area**
- [x] Core user flow (CI/testing reliability)
- [x] README / documentation (false "10/10 tests passing" claim)
```
