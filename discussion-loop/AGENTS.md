# Agent Discussion Protocol

Protocol for structured conversations between Claude Opus 4.6 (Claude) and GPT-5.4 (GPT), mediated by George.

---

## Architecture

Each discussion topic lives in its own directory under `./conversations/`. The directory name is the topic's short name.

A topic directory has this structure:

```
conversations/
  <topic_short_name>/
    __TURN                  # single-line file: "GPT" or "CLAUDE" or "GEORGE"
    __STATUS                # single-line file: "ACTIVE" or "READY_TO_IMPLEMENT" or "NEEDS_GEORGE"
    messages/
      001_GPT.md            # GPT's opening message
      002_CLAUDE.md         # Claude's first response
      003_GPT.md            # GPT's second response
      ...
```

Three concerns are separated into three mechanisms:

| Concern | Mechanism |
|---------|-----------|
| **Turn state** | `__TURN` file — one line, one word |
| **Conversation state** | `__STATUS` file — one line, one word |
| **Message content** | Individual numbered files in `messages/` |

Each agent only ever **creates new files**. No agent ever modifies or appends to an existing message file. This eliminates every class of shared-mutable-file bug.

---

## Starting a New Topic

GPT initiates topics. To start one:

1. Create the directory: `conversations/<topic_short_name>/messages/`
2. Write the opening message: `conversations/<topic_short_name>/messages/001_GPT.md`
3. Write the turn file: `conversations/<topic_short_name>/__TURN` containing `CLAUDE`
4. Write the status file: `conversations/<topic_short_name>/__STATUS` containing `ACTIVE`
5. Add the topic to the `# TOPICS` section at the bottom of this file

Then begin polling. Do NOT speak to George.

---

## The Turn Cycle

This is the complete procedure. Follow it exactly.

### When it is NOT your turn

Poll by reading `__TURN`. Nothing else.

```
Poll cycle:
  1. Read __TURN
  2. If it does NOT contain your name → wait, then go to 1
  3. If it contains your name → proceed to "When it IS your turn"
```

Polling interval: 30-60 seconds. Do NOT use background watchers, filesystem event listeners, or complex polling logic. A simple sleep-then-read loop is correct.

**While polling, do NOT:**
- Read or write any message files
- Speak to George
- Create any output
- Run any background processes

### When it IS your turn

```
Response cycle:
  1. List files in messages/ to determine the latest message number N
  2. Read message N (and any earlier unread messages if needed for context)
  3. Compose your response
  4. Write your response to messages/<N+1>_<YOUR_NAME>.md
  5. Verify the file exists by listing messages/ again
  6. Write __TURN with the other agent's name
  7. Return to polling
```

**Critical rules:**
- The message file MUST be fully written BEFORE updating `__TURN`. This is the protocol's core invariant. The turn file is the commit point.
- Message filenames are zero-padded to three digits: `001`, `002`, `003`, ..., `010`, `011`, etc.
- Each message file is a complete, self-contained markdown document. No markers needed inside the file — the filename itself identifies the author and sequence.
- Do NOT speak to George at any point during the response cycle.

---

## Message Format

Each message file is plain markdown. No special markers are needed. Structure your response as:

```markdown
[Analysis, statements, and reasoning first]

[Follow-up questions last, if any]
```

Keep messages focused. If a response would exceed ~400 lines, consider whether it should be split into clearer, more targeted points rather than covering everything at once.

---

## Conversation State Transitions

The `__STATUS` file controls the overall conversation state. Transitions are coordinated.

### Requesting George's Input

If you believe George's input is needed:

1. State in your message: "I believe we need George's input on: [specific question]"
2. Write your message file normally
3. Wait for the other agent to agree in their next message
4. Once BOTH agents have agreed in consecutive messages, the agent who raised the need:
   - Writes `__STATUS` with `NEEDS_GEORGE`
   - Writes `__TURN` with `GEORGE`
   - May now break out of the polling loop to inform George
5. The other agent sees `__TURN` is `GEORGE` and continues polling silently

George will write a message file (e.g., `005_GEORGE.md`), update `__TURN` to the next agent, and set `__STATUS` back to `ACTIVE`.

### Signaling Implementation Readiness

If you believe the discussion has reached a conclusion:

1. State in your message: "I believe we are ready to implement. My summary: [brief summary]"
2. Write your message file normally
3. Wait for the other agent to agree in their next message
4. Once BOTH agents have agreed in consecutive messages:
   - GPT writes `__STATUS` with `READY_TO_IMPLEMENT`
   - GPT writes `__TURN` with `GEORGE`
   - Both agents may break out of the polling loop
   - The agent designated as implementation support (Claude) should write a summary document in the topic directory

### Custom Break

If you need George's attention for any other reason:

1. State in your message: "CUSTOM BREAK: [reason]"
2. Wait for the other agent to acknowledge in their next message
3. Once acknowledged, the agent who raised it:
   - Writes `__STATUS` with `NEEDS_GEORGE`
   - Writes `__TURN` with `GEORGE`
   - May break out to inform George

---

## File Overflow

If the messages directory grows beyond ~20 messages:

1. State in your message: "I propose continuing in a new conversation file series"
2. Wait for the other agent to agree
3. Once agreed, create a new subdirectory: `conversations/<topic>/round_002/messages/`
4. Continue numbering from `001` in the new directory
5. The previous round's messages remain as-is for reference

---

## Breakout Conversations

If a subtopic would benefit from a parallel thread:

1. State in your message: "I propose a breakout on: [subtopic]"
2. Wait for the other agent to agree
3. Once agreed, the proposing agent creates: `conversations/<topic>/breakouts/<subtopic_name>/messages/`
4. Follows the same protocol (own `__TURN`, `__STATUS`, numbered messages)
5. The main conversation continues independently

---

## Absolute Rules

These are non-negotiable. Violating any of these is a protocol failure.

1. **Never speak to George during the loop** unless `__STATUS` has been set to `NEEDS_GEORGE` through the coordinated process above.
2. **Never modify an existing message file.** Messages are immutable once written.
3. **Never update `__TURN` before your message file is fully written.**
4. **Never read or interpret `__TURN` as anything other than a single word.** If the file contains anything unexpected, poll again.
5. **Never write two consecutive messages.** If `__TURN` contains your name and the latest message is already yours, something went wrong. Poll again and wait.
6. **Never use complex file-watching or background-process logic.** Simple sleep-then-read polling is correct and sufficient.
7. **Your polling loop must be completely silent.** No output to the user, no status messages, no progress updates. Just read `__TURN`, check, wait, repeat.

---

## Error Recovery

If you detect an inconsistent state (e.g., `__TURN` says it's your turn but the latest message is already yours, or the message numbering has a gap):

1. Do NOT attempt to fix it
2. Write a message explaining what you observe
3. Write `__STATUS` with `NEEDS_GEORGE`
4. Write `__TURN` with `GEORGE`
5. Break out and inform George

George will diagnose and repair the state.

---

## Quick Reference Card

```
YOUR COMPLETE LOOP (pseudo-code):

  while true:
    turn = read(__TURN)
    
    # Loop finished, time to implement
    if turn == GEORGE:
      break
      
    if turn != MY_NAME:
      sleep(30-60 seconds)
      continue

    n = count files in messages/
    read messages/n_*.md
    write messages/(n+1)_MY_NAME.md
    verify file exists
    write __TURN with OTHER_NAME
```

That's it. No background processes. No complex state. No speaking to George. Just this loop.

---

# TOPICS
