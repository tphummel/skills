---
name: modern-cli-tools
description: Use fd, rg, fzf, and bat as modern replacements for find, grep, interactive selection, and cat. Prefer these tools for faster, friendlier, and more ergonomic file searching, content searching, fuzzy filtering, and file viewing.
---

# Modern CLI Tools

Use `fd`, `rg`, `fzf`, and `bat` instead of their legacy counterparts. Each is faster, has sensible defaults (respects `.gitignore`, colorizes output, handles Unicode), and has a more consistent interface.

| Modern | Replaces | Purpose |
|--------|----------|---------|
| `fd`   | `find`   | Find files and directories |
| `rg`   | `grep -r` | Search file contents |
| `fzf`  | manual selection | Fuzzy-filter any list interactively |
| `bat`  | `cat`    | View file contents with syntax highlighting |

---

## fd — Find Files

**Install**: `brew install fd` / `apt install fd-find` (binary may be `fdfind`)

### Basic Usage

```bash
fd PATTERN               # find files matching pattern (name search)
fd PATTERN PATH          # search within a specific directory
fd -e py                 # find by extension
fd -t f                  # files only (-t d = dirs, -t l = symlinks)
fd -H                    # include hidden files
fd -I                    # include files ignored by .gitignore
fd -0                    # null-delimited output (for xargs -0)
```

### Common Replacements

```bash
# find . -name "*.md" -type f
fd -e md

# find . -name "*.log" -type f -mtime +7
fd -e log --changed-before 7d

# find . -name "*.py" -exec grep -l "import os" {} \;
fd -e py -x grep -l "import os"

# find . -type d -name __pycache__
fd -t d __pycache__

# find . -maxdepth 2 -name "*.json"
fd -e json --max-depth 2
```

### Execute Commands on Results

```bash
fd -e py -x wc -l        # run wc -l on each .py file
fd -e log -X rm          # delete all .log files (X = batch, x = one-by-one)
fd -e html -x sed -i 's/foo/bar/g'
```

---

## rg — Ripgrep (Search File Contents)

**Install**: `brew install ripgrep` / `apt install ripgrep`

### Basic Usage

```bash
rg PATTERN               # search in current directory recursively
rg PATTERN PATH          # search in specific path
rg -i PATTERN            # case-insensitive
rg -l PATTERN            # list matching files only
rg -c PATTERN            # count matches per file
rg -n PATTERN            # show line numbers (on by default)
rg -v PATTERN            # invert match
rg -w PATTERN            # whole-word match
rg -F PATTERN            # fixed string (no regex)
rg -U PATTERN            # multiline mode
```

### Filtering by File Type

```bash
rg PATTERN -t py         # Python files only
rg PATTERN -t js -t ts   # JS and TS files
rg PATTERN -T json       # exclude JSON files
rg --type-list           # list all known types
```

### Common Replacements

```bash
# grep -r "TODO" . --include="*.py"
rg "TODO" -t py

# grep -rn "def main" src/
rg "def main" src/

# grep -rli "import React"
rg -l "import React"

# grep -r "password" . --exclude-dir=.git
rg "password"            # .gitignore respected automatically

# grep -rn "^class " --include="*.rb"
rg "^class " -t ruby
```

### Useful Flags

```bash
rg PATTERN -A 3          # 3 lines after match
rg PATTERN -B 3          # 3 lines before match
rg PATTERN -C 3          # 3 lines context (before + after)
rg PATTERN --hidden      # search hidden files
rg PATTERN -g "*.md"     # glob filter
rg PATTERN -g "!vendor/" # exclude glob
rg PATTERN --no-ignore   # ignore .gitignore
rg PATTERN -j 4          # use 4 threads
rg PATTERN -o            # print only the matching part
rg PATTERN --stats       # show search statistics
```

---

## fzf — Fuzzy Finder

**Install**: `brew install fzf` / `apt install fzf`

`fzf` reads from stdin and lets the user interactively fuzzy-filter the input. It prints the selected item(s) to stdout, making it composable with any other command.

### Basic Interactive Use

```bash
fzf                      # fuzzy-search stdin, print selection
fzf -m                   # multi-select (Tab to mark, Enter to confirm)
fzf -q "initial query"   # start with pre-filled search
fzf --height 40%         # render inline (not full-screen)
fzf --reverse            # input at top
fzf --border             # draw border
```

### Find and Open Files

```bash
# interactively select a file and open it
$EDITOR $(fzf)

# use fd as the source (respects .gitignore, faster)
FZF_DEFAULT_COMMAND='fd -t f' fzf

# preview file contents while browsing
fzf --preview 'bat --color=always {}'
```

### Search and Jump

```bash
# interactively search git log and show diff
git log --oneline | fzf --preview 'git show {1}'

# pick a branch to checkout
git branch | fzf | xargs git checkout

# kill a process interactively
ps aux | fzf -m | awk '{print $2}' | xargs kill

# cd into a directory chosen interactively
cd $(fd -t d | fzf)
```

### Pipe Integration

```bash
# grep through rg results interactively
rg --line-number "TODO" | fzf

# select from history
history | fzf | sh

# select multiple files for git add
git status -s | fzf -m | awk '{print $2}' | xargs git add
```

### Shell Integration

After `$(brew prefix)/opt/fzf/install` or `fzf --bash`:
- `Ctrl-R` — fuzzy search shell history
- `Ctrl-T` — fuzzy-insert file path at cursor
- `Alt-C` — fuzzy cd into subdirectory

---

## bat — Better cat

**Install**: `brew install bat` / `apt install bat` (binary may be `batcat`)

### Basic Usage

```bash
bat FILE                 # view file with syntax highlighting + line numbers
bat FILE1 FILE2          # concatenate multiple files
bat -n FILE              # line numbers only (no other decorations)
bat -A FILE              # show non-printable characters
bat --plain FILE         # no decoration (plain output like cat)
bat --paging=never FILE  # no pager, stream directly
bat -l json FILE         # force a language for highlighting
bat --list-languages     # show all supported languages
```

### Common Replacements

```bash
# cat file.py
bat file.py

# cat -n file.py
bat -n file.py

# cat *.md | less
bat *.md

# cat package.json | jq
bat --plain package.json | jq   # or just: cat package.json | jq
```

### Use as a Pager or Previewer

```bash
# use bat as git's pager for diffs
git diff | bat -l diff

# use bat in fzf preview (colorized)
fzf --preview 'bat --color=always --style=numbers {}'

# set as default MANPAGER
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
```

### Useful Flags

```bash
bat --style=plain        # no line numbers, no git markers, no border
bat --style=numbers,grid # pick specific style components
bat --color=always       # force color even when piped (e.g., into fzf)
bat --wrap=never         # disable line wrapping
bat -r 10:20 FILE        # show only lines 10–20
bat --diff FILE          # show only changed lines (requires git)
```

---

## Combined Patterns

### Find → Search → Preview → Edit

```bash
# Find Python files with TODO, preview with bat, open selected in editor
rg -l "TODO" -t py | fzf --preview 'bat --color=always {}' | xargs $EDITOR
```

### Find Files and View

```bash
# Fuzzy-pick a file and view it with bat
bat $(fd -t f | fzf)
```

### Search Log Files

```bash
# Tail + fzf filter (static snapshot)
rg "ERROR" app.log | fzf
```

### Batch Rename

```bash
# Find and preview files before acting
fd -e txt -x bat --style=header {}
```

---

## Availability Check

Before using these tools in scripts, verify they are available:

```bash
command -v fd  || echo "fd not found: brew install fd"
command -v rg  || echo "rg not found: brew install ripgrep"
command -v fzf || echo "fzf not found: brew install fzf"
command -v bat || command -v batcat || echo "bat not found: brew install bat"
```

On Debian/Ubuntu `fd` may be `fdfind` and `bat` may be `batcat`. Check with `which fd fdfind bat batcat`.

---

## When to Stick with Legacy Tools

- **`grep`**: When the environment is guaranteed to lack `rg` (e.g., minimal Docker images, POSIX-only scripts).
- **`find`**: In portable shell scripts (`/bin/sh`) where `fd` cannot be assumed.
- **`cat`**: When piping raw bytes, using `-A` for non-printable inspection already provided by `bat -A`, or in pipelines where adding a pager would break things.
- **`less`/`more`**: For arbitrary paging; `bat` uses them internally anyway.
