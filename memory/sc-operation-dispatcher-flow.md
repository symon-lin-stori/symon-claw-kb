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
- **Dual-channel dispatch.** Channel 1 is the normal reply: a progress dashboard with bold
  section titles, a fixed-header Markdown table, a backticked Unicode progress bar, no
  emoji, and **no `@`, no `<@...>`, no bare user IDs**. Channel 2 is a *separate* Slack-tool
  message used purely for the hand-off: `Hand-off: [instruction]. <@ID>`, mention as plain
  text at the absolute end.
- **Mention tags are literal.** Send the exact `<@USER_ID>` string, angle brackets
  included, unwrapped. `@USER_ID` and a bare `USER_ID` are inert: they look like a
  successful mention and notify nobody.
- **One blocking step at a time.** Wake only the next agent, wait for its reply in the
  thread, then continue. Never fan out.
- **Resolve every variable before dispatching.** No raw `{{ }}` may leave the dispatcher.
  Sources: upstream output, system-generated (`uuid`), user input, and roster lookup
  (`{{assignee_tag}}`).
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
  wrong audience, collision with a live campaign); procedural stalls go below.
- **Purity.** Never echo internal scenario codes or underlying system warnings to the user.

## Requester resolution

The requester is the **provenance `senderId` whose `senderType` is `user`**.

`senderType` is the discriminator — not position in the thread, and not the tag on the
`/new` root line. That root-line tag addresses the *application* the command was sent to;
using it yields a bot, and the "Requested by" field renders empty. If no `user`-type
sender resolves, stop and ask rather than guessing.

## Notification timing

While a campaign is **in flight**, the mention belongs to the agent continuing the work.

- Mid-flight Channel-2 hand-offs carry two tags: `{{dispatcher_tag}}` inside the body,
  labelled `Reply-to tag:`, and the assignee's routing mention at the absolute end. Never
  embed a **human** tag mid-flight, and never ask a downstream agent to ping a person.
- The reply-to tag is what makes the downstream mention skill work at all: that skill
  extracts a tag from the instruction, so an instruction with no tag produces a reply with
  no tag. Mid-flight the reply-to tag is the coordinator; in the final step it is the
  requester.
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
2. **Home Agent** — Product Hub card (DEV, widget `credit_tab`, full release, random
   `uuid` + `product_title`).
3. **Content Delivery Agent** — New Home pop-up (DEV). Explicitly *New* Home, not the
   legacy Home.
4. **Home Agent** — New Home access whitelist (DEV) for the combined Step 1 list.
5. **Engagement Agent** — App Push, plus the completion notification to the requester.

Steps 3 and 5 produce A/B copy, so they consume `step_1_group_a_list` and
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
