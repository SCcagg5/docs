---
description: When Dagu Console opens and resolves monitoring alerts, who gets the emails, how to send alerts to Slack, Microsoft Teams, PagerDuty, and webhooks, how outside addresses are verified and unsubscribe, and how to mute a noisy workflow.
---

# Monitoring Alerts and Email

Dagu Console opens an alert when a workflow fails or a server stops reporting, and emails the workspace's alert recipients when it opens and when it resolves. On a paid plan, it also sends the alert to the workspace's [Slack and Microsoft Teams channels, PagerDuty services, and webhooks](#slack-teams-pagerduty-and-webhooks). See [Monitoring (Beta)](/overview/console-monitoring) to connect a server, and [Alerts](/overview/console-monitoring-pages#alerts) for the page that lists them.

## Workflow Alerts

Workflow alerts need the `runs` level. An alert belongs to one workflow on one server.

- A run with status failed opens an alert. Further failures add to its count.
- A later run that succeeds or partially succeeds resolves it.
- Aborted and rejected runs count as failures in totals, but they neither open nor resolve an alert.

## Stopped-Reporting Alerts

Stopped-reporting alerts need `health` or `runs`.

A server that sends no report for 10 minutes, or 25 minutes on [free monitoring](/overview/console-monitoring#free-and-paid), opens an alert, and its [recipients](#who-gets-alert-emails) are emailed. Its next report resolves the alert, and that email says how long the server was silent.

- Every server that has reported is watched. To stop watching one, a workspace owner opens its **Settings** tab, clears **Alert when this server stops reporting** under **Alerts**, and selects **Save changes**. Turning it off closes an open alert without an email.
- A server whose monitoring is turned off opens no alert. Turning monitoring off sends one last report that says so: the server shows **Monitoring off**, and an open stopped-reporting alert closes without an email.
- When many servers stop reporting at the same moment, Dagu Console takes the problem to be on its own side and opens no alerts for them.

## Who Gets Alert Emails

A workspace owner chooses who is emailed in **Settings** under **Alert emails**. Alert emails come from `no-reply@dagu.sh` when an alert opens and when it resolves. Alerts that open or resolve together arrive in one email.

| Recipient | Added |
| --- | --- |
| **Owners and admins** | Always listed. The workspace's owners and admins, on by default. |
| A member | At once, by entering their address in **Add a recipient**. |
| An outside address | After it is verified. Entering any other address sends it a link to verify it, which works for 7 days and opens a page with **Verify address**. The address is emailed nothing else until then. **Resend link** sends a new link, at most once every 10 minutes. |

- A recipient gets Dagu server alerts while **Dagu servers** is checked on its row. Clear it and select **Save changes** to stop them, for example on **Owners and admins**.
- On [free monitoring](/overview/console-monitoring#free-and-paid), only **Owners and admins** get Dagu server alerts. The **Dagu servers** box on other rows is unavailable and keeps its setting for when the workspace has a trial or plan.
- **Send test** emails the row a test alert. **Remove** takes a member or address off the list.
- Up to 20 members and addresses besides owners and admins, of which up to 5 may wait to be verified. Verification and test emails are capped per workspace each UTC day.
- **Email language** sets the language of alert, test, and verification emails: English (default), Français, 日本語, 简体中文, العربية, or 한국어.
- An address that the email service refuses is skipped. The other recipients still get the email.

## Unsubscribe

Every alert email ends with a link to unsubscribe, and mail clients that offer one-click unsubscribe can use it too. The link opens a page with **Unsubscribe**. Nothing changes until it is selected.

An address that unsubscribed gets no alert emails from the workspace, even when it is added again. **Alert emails** shows it as **Unsubscribed**. Only the person at that address can subscribe again, with **Subscribe again** on the page the link opens.

## Slack, Teams, PagerDuty, and Webhooks

On a Team or Pro plan, a workspace owner adds up to 10 destinations on the **Notifications** page. Each one gets Dagu server alerts, Kitewell alerts, or both. An alert goes to each destination that gets its product when it opens and when it resolves, as for its email. Under **Add a destination**, each kind opens a short setup that says where to find what it needs.

| Destination | What it takes | What it gets |
| --- | --- | --- |
| **Slack** | The Webhook URL of a Slack app's incoming webhook, which starts with `https://hooks.slack.com/services/`. Create an app at [api.slack.com/apps](https://api.slack.com/apps), turn on **Incoming Webhooks**, and select **Add New Webhook** for the channel. | One message per alert. |
| **Microsoft Teams** | The URL of a Teams workflow. In the channel, select **More options** (⋯), then **Workflows**, and add **Post to a channel when a webhook request is received**. Office 365 connector URLs, which Microsoft retired, are refused. | One Adaptive Card per alert. |
| **PagerDuty** | The **Integration Key** of an **Events API V2** integration on the service, and its service region, US or EU. | An incident when an alert opens, resolved when the alert resolves. |
| **Webhook** | Any `https://` URL on the public internet. The page that adds it shows its signing secret once. | One signed JSON post per alert. |

- A Slack message or Teams card names what failed and where, quotes the end of its error when the server [sends errors](/overview/console-monitoring#what-is-sent), and has **View run** and, when the server has a [Dagu URL](/overview/console-monitoring-pages#servers), **Open in Dagu**. A pass sends at most 20 alerts to a chat channel or webhook and sums up the rest in one more message.
- A PagerDuty incident's summary names what failed and the last line of its error. Its details carry the host, the workflow, the run, the failed steps, and the error, with links to the run. A workflow's failure has severity `error`, and a server that stopped reporting `critical`. Each alert is its own incident, keyed `dagu:<workspace>:<alert>`, so an event sent twice opens no second incident. Every alert of a pass is sent.

**Send test** sends the workspace's last workflow alert of a product the destination gets, as it looked while failing, or an example when there is none, under a line saying it is a test. It runs at most once a minute.

- Adding a Slack or Teams channel sends a test at once. Teams answers before it posts the card, so only the channel shows whether it arrived.
- A PagerDuty test opens an incident and resolves it at once, so whoever is on call may be paged. The page asks before it is sent.

The destination's **Status** shows when it last took a post, or its last refusal. A post answered with 429 or a 5xx status is sent once more; an alert that neither an email nor a destination took is sent again later. Destination URLs, signing secrets, and integration keys are stored encrypted.

A webhook post carries these headers:

| Header | Value |
| --- | --- |
| `X-Dagu-Event` | `alert.opened`, `alert.resolved`, `alerts.more`, or `test`. A `test` carries a sample alert, as **Send test** describes. |
| `X-Dagu-Delivery` | A random ID of the post |
| `X-Dagu-Signature` | `t=<unix seconds>,v1=<hex>`: the HMAC-SHA256 of `t`, a `.`, and the body, keyed by the signing secret |

```json
{
  "type": "alert.opened",
  "sent_at": "2026-10-12T05:41:30Z",
  "workspace_id": "<workspace>",
  "alert": {
    "id": 42,
    "rule": "workflow_failed",
    "workflow": "nightly-etl",
    "host": { "kind": "dagu_server", "id": 7, "name": "etl-prod-1" },
    "failures": 1,
    "failed_steps": ["load"],
    "errors": [{ "step": "load", "message": "exit status 1\nrecent stderr (tail):\nconnection refused" }],
    "last_run_id": "0193f1c2a7d84e6b",
    "opened_at": "2026-10-12T05:41:00Z",
    "last_failed_at": "2026-10-12T05:41:00Z",
    "resolved_at": null
  },
  "links": {
    "alerts": "https://console.dagu.sh/alerts?ws=<workspace>",
    "run": "https://console.dagu.sh/runs?run=0193f1c2a7d84e6b&server=7&ws=<workspace>",
    "dagu": "https://dagu.example.com/dag-runs/nightly-etl/0193f1c2a7d84e6b?step=load"
  }
}
```

A stopped-reporting alert has `rule` `runner_offline` and no run. `errors` is present only when the server sends errors.

## Mute a Workflow

A workspace owner mutes a noisy workflow from [its page](/overview/console-monitoring-pages#workflow-pages): **Mute**, then **For 1 hour**, **For 1 day**, **For 7 days**, or **Until unmuted**. **Unmute** ends it early.

While a workflow is muted, its alerts still open and resolve, marked **Muted**, but nobody is emailed. Muting covers a workflow's alerts only. A server that stops reporting is still emailed.

## Emailed Column

On the **Alerts** page, **Emailed (UTC)** shows when an alert was emailed, or why it was not:

| Shown | Meaning |
| --- | --- |
| A time | When the alert was emailed. |
| **Pending** | The email is waiting to go out. |
| **Not sent** | The alert resolved before its email went out. |
| **No recipients** | No [alert recipient](#who-gets-alert-emails) gets Dagu server alerts. |
| **Muted** | The alert opened while its workflow was [muted](#mute-a-workflow). |

## Example Email

Alert emails are plain text. A failure and a recovery that happen together arrive as one email. A failure quotes the last lines of its error when the server sends errors, and **Open in Dagu** appears when the server has a Dagu URL:

```text
Subject: [Dagu] nightly-etl failed on etl-prod-1 (1 more alert)

Failed

- nightly-etl on etl-prod-1 at 2026-10-12 05:41 UTC
  Run 0193f1c2a7d84e6b, failed at load
    connection refused
  View run: https://console.dagu.sh/runs?run=0193f1c2a7d84e6b&server=7&ws=<workspace>
  Open in Dagu: https://dagu.example.com/dag-runs/nightly-etl/0193f1c2a7d84e6b?step=load
  Workflow: https://console.dagu.sh/workflows/12?ws=<workspace>

Reporting again

- backup-01 (Dagu server) reported at 2026-10-12 05:48 UTC, after 12 minutes without a report

Alerts: https://console.dagu.sh/alerts?ws=<workspace>

You get this email as an owner of the workspace Acme Operations on Dagu Console.
Unsubscribe: https://console.dagu.sh/alert-emails/unsubscribe?r=<token>&w=<workspace>
```

## Related Pages

- [Monitoring (Beta)](/overview/console-monitoring)
- [Console Pages](/overview/console-monitoring-pages)
- [Notifications](/web-ui/notifications)
