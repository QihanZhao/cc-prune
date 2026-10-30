# cc-prune

Surgical context recovery for Claude Code sessions.

When a session approaches its context limit, the usual answer is `/compact` — which
summarizes everything, including the reasoning you may still want. `cc-prune` takes a
narrower cut: it clears **tool output that Claude has already digested** from the
session transcript, leaving the conversation, the reasoning, and the structure intact.

Nothing is summarized. No line is deleted. Thinking blocks are never touched, unless you
pass the opt-in `--drop-all-thinking` or `--drop-failing-thinking`
([Troubleshooting](docs/troubleshooting.md)). `cut` is the one command that removes
conversation: a stretch of whole turns you pick ([Cutting turns](#cutting-turns)).

```
86.8% full  ──[ clear --results ]──►  57% full
```

That is a real measurement from one session. Your mileage will vary a lot — see
[What to expect](#what-to-expect).

---

<div align="center">

[Scope](#honest-scope) · [Install](#install) · [Quick start](#quick-start) · [Workflow](#workflow) · [Cutting](#cutting-turns) · [Troubleshoot](#troubleshooting) · [Commands](#commands) · [Invariants](#invariants)

</div>

---

## Honest scope

**This is not lossless.** Cleared tool results are gone from the model's view. It works
because Claude has usually already written its conclusions into its response text
before you get here — but if you clear a result the model still needs, it has to re-run
the command or re-read the file. `cc-prune audit` exists to help you check before you cut.

**This is reverse-engineered.** The transcript format is undocumented and changes
between Claude Code versions. Everything here was derived by measurement, not from
a spec. It may break on your version. Keep the snapshot.

**This is not a substitute for `/compact`.** If your session is dominated by *thinking*
rather than tool output, `cc-prune` cannot help much — thinking blocks are
cryptographically signed and cannot be edited (editing one makes the API reject the turn or
drop the block).
`inspect` will tell you which situation you are in.

---

## Install

```bash
git clone https://github.com/QihanZhao/cc-prune
cd cc-prune
python3 cc_prune.py --version
```

Python 3.9+, no dependencies.

---

## Quick start

1. **Exit the Claude Code session** (`exit`, not `kill`). Commands that edit a transcript
   refuse one that looks live; `--force` overrides that. A session suspended with ctrl+z is
   still running: bring it back with `fg` and exit it.
2. **Find the transcript.** This lists your sessions, newest first, with their context size:

   ```bash
   python3 cc_prune.py sessions
   F=$(python3 cc_prune.py sessions --paths --limit 1)   # or the path of the one you want
   ```

3. **Prune.** This is the step that frees context. It clears old tool results and tool
   inputs, deletes the thinking blocks, takes a snapshot first, and rewrites the recorded
   usage so the session can be resumed:

   ```bash
   python3 cc_prune.py clear "$F" --results 1 --inputs 1 --images --drop-all-thinking --yes
   ```

   The thinking flag keeps the file from growing after the resume. A prune makes Claude
   Code's API drop the thinking blocks that came after the first edit and report it on every
   record, which can add tens of KB per record; with no thinking left there is nothing to
   drop. It changes the file's size, not the model's context. Leave it off (and `--yes`,
   which only it needs) to keep the reasoning in the transcript.

4. **Resume, send a real message, and read the context from the status bar.** The number
   the tool printed is an estimate; the first reply replaces it with the API's own count.
   To undo: `python3 cc_prune.py restore "$F" 0`.

That is the whole job. Everything below is detail.

**Pruned and resumed before without the thinking flag?** Then the file already holds the
saved reports, and step 3 stops new ones but leaves those. Wait 30 seconds after step 3 (the
live-session check counts the tool's own write) and remove them:

```bash
python3 cc_prune.py clear "$F" --transformations --keep-tail 0
```

On a session's first prune there is nothing to remove. The thinking flags are experimental;
see [Troubleshooting](docs/troubleshooting.md).

---

## The one rule

**Claude Code holds the conversation in memory and writes it back.** If a session is
running when you edit its transcript, your changes get overwritten — or worse, half
overwritten.

```bash
pgrep -af claude | grep -v grep     # must be empty
```

If it isn't, check `ls -l /proc/<pid>/cwd` to see which project it belongs to. Exit that
session normally (`exit`, not `kill`) before continuing.

---

## Workflow

### 1. Find the transcript

Transcripts live under `~/.claude/projects/<slugified-cwd>/<session-id>.jsonl`. The
directory is keyed on the working directory the session started in, not the project.
`sessions` lists them, newest first, with their context size:

```bash
python3 cc_prune.py sessions --sort ctx --limit 10   # biggest context first
python3 cc_prune.py sessions --grep vpn              # match title, prompt, cwd or id
F=$(python3 cc_prune.py sessions --paths --limit 1)  # newest session
```

Or by id, without `sessions`:

```bash
F=$(find ~/.claude/projects -name '<session-id>.jsonl' | head -1)
```

### 2. Get the real number

```bash
python3 cc_prune.py usage "$F"
```

`last request` is the input token count the API actually charged. This is the only
number you should trust. See [The /context trap](#the-context-trap).

### 3. See where the bytes are

```bash
python3 cc_prune.py inspect "$F"
```

Buckets, in rough order of how much they are worth clearing:

| bucket | flag | verdict |
|---|---|---|
| `results` | `--results N` | tool output. Safest, usually the biggest win. |
| `inputs` | `--inputs N` | `Write`/`Edit` bodies. Real semantic cost — the model forgets what it wrote. `file_path` is preserved. |
| `commands` | `--commands N` | `Bash` `command` values (`--inputs` never touches them). Long heredocs add up; the model forgets exactly what it ran. |
| `attachments` | `--attachments N` | mostly harmless, but the system-prompt and tool-schema attachments are skipped (clearing them broke a resume). Verify before trusting. |
| `images` | `--images` | billed by pixels, not characters. ~0 tokens. Skip. |
| `thinking` | `--drop-all-thinking` (experimental) | **locked by default.** Signed, and each block is bound to everything before it, so a prune makes the API drop later blocks anyway. The flag deletes them. See [Troubleshooting](docs/troubleshooting.md). |
| `transformations` | `--transformations` (experimental) | the API's own `input_transformations` report, saved on assistant records. Not conversation. |
| `text` | — | the conversation. No flag, by design. |

`inspect` reports **bytes, not tokens** — deliberately. See [Density](#density-varies-within-one-file).

### 4. Cut

```bash
python3 cc_prune.py clear "$F" --results 500
```

`N` is a length threshold in characters: results *longer* than `N` are replaced whole
(not truncated to `N`). Use `--results 1` to clear the bucket entirely.

`--keep-tail` (default `0.15`) protects the most recent records so the model can still
see what it just did. Lowering it recovers a little more and costs a lot of continuity.

Every `clear` writes a numbered snapshot first, validates the result before replacing
the original, and refuses to write if any invariant broke.

`clear` also rewrites the recorded `usage` on the last 5 requests to an estimate of the
new size; without it a session that was near its limit refuses to resume. The next real
reply replaces the estimate with the API's own count ([how it works](docs/estimate.md)).
`--no-reset-usage` leaves usage alone.

### 5. Verify

Resume the session and **send a real message** — not a slash command. Read the status
bar.

```bash
claude --resume <session-id>
```

### 6. Roll back if needed

```bash
python3 cc_prune.py snapshots "$F"
python3 cc_prune.py restore  "$F" 0     # 0 = pristine
```

---

## Troubleshooting

Symptoms after a prune, with fixes, in [docs/troubleshooting.md](docs/troubleshooting.md):

- `Context limit reached` or `Prompt is too long` right after resuming
- `tools.0.custom.input_schema: JSON schema is invalid` after resuming
- the transcript grows tens of KB per turn after a prune
- `clear` refuses a transcript because a session looks live

---

## What to expect

Three sessions from the same project, same week:

| session | thinking | results | thinking:results | recovered |
|---|---|---|---|---|
| A | 0.88 MB | 0.78 MB | 1.1× | 98% → 72% |
| B | 1.50 MB | 0.52 MB | 2.9× | 86.8% → 57% |
| C | 1.92 MB | 0.41 MB | 4.7× | 92.9% → 80% |

**The `thinking:results` ratio predicts almost everything.** Tool-heavy sessions recover
well. Reasoning-heavy sessions don't, because the one large bucket is the one you can't
touch. Run `inspect` first and decide accordingly.

---

## Things we measured

Three findings, all from direct measurement against the API's own token counts.

### The `/context` trap

`/context` recomputes usage **locally**. The status bar reflects the `usage` field from
the last real API response. They normally agree — and diverge completely once you edit
the transcript.

We nearly threw away a successful prune because of this: the status bar read 72%,
`/context` read 98%, and it looked like the context had ballooned back. It hadn't. The
next real message settled at 72%.

> After editing a transcript, verify with `cc_prune.py usage` or by sending a real
> message. Never with `/context`.

This also means **auto-compaction may fire on the stale estimate**. Consider
`claude --resume <id> --autocompact 1M` while working on a pruned session.

### Images are billed by pixels, not characters

Three PNGs occupied 795,360 characters of base64 in the transcript. Clearing them moved
real usage by roughly nothing — image tokens scale with `width × height / 750`, so those
three were about 3,200 tokens total, not the ~300,000 their byte count suggests.

> Transcript size and context pressure are only loosely related. `--images` is almost
> never worth a cache miss.

### Density varies within one file

Characters per token, measured in a single session:

- numeric training logs, tensor shapes, test output: **~1.5 chars/token**
- prose reasoning: **~4.0 chars/token**

A 2.7× spread. Any single conversion constant will mislead you — ours did, repeatedly,
until we stopped estimating and started measuring. That is why `inspect` reports bytes
and refuses to guess tokens.

To learn a bucket's real density in *your* session:

```bash
python3 cc_prune.py density --before 867650 --after 570000 --mb 0.45
# => 1.51 chars/token for that bucket
```

---

## Cutting turns

When most of what is left is conversation, clearing buckets stops paying. `cut` removes a
stretch of whole turns from the middle and keeps the start and the recent end verbatim.

```bash
python3 cc_prune.py cut "$F" --list                          # numbered turns, sizes, prompts
python3 cc_prune.py cut "$F" --keep-head 5 --drop 0.5        # preview
python3 cc_prune.py cut "$F" --keep-head 5 --drop 0.5 --yes  # apply
```

The model loses everything in the cut turns, and the first kept prompt gets a note saying
so. Details, flags and what was tested: [docs/cut.md](docs/cut.md).

---

## Preventing the problem

Surgery is triage. The durable fixes are upstream:

```bash
export BASH_MAX_OUTPUT_LENGTH=4000     # default is 30000
```

- Pipe long output through `head`/`tail`/`grep`; redirect to a file and read selectively.
- `Grep` to locate, then `Read` with `offset`/`limit`. Avoid whole-file reads.
- Delegate heavy work to subagents — their context is separate, and their thinking never
  enters the main thread. This is the only structural lever against thinking growth.

---

## Commands

```
sessions [--project X ...]     list and filter session transcripts
usage FILE                     real API token counts
inspect FILE                   byte accounting by bucket
clear FILE --results N ...     clear buckets (snapshots first, resets usage)
reset-usage FILE --tokens N    rewrite stale recorded usage so a pruned session resumes
snapshots FILE                 list snapshots
restore FILE N                 restore snapshot N
audit FILE LINE...             is this specific result safe to clear?
strip-commands FILE --orphans  clear /context, /cost output
cut FILE --drop F | --from A --to B   cut a stretch of turns (previews unless --yes)
clearcut FILE --drop F         cut, then clear, with one snapshot (previews unless --yes)
density --before --after --mb  derive chars/token from a measured cut
```

### `audit`

Before clearing a large result, check whether it matters:

```bash
python3 cc_prune.py audit "$F" 2328 2355
```

Reports which call produced it, whether the source file still exists on disk, whether it
was read again later, how often it's referenced downstream, and what the model said
immediately after. A result whose file is still on disk and never referenced again is
safe to clear; one whose source is gone and is still an active topic is not.

---

## Invariants

Every mutating command validates before replacing the original, and aborts leaving the
file untouched if any of these fail:

- record count and `uuid` / `parentUuid` chains unchanged
- `thinking` / `redacted_thinking` blocks identical, apart from those a thinking flag was
  asked to delete
- every `tool_result.tool_use_id` still has a matching `tool_use`

`cut` changes the chain on purpose and checks its own set instead; see
[docs/cut.md](docs/cut.md).

---

## Contributing

The transcript schema is undocumented. If `inspect` reports a large `unattributed`
share, or `usage` finds no fields, please open an issue with your Claude Code version
and the output of:

```bash
python3 cc_prune.py inspect "$F" | head -30
```

Please redact paths and project names first — transcripts contain your work.

---

## License

MIT