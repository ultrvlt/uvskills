# Ultraviolet Observability — agent skills

Skills and instructions that let your coding agent help you install and run
[Ultraviolet Observability](https://uvobservability.com): self-hosted
observability that runs entirely in your own Kubernetes cluster.

They describe Ultraviolet **0.1.143**, the current stable release
(see `VERSION`). The full documentation is at
[docs.uvobservability.com](https://docs.uvobservability.com).

## What's here

| Skill | Helps you |
| --- | --- |
| [`ultraviolet-install`](skills/ultraviolet-install/SKILL.md) | Size, verify and install the core, sign in, and turn on data sources. |
| [`ultraviolet-add-site`](skills/ultraviolet-add-site/SKILL.md) | Connect another cluster or region to your core, securely. |
| [`ultraviolet-upgrade`](skills/ultraviolet-upgrade/SKILL.md) | Upgrade the core and sites without losing settings or data. |
| [`ultraviolet-troubleshoot-sensor`](skills/ultraviolet-troubleshoot-sensor/SKILL.md) | Find out why a node's sensor isn't capturing, or is using too much memory. |
| [`ultraviolet-automatic-fixes`](skills/ultraviolet-automatic-fixes/SKILL.md) | Let runbooks fix incidents on their own, safely scoped. |

[`AGENTS.md`](AGENTS.md) is a short overview for agents that don't load skills.

## Using them

**Claude Code** — add this repo as a plugin marketplace and install the plugin:

```
/plugin marketplace add ultrvlt/uvskills
/plugin install ultraviolet@ultraviolet
```

**Other agents that read Agent Skills** — copy (or submodule) the `skills/`
folders into wherever your agent loads skills from, for example
`.claude/skills/` or `~/.codex/skills/`.

**Any agent** — point it at `AGENTS.md`, or copy it into your own repo's.

Everything is plain Markdown, so it works offline and air-gapped too.

## Safety

The skills change your clusters only through `helm` and `kubectl`, which you
run with your own credentials. They are written to show you what will change
(a rendered diff or a dry run) and ask before applying it, and never to ask you
to paste a license, token or password into the chat.

## Feedback

These files are generated from Ultraviolet's own repository with each stable
release, so pull requests here are overwritten. Open an issue instead, and
we'll fix it at the source.

## License

[MIT](LICENSE). Copy them into your own repos and change them as you like.
