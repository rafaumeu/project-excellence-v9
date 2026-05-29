# Playwright Session Sharing Between Host and Docker

## Pattern: Volume mount + env var

When a Playwright-based scraper runs inside Docker but needs login sessions from the host:

### 1. Docker Compose

```yaml
services:
  web:
    volumes:
      - ./data:/app/data
      - ~/.career-ops/sessions:/app/sessions   # share sessions
    environment:
      - SESSION_DIR=/app/sessions               # tell code where sessions live
```

### 2. Base scraper respects env var

```ts
import { homedir } from "os";
import { resolve } from "path";

const SESSION_DIR = process.env.SESSION_DIR || resolve(homedir(), ".career-ops/sessions");
```

Host CLI uses `~/.career-ops/sessions/` (default). Docker container uses `/app/sessions/` (env var).

### 3. Login generates session on host

```bash
npx tsx src/cli.ts auth linkedin
# Opens Chromium → user logs in → session saved to ~/.career-ops/sessions/linkedin.json
```

### 4. Docker container reads same session via volume mount

Both host CLI and Docker container share the same JSON file. No copy step needed.

---

## Anti-Detection for LinkedIn Scraping

### User-Agent Rotation

```ts
const USER_AGENTS = [
  "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
  "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
  "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36",
];
```

### Viewport Randomization

```ts
const VIEWPORTS = [
  { width: 1920, height: 1080 },
  { width: 1440, height: 900 },
  { width: 1536, height: 864 },
];
```

### Context Setup

```ts
const context = await browser.newContext({
  userAgent: USER_AGENTS[Math.floor(Math.random() * USER_AGENTS.length)],
  viewport: VIEWPORTS[Math.floor(Math.random() * VIEWPORTS.length)],
  locale: "pt-BR",
  timezoneId: "America/Sao_Paulo",
  extraHTTPHeaders: { "Accept-Language": "pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7" },
  storageState: sessionPath,  // reuse login session
});
```

### Natural Delays

```ts
function randomDelay(minMs = 2000, maxMs = 5000): number {
  return minMs + Math.floor(Math.random() * (maxMs - minMs));
}

async function naturalScroll(page: Page): Promise<void> {
  const amount = 300 + Math.floor(Math.random() * 500);
  await page.evaluate((a) => window.scrollBy(0, a), amount);
  await sleep(randomDelay(800, 2000));
}
```

### Rate Limits

- Max 5 job detail fetches per scrape (full descriptions)
- Max 3 job alert scrapes per run
- 2-5s random delays between page navigations
- 3-6s delays between alert scrapes
- Total timeout: 120-180s per scraper

---

## LinkedIn API Limitations

**Critical fact:** LinkedIn Jobs API is NOT available for regular developers. Only LinkedIn Talent Solutions partners can access job search/recommendation endpoints via API.

What OAuth CAN do (scopes: `openid profile w_member_social`):
- Read user profile (`GET /v2/userinfo`)
- Read user posts (`GET /v2/ugcPosts`)
- Publish posts (`POST /v2/ugcPosts`)

What OAuth CANNOT do:
- Search jobs
- Get recommended jobs
- Get saved jobs or alerts
- Access any job-related data

**Conclusion:** For job scraping, Playwright browser automation is the ONLY option. OAuth is useful only for enriching user profile data or publishing content.
