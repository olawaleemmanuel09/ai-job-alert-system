# AI Job Alert System

An automated job hunting pipeline built in n8n. It fetches fresh AI and automation job listings daily, scores each one against a candidate profile using Google Gemini, and delivers only strong matches to Telegram with a score, a one line explanation, and a direct apply link.

Built by [Whale (Olawale Emmanuel)](https://olawaleemmanuel09.github.io) with a student at Laoni AI Academy as a live teaching project.

## The problem

Job boards are noisy. Finding relevant roles means scrolling hundreds of listings daily, most of them irrelevant, many of them duplicates, and the good ones get buried. Applying late costs interviews. This system inverts the process: the jobs come to you, pre scored, with reasoning, within hours of being posted.

## How it works

```
Schedule Trigger (daily 08:00)
  -> HTTP Request: Adzuna API (fetch listings)
  -> Split Out (one item per job)
  -> Code: keyword filter (AI/automation roles only)
  -> Remove Duplicates (cross execution memory, never resends a job)
  -> Code: build Gemini prompt (job + candidate profile)
  -> HTTP Request: Gemini 2.5 Flash (score 0 to 100 + verdict + reason)
  -> Code: parse response, merge with job data
  -> Code: threshold gate (score >= 65 only)
  -> Telegram (formatted alert with apply link)
```

Ten nodes. Three free APIs. Zero running cost.

## Node breakdown

| # | Node | Type | Function |
|---|------|------|----------|
| 1 | Schedule Trigger | scheduleTrigger | Fires the workflow daily at 08:00 |
| 2 | Adzuna - Get Jobs | httpRequest | GET request to Adzuna search endpoint with keyword and recency params |
| 3 | Split Jobs | splitOut | Converts the results array into one item per job |
| 4 | Filter AI Roles | code | Keeps jobs whose title or description matches an AI/automation keyword list |
| 5 | Remove Seen Jobs | removeDuplicates | Cross execution dedup keyed on redirect_url, 10k history |
| 6 | Build Prompt | code | Assembles the Gemini prompt from the job fields and the candidate profile |
| 7 | Gemini - Score Job | httpRequest | POST to Gemini 2.5 Flash with responseMimeType application/json for structured output |
| 8 | Parse & Merge | code | Unwraps Gemini's JSON string answer and merges it with the original job fields, with a try/catch fallback |
| 9 | Keep Strong Matches | code | Passes only items with match_score >= 65 |
| 10 | Send Telegram Alert | telegram | Sends an HTML formatted alert per strong match |

## Prerequisites

You need three free accounts:

1. **Adzuna developer account** at developer.adzuna.com. Copy your App ID and App Key.
2. **Gemini API key** from Google AI Studio. No credit card required for the free tier.
3. **Telegram bot**. Message @BotFather, run /newbot, copy the token. Then send your new bot any message (a bot cannot message you until you message it first) and get your chat ID from @userinfobot or from `https://api.telegram.org/bot<TOKEN>/getUpdates`.

You also need a running n8n instance, self hosted or cloud.

## Setup

1. Download `ai-job-alerts.n8n.json` from this repo.
2. In n8n: Workflows, then the three dot menu, then Import from File.
3. Replace the four placeholders:

| Location | Placeholder |
|----------|-------------|
| Adzuna - Get Jobs node, query parameters | `YOUR_ADZUNA_APP_ID` and `YOUR_ADZUNA_APP_KEY` |
| Gemini - Score Job node, x-goog-api-key header | `YOUR_GEMINI_API_KEY` |
| Send Telegram Alert node, Chat ID field | `YOUR_TELEGRAM_CHAT_ID` |

4. On the Send Telegram Alert node, create a Telegram credential with your bot token. Leave the Base URL as the default `https://api.telegram.org`.
5. Open the Build Prompt node and replace the `profile` string with your own background, stack and target roles. This is what Gemini scores against, so make it honest and specific.
6. Run the workflow once manually. The first run clears the backlog, so expect a burst of alerts. From the second run onward, only new postings come through.
7. Toggle the workflow Active.

## Customisation

**Country.** The Adzuna country code lives in the URL path (`/jobs/gb/search/1`). Adzuna covers gb, us, de, fr, au, nz, ca, in, pl, br, at and za. There is no Nigeria endpoint, so this build targets remote friendly markets. To search several countries, duplicate the Adzuna node per country and merge before Split Jobs.

**Keywords.** Edit the array at the top of the Filter AI Roles node.

**Threshold.** The score gate lives in Keep Strong Matches. Raise it to 75 for fewer, sharper alerts. Lower it to 55 for a wider net.

**Schedule.** Change the Schedule Trigger to run twice daily if speed matters to you.

**Rate limits.** Gemini 2.5 Flash free tier allows roughly 15 requests per minute and 1500 per day. If a run produces more than about 15 surviving jobs, add a Wait node of 2 seconds before the Gemini node, or reduce results_per_page.

## Design decisions

**Why the keyword filter and dedup sit before the AI call.** Gemini is the expensive node in both time and quota. Cheap filters run first so only jobs worth scoring get scored. This keeps the system comfortably inside the free tier.

**Why an HTTP Request node instead of n8n's native AI nodes.** The raw request makes the prompt, the structured output setting and the response shape visible, which is exactly what this project needed to teach. Swapping in a Basic LLM Chain with a Structured Output Parser is a clean upgrade if you prefer it.

**Why the prompt is built in a Code node.** Job descriptions contain quotes and line breaks. Injecting the text as a variable into an n8n object expression means the request body can never be broken by the content, which removes an entire class of intermittent failures.

**Why Remove Duplicates instead of a database.** n8n's cross execution dedup is one node with zero infrastructure. If you need a queryable history of everything seen, swap it for a Postgres or Supabase insert with a conflict clause.

## Skills demonstrated

REST API integration, query and header authentication, JSON parsing and nested field access, cross execution state, prompt design with structured LLM output, defensive parsing with fallbacks, threshold logic, and webhook style delivery to a messaging platform.

## Repo contents

```
ai-job-alerts.n8n.json    the importable workflow
README.md                 this file
```

## License

MIT. Use it, fork it, point it at your own job hunt.
