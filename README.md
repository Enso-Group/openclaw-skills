# openclaw-skills

Custom **OpenClaw AgentSkills** for the Clawdi-hosted GTM agent. Each folder is one installable skill
(a `SKILL.md` brain + depth files the agent reads on demand).

| Skill | What it does |
|---|---|
| `gtm-campaign/` | the always-on autonomous GTM engine — Find → Warmup → First message → A/B → Engagement → Insights, across all ICPs. Reads strategy/settings from the control-plane DB and writes results back (token-scoped). |
| `reddit-engagement/` | transparent disclosed-expert Reddit engagement (map → observe → compliance → draft → human approves → post via Composio → track + learn). Called by `gtm-campaign`'s engagement stage. |

## Install into a Clawdi agent (Console → Terminal)
```bash
SKILLS_DIR="/root/.openclaw/plugin-skills"   # where this agent already loads skills from
cd "$SKILLS_DIR" && git clone https://github.com/Enso-Group/openclaw-skills tmp \
  && rm -rf gtm-campaign reddit-engagement \
  && cp -R tmp/gtm-campaign tmp/reddit-engagement . \
  && rm -rf tmp && ls -1
```
This **replaces** the generic bundled `gtm-campaign` with the autonomous engine and adds `reddit-engagement`.

## Update later
```bash
cd /root/.openclaw/plugin-skills && rm -rf tmp && git clone https://github.com/Enso-Group/openclaw-skills tmp \
  && cp -R tmp/gtm-campaign tmp/reddit-engagement . && rm -rf tmp
```

## Notes
- Skills reference secrets by **name only** (`PLATFORM_URL`, `PLATFORM_AGENT_TOKEN`, etc.) — **no secret values** are in this repo.
- The agent reads `social_engagement_actions` (its own rows) but every other agent-write table is **write-blind** (no SELECT); cross-run memory uses Clawdi memory.
- Human keeps **Approve** (engagement drafts) + **STOP** (`gtm_pipeline_settings.paused`).
