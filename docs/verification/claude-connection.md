# Claude connection failures

Claude Code is a CLI that calls its API itself: `https://api.anthropic.com`, or a custom endpoint's `ANTHROPIC_BASE_URL`
(`server/claude-endpoints.ts`). OpenMausBot's own servers are reached over `http://127.0.0.1`. A refused connection arrives as
an `assistant` frame (`error: "server_error"`, "API Error: Connection refused — a firewall or proxy may be blocking it
(ECONNREFUSED)") and an error `result`, and the process stays up (captured from Claude Code 2.1.327).

## What the driver does

- Loopback never rides a proxy: `claudeEnvironment` in `server/drivers/claude.ts` adds `127.0.0.1,localhost,::1` to `NO_PROXY`
  and `no_proxy` for every spawn (turns, sign-in probes, helper calls), whether or not `OMB_OWN_MCP_HTTP` is on.
- A refused connection is not announced as it arrives. The driver ends that process, and its close sends the turn again on a
  fresh one with `--resume`, by the rules in `server/drivers/retry.ts`: three launches, 1 s and 3 s apart, shown as "retrying"
  rows. A turn that already streamed text or had a model call answered is never sent again.
- When the retries are spent or not allowed, the chat gets one error naming where the CLI was calling and the proxy variables it
  had (`Target: https://api.anthropic.com. No proxy set.`). The target counts a gateway the person set in their Claude settings
  (`env.ANTHROPIC_BASE_URL`), which OpenMausBot passes on to each turn. Credentials in a proxy URL are masked.
- That failure also makes the engine `unavailable`, with the same words as its reason, in `GET /api/instances` and Settings →
  Engines. A turn or helper call that gets through clears it, and so does a passing `POST /api/instances/:id/check`. A cached read can
  lag by up to a minute; `?fresh=1` probes again.

## Driving it

`FAKE_CLAUDE_REFUSED_RESULTS=N` (with `FAKE_CLAUDE_STATE`, a launch counter file) makes the first N launches of
`server/testing/fake-claude-cli.ts` answer those frames and stay alive; `FAKE_CLAUDE_PARTIAL_FAILS=1` streams text first.

```sh
FAKE_CLAUDE_REFUSED_RESULTS=1 FAKE_CLAUDE_STATE=/tmp/omb-refused-launches FAKE_CLAUDE_RETRY_SCALE=0.01 \
  node --experimental-strip-types scripts/control-omb.ts launch
pnpm control:omb bots --url http://127.0.0.1:PORT
pnpm control:omb run --bot BOT_ID --text "ping" --wait turn --url http://127.0.0.1:PORT
pnpm control:omb messages --bot BOT_ID --url http://127.0.0.1:PORT
```

The run settles and the transcript holds `retrying — attempt 2/3 … connection_reset`, then the reply. Launch again with
`FAKE_CLAUDE_REFUSED_RESULTS=9` and a fresh `FAKE_CLAUDE_STATE`: the run fails after three launches, the last row reads
`error: API Error: Connection refused … Target: https://api.anthropic.com. No proxy set.`, and
`curl "$URL/api/instances?fresh=1"` shows the Claude instance `unavailable` with that reason.

## Regression checks

```sh
pnpm exec vitest run server/drivers/claude.test.ts server/drivers/own-mcp-http.test.ts server/drivers/engine-errors.test.ts
```
