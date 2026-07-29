# vscode-clone

Make a second VS Code on macOS — a real, separate app with its own Dock icon,
its own settings, and its own extensions. Your original VS Code is never touched.

Useful for keeping work and personal setups apart, running a clean profile for
screencasts or pairing, or testing an extension set without polluting your daily
editor.

```bash
vscode-clone work "VS Code (Work)"
```

You now have `/Applications/Code-work.app`. Open both at once; they don't share
anything.

## Install

```bash
sudo mkdir -p /usr/local/bin && sudo curl -fsSL https://raw.githubusercontent.com/AmirDaliri/vscode-clone/main/vscode-clone -o /usr/local/bin/vscode-clone && sudo chmod +x /usr/local/bin/vscode-clone
```

`/usr/local/bin` is on the default macOS `PATH` but is root-owned (and often missing
on Apple Silicon), hence `sudo`. Without it curl fails with
`(56) Failure writing output to destination`.

Prefer no sudo? Install to `~/.local/bin`:

```bash
mkdir -p ~/.local/bin && curl -fsSL https://raw.githubusercontent.com/AmirDaliri/vscode-clone/main/vscode-clone -o ~/.local/bin/vscode-clone && chmod +x ~/.local/bin/vscode-clone
```

then add it to your `PATH` if it isn't already:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

**Requirements:** macOS, VS Code, and Xcode Command Line Tools (`xcode-select --install`)
— the tool compiles a small launcher, so it needs `clang`.

## Usage

```
vscode-clone <slug> [display name]   create a clone
vscode-clone --list                  list clones
vscode-clone --update <slug>         rebuild from the updated original
vscode-clone --remove <slug>         delete a clone (keeps its profile data)
```

### Two Claude Code accounts on one Mac

Cloning VS Code does **not** by itself give you a second Claude Code login.
Claude Code keeps its auth in `~/.claude` and the macOS Keychain — outside VS Code
entirely — so every clone shares your existing account by default.

Pass `--separate-claude` and the clone gets its own:

```bash
vscode-clone alt "VS Code (Alt Claude)" --separate-claude
```

The launcher sets `CLAUDE_CONFIG_DIR` for that app only, which Claude Code treats
as a distinct auth scope — signing in there leaves your main login untouched, and
vice versa. Install the Claude Code extension in the clone and sign in with the
other account.

Works on VS Code, VS Code Insiders, and VSCodium. It picks the first one it
finds in `/Applications`; override with `VSCODE_APP`:

```bash
VSCODE_APP="/Applications/VSCodium.app" vscode-clone oss
```

Profile data lives in `~/Library/Application Support/VSCodeProfiles/<name>/`.
Removing a clone leaves that directory alone, so you can recreate the app and
keep your setup.

## How it works

VS Code is an Electron app, so a clone is a copy of the bundle with a new
identity and a launcher that forces it onto its own directories:

1. **Copy** the `.app` with `ditto`.
2. **Rewrite `Info.plist`** — new `CFBundleIdentifier`, `CFBundleName`,
   `CFBundleDisplayName`, and a new `CFBundleExecutable` pointing at the launcher.
3. **Rename the four helper bundles** and their inner executables to match.
4. **Rebrand `product.json`** so the clone's dot-directories are its own.
5. **Compile a launcher** that bakes in `--user-data-dir` and `--extensions-dir`
   (plus `CLAUDE_CONFIG_DIR` with `--separate-claude`), strips any caller-supplied
   copies, and `execv`s the real binary.
6. **Re-sign ad-hoc**, since every edit above invalidated Apple's signature.

Four details are load-bearing and each one breaks the clone in a different way:

- **`CFBundleName` must stay filesystem-safe.** Electron locates helpers as
  `<CFBundleName> Helper*.app`. Set it to a pretty name with spaces and parens
  and the app dies with `FATAL: Unable to find helper app`. The pretty name goes
  in `CFBundleDisplayName` instead.
- **Helper `Info.plist` files have no `CFBundleExecutable` key**, so `PlistBuddy Set`
  fails on them. `plutil -replace` creates the key and handles spaces in values.
- **`product.json` has three folder keys**, not one: `dataFolderName`,
  `sharedDataFolderName`, `serverDataFolderName`. Patch only the first and your
  clone quietly keeps sharing `~/.vscode-shared` with the real VS Code.
- **`--user-data-dir` alone is not isolation.** Extensions live outside it, so
  `--extensions-dir` has to be baked in too.

## Caveats

- **Auto-update is disabled in clones** (`"update.mode": "none"` is seeded into the
  clone's settings). VS Code's updater swaps in a fresh `.app`, which would restore
  the original `Info.plist` — and since that points `CFBundleExecutable` at a launcher
  the new bundle doesn't contain, the clone would stop launching entirely. After you
  upgrade the real VS Code, run `vscode-clone --update <slug>` to rebuild the clone
  from it; your settings, extensions and Claude login are preserved.
- **Clones are ad-hoc signed**, not notarized. They run fine locally; Gatekeeper may
  prompt on first launch.
- **Clones share the original's icon.** Drop in a different `.icns` if you want them
  visually distinct.
- **Don't redistribute the cloned `.app`.** Microsoft's VS Code builds are under a
  proprietary license. Sharing *this script* is fine — it only modifies a copy on
  your own machine. Shipping the resulting app to other people is not.

## Prior art

The technique is reverse-engineered from MadAppGang's *Claude Profiles*, which does
the same for the Claude and ChatGPT desktop apps. That app is closed-source and
hardcoded to those two, which is why this exists. No code is shared between them.

## License

MIT
