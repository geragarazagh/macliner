<picture><source media="(prefers-color-scheme: dark)" srcset="assets/icon-github-dark.png"><img src="assets/icon-github.png" width="128" alt="mac-clean icon: the delete key sign, a key pointing left with a cross inside"></picture>

# mac-clean: a skill for AI assistants that cleans, updates and checks a Mac

A free, open source agent skill for macOS. Ask the assistant to «clean up my Mac» («free some space», «update everything», «is my Mac secure?») and it measures the Mac first, shows a plan with sizes, deletes at once only what comes back by itself, asks before anything a person might want, installs the updates it was asked for, and measures again to check that everything planned was done.

- **Deleted without asking** (it comes back by itself or was already thrown away): app caches, app logs and crash reports, Trash items trashed more than a day ago, Homebrew's old versions and downloads, unfinished downloads, start-up items whose program was removed, Xcode build caches, device symbols of older iOS versions and simulators whose runtime is gone. Caches of open apps are skipped.
- **Asked first, with the size of each**: developer package caches (pip, npm, Yarn, Go and others), single caches over 1 GB, installers and 90-day-old items in Downloads, files over 1 GB, duplicate files (the oldest copy stays), data of apps that are no longer installed, apps not opened for 180 days (removed with their data), Recent Items lists, Safari history and website data, unsigned programs that start by themselves.
- **Updates**: Homebrew formulae and apps, App Store apps (with [mas](https://github.com/mas-cli/mas)); macOS updates are only named, since installing them restarts the Mac. A Homebrew app that is open is held back, because upgrading it quits it.
- **Checks**: System Integrity Protection, Gatekeeper, FileVault, XProtect's age, automatic security responses, automatic login, guest user, screen lock, device management, programs that accept network connections, shared folders; battery cycles and capacity, the internal disk's SMART status, heat throttling, kernel panics of the last 30 days.
- **Settings** that trade comfort for safety or privacy (firewall, stealth mode, AirPlay Receiver, Mac Analytics, automatic macOS updates, File Sharing, Remote Login) are named with their state and trade-off and changed only when the person asks. The baseline is how macOS ships.
- **Never touched**: Desktop, Documents, iCloud Drive and other cloud folders, the Photos, Music, TV, iMovie and Final Cut libraries, Mail, Messages, keychains, iPhone backups, SSH and GPG keys, tool settings in `~/.config` and `~/.local`, the folders of AI assistants and coding agents, virtual machines, Xcode simulators and archives, downloaded AI models, files on other disks. The helper script checks every path against this list again just before deleting it.

Works in assistants that run commands on the Mac itself: terminal agents such as Claude Code, Codex, Gemini CLI and Antigravity, OpenCode, OpenClaw, Hermes and others. Chat apps cannot reach the Mac and cannot use it. One file, `SKILL.md`, carries the method; the helper script `scripts/macclean.py` (Python 3 standard library, about 1,000 lines) does the measuring and deleting, so that the plan and the deletion are the same list.

## Install

Each way is marked by how it was checked: **run** means installed and used for real on the date in «Tested on»; **docs** means taken from the vendor's documentation on 2026-11-18 and not yet run.

### The tools it needs

Python 3 from Apple's command line tools (`xcode-select --install`; a Mac with Homebrew already has them). Optional: [Homebrew](https://brew.sh) for its cleanup and updates, and `brew install mas` for App Store updates. The assistant checks for them and, when one is missing, names the command and waits for a yes.

### Terminal agents

One command for every agent found on the computer (needs Node.js; `-g` installs for the user, without it for the current project only), **docs**:

```sh
npx skills add geragarazagh/mac-clean -g
```

It puts the skill into `~/.agents/skills/mac-clean`, which Codex, Cursor, Cline, OpenCode and most other agents read, and links it into `~/.claude/skills` for Claude Code.

Claude Code, as a plugin, **docs**:

```sh
claude plugin marketplace add geragarazagh/mac-clean
claude plugin install mac-clean@mac-clean
```

By hand, any agent that reads the Agent Skills folder (Codex, OpenCode, OpenClaw, Hermes, Grok Build), **docs**: copy `skills/mac-clean` to `~/.agents/skills/mac-clean`. Claude Code reads `~/.claude/skills/mac-clean`; Antigravity reads `~/.gemini/config/skills/mac-clean`.

### No skills in the agent

Paste the text of [`SKILL.md`](skills/mac-clean/SKILL.md) into the agent's instructions (`AGENTS.md`, `CLAUDE.md`, `GEMINI.md` or the agent's settings), or as the first message of a session. Without the script the assistant follows the shorter route by hand in the same text: measure, show the plan, delete only the listed paths, measure again.

### Full Disk Access

macOS hides Downloads, the Trash, Safari's data and the apps' own containers from a terminal until it has Full Disk Access. Without it the plan skips them and says so. To include them: System Settings > Privacy & Security > Full Disk Access, turn it on for the terminal or the assistant's app, reopen that app.

## Use

Say what is wanted, in any language. To call the skill by name: `/mac-clean` in Claude Code (`/mac-clean:mac-clean` when installed as the plugin) and Antigravity; `$mac-clean` in Codex.

```
Clean up my Mac and update everything.
How much space can I free on this Mac?
Check my Mac's security settings and battery.
Remove the apps I haven't opened in months.
```

A run: the plan (30 seconds to 3 minutes, read-only), the lines that need no choice, one message with the choices and their sizes, the lines the person said yes to, the plan again to verify, a short report with the space freed. The helper script can also be run by hand:

```sh
/usr/bin/python3 skills/mac-clean/scripts/macclean.py plan
/usr/bin/python3 skills/mac-clean/scripts/macclean.py apply
/usr/bin/python3 skills/mac-clean/scripts/macclean.py apply --only dev-caches 'app:Some App.app'
```

Standing answers («always clear developer caches», «never ask about Safari», «never delete this folder») are kept, on request, in `~/.config/mac-clean/answers.json`.

## What this skill does on the computer

- Runs macOS's own commands (`du`, `find`, `mdls`, `mdfind`, `lsappinfo`, `system_profiler`, `diskutil`, `pmset`, `csrutil`, `spctl`, `fdesetup`, `xprotect`, `netstat`, `sharing`, `softwareupdate --list`, `defaults read`, `codesign -v`, `xcrun simctl`), and Homebrew and mas when installed, through the helper script (read it first: one file, standard library only).
- Reads sizes in the home folder and its Library, the Trash and the start-up item folders, and looks for files over 1 GB outside Library and the never-touched folders. Hashes files of 1 MB and more of equal size to find duplicates. Writes the plan to a file in the system's temporary folder, and `~/.config/mac-clean/answers.json` only when asked.
- Deletes for real (no Trash), only the paths of plan lines that need no choice or that the person said yes to, each checked again against the never-touched list.
- Installs Homebrew and App Store updates only when the person asked for updates. Never installs macOS updates, never restarts the Mac, never changes a setting the person did not ask to change.
- Never asks for or types a password and never runs `sudo` without a password the system already holds; what needs an administrator is printed as commands for the person.
- Connects to nothing itself. The update checks (`brew outdated` reads Homebrew's local index; `mas outdated` and `softwareupdate --list` ask Apple) and the updates are the only network use. No telemetry, no accounts, no keys. Nothing keeps running after the run.
- The installers are other people's programs with their own rules: `npx skills` (Vercel) sends an anonymous install count unless `DISABLE_TELEMETRY=1` is set.

## Tested on

2026-11-18, by hand on an Apple M3 Pro MacBook Pro with macOS 27.2.1, Apple's Python 3.9.6, Homebrew 7.0.14 and mas 7.0.0:

- The plan on the real Mac, read-only: 31 seconds, every line compared with the output of the command behind it (the settings, the listening programs, the battery, the disk, the updates); the same plan with Full Disk Access treated as off: Downloads, the Trash, Safari and app containers left out, 21 seconds.
- `apply` in a test home folder: a cache, a log, an unfinished download, a start-up item of a removed program, a developer cache, a Trash item, an installer, one copy of a duplicate pair and a 1.1 GB file with a space in its name were deleted; the older copy of the duplicate stayed; a file on the Desktop and a link into the Desktop, put into the plan by hand, were refused. A plan older than 3 hours and an id not in the plan were refused. Standing answers changed the plan as expected.

Then in a test virtual machine (macOS 27.2, a fresh user account, no Homebrew), with caches, logs, a Trash item, developer caches, an installer, a duplicate pair, an old download and a Desktop file planted: `plan`, `apply`, `apply --only` with the choices, `plan` again. The freed sizes reported by the script matched the sizes measured before deleting, the Desktop files and the duplicate's original kept the same checksums, no password was asked, no line failed, and the second plan listed nothing left to clean.

Not tested: a run by a model, other agents, `apply` on a Mac in daily use, Intel Macs and older macOS versions, a Mac really without Full Disk Access, the installs marked **docs**.

## What it cannot do

- Install macOS updates or upgrades: they restart the Mac, so the person installs them in System Settings.
- Scan for malware beyond reporting XProtect's state, or remove malware.
- Free «purgeable» space held by Time Machine's local snapshots: macOS frees it by itself when the disk fills.
- Clean other disks, iCloud Drive or the Photos library, or decide which photos or documents to keep.
- Run on Windows or Linux.

## License

MIT
