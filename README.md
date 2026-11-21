# Internal-Tools

The D.A.W.N. team's internal bots and tools that aren't part of the product or
its CI. Each one lives in its own folder, with its workflow in
`.github/workflows/`.

| Tool | What it does |
| --- | --- |
| [`shame-bot/`](shame-bot/README.md) | Shames PR habits in dawn-care/DAWN on Discord and logs them for the team website's `/shame` page. |

Data the tools produce (for the team website to read) goes on the `metrics`
branch, under `metrics/`. This repo is public so the website can read it
without a token: don't put anything here that shouldn't be.

```bash
node --test shame-bot/rules.test.mjs
```
