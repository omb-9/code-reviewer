Code Reviewer



Code Review Assistant (Telegram + OpenRouter)


An n8n workflow that turns Telegram into the front end for a structured, schema-enforced code review tool. Paste a snippet, get back severity-ranked findings, concrete fixes, test suggestions, and an optional rewrite, all generated through OpenRouter with automatic model fallback.


Why this exists


Most "AI code review bot" demos pipe a prompt into a model and post whatever comes back. This one forces the model to answer inside a strict JSON schema, so every finding has a severity, a category, a quoted line fragment, an explanation, and a fix. That structure is what makes the output usable instead of just plausible-sounding prose. The prompt logic, flag parsing, and JSON schema all live in a single readable Code node rather than being buried inside an AI Agent node's configuration, so the review contract is easy to read, version, and change.


Features




Accepts plain pasted code or fenced code blocks with a language tag


Detects the language automatically across nine common grammars when none is declared


Inline flags to control depth and focus without touching the workflow


Structured JSON output enforced through OpenRouter's response_format schema


Automatic retry and model fallback if the primary model fails or times out


Defensive parsing that degrades gracefully instead of throwing on malformed model output


Long reviews are split into ordered, Telegram-safe message chunks




Flags


Send these before your code, in any combination.




Flag
Effect




--quick
Top issues only, one-sentence explanations


--deep
Adds architectural notes and testing gaps, up to 15 findings


--security
Reorders priority toward injection, auth, secrets, and access control


--perf
Reorders priority toward complexity, blocking calls, and caching


--lang=python
Forces a language instead of auto-detecting one




Example:


/review --security
```python
import os
def run(cmd):
    return os.system('sh -c ' + cmd)




### Setup

1. Import `code-review-assistant.n8n.json` into n8n.
2. Create a Telegram credential (BotFather token) and attach it to the four Telegram nodes: Telegram Trigger, Send Help, Show Typing, Send Unsupported, Send Empty Prompt, Send Review, and Send Failure Notice.
3. Create an HTTP Header Auth credential named `OpenRouter API Key`:
   - Header name: `Authorization`
   - Header value: `Bearer sk-or-your-key-here`
4. Attach that credential to the OpenRouter Code Review node.
5. Open the Config node and set your preferred primary and fallback models, for example `anthropic/claude-sonnet-4.5` and `openai/gpt-4.1-mini`.
6. Activate the workflow, then message the bot `/help` to confirm it responds.

### How it works

The Telegram Trigger receives the message, and the Route Input node splits traffic three ways: `/help` and `/start` go to a static help message, anything with text or a caption goes to the review path, and anything else (a sticker, a photo with no caption) gets a short unsupported-input reply.

On the review path, Prepare Review Request strips the `/review` command, parses the flags, extracts code from a fenced block if present, sniffs the language if it wasn't declared, truncates oversized input, and assembles the full OpenRouter payload, including the JSON schema the model must follow. A quick check filters out messages that are too short to be real code before any API call is made.

The OpenRouter Code Review node sends the request, retries up to three times on failure, and falls back to a second model using OpenRouter's `models` array if the primary one is unavailable. A dedicated error output routes to a failure message so the user is never left without a reply.

Format Review parses the model's JSON response, tolerating fenced code blocks or partially malformed output, sorts findings by severity, and renders the result as Telegram-formatted HTML with severity icons, syntax-highlighted code blocks, and a token usage footer. If the response can't be parsed at all, it falls back to showing the raw text rather than failing silently.

### Customization ideas

- Swap the Telegram Trigger for a GitHub webhook to review pull request diffs instead of pasted snippets, reusing the same prompt and schema.
- Add a persistence step (a database or spreadsheet node) after Format Review to keep a history of reviews per chat.
- Extend the schema with a `confidence` field per finding if you want the model to flag its own uncertainty.

### Limitations

- Very large files will hit the character truncation limit set in the Config node; raise `maxCodeChars` if needed, keeping model context limits in mind.
- Language detection is heuristic-based and can misfire on unusual or minified code.
- This reviews single snippets in isolation, so it has no awareness of the rest of a codebase.
