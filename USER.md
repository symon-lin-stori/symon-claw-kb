# USER.md — Who The Agent Serves

## Primary User

- **Name:** Symon Lin
- **Slack mention tag:** not recorded, and not required for dispatch — the requester is
  resolved at runtime from the thread root. Record it only if an out-of-thread
  notification or a cross-check is ever needed.
- **Timezone:** UTC+8
- **Role:** Owner of the dispatch scenarios and of this workspace.

## Requester Resolution

The "requester" is the person a completion notification must reach. It is **not**
necessarily the person whose message you are reading.

- Take the requester from the **`/new` command line at the thread root**.
- The platform may strip that prefix from the text you see, leaving the message sender's
  ID as the only visible candidate. **Do not substitute the sender.**
- If the requester cannot be resolved, stop and ask. Do not dispatch with a guessed tag —
  a hand-off that notifies the wrong person is worse than a delayed one.
- Pass the resolved tag into hand-off templates wherever `{{slack_requester}}` appears,
  so downstream agents can notify the right person directly.

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
| Activity data retention | None. Audience lists, user IDs, and copy are never written to durable files. |

## Decision Log

Durable choices made with the user, newest last. Record the decision, not the payload.

- **Content Delivery Agent owns Home pop-ups.** New Home pop-up configuration is
  dispatched to the Content Delivery Agent, not to the Home Agent or the Engagement
  Agent. Its mention tag lives in the roster in `AGENTS.md`.
- **`placementCode` is not passed.** Pop-up hand-offs do not carry a placement code. The
  target surface is identified in prose instead: "the pop-up on the New Home page (not
  the legacy Home)".
- **Step 1 hand-offs carry the requester tag.** The User Insight Agent is explicitly told
  to append it as plain text at the absolute end, and that this overrides any
  "return the payload only" restriction in its own configuration.
- **Escalation goes to the requester.** No fixed escalation contact is maintained. When
  the retry budget is exhausted, hand the blocked task to whoever opened the thread.
- **Playbook step order stands as written.** In `SC_0_DEPOSIT_AB_CASHBACK_CLIP_PUSH`, the
  New Home pop-up (Step 3) is configured before the New Home access whitelist (Step 4).
  The end state is identical, and this is a demo flow — do not reorder it.
