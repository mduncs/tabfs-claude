# TabFS for Claude Code

browser tabs as a filesystem. `ls` your tabs. `rm` to close. `cat` to read content. pipe tab data to scripts.

## purpose

expose browser state to claude code for web development workflows:
- read tab URLs/titles/content during extension development
- close problematic tabs programmatically
- inject JS into pages for testing
- pipe page content to analysis tools

tabs appear at `TabFS/fs/mnt/` after mounting.

## architecture

```
shell commands (cat, ls, rm, echo)
         ↓
    FUSE kernel module (macFUSE)
         ↓
    fs/tabfs.c (thin C stub - stdin/stdout JSON)
         ↓
    browser extension (background.js - all the logic)
         ↓
    browser APIs (tabs, debugger, windows)
```

## filesystem structure

```
mnt/
├── tabs/
│   ├── by-id/
│   │   └── [TAB_ID]/
│   │       ├── url.txt          # read/write - navigate by writing
│   │       ├── title.txt        # read-only
│   │       ├── text.txt         # page text content
│   │       ├── body.html        # page HTML
│   │       ├── active           # write "true" to focus
│   │       ├── control          # write: reload, remove, goForward, goBack
│   │       ├── evals/           # drop JS files here to execute
│   │       └── watches/         # touch expression files, cat to read result
│   ├── by-title/
│   │   └── [TITLE].[TAB_ID]     # symlinks to by-id
│   ├── by-window/
│   │   └── [WINDOW_ID].[TITLE].[TAB_ID]
│   ├── last-focused             # symlink to active tab
│   └── create                   # write URL to open new tab
├── windows/
│   └── [WINDOW_ID]/
│       ├── focused              # read/write focus state
│       └── visible-tab.png      # screenshot
└── extensions/                  # installed extensions
```

## quick start

### 1. install macFUSE

```bash
# intel mac
brew install macfuse

# apple silicon - needs system extension approval
brew install macfuse
# then: System Settings → Privacy & Security → allow macfuse extension
# reboot required
```

### 2. compile the filesystem

```bash
cd TabFS/fs
mkdir -p mnt
make
```

### 3. install browser extension

**firefox (recommended for this project):**
1. open `about:debugging#/runtime/this-firefox`
2. click "Load Temporary Add-on"
3. select `TabFS/extension/manifest.json`

note: firefox loads extensions temporarily per-session. for permanent install, need to sign via AMO or use developer edition with `xpinstall.signatures.required = false`.

**chrome/brave/arc:**
1. open `chrome://extensions` (or equivalent)
2. enable Developer mode
3. "Load unpacked" → select `TabFS/extension/` folder
4. copy the extension ID (32-char string)

### 4. register native messaging host

```bash
cd TabFS

# firefox
./install.sh firefox

# chrome (replace YOUR_EXTENSION_ID)
./install.sh chrome YOUR_EXTENSION_ID

# brave
./install.sh brave YOUR_EXTENSION_ID

# arc
./install.sh arc YOUR_EXTENSION_ID
```

### 5. reload extension & mount

go back to extension page, hit reload. tabs appear in `TabFS/fs/mnt/`.

## usage examples

```bash
# list all open tabs
ls TabFS/fs/mnt/tabs/by-id/

# get all tab titles
cat TabFS/fs/mnt/tabs/by-id/*/title.txt

# close stackoverflow tabs
rm TabFS/fs/mnt/tabs/by-title/*Stack_Overflow*

# open new tab
echo "https://example.com" > TabFS/fs/mnt/tabs/create

# get page text for current tab
cat TabFS/fs/mnt/tabs/last-focused/text.txt

# inject JS - change background color
touch TabFS/fs/mnt/tabs/last-focused/watches/'document.body.style.background="green"'

# read a JS expression value
cat TabFS/fs/mnt/tabs/last-focused/watches/window.scrollY

# reload current tab
echo reload > TabFS/fs/mnt/tabs/last-focused/control

# screenshot a window
cp TabFS/fs/mnt/windows/*/visible-tab.png ~/Desktop/
```

## firefox vs chrome

| feature | firefox | chrome |
|---------|---------|--------|
| basic tab info | ✓ | ✓ |
| tab control | ✓ | ✓ |
| JS eval/watches | ✓ | ✓ |
| debugger API | ✗ | ✓ |
| page resources | ✗ | ✓ |
| permanent install | harder | easy |

firefox lacks `chrome.debugger` API so no access to network resources or script injection beyond eval. for extension development, this is usually fine.

## debugging

open extension background page inspector:
- firefox: `about:debugging` → TabFS → Inspect
- chrome: extensions page → TabFS → "Inspect views: background page"

console shows all filesystem operations and errors.

## claude code integration

for extension development, point claude at tab state:

```bash
# dump all tab URLs to a file claude can read
cat TabFS/fs/mnt/tabs/by-id/*/url.txt > ~/.claude/current-tabs.txt

# in CLAUDE.md or project instructions:
# "check ~/.claude/current-tabs.txt for current browser state"
```

or just use the paths directly when describing what's open.

## files

```
TabFS/
├── extension/
│   ├── background.js       # all route logic (~1000 lines)
│   ├── manifest.json       # browser extension manifest
│   └── vendor/
│       └── browser-polyfill.js
├── fs/
│   ├── tabfs.c            # FUSE implementation (~500 lines)
│   ├── Makefile
│   ├── mnt/               # mount point (create this)
│   └── vendor/
│       └── frozen.[ch]    # JSON parsing
├── install.sh             # native messaging setup
└── test/                  # test suite
```

## running it

1. **compile**: `cd TabFS/fs && make`
2. **install extension**: load `TabFS/extension/` in browser
3. **register host**: `cd TabFS && ./install.sh firefox` (or chrome + ID)
4. **reload extension**: in browser extension page
5. **access tabs**: `ls TabFS/fs/mnt/tabs/by-id/`

the `tabfs` binary runs automatically when the extension starts. if it fails, check:
- extension background console for errors
- that macFUSE is installed and allowed
- that `mnt/` directory exists

## troubleshooting

**"mount point does not exist"**
```bash
mkdir -p TabFS/fs/mnt
```

**"macfuse not found" on apple silicon**
- make sure you allowed the system extension in System Settings
- reboot after allowing

**"native messaging host not found"**
- re-run `./install.sh` from TabFS directory
- reload the extension

**tabs not appearing**
- check extension background console for errors
- make sure `fs/tabfs` binary is compiled
- verify native messaging manifest points to correct path

**permission denied**
- SIP may block FUSE. for development, shouldn't be an issue unless mounting system paths

## license

GPLv3 (upstream TabFS by Omar Rizwan)
