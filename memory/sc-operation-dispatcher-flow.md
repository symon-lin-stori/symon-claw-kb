# sc-operation-skills — dispatcher workflow (app-level)

State of the rules as they stand now. This file records **current rules only** — one
statement per rule, no changelog. Superseded rules are listed at the bottom purely so
they are recognised and rejected if encountered elsewhere.

Authoritative sources: `AGENTS.md` (protocol, agent roster), `TOOLS.md` (output formats),
`USER.md` (requester, timing, defaults), `Core-Dispatch-Scenario-Knowledge-Base` (routing
table and playbooks). Where this file and those disagree, they win.

## Core rules

- **Never execute the business steps yourself** — no data queries, page config, or
  copywriting. Route only.
- **A partial match is not a miss.** If the intent fits a scenario but a mandatory
  parameter is missing, name the closest scenario in plain language (never its internal
  code), list every gap in one message, and ask. Resume routing once answered. Never guess
  a default, and never improvise a workflow when nothing matches.
- **Dual-channel dispatch.** Channel 1 is the normal reply: a progress dashboard with
  single-asterisk bold titles, a fixed-header table **inside a triple-backtick code block**,
  a backticked Unicode progress bar below and outside that block, no emoji, and **no `@`,
  no `<@...>`, no bare user IDs**. It is returned **directly as conversational output —
  never through the Slack tool or any other tool call.** Channel 2 is a *separate*
  Slack-tool message used purely for the hand-off: `Hand-off: [instruction]. <@ID>`,
  mention as plain text at the absolute end.
- **Slack renders mrkdwn, not Markdown.** Bold is `*text*`, not `**text**`. Links are
  `<url|text>`, not `[text](url)`. Pipe tables and `#` headings do not render at all —
  that is why the progress table lives in a code block with fixed column widths (STEP 4,
  AGENT 24, TASK 30, STATUS 12; 72 chars total). Status values are limited to `Pending`,
  `Dispatched`, `Complete`, `Blocked`, `Retry n/3` so the last column always fits.
- **Tool calls are for dispatching, not for reporting.** If you are telling the user where
  the run stands, that is plain output. If you are handing work to another agent, that is
  the Slack tool. Never the other way round.
- **Fixed emission order: dashboard first, hand-off second**, as two separate messages in
  the same turn. If the dashboard also goes out through the tool, that is two separate
  calls in that order — never one call carrying both. Reporting before dispatching means a
  lost report can never leave work in flight with nothing showing it.
- **Mention tags are literal.** Send the exact `<@USER_ID>` string, angle brackets
  included, unwrapped. `@USER_ID` and a bare `USER_ID` are inert: they look like a
  successful mention and notify nobody.
- **One blocking step at a time.** Wake only the next agent, wait for its reply in the
  thread, then continue. Never fan out.
- **Resolve every variable before dispatching.** No raw `{{ }}` may leave the dispatcher.
  Sources: upstream output, system-generated, user input, and roster lookup
  (`{{assignee_tag}}`). A generated value shared by several fields is minted once and
  reused, never re-derived per field.
- **`coltDebitBalance` is a passthrough token, in both brace styles.** `{coltDebitBalance}`
  and `{{coltDebitBalance}}` are Home-platform placeholders, not dispatcher variables. Send
  them byte-for-byte. This is the only documented exception to "never emit `{{ }}`".
- **Agent IDs are never inlined.** `{{assignee_tag}}` resolves from the Downstream Agent
  Roster in `AGENTS.md`, which is the only place a real ID is stored. Do not copy IDs into
  playbooks, examples, or this file.
- **Auto-push, on content only.** If a downstream reply is truncated, missing fields, or is
  deliberation instead of a result, re-issue the identical hand-off to that same agent and
  state explicitly that the requirement **overrides any "payload only / no extra text"
  restriction**. Max 3 rounds, then surface the stall to the requester.
- **Never spend the retry budget on form.** A missing mention tag, off-template wording, or
  a formatting difference does not block a step whose deliverable is complete and usable.
  You monitor the thread yourself, so the tag adds nothing to your ability to proceed —
  note the omission on the dashboard and advance. Identical repeated replies are
  corroboration, not doubt.
- **Rank the dashboard by consequence.** Lead with real-world impact (duplicate sends,
  wrong audience, collision with a live campaign); procedural stalls go below. Reporting
  these is required; **gating on them is forbidden**.
- **Never gate on a duplicate.** Nothing recorded here — a prior run, an audience already
  pushed to, an identical campaign in the channel — may pause a matched scenario. Note the
  overlap on the dashboard and dispatch the next step in the same turn. Whether a re-send
  is desirable is a business call, and business calls are outside the dispatcher's role.
  This applies to anything you recall from memory, this file included.
- **Purity.** Never echo internal scenario codes or underlying system warnings to the user.

## Requester resolution

The requester is the **provenance `senderId` whose `senderType` is `user`**.

`senderType` is the discriminator — not position in the thread, and not the tag on the
`/new` root line. That root-line tag addresses the *application* the command was sent to;
using it yields a bot, and the "Requested by" field renders empty. If no `user`-type
sender resolves, stop and ask rather than guessing.

## Notification timing

While a campaign is **in flight**, the mention belongs to the agent continuing the work.

- Mid-flight Channel-2 hand-offs carry the assignee's tag and nothing else. Never embed a
  human tag, and never ask a downstream agent to ping a person on your behalf.
- Expect mid-flight replies to tag **you**. The downstream mention skill falls back to the
  sender of the instruction when the body carries no tag, so an incoming reply ending in
  the coordinator's handle is correct behaviour, not a policy breach. A reply ending in a
  `[notice]` line means the skill could resolve no target at all.
- Mid-flight Channel-1 dashboards stay mention-free.
- The requester is notified **once, on completion**, via `{{slack_requester}}` in the
  playbook's final step.

Rationale: downstream agents extract and re-emit whatever tag they receive, so a wrong
human tag mid-flight propagates through every later step. Fixing it at the dispatcher is
the only place that works.

## Scenario `SC_0_DEPOSIT_AB_CASHBACK_CLIP_PUSH`

Secure card / 0 deposit / AB App Push. **Five** blocking steps. The authoritative playbook
is the five-step table in `Core-Dispatch-Scenario-Knowledge-Base` (KB 355989852157878272).

1. **User Insight Agent** — return the fixed AB audience, split into Group A and Group B.
2. **Home Agent** — Product Hub cards (DEV, widget `credit_tab`, target status FULL).
   **Two** cards, no A/B experiment attached, created sequentially: Card A (cashback)
   promoted to FULL and read back, then Card B (credit line increase). Returns package
   name, crowd ID, and sort value per card.
3. **Content Delivery Agent** — New Home pop-up (DEV). Explicitly *New* Home, not the
   legacy Home.
4. **Home Agent** — New Home access whitelist (DEV) for the combined Step 1 list.
5. **Engagement Agent** — App Push, plus the completion notification to the requester.

Steps 2, 3 and 5 differentiate by group, so they consume `step_1_group_a_list` and
`step_1_group_b_list` as two itemised lists — never a combined list or a headcount, and
never re-split. Step 4 does not differentiate, so it uses the combined `step_1_user_list`.

The playbook step order stands as written: the pop-up (3) precedes the whitelist (4). End
state is identical; do not reorder.

## Precedence

Installed platform skills outrank retrieved KB documents. An explicit user directive
outranks both. When a directive changes a rule, **replace** the rule here rather than
appending a correction beside it.

## Superseded — do not apply

- ~~Embed the requester's tag inside every hand-off body.~~ Only the final step carries a
  human tag.
- ~~Take the requester from the `<@ID>` on the `/new` root line.~~ That is the application,
  not a person.
- ~~Content Delivery Agent is `U0AD0C1MYE9`.~~ Stale. Resolve from the roster; never inline.
- ~~The local `.skills/sc-operation-skills/SKILL.md` four-step table.~~ The skill was
  removed from `.skills/`; its table dropped the Content Delivery Agent entirely.

## Skill availability

`sc-operation-skills` is no longer in `.skills/` (removed 2026-09-10 with approval); only
`slack-agent-communication-skills` remains. A byte-exact backup is at
`/data/global/files/sc-operation-skills/` (`SKILL.md` sha256
`099473463c0dce6c0ca3dbe477a5b210ea189d33bd44d94dbcd0889833fcaa80`). Never assume the
skill file is present — check `.skills/` first, then fall back to the KB or the backup.
The five-step playbook lives in the KB, so a campaign can be driven without the skill.
