---
name: tabfs-browser-access
description: Access browser tabs as filesystem. Use when doing web/extension development, debugging pages, checking console logs, or inspecting DOM. Mount point at TabFS/fs/mnt/
---

# TabFS Browser Access

## Mount Point
`TabFS/fs/mnt/` (relative to this repo)

## Quick Check
```bash
ls TabFS/fs/mnt/tabs/by-id/
```

## Reading Browser State

| route | what it gives |
|-------|---------------|
| `tabs/by-id/[ID]/title.txt` | page title |
| `tabs/by-id/[ID]/url.txt` | current URL |
| `tabs/by-id/[ID]/text.txt` | page body text |
| `tabs/by-id/[ID]/document.html` | full page HTML |
| `tabs/by-id/[ID]/meta.json` | page metadata |
| `tabs/by-id/[ID]/console.json` | captured console output |
| `tabs/by-id/[ID]/errors.json` | JavaScript errors |
| `tabs/by-id/[ID]/selection.txt` | selected text |

## DOM Queries (for extension dev)

```bash
# create a watch for any JS expression
touch "tabs/by-id/[ID]/watches/document.querySelector('h1').innerText"
cat "tabs/by-id/[ID]/watches/document.querySelector('h1').innerText"

# twitter example
touch "tabs/by-id/[ID]/watches/document.querySelector('[data-testid=\"tweetText\"]').innerText"
cat "tabs/by-id/[ID]/watches/document.querySelector('[data-testid=\"tweetText\"]').innerText"
```

## Controlling Tabs

| action | command |
|--------|---------|
| navigate | `echo "URL" > tabs/by-id/[ID]/url.txt` |
| reload | `echo reload > tabs/by-id/[ID]/control` |
| close | `echo remove > tabs/by-id/[ID]/control` |
| new tab | `echo "URL" > tabs/create` |
| focus | `echo true > tabs/by-id/[ID]/active` |

## Execute JavaScript

```bash
# one-off expression via watches
touch "tabs/by-id/[ID]/watches/document.title"
cat "tabs/by-id/[ID]/watches/document.title"

# run script file via evals
echo "console.log('hello')" > tabs/by-id/[ID]/evals/test.js
```

## Console Capture

First read installs interceptor:
```bash
cat tabs/by-id/[ID]/console.json
# returns: [{"t":timestamp,"l":"log|error|warn","m":"message"},...]
```

## Finding Tabs

```bash
# by ID
ls tabs/by-id/

# by title (grep for keyword)
ls tabs/by-title/ | grep -i "twitter"

# focused tab
cat tabs/last-focused/title.txt
```

## Extension Development Workflow

```bash
# 1. find the tab you're working on
ls tabs/by-title/ | grep -i "mysite"

# 2. read console output
cat tabs/by-id/[ID]/console.json | jq .

# 3. query DOM elements
touch "tabs/by-id/[ID]/watches/document.querySelectorAll('button').length"
cat "tabs/by-id/[ID]/watches/document.querySelectorAll('button').length"

# 4. check for errors
cat tabs/by-id/[ID]/errors.json

# 5. reload after code changes
echo reload > tabs/by-id/[ID]/control
```

## Limitations

- No script injection on about: pages or strict CSP sites
- Console capture only starts AFTER first read
- Firefox only (Chrome debugger API not available)
