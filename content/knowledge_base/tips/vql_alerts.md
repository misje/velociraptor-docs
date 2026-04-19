# Using alerts in Velociraptor

<!-- TODO: Expand this article -->

The [`alert()`]({{< ref "/vql_reference/other/alert/" >}}) function routes a
message into the `Server.Internal.Alerts` event queue. Alerts are intended for
high-value, low-frequency events that require attention: a detection artifact
found a match, a honeyfile was accessed, a network connection matched an IOC.

This is distinct from `log()`, which records diagnostic information in the
artifact's own log. Alert messages are collected centrally on the server and
can be acted on by a server event artifact such as
[`Server.Monitor.Alerts`]({{< ref "/exchange/artifacts/pages/server.monitor.alerts/" >}}),
which forwards them by e-mail.

---

### Creating an alert

```vql
SELECT alert(
    name="Honeyfile accessed",
    Path=FullPath,
    ProcessName=Process.Name,
    Pid=Process.Pid
)
FROM ...
```

The `name` argument is required. All other keyword arguments are passed through
as context and appear in the notification. The more relevant context you add,
the more useful the resulting notification will be.

#### Deduplication

By default, identical alert names are suppressed for 2 hours
(`dedup=7200`). Set `dedup=-1` to disable deduplication entirely, or set a
shorter interval when testing.

---

### What to use `alert()` for

Velociraptor does not use `alert()` internally — it is a user-space mechanism.
Good candidates are:

- Detection hits in client event artifacts (file access, process execution,
  network connections matching IOCs)
- Sigma or YARA matches from monitoring artifacts
- Client-side conditions that indicate something worth investigating immediately

For operational problems — event query errors, failing artifacts — see
[How to monitor event artifact errors]({{< ref "/knowledge_base/tips/monitoring_artifact_errors/" >}}).

---

### Adding context

Any keyword arguments passed to `alert()` beyond `name` and `dedup` are
available in `event_data` when the alert is received by `Server.Monitor.Alerts`.
Pass whatever fields help identify the event:

```vql
SELECT alert(
    name="Suspicious network connection",
    dedup=300,
    RemoteAddr=RemoteAddr,
    LocalAddr=LocalAddr,
    Pid=Pid,
    ProcessName=ProcessName,
    CommandLine=CommandLine
)
FROM ...
```

---

### Receiving alerts by e-mail

[`Server.Monitor.Alerts`]({{< ref "/exchange/artifacts/pages/server.monitor.alerts/" >}})
watches `Server.Internal.Alerts` and sends an e-mail for each matching alert.
If your server has internet access, run
[`Server.Import.Extras`]({{< ref "/artifact_references/pages/server.import.extras/" >}})
to import it. Then add it as a server event artifact and point it at your SMTP
secret.

Key parameters:

- `Secret` — SMTP secret name (required)
- `Recipients` — who to notify
- `SeverityTransforms` — derive a normalised severity string from context fields
- `SeverityThreshold` — only notify for alerts at or above a given severity
- `ContextInclude` / `ContextExclude` — control which context fields appear in
  the notification
- `FlattenContext` — flatten nested dicts in the context for readability

{{< figure src="alert_email.png" caption="An alert notification in Mailpit" >}}

#### Severity

If your alert context includes a field such as `level` or `severity`, you can
use `SeverityTransforms` to map it to a normalised value. For example:

```
Member,Regex,Replace
level,(?i)warning,medium
level,(?i)critical,high
```

Set `SeverityThreshold` to `["medium", "high"]` to suppress low-severity
alerts. The derived severity appears in the notification subject and body.

---

### Example: honeyfile detection

<!-- TODO: full example artifact + screenshot -->

A client event artifact monitors a set of sensitive files for access. When a
match is found, `alert()` is called with the file path, the process name and
PID, and the accessing user.

Tags: #alerts #vql #detection #notifications
