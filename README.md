<h1 align="center">production-audit</h1>

<p align="center">
  <strong>A Claude Code skill that audits a shipped repo for the production-readiness gaps ~70% of AI-coded projects miss.</strong>
</p>

<p align="center">
  <a href="https://commit.show"><img src="https://img.shields.io/badge/commit.show-engine-F0C040?style=flat-square" alt="commit.show"></a>
  <a href="https://www.npmjs.com/package/commitshow"><img src="https://img.shields.io/npm/v/commitshow?label=cli&color=F0C040&style=flat-square" alt="cli"></a>
  <img src="https://img.shields.io/badge/skill-claude%20code-0F2040?style=flat-square" alt="skill">
  <img src="https://img.shields.io/badge/license-MIT-0F2040?style=flat-square" alt="license">
</p>

<p align="center">
  <em>Companion to in-session security skills — scans the <strong>shipped product</strong> (deployed URL + GitHub signals + repo structure), not the editor buffer.</em>
</p>

---

## What it does

When you've just merged a feature, are about to demo, or are 3 minutes away from posting to Show HN — this skill answers one question:

> "What does this project ship without that 70% of vibe-coded projects also ship without?"

It runs an external audit against the repo's deployed state and surfaces the top 2–3 actionable concerns: missing webhook idempotency, exposed service-role keys, RLS gaps on writable tables, missing rate limits, indexes vs FK columns, prompt-injection surface, mobile-input zoom, column GRANT mismatches, Stripe API idempotency, and ten other failure modes that AI-assisted projects routinely miss.

The audit engine is the same one that powers [commit.show](https://commit.show) — scoring, ranking, and a 14-frame failure framework calibrated against a reference set of real OSS projects.

---

## Why a separate skill?

In-session skills (`security-review` · `vibesec` · OWASP-style) scan your **editor buffer at write-time**. They're great for catching insecure patterns as you type. But they can't see:

- whether your live URL actually returns 200
- whether `og:image` and `manifest.json` are wired
- whether your Lighthouse score collapses on mobile
- whether secrets shipped to your `dist/` bundle
- whether the writable table you defined yesterday actually has an RLS policy
- whether yesterday's `ADD COLUMN` migration silently broke every PostgREST query that includes it (column-level GRANT trap)

`production-audit` scans the **shipped state** — exactly what your users see and what your cloud bills. Run both. They catch different things.

---

## Install

Three ways, pick one:

**`npx skills add` (recommended)**

```bash
npx skills add commitshow/production-audit
```

When the skill triggers, it calls our CLI with `--source=production-audit-skill`
so we can tell skill-driven audits apart from manual CLI runs in our
funnel analytics. No PII; the source is a self-reported tag, drop the
flag if you'd rather stay completely anonymous.

**Inside Claude Code**

```
/plugin marketplace add https://github.com/commitshow/production-audit
/plugin install production-audit@commitshow
```

**Manual**

```bash
git clone https://github.com/commitshow/production-audit ~/tmp/pa && \
  mkdir -p ~/.claude/skills && \
  cp -r ~/tmp/pa/.claude/skills/production-audit ~/.claude/skills/
```

---

## Usage

After install, just describe the problem in plain language:

```
> is this production-ready?
> what would break in prod?
> score my project
> audit my repo
```

Claude Code triggers `production-audit`, the skill runs `npx commitshow audit . --json`, parses the envelope, and surfaces the top concerns with file paths.

Sample output:

```
Score: 82/100 (+5 since yesterday) · band: strong

Top concerns:
  ↓ [Security] No API rate limiting on /auth — IP cap missing
  ↓ [Infrastructure] api/stripe-webhook.ts — signature verified but no
    idempotency-key check (replay window open)

Want me to fix the webhook idempotency gap first?
```

The skill writes `.commitshow/audit.{md,json}` to the repo so future Claude Code sessions can read prior state without re-running the engine.

---

## What gets checked

14 failure-mode frames, calibrated from real production incidents:

1. **Webhook idempotency** · Stripe / payment retry safety
2. **RLS gaps** · per-table coverage on Postgres
3. **Secret client exposure** · service-role keys in shipped bundles
4. **DB missing indexes** · FK columns vs `CREATE INDEX` count
5. **Observability** · Sentry / Datadog / Pino / OTel presence
6. **Rate limit** · API routes without limiter middleware
7. **Prompt injection** · raw user input flowing into model prompts
8. **Hardcoded URLs** · `localhost:3000` leaks shipped to prod
9. **Mock data** · inline arrays of object literals in app paths
10. **Webhook signature** · `Stripe-Signature` / `X-Hub-Signature` verification
11. **CORS permissive** · `Access-Control-Allow-Origin: *` on auth endpoints
12. **Mobile input zoom** · iOS Safari focus-zoom pitfall on text-sm inputs
13. **Column GRANT mismatch** · new migration column without matching `GRANT SELECT` on column-level-grant tables (silent 42501 across all reads)
14. **Stripe API idempotency** · outbound `stripe.checkout.sessions.create` without `idempotencyKey` (duplicate-charge surface)

Plus a Claude Sonnet qualitative pass that reads the README + Build Brief + Lighthouse + GitHub commit cadence and writes the bullets you actually see.

Full detection logic: <https://github.com/commitshow/commitshow/blob/main/supabase/functions/analyze-project/index.ts>

---

## Companion skills

This skill works **alongside** in-session skills, not in place of them. If you also use:

| Skill | What it covers |
|---|---|
| [security-review](https://github.com/anthropics/skills/tree/main/security-review) · `vibesec` · OWASP-style | Editor-buffer scan at write-time. Line-level patterns. |
| **`production-audit`** (this) | Shipped-state scan post-merge. Deployment-time + integration-time gaps. |
| `tdd-workflow` · `test-coverage` | Test discipline lens. Catches what audit's `tests` slot only loosely signals. |

Both lenses miss what the other catches. Run both for serious launches.

---

## Constraints

- Calls commit.show API (`https://api.commit.show`). Behind a firewall blocking `*.supabase.co`, the skill won't work. Use `npx commitshow audit . --json --no-network` for the deterministic-only fallback.
- Anonymous rate limits: 20 audits/IP/day · 5/repo/day · 2000/day global. Login + paid credit raises the per-repo cap.
- Calibrated for **deployed apps** with a live URL. Library / CLI / scaffold form gets a partial-substitute score (max ~45/50 on the audit pillar) — fair but not flattering.
- Cold audit takes 60–90 s. Cached audits (< 7 days) return instantly. `--refresh` force-bypasses cache.
- Public GitHub repos only. Private repos return `not_found`.

---

## License

[MIT](./LICENSE) · © 2026 [Madeflo Inc.](https://commit.show)

The audit engine, CLI, and platform are operated by Madeflo Inc., a Delaware corporation. "commit.show" is the product brand.
