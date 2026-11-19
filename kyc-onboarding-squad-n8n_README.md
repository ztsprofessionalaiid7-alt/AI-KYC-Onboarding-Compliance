# AI KYC / Onboarding Compliance Squad: 5 Agents (n8n)

An n8n workflow where five AI agents handle a customer onboarding review: routing the depth of review, verifying documents, screening names against a watchlist, scoring the risk and writing the decision. Rules in code always override the AI, so a watchlist hit, an expired document or a high risk can never be auto-approved. Synthetic data only.

▶ **Demo video:** https://drive.google.com/file/d/1q9I6vgLDK17PvM1_DLJTqxXxEaAi7xCN/view

## How it works

1. Mock applications arrive (name, document, country, occupation, source of funds, expected monthly volume) and the watchlist is read from Postgres
2. Agent 1, Router: chooses standard or enhanced due diligence (cash-heavy business, unclear source of funds, volume over 20,000 USD a month or high-risk country)
3. Agent 2, Document Verifier: checks that the name on the document matches the applicant (even small spelling differences) and that the document is not expired
4. Agent 3, Watchlist Screener: looks for exact or close name matches (spelling variants, transliterations) with a confidence level
5. Agent 4, Risk Scorer: returns a risk level and a score from 0 to 100. Agent 5, Decision Writer: proposes approve, manual_review or reject with a rationale and a customer message
6. A Code node applies the rules (below), and a Switch routes by decision: approve sends the customer email, manual_review and reject go to Slack. Everything is saved to Postgres

```
Manual Trigger -> Mock Applications -> Get Watchlist (Postgres) -> Attach Watchlist
-> Router -> Document Verifier -> Watchlist Screener -> Risk Scorer -> Decision Writer
-> Parse Decision -> Decision?
   approve       -> Send Approval Email + Save Decision
   manual_review -> Slack Review Alert + Save Decision
   reject        -> Slack Review Alert + Save Decision
```

## Agents and models

Each agent uses its own model, from three providers (all have free tiers):

| Agent | Provider | Model |
|---|---|---|
| Router Agent | Groq | `llama-3.1-8b-instant` |
| Document Verifier Agent | Google Gemini | `models/gemini-2.5-flash` |
| Watchlist Screener Agent | Groq | `llama-3.3-70b-versatile` |
| Risk Scorer Agent | OpenRouter | `meta-llama/llama-3.3-70b-instruct:free` |
| Decision Writer Agent | Google Gemini | `models/gemini-2.5-flash` |

## Rules in code

- An exact name match on the watchlist sets the decision to `reject` if the reason mentions sanctions, otherwise `manual_review`
- A possible (fuzzy) watchlist match, a document issue or a high risk turns `approve` into `manual_review`
- Enhanced review with a risk level above low turns `approve` into `manual_review`
- Invalid AI output defaults to `manual_review`

## Tech stack

n8n, AI Agents, Groq, OpenRouter, Google Gemini, Slack, Postgres (Supabase), SMTP

## Setup

1. Copy the JSON from `workflow.md` into a file named `kyc-squad-cloud.json` and import it into n8n
2. Create credentials: Groq (API key), OpenRouter (API key), Google Gemini (API key), Slack (Bot token and signing secret), Postgres (Supabase Session pooler, SSL: Require), SMTP (for the email node, for example Gmail with an app password)
3. Open each model node and pick your own credential for its provider
4. In the Slack node set channel `kyc-reviews`, then run `/invite @your-bot` in that channel
5. In the Send Email node replace `YOUR_EMAIL@gmail.com` with your address
6. Create the database tables (below)
7. In the Postgres save nodes remove the `id` and `created_at` fields, the database fills them
8. Click **Execute workflow**: the mock inputs run through all branches

Free-tier models have rate limits. If you see "Too Many Requests", wait a few minutes and run again.

## Database

```sql
create table if not exists public.kyc_watchlist (
  id bigint generated always as identity primary key,
  name text,
  country text,
  reason text
);

create table if not exists public.kyc_decisions (
  id bigint generated always as identity primary key,
  application_id text,
  name text,
  country text,
  review_level text,
  documents_ok text,
  watch_match text,
  risk_level text,
  risk_score numeric,
  decision text,
  rationale text,
  created_at timestamptz default now()
);

insert into public.kyc_watchlist (name, country, reason) values
  ('Viktor Petrow', 'CY', 'Sanctions list: suspected sanctions evasion through trading companies'),
  ('Boris Fakenov', 'IR', 'Politically exposed person with ties to sanctioned entities'),
  ('Mario Testa', 'IT', 'Adverse media: fraud investigation');
```

## Demo data

| Mock input | Expected result |
|---|---|
| KYC-1001 Aisha Rahman: valid passport, salary, 3,000 USD a month | Standard review, likely approve |
| KYC-1002 Victor Petrov: ID says "Viktor Petrow" and expired, cash business, 85,000 USD a month | Spelling mismatch, expired document, possible watchlist match, never approved |
| KYC-1003 Boris Fakenov: exact watchlist hit (politically exposed person) | Watchlist rule, manual review, never approved |

AI answers can vary between runs. The rules in code do not.

## Notes

- All applicants and the watchlist are SYNTHETIC. Never put real personal data or real sanctions lists into a demo
- The exact match in code only catches identical names. Spelling variants are caught by the Watchlist Screener agent, which is why that case also needs the document check and the risk rules
- The approval email goes to `YOUR_EMAIL@gmail.com` in the demo, use the applicant email only in a real system
- Real KYC needs a compliance review and regulated data handling. This is a demo of the pattern, not a compliance tool
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
