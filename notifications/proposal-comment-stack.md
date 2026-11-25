---
title: One Event End to End
author: Ivan Willig
date: 2026-11-24
description: The proposalComment prototype stack on the ADR 0023 storage model.
subtitle: '[This deck](https://oneprojectorg.github.io/technical-presentations/notifications/proposal-comment-stack.html) · schema [#2706](https://github.com/oneprojectorg/common/pull/2706) · stack [#2707](https://github.com/oneprojectorg/common/pull/2707) / [#2708](https://github.com/oneprojectorg/common/pull/2708) / [#2709](https://github.com/oneprojectorg/common/pull/2709)'
---

# The demo

::: columns
::: column

![](notifications/images/01-bell-dropdown.png){width=100%}

:::
::: column

The unread badge:

```sql
select count(*)
from notifications
where user_id = $1
  and read_at is null;
```

The dropdown rows:

```sql
select a.snapshot, n.read_at,
       n.created_at
from notifications n
join activities a
  on a.id = n.activity_id
where n.user_id = $1
order by n.created_at desc,
         n.id desc
limit 7;
```

:::
:::

# We need to query notifications

::: columns
::: column

![](notifications/images/02-all-notifications.png){width=100%}

:::
::: column

The All tab, one page per query:

```sql
select n.id, n.category,
       n.seen_at, n.read_at,
       a.event_name, a.snapshot,
       n.created_at
from notifications n
join activities a
  on a.id = n.activity_id
where n.user_id = $1
  and (n.created_at, n.id)
      < ($2, $3)
order by n.created_at desc,
         n.id desc
limit 20;
```

The day headers are rendering; the
order is the index's.

:::
:::

# Some notifications have per-user state

::: columns
::: column

![](notifications/images/03-needs-action.png){width=100%}

:::
::: column

The Needs-action tab:

```sql
select n.id, a.snapshot,
       n.created_at
from notifications n
join activities a
  on a.id = n.activity_id
where n.user_id = $1
  and n.action_required
  and n.read_at is null
order by n.created_at desc,
         n.id desc;
```

The tab and its count both ride
the needs-action partial index.

:::
:::

# PG Schema

[https://oneprojectorg.github.io/technical-presentations/notifications/notifications-storage.html#(3)](https://oneprojectorg.github.io/technical-presentations/notifications/notifications-storage.html#(3))

# Reworked one event in this new architecture {.dark}

`proposalComment/posted`, end to end on the ADR 0023 model.

The stack: [#2706](https://github.com/oneprojectorg/common/pull/2706) → [#2707](https://github.com/oneprojectorg/common/pull/2707) → [#2708](https://github.com/oneprojectorg/common/pull/2708) → [#2709](https://github.com/oneprojectorg/common/pull/2709)

# The pipeline

```plantuml
@startuml
left to right direction
rectangle "createPost" as A
database "event_outbox" as B
rectangle "outboxRelay (cron)" as C
rectangle "Inngest" as D
rectangle "handleProposalCommentPosted" as E
database "activities" as F
database "notifications" as G
database "notification_deliveries" as H
rectangle "notifications.list / unreadCount" as I
A --> B : same tx
B --> C : sweep unsent
C --> D : event id = row id
D --> E
E --> F
E --> G
E --> H
G --> I
@enduml
```

Ships as [#2707](https://github.com/oneprojectorg/common/pull/2707) (outbox write + relay), [#2708](https://github.com/oneprojectorg/common/pull/2708) (consumer), and [#2709](https://github.com/oneprojectorg/common/pull/2709) (read API).

# Emitting the event

::: columns
::: column

Before: sent after the commit, inside `waitUntil` — a rollback can
still send, and a crash drops the event.

```ts
// after the commit, in waitUntil
await event.send({
  name: 'proposalComment/posted',
  data: {
    postId,
    proposalId,
    authorProfileId,
  },
});
```

:::
::: column

After: the outbox row commits or rolls back with the post. The relay
does the send.

```ts
// inside db.transaction
await writeOutboxEvent({
  eventName: 'proposalComment/posted',
  payload: {
    postId,
    proposalId,
    authorProfileId,
  },
  db: tx,
});
```

:::
:::

The change in [#2707](https://github.com/oneprojectorg/common/pull/2707): [`createPost.ts` diff](https://github.com/oneprojectorg/common/pull/2707/changes#diff-15b5a5b5fb786c9bb9a50672a1a201fd28015b6f4b3a2c352b5a8d69b5cea19f).

# Delivering the event

::: columns
::: column

The [relay](https://github.com/oneprojectorg/common/pull/2707/changes#diff-2f52b20991097f86238ff70adf143414d5347119aaa0e5d39e217de9efc6113a)
sweeps unsent rows, oldest first. The `sent_at IS NULL`
predicate has a partial index, so the sweep stays fast at any
table size.

```ts
const unsent =
  isNull(eventOutbox.sentAt);
const rows = await db
  .select()
  .from(eventOutbox)
  .where(unsent)
  .orderBy(asc(createdAt), asc(id))
  .limit(100);
```

:::
::: column

Success stamps `sent_at`. Failure only bumps `attempts`; the next
sweep retries.

```ts
for (const row of rows) {
  try {
    // event id = row.id
    await send(buildOutboxEvent(row));
    // set sent_at, attempts + 1
  } catch (error) {
    // attempts + 1, row stays unsent
  }
}
```

A crash between send and stamp re-sends — harmless: the event id is
the row id, and every consumer write dedupes.

:::
:::

The sweep in [#2707](https://github.com/oneprojectorg/common/pull/2707): [`relayOutboxEvents.ts` diff](https://github.com/oneprojectorg/common/pull/2707/changes#diff-2f52b20991097f86238ff70adf143414d5347119aaa0e5d39e217de9efc6113aR19).

# Reading it back

::: columns
::: column

```ts
list: networkAuthenticatedProcedure()
  .input(z.object({ limit, cursor }))
  .output(
    paginated(notificationEncoder))
  .query(({ ctx, input }) =>
    listNotifications({
      authUserId: ctx.user.id,
      ...input,
    })),
```

:::
::: column

The new `notifications` router: `list`, `unreadCount`, `markRead`,
`markAllRead`.

- Thin router: the logic lives in `@op/common`
- Every read and write is scoped `user_id = caller` on the server
- Cursor-paged `{ items, next }`, in the list index's order

:::
:::

The router in [#2709](https://github.com/oneprojectorg/common/pull/2709): [`routers/notifications` diff](https://github.com/oneprojectorg/common/pull/2709/changes#diff-c95b8ecb6bc98d5cbada5e4dc1c8a1233d43f33f43a483d4ea3b0082cd8ac248R13).

# It runs end to end

::: columns
::: column

![](notifications/images/live-01-comment.png){width=100%}

The comment goes in; the outbox row commits with it. One relay sweep
later the consumer has written the activity and the notification.

:::
::: column

![](notifications/images/live-02-bell.png){width=100%}

The badge and the row read straight from `notifications` and the
activity snapshot. Opening the row stamps `read_at` and `seen_at`.

:::
:::

Live on the local stack; the bell ships in
[#2713](https://github.com/oneprojectorg/common/pull/2713), behind the
`in_app_notifications` feature flag.

# How does this scale?

Valentino's four cases:

- Alice merged proposal A into proposal B
- Bob requested a revision on his review of proposal A
- Alice replied by updating proposal A
- Alice submitted custom form X

# One decision per field: does a feed query by it?

::: columns
::: column

The shipped write, `recordProposalCommentActivity.ts` in [#2708](https://github.com/oneprojectorg/common/pull/2708):

```ts
await db
  .insert(activities)
  .values({
    outboxId,
    eventName: Events
      .proposalCommentPosted.name,
    // feeds query by these four:
    // each is a real FK column
    actorProfileId: authorProfileId,
    processInstanceId:
      proposal.processInstanceId,
    proposalId,
    postId,
    // everything render-only:
    snapshot,
  })
```

:::
::: column

The snapshot, same file — the fields no feed queries by, copied at
write time:

```ts
const snapshot = {
  commenterName:
    context.commenterName,
  proposalTitle,
  processName: context.processName,
  decisionSlug: context.decisionSlug,
  // an id, but nothing queries
  // activities by it: no FK
  proposalProfileId:
    context.proposalProfileId,
  commentExcerpt:
    context.commentContent
      .slice(0, EXCERPT_LENGTH),
};
```

:::
:::

What those columns buy: every activity feed is the same one-predicate,
index-ordered read, swapping one column (no feed read ships yet):

```sql
select snapshot, created_at
from activities
where proposal_id = $1            -- proposal page
--    organization_id = $1        -- org feed
--    process_instance_id = $1    -- process feed
--    post_id = $1                -- comment thread
--    actor_profile_id = $1       -- everything Alice did
order by created_at desc, id desc
limit 20;
```

The query access cases drive the schema — a new feed is what earns a
new FK column. The snapshot is the record of what the objects looked
like at that point in time: rendering never touches the domain
tables, and the row still reads after the proposal is renamed or
deleted — every FK sets null, and the snapshot carries the render.

# The four cases fit the same columns

::: columns
::: column

Three of the verbs already exist — [`services/events/src/types.ts`](https://github.com/oneprojectorg/common/blob/dev/services/events/src/types.ts):

```ts
proposalMerged: {
  name: 'proposal/merged',
  schema: z.object({
    relationshipId:
      z.string().uuid(),
    actorAuthUserId:
      z.string().uuid(),
  }),
},
reviewRevisionRequested: {
  name: 'review/revision-requested',
  schema: z.object({
    assignmentId: z.string().uuid(),
    revisionRequestId:
      z.string().uuid(),
  }),
},
reviewProposalRevisionSubmitted: {
  name: 'review/' +
    'proposal-revision-submitted',
  schema: z.object({
    proposalId: z.string().uuid(),
    proposalHistoryId:
      z.string().uuid(),
    revisionRequestIds: z.array(
      z.string().uuid()).min(1),
  }),
},
```

:::
::: column

```plantuml
@startuml
left to right direction
rectangle "org feed" as OF
rectangle "process feed" as PF
rectangle "proposal page" as PR
rectangle "comment thread" as PO
database "activities" as A
rectangle "organizations" as O
rectangle "process_instances" as PI
rectangle "proposals" as P
rectangle "posts" as PS
OF --> A : organization_id = $1
PF --> A : process_instance_id = $1
PR --> A : proposal_id = $1
PO --> A : post_id = $1
A --> O : FK, set null
A --> PI : FK, set null
A --> P : FK, set null
A --> PS : FK, set null
@enduml
```

Each query-by column is a direct FK to its table.

Where each case lands when its consumer moves onto the model, like the comment one did:

| Case | Scopes | In `snapshot` |
| --- | --- | --- |
| merged A into B | `proposal_id` | relationship, titles |
| revision requested | `proposal_id` | assignment, request |
| revision submitted | `proposal_id` | history, request ids |
| form X submitted | `organization_id` | no event yet |

:::
:::

No case adds a column: nothing queries activities by review, merge edge, or form. An object that needs that query gets a direct FK column and its index.

# The actor / verb / object option

::: columns
::: column

Activity streams as columns — an option, not what we built:

```sql
actor_id    uuid,
verb        text,
object_type text,
object_id   uuid,  -- no FK
target_type text,
target_id   uuid   -- no FK
```

:::
::: column

What shipped in [#2706](https://github.com/oneprojectorg/common/pull/2706), `activities.sql.ts`:

```ts
eventName: varchar(),  // verb
actorProfileId: uuid() // actor
  .references(() => profiles.id,
    { onDelete: 'set null' }),
proposalId: uuid()     // object
  .references(() => proposals.id,
    { onDelete: 'set null' }),
snapshot: jsonb().notNull(),
```

:::
:::

The stream vocabulary is in the shipped table: `event_name` is the verb, `actor_profile_id` the actor, the scope ids and `snapshot` the object. The typed columns say what a `(type, id)` pair cannot: a deleted object sets its column null and the row keeps rendering from the snapshot; a bare `(type, id)` just dangles.

# The trade: pay at write time, read simply

No feed read ships yet; these are the shapes the two schemas force.

::: columns
::: column

Proposal A's feed on the shipped columns — one predicate, one index:

```sql
select a.snapshot, a.created_at
from activities a
where a.proposal_id = $1
order by a.created_at desc,
         a.id desc
limit 20;
```

When a read needs the live object, the join is typed:

```sql
left join proposals p
  on p.id = a.proposal_id
-- deleted: proposal_id is null,
-- the snapshot still renders
```

:::
::: column

The same read on `(type, id)` — the option:

```sql
select a.snapshot, a.created_at
from activities a
where a.object_type = 'proposal'
  and a.object_id = $1
-- + a partial index per type
order by a.created_at desc,
         a.id desc
limit 20;
```

```sql
-- one join per type, every read
left join proposals p
  on a.object_type = 'proposal'
 and p.id = a.object_id
left join posts po
  on a.object_type = 'post'
 and po.id = a.object_id
-- a deleted object leaves a
-- dangling id: dead and live
-- look the same until you join
```

:::
:::

The shipped shape pays once, at write time: a column, an FK, an index, a migration per queryable object, and the consumer fills the scopes. The `(type, id)` shape inverts the trade: free writes, and every read carries the type dispatch, the per-type joins, and the orphan filtering.


# Appendix

# The four tables

| Table | Holds |
| --- | --- |
| `event_outbox` | the event, written in the domain transaction |
| `activities` | one feed row per event, scoped by ids |
| `notifications` | one row per recipient, with read state |
| `notification_deliveries` | one row per channel attempt |

Schema PR: [#2706](https://github.com/oneprojectorg/common/pull/2706) — merges on its own, ahead of the stack.

[https://oneprojectorg.github.io/technical-presentations/notifications/notifications-storage.html#(3)](https://oneprojectorg.github.io/technical-presentations/notifications/notifications-storage.html#(3))

# Idempotency, link by link

- The relay sends with the outbox row id as the Inngest event id
- `activities.outbox_id` is unique: one activity per event
- `(activity_id, user_id)` is unique: one row per recipient
- Email sends skip notifications that already hold a delivery row

A replayed event changes nothing. Delivery is at least once; storage
dedupes every step.


# The pull requests

| PR | What it ships |
| --- | --- |
| [#2706](https://github.com/oneprojectorg/common/pull/2706) | the four tables (stands alone) |
| [#2707](https://github.com/oneprojectorg/common/pull/2707) | outbox write in `createPost` + relay cron |
| [#2708](https://github.com/oneprojectorg/common/pull/2708) | activity, fan-out, deliveries |
| [#2709](https://github.com/oneprojectorg/common/pull/2709) | `notifications.list`, `unreadCount`, `markRead` |
| [#2713](https://github.com/oneprojectorg/common/pull/2713) | the header bell: badge, list, mark read (`in_app_notifications` flag) |

ADR 0023 is merged ([#2532](https://github.com/oneprojectorg/common/pull/2532)).

# What to do next

- Review [#2706](https://github.com/oneprojectorg/common/pull/2706) — it merges on its own
- Then the stack, bottom up: [#2707](https://github.com/oneprojectorg/common/pull/2707) → [#2708](https://github.com/oneprojectorg/common/pull/2708) → [#2709](https://github.com/oneprojectorg/common/pull/2709)
- The MVP bell UI is [#2713](https://github.com/oneprojectorg/common/pull/2713); the full notifications page follows
