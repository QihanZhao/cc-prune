# Cutting turns

When most of what is left is conversation, clearing buckets stops paying. `cut` removes a
stretch of whole turns from the middle and keeps the start and the recent end verbatim. A
turn is one prompt and everything up to the next prompt.

```bash
python3 cc_prune.py cut "$F" --list                          # numbered turns, sizes, prompts
python3 cc_prune.py cut "$F" --keep-head 5 --drop 0.5        # preview
python3 cc_prune.py cut "$F" --keep-head 5 --drop 0.5 --yes  # apply
python3 cc_prune.py cut "$F" --from 6 --to 120 --yes         # exact turns, inclusive
```

- `--keep-head N` (default 5) keeps the first N turns. `--drop F` then cuts that fraction of
  the rest, by size, starting right after the head. The first and the last turn always stay.
- The first kept turn is re-linked onto the head. By default the cut turns stay in the
  file as an unreachable branch; `--purge` deletes them. The snapshot holds them either way.
- Both ends of a cut land where no tool call is waiting for its result. `--list` marks
  those turns `ok`.
- Attachments that carry harness state (system prompt and tool schemas, deferred tool,
  MCP, agent and skill listings) are copied over from the cut stretch, so the tail keeps the
  tools it had.
- A tool that ToolSearch loaded inside the cut stretch still works afterwards if the kept
  turns call it: Claude Code rebuilds the loaded set from the history it resumes.
- The first kept prompt gets a note, `[cc-prune: N earlier turns were cut from the
  conversation here]`, so the model knows why it cannot see them. `--no-note` skips it.
  Repeated cuts keep one note with the running total: a note inside the cut stretch is
  added to the new count, and a note already on the first kept prompt is merged.
- Recorded usage is reset to an estimate, as with `clear` ([how the estimate works](estimate.md)).
  The preview shows it before anything is written.

The model loses everything in the cut turns. Kept turns that refer to cut ones stay as
they are, so the model may notice it is missing something they depend on. Thinking blocks
after the cut no longer match their signatures, so the API drops them, the same as after
any prune; running `clear --drop-all-thinking` after a cut avoids that.

Tested on Claude Code 2.1.296, resuming with Sonnet 5.5 (thinking) and Haiku: synthetic
sessions and copies of real interactive sessions, both modes, a second cut on a cut file,
parallel tool results on either side of the cut and inside it, and a ToolSearch-loaded
tool called after its loading turn was cut. Every resume worked; the model recalled the
kept turns and none of the cut ones, and the next turn was appended to the kept tail.
Without the note the model concluded the cut turns had never happened, so leave it on.
