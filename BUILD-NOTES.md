# BUILD-NOTES.md

Honest build log for YouTube Comment Miner v1. Q&A format, written for anyone who wants to fork this or ask questions in an interview.

---

## Why did you build this?

Consumer Health brand teams in India read 1mg reviews and Amazon reviews. Both are post-purchase, templated, sometimes shaped by discount code incentives. The predictive signal — what consumers ask before they buy — sits in YouTube comment sections, 500 to 5,000 per popular review video, raw and unformatted.

No agency in India is productising this for CH brands. Research agencies charge Rs 10–25L for a single qualitative study that produces thinner data, slower, without the weekly refresh. The gap was obvious. The build took 4 sessions across 5 days.

---

## Why n8n and not Python or Claude Code?

Third application of my n8n Import-to-Production SOP v1. Scaffolding was already shipped on FB Ad Spy Tool and MR Field Companion. Rebuilding this in Python would mean re-doing the scheduler, the credential store, the retry logic, the Google Sheets connector. n8n gives me all of that for free. Claude Code would be overkill — this is a linear pipeline, not an agent decision tree.

Target reuse was 60%+ off shipped tools. Actual reuse estimate: 68%.

---

## What did you reuse from previous builds?

- **n8n Import SOP v1** (8-stage process): template fork → localise → credentials → stub test → full test → harden → document → publish
- **FB Ad Spy Tool**: Google Sheet + Looker Studio delivery pattern, retry/OAuth refresh handling, Cron pattern, lessons log (default to credential store from Stage 2, not Stage 3)
- **Competitor Intel Agent**: Google Sheets Append node pattern, verbatim-quote system prompt (critical for consumer language integrity)
- **MR Field Companion**: full RAG spine reserved for v2 chat layer (Supabase pgvector + OpenAI embeddings + Cohere Rerank)
- **Brand Analysis Skill**: output schema discipline for the monthly PDF report

---

## What broke and how did you fix it?

Five breaks in one afternoon, all fixed inside 15 minutes each. Order mattered.

1. **Query Auth allowed domains**. Set it to bare `googleapis.com`, request failed because YouTube is served from `www.googleapis.com`. Changed to `*.googleapis.com` wildcard. Fix: 30 seconds.
2. **OpenAI API key expired**. The stored key had hit its 1-year expiry. OpenAI removed "Never expire" as an option in 2025. Rotated key, updated the shared credential, auto-fixed 6 other workflows. Added a calendar reminder to rotate again before Nov 2027.
3. **n8n Code node sandbox blocks `require('crypto')`**. Can't hash author IDs the standard way. Replaced with pure JS djb2 hash, 8 hex chars, non-cryptographic but good enough for anonymisation.
4. **Format Row Mode was "Run Once for All Items"**. Needed "Run Once for Each Item" to process per-comment. One dropdown change.
5. **Google Sheets tab bound to literal `Sheet1` string instead of the sheet's gid**. n8n rejected the append. Re-selected "Sheet1" from the live tab dropdown so it stored the gid. 15 seconds.

Lesson for SOP v1.1: run a pre-flight check before Stage 4 stub test — verify OpenAI balance and key expiry, YouTube API key alive, Google OAuth freshness. Catches 3 of the 5 breaks above before they happen.

---

## Why GPT-4o-mini for classification?

Cheap, fast, consistent. Temperature 0.1 for determinism. JSON output mode for structured return. Monthly cost at current volume: ~Rs 20. A larger model would push Rs 300+/month without meaningful accuracy lift at this task. Comment classification is a 6-way labelled routing problem, not a reasoning problem.

Reserved Claude Sonnet 4.5 for the monthly PDF report narrative instead, where brand-audit voice matters more.

---

## Why djb2 hash and not SHA-256?

n8n Code node sandbox blocks `require('crypto')`. Can't call SHA. Tried. Blocked.

djb2 is a non-cryptographic hash. It's not for passwords. It's for anonymisation and stability — the same commenter across 3 videos gets the same 8-char hash, so brand managers can see repeat voices, but the original channel ID is lost forever. That's the DPDP compliance posture we need. Overkill cryptography is not the bar; one-way mapping and stability is.

Documented this in the Format Row node notes so the next person doesn't waste 30 min on the same blocker.

---

## How did you validate the classifier?

- **Stage 4 stub test**: 1 video, 2 comments. Pipeline end-to-end green. Both comments classified correctly as "noise" (emoji-only + single-word).
- **Stage 5 full test**: 3 videos, 190 comments. Spot-checked 50 rows across English, Hindi, Hinglish, Gujarati. Accuracy ~92–95%. Classifier correctly caught use cases like "Can it be taken by rheumatoid arthritis patient" and objections like "This is too sweet. Not suitable for sugar patients".
- **Stage 5.1 competitor check**: swapped in an Ensure vs Horlicks Short with 65 comments. Classifier caught Beast Life, Avvatar, Nutrija, Willmaxx, Quenchlabz, Progenz, Myherb — the actual competitive mindshare in Indian protein powders — correctly tagged as competitor_mention. Didn't catch Horlicks or Protinex because commenters didn't mention them. Not a classifier bug. Their mental model of competitors is from 2015. Consumers moved on.

---

## What did you learn about the Indian Consumer Health category by reading these comments?

Three things that would cost Rs 15L to learn from a research agency:

1. **Legacy brand competitive mindshare is dying.** Horlicks, Complan, Protinex don't come up organically when Indian consumers talk about protein powders in 2026. Beast Life, Avvatar, Nutrija, Willmaxx do. Brand teams at Abbott Nutrition are probably benchmarking against the wrong set.
2. **Trust in parent manufacturer matters more than product features.** "Abbot is not manufacturing this product" repeated by multiple commenters. Consumers are checking the back of the pack and noticing changes. Brand trust is auditable to them now.
3. **Underserved use cases are the whitespace.** Dual comorbidity (diabetes + heart failure), post-childbirth recovery, tuberculosis patient nutrition, rheumatoid arthritis — these come up repeatedly and get zero brand-authored content. Each one is a Rs 5L campaign idea.

---

## What's v1.1 vs v2?

**v1.1** (small, same architecture):
- Pagination beyond 100 comments per video
- Error log tab for silent-drop monitoring
- Cost swap on video analysis (Gemini free tier was unreliable, moving to GPT-4o on paid tier)
- Prompt tightening to stop tagging the subject brand ("Ensure") as its own competitor

**v2** (adds infrastructure):
- Supabase pgvector + OpenAI embeddings + Cohere Rerank for RAG
- n8n AI Agent + chat widget
- Brand manager types: "show me all comments about artificial sweeteners across protein brand review videos, last 90 days" and gets ranked passages with video+timestamp citations
- Priced as an upgrade tier on the monthly subscription

---

## How much did v1 cost to build?

- OpenAI API: $0.07 (~Rs 6) of actual spend across 4 build sessions
- YouTube Data API v3: Rs 0 (free tier, used ~1200 quota units of 10,000/day limit)
- Google Sheets, n8n hosting, GitHub: Rs 0 (existing infrastructure)
- My time: ~10 hours across 5 days

Running cost at current volume: ~Rs 145/month.

---

## Would you build this the same way again?

Mostly yes, with 3 changes:

1. Run the pre-flight checklist before Stage 4 stub test. Saved 45 min of debugging.
2. Lock the Query Auth allowed domain to `*.googleapis.com` in the template, not the instance.
3. Pick the seed comparison video by scrolling its comments first, not its title. "Ensure vs Horlicks" in the title doesn't mean commenters compared them.

Everything else held.
