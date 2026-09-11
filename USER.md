# USER.md — Who The Agent Serves

## Primary User

- **Name:** Symon Lin
- **Slack mention tag:** not recorded, and not required for dispatch — the requester is
  resolved at runtime from provenance. Record it only if an out-of-thread notification or
  a cross-check is ever needed.
- **Timezone:** UTC+8
- **Role:** Owner of the dispatch scenarios and of this workspace.

## Requester Resolution

The "requester" is the human a completion notification must reach.

- Take the requester from the **provenance `senderId` whose `senderType` is `user`**.
- **Do not use the mention tag on the `/new` command line at the thread root.** That tag
  addresses the *application* the command was sent to, not a person. Using it produces a
  hand-off whose "Requested by" field resolves to a bot and renders empty. Confirmed in a
  live thread: the root-line tag resolved to the dispatcher app itself (SCOperationClaw).
- `senderType` is the discriminator, not position in the thread. Ignore any `senderId`
  whose `senderType` is not `user`.
- If no `user`-type sender can be resolved, stop and ask. Do not dispatch with a guessed
  tag — notifying the wrong human is worse than notifying late.
- Pass the resolved tag into hand-off templates wherever `{{slack_requester}}` appears.
  In the current playbook that is the **final step only**; see Notification Timing below.

## Notification Timing

While a campaign is **in flight**, the mention belongs to the agent that continues the
work, not to the person who asked for it.

- Mid-flight Channel-2 hand-offs carry the assignee's tag and nothing else. Never embed a
  human tag in them, and never ask a downstream agent to ping a person on your behalf.
- Mid-flight Channel-1 dashboards stay mention-free, as always.
- The requester is notified **once, on completion** — via `{{slack_requester}}` in the
  final step of the playbook.

The reason for the restriction: an embedded tag propagates. Downstream agents extract the
tag from the hand-off they receive and re-emit it, so one wrong tag mid-flight gets
echoed onward by every agent after it.

## Language Preferences

- **All user-facing output is English.** This includes the progress dashboard, summaries,
  questions, and error explanations.
- Playbooks, skills, and knowledge-base source material may be authored in Chinese. That
  does **not** make Chinese the output language. Do not copy the source language of a
  template into a message addressed to the user.
- **Hand-off instructions keep the scenario template's wording**, unchanged. Those are
  machine-readable task specs for other agents, not messages to a person.
- Campaign copy language is a business parameter, not a preference — currently Spanish
  for the A/B scenarios.

## Defaults & Standing Decisions

| Topic | Decision |
| --- | --- |
| Target environment | **DEV** for all scenario steps. Any other environment requires an explicit instruction. |
| Dashboard verbosity | Full dashboard on every state transition. Do not shorten it or move it to a file. |
| Retry budget | 3 follow-up rounds per step, then hand over to a human. |
| Escalation target | The **resolved requester** for the current thread. There is no standing escalation roster — whoever opened the thread receives the blocked task. |
| Duplicate campaigns | Never a reason to pause. Report the overlap on the dashboard and continue to the next step in the same turn. |
| Activity data retention | None. Audience lists, user IDs, and copy are never written to durable files. |

## Decision Log

Durable choices made with the user, newest last. Record the decision, not the payload.

- **Content Delivery Agent owns Home pop-ups.** New Home pop-up configuration is
  dispatched to the Content Delivery Agent, not to the Home Agent or the Engagement
  Agent. Its mention tag lives in the roster in `AGENTS.md`.
- **`placementCode` is not passed.** Pop-up hand-offs do not carry a placement code. The
  target surface is identified in prose instead: "the pop-up on the New Home page (not
  the legacy Home)".
- **Only the final step carries the requester tag.** Superseded an earlier decision that
  put `{{slack_requester}}` in Step 1. Mid-flight hand-offs are agent-to-agent only; the
  requester is notified once, when the campaign completes.
- **The whitelist business purpose is fixed in the template.** Step 4 carries a standing
  purpose line rather than asking per campaign. Revisit it if the scenario's intent
  changes, since it lands in the audit record.
- **A duplicate run is not a blocker.** Regardless of what memory, an earlier thread, or a
  previous delivery says about the same audience, run every step of a matched scenario to
  completion. Mention the overlap, never gate on it.
- **The requester is the `user`-type provenance sender.** Superseded an earlier rule that
  read the tag off the `/new` root line — that tag belongs to the application, not a
  person, and resolved to a bot in a live run.
- **Escalation goes to the requester.** No fixed escalation contact is maintained. When
  the retry budget is exhausted, hand the blocked task to whoever opened the thread.
- **Playbook step order stands as written.** In `SC_0_DEPOSIT_AB_CASHBACK_CLIP_PUSH`, the
  New Home pop-up (Step 3) is configured before the New Home access whitelist (Step 4).
  The end state is identical, and this is a demo flow — do not reorder it.
