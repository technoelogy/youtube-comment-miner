# BUILD-NOTES.md

Honest build log for YouTube Comment Miner v1.1. Q&A format, written for anyone who wants to fork this or ask questions in an interview.

---

## Why did you build this?

Consumer Health and skincare brand teams in India read Nykaa and Amazon PDP reviews. Both are post-purchase, templated, sometimes shaped by discount code incentives. The predictive signal — what consumers ask before they buy — sits in YouTube comment sections, 500 to 5,000 per popular review video, raw and unformatted.

No agency in India is productising this for CH or skincare brands at scale. Research agencies charge Rs 10–25L for a single qualitative study that produces thinner data, slower, without the weekly refresh. The gap was obvious. v1 took 4 sessions across 5 days. v1.1 (skincare pivot) took a single session on top of the shipped v1.

---

## Why did you pivot v1.1 from protein nutrition to skincare?

v1 shipped on Ensure and Horlicks Protein+. It worked. 157 rows, clean classification. But two things surfaced during review:

1. **Comment volume per video in adult nutrition is modest.** 100–150 comments per popular video.
2. **Legacy brand competitive mindshare in nutrition is low.** Consumers talk about new D2C whey challengers, not Protinex or Complan, so the "competitor_mention" column was mostly empty.

Skincare in India is the opposite on both counts. Popular review videos pull 500–5,000 comments. Consumers name 10+ competing brands organically in a single video. v1.1 on three skincare videos produced 269 classified rows — **71% more insight density than v1**, same workflow, same cost, same effort.

**Lesson learned:** specialty selection before build is a hard rule now, not a judgement call. Added as Stage 0.5 of the Pharma Build Kickoff SOP.

---

## Why n8n and not Python or Claude Code?

Third application of my n8n Import-to-Production SOP v1. Scaffolding was already shipped on FB Ad Spy Tool and MR Field Companion. Rebuilding this in Python would mean re-doing the scheduler, the credential store, the retry logic, the Google Sheets connector. n8n gives me all of that for free. Claude Code would be overkill — this is a linear pipeline, not an agent decision tree.

Target reuse was 60%+ off shipped tools. Actual reuse estimate: 68%.

---

## What did you reuse from previous builds?

- **n8n Import SOP v1** (8-stage process): template fork → localise → credentials → stub test → full test → harden → document → publish
- **FB Ad Spy Tool**: Google Sheet + Looker Studio delivery pattern, retry/OAuth refresh handling, Cron pattern, lessons log
- **Competitor Intel Agent**: Google Sheets Append node pattern, verbatim-quote system prompt (critical for consumer language integrity)
- **MR Field Companion**: full RAG spine reserved for v2 chat layer (Supabase pgvector + OpenAI embeddings + Cohere Rerank)
- **Brand Analysis Skill**: output schema discipline for the monthly PDF report

---

## What broke and how did you fix it?

Five breaks in one afternoon during v1 build, all fixed inside 15 minutes each. Order mattered.

1. **Query Auth allowed domains**. Set it to bare `googleapis.com`, request failed because YouTube is served from `www.googleapis.com`. Changed to `*.googleapis.com` wildcard. Fix: 30 seconds.
2. **OpenAI API key expired**. The stored key had hit its 1-year expiry. OpenAI removed "Never expire" as an option in 2025. Rotated key, updated the shared credential, auto-fixed 6 other workflows. Added a calendar reminder to rotate again before Nov 2027.
3. **n8n Code node sandbox blocks `require('crypto')`**. Can't hash author IDs the standard way. Replaced with pure JS djb2 hash, 8 hex chars, non-cryptographic but good enough for anonymisation.
4. **Format Row Mode was "Run Once for All Items"**. Needed "Run Once for Each Item" to process per-comment. One dropdown change.
5. **Google Sheets tab bound to literal `Sheet1` string instead of the sheet's gid**. n8n rejected the append. Re-selected "Sheet1" from the live tab dropdown so it stored the gid. 15 seconds.

Lesson for SOP v1.1: run a pre-flight check before Stage 4 stub test — verify OpenAI balance and key expiry, YouTube API key alive, Google OAuth freshness. Catches 3 of the 5 breaks above before they happen.

---

## What changed in v1.1 when you pivoted to skincare?

Three things. Zero infrastructure rework.

1. **Seed List node** — swapped three nutrition video IDs for three skincare video IDs (two Minimalist, one The Derma Co).
2. **Classify Comment system prompt** — rewrote the prompt from "nutrition drink products with competitors Protinex, Horlicks, Complan" to "skincare products with competitors Minimalist, The Derma Co, Dot & Key, Plum, Cetaphil, CeraVe, La Roche-Posay, The Ordinary, Mamaearth, Lotus, Biotique, Foxtale, Deconstruct, etc." Theme examples shifted to acne breakout, pigmentation, sensitive skin, pregnancy safety, retinol tolerance, niacinamide irritation.
3. **Google Sheet rows deleted** — kept the schema, cleared old protein data.

Pipeline ran first time without errors. 269 skincare rows landed cleanly on the first execution. The architecture held up across a category pivot with 20 minutes of configuration change.

---

## Why GPT-4o-mini for classification?

Cheap, fast, consistent. Temperature 0.1 for determinism. JSON output mode for structured return. Monthly cost at current volume: ~Rs 25. A larger model would push Rs 300+/month without meaningful accuracy lift at this task. Comment classification is a 6-way labelled routing problem, not a reasoning problem.

Reserved Claude Sonnet 4.5 for the monthly PDF report narrative instead, where brand-audit voice matters more.

---

## Why djb2 hash and not SHA-256?

n8n Code node sandbox blocks `require('crypto')`. Can't call SHA. Tried. Blocked.

djb2 is a non-cryptographic hash. It's not for passwords. It's for anonymisation and stability — the same commenter across 3 videos gets the same 8-char hash, so brand managers can see repeat voices, but the original channel ID is lost forever. That's the DPDP compliance posture we need. Overkill cryptography is not the bar; one-way mapping and stability is.

Documented this in the Format Row node notes so the next person doesn't waste 30 min on the same blocker.

---

## How did you validate the classifier?

- **Stage 4 stub test**: 1 video, 2 comments. Pipeline end-to-end green. Both comments classified correctly as "noise" (emoji-only + single-word).
- **Stage 5 full test (v1 protein)**: 3 videos, 190 comments. Spot-checked 50 rows across English, Hindi, Hinglish, Gujarati. Accuracy ~92–95%.
- **Stage 5.1 competitor validation (v1 protein)**: swapped in a comparison Short. Classifier caught Beast Life, Nutrija, WishCare, Willmaxx — the actual competitive mindshare in Indian protein powders. Didn't catch Protinex or Horlicks because commenters didn't mention them. Not a classifier bug. The brand team's mental model of competitors was outdated. Consumers moved on.
- **v1.1 skincare**: 3 videos, 269 comments. Spot-checked ~80 rows. Accuracy ~92–95%. Classifier correctly caught WishCare, The Ordinary, SKIN 1004, Isntree, Purito, Deconstruct, Dot & Key, Aminu, Plum, Versed, Conscious Chemist, Meesho (as counterfeit concern). Also surfaced trust-crisis anxiety that one Indian D2C brand may have been acquired by a multinational FMCG — raised independently by multiple commenters across two different videos.

---

## What did you learn about the Indian skincare category by reading these comments?

Three things that would cost Rs 15L to learn from a research agency:

1. **Legacy brand competitive mindshare is dead.** Olay, Lakme, Pond's, Biotique, Garnier do not come up organically when Indian consumers talk about skincare in 2026. D2C and K-beauty brands do — WishCare, Deconstruct, The Ordinary, SKIN 1004, Isntree, Purito, Dot & Key, Aminu, Plum, Versed, Conscious Chemist. Brand teams at legacy MNCs are probably benchmarking against the wrong set.
2. **Trust in parent manufacturer matters.** Public speculation about one Indian D2C brand being acquired by a multinational FMCG is spreading across comment sections, with no official holding statement anywhere visible. Brand reputation is auditable to consumers now, and silence in the surface where the question is spreading is a losing posture.
3. **Underserved use cases are the whitespace.** PCOS skin, melasma management, male oily T-zone, teenager acne starter routines, pre-wedding routines, cold-weather skincare, pregnancy and lactation safety for actives — these come up repeatedly and get little to no brand-authored direct content. Each one is a Rs 5–10L campaign brief sitting unclaimed.

---

## What's v1.2 vs v2?

**v1.2** (small, same architecture):
- Pagination beyond 100 comments per video
- Error log tab for silent-drop monitoring
- Cost swap on video analysis (Gemini free tier was unreliable, moving to GPT-4o on paid tier)
- Prompt tightening to stop tagging the subject brand as its own competitor
- Multi-category prompt auto-switch (skincare vs nutrition vs OTC) based on brand field

**v2** (adds infrastructure):
- Supabase pgvector + OpenAI embeddings + Cohere Rerank for RAG
- n8n AI Agent + chat widget
- Brand manager types: "show me all comments about niacinamide irritation across skincare review videos, last 90 days" and gets ranked passages with video+timestamp citations
- Priced as an upgrade tier on the monthly subscription

---

## How much did v1 + v1.1 cost to build?

- OpenAI API: ~$0.15 (~Rs 13) of actual spend across all build sessions
- YouTube Data API v3: Rs 0 (free tier, used ~2,500 quota units of 10,000/day limit)
- Google Sheets, n8n hosting, GitHub: Rs 0 (existing infrastructure)
- My time: ~12 hours across 6 days end-to-end (v1 + v1.1)

Running cost at current volume: ~Rs 150/month.

---

## Would you build this the same way again?

Mostly yes, with 4 changes:

1. **Lock specialty before build.** Dermatology/skincare ships 71% more insight density per video than adult nutrition. Pick the specialty where the signal is densest, not whichever seems easiest.
2. **Run the pre-flight checklist before Stage 4 stub test.** Saves 45 min of debugging credential and quota issues.
3. **Lock the Query Auth allowed domain to `*.googleapis.com` in the template, not the instance.** Prevents the subdomain-mismatch 30-second fix during the first live run.
4. **Pick the seed comparison video by scrolling its comments first, not its title.** "Minimalist vs The Derma Co" in the title doesn't guarantee commenters actually compare them. Validate by Ctrl+F in the comment section before adding the ID to the seed list.

Everything else held.
