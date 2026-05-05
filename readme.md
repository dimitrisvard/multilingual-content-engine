<div align="center">

# multilingual-content-engine

A production multilingual AI content pipeline. **Tiered model routing.** Five Supabase Edge Functions running in [Microns Hub](https://micronshub.eu) — generates one SEO-grounded article per day with Claude, then ships it through a 13-language translation pipeline with Gemini. Lifted out of the production codebase as a portfolio piece.

![TypeScript](https://img.shields.io/badge/TypeScript-Deno-3178C6?logo=typescript&logoColor=white)
![Anthropic](https://img.shields.io/badge/Anthropic-Claude-D4A27F)
![Gemini](https://img.shields.io/badge/Google-Gemini-4285F4?logo=google&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-Edge%20Functions-3ECF8E?logo=supabase&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

*~3,600 lines of working production code across 5 Edge Functions.*

</div>

---

## What this is

The actual content pipeline that powers [micronshub.eu](https://micronshub.eu)'s multilingual SEO surface. No simplification — only changes are sanitising env-var examples and writing this README.

Five Edge Functions:

1. **`generate-daily-article`** — picks one of five manufacturing topic silos in rotation, calls Claude to generate a fully-structured English article with internal links, writes it to Postgres.
2. **`auto-translate-articles`** — orchestrator. Finds the new English article, kicks off translation to 13 languages one at a time (Edge Functions have a 150s timeout, so we serialise).
3. **`translate-article`** — translates via Gemini, with per-language slug rewriting (`/services` → `/dienstleistungen` for German) and brand-name preservation rules.
4. **`process-article-queue`** — queue processor for retries.
5. **`fix-article-links`** — periodic broken-link sweep.

The value isn't the pipeline; it's the architectural pattern.

## Tiered model routing

| Step | Model | Why |
|---|---|---|
| English article generation | **Claude (Anthropic)** | High stakes — canonical text every translation derives from. |
| Translation to 13 languages | **Gemini (Google)** | Higher volume — 13× the call count. Gemini handles structured translation cleanly at fraction of the cost. |

Two providers, one pipeline. Cost graph stops following volume, starts following high-stakes-call count.

## The 13-language surface

`de` German · `fr` French · `es` Spanish · `it` Italian · `nl` Dutch · `pt` Portuguese · `sv` Swedish · `da` Danish · `nb` Norwegian · `pl` Polish · `cs` Czech · `hu` Hungarian · `fi` Finnish

Each language has a service-slug map (`cnc-machining` → `cnc-bearbeitung` in German, `usinage-cnc` in French) so internal links land on the right localised URL. See `supabase/functions/translate-article/index.ts`.

## Architecture & decisions

### One function per pipeline step
Considered: a single megafunction. Rejected because Edge Functions have a 150s timeout. Generation alone takes 30–60s; serialising 13 translations after that would burn the budget. Splitting steps means each runs against its own clock.

### Translate one language at a time, not in parallel
Earlier version translated all 13 in parallel. Hit the timeout once. Now it translates serially, easy languages first (German/French/Spanish first, Polish/Czech/Hungarian/Finnish last). If the function gets cut off, simpler languages already landed.

### Brand name as an explicit "do not translate" rule
`BRAND_NAME = "Microns Hub"` constant in both prompts. The translator gets a hard instruction not to translate it. Every multilingual product gets this wrong on day one.

### Silo rotation by day-of-year
`(dayOfYear - 1) % 5`. Considered random pick — rejected because predictable rotation lets me look at any future date and know what topic publishes. Useful for editorial planning.

### Internal links computed at generation time
Generator fetches related articles from the same silo before calling Claude, includes them in the prompt as candidate link targets. The LLM weaves them in contextually better than a regex-based linker would.

### IndexNow for fast indexation
After translation, each new URL is pinged via IndexNow (Bing-driven, picked up by Yandex). Optional via `INDEXNOW_KEY`.

### Deno + Edge Functions
Live next to the database, deploy in seconds, direct Postgres access without a network hop. Trade-off: Deno runtime, slightly different ergonomics from Node.

## Install / deploy

```bash
git clone https://github.com/dimitrisvard/multilingual-content-engine
cd multilingual-content-engine

brew install supabase/tap/supabase    # macOS
# or: scoop install supabase           # Windows

supabase link --project-ref <your-project-ref>

supabase secrets set ANTHROPIC_API_KEY=sk-ant-...
supabase secrets set GEMINI_API_KEY=...
supabase secrets set SITE_URL=https://your-domain.example.com
supabase secrets set INDEXNOW_KEY=...    # optional

supabase functions deploy generate-daily-article
supabase functions deploy translate-article
supabase functions deploy auto-translate-articles
supabase functions deploy process-article-queue
supabase functions deploy fix-article-links
```

## Trade-offs

- **Two LLM providers, not one.** Translation cost savings justify two API keys + two response parsers. Below ~5 translations/day this isn't worth it.
- **Edge Functions, not a long-running service.** Free hosting, free auto-scale, zero infra. Lose in-memory state across calls — every translation re-fetches the parent article. Clearly winning trade at this size.
- **No translation memory / glossary system.** Brand-name + service-slug rules cover 80% of consistency value at 1% of the engineering cost.

## What I'd do differently

- **Eval harness for the generation step from day one.** Result: I've shipped subtle prompt regressions twice.
- **Make the translation step idempotent at the article level.** Right now repeated calls overwrite. An `if exists return existing` check would let me retry safely.
- **Surface per-article cost in the admin dashboard.** Today I look at it via Anthropic + Google billing. Pulling it into the database alongside each article would put cost next to quality.

## License

MIT — see [LICENSE](./LICENSE).

## Contact

Dimitris Vardalachakis · `dimitrisvard@hotmail.com` · [github.com/dimitrisvard](https://github.com/dimitrisvard) · [linkedin.com/in/dimitrisvard](https://www.linkedin.com/in/dimitrisvard)

Built while running [Microns Hub](https://micronshub.eu). Open to remote AI Product Engineer / Founding Engineer roles in Europe.
