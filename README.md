# FreeFlow Dictation for Claude Code

Answer Claude Code by voice. With this plugin installed, every moment a Claude Code session waits on you becomes a small card on your Mac, shown by [FreeFlow Dictation](https://freeflow.audio/claude-code): a plan to approve, a question to answer, a command to allow, or a finished turn to reply to. Click an option, or hold ⌥ Option and say it. The answer goes straight back into that session. The card never takes keyboard focus, so your cursor stays where you were typing.

Everything runs on your Mac. Speech is transcribed on-device by FreeFlow Dictation, and the plugin talks to Claude Code through files, not the network.

## What you get

| Claude Code moment | Card | How you answer |
| :- | :- | :- |
| A plan is ready for approval (`ExitPlanMode`) | Plan | Click Approve or Reject, or hold ⌥ and say "yes", "ship it", or what you want changed. Anything other than a clear yes goes back to Claude as feedback. |
| Claude asks you a question (`AskUserQuestion`) | Question | Click an option, or hold ⌥ and say the option. Speech that names no option goes through as a free-form answer. Cards with more than one question are click-only. |
| Claude needs permission for a tool (`PermissionRequest`) | Permission | Click Allow, Deny, or Always. Click-only on purpose: a mis-heard word must never run a command. Always writes a rule into that project's own `.claude/settings.local.json`, scoped to the exact command or file shown. |
| Claude finished its turn (`Stop`) | Finished | Hold ⌥ and dictate your next instruction without switching windows, or click to dismiss. |

Ignore a card for 45 seconds, or press Esc, and the prompt goes back to Claude Code's own window, exactly as if the plugin were not there.

## Requirements

- macOS 14 or later on Apple Silicon.
- [FreeFlow Dictation](https://freeflow.audio) **1.7.4 or later**, installed at `/Applications/FreeFlow.app` and running. The app notices the plugin by itself (Settings → Claude Code reads “Connected through the Claude Code plugin”), so you do not click Connect there. The plugin is a thin set of hook registrations; the program that does the work ships inside the app.
- Claude Code 2.1.292 or later for the one-line install below. Older versions use the two-step form.

The plugin does nothing on Linux or Windows, and nothing on a Mac without FreeFlow Dictation: every hook exits immediately and Claude Code prompts as usual.

## Install

One line, from your shell:

```bash
claude plugin install freeflow-dictation --marketplace freeflow-dictation/freeflow-dictation-plugins
```

Or in two steps:

```bash
claude plugin marketplace add freeflow-dictation/freeflow-dictation-plugins
claude plugin install freeflow-dictation@freeflow-dictation
```

Inside a Claude Code session, `/plugin marketplace add freeflow-dictation/freeflow-dictation-plugins` then `/plugin install freeflow-dictation@freeflow-dictation` does the same. New sessions pick the hooks up at startup; run `/reload-plugins` in a session that is already open.

Then open FreeFlow Dictation, go to Settings, and make sure Claude Code is turned on. A card appears the next time a session asks you something.

If your Mac has no GitHub SSH key, set `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1` in your shell before installing so Claude Code clones over HTTPS.

### The alternative: let FreeFlow register the hooks itself

FreeFlow Dictation can also write the same five hook entries directly into `~/.claude/settings.json`: open Settings in the app and click Connect next to Claude Code. That path needs no plugin and no marketplace. Use one or the other, not both; with both active, each prompt would reach FreeFlow twice.

## How it works

The plugin registers one command, the `freeflow-agent-hook` helper inside FreeFlow.app, on five Claude Code hook events:

| Event | Matcher | What happens |
| :- | :- | :- |
| `PreToolUse` | `ExitPlanMode\|AskUserQuestion` | Plan and question cards |
| `PermissionRequest` | all tools | Permission card |
| `Stop` | | Finished-turn card |
| `UserPromptSubmit` | | Housekeeping only, no card |
| `SessionEnd` | | Housekeeping only, no card |

Each hook has a 130 second timeout, longer than FreeFlow's own 45 second limit so that the app, not Claude Code, is always the one that decides.

When an event fires, Claude Code runs the helper and hands it the event as JSON on standard input. The helper:

1. Checks that FreeFlow Dictation is running and listening. FreeFlow holds an advisory lock on a file while it listens; if nobody holds it, the helper exits at once and prints nothing.
2. Writes the JSON, unchanged, into `~/Library/Application Support/FreeFlow/agent/inbox/` as one file per request. FreeFlow watches that folder and shows the card.
3. Waits for FreeFlow to write the decision into `~/Library/Application Support/FreeFlow/agent/responses/`, prints it on standard output, deletes both files, and exits.

The helper always exits 0. A non-zero exit from a hook is an error that Claude Code shows in the session, so every failure here, from FreeFlow not running to a timeout to malformed JSON, is the same quiet outcome: print nothing and let Claude Code ask you itself. If FreeFlow quits while a card is up, the helper notices within a second and yields. If FreeFlow never answers, the helper gives up after 120 seconds.

There is no port, no socket, and no network call anywhere in this path.

## Privacy

- The hook payload Claude Code sends contains the plan text, the question, the command or file about to be used, the working directory, and for a finished turn, Claude's last message. FreeFlow writes it to a local folder, shows it on the card, and deletes it when the card is answered or dismissed.
- Nothing leaves your Mac. The plugin adds no network call, and FreeFlow Dictation itself sends no telemetry. The app's complete list of network calls is on [freeflow.audio/privacy](https://freeflow.audio/privacy).
- Your spoken answer is transcribed on-device like any other FreeFlow dictation.
- The plugin contains no code. It is two JSON files and this document; the helper it points at is signed and notarized as part of FreeFlow.app.

## Uninstall

```bash
claude plugin uninstall freeflow-dictation@freeflow-dictation
```

To drop the marketplace too:

```bash
claude plugin marketplace remove freeflow-dictation
```

Uninstalling removes the plugin's enable flag from `~/.claude/settings.json` and leaves the rest of the file as it was. If you used FreeFlow's own Connect button instead of the plugin, click Disconnect in the app's Settings; that removes exactly the entries it added.

## Troubleshooting

**No card appears.** In order: is FreeFlow Dictation running (menu bar icon)? Is Claude Code turned on in its Settings? Does `claude plugin list` show `freeflow-dictation@freeflow-dictation` as enabled? Does `claude plugin details freeflow-dictation` list five hooks? Start a new session, or run `/reload-plugins` in the open one. If a card still does not appear, look for lines starting `agent:` in `~/Library/Application Support/FreeFlow/Logs/`.

**Claude Code shows a hook error such as "No such file or directory".** The plugin points at a fixed path, `/Applications/FreeFlow.app/Contents/MacOS/freeflow-agent-hook`. Move FreeFlow.app back into `/Applications`, or use the in-app Connect button instead, which records wherever the app currently lives.

**Every prompt produces two cards.** Both the plugin and FreeFlow's Connect are active. Click Disconnect in FreeFlow's Settings, or uninstall the plugin, and keep one.

**The card disappeared on its own.** After 45 seconds an unanswered card yields and Claude Code prompts in its own window. To change the limit, quit FreeFlow and run `defaults write com.freemans.freeflow agentYieldSeconds -int 90` (any number of seconds up to 120), then start it again.

**A permission card ignores my voice.** By design. Permissions are click-only because the thing on the other side runs a command.

**Install fails with a git or SSH error.** Set `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1` and try again, or add the marketplace by URL: `claude plugin marketplace add https://github.com/freeflow-dictation/freeflow-dictation-plugins`.

## Links

- FreeFlow Dictation for Claude Code: [freeflow.audio/claude-code](https://freeflow.audio/claude-code)
- Privacy: [freeflow.audio/privacy](https://freeflow.audio/privacy)
- Support: [support@freeflow.audio](mailto:support@freeflow.audio)
- Marketplace repository: [github.com/freeflow-dictation/freeflow-dictation-plugins](https://github.com/freeflow-dictation/freeflow-dictation-plugins)

## Licence

This plugin, the hook registrations and documentation in this repository, is released under the MIT licence; see [LICENSE](LICENSE). FreeFlow Dictation itself, including the `freeflow-agent-hook` helper the plugin points at, is a separate commercial product and is not covered by this licence.
