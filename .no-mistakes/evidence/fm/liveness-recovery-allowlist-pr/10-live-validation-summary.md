# Live validation: agy/cursor secondmate automatic recovery

Change under test: `bin/fm-secondmate-liveness-lib.sh` adds `agy` and `cursor` to the
recovery-verified harness allowlist so the session-start secondmate liveness sweep
relaunches a dead/missing mate on either runtime instead of skipping it as
"unverified for recovery".

## Setup (disposable, torn down)

- Lab home: `bin/fm-lab-home.sh create /tmp/fm-lab.*/home` with `config/backend=tmux`,
  `config/secondmate-harness=agy` (then `cursor`), supervision-host-off.
- Private tmux server on `TMUX_TMPDIR=/tmp/fm-lab.*/tmux`, session `firstmate`.
- Real secondmate homes seeded with `bin/fm-home-seed.sh --no-projects`, launched with
  the real `bin/fm-spawn.sh --secondmate`, producing real `agy` and `cursor-agent`
  processes in tmux panes (`pane_current_command=agy` / `cursor-agent`).
- Failure simulated the way it happens in the field: SIGTERM the agent process leaving
  the pane's bare shell (backend state `dead`), or kill the whole window (`missing`).
- Sweep driven by the real product entrypoint `bin/fm-bootstrap.sh` with
  `FM_HOME` on the lab and no test stubs on PATH.

## Results

| Scenario | Pre-fix | Post-fix |
|---|---|---|
| Dead agy mate (bare shell) | `SECONDMATE_LIVENESS: secondmate labmate: skipped: recorded harness 'agy' is unverified for recovery (backend=tmux)`; no relaunch | silent relaunch; ledger `attempt` -> `relaunched`; pane back to `agy` |
| Missing agy window | -- | window recreated; pane `agy`; ledger `attempt` -> `relaunched` |
| Dead cursor mate | -- | silent relaunch; pane `cursor-agent`; ledger `attempt` -> `relaunched` |
| Missing cursor window | -- | window recreated; pane `cursor-agent`; ledger `attempt` -> `relaunched` |
| Guard: harness outside the allowlist (`gemini`, dead pane) | -- | still `skipped: recorded harness 'gemini' is unverified for recovery`; window untouched, no ledger |
| Alive agy + cursor mates | -- | sweep printed nothing about them; relaunch ledgers byte-identical (untouched) |

The agy pane scrollback after automatic recovery shows the live agent idling
("session start and state reconciliation are complete ... idling silently").
See `08-pane-and-ledger-state.txt`.

The new regression tests also fail on the pre-fix allowlist
(`not ok - a recorded agy secondmate should be verified for recovery`) and pass with
the fix; full targeted suite result in `09-targeted-liveness-suite.txt`.
