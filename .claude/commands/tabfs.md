# TabFS Browser Access

**NOTE:** For full documentation and patterns, see the skill:
`.claude/skills/tabfs-browser-access/SKILL.md`

Use this skill when doing web development, extension debugging, or needing to inspect browser state.

## Mount Point

```
/Users/md/code/tabs-fs-claude/TabFS/fs/mnt/
```

## What I Can Do

### Read Browser State
| path | what it gives me |
|------|------------------|
| `tabs/by-id/*/title.txt` | all tab titles |
| `tabs/by-id/[ID]/url.txt` | tab URL |
| `tabs/by-id/[ID]/text.txt` | page body text |
| `tabs/by-id/[ID]/body.html` | body innerHTML |
| `tabs/by-id/[ID]/document.html` | full page HTML including head |
| `tabs/by-id/[ID]/selection.txt` | currently selected text |
| `tabs/by-id/[ID]/meta.json` | page metadata (title, description, charset, element counts) |
| `tabs/by-id/[ID]/console.json` | captured console.log/error/warn (installs interceptor on first read) |
| `tabs/by-id/[ID]/errors.json` | JavaScript errors from the page |
| `tabs/by-id/[ID]/inputs/[name].txt` | form input values |
| `windows/[ID]/visible-tab.png` | screenshot of window |

### Control Browser
| action | how |
|--------|-----|
| navigate tab | `echo "https://example.com" > tabs/by-id/[ID]/url.txt` |
| focus tab | `echo true > tabs/by-id/[ID]/active` |
| reload tab | `echo reload > tabs/by-id/[ID]/control` |
| close tab | `echo remove > tabs/by-id/[ID]/control` or `rm tabs/by-id/[ID]` |
| go back | `echo goBack > tabs/by-id/[ID]/control` |
| go forward | `echo goForward > tabs/by-id/[ID]/control` |
| open new tab | `echo "https://example.com" > tabs/create` |
| reload extension | `echo 1 > runtime/reload` |

### Execute JavaScript
| method | how |
|--------|-----|
| eval expression | `touch "tabs/by-id/[ID]/watches/document.title"` then `cat` it |
| run script | write JS to `tabs/by-id/[ID]/evals/myscript.js` |
| fill form input | `echo "value" > tabs/by-id/[ID]/inputs/[inputId].txt` |

## Common Workflows

### Extension Development
```bash
# see console output from my extension's page
cat tabs/by-id/[ID]/console.json | jq .

# check for JS errors
cat tabs/by-id/[ID]/errors.json | jq .

# get full page HTML to inspect DOM
cat tabs/by-id/[ID]/document.html

# reload extension after code changes
echo 1 > runtime/reload
```

### Debugging a Web Page
```bash
# find tab by title
ls tabs/by-title/ | grep -i "mysite"

# get metadata
cat tabs/by-id/[ID]/meta.json

# check what's selected
cat tabs/by-id/[ID]/selection.txt

# evaluate JS expression
touch "tabs/by-id/[ID]/watches/document.querySelectorAll('button').length"
cat "tabs/by-id/[ID]/watches/document.querySelectorAll('button').length"
```

### Batch Operations
```bash
# list all open tabs
for f in tabs/by-id/*/title.txt; do cat "$f"; done

# close all tabs matching pattern
rm tabs/by-title/*stackoverflow*

# save all tab URLs
cat tabs/by-id/*/url.txt > ~/all-tabs.txt
```

## Limitations

- **about: pages** - can't inject scripts into Firefox internal pages
- **strict CSP sites** - some sites block script injection (timeout error 110)
- **debugger API** - not available in Firefox (Chrome-only)
- **console capture** - only captures logs AFTER first read installs interceptor

## Finding Tabs

```bash
# by ID (stable)
ls tabs/by-id/

# by title (human readable)
ls tabs/by-title/

# current focused tab
cat tabs/last-focused/title.txt

# by window
ls tabs/by-window/
```

## Quick Reference

```bash
MNT="/Users/md/code/tabs-fs-claude/TabFS/fs/mnt"

# is tabfs mounted?
ls $MNT/tabs/

# find a tab
ls $MNT/tabs/by-title/ | grep -i "keyword"

# read console
cat $MNT/tabs/by-id/[ID]/console.json

# get page info
cat $MNT/tabs/by-id/[ID]/meta.json
```
