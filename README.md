# Remote Workspace

[English](README.en.md) | **中文**

> 一个 Agent Skill，把任意 SSH 主机变成你的 Agent 真正能干活的地方。

`remote-workspace` 教 coding agent（Claude Code、Codex、Pi 等）如何连上一台远程机器，
然后在上面以"和本地一样的严谨程度"工作：探测环境、搜索、安全地改文件、跑长任务，
并且**绝不在任务真正结束前宣称它完成了**。

只要你的 Agent 能读 Markdown 格式的 skill 文件、能执行 shell 命令，就能用。

## 解决什么问题

Agent 写 `ssh host 'command'` 很在行，但之后的事基本全错：

- **认证**：它没法替你输密码或 2FA，于是要么卡住，要么自创一个不靠谱的绕法。
  （正确做法是让你手动认证一次。）
- **瞎猜环境**：`rg`、`python`、`bash`、`tmux`、`conda` 在远程机器上没有一个能默认存在，
  但它照样按有来写，然后失败。
- **破坏性改文件**：`sed -i`、嵌了三层引号的 heredoc、改完不看 diff。文件被悄悄改坏。
- **假完成**：把任务丢到后台，看到命令返回就宣布成功——而任务还在跑，或者启动就死了。
- **孤儿进程**：停止任务时只 kill 父进程，子进程永远留在机器上。

这个 skill 就是针对这五件事的操作手册。

## 它会做什么

| 步骤 | 行为 |
| --- | --- |
| 0 | 检查 SSH 主连接是否存活（`ssh -O check`）；没有则用 `BatchMode` 探测一次——密钥主机静默建连，密码主机会快速失败而不是挂住——然后把登录交给你 |
| 1 | 验证主机可达，并报告远程当前目录 |
| 2 | 确认并规范化远程 workspace，绝不复用上一次会话的路径 |
| 3 | **只探测一次**远程环境（`shell`、`python3`、`rg`、`git`、`tmux`、GPU、磁盘、venv、是否 git 仓库），以短表回显；之后每个选择都依据这次探测，而不是猜 |
| 4 | 所有命令都在已确认的 workspace 内执行；根据探测结果选 `rg` 或 `grep` |
| 5 | 改文件走"精确匹配替换"（原文必须恰好出现一次，否则一个字都不写），流程是：备份 → 应用 → 读回 diff → 删除备份或还原 |
| 6 | 任何超过一分钟的任务都放到 `.agent-runs/<name>/` 里脱离终端运行，并且只有写出 `exit_code` 文件才算完成 |
| 7 | 切换到别的主机或 workspace 时，重做 0–3 步 |

## 为什么可以放心指向真实服务器

- **不代填任何密码，也不改 `~/.ssh/config`**。它只检查连接复用是否生效，缺什么就告诉你，配置由你自己来。
- 每次改文件都**先备份、再应用、然后读回 diff**，确认无误才删备份；对不上就还原。**绝不留下 `.bak` 文件**。
- 精确匹配替换在原文不是恰好出现一次时**拒绝写入**，所以过期的、歧义的补丁不可能只改一半。
- 长任务只有在 `.agent-runs/<name>/exit_code` 内容为 `0` 时才算完成。状态只有三种——
  `running`、`exited code=N`、`dead-without-exit-code`——**只有前两种（含 0）能读作结果**。
  被 kill 掉的任务会如实报 `dead-without-exit-code`，而不是伪装成跑完了。
- 除非你明确要求，否则不执行任何破坏性远程命令。

## 环境要求

- 本地有 OpenSSH 客户端（`ssh`），并且你本来就能连上那台主机。
- 远程需要一个 POSIX shell。其余全部靠探测得出，不做假设：
  `python3` 存在才能用精确匹配替换（否则退回整文件替换），`rg` 让搜索更快，
  `pgrep`/`pkill` 让停止任务更干净。
- 建议启用连接复用，这样每台主机只需认证一次：

  ```
  # ~/.ssh/config
  Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
    ServerAliveInterval 30
  ```

  这个 skill 只检查它是否生效；没生效会告诉你。

## 安装

克隆到你的 Agent 的 skill 目录，例如：

```sh
# Claude Code
git clone https://github.com/c0ffee-milk/remote-workspace.git ~/.claude/skills/remote-workspace

# Pi
git clone https://github.com/c0ffee-milk/remote-workspace.git ~/.pi/agent/skills/remote-workspace

# Codex
git clone https://github.com/c0ffee-milk/remote-workspace.git ~/.codex/skills/remote-workspace
```

如果你已经给别的 harness 装过，Pi 可以直接复用目录，不必再克隆一份——在
`~/.pi/agent/settings.json` 里加：

```json
{
  "skills": ["~/.claude/skills"]
}
```

## 使用

用自然语言触发即可：

```
连接远程服务器 my-gpu-box            # SSH 别名、主机名、IP、user@host 都行
登录服务器 my-gpu-box 并在 /home/me/proj 里跑训练
ssh 到 user@10.0.0.5，看看上一个任务为什么失败
connect to my-gpu-box, the workspace is /srv/app — find where AUTH_TIMEOUT is set
```

如果你没说主机，它会先问主机；说了主机没说目录，它会先问目录。密码/2FA 主机上，
它会请你亲自认证一次；之后所有命令都复用这条连接。

## 仓库结构

```
remote-workspace/
├── SKILL.md              # 工作流：触发条件、会话事实、步骤 0–7、规则、验收清单
├── references/
│   └── recipes.md        # 五段逐字可用的命令模板，按 § 编号被引用
├── README.md             # 中文说明（默认显示）
├── README.en.md          # English
└── LICENSE               # MIT
```

`SKILL.md` 说**做什么、什么时候做**；`references/recipes.md` 说**具体怎么做**。
复制一段 recipe，替换掉 `<host>`、`<ws>`、`<path>`、`<name>`、`<pattern>`、`<cmd>`，
它就能直接跑。这种拆分意味着你可以只改命令模板而不碰工作流，反过来也一样。

## 已在真机验证

`references/recipes.md` 里的每一段 recipe 都在提交前于一台真实 Linux 主机上端到端跑过
（bash、`python3.8`、无 `ripgrep`、4 卡 GPU），包括两个负向用例：原文不存在的编辑必须
失败且不动文件；被 kill 的任务必须报 `dead-without-exit-code` 而不是成功。

这次实测揪出并修掉了两个真实 bug，两者都写进了 recipe 本身：
用 `... & echo $! > pid` 启动任务会让调用方阻塞任务全程（启动必须套一层子 shell）；
停止任务时先 kill 父进程会让子进程变成孤儿（必须先给子进程发信号）。

## 参与贡献

欢迎提 issue 和 PR。对 recipe 的改动有两条硬规则：

1. **先在一台真机上跑过。** 没执行过的命令模板一文不值——请说明你在哪台主机、哪个 shell 上测的。
2. **保持 `SKILL.md` 与 `references/recipes.md` 同步。** 如果新增的工作流步骤需要命令，
   就加一个带编号的 recipe 小节，并用 § 编号引用它。

## 许可证

MIT —— 见 [LICENSE](LICENSE)。
