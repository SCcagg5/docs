---
description: See every self-hosted Dagu server in one place, get an email when a workflow fails or a server stops reporting, and track trends. Opt-in, outbound HTTPS only, metadata only.
---

# Monitoring (Beta)

::: info Beta
Monitoring is in beta. The retention periods and limits on this page may change during the beta.
:::

Connect your Dagu servers to [Dagu Console](https://console.dagu.sh) and see all of them in one place:

- **Every server at a glance**: which servers report, what failed today, open alerts, and what needs attention, such as a stopped scheduler or a server that went silent.
- **Alerts by email**: when a workflow fails and when it recovers, and when a server stops reporting, to the members and addresses you choose.
- **Trends**: runs, success rate, and p50 and p95 duration for each workflow over 7, 30, or 90 days.
- **Nothing sensitive leaves the server**: reporting is opt-in, outbound HTTPS only, and carries metadata. Logs, outputs, parameters, and secrets are [never sent](#never-sent), and each server shows the exact JSON it last sent.

Workflows still run only on your servers. Dagu Console never runs them, and a server does not need the console to run them.

```text
Dagu server ── HTTPS, outbound, every 60 s ──▶ Dagu Console ──▶ Overview, Servers, Runs, Alerts, Insights
(runs workflows)                                            └──▶ alert emails to the workspace's recipients
```

## Requirements

- Dagu v2.19.1 or later.
- A Dagu Console workspace on the Team or Pro plan, or in a trial. See [License Comparison](/overview/self-host-license).
- The server is connected to Dagu Console with a license that checks in: approved from **Plan & features**, activated with `dagu license activate`, or started with a server key. See [Connect a Server](/overview/self-host-license#connect-a-server). Servers with offline licenses do not report.
- The server can reach `https://console.dagu.sh` over HTTPS.
- For run status, the [event store](/server-admin/reference#event-store) is enabled. It is by default.

## Get Your First Alert

1. [Connect the server](/overview/self-host-license#connect-a-server) to Dagu Console from **Plan & features** in Dagu.
2. On the same page, select **Turn on** in the **Monitoring** section, choose **Health and run status (recommended)**, and select **Turn on**.
3. In Dagu Console, open **Settings**. Under **Alert emails**, owners and admins already get alert emails. Add any other member or address there.
4. Select **Send test** on a row to check that the email arrives.

Within a minute, the server shows **Reporting** on the console's **Servers** page. From then on, a failed workflow, or a server silent for 10 minutes, emails the recipients. See [Alerts and Email](/overview/console-monitoring-alerts) for who gets what, and [Console Pages](/overview/console-monitoring-pages) for what each page shows.

## Turn It On

Monitoring is off by default. Upgrading or connecting a server sends nothing beyond its license check-in until an administrator of the server chooses what to report.

### From the web UI

1. Open **Plan & features** as an administrator and select **Turn on** in the **Monitoring** section.
2. Choose **Health and run status (recommended)** or **Health only**. The dialog lists exactly what each choice sends and what is never sent.
3. Select **Turn on**.

Dagu also asks on its own. Right after a server connects from **Plan & features**, it opens **Monitor this server from Dagu Console**. On a server that was connected earlier, administrators see a **Monitor this server from Dagu Console** notice above other pages until they choose a level or dismiss it.

The choice applies at once, without a restart, and is kept in the data directory (`cloud/report-settings.json` under `paths.data_dir`). Persist that directory in containers, as you do for the [license](/overview/self-host-license#keep-the-server-s-identity).

The **Monitoring** section on **Plan & features** shows:

- **Reports**: the level, with **Change** and **Turn off**.
- **Last report**: when Dagu Console last accepted a report. After 10 minutes without one, it shows **Not reporting since** and the time.
- **View last report**: the exact JSON of the last report, with the heartbeat secret hidden.

### In configuration

For servers managed as code, set the level in the [configuration file](/server-admin/reference#configuration-file):

```yaml
cloud:
  report: runs   # off, health, or runs
```

or with `DAGU_CLOUD_REPORT=runs`.

| `cloud.report` | Shown in the web UI as | Sends |
| --- | --- | --- |
| empty (default) | Off, until an administrator chooses | Nothing beyond the license check-in |
| `off` | Off | Nothing beyond the license check-in |
| `health` | Health only | [Health](#health) |
| `runs` | Health and run status | [Health](#health) and [run status](#run-status) |

A configured value, `off` included, wins over the choice in the web UI. The **Monitoring** section shows **Set in configuration** and offers no **Turn on**, **Change**, or **Turn off**. To change the level, change the configuration and restart Dagu.

## What Is Sent

### Health

Sent at `health` and `runs`:

- Dagu version, operating system, and architecture
- When the server process started
- Which services run: the number of schedulers holding the scheduler lock, and the number of registered coordinators

### Run status

Sent at `runs` only:

- **Each top-level run that finishes**, once: the DAG name, run ID, attempt ID, status (succeeded, partially succeeded, failed, aborted, or rejected), how it was triggered (scheduler, manual, webhook, retry, or catchup), when it was queued, started, and finished, and the names of its failed steps, up to 50.
- **The runs in progress**: the runs queued, running, or waiting, with the same fields up to the start time. A report lists up to 200 of them, the most recently started or queued first, and the total count.

DAG names and step names are sent only at `runs`. Choose `health` if you consider them sensitive.

### Never sent

Logs, outputs, parameters, environment, step commands, DAG YAML, the values of workflow secrets, and error message text are never sent, at any level.

Every report also carries what identifies and authenticates the server, as its license check-in does: the protocol version, license ID, server ID, and heartbeat secret. It also carries the level. At `runs`, it also carries the reporter's position in the event store and, after events were lost, the time they were lost from.

[RFC 002: Dagu Cloud Connector](https://github.com/dagucloud/dagu/blob/main/rfcs/002-dagu-cloud-connector.md) specifies the wire contract.

## Cadence and Network

- The server reports every 60 seconds.
- It reports early, within about 5 seconds, when a run fails, is aborted, or is rejected, and when the runs in progress change: a run is queued, starts, begins waiting, or finishes.
- The first report after Dagu starts waits a random 0 to 60 seconds, so servers started together do not report at once.
- Reports are outbound HTTPS requests to `https://console.dagu.sh/api/v1/servers/report`. Nothing connects into the server, and no inbound port is opened.
- The process that holds the license sends the reports: `dagu server` or `dagu start-all`. When several processes share a data directory, one of them reports.

If Dagu Console cannot be reached, workflows are not affected. The server keeps its place in the event store, retries with a backoff of up to 5 minutes, and sends the runs it missed once the console is back. Dagu Console stores each finished run once, so a resent report creates no duplicates.

The event store keeps one day of events by default (`event_store.retention_days`). A server cut off for longer loses the runs that finished in between. Its next report says so, and Dagu Console shows **Events lost** for the server instead of guessing those runs.

## Retention and Limits

During the beta, Dagu Console keeps:

| Data | Kept for |
| --- | --- |
| Finished runs | 14 days |
| Daily totals per workflow | 13 months |
| Resolved alerts | 90 days after they resolve |
| Open alerts | Until they resolve |
| Health and runs in progress | Until the next report replaces them |

A server that sends no report for 30 days is removed from monitoring, with its daily totals and alerts.

Each workspace stores up to 500 finished runs per UTC day, counted by when Dagu Console records them. Past the limit, runs are still counted in the daily totals and still open and resolve alerts, but they are not stored, so they never appear in run lists. Storing resumes at 00:00 UTC. Until then:

- the console's **Overview** shows **Daily run limit reached** with the time;
- workspace owners get one email that day;
- the server's **Monitoring** section on **Plan & features** shows **Run history paused until** 00:00 UTC.

To raise the limit, [contact the Dagu team](https://dagu.sh/contact).

## Turn It Off

- **On the server**: select **Turn off** in the **Monitoring** section of **Plan & features**, or set `cloud.report: off` and restart Dagu. Reporting stops at once, including a report in flight, and a last report tells Dagu Console, which shows **Monitoring off** and opens no [stopped-reporting alert](/overview/console-monitoring-alerts#stopped-reporting-alerts). Turning it on later does not send the runs that finished in between.
- **Delete reports**: turning monitoring off keeps what the server already sent. To remove it, a workspace owner selects **Delete reports** on the server's **Settings** tab. This deletes the server's health reports, run history, and alerts. A server that still has monitoring on reports again, so turn it off on the server first.
- **Disconnect**: disconnecting the server, from **Plan & features** or with **Disconnect server** in Dagu Console, also stops reporting and removes the server from the monitoring pages. See [Check-ins and disconnecting](/overview/self-host-license#check-ins-and-disconnecting).

## Related Pages

- [Alerts and Email](/overview/console-monitoring-alerts)
- [Console Pages](/overview/console-monitoring-pages)
- [Connect a Server](/overview/self-host-license#connect-a-server)
- [Configuration Reference](/server-admin/reference)
- [Notifications](/web-ui/notifications)
- [Incident Routing](/web-ui/incidents)
- [Prometheus Metrics](/server-admin/prometheus-metrics)
