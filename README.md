# apple-foundation-cc-plugin

A Claude Code plugin experiment (`fm-squeeze`). It uses Apple's on-device Foundation Model on macOS 27 to shrink large tool output before Claude reads it.

> **Status: early experiment, not usable yet.** This README describes the plan. Nothing here is ready to install. The project may be dropped if the measurements below show no benefit.

## The idea

Most of the tokens a coding agent spends go on *reading*: test logs, build output, lint reports. Most of that is noise, such as passing tests, progress bars and install chatter.

This plugin adds a Claude Code `PostToolUse` hook on the Bash tool. When a command's output is large:

1. The raw output is saved to `~/.fmo/raw/<session>/<id>.log`, so Claude can always open the full thing.
2. A **deterministic extractor** keeps every error line, stack frame and `file:line` reference **verbatim**, and drops the noise.
3. Optionally, **one** call to Apple's on-device model (via the `fm` CLI that ships with macOS 27) summarizes whatever is left.
4. Claude receives the compressed result plus the path to the raw log.

The plugin compresses context. It does not delegate coding tasks. The on-device model is small (a few thousand tokens of context) and is never asked to write code, run commands, or rewrite error lines.

## The question it answers

Does the Apple model add anything over plain deterministic filtering? The plugin has three modes ("arms"):

| Arm | What happens |
|---|---|
| `off` | Nothing. Claude sees the original output. |
| `det` | Deterministic extractor only. |
| `det+fm` | Deterministic extractor plus one on-device model call. |

There are two kill points:

1. **Offline gate:** replay 20–30 real saved logs. `det+fm` must succeed on about 95% of them and lose zero must-keep facts.
2. **Paired A/B:** identical headless Claude Code tasks run in each arm, compared on cost-weighted tokens, turns and success rate. If `det+fm` doesn't beat `det`, the model has no role here.

## Design rules

- **Fail open:** any error, timeout or unexpected input means Claude gets the original output, unchanged.
- **On-device only:** no Private Cloud Compute, no network calls.
- **No raw output in logs:** `~/.fmo/calls.jsonl` records only sizes, timings and outcomes.
- **Off by default:** you opt in with `FMO_ARM=det` or `FMO_ARM=det+fm`.

## Requirements (planned)

- A Mac with Apple silicon and Apple Intelligence enabled
- macOS 27, with the `fm` license accepted (`sudo fm license`)
- Node.js 22+ (already required by Claude Code)
- Claude Code

## Install (once it exists)

```sh
git clone https://github.com/jbknick/apple-foundation-cc-plugin
cd apple-foundation-cc-plugin
npm install && npm run build
```

Then, in Claude Code:

```
/plugin marketplace add /path/to/apple-foundation-cc-plugin
/plugin install fm-squeeze@fm-squeeze
```

Turn it on for a session:

```sh
FMO_ARM=det claude
```

Check what it did with `/offload-stats` inside Claude Code, or with `fmo stats` in a terminal.

## Configuration

`~/.fmo/config.json` (all fields optional; `FMO_HOME` moves the directory):

```json
{ "arm": "off", "thresholdBytes": 8000, "modelTimeoutMs": 8000, "maxModelInputTokens": 3000, "tailLines": 20 }
```

`FMO_ARM` overrides `arm`. `FMO_FM_BIN` overrides the path to `fm` (default `/usr/bin/fm`).

## Roadmap

- [ ] Spike: verify `fm` CLI behaviour and Claude Code hook output replacement
- [ ] Repo scaffold, `fm` adapter, deterministic extractor, call log
- [ ] Corpus capture tool and offline replay harness
- [ ] **Offline gate:** go or kill
- [ ] PostToolUse hook and `/offload-stats`
- [ ] **Paired A/B:** keep or kill

## Prior art

- [yrangana/fm-mcp](https://github.com/yrangana/fm-mcp): an MCP server and Claude Code plugin exposing summarise, extract, classify and OCR on the on-device model.

## Disclaimer

This project is not affiliated with or endorsed by Apple or Anthropic. "Apple Intelligence" and "Foundation Models" are Apple's; "Claude" and "Claude Code" are Anthropic's. Using `fm` is subject to Apple's own license terms, which you accept with `sudo fm license`.

## License

[MIT](LICENSE). Anyone can use, copy, modify and distribute this, including commercially. The only condition is keeping the copyright notice.
