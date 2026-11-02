# Changelog

## 0.3.0 (2026-10-30)

### Added

- `clearcut FILE --drop F [--keep-head N] [--keep-tail F] --yes`: a `cut` followed by a
  `clear` in one call. The cut uses `--purge` and keeps the first turn by default
  (`--keep-head 1`, `--branch` keeps the cut turns as an unreachable branch instead); the
  clear uses `--results 1 --inputs 1` and `--images` (`--no-images` leaves them). A single
  snapshot is taken before the cut, so one `restore` undoes both steps. Without `--yes` it
  previews the cut only and prints the flags the clear would run with. `--keep-tail`,
  `--commands`, `--attachments`, `--transformations`, `--drop-failing-thinking`,
  `--drop-all-thinking`, `--no-note`, `--no-reset-usage` and `--force` pass through to
  the two commands. It calls `cmd_cut` and `cmd_clear` through the real subparsers, so
  their defaults stay in one place, and the result is byte-identical to running the two
  by hand. The live-session guard applies to the cut; the clear step then runs with
  `--force`, because the cut has just rewritten the file.

### Changed

- The argument parser is built by `build_parser()`. `clear` has a hidden `--no-snapshot`,
  used by `clearcut`. The defaults of `cut` and `clear` are unchanged.

## 0.2.2 (2026-10-30)

### Changed

- The usage estimate `clear` and `cut` write is rebuilt. It splits the conversation at
  the requests made since the last edit, whose recorded counts are exact, and estimates
  only how each segment divides among its records (text by characters, thinking by
  signature length). Thinking the edit deletes or invalidates now counts as removed,
  carried attachments count as moved, and an edit right after another starts from the
  snapshot that still holds real counts. The result is lowered by 2.5%. The old estimate
  (removed characters at 1.5 chars/token) was up to 12% over the real count; on 27 real
  before/after pairs the new one was under on every pair, by 0.3% to 15%.
- README is back to the original's structure and length, plus the Quick start. The
  Troubleshooting section, the `cut` documentation and the usage-estimate explanation moved
  to `docs/troubleshooting.md`, `docs/cut.md` and `docs/estimate.md`, with short sections
  in the README linking to them.
- Records whose usage cc-prune rewrote carry `"ccPruneEstimate": true`.
- Character counts for tool results and attachments no longer include JSON keys and
  escaping, which changes the sizes `cut --list` shows and the `--drop` boundaries a little.

### Fixed

- `cut` on a session cut before: the note now carries the running total. A note inside
  the cut stretch is added to the new count (it used to be dropped with its turn), and a
  note already on the first kept prompt is merged instead of stacked. The preview shows
  the earlier total.
- `sessions` showed the auto-generated title for a renamed session. `custom-title` records
  inside the transcript are now read and win over `ai-title` and `summary`; the latest
  one counts.
- Sessions written by Claude Code 2.1.300 and up carry `api-request` and
  `api-request-blob` records, and the blobs hold copies of tool results. The tool_use /
  tool_result pairing check in `validate` and `validate_cut` counted those copies, so a
  `cut --purge` aborted on results whose tool call sat in the part being deleted. Both
  checks now look at conversation records only.

## 0.2.1 (2026-10-14)

### Added

- `cut`: remove a stretch of whole turns from the middle of the conversation and keep the
  start and the recent end. `--keep-head N --drop F` cuts a fraction of the conversation
  after the head by size; `--from A --to B` cuts exact turns; `--list` numbers the turns.
  Previews unless `--yes`. The first kept turn is re-linked onto the head; the cut turns
  stay as an unreachable branch, or are deleted with `--purge`. Both ends land where no
  tool call is waiting for a result. State attachments in the cut stretch (system prompt,
  tool schemas, tool/MCP/agent/skill listings) are copied over. The first kept prompt gets
  a note unless `--no-note`. Recorded usage is reset to an estimate. Same live-session
  guard and snapshot as `clear`, with its own validation.
- `clear --commands N`: clear `Bash` `command` values longer than `N` characters.
  `--inputs` keeps them (they are in `KEEP_PARAMS`), so long heredocs and inline scripts
  had no flag before.

### Fixed

- A second `clear --inputs` pass re-cleared values the first pass had already replaced
  with `[cleared] N chars` and counted them as freed, which inflated the hit count and
  lowered the usage estimate. Those values are now skipped.

### Changed

- README Quick start: the prune step now includes `--drop-all-thinking --yes`, so a first
  prune is one command. The `--transformations` pass is documented only for sessions pruned
  and resumed before without the thinking flag. Suspended (ctrl+z) sessions count as running.

## 0.2.0 (2026-10-08)

Builds on 0.1.0, the original release by Qihan Zhao.

### Added

- `sessions`: list and filter Claude Code transcripts without knowing the session id.
  Shows start, last activity, project, context (`CTX`), a one-line "about", size and id.
  Filters and options: `--project`, `--grep`, `--since`, `--until`, `--last`, `--count`,
  `--sort {date,size,project,ctx}`, `--reverse`, `--limit`, `--all`, `--timezone`, `--json`,
  `--paths`, `--subagents`, `--root`.
- Running `cc_prune.py` with no arguments prints a quick start and the command list.
- `reset-usage FILE --tokens N`: rewrite the recorded token usage on the last requests.
  A pruned session that was near its limit is refused on resume ("Context limit reached")
  until the recorded usage is lowered.
- `clear` now does that automatically after pruning, from an estimate of the freed
  characters. Pass `--no-reset-usage` to leave usage alone.
- Live-session guard on `clear`, `strip-commands`, `reset-usage` and `restore`. A transcript
  is refused when a running Claude Code process is registered for it (including subagent
  logs; the process start time is checked on Linux) or when it was edited within
  `LIVE_IF_EDITED_WITHIN` seconds (default 30). `--force` overrides. Honors
  `CLAUDE_CONFIG_DIR`.
- A settings block near the top of `cc_prune.py`.
- Experimental, opt-in `clear` flags for the `input_transformations` field the API returns
  and Claude Code stores on every assistant record, which can make a pruned transcript grow
  by tens of KB per record:
  - `--transformations` deletes the saved reports.
  - `--drop-failing-thinking` deletes the thinking blocks the API reported dropping.
  - `--drop-all-thinking` deletes every thinking block.
  - The two thinking flags preview unless `--yes` is given, refuse layouts they have not
    been checked against, and keep the records in place with empty content.
- `inspect` reports the `transformations` bucket.
- README: sessions workflow, Troubleshooting section (resume refused, invalid tool schema,
  transcript growth), live-session documentation.

### Changed

- `restore` validates the snapshot first, saves the current file as a new snapshot, and
  swaps atomically, so a restore can no longer silently discard work done since the prune.
- Rewritten transcripts keep the original file mode. The temporary file is created `0600`.
- Validation compares whole thinking blocks, not only their signatures.
- `--attachments` skips `prompt_snapshot` and `deferred_tools_record` attachments. They hold
  the system prompt and tool schemas the API request is rebuilt from, and clearing them made
  a resume fail with `tools.0.custom.input_schema: JSON schema is invalid`.

### Fixed

- `strip-commands` could rewrite the text of a thinking block that contained
  `<local-command-stdout>` tags.
- Snapshot listing crashed on a stray `*.snapshot` file and treated glob characters in the
  path as wildcards.

### Known limits

- The thinking flags have been run on throwaway copies and a few real sessions only.
- The live-session guard was tested on Linux. The macOS and Windows branches are simulated.
- The transcript format is undocumented and changes between Claude Code versions.
- Python 3.9 is the stated minimum but only 3.13 has been run; the code parses under the 3.9
  grammar.
- The tool rewrites JSON with a space after `:` and `,`, which grows a rewritten file by
  about 3% compared with Claude Code's compact form.

## 0.1.0

Original release by Qihan Zhao: `usage`, `inspect`, `clear`, `snapshots`, `restore`,
`audit`, `strip-commands` and `density`.
