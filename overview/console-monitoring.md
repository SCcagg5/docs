---
description: Report a self-hosted Dagu server's health and run status to Dagu Console, see every server in one place, and get an email when a workflow fails.
---

# Monitoring (Beta)

::: info Beta
Monitoring is in beta. The retention periods and limits on this page may change during the beta.
:::

A Dagu server connected to [Dagu Console](https://console.dagu.sh) can report its health and run status. The workspace then sees all of its servers in one place, and its owners and admins get an email when a workflow fails. Workflows still run only on the server. Dagu Console never runs them, and the server does not need the console to run them.

```text
Dagu server ── HTTPS, outbound, every 60 s ──▶ Dagu Console ──▶ Overview, Servers, Runs, Alerts
(runs workflows)                                            └──▶ alert emails to owners and admins
```

## Requirements

- Dagu v2.19.0 or later.
- The server is connected to Dagu Console with a license that checks in: approved from **Plan & features**, activated with `dagu license activate`, or started with a server key. See [Connect a Server](/overview/self-host-license#connect-a-server). Servers with offline licenses do not report.
- The server can reach `https://console.dagu.sh` over HTTPS.
- For run status, the [event store](/server-admin/reference#event-store) is enabled. It is by default.

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

Every report also carries what identifies and authenticates the server, as its license check-in does: the protocol version, license ID, server ID, and heartbeat secret. At `runs`, it also carries the reporter's position in the event store and, after events were lost, the time they were lost from.

[RFC 002: Dagu Cloud Connector](https://github.com/dagucloud/dagu/blob/main/rfcs/002-dagu-cloud-connector.md) specifies the wire contract.

## Cadence and Network

- The server reports every 60 seconds.
- It reports early, within about 5 seconds, when a run fails, is aborted, or is rejected, and when the runs in progress change: a run is queued, starts, begins waiting, or finishes.
- The first report after Dagu starts waits a random 0 to 60 seconds, so servers started together do not report at once.
- Reports are outbound HTTPS requests to `https://console.dagu.sh/api/v1/servers/report`. Nothing connects into the server, and no inbound port is opened.
- The process that holds the license sends the reports: `dagu server` or `dagu start-all`. When several processes share a data directory, one of them reports.

If Dagu Console cannot be reached, workflows are not affected. The server keeps its place in the event store, retries with a backoff of up to 5 minutes, and sends the runs it missed once the console is back. Dagu Console stores each finished run once, so a resent report creates no duplicates.

The event store keeps one day of events by default (`event_store.retention_days`). A server cut off for longer loses the runs that finished in between. Its next report says so, and Dagu Console shows **Events lost** for the server instead of guessing those runs.

## In Dagu Console

The **Monitoring** group in the console's sidebar holds the pages below. Every member of the workspace can view them. Only workspace owners can delete reports, disconnect servers, and change the email setting.

### Overview

- **Servers reporting**, **Failed today (UTC)**, **Open alerts**, and **Success (7 UTC days)**.
- **Needs attention**: problems to act on, each naming the servers it affects: Not reporting, Not checking in, Offline, Scheduler stopped, Failing workflows, Events lost, Monitoring off, and, for owners, Over capacity.
- **Open alerts** and **Failures today (UTC)**, up to 10 of each. Failures include aborted and rejected runs.

### Servers

**Connected servers** lists each server with its status, Dagu version, last report, failures today (UTC), success rate over 7 UTC days, and open alerts.

| Status | Meaning |
| --- | --- |
| **Reporting** | A report arrived in the last 5 minutes. |
| **Not reporting** | The server reported before, but not in the last 5 minutes. |
| **Not checking in** | The server has not checked in with its license for more than 26 hours. |
| **Offline** | The server has not checked in with its license for more than 72 hours. |
| **Monitoring off** | Dagu Console holds no report from the server: monitoring was never turned on, or its reports were deleted. |

**Scheduler stopped** appears next to the status when the last report counts no running scheduler, so scheduled workflows do not start.

Each server's page has these tabs:

- **Overview**: the server's details, such as its ID, Dagu version, OS and architecture, process start, last report, and last check-in; the services from the last report; open alerts; and failures in the last 14 days.
- **Runs**: the server's runs.
- **Workflows**: each workflow's finished runs per UTC day over the last 14 days as bars, with runs, failures, success rate, and average and longest duration. **Failing** marks a workflow with an open alert.
- **Settings**, for owners: **Delete reports** and **Disconnect server**.

### Runs

Runs in progress come first, then finished runs, newest recorded first. Filter by:

- **Server**
- **Workflow**
- **Status**: Failed (including aborted and rejected), Succeeded (including partially succeeded), Running, Waiting, or Queued
- Range: **Last hour**, **Last 24 hours** (default), **Last 7 days**, or **Last 14 days**, by when Dagu Console recorded the run

Each run has a permalink to a page that lists its attempts. A run in progress on a server that has not reported for 5 minutes shows **Unknown**.

### Alerts

**Open** and **Resolved** alerts, filtered by server. Each alert shows the workflow, the server, when it opened, the last failure, the number of failures, the last run, the failed steps, and when it was emailed.

## Alerts and Email

An alert belongs to one workflow on one server and needs the `runs` level.

- A run with status failed opens an alert. Further failures add to its count.
- A later run that succeeds or partially succeeds resolves it.
- Aborted and rejected runs count as failures in totals, but they neither open nor resolve an alert.
- A server that stops reporting opens no alert. It appears under **Needs attention** on the **Overview** page.

Workspace owners and admins are emailed from `no-reply@dagu.sh` when an alert opens and when it resolves. Alerts that open or resolve together arrive in one email, which links to the **Alerts** page.

To stop the emails, a workspace owner opens **Settings** in Dagu Console, clears **Email alerts for Dagu servers** under **Monitoring**, and selects **Save changes**. Alerts are kept either way. The setting applies to alerts that open afterwards. An alert that is already open keeps the setting it opened with.

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

Each workspace stores up to 200 finished runs per UTC day, counted by when Dagu Console records them. Past the limit, runs are still counted in the daily totals and still open and resolve alerts, but they are not listed until 00:00 UTC. Until then:

- the console's **Overview** shows **Daily run limit reached** with the time;
- workspace owners get one email that day;
- the server's **Monitoring** section on **Plan & features** shows **Run history paused until** 00:00 UTC.

To raise the limit, [contact the Dagu team](https://dagu.sh/contact).

## Turn It Off

- **On the server**: select **Turn off** in the **Monitoring** section of **Plan & features**, or set `cloud.report: off` and restart Dagu. Reporting stops at once, including a report in flight. Turning it on later does not send the runs that finished in between.
- **Delete reports**: the server then shows **Not reporting** in Dagu Console. To remove what it sent, a workspace owner selects **Delete reports** on the server's **Settings** tab. This deletes the server's health reports, run history, and alerts. A server that still has monitoring on reports again, so turn it off on the server first.
- **Disconnect**: disconnecting the server, from **Plan & features** or with **Disconnect server** in Dagu Console, also stops reporting and removes the server from the monitoring pages. See [Check-ins and disconnecting](/overview/self-host-license#check-ins-and-disconnecting).

## Related Pages

- [Connect a Server](/overview/self-host-license#connect-a-server)
- [Configuration Reference](/server-admin/reference)
- [Notifications](/web-ui/notifications)
- [Incident Routing](/web-ui/incidents)
- [Prometheus Metrics](/server-admin/prometheus-metrics)
