# web scraping rate limit: how to avoid 429 errors and IP bans without slowing your crawl to a crawl

Most scrapers don't get blocked because of a clever fingerprint check. They get blocked because they hit a server too hard, too fast, from too few IPs. That's the core of the web scraping rate limit problem, and it's fixable with two approaches: throttle your own requests properly, or offload IP rotation and concurrency to a scraping API like ScraperAPI. This guide covers both, plus the actual pricing mechanics most reviews skip — including which domains quietly burn 25–75 credits per request.

> **Quick picks**
> - Just testing? The free plan gives you 1,000 credits/month at 5 concurrent connections, plus a 7-day trial with 5,000 requests — no credit card needed for the trial. 👉 [Sign up and get your 5,000 free trial credits](https://www.scraperapi.com/?fp_ref=coupons)
> - Small-to-mid projects hitting 429s: Hobby at $49/month (100K credits, 20 threads) or Startup at $149/month (1M credits, 50 threads).
> - Annual billing takes 10% off every plan, and there's a 7-day no-questions-asked refund if it doesn't work out.

## What a rate limit actually is (and what it isn't)

A rate limit is a server-side cap on how often one client can make requests in a given time window. If a site allows 10 requests per minute and your scraper sends the 11th, it gets blocked — usually with an HTTP 429 Too Many Requests response. The count is typically keyed to your IP address, your API key, or your account.

One thing worth understanding before you write any countermeasure code: rate limits exist to protect server performance, not to hunt down bots for sport. A flood of rapid requests looks like spam, abuse, or even a DDoS attack from the server's perspective, so the site throttles first and bans later. This matters practically, because it means a well-paced crawler often coexists fine with rate limiting — you only trigger the heavy machinery when you're careless or impatient.

Modern CDNs like Cloudflare and AWS don't just count raw requests either. They analyze TLS fingerprints, headers, and whether your traffic behaves like a real browser. So even a "slow" scraper with perfect 1.5-second intervals can stand out, because humans don't request things with machine-precision timing.

## Soft limits vs. hard limits: know which one hit you

The response code tells you which wall you ran into. Soft limits (throttling) let you exceed the baseline briefly — the server slows responses, injects artificial delays, or returns a 429 with a `Retry-After` header, like Cloudflare's Error 1015. Back off and you can keep going. Hard limits lock you out entirely: an API allowing 1,000 requests per hour will reject everything for the rest of that hour once you cross the line, no warnings, no slowdowns.

From there it escalates. Frequent violations turn into temporary bans of minutes to hours, where every request fails with 403 Forbidden — and if you keep triggering it, the timeout window can extend automatically. The end state is a permanent ban: your IP, account, or entire proxy subnet gets blacklisted, and every request is blocked no matter how slow or well-formed. At that point the only way out is a new IP, rotated accounts, or a more capable scraping setup.

| Signal | What it means | What to do |
| --- | --- | --- |
| HTTP 429 | You hit the rate limit | Back off, honor `Retry-After`, retry later |
| Cloudflare Error 1015 | Rate limited by Cloudflare | Same as 429 — it usually rides on a 429 status |
| HTTP 403 | Likely a ban; your IP or session is flagged | Slow down, rotate IP, check headers |
| HTTP 503 | Temporary rejection or bot challenge | Common with Cloudflare; reduce pressure |
| CAPTCHA page (HTTP 200) | Soft block after crossing a request threshold | Slow down or solve the challenge |
| Sudden timeouts / blank pages | Silent block or temporary ban | Pause, then resume at a much lower rate |

The single most common mistake here: ignoring `Retry-After`. That header is the server telling you exactly how long to wait, and blowing past it is the fastest route to a longer ban.

## How to throttle your own scraper properly

Throttling and concurrency are two different levers, and people constantly mix them up. **Delay** controls the gap between requests on a single worker. **Concurrency** controls how many workers run in parallel. You need both — a 1-second delay means nothing if you spin up 500 parallel workers each obeying it.

Concretely: with a concurrency of 8 and a 0.5-second minimum gap per request, you cap out around 16 requests per second with at most 8 connections open. Tune down for fragile sites, up for CDN-backed giants.

A few practices that actually move the needle:

1. **Add random jitter to your delays.** A fixed 1.5-second interval is mechanically regular and easy for behavior detectors to flag. Randomize around your base delay instead.
2. **Use adaptive backoff on failures.** On success, ease your delay back down; on a 429, double it, with a cap (60 seconds is a common ceiling). Add jitter to the backoff too, or your parallel workers will all retry simultaneously and trigger a retry storm.
3. **Honor `Retry-After`** whenever the server sends it.
4. **Consider a token bucket** for anything beyond a toy script: tokens refill at a fixed rate, each request consumes one, and the bucket absorbs short bursts while keeping your long-run average constant.
5. **Key your limits per domain,** not globally — a separate rate bucket per hostname keeps a multi-site crawl from pouring everything into one target.

If you use Scrapy, its `CONCURRENT_REQUESTS` and `DOWNLOAD_DELAY` settings map directly onto these two levers, and the built-in AutoThrottle extension adjusts delays automatically based on observed response times. In Go, an errgroup with a buffered channel does the job; in Node.js, a p-limit-style pool works fine.

For a sense of scale: a human browsing generates maybe one request every few seconds. A single IP firing 200 requests per second gets flagged trivially. And throttling isn't only about avoiding bans — an overloaded origin server returns timeouts, partial HTML, and 5xx errors, so a steady moderate rate gets you cleaner data than a burst that trips 503s.

Self-throttling works until it doesn't. If your target sits behind Cloudflare or Datadome, or you need thousands of pages per hour, per-IP politeness only gets you so far — you eventually need more IPs, which brings us to the API route.

## When self-throttling stops being enough: the ScraperAPI route

A scraping API solves the rate limit problem structurally: every request goes out through a rotating pool of residential and mobile IPs, so no single address accumulates enough requests to trip a per-IP threshold. ScraperAPI handles this server-side, along with CAPTCHA solving, JS rendering, automatic retries, and advanced bypassing for protected targets — features included in every paid plan, not gated behind an enterprise tier.

The concurrency model is straightforward. Each plan comes with a fixed number of concurrent threads (5 on the free plan, up to 500 on Advanced). If you exceed your plan's concurrency, you get a 429 asking you to bring concurrency back within limits and retry. Here's the part worth knowing: **there's no extra charge for going over** — the request is simply turned away rather than queued. Your client just needs to handle that 429 the same way it would handle a target site's rate limit: back off and retry.

Rendered requests have their own pacing rule on top of that. The default burst limit is 10 rendered requests per second, which controls how many rendering jobs you can *start* each second. The math matters: if each rendered request takes about 25 seconds to complete and you start 10 per second, you can have up to 250 rendering jobs in flight simultaneously — so plan your concurrency headroom around job duration, not just the burst number.

One honest caveat: with a rotating proxy API you lose direct control over per-request pacing, since the service manages the IP pool and throttling logic. And the concurrency caps are real — a dev.to review of ScraperAPI notes that on lower tiers you'll hit the ceiling if you need to blast through a large dataset quickly. If you need more parallelism than your plan allows, you can contact support to have the limit raised, or move up a tier.

## Full plan comparison (verified from the official pricing page)

| Plan | Monthly price | Yearly (per month) | Credits/month | Concurrent threads | Get it |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 connections | [Start free](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom quote | — | 22,000,000+ | 500+ | [Contact sales](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

The documentation also lists a Student plan among the paid tiers, though the public pricing page doesn't show its price — check the dashboard if that applies to you.

Geographic targeting and analytics differ by tier in ways that matter for some projects: Hobby and Startup are limited to US and EU geotargeting, while Business and above get country-level targeting. Hobby and Startup analytics dashboards only retain the last 30 days; Business and up get unlimited history.

Every paid plan includes the same feature set — JS rendering, premium residential and mobile IPs, JSON auto-parsing, custom headers and sessions, CAPTCHA handling, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee.

## The credit math nobody mentions until the bill arrives

Here's the biggest gap in most rate-limit guides: "credits" are not requests. A standard page costs 1 credit, but harder targets cost multiples of that, and rendering multiplies everything again.

- Standard page: 1 credit
- Amazon: 5 credits
- Google and Bing (including subdomains): 25 credits
- LinkedIn: 30 credits
- Sites behind Cloudflare, Datadome, or PerimeterX: +10 credits per request when the bypass succeeds
- JS rendering: ×10 (so a rendered protected page costs 75 credits)

Run the numbers and the free tier looks different. Your 1,000 monthly credits cover 1,000 plain pages — or about 13 rendered pages behind bot protection. On Professional's 10.5M credits, that same rendered protected page works out to roughly 140,000 pages per month. Same API, wildly different effective capacity, depending entirely on what you're scraping and which parameters you send.

Two tools keep this from becoming a surprise: the Domain Cost Estimator in your dashboard shows the exact credit cost for any URL before you commit, and a `max_cost` parameter caps what any single request can consume. If your target list is heavy on Google SERPs or LinkedIn, budget in multiplied credits, not raw request counts.

Before committing to a tier, it's worth checking what your actual targets cost: 👉 [Check current pricing and estimate your credit costs](https://www.scraperapi.com/pricing?fp_ref=coupons)

## What happens when you actually hit your limits

Running out of credits doesn't kill your project mid-month by default — scraping just pauses until credits renew. But each tier has options, and they differ in ways that affect which plan you should pick:

- **Auto-upgrade** (Hobby through Professional, monthly plans): opt-in, and when credits run out your account moves to the next tier automatically, usually at a better price per credit.
- **Pay-As-You-Go** (Scaling, Professional, Advanced, Enterprise): keep scraping past your limit at a fixed per-credit rate without changing plans. You can set a monthly spending cap, and service will not exceed it unless you change the setting yourself. Note the wording in the official docs: it's a spending *limit*, not an amount you get pre-charged.
- **Annual plans** that exhaust their credits early can renew at the next billing date for a fresh batch, or switch to a monthly plan that fits the new volume.
- Enterprise accounts get a mid-cycle pop-up option at a fixed rate, or a conversation with sales about resizing.

One billing detail that's easy to miss: ScraperAPI charges per request, not per bandwidth. There's no bandwidth-based pricing anywhere in the lineup. Pricing models vary across the scraping API market — at least one competitor advertises billing only for successful requests, per its own documentation — so if that distinction matters to your budget, check each provider's terms before committing.

If you're unsure whether the volume justifies a paid tier, the trial settles it cheaply: 👉 [Start the 7-day trial with 5,000 free requests](https://www.scraperapi.com/?fp_ref=coupons)

## Risk-free ways to find out if it fits

The free side of things is more generous than most: a permanent free plan with 1,000 credits per month at 5 concurrent connections, plus a 7-day window after signup with 5,000 free requests for larger-scale testing. The trial requires no credit card.

On the paid side, the refund policy is explicit: "We offer a 7-day no questions asked refund policy." If the service doesn't work for your targets, support processes the refund — you're not locked into an annual commitment on faith. Canceling itself is free anytime from the dashboard.

That combination (free trial first, refundable paid month second) means the real risk of testing ScraperAPI against your specific target sites is roughly zero. The thing to actually verify during the trial is the credit cost of your hardest domain, using the Domain Cost Estimator — that number, more than the sticker price, determines which plan makes sense.

## Who should pick which plan

A few decision rules based on the verified numbers:

- **You scrape a handful of sites occasionally, all standard pages.** Stay free, or don't pay anyone. 1,000 credits/month covers light personal projects, and self-throttling (Section 3) handles the rate limits.
- **You're hitting 429s on protected sites regularly, volume under ~800K standard pages/month.** Hobby ($49) or Startup ($149). Both are US/EU geotargeting only, which is fine for most e-commerce and SERP work; if you need country-level targeting, jump to Business.
- **Production pipelines, multiple targets, need country targeting and full analytics.** Business ($299) or Scaling ($475). Scaling is the plan the pricing page marks as most popular, and it's the first tier with Pay-As-You-Go built in, which removes the cliff-edge feeling of hitting a monthly cap.
- **Continuous high-frequency crawls or multi-source data pipelines.** Professional ($975) or Advanced ($1,975). These are the only monthly plans with Pay-As-You-Go included from the start.
- **22M+ credits or custom needs.** Enterprise quote territory.

Skip the API entirely if your targets are small, unprotected sites and your volume is low — a polite scraper with jittered delays and per-domain limits does the job, and no subscription beats $0.

## FAQ

**Why do I keep getting 429 errors even at low request rates?**
Either your per-IP count crossed the site's window threshold, or the concurrency limit on your own plan (at ScraperAPI or any API) is the source. Check the response: a 429 from the target site means back off and honor `Retry-After`; a 429 from ScraperAPI means reduce your parallel threads below your plan's limit and retry — the rejected requests cost nothing.

**What happens when I exceed my ScraperAPI concurrency limit?**
You get a 429 response asking you to bring concurrency back within your plan's limit and try again. The over-limit request is turned away rather than queued, and there's no charge for it. Retry once your in-flight count drops below the cap.

**Can I increase my concurrency without upgrading?**
Yes. Official docs say to contact support to enquire about raising the concurrency limit on your account — though at some point the per-tier pricing makes an upgrade the cheaper route, since higher tiers come with a better price per credit.

**What happens if I exceed my monthly credits?**
By default, scraping pauses until credits renew. Depending on your plan, you can enable auto-upgrade (Hobby–Professional monthly), use Pay-As-You-Go with a spending cap (Scaling and up), or renew an annual plan early for fresh credits.

## Before you commit

Match the plan to your hardest domain, not your average one. If most of your volume is standard pages but 10% is LinkedIn or Google, that 10% can consume the majority of your credit budget — model it with the Domain Cost Estimator before choosing between Startup and Business. The 7-day trial with 5,000 free requests is enough to measure those real costs, so you'll know whether Hobby's 100K credits is comfortable or tight before any money changes hands. If the numbers look good, 👉 [compare all plans and start your free trial on the pricing page](https://www.scraperapi.com/pricing?fp_ref=coupons) — and pick annual billing if you already know you'll keep the subscription.
