# AI Job Intelligence & Alerts

I built this system to find engineering opportunities and deliver a shortlist to Telegram. I currently run a seven-source version privately. This repository preserves the earlier single-source teaching edition.

## Inside the public workflow

The [10-node n8n export](ai-job-alerts.n8n.json) starts on a schedule, fetches jobs from Adzuna, filters roles, removes seen jobs and builds a Gemini scoring request. The final steps parse the response, keep strong matches and send a Telegram alert. The export is saved inactive.

## Review or test locally

Open the JSON to inspect the nodes and connections. For an isolated trial, import a copy into a disposable n8n workspace, keep the schedule disabled and configure your own Adzuna, Gemini and Telegram credentials. Use a test chat and synthetic job records to check duplicate handling, malformed model output and score thresholds before enabling delivery.

The private edition is owner-reported operational. Its source coverage, recent run counts and behavior under provider limits need a redacted export or run record for independent verification. This public snapshot establishes only the Adzuna path.
