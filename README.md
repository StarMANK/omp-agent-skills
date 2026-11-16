# Agent skills for omp

Workflow skills for AI coding agents: disciplined protocols for hunting bugs and building features. Written for [omp](https://github.com/can1357/oh-my-pi) ([docs](https://omp.sh/docs), `SKILL.md` format), compatible with any agent that reads `<skill-dir>/SKILL.md` with YAML frontmatter.

## Layout

    skills/
      bug-hunting/SKILL.md
      feature-development/SKILL.md

Each skill is one directory containing one `SKILL.md`. Discovery is **non-recursive**: exactly one level under `skills/`. `skills/team/foo/SKILL.md` is not found.

## Install

Copy into the agent skills root:

    cp -r skills/* ~/.agents/skills/

or clone and copy in one step:

    git clone https://github.com/StarMANK/omp-agent-skills && cp -r omp-agent-skills/skills/* ~/.agents/skills/

omp discovers `~/.agents/skills/<name>/SKILL.md` automatically at startup (user-level) and `.agents/skills/<name>/SKILL.md` inside a project (project-level). Restart the session, then confirm:

    omp skill list --json

The new skills must appear with `source: "agents:user"` and an empty `warnings` array. A missing entry means the file is at the wrong directory depth or the frontmatter did not parse.

## Use

Only `name` + `description` live in the system prompt; the body is read on demand. omp invokes a skill by matching the task to the `description`, then reading `skill://<name>`. To force one, type `/skill:<name>` (needs `skills.enableSkillCommands`) or read the URL directly:

    read skill://bug-hunting

## Skills

| Skill | When it applies | First instruction |
| --- | --- | --- |
| `bug-hunting` | Crash, exception, wrong output, flaky test, race, regression | Reproduce before fixing; capture the failure verbatim |
| `feature-development` | New endpoint, command, module, integration, or behavior | Read the closest existing analogue before writing |
| `optimization` | Latency, throughput, memory, efficiency | Define target metrics and measure the baseline |

`optimization` lives at `~/.agents/skills/optimization/SKILL.md` on this machine and is not mirrored in this repo — it is user-owned.

## Frontmatter

| Field | Effect |
| --- | --- |
| `name` | Must match the directory name; used for `skill://<name>` and matching |
| `description` | The only body text visible before invocation — this is what the model matches on; start it with "Use when ..." |
| `allowed-tools` | Enforced by Claude Code, preserved as inert metadata by omp; use real tool names |

Also accepted by omp: `globs`, `alwaysApply`, `hide` (catalog-hidden but still reachable), `disableModelInvocation`, `enabled: false`. In omp, `globs` and `alwaysApply` are metadata only — they do not auto-apply a skill.
