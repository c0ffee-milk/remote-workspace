# Recipes

Verbatim command templates referenced from `SKILL.md`. Replace placeholders:

- `<host>` — SSH alias, hostname, IP, or `user@host`
- `<ws>` — confirmed absolute remote workspace path
- `<path>` — file path relative to `<ws>`
- `<name>` — short job name, `[a-zA-Z0-9_-]+` only
- `<pattern>` — search text for recipes §3 (the `rg` form treats it as a regex)
- `<cmd>` — the command a long job runs; recipes §5a puts it in `run.sh`

All commands run from the local shell. Every remote command is wrapped in single quotes; if a template needs a nested quote, it sends content through stdin instead.

## 1. Master connection check

Run before anything else. A live master connection means later commands skip authentication entirely.

```sh
ssh -O check <host>
```

- Exit 0 and output `Master running (pid=...)` — proceed.
- Anything else — there is no master yet. On a key host, one command creates it implicitly:

```sh
ssh -o BatchMode=yes -o ConnectTimeout=10 <host> true
```

  `BatchMode=yes` keeps this safe on password hosts: it fails fast (exit 255) instead of blocking on a prompt. Re-run the check:

```sh
ssh -O check <host>
```

- Check still failing — the host needs an interactive login. Tell the user to run this themselves (in Claude Code, prefix with `!`), enter the password or 2FA once, then continue:

```sh
ssh -fNM <host>
```

Re-run the check after the user confirms. Do not use `sshpass`, do not put passwords on the command line.

A cold `ssh -O check` fails on a key host too — `-O` never opens a connection, so the implicit master only appears after the probe above. If the master still does not appear after the probe on a host you know uses keys, `~/.ssh/config` is missing the `Host *` ControlMaster block; report that to the user instead of editing the config.

## 2. Environment probe

Run once, immediately after the workspace is confirmed. Every item tolerates absence; the command never fails as a whole.

```sh
ssh <host> 'cd <ws> && echo "shell=$SHELL" && for t in python3 rg git tmux nvidia-smi conda bash; do echo "$t=$(command -v $t 2>/dev/null || echo missing)"; done && echo "python3_version=$(python3 --version 2>&1)" && echo "gpu=$(nvidia-smi --query-gpu=name,memory.used,memory.total --format=csv,noheader 2>/dev/null | tr "\n" ";" || echo none)" && echo "disk=$(df -h . | tail -1 | awk "{print \$4\" free of \"\$2}")" && echo "venv=$(ls -d .venv venv 2>/dev/null | tr "\n" " ")" && echo "git_repo=$([ -d .git ] && echo yes || echo no)"'
```

Record the output as session facts and echo them to the user once as a short table:

| Fact | Value |
| --- | --- |
| host | `<host>` |
| workspace | `<ws>` |
| python3 | path + version, or `missing` |
| rg | path or `missing` |
| git repo | yes / no |
| gpu | summary or `none` |
| disk free | value |

Repeat the echo only when a fact changes (host or workspace switch).

## 3. Search

If probe says `rg` exists:

```sh
ssh <host> 'cd <ws> && rg -n "<pattern>" --glob "!.agent-runs/**"'
```

Otherwise:

```sh
ssh <host> 'cd <ws> && grep -rn --exclude-dir=.git --exclude-dir=.agent-runs --exclude-dir=node_modules --exclude-dir=__pycache__ "<pattern>" .'
```

List files:

```sh
ssh <host> 'cd <ws> && find . -type f -not -path "./.git/*" -not -path "./.agent-runs/*" | head -200'
```

## 4. File edits

### 4a. Exact-match replace (preferred, needs remote python3)

Semantics match a local `Edit` tool: the old text must occur exactly once, otherwise nothing is written.

Step 1 — write the script locally to `/tmp/rw-edit.py`. Use raw triple-quoted strings. If either block contains `"""` or ends with a backslash, use 4b instead.

```python
import sys, pathlib
p = pathlib.Path("<path>")
old = r"""<exact old text>"""
new = r"""<new text>"""
s = p.read_text()
n = s.count(old)
if n != 1:
    sys.exit(f"expected exactly 1 match, found {n}")
p.write_text(s.replace(old, new))
print("ok")
```

Step 2 — back up, apply, and read the diff back:

```sh
ssh <host> 'cd <ws> && cp <path> <path>.bak && python3 -' < /tmp/rw-edit.py
ssh <host> 'cd <ws> && diff <path>.bak <path>; echo "diff_exit=$?"'
```

`diff_exit=1` with the expected hunks means success. `diff_exit=0` means nothing changed (the script must have failed; read its stderr). Then:

- Diff is right: `ssh <host> 'cd <ws> && rm <path>.bak'`
- Diff is wrong: `ssh <host> 'cd <ws> && mv <path>.bak <path>'` and retry.

Never leave `.bak` files behind.

### 4b. Whole-file replace (fallback)

Write the complete new content to a local temp file, then:

```sh
ssh <host> 'cd <ws> && cp <path> <path>.bak && cat > <path>' < /tmp/rw-newfile
ssh <host> 'cd <ws> && diff <path>.bak <path>; echo "diff_exit=$?"'
```

Same confirm/restore/cleanup rules as 4a. For a brand-new file, skip the `cp` and the diff; verify with `cat` instead.

## 5. Long-running jobs

Anything expected to run longer than about a minute, or that must survive the SSH session ending.

### 5a. Start

Step 1 — create the run directory and write `run.sh` through stdin (no nested quoting):

```sh
ssh <host> 'mkdir -p <ws>/.agent-runs/<name> && cat > <ws>/.agent-runs/<name>/run.sh' <<'EOF'
cd <ws>
<cmd>
echo $? > <ws>/.agent-runs/<name>/exit_code
EOF
```

Step 2 — launch detached and record the pid:

```sh
ssh <host> 'cd <ws>/.agent-runs/<name> && rm -f exit_code && ( nohup sh run.sh > stdout.log 2>&1 < /dev/null & echo $! > pid )'
```

The subshell around the launch is load-bearing. Without it, ssh keeps the channel open until the job finishes and the caller blocks for the job's whole runtime (measured: a 5s job blocked the caller for 5s). The `pid` write sits inside the subshell so `$!` still resolves to the job. Dropping the subshell and keeping everything else blocked the caller every time: plain `&`, `nohup ... &`, `setsid ... &`, and `ssh -n` each waited out the job's full runtime.

Use `bash run.sh` instead of `sh run.sh` if `<cmd>` needs bash features and the probe found bash.

Step 3 — if the workspace is a git repo, keep run artifacts out of it:

```sh
ssh <host> 'cd <ws> && [ -d .git ] && { grep -qx ".agent-runs/" .gitignore 2>/dev/null || echo ".agent-runs/" >> .gitignore; }; true'
```

### 5b. Status

```sh
ssh <host> 'cd <ws>/.agent-runs/<name> && if [ -f exit_code ]; then echo "state=exited code=$(cat exit_code)"; elif kill -0 "$(cat pid)" 2>/dev/null; then echo "state=running"; else echo "state=dead-without-exit-code"; fi; echo "--- last 20 lines ---"; tail -n 20 stdout.log'
```

Three states, and only one of them is "done":

- `state=exited code=0` — done. Only now may completion be reported.
- `state=exited code=N` (N≠0) — failed. Read the log, report the failure.
- `state=running` — still going. Report progress from the log tail, never completion.
- `state=dead-without-exit-code` — the process died without reaching the `echo` (OOM kill, reboot, `kill -9`). Treat as failed; say so explicitly.

### 5c. Stop

```sh
ssh <host> 'cd <ws>/.agent-runs/<name> && p=$(cat pid); kids=$(pgrep -P "$p" 2>/dev/null); kill -TERM $kids "$p" 2>/dev/null; sleep 1; kill -KILL $kids "$p" 2>/dev/null; true'
```

Children are collected and signalled **before** the parent, and the pid list is captured up front. Killing the parent first reparents its children to init, after which nothing can find them — the measured result was a `sleep 30` that outlived its own job's stop.

The list only holds direct children, so a job that forks its own grandchildren can leave strays behind. Stopping is best-effort: always run 5b afterwards to confirm the state, and check for leftover processes when it matters.
