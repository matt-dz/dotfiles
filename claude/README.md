# claude

Files should be symlinked to the `~/.claude` directory

```sh
mkdir -p ~./claude
ln -s $(realpath ./skills) ~/.claude
ln -s $(realpath ./CLAUDE.md) ~/.claude/CLAUDE.md
ln -s $(realpath ./settings.json) ~/.claude/settings.json
```
