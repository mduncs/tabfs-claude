# Contributing / Development Guide

quick reference for extending TabFS.

## Setup After Clone

```bash
cd ~/code/tabs-fs-claude

# install macfuse (if not already)
brew install macfuse
# allow in System Settings → Privacy & Security, reboot

# compile
cd TabFS/fs && make

# install native messaging host
cd TabFS && ./install.sh firefox

# load extension
# firefox: about:debugging → Load Temporary Add-on → TabFS/extension/manifest.json

# verify mount
ls TabFS/fs/mnt/tabs/
```

## Adding New Routes

all routes are in `TabFS/extension/background.js`.

### Simple Read-Only Route (using existing helpers)

```javascript
// inside the IIFE around line 350, after body.html definition:

Routes["/tabs/by-id/#TAB_ID/my-new-file.txt"] = {
  description: `Description shown in /runtime/routes.html`,
  usage: 'cat $0',
  ...routeFromScript(`document.querySelector('h1')?.innerText || ''`)
};
```

`routeFromScript(code)` - executes JS in tab, returns result as file content.

### Route with Tab Metadata

```javascript
Routes["/tabs/by-id/#TAB_ID/info.txt"] = {
  description: `Tab info from browser API`,
  usage: 'cat $0',
  ...routeForTab(tab => `${tab.title}\n${tab.url}\n`)
};
```

`routeForTab(readFn, writeFn)` - reads from browser.tabs API, not page content.

### Writable Route

```javascript
Routes["/tabs/by-id/#TAB_ID/my-control"] = {
  async write({tabId, buf}) {
    const value = buf.trim();
    await browser.tabs.executeScript(tabId, {code: `doSomething("${value}")`});
    return {size: buf.length};
  },
  async truncate() { return {}; }
};
```

### After Adding Routes

1. reload extension: `echo 1 > TabFS/fs/mnt/runtime/reload`
2. verify route appears: `ls TabFS/fs/mnt/tabs/by-id/1/`
3. test it: `cat TabFS/fs/mnt/tabs/by-id/1/my-new-file.txt`

## Architecture

```
User runs: cat mnt/tabs/by-id/1/title.txt
              ↓
         macFUSE kernel
              ↓
         fs/tabfs.c (C, ~500 lines)
         - receives FUSE operations
         - converts to JSON: {"op":"read","path":"/tabs/by-id/1/title.txt",...}
         - sends via stdout to browser
              ↓
         extension/background.js (~1000 lines)
         - receives JSON via native messaging stdin
         - matches path to Route handler
         - executes handler (may call browser APIs or inject scripts)
         - returns JSON response via stdout
              ↓
         fs/tabfs.c
         - receives response
         - returns data to FUSE/kernel
              ↓
         User sees: "Google"
```

### Key Files

| file | purpose |
|------|---------|
| `extension/background.js` | all route handlers, the brain |
| `extension/manifest.json` | permissions, native messaging config |
| `fs/tabfs.c` | FUSE stub, JSON protocol |
| `fs/Makefile` | build config, macfuse paths |
| `install.sh` | registers native messaging host |

### JSON Protocol

request (tabfs.c → background.js):
```json
{"id":1,"op":"getattr","path":"/tabs/by-id/1/title.txt"}
```

response (background.js → tabfs.c):
```json
{"id":1,"st_mode":33188,"st_size":6}
```

operations: `getattr`, `open`, `read`, `write`, `readdir`, `readlink`, `mknod`, `unlink`, `truncate`, `release`

## Debugging

### Extension Console
1. `about:debugging#/runtime/this-firefox`
2. find TabFS → Inspect
3. all filesystem operations logged here

### Common Issues

**mount empty after reload**
- extension disconnected from native host
- check extension console for "DISCONNECTED" message
- remove and re-add extension

**timeout errors (110)**
- page has strict CSP blocking script injection
- try different tab (not about: pages)

**"native messaging host not found"**
- re-run `./install.sh firefox`
- check path in `~/Library/Application Support/Mozilla/NativeMessagingHosts/com.rsnous.tabfs.json`

### Rebuild After C Changes

```bash
cd TabFS/fs
make clean && make
# then reload extension
```

## Testing

```bash
cd TabFS/test
make
./test  # requires running browser with extension
```

or manual:
```bash
# basic sanity
ls TabFS/fs/mnt/tabs/by-id/
cat TabFS/fs/mnt/tabs/by-id/1/title.txt

# test writes
echo "https://example.com" > TabFS/fs/mnt/tabs/create
echo reload > TabFS/fs/mnt/tabs/last-focused/control

# test JS execution
touch "TabFS/fs/mnt/tabs/by-id/1/watches/1+1"
cat "TabFS/fs/mnt/tabs/by-id/1/watches/1+1"  # should return 2
```

## Local Enhancements (vs upstream)

we added these routes not in omar's original:

| route | purpose |
|-------|---------|
| `document.html` | full outerHTML including head |
| `selection.txt` | currently selected text |
| `meta.json` | page metadata as JSON |
| `console.json` | captured console.log/error/warn |
| `errors.json` | captured JS errors |

these are marked with `// === CLAUDE CODE ENHANCEMENTS ===` in background.js.

## Firefox vs Chrome

| feature | firefox | chrome |
|---------|---------|--------|
| basic routes | ✓ | ✓ |
| tab control | ✓ | ✓ |
| JS execution | ✓ | ✓ |
| debugger API | ✗ | ✓ |
| permanent install | needs signing | easy |

firefox can't access `debugger/resources` or `debugger/scripts` routes.
