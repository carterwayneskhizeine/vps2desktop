# bash completion (bash-completion)

| | |
|---|---|
| id | `bash-completion` |
| group | base |
| deps | none |
| channel | apt |

Programmable Tab completion for the stock `/bin/bash` login shell — argument,
option and subcommand completion for hundreds of third-party CLIs (docker,
git, systemctl, kubectl, …) without changing the user's shell.

Install it instead of switching to zsh when the goal is "make Tab work" but the
shell itself must stay bash (see Known Pitfalls for the upgrade path).

## Guard (idempotency)

```bash
dpkg -s bash-completion >/dev/null 2>&1 \
  && grep -qE '^[[:space:]]*\. /etc/bash_completion' ~/.bashrc \
  && echo active
```

Both halves matter: the package alone is not enough, see Pitfall 1.

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y bash-completion
```

Then enable it — Ubuntu's `~/.bashrc` ships with the sourcing block commented
out, so drop the leading `#` from all three lines:

```bash
if [ -f /etc/bash_completion ] && ! shopt -oq posix; then
    . /etc/bash_completion
fi
```

Both `/etc/bash_completion` (a one-line stub) and
`/usr/share/bash-completion/bash_completion` (the real thing) work; the stock
`.bashrc` block uses the former.

Activation is per-shell — an already-open session needs `exec bash` or a
reconnect.

## Verify

```bash
bash -ic 'complete -p -D' | grep -q _completion_loader && echo active
ls /usr/share/bash-completion/completions | wc -l
```

Expect `active` and a few hundred scripts (936 on Ubuntu 24.04).

## Known Pitfalls

- **Installing the package changes nothing on its own.** Ubuntu's `~/.bashrc`
  keeps the `. /etc/bash_completion` lines commented, so a fresh shell never
  loads it. The block is the only thing that has to be edited.
- **`complete -p <cmd>` reporting "no completion specification" does not mean
  completion is broken.** bash-completion registers a *default* handler
  (`complete -D -F _completion_loader`) that sources a command's script on the
  first Tab press; there is deliberately no per-command entry to find. Check
  the default handler, not an individual command.
- **`complete -D` without `-p` performs a completion instead of printing one**,
  so it exits silently. Always use `complete -p -D`.
- **Non-interactive shells never load it.** `~/.bashrc` opens with
  `[ -z "$PS1" ] && return`, so `bash -lc`, `ssh host cmd` and cron jobs do not
  get completion. Verify with `bash -ic`, not `bash -lc`.
- **`_EmptycmD_`/`_DefaultCmD_` errors from `complete -p -E`/`-D` are lookup
  artifacts**, not failures — ignore them.
- **bash stays bash.** No fuzzy history search, no inline autosuggestions, no
  syntax colouring, no error correction, no rich prompt — those are zsh (or
  oh-my-zsh) territory. This component only makes Tab useful.
- **zsh remains the larger upgrade** if the user is willing to keep bash as the
  *login* shell and drop into `zsh` interactively; that was evaluated and
  declined on light-node-01 in favour of this lighter component.

## Manual follow-ups

- none
