# playwright rotating proxy: how to rotate by browser context in Node and Python without 407 errors or burned bandwidth

The first request comes back `407 Proxy Authentication Required`. You check the proxy URL, it looks fine, it's the exact string the provider's dashboard gave you. You paste it into `curl` and it works. It only breaks inside Playwright.

That failure has a specific cause, and it has nothing to do with rotation. Once you understand why it happens, the rest of a working playwright rotating proxy setup is mostly about deciding *what* you're rotating and *how often* — which is a decision about your proxy endpoint, not about Playwright.

## The proxy URL you copied won't authenticate

Playwright's `proxy` object takes four fields: `server`, `username`, `password`, `bypass`. Chromium ignores credentials embedded in the `server` URL. It doesn't warn you, it doesn't throw, it just drops them and the proxy answers with a 407.

js
// Silent failure: Chromium drops the inline credentials
const context = await browser.newContext({
  proxy: { server: 'http://user:pass@gw.example.com:8000' },
});


js
// What actually authenticates
const context = await browser.newContext({
  proxy: {
    server: 'http://gw.example.com:8000',
    username: process.env.PROXY_USER,
    password: process.env.PROXY_PASS,
  },
});


Keep the credentials out of the code. Templating a password that contains `@` or `#` into a URL string is its own afternoon of debugging.

## Rotation isn't a Playwright feature

This is the part that gets buried. Playwright has no rotation setting. Geo targeting, session length, whether the IP changes per request or holds for ten minutes — all of that is decided by the proxy endpoint you point the browser at. Playwright's only job is to route traffic there.

So "playwright rotating proxy" really means two separate questions:

1. Is my proxy endpoint a rotating one, and in what mode?
2. Does my script use that rotation the way a real visitor would?

Skipping the second question is the most common mistake.

## One context, one identity

A `BrowserContext` gets its own cookies, storage, cache and — when you set it there — its own proxy. Pages inside a context share that exit IP.

The tempting pattern is to flip the IP between page loads inside one context. Don't. A session whose cookies say "the visitor from Germany" and whose next request arrives from Japan isn't two anonymous visitors; it's one identity contradicting itself. That's easier for a bot-detection system to catch than a single mediocre IP would be.

The pattern that holds up:

- **One context = one sticky identity.** All page loads in it exit from one IP, like a normal user.
- **Rotate by creating a new context.** New context, new IP, fresh cookie jar.
- **Close the old context** when its work is done.

Per-request rotation is correct in exactly one narrow case: stateless HTTP calls with no login, no cart, no cookies to preserve. The moment a multi-step flow is involved — sign-in, checkout, a form wizard — you're back to one session, one IP.

## The Chromium gotcha that eats your per-context proxies

Here's the one that costs people an afternoon: in Chromium, per-context proxies only take effect if the browser was launched with a proxy option already present. Launch with no proxy at all and every `newContext({ proxy })` is silently ignored — your traffic leaves from the host IP and nothing tells you.

The fix is a placeholder:

python
from playwright.sync_api import sync_playwright
import os

with sync_playwright() as p:
    # Placeholder at launch, real proxies per context
    browser = p.chromium.launch(proxy={"server": "http://per-context"})

    for i in range(5):
        context = browser.new_context(
            proxy={
                "server": "http://gw.example.com:8000",
                "username": os.environ["PROXY_USER"],
                "password": os.environ["PROXY_PASS"],
            },
            locale="en-US",
            timezone_id="America/New_York",
        )
        page = context.new_page()
        page.goto("https://httpbin.org/ip")
        print(i, page.inner_text("body"))
        context.close()

    browser.close()


Firefox and WebKit don't strictly need the placeholder, but setting it is harmless and keeps your code browser-agnostic. It's the kind of bug that only shows up the day you switch engines.

Two more limits worth knowing before you pick a protocol:

- **Authenticated SOCKS5 doesn't work in Playwright's Chromium.** No error, no authentication — the requests just don't arrive. If you need username/password auth, use the HTTP endpoint. 9Proxy publishes both HTTP/HTTPS and SOCKS5, so the HTTP gateway is the one to wire into Playwright.
- **Proxy settings don't cover WebRTC.** A correctly proxied browser can still expose the host address over WebRTC. Test for leaks before you trust the setup.

## What "rotating" means on the provider side

Residential proxy providers hand you rotation in one of two shapes, and they're not interchangeable for a browser.

**Rotation encoded in the credentials.** The gateway host stays fixed and the username carries the targeting — country, city, ASN, a sticky-session ID, a TTL. Drop the session ID and every new connection gets a new IP; add it and the IP holds for the TTL. This gives you per-context sticky behaviour with full control from inside the script.

**Rotation as a dashboard setting.** You choose rotating mode or sticky mode in the account, and the session duration is configured there. 9Proxy works this way for its bandwidth-based product: rotating mode switches to a new IP automatically, sticky mode holds the IP until the configured session time runs out. You authenticate with username/password or an IP whitelist, and there's no desktop app in the path.

For a Playwright workload on a Linux box or in CI, the second shape is the one that connects without extra infrastructure.

## Which 9Proxy model fits a browser — and which one surprises people

9Proxy sells two residential models, and the difference matters more for Playwright than the headline price.

|  | Residential by GB | Residential by IPs |
| --- | --- | --- |
| Billing | per GB consumed | per IP, fixed package |
| Bandwidth | limited by purchased GB | unlimited while the IP is live |
| Rotation | set in dashboard, rotating or sticky | via the desktop app's Auto Rotation Proxy, on a custom interval |
| Auth | username/password or IP whitelist | requires the 9Proxy desktop app with local port forwarding |
| Validity | 180 days (no expiry on Enterprise GB) | unused IPs never expire |
| Works from a headless server | yes, straight from the dashboard | needs the desktop app running |

For most Playwright work, the GB-based model is the one that plugs in directly: one HTTP endpoint, credentials, done. The IP-based model is attractive for a different reason — bandwidth is unlimited, so a browser downloading images and fonts all day costs the same as one downloading nothing. But it routes through the desktop app, and every automatic rotation consumes a new IP from your pool.

> If you're running Playwright headless on a server, the IP-based model means the desktop app has to be in the path somewhere. Check that against your deployment before you buy on price alone.

There's a second wrinkle, and it's the one worth reading reviews about. The IP-based packages are residential IPs, not a rotating endpoint. If what you wanted was "one credentials string and a new IP every request," the GB-based product is the match. More than one buyer has picked up the cheapest IP package expecting an endless rotation pool and been disappointed — and the balance isn't refundable.

## All current 9Proxy plans and prices

Every plan family published on the pricing page is below. Prices for IP-based packages and bundles were adjusted on 1 June 2026; GB-based pricing was left unchanged at that point.

| Plan | Family | Price | Unit price | Validity | Buy |
| --- | --- | --- | --- | --- | --- |
| 5 GB | GB-based | $15 | $3.00/GB | 180 days | [Buy 5 GB](https://bit.ly/9-Proxy) |
| 50 GB + 5 GB bonus | GB-based | $105 | $2.10/GB | 180 days | [Buy 55 GB](https://bit.ly/9-Proxy) |
| 100 GB | GB-based | $150 | $1.50/GB | 180 days | [Buy 100 GB](https://bit.ly/9-Proxy) |
| 200 GB | GB-based | $200 | $1.00/GB | 180 days | [Buy 200 GB](https://bit.ly/9-Proxy) |
| 1,000 GB | GB-based | $800 | $0.80/GB | 180 days | [Buy 1,000 GB](https://bit.ly/9-Proxy) |
| 2,000 GB | GB-based | $1,500 | $0.75/GB | 180 days | [Buy 2,000 GB](https://bit.ly/9-Proxy) |
| 3,000 GB | Enterprise GB | $2,160 | $0.72/GB | no expiry | [Buy 3,000 GB](https://bit.ly/9-Proxy) |
| 6,000 GB | Enterprise GB | $4,200 | $0.70/GB | no expiry | [Buy 6,000 GB](https://bit.ly/9-Proxy) |
| 10,000 GB | Enterprise GB | $6,800 | $0.68/GB | no expiry | [Buy 10,000 GB](https://bit.ly/9-Proxy) |
| 100 IPs | IP-based | $24 | $0.24/IP | IPs don't expire | [Buy 100 IPs](https://bit.ly/9-Proxy) |
| 500 IPs | IP-based | $72 | $0.144/IP | IPs don't expire | [Buy 500 IPs](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus | IP-based | $126 | $0.084/IP | IPs don't expire | [Buy 1,500 IPs](https://bit.ly/9-Proxy) |
| 2,500 IPs | IP-based | $210 | $0.084/IP | IPs don't expire | [Buy 2,500 IPs](https://bit.ly/9-Proxy) |
| 5,000 IPs | IP-based | $360 | $0.072/IP | IPs don't expire | [Buy 5,000 IPs](https://bit.ly/9-Proxy) |
| 15,000 IPs | IP-based | $720 | $0.048/IP | IPs don't expire | [Buy 15,000 IPs](https://bit.ly/9-Proxy) |
| 25,000 IPs | IP-based | $863 | $0.035/IP | IPs don't expire | [Buy 25,000 IPs](https://bit.ly/9-Proxy) |
| 50,000 IPs | IP-based | $1,438 | $0.029/IP | IPs don't expire | [Buy 50,000 IPs](https://bit.ly/9-Proxy) |
| 100,000 IPs | Business IP | $2,300 | $0.023/IP | IPs don't expire | [Buy 100,000 IPs](https://bit.ly/9-Proxy) |
| 200,000 IPs | Business IP | $4,140 | $0.021/IP | IPs don't expire | [Buy 200,000 IPs](https://bit.ly/9-Proxy) |
| 500,000 IPs | Business IP | $8,625 | $0.018/IP | IPs don't expire | [Buy 500,000 IPs](https://bit.ly/9-Proxy) |
| 100 IPs + 5 GB | Bundle | $30 | — | 180 days on the traffic | [Buy the starter bundle](https://bit.ly/9-Proxy) |
| 1,500 IPs + 50 GB | Bundle | $180 | — | 180 days on the traffic | [Buy the popular bundle](https://bit.ly/9-Proxy) |
| 5,000 IPs + 500 GB | Bundle | $720 | — | 180 days on the traffic | [Buy the Pro bundle](https://bit.ly/9-Proxy) |

The reported pool behind these plans is 20M+ residential IPs across 90+ countries, with targeting down to country, city, ZIP code and ISP.

## What a Playwright job actually costs on per-GB pricing

This is where browser automation differs from an HTTP scraper, and it's the number that decides your plan.

A headless browser loads everything on the page by default — images, fonts, video, tracking scripts — most of which your scrape never reads. Strip the heavy resource types and per-page consumption typically drops by more than half.

Rough arithmetic, with the assumptions stated: if a page view costs about 1.5 MB after blocking images and fonts, 20,000 page views is roughly 30 GB. On the 100 GB package at $1.50/GB, that's $45 for the batch. Leave images enabled and the same run might cost two or three times as much. Neither figure is a benchmark — measure your own pages with `httpbin`-style byte counting or the provider's usage meter before you commit to a volume tier.

Blocking resources is the highest-leverage bandwidth change you can make:

js
await context.route('**/*', (route) => {
  const type = route.request().resourceType();
  if (['image', 'media', 'font'].includes(type)) return route.abort();
  return route.continue();
});


Keep `document`, `script`, `xhr` and `fetch` — those carry the data and the JavaScript that renders it. If a target lazy-loads content behind images, test that your scrape still works before shipping this.

## Match the fingerprint to the IP

A German residential IP paired with an `en-US` locale and a New York timezone is a contradiction that anti-bot systems flag on sight. Playwright lets you align all three per context:

js
const context = await browser.newContext({
  proxy: { server: '...', username: '...', password: '...' },
  locale: 'de-DE',
  timezoneId: 'Europe/Berlin',
});


One caveat specific to per-context proxies: if your tooling resolved the timezone once at browser launch, every context afterwards inherits that launch-time zone regardless of where its own proxy exits. Set `timezoneId` explicitly per context rather than relying on automatic resolution.

## The errors you'll actually see, and what each one means

| Symptom | Cause | Fix |
| --- | --- | --- |
| `net::ERR_TUNNEL_CONNECTION_FAILED` | Auth failed, or a targeting flag is misspelled | Fix the `username`/`password` fields; retrying won't help |
| `net::ERR_PROXY_CONNECTION_FAILED` | Gateway unreachable, or no IP matched an over-tight filter | Check connectivity, then loosen the filter |
| Page loads but shows a block or CAPTCHA | The target is refusing you, not the proxy | New context, new IP, slower pacing, align the fingerprint |
| Proxies work in `curl` but not in Playwright | Inline credentials, or missing launch placeholder | Split the credentials out; add the placeholder |
| Every context exits from the same IP | Per-context proxy silently ignored in Chromium | Set any proxy at `launch()` |

Note that Playwright surfaces proxy problems as navigation errors, not HTTP status codes. A 407 from the proxy layer usually looks like a tunnel failure.

## A retry that gets a genuinely different route

Retrying inside the same context reuses the same IP, so a failed request fails the same way. Build the retry so it produces a fresh context:

js
async function fetchWithRetry(browser, url, attempts = 3) {
  for (let i = 0; i < attempts; i++) {
    const context = await browser.newContext({ proxy: nextProxy() });
    try {
      const page = await context.newPage();
      await page.goto(url, { timeout: 30000, waitUntil: 'domcontentloaded' });
      const html = await page.content();
      await context.close();
      return html;
    } catch (err) {
      await context.close();
      if (i === attempts - 1) throw err;
    }
  }
}


Two practical notes. Close the context in a `finally` block in production — leaking contexts leaks IPs and, on a per-GB plan, keeps billing. And run a small pool of contexts concurrently rather than fifty at once; a burst of fresh IPs hitting one target hard trips behavioural detection even when every IP is clean.

## Before you commit a budget

Feedback on 9Proxy is mixed but leans positive — the Trustpilot page for the brand sits around 4.6/5, and the recurring themes in published reviews are price, connection speed, and compatibility with antidetect browsers. The recurring complaint is refunds: at least one reviewer bought the smallest IP-based package expecting rotating proxies, found it was a residential IP pool, and couldn't get the balance refunded. Read the plan descriptions before checkout, not after.

Separately, a competing vendor's comparison page has claimed service outages during 2026. Treat that source with the scepticism its commercial interest deserves, and treat the underlying point as sound regardless: never park your entire budget in a single provider's wallet. Start with the smallest package that covers a real test run, verify your exit IPs and success rates against your actual targets, and scale once the numbers hold.

You can 👉 [open the 9Proxy sign-up and start with the 5 GB package](https://bit.ly/9-Proxy) — that's $15 for 180 days of validity, which is enough traffic to run a real Playwright job and find out whether the rotation model fits your script. If your workload is bandwidth-hungry rather than request-hungry, 👉 [compare the IP-based packages](https://bit.ly/9-Proxy) instead, where unlimited bandwidth per IP changes the maths entirely.

## The short version

Fix the credentials first — separate fields, never inline in the server URL. Add the launch-time placeholder so Chromium honours per-context proxies. Give each context one sticky IP and rotate by opening new contexts, not by swapping IPs mid-session. Align timezone and locale with the exit country. Block images and fonts. Then pick the pricing model that matches what your job actually burns: requests or gigabytes.
