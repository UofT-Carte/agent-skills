# agent-skills

A collection of [agent skills](https://agentskills.io) for AI coding assistants
(Claude Code, Codex, Copilot CLI, Gemini CLI, and others).

## Skills

| Skill | What it's for |
|---|---|
| [`running-on-alliance-hpc`](skills/running-on-alliance-hpc) | Running experiments, jobs, and GPU/vLLM workloads on Digital Research Alliance of Canada HPC clusters (Nibi, Fir, Narval, Rorqual, Trillium, PAICE) — Slurm sbatch, MIG slices, module+wheelhouse environments, staging code/data, and diagnosing failed jobs. |
| [`choosing-an-alliance-cluster`](skills/choosing-an-alliance-cluster) | Deciding which Alliance cluster to run on or request an allocation for — by GPU model, MIG/FP8 support, interconnect, compute-node internet, allocation/account, and data locality. Includes a full per-cluster spec sheet. |

Both skills are MCP-aware: they look up authoritative, changing facts (quotas,
RGU/billing, MIG profiles, cluster specs) via the public **Alliance Docs MCP**,
since the official wiki (`docs.alliancecan.ca`) blocks automated web access. The
skill offers to install it when it isn't already connected:

```bash
claude mcp add --transport http alliance-docs https://alliance-docs-mcp.fly.dev/mcp/
```

## Install

Using the [`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
# install every skill in this repo
npx skills add UofT-Carte/agent-skills

# or install just one skill
npx skills add https://github.com/UofT-Carte/agent-skills/tree/main/skills/running-on-alliance-hpc
npx skills add https://github.com/UofT-Carte/agent-skills/tree/main/skills/choosing-an-alliance-cluster
```

Or copy a skill directory straight into your assistant's skills folder
(e.g. `~/.claude/skills/` for Claude Code):

```bash
git clone https://github.com/UofT-Carte/agent-skills
cp -R agent-skills/skills/running-on-alliance-hpc ~/.claude/skills/
```

## License

[MIT](LICENSE)
