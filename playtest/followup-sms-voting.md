---
title: Playtest Feedback Follow
author: Ivan Willig
date: 2026-11-19
description: Feedback from Monday's SMS voting playtest, and the follow-up work — auto join, an LLM voting assistant, and conversation storage.
---

# What am I sharing

- We did an awesome playtest on Monday
  - Thanks again to Iza for running them.

- Lots of great feedback
- Three important parts
  - Auto join is necessary, remove "JOIN" (Elana) — [task](https://app.asana.com/1/1108348036672214/project/1215095416548883/task/1220618005883957)
  - LLM? What are we doing? (Juan) — [task](https://app.asana.com/1/1108348036672214/project/1215095416548883/task/1220618073647866)
  - Juan and Elana both had dropped messages that needed a repeat attempt. (Reliability) — [task](https://app.asana.com/1/1108348036672214/project/1215095416548883/task/1220719214535615)

# What I worked on this week

- Implemented AUTO JOIN — [task](https://app.asana.com/1/1108348036672214/project/1215095416548883/task/1220618005883957)
  - A new number signs up on first contact, no steps.
  - Demo
- Designed a PostgreSQL schema for conversational interactions — [task](https://app.asana.com/1/1108348036672214/project/1215095416548883/task/1220686869650183)
- Created an LLM agent that helps people vote — [task](https://app.asana.com/1/1108348036672214/project/1215095416548883/task/1220706298124572)
  - Reviews proposals, registers members, builds and casts ballots.
  - Uses glm-5.3-flash to stay quick and focused.
  - Designed to work with the SMS system.
- Created a task to test reliability and scale, not started — [task](https://app.asana.com/1/1108348036672214/project/1215095416548883/task/1220719214535615)

# Demo

- What am I showing?
- A simple TUI LLM agent with a custom system prompt — [task](https://app.asana.com/1/1108348036672214/project/1215095416548883/task/1220709734035100)
- A `common` specific LLM tool chain
- A memory system in postgresql (see [schema](https://oneprojectorg.github.io/technical-presentations/conversations/conversation-storage.html#(1)))

- Important: agent is not "prompt hardned" and subject to prompt injection attacks.
- Lots of stuff we need to figure out from design
  - conversation flow
  - interface.... nature language.

- review built in data model.
- put this in the admin ui.

# The boot script

`services/workflows/scripts/assistant-local.sh` boots the REPL against the local dev database:

```bash
./services/workflows/scripts/assistant-local.sh \
  --email ivan@oneproject.org --decision sms-voting-test
```

```bash
set -euo pipefail

...

exec pnpm w:workflows assistant:cli "$@"
```
