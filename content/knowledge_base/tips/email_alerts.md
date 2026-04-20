# How to set up e-mail notifications for flow completions

<!-- TODO: Expand this article -->

[`Server.Monitor.FlowCompletion`]({{< ref "/exchange/artifacts/pages/server.monitor.flowcompletion/" >}})
sends an e-mail when a client flow completes, with support for HTML formatting,
inline result tables, and file attachments. It monitors
`System.Flow.Completion` and applies a configurable set of filters before
deciding whether to send a notification.

An SMTP secret is required. See
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

For `IncludeResultTableFrom`, rows, columns, and cell values are automatically truncated
when they exceed the hard limits: 100 rows, 4 columns (`ResultTableMaxColumns`), and
10,000 characters per cell. Tables with many columns render poorly in e-mail clients even
within these limits. Only configure inline tables for specific artifact sources with
compact, predictable output, and use the `Columns` regex to select only the fields you need.

`IncludeResultAttachmentFrom` has no row or column limits, but if the total size of all
attachments exceeds `AttachmentsMaxMiB` (default 100 MiB), all attachments are dropped.
Keep the source selection specific here as well.

---

### Example use cases

<!-- TODO: screenshots and full parameter tables for each example -->

#### Notify the analyst who ran the collection

Enable `NotifyExecutor`. If the Velociraptor username is an e-mail address,
the result is sent directly to whoever scheduled the flow, with no fixed
`Recipients` list needed. If usernames are not e-mail addresses, use
`NotifyExecutorDomains` to map them: a row `.+,example.org` appends
`@example.org` to any username.

This works well as a general setting so analysts naturally receive results from
their own collections without needing to poll the GUI.

#### Notify when an offline client finally checks in

You have scheduled a collection on an offline client.
Set `DelayThreshold` to something larger than the expected round-trip time
(e.g. 300 for five minutes) so you are only notified when the client had to
wait. Use `NotifyExecutor` to send the result to the analyst who scheduled it,
and set `IncludeResultAttachmentFrom` to attach the results directly to the
e-mail so the analyst does not need to open the GUI at all.

#### Get notified when hunt flows fail

You are running a large hunt and want to know about failing clients without
being flooded by success notifications. Leave `NotifyHunts` off (so successful
hunt flows are silent), but add `IncludeHunts` to `ErrorHandling`. Failed
flows in the hunt will still notify, regardless of the `NotifyHunts` setting.

Combine with `IgnoreArtifactFilters` in `ErrorHandling` if you want failure
notifications regardless of `ArtifactsToIgnore`.

#### Audit shell and EXECVE artifact use

Set `ArtifactPermToAlertOn` to `EXECVE` and `DelayThreshold` to `0` to get a
notification for every completed flow that includes an artifact requiring shell
access. Set `Recipients` to a security mailbox.

To include the command output directly in the e-mail body, configure
`IncludeResultTableFrom`:

| Source | Columns | MaxRows | CellLimit |
| ------ | ------- | ------- | --------- |
| bash\|powershell | Stdout\|Stderr | 20 | 1000 |
| generic\.client\.vql$ | .+ | | 3000 |

#### Alert on results from priority clients

Apply the label `notify_results` to clients that warrant immediate attention
regardless of other filters: a server under active investigation, a VIP's
workstation, a honeypot. Whenever a flow on such a client produces any results,
`NotifyIfResultsLabels` (default `^notify_results$`) causes a notification to
be sent, bypassing `ArtifactsToIgnore`, `DelayThreshold`, and label filters.

The same mechanism works for uploads: use `NotifyIfUploadsLabels` with a label
such as `notify_uploads` on clients where any file collection should always
trigger a notification.

#### New client enrolled

Set `NewClientArtifacts` to a regex matching the artifacts you collect on
enrollment (e.g. `Generic.Client.Info`). When a client that is newer than
`NewClientThreshold` seconds completes such a flow, a notification is sent
regardless of other filters. Useful for getting an e-mail whenever a new
endpoint appears on the server.

#### Notify the device owner

If client metadata includes the device owner's e-mail address, set
`NotifyMetadataEMail` to that field name. The owner is notified whenever a
collection on their device finishes. This can run alongside `Recipients` and
`NotifyExecutor`, so the analyst, a central mailbox, and the device owner all
receive the notification independently.

Tags: #email #notifications #flows #configuration
