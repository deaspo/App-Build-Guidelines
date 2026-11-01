# App Build Guidelines (agent rules)

Owner: Poly. Version: 2026-11-01.
Audience: any coding agent (Claude Code, Cowork, Codex, Copilot CLI, etc.).

## 0. How to apply this file

- [ ] Read section 1. Decide which later sections apply to the current project. Apply only those. State at the start of the task which sections you are applying.
- [ ] A section's "Applies when" line is the trigger. If the trigger is not met, the section is inert.
- [ ] Project-level instructions (CLAUDE.md, AGENTS.md, CLAUDE.local.md) override this file where they conflict.
- [ ] Never report a checklist item as done without evidence (command output, diff, URL response). "Skip (N/A)" with a one-line reason is acceptable; silent omission is not.

### Install locations (verified against Claude Code docs, 2026-11)

- Global, every project, no approval dialog: `~/.claude/rules/app-build-guidelines.md`
- Per project, committed: `.claude/rules/app-build-guidelines.md`
- Per project, personal, gitignored: import from `CLAUDE.local.md` with a line `@~/.claude/rules/app-build-guidelines.md`
- Other agents: reference or paste into `AGENTS.md` (Codex reads it; Claude Code reads it when no CLAUDE.md exists, or when "Project instructions" is set to `claude-md-and-agents-md`).
- Optional path scoping: split a section into its own file under `.claude/rules/` with frontmatter so it loads only when matching files are touched, e.g.

```markdown
---
paths:
  - "src/**/*.{ts,tsx,astro,html}"
  - "public/**"
---
# SEO / pre-release rules
...
```

## 1. Applicability matrix

| Section | Applies when |
|---|---|
| 2 Design-first (EGM / spec-kit) | New feature or change that involves a design decision; not for small, clear bug fixes |
| 3 Decision escalation (advisor) | Long multi-step task, recurring error, or before declaring done |
| 4 Pre-release web checklist | Any public web page/site about to ship or be re-indexed |
| 5 Free-tier / AI-credit abuse controls | App exposes AI calls (chat, generation) or any metered credit to self-serve signups |
| 6 Agent token economy | Repeated sessions in the same repo; cost matters |

## 2. Design-first workflow

Applies when: the task needs architectural or product decisions. Skip for one-line fixes.

Two tools cover this. Use EGM for design + review; use spec-kit when the project wants a full spec→plan→tasks→implement pipeline. Do not run both for the same feature unless the project's CLAUDE.md says so.

### 2a. Elephant-Goldfish (design review) — google/elephant-goldfish-model

Core rule: the agent with full context (elephant) drafts; fresh-context reviewers (goldfish) that see only the document and referenced source critique it; the human decides trade-offs.

- [ ] Brainstorm → write `concepts_brief.md` (agreed direction, alternatives rejected and why).
- [ ] Draft `design_draft.md` section by section; pause for human feedback after each section.
- [ ] Review with fresh subagents that receive only the draft + referenced files, never the conversation. Review for: hidden assumptions, unclear requirements, fit with existing code, implementation risk, test plan.
- [ ] Resolve: human decides which findings to address. Record decisions in the draft.
- [ ] Save approved spec under `docs/designs/`. Implement only after approval.
- [ ] If the session cannot spawn fresh subagents, say so and stop the review step; do not self-review and call it a goldfish review.
- [ ] Install (Claude Code): `claude plugin marketplace add google/elephant-goldfish-model` then `claude plugin install egm@egm`; commands are `/egm:egm-brainstorm`, `/egm:egm-draft`, `/egm:egm-review`, `/egm:egm-resolve`. Codex: `codex plugin marketplace add google/elephant-goldfish-model`, `codex plugin add egm@egm`.
- [ ] Without the plugin, emulate: write the two documents, then launch a `general-purpose` subagent with the prompt "You have no prior context. Review this design document and the files it references. List assumptions, ambiguities, risks, missing tests. Do not edit anything."

### 2b. Spec-kit (spec-driven implementation) — github/spec-kit

- [ ] Prerequisites: Python 3.11+, `uv`. Install: `uv tool install specify-cli`; `specify init <project> --integration <agent>`.
- [ ] Order: `/speckit-constitution` (once per project) → `/speckit-specify` → `/speckit-plan` → `/speckit-tasks` → `/speckit-implement` → `/speckit-converge` until it reports `Converged`.
- [ ] The constitution is the place for non-negotiables (stack, test policy, security). Put project-wide rules there, not in the spec.
- [ ] Spec describes what and why; plan describes how; tasks are small, ordered, testable. Reject a task list whose items cannot each be verified.
- [ ] Bundled extensions exist for bug fixing and idea assessment; use the bug-fix extension for non-trivial bugs instead of the full SDD flow.

## 3. Decision escalation

Applies when: a long task where plan quality decides the outcome; an error recurs; before declaring a task complete.

Reference: Claude Code advisor tool (experimental; Anthropic API only; not on Bedrock/Vertex/Foundry; needs feature-flag fetching, so `DISABLE_TELEMETRY` turns it off).

- [ ] In Claude Code: enable with `/advisor opus` (or `fable` where available), `"advisorModel": "opus"` in settings, or `claude --advisor opus`. Advisor must rank at or above the main model.
- [ ] Consult (advisor, or a stronger-model subagent where no advisor exists) at exactly these points: (1) before committing to an approach on a multi-file change, (2) when the same error recurs a third time, (3) before reporting done.
- [ ] Give the reviewer the full transcript or, for a subagent, the plan + diff + failing output. Ask for: flaws, a cheaper alternative, what would prove the task done.
- [ ] If the advice contradicts observed evidence (a step fails, file contents differ), surface the conflict; do not follow advice blindly.
- [ ] Cost rule: advisor tokens bill at the advisor model's rate; each call re-reads the whole transcript uncached. Do not request it on routine turns.
- [ ] Cowork / non-Claude-Code sessions: no advisor. Use a `fork` subagent (inherits context) for "check my plan" and a fresh `general-purpose` subagent for "check my result" so the completion check is not biased by the plan.

## 4. Pre-release web checklist

Applies when: a public website or web app is about to ship, migrate, or be re-indexed. Not for internal tools behind auth (then only 4e applies).

Execution rule: dispatch 3 read-only subagents (crawling / per-page / site-wide). Each lists failing URLs and the source files behind them. Fix everything yourself in one pass. Before any change, `curl -A "Googlebot"` every main page and record `<title>` plus the first 200 chars of body text; repeat after and present before/after. Skip items that do not apply, say which. End with the list of items that need Search Console to confirm (indexing status, sitemap acceptance, manual actions, Core Web Vitals field data).

### 4a. Crawlability

- [ ] `curl -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"` each main page: title and body text present without JS. Googlebot does render JS, but rendering is queued and many other crawlers (Bing, AI crawlers, link previews) do not; SSR or prerender if the HTML is an empty root.
- [ ] Non-existent paths return HTTP 404 (or 410), not 200 with "not found" text. Verify with `curl -o /dev/null -w "%{http_code}" <bad-url>`.
- [ ] Block admin/login/account pages with `noindex` (meta or `X-Robots-Tag`). Do not also disallow them in `robots.txt`: a page blocked in robots.txt cannot be crawled, so its `noindex` is never seen. Use robots.txt for crawl-budget waste (search result pages, infinite filters), `noindex` for "crawl but don't index".
- [ ] Navigation uses `<a href>` with real URLs. `onClick` router pushes without an `href` are invisible to crawlers.
- [ ] Staging/preview environments: `noindex` site-wide or auth-gated; confirm production does not inherit that header.
- [ ] Pagination and faceted navigation: canonical to the clean URL; `rel="next/prev"` is ignored by Google but harmless.

### 4b. Every page

- [ ] Unique `<title>` (≤ ~60 chars) and `<meta name="description">` (≤ ~155 chars) per page.
- [ ] `<link rel="canonical">` pointing at the page's own absolute URL, same host and scheme as the final redirect target. A site-wide canonical to the homepage is a defect.
- [ ] Core Web Vitals (lab, Lighthouse mobile): LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1. INP replaced FID in March 2024; a checklist that still says FID is stale.
- [ ] Images: WebP/AVIF, explicit `width` and `height` (prevents CLS), `alt` text, `loading="lazy"` below the fold, `fetchpriority="high"` on the LCP image.
- [ ] `<meta name="viewport" content="width=device-width, initial-scale=1">`, `<html lang>`, favicon.
- [ ] Open Graph + Twitter card meta on shareable pages (title, description, image ≥ 1200×630).
- [ ] Structured data: only types Google still renders. Do not add FAQPage: Google deprecated FAQ rich results effective 2026-05-07 and removed the documentation 2026-06-15 (Search Central changelog). Validate remaining schema with the Rich Results Test.
- [ ] Headings: one `<h1>`, logical order; no empty headings.
- [ ] Accessibility smoke test (axe or Lighthouse a11y ≥ 90): labels on inputs, contrast, focus order, no keyboard traps.

### 4c. Site-wide

- [ ] `sitemap.xml`: only canonical, indexable, 200 URLs; include `lastmod` only if it is real; ≤ 50 000 URLs / 50 MB per file, else a sitemap index. Reference it in `robots.txt` with `Sitemap: https://host/sitemap.xml`.
- [ ] Crawl the site (e.g. `wget --spider -r`, Screaming Frog, or a script); fix every broken internal link; every redirect is a single hop; http→https and www/non-www normalise in one redirect.
- [ ] Orphan pages (no inbound internal link) get linked from a relevant page or are removed from the sitemap.
- [ ] `robots.txt` returns 200 and does not block CSS/JS/images.
- [ ] If multilingual: `hreflang` reciprocal pairs including `x-default`.
- [ ] 404 page is useful (search, links) and returns 404.

### 4d. Performance and delivery

- [ ] Compression (br/gzip), cache headers on static assets with hashed filenames, CDN if traffic is public.
- [ ] Fonts: `font-display: swap` or optional, preload the primary font, subset.
- [ ] Third-party scripts: load `defer`/`async`, measure their share of INP; remove unused ones.

### 4e. Security and operations (applies to internal apps too)

- [ ] HTTPS only, HSTS, `Content-Security-Policy` at least report-only, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`.
- [ ] No secrets in the bundle: grep built assets for API keys; `.env` not served.
- [ ] Error tracking and uptime monitoring wired (Sentry or equivalent, a `/healthz`).
- [ ] Backups and restore tested once for any stateful service.
- [ ] Dependency audit (`npm audit`, `pip-audit`, `cabal`/`stack` advisories, etc.) with no critical findings.
- [ ] Rate limiting on auth and form endpoints (see §5 for AI endpoints).

## 5. Free-tier and AI-credit abuse controls

Applies when: new accounts receive anything that costs you per use (LLM tokens, image generation, SMS, storage) without paying first.

Threat model: signup farms (scripted, residential proxies, disposable or catch-all emails), credential stuffing of real accounts, and single users running automation against a "free" chat. The loss is unbounded unless caps exist on the server. Collected experiences (A–G) agree on: never give unlimited credits to new accounts; rate-limit by IP and device, not only by account; start low and raise with trust; monitor; have a kill switch.

### 5a. Non-negotiable server-side limits (implement first)

- [ ] Credits are stored and deducted server-side; the client never reports its own balance. Deduct before the model call (reserve), settle after; refund on provider failure; idempotency key per request.
- [ ] Hard caps in four layers: per request (max input tokens, max output tokens, max images), per account per day, per IP/ASN per day, global per hour. Any layer exceeded returns 429 with no partial work.
- [ ] Provider-side spend limits and alerts (OpenAI/Anthropic/etc. monthly cap, alert at 50/80 %). A code bug must not be able to exceed the provider cap.
- [ ] Kill switch: one feature flag disables free-tier AI calls instantly without a deploy. Test it before launch.
- [ ] Concurrency cap per account (e.g. 1–2 in-flight requests on free tier); stop streaming when the balance hits zero.
- [ ] Free tier uses the cheapest adequate model; expensive models require payment.

### 5b. Signup friction (cheap for humans, expensive for farms)

- [ ] Bot challenge at signup and at first credit-consuming action: Cloudflare Turnstile (managed mode; always verify the token server-side via siteverify) or equivalent. CAPTCHA alone is defeated by solver services; treat it as one layer.
- [ ] Rate-limit signups per IP, per /24 or ASN, per device fingerprint, per email domain. Flag spikes (e.g. > 3× the 7-day median per hour) and auto-reduce new-account credits while a spike is open.
- [ ] Email: block disposable domains (maintained list), require verification, normalise Gmail dots and `+` aliases before uniqueness checks, treat catch-all domains as lower trust.
- [ ] Prefer OAuth sign-in (Google/GitHub/Apple) for free credits: account age and existing reputation raise the farm cost.
- [ ] Device/browser fingerprint (FingerprintJS/open-source alternatives) stored at signup; count accounts per fingerprint.
- [ ] Require JS to complete onboarding (blocks naive curl bots; headless browsers still pass, so this is a cheap layer, not a control).
- [ ] Deny or reduce free credits for datacenter/VPN/Tor IPs (ASN lookup; Stripe Radar exposes `is_anonymous_ip` for payments). Allow paid users regardless.
- [ ] Honeypot form field and minimum time-on-form checks.
- [ ] Optional multi-step onboarding that cannot be completed in a single POST (script farms often abandon flows that exceed their timeout). Unverified folk claim; low cost, keep as optional.
- [ ] Phone verification only if the product can absorb SMS pumping fraud (attackers trigger SMS to premium-rate ranges). If used: restrict countries, rate-limit sends, use a provider with fraud guard, and never send SMS before Turnstile passes.
- [ ] Gamified redemption (Experience G: a game must be played to redeem credits): raises scripting cost once, until someone automates it. Acceptable as novelty friction; not a substitute for 5a.

### 5c. Card-on-file gating (Experience C)

- [ ] Require a payment method via Stripe SetupIntent (no charge, or a €0/1 auth) before unlocking the larger free allotment. Store `payment_method.card.fingerprint` and reject when the fingerprint is already attached to ≥ 2 accounts (your own DB check works without any Radar tier).
- [ ] Stripe Radar attributes usable in custom rules: `card_fingerprint`, `card_count_for_email_*`, `card_count_for_ip_address_*`, `total_charges_per_card_number_daily`, `is_disposable_email`, `is_anonymous_ip`, `card_funding` (block `prepaid` for free credits). Custom rules require a Radar plan that supports them (Radar for Fraud Teams); confirm in the dashboard.
- [ ] Request 3DS for new cards (`Request 3D Secure if :is_new_card_on_customer:`).

### 5d. Trust tiers

- [ ] Tier 0 (new, unverified): minimal credits, cheapest model, low concurrency.
- [ ] Tier 1 (email verified + Turnstile + clean IP): small daily allowance.
- [ ] Tier 2 (card on file or OAuth with aged account, N days of normal use): standard free allowance.
- [ ] Paid: product limits only.
- [ ] Demotion is automatic on any abuse signal; promotion is time + behaviour based, never instant.

### 5e. Detection and response

- [ ] Log per request: account, IP, ASN, UA, fingerprint, TLS fingerprint (JA3/JA4 if the edge exposes it), prompt hash, tokens, cost. Keep 90 days.
- [ ] Alerts: cost per hour vs baseline; accounts per IP/fingerprint; identical prompt hashes across accounts (the "red fox images non-stop" pattern); signups from one ASN; ratio of signups to activated users.
- [ ] Correlate IPs, email patterns, signup timing after any incident and turn findings into rules.
- [ ] Response playbook: flip kill switch → revoke credits on flagged cohort → block ASN/fingerprint → review false positives within 24 h. Pre-write the user-facing message.
- [ ] Prompt/output safety on the chat surface: moderation on inputs and outputs, system-prompt isolation, no tool access from free tier, max conversation length.
- [ ] Cache identical generations (same prompt hash + params) and serve the cached result at zero cost.

## 6. Agent token economy — microsoft/TokenEconomizer

Applies when: the same repository is worked on across many agent sessions and cost matters. Optional tool, not a rule.

- [ ] What it does: learns across sessions in one repo. Tool pruning (on by default: drops tools unused across 10 prompts over ≥ 2 sessions or 5 sessions; one command restores). Loop guard (on: a judge model redirects a turn it is ≥ 70 % confident is stuck, once). Micro-skills (off by default: injects repo facts confirmed by 3 sessions).
- [ ] Reported savings: 18.4–30.9 % mean cost on 277 SWE-Bench tasks (authors' own benchmark; one combination lost 1.0 Pass@1). Expect less on short or one-off sessions, since learning needs repeated sessions.
- [ ] Version pinning: tested on Claude Code 2.1.300–2.1.310 and Codex 0.153.1–0.164.0 only. Check `claude --version` before installing; after any Claude Code update run `/tokeneconomizer:toolprune refresh-catalog`.
- [ ] Install (Claude Code): clone repo; `claude plugin marketplace add "$PWD/TokenEconomizer"`; `claude plugin install tokeneconomizer@tokeneconomizer`; `/tokeneconomizer:status`.
- [ ] Data lives in `~/.tokeneconomizer`; judge/proposer calls add cost that the reported numbers include.
- [ ] Independent of the plugin: keep CLAUDE.md under the context budget, use path-scoped rules, `/compact` at milestones, delegate large reads to subagents, and avoid re-reading files already in context.

## 7. Provenance and corrections to the collected tips

- FAQ schema: claim "Google stopped showing FAQ rich results in May 2026" is correct; deprecation notice dated 2026-05-08 states the feature stopped appearing from 2026-05-07; docs removed 2026-06-15.
- "Only use robots.txt and noindex to block pages": incomplete; the two conflict when combined on one URL (see 4a).
- Lighthouse thresholds: LCP 2.5 s / CLS 0.1 correct; INP 200 ms was missing.
- Spec-kit: active (v1.0, 2026); command names are `/speckit-*`; `clarify`/`analyze`/`checklist` are documented topics, not top-level commands in the current README; `/speckit-converge` is new.
- Advisor: experimental, Anthropic API only, billed at advisor model rates, transcript re-read uncached per call.
- EGM: real Google repo (Apache-2.0, "not an officially supported Google product"); review step aborts without fresh-subagent support.
- TokenEconomizer: real Microsoft repo; narrow supported version ranges; savings are self-reported.
- Experience C (Stripe card limit): feasible via `card_fingerprint`; custom Radar rules are plan-gated, a DB-side check is not.
- Experience D (phone numbers): valid but introduces SMS pumping fraud exposure.
- Experience F (timeout, require JS, TLS fingerprinting): directionally sound; the timeout tactic is anecdotal.
- Experience G (game to redeem): friction, not a control.

Sources: https://github.com/github/spec-kit · https://code.claude.com/docs/en/advisor · https://github.com/google/elephant-goldfish-model · https://github.com/microsoft/TokenEconomizer · https://developers.google.com/search/docs/appearance/structured-data/faqpage · https://docs.stripe.com/radar/rules · https://docs.stripe.com/radar/rules/supported-attributes · https://developers.cloudflare.com/turnstile/ · https://code.claude.com/docs/en/memory
