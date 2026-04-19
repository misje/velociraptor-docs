# How to monitor event artifact errors

<!-- TODO: Expand this article -->

Client and server event artifacts run continuously in the background, and when
they fail the errors are written to the monitoring log — not surfaced anywhere
visible. A VQL syntax error, a missing executable, an eBPF policy conflict:
all of these can silently break monitoring that you depend on.

[`Server.Monitor.Client.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.client.errors.alert/" >}})
and
[`Server.Monitor.Errors.Alert`]({{< ref "/exchange/artifacts/pages/server.monitor.errors.alert/" >}})
periodically inspect those logs and call `alert()` for matching entries, which
can then be forwarded by e-mail via
[`Server.Monitor.Alerts`]({{< ref "/exchange/artifacts/pages/server.monitor.alerts/" >}}).

---

### Server.Monitor.Client.Errors.Alert

Inspects the monitoring logs for all clients that have active event artifacts.
Useful for catching errors in artifacts like file access monitors, eBPF probes,
or network monitors that run on endpoints.

<!-- TODO: Screenshot of IncludeFilter configuration -->

---

### Server.Monitor.Errors.Alert

Same as above, but for server event artifacts. Useful for catching errors in
`Server.Monitor.Alerts` itself, or in any other server-side monitoring artifact.

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

{{% notice warning %}}

Many errors from native VQL functions are logged at level `DEFAULT`, not
`ERROR`. Make sure your filters include `DEFAULT` if you want to catch these.

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
from the Artifact Exchange. Then add `Server.Monitor.Client.Errors.Alert`,
`Server.Monitor.Errors.Alert`, and `Server.Monitor.Alerts` as server event
artifacts, pointing all of them at the same SMTP secret. See
[Using alerts in Velociraptor]({{< ref "/knowledge_base/tips/vql_alerts/" >}})
for `Server.Monitor.Alerts` configuration details.

Tags: #alerts #monitoring #notifications #troubleshooting
