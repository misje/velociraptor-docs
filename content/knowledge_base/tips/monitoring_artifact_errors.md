# How to monitor event artifact errors

Client and server event artifacts run continuously in the background, and when
they fail the errors are written to the monitoring log, not surfaced anywhere
visible. A VQL syntax error, a missing executable, a failed S3 upload or unsuccessful
JSON parsing: all of these can silently break monitoring.

Monitoring logs are available in the **Server Events** interface, but you are not
notified about errors or warnings in these logs out of the box.

![Server event logs for a selected artifact](server_event_logs.png)

[`Server.Monitor.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.errors.alert/" >}})
and
[`Server.Monitor.Client.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.client.errors.alert/" >}})
periodically inspect those logs and call [`alert()`]({{< ref "/vql_reference/other/alert/" >}}) for matching entries, which
can then be forwarded by e-mail via
[`Server.Monitor.Alerts`]({{< ref "/exchange/artifacts/pages/server.monitor.alerts/" >}}).


### Server.Monitor.Errors.Alert

If you have any custom server event artifacts, you have likely configured some automation, like fetching data from APIs, uploading data to S3/Elastic etc.
You probably want to be notified if any of this automation fails.

### Server.Monitor.Client.Errors.Alert

Unlike server event artifacts, client event artifacts run on many endpoints.
You probably do not want the same kind of log monitoring as for server event
artifacts, but this artifact allows you to do so, if deemed necessary. Perhaps
you run important monitoring on some tagged endpoints, and it is critical that
this monitoring is running without errors.

---

### Configuring filters

Both artifacts share the same filter model. `IncludeFilter` is a CSV table
with columns `Artifact`, `Level`, and `Message` (all regexes), plus an
optional `Severity` column and any additional custom columns:

| Artifact | Level | Message | Severity |
| -------- | ----- | ------- | -------- |
| .+ | ERROR | .+ | medium |
| .+ | DEFAULT | fork/exec .+ no such file or directory | high |

Rows are matched top-to-bottom; the first match wins. `ExcludeFilter` works
the same way and is applied after `IncludeFilter`.

{{% notice info %}}

Many errors from native VQL functions are logged at level `DEFAULT`, not
`ERROR`. Include `DEFAULT` in your filters to catch these.
[`Server.Monitor.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.errors.alert/" >}})
includes a list of known functions that log at `DEFAULT`.

{{% /notice %}}

#### Custom columns

Any column added to `IncludeFilter` beyond the built-in ones is passed through
as extra context on the alert. This is useful for attaching a human-readable
description or a suggested action to a known error pattern:

| Artifact | Level | Message | Severity | Explanation |
| -------- | ----- | ------- | -------- | ----------- |
| .+ | DEFAULT | watch_ebpf: .+ already exist | medium | eBPF policy conflict |
| .+ | ERROR | .+ | medium | |

---

### Deduplication

Both artifacts suppress repeated alerts using `DedupInterval` (default 3600 s).
Deduplication is keyed on the alert name, which includes the artifact name, so
at most one alert is produced per artifact per interval. Check the monitoring
log directly if you suspect there are more errors than the alerts indicate.

---

### Routing to e-mail

If your server has internet access, run
[`Server.Import.Extras`]({{< ref "/artifact_references/pages/server.import.extras/" >}})
to import all three artifacts at once. Otherwise, copy the definitions manually
from the Artifact Exchange. Then add [`Server.Monitor.Client.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.client.errors.alert/" >}}),
[`Server.Monitor.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.errors.alert/" >}}), and [`Server.Monitor.Alerts`]({{< ref "/exchange/artifacts/pages/server.monitor.alerts/" >}}) as server event
artifacts, pointing all of them at the same SMTP secret. See
[Using alerts in Velociraptor]({{< ref "/knowledge_base/tips/vql_alerts/" >}})
for `Server.Monitor.Alerts` configuration details.

## See also

- Monitor client event artifact errors: [`Server.Monitor.Client.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.client.errors.alert/" >}})
- Monitor server event artifact errors: [`Server.Monitor.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.errors.alert/" >}})
- Forward alerts by e-mail: [`Server.Monitor.Alerts`]({{< ref "/exchange/artifacts/pages/server.monitor.alerts/" >}})
- [Using alerts in Velociraptor]({{< ref "/knowledge_base/tips/vql_alerts/" >}})
- [Alerts and e-mail notifications in Velociraptor]({{< ref "/blog/2026/2026-04-19-alerts-and-email/" >}})


Tags: #alerts #monitoring #notifications #troubleshooting