---
title: PG Schema for Conversations
author: Ivan Willig
date: 2026-11-18
description: A channel-agnostic storage schema for SMS and LLM conversations.
---

# The problem

ADR: `0023. Add conversation storage tables` (common, branch `issue-1220686869650183`)

- SMS commands, replies, and LLM exchanges leave no record
- The L assistant changes and submits ballots with no audit trail
- Every LLM question starts from zero context — no memory
- Future agents (web UI) need the same storage, not an SMS-shaped one

# Overview

<style>
.columns { display: flex; gap: 2rem; align-items: flex-start; }
.columns .column { min-width: 0; }
</style>

:::: {.columns}

::: {.column style="flex: 0 0 38%"}

| Table | Holds |
| --- | --- |
| `conversations` | one row per thread; channel-agnostic anchor |
| `conversation_items` | ordered, typed items: messages, tool calls, tool outputs, commands |
| `sms_messages` | every SMS crossing a line, with delivery status |
| `decision_chat_sessions` | existing; keeps typed SMS state, gains `conversation_id` |

Items store ModelMessage-compatible jsonb, so a stored thread replays
directly as model input. Domain truth stays in the domain tables
(ADR 0009).

:::

::: {.column style="flex: 1 1 62%"}

```plantuml
@startuml
skinparam backgroundColor transparent
hide circle
skinparam linetype ortho
hide stereotype
skinparam class {
  BackgroundColor<<new>> #D7EBEF
  BorderColor<<new>> #387582
  FontColor<<new>> #142426
  AttributeFontColor<<new>> #142426
  BackgroundColor<<changed>> #FDF3DC
  BorderColor<<changed>> #B08A2E
  FontColor<<changed>> #142426
  AttributeFontColor<<changed>> #142426
  BackgroundColor<<existing>> #FAFBFB
  BorderColor<<existing>> #DEE0E1
  FontColor<<existing>> #9AA3AF
  AttributeFontColor<<existing>> #AAB0B1
}

' ── new tables ──────────────────────────────
entity "conversations" as conversations <<new>> {
  * id : uuid <<PK>>
  --
  * profile_id : uuid <<FK>>
  process_instance_id : uuid <<FK>>
  phone_number_id : uuid <<FK>>
  * created_at : timestamptz
  * updated_at : timestamptz
}

entity "conversation_items" as items <<new>> {
  * id : uuid <<PK>>
  --
  * conversation_id : uuid <<FK>>
  run_id : uuid
  agent_id : text
  * kind : text
  role : text
  * content : jsonb
  sms_message_id : uuid <<FK>>
  prompt_version : text
  * created_at : timestamptz
}

entity "sms_messages" as sms <<new>> {
  * id : uuid <<PK>>
  --
  phone_number_id : uuid <<FK>>
  auth_user_id : uuid <<FK>>
  counterpart_e164 : text
  * direction : text
  * body : text
  provider_sid : text <<unique>>
  segments : int
  * status : text
  * created_at : timestamptz
  * updated_at : timestamptz
}

' ── changed tables ──────────────────────────
entity "decision_chat_sessions" as sessions <<changed>> {
  * id : uuid <<PK>>
  --
  conversation_id : uuid <<FK, unique>>
  * process_instance_id : uuid <<FK>>
  * profile_id : uuid <<FK>>
  phone_number_id : uuid <<FK>>
  page_cursor : int
  reopened_at : timestamptz
}

' ── existing tables ─────────────────────────
entity "profiles" as profiles <<existing>> {
  * id : uuid <<PK>>
}
entity "decision_process_instances" as instances <<existing>> {
  * id : uuid <<PK>>
}
entity "decision_phone_numbers" as lines <<existing>> {
  * id : uuid <<PK>>
}
entity "auth.users" as users <<existing>> {
  * id : uuid <<PK>>
}

' ── relationships ───────────────────────────
profiles ||--o{ conversations : participates
instances |o--o{ conversations : optional scope
lines |o--o{ conversations : optional binding

conversations ||--o{ items : holds
conversations |o--o| sessions : keeps SMS state in

lines ||--o{ sms : carries
users |o--o{ sms : resolved sender
sms |o--o{ items : crossed the wire as
@enduml
```

:::

::::

# Items and the transport log

```plantuml
@startuml
skinparam backgroundColor transparent
hide circle
skinparam linetype ortho
hide stereotype
skinparam class {
  BackgroundColor<<new>> #D7EBEF
  BorderColor<<new>> #387582
  FontColor<<new>> #142426
  AttributeFontColor<<new>> #142426
}

entity "conversation_items" as items <<new>> {
  * kind : message | tool_call | tool_output | command
  role : user | assistant (message items only)
  * content : jsonb (ModelMessage shape)
  agent_id : which agent answered
  run_id : groups one exchange
  sms_message_id : set when it crossed the wire
}

entity "sms_messages" as sms <<new>> {
  * direction : inbound | outbound
  * body : text
  * status : received | queued | sent | delivered | failed
  provider_sid : idempotency on redelivery
}

sms |o--o{ items : crossed the wire as
@enduml
```

- One L exchange: a `user` message, the assistant's `tool_call` /
  `tool_output` items, one `assistant` message
- Tool calls never cross the wire, so they carry no `sms_message_id`
- A stranger's text gets an `sms_messages` row and no item

::: handout
The transport log records traffic before identity: strangers, STOP and
HELP keywords, failed deliveries. The item layer is the reusable
conversation API; SMS is one transport feeding it.
:::
