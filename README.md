# Internal-Tools

The D.A.W.N. team's internal bots and tools that aren't part of the product or
its CI. Each one lives in its own folder, with its workflow in
`.github/workflows/`.

| Tool | What it does |
| --- | --- |
| [`pr-notifications.yml`](.github/workflows/pr-notifications.yml) | Posts what's happening on dawn-care/DAWN's PRs to Discord: opened, merged, review requested, approved or denied, comments (batched into one message per reviewer) and pushes. |
| [`shame-bot/`](shame-bot/README.md) | Shames PR habits in dawn-care/DAWN on Discord and logs them for the team website's `/shame` page. |

## Where the DAWN events come from

DAWN's `.github/workflows/internal-tools-forwarder.yml` sends every PR event
(opened, pushed, reviewed, commented on, closed) here as one `pr-event`
repository_dispatch, trimmed to the fields the tools use, and every workflow
that cares listens for it. It needs DAWN's `TOOLS_DISPATCH_TOKEN` secret: a
fine-grained token with Contents: write on dawn-care/Internal-Tools only. When
it expires, every tool here stops hearing about DAWN, so rotate it before then.
Anything else they need, they read from DAWN's API (it's public, so this
repo's own token can).

Because they're triggered by repository_dispatch, the workflows always run
from `main`. To try a change, push it to `main` (or run the shame bot by hand).

Secrets here: `DISCORD_WEBHOOK_URL`, `DISCORD_USER_MAP` (JSON of GitHub login
to Discord user ID), and optionally `DISCORD_SHAME_WEBHOOK_URL` to post shames
in a separate channel.

Data the tools produce (for the team website to read) goes on the `metrics`
branch, under `metrics/`. This repo is public so the website can read it
without a token: don't put anything here that shouldn't be.

```bash
node --test shame-bot/rules.test.mjs
```
