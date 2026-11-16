---
title: SMS Voting
author: Ivan Willig
date: 2026-11-16
description: An overview of each SMS voting keyword command.
---

# SMS voting commands

# Query the demo state

- Shows how far the test phone number got: signup, registration, draft, cast ballot.
- No row means the number is fully reset; a null column means that step has not happened.

```sql
SELECT
  au.id        AS auth_user_id,
  u.profile_id,
  e.enrolled_at,
  d.id         AS draft_id,
  s.id         AS submission_id
FROM auth.users au
LEFT JOIN users u ON u.auth_user_id = au.id
LEFT JOIN decision_voter_enrolments e ON e.profile_id = u.profile_id
LEFT JOIN decisions_vote_drafts d ON d.profile_id = u.profile_id
LEFT JOIN decisions_vote_submissions s ON s.submitted_by_profile_id = u.profile_id
WHERE au.phone = '1<your 10 digits>';
```


# Reset the demo

- Removes the test phone number's account and ballot data so the flow starts over.
- Run against staging before each demo run.

```sql
-- 1. Delete the personal profile.
DELETE FROM profiles
WHERE id = (
  SELECT u.profile_id
  FROM users u
  JOIN auth.users au ON au.id = u.auth_user_id
  WHERE au.phone = '1<your 10 digits>'
);

-- 2. Delete the auth account.
DELETE FROM auth.users
WHERE phone = '1<your 10 digits>';
```

# JOIN

- Does nothing on a known number.
- Text `JOIN`; the reply says you already have an account.
- Unknown numbers go to signup.

# SHOW

- Lists the decisions you belong to.
- Text `SHOW`; the reply gives one line per decision.

# SHOW <slug>

- Describes one decision.
- Text `SHOW` and a decision slug.
- The reply gives the name, phase, voting status, and proposal count.

# VOTE

- Starts voter registration when enrolment is open.
- Text `VOTE`; add a slug on the shared number.
- The system asks for your address.

## "a street address"

- Resolves your address to a district during registration.
- Text your street address.
- Example: `90 W Broad St, Columbus, OH 43215` (City Hall) resolves to District 7.
- The reply names your district and asks you to affirm.

## "AGREE"

- Affirms the attestation and completes registration.
- Text `AGREE` after the district reply.
- The reply confirms your budget.

# PROPOSALS

- Shows the first page of your district ballot.
- Starts a draft with a page cursor.
- Text `PROPOSALS`; add a slug on the shared number.

## MORE

- Shows the next page of proposals.
- Text `MORE` after `PROPOSALS`.

## INFO

- Shows one proposal's title, cost, and summary.
- Text `INFO` and a proposal code.

# LIST

- Shows your draft ballot in rank order.
- Text `LIST`.
- The reply shows each pick, the allocated amount, and the unspent amount.

# DONE

- Shows the full ballot for review.
- Text `DONE`; the reply asks you to text `SUBMIT` to cast it.

# SUBMIT

- Casts your draft ballot and clears the draft.
- Text `SUBMIT` after `DONE`.
- The reply confirms your ballot is in.

# Anything else

- Any other text returns help.
- The reply tells you to text `SHOW`, or `SHOW` with a decision name.
- Reply `STOP` to unsubscribe.
