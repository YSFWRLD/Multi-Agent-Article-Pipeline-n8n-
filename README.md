# Telegram Multi-Agent Article Pipeline for n8n

An importable n8n workflow that turns an article title sent through Telegram into a reviewed, structured article. Four separate language-model roles plan, review, write, and review again. n8n acts as the deterministic root orchestrator, so approval decisions and retry limits stay under workflow control.

## What it does

```text
Telegram title
      |
      v
Root initializes state
      |
      v
Planner <--------------------+
      |                       |
      v                       | rejected with feedback
Outline Checker -------------+
      |
      | approved
      v
Root accepts outline
      |
      v
Writer <---------------------+
      |                       |
      v                       | rejected with feedback
Article Checker -------------+
      |
      | approved
      v
Root formats the result
      |
      v
Telegram article
```

The workflow includes:

- Telegram input and output in the same chat.
- A Planner that returns a title, introduction plan, bullet points, body structure, and conclusion plan.
- A dedicated Outline Checker that sends actionable feedback back to the Planner.
- A Writer that follows the approved outline.
- A separate Article Checker that sends feedback back to the Writer.
- JSON validation between model calls.
- Bounded retries to prevent infinite loops and uncontrolled credit usage.
- Telegram-safe message chunking for long articles.
- `/start`, `/help`, and `/article` command support.

## Repository files

| File | Purpose |
| --- | --- |
| [`n8n-multi-agent-article-workflow.json`](n8n-multi-agent-article-workflow.json) | Sanitized workflow template to import into n8n. |
| [`n8n-multi-agent-article-setup.md`](n8n-multi-agent-article-setup.md) | Short setup guide and operating notes. |

## Requirements

- An n8n instance with the LangChain nodes used by this workflow.
- A Telegram bot created through `@BotFather`.
- A Telegram API credential in n8n.
- The `n8n free OpenAI API credits` credential, or your own OpenAI API credential.
- A publicly reachable n8n URL so Telegram can call the trigger webhook.

## Quick start

### 1. Import the workflow

1. Download [`n8n-multi-agent-article-workflow.json`](n8n-multi-agent-article-workflow.json).
2. In n8n, open **Workflows**.
3. Choose **Import from File** and select the JSON file.

### 2. Connect Telegram

1. Message `@BotFather` in Telegram and send `/newbot`.
2. Copy the token BotFather gives you.
3. In n8n, open `Telegram Trigger` and create or select a Telegram API credential.
4. Open `Send Result to Telegram` and select the same credential.

Keep the BotFather token inside n8n Credentials. Never paste it directly into the workflow JSON or commit it to GitHub.

### 3. Connect the n8n free OpenAI credits

Open each model node and select `n8n free OpenAI API credits`:

- `OpenAI - Planner`
- `OpenAI - Outline Checker`
- `OpenAI - Writer`
- `OpenAI - Article Checker`

Use the same settings on all four nodes:

| Setting | Value |
| --- | --- |
| Model | `gpt-5-mini` |
| Use Responses API | On |
| Built-in Tools | Empty |
| Options | Empty |

The Responses API switch matters. Leaving it off can cause `Unsupported parameter: 'stop' is not supported with this model` in some n8n OpenAI-node configurations.

### 4. Restrict access before activation

The template accepts messages from any Telegram account that can reach the bot. That is convenient for testing, but a public bot can let strangers consume your n8n and OpenAI credits.

Before production use, add an `If` node after `Telegram Trigger` and allow only your Telegram `message.chat.id` or `message.from.id`. Keep the allowed ID in your private n8n configuration. Do not commit your personal chat ID if you do not want it public.

### 5. Activate and test

1. Save and activate the workflow.
2. Send the bot a title:

   ```text
   How Small Businesses Can Use AI Without Losing the Human Touch
   ```

3. Wait for the approval summary and article to return to the same Telegram chat.

You can also use:

```text
/article How Small Businesses Can Use AI Without Losing the Human Touch
```

Send `/start` or `/help` to display the bot instructions.

## Agent responsibilities

| Role | Responsibility | Output |
| --- | --- | --- |
| Root | Stores state, routes branches, enforces retry limits, and formats Telegram messages. | Deterministic workflow state. |
| Planner | Converts the requested title into a complete article structure. | Structured outline JSON. |
| Outline Checker | Scores the outline and returns approval or specific revision feedback. | Outline review JSON. |
| Writer | Writes the article using the approved outline and checker feedback. | Article JSON. |
| Article Checker | Scores structure, clarity, style, and length, then approves or requests revisions. | Article review JSON. |

The Root is implemented with n8n Code, If, Merge, and No Operation nodes. It is not another language-model call. This keeps routing predictable and saves free credits.

## Approval and retry rules

| Rule | Default |
| --- | --- |
| Outline approval score | At least `80/100` |
| Article overall score | At least `80/100` |
| Article category scores | Every category at least `7/10` |
| Article length | `800-1200` words |
| Planner attempts | Maximum `3` |
| Writer attempts | Maximum `3` |
| Telegram chunk size | About `3400` characters |

An article that passes both checks immediately uses four model calls. Rejections create more calls. With the default limits, one request can use up to twelve model calls.

To reduce free-credit usage, change `maxPlanAttempts` and `maxDraftAttempts` in `ROOT - Initialize State` from `3` to `2`.

## Result behavior

- Approved outlines move to the Writer.
- Rejected outlines return to the Planner with checker feedback.
- Approved articles return to Telegram.
- Rejected articles return to the Writer with checker feedback.
- If the outline never passes, Telegram receives the last outline and review feedback.
- If the article never passes, Telegram receives the best available draft with a human-review warning.
- Long output is split into separate plain-text Telegram messages.

## GitHub safety checklist

The workflow file in this repository is a sanitized template. It contains no n8n credential references, n8n instance ID, workflow ID, version ID, webhook ID, BotFather token, or OpenAI API key.

Do not commit a raw export from your active n8n instance without reviewing it first. Check for and remove:

- `credentials` objects containing credential names and internal credential IDs.
- Top-level `id`, `versionId`, and `meta.instanceId` values.
- Node-level `webhookId` values.
- Non-empty `pinData`, which can contain real Telegram messages, chat IDs, usernames, article titles, or model output.
- BotFather tokens, OpenAI API keys, bearer tokens, private keys, and password-bearing URLs.
- Execution exports, screenshots, or logs that show credential names or personal Telegram data.
- `.env` files and private backup exports.

Before every push, inspect the staged files:

```bash
git status
git diff --cached
```

If a real token was ever committed, deleting it from the latest file is not enough. Revoke and rotate it, then clean it from Git history before making the repository public.

## Troubleshooting

### `Unsupported parameter: 'stop'`

Use the Basic LLM Chain nodes included in this template. In every OpenAI model node, select `gpt-5-mini`, enable **Use Responses API**, leave **Built-in Tools** empty, and leave **Options** empty.

### Telegram Trigger does not receive messages

- Confirm the workflow is active.
- Confirm the Telegram credential uses the current BotFather token.
- Confirm your n8n instance has a public HTTPS URL.
- Check whether another active workflow is using the same bot webhook.

### The workflow runs but Telegram receives no reply

- Open the n8n execution and find the first failed node.
- Confirm `Send Result to Telegram` uses the same Telegram credential as the trigger.
- Confirm the incoming update contains a text message and `message.chat.id`.

### The agents keep rejecting output

- Inspect the checker JSON in the execution data.
- Confirm every model node uses the same working model configuration.
- Reduce the approval thresholds only if the lower quality is acceptable.
- Keep retry limits bounded when using free credits.

## Security and privacy

Telegram message content, article titles, outlines, drafts, and checker feedback are processed by n8n and sent to the configured model provider. Review the privacy requirements for your use case before sending confidential material.

Use n8n Credentials for all secrets. Restrict the bot to approved chat or user IDs, keep execution retention as short as practical, and review pinned data before exporting a workflow.

## Limitations

- The checker reviews structure and writing quality, not factual accuracy.
- The workflow receives only a title and does not perform web research.
- Generated content can still contain factual errors or unsupported claims.
- Telegram is the only input and output channel in this template.

For sourced articles, add a Research Agent before the Planner and pass its source packet to the Planner, Writer, and Article Checker.
