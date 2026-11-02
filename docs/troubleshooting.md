# Troubleshooting

**`Context limit reached` or `Prompt is too long` right after resuming a pruned session.**
Claude Code refuses the message in a few milliseconds, before the API counts anything. The
recorded `usage` on the last requests still holds the pre-prune size (for example 957k),
and `clear` does not change the conversation's recorded usage by itself. Fix:

```bash
python3 cc_prune.py reset-usage "$F" --tokens 335000    # a figure below your window
```

Pick a number comfortably under your context window. The next real reply corrects it. A
`clear` run with the default settings already does this. Use `reset-usage` on files that
were pruned with `--no-reset-usage` or by an older version. `--autocompact` is not a fix:
it sets the compaction window, not this check.

**`tools.0.custom.input_schema: JSON schema is invalid` after resuming.**
`--attachments` replaced strings inside the `prompt_snapshot` or `deferred_tools_record`
attachments, which hold the system prompt and tool schemas, including each schema's
`$schema` URL. Current versions skip those two kinds. A file already damaged cannot be
repaired from itself: restore a snapshot taken before the clear, or the pristine
original, and clear again.

Clearing `--attachments` is otherwise untested for the other attachment kinds. If a
resume fails, leave `--attachments` out first.

**The transcript keeps growing fast after a prune (tens of KB per assistant turn), and
assistant records carry `input_transformations` entries.**
Each entry is `{"type": "thinking_dropped", "reason": "prefix_binding_mismatch", "path":
"messages.N.content.0"}`. This is the API's own report, and Claude Code stores it. A thinking
block's signature binds the system prompt, the tools, every earlier message and the model.
Clearing earlier tool results edits that history, so on models that enforce the check
(observed here: Opus 5.5 and Sonnet 5.5; Haiku 4.5 does not run it) the API drops the first
mismatched block and every thinking block after it, on every request, unbilled. The
reasoning is lost from the prune onward and the report is rewritten each turn, growing the
file by about 107 bytes per dropped block per turn. Measured: 385 entries and about 43 KB
per turn after one prune of a large session, 698 entries and about 77 KB per turn after a
second.

Two experimental flags, both opt-in:

```bash
python3 cc_prune.py clear "$F" --transformations         # delete the saved reports (shrinks history once)
python3 cc_prune.py clear "$F" --drop-failing-thinking   # delete the blocks the API reported dropping
```

`--transformations` alone does not stop the growth: the field is regenerated every turn
while blocks keep failing. `--drop-failing-thinking` removes the blocks the API reported
dropping and leaves their records in place with empty content. On a small test session the
entries stopped, the context count was unchanged, and new thinking blocks written afterwards
were accepted. The next prune invalidates the blocks after it again.

Safeguards on `--drop-failing-thinking`:

- **Preview by default.** Without `--yes` it prints what it would delete (counts, record
  range, models, how many blocks stay) and writes nothing, not even a snapshot. Every other
  flag in the same command is held back too.
- **It refuses layouts it has not been checked against.** The failing blocks are the ones
  that existed when the history was last edited (found from where the latest run of equal
  reports starts). It only acts if those are every earlier block (the oldest is removed too,
  so the removal is from the front) or the newest blocks in the file. A report with valid
  blocks on both sides is refused, and so is a report that lists more blocks than exist.
- **Exact validation.** The write is rejected unless precisely the planned blocks are gone
  and everything else, including the uuid chain, is unchanged.
- Blocks written after the last edit are never listed or deleted.

`--keep-tail` is respected by `--transformations` and ignored by the two thinking flags.

**`--drop-all-thinking` (experimental).** `--drop-failing-thinking` can only remove blocks
the API has already reported, so blocks written since the last prune are missed: the next
prune invalidates them and the reports come back, smaller. `--drop-all-thinking` deletes every
thinking block in the file, so nothing is left for the API to drop. It has the same
safeguards (preview unless `--yes`, live-session guard, exact validation, records kept with
empty content) and can be combined with a prune in one pass:

```bash
python3 cc_prune.py clear "$F" --results 1 --inputs 1 --images --drop-all-thinking --yes
# only if earlier resumes already saved reports: wait out the live-edit window
# (30 s by default), then remove them:
python3 cc_prune.py clear "$F" --transformations --keep-tail 0
```

What it costs: the reasoning is gone from the transcript, although with the default
`display: "omitted"` the thinking text is blank anyway and the API was already dropping the
blocks. Run it before your first message after a prune and the cache rewrite that message
causes is the only cost. Tested on throwaway copies and on two real sessions (about 7,000 and
3,900 records); on both, every record written after the resume reported nothing dropped. The
two thinking flags have not been run on many real sessions.

**Running sessions.** `clear`, `strip-commands`, `reset-usage` and `restore` refuse to edit a
transcript under the Claude config directory (`$CLAUDE_CONFIG_DIR`, default `~/.claude`)
that looks live, unless you pass `--force`. Two independent signals are used:

- **The process registry.** `<config>/sessions/<pid>.json` maps a running process to its
  session id. A transcript counts as live if its file name, or the session folder above it
  (subagent logs), matches a registered process that is still running. On Linux the process
  start time must match too, so a reused pid is not mistaken for the session. Without `/proc`
  (macOS) only existence is checked, and on Windows nothing can be, so a registered session
  is assumed running.
- **Recent edits.** A transcript modified within the last `LIVE_IF_EDITED_WITHIN` seconds
  (a setting near the top of `cc_prune.py`, default 30, `0` turns it off) is treated as live.
  This catches a stale registry. The tool's own writes count too, so a second edit of the
  same transcript within that window needs a short wait or `--force`.

Copies outside the config directory are never blocked. The registry is an undocumented
format, so the first signal only works while it stays readable; the second does not depend
on it. Neither replaces exiting the session first.
