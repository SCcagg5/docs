---
description: When Dagu Console opens and resolves monitoring alerts, who gets the emails, how outside addresses are verified and unsubscribe, and how to mute a noisy workflow.
---

# Monitoring Alerts and Email

Dagu Console opens an alert when a workflow fails or a server stops reporting, and emails the workspace's alert recipients when it opens and when it resolves. See [Monitoring (Beta)](/overview/console-monitoring) to connect a server, and [Alerts](/overview/console-monitoring-pages#alerts) for the page that lists them.

## Workflow Alerts

Workflow alerts need the `runs` level. An alert belongs to one workflow on one server.

- A run with status failed opens an alert. Further failures add to its count.
- A later run that succeeds or partially succeeds resolves it.
- Aborted and rejected runs count as failures in totals, but they neither open nor resolve an alert.

## Stopped-Reporting Alerts

Stopped-reporting alerts need `health` or `runs`.

A server that sends no report for 10 minutes opens an alert, and its [recipients](#who-gets-alert-emails) are emailed. Its next report resolves the alert, and that email says how long the server was silent.

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
- **Send test** emails the row a test alert. **Remove** takes a member or address off the list.
- Up to 20 members and addresses besides owners and admins, of which up to 5 may wait to be verified. Verification and test emails are capped per workspace each UTC day.
- **Email language** sets the language of alert, test, and verification emails: English (default), Français, 日本語, 简体中文, العربية, or 한국어.
- An address that the email service refuses is skipped. The other recipients still get the email.

## Unsubscribe

Every alert email ends with a link to unsubscribe, and mail clients that offer one-click unsubscribe can use it too. The link opens a page with **Unsubscribe**. Nothing changes until it is selected.

An address that unsubscribed gets no alert emails from the workspace, even when it is added again. **Alert emails** shows it as **Unsubscribed**. Only the person at that address can subscribe again, with **Subscribe again** on the page the link opens.

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

Alert emails are plain text. A failure and a recovery that happen together arrive as one email:

```text
Subject: [Dagu] nightly-etl failed on etl-prod-1 (1 more alert)

Failed

- nightly-etl on etl-prod-1 at 2026-10-12 05:41 UTC
  Run 0193f1c2a7d84e6b, failed at load
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
