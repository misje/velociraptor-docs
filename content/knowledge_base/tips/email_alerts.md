# How to set up e-mail notifications for flow completions

<!-- TODO: Expand this article -->

[`Server.Monitor.FlowCompletion`]({{< ref "/exchange/artifacts/pages/server.monitor.flowcompletion/" >}})
sends an e-mail when a client flow completes, with support for HTML formatting,
inline result tables, and file attachments. It monitors
`System.Flow.Completion` and applies a configurable set of filters before
deciding whether to send a notification.

Before setting this up, make sure you have an SMTP secret configured. See
[How to send e-mails from Velociraptor]({{< ref "/knowledge_base/tips/sending_email/" >}}).

---

### Installation

<!-- TODO: Screenshot of adding the artifact as a server event monitor -->

If your server has internet access, run
[`Server.Import.Extras`]({{< ref "/artifact_references/pages/server.import.extras/" >}})
to import all exchange artifacts, including `Server.Monitor.FlowCompletion`. Otherwise,
copy the artifact definition manually from the
[Artifact Exchange]({{< ref "/exchange/artifacts/pages/server.monitor.flowcompletion/" >}}).

Add `Server.Monitor.FlowCompletion` as a server event artifact. Set `Secret` to the name
of your SMTP secret. Configure at least one recipient in `Recipients`, or
enable `NotifyExecutor` to notify whoever scheduled the flow.

---

### Filtering

By default, `Server.Monitor.FlowCompletion` notifies on all flows except
`Generic.Client.Info`. The main parameters for controlling what triggers a
notification are:

- `ArtifactsToAlertOn` — regex; only flows collecting matching artifacts notify
- `ArtifactsToIgnore` — regex; suppresses notifications for single-artifact flows
- `ClientLabelsToAlertOn` / `ClientLabelsToIgnore` — filter by client label
- `NotifyHunts` — include flows that are part of a hunt (off by default)
- `DelayThreshold` — only notify if the flow took longer than N seconds to
  complete (default 10 s); useful for skipping flows that complete immediately

---

### Throttling

`SendInterval` controls how many seconds must pass between notifications.
The default is 10 seconds. Set it to `-1` to disable throttling once you have
confirmed the configuration works correctly. See the
[throttling section]({{< ref "/knowledge_base/tips/sending_email/#throttling" >}})
in the e-mail setup guide for how dropped messages are logged.

---

### E-mail content

`Server.Monitor.FlowCompletion` sends HTML by default (`HTML=true`). The message includes
client details, flow details, and optionally:

- Inline result tables from selected artifact sources (`IncludeResultTableFrom`)
- JSONL or CSV attachments (`IncludeResultAttachmentFrom`)
- Direct download links to uploaded files (`IncludeUploadsTableRows`)

{{< figure src="mail_flow_completion.png" caption="An HTML flow completion notification in Mailpit" >}}

---

### Example use cases

<!-- TODO: expand with screenshots and full parameter examples -->

#### Get notified when an offline client completes a collection

You have scheduled a collection on a client that is not currently online.
Set `DelayThreshold` to a value larger than the expected round-trip time so
you only get notified when the client actually had to wait. Enable
`NotifyExecutor` if you want the notification sent to your own address.

#### Audit privileged artifact use

Set `ArtifactPermToAlertOn` to `EXECVE` to get a notification whenever a flow
completes that includes an artifact requiring shell access. Set
`DelayThreshold` to `0` to catch even fast completions.

#### Notify the device owner

If client metadata includes an e-mail field, set `NotifyMetadataEMail` to that
field name. The owner is notified whenever a collection on their device
completes.

Tags: #email #notifications #flows #configuration
