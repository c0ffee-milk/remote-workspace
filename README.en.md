# Remote Workspace

**English** | [中文](README.md)

> An agent skill that turns any SSH host into a machine your agent can actually work on.

`remote-workspace` teaches a coding agent (Claude Code, Codex, Pi, …) how to open an
SSH connection to a remote machine and then work there with the same rigor it uses
locally: probe the environment, search, edit files without losing data, run
long-running jobs, and never report a job as finished before it actually is.

Works with any agent that can read a Markdown skill file and run shell commands.

## The problem

Agents are good at `ssh host 'command'` and bad at everything after it:

- **Credentials.** It cannot type your password or 2FA code, so it either stalls or
  invents a workaround. (It should ask you to authenticate once.)
- **Blind guesses.** `rg`, `python`, `bash`, `tmux`, `conda` — none of them can be
  assumed on a remote box, but the agent assumes anyway and fails.
- **Destructive edits.** `sed -i`, here-docs nested three quoting levels deep, and no
  diff read back afterwards. Files get quietly mangled.
- **Fake completion.** Launching a background job, seeing the shell return, and
  announcing success — while the job is still running, or died at startup.
- **Leaked processes.** Stopping a job by killing the parent, leaving its children
  behind forever.

This skill is the missing operating manual for all five.

## What it does

| Step | Behavior |
| --- | --- |
| 0 | Checks for a live SSH master connection (`ssh -O check`); if absent, probes with `BatchMode` so key hosts connect silently and password hosts fail fast instead of hanging, then hands the login to you |
| 1 | Verifies the host is reachable and reports the remote working directory |
| 2 | Confirms and normalizes the remote workspace — never assumes a path from an earlier session |
| 3 | Probes the remote environment **once** (`shell`, `python3`, `rg`, `git`, `tmux`, GPU, disk, venv, git repo) and echoes it back as a short table; every later choice follows from this probe |
| 4 | Runs every command from the confirmed workspace, picking `rg` or `grep` based on the probe |
| 5 | Edits files by exact-match replacement (must match exactly once, or nothing is written), with backup → apply → read the diff back → delete or restore |
| 6 | Runs anything longer than a minute detached under `.agent-runs/<name>/`, and gates completion on a written `exit_code` file |
| 7 | Handles switching to a different host or workspace by redoing steps 0–3 |

## Why it is safe to point at real servers

- It **never supplies passwords** and **never edits `~/.ssh/config`**. It checks
  connection reuse and reports what is missing; you configure it.
- Every file edit is **backed up, applied, and diffed back** before the backup is
  removed. If the diff is wrong, the backup is restored. A `.bak` file is never left
  behind.
- The exact-match edit **refuses to write** unless the old text occurs exactly once,
  so a stale or ambiguous patch cannot half-apply.
- A long job is only "done" when `.agent-runs/<name>/exit_code` contains `0`. Three
  states are recognized — `running`, `exited code=N`, `dead-without-exit-code` — and
  only the first two are readable as progress. A job that was killed reports
  `dead-without-exit-code` rather than pretending to have finished.
- Destructive remote commands are not run unless you explicitly ask.

## Requirements

- An OpenSSH client locally (`ssh`), and a host you can already reach.
- A POSIX shell on the remote host. Everything else is detected, not assumed:
  `python3` enables exact-match edits (`grep` fallback otherwise), `rg` speeds up
  search, `pgrep`/`pkill` improve job stopping.
- Recommended, so authentication happens once per host instead of per command:

  ```
  # ~/.ssh/config
  Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
    ServerAliveInterval 30
  ```

  The skill only checks that this is in effect. If it is missing, it tells you.

## Install

Clone into your agent's skill directory, e.g.:

```sh
# Claude Code
git clone https://github.com/c0ffee-milk/remote-workspace.git ~/.claude/skills/remote-workspace

# Pi
git clone https://github.com/c0ffee-milk/remote-workspace.git ~/.pi/agent/skills/remote-workspace

# Codex
git clone https://github.com/c0ffee-milk/remote-workspace.git ~/.codex/skills/remote-workspace
```

Pi can also consume skills installed for another harness instead of duplicating the
clone — add the directory to settings (`~/.pi/agent/settings.json`):

```json
{
  "skills": ["~/.claude/skills"]
}
```

## Usage

Trigger it by asking for remote work in plain language:

```
连接远程服务器 my-gpu-box           # any SSH alias, hostname, IP, or user@host
登录服务器 my-gpu-box 并在 /home/me/proj 里跑训练
ssh 到 user@10.0.0.5 and check why the last job failed
connect to my-gpu-box, the workspace is /srv/app — find where AUTH_TIMEOUT is set
```

You will be asked for the host first, then the workspace, if you did not supply them.
On a password/2FA host you will be asked to authenticate once yourself; after that,
every command reuses the connection.

## Repository layout

```
remote-workspace/
├── SKILL.md              # the workflow: triggers, session facts, steps 0–7, rules, checklist
├── references/
│   └── recipes.md        # five verbatim command templates, referenced by § number
├── README.md             # 中文说明 (shown by default)
├── README.en.md          # this file
└── LICENSE               # MIT
```

`SKILL.md` says **what** and **when**; `references/recipes.md` says **exactly how**.
Copy a recipe, replace `<host>`, `<ws>`, `<path>`, `<name>`, `<pattern>`, `<cmd>`, and
it runs. The split means you can adapt a command template without touching the
workflow, and vice versa.

## Verified against a real host

Every recipe in `references/recipes.md` was executed end to end on a live Linux host
(bash, `python3.8`, no `ripgrep`, 4× GPU) before being committed — including the two
negative cases: an edit whose old text does not exist must fail without touching the
file, and a killed job must report `dead-without-exit-code` rather than success.

That testing found and fixed two real bugs, both of which are documented in the
recipes themselves: launching a job with `... & echo $! > pid` blocks the caller for
the job's entire runtime (the launch needs a subshell), and stopping a job by killing
the parent first orphans its children (signal children before parents).

## Contributing

Issues and pull requests are welcome. Two rules for changes to the recipes:

1. **Run it on a real host first.** A command template that has never executed is
   worth nothing — say which host and shell you tested.
2. **Keep `SKILL.md` and `references/recipes.md` in sync.** If a workflow step needs
   a command, add a numbered recipe section and reference it by § number.

## License

MIT — see [LICENSE](LICENSE).
