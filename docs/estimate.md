# How the usage estimate works

`clear` also rewrites the recorded `usage` on the last 5 requests to an estimate of the
new size. Without this a session that was near its limit refuses to resume, see
[Troubleshooting](troubleshooting.md). The next real reply replaces it with the API's own
count. Pass `--no-reset-usage` to leave usage alone.

How the estimate works (`clear` and `cut`): every request made since the last edit
recorded the exact token count of everything before it, so the conversation splits into
segments of known size. Only how a segment divides among its records is estimated: text
by characters, thinking by the length of its signature (Opus thinking shows little text
but counts in full; the signature carries it). Thinking the edit invalidates counts as
removed. When an edit follows another with no request in between, the estimate starts
from the snapshot that still holds real counts. The result is lowered by 2.5% so it lands
under the real count. Checked against 27 real before/after pairs from pruned sessions:
every estimate was under the first real count, by 0.3% to 15% (the larger gaps come from
what Claude Code adds on resume, which no estimate can see). Records whose usage cc-prune
rewrote carry `"ccPruneEstimate": true`, so a later estimate never treats them as real.
