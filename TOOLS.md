# TOOLS.md — Tool Inventory & Output Contracts

## Notation warning

Mention tags are written **unwrapped** throughout this file (`<@U0EXAMPLE01>`, not
`` `<@U0EXAMPLE01>` ``). This is deliberate. Backticks around a tag are copied verbatim
into outgoing messages, and a wrapped tag renders as literal text and fires no
notification. Never add quoting for readability.

## Mention Tag Literal Form

A mention tag is the **complete string `<@` + user ID + `>`**. The angle brackets are what
make Slack render it as a mention; without them it is inert text that notifies nobody
while still looking plausible in the sent message.

When resolving `{{assignee_tag}}` or `{{slack_requester}}`, copy the value from the
roster **character for character**. Do not reformat, prettify, or substitute a name.

`U0EXAMPLE01` below is a stand-in. Real IDs live only in the roster in `AGENTS.md` — never
copy one out of this file.

| | Example |
| --- | --- |
| Correct | `<@U0EXAMPLE01>` |
| Wrong — no angle brackets | `@U0EXAMPLE01` |
| Wrong — bare ID | `U0EXAMPLE01` |
| Wrong — display name | `@Content Delivery Agent` |
| Wrong — wrapped in Markdown | `` `<@U0EXAMPLE01>` `` |
| Wrong — link syntax | `[@Content Delivery Agent](...)` |

If you cannot produce the exact correct form — for example the ID is missing from the
roster — **stop and ask**. Sending a hand-off with an inert tag looks successful and
fails silently, which is worse than not sending it.

---

## Dual-Channel Dispatch

Every state transition produces output on **two independent channels**. They are not
alternatives, and their formatting rules are opposites. Emit both, **Channel 1 first**.

| | Channel 1 — Progress Dashboard | Channel 2 — Hand-off |
| --- | --- | --- |
| Delivery | Default text reply, **no tool call** | Forced call to the Slack tool, as a separate message |
| Audience | The human requester | The downstream agent |
| Formatting | Slack mrkdwn: single-asterisk bold, code-block table, backticked progress bar | Forbidden — plain text only |
| Mention tags | **Strictly none** | Exactly one, at the absolute end |
| Language | English | The scenario template's wording, verbatim |
| Order | Sent first | Sent second, after the dashboard |

### Emission order

Within one state transition: **dashboard, then hand-off**, as two separate messages, never
reversed and never merged into one. If your setup delivers the dashboard through the tool
as well, that is two separate tool calls in the same order — one call must never carry both
payloads.

The order matters because the dashboard is the record of what is about to happen. Dispatch
first and a lost report leaves work in flight with nothing showing it; report first and the
state is visible before the action. After the hand-off goes out, return to blocking — do
not batch, interleave, or defer either channel.

### Channel 1 — Progress Dashboard

Sent as your ordinary conversational reply. **Red line: never route the dashboard through
the Slack tool, or through any other tool call.** Progress reporting is plain output, not
an action. Tool invocation belongs to Channel 2 only.

**Red line: this channel must not contain any `@` symbol, any
`<@...>` tag, or any bare user ID.** Refer to agents by their display name only — write
"Content Delivery Agent", never `<@U0EXAMPLE01>`, `@U0EXAMPLE01`, or `U0EXAMPLE01`. This
applies to the Assignee Agent column of the progress table, where the temptation to paste
an ID is strongest.

**Slack renders mrkdwn, not standard Markdown.** Three differences matter here:

| Standard Markdown | Slack mrkdwn |
| --- | --- |
| `**bold**` | `*bold*` — double asterisks print literally |
| `[text](url)` | `<url\|text>` |
| Pipe tables, `#` headings | Not supported at all |

Inline code and triple-backtick code blocks work normally, which is what the layout below
relies on.

Structure:

- **Bold section titles**, single asterisks: `*Execution Progress*`,
  `*Node Output Summary*`, `*Action Items*`.
- **Execution Progress table**, inside a triple-backtick code block so the monospace font
  aligns space padding. Fixed headers and fixed widths — `STEP` 4, `AGENT` 24, `TASK` 30,
  `STATUS` 12, one space between columns, 72 characters total. That is the mobile width
  budget: shorten the wording to fit, never let a row wrap.
- **Status vocabulary**, to keep the last column inside 12 characters: `Pending`,
  `Dispatched`, `Complete`, `Blocked`, `Retry n/3`. Nothing else.
- **Backticked progress bar**, placed *below and outside* the code block — backticks do
  not nest, so a bar inside the fence loses its formatting. Plain-text Unicode blocks,
  emojis strictly prohibited.
- **Node Output Summary.** When an agent completes its step, synthesize its results into a
  structured list below the table. Use a second code block only when the content needs
  column alignment.

Reference layout:

````
*Execution Progress*

```
STEP  AGENT                    TASK                          STATUS
1     User Insight Agent       Fixed A/B audience            Complete
2     Home Agent               Two Product Hub cards (DEV)   Dispatched
3     Content Delivery Agent   New Home pop-up (DEV)         Pending
4     Home Agent               New Home whitelist (DEV)      Pending
5     Engagement Agent         A/B Push + completion         Pending
```

Progress: `██░░░░░░░░` 1/5 · Step 2

*Node Output Summary*
*Step 1 — User Insight Agent*: Group A 5 users, Group B 5 users.
````

### Channel 2 — Hand-off

You **must** force a call to the Slack tool to send a separate, independent message. This
message exists only to dispatch a task. Do not fold it into the dashboard reply.

Format:

```
Hand-off: [instruction details] <@AGENT_ID>
```

Rules, all of which are hard requirements:

1. The instruction body is the playbook's template with **all variables already
   resolved**. No `{{ }}` may survive into a sent message.
2. Strip any `<@...>` tag that the body would otherwise contain mid-sentence. The
   requester's tag appears inside the sentence **only in a playbook's final step** — the
   trailing tag is always the **assignee's**. Mid-flight hand-offs contain no human tag
   at all.
3. The trailing tag is **plain text**. No backticks, no bold, no code fence.
4. The trailing tag is the **last thing in the message**. No period, no exclamation mark,
   no newline, no trailing note after it.
5. There is **exactly one** trailing tag. One hand-off addresses one agent.
6. Append the tag on both success and failure paths — a status message with no tag
   notifies nobody.

Worked example:

```
Hand-off: Configure New Home access whitelist. Target environment: DEV. Target list: U123, U456, U789. <@U0EXAMPLE01>
```

## Tool Inventory

| Tool | Use for | Notes |
| --- | --- | --- |
| Slack send-message tool | All Channel 2 hand-offs, and follow-ups on incomplete replies | Mandatory. Never emulate a hand-off by writing it into the dashboard reply. |
| `read_file` / `write_file` / `edit_file` | Workspace context files | Read before writing. Prefer `edit_file` for small changes. |
| `glob` / `grep` | File-name discovery and content search | Cheaper than reading whole files. |
| `exec` / `python` | Local analysis, validation, formatting | Keep output small; redirect noisy output. |
| Skills (`.skills/{slug}/SKILL.md`) | Documented procedures | Read the skill before invoking the procedure it describes. |

## Credentials & Paths

- Slack credentials are supplied by the runtime. Do not hardcode tokens, and do not echo
  them into chat or into workspace files.
- Uploaded files arrive in `uploads/`.
- Scratch analysis belongs in `.tmp/`; clean it up when the task ends.

## Gotchas

- **The Slack tool is not optional.** If a hand-off appears only in your text reply, the
  downstream agent never receives it and the workflow silently stalls.
- **Backticked tags do not notify.** This is the single most common failure. The scenario
  knowledge base currently displays tags wrapped in backticks for readability — strip
  them before sending.
- **The two channels drift apart under pressure.** When a step fails, the instinct is to
  explain the failure in the hand-off and mention the requester in the dashboard. Both
  are wrong. Failures are explained on the dashboard; the requester is only ever
  mentioned in Channel 2.
- **System warnings are not user-facing.** If the runtime emits something like
  "slack notification is not configured...", never repeat it verbatim. Translate it into
  a plain statement of what is blocked.
- **Downstream agents may be configured to return payload only.** Some will drop the
  trailing mention tag because their own prompt forbids extra text. When a template needs
  the tag echoed back, state explicitly that the requirement overrides any
  "return the payload only / no extra text" restriction.
