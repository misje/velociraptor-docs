---
title: "Alerts and e-mail notifications in Velociraptor"
description: |
    How to use Velociraptor's alert system and new exchange artifacts to get
    e-mail notifications for flow completions, detection hits, and operational
    errors.
date: 2026-04-19T00:00:00Z
draft: true
tags:
  - alerts
  - notifications
  - email
  - detection
---

<!-- TODO: Expand all sections. Screenshots needed throughout. -->

Velociraptor runs a lot of things silently in the background: event queries on
endpoints, server-side monitoring, hunts, scheduled collections. Getting
notified when something finishes, fails, or finds a match requires some
configuration. This post walks through a set of new exchange artifacts that
cover the most common notification needs, and explains how they fit together.

## The building blocks

Three things underpin all e-mail notifications in Velociraptor:

- **SMTP secrets** store connection credentials. See
  [How to send e-mails from Velociraptor]({{< ref "/knowledge_base/tips/sending_email/" >}}).
- **[`Generic.Utils.SendEmail`]({{< ref "/artifact_references/pages/generic.utils.sendemail/" >}})**
  handles MIME encoding, HTML, and attachments, so other artifacts do not have
  to.
- **[`alert()`]({{< ref "/vql_reference/other/alert/" >}})** routes a message
  to `Server.Internal.Alerts`, a central server-side queue that server event
  artifacts can watch and act on.

If you have not set up SMTP yet, start with the
[sending e-mail guide]({{< ref "/knowledge_base/tips/sending_email/" >}}).
It also covers testing locally with Mailpit, which is useful before connecting
a real mail server.

## Flow completion notifications

<!-- TODO: screenshot of a flow completion e-mail -->

[`Server.Monitor.FlowCompletion`]({{< ref "/exchange/artifacts/pages/server.monitor.flowcompletion/" >}})
sends an e-mail whenever a client flow completes. It supports filtering by
artifact name, client label, hunt participation, flow outcome, and more, and
can include inline result tables or attachments in the notification.

Some example use cases:

- Get notified when an offline client finally completes a collection.
- Audit all use of EXECVE-permission artifacts.
- Notify the device owner when data is collected from their endpoint.

See [How to set up e-mail notifications for flow completions]({{< ref "/knowledge_base/tips/email_alerts/" >}})
for a full configuration guide.

## Alerts

<!-- TODO: screenshot of an alert e-mail -->

The [`alert()`]({{< ref "/vql_reference/other/alert/" >}}) function lets a VQL
artifact signal that something worth immediate attention has happened. Unlike
`log()`, which writes to the artifact's own log, `alert()` routes the message
to `Server.Internal.Alerts` on the server, where it can be picked up by
[`Server.Monitor.Alerts`]({{< ref "/exchange/artifacts/pages/server.monitor.alerts/" >}})
and forwarded by e-mail.

Velociraptor itself does not currently use `alert()` — it is entirely a
user-space mechanism. Detection artifacts are a natural fit: a honeyfile
monitor, a YARA scanner, a network connection check against an IOC list. When
a match is found, call `alert()` with whatever context helps you act on the
finding.

```vql
SELECT alert(
    name="Honeyfile accessed",
    Path=FullPath,
    ProcessName=Process.Name,
    Pid=Process.Pid
)
FROM ...
```

`Server.Monitor.Alerts` picks that up and sends an HTML e-mail with the alert
context, client details, and optional severity classification.

See [Using alerts in Velociraptor]({{< ref "/knowledge_base/tips/vql_alerts/" >}})
for the full picture, including the `dedup` parameter, severity transforms, and
context formatting.

## Monitoring event artifact errors

<!-- TODO: screenshot of an error alert e-mail -->

Event artifacts can fail silently. A missing executable, a VQL syntax error, an
eBPF conflict — all of these write to the monitoring log but produce no visible
notification.

[`Server.Monitor.Client.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.client.errors.alert/" >}})
and
[`Server.Monitor.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.errors.alert/" >}})
periodically inspect those logs and call `alert()` for entries that match a
configurable filter. Combined with `Server.Monitor.Alerts`, this means you get
an e-mail whenever a monitoring artifact on a client or on the server starts
failing.

See [How to monitor event artifact errors]({{< ref "/knowledge_base/tips/monitoring_artifact_errors/" >}}).

## Putting it together

<!-- TODO: architecture diagram showing the flow from alert() to email -->

A complete setup looks like this:

1. Add an SMTP secret (type **SMTP Creds**).
2. Add `Server.Monitor.FlowCompletion` as a server event artifact for flow completion
   notifications.
3. Add `Server.Monitor.Alerts` as a server event artifact for alert-based
   notifications.
4. Add `Server.Monitor.Client.Errors.Alert` and `Server.Monitor.Errors.Alert`
   to catch failing event queries.
5. Write detection artifacts that call `alert()` when they find something.

The error monitoring and detection alerts both feed into `Server.Monitor.Alerts`,
so a single artifact handles all alert-to-email routing.
