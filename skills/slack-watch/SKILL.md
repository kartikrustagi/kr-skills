---
name: slack-watch
description: Watch a Slack channel for new messages and act on them as natural-language instructions, replying in-thread with the result. Supports multiple agent sessions sharing one channel via aliases.
---

# Slack Watch

Turn a Slack channel into a remote control for this agent: anyone who can post in the
channel can give it a natural-language instruction, exactly as if they'd typed it
directly into this session, and it replies in-thread with the result.

Several agent sessions can watch the same channel at once. Each session may be given an
**alias**; a message addressed to an alias is picked up only by the session running with
that alias.

**This has no access restriction.** Any message from anyone who can post in the
configured channel gets executed with this session's full tool access. Only point this
at a channel where that's genuinely intended.

## Usage

This skill does the per-check work; the recurring schedule comes from wrapping it in
`/loop`:

```
/loop 5m /slack-watch <channel> [alias]
```

`<channel>` is required every time — there is no default channel. `5m` is the
recommended default interval; replace it with any interval `/loop` accepts, or omit it
to let `/loop` self-pace.

`[alias]` is optional. Aliases are case-insensitive and may contain only `a-z`, `0-9`,
`_` and `-`. A session started without an alias is **unaliased**: it handles every
thread that isn't addressed to an alias (today's single-agent behaviour).

### Addressing an agent

Start the **first message of a thread** with `alias:`, e.g. `build: run the tests`. You
don't repeat the alias on follow-up messages in that thread — the thread stays with that
agent. See step 3 for the exact rules.

## 1. Resolve the channel and register the alias

`<channel>` may be a channel ID, or a name like `#team-updates` or `team-updates`. If it
doesn't already look like a Slack channel ID, resolve it with `slack_search_channels`
before doing anything else.

**Aliased sessions only:** make sure the alias is registered. Search the channel for an
existing announcement of this alias (see step 2 for the search). If none exists, post
one as a top-level message:

```
🤖 _slack-watch:_ alias "<alias>" is online
```

If one already exists, don't post another — just carry on watching (this is how a
restarted session takes over its alias, and how later `/loop` ticks skip re-announcing).
This announcement is the one exception to the "stay silent" rule in step 7.

Announcements never expire. To retire an alias, delete its announcement message in
Slack. Two live sessions with the same alias is user error — both would respond.

## 2. Learn which aliases exist

Every session (aliased or not) needs the set of registered aliases, to tell
`build: ...` (addressed to an agent) from `Note: ...` (plain text). Find them with
`slack_search_public_and_private`, querying for the announcement text scoped to the
channel, e.g. `"slack-watch" "is online" in:#<channel>`. Search is used instead of
`slack_read_channel` so announcements are found however far back they are. Keep only
results whose text, from the very start of the message, is exactly
`🤖 _slack-watch:_ alias "<alias>" is online`, and read the alias out of the quotes
(lowercased).

## 3. Identify the marker, addressing and thread ownership

Every automated reply this skill posts is prefixed with a marker:

- Aliased session: `🤖 _slack-watch (<alias>):_`
- Unaliased session: `🤖 _slack-watch:_`

This is the only way to tell the agent's own past messages apart from the messages you
post yourself — **the Slack integration sends as your own personal account, not a
distinct bot identity**, so this marker is load-bearing, not cosmetic. Never post a
message without it, and never let a message that merely quotes or discusses the marker
be mistaken for one that starts with it (only the literal prefix at the very start of
the message counts). Announcements (step 1) also start with the unaliased marker; they
are never instructions and are never treated as unhandled input.

**Addressing.** A human message is *addressed* to an alias when it starts, at the very
first character, with `<alias>:` and `<alias>` (case-insensitive) is a registered alias
from step 2. A leading `word:` that is not a registered alias is ordinary text.

**Owner of a thread** (parent + replies, or a lone message):

- The owner is the alias of the **most recent human message in the thread that is
  addressed**. Follow-ups with no prefix stay with that owner. An addressed message
  mid-thread (e.g. `deploy: now ship it` in a thread `build` owns) hands the thread off
  to that alias from then on; the new owner sees the whole thread as context.
- If no human message in the thread is addressed, the thread has no alias owner and
  belongs to **unaliased sessions only**; aliased sessions ignore it.
- There is no way to hand a thread back to an unaliased session mid-thread; start a new
  thread instead.

A session only ever handles threads it owns: an aliased session owns threads whose owner
is its alias; an unaliased session owns threads with no alias owner.

## 4. Find unhandled threads

Fetch recent messages: `slack_read_channel(channel_id, limit=100)`. For each top-level
message:

- If it has thread replies, fetch the full thread with `slack_read_thread`.
- If it has none, treat the message itself as a thread of one.

Skip threads this session doesn't own (step 3). For each remaining thread:

- Find the timestamp of the latest automated message in the thread — the most recent
  message starting with a `🤖 _slack-watch` marker, from **any** session (aliased or
  not) — if any.
- Find the timestamp of the latest human message in the thread.
- **Unhandled** means: no automated reply exists yet, or the latest human message is
  newer than the latest automated reply. A thread already replied in is only unhandled
  again once a newer human message arrives after that reply — this is what lets a
  confirmation question and its answer work as a normal back-and-forth (see step 6),
  and what makes a handoff to another alias count as new input.
- If unhandled, collect every human message in the thread posted after the latest
  automated reply (or all of them, if there isn't one yet) as this thread's batch.

Different threads (including different standalone top-level messages) are separate
batches, handled independently. Process unhandled threads **in order of their oldest
unhandled message**, oldest first.

## 5. Execute each batch

For each unhandled thread's batch, in order: strip a leading `<alias>:` addressing
prefix from each message in the batch, treat the combined remaining text as one
natural-language instruction, and carry it out inline in this session, using your normal
tools and judgement, exactly as you would if the same text had been typed directly into
this session.

Reply with the result as a threaded reply (`thread_ts` = the thread's parent message
timestamp), prefixed with this session's marker (step 3). If the reply would exceed
Slack's ~5000-character limit per message, split it into multiple sequential threaded
replies instead of truncating, each one still prefixed with the marker.

## 6. Risky or destructive actions

If the instruction would normally make you pause and confirm before proceeding (same
judgement as interactive use — e.g. a destructive or hard-to-reverse action), don't
skip that here just because the request arrived over Slack. Reply in-thread (marker
included) asking the clarifying/confirmation question, and stop for this batch. Because
step 4's "unhandled" check compares against the *latest* automated reply in the thread,
whatever comes next in that thread — even a short "yes" with no alias — will be picked up
as a new unhandled batch on a later check by the same owner and treated as the answer,
continuing the work from there.

## 7. Nothing unhandled

If no thread has anything unhandled, do nothing — don't post a status update, don't
re-announce that you're watching. Silence is the expected steady state between
instructions. The only unprompted post this skill ever makes is the one-time alias
announcement in step 1.
