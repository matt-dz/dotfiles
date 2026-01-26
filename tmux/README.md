# tmux

Setup with [ohmytmux](https://github.com/gpakosz/.tmux).

Files are symlinked to the home directory

```sh
# Configs
ln -s $(realpath .tmux.conf) ~/.tmux.conf
ln -s $(realpath .tmux.conf.local) ~/.tmux.conf.local

# Plugins
mkdir -p ~/.tmux/plugins
ln -s $(realpath ./plugins/catppuccin/) ~/.tmux
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```
