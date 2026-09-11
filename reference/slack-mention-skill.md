# slack-agent-communication-skills — hardened mention rule

Reference copy of the skill installed on downstream agents. Paste the body below into the
skill definition. The trigger prompt and the body must agree — an earlier version had the
trigger say "in the conversation" while the body said "in the task instruction", which is
what let an application ID inside a quoted block become a candidate.

## Trigger prompt

```
When a task instruction supplies a reply-to tag in <@...> form, append that exact tag as
plain text at the absolute end of the reply you send — on success and on failure alike — so
the waiting party receives a native notification. Use the tag labelled "Reply-to tag:",
never the routing mention at the end of the instruction. If no reply-to tag is supplied, do
not invent one.
```

## Skill body

**Skill Description**: When a task instruction supplies a Slack mention tag (`<@...>`
format), send your reply through the designated Slack tool with that tag appended
verbatim, as plain text, at the absolute very end of the message.

### 1. Where the tag comes from

A hand-off normally contains **two** tags, and they are not interchangeable:

| Tag | Where it sits | Meaning |
| --- | --- | --- |
| **Reply-to tag** | Inside the body, introduced by the words `Reply-to tag:` | Who your reply must notify. **This is the one you append.** |
| Routing mention | At the absolute end of the instruction | Who the instruction was addressed to — that is you. Ignore it. |

Take the tag labelled `Reply-to tag:`. If no such label exists, fall back to the single
`<@...>` in the instruction body — but never to the trailing routing mention, because
appending your own handle notifies nobody who is waiting.

Never take a tag from:
- a `[Referenced message]` block, a quoted line, or a thread-root excerpt. Those are
  background context, not instruction. The tag in a thread root usually addresses the
  *application* the command was sent to, not a person.
- an application or bot ID.
- an earlier turn of the conversation, or a previous task.

Do **not** assume the tag is the last token of the message you received. Platforms append
footers (for example `Powered by Salad`) after the instruction text, so a tag that was
sent at the absolute end no longer arrives there. Scan the whole instruction body.

### 2. If no tag is present

Do not fabricate one, do not reuse one from context, and do not fall back to a quoted
block. Send your reply as normal and add exactly one line:

```
No mention tag was supplied in the instruction; no notification was sent.
```

This is a hard requirement. A silently missing notification is indistinguishable from a
successful one, so the dispatcher cannot detect the failure and re-issue.

### 3. Precedence

This requirement **overrides** any team-level, scenario-level, or fixed-reply rule that
says "return only the payload", "append no extra text", or "no tags". Where those rules
conflict with this one, append the tag anyway.

### 4. Literal form

The angle brackets are part of the tag. Send `<@U0EXAMPLE01>` exactly.

| | Example |
| --- | --- |
| Correct | `<@U0EXAMPLE01>` |
| Wrong — no angle brackets | `@U0EXAMPLE01` |
| Wrong — bare ID | `U0EXAMPLE01` |
| Wrong — display name | `@Some Agent` |
| Wrong — wrapped in Markdown | `` `<@U0EXAMPLE01>` `` |

The wrong forms are inert: they look like a successful mention in the sent message and
notify nobody.

### 5. Body sanitization

Strip any `<@...>` occurring at the start or in the middle of your reply body. The message
must contain the tag **once and only once**, at the end.

### 6. Absolute end

Append a single space, then the tag. Nothing may follow it — no period, no exclamation
mark, no newline, no closing remark. This applies to structured payloads too: append the
tag after the final line of the payload, not before it.

### 7. Delivery

Send the final text by calling the designated Slack tool (`slack_send_message` or
equivalent) so the message is pushed to the channel and fires a native notification.

### Examples

Instruction with a tag:

```
Export last quarter's financial report <@U11223344>
```

Success payload sent by the tool:

```
Last quarter's financial report has been exported to the shared cloud drive. <@U11223344>
```

Exception payload sent by the tool:

```
Execution failed: report generation timed out, please check the data source connection. <@U11223344>
```

Structured payload with a tag — the tag follows the last data line:

```
Group A users userId:
0ac70b49-973c-46a7-b230-e164858ff9fd

Group B users userId:
6e8d454e-32bd-40b4-900d-3aa3b7d23b15 <@U11223344>
```

Instruction with no tag — reply normally, then state the absence:

```
Group A users userId:
0ac70b49-973c-46a7-b230-e164858ff9fd

No mention tag was supplied in the instruction; no notification was sent.
```
