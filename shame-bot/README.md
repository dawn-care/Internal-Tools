# Shame bot

Judges PRs in dawn-care/DAWN, posts shame to Discord, and logs it on this
repo's `metrics` branch, where the team website (dawn-care/Internal-Website)
reads it. Run by [`shame-bot.yml`](../.github/workflows/shame-bot.yml). The
rules are in `rules.mjs`, and everything else (GitHub, the log, Discord) is in
`shame.mjs`.

```bash
node --test shame-bot/rules.test.mjs
```

Try a run without saving or posting anything (`DRY_RUN=1`):

```bash
mkdir -p /tmp/shame/metrics
GH_TOKEN=$(gh auth token) DRY_RUN=1 METRICS_DIR=/tmp/shame WATCHED_REPO=dawn-care/DAWN \
  GITHUB_EVENT_NAME=schedule node shame-bot/shame.mjs
```

## Where it hears about PRs

- **PR events** (opened, pushed, reviewed, commented on, closed): DAWN's
  `.github/workflows/internal-tools-forwarder.yml` sends each one here as a
  `pr-event` repository_dispatch, trimmed to the fields `shame.mjs` uses:

  ```json
  {
    "event_type": "pr-event",
    "client_payload": {
      "repo": "dawn-care/DAWN",
      "event": "pull_request | pull_request_review | issue_comment",
      "payload": { "action": "...", "sender": {}, "pull_request": {}, "review": {}, "comment": {}, "issue": {} }
    }
  }
  ```

  It needs the `TOOLS_DISPATCH_TOKEN` secret in DAWN. Runs whose `repo` isn't
  `WATCHED_REPO` (set in the workflow) are refused.
- **Overdue reviews and idle authors**: an hourly run reads DAWN's open PRs.
  DAWN is public, so this repo's own `GITHUB_TOKEN` can.
- **`/shame`**: the team website sends a `manual-shame` repository_dispatch
  (below).

Secrets here:

- `DISCORD_SHAME_WEBHOOK_URL`: a webhook for the shame channel. The bot fails
  without it rather than post anywhere else (`DISCORD_WEBHOOK_URL` is the PR
  notifications' channel).
- `DISCORD_USER_MAP`: JSON of **GitHub login** to Discord user ID, like
  `{"ha4716": "123456789012345678"}`. Not Discord usernames: every shame is
  logged under the GitHub login, so the website can match it to the team, and
  automatic shames only know GitHub logins. The PR notifications and the team
  website's `DISCORD_USER_MAP` use the same map. A run warns about any login it
  couldn't ping.

## `metrics/shame.json`

Every shame that has happened, one entry each, oldest first. It only grows:
changing a rule never changes past shames. Like DAWN's `pull-requests.json`, it holds
facts, and the website does the counting (by person, by category, over time).
Read it from
`https://raw.githubusercontent.com/dawn-care/Internal-Tools/metrics/metrics/shame.json`.

```json
{
  "categories": { "reviews": "Reviews", "meetings": "Meetings", "...": "..." },
  "shames": [
    { "at": "2026-11-19T13:17:02.000Z", "login": "bob", "category": "reviews", "type": "slow_review", "pr": 487 },
    {
      "at": "2026-11-20T15:00:00.000Z", "login": "bob", "category": "meetings", "type": "manual",
      "reason": "late to standup", "by": "alice", "endorsedBy": ["carol", "dave"]
    }
  ],
  "updatedAt": "2026-11-20T15:00:00.000Z"
}
```

- `categories`: category id to its label, from `CATEGORIES` in `rules.mjs`.
  Show the label and group by the id; ids never get renamed.
- `shames[].at`: when the bot recorded it (UTC).
- `shames[].login`: the GitHub login of the person shamed.
- `shames[].category`: one of the `categories` ids.
- `shames[].type`: what caught it. `behemoth` (PR size), `late_night` (sleep),
  `slow_review` (reviews), `idle_author` (follow-through), or
  `manual` (a `/shame` in Discord, any category).
- `shames[].pr`: the PR number, for shames about a PR.
- `reason`, `by`, `endorsedBy`: only on `manual` shames. The reason is what the
  shamer typed; `by` and `endorsedBy` are GitHub logins.
- `updatedAt`: when the last shame was added.

`metrics/shame-state.json` is the bot's own bookkeeping (which shames it has
already logged, and when each open PR was pushed to). It isn't a metric, and
its format can change at any time.

## `/shame` (sent by the team website)

`/shame` and its Endorse button run in the team website's `/api`, since
Discord sends commands and button clicks to a web address, and the webhook this
bot posts with can't receive anything. Once a `/shame` has enough
endorsements, the website sends it here:

```http
POST https://api.github.com/repos/dawn-care/Internal-Tools/dispatches
Authorization: Bearer <token with Contents: write on dawn-care/Internal-Tools>

{
  "event_type": "manual-shame",
  "client_payload": {
    "messageId": "<the /shame message's Discord ID>",
    "target": "<Discord user ID of the person shamed>",
    "by": "<Discord user ID of the shamer>",
    "endorsedBy": ["<Discord user ID>", "..."],
    "category": "<a categories id>",
    "reason": "<up to 300 characters>"
  }
}
```

The bot checks it again (`manualShame` in `rules.mjs`) and refuses it, failing
the run, unless:

- everyone in it is in the `DISCORD_USER_MAP` secret (so only the team can
  shame, be shamed or endorse);
- at least 2 endorsers are neither the shamer nor the person shamed;
- the category is one of `categories`, and the reason isn't empty or over 300
  characters.

Each `messageId` is logged once, so sending the same one again (say, two
Endorse clicks at the same moment) does nothing. The bot posts the shame to
Discord, mentioning only the person shamed, so anything typed in the reason
can't ping anyone.
