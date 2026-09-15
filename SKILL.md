---
name: remote-workspace
description: Use when the user asks to connect to any remote server over SSH and work there; identify the remote host, connect first, confirm any provided remote workspace, probe the remote environment once, then run commands, edit files, and manage long-running jobs from that workspace with the same rigor as working locally.
---

# Remote Workspace

## Trigger

Use this skill when the user asks to connect to or operate on a remote server, including requests such as:

- `连接远程服务器`
- `登录服务器`
- `连接主机 <host>`
- `ssh 到 <host>`
- work on a named SSH host such as `YuwanZ0914`

The host can be an SSH config alias, hostname, IP address, or `user@host`. Do not assume a remote project directory from prior sessions unless the user provides or confirms it in the current session.

## Inputs to Identify

1. Remote host or SSH alias.
2. Remote workspace path, if supplied.
3. Any project-specific environment or interpreter path, if supplied or discovered.

If the host is missing, ask for the remote host before running commands. If the host is present but the workspace is missing, connect first, then ask for the remote workspace path before project-specific work.

## Session Facts

Maintain these facts for the current conversation and keep them current:

- active host
- active workspace (absolute path)
- master connection status
- `python3` path and version, or `missing`
- `rg` available or not
- workspace is a git repo or not
- GPU summary, or `none`
- running job names under `.agent-runs/`

Facts are session context only. Do not carry them into unrelated future conversations.

## Core Workflow

Command templates for every step live in [references/recipes.md](references/recipes.md). Use them verbatim.

0. **Check the master connection** (recipes §1). Run `ssh -O check <host>`. If there is no master, run the `BatchMode` probe from recipes §1 — on a key host it creates the master and you continue; on a password host it fails fast instead of blocking. Only if the check still fails, ask the user to run `ssh -fNM <host>` themselves so they can complete any password or 2FA prompt once, then re-check. Do not attempt to supply credentials.

1. **Verify the connection:**

   ```sh
   ssh <host> 'echo connected && pwd'
   ```

2. **Confirm the workspace.** If the user did not provide a remote folder, stop project-specific work and ask. If they did, verify and normalize it:

   ```sh
   ssh <host> 'cd <remote-workspace> && pwd'
   ```

   Treat the printed absolute path as the active workspace.

3. **Probe the environment once** (recipes §2). Record the results as session facts and echo them to the user as a short table. Every later choice (search tool, edit method, shell for jobs) follows from this probe, not from guesses.

4. **Run commands from the workspace.** Every remote command is `ssh <host> 'cd <ws> && ...'`. For search, follow recipes §3 based on whether `rg` exists.

5. **Edit files** (recipes §4). Prefer the exact-match replace script when `python3` exists; fall back to whole-file replace otherwise. Always: back up, apply, read the diff back, then delete the backup or restore from it. Report the diff, not just "edited".

6. **Run long jobs detached** (recipes §5). Anything over about a minute goes through `.agent-runs/<name>/`. Query state with the status recipe. Report completion only when the state is `exited code=0`.

7. **Switching host or workspace.** If the user names a different host or folder, repeat steps 0–3 for it and state the new active host/workspace explicitly.

## Operating Rules

- Keep the user informed of the active remote host and workspace.
- Avoid project-specific remote commands until both host and workspace are confirmed and the probe has run.
- Avoid destructive remote commands unless the user explicitly asks for them.
- Prefer short, verifiable commands before long-running jobs.
- Never report a long-running job as finished without an `exit_code` file containing `0`. "The command returned" is not "the job finished"; `state=running` is progress, not completion.
- Never leave `.bak` files behind after an edit. Confirm the diff, then delete or restore.
- Never edit `~/.ssh/config` or supply passwords. If connection reuse is not working, tell the user what is missing.
- Preserve the confirmed host and workspace as session context only; do not treat them as permanent defaults for unrelated future conversations.

## Remote Shell Habits

Remote shells differ from local shells. The probe answers most questions up front; for the rest:

- `rg` may be unavailable; use `grep -rn` with `--exclude-dir` (recipes §3).
- `python` may be unavailable; use the `python3` path from the probe, or a project-specific interpreter the user names.
- Noninteractive SSH shells may not load conda or shell init files; use full environment paths when needed.
- Nested quoting is fragile. Send file contents and scripts through stdin (`ssh <host> 'cat > file' < local` or `python3 -`), never through inline heredocs inside the ssh argument.
- Backgrounded processes must redirect stdin from `/dev/null` or the ssh session hangs.

## Validation Checklist

- `ssh -O check <host>` reports a running master, or the user was told exactly what to run.
- SSH command returns `connected`.
- If a workspace was provided, `cd <remote-workspace> && pwd` succeeds and the path is absolute.
- The environment probe ran once and its results were echoed to the user.
- Subsequent commands use `ssh <host> 'cd <remote-workspace> && ...'`.
- Every file edit produced a diff that was read back, and no `.bak` remains.
- Every long-running job has a `.agent-runs/<name>/` directory, and completion was reported only on `exit_code` = `0`.
- The final response states the active remote host/workspace when remote work continues.
