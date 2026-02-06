# claude

Files should be symlinked to the `~/.claude` directory

```sh
mkdir -p ~./claude
ln -s $(realpath ./skills) ~/.claude
ln -s $(realpath ./AGENTS.md) ~/.claude/CLAUDE.md
ln -s $(realpath ./settings.json) ~/.claude/settings.json
```

# codex

Flies should be symlinked to `~/.codex`.

```sh
mkdir -p ~/.codex
mkdir -p ~/.agents
ln -s $(realpath ./AGENTS.md) ~/.codex/AGENTS.md
ln -s $(realpath ./skills) ~/.agents
```
