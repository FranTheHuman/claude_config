# Claude Config

Personal Claude Code configuration: sub-agents, skills, and slash commands.

## Structure

```
.
├── agents/                  # Sub-agent definitions
│   ├── akka-master.md
│   ├── fe-master.md
│   ├── minio-master.md
│   ├── nginx-master.md
│   ├── nodejs-master.md
│   ├── quarkus-master.md
│   ├── quine-master.md
│   ├── react-master.md
│   └── tigerbeetle-master.md
│
├── skills/                  # Knowledge bases loaded into agents and skills
│   ├── akka-master/
│   ├── fe-architecture/
│   ├── minio-ops/
│   ├── nginx-ops/
│   ├── nodejs-best-practices/
│   ├── quarkus-expert/
│   ├── quine-architecture/
│   ├── react-best-practices/
│   ├── tigerbeetle-java/
│   └── tigerbeetle-ops/
│
└── commands/                # Slash commands grouped by domain
    ├── akka/                # /akka:review, /akka:specify, /akka:plan, /akka:new-component, /akka:diagnose, /akka:explain
    ├── fe/                  # /fe:review, /fe:architecture, /fe:ux-audit, /fe:explain, /fe:diagnose
    ├── git/                 # /git:commit, /git:commit-es
    ├── minio/               # /minio:health, /minio:backup, /minio:migrate, /minio:sdk, /minio:diagnose
    ├── nginx/               # /nginx:review, /nginx:setup, /nginx:harden, /nginx:add-service, /nginx:diagnose
    ├── nodejs/              # /nodejs:review, /nodejs:scaffold, /nodejs:new-feature, /nodejs:explain, /nodejs:diagnose
    ├── quarkus/             # /quarkus:review, /quarkus:new-endpoint, /quarkus:new-entity, /quarkus:explain, /quarkus:diagnose
    ├── quine/               # /quine:review, /quine:design, /quine:model, /quine:standing-query, /quine:deploy
    ├── react/               # /react:review, /react:scaffold, /react:new-feature, /react:explain, /react:diagnose
    └── tb/                  # /tb:review, /tb:scaffold, /tb:model, /tb:recipe, /tb:explain
```

## How it works

| Component | What it does | Location in `~/.claude/` |
|-----------|-------------|--------------------------|
| **Agent** | Specialized sub-agent with its own persona, tools, and skill set. Launched via the `Agent` tool. | `~/.claude/agents/` |
| **Skill** | Knowledge base (rules, patterns, best practices) loaded into a skill or agent at runtime. | `~/.claude/skills/<name>/` |
| **Command** | Slash command available in the Claude Code REPL (e.g. `/akka:review`). | `~/.claude/commands/<domain>/` |

---

## Step-by-step: install on a new machine

```bash
# 1. Clone the repo
git clone git@github.com:FranTheHuman/claude_config.git ~/Desktop/PERSONAL/claude_config
cd ~/Desktop/PERSONAL/claude_config

# 2. Back up existing ~/.claude config (optional but recommended)
cp -r ~/.claude ~/.claude.bak

# 3. Copy agents, skills, and commands into ~/.claude/
cp -r agents/*   ~/.claude/agents/
cp -r skills/*   ~/.claude/skills/
cp -r commands/* ~/.claude/commands/

# 4. Verify — open Claude Code and check available slash commands
claude
# Inside Claude: type / to see all available commands
```

---

## Step-by-step: pull latest changes

```bash
# 1. Pull latest version
git pull origin main

# 2. Re-sync to ~/.claude/
cp -r agents/*   ~/.claude/agents/
cp -r skills/*   ~/.claude/skills/
cp -r commands/* ~/.claude/commands/
```

> Tip: you can alias this in your shell:
> ```bash
> alias claude-sync="cd ~/Desktop/PERSONAL/claude_config && git pull \
>   && cp -r agents/* ~/.claude/agents/ && cp -r skills/* ~/.claude/skills/ && cp -r commands/* ~/.claude/commands/"
> ```

---

## Step-by-step: add a new skill

```bash
# 1. Create the skill directory and SKILL.md
mkdir -p skills/my-skill
touch skills/my-skill/SKILL.md
```

```markdown
# skills/my-skill/SKILL.md

---
name: my-skill
description: >
  One-line description of when this skill should be used.
license: MIT
---

You are an expert in ...
[Add rules, patterns, best practices, and examples here]
```

```bash
# 2. Test locally — copy to ~/.claude/ and open Claude Code
cp -r skills/my-skill ~/.claude/skills/

# 3. Reference the skill from an agent or command (optional)
# In agents/my-agent.md, add:  skills: [my-skill]

# 4. Commit and push
git add skills/my-skill/
git commit -m "feat(skills): add my-skill"
git push origin main
```

---

## Step-by-step: add a new agent

```bash
# 1. Create the agent file
touch agents/my-agent.md
```

```markdown
# agents/my-agent.md

---
name: my-agent
description: >
  Describe WHEN to use this agent so Claude routes to it automatically.
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet
skills:
  - my-skill        # skill files to preload
color: purple
---

You are a senior ...
[Agent system prompt — persona, workflow, hard rules]
```

```bash
# 2. Test locally
cp agents/my-agent.md ~/.claude/agents/

# 3. Commit and push
git add agents/my-agent.md
git commit -m "feat(agents): add my-agent"
git push origin main
```

---

## Step-by-step: add a new slash command

```bash
# 1. Create the command file under the relevant domain
touch commands/my-domain/my-command.md
```

```markdown
# commands/my-domain/my-command.md

# /my-domain:my-command — Short description

One sentence explaining what this command does.

## Instructions

1. Step one ...
2. Step two ...
3. Step three ...

## Output format

...
```

```bash
# 2. Test locally
mkdir -p ~/.claude/commands/my-domain
cp commands/my-domain/my-command.md ~/.claude/commands/my-domain/
# In Claude Code, run:  /my-domain:my-command

# 3. Commit and push
git add commands/my-domain/my-command.md
git commit -m "feat(commands): add /my-domain:my-command"
git push origin main
```

---

## Current agents

| Agent | Specialty |
|-------|-----------|
| `akka-master` | Akka Agentic Platform SDK — Agents, Workflows, Entities, Views, SDD |
| `fe-master` | Frontend architecture, UX/UI, design systems, Core Web Vitals |
| `minio-master` | MinIO on VPS — health, buckets, access control, SDK integration, migration |
| `nginx-master` | Nginx reverse proxy — TLS, hardening, rate limiting, Fail2ban, troubleshooting |
| `nodejs-master` | Node.js / Express APIs, async patterns, validation, Docker |
| `quarkus-master` | Quarkus reactive stack — Mutiny, Panache, Kafka, GraalVM |
| `quine-master` | Quine streaming graph database — standing queries, ingest, JSON-to-graph modeling |
| `react-master` | React 19 — App Router, Server Components, hooks, TanStack |
| `tigerbeetle-master` | TigerBeetle financial accounting database — data modeling, recipes, Java SDK, ops |

## Current skills

| Skill | Covers |
|-------|--------|
| `akka-master` | Akka SDK components, SDD workflow, testing, deployment |
| `fe-architecture` | Rendering strategies, atomic design, accessibility, conversion UX |
| `minio-ops` | MinIO operations, bucket policies, S3 SDK, backup, migration signals |
| `nginx-ops` | Hardened Nginx config, TLS 1.2/1.3, security headers, rate limiting, CVE tracking |
| `nodejs-best-practices` | Express, Zod, Pino, Jest, security middleware |
| `quarkus-expert` | Mutiny, Hibernate Reactive, Panache, Kafka/Pulsar |
| `quine-architecture` | Quine architecture, Three-Part System design, ID strategies, idempotency |
| `react-best-practices` | React 19 APIs, Zustand, TanStack Query, React Compiler |
| `tigerbeetle-java` | TigerBeetle Java SDK best practices, batches, error handling |
| `tigerbeetle-ops` | TigerBeetle cluster setup, replication, troubleshooting |

## Current commands

| Domain | Commands |
|--------|----------|
| `akka` | `/akka:review`, `/akka:specify`, `/akka:plan`, `/akka:new-component`, `/akka:explain`, `/akka:diagnose` |
| `fe` | `/fe:review`, `/fe:architecture`, `/fe:ux-audit`, `/fe:explain`, `/fe:diagnose` |
| `git` | `/git:commit`, `/git:commit-es` |
| `minio` | `/minio:health`, `/minio:backup`, `/minio:migrate`, `/minio:sdk`, `/minio:diagnose` |
| `nginx` | `/nginx:review`, `/nginx:setup`, `/nginx:harden`, `/nginx:add-service`, `/nginx:diagnose` |
| `nodejs` | `/nodejs:review`, `/nodejs:scaffold`, `/nodejs:new-feature`, `/nodejs:explain`, `/nodejs:diagnose` |
| `quarkus` | `/quarkus:review`, `/quarkus:new-endpoint`, `/quarkus:new-entity`, `/quarkus:explain`, `/quarkus:diagnose` |
| `quine` | `/quine:review`, `/quine:design`, `/quine:model`, `/quine:standing-query`, `/quine:deploy` |
| `react` | `/react:review`, `/react:scaffold`, `/react:new-feature`, `/react:explain`, `/react:diagnose` |
| `tb` | `/tb:review`, `/tb:scaffold`, `/tb:model`, `/tb:recipe`, `/tb:explain` |
