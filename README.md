# Telegram Multi-Agent Article Pipeline for n8n

An importable n8n workflow that turns an article title sent through Telegram
into a planned, reviewed, written, and quality-checked article. Four language-
model roles handle the content while n8n keeps routing, state, and retry limits
deterministic.

## Workflow

```text
Telegram title
    -> Planner
    -> Outline Checker -- feedback -> Planner (up to 3 attempts)
    -> Writer
    -> Article Checker -- feedback -> Writer (up to 3 attempts)
    -> Telegram result
```

| Role | Responsibility |
| --- | --- |
| Root orchestrator | Stores state, validates JSON, routes branches, and limits retries |
| Planner | Produces a structured article outline |
| Outline Checker | Scores the plan and returns approval or revision feedback |
| Writer | Produces the full article from the accepted outline |
| Article Checker | Reviews structure, clarity, style, and length |

The root orchestrator uses ordinary n8n Code, If, Merge, and No Operation nodes.
It is not another model call.

## Features

- Telegram input and output in the same chat
- `/start`, `/help`, and `/article` command support
- JSON validation between model calls
- Bounded planning and writing retries
- Telegram-safe message chunking for long output
- Graceful fallback with the best draft and latest feedback

## Requirements

- An n8n instance with the workflow's LangChain nodes
- A Telegram bot created with `@BotFather`
- A Telegram credential in n8n
- An OpenAI-compatible chat-model credential
- A public HTTPS n8n URL for the Telegram webhook

## Quick start

1. Import [`n8n-multi-agent-article-workflow.json`](n8n-multi-agent-article-workflow.json).
2. Connect `Telegram Trigger` and `Send Result to Telegram` to your bot credential.
3. Connect the four OpenAI model nodes to your model credential.
4. Keep **Use Responses API** enabled when using the configured `gpt-5-mini` model.
5. Restrict the bot to approved Telegram chat or user IDs before production use.
6. Save and activate the workflow.
7. Send the bot an article title or use:

   ```text
   /article How Small Businesses Can Use AI Without Losing the Human Touch
   ```

## Quality and retry rules

| Rule | Default |
| --- | ---: |
| Outline approval | 80/100 or higher |
| Article approval | 80/100 or higher |
| Category scores | At least 7/10 each |
| Article length | 800-1200 words |
| Planner attempts | Maximum 3 |
| Writer attempts | Maximum 3 |
| Telegram chunk | About 3400 characters |

An immediately approved article uses four model calls. With all retries, one
request can use up to twelve calls.

## Security checklist

- Store Telegram and model keys only in n8n Credentials.
- Add an allow-list after the Telegram trigger to prevent unauthorized usage.
- Review raw workflow exports for credential IDs, webhook IDs, instance IDs,
  execution data, and pinned Telegram messages before committing them.
- Revoke any token that was ever committed; removing it from the latest file is
  not enough.

The committed workflow is a sanitized template without BotFather or OpenAI keys.

## Limitations

- The checker evaluates writing quality, not factual accuracy.
- The workflow receives only a title and performs no web research.
- Telegram is the only input and output channel in this template.
- Generated articles still require human factual review before publishing.

For sourced articles, add a research stage before planning and provide its
source packet to the Planner, Writer, and Article Checker.
