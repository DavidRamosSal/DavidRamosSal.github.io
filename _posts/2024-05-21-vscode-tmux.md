---
title: Truly persistent terminals inside VS Code with tmux
tags: [tmux, VS Code, Bash]
style: 
color: 
description: Using tmux and bash scripting to create an uninterrupted terminal experience inside VS Code.
---

There are two features I always find myself wanting when using VS Code's integrated terminals:

* a persistent state of the terminals I've created inside a given folder.
* a consistent recovery of the command history for each of the terminals in the folder.

Using tmux and some bash hackery I got to a terminal experience that has these two features (only in Linux) and I want to share how I went about it.

# tmux set up

The first step is to install [tmux](https://github.com/tmux/tmux). tmux is a window manager of sorts for your terminals, it allows you to (among many other things):
- create many windows (pseudo-terminals) inside a single terminal session
- keep terminal sessions running in the background, so that closing the terminal screen does not stop the programs running in the terminals

Most linux distributions offer prebuilt packages of tmux, see installation instructions [here](https://github.com/tmux/tmux/wiki/Installing).

tmux keeps the state of your terminals running in the background as long as the tmux server is running, but in the case of a reboot the state of the server is not saved. With the help of tmux plugins (tmux-resurrect and tmux-continuum) tmux can automatically restore the environment after a reboot (running programs on each terminal will have to be re run). 

To use tmux plugins you first need to install [tmux's plugin manager](https://github.com/tmux-plugins/tpm?tab=readme-ov-file#installation). Then install [tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect?tab=readme-ov-file#installation-with-tmux-plugin-manager-recommended) and [tmux-continuum](https://github.com/tmux-plugins/tmux-continuum?tab=readme-ov-file#installation-with-tmux-plugin-manager-recommended), in that order (as tmux-continuum depends on tmux-resurrect). 

With the plugins installed, you can set up automatic restore by:
* [activating automatic restore of your tmux environment](https://github.com/tmux-plugins/tmux-continuum?tab=readme-ov-file#automatic-restore)
* setting the interval (15 minutes in this example) at which you want the environment to be saved by adding the following to `.tmux.conf`: 
    ```
    set -g @continuum-save-interval '15'
    ```
* save your environment manually for the first time by executing the following inside of tmux:
```
prefix + Ctrl-s
```

## Command history restoration (optional)
Up until now, everything has been straightforward. However, command history can not be restored using tmux plugins as of now. Therefore, a bash script to manage bash command history globally is required. The following workaround is finnicky and very specific to my tmux workflow (which completely ignores the existence of panes and relies solely on the usage of sessions and windows inside of VS Code), so proceed with caution.

Create and save the the following script ([source](https://github.com/tmux-plugins/tmux-resurrect/issues/288#issuecomment-706772667)) in your home folder: `tmux-bash-history.sh`

```bash
# History control

# Avoids duplicates and commands starting with a space from being saved in the history file
export HISTCONTROL=ignoredups:erasedups:ignorespace

HISTS_DIR=$HOME/.bash_history.d
mkdir -p "${HISTS_DIR}"

# Function to get the appropriate history file based on tmux context
function getHistFile() {
    if [ -n "${TMUX_PANE}" ]; then
        echo "${HISTS_DIR}/history_tmux_$(tmux display-message -t $TMUX_PANE -p '#S:#I')"
    else
        echo "${HISTS_DIR}/history_non_tmux"
    fi
}

# Function to initialize history
function initHist() {
    HISTFILE=$(getHistFile)
    history -c
    history -r
    HISTFILE_LOADED=$HISTFILE
}

# Initialize history immediately on shell startup
initHist

# Function to update history file if necessary
function updateHistFile() {
    local CURRENTHISTFILE=$(getHistFile)
    if [[ "$CURRENTHISTFILE" != "$HISTFILE_LOADED" ]]; then
        history -w
        HISTFILE_LOADED=$CURRENTHISTFILE
    fi
}

# Ensure updateHistFile runs after each command
PROMPT_COMMAND="updateHistFile; history -a; ${PROMPT_COMMAND:-}"
```

Next, source the script by adding this to your `.bashrc`:

```bash
# Source the tmux-bash-history.sh script
if [ -f "$HOME/tmux-bash-history.sh" ]; then
  source "$HOME/tmux-bash-history.sh"
fi
```

This script saves the command history of each `session:window` pair of your tmux environment in individual files and an additional history file for commands executed outside of tmux. All of them stored in `~/.bash_history.d`. 

Due to the innerworkings of tmux-resurrect, this script doesn't correctly assign the history file to all of the windows in a session. To circumvent this, a tmux hook can be set to correctly initialize the history of all windows in a terminal session at start.

Create and save the following script in `~/.tmux/scripts/`: `reload_history_inactive.sh`

```bash
# Force tmux session to reinitialize history of inactive windows
SESSION_NAME=$(tmux display-message -p '#S')
ACTIVE_WINDOW_INDEX=$(tmux display-message -p '#I')
tmux list-windows -t "$SESSION_NAME" -F '#I' | while read -r WINDOW_INDEX; do
    if [[ "$WINDOW_INDEX" != "$ACTIVE_WINDOW_INDEX" ]]; then
        tmux send-keys -t "${SESSION_NAME}:${WINDOW_INDEX}" " clear" Enter
    fi
done
```

Finally, add the following to your `.tmux.conf` to set up the hook:
```
# Ensure tmux panes reinitialize history on window-linked
set-hook -g window-linked 'run-shell  "~/.tmux/scripts/reload_history_inactive.sh" ; set-hook -u window-linked'
```

Now indepedent command history tracking for each tmux window should work correctly from start.

# VS Code set up

Inside VS Code, the behaviour I wanted was a unique tmux session named after the base folder VS Code is running from. To achieve this, you can create a costum terminal profile by adding the following ([source](https://george.honeywood.org.uk/blog/vs-code-and-tmux/)) to your `settings.json`:

```json
{
    "terminal.integrated.profiles.linux": {
        "bash": null,
        "tmux": {
            "path": "bash",
            "args": ["-c", "tmux new -ADs ${PWD##*/}"],
            "icon": "terminal-tmux",
        },
    },
    "terminal.integrated.defaultProfile.linux": "tmux",
}
```

So that's it, now you will have unique tmux terminal sessions for each folder you open in VS Code and your tmux workspace and command history will be persistent even after you close VS Code or shutdown your computer.