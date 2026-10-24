# playwright proxy rotation: The Chromium Per-Context Trap, the Setup That Actually Works, and What Rotation Really Costs

Rotating proxies in Playwright sounds simple until you try it. The `proxy` option is read exactly once when a browser or context object is created, and after that it's locked in. There is no `context.setProxy()`. Rotation in Playwright is a lifecycle problem: you rotate by destroying the object holding the proxy and creating a new one. Once you accept that, there are two paths forward — manage your own proxy pool with per-context rotation inside Playwright, or hand the rotation to a scraping API like ScraperAPI and let its endpoint pick a fresh exit IP for every request. This guide covers both, plus the pricing math that most tutorials skip entirely.

> **Quick pick:** if your goal is JS-heavy scraping without getting blocked, ScraperAPI's Hobby plan at $49/month (100,000 credits, 20 concurrent requests) covers proxy rotation, JavaScript rendering and CAPTCHA handling without you touching any proxy configuration — and a 7-day free trial with 5,000 credits needs no credit card. 👉 [Check ScraperAPI's current plans and prices](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## How proxy rotation actually works in Playwright

Playwright exposes the `proxy` option at three levels: `browserType.launch()`, `browser.newContext()`, and `apiRequest.newContext()`. A `BrowserContext` inherits its network stack — proxy, user agent, locale, cookie jar — at construction and keeps it for life. You can't swap the proxy mid-flight, because the browser process wires the proxy into its network service before the first byte goes out.

So rotation means one of three granularities:

1. **Launch-level**: one proxy for the entire browser run. Right choice when the whole job must exit from one region, say for compliance reasons or country-specific pricing.
2. **Context-level**: the default for most rotation work. Each task gets its own context with its own proxy; closing the context ends that identity.
3. **Request-level**: `apiRequest.newContext()` runs without a browser, useful when you've discovered URLs from an HTML crawl and just need to hit JSON endpoints.

### The Chromium per-context trap

Here's the bug that eats most people's afternoon: Chromium only enables its per-context proxy plumbing when the browser process itself was started with a proxy configured. If you launch Chromium plainly and then set `proxy` on individual contexts, Chromium silently ignores those settings and all your traffic exits from your real IP. Your rotation "works" in the sense that the code runs — it just doesn't do anything.

The fix is a documented sentinel value:

js
const browser = await chromium.launch({
  proxy: { server: 'per-context' }
});


This switches the proxy plumbing on without binding any real upstream, leaving each context free to declare its own. Note that this behaves differently in Firefox, which handles per-context proxies without the sentinel — code that works in Firefox can leak your real address on Chromium.

Two more details that trip people up:

- **Credentials go in `username` and `password`, never in the URL.** Chromium strips the userinfo from `http://user:pass@host:port` and then the gateway returns `407 Proxy Authentication Required`. Keep `server` as scheme, host, and port only.
- **Closing the context is the rotation.** `await context.close()` releases the exit IP and discards the cookie jar with it. If you carry last session's tracking cookies onto a new IP, that inconsistency is something fingerprinting checks look for directly.

A debugging habit worth stealing from the better technical writeups on this topic: before your first real navigation, fire one cheap `context.request.get()` against an IP echo endpoint like `https://api.ipify.org?format=json`. `context.request` shares the context's proxy settings, so the answer reflects what the page will actually use. And if you assert your set of observed exit IPs, do it after the loop, not inside it — an in-loop assertion passes on the first lease and hides the classic failure where the pool collapses to a single exit from the second lease onward.

## Why ScraperAPI's proxy port doesn't work with Playwright (and what to do instead)

Most rotation guides assume you'll plug a proxy gateway into Playwright's `proxy` option. With ScraperAPI that fails, and the reason is specific: ScraperAPI's proxy mode authenticates by API key in a query string, and Playwright's proxy configuration only accepts Basic Auth or IP-based authentication. Pass your key through the `proxy` launch option and you get exactly one thing back: `Proxy Authentication Required`.

ScraperAPI's own docs are upfront that the proxy port (`proxy-server.scraperapi.com:8001`) is not the way to wire up Playwright. The supported method is the **API endpoint approach**: you navigate Playwright to a ScraperAPI URL that wraps your target, and ScraperAPI handles proxy selection, rotation, JS rendering, and CAPTCHAs server-side.

js
const { chromium } = require('playwright');
require('dotenv').config();
const SCRAPERAPI_KEY = process.env.SCRAPERAPI_KEY;

const targetUrl = 'https://httpbin.org/ip';
const scraperApiUrl = `https://api.scraperapi.com?api_key=${SCRAPERAPI_KEY}&url=${encodeURIComponent(targetUrl)}`;

(async () => {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  const page = await context.newPage();
  await page.goto(scraperApiUrl, { waitUntil: 'domcontentloaded' });
  const content = await page.textContent('body');
  console.log('IP Details:', content);
  await browser.close();
})();


The setup is short: Node.js 18+, `npm install playwright dotenv`, and your API key in a `.env` file. One quirk from the official guide — the key goes in without quotes (`SCRAPERAPI_KEY=your_actual_key`), because quotes become part of the value.

Useful query parameters you append to that endpoint:

- `render=true` — JS rendering for dynamic pages
- `country_code=us` — route through a specific country's IPs
- `session_number=123` — pin the same proxy session across requests, handy for pagination
- `premium=true` — residential proxies for harder targets
- `wait_for_selector=.product-title` — wait for a CSS selector (requires `render=true`)

The trade-off is worth stating plainly. With the endpoint method, ScraperAPI fetches the page and returns the result to your browser — you're not driving the target site directly. If your automation depends on interacting with the live page (clicking, typing, multi-step flows in the target's own DOM), self-managed per-context rotation is the better fit. If your goal is reliably getting HTML or JSON from pages that fight back, the endpoint method removes the entire proxy-management layer.

## What proxy rotation via ScraperAPI actually costs

The credit system is where most buyers misjudge their bill. A credit is officially "the cost of a standard page request," but consumption varies by target site and by the parameters you send — and the two multiply.

**Domain-based costs trigger automatically:**

| Target | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon | 5 |
| Google / Bing (incl. subdomains) | 25 |
| LinkedIn | 30 |
| Cloudflare / Datadome / PerimeterX bypass | +10 on top |

**Parameter surcharges:**

| Parameter | Extra credits |
| --- | --- |
| `render=true` | +10 |
| `premium=true` | +10 |
| `ultra_premium=true` | +30 |
| `screenshot=true` | +10 |
| `country_code`, `session_number`, `wait_for_selector` | Free |

Now the part almost nobody mentions: surcharges stack **non-linearly**. `render=true` plus `premium=true` costs +25, not +20. `ultra_premium=true` plus `render=true` costs +75, not the +40 you'd get from adding the parts. This is ScraperAPI's most consistently misunderstood pricing feature, and it matters most right before a big batch job.

To make it concrete: Hobby's 100,000 credits means 100,000 requests against simple HTML blogs, roughly 6,600 Amazon product pages with `render=true`, or about 4,000 Google SERPs. That spread is why testing against your real targets during the trial tells you more than the headline credit number ever will. The dashboard's Domain Cost Estimator shows the exact cost for any URL, and you can set a `max_cost` per scrape so no single request blows past your budget.

Two billing facts in your favor: failed requests don't burn credits (only 200 and 404 responses count as billable), and exceeding your concurrency limit returns a 429 with no overage charge — the request is simply turned away rather than queued.

## ScraperAPI plans compared

Here's the full plan lineup as shown on the official pricing page, with annual billing at 10% off applied automatically at checkout — no code needed.

| Plan | Monthly | Annual (per month) | API Credits | Concurrency | Subscribe |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 | [ Sign up free](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [ Get Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [ Get Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [ Get Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [ Get Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [ Get Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [ Get Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | [ Contact sales](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Every plan, including the free tier, ships with the same core feature set: JS rendering, premium proxies, JSON auto-parsing, rotating proxy pools, custom headers, CAPTCHA and anti-bot detection, custom session support, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. Paid plans add premium residential and mobile IPs, advanced bypassing, structured data APIs, DataPipeline access, and pay-as-you-go eligibility.

Before you pick a tier, check the fine print that doesn't make it onto the headline cards:

- **Geotargeting is gated.** Hobby and Startup are limited to US and EU regions. If you need IPs from other countries at the country level, that starts at Business.
- **Pay-as-you-go is only on Scaling and above.** Run out of credits mid-month on Hobby, Startup, or Business and scraping pauses until renewal — your options are upgrading or waiting. (Auto-upgrade, which bumps you to the next plan automatically, is available and opt-in on Hobby through Professional.)
- **Credits don't roll over.** Unused credits reset at renewal, so match the plan to your real monthly volume rather than averaging in months you won't use.

Annual billing saves 10% across the board — that's $58.80/year on Hobby, $178.80/year on Startup, $358.80/year on Business. And the entry point is forgiving: the free plan gives 1,000 credits with 5 concurrent connections, and the trial adds 7 days with 5,000 credits, no card required. 👉 [Start the free trial with 5,000 credits](https://www.scraperapi.com/?fp_ref=coupons)

## DIY pool rotation vs. a scraping API: which one fits your case

If your targets are mild and your proxy pool is small, rolling your own rotation in Playwright is entirely reasonable. The recipe from the sharpest technical guide on the subject, in five steps:

1. Launch once per worker with `proxy: { server: 'per-context' }`.
2. Model your pool as data — entries with host, port, credentials, sticky token, a `state` field (`available`/`leased`/`cooling`/`retired`), and a `cooldownUntil` timestamp.
3. Create one context per lease, work, then `await context.close()`.
4. Classify failures: `net::ERR_PROXY_CONNECTION_FAILED` means retire the lease; a 429 or challenge page means the exit is burned and goes to cooldown; a 500 from the origin means retry on the same lease.
5. Verify the exit IP with an echo request before the first real navigation.

That's real engineering, and it's the right call when you need interactive browser sessions against the target or when you already own quality proxies.

The calculus changes when anti-bot protection enters the picture. Sites behind Cloudflare, Datadome, or PerimeterX burn DIY pools fast, and each bypass on ScraperAPI adds 10 credits on top of the base request — you're paying for solved CAPTCHAs and rotated residential IPs as a unit price instead of paying for the infrastructure and your own time maintaining it. One third-party review of the service puts the typical user profile well: people who already have scraper code — including Playwright — that gets blocked, and who want one endpoint handling proxies, retries, and rendering.

## FAQ

**Can Playwright rotate proxies natively?**
Yes, by creating new contexts or browsers with different proxy settings — one proxy identity per bounded task, with the context closed when the task finishes. There's no runtime proxy swap; rotation happens at object creation.

**Why does my per-context proxy work in Firefox but not Chromium?**
Chromium only enables per-context proxy support when the browser process itself starts with a proxy configured. Pass `proxy: { server: 'per-context' }` to `launch()` to switch the plumbing on without binding a real upstream.

**Does ScraperAPI's proxy port work with Playwright?**
No. Playwright doesn't support query-string authentication in proxy URLs, which is how ScraperAPI's proxy mode authenticates. Use the API endpoint method instead — navigate Playwright to `api.scraperapi.com` with your key, target URL, and options.

**Does ScraperAPI charge for failed requests?**
Only 200 and 404 responses consume credits. Other response codes are free.

**What happens if I exceed my concurrency limit?**
You get a 429 response asking you to bring concurrency back within your plan's limit. There's no extra charge; the request is turned away rather than queued.

## Which setup to pick, and one thing to check before you pay

Running per-context rotation yourself: pick this if you control good proxies and need live page interaction — just don't forget the `per-context` sentinel on Chromium.

ScraperAPI's free tier or Hobby: pick this if your scraper already works but keeps getting blocked. Hobby handles 20 concurrent requests and 100,000 credits, which covers most solo projects, and the trial lets you test your actual target sites before spending anything.

Business or Scaling: this is where country-level geotargeting unlocks and pay-as-you-go becomes available, so mid-size teams scraping internationally or running spiky workloads should start there rather than overbuying credits upfront.

One last check before checkout: confirm your target domains in the Domain Cost Estimator and multiply by your expected request volume, surcharges included. The headline credit number and your real request count can differ by a factor of 25 or more, and it's much cheaper to discover that on the trial than on an invoice. 👉 [Start your 7-day ScraperAPI trial](https://www.scraperapi.com/?fp_ref=coupons)
